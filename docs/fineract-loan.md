# Moduł Pożyczek (fineract-loan)

[Powrót do dokumentacji głównej](README.md)

## Opis
Moduł `fineract-loan` jest jednym z najważniejszych i największych komponentów systemu Apache Fineract. Odpowiada on za pełny cykl życia pożyczki (Loan) i kredytu, od momentu jego zaakceptowania, przez wypłatę (Disbursement), naliczanie odsetek i harmonogramowanie (Scheduling), po spłaty (Repayment) oraz zamknięcie pożyczki (lub odpisanie w straty - Charge-Off/Write-Off). 

Moduł ten realizuje kluczowe zasady biznesowe dla produktów kredytowych w tym:
* Generowanie różnych typów harmonogramów (raty równe, malejące, odsetki płatne z góry/z dołu).
* Obsługę prowizji i kar (Charges & Penalties).
* Wyliczanie zaległości i śledzenie wskaźników Delinquency (opóźnienia w spłacie).
* Przewalutowania, reschedulowanie i renegocjacje warunków (Reschedule, Re-aging, Re-amortization).

## Kluczowe komponenty

| Komponent (Pakiet/Klasa) | Odpowiedzialność biznesowa i techniczna |
| :--- | :--- |
| **`Loan`**, **`LoanTransaction`**, **`LoanRepaymentScheduleInstallment`** | Encje domenowe JPA stanowiące rdzeń danych pożyczki. Przechowują odpowiednio: główny rekord kredytu, każdą operację finansową i niefinansową, oraz definicję rat harmonogramu. |
| **`LoanProduct`** (`portfolio.loanproduct`) | Definicja szablonu pożyczki (np. Oprocentowanie, Typ Amortyzacji, Strategia księgowania). Instancje pożyczek są tworzone na podstawie konfiguracji `LoanProduct`. |
| **`LoanWritePlatformService`** | Główny fasadowy serwis domenowy do operacji zmieniających stan pożyczki (np. Akceptacja, Wypłata środków, Spłata). Deleguje akcje do bardziej wyspecjalizowanych serwisów. |
| **`LoanRepaymentScheduleTransactionProcessor`** | Wzorzec strategii określający, w jakiej kolejności alokowane są środki ze spłaty (np. najpierw Kary -> Prowizje -> Odsetki -> Kapitał). Implementacje np. `MifosStandard`, `Creocore`, `RBIIndia`. |
| **`LoanScheduleGenerator`** | Generowanie oraz przeliczanie harmonogramów spłat (rat) bazujących na datach, kwocie oraz wybranej matematyce finansowej (np. Flat, Declining Balance). |
| **`Delinquency`** (`portfolio.delinquency`) | Pakiet usług określający w jaki sposób system rozpoznaje i kategoryzuje opóźnienia w spłatach oraz śledzi tzw. koszyki opóźnień (Delinquency Buckets). |

## Architektura modułu

Architektura zachowuje warstwowość typową dla aplikacji Spring Boot, wdrażając dodatkowo mechanizmy CQRS z podziałem na Read i Write Platform.

```plantuml
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml
title Model C4 - Komponenty modułu fineract-loan

Component(command_api, "Command Handlers", "Spring Component", "Przechwytują komendy dla domen Loan (np. LoanRepaymentCommandHandler)")
Component(loan_write_service, "LoanWritePlatformService", "Serwis (Transaction)", "Orkiestracja logiki biznesowej dla modyfikacji (Wypłaty, Spłaty)")
Component(loan_read_service, "LoanReadPlatformService", "Serwis (JDBC)", "Szybki odczyt danych pożyczki z pominięciem JPA (użycie surowego JDBC / RowMapperów)")
Component(loan_domain_service, "LoanAccountDomainService", "Serwis Domenowy", "Enkapsulacja reguł tworzenia transakcji i wyliczania sald pożyczki")
Component(schedule_generator, "LoanScheduleGenerator", "Serwis Domenowy", "Generowanie harmonogramów (rat)")

SystemDb_Ext(db, "Relational Database", "Baza Danych Dzierżawcy (Tenanta)")

Rel(command_api, loan_write_service, "Deleguje akcje zapisujące")
Rel(loan_write_service, loan_domain_service, "Implementacja logiki biznesowej")
Rel(loan_domain_service, schedule_generator, "Żąda utworzenia/aktualizacji harmonogramu")
Rel(loan_domain_service, db, "Aktualizuje encje JPA (Zapis)")
Rel(loan_read_service, db, "Bezpośrednie zapytania SQL (Odczyt)")

@enduml
```

## Przepływ danych (Przykład: Rejestracja Spłaty Pożyczki)

Poniższy diagram ilustruje przepływ danych w trakcie księgowania standardowej spłaty pożyczki:

```plantuml
@startuml
title Sekwencja - Rejestracja Spłaty (Repayment)

actor Klient / Kasjer as user
participant "API / CommandHandler" as api
participant "LoanTransactionService" as tx_service
participant "LoanAccountDomainService" as domain_service
participant "Loan" as loan_entity
participant "TransactionProcessor" as tx_processor
participant "BusinessEventPublisher" as event_publisher
participant "Accounting / EventListener" as accounting

user -> api: POST /loans/{id}/transactions (spłata)
api -> tx_service: makeRepayment(loanId, repaymentData)
activate tx_service
tx_service -> domain_service: makeRepayment(...)
activate domain_service
domain_service -> loan_entity: pobierz encję Loan
domain_service -> tx_processor: alokuj wpłaconą kwotę (Interest, Principal, Fees)
activate tx_processor
tx_processor -> loan_entity: aktualizuj Installment (Raty harmonogramu)
tx_processor --> domain_service: wynik alokacji
deactivate tx_processor
domain_service -> loan_entity: dodaj LoanTransaction (typ=REPAYMENT)
domain_service --> tx_service: zaktualizowany Loan
deactivate domain_service
tx_service -> db: Zapisz zmiany w bazie danych
tx_service -> event_publisher: publikuj LoanTransactionMakeRepaymentPostBusinessEvent
deactivate tx_service

event_publisher -> accounting: Odbiór zdarzenia spłaty asynchronicznie/synchronicznie
accounting -> accounting: Wygeneruj wpisy do Księgi Głównej (Journal Entries)

@enduml
```

## Zależności wewnętrzne i Integracje

*   **`fineract-accounting`**: Moduł pożyczek sam w sobie nie zapisuje na konta księgowe (General Ledger). Każde zdarzenie modyfikujące status finansowy (np. `LoanTransactionMakeRepaymentPostBusinessEvent`, `LoanDisbursalBusinessEvent`) generuje obiekt `BusinessEvent`, który jest nasłuchiwany przez silnik księgowy. Silnik ten na podstawie mapowań przypisuje odpowiednie konta GL.
*   **`fineract-charge`**: Wykorzystywany współdzielenie definicji opłat (Charge) stosowanych w pożyczkach.
*   **Wewnętrzne Joby (fineract-cob)**: Moduł posiada zdefiniowane kroki *Close of Business* (COB) typu `LoanCOBBusinessStep`, odpowiedzialne za np. naliczanie zaległości każdego dnia w nocy.

## Zarządzanie stanem i baza danych

Model danych skupiony jest wokół tabel prefiksowanych najczęściej `m_loan`. Najważniejsze z nich to:

*   `m_loan`: Rekord główny instancji kredytu (zawiera kwoty główne, daty wypłaty, referencję do `m_client` oraz `m_product_loan`).
*   `m_product_loan`: Konfiguracja wybranego produktu pożyczkowego, z którego utworzono `m_loan`.
*   `m_loan_transaction`: Historia transakcji dokonywanych na kredycie (Wypłaty, Spłaty, Zwolnienia z opłat, Nałożenie kar).
*   `m_loan_repayment_schedule`: Harmonogram spłat. Każdy rekord odpowiada jednej racie, rozpisanej z uwzględnieniem kwoty kapitału (`principal`), odsetek (`interest`), opłat (`fees`) i kar (`penalties`). Przechowuje informację o kwotach należnych (Due) i zapłaconych (Paid).
*   `m_loan_charge`: Skonkretyzowane obciążenia dla danej pożyczki (opłaty, prowizje).
*   `m_loan_status_change_history`: Audyt przejść pożyczki w cyklu życia (np. PENDING -> APPROVED -> ACTIVE -> CLOSED).
*   `m_loan_delinquency_tags`: Przypisanie koszyków opóźnień do rat i pożyczki. 
