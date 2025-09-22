# PasswordCracker — Master/Worker (distribueret)

> Et undervisningsprojekt til IT-sikkerhedskurset: en simpel distribueret master/worker password-cracker (ordbog + varianter).
>
> **VIGTIGT:** Dette projekt må kun bruges i et kontrolleret testmiljø eller på egne data. Misbrug (forsøg på at kompromittere andres konti eller systemer uden udtrykkelig tilladelse) er ulovligt og etisk forkasteligt.

---

## Kort beskrivelse
Dette repository demonstrerer en enkel master/worker-arkitektur:
- **Master**: Læser users/hashes og en ordbog, opdeler ordbogen i batches og sender arbejde ud til workers. Samler hits og logger dem.
- **Worker**: Forbinder til master, modtager batches med ord, genererer varianter, hasher og sammenligner med brugerhashes. Sender hits tilbage til master.
- **Shared**: Fælles modeller, simpel line-delimited JSON-protokol og hashing-hjælpere.

Formålet er at vise distribution, simpel protokoldesign og genbrug af cracking-logik.

---

## Repository struktur (eksempel)
```
/PasswordCrackerDistributed/
  /Shared/                 # fælles modeller + JsonLines + Hashing
  /Master/                 # Master server (konsol app)
  /Worker/                 # Worker client (konsol app)
README.md
LICENSE
```

---

## Forudsætninger
- .NET SDK (7/8 eller nyere). Tjek med `dotnet --version`.
- Kør på samme netværk eller samme maskine (referencen bruger ikke TLS).
- Testdata:
  - `passwords.txt` — format: `username:BASE64HASH` per linje (fx `alice:AbCdEf...`)
  - `webster-dictionary.txt` — ordbogsord, ét ord per linje

**OBS:** Sørg for at Base64-hashene er genereret med den samme hashing-funktion & byte-encoding som `Shared/Hashing.cs` (f.eks. SHA1 over ASCII/UTF8). Ellers matcher de ikke.

---

## Hurtig start — byg & kør
Byg hele solution:
```bash
dotnet build
```

Start Master (default port 5000):
```bash
cd Master
dotnet run -- --port 5000 --dict ../data/webster-dictionary.txt --pw ../data/passwords.txt --batch 500
```

Start Worker (på samme eller anden maskine i netværket):
```bash
cd Worker
dotnet run -- 127.0.0.1 5000
# eller
dotnet run -- <master-ip> 5000
```

Master logger fundne hits i konsollen:
```
HIT: alice => password123
```

Når alle batches er udleveret, sender Master `DONE` til workers og afslutter.

---

## CLI-options (eksempel)
**Master**
- `--port <port>` — port at lytte på (default: `5000`)
- `--dict <path>` — sti til ordbog
- `--pw <path>` — sti til user/hash fil
- `--batch <n>` — antal ord pr. batch

**Worker**
- Arg 1: master host (default `127.0.0.1`)
- Arg 2: master port (default `5000`)

---

## Protokol (kort)
Line-delimited JSON (én JSON-meddelelse per linje). Nogle vigtige meddelelsestyper:
- `HELLO` / `HELLO_OK` — handshake
- `PASSWORD_SET` — master sender list af users + hashes
- `READY` — worker beder om arbejde
- `WORK` — master sender `{ batchId, words[] }`
- `RESULTS` — worker sender fundne hits (`username` + `password`)
- `DONE` — ingen mere arbejde

Se `Shared/` for modeller og hjælpefunktioner til serialisering.

---

## Variants & Hashing
Variationslogikken (fx `Variants(word)`) ligger i Worker og kan indeholde:
- originalt ord, uppercase, capitalized, reversed, suffix/prefix tal (fx `word`, `Word`, `WORD`, `word1`, `1word`, `word01`, osv.)
Hashing sker i `Shared/Hashing.cs`. Sørg for at bruge samme encoding som i dine eksisterende udleverede filer (fx ASCII eller UTF8).

---

## Fejl-håndtering & forbedringsforslag
- Re-queue: re-indsæt batch i kø hvis worker disconnecter midt i et batch.
- Persistens: gem completed batch-IDs så master kan resume efter genstart.
- Sikkerhed: implementér TLS (SslStream) og simpel token-auth før accept af worker.
- Metrics: eksporter til Prometheus/logging for performance målinger.
- Parallelisme: lad workers processere flere tråde/kerner internt for at udnytte CPU.

---

## Etik & jura
Dette værktøj kan misbruges til skadelige formål. Brug kun:
- i isolerede undervisningslab miljøer
- på egne testdata
- eller med eksplicit, skriftlig tilladelse fra ejeren af data/conti

Jeg hjælper ikke med at angribe produktions-systemer eller konti du ikke ejer.

---

## Bidrag & videre arbejde
Hvis du vil have hjælp til:
- at tilpasse `Hashing` til præcis samme byte-encoding som i dine udleverede filer (så Base64-hashene matcher), eller
- at lave en `.sln` med tre projekter klar til build, eller
- at implementere re-queue/failover eller TLS/auth,

så skriv det i et issue eller åbn en PR — eller skriv lige her i chatten, så kan jeg levere de ændringer.

---

## Licens
Anbefalet: MIT. Sæt en `LICENSE` hvis du vil genbruge/udlevere projektet.

---

## Kontakt
Spørg her i repoets issues eller skriv i chatten hvis du vil have hjælp til at integrere din eksisterende `Cracking.cs`, `UserInfo*.cs` mv. i den distribuerede løsning.

