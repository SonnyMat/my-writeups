# Writeup: 2. Enumeracja sieci & Pentest Scenario

🛡️ Enumeracja sieci & Pentest Scenario
Projekt edukacyjny: Rekonesans usług sieciowych i eksploatacja błędnych konfiguracji.

## 📖 O projekcie
Celem projektu było przeprowadzenie pełnego procesu enumeracji (rozpoznania) maszyny wewnątrz sieci laboratoryjnej oraz odzyskanie ukrytych "flag" (plików tekstowych) z różnych usług sieciowych. Projekt symuluje realne błędy konfiguracyjne, które często spotyka się w środowiskach korporacyjnych.
---

Czego się nauczyłam? 
- Metodyka skanowania (Nmap): Dobór odpowiednich technik skanowania (np. Stealth Scan, Connect Scan) w celu wykrywania aktywnych hostów i otwartych portów.
- Fingerprinting usług: Rozpoznawanie dokładnych wersji oprogramowania oraz systemów operacyjnych na podstawie charakterystycznych cech odpowiedzi sieciowych (-sV, -O).
- Automatyzacja z NSE: Wykorzystanie silnika skryptów Nmap (Nmap Scripting Engine) do automatycznego wykrywania błędów konfiguracyjnych i znanych podatności.
- Enumeracja protokołów (FTP/SMB/HTTP): Praktyczne zdobywanie danych z konkretnych usług, np. poprzez dostęp anonimowy (FTP) czy listowanie udziałów sieciowych (SMB).
- Analiza ścieżki sieciowej: Wykorzystanie narzędzi diagnostycznych (Traceroute) do mapowania struktury sieci i identyfikacji maszyn pośredniczących.
- Eksploatacja błędnych konfiguracji: Wykorzystywanie publicznie dostępnych zasobów (tzw. Null Sessions lub Anonymous Login) do pozyskiwania wrażliwych plików (flag).
--------------------------------------------------------------------------------
## 🛠️ Wykorzystane Narzędzia
Nmap: Skanowanie portów, detekcja wersji usług i systemów operacyjnych.

SMBClient: Interakcja z udziałami sieciowymi Windows

FTP/cURL/Wget: Pobieranie danych z serwerów plików i serwerów WWW.

## 🚀 Przebieg  
Krok 1: Nmap

Bash
nmap -sT -sV -sC -O --osscan-guess 192.168.***.**
-sT - inaczej TCP Connect Scan, tworzy tzw. 3 handshake, który działa bez uprawnień root - dla hakerów jest "głośny" zostawia ślady w logach, może być wykryty przez IDS/IPS (Detection i Prevention System - alarm i powstrzyamnie) 
-sV - próbuje określić wersję usługi na porcie, czyli 22/tcp open 
-sC - Uruchamia domyślne skrypty NSE (Nmap Scripting Engine), czyli może wykrywać podatności, niebezpieczne konfiguracje, anon FTP (logowanie anonimowe - FTP to protokół do przesyłania plików - możliwy nieautoryzowany dostęp do plików), SSL issues (SSL/TLS służy do bezpiecznej komunikacji - ryzyko MITM, złe szyfrowanie, compliance), SMB info (Server Message Block) to protokół do współdzielenia plików i drukarek w sieci - Potencjalne podatności w sieci i lateral movement
-O - 

Krok 2: Eksploatacja HTTP (Serwer WWW)
Serwer udostępniał pliki bez żadnego zabezpieczenia.

Bash
wget http://192.168.***.**/flag.txt


Krok 3: Eksploatacja FTP (Brak autoryzacji)
Wiele serwerów pozwala na tzw. "Anonymous Login".

Bash
ftp 192.168.***.**
# Logowanie jako: anonymous


Krok 4: Eksploatacja SMB (Udziały sieciowe)
SMB to protokół do współdzielenia plików w sieci. Sprawdzam, co jest dostępne:

Bash
smbclient -N -L //192.168.200.52/  # Listowanie zasobów
smb: \> ls                         # Wyświetlenie zawartości (list)
smb: \> get flag2.4p.txt           # Pobranie pliku
Oczami Pentestera:

ls (List): To jak zajrzenie do wspólnej szafki w biurze. Widzę wszystkie segregatory i sprawdzam, co jest w środku.

-N (No password): Wchodzę tam "na pewniaka", bo szafka nie ma kłódki.

🎓 Czego się nauczyłam?
Kolejność działań: Najpierw dokładny zwiad (Nmap), potem precyzyjny atak. Bez dobrych informacji marnujesz czas.

Błędy konfiguracji (Misconfigurations): Większość włamań nie wynika z "magicznego hakerstwa", ale z pozostawienia domyślnych haseł lub ich braku (jak w przypadku SMB i FTP).

Automatyzacja: Skrypty NSE w Nmapie potrafią wykryć podatności szybciej niż człowiek.

📈 Jak uruchomić?
Skonfiguruj tunel SSH do sieci laboratoryjnej.

Zainstaluj nmap oraz smbclient.

Wykonaj skanowanie i spróbuj uzyskać dostęp do wymienionych usług.


Jak się chronić? 
Rekomendacje: Jak naprawić te luki? (Blue Teaming)
Jako pentester nie tylko znajduję błędy, ale też doradzam, jak je naprawić. Oto co powinien zrobić administrator maszyny 192.168.200.52:

1. Wyłączenie anonimowego dostępu (FTP i SMB)
Błąd: Serwery pozwalały na logowanie bez hasła (użytkownik anonymous lub flaga -N).

Poprawka: Należy skonfigurować usługi tak, aby wymagały silnego uwierzytelnienia (login + unikalne hasło).

Zasada: Principle of Least Privilege (Zasada najmniejszych uprawnień) – nikt nie powinien mieć dostępu do plików, których nie potrzebuje do pracy.

2. Ograniczenie widoczności (Hardening portów)
Błąd: Wszystkie usługi (HTTP, FTP, SMB) były widoczne dla każdego w sieci.

Poprawka: Konfiguracja Firewalla (np. iptables lub ufw). Jeśli serwer plików SMB ma służyć tylko działowi kadr, zablokuj dostęp dla wszystkich innych adresów IP.

Analogia: Jeśli nie zapraszasz gości, zamknij bramę na posesję, a nie tylko drzwi do domu.

3. Ukrywanie wersji usług (Banner Grabbing)
Błąd: Nmap odczytał dokładne wersje oprogramowania (-sV), co pozwala hakerowi szybko znaleźć gotowy exploit w sieci.

Poprawka: Wyłączenie "bannerów" w plikach konfiguracyjnych (np. w Apache/Nginx). Serwer powinien odpowiadać "Jestem serwerem WWW", a nie "Jestem serwerem Apache w wersji 2.4.41".

4. Szyfrowanie przesyłu danych
Błąd: FTP i HTTP przesyłają dane otwartym tekstem. Każdy w tej samej sieci mógłby "podpatrzeć" (sniffing) moje flagi.

Poprawka: Zamiana protokołów na ich bezpieczne odpowiedniki:

HTTP ➡️ HTTPS (port 443)

FTP ➡️ SFTP (port 22)

