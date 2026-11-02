# Fix: Szafir Host + SimplySign na macOS (Apple Silicon)

Ten poradnik rozwiązuje problemy z działaniem aplikacji **Szafir Host** oraz **SimplySign** na komputerach Mac z procesorami **M1, M2 oraz M3**. Naprawa eliminuje błędy typu "crash" (zamknięcie aplikacji), błędy pamięci oraz brak wykrywania certyfikatów w chmurze.

---

# Instrukcja naprawy: Szafir Host + SimplySign na macOS (Apple Silicon)

**Problem:** Aplikacja Szafir Host nie widzi bibliotek, zawiesza się lub wyłącza (crash) podczas próby podpisu w chmurze na komputerach z procesorem M1/M2/M3.

### Wymagania wstępne
Sprawdzone na: **SimplySign Desktop 2.10.22**, **SzafirHost 1.1.0** (oficjalnie dostępna jest już 1.2.2, nie testowane), Temurin 17 x64, macOS na Apple Silicon.

* Zainstalowany **SimplySign Desktop** (aplikacja musi być aktywna, a użytkownik musi być zalogowany).
* Zainstalowany **Szafir Host**.
* Zainstalowana w przeglądarce wtyczka **Szafir**

---

### Krok 1: Zainstaluj Javę 17 (Wersja Intel/x64)
Szafir na Macu wymaga starej architektury (Intel), aby poprawnie współpracować z wbudowanymi bibliotekami. **Nie instaluj wersji ARM64!**

1.  Pobierz **Eclipse Temurin JDK 17 (LTS)**.
2.  Wybierz architekturę **x64** (nie aarch64!).
3.  Zainstaluj pakiet `.pkg`.

### Krok 2: Skopiuj bibliotekę SimplySign do Szafira
Szafir ma problem z dostępem do ukrytych plików systemowych (`/usr/local/lib`), więc kopiujemy bibliotekę bezpośrednio do folderu aplikacji.

Otwórz Terminal i wpisz:

```bash
cp /usr/local/lib/libSimplySignPKCS.dylib /Applications/SzafirHost/
sudo xattr -cr /Applications/SzafirHost/libSimplySignPKCS.dylib
```

> **Po każdej aktualizacji SimplySign Desktop skopiuj bibliotekę ponownie** (te same dwie komendy). W `/usr/local/lib` jest tylko symlink do aktualnej wersji, a kopia w SzafirHost się nie aktualizuje.

### Krok 3: Usuń wadliwy wrapper Szafira (Naprawa Crasha)
Wbudowana biblioteka Szafira (`libpkcs11wrapper`) w wersji Intel powoduje błędy pamięci (`SIGABRT`) przy próbie załadowania na nowszych systemach. Musimy ją wyłączyć (zmienić nazwę), aby Java użyła własnego, stabilnego mechanizmu.

Wpisz w Terminalu:

```bash
mv /Applications/SzafirHost/intel64/libpkcs11wrapper.jnilib /Applications/SzafirHost/intel64/libpkcs11wrapper.jnilib.bak
```

### Krok 4: Napraw skrypt startowy (SzafirHost.sh)
Musimy wymusić na aplikacji pracę w konkretnym folderze i z konkretną wersją Javy, inaczej program nie znajdzie swoich plików.

1. Edytuj plik w Terminalu:
   ```bash
   nano /Applications/SzafirHost/SzafirHost.sh
   ```

2. Skasuj obecną zawartość i wklej dokładnie ten kod:
   ```bash
   #!/bin/bash
   cd /Applications/SzafirHost
   export JAVA_HOME="/Library/Java/JavaVirtualMachines/temurin-17.jdk/Contents/Home"
   exec "$JAVA_HOME/bin/java" $SZAFIR_HOST_OPTS -Djava.library.path=/Applications/SzafirHost -jar "SzafirHost.jar" "$@"
   ```

3. Zapisz (`Control + O`, `Enter`) i wyjdź (`Ctrl + X`).

4. Nadaj uprawnienia uruchamiania:
   ```bash
   chmod +x /Applications/SzafirHost/SzafirHost.sh
   ```

### Krok 5: Skonfiguruj wskazanie na kartę (config.xml)
Wskazujemy lokalny plik z Kroku 2 i wymuszamy Slot 1 (z nowszym SimplySign Desktop 2.10.x działa slot 1; jeśli nadal widzisz "Signing certificates not read", spróbuj `0`). Jest to kluczowe, aby uniknąć błędu "Signing certificates not read".

1. Edytuj plik w Terminalu (lub utwórz go - może się zdarzyć, że nie ma go w katalogu z aplikacją):
   ```bash
   nano /Applications/SzafirHost/config.xml
   ```

2. Skasuj obecną zawartość i wklej ten kod:
   ```xml
   <?xml version="1.0" encoding="UTF-8" standalone="yes"?>
   <configuration>
       <crypto>
           <modules>
               <module path="/Applications/SzafirHost/libSimplySignPKCS.dylib">
                   <slotListIndex>1</slotListIndex>
               </module>
           </modules>
       </crypto>
   </configuration>
   ```

3. Zapisz (`Control + O`, `Enter`) i wyjdź (`Ctrl + X`).

---

### Krok 6: Finał
1.  **Zrestartuj przeglądarkę** (Koniecznie: `Cmd + Q` nie tylko zamknięcie okna).
2.  Upewnij się, że **SimplySign Desktop** jest uruchomiony i połączony.
3.  Wejdź na stronę (np. wFirma) i podpisz dokument. Powinieneś otrzymać powiadomienie z aplikacji (podanie PIN).

---

### Znany problem
Nawet po naprawie SzafirHost może czasem wysypać się z `SIGSEGV` w `libccpkip11.dylib` (log `hs_err_pid*.log` w katalogu aplikacji).
