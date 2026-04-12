# 🛡️ Cybersecurity Learning Portfolio — Web Security & Penetration Testing

> Dokumentacja nauki z zakresu bezpieczeństwa aplikacji webowych, przeprowadzania testów penetracyjnych oraz naprawy wykrytych podatności.  
> Projekt edukacyjny realizowany w ramach kursu na **Polsko-Japońskiej Akademii Technik Komputerowych (PJATK)** na platformie **HackingDept**.

---

## 📋 Spis treści

- [O projekcie](#o-projekcie)
- [Technologie i narzędzia](#technologie-i-narzędzia)
- [Moduł 1 — Enumeracja sieci i Pentest sieciowy](#moduł-1--enumeracja-sieci-i-pentest-sieciowy)
- [Moduł 2 — Bezpieczeństwo Web: Burp Suite](#moduł-2--bezpieczeństwo-web-burp-suite)
- [Moduł 3 — Pentest Webowy (Scenariusze)](#moduł-3--pentest-webowy-scenariusze)
- [Jak naprawić wykryte podatności (Blue Team)](#jak-naprawić-wykryte-podatności-blue-team)
- [Czego się nauczyłam](#czego-się-nauczyłam)

---

## O projekcie

Celem projektów było praktyczne poznanie metod testowania bezpieczeństwa — zarówno od strony atakującego (**Red Team / Pentester**), jak i obrońcy (**Blue Team**).

Realizowane scenariusze symulują **realne błędy konfiguracyjne**, z jakimi można zetknąć się w środowiskach korporacyjnych: od niezabezpieczonych usług sieciowych (FTP, SMB, HTTP), przez podatności w aplikacjach webowych, po błędy logiki biznesowej i obfuskację kodu JavaScript.

Każde zadanie kończy się nie tylko "zdobyciem flagi", ale też **analizą przyczyny podatności i rekomendacją jej naprawy**.

---

## Technologie i narzędzia

| Narzędzie | Zastosowanie |
|---|---|
| **Nmap** | Skanowanie portów, fingerprinting usług i systemów operacyjnych, skrypty NSE |
| **Burp Suite Community** | Proxy HTTP/HTTPS, przechwytywanie i modyfikowanie ruchu webowego |
| **FTP / cURL / Wget** | Pobieranie plików z serwerów, eksploatacja anonymous login |
| **SMBClient** | Enumeracja i eksploatacja udziałów sieciowych Windows |
| **OpenVPN** | Dostęp do sieci laboratoryjnej |
| **obf-io.deobfuscate.io** | Deobfuskacja kodu JavaScript |
| **Kali Linux** | Środowisko do testów penetracyjnych |

---

## Moduł 1 — Enumeracja sieci i Pentest sieciowy

### Cel

Przeprowadzenie pełnego rekonesansu maszyny w sieci laboratoryjnej i odzyskanie ukrytych flag z różnych usług sieciowych.

### Przebieg

#### Krok 1 — Skanowanie Nmap

```bash
nmap -sT -sV -sC -O --osscan-guess 192.168.X.X
```

| Flaga | Co robi |
|---|---|
| `-sT` | TCP Connect Scan — pełny handshake (nie wymaga root, zostawia ślady w logach) |
| `-sV` | Fingerprinting wersji usług na portach |
| `-sC` | Uruchamia domyślne skrypty NSE (wykrywanie anonymous FTP, błędów SSL, info SMB) |
| `-O` | Detekcja systemu operacyjnego |

> **Czego się nauczyłam:** Skan `-sT` jest "głośny" — generuje wpisy w logach i może być wykryty przez systemy IDS/IPS. W realnym pentescie dobór techniki skanowania zależy od wymagań zlecenia (stealth vs. speed).

---

#### Krok 2 — Eksploatacja HTTP (serwer WWW bez zabezpieczeń)

```bash
wget http://192.168.X.X/flag.txt
```

Serwer udostępniał pliki bez żadnego uwierzytelnienia — klasyczny przykład **directory listing** i braku kontroli dostępu.

---

#### Krok 3 — Eksploatacja FTP (Anonymous Login)

```bash
ftp 192.168.X.X
# login: anonymous
# hasło: (puste lub dowolny e-mail)
```

FTP z włączonym anonymous login pozwala nieautoryzowanemu użytkownikowi przeglądać i pobierać pliki z serwera.

---

#### Krok 4 — Eksploatacja SMB (Null Session)

```bash
# Wylistowanie dostępnych udziałów
smbclient -N -L //192.168.200.52/

# Pobranie pliku z udziału
smb: \> ls
smb: \> get flag2.4p.txt
```

| Flaga | Znaczenie |
|---|---|
| `-N` | No password — łączenie bez hasła (Null Session) |
| `-L` | Listowanie dostępnych udziałów sieciowych |

> **Analogia:** Wchodzę do wspólnej szafki w biurze — drzwi są otwarte, bo nikt nie założył kłódki.

---

## Moduł 2 — Bezpieczeństwo Web: Burp Suite

### Wstęp

**Burp Suite** to kompleksowe narzędzie do testowania bezpieczeństwa aplikacji webowych, szeroko stosowane przez pentesterów. Umożliwia przechwytywanie i modyfikowanie ruchu HTTP/HTTPS, automatyzację testów oraz wykrywanie podatności takich jak SQL Injection czy XSS.

> Dostępne wersje: **Community** (darmowa) i **Professional** (płatna z automatycznym skanerem).

### Alternatywy dla Burp Suite

- **OWASP ZAP** — najlepsza darmowa alternatywa, w pełni otwartoźródłowa, zawiera automatyczny skaner podatności
- **Fiddler** — popularny wśród developerów, zaawansowana inspekcja HTTP/HTTPS
- **Mitmproxy** — narzędzie CLI, obsługa skryptów Python
- **Charles Proxy** — dedykowany do analizy ruchu aplikacji mobilnych (iOS/Android)
- **Chrome/Firefox DevTools** — do podstawowej analizy ruchu bez instalowania dodatkowych narzędzi

---

### Zadanie 3.1 — HTTP History (odczyt flagi z nagłówka odpowiedzi)

**Cel:** Zapoznanie z historią HTTP w Burp Suite.

**Kroki:**
1. Otwarcie wbudowanej przeglądarki Burp i wejście na `http://192.168.100.3`
2. W zakładce `Proxy → HTTP History` odnalezienie zapytania GET na `/`
3. W zakładce **Response** odczytanie wartości nagłówka `FLAG_3_1`

**Czego się nauczyłam:**
- Serwery mogą przesyłać wrażliwe dane (tokeny, flagi, informacje diagnostyczne) w **nagłówkach HTTP odpowiedzi**
- HTTP History rejestruje cały ruch przechodzący przez proxy — umożliwia analizę po fakcie
- Analiza nagłówków to jeden z pierwszych kroków rekonesansu aplikacji webowej

---

### Zadanie 3.2 — Intercept (modyfikacja żądania w locie)

**Cel:** Przechwycenie i modyfikacja parametru w żądaniu HTTP przed wysłaniem go do serwera.

**Kroki:**
1. Włączenie interceptu w Burp (`Proxy → Intercept → Intercept is on`)
2. Wywołanie strony `/solve/` w przeglądarce — Burp zatrzymuje żądanie
3. Modyfikacja parametru: `autosolve=no` → `autosolve=yes`
4. Wysłanie zmodyfikowanego żądania przyciskiem **Forward**

**Czego się nauczyłam:**
- Parametry wysyłane przez przeglądarkę można **dowolnie modyfikować** — nigdy nie wolno ufać danym od klienta
- Intercept pozwala symulować ataki i badać reakcję serwera na nieoczekiwane dane
- Dobrą praktyką jest wyłączanie interceptu gdy nie jest potrzebny — inaczej blokuje cały ruch

---

### Zadanie 3.3 — Repeater (wielokrotne wysyłanie żądań)

**Cel:** Manipulacja wynikami ankiety poprzez wielokrotne wysyłanie tego samego żądania.

**Kroki:**
1. Oddanie głosu na stronie `/vote/`
2. W HTTP History — kliknięcie PPM na żądanie głosowania → **Send to Repeater**
3. W zakładce **Repeater** — kliknięcie **Send** trzy razy (wysłanie 3 głosów na NIE)

**Czego się nauczyłam:**
- Jeśli serwer nie weryfikuje unikalności głosujących (np. nie sprawdza sesji ani IP), można **wielokrotnie powtarzać żądania**
- Repeater pozwala na szybkie iteracje bez ręcznego klikania w przeglądarce
- Podatność ta nosi nazwę **Broken Access Control / Missing Rate Limiting**

---

### Zadanie 3.4 — Intruder (automatyzacja ataków)

**Cel:** Zdobycie ponad 100 głosów w ankiecie za pomocą automatycznego ataku.

**Kroki:**
1. Wysłanie żądania głosowania do Intrudera (z Repeatera lub HTTP History)
2. Wyczyszczenie pozycji payloadów (`Clear §`)
3. Ustawienie: `Payload type → Null payloads` (Burp wysyła oryginalne żądanie bez modyfikacji)
4. Liczba payloadów: `101`
5. Kliknięcie **Start attack**

**Czego się nauczyłam:**
- Intruder umożliwia **automatyzację dużej liczby żądań** — brute force, fuzzing, enumeracja
- `Null payloads` to technika "flood" — serwer odbiera to samo żądanie wiele razy
- Brak limitowania liczby żądań (Rate Limiting) to krytyczna podatność w aplikacjach webowych

---

### Zadanie 3.5 — Decoder (dekodowanie danych)

**Cel:** Zdekodowanie zakodowanej flagi ze strony `/encoded/`.

**Kroki:**
1. Wejście na `http://192.168.100.3/encoded/` — wyświetla się zakodowana flaga (URL encoding)
2. Skopiowanie flagi do zakładki **Decoder** w Burp
3. Wybranie opcji `Decode as → URL` → uzyskanie czytelnej flagi

**Czego się nauczyłam:**
- Dane zakodowane (Base64, URL, Hex) to **NIE jest szyfrowanie** — każdy może je zdekodować
- Decoder jest niezastąpiony podczas analizy tokenów sesji, cookies i parametrów URL
- Rozróżnienie między kodowaniem (encoding) a szyfrowaniem (encryption) jest kluczowe w security

---

## Moduł 3 — Pentest Webowy (Scenariusze)

### Zadanie 3.1p — Ciasteczka (Cookie Manipulation)

**Cel:** Modyfikacja wartości ciasteczka `admin` z `false` na `true`.

**Kroki:**
1. Otwarcie strony `http://192.168.100.53/` w przeglądarce Burp
2. W HTTP History → Send to Repeater
3. W Repeaterze — zmiana nagłówka `Cookie: admin=false` na `Cookie: admin=true`
4. Wysłanie zmodyfikowanego żądania

**Czego się nauczyłam:**
- **Ciasteczka po stronie klienta można dowolnie modyfikować** — nigdy nie powinny decydować o uprawnieniach
- Weryfikacja uprawnień musi zawsze odbywać się **po stronie serwera**
- Podatność: **Insecure Direct Object Reference / Broken Access Control**

---

### Zadanie 3.2p — Przekierowanie (Open Redirect Bypass)

**Cel:** Ominięcie przekierowania HTTP i dostęp do panelu admina na `/login/`.

**Kroki:**
1. Wejście na `http://192.168.100.53/login/`
2. Strona przekierowuje na stronę logowania, ale **odpowiedź HTTP już zawiera chronioną treść**
3. W HTTP History → kliknięcie na żądanie `/login/` → zakładka **Response** → **Render**
4. Wyświetlenie panelu admina bezpośrednio z odpowiedzi, mimo że przeglądarka wykonała redirect

**Czego się nauczyłam:**
- Przekierowanie `302 Found` to **wskazówka dla przeglądarki**, nie faktyczna blokada dostępu
- Serwer musi zwracać treść chronioną **dopiero po weryfikacji** — nie można polegać wyłącznie na redirectzie
- Podatność: **Improper Access Control — reliance on client-side redirect**

---

### Zadanie 3.3p — Enumeracja użytkowników po ID

**Cel:** Wylistowanie użytkowników w panelu admina poprzez iterowanie po ID w URL.

**Kroki:**
1. Po uzyskaniu dostępu do panelu: URL `/userdata/1/`, `/userdata/2/`, ...
2. W Burp: Send to Intruder → zaznaczenie wartości ID w URL → **Add §**
3. Payload type: `Numbers` (zakres 1–50)
4. Start attack → analiza odpowiedzi — status 200 = użytkownik istnieje, 404 = nie istnieje
5. Odnalezienie użytkownika `Hacker`

**Czego się nauczyłam:**
- **IDOR (Insecure Direct Object Reference)** — brak weryfikacji czy zalogowany użytkownik ma prawo dostępu do danego zasobu
- Endpoint `/userdata/[ID]/` powinien sprawdzać sesję, nie tylko czy ID istnieje
- Intruder z payloadem Numbers to klasyczne narzędzie do tego rodzaju enumeracji

---

### Zadanie 3.4p — Dekodowanie danych użytkownika

**Cel:** Dostęp do unikalnego URL użytkownika Hacker poprzez dekodowanie Base64.

**Kroki:**
1. Z wyników enumeracji (Intruder) — odnalezienie danych użytkownika Hacker (zakodowane w Base64)
2. Zakładka **Decoder** → wklejenie wartości → `Decode as → Base64`
3. Odczytanie: `name: Hacker, password: [hasło], URL: [ścieżka]`
4. Wejście na odkodowany URL

**Czego się nauczyłam:**
- **Base64 to kodowanie, NIE szyfrowanie** — nie chroni wrażliwych danych
- Hasła i wrażliwe dane nigdy nie powinny być przechowywane ani przesyłane w Base64
- Dekodowanie danych z odpowiedzi HTTP to standardowy krok analizy aplikacji webowej

---

### Zadanie 3.5p — Deobfuskacja JavaScript

**Cel:** Odnalezienie ukrytej ścieżki w zaciemnionym kodzie JavaScript.

**Kroki:**
1. Wejście na stronę użytkownika Hacker → podgląd źródła strony (`Ctrl+U`)
2. Kod JavaScript jest celowo **zaciemniony (obfuskowany)** — niemożliwy do bezpośredniego odczytania
3. Skopiowanie kodu i wklejenie do [obf-io.deobfuscate.io](https://obf-io.deobfuscate.io/)
4. W zdeobfuskowanym kodzie odnalezienie fragmentu:
   ```javascript
   if(secret)
     location.href = "secret1337.php"
   ```
5. Wejście na stronę `[URL_hackera]/secret1337.php`

**Czego się nauczyłam:**
- **Obfuskacja JavaScript ≠ bezpieczeństwo** — to security through obscurity, które nie działa
- Wrażliwa logika biznesowa (linki, klucze API, hasła) nigdy nie powinna znajdować się w kodzie po stronie klienta
- Podatność: **Sensitive Data Exposure w kodzie JavaScript**

---

## Jak naprawić wykryte podatności (Blue Team)

Jako pentester nie tylko identyfikuję błędy, ale też rekomenduje ich naprawę.

### 🔒 Uwierzytelnianie i autoryzacja

| Podatność | Poprawka |
|---|---|
| Cookie manipulation (`admin=true`) | Weryfikacja uprawnień **wyłącznie po stronie serwera** — sesja w bazie danych, nie w ciasteczku |
| IDOR (dostęp do danych innych użytkowników) | Sprawdzanie czy zalogowany użytkownik jest właścicielem zasobu przed każdym żądaniem |
| Bypass redirectu | Serwer powinien zwracać `403 Forbidden` lub pustą odpowiedź — nigdy chronioną treść w body redirectu |
| Anonymous login (FTP/SMB) | Wyłączenie logowania anonimowego; wymagane silne uwierzytelnianie (login + hasło) |

### 🔒 Przechowywanie i przesyłanie danych

| Podatność | Poprawka |
|---|---|
| Dane w Base64 w odpowiedzi HTTP | Nigdy nie przesyłaj haseł ani wrażliwych danych — nawet "zakodowanych" |
| HTTP zamiast HTTPS | Wdrożenie TLS — wszystkie protokoły powinny mieć zaszyfrowane odpowiedniki: HTTP→HTTPS, FTP→SFTP |
| Dane wrażliwe w JavaScript | Całkowite przeniesienie logiki biznesowej i sekretów na backend |
| Obfuskacja jako "zabezpieczenie" | Nie stosować — obfuskacja opóźnia analizę o minuty, nie chroni |

### 🔒 Konfiguracja serwera

| Podatność | Poprawka |
|---|---|
| Banner grabbing (ujawnianie wersji) | Wyłączenie/ukrycie nagłówków serwera (np. `Server: Apache/2.4.41`) |
| Directory listing | Wyłączenie listowania katalogów w konfiguracji serwera WWW |
| Brak rate limitingu | Implementacja limitowania liczby żądań (np. max 5 głosów/minutę/IP) |
| SMB/FTP widoczne dla wszystkich | Firewall (iptables/ufw) — dostęp tylko z zaufanych adresów IP |

### 🔒 Zasady ogólne

- **Principle of Least Privilege** — każdy użytkownik i usługa powinny mieć dostęp tylko do tego, czego faktycznie potrzebują
- **Defense in Depth** — wielowarstwowe zabezpieczenia; nie polegaj na jednej warstwie ochrony
- **Nigdy nie ufaj danym od klienta** — każdy parametr (cookie, nagłówek, URL, body) może być zmodyfikowany

---

## Czego się nauczyłam

✅ Metodyka skanowania sieciowego — dobór technik Nmap w zależności od celu i kontekstu  
✅ Fingerprinting usług i systemów operacyjnych  
✅ Przechwytywanie, analiza i modyfikacja ruchu HTTP/HTTPS za pomocą Burp Suite  
✅ Praktyczne użycie HTTP History, Intercept, Repeater, Intruder i Decoder  
✅ Eksploatacja błędnych konfiguracji: anonymous FTP, Null Session SMB, brak HTTPS  
✅ Ataki na logikę aplikacji webowych: cookie manipulation, IDOR, redirect bypass  
✅ Automatyzacja ataków (brute force, enumeration, flood) za pomocą Intrudera  
✅ Dekodowanie danych: Base64, URL encoding  
✅ Analiza i deobfuskacja kodu JavaScript  
✅ Myślenie obronne — rekomendacje naprawy każdej wykrytej podatności  

---

*Projekt edukacyjny | PJATK | HackingDept Platform | 2025*
