# Systembeskrivelse og arkitektur (fase 1)

Baseline: `main` @ `3da62b4`. Alle henvisninger peger på den uændrede kode.

## 1. Komponenter

| Komponent | Teknologi | Fil(er) | Rolle |
|---|---|---|---|
| Frontend | Statisk HTML/JS (jQuery, Bootstrap via CDN) | `web-app/index.html`, `web-app/book.html`, `web-app/service.html`, `web-app/js/util.js`, `web-app/vendor/` | Login, flysøgning, booking og statusside i browseren |
| API | PHP 7.3 + PDO | `api/API.php`, `api/database.php` | Login, søgning (lufthavne/fly) og booking mod MySQL |
| CGI | C++ (kompileret binær kørt af Apache) | `cgi-bin/status_check.cpp`, `cgi-bin/status_check` | Viser oppetid for MySQL og Apache via et shell-kald |
| Database | MySQL 8.0 | `sql/flybook.sql` | Passagerer, lufthavne, fly og bookinger |
| Container | Docker, `php:7.3-apache` + `mysql:8.0` | `Dockerfile`, `docker-compose.yml` | Kører web+CGI (port 8000) og DB (port 3307) |

Begge services kører i samme Docker-netværk (`localnetwork`). Web-containeren eksponerer port 80 som 8000 på værten; DB-containeren eksponerer 3306 som 3307 på værten.

## 2. Dataflow

Alle kald fra browseren er **GET over HTTP** (ingen HTTPS). Der er to adskilte veje ind i backend:

### Vej A – browser → Apache/PHP → MySQL
1. Browseren kalder `http://localhost:8000/api/API.php?API_KEY=<nøgle>&action=<get|login|book>&...`.
2. `API.php:4` tjekker, at `API_KEY` matcher den faste streng `Flight booker 1.2 plus edition`.
3. Alt efter `action` kaldes `Get()`, `Login()` eller `Book()`, som bygger SQL-strenge og kører dem via PDO (`database.php:8`).
4. Svaret returneres som JSON.

Konkrete kald fra frontenden:
- **Login** (`index.html:48-50`): `...&action=login&email=<email>&pass=<password>` — e-mail og password sendes i klartekst i URL'en.
- **Søg lufthavne** (`book.html:171-173`): `...&action=get&items=airport&search=<tekst>`.
- **Søg fly** (`book.html:222-226`): `...&action=get&items=flights&date=<dato>&from=<id>&to=<id>`.
- **Book** (`book.html:273-276`): `...&action=book&passenger=<id>&token=<token>&flight=<id>`.

Efter login gemmes brugerobjektet (inkl. `token`) i `localStorage` (`index.html:55`) og bruges til booking.

### Vej B – browser → Apache CGI (shell)
1. Statussiden kalder hvert 5. sekund `http://localhost:8000/cgi-bin/status_check?<proces>` (`service.html:70-90`, `util.js:4`), fx `...status_check?mysqld` og `...status_check?apache`.
2. Apache kører den kompilerede binær `status_check`.
3. Binæren læser `QUERY_STRING`, sammensætter en `ps ... | grep <QUERY_STRING> | ...`-kommando og kører den med `system()` (`status_check.cpp:9-13`).
4. Kommandoens stdout sendes retur og indsættes i siden med `.html(res)` (`service.html:77,86`).

## 3. Trust boundaries

| # | Grænse | Beskrivelse | Observationer |
|---|---|---|---|
| TB1 | Browser ↔ Apache/PHP | Netværksgrænse mellem klient og webserver | HTTP (ingen TLS); credentials og API-nøgle i URL; "auth" er en fast nøgle, der ligger i frontend-koden |
| TB2 | PHP ↔ MySQL | Applikation mod database | SQL bygges med strengsammensætning af brugerinput; DB-bruger er `root` |
| TB3 | Apache ↔ OS/shell (CGI) | CGI-program kører en shell-kommando | `QUERY_STRING` sættes direkte ind i en `system()`-kommando |
| TB4 | Vært ↔ container | Eksponerede porte | 8000 (web) og 3307 (MySQL) er åbne på værten |

Al validering og "beslutningstagning" sker på en betroet server, men inputtet krydser grænserne uden sanering. Den reelle angrebsflade er TB1 (alt input), TB2 (SQL) og TB3 (shell).

## 4. Aktiver og ønsket sikkerhedsniveau

| Aktiv | Hvor | Følsomhed |
|---|---|---|
| Persondata (navn, e-mail, telefon, pasnummer) | `passengers` i MySQL | Høj – persondata/GDPR |
| Passwords | `passengers.password` (MD5-hash) | Høj |
| Session-tokens | `passengers.token` (`md5(rand())`) | Høj |
| Bookinger | `bookings` | Middel |
| Fly- og lufthavnsdata | `flights`, `airports`, `airplanes` | Lav (offentligt) |

Et rigtigt bookingsystem håndterer persondata og betalingsrelaterede handlinger og bør derfor som minimum have: TLS på al trafik, parameteriserede SQL-forespørgsler, stærk password-hashing (bcrypt/argon2), rigtige sessioner/tokens med tilstrækkelig entropi, server-side autorisation, begrænsning af login-forsøg, inputvalidering og logning. FlyBook lever ikke op til nogen af disse, hvilket resten af vurderingen dokumenterer.

## 5. Diagram

```mermaid
flowchart LR
    subgraph Client["Browser (betroet zone: nej)"]
        FE["Frontend HTML/JS<br/>index/book/service.html<br/>util.js, localStorage"]
    end

    subgraph Host["Docker-vært (localhost)"]
        subgraph Web["Web-container (php:7.3-apache, :8000)"]
            API["PHP API<br/>API.php + database.php"]
            CGI["C++ CGI<br/>status_check"]
        end
        subgraph DB["DB-container (mysql:8.0, :3307)"]
            MySQL[("MySQL<br/>flybook")]
        end
    end

    FE -- "GET ?API_KEY&action=login/get/book<br/>(HTTP, credentials i URL)" --> API
    FE -- "GET ?QUERY_STRING<br/>(HTTP)" --> CGI
    API -- "PDO, SQL via strengsammensætning<br/>bruger: root" --> MySQL
    CGI -- "system(\"ps ... grep QUERY_STRING ...\")" --> Shell[["OS-shell i web-container"]]

    FE -.->|"TB1"| API
    FE -.->|"TB1"| CGI
    API -.->|"TB2"| MySQL
    CGI -.->|"TB3"| Shell
```

Stiplede linjer markerer trust boundaries (TB1–TB3). TB4 (vært ↔ container) er portene 8000 og 3307.
