# 🔍 Enumeracja Sieci — Laboratoria PJA (Zajęcia nr 2)

> Materiały z zajęć praktycznych z zakresu **enumeracji sieci i testów penetracyjnych** realizowanych na platformie HackingDept w Polsko-Japońskiej Akademii Technik Komputerowych.

---

## 📚 Czego się nauczyłam

### Teoria

- **Czym jest enumeracja sieci** — aktywny proces zbierania informacji o infrastrukturze: adresach IP, otwartych portach, nazwach hostów, systemach operacyjnych i wersjach usług.
- **Rola enumeracji w pentestingu** — to pierwszy i kluczowy krok przed eskalacją uprawnień czy eksploitacją podatności.
- **Nmap i alternatywy** — poznałam najpopularniejsze narzędzia do skanowania sieci:
  - `nmap` — skanowanie portów, wykrywanie usług i OS
  - `masscan` — bardzo szybki skaner portów
  - `rustscan` — nowoczesny skaner z integracją Nmap
  - `zmap` — skanowanie dużych sieci
  - `unicornscan` — zbieranie metadanych
- **Enumeracja na Windowsie** — PowerShell (`Test-NetConnection`, `Get-NetTCPConnection`), SharpHound, Advanced IP Scanner
- **Zaawansowane techniki enumeracji**:
  - DNS (dnsrecon, Fierce, Sublist3r)
  - SNMP (snmpwalk)
  - Repozytoria kodu (Gitwalk, Gitdump)
  - SMB (smbclient)

---

## ✅ Co zrobiłam — Zadania

### Podstawowe (`2.1` – `2.8`)

| Zadanie | Opis | Komenda |
|--------|------|---------|
| 2.1 | Proste skanowanie hosta, znalezienie SSH na niestandardowym porcie | `nmap 192.168.100.2` / `nmap -A 192.168.100.2` |
| 2.2 | Połączenie SSH na znalezionym porcie | `ssh student@192.168.100.2 -p 23` |
| 2.3 | Przeglądanie opcji Nmap | `nmap -h` |
| 2.4 | Odkrywanie aktywnych hostów w sieci bez skanowania portów | `nmap -sn 192.168.100.0/24` |
| 2.5 | Skanowanie otwartych portów TCP | `nmap -sT 192.168.100.0/24` |
| 2.6 | Detekcja wersji usług | `nmap -sV 192.168.100.0/24` |
| 2.7 | Uruchamianie domyślnych skryptów NSE | `nmap -sC 192.168.100.0/24` |
| 2.8 | Wykrywanie systemu operacyjnego | `nmap -O --osscan-guess 192.168.100.0/24` |

### Pentest (`2.1p` – `2.4p`)

| Zadanie | Opis | Komenda |
|--------|------|---------|
| 2.1p | Kompleksowe skanowanie maszyny docelowej | `nmap -sT -sV -sC -O --osscan-guess 192.168.200.52` |
| 2.2p | Pobranie flagi z serwera HTTP | `curl http://192.168.200.52/flag2.2p.txt` |
| 2.3p | Pobranie flagi z serwera FTP (logowanie anonimowe) | `ftp 192.168.200.52` → `anonymous` → `get flag2.3p.txt` |
| 2.4p | Pobranie flagi z udziału SMB bez hasła | `smbclient -N //192.168.200.52/FLAG` → `get flag2.4p.txt` |

---

## 🔓 Znalezione podatności i ich naprawy

### 1. SSH na niestandardowym porcie bez ograniczeń dostępu

**Problem:** Usługa SSH nasłuchiwała na porcie 23 (domyślny port Telnetu), co może być próbą ukrycia usługi — ale nie stanowi realnego zabezpieczenia. Brak ograniczeń IP i silnej polityki haseł.

**Naprawa:**
```bash
# /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no   # wymuś klucze SSH
AllowUsers student
```
Dodatkowo: ogranicz dostęp przez firewall (`ufw allow from <trusted_ip> to any port 23`).

---

### 2. Anonimowy dostęp do FTP

**Problem:** Serwer FTP akceptował logowanie jako `anonymous` bez hasła, umożliwiając odczyt plików każdemu.

**Naprawa:**
```bash
# /etc/vsftpd.conf
anonymous_enable=NO
local_enable=YES
chroot_local_user=YES
```
Rozważ zastąpienie FTP protokołem SFTP lub FTPS (szyfrowanie w transmisji).

---

### 3. Udział SMB dostępny bez uwierzytelnienia

**Problem:** Zasób `FLAG` na serwerze SMB był dostępny bez podania hasła (przełącznik `-N` w smbclient). Brak uwierzytelnienia na udziale sieciowym to poważne ryzyko w środowiskach produkcyjnych.

**Naprawa:**
```bash
# /etc/samba/smb.conf
[FLAG]
   path = /srv/smb/flag
   valid users = @smbusers
   read only = yes
   guest ok = no
```
Wymuś uwierzytelnienie i ogranicz dostęp do konkretnych użytkowników/grup.

---

### 4. Serwer HTTP z listowaniem katalogu i plikami tekstowymi

**Problem:** Serwer HTTP udostępniał pliki `.txt` z flagami bezpośrednio w katalogu głównym, a `wget -r` pozwalało pobrać całą zawartość serwera.

**Naprawa:**
```apache
# /etc/apache2/sites-available/000-default.conf
<Directory /var/www/html>
    Options -Indexes        # wyłącz listowanie katalogów
    AllowOverride None
</Directory>
```
Wrażliwe pliki nie powinny być dostępne publicznie — należy je chronić uwierzytelnieniem lub trzymać poza katalogiem web root.

---

### 5. Brak segmentacji sieci — maszyna widoczna z wewnątrz VPN

**Problem:** Maszyna `192.168.200.52` (pentest) była dostępna z poziomu sieci `192.168.100.x` bez dodatkowych barier. W realnym środowisku DMZ powinno być odizolowane od sieci wewnętrznej.

**Naprawa:**
- Wdrożenie reguł firewalla między segmentami sieci
- Zastosowanie VLAN-ów i polityki least-privilege w routingu
- Monitoring połączeń z systemem IDS/IPS

---

## 🛠️ Narzędzia

| Narzędzie | Zastosowanie |
|-----------|-------------|
| `nmap` | Skanowanie portów, wykrywanie usług i OS |
| `ssh` | Zdalne połączenie z maszyną |
| `curl` / `wget` | Pobieranie zasobów HTTP |
| `ftp` | Klient FTP |
| `smbclient` | Klient SMB/Samba |
| `openvpn` | Połączenie z siecią laboratoryjną |

---

## 📖 Referencje

- [Nmap — polski manual](https://nmap.org/man/pl/man-briefoptions.html)
- [Nmap — dokumentacja oficjalna](https://nmap.org/docs.html)
- Materiały z platformy HackingDept, PJA — Zajęcia nr 2 (2025-12-14)
