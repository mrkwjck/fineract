# Instrukcja operacyjna

<details>
<summary><strong>Skocz do dowolnego dokumentu</strong></summary>

**[Indeks](../README.md)**

**Biznesowe:** [01 Podsumowanie wykonawcze](../business/01-executive-summary.md) · [02 Przegląd produktu](../business/02-product-overview.md) · [03 Procesy biznesowe](../business/03-business-processes.md) · [04 Słownik domenowy](../business/04-domain-glossary.md) · [05 Reguły biznesowe](../business/05-business-rules.md) · [06 Integracje i interesariusze](../business/06-integrations-and-stakeholders.md) · [07 Ryzyka i luki](../business/07-risks-and-gaps.md)

**Techniczne:** [01 Przegląd architektury](01-architecture-overview.md) · [02 Stos technologiczny](02-tech-stack.md) · [03 Mapa repozytorium](03-repository-map.md) · [04 Model danych](04-data-model.md) · [05 Dokumentacja API](05-api-reference.md) · [06 Przepływy wykonawcze](06-runtime-flows.md) · [07 Infrastruktura i wdrożenie](07-infrastructure-and-deployment.md) · [08 Konfiguracja i flagi funkcji](08-configuration-and-feature-flags.md) · [09 Model bezpieczeństwa](09-security-model.md) · **10 Instrukcja operacyjna** · [11 Strategia testowania](11-testing-strategy.md) · [12 Rejestr decyzji](12-decision-log.md)

</details>

> Niezbędnik operacyjny (Day-2): uruchamianie lokalne, monitorowanie na produkcji, typowe incydenty.

## Uruchamianie lokalne

### Wymagania wstępne

- JDK 21 (`build.gradle:393-399`).
- Docker (dla dołączonej bazy danych lub pełnych stosów Compose).
- Kopia repozytorium w `/path/to/fineract`.

### Opcja A — Docker Compose (najszybsza)

```
docker compose -f docker-compose.yml up        # MariaDB + Fineract
docker compose -f docker-compose-postgresql.yml up
docker compose -f docker-compose-postgresql-kafka.yml up   # manager + 2 workery + Kafka
```

Gdy system będzie gotowy:

```
curl -k https://localhost:8443/fineract-provider/actuator/health
```

Domyślny najemca: `default`. Domyślny użytkownik: `mifos` / hasło `password` *(wnioskowane, zweryfikuj z danymi początkowymi najemcy w `0002_initial_data.xml`)*.

### Opcja B — Uruchomienie ze źródeł

```
./gradlew :fineract-provider:bootRun
```

Uruchamia to system z wbudowanym Tomcatem + skonfigurowaną bazą danych (domyślnie oczekiwana MariaDB na `localhost:3306`, patrz `application.properties:44-47`). Nadpisz za pomocą zmiennych środowiskowych:

```
FINERACT_DEFAULT_TENANTDB_HOSTNAME=localhost \
FINERACT_DEFAULT_TENANTDB_PORT=5432 \
FINERACT_HIKARI_DRIVER_CLASS_NAME=org.postgresql.Driver \
FINERACT_HIKARI_JDBC_URL=jdbc:postgresql://localhost:5432/fineract_tenants \
./gradlew :fineract-provider:bootRun
```

### Opcja C — Tylko Liquibase

Uruchom migracje i zakończ:

```
./gradlew :fineract-provider:bootRun --args="--spring.profiles.active=liquibase-only"
```

Używaj tego w CI lub jako krok przedwdrożeniowy na produkcji.

### Test dymny (Smoke test)

```
curl -k -u mifos:password \
  -H "Fineract-Platform-TenantId: default" \
  https://localhost:8443/fineract-provider/api/v1/offices
```

## Stan zdrowia i sondy

| URL | Cel |
| --- | --- |
| `/fineract-provider/actuator/health` | Zbiorczy stan zdrowia. |
| `/fineract-provider/actuator/health/liveness` | Używane przez sondę liveness k8s (`kubernetes/fineract-server-deployment.yml:73`). |
| `/fineract-provider/actuator/health/readiness` | Używane przez sondę readiness k8s. |
| `/fineract-provider/actuator/info` | Informacje o kompilacji / git (obsługiwane przez `gorylenko.gradle-git-properties`). |
| `/fineract-provider/actuator/prometheus` | Metryki (jeśli włączone). |

## Logi

- Domyślny format logów: `application.properties:329` — zawiera identyfikator korelacji i identyfikator najemcy.
- Dla logów JSON (np. wysyłanie do ELK / Loki): ustaw `FINERACT_LOGGING_JSON_ENABLED=true`.
- Do debugowania SQL: `FINERACT_STATEMENT_LOGGING_ENABLED=true` (bardzo szczegółowe; nie zostawiaj włączone).

## Kopie zapasowe / DR

- **Baza danych magazynu najemców (tenant store)** zawiera rejestr najemców i parametry połączeń. Twórz jej kopie zapasowe z takim samym RPO jak dla najemców.
- **Bazy danych najemców** przechowują dane bankowe. Twórz kopie zapasowe dla każdego najemcy; kolejność przywracania: najpierw magazyn najemców, potem dane najemcy.
- **Dokumenty** — jeśli `fineract.content.s3.enabled=true`, stosuje się cykl życia/replikację S3. Jeśli na lokalnym systemie plików (`${user.home}/.fineract`), zamontuj go na wolumenie z kopią zapasową.

> TODO (wymaga potwierdzenia przez SME): dokładne cele RPO/RTO, okna retencji i konfiguracja PITR.

## Typowe incydenty

### "tenant unknown" (HTTP 400 / 404 przy każdym żądaniu)

- Brak nagłówka `Fineract-Platform-TenantId` lub jest on nierozpoznany.
- Najemca istnieje w magazynie najemców, ale jego baza danych jest nieosiągalna; sprawdź logi `RoutingDataSource` / Hikari i zweryfikuj łączność z `fineract.tenant.host:port` dla danego najemcy.

### Migracja Liquibase kończy się niepowodzeniem

- Każdy zestaw zmian (changeset) Liquibase posiada sumę kontrolną. Jeśli część została edytowana po wdrożeniu, Liquibase odmówi uruchomienia.
- Użyj profilu **liquibase-only**, aby wyświetlić błąd bez uruchamiania serwera WWW.
- Nigdy nie zmieniaj kolejności istniejących zestawów zmian; dopisuj nowe na dole odpowiedniego pliku `module-changelog-master.xml`.

### Zablokowany proces COB pożyczki

Objawy: data biznesowa przestaje się przesuwać; `/v1/internal/cob/oldest-cob-closed` zwraca starą datę.

1. Sprawdź `/v1/internal/cob/is-catch-up-running`.
2. Sprawdź tabele Spring Batch: `BATCH_JOB_EXECUTION` pod kątem utkniętych rekordów `STARTED`.
3. Jeśli partycja zakończyła się niepowodzeniem, zadanie zostanie ponowione do limitu `LOAN_COB_RETRY_LIMIT` (domyślnie 5).
4. Ręczne przywracanie: wywołaj `/v1/internal/cob/catch-up`, aby przejść dalej, lub odblokuj poszczególne pożyczki przez `/v1/loans/{loanId}/place-lock/{lockOwner}`.

### Zadanie Quartz nie uruchamia się

- Zweryfikuj, czy `fineract.mode.batch-manager-enabled=true` na co najmniej jednym węźle.
- Sprawdź `job_run_history` pod kątem statusu ostatniego wykonania.
- Potwierdź, że `c_configuration` ma globalny przełącznik "scheduler" ustawiony na włączony (GET `/v1/scheduler`).

### Zdarzenia zewnętrzne nie docierają

- Czy `fineract.events.external.enabled=true`?
- Czy producent jest włączony (`producer.kafka.enabled` lub `producer.jms.enabled`)?
- Zadanie Quartz `SEND_ASYNCHRONOUS_EVENTS` musi być uruchomione. Przesyła ono zawartość trwałej skrzynki nadawczej (outbox) do brokera.
- Jeśli używasz Kafka: potwierdź `fineract.events.external.producer.kafka.bootstrap-servers` oraz automatyczne tworzenie tematów (`topic.auto-create=true`).

### Wysokie opóźnienie żądań

1. Sprawdź metryki Hikari (`management.endpoints.web.exposure.include=...,metrics`): oczekujące połączenia.
2. Sprawdź obciążenie CPU bazy danych najemcy i powolne zapytania.
3. Sprawdź logi JVM GC (Jib włącza domyślną ergonomię GC).
4. Potwierdź flagi trybu węzła: obciążenie z dużą ilością zapisów na węźle z `WRITE_ENABLED=false` zostanie przekierowane lub zakończy się błędem.

## Zmiany trybu w czasie rzeczywistym

`/v1/instance-mode` przełącza flagi odczytu/zapisu/menedżera/workera bez restartu. Używaj oszczędnie — przełączanie trybu zapisu podczas trwających zapisów może spowodować wyciek częściowo zaaplikowanego stanu.

## Dodawanie najemcy

1. Wstaw wiersz do tabeli `tenants` (baza tenant-store) z identyfikatorem nowego najemcy, nazwą hosta bazy danych, portem itp. Użyj pomocnika szyfrowania (ten sam seed klucza: `fineract.tenant.master-password`).
2. Przy następnym uruchomieniu Liquibase automatycznie uruchomi się dla nowej bazy danych najemcy.
3. Domyślny schemat jest pusty; zasil dane podstawowe (waluty, kody, biura) przez API.

> TODO (wymaga potwierdzenia przez SME): czy istnieje CLI / skrypt do aprowizacji najemców, czy proces opiera się wyłącznie na SQL + restarcie?

## Aktualizacja Fineract

1. Zabezpiecz kopie zapasowe magazynu najemców i wszystkich baz danych najemców.
2. Pobierz nowy obraz.
3. Uruchom `./gradlew :fineract-provider:bootRun --args="--spring.profiles.active=liquibase-only"` (lub to samo w nowym obrazie), aby zastosować migracje.
4. Zaktualizuj repliki aplikacji (strategia `Recreate` w Kubernetes jest domyślna w `fineract-server-deployment.yml:50`).

## Narzędzia do rozwiązywania problemów

| Potrzeba | Punkt końcowy / tabela |
| --- | --- |
| Audyt API | `request_audit_table`, `/v1/audits` |
| Kolejka maker-checker | `/v1/makercheckers` |
| Status zadania | `/v1/jobs`, `job_run_history` |
| Internalia COB | `/v1/internal/cob/*` |
| Czyszczenie pamięci podręcznej | `/v1/caches` |
| Konfiguracja | `/v1/configurations`, `c_configuration` |
| Zdarzenia zewnętrzne | `/v1/externalevents/configuration`, `f_external_event` *(wnioskowana nazwa)* |

## Czego nie robić

- Nie używaj `git rebase -i` do zmiany kolejności zestawów zmian Liquibase po utworzeniu tagu wydania.
- Nie uruchamiaj dwóch węzłów typu manager przeciwko tej samej bazie danych najemcy, chyba że koordynują one pracę przez brokera (`fineract.remote-job-message-handler.kafka.enabled=true` lub JMS).
- Nie wyłączaj `fineract.job.loan-cob-enabled`, gdy naliczane są pożyczki — dzienne naliczanie zaległości, odsetek i starzenie się zaległości zostaną wstrzymane.


---

← Poprzedni: [Model bezpieczeństwa](09-security-model.md) · ↑ [Indeks](../README.md) · Następny: [Strategia testowania](11-testing-strategy.md) →
