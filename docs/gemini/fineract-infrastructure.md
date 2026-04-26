# Usługi Infrastrukturalne: SMS, Email, SPM (fineract-infrastructure)

[Powrót do dokumentacji głównej](README.md)

## Opis
Katalog `fineract-provider/src/main/java/org/apache/fineract/infrastructure` to oprócz kodu uruchomieniowego dom dla kluczowych sub-modułów i zintegrowanych systemów wsparcia operacyjnego w Fineract. Należą do nich między innymi procesy automatyzacji marketingu, powiadomienia klientów (Notification) o zbliżającej się racie oraz zbieranie danych społecznych.

## Kluczowe Moduły Wewnętrzne

### 1. SMS i Email (Kampanie i Powiadomienia)
Nowoczesne platformy pożyczkowe muszą ostrzegać klientów o ryzyku braku spłaty (Arrears) lub witać w systemie. 
*   **`Campaigns` (Kampanie):** Rozbudowany silnik pozwalający na definiowanie reguł biznesowych zapytań (np. "Wybierz wszystkich klientów, których pożyczka wchodzi w zaległość pojutrze") i podpięcie ich pod wbudowanego crona. 
*   **Bramki SMS/Email:** Silnik kampanii po "trafieniu" odpowiednich klientów renderuje treść wiadomości (wykorzystując wbudowane szablony np. `Drogi {{clientName}}, Twoja rata w wysokości {{amount}} upływa jutro`) i odkłada do kolejki. Następnie zintegrowane konektory wypychają żądania do zewnętrznych operatorów bramkowych (np. Twilio, Infobip) lub za pomocą zewnętrznych konektorów (Apache Camel).

### 2. Notification (Powiadomienia wewnątrz-systemowe)
Nie mylić z wiadomościami do Klienta. Moduł notyfikacji działa dla użytkowników panelu zarządzania (Staff / AppUser). Jeżeli w module CQRS `fineract-command` Kasjer (Maker) złoży wniosek o wypłatę gotówki przekraczający jego limit, moduł notyfikacji automatycznie wygeneruje alert systemowy wędrujący na ekran Menedżera Oddziału (Checker), by ten to zaakceptował.

### 3. SPM (Social Performance Management)
Framework ankiet i punktacji zintegrowany ściśle ze stowarzyszeniami zajmującymi się ubóstwem i mikro-kredytami (np. The PPI - Progress out of Poverty Index).
*   Pozwala na tworzenie w banku dynamicznych ankiet (Surveys) - zestawów pytań z predefiniowanymi wagami punktowymi (np. "Ile posiłków dziennie spożywa rodzina?").
*   Ankieter udając się w teren by zakwalifikować kogoś do pomocy społecznej (Loan), wypełnia ankietę w systemie Fineract, a moduł SPM na podstawie logiki wylicza tzw. PPI Score i załącza ten wynik w postaci punktowej do decyzyjności wniosku kredytowego klienta, udowadniając zgodność działań fundacji czy banku z ich statutowymi celami CSR (Corporate Social Responsibility).
