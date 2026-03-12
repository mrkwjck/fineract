# Dokumentacja Głównej Architektury Apache Fineract

## Opis
Apache Fineract to otwartoźródłowy, bezpieczny i modularny system bankowości centralnej (Core Banking System) o ugruntowanej architekturze wspierający instytucje finansowe, banki spółdzielcze, instytucje mikrofinansowe (MFI) oraz nowoczesne Fintechy. 
Rozwiązanie to implementuje podstawowe funkcjonalności finansowe (Core Banking) ułatwiające zarządzanie klientami (CRM), portfelami kredytowymi, kontami depozytowymi, produktami oszczędnościowymi oraz kompleksowym systemem księgowym (General Ledger). Projekt został zaprojektowany zgodnie z zasadami Domain-Driven Design (DDD) w modelu modularnego monolitu (Modular Monolith) i dostarcza obszerny zestaw końcówek REST API, ułatwiając integrację i budowanie kanałów omni-channel.

## Pełna Lista Modułów
System został podzielony na wysoce wyspecjalizowane podmoduły. Główny podział obejmuje rdzeń aplikacji, domenowe moduły finansowe, zarządzanie dokumentami, podatkami oraz aplikacje klienckie.

**Rdzeń i Infrastruktura:**
- **[Fineract Provider (Główny silnik / CRM)](fineract-provider.md)**: Główny moduł integrujący, zawiera punkt startowy aplikacji Spring Boot, a także domenę zarządzania Klientami, biurami (Offices) i oficerami kredytowymi (Staff).
- **[Fineract Core](fineract-core.md)**: Fundament infrastrukturalny – mapowanie wyjątków, obsługa zdarzeń asynchronicznych (Event Broker), filtry bazodanowe dla Tenanta oraz klasy bazowe i walidacyjne JPA.
- **[Fineract Security (Bezpieczeństwo)](fineract-security.md)**: Moduł odpowiedzialny za bezpieczeństwo autoryzacji, wdrożenie Multi-Tenancy (wybór bazy w locie) oraz silnik uprawnień (RBAC).
- **[Fineract Validation (Walidacja)](fineract-validation.md)**: Współdzielony, ujednolicony silnik do logiki walidacji wielodomenowych na wprowadzanych do systemu parametrach.
- **[Fineract Command (Magistrala / CQRS)](fineract-command.md)**: Obsługa zapytań modyfikujących, Maker-Checker, Idempotentność API i trzymanie dziennika modyfikacji (Audit/Command Source).
- **[Fineract COB (Close Of Business)](fineract-cob.md)**: Asynchroniczny, wielowątkowy silnik Spring Batch do zamykania dnia księgowego (masowe naliczanie kar i odsetek z twardymi blokadami kont).

**Główne Finanse (Portfel i Księgowość):**
- **[Fineract Loan (Pożyczki)](fineract-loan.md)**: Kompleksowy cykl życia pożyczki – harmonogramy, naliczanie odsetek i kar, autoryzacja wypłat, spłaty.
- **[Fineract Loan Origination](fineract-loan-origination.md)**: Proces składania i procesowania wniosków kredytowych, etap "przed" udzieleniem samej pożyczki (Decision Engine integration).
- **[Fineract Progressive Loan](fineract-loan-origination.md)**: Nowoczesny standard pożyczek ze zmiennymi i elastycznymi harmonogramami.
- **[Fineract Working Capital Loan](fineract-working-capital-loan.md)**: Obsługa płynności finansowej oraz linii kredytowych ze zmiennym kapitałem obrotowym dla MŚP.
- **[Fineract Accounting (Księgowość)](fineract-accounting.md)**: Silnik księgowy z obsługą Księgi Głównej (GL), mapowaniem kont i księgowań podwójnych.
- **[Fineract Savings (Oszczędności i Depozyty)](fineract-savings.md)**: Obsługa kont oszczędnościowych, lokat terminowych (Fixed Deposits) oraz powtarzalnych (Recurring Deposits).

**Finanse Opcjonalne i Opłaty:**
- **[Fineract Charge (Prowizje i Opłaty)](fineract-charge.md)**: Globalny słownik i menedżer opłat oraz kar przypisywanych następnie do pożyczek, oszczędności lub klientów.
- **[Fineract Rates (Stopy Procentowe)](fineract-rates.md)**: Konfiguracja zmiennych stóp referencyjnych (np. WIBOR/LIBOR) propagowana masowo na podpięte umowy.
- **[Fineract Tax (Podatki)](fineract-tax.md)**: Mechanizmy nakładania i grupowania podatków (VAT, Withholding Tax) na transakcje finansowe i oprocentowanie depozytów.
- **[Fineract Branch (Limity Oddziałowe)](fineract-branch.md)**: Weryfikacja przesunięć i limitów zasilenia gotówką u kasjerów na przestrzeni poszczególnych oddziałów.
- **[Fineract Investor (Zarządzanie Portfelem Inwestorów)](fineract-investor.md)**: P2P Lending - sprzedaż i kupno pul pożyczkowych przez zewnętrznych inwestorów z zapewnieniem zwrotów (ROI).

**Usługi Poboczne i Raportowanie:**
- **[Fineract Report (Raporty)](fineract-report.md)**: Ekstrakcja danych Read-Only w postaci surowego SQL lub integracja z renderowaniem raportów Pentaho (PDF, XLS).
- **[Fineract MIX (Raporty XBRL)](fineract-mix.md)**: Wymiana danych standardami taksonomii MIX Market XBRL ze zintegrowanego modelu Księgi Głównej.
- **[Fineract Document (Zarządzanie Dokumentacją)](fineract-document.md)**: Przechowywanie skanów dokumentów tożsamości, umów i zdjęć z możliwością użycia AWS S3 lub lokalnego systemu plików.
- **[Fineract Share Accounts (Udziały/Dywidendy)](fineract-shareaccount.md)**: System zarządzania limitami i sprzedażą udziałów członkowskich dla unii kredytowych wraz z corocznym księgowaniem dywidend.
- **[Fineract Interoperation (Płatności Mobilne)](fineract-interoperation.md)**: Integracje standardu GSMA i Mojaloop, ujednolicające w czasie rzeczywistym wymianę transakcji z sieciami telekomunikacyjnymi Mobile Money.
- **[Fineract Infrastructure (SMS, Kampanie, SPM)](fineract-infrastructure.md)**: Submoduły utrzymujące marketing, reguły powiadomień oraz zaawansowane ankiety i metryki ubóstwa (PPI/Social Performance).
- **[Fineract Avro Schemas (Kafka)](fineract-avro-schemas.md)**: Definicja binarnych schematów kontraktów dla Zdarzeń Domenowych wychodzących z systemu na szynę Message Brokera.
- **[Fineract Doc (Dokumentacja)](fineract-doc.md)**: Silnik renderowania bazy wiedzy w standardzie Asciidoctor/Antora.
- **[Fineract DB (Migracje Baz Danych)](fineract-db.md)**: Skrypty SQL i automatyzacja migracji struktury relacyjnej Flyway/Liquibase dla modelu Multi-Tenant.
- **[Fineract Self-Service (Konta Klienta)](fineract-self-service.md)**: Wyizolowana i zabezpieczona (przed IDOR) warstwa API przeznaczona wyłącznie dla aplikacji mobilnych logujących klienta końcowego.
- **[Fineract Portfolio Transfers (Zlecenia i Arkusze)](fineract-portfolio-transfers.md)**: Narzędzia do zleceń stałych, przelewów między rachunkami oraz zbiorczych arkuszy inkasowych dla terenowych spotkań mikrofinansowych (JLG).

**Klienci SDK i Testowanie:**
- **[Fineract Clients & Tests SDK](fineract-testing.md)**: Opis natywnych wygenerowanych klientów Java/Feign oraz zestawienia gigantycznej puli testów Rest Assured (Integracyjnych, E2E i zabezpieczeń OAuth2).

**Wdrożenie:**
- **[Fineract Deployment (Infrastruktura)](fineract-deployment.md)**: Architektura wdrażania za pomocą wbudowanych manifestów K8s, Docker-Compose oraz budowy plików .war do kontenerów Cloud-Native.

## Architektura aplikacji (Model C4)

Aplikacja oparta jest o architekturę wielowarstwową oraz modularnego monolitu. Z biznesowego punktu widzenia stosowany jest paradygmat Domain-Driven Design (w poszczególnych domenach biznesowych i pakietach) oraz implementacja mechanizmów CQRS (Command Query Responsibility Segregation) rozdzielających zapytania (Query) od operacji modyfikujących dane (Command). Część architektury Fineract wykorzystuje `CommandSource` jako wzorzec Audit i Event Sourcing, w którym każda kluczowa zmiana stanu w systemie (np. uruchomienie pożyczki) jest logowana jako polecenie w tabeli operacyjnej `m_portfolio_command_source`.

```plantuml
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml

LAYOUT_WITH_LEGEND()

title Model C4 (Component level) - Apache Fineract

Container(webapp, "Kanały Dostępowe (Web / Mobile)", "Angular / Flutter", "Aplikacje Front-End (np. Mifos X)")
Container(api_gateway, "API Gateway / LB", "Nginx/Kong", "Punkt wejściowy REST API dla kanałów Fineract")

System_Boundary(fineract_core, "Apache Fineract (Core Banking)") {
    Component(security, "fineract-security", "Spring Security, OAuth2", "Uwierzytelnianie, sprawdzanie uprawnień i dzierżawy (Multi-Tenancy)")
    
    Component(command_handler, "fineract-command", "Command Bus", "Przechwytuje akcje zmieniające stan i rejestruje je dla pełnej historii audytowej.")
    
    Component(portfolio_module, "fineract-provider (CRM)", "CRM Domain", "Zarządzanie klientami, pracownikami, powiązaniami.")
    
    Component(loan_module, "fineract-loan\nfineract-progressive-loan", "Lending Domain", "Obsługa cyklu życia kredytów, harmonogramów i spłat.")
    
    Component(savings_module, "fineract-savings", "Deposit Domain", "Zarządzanie kontami oszczędnościowymi i depozytowymi, operacje wypłat, lokaty.")
    
    Component(accounting_module, "fineract-accounting", "General Ledger", "Księga Główna, mapowania transakcji na podwójne wpisy księgowe.")
    
    Component(charge_tax_module, "fineract-charge & tax", "Pricing Domain", "Obrabianie kar, opłat członkowskich, prowizji kredytowych oraz podatków zysków.")
    
    Component(batch_cob, "fineract-cob", "Spring Batch", "Aplikacje wsadowe wykonujące zadania końca dnia, blokady kont i odświeżanie.")

    Rel(api_gateway, security, "Żądania do autoryzacji")
    Rel(security, command_handler, "Modyfikacje do CQRS")
    
    Rel(command_handler, loan_module, "Uruchamia serwisy biznesowe")
    Rel(command_handler, savings_module, "Uruchamia serwisy biznesowe")
    Rel(command_handler, portfolio_module, "Zmienia stan i powiązania Klienta")
    
    Rel(charge_tax_module, loan_module, "Narzuca kwoty do spłaty raty")
    Rel(loan_module, accounting_module, "Wyzwala księgowania zdarzeń kredytowych (np. spłata)")
    Rel(savings_module, accounting_module, "Wyzwala księgowania zdarzeń kontowych (np. odsetki)")
}

SystemDb_Ext(database, "Relacyjna Baza Danych (Tenant DB)", "MySQL / PostgreSQL", "Przechowuje stan dzierżawców oraz system bankowy")
SystemQueue_Ext(event_broker, "Message Broker", "ActiveMQ / Kafka", "Służy do asynchronicznego wymieniania zdarzeń biznesowych (np. w webhookach)")

Rel(webapp, api_gateway, "Wywołania REST API", "JSON/HTTPS")

Rel(loan_module, database, "Odczyt/Zapis (JPA / JDBC)")
Rel(savings_module, database, "Odczyt/Zapis (JPA / JDBC)")
Rel(accounting_module, database, "Odczyt/Zapis (JPA / JDBC)")
Rel(batch_cob, database, "Masowy Odczyt/Zapis (JPA / JDBC)")

Rel(loan_module, event_broker, "Emituje zdarzenia domenowe")
Rel(savings_module, event_broker, "Emituje zdarzenia domenowe")

@enduml
```

## Stos technologiczny

Apache Fineract to nowoczesna, wysoce skalowalna aplikacja napisana w języku Java, która czerpie w pełni z ekosystemu Spring. Główne technologie wykorzystywane w projekcie to:

*   **Język i Framework:** Java (obecnie minimum Java 17), z frameworkiem Spring Boot jako podstawa inwersji sterowania i wstrzykiwania zależności (IoC / DI).
*   **Warstwa Sieciowa API:** Spring Web (REST API, obiekty reprezentowane przy pomocy specyfikacji OpenAPI).
*   **Architektura Danych:** 
    *   **Bazy Danych:** System wielodzierżawny (Multi-Tenant) wykorzystujący relacyjne bazy danych: MySQL, MariaDB oraz PostgreSQL. 
    *   **ORM:** Spring Data JPA oraz EclipseLink jako dostawca rozwiązań JPA. Użycie RowMapperów JDBC dla operacji szybkiego odczytu (CQRS Query side).
    *   **Migracja Danych:** Flyway / Liquibase.
*   **Architektura Wsadowa i Raportowa:** Spring Batch (dla COB - Close Of Business), Pentaho (dla raportowania PDF/XLS).
*   **Architektura Komunikatów i Zdarzeń:** Apache Kafka, Apache ActiveMQ (Schematy Avro).
*   **Zarządzanie Cyklem Życia i Budowa:** Gradle (w tym środowisko wielomodułowe `subprojects`), testy JUnit (Rest Assured). Wdrażanie oparte jest o obrazy Docker, wtyczkę Jib i gotowe skrypty Kubernetes.
*   **Biblioteki Narzędziowe:** Lombok (redukcja szablonów), Swagger / OpenAPI, SpotBugs & CheckStyle do rygorystycznej weryfikacji jakości kodu.
