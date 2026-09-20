# Materialuttag

En enkel webbapp för att registrera materialuttag ur förrådet — antingen mot en
befintlig arbetsorder eller som ett akututtag mot en maskin/kostnadsställe.
Körs som en enda fristående HTML-fil, fungerar offline och kan synka mellan
flera användare/skift via Firebase.

## Innehåll

- [Funktioner](#funktioner)
- [Kom igång (Firebase-uppsättning)](#kom-igång-firebase-uppsättning)
- [Inloggning](#inloggning)
- [Registret (maskiner & kostnadsställen)](#registret-maskiner--kostnadsställen)
- [Driftsättning (GitHub Pages)](#driftsättning-github-pages)
- [Använda appen](#använda-appen)
- [Offline-läge](#offline-läge)
- [Datastruktur i Firebase](#datastruktur-i-firebase)
- [Felsökning](#felsökning)
- [Tekniskt](#tekniskt)

## Funktioner

**Uttag akut**
- Sök upp maskin eller kostnadsställe (fritext, eller bläddra i "Alla KST" —
  sorterad efter vad som används mest/senast).
- EAMA-nummer: skriv bara siffrorna så läggs "EAMA" på automatiskt, öppnar
  siffertangentbord, eller skanna QR-koden med kameran.
- Antal uttaget med valbar enhet (St / Frp / Cm / Dm / M), samt valfritt
  "Antal kvar" och kommentar.

**Uttag med arbetsorder**
- Skapa en arbetsorder (kostnadsställe/maskinnr + valfri maskinbeskrivning)
  och gör sedan uttag mot den utan att behöva ange kostnadsstället varje gång.

**Dagens uttag**
- Lista över allt som registrerats. Varje rad kan justeras i efterhand
  (antal, enhet, antal kvar, kommentar) via kugghjulet, eller tas bort.
  Kostnadsställe och EAMA-nummer går inte att ändra i efterhand.

**Export till Excel**
- Riktig `.xlsx`-fil (inte CSV) med rätt kolumnbredder och kostnadsställe
  låst till textformat (så inledande nollor aldrig försvinner).
- Två lägen:
  - **Skiftets uttag** — exporterar och rensar bara det egna skiftets
    historik (den vanliga, dagliga avslutningen).
  - **Alla uttag – alla skift** — hämtar och slår ihop samtliga skift
    (2A/2B/3A/3B/3C/4A/4B/4C/4D/Dagtid) till en fil. Rensar **allt** för
    **alla** skift efter en tydlig bekräftelse. Är tänkt att bara användas
    av den som hämtar ut den kompletta rapporten för vidare bearbetning.

**Register (maskiner & kostnadsställen)**
- Sökningen i akututtaget bygger på ett register som ligger i Firebase,
  separat från själva uttagshistoriken.
- Under Inställningar (kugghjulet) kan man:
  - **Lägga till i register** — ny maskin eller nytt kostnadsställe.
  - **Ändra maskindata** — måste söka upp en befintlig maskin (går inte att
    skapa nya härifrån). Kostnadsställe kan bara väljas från de som redan
    finns i registret.
  - **Ta bort ur register** — sök upp och ta bort en maskin eller ett
    kostnadsställe, med bekräftelse.

**Inloggning**
- Ett delat lösenord skyddar hela appen och databasen (se
  [Inloggning](#inloggning)).

**Offline**
- Registret cachas lokalt på enheten så sökningen fungerar även utan nät.
  Se [Offline-läge](#offline-läge).

## Kom igång (Firebase-uppsättning)

Appen fungerar helt lokalt (ingen synk mellan enheter) tills du kopplar på
Firebase. Så här sätter du upp synk:

1. Skapa ett gratis projekt på https://console.firebase.google.com
2. **Build → Realtime Database** → skapa en databas (valfri region).
3. **Build → Authentication** → aktivera inloggningsmetoden **Email/Password**
   (se [Inloggning](#inloggning) för hur kontot sätts upp).
4. Projektinställningar (kugghjulet uppe till vänster) → **Your apps** →
   lägg till en webbapp (`</>`-ikonen) → kopiera `firebaseConfig`-objektet.
5. Klistra in värdena i `const firebaseConfig = {...}` i HTML-filen.
6. Sätt databasreglerna (**Realtime Database → Rules**):
   ```json
   {
     "rules": {
       ".read": "auth != null",
       ".write": "auth != null"
     }
   }
   ```
   Firebase varnar att detta "inte är säkert" (att alla inloggade får läsa/
   skriva allt) — det är förväntat och avsiktligt här, eftersom alla som
   loggat in med det delade lösenordet ska kunna se och skriva all data.
   Klicka **Publish anyway**.

Så länge `apiKey` i filen fortfarande står som `"YOUR_API_KEY"` körs appen
helt lokalt (ingen inloggning, ingen synk) — bra för att testa innan
Firebase är på plats.

## Inloggning

Appen använder **ett enda delat konto** för alla användare — det är alltså
inte tänkt att varje person ska ha ett eget konto.

1. **Authentication → Users → Add user** i Firebase-konsolen.
2. Mejladressen behöver inte vara riktig eller gå att nå — Firebase kräver
   bara att den *ser ut* som en e-postadress (`namn@nagot`). Den skickas
   aldrig mejl till. **Skriv den med enbart små bokstäver** — Firebase
   normaliserar annars adressen till gemener internt, vilket gör att den
   inte matchar om du skrivit versaler i filen.
3. Lösenordet är det som faktiskt skyddar datan — välj ett bra ett och håll
   det inom teamet.
4. Uppdatera `const SHARED_LOGIN_EMAIL = "..."` i HTML-filen till samma
   mejladress (i små bokstäver).

Lösenordet behöver bara anges en gång per enhet — sessionen sparas lokalt
i webbläsaren. Det finns en **"LOGGA UT"**-knapp under Inställningar.

**Om lösenordet glöms bort:** det går *inte* att återställa via mejl (eftersom
adressen inte är riktig). Enklaste lösningen: gå till Authentication → Users,
ta bort kontot, skapa ett nytt med nytt lösenord. Kontot är bara en nyckel för
att komma in — det är inte kopplat till någon data i sig, så det är
riskfritt att byta ut.

## Registret (maskiner & kostnadsställen)

Registret ligger i Firebase under två grenar, oberoende av
uttagshistoriken:

```
register/
  machines/
    <nyckel>: { machineNo, name, costCenter, manufacturer, department }
  costCenters/
    <nyckel>: { costCenter, name, department }
```

- `department` finns med i strukturen men används inte av appen idag.
- Flera poster kan dela samma `costCenter` (t.ex. flera maskiner på samma
  kostnadsställe, eller flera namngivna underplatser på samma
  kostnadsställe) — de får då unika nycklar, t.ex. `5340` och `5340_2`.

**Bulkimport:** Har du en stor mängd maskiner/kostnadsställen (t.ex. från ett
Excel-underlag) går det snabbast att importera en hel JSON-fil i rätt
struktur. **Viktigt:** importera den till just grenen `register` (högerklicka
på `register`-noden i Firebase-konsolen och välj Import JSON), **aldrig** vid
databasens rot — en import vid roten skriver över *hela* databasen, inklusive
alla skifts pågående uttagshistorik.

**Löpande ändringar** (en enstaka maskin i taget) görs enklast direkt i appen
via Inställningar → Lägg till/Ändra/Ta bort i register — det påverkar aldrig
resten av databasen.

## Driftsättning (GitHub Pages)

1. Lägg HTML-filen (döpt t.ex. `index.html`) i ett GitHub-repo.
2. Repot måste vara **publikt** för att GitHub Pages ska fungera på gratisnivå.
3. **Settings → Pages** → välj branch (oftast `main`) och mapp (`/root` eller
   `/docs`) → Save.
4. Sidan nås på `https://<användarnamn>.github.io/<reponamn>/`.

**Om sidan ger 404** efter att ha växlat repot mellan privat/publikt: det är
ett känt beteende där Pages-publiceringen "fastnar". Kolla att en källa
fortfarande är vald under Settings → Pages, och gör sedan en ny commit till
branchen — det brukar räcka för att trigga om publiceringen.

## Använda appen

- **Hemskärm:** Uttag akut, Lägg till arbetsorder, Visa dagens uttag,
  Exportera Excel.
- **Kugghjulet** (uppe till höger): Skift/Användare, Logga ut, samt de tre
  register-funktionerna.
- Innan man kan registrera ett uttag måste **Skift** och **Namn/signatur**
  vara ifyllda (efterfrågas automatiskt första gången om de saknas).

## Offline-läge

- **Registret** (maskiner/kostnadsställen) sparas i telefonens `localStorage`
  varje gång det synkas från Firebase. Appen laddar in den senaste sparade
  kopian direkt vid start, så sökningen fungerar även utan nät. En liten
  statusrad i akututtaget visar om registret är live-synkat, offline (kör på
  cache), eller ansluter.
- **Skiftets uttag/arbetsorder** kräver däremot att man är inloggad och
  ansluten mot Firebase för att synka — de är inte cachade på samma sätt.
- **Inloggningen** i sig kräver nät första gången, men sessionen sparas
  sedan lokalt så man slipper logga in på nytt varje gång.

## Datastruktur i Firebase

```
shifts/
  <skiftkod>/            (2A, 2B, 3A, 3B, 3C, 4A, 4B, 4C, 4D, Dagtid)
    orders/
      <nyckel>: { no, machine, done }
    withdrawals/
      <nyckel>: { time, article, order, machine, qty, unit, remaining,
                  comment, user, shift, acute, ts }
register/
  machines/<nyckel>: { machineNo, name, costCenter, manufacturer, department }
  costCenters/<nyckel>: { costCenter, name, department }
```

## Felsökning

**"XLSX is not defined" vid export**
SheetJS (biblioteket som gör Excel-filen) slutade publicera på npm/unpkg för
några år sedan. Filen pekar mot deras egna CDN
(`cdn.sheetjs.com`) — om det skulle sluta fungera i framtiden, kolla
https://docs.sheetjs.com för aktuell CDN-länk.

**Varningen om Firebase Dynamic Links i Authentication-inställningarna**
Kan ignoreras helt. Den gäller bara lösenordsfri "email link"-inloggning och
Cordova-OAuth — inte vanlig e-post/lösenord-inloggning, som är det appen
använder.

**"Your security rules are not secure"-varningen vid publicering av regler**
Förväntad och avsiktlig här (se [Kom igång](#kom-igång-firebase-uppsättning)).
Klicka "Publish anyway".

**"auth/invalid-email" vid inloggning**
`SHARED_LOGIN_EMAIL` i filen är inte korrekt formaterad eller matchar inte
kontot (se [Inloggning](#inloggning) om gemener/versaler).

**Ett registrerat uttag dyker inte upp i "Dagens uttag"**
Har hänt tidigare pga. att synken kopplades upp innan inloggningen hunnit
bekräftas — ska vara löst i nuvarande version (synken startar först efter
lyckad inloggning). Om det återkommer: kontrollera att databasreglerna
faktiskt är publicerade och att kontot har läs/skrivrättigheter.

## Tekniskt

- En enda HTML-fil (`index.html`) — ingen build-process, inget npm-projekt.
- Bibliotek laddas via CDN i `<head>`:
  - `html5-qrcode` — QR-skanning
  - `xlsx` (SheetJS) — Excel-export
  - `firebase-app-compat` / `firebase-database-compat` / `firebase-auth-compat`
    — synk och inloggning
- All egen kod (HTML, CSS, JS) ligger i samma fil.
- Data sparas i `localStorage` (lokalt läge, register-cache, KST-användning)
  och i Firebase Realtime Database (synkat läge).
