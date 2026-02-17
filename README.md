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

• Zarządzanie połączeniami zdalnymi: Bezpieczne logowanie przez SSH z wykorzystaniem haseł oraz kluczy publicznych/prywatnych.
• Poruszanie się w terminalu: Efektywne korzystanie ze skrótów klawiszowych i historii komend.
• Struktura systemu: Zrozumienie hierarchii katalogów (np. /etc dla konfiguracji, /bin dla plików wykonywalnych).
• Diagnostyka systemu i sieci: Sprawdzanie zasobów sprzętowych (CPU, RAM, dyski) oraz aktywnych połączeń sieciowych.
• Automatyzacja i usługi: Zarządzanie procesami w tle (systemd) oraz zadaniami harmonogramu (Cron).
• Eskalacja uprawnień: Praktyczne wykorzystanie błędnej konfiguracji uprawnień plików do przejęcia konta innego użytkownika.

--------------------------------------------------------------------------------
2. Komendy i ich znaczenie
   
Uzytkownik:

• whoami - Wyświetla nazwę aktualnie zalogowanego użytkownika w systemie + Przydatne, gdy chcesz sprawdzić, na jakim koncie pracujesz np. serwer lub kontener
• echo $USER - Wypisuje wartość zmiennej środowiskowej USER, czyli nazwę bieżącego użytkownika.
• env - Wyświetla wszystkie zmienne środowiskowe dostępne w aktualnej sesji. Często używane do: debugowania, sprawdzania konfiguracji aplikacji, pracy z Dockerem
• id - Pokazuje informacje o użytkowniku, UID (User ID), GID (Group ID), grupy, do których użytkownik należy - **Bardzo ważne - przydatne przy sprawdzaniu uprawnień w systemie Linux.**

Zarządzanie systemem i sprzętem: 

• uname -a – Wyświetla pełne informacje o jądrze i architekturze systemu.
• cat /proc/cpuinfo oraz cat /proc/meminfo – Szczegółowe dane o procesorze i pamięci RAM.
• lsblk – Listuje urządzenia blokowe (dyski i partycje).
• lspci - Wyświetla urządzenia podłączone do magistrali PCI, czyli pokazuje kartę graficzną, kartę sieciową i kontrolery dysków - Przydatne do sprawdzania sprzętu wewnętrznego komputera
• lsusb - - Wyświetla urządzenia podłączone przez USB, np. mysz. klwiatura, kamera, pendrive i telefon 
• df -h – Pokazuje zużycie miejsca na dyskach w czytelnym formacie.
• free -h – Wyświetla ilość wolnej i zużytej pamięci RAM.
• top – Dynamiczny podgląd uruchomionych procesów i obciążenia systemu.

Operacje na plikach i wyszukiwanie: 

• ls -la – Listuje wszystkie pliki (w tym ukryte) z ich szczegółowymi uprawnieniami.
• touch [plik] – Tworzy nowy, pusty plik.
• cp, mv, rm – Odpowiednio: kopiowanie, przenoszenie/zmiana nazwy i usuwanie plików.
• find / -name '*.conf' 2>/dev/null – Wyszukuje pliki konfiguracyjne w całym systemie, ignorując błędy dostępu.
• grep [fraza] [plik] – Wyszukuje konkretny tekst wewnątrz pliku.

Sieć i SSH:

• ssh [user]@[ip] – Łączenie się ze zdalnym serwerem.
• ssh-keygen – Generowanie pary kluczy SSH.
• ip a – Wyświetla adresy IP skonfigurowane na interfejsach sieciowych.
• ss -tlp – Listuje procesy nasłuchujące na portach TCP.
• cat /etc/resolv.conf – Sprawdzanie skonfigurowanych serwerów DNS.

Zarządzanie pakietami (apt):

• apt update – Odświeża listę pakietów z repozytoriów.
• apt upgrade – Aktualizuje zainstalowane oprogramowanie.
• apt autoremove – Usuwa niepotrzebne już zależności.

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


