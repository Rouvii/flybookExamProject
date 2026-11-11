# FlyBook – sikkerhedsvurdering (eksamensprojekt, Software Security)

## Kontekst
Vi er en studiegruppe, der laver en sikkerhedsvurdering af FlyBook, en bevidst sårbar flybookingapplikation.
Rapporten skrives på dansk. Eksamen er mundtlig og individuel, så hvert fund skal forklares, så vi kan genfortælle det.

Stak:
- `web-app/`: statisk HTML/JS-frontend
- `api/API.php` og `api/database.php`: PHP-API med PDO mod MySQL
- `cgi-bin/status_check.cpp`: C++ CGI-program, der køres af Apache
- `Dockerfile` (php:7.3-apache) og `docker-compose.yml` (MySQL 8.0)
- `sql/flybook.sql`: database-seed

Appen kører lokalt på http://localhost:8000 (`docker-compose up -d`).

## Generelle regler
- Svar og skriv altid på dansk.
- Test KUN mod vores egen lokale instans (localhost). Aldrig mod eksterne systemer.
- Spørg, før du installerer software eller kører langvarige scanninger.
- Alt, hvad der findes, SKAL dokumenteres i `assessment/` som beskrevet nedenfor. Intet fund må kun nævnes i chatten.
- Efter hver fase: giv et kort resumé i chatten af, hvad der blev fundet, og hvilke filer i `assessment/` der blev opdateret.

## Linjenumre skal være stabile
- Alle fil- og linjehenvisninger refererer til den UÆNDREDE kode på `main`.
- Før gennemgangen starter: notér commit-hashen på `main` øverst i `assessment/findings.md` (`git rev-parse --short HEAD`).
- Henvis altid som `sti/til/fil:linje` eller `sti/til/fil:start-slut`, fx `api/API.php:22`.
- Dobbelttjek linjenumre med `grep -n` eller `nl -ba`, før de skrives ned.

## Dokumentation af fund (til rapporten)
Hvert fund får sin egen sektion i `assessment/findings.md` med fortløbende ID (F01, F02, ...) og PRÆCIS denne skabelon:

```
### F<nr>: <kort, beskrivende titel>

| Felt | Værdi |
|---|---|
| Placering | `fil:linje` (en linje pr. sted, hvis flere) |
| Komponent | Frontend / API / CGI / Database / Container |
| Fundet med | <værktøj og regel-ID> / manuel kodegennemgang / manuel pentest |
| Bekræftet dynamisk | Ja (hvordan) / Nej |
| CWE | CWE-<nr>: <navn> (link til cwe.mitre.org) |
| OWASP Top 10 | <kategori, hvis relevant> |
| Risiko | Informational / Low / Medium / High |
| Sandsynlighed / Konsekvens | <Lav/Middel/Høj> / <Lav/Middel/Høj> |
| Indsats for at rette | Lav / Middel / Høj |
| Rettet i | <commit-hash på fixes-branchen> eller "Ikke rettet" |

**Kodeuddrag** (de relevante linjer med linjenumre):
    <uddrag>

**Beskrivelse:** Hvad er problemet, og hvorfor opstår det i koden?

**Udnyttelse:** Hvordan kan en angriber i princippet udnytte det? Inkludér det request eller input, der blev brugt til at bekræfte det lokalt.

**Konsekvens:** Hvad kan angriberen opnå (fortrolighed, integritet, tilgængelighed)?

**Anbefaling:** Hvad bør ændres og hvordan, på overordnet niveau (ikke en fuld omskrivning).

**Til rapporten:** 3–5 sætninger på dansk, der kort opsummerer fundet, så det kan bruges som udgangspunkt i rapportteksten.

**Til eksamen:** 1–2 sætninger, der forklarer fundet helt enkelt, som man ville sige det mundtligt.
```

Derudover vedligeholdes:
- `assessment/overview.md`: én tabel med alle fund (ID, titel, fil:linje, CWE, risiko, indsats, rettet ja/nej), sorteret efter risiko.
- `assessment/tools.md`: hvert værktøj, version, præcis kommando, indstillinger/regelsæt, tuning, antal fund før og efter frasortering af falske positiver, og hvorfor de blev frasorteret.
- `assessment/sast/`, `assessment/dast/`: rå værktøjsoutput og konfigurationsfiler (bilag).
- `assessment/architecture.md`: komponenter, dataflow, trust boundaries og et Mermaid-diagram.
- `assessment/risk.md`: detaljeret risikoanalyse og prioriteret rækkefølge for rettelser.

## Faser (vi tager én ad gangen og venter på "fortsæt")

### Fase 0: Opsætning
1. Tjek, at Docker kører, og byg/start med `docker-compose build && docker-compose up -d`.
2. Bekræft, at http://localhost:8000, `/book.html` og `/service.html` svarer.
3. Opret `assessment/` med filerne ovenfor, og notér commit-hashen på `main`.

### Fase 1: Systembeskrivelse
- Kortlæg komponenter, dataflow og trust boundaries (browser → Apache/PHP → MySQL, browser → CGI).
- Hvordan sendes data? GET eller POST? HTTP eller HTTPS? Hvad ligger i URL'er?
- Hvilke aktiver findes (persondata, passwords, bookinger), og hvilket sikkerhedsniveau bør et bookingsystem have?
- Skriv det i `assessment/architecture.md`.

### Fase 2: Automatiseret statisk analyse (SAST, SCA, IaC)
Kør, gem output i `assessment/sast/`, og dokumentér i `assessment/tools.md`:
- Semgrep: `semgrep --config p/php --config p/owasp-top-ten --config p/secrets .`
- Flawfinder: `flawfinder cgi-bin/`
- cppcheck: `cppcheck --enable=all cgi-bin/`
- Hadolint: `hadolint Dockerfile`
- Trivy: `trivy config .` og `trivy image <web-imagets navn fra docker images>`
- gitleaks: `gitleaks detect --source .`

Sorter falske positiver fra, og forklar hvorfor. Relevante fund indføres i `findings.md`.
Lav en sammenligning: hvilke værktøjer fandt hvad, og hvad missede de?

### Fase 3: Manuel kodegennemgang
Gå alle filer igennem linje for linje, med særligt fokus på:
- Hvordan SQL-forespørgsler bygges i `api/API.php`
- Autentificering, API-nøgle og hashing af passwords
- Hvordan `cgi-bin/status_check.cpp` håndterer `QUERY_STRING` (input til shell, bufferstørrelser)
- Fejl- og undtagelseshåndtering, og hvad der lækker til brugeren
- Logging: hvad logges, og hvad mangler?
- Container og compose: credentials, root-brugere, eksponerede porte, EOL-versioner
- Frontend: output-encoding og XSS i `web-app/`
  Notér også, hvis et fund overlapper med et SAST-fund (henvis til ID'et i stedet for at oprette en dublet).

### Fase 4: Dynamisk analyse (DAST og manuel pentest)
- OWASP ZAP baseline/full scan mod http://localhost:8000. Gem rapport og konfiguration i `assessment/dast/`.
- Bekræft fund fra fase 2–3 dynamisk (fx sqlmap mod den lokale API-parameter, curl-requests mod CGI'en).
- Opdatér "Bekræftet dynamisk" og tilføj det præcise request som bevis.
- Notér fund, der kun ses dynamisk, og fund, der kun ses statisk.

### Fase 5: Risikoanalyse og prioritering
- Risikovurder hvert fund (sandsynlighed × konsekvens) og tildel Informational/Low/Medium/High med begrundelse.
- Lav en prioriteret liste i `assessment/risk.md`: kritiske fund, hurtige gevinster (lav indsats, høj effekt), og hvor vi ville starte.
- Opdatér `assessment/overview.md`.

### Fase 6: Rettelser (på separat branch)
Regler for rettelser:
1. Alle rettelser laves på branchen `fixes`, oprettet fra den noterede commit på `main`: `git checkout -b fixes <hash>`.
   `main` må ALDRIG ændres.
2. Én commit pr. fund. Commit-besked: `F<nr> (CWE-<nr>): <kort beskrivelse af rettelsen>`.
3. Hver ændring markeres tydeligt i koden med en kommentarblok over den ændrede kode:
   ```
   // [FIX F<nr> | CVE-<nr>] <hvad blev ændret og hvorfor>
   // ORIGINAL: <den oprindelige linje(r), udkommenteret>
   ```
   Brug kommentarsyntaksen for det pågældende sprog (`//` i PHP/C++/JS, `#` i Dockerfile/YAML, `<!-- -->` i HTML).
4. Efter hver rettelse: genbyg containeren, gentag det angreb, der bekræftede fundet, og vis, at det ikke længere virker.
   Notér resultatet.
5. Opdatér "Rettet i" i `findings.md` med commit-hashen.
6. Vedligehold `assessment/fixes.md` med en tabel: fund-ID, fil:linje (ny placering på `fixes`), commit-hash, hvad der blev ændret, og hvordan det blev verificeret.
7. Til sidst: generér `assessment/fixes.diff` med `git diff <main-hash>..fixes -- . ':!assessment'`, så alle ændringer kan ses samlet.

## Til sidst
Lav et tjek af, at rapportens krav er dækket: systembeskrivelse, mindst to SAST-værktøjer, manuel kodegennemgang,
mindst ét DAST-værktøj, fund med risikokategorier og CWE, anbefalinger, risikoanalyse og prioritering.
Skriv eventuelle huller i `assessment/overview.md`.