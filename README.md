# Linux Privilege Escalation — Write-up
**Platforma:** TryHackMe  
**Room:** Linux PrivEsc  
**Data:** Marzec 2026  
**Poziom:** Beginner/Intermediate 

## Środowisko
Maszyna docelowa: Debian Linux  
Cel: eskalacja uprawnień z użytkownika "user" do "root"  
Punkt startowy: shell z ograniczonymi uprawnieniami

## Wprowadzenie
Po uzyskaniu początkowego dostępu do maszyny dysponuję shellem 
z ograniczonymi uprawnieniami — mogę wykonywać tylko część komend, 
czytać wybrane pliki i nie mogę modyfikować zasobów systemowych.
Celem privilege escalation jest uzyskanie uprawnień roota — 
pełnej kontroli nad systemem.

### 1. Service Exploit — MySQL
**Mechanizm:** MySQL działał jako root z pustym hasłem. 

**Exploitacja:** Użyłem exploita EDB-ID: 1518 (2006) który wykonuje 
kod w kontekście procesu MySQL. 

**Dlaczego działa:** Uzyłem niezabezpieczonego hasłem MySQL do uzyskania powłoki roota poprzez exploit z 2006 roku o EDB-ID: 1518. MySQL działał jako root więc kod wykonany przez MySQL miał uprawnienia roota. 

**Efekt:** Shell roota. 

### 2. Złe Uprawnienia — /etc/shadow
**Mechanizm:** Plik /etc/shadow posiada hashe wszystkich uzytkownikow i standardowo jest do odczytu tylko przez roota. 
**Exploitacja:** Odczytalem plik /etc/shadow poprzez błąd w dostępie do pliku. 
**Narzędzie:** John the Ripper, po odczytaniu hasha uruchomiłem na mojej maszynie, lokalnie program i byłem w stanie uzyskać hasło. Jest to groźne dlatego, że łamanie bylo offline i nie zostawiło logów. 
**Efekt:** Uzyskanie hasła roota. 

### 3. Sudo Exploit — apache2 (File Disclosure)
**Mechanizm:** Plik /etc/shadow nie był dostępny bezpośrednio. 
Znalazłem technikę na GTFOBins wykorzystującą apache2.
**Exploitacja:** sudo apache2 -C 'Include /etc/shadow'. 
**Dlaczego działa:** Apache próbował sparsować /etc/shadow 
jako plik konfiguracyjny. Nie rozumiejąc składni hashy — 
ujawnił hash roota w komunikacie błędu. 
**Efekt:** Hash roota ujawniony bez bezpośredniego dostępu do pliku. 

### 4. PATH Hijacking
**Mechanizm:** Zmienna PATH określa kolejność katalogów 
w których system szuka programów. Przeszukiwane są od lewej do prawej.
**Podatność w kodzie:** Developer wywołał "service" bez pełnej 
ścieżki (/usr/sbin/service) — system szukał go w PATH.
**Exploitacja:** 
1. Skompilowałem złośliwy kod C do pliku wykonywalnego "service":
   gcc -o service /home/user/tools/suid/service.c
2. Zmieniłem PATH tak żeby aktualny katalog był przeszukiwany pierwszy:
   PATH=.:$PATH /usr/local/bin/suid-env
3. suid-env szukał "service" — znalazł mój plik zamiast prawdziwego.
**Efekt:** Shell roota.

### 5. LD_PRELOAD
**Mechanizm:** Biblioteki dynamiczne są ładowane w momencie 
uruchomienia programu — nie podczas kompilacji. 
LD_PRELOAD pozwala wskazać bibliotekę która zostanie 
załadowana pierwsza, przed wszystkimi innymi.
**Podatność:** Program był dostępny przez sudo (sudo -l).
Sudo zachowało zmienną LD_PRELOAD — co jest błędem konfiguracji.
**Exploitacja:** 
1. Skompilowałem złośliwą bibliotekę .so, która umożliwia uzyskanie shell roota
2. Ustawiłem LD_PRELOAD wskazując na tę bibliotekę
3. Uruchomiłem program przez sudo
4. System załadował moją bibliotekę PRZED właściwymi — 
   wykonała się jako root
**Różnica względem PATH Hijacking:** 
PATH podmienia program, LD_PRELOAD podmienia bibliotekę 
ładowaną przez program.
**Efekt:** Shell roota.

