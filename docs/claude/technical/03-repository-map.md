# Mapa repozytorium

<details>
<summary><strong>Przejdź do dowolnego dokumentu</strong></summary>

**[Indeks](../README.md)**

**Biznesowe:** [01 Podsumowanie menedżerskie](../business/01-executive-summary.md) · [02 Przegląd produktu](../business/02-product-overview.md) · [03 Procesy biznesowe](../business/03-business-processes.md) · [04 Słownik domenowy](../business/04-domain-glossary.md) · [05 Reguły biznesowe](../business/05-business-rules.md) · [06 Integracje i interesariusze](../business/06-integrations-and-stakeholders.md) · [07 Ryzyka i luki](../business/07-risks-and-gaps.md)

**Techniczne:** [01 Przegląd architektury](01-architecture-overview.md) · [02 Stos technologiczny](02-tech-stack.md) · **03 Mapa repozytorium** · [04 Model danych](04-data-model.md) · [05 Dokumentacja API](05-api-reference.md) · [06 Przepływy wykonawcze](06-runtime-flows.md) · [07 Infrastruktura i wdrożenie](07-infrastructure-and-deployment.md) · [08 Konfiguracja i flagi funkcji](08-configuration-and-feature-flags.md) · [09 Model bezpieczeństwa](09-security-model.md) · [10 Instrukcja operacyjna (Runbook)](10-operational-runbook.md) · [11 Strategia testowania](11-testing-strategy.md) · [12 Dziennik decyzji](12-decision-log.md)

</details>

> Przewodnik moduł po module po wielomodułowym projekcie Gradle Apache Fineract.

Repozytorium to wielomodułowy projekt Gradle (settings.gradle:49-96) tworzący aplikację bankowości rdzeniowej opartą na Spring Boot. Moduły są pogrupowane na: wdrażalnego dostawcę (provider), moduły domen biznesowych, moduły infrastrukturalne, generowane zestawy SDK klienta, artefakty pakowania oraz środowiska testowe.

## Układ najwyższego poziomu

| Ścieżka | Cel |
| --- | --- |
| `build.gradle`, `settings.gradle`, `gradle.properties`, `static-weaving.gradle` | Główny projekt Gradle, lista modułów, argumenty JVM. |
| `buildSrc/` | Niestandardowe wtyczki Gradle współdzielone przez podprojekty (wtyczka wydań, zależności). |
| `config/` | Konfiguracje narzędzi: Checkstyle, SpotBugs, formater Eclipse, współdzielone usługi docker/compose/*.yml. |
| `docker/` | Statyczne zasoby spakowane w obrazie wykonawczym (np. server.xml). |
| `docker-compose-*.yml` | Jeden plik główny na każde wspierane środowisko uruchomieniowe (MariaDB, MySQL, Postgres, Postgres+Kafka, Postgres+ActiveMQ, MSK, dev, aplikacje webowe/społecznościowe). |
| `kubernetes/` | Zwykłe manifesty YAML dla serwera Fineract + bazy danych MariaDB; skrypty inicjujące. |
| `scripts/` | Skrypty pomocnicze: split-features.sh, split-tests.sh, verify-signed-commits.sh. |
| `.github/workflows/` | 19 przepływów pracy CI (budowanie dla każdej bazy danych, budowanie Dockera, e2e, Cucumber, SonarCloud itp.). |
| `custom/` | Wtykowe moduły niestandardowe wykrywane dynamicznie (settings.gradle:84-95). |
| `docs/` | Niniejszy zestaw dokumentacji. |
| `LICENSE_*`, `NOTICE_*`, `APACHE_LICENSETEXT.md` | Licencja Apache i powiadomienia. |

## Moduły Java

Pełna lista modułów pochodzi z `settings.gradle:49-96`. Lista `fineractJavaProjects` w `build.gradle:26-58` wylicza moduły korzystające ze standardowego łańcucha narzędzi Java (Java 21 — `build.gradle:393-399`).

### Aplikacja wdrażalna

| Moduł | Cel |
| --- | --- |
| `fineract-provider` | Główna aplikacja Spring Boot. Hostuje JAX-RS REST API pod `/api`, konfigurację Spring Security, orkiestrację Liquibase, szkielet zaplanowanych zadań oraz most do wszystkich modułów biznesowych. Klasa główna: `org.apache.fineract.ServerApplication` (`fineract-provider/src/main/java/org/apache/fineract/ServerApplication.java:42-58`). |
| `fineract-war` | Moduł składania pliku WAR; agreguje moduły biznesowe i pakuje dystrybucje binarne/źródłowe. |

### Infrastruktura przekrojowa

| Moduł | Cel |
| --- | --- |
| `fineract-core` | Typy fundamentalne: prymitywy domenowe, encje bazowe, DTO wsadowe, wyliczenie `JobName`, definicje zadań harmonogramu, podstawowe encje administracji organizacją/użytkownikami, narzędzia (util). Używany przez każdy moduł biznesowy. |
| `fineract-security` | Infrastruktura bezpieczeństwa (filtry, polityka haseł, pomocnicy bezpieczeństwa). |
| `fineract-cob` | Silnik przetwarzania zamknięcia dnia (Close-of-Business): `COBBusinessStep`, `COBBusinessStepService`, abstrakcje wykonawcy/taskletu. Konkretne kroki znajdują się w modułach domenowych. |
| `fineract-command` | Command sourcing: trwały magazyn poleceń, powtarzanie, audyt (Liquibase: `fineract-command/src/main/resources/db/changelog/tenant/module/command/module-changelog-master.xml`). |
| `fineract-validation` | Wielokrotnego użytku adnotacje ograniczeń i walidatory JSR-380. |
| `fineract-doc` | Źródła AsciiDoc dla przewodnika użytkownika/operatora projektu; generuje HTML i PDF. |
| `fineract-avro-schemas` | Definicje Apache Avro `.avsc` dla zdarzeń zewnętrznych; generuje zestaw SDK Java konsumowany przez producentów/konsumentów zdarzeń. |

### Moduły domen biznesowych

Większość modułów biznesowych dostarcza dziennik zmian Liquibase w lokalizacji `src/main/resources/db/changelog/tenant/module/<name>/module-changelog-master.xml`, który jest dołączany przez `fineract-provider/src/main/resources/db/changelog/db.changelog-master.xml:35-42`.

| Moduł | Cel | Ma Liquibase najemcy? |
| --- | --- | --- |
| `fineract-loan` | Domena kont i produktów pożyczkowych (zaległości, zabezpieczenia, przerwy w naliczaniu odsetek itp.). | tak |
| `fineract-loan-origination` | Usługi przepływu pracy przyznawania (origination). | tak |
| `fineract-progressive-loan` | Model pożyczek progresywnych (cykliczne przeliczanie harmonogramu). | tak |
| `fineract-progressive-loan-embeddable-schedule-generator` | Samodzielna biblioteka generatora harmonogramów; może być osadzona poza Fineract. | nie (biblioteka) |
| `fineract-working-capital-loan` | Specjalizacja pożyczek obrotowych; dostarcza własne zadania COB. | tak |
| `fineract-savings` | Konta oszczędnościowe, lokaty terminowe, depozyty cykliczne; tabele stóp procentowych. | tak |
| `fineract-accounting` | Księga główna, wpisy do dziennika, rozliczenia międzyokresowe, zamknięcia, tworzenie rezerw, bilans próbny. | tak |
| `fineract-investor` | Zewnętrzni właściciele aktywów i wzbogacanie danych inwestorów. | tak |
| `fineract-charge` | Definicje opłat / prowizji / kar współdzielone między produktami. | tak |
| `fineract-rates` | Definicje zmiennych stóp procentowych i ocena okresu stopy. | tak |
| `fineract-tax` | Komponenty i grupy podatku u źródła. | osadzone w schemacie rdzeniowym |
| `fineract-branch` | Domena oddziałów, personelu, kasjerów. | tak |
| `fineract-document` | Zarządzanie dokumentami; abstrahuje magazyny zawartości w systemie plików i S3. | osadzone w schemacie dostawcy |
| `fineract-report` | Definicje raportów Stretchy i Pentaho; metadane zapytań ad-hoc. | osadzone w schemacie dostawcy |
| `fineract-mix` | Raportowanie MIX-Market (taksonomia / mapowania XBRL). | osadzone w schemacie dostawcy |

### Generowane zestawy SDK klienta

| Moduł | Cel |
| --- | --- |
| `fineract-client` | Klient REST wygenerowany przez OpenAPI (wtyczka `org.openapi.generator` w `build.gradle:125`). |
| `fineract-client-feign` | Wariant klienta REST w stylu Feign. |

### Moduły testowe

| Moduł | Cel |
| --- | --- |
| `integration-tests` | Testy integracyjne oparte na JUnit uruchamiane przeciwko działającemu serwerowi Fineract. |
| `twofactor-tests` | Testy integracyjne dla przepływu uwierzytelniania dwuskładnikowego. |
| `oauth2-tests` | Testy integracyjne dla przepływu serwera zasobów OAuth2. |
| `fineract-e2e-tests-core` | Szkielet BDD Cucumber/Gherkin: definicje kroków i współdzielone asercje. |
| `fineract-e2e-tests-runner` | Moduł uruchamiający Cucumber; ładuje funkcje (features) i wiąże je z definicjami kroków. |

### Moduły niestandardowe

`custom/<company>/<category>/<module>` jest wykrywany w czasie konfiguracji przez pętle `each` w `settings.gradle:84-96` i `build.gradle:372-385`. Podprojekt `custom/docker` buduje niestandardowy obraz Dockera. Ta konwencja pozwala organizacjom nakładać własne moduły na bazowy Fineract bez edycji głównego pliku `settings.gradle`.

## Istotne zasoby wewnątrz fineract-provider

| Ścieżka | Cel |
| --- | --- |
| `src/main/resources/application.properties` | Główna konfiguracja (bezpieczeństwo, najemcy, pamięć podręczna, zadania, integracje, magazyn zawartości, obserwowalność). |
| `src/main/resources/application-liquibase-only.properties` | Aktywuje tryb bez interfejsu webowego (`spring.main.web-application-type=none`), który uruchamia migracje i kończy działanie. |
| `src/main/resources/db/changelog/db.changelog-master.xml` | Główny dziennik zmian Liquibase: obsługuje schematy `tenant_store`, a następnie `tenant`, a potem dołącza wszystkie dzienniki zmian modułów. |
| `src/main/resources/db/changelog/tenant-store/` | Schemat tenant-store (rejestr najemców, ciągi połączeń, repliki tylko do odczytu). |
| `src/main/resources/db/changelog/tenant/` | Schemat najemcy; 218 części dziennika zmian w momencie pisania (`tenant/parts/`). |
| `src/main/resources/sql/` | Narzędzia SQL. |
| `src/main/resources/META-INF/` | Konfiguracja mechanizmu Service-loader. |
| `src/main/resources/logback-spring.xml` | Konfiguracja logowania. |
| `src/main/resources/keystore.jks` | Domyślny magazyn kluczy programistycznych dla HTTPS na porcie 8443. **Nie używać w środowisku produkcyjnym.** |
| `src/main/resources/messages*.properties` | Pakiety komunikatów i18n (obecnie domyślny `en` + `de`). |
| `src/main/resources/ESAPI.properties`, `validation.properties` | Konfiguracja OWASP ESAPI i wzorce walidacji. |

## Gdzie znajdują się typowe elementy w fineract-provider

| Obszar | Pakiet |
| --- | --- |
| Uruchamianie Spring Boot | `org.apache.fineract` (`ServerApplication`) oraz `org.apache.fineract.infrastructure.core.boot` (`FineractWebApplicationConfiguration`, `FineractLiquibaseOnlyApplicationConfiguration`). |
| Uruchamianie JAX-RS | `org.apache.fineract.infrastructure.core.jersey.JerseyConfig` (montuje `/api`, rejestruje ziarna `@Path` i `@Provider`). |
| Wielonajemowość (Multi-tenancy) | `org.apache.fineract.infrastructure.core.config` (`JdbcConfig`), `org.apache.fineract.infrastructure.security.filter.TenantAwareBasicAuthenticationFilter`. |
| Konfiguracja bezpieczeństwa | `org.apache.fineract.infrastructure.security.config` (`SecurityConfig`, `AuthorizationServerConfig`). |
| Harmonogramowanie | `org.apache.fineract.infrastructure.jobs` (konfiguracja, usługa, filtr, handler). |
| Powiadomienia i zdarzenia asynchroniczne | `org.apache.fineract.notification.eventandlistener`. |
| Punkty wejścia Portfolio | `org.apache.fineract.portfolio.<sub>.api` (np. `loanaccount.api.LoansApiResource`, `savings.api.SavingsAccountApiResource`, `client.api.ClientApiResource`). |
| Punkty wejścia Księgowości | `org.apache.fineract.accounting.<sub>.api` (np. `journalentry.api.JournalEntriesApiResource`). |
| Punkty wejścia Organizacji | `org.apache.fineract.organisation.<sub>.api`. |
| Administracja użytkownikami | `org.apache.fineract.useradministration.api`. |
| Interfejsy API samoobsługi | `org.apache.fineract.portfolio.self.*` i ścieżki `/v1/self/*`. |


---

← Poprzedni: [Stos technologiczny](02-tech-stack.md) · ↑ [Indeks](../README.md) · Następny: [Model danych](04-data-model.md) →
