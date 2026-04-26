# Przepływy wykonawcze (Runtime Flows)

<details>
<summary><strong>Skocz do dowolnego dokumentu</strong></summary>

**[Indeks](../README.md)**

**Biznesowe:** [01 Podsumowanie menedżerskie](../business/01-executive-summary.md) · [02 Przegląd produktu](../business/02-product-overview.md) · [03 Procesy biznesowe](../business/03-business-processes.md) · [04 Słownik domenowy](../business/04-domain-glossary.md) · [05 Reguły biznesowe](../business/05-business-rules.md) · [06 Integracje i interesariusze](../business/06-integrations-and-stakeholders.md) · [07 Ryzyka i luki](../business/07-risks-and-gaps.md)

**Techniczne:** [01 Przegląd architektury](01-architecture-overview.md) · [02 Stos technologiczny](02-tech-stack.md) · [03 Mapa repozytorium](03-repository-map.md) · [04 Model danych](04-data-model.md) · [05 Referencja API](05-api-reference.md) · **06 Przepływy wykonawcze** · [07 Infrastruktura i wdrożenie](07-infrastructure-and-deployment.md) · [08 Konfiguracja i flagi funkcji](08-configuration-and-feature-flags.md) · [09 Model bezpieczeństwa](09-security-model.md) · [10 Podręcznik operacyjny](10-operational-runbook.md) · [11 Strategia testowania](11-testing-strategy.md) · [12 Dziennik decyzji](12-decision-log.md)

</details>

> Diagramy sekwencji dla najważniejszych ścieżek wykonawczych.

Konwencje: nazwy aktorów odzwierciedlają nazwy klas Java. Szczegóły wewnętrzne zostały uproszczone do poziomu niezbędnego do zrozumienia współpracy między warstwami.

## 1. Uwierzytelnione żądanie REST — podstawowe uwierzytelnianie (basic auth), komenda zapisu

Używane dla dowolnego API zapisu, np. `POST /api/v1/loans/{loanId}/transactions/{commandTransactionId}`.

```plantuml
@startuml
participant Client
participant "TomcatServlet\nfilter chain" as Filter
participant "Tenant filter" as TenantF
participant "Auth filter\n(Basic / OAuth2 / 2FA)" as AuthF
participant "Jersey /api" as Jersey
participant "LoansApiResource" as Resource
participant "CommandWrapper\n(builder)" as Cmd
participant "CommandSourceWritePlatformService" as Source
participant "CommandHandler" as Handler
participant "Domain service\n(LoanWritePlatformService)" as Service
participant "EclipseLink JPA" as JPA
database "Tenant DB" as DB
participant "ExternalEventService" as Events
queue "JMS / Kafka" as Broker

Client -> Filter : HTTPS request\n+ Fineract-Platform-TenantId\n+ Authorization\n+ optional Idempotency-Key
Filter -> TenantF : resolve tenant
TenantF -> TenantF : ThreadLocalContextUtil\n.setTenant(...)
TenantF -> AuthF : continue
AuthF -> AuthF : authenticate user, load roles
AuthF -> Jersey : forward
Jersey -> Resource : invoke @Path method
Resource -> Cmd : build CommandWrapper
Cmd -> Source : logCommandSource (idempotency check)
alt new request
  Source -> Handler : handle(command)
  Handler -> Service : domain operation
  Service -> JPA : persist / update
  JPA -> DB : SQL
  Service -> Events : queue domain event
end
Resource <-- Source : commandProcessingResult
Resource --> Jersey : JSON
Jersey --> Client : 200 OK
... po zatwierdzeniu (commit) ...
Events -> Broker : publish (Avro)
@enduml
```

Kluczowe punkty:

- Nagłówek najemcy (tenant) jest obowiązkowy; żądanie zostanie odrzucone, jeśli nie można go rozpoznać (`TenantAwareBasicAuthenticationFilter`).
- Maker-checker: komenda zapisu może zostać **utrwalona jako oczekująca (pending)**, jeśli rola wywołująca ma tylko uprawnienia "maker"; inny użytkownik z uprawnieniami "checker" zatwierdza ją następnie przez `/v1/makercheckers`.
- Zdarzenia zewnętrzne są emitowane **po** zatwierdzeniu transakcji (commit) za pośrednictwem `ExternalEventService`, dzięki czemu konsumenci widzą tylko trwały stan. Można to przełączyć przez `fineract.events.external.enabled`.

## 2. API odczytu — `GET /api/v1/loans/{loanId}`

```plantuml
@startuml
participant Client
participant "Filter chain" as Filter
participant "LoansApiResource" as Resource
participant "LoanReadPlatformService" as ReadSvc
participant "JdbcTemplate" as Jdbc
database "Tenant DB" as DB

Client -> Filter : GET /api/v1/loans/123
Filter -> Resource : authenticated, tenant set
Resource -> ReadSvc : retrieveLoanAccountDetails(123)
ReadSvc -> Jdbc : SQL with RowMapper
Jdbc -> DB : SELECT
DB --> Jdbc : rows
Jdbc --> ReadSvc : LoanAccountData
ReadSvc --> Resource : data
Resource --> Client : 200 + JSON
@enduml
```

Usługi odczytu używają bezpośrednio Spring `JdbcTemplate` z ręcznie przygotowanym SQL (CQRS-lite); zapisy przechodzą przez JPA i szynę komend.

## 3. Zakończenie dnia dla pożyczek (LOAN_COB)

Najbardziej złożony regularny przepływ. Domyślne wartości: rozmiar paczki (chunk size) 100, rozmiar partycji 100, 5 wątków roboczych (`application.properties:88-95`).

```plantuml
@startuml
actor Operator
participant "Quartz scheduler" as Quartz
participant "Spring Batch\nLOAN_COB JobLauncher" as Launcher
participant "Manager node\n(batch-manager-enabled=true)" as Manager
queue "Broker (JMS / Kafka /\nSpring Events)" as Broker
participant "Worker node\n(batch-worker-enabled=true)" as Worker
participant "COBBusinessStepService" as Steps
participant "Loan business steps\n(Set delinquency tags,\nAccrual posting,\nApply penalty,\nCheck repayment due, ...)" as BS
database "Tenant DB" as DB
participant "ExternalEventService" as Events

Operator -> Quartz : scheduled trigger\n(or manual /v1/jobs)
Quartz -> Launcher : run LOAN_COB
Launcher -> Manager : start partitioned job
Manager -> DB : claim loans for COB date,\nlock them
Manager -> Broker : enqueue partitions (100 loans each)
Broker -> Worker : deliver partition
Worker -> Steps : process partition (chunk 100)
loop per loan
  Steps -> BS : run configured business steps
  BS -> DB : updates (schedule,\ndelinquency, journal entries)
  BS -> Events : queue events
end
Worker -> DB : release lock,\nadvance loan COB date
Worker --> Manager : partition done
Manager --> Launcher : aggregate, retry stuck partitions
Launcher --> Quartz : status
Events -> Broker : publish (after commit)
@enduml
```

Konfigurowalne kroki znajdują się w `fineract-provider/src/main/java/org/apache/fineract/cob/loan/` i obejmują (przykładowo):

- `CheckLoanRepaymentDueBusinessStep` (sprawdzenie terminu spłaty pożyczki)
- `CheckLoanRepaymentOverdueBusinessStep` (sprawdzenie zaległości w spłacie pożyczki)
- `CheckDueInstallmentsBusinessStep` (sprawdzenie wymagalnych rat)
- `SetLoanDelinquencyTagsBusinessStep` (ustawienie tagów opóźnień)
- `ApplyChargeToOverdueLoansBusinessStep` (naliczenie opłat dla pożyczek z zaległościami)
- `AccrualActivityPostingBusinessStep` (księgowanie naliczeń)
- `AddPeriodicAccrualEntriesBusinessStep` (dodawanie okresowych wpisów naliczeń)
- `CapitalizedIncomeAmortizationBusinessStep` (amortyzacja skapitalizowanego dochodu)
- `BuyDownFeeAmortizationBusinessStep` (amortyzacja opłaty buy-down)
- `LoanInterestRecalculationCOBBusinessStep` (ponowne przeliczenie odsetek pożyczki COB)
- `UpdateLoanArrearsAgingBusinessStep` (aktualizacja starzenia się zaległości)

Najemcy włączają / ustalają kolejność kroków poprzez `m_batch_business_step_configuration` (`db/changelog/tenant/parts/0022_add_batch_business_step_configuration_table.xml`). Punkty końcowe nadrabiania zaległości pod adresem `/v1/internal/cob` pozwalają na ponowne uruchomienie COB dla dni, w których najemca ma luki.

## 4. Zatwierdzenie Maker-checker (na cztery oczy)

```plantuml
@startuml
actor Maker
actor Checker
participant "LoansApiResource" as ApiM
participant "CommandSourceWritePlatformService" as Source
database "f_command_source" as Store
participant "MakerCheckersApiResource" as ApiC
participant "CommandHandler" as Handler

Maker -> ApiM : POST /v1/loans (body)\nrole = MAKER
ApiM -> Source : log command\n(state = PENDING)
Source -> Store : persist command
Source --> ApiM : 200, "auditId" returned
ApiM --> Maker : pending response

Checker -> ApiC : POST /v1/makercheckers/{auditId}
ApiC -> Source : approve
Source -> Handler : execute persisted command
Handler -> Store : mark APPROVED
Handler --> ApiC : final response
ApiC --> Checker : 200
@enduml
```

Checker może również odrzucić operację (`/v1/makercheckers/{auditId} DELETE`). Magazyn komend przetrwa restarty serwera.

## 5. Logowanie i sesja — podstawowe uwierzytelnianie (basic auth)

```plantuml
@startuml
participant Client
participant "AuthenticationApiResource" as Auth
participant "TenantAwareBasicAuthenticationFilter" as Filter
participant "TenantAwareJpaPlatformUserDetailsService" as UserSvc
database "Tenant DB" as DB

Client -> Auth : POST /v1/authentication\nbasic creds + tenant header
Auth -> Filter : delegated to filter chain
Filter -> UserSvc : loadUserByUsername (per tenant)
UserSvc -> DB : SELECT m_appuser, joins roles + permissions
DB --> UserSvc : user row + permissions
UserSvc --> Filter : UserDetails
Filter -> Filter : verify password (bcrypt)
Filter --> Auth : authenticated
Auth --> Client : 200, base64 token + permissions
@enduml
```

Kolejne żądania mogą ponownie przesyłać ten sam nagłówek `Authorization: Basic` — domyślnie nie ma sesji po stronie serwera.

## 6. Tryb serwera zasobów OAuth2

Gdy `FINERACT_SECURITY_OAUTH_ENABLED=true` (`application.properties:25`), filtr basic-auth jest zastępowany przez serwer zasobów OAuth2 ze Spring Security. Ta sama klasa `SecurityConfig` konfiguruje oba tryby; aktywne ziarna (beans) zależą od właściwości. `AuthorizationServerConfig` konfiguruje **wewnętrzny** serwer autoryzacji używany przez przykładową rejestrację "frontend-client" (`application.properties:38-42`).

> DO ZROBIENIA (wymaga potwierdzenia przez eksperta): czy we wdrożeniach produkcyjnych używany jest dołączony serwer autoryzacji, czy zalecany jest zewnętrzny IdP (Keycloak itp.)? Przykładowa rejestracja sugeruje, że serwer w pamięci jest dostarczany głównie do samodzielnego programowania.

## 7. Publikowanie zdarzeń zewnętrznych

```plantuml
@startuml
participant "Domain service" as Svc
participant "ExternalEventService" as Events
database "f_external_event\n(durable inbox)" as Inbox
participant "AsynchronousExternalEventSender" as Sender
queue "JMS / Kafka" as Broker

Svc -> Events : raise(event, payload)
Events -> Inbox : INSERT (in same tx)
note right of Inbox : Send Asynchronous Events job\nflushes after commit
Sender -> Inbox : SELECT batch
Sender -> Broker : produce
Sender -> Inbox : mark as sent
@enduml
```

Semantyka Outbox: zdarzenia są zapisywane **wewnątrz** tej samej transakcji DB co zmiana domenowa, a następnie zadanie Quartz (`SEND_ASYNCHRONOUS_EVENTS` w `JobName`) przesyła je do brokera. Pozwala to uniknąć problemu podwójnego zapisu (dual-write).

## 8. Routing żądań wielonajemcowych (multi-tenant)

```plantuml
@startuml
participant Caller
participant "TenantModuleRootFilter" as Resolver
participant "ThreadLocalContextUtil" as Ctx
participant "RoutingDataSource" as Routing
database "Tenant store DB" as TS
database "Tenant A DB" as A
database "Tenant B DB" as B

Caller -> Resolver : header Fineract-Platform-TenantId = A
Resolver -> TS : SELECT connection for A (cached)
Resolver -> Ctx : set tenant A
Resolver --> Caller : continue to filters
Caller -> Routing : JDBC operation
Routing -> Ctx : determineCurrentLookupKey()
Ctx --> Routing : "A"
Routing -> A : real query
@enduml
```

`AbstractRoutingDataSource` zamienia bazowe źródło danych (`DataSource`) dla każdego żądania w oparciu o identyfikator najemcy w `ThreadLocalContextUtil`.

## 9. Przepływ importu masowego

```plantuml
@startuml
actor User
participant "ImportsApiResource" as Api
participant "BulkImportEventService" as Bulk
database "Tenant DB" as DB
participant "Async worker thread" as Worker

User -> Api : POST /v1/imports\n(Excel/CSV file)
Api -> DB : create m_import_document row (status=IN_PROGRESS)
Api -> Bulk : submit
Bulk -> Worker : dispatch
Worker -> DB : parse + create entities
Worker -> DB : update status (COMPLETED/FAILED)
User -> Api : GET /v1/imports/{id}\npoll status
@enduml
```

## 10. Rozsyłanie powiadomień (fan-out)

```plantuml
@startuml
participant "Domain service" as Svc
participant "NotificationGenerator" as Gen
database "notification_generator\n+ topic_subscriber" as DB
participant "Spring events listener\n@Profile activeMqEnabled" as JmsL
queue ActiveMQ
participant "In-memory listener\n@Profile !activeMqEnabled" as MemL

Svc -> Gen : raiseNotification(...)
Gen -> DB : record notification
alt activeMqEnabled
  Gen -> ActiveMQ : send
  ActiveMQ -> JmsL : deliver
  JmsL -> DB : mark delivered, fan out to subscribers
else default
  Gen -> MemL : Spring event
  MemL -> DB : fan out
end
@enduml
```

Jest to konfigurowane w `notification/eventandlistener` i przełączane przez profil Spring `activeMqEnabled`.

---

Informacje na temat komponentów i pakietów wymienionych powyżej można znaleźć w `01-architecture-overview.md` i `03-repository-map.md`.


---

← Poprzedni: [Referencja API](05-api-reference.md) · ↑ [Indeks](../README.md) · Następny: [Infrastruktura i wdrożenie](07-infrastructure-and-deployment.md) →
