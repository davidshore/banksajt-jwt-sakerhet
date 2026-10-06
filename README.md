# Workshop – JWT och säkrare inloggning i banken

I den här workshopen fortsätter ni med banksajten. Nu ska ni göra inloggningen säkrare genom att:

1. spara lösenord som hashvärden i databasen
2. skapa en JWT när användaren loggar in
3. skicka JWT:n som en Bearer-token när frontend anropar banken
4. kontrollera token i backend innan banken lämnar ut saldo eller gör en insättning

Ni ska också uppdatera era tester och svara på några korta säkerhetsfrågor.

```text
Registrera användare
        ↓
Hasha lösenordet med bcrypt
        ↓
Spara hashvärdet i databasen

Logga in
        ↓
Jämför lösenordet med hashvärdet
        ↓
Skapa JWT
        ↓
Frontend skickar Authorization: Bearer <token>
        ↓
Backend verifierar token före skyddade anrop
```

## Mål

Efter workshopen ska du kunna:

- förklara varför lösenord aldrig ska sparas i klartext
- hasha och jämföra lösenord med bcrypt
- skapa en kortlivad JWT med ett användar-id i `sub`
- verifiera en JWT i Express innan en skyddad route körs
- skicka en Bearer-token från Next.js till Express
- förklara skillnaden mellan att avkoda och verifiera en JWT
- hålla hemligheter och lösenord utanför Git och ur källkoden

## Del 1 – Se vad som behöver ändras

Titta först på er nuvarande backend. I många av era projekt sparas lösenordet direkt i tabellen `users` och en inloggning skapar en sexsiffrig kod i tabellen `sessions`.

Det räcker för att förstå flödet, men det är inte ett säkert sätt att hantera riktiga användare:

- Ett läsbart lösenord i databasen blir synligt för alla som får tillgång till databasen.
- En kort numerisk kod har för få möjliga värden för att fungera som en säker inloggningstoken.

Ni ska ersätta lösenord i klartext med hashvärden och ersätta engångskoden med en JWT.

## Del 2 – Hasha lösenord med bcrypt

I `backend/` installerar ni bcrypt och JSON Web Token-biblioteket:

```bash
npm install bcryptjs jsonwebtoken
```

Importera paketen i er backend:

```js
import bcrypt from "bcryptjs";
import jwt from "jsonwebtoken";
```

### Registrering

När en användare registrerar sig ska ni kontrollera att användarnamn och lösenord har ett värde. Kräv också minst åtta tecken i lösenordet.

Hasha sedan lösenordet innan ni kör `INSERT` mot databasen:

```js
const passwordHash = await bcrypt.hash(password, 12);
```

Spara `passwordHash` i databasen. Ni kan använda er nuvarande lösenordskolumn om den bara innehåller hashvärden efter ändringen, men `password_hash` är ett tydligare kolumnnamn i en ny databas.

En bcrypt-hash ser ungefär ut så här:

```text
$2b$12$...
```

Den är avsiktligt lång och innehåller ett unikt salt. Två användare med samma lösenord ska därför inte få exakt samma hashvärde.

Om ni ändrar `database/init.sql` behöver ni också hantera er befintliga databas på datorn och på EC2. `init.sql` körs bara när MySQL-volymen skapas första gången. Återskapa därför er utvecklingsdatabas på Ec2 genom att ta bort den förra med:

```bash
docker compose down -v
```

### Inloggning

Vid inloggning hämtar ni användaren och dess hashvärde från databasen. Jämför sedan lösenordet från formuläret med hashen:

```js
const passwordIsCorrect = await bcrypt.compare(password, user.password_hash);

if (!passwordIsCorrect) {
  return res.status(401).json({ error: "Fel användarnamn eller lösenord" });
}
```

Skicka aldrig tillbaka lösenordet eller hashvärdet i ett API-svar. Använd samma felmeddelande oavsett om användarnamnet saknas eller lösenordet är fel.

Testa registreringen och kontrollera i databasen att ni ser en bcrypt-hash, inte lösenordet som skrevs i formuläret.

## Del 3 – Skapa och kontrollera JWT

En JWT består av:

```text
header.payload.signature
```

Payload är Base64url-kodad och kan läsas av den som har token. Den är alltså inte hemlig. Lägg aldrig lösenord, personnummer, bankuppgifter eller signeringsnyckeln i payloaden.

Signaturen gör att servern kan se om någon har ändrat token. Servern måste verifiera signaturen innan den litar på innehållet.

### Lägg signeringsnyckeln i miljövariabler

Skapa en lokal `.env`-fil i backendprojektet:

```dotenv
JWT_SECRET=klistra-in-en-slumpmassig-nyckel-har
```

Skapa ett eget slumpmässigt värde i terminalen:

```bash
openssl rand -hex 32
```

Lägg resultatet efter `JWT_SECRET=`. `.env` får inte checkas in i Git. Lägg den i `.gitignore` både i backendprojektet och i projektets rot om ni använder Docker Compose där.

När er Express-container körs via Docker Compose behöver den få nyckeln som miljövariabel:

```yaml
express:
  environment:
    JWT_SECRET: ${JWT_SECRET}
```

Använd ett annat, slumpmässigt värde för er testmiljö och ett eget värde på EC2. Skriv aldrig ut nyckeln i loggar, i README eller i GitHub Actions-workflowet.

### Skapa token vid korrekt inloggning

Efter `bcrypt.compare()` har lyckats ersätter ni den gamla session- eller OTP-koden med en JWT:

```js
const token = jwt.sign({ sub: String(user.id) }, process.env.JWT_SECRET, {
  algorithm: "HS256",
  issuer: "banken",
  audience: "banken-api",
  expiresIn: "15m",
});

res.json({ token });
```

`sub` är användarens id. `expiresIn` gör att token går ut efter 15 minuter. `issuer` och `audience` anger vilken app som skapade token och vilket API den är avsedd för.

Kontrollera att `JWT_SECRET` finns när servern startar. Servern ska avsluta med ett tydligt fel om nyckeln saknas eller är för kort:

```js
const secret = process.env.JWT_SECRET;

if (!secret || Buffer.byteLength(secret, "utf8") < 32) {
  throw new Error("JWT_SECRET måste vara minst 32 byte");
}
```

### Middleware för skyddade routes

Skriv själva en middleware-funktion som heter `requireAuth` och tar emot `req`, `res` och `next`. Funktionen ska kontrollera inloggningen innan en skyddad route får köras.

Implementera följande steg:

1. Läs headern `Authorization`, till exempel med `req.get("authorization")`. Kontrollera att den har formatet `Bearer <token>` och plocka ut själva tokenvärdet. Om headern saknas eller formatet är fel ska ni svara med status `401` och avbryta funktionen.
2. Verifiera token med `jwt.verify()` och er signeringsnyckel `secret`. Ange `algorithms: ["HS256"]`, `issuer: "banken"` och `audience: "banken-api"`, så att verifieringen använder samma inställningar som när token skapades. `jwt.verify()` kontrollerar också tokenens utgångstid.
3. Hantera verifieringen med `try`/`catch`. Om token är ogiltig eller har gått ut ska ni svara med status `401`, till exempel med felmeddelandet `"Ogiltig eller utgången token"`.
4. Kontrollera att den verifierade payloaden är ett objekt och att `sub` är en sträng med ett användar-id. Omvandla id:t till ett tal med `Number()` och kontrollera med `Number.isSafeInteger()` att det är ett giltigt heltal. Kräv också att id:t är större än noll. Svara med status `401` om kontrollerna misslyckas.
5. Lägg det kontrollerade användar-id:t i `req.userId`. Anropa sedan `next()` så att den skyddade routen får köras.

Anropa aldrig `next()` när någon kontroll har misslyckats. Avsluta funktionen direkt efter ett felsvar, till exempel med `return`, så att den inte fortsätter till routen.

Använd middleware-funktionen på alla routes som visar eller ändrar kontouppgifter. När ni har `req.userId` ska SQL-frågor använda det id:t. Ett id från request body får aldrig bestämma vilket konto som ska hämtas eller ändras.

Exempel:

```js
app.post("/me/accounts", requireAuth, async (req, res) => {
  const [accounts] = await pool.query(
    "SELECT balance FROM accounts WHERE user_id = ?",
    [req.userId],
  );

  // Svara med användarens saldo här.
});
```

Uppdatera även insättningsrouten på samma sätt. Token ska inte längre skickas i JSON-body.

## Del 4 – Skicka Bearer-token från frontend

När användaren har loggat in får frontend tillbaka `{ token }`. Spara den kortlivade access-token i React state, gärna i en `AuthContext`. Skicka sedan med den när frontend anropar banken:

```js
const response = await fetch(`${apiUrl}/me/accounts`, {
  method: "POST",
  headers: {
    Authorization: `Bearer ${accessToken}`,
    "Content-Type": "application/json",
  },
});
```

Gör samma ändring för insättning och andra skyddade anrop. Lägg också till en **Logga ut**-knapp som rensar token från React state och skickar användaren till inloggningssidan.

När token bara finns i React state försvinner den vid sidomladdning. Det är okej i den här uppgiften: användaren får logga in igen. Lägg inte token i `localStorage` för att få den att överleva en omladdning.

## Del 5 – Testa hela flödet

Testa lokalt före ni pushar:

1. Registrera en ny användare och kontrollera att databasen bara innehåller en hash.
2. Logga in med rätt lösenord och kontrollera att frontend får en JWT.
3. Kontrollera att kontosidan kan hämta saldo och göra en insättning med Bearer-token.
4. Logga ut och kontrollera att frontend inte längre kan göra skyddade anrop med den tidigare token som låg i state.
5. Ändra ett tecken i en egen JWT och prova ett skyddat anrop. Servern ska neka token.
6. Prova ett felaktigt lösenord. Servern får inte lämna ut en token.

Uppdatera testerna från förra workshopen så att de passar det nya flödet. Minst ett automatiskt test ska kontrollera att ett kontoanrop med en giltig Bearer-token fungerar och att ett anrop utan eller med en ändrad token nekas.

Se till att er GitHub Actions-kedja fortfarande kör lint, build och tester. Teststacken i Actions behöver en egen `JWT_SECRET` via en GitHub Secret eller en miljövariabel som bara används för tester. Använd aldrig produktionsnyckeln i tester.

## Del 6 – Säkerhetsfrågor

Svara kort i ert projekts README:

1. Varför kan du läsa en JWT-payload utan signeringsnyckeln, och vad skyddar signaturen?
2. Vad händer om någon stjäl en giltig token innan den går ut? Stoppar en signatur den personen?
3. Vad är skillnaden mellan att hasha ett lösenord och att signera en token?
4. Vilken risk finns med att lagra en token i `localStorage` om sidan får en XSS-sårbarhet?
5. Varför kan servern inte automatiskt veta att en JWT ska sluta gälla när användaren klickar på Logga ut?
6. Vilka hemligheter finns i ert projekt, och var ska de lagras lokalt, i GitHub Actions och på EC2?

## Inlämning och bedömning

Lämna in en länk till ert GitHub-repo. Lägg in följande i repots README:

- hur ni startar projektet lokalt
- vilka miljövariabler som behövs, utan deras verkliga värden
- hur ni kör testerna
- svar på säkerhetsfrågorna
- en länk till en grön GitHub Actions-körning

### VG – JWT i HttpOnly-cookie

För VG ska ni bygga om inloggningen så att frontend aldrig kan läsa JWT:n. I stället ska Express skicka token som en cookie när användaren loggar in.

G-delen använder Bearer-token för att göra JWT-flödet tydligt. I denna del byter ni till ett vanligt alternativ för webbappar:

```text
Logga in
    ↓
Express skapar JWT och skickar Set-Cookie
    ↓
Webbläsaren sparar cookien
    ↓
Webbläsaren skickar cookien automatiskt vid API-anrop
    ↓
Express läser och verifierar JWT:n från cookien
```

1. Ändra inloggningsrouten så att den sätter en cookie, exempelvis `access_token`, med JWT:n. Cookien ska ha `httpOnly: true`, `sameSite: "lax"` och samma livslängd som token. `secure` ska vara `true` i produktion, där sajten körs med HTTPS.

   ```js
   res.cookie("access_token", token, {
     httpOnly: true,
     secure: process.env.NODE_ENV === "production",
     sameSite: "lax",
     maxAge: 15 * 60 * 1000,
     path: "/",
   });

   res.json({ message: "Inloggningen lyckades" });
   ```

2. Ta bort token från JSON-svaret vid inloggning. Frontend ska inte spara token i React state, `localStorage` eller `sessionStorage`.
3. Låt frontend använda `credentials: "include"` i sina `fetch`-anrop. Konfigurera CORS i Express med frontendens exakta adress och `credentials: true`; använd inte `origin: "*"` tillsammans med cookies.
4. Ändra `requireAuth` så att den läser token från cookien och verifierar den på samma sätt som tidigare. Installera och använd `cookie-parser` om ni behöver läsa `req.cookies`.
5. Skapa en `POST /logout`-route som rensar cookien. Rensa cookien med samma grundinställningar, till exempel samma `sameSite` och `path`, som när den sattes.
6. Uppdatera frontend med en Logga ut-knapp som anropar `/logout` med `credentials: "include"` och skickar användaren till inloggningen.
7. Uppdatera era automatiska tester. De ska visa att en inloggad användare kan se sitt konto via cookien och att utloggning gör att kontot inte längre går att hämta i samma webbläsare.

### Vidare läsning

- [OWASP: lagring av lösenord](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [RFC 7519: JSON Web Token](https://www.rfc-editor.org/rfc/rfc7519)
- [RFC 8725: säkerhetsrekommendationer för JWT](https://www.rfc-editor.org/rfc/rfc8725)
- [jsonwebtoken: skapa och verifiera tokens](https://github.com/auth0/node-jsonwebtoken)
