# 🔍 Web Enumeration & Pentesting — Zajęcia nr 4

> **Dokumentacja laboratoryjna** | Polsko-Japońska Akademia Technik Komputerowych  
> Platforma: HackingDept | Środowisko: Kali Linux + OpenVPN

---

## 📋 Spis treści

- [O projekcie](#o-projekcie)
- [Środowisko](#środowisko)
- [Zdobyte umiejętności](#zdobyte-umiejętności)
- [Wykonane zadania](#wykonane-zadania)
- [Odkryte podatności i ich mitigacja](#odkryte-podatności-i-ich-mitigacja)
- [Narzędzia](#narzędzia)
- [Wnioski](#wnioski)

---

## O projekcie

Laboratorium poświęcone enumeracji zasobów serwisów WWW w kontrolowanym środowisku testowym. Celem było praktyczne zastosowanie technik rekonesansu webowego na serwerze `192.168.100.54` — od odczytu metadanych certyfikatu SSL, przez odkrycie ukrytych podstron i subdomen, po przeprowadzenie słownikowego ataku bruteforce na panel logowania.

Wszystkie działania były wykonywane wyłącznie na dedykowanych maszynach laboratoryjnych w izolowanej sieci VPN, zgodnie z zasadami etycznego pentestingu.

---

## Środowisko

| Komponent | Szczegóły |
|---|---|
| Maszyna atakująca | Kali Linux |
| Cel (Web Enum) | `192.168.100.4` |
| Cel (Web Enum Pentest) | `192.168.100.54` |
| Połączenie | OpenVPN (`sudo openvpn /path/to/config.ovpn`) |
| Platforma | HackingDept |

---

## Zdobyte umiejętności

### 🔎 Enumeracja katalogów i plików
Nauczyłam się wykrywać ukryte zasoby na serwerach WWW przy użyciu ataków słownikowych. Dobrze skonfigurowany serwer nie wyświetla zawartości katalogów, ale jeśli zna się ścieżkę — serwer odpowiada. Narzędzia takie jak Gobuster i ffuf automatyzują testowanie tysięcy potencjalnych ścieżek.

### 🌐 Enumeracja wirtualnych hostów i subdomen
Serwer HTTP może obsługiwać wiele różnych witryn pod tym samym adresem IP — rozróżnianych nagłówkiem `Host`. Nauczyłam się wykrywać takie hosty metodą słownikową (vhost fuzzing) oraz odczytywać nazwy hostów z certyfikatów SSL (pole `Common Name`).

### 🔐 Atak słownikowy (bruteforce) na formularz logowania
Przy użyciu Hydry przeprowadziłam automatyczny atak na formularz POST, testując hasła ze słownika najczęściej używanych haseł. Nauczyłam się parametryzować takie ataki — wskazywać URL, pola formularza oraz string identyfikujący błędne logowanie.

### 🛡️ Zrozumienie koncepcji "security through obscurity"
Ukrywanie zasobów (nieujawnianie linków, niestandardowe nazwy katalogów) **nie jest** skuteczną metodą zabezpieczenia. To jedynie opóźnia, nie blokuje atakującego.

---

## Wykonane zadania

### Zadanie 4.1p — Certificate Common Name

**Cel:** Odczytanie nazwy hosta z certyfikatu SSL i połączenie się z wirtualnym hostem.

**Metoda:**
1. Wejście na `https://192.168.100.54/` i wybór opcji "View Certificate"
2. Odczyt pola `Common Name` → wartość: `webpjatk`
3. Dodanie wpisu do `/etc/hosts`:
```bash
echo '192.168.100.54 webpjatk' | sudo tee -a /etc/hosts
```
4. Wejście na `https://webpjatk/` → zadanie zaliczone ✅

**Czego się nauczyłam:** Certyfikaty SSL mogą ujawniać nazwy wewnętrznych hostów, które nie są publicznie rozwiązywalne przez DNS. To częsty punkt startowy rekonesansu.

---

### Zadanie 4.2p — Enumeracja subdomen

**Cel:** Znalezienie aktywnej subdomeny domeny `webpjatk`.

**Narzędzie:** Gobuster w trybie `vhost`

```bash
gobuster vhost \
  -u https://192.168.100.54/ \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  --domain webpjatk \
  --append-domain \
  -k
```

**Wynik:** Odkryto subdomenę `admin.webpjatk`

```bash
echo '192.168.100.54 admin.webpjatk' | sudo tee -a /etc/hosts
```

Wejście na `https://admin.webpjatk/` → zadanie zaliczone ✅

**Czego się nauczyłam:** Flaga `-k` pozwala na połączenie z serwerem pomimo niezaufanego certyfikatu SSL. `--append-domain` automatycznie dołącza bazową domenę do każdego słowa ze słownika.

---

### Zadanie 4.3p — Enumeracja katalogów (admin panel)

**Cel:** Znalezienie ukrytego katalogu na `https://admin.webpjatk/`.

```bash
gobuster dir \
  -u https://admin.webpjatk/ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-small-directories.txt \
  -k
```

**Wynik:** Odkryto `/panel/` (status 301 → redirect do `https://admin.webpjatk/panel/`) ✅

---

### Zadanie 4.4p — Enumeracja plików (panel logowania)

**Cel:** Znalezienie pliku PHP w katalogu `/panel/`.

```bash
gobuster dir \
  -u https://admin.webpjatk/panel/ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-small-files.txt \
  -k
```

**Wynik:** Odkryto `login.php` → wejście na `https://admin.webpjatk/panel/login.php` ✅

**Uwaga:** Gobuster ujawnił też `.htaccess` (403) oraz inne pliki systemowe — samo ich istnienie jest informacją o strukturze serwera.

---

### Zadanie 4.5p — Bruteforce hasła (Hydra)

**Cel:** Złamanie hasła do formularza logowania metodą słownikową.

**Narzędzie:** Hydra

```bash
hydra \
  -l anything \
  -P /usr/share/seclists/Passwords/xato-net-10-million-passwords-100.txt \
  'https-post-form://admin.webpjatk/panel/login.php:p=^PASS^:Wrong password'
```

| Parametr | Znaczenie |
|---|---|
| `-l anything` | Login (w tym przypadku nieistotny — formularz weryfikuje tylko hasło) |
| `-P <wordlist>` | Słownik haseł |
| `https-post-form` | Moduł ataku na formularze POST przez HTTPS |
| `p=^PASS^` | Pole POST, gdzie `^PASS^` to placeholder zastępowany kolejnymi hasłami |
| `Wrong password` | Ciąg zwracany przy błędnym haśle — Hydra używa go jako warunku porażki |

**Wynik:** Hasło `dragon` znalezione po ~99 próbach ✅

---

## Odkryte podatności i ich mitigacja

### 1. 🔓 Ujawnianie Common Name w certyfikacie SSL

**Podatność:** Pole CN certyfikatu zawierało nazwę wewnętrznego wirtualnego hosta (`webpjatk`), niedostępnego publicznie przez DNS — ale możliwego do odwiedzenia po ręcznym dodaniu do `/etc/hosts`.

**Ryzyko:** Atakujący może odkryć wewnętrzną strukturę serwerów i zaatakować zasoby, które właściciel uważał za ukryte.

**Mitigacja:**
- Stosowanie certyfikatów z generycznym CN lub wildcard (`*.example.com`) tam, gdzie nazwy hostów są wrażliwe
- Używanie Subject Alternative Names (SAN) z rozwagą
- Wdrożenie Certificate Transparency monitoring — aby wiedzieć, co jest publicznie widoczne

---

### 2. 🔓 Brak ochrony przed enumeracją vhostów i subdomen

**Podatność:** Serwer odpowiadał różnymi kodami HTTP (200 vs 421/403) dla istniejących i nieistniejących hostów — co pozwoliło Gobusterowi na automatyczne wykrywanie aktywnych hostów.

**Ryzyko:** Odkrycie paneli administracyjnych i wewnętrznych aplikacji niewidocznych z zewnątrz.

**Mitigacja:**
- Konfiguracja serwera tak, aby zwracał identyczną odpowiedź (np. zawsze 404 lub przekierowanie) dla nieznanych nagłówków `Host`
- Segmentacja sieci — panele administracyjne powinny być dostępne wyłącznie z VPN lub określonych IP
- Stosowanie WAF z regułami blokującymi masowe zapytania z różnymi nagłówkami `Host`

---

### 3. 🔓 Dostępność katalogu administracyjnego bez uwierzytelnienia

**Podatność:** Katalog `/panel/` był dostępny publicznie (po odkryciu adresu) i zwracał kod 200 bez żadnego uwierzytelnienia na poziomie katalogu.

**Ryzyko:** Atakujący po odkryciu ścieżki ma bezpośredni dostęp do panelu logowania.

**Mitigacja:**
- Ograniczenie dostępu do paneli administracyjnych przez IP (allowlist w konfiguracji serwera lub WAF)
- Dodatkowe uwierzytelnienie na poziomie HTTP (Basic Auth jako dodatkowa warstwa)
- Zastosowanie niestandardowych, losowych ścieżek do panelu (nie gwarantuje bezpieczeństwa, ale utrudnia atak)

---

### 4. 🔓 Brak ochrony przed atakami bruteforce na formularz logowania

**Podatność:** Formularz logowania w `login.php` nie posiadał żadnych mechanizmów ochrony przed zautomatyzowanymi atakami — brak rate limitingu, CAPTCHA ani blokady po N nieudanych próbach. Hasło `dragon` znajdowało się w top 100 najczęstszych haseł.

**Ryzyko:** Pełne przejęcie konta w ciągu sekund przy użyciu publicznie dostępnych słowników.

**Mitigacja:**

| Mechanizm | Opis |
|---|---|
| **Rate limiting** | Ograniczenie liczby prób logowania na IP (np. max 5/minutę) |
| **Account lockout** | Tymczasowa blokada konta po N nieudanych próbach |
| **CAPTCHA** | Wymuszenie interakcji człowieka po kilku błędnych próbach |
| **Multi-Factor Authentication (MFA)** | Nawet znane hasło nie wystarczy |
| **Silna polityka haseł** | Minimalna długość, złożoność, zakaz popularnych haseł |
| **Monitoring / alerting** | Wykrywanie anomalii (wiele prób z jednego IP) i alertowanie |

---

### 5. 🔓 Ujawnianie struktury plików przez Gobuster

**Podatność:** Pliki takie jak `.htaccess`, `.htpasswd`, backup files mogły być widoczne w wynikach enumeracji (nawet jeśli zwracały 403, samo ich istnienie jest informacją).

**Mitigacja:**
- Usuwanie plików tymczasowych, backupów i plików konfiguracyjnych z katalogów webowych
- Konfiguracja serwera, aby nie ujawniał informacji o plikach (np. jednolity kod 404 dla wszystkich nieauthorizowanych żądań)
- Regularne audyty zawartości serwerów webowych

---

## Narzędzia

### Gobuster
Narzędzie do brutalnego skanowania katalogów, plików, vhostów i subdomen. Wysyła masowe zapytania HTTP i analizuje odpowiedzi.

```bash
# Enumeracja katalogów
gobuster dir -u <URL> -w <wordlist> [-k]

# Enumeracja vhostów/subdomen
gobuster vhost -u <URL> -w <wordlist> --domain <domena> --append-domain [-k]

# Fuzzing parametrów
gobuster fuzz -u <URL>?param=FUZZ -w <wordlist>
```

### ffuf (Fuzz Faster U Fool)
Elastyczne narzędzie do HTTP fuzzingu. Placeholder `FUZZ` można umieszczać w dowolnym miejscu zapytania.

```bash
ffuf -w <wordlist> -u <URL>/FUZZ          # katalogi
ffuf -w <wordlist> -u <URL>/FUZZ -fs 316  # z filtrem rozmiaru odpowiedzi
ffuf -w <wordlist> -u <URL> -H 'Host: FUZZ.domena'  # vhosty
```

### Hydra
Narzędzie do ataków bruteforce na różne protokoły (HTTP, SSH, FTP i inne).

```bash
hydra -l <user> -P <wordlist> 'https-post-form://<host>/<path>:<POST_data>:<failure_string>'
```

### SecLists
Kolekcja słowników używanych we wszystkich powyższych atakach.

```bash
sudo apt install seclists
# lub
git clone https://github.com/danielmiessler/SecLists.git
```

Używane słowniki:
- `Discovery/Web-Content/raft-small-directories.txt` — katalogi
- `Discovery/Web-Content/raft-small-files.txt` — pliki
- `Discovery/DNS/subdomains-top1million-5000.txt` — subdomeny
- `Passwords/xato-net-10-million-passwords-100.txt` — hasła

---

## Wnioski

Laboratorium pokazało, że wiele podatności wynika nie z błędów w kodzie aplikacji, ale z **błędów konfiguracyjnych** i **złych praktyk operacyjnych**: ujawnianie struktury przez certyfikaty, brak segmentacji sieci, dostępność paneli administracyjnych z internetu, brak rate limitingu.

Kluczowy wniosek: **security through obscurity nie działa**. Ukrywanie zasobów poprzez niepublikowanie linków czy niestandardowe nazwy katalogów może opóźnić atak, ale żadne narzędzie do enumeracji nie jest tym ograniczone. Jedynym skutecznym zabezpieczeniem jest uwierzytelnienie, autoryzacja i monitoring.

---

> ⚠️ **Disclaimer:** Wszystkie techniki opisane w tej dokumentacji były stosowane wyłącznie w kontrolowanym środowisku laboratoryjnym, za wyraźną zgodą, w celach edukacyjnych. Używanie tych narzędzi na systemach bez autoryzacji jest nielegalne.
