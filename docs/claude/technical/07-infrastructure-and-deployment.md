# Infrastruktura i wdrożenie

<details>
<summary><strong>Skocz do dowolnego dokumentu</strong></summary>

**[Indeks](../README.md)**

**Biznesowe:** [01 Podsumowanie wykonawcze](../business/01-executive-summary.md) · [02 Przegląd produktu](../business/02-product-overview.md) · [03 Procesy biznesowe](../business/03-business-processes.md) · [04 Słownik domenowy](../business/04-domain-glossary.md) · [05 Reguły biznesowe](../business/05-business-rules.md) · [06 Integracje i interesariusze](../business/06-integrations-and-stakeholders.md) · [07 Ryzyka i luki](../business/07-risks-and-gaps.md)

**Techniczne:** [01 Przegląd architektury](01-architecture-overview.md) · [02 Stos technologiczny](02-tech-stack.md) · [03 Mapa repozytorium](03-repository-map.md) · [04 Model danych](04-data-model.md) · [05 Dokumentacja API](05-api-reference.md) · [06 Przepływy wykonawcze](06-runtime-flows.md) · **07 Infrastruktura i wdrożenie** · [08 Konfiguracja i flagi funkcji](08-configuration-and-feature-flags.md) · [09 Model bezpieczeństwa](09-security-model.md) · [10 Podręcznik operacyjny](10-operational-runbook.md) · [11 Strategia testowania](11-testing-strategy.md) · [12 Dziennik decyzji](12-decision-log.md)

</details>

> Środowiska, budowanie kontenerów, stosy Compose, Kubernetes, CI/CD.

## Obraz kontenera

- Budowany przy użyciu **Google Jib** (brak pliku Dockerfile w repozytorium dla obrazu wykonawczego). Konfiguracja Jib znajduje się w `fineract-provider/build.gradle`.
- Obraz bazowy: `azul/zulu-openjdk-alpine:21`.
- Obraz docelowy: `fineract:latest` (oraz tag z wersją).
- Klasa główna: `org.apache.fineract.ServerApplication`.
- Domyślne flagi JVM wstrzykiwane przez Jib: `-Duser.home=/tmp`, `-Dfile.encoding=UTF-8`, `-Duser.timezone=UTC`, `-Djava.security.egd=file:/dev/./urandom`.
- Dodatkowy classpath: `/app/plugins/*` (pliki JAR typu drop-in).
- Eksponowane porty: **8080** (HTTP) i **8443** (HTTPS).
- Budowanie lokalne: `./gradlew :fineract-provider:jibDockerBuild`.
- Podprojekt Gradle `custom/docker` (`settings.gradle:82`, `custom/docker/build.gradle`) buduje obraz zawierający wszystkie niestandardowe moduły umieszczone w `custom/<firma>/<kategoria>/<moduł>`.

## Docker Compose — wybór stosu

**Nie istnieje jeden kanoniczny plik compose**. Repozytorium zawiera pliki w katalogu głównym dla każdego wspieranego środowiska wykonawczego; należy wybrać ten pasujący do docelowej bazy danych / silnika komunikatów:

| Plik | Baza danych | Broker | Topologia |
| --- | --- | --- | --- |
| `docker-compose.yml` | MariaDB 12.2 | brak | pojedynczy węzeł Fineract, debug 5000 + HTTPS 8443 |
| `docker-compose-mariadb.yml` | MariaDB 12.2 | brak | jak wyżej (jawnie) |
| `docker-compose-mysql.yml` | MySQL | brak | pojedynczy węzeł |
| `docker-compose-postgresql.yml` | PostgreSQL 18.3 | brak | pojedynczy węzeł |
| `docker-compose-postgresql-activemq.yml` | PostgreSQL | ActiveMQ Classic 5.18 | manager + 2 workery |
| `docker-compose-postgresql-kafka.yml` | PostgreSQL | Apache Kafka 4.1 | manager + 2 workery |
| `docker-compose-postgresql-kafka-msk.yml` | PostgreSQL | AWS MSK | manager + 2 workery |
| `docker-compose-postgresql-test-activemq.yml` | PostgreSQL | ActiveMQ | zoptymalizowany pod testy integracyjne |
| `docker-compose-development.yml` | MariaDB | — | profil deweloperski (live reload) |
| `docker-compose-custom.yml` | — | — | używa obrazu zbudowanego niestandardowo z `custom/docker` |
| `docker-compose-community-app.yml` | — | — | dodaje klienta webowego Mifos community |
| `docker-compose-web-app.yml` | — | — | dodaje alternatywnego klienta webowego |

Każdy główny plik compose rozszerza (`extends:`) współdzielone usługi w `config/docker/compose/`:

| Plik | Co definiuje |
| --- | --- |
| `config/docker/compose/fineract.yml` | Kontener Fineract (obraz, healthcheck na 8443). |
| `config/docker/compose/mariadb.yml` | MariaDB 12.2 z healthcheckiem. |
| `config/docker/compose/postgresql.yml` | Postgres 18.3 ze skryptem inicjalizacyjnym i sondą health probe `pg_isready`. |
| `config/docker/compose/activemq.yml` | ActiveMQ Classic 5.18.3. |
| `config/docker/compose/observability.yml` | Prometheus + Grafana + Loki + Tempo. |

Środowisko jest warstwowane poprzez pliki env w `config/docker/env/`:

- `fineract-common.env` — domyślne ustawienia inne niż DB, współdzielone przez każdy stos.
- `fineract.env` — domyślne ustawienia węzła manager.
- `fineract-mariadb.env`, `fineract-postgresql.env`, `fineract-mysql.env` — specyficzne dla danej bazy danych.
- `fineract-manager.env`, `fineract-worker.env` — flagi trybu manager/worker.
- `kafka-client.env` — tuning Kafki.

## Topologia Manager / worker

Ten sam obraz jest uruchamiany z różnymi flagami środowiskowymi:

| Rola | Flagi |
| --- | --- |
| Monolit odczyt+zapis (domyślny) | `FINERACT_MODE_*=true` dla odczytu, zapisu, batch-managera, batch-workera. |
| Manager | `FINERACT_MODE_BATCH_MANAGER_ENABLED=true`, `FINERACT_MODE_BATCH_WORKER_ENABLED=false` i włączony handler zdalnych zadań. |
| Worker | `FINERACT_MODE_BATCH_WORKER_ENABLED=true`, `FINERACT_MODE_BATCH_MANAGER_ENABLED=false`. |
| Węzeł API tylko do odczytu | `FINERACT_MODE_WRITE_ENABLED=false`. |

Stosy Postgres+Kafka i Postgres+ActiveMQ demonstrują podział: manager na `8443`, workery na `8444` i `8445`.

## Kubernetes

Czyste manifesty YAML w katalogu `kubernetes/`. **Nie ma wykresu Helm**.

| Plik | Cel |
| --- | --- |
| `fineract-server-deployment.yml` | `Service` (LoadBalancer, port 8443) + `Deployment` (obraz `apache/fineract:latest`). InitContainer czeka na `fineractmysql:3306`. CPU 200m–1000m, pamięć 1Gi–2Gi, `JAVA_TOOL_OPTIONS=-Xmx1G -XX:MaxMetaspaceSize=256m`. |
| `fineractmysql-deployment.yml` | PV (10Gi) + PVC (5Gi) + bezstanowy `Service` + `Deployment` (MariaDB 12.2). |
| `fineractmysql-configmap.yml` | Konfiguracja inicjalizacji bazy danych. |
| `fineract-mifoscommunity-deployment.yml` | Wariant aplikacji webowej Community. |
| `kubectl-startup.sh`, `kubectl-shutdown.sh` | Skrypty pomocnicze do wdrażania / usuwania manifestów. |

Sondy (probes) kierują do Spring Boot Actuator przez HTTPS na `/fineract-provider/actuator/health/{liveness,readiness}` z `initialDelaySeconds=90/60`.

Dane uwierzytelniające bazy danych są pobierane z Secretu Kubernetes `fineract-tenants-db-secret` (musi być przygotowany poza procesem).

> TODO (wymaga potwierdzenia od eksperta dziedzinowego): manifesty w `kubernetes/` przypinają publiczny obraz Docker Hub `apache/fineract:latest` i pojedynczą instancję MariaDB. Wdrożenia produkcyjne zazwyczaj wymagają zarządzanej bazy danych oraz Ingressa + terminacji TLS — czego nie ma w tym repozytorium.

## CI / CD

Przepływy GitHub Actions w `.github/workflows/`:

| Przepływ | Wyzwalacz |
| --- | --- |
| `build-postgresql.yml`, `build-mariadb.yml`, `build-mysql.yml` | PR / push: budowanie i testowanie względem każdej bazy danych; PostgreSQL podzielony na 5 shardów macierzy (`test-core-1` … `test-core-5`). |
| `build-docker.yml` | Budowanie obrazu Jib i testowanie go na stosach Compose MariaDB i PostgreSQL. |
| `build-cucumber.yml` | Uruchamianie testów BDD Cucumber (`fineract-e2e-tests-runner`). |
| `build-e2e-tests.yml` | Pełny zestaw e2e. |
| `run-integration-test-sequentially-postgresql.yml` | Sekwencyjny wariant testów integracyjnych dla wolniejszych maszyn. |
| `smoke-messaging.yml` | Uruchamianie stosów Kafka i ActiveMQ oraz wykonywanie testów dymnych (smoke tests). |
| `liquibase-only-postgresql.yml` | Uruchomienie w trybie tylko Liquibase (`application-liquibase-only.properties`) — weryfikuje, czy migracje nie kończą się błędem. |
| `verify-api-backward-compatibility.yml` | `swagger-brake` względem połączonego bazowego swagger.json. |
| `verify-liquibase-backward-compatibility.yml` | Wykrywanie zmian wstecznie niekompatybilnych w Liquibase względem bazy. |
| `sonarqube.yml` | Bramka jakości SonarCloud. |
| `pr-title-check.yml`, `pr-one-commit-per-user-check.yml`, `verify-commits.yml` | Konwencje PR. |
| `publish-dockerhub.yml`, `mifos-fineract-client-publish.yml` | Wydanie: wypchnięcie obrazu do Docker Hub, publikacja wygenerowanego klienta w npm. |
| `build-documentation.yml` | Budowanie artefaktów AsciiDoc `fineract-doc`. |
| `stale.yml` | Zarządzanie nieaktywnymi (stale) zgłoszeniami. |

Wszystkie buildy działają na `ubuntu-24.04`, JDK 21 (Zulu). Używana jest pamięć podręczna Develocity build cache (`https://develocity.apache.org`); klucz dostępu przekazywany przez secret `DEVELOCITY_ACCESS_KEY`.

## Sekrety i konfiguracja

- Żadne sekrety nie są commitowane; przykładowe tokeny / hasła w plikach env compose to oczywiste domyślne ustawienia deweloperskie (`mysql`, `fineract`).
- Manifesty Kubernetes pobierają dane uwierzytelniające bazy danych z Secretu `fineract-tenants-db-secret`.
- Hasła baz danych dla poszczególnych najemców (tenants) są szyfrowane w spoczynku w magazynie najemców przy użyciu `fineract.tenant.master-password` + `fineract.tenant.encrytion=AES/CBC/PKCS5Padding` (`application.properties:53-54`).

## Obserwowalność (Observability)

- `management.endpoints.web.exposure.include` domyślnie ustawione na `health,info,prometheus`.
- Liveness: `/fineract-provider/actuator/health/liveness`.
- Readiness: `/fineract-provider/actuator/health/readiness`.
- Prometheus scrape: `/fineract-provider/actuator/prometheus` (włączane przez `FINERACT_MANAGEMENT_PROMETHEUS_ENABLED=true`).
- Eksporter śladów OTLP: włączany przez `FINERACT_MANAGEMENT_OLTP_ENABLED=true` z punktem kończnym `management.otlp.metrics.export.endpoint` skierowanym na kolektor (domyślnie `http://tempo:4318/v1/traces`).
- Metryki CloudWatch: włączane przez `FINERACT_MANAGEMENT_CLOUDWATCH_ENABLED=true`, przestrzeń nazw `fineract`.
- Tag aplikacji: `management.metrics.tags.application=fineract`.
- `config/docker/compose/observability.yml` uruchamia Prometheus 2.47, Grafana 10.2, Loki 2.9, Tempo 2.2 obok Fineract.

## HTTPS

Dołączony `keystore.jks` (`fineract-provider/src/main/resources/keystore.jks`) to deweloperski certyfikat samopodpisany. Wdrożenia produkcyjne muszą go zastąpić (typowy wzorzec: zewnętrzna terminacja TLS na LB / Ingress).

## Tryb tylko Liquibase (Liquibase-only mode)

Ustawienie `spring.profiles.active=liquibase-only` (lub uruchomienie z `application-liquibase-only.properties`) uruchamia aplikację z `spring.main.web-application-type=none`, wykonuje migracje na skonfigurowanych bazach danych i kończy działanie. CI używa tego w `liquibase-only-postgresql.yml`. Przydatne przy wdrożeniach produkcyjnych: uruchomienie jednorazowego zadania migracji przed przełączeniem na nową wersję aplikacji.

---

← Poprzedni: [Przepływy wykonawcze](06-runtime-flows.md) · ↑ [Indeks](../README.md) · Następny: [Konfiguracja i flagi funkcji](08-configuration-and-feature-flags.md) →
