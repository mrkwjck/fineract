# Moduł Samoobsługi Klienta (Self-Service API)

[Powrót do dokumentacji głównej](README.md)

## Opis
Główny rdzeń API systemu Apache Fineract budowany jest z myślą o użytkownikach "Back-Office" (Pracownikach banku, Kasjerach, Oficerach kredytowych). Takie końcówki pozwalają na wyciągnięcie pełnych informacji o każdym kliencie w ramach oddziału. 
Jednak we współczesnej bankowości istnieje potrzeba tworzenia aplikacji mobilnych (np. bankowości internetowej), gdzie ostatecznym konsumentem API jest sam Klient (Użytkownik końcowy). 

Pakiet `fineract-provider/.../portfolio/self` (Self-Service) odpowiada za dostarczenie specjalnie wyizolowanych i zabezpieczonych końcówek REST API (z prefiksem `/self/...`), dedykowanych wyłącznie dla portali klienckich i aplikacji mobilnych.

## Kluczowe mechanizmy

| Funkcja | Opis implementacji |
| :--- | :--- |
| **Izolacja Danych (Data Scoping)** | Standardowe API pożyczki pozwala odczytać `GET /loans/1`. W trybie Self-Service żądanie kierowane jest na `GET /self/loans/1`. Różnica polega na tym, że mechanizm bezpieczeństwa upewnia się, że autoryzowany poprzez OAuth2 użytkownik (AppUser z rolą Self-Service) jest fizycznym właścicielem tego rekordu, zapobiegając atakom IDOR (Insecure Direct Object Reference). |
| **Proces Tworzenia Konta (Self Registration)** | Zewnętrzni użytkownicy mogą samodzielnie otworzyć profil kliencki, podając swoje dane w formularzu, zanim zostaną zweryfikowani przez procedury KYC. |
| **Własne Transfery (Self Transfers)** | Zabezpieczony mechanizm (`/self/accounttransfers`) pozwalający klientowi we własnej aplikacji mobilnej wykonać polecenie przelewu z jego konta oszczędnościowego na spłatę własnej raty kredytu (Loan Repayment) bądź przelew na zdefiniowane konto zewnętrzne. |

## Zależności i Architektura

Moduł "Self" działa w Fineract jako wzorzec Proxy/Fasady.
1. Odbiera zapytania na ścieżkach `/self/*`.
2. Weryfikuje mapowanie pomiędzy zalogowanym `AppUser` a `Client ID` w bazie danych (Czy ten login bankowości mobilnej należy do tego właściciela konta?).
3. Jeśli tak, w locie podmienia kontekst i deleguje (Forward) żądanie do normalnych, ukrytych niżej serwisów z innych modułów domenowych (np. wstrzykuje wywołanie do `LoanReadPlatformService` w `fineract-loan`).

Dzięki temu programiści utrzymują tylko jedną logikę biznesową dla pożyczek, wystawiając ją w bezpieczny sposób na zewnątrz przez ten moduł fasadowy.
