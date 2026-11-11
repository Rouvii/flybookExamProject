# FlyBook – fund

**Baseline:** `main` @ `3da62b4` (commit før CLAUDE.md). Alle fil- og linjehenvisninger refererer til denne commit.

Hvert fund følger skabelonen i CLAUDE.md. ID'er tildeles fortløbende (F01, F02, ...).

<!-- Fund indsættes nedenfor -->

---

### F01: Ingen begrænsning af login-forsøg (brute force muligt)

| Felt | Værdi |
|---|---|
| Placering | `api/API.php:7`<br>`api/API.php:54-78`<br>`sql/flybook.sql:4032-4040` |
| Komponent | API |
| Fundet med | manuel kodegennemgang + manuel pentest |
| Bekræftet dynamisk | Ja – 200 forkerte login-forsøg i træk mod `alice@its.dk` uden forsinkelse eller lockout, samt ordbogsangreb med 100 passwords pr. konto (se `assessment/dast/F01-bruteforce/`) |
| CWE | CWE-307: Improper Restriction of Excessive Authentication Attempts (https://cwe.mitre.org/data/definitions/307.html) |
| OWASP Top 10 | A07:2021 – Identification and Authentication Failures |
| Risiko | Medium (foreløbig – endelig vurdering i fase 5) |
| Sandsynlighed / Konsekvens | Høj / Middel |
| Indsats for at rette | Middel |
| Rettet i | Ikke rettet |

**Kodeuddrag** (`api/API.php`):

     7	        case "login":Login();break;
    ...
    54	function Login(){
    55	    global $conn;
    56	//    echo "1";
    57	    $records = $conn->prepare('SELECT * FROM passengers WHERE email="'.$_GET['email'].'" and password= "'.md5($_GET['pass']).'"');
    58	    $records->execute();
    59	    $results = $records->fetchAll(PDO::FETCH_ASSOC);
    60	    // var_dump( $results);
    61	    if(count($results) > 0) {
    ...
    73	    else {
    74	        $obj=["status"=>"450","statusDescription"=>"No user found"];
    75	        $jsonObj="[".json_encode($obj)."]";
    76	    }
    77	    echo $jsonObj;
    78	}

`sql/flybook.sql` – tabellen har ingen felter til fejlede forsøg eller spærring:

    4032	CREATE TABLE `passengers` (
    4033	  `id` int(11) NOT NULL,
    4034	  `name` varchar(255) NOT NULL,
    4035	  `email` varchar(255) NOT NULL,
    4036	  `phone` varchar(255) NOT NULL,
    4037	  `travel_document_no` varchar(255) NOT NULL,
    4038	  `password` varchar(255) NOT NULL,
    4039	  `token` varchar(255) NOT NULL
    4040	) ENGINE=InnoDB DEFAULT CHARSET=latin1;

**Beskrivelse:** `Login()` slår e-mail og password op og svarer med det samme, uanset hvor mange gange samme klient eller samme konto har fejlet før. Der findes ingen tæller for fejlede forsøg, ingen midlertidig spærring af kontoen, ingen stigende forsinkelse, ingen CAPTCHA og ingen begrænsning pr. IP – hverken i PHP-koden, i databasen (`passengers` har ingen kolonne til det) eller i Apache-opsætningen (`Dockerfile` aktiverer kun `rewrite` og `cgi`). Den eneste "beskyttelse" er API-nøglen i `api/API.php:4`, men den er en fast tekststreng, der ligger offentligt i frontenden (`web-app/js/util.js:1`), så den stopper ikke en angriber. Fejlede forsøg logges heller ikke af applikationen, så et angreb ville ikke blive opdaget. Positivt: svaret er det samme ("No user found") for forkert e-mail og forkert password, så man kan ikke direkte afgøre, om en e-mail findes.

**Udnyttelse:** En angriber kan sende et ubegrænset antal login-requests og afprøve passwords fra en ordbog eller lækkede password-lister (credential stuffing). Lokalt blev det bekræftet med:

    K='Flight%20booker%201.2%20plus%20edition'
    B="http://localhost:8000/api/API.php?API_KEY=$K&action=login"
    for i in $(seq 1 200); do
      curl -s -o /dev/null -w "%{http_code} %{time_total}\n" "$B&email=alice@its.dk&pass=forkert$i"
    done

Alle 200 forsøg blev besvaret med en gennemsnitlig svartid på 0,0032 s (ca. 300 forsøg pr. sekund fra én klient), og forsøg 201 fik stadig det normale svar `[{"status":"450","statusDescription":"No user found"}]`. Et efterfølgende ordbogsangreb med 100 almindelige passwords mod både `alice@its.dk` og `bob@its.dk` kørte uhindret igennem (intet match, fordi kontiene ikke bruger et password fra listen).

**Konsekvens:** Hvis en bruger har et svagt eller genbrugt password, kan angriberen overtage kontoen. Det giver adgang til passagerens persondata (navn, e-mail, telefon, pasnummer, jf. `passengers`-tabellen) og et gyldigt token, som kan bruges til at foretage bookinger i offerets navn (fortrolighed og integritet). Ubegrænsede requests kan desuden belaste API og database (tilgængelighed).

**Anbefaling:** Indfør begrænsning af login-forsøg, fx en tæller for fejlede forsøg pr. konto og pr. IP med midlertidig spærring eller stigende forsinkelse efter et lille antal fejl (fx 5), og nulstil tælleren ved et vellykket login. Supplér med rate limiting på webserver-niveau og logning/alarmering af gentagne fejlede forsøg. Kombinér med krav til password-styrke og en langsom password-hash (se kommende fund om MD5).

**Til rapporten:** FlyBooks login-endpoint (`api/API.php:54-78`) begrænser ikke antallet af login-forsøg. Der er hverken lockout, forsinkelse, CAPTCHA eller rate limiting, og fejlede forsøg logges ikke. Vi bekræftede det ved at sende 200 forkerte login-forsøg mod samme konto på under ét sekund i alt, uden at systemet reagerede. Det gør brute force- og credential stuffing-angreb mulige, og en angriber kan overtage konti med svage passwords og få adgang til persondata og bookingfunktionen. Fundet svarer til CWE-307 og OWASP A07:2021.

**Til eksamen:** Login-funktionen tæller ikke, hvor mange gange man har gættet forkert, så en angriber kan bare blive ved med at prøve passwords med et script. Vi sendte 200 forkerte forsøg på under et sekund, og systemet reagerede slet ikke.
