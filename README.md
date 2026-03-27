# Linux Privilege Escalation — Write-up
**Platforma:** TryHackMe  
**Room:** Linux PrivEsc  
**Data:** Marzec 2026  
**Poziom:** Beginner/Intermediate 

## Środowisko
Maszyna docelowa: Debian Linux  
Cel: eskalacja uprawnień z użytkownika "user" do "root"  
Punkt startowy: shell z ograniczonymi uprawnieniami

## Narzędzia
- John the Ripper - łamanie hashy offline  
- GTFOBins - baza podatnych binariów  
- gcc - kompilacja exploitów w C  
- msfvenom - generowanie payloadów  

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
**Mechanizm:** Biblioteki dynamiczne są ładowane w momencie uruchomienia programu nie podczas kompilacji. LD_PRELOAD pozwala wskazać bibliotekę która zostanie załadowana pierwsza, przed wszystkimi innymi.  
**Podatność:** Program był dostępny przez sudo (sudo -l). Sudo zachowało zmienną LD_PRELOAD — co jest błędem konfiguracji.  
**Exploitacja:** 
1. Skompilowałem złośliwą bibliotekę .so, która umożliwia uzyskanie shell roota
2. Ustawiłem LD_PRELOAD wskazując na tę bibliotekę
3. Uruchomiłem program przez sudo
4. System załadował moją bibliotekę PRZED właściwymi, wykonała się jako root
**Różnica względem PATH Hijacking:** 
PATH podmienia program, LD_PRELOAD podmienia bibliotekę ładowaną przez program.  
**Efekt:** Shell roota.  

### 6. Crontab
**Mechanizm:** Crontab wykonuje zadania cyklicznie czyli jeśli zadanie jest uruchamiane przez roota, to złośliwy kod również wykona się jako root.  
**Podatność:** Plik wykonywany co minutę byl źle skonfigurowany, mógł zmienić go każdy.  
**Exploitacja:** Zmieniłem skrypt, który pozwolił mi połączyć sie mojej maszynie jako root.  
**Efekt:** Shell roota.  

### 7. SSH Key
**Mechanizm:** Klucz SSH składa się z pary: prywatny (u klienta) i publiczny (na serwerze). Posiadanie klucza prywatnego zastępuje hasło. Umożliwia logowanie bez jego znajomości.  
**Podatność:** Klucz prywatny roota znajdował się w /.ssh/ z błędnymi uprawnieniami, był czytelny dla każdego użytkownika.  
**Exploitacja:** Skopiowałem klucz prywatny na swoją maszynę i użyłem go do zalogowania jako root: ssh -i id_rsa root@[IP]  
**Efekt:** Dostęp jako root bez znajomości hasła.  

### 8. History/Config files
**Mechanizm:** Mechanizm jest bardzo prosty lecz ciężki do uniknięcia w 100%. Jest to groźne z powodu braku generowania alertów, jest to ciche rozwiązanie
**Podatność:**  Polega na odczytaniu historii komend uzytych np cat .bash_history lub config.php i znalezienia wrażliwych danych z punktu widzenia systemu.
**Exploitacja:** cat ~/.bash_history - historia komend z hasłami. cat config.php - hasła do bazy danych w plaintext
**Efekt:** dostęp do wrazliwych danych

## Wnioski dla Blue Teamu

### Co wykryć i jak zapobiec:
**MySQL Service** : Aktualizacja, ponieważ exploit ktory wykorzystalem jest z 2006 roku. Serwis nie powinien działać jako root, zastosować zasadę least privilege.  
**/etc/shadow** : Właściwe ograniczenie uprawnień.  
**apache2 sudo** :  Ograniczenie używania przez sudo.  
**PATH Hijacking** : Podawanie pełnej ścieżki do pliku ktory ma zostac wykonany.  
**LD_PRELOAD** : Sudo nie powinien zachowywać zmiennej LD_PRELOAD.  
**Crontab** : Ograniczenie uprawnien do plików wykonywanych.  
**SSH Key** : Dobre schowanie klucza a nie tylko ukrycie.  
**History/Config** : Hasła nie powinny być przechowywane w plikach konfiguracyjnych jako plaintext. Należy używać secrets managera lub zmiennych środowiskowych.  

## Kluczowy wniosek
Największym zagrożeniem nie są skomplikowane exploity lecz błędy konfiguracji i złe zarządzanie uprawnieniami, które są trudne do wykrycia przez Blue Team.
