# Konfiguracja i flagi funkcji (Feature Flags)

<details>
<summary><strong>Skocz do dowolnego dokumentu</strong></summary>

**[Indeks](../README.md)**

**Biznesowe:** [01 Podsumowanie wykonawcze](../business/01-executive-summary.md) · [02 Przegląd produktu](../business/02-product-overview.md) · [03 Procesy biznesowe](../business/03-business-processes.md) · [04 Słownik domenowy](../business/04-domain-glossary.md) · [05 Reguły biznesowe](../business/05-business-rules.md) · [06 Integracje i interesariusze](../business/06-integrations-and-stakeholders.md) · [07 Ryzyka i luki](../business/07-risks-and-gaps.md)

**Techniczne:** [01 Przegląd architektury](01-architecture-overview.md) · [02 Stos technologiczny](02-tech-stack.md) · [03 Mapa repozytorium](03-repository-map.md) · [04 Model danych](04-data-model.md) · [05 Dokumentacja API](05-api-reference.md) · [06 Przepływy uruchomieniowe](06-runtime-flows.md) · [07 Infrastruktura i wdrożenie](07-infrastructure-and-deployment.md) · **08 Konfiguracja i flagi funkcji** · [09 Model bezpieczeństwa](09-security-model.md) · [10 Podręcznik operacyjny](10-operational-runbook.md) · [11 Strategia testowania](11-testing-strategy.md) · [12 Dziennik decyzji](12-decision-log.md)

</details>

> Skatalogowane na podstawie `fineract-provider/src/main/resources/application.properties`. Każdy klucz jest w formacie `${ENV_VAR:default}`; zmienna środowiskowa pozwala operatorom na nadpisanie ustawień bez modyfikacji pliku.

Ten dokument jest pogrupowany według domen, a nie alfabetycznie, aby operatorzy mogli łatwo znaleźć konkretny temat. Dokumentacja **nie jest wyczerpująca** — pełny plik ma około 1500 linii; tutaj wymieniono najważniejsze przełączniki. Dla każdej grupy podano zakres linii w pliku źródłowym.

## Serwer (`application.properties:381-399`)

| Klucz | Środowisko (Env) | Domyślnie | Uwagi |
| --- | --- | --- | --- |
| `server.port` | `FINERACT_SERVER_PORT` | `8443` | Port HTTPS. |
| `server.servlet.context-path` | `FINERACT_SERVER_SERVLET_CONTEXT_PATH` | `/fineract-provider` | Wszystkie ścieżki są względne względem tego kontekstu. |
| `server.compression.enabled` | `FINERACT_SERVER_COMPRESSION_ENABLED` | `true` | Kompresja gzip odpowiedzi. |
| `server.ssl.enabled` | `FINERACT_SERVER_SSL_ENABLED` | `true` | Dołączony magazyn kluczy (keystore) w `classpath:keystore.jks` (tylko dla celów deweloperskich). |
| `server.ssl.key-store-password` | `FINERACT_SERVER_SSL_KEY_STORE_PASSWORD` | `openmf` | Należy zmienić przed wdrożeniem produkcyjnym. |
| `server.shutdown` | — | `graceful` | Tomcat kończy realizowane żądania przed wyłączeniem. |
| `spring.lifecycle.timeout-per-shutdown-phase` | `FINERACT_TIMEOUT_PER_SHUTDOWN` | `30s` | |
| `server.tomcat.max-connections`, `server.tomcat.accept-count` | zmienne środ. | 8192 / 100 | Tuning połączeń. |

## Tożsamość / wielonajemność (multi-tenancy) (`application.properties:22-65`)

| Klucz | Środowisko (Env) | Domyślnie |
| --- | --- | --- |
| `fineract.node-id` | `FINERACT_NODE_ID` | `1` |
| `fineract.tenant.host` | `FINERACT_DEFAULT_TENANTDB_HOSTNAME` | `localhost` |
| `fineract.tenant.port` | `FINERACT_DEFAULT_TENANTDB_PORT` | `3306` |
| `fineract.tenant.username` | `FINERACT_DEFAULT_TENANTDB_UID` | `root` |
| `fineract.tenant.password` | `FINERACT_DEFAULT_TENANTDB_PWD` | `mysql` |
| `fineract.tenant.identifier` | `FINERACT_DEFAULT_TENANTDB_IDENTIFIER` | `default` |
| `fineract.tenant.master-password` | `FINERACT_DEFAULT_TENANTDB_MASTER_PASSWORD` | `fineract` |
| `fineract.tenant.encrytion` (sic) | `FINERACT_DEFAULT_TENANTDB_ENCRYPTION` | `AES/CBC/PKCS5Padding` |
| `fineract.tenant.read-only-host`, `-port`, `-username`, `-password`, `-name` | zmienne środ. | puste | Opcjonalna replika tylko do odczytu na najemcę. |
| `fineract.tenant.config.min-pool-size`, `max-pool-size` | zmienne środ. | `-1` | HikariCP na najemcę. |
| `fineract.tenant.config.rounding-mode` | `FINERACT_CONFIG_ROUNDING_MODE` | `6` (HALF_EVEN — zaokrąglanie bankierskie). |

## Bezpieczeństwo (`application.properties:24-42`)

| Klucz | Środowisko (Env) | Domyślnie |
| --- | --- | --- |
| `fineract.security.basicauth.enabled` | `FINERACT_SECURITY_BASICAUTH_ENABLED` | `true` |
| `fineract.security.oauth2.enabled` | `FINERACT_SECURITY_OAUTH_ENABLED` | `false` |
| `fineract.security.2fa.enabled` | `FINERACT_SECURITY_2FA_ENABLED` | `false` |
| `fineract.security.hsts.enabled` | `FINERACT_SECURITY_HSTS_ENABLED` | `false` |
| `fineract.security.cors.*` | zmienne środ. | dozwolone `*` | Ustawienia CORS. |
| `fineract.security.oauth2.client.registrations.frontend-client.*` | zmienne środ. | demo `frontend-client` | Przykładowy klient OAuth2 (deweloperski). |

## Flagi trybu pracy (`application.properties:67-70`)

| Klucz | Środowisko (Env) | Domyślnie |
| --- | --- | --- |
| `fineract.mode.read-enabled` | `FINERACT_MODE_READ_ENABLED` | `true` |
| `fineract.mode.write-enabled` | `FINERACT_MODE_WRITE_ENABLED` | `true` |
| `fineract.mode.batch-worker-enabled` | `FINERACT_MODE_BATCH_WORKER_ENABLED` | `true` |
| `fineract.mode.batch-manager-enabled` | `FINERACT_MODE_BATCH_MANAGER_ENABLED` | `true` |

## Przełączniki modułów (`application.properties:217-219`)

| Klucz | Środowisko (Env) | Domyślnie |
| --- | --- | --- |
| `fineract.module.self-service.enabled` | `FINERACT_MODULE_SELF_SERVICE_ENABLED` | `false` |
| `fineract.module.investor.enabled` | `FINERACT_MODULE_INVESTOR_ENABLED` | `true` |
| `fineract.module.loan-origination.enabled` | `FINERACT_MODULE_LOAN_ORIGINATION_ENABLED` | `true` |

## Zadania / przetwarzanie wsadowe (`application.properties:79-95`)

| Klucz | Środowisko (Env) | Domyślnie |
| --- | --- | --- |
| `fineract.job.stuck-retry-threshold` | `FINERACT_JOB_STUCK_RETRY_THRESHOLD` | `5` |
| `fineract.job.loan-cob-enabled` | `FINERACT_JOB_LOAN_COB_ENABLED` | `true` |
| `fineract.job.journal-entry-aggregation.enabled` | `FINERACT_JOB_JOURNAL_ENTRY_AGGREGATION_ENABLED` | `true` |
| `fineract.job.journal-entry-aggregation.exclude-recent-N-days` | `FINERACT_JOB_JOURNAL_ENTRY_AGGREGATION_EXCLUDE_RECENT_N_DAYS` | `1` |
| `fineract.job.journal-entry-aggregation.chunk-size` | `FINERACT_JOB_JOURNAL_ENTRY_AGGREGATION_CHUNK_SIZE` | `2000` |
| `fineract.partitioned-job.partitioned-job-properties[0].*` (LOAN_COB) | zmienne środ. (`LOAN_COB_*`) | chunk 100, partycja 100, wątki 5/5/kolejka 20, powtórzenia 5, odczyt co 500 ms |

## Broker zadań zdalnych / zdarzeń zewnętrznych (`application.properties:97-152`)

Trzy typy transportu — Spring Events (domyślny), JMS (ActiveMQ), Kafka:

| Klucz | Środowisko (Env) | Domyślnie |
| --- | --- | --- |
| `fineract.remote-job-message-handler.spring-events.enabled` | `…SPRING_EVENTS_ENABLED` | `true` |
| `fineract.remote-job-message-handler.jms.enabled` | `…JMS_ENABLED` | `false` |
| `fineract.remote-job-message-handler.jms.broker-url` | `…JMS_BROKER_URL` | `tcp://127.0.0.1:61616` |
| `fineract.remote-job-message-handler.jms.request-queue-name` | `…JMS_QUEUE_NAME` | `JMS-request-queue` |
| `fineract.remote-job-message-handler.kafka.enabled` | `…KAFKA_ENABLED` | `false` |
| `fineract.remote-job-message-handler.kafka.bootstrap-servers` | `…KAFKA_BOOTSTRAP_SERVERS` | `localhost:9092` |
| `fineract.remote-job-message-handler.kafka.topic.name` | `…KAFKA_TOPIC_NAME` | `job-topic` |
| `fineract.remote-job-message-handler.kafka.topic.partitions` / `replicas` | zmienne środ. | 10 / 1 |
| `fineract.events.external.enabled` | `FINERACT_EXTERNAL_EVENTS_ENABLED` | `false` |
| `fineract.events.external.producer.jms.enabled` | zmienna środ. | `false` |
| `fineract.events.external.producer.kafka.enabled` | `FINERACT_EXTERNAL_EVENTS_KAFKA_ENABLED` | `false` |
| `fineract.events.external.producer.kafka.topic.name` | zmienna środ. | `external-events` |
| `fineract.events.external.producer.kafka.topic.partitions` / `replicas` | zmienne środ. | 10 / 1 |

Obaj producenci/konsumenci/admin Kafka wspierają dowolne dodatkowe parametry poprzez pole `extra-properties` z użyciem separatorów `|` i `=`.

## Procesory transakcji pożyczkowych (`application.properties:165-179`)

Każda strategia może być przełączana. Są to wtykalne strategie alokacji spłat.

| Klucz | Środowisko (Env) | Domyślnie |
| --- | --- | --- |
| `fineract.loan.transactionprocessor.creocore.enabled` | `…CREOCORE_ENABLED` | `true` |
| `fineract.loan.transactionprocessor.early-repayment.enabled` | zmienna środ. | `true` |
| `fineract.loan.transactionprocessor.mifos-standard.enabled` | zmienna środ. | `true` |
| `fineract.loan.transactionprocessor.heavensfamily.enabled` | zmienna środ. | `true` |
| `fineract.loan.transactionprocessor.interest-principal-penalties-fees.enabled` | zmienna środ. | `true` |
| `fineract.loan.transactionprocessor.principal-interest-penalties-fees.enabled` | zmienna środ. | `true` |
| `fineract.loan.transactionprocessor.rbi-india.enabled` | zmienna środ. | `true` |
| `fineract.loan.transactionprocessor.advanced-payment-strategy.enabled` | `…ADVANCED_PAYMENT_STRATEGY_ENABLED` | `true` |
| `fineract.loan.transactionprocessor.due-penalty-fee-interest-principal-in-advance-principal-penalty-fee-interest.enabled` | zmienna środ. | `true` |
| `fineract.loan.transactionprocessor.due-penalty-interest-principal-fee-in-advance-penalty-interest-principal-fee.enabled` | zmienna środ. | `true` |
| `fineract.loan.transactionprocessor.error-not-found-fail` | `…ERROR_NOT_FOUND_FAIL` | `true` |
| `fineract.loan.status-change-history-statuses` | `FINERACT_LOAN_STATUS_CHANGE_HISTORY_STATUSES` | `NONE` (lub `ALL` / lista po przecinku). |

## Magazyn treści (Content store) (`application.properties:181-194`)

| Klucz | Środowisko (Env) | Domyślnie |
| --- | --- | --- |
| `fineract.content.regex-whitelist-enabled` | `FINERACT_CONTENT_REGEX_WHITELIST_ENABLED` | `true` |
| `fineract.content.regex-whitelist` | zmienna środ. | `*.pdf,*.doc,*.docx,*.xls,*.xlsx,*.jpg,*.jpeg,*.png` |
| `fineract.content.mime-whitelist-enabled` | zmienna środ. | `true` |
| `fineract.content.mime-whitelist` | zmienna środ. | PDF, MS Office, JPEG, PNG. |
| `fineract.content.filesystem.enabled` | `FINERACT_CONTENT_FILESYSTEM_ENABLED` | `true` |
| `fineract.content.filesystem.rootFolder` | `FINERACT_CONTENT_FILESYSTEM_ROOT_FOLDER` | `${user.home}/.fineract` |
| `fineract.content.s3.enabled` | `FINERACT_CONTENT_S3_ENABLED` | `false` |
| `fineract.content.s3.{bucketName,accessKey,secretKey,region,endpoint,path-style-addressing-enabled}` | zmienne środ. | puste / `false` |

## Raportowanie (`application.properties:202-203`)

| Klucz | Środowisko (Env) | Domyślnie |
| --- | --- | --- |
| `fineract.report.export.s3.enabled` | `FINERACT_REPORT_EXPORT_S3_ENABLED` | `false` |
| `fineract.report.export.s3.bucket` | `FINERACT_REPORT_EXPORT_S3_BUCKET_NAME` | puste |

## Ochrona przed wstrzykiwaniem SQL (`application.properties:228-319`)

Blok `fineract.sql-validation.*` definiuje wzorce regex oraz **profile** (`main`, `adhoc`, `column`) używane do czyszczenia zapytań SQL przekazywanych przez użytkowników (raporty Stretchy, API zapytań ad-hoc, datatables).

Wzorce obejmują `inject-blind`, `detect-entry-point`, `inject-timing`, `detect-backend`, `detect-column`, `detect-out-of-bands`, `inject-stacked-query`, `inject-comment`. Każdy profil składa się z uporządkowanego podzbioru wzorców. Wszystkie profile są domyślnie włączone.

## Idempotentność, korelacja, śledzenie IP (`application.properties:74-80,163`)

| Klucz | Środowisko (Env) | Domyślnie |
| --- | --- | --- |
| `fineract.idempotency-key-header-name` | `FINERACT_IDEMPOTENCY_KEY_HEADER_NAME` | `Idempotency-Key` |
| `fineract.correlation.enabled` | `FINERACT_LOGGING_HTTP_CORRELATION_ID_ENABLED` | `false` |
| `fineract.correlation.header-name` | `FINERACT_LOGGING_HTTP_CORRELATION_ID_HEADER_NAME` | `X-Correlation-ID` |
| `fineract.ip-tracking.enabled` | `FINERACT_CLIENT_IP_TRACKING_ENABLED` | `false` |
| `fineract.api.body-item-size-limit.inline-loan-cob` | `FINERACT_API_REQUEST_BODY_SIZE_LIMIT_INLINE_COB` | `1000` |
| `fineract.query.in-clause-parameter-size-limit` | `FINERACT_QUERY_PARAMETER_SIZE` | `1000` |

## Logowanie (`application.properties:209,328-330`)

| Klucz | Środowisko (Env) | Domyślnie |
| --- | --- | --- |
| `fineract.logging.json.enabled` | `FINERACT_LOGGING_JSON_ENABLED` | `false` (przełącza na enkoder JSON dla ELK / Loki). |
| `logging.pattern.console` | `CONSOLE_LOG_PATTERN` | Zawiera ID korelacji oraz ID najemcy. |
| `logging.pattern.level` | — | Dodaje `traceId` / `spanId` dla celów śledzenia. |
| `fineract.jpa.statementLoggingEnabled` | `FINERACT_STATEMENT_LOGGING_ENABLED` | `false` (szczegółowe logowanie SQL przez JPA). |

## Próbkowanie (`application.properties:211-214`)

| Klucz | Środowisko (Env) | Domyślnie |
| --- | --- | --- |
| `fineract.sampling.enabled` | `FINERACT_SAMPLING_ENABLED` | `false` |
| `fineract.sampling.samplingRate` | `FINERACT_SAMPLING_RATE` | `1000` |
| `fineract.sampling.sampledClasses` | zmienna środ. | puste |
| `fineract.sampling.resetPeriodSec` | zmienna środ. | `60` |

## Pamięć podręczna (Cache) (`application.properties:321-326`)

Domyślny szablon Ehcache: TTL 1 minuta, maks. 1000 wpisów. Niestandardowy szablon `userTFAccessToken` (tokeny 2FA): TTL 2 godziny, maks. 10000. Dodatkowe szablony mogą być dodawane przez `fineract.cache.custom-templates.<name>.{ttl,maximum-entries}`.

## AWS (`application.properties:362-378`)

| Klucz | Środowisko (Env) | Domyślnie |
| --- | --- | --- |
| `spring.cloud.aws.cloudwatch.enabled` | `FINERACT_MANAGEMENT_CLOUDWATCH_ENABLED` | `false` |
| `spring.cloud.aws.region.static` | `FINERACT_AWS_REGION_STATIC` | `us-east-1` |
| `spring.cloud.aws.credentials.{access-key,secret-key,profile.name,profile.path,instance-profile}` | zmienne środ. | puste / `false` |
| `spring.cloud.aws.endpoint` | `FINERACT_AWS_ENDPOINT` | puste (np. dla LocalStack). |
| `spring.autoconfigure.exclude` | `FINERACT_AUTOCONFIGURE_EXCLUDE` | domyślnie wyklucza autokonfigurację AWS. |

## Obserwowalność (Observability) (`application.properties:332-367`)

| Klucz | Środowisko (Env) | Domyślnie |
| --- | --- | --- |
| `management.endpoints.web.exposure.include` | `FINERACT_MANAGEMENT_ENDPOINT_WEB_EXPOSURE_INCLUDE` | `health,info,prometheus` |
| `management.health.{livenessState,readinessState}.enabled` | — | `true` |
| `management.health.jms.enabled` | `FINERACT_MANAGEMENT_HEALTH_JMS_ENABLED` | `false` |
| `management.tracing.enabled` | `FINERACT_MANAGEMENT_TRACIING_ENABLED` (sic) | `false` |
| `management.otlp.metrics.export.enabled` / `url` | `FINERACT_MANAGEMENT_OLTP_*` | `false` / `http://tempo:4318/v1/traces` |
| `management.prometheus.metrics.export.enabled` | `FINERACT_MANAGEMENT_PROMETHEUS_ENABLED` | `false` |
| `management.metrics.export.cloudwatch.{enabled,namespace,step}` | zmienne środ. | `false` / `fineract` / `1m` |
| `management.metrics.tags.application` | zmienna środ. | `fineract` |

## Globalna konfiguracja bazy danych (tabela `c_configuration`)

Oprócz plikowej konfiguracji `application.properties`, flagi funkcji zmienialne w czasie rzeczywistym znajdują się w tabeli `c_configuration` (początkowe rekordy pochodzą z `db/changelog/tenant/parts/0002_initial_data.xml`) i są zarządzane przez `/v1/configurations` oraz `/v1/configurations/name/{configName}`. Przykłady widoczne w schemacie i testach obejmują:

- `enable-business-date`, `enable-automatic-cob-date-adjustment` — logika daty biznesowej.
- `feature-cob-bypass-user`, `feature-cob-bypass-user-list` — lista użytkowników pomijających COB.
- `is-savings-account-transaction-reversal-enabled`.
- `enable-loan-status-change-history`.
- `enable-payment-hub-integration`.
- … oraz wpisy zdefiniowane przez najemcę.

> TODO (wymaga potwierdzenia eksperta merytorycznego): opublikować ostateczną listę kluczy `c_configuration` po uruchomieniu `SELECT name FROM c_configuration` na aktualnej bazie danych najemcy.

## Profil: `liquibase-only`

`application-liquibase-only.properties` ustawia `spring.main.web-application-type=none`. Służy do uruchomienia migracji i zakończenia działania:

```
./gradlew :fineract-provider:bootRun --args="--spring.profiles.active=liquibase-only"
```

## Profil: `activeMqEnabled`

Przełącza słuchaczy powiadomień między wewnątrzpamieciowymi zdarzeniami Springa a ActiveMQ JMS. Aktywowany automatycznie, gdy zmienne środowiskowe JMS są wypełnione.

## Właściciel konfiguracji *(wnioskowane)*

| Temat | Prawdopodobny właściciel |
| --- | --- |
| Dane dostępowe do bazy najemcy | Zespół platformy / DBA. |
| Rejestracje klientów OAuth2 | Zespół ds. tożsamości / bezpieczeństwa. |
| Strategie procesora transakcji pożyczkowych | Zespół ds. pożyczek / produktu. |
| Producent zdarzeń zewnętrznych (tematy Kafka, partycje) | Zespół ds. integracji / platformy danych. |
| Magazyn treści (S3 bucket / FS root) | Zespół platformy. |
| Flagi obserwowalności (`FINERACT_MANAGEMENT_*`) | Zespół SRE / obserwowalności. |


---

← Poprzedni: [Infrastruktura i wdrożenie](07-infrastructure-and-deployment.md) · ↑ [Indeks](../README.md) · Następny: [Model bezpieczeństwa](09-security-model.md) →
