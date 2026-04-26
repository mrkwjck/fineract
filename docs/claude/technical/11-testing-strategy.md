# Strategia testowania

<details>
<summary><strong>Przejdź do dowolnego dokumentu</strong></summary>

**[Indeks](../README.md)**

**Biznesowe:** [01 Podsumowanie wykonawcze](../business/01-executive-summary.md) · [02 Przegląd produktu](../business/02-product-overview.md) · [03 Procesy biznesowe](../business/03-business-processes.md) · [04 Słownik domenowy](../business/04-domain-glossary.md) · [05 Reguły biznesowe](../business/05-business-rules.md) · [06 Integracje i interesariusze](../business/06-integrations-and-stakeholders.md) · [07 Ryzyka i luki](../business/07-risks-and-gaps.md)

**Techniczne:** [01 Przegląd architektury](01-architecture-overview.md) · [02 Stos technologiczny](02-tech-stack.md) · [03 Mapa repozytorium](03-repository-map.md) · [04 Model danych](04-data-model.md) · [05 Dokumentacja API](05-api-reference.md) · [06 Przepływy uruchomieniowe](06-runtime-flows.md) · [07 Infrastruktura i wdrożenie](07-infrastructure-and-deployment.md) · [08 Konfiguracja i flagi funkcji](08-configuration-and-feature-flags.md) · [09 Model bezpieczeństwa](09-security-model.md) · [10 Podręcznik operacyjny](10-operational-runbook.md) · **11 Strategia testowania** · [12 Rejestr decyzji](12-decision-log.md)

</details>

> Co jest testowane, gdzie i czego brakuje.

## Warstwowa piramida testów

| Warstwa | Lokalizacja | Narzędzia | Przybliżona liczba |
| --- | --- | --- | --- |
| **Jednostkowe** | `src/test/java/**/*Test.java` w każdym module | JUnit 5 + Mockito + AssertJ | ~517 plików `*Test.java`. |
| **Testy integracyjne** (względem działającego providera) | `integration-tests/src/test` | JUnit 5 + klienci REST (Retrofit) | jeden zestaw na obszar domenowy; uruchamiane w `build-postgresql.yml`, `build-mariadb.yml`, `build-mysql.yml` (rozproszone na 5 zadań macierzy GH-Actions). |
| **Testy przepływów dwuskładnikowych** | `twofactor-tests/src/test` | JUnit 5 | dedykowany moduł, dzięki czemu standardowy zestaw nie wymaga 2FA. |
| **Testy przepływów OAuth2** | `oauth2-tests/src/test` | JUnit 5 | testuje przepływ tokena bearer. |
| **End-to-end Cucumber/BDD** | `fineract-e2e-tests-runner` (runner) + `fineract-e2e-tests-core` (kroki + pomocnicy) | Cucumber 7.20.1, JUnit Platform engine | 99 plików `.feature`. |
| **Smoke** | `.github/workflows/smoke-messaging.yml` | docker compose + curl | obejmuje topologie ActiveMQ i Kafka. |

## Charakterystyka testów modułowych

### Testy jednostkowe

- Używają Mockito + Hibernate-Validator do testów walidacji.
- AssertJ dla płynnych asercji.
- BOM Testcontainers (1.20.4) znajduje się w ścieżce klas; testy jednostkowe o charakterze integracyjnym mogą lokalnie uruchamiać Postgres / MariaDB.
- Warstwy testowe Spring Boot (`@WebMvcTest`, `@DataJpaTest`) są używane oszczędnie — większość testów to poziom POJO.

### Testy integracyjne (`integration-tests`)

- Uruchamiają instancję `fineract-provider` względem rzeczywistej bazy MariaDB / MySQL / PostgreSQL.
- Używają wygenerowanego `fineract-client` (Retrofit) do obsługi API REST.
- Testują przepływy tworzenia dla: produkty → klienci → konta → transakcje.
- Każdy fragment CI (`test-core-1` … `test-core-5`) uruchamia równolegle część zestawu.

### BDD / E2E (`fineract-e2e-tests-*`)

- Funkcjonalności Gherkin w `fineract-e2e-tests-runner/src/test/resources/features/**.feature`.
- Definicje kroków w `fineract-e2e-tests-core` (Spring + Cucumber).
- 99 plików feature według ostatnich obliczeń — bogate pokrycie cykli życia pożyczek i oszczędności, COB, księgowości.
- Uruchamiane przez workflow GH `build-cucumber.yml`.
- Repozytorium zawiera skrypty `scripts/split-features.sh` i `scripts/split-tests.sh` do dzielenia funkcjonalności na grupy dla wykonania rozproszonego.

### Testy modułów dwuskładnikowych i OAuth2

- Izolowane, aby uniknąć przenikania ich konfiguracji do głównego zestawu testów.
- Uruchamiają providera odpowiednio z `FINERACT_SECURITY_2FA_ENABLED=true` / `FINERACT_SECURITY_OAUTH_ENABLED=true`.
- Weryfikują poprawność przepływu, buforowanie tokenów, zachowanie ponowień OTP.

### Testy kompatybilności wstecznej Liquibase

- `verify-liquibase-backward-compatibility.yml` uruchamia migracje względem bazy połączonej, a następnie na bazie zmian z PR — wykrywa przypadkowe edycje historycznych zestawów zmian.

### Testy kompatybilności wstecznej API

- `verify-api-backward-compatibility.yml` wywołuje `swagger-brake` (`build.gradle:129`), aby porównać plik `fineract.json` z PR z gałęzią bazową.

## Co jest dobrze pokryte

- Przepływy end-to-end pożyczek / oszczędności / księgowości (funkcjonalności BDD dotykają każdego modułu).
- Kompatybilność z wieloma bazami danych (CI dla każdego silnika bazy danych).
- Bezpieczeństwo migracji Liquibase.
- Stabilność publicznego API.

## Co nie jest jawnie pokryte (lub jest pokryte słabo)

- **Wydajność / obciążenie.** Brak zestawu JMeter / Gatling. JMH znajduje się w ścieżce klas (`build.gradle:127`), ale jest używany tylko do mikro-benchmarków tam, gdzie to zasadne.
- **Testy bezpieczeństwa.** Brak workflow DAST / SAST w `.github/workflows/`. SonarCloud (`sonarqube.yml`) oznacza punkty zapalne, ale nie jest audytem bezpieczeństwa.
- **Chaos / odporność.** Brak wstrzykiwanych awarii w macierzy CI (np. pad bazy danych, przerwy w działaniu brokera). Resilience4j znajduje się w ścieżce klas (używane w usługach), ale brak jawnych testów.
- **Obciążenie wielu najemców (multi-tenant).** Większość testów używa domyślnego najemcy (`default`); izolacja między najemcami jest pokryta pośrednio, ale nie wyczerpująco.
- **Przepływy samoobsługowe przy wrogich danych wejściowych.** Ścieżki `/v1/self/*` mają pokrycie integracyjne, ale brak testów typu fuzz.
- **Wycofywanie zmian Liquibase (rollbacks).** Migracje są tylko do przodu; brak testów wycofywania.

## Punkty zapalne pokrycia (jakościowo)

| Moduł | Poziom pokrycia |
| --- | --- |
| `fineract-loan` (w tym progresywne, kapitał obrotowy) | Najwyższy — kluczowy dla systemu, najsilniejsze pokrycie BDD. |
| `fineract-savings` | Wysoki. |
| `fineract-accounting` | Średnio-wysoki; złożona agregacja GL posiada testy jednostkowe + integracyjne, ale przypadki brzegowe naliczeń są oznaczone jako TODO w plikach feature *(wnioskowane)*. |
| `fineract-cob` | Średni; testy partycji są na poziomie integracyjnym. |
| `fineract-investor`, `fineract-rates`, `fineract-tax`, `fineract-charge` | Średni; głównie poziom integracyjny. |
| `fineract-mix`, `fineract-report` | Niższy; wyniki raportowania są trudne do asercji. |
| `fineract-document` | Niższy; zależy od mocków FS / S3. |

> TODO (wymaga potwierdzenia eksperta): opublikować liczby JaCoCo na moduł. Build stosuje wtyczkę JaCoCo (`build.gradle:431`), ale zestaw dokumentacji nie zawiera jeszcze raportu pokrycia.

## Jak uruchomić testy lokalnie

```
./gradlew test                                          # wszystkie testy jednostkowe (równolegle)
./gradlew :integration-tests:integrationTest            # zestaw integracyjny
./gradlew :fineract-e2e-tests-runner:cucumber           # zestaw BDD
./gradlew :twofactor-tests:integrationTest              # zestaw 2FA
./gradlew :oauth2-tests:integrationTest                 # zestaw OAuth2
./gradlew :fineract-provider:bootRun &                  # do testów ad-hoc
```

Lokalna baza danych domyślnie to MariaDB. Aby uruchomić wariant postgres lokalnie:

```
docker compose -f docker-compose-postgresql.yml up -d
./gradlew test -Pdb=postgresql
```

> TODO (wymaga potwierzenia eksperta): przełącznik `-Pdb=postgresql` jest przywoływany w CI (`build-postgresql.yml`); sprawdź nazwę właściwości w `build.gradle`.


---

← Poprzedni: [Podręcznik operacyjny](10-operational-runbook.md) · ↑ [Indeks](../README.md) · Następny: [Rejestr decyzji](12-decision-log.md) →
