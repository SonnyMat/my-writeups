# 🔐 Protokoły Sieciowe & Transfer Plików 

> **Zajęcia nr 5 | Polsko-Japońska Akademia Technik Komputerowych**  
> Platforma: HackingDept | Środowisko: Kali Linux + OpenVPN

---

## 📋 O projekcie

Seria praktycznych ćwiczeń z zakresu **sieciowych protokołów transferu plików** oraz **podstaw pentestingu**. Zadania były realizowane w izolowanym środowisku laboratoryjnym z dostępem przez VPN, na dedykowanych maszynach wirtualnych.

---

## 🧠 Czego się nauczyłam

### Protokoły warstwy transportowej
| Protokół | Zastosowanie w ćwiczeniu | Narzędzie |
|----------|--------------------------|-----------|
| **TCP connect** | Pobieranie i wysyłanie plików (klient inicjuje) | `nc`, `socat` |
| **TCP listen** | Pobieranie i wysyłanie plików (serwer czeka) | `nc -lvp` |
| **UDP connect** | Transfer bez gwarancji dostarczenia | `nc -u` |
| **UDP listen** | Odbieranie datagramów | `nc -luvp` |

### Protokoły warstwy aplikacji
- **HTTP** — uruchamianie prostego serwera plików (`python3 -m http.server`)
- **HTTPS** — konfiguracja Apache2 z SSL/TLS (`a2enmod ssl`, certyfikat self-signed)
- **FTP** — serwer z anonimowym dostępem (`pyftpdlib`)
- **SMB** — udostępnianie zasobów sieciowych (`impacket smbserver.py`)
- **SCP/SSH** — szyfrowany transfer plików (`openssh-server`)

### Umiejętności pentestingu (moduł Pentest)
- Przenoszenie narzędzi na maszynę bez dostępu do internetu (`scp`)
- Skanowanie portów i detekcja usług (`nmap -sV -p-`)
- Analiza wyników skanu i identyfikacja otwartych usług
- Pobieranie zawartości serwera webowego (`wget -r --no-parent`)
- Analiza kluczy SSH z udostępnionych zasobów
- Logowanie przez SSH z użyciem klucza prywatnego
- Eskalacja uprawnień przez błędną konfigurację `sudo` (NOPASSWD)

---

## 🛡️ Zidentyfikowane podatności i ich mitigacja

### 1. 🔴 Krytyczna — Klucze SSH w publicznym katalogu WWW
**Opis:** Serwer HTTP wystawiał katalog domowy użytkownika (`.ssh/`) jako listę plików. Klucz prywatny `id_ed25519` był dostępny bez uwierzytelnienia.

**Jak zostało wykorzystane:**
```bash
wget -r --no-parent http://192.168.200.55
chmod 600 id_ed25519
ssh -i id_ed25519 webadmin@192.168.200.55
```

**Naprawa:**
```bash
# Nigdy nie hostuj katalogu domowego jako root WWW
# Ogranicz dostęp do .ssh/
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519

# Skonfiguruj Apache żeby nie listował plików
echo "Options -Indexes" >> /etc/apache2/conf-enabled/security.conf

# Użyj dedykowanego katalogu dla serwera webowego
DocumentRoot /var/www/html   # NIE /home/user
```

---

### 2. 🔴 Krytyczna — sudo NOPASSWD dla wszystkich komend
**Opis:** Użytkownik `webadmin` miał skonfigurowane sudo bez hasła na wszystkie polecenia (`(ALL) NOPASSWD: ALL`), co umożliwiło natychmiastową eskalację do roota.

**Jak zostało wykorzystane:**
```bash
sudo su       # bez pytania o hasło
cat /root/root.txt
```

**Naprawa:**
```bash
# Edytuj /etc/sudoers przez visudo
# BŁĘDNA konfiguracja (nigdy nie używaj!):
webadmin ALL=(ALL) NOPASSWD: ALL

# POPRAWNA konfiguracja — tylko konkretne, niezbędne polecenia:
webadmin ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx
# lub całkowity zakaz sudo dla tego użytkownika
```

---

### 3. 🟠 Wysoka — FTP bez szyfrowania (cleartext)
**Opis:** Serwer FTP (`pyftpdlib`) przesyła dane i dane logowania niezaszyfrowanym tekstem. Możliwy atak Man-in-the-Middle oraz przechwycenie credentiali.

**Naprawa:**
```bash
# Użyj SFTP (SSH) zamiast FTP
# lub skonfiguruj FTPS (FTP over TLS):
python3 -m pyftpdlib --certfile=/etc/ssl/certs/server.crt \
                      --keyfile=/etc/ssl/private/server.key \
                      --tls-required
                      
# Jeszcze lepiej — całkowicie zastąp FTP przez SFTP/SCP
```

---

### 4. 🟠 Wysoka — HTTP zamiast HTTPS (dane w plaintext)
**Opis:** Serwer na porcie 80 przesyła dane bez szyfrowania. Zawartość jest widoczna dla każdego w tej samej sieci.

**Naprawa:**
```bash
# Wymuszenie HTTPS w Apache
sudo a2enmod rewrite
# W konfiguracji VirtualHost na porcie 80:
# Redirect permanent / https://twoja-domena.pl/

# Użyj Let's Encrypt dla ważnych certyfikatów (nie self-signed)
sudo apt install certbot python3-certbot-apache
sudo certbot --apache
```

---

### 5. 🟡 Średnia — SMB bez uwierzytelnienia (null session)
**Opis:** Serwer SMB akceptuje połączenia bez podania hasła (`-N`), umożliwiając dostęp do udostępnionych zasobów każdemu w sieci.

**Naprawa:**
```bash
# Skonfiguruj wymagane uwierzytelnienie w impacket lub Samba:
# W smb.conf:
[global]
   restrict anonymous = 2
   map to guest = Never

# Dla impacket — dodaj parametry użytkownika i hasła:
smbserver.py -username user -password StrongPass123 sharename /path
```

---

### 6. 🟡 Średnia — Directory Listing włączony
**Opis:** Serwer HTTP wyświetlał pełną zawartość katalogów, umożliwiając eksplorację struktury plików bez znajomości konkretnych ścieżek.

**Naprawa:**
```bash
# Apache — wyłącz directory listing globalnie
echo "Options -Indexes" >> /etc/apache2/apache2.conf
sudo systemctl restart apache2

# Python http.server — brak prostej opcji wyłączenia,
# nigdy nie używaj w produkcji
```

---

### 7. 🟢 Niska — Komentarz w kluczu publicznym SSH ujawnia nazwę użytkownika
**Opis:** Standardowy format klucza publicznego zawiera komentarz `user@hostname`, który w tym przypadku ujawnił nazwę użytkownika systemowego.

**Naprawa:**
```bash
# Generuj klucze bez komentarza lub z neutralnym:
ssh-keygen -t ed25519 -C "deploy-key-2024"   # bez nazwy użytkownika

# Usuń komentarz z istniejącego klucza publicznego:
# Edytuj plik .pub i usuń ostatnią część po ostatniej spacji
```

---

## 🔧 Użyte narzędzia

| Narzędzie | Zastosowanie |
|-----------|-------------|
| `netcat (nc)` | Transfer plików TCP/UDP, testowanie połączeń |
| `socat` | Zaawansowane przekierowania i transfery |
| `nmap` | Skanowanie portów, detekcja wersji usług |
| `wget` | Pobieranie zawartości HTTP rekurencyjnie |
| `curl` | Testowanie endpointów HTTP |
| `scp` | Szyfrowany transfer plików przez SSH |
| `smbclient` | Klient protokołu SMB |
| `pyftpdlib` | Szybki serwer FTP w Pythonie |
| `impacket` | Zestaw narzędzi sieciowych (smbserver.py) |
| `apache2` | Serwer HTTP/HTTPS |
| `OpenSSH` | Serwer SSH/SCP |
| `OpenVPN` | Połączenie z siecią laboratoryjną |

---

## 📁 Struktura środowiska laboratoryjnego

```
Kali Linux (atakujący)
    │
    │ OpenVPN (10.65.0.x)
    │
    ├── 192.168.100.5 — NETWORK PROTOCOLS (maszyna ćwiczeniowa)
    │       SSH: student / pjatk
    │       Zadania 5.1–5.14
    │
    └── 192.168.200.55 — NETWORK PROTOCOLS PENTEST (cel)
            Dostępny tylko przez 192.168.100.5
            Porty: 22/tcp (SSH), 80/tcp (HTTP), 5355/tcp
```

---

## 📚 Materiały i referencje

- [Types of Network Protocols](https://www.geeksforgeeks.org/types-of-network-protocols-and-their-uses/)
- [Networking Basics: TCP, UDP, TCP/IP, OSI Models](https://www.pluralsight.com/blog/it-ops/networking-basics-tcp-udp-tcpip-osi-models)
- [Nmap Reference Guide](https://nmap.org/book/man.html)
- [OWASP — Sensitive Data Exposure](https://owasp.org/www-project-top-ten/)

---

## ⚠️ Disclaimer

Wszystkie ćwiczenia były realizowane **wyłącznie w izolowanym środowisku laboratoryjnym** udostępnionym przez uczelnię. Przedstawione techniki służą celom edukacyjnym — rozwijaniu umiejętności defensywnych i zrozumieniu metod atakujących.

> *"Żeby skutecznie bronić, trzeba rozumieć jak atakować."*

---

*Polsko-Japońska Akademia Technik Komputerowych | Protokoły Sieciowe | Zajęcia nr 5*
