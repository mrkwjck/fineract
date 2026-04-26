# Log decyzji

<details>
<summary><strong>Skocz do dowolnego dokumentu</strong></summary>

**[Indeks](../README.md)**

**Biznesowe:** [01 Podsumowanie menedżerskie](../business/01-executive-summary.md) · [02 Przegląd produktu](../business/02-product-overview.md) · [03 Procesy biznesowe](../business/03-business-processes.md) · [04 Słownik domenowy](../business/04-domain-glossary.md) · [05 Reguły biznesowe](../business/05-business-rules.md) · [06 Integracje i interesariusze](../business/06-integrations-and-stakeholders.md) · [07 Ryzyka i luki](../business/07-risks-and-gaps.md)

**Techniczne:** [01 Przegląd architektury](01-architecture-overview.md) · [02 Stos technologiczny](02-tech-stack.md) · [03 Mapa repozytorium](03-repository-map.md) · [04 Model danych](04-data-model.md) · [05 Referencja API](05-api-reference.md) · [06 Przepływy uruchomieniowe](06-runtime-flows.md) · [07 Infrastruktura i wdrożenie](07-infrastructure-and-deployment.md) · [08 Konfiguracja i flagi funkcji](08-configuration-and-feature-flags.md) · [09 Model bezpieczeństwa](09-security-model.md) · [10 Podręcznik operacyjny](10-operational-runbook.md) · [11 Strategia testowania](11-testing-strategy.md) · **12 Log decyzji**

</details>

> Wnioskowane ADR-y (Architectural Decision Records) zrekonstruowane na podstawie kodu, plików budowania i obserwowalnych wzorców. Oznaczone jako **wnioskowane**, ponieważ w repozytorium nie ma formalnego pliku ADR.

Każdy wpis jest zgodny z lekkim formatem ADR: kontekst → decyzja → konsekwencje. Zostały one zrekonstruowane retrospektywnie, a nie jako plany przyszłościowe.

---

## ADR-001 — Modularny monolit zamiast mikroserwisów

*(wnioskowane z `settings.gradle:50-80`, `build.gradle:26-86`, pojedynczego artefaktu wdrożeniowego `fineract-provider`)*

**Kontekst.** Fineract jest pochodną Mifos (poprzedniej platformy bankowej). Obsługuje regulowane instytucje finansowe, gdzie integralność transakcyjna dominuje nad skalowaniem poziomym.

**Decyzja.** Utrzymanie jednej aplikacji Spring Boot (`fineract-provider`), która agreguje moduły domenowe jako subprojekty Gradle (`fineract-loan`, `fineract-savings`, `fineract-accounting`, …). Moduły mogą być włączane/wyłączane (`fineract.module.*.enabled`), ale działają w tej samej JVM.

**Konsekwencje.** Silna spójność w różnych domenach dzięki pojedynczej transakcji bazy danych najemcy. Wolniejsze cykle kompilacji i testów niż w projekcie jednomodułowym. Skalowanie poziome opiera się na powielaniu całej aplikacji, sterowanym flagami trybu (odczyt/zapis/menedżer/worker).

---

## ADR-002 — Jedna baza danych na najemcę, z magazynem najemców

*(wnioskowane z `db.changelog-master.xml:30-44`, `JdbcConfig`, `application.properties:44-65`)*

**Kontekst.** Fineract jest wielonajemcowy (multi-tenant) dla regulowanych banków. Dane klientów muszą być audytowalne i możliwe do oddzielenia dla poszczególnych regulatorów.

**Decyzja.** Podział na dwie bazy danych:
- Schemat **tenant store** z rejestrem najemców.
- Jeden schemat **tenant** na najemcę, w pełni migrowany przez Liquibase przy starcie.

`AbstractRoutingDataSource` przełącza aktywne połączenie na podstawie identyfikatora najemcy pobranego z nagłówka `Fineract-Platform-TenantId`.

**Konsekwencje.** Tworzenie kopii zapasowych/przywracanie danych dla poszczególnych najemców jest proste (punkt w czasie na regulatora). Zapytania między najemcami są niemożliwe (to cecha, nie błąd). Czas uruchomienia skaluje się wraz z liczbą najemców (każdy najemca uruchamia Liquibase). Dane uwierzytelniające DB są szyfrowane w spoczynku (`fineract.tenant.master-password`).

---

## ADR-003 — Wzorzec Command + maker-checker dla zapisów

*(wnioskowane z modułu `fineract-command`, rodziny tabel `f_command_source`, API `/v1/makercheckers`)*

**Kontekst.** Zapisy bankowe wymagają zatwierdzenia przez drugą osobę (four-eyes approval), idempotentnych ponowień i pełnego audytu. Sam audyt na poziomie wyzwalaczy SQL jest niewystarczający.

**Decyzja.** Każda operacja zapisu jest hermetyzowana w `CommandWrapper`, zapisywana w magazynie poleceń i wykonywana przez zarejestrowany `CommandHandler`. Oczekujące polecenia (tylko od twórcy - maker) są przechowywane w oczekiwaniu na zatwierdzenie przez sprawdzającego (checker). Idempotentność wykorzystuje ten sam magazyn: ponowne przesłanie żądania z tym samym `Idempotency-Key` zwraca pierwotny wynik.

**Konsekwencje.** Silny audyt i odtwarzanie; wbudowane wsparcie dla "maker-checker". Dodaje opóźnienie (dodatkowe zapisy) dla każdej mutacji. Interfejsy API odczytu całkowicie omijają ten magazyn (CQRS-lite — patrz ADR-004).

---

## ADR-004 — CQRS-lite: oddzielenie usług odczytu od usług zapisu

*(wnioskowane z par `*ReadPlatformService` vs `*WritePlatformService` w całym kodzie)*

**Decyzja.** API odczytu używają Spring `JdbcTemplate` z ręcznie pisany kodem SQL i `RowMapper`ami; API zapisu przechodzą przez JPA + magazyn poleceń. Nie ma osobnego modelu odczytu; oba typy usług odpytują te same tabele.

**Konsekwencje.** Odczyty są dostrojone do rzeczywistego schematu relacyjnego bez narzutu JPA. Zapisy korzystają z blokowania optymistycznego JPA i cyklu życia encji. Kosztem jest powielenie kodu między DTO odczytu a encjami.

---

## ADR-005 — EclipseLink JPA ze statycznym przeplataniem (nie Hibernate)

*(wnioskowane z `build.gradle:95`, `static-weaving.gradle`, `dep org.eclipse.persistence:org.eclipse.persistence.jpa:4.0.2`)*

**Decyzja.** Użycie EclipseLink jako dostawcy JPA i przeplatanie (weaving) encji w czasie budowania za pomocą dedykowanej wtyczki `static-weaving.gradle` stosowanej do każdego subprojektu (`build.gradle:165-167`).

**Konsekwencje.** Pozwala uniknąć przeplatania w czasie ładowania (load-time weaving) / wtyczek IDE. Leniwe ładowanie (lazy-loading) relacji `@OneToOne` typu mapped-by w EclipseLink działa poprawnie. Rozwiązanie poza domyślną ścieżką Spring Boot — operatorzy muszą rozumieć proces budowania, aby debugować problemy z encjami.

---

## ADR-006 — Liquibase, kontekstowe dzienniki zmian, historia tylko do dopisywania

*(wnioskowane z `db.changelog-master.xml`, strażników kontekstu `tenant_db AND !initial_switch`, 218 numerowanych części w `tenant/parts`)*

**Decyzja.** Wszystkie zmiany schematu to zestawy zmian (changesets) Liquibase XML, ograniczone kontekstami Liquibase (`tenant_db`, `tenant_store_db`, `initial_switch`, `custom_changelog`). Dzienniki zmian modułów znajdują się w każdym module i są dołączane w kolejności **append-only** w pliku głównym (`db.changelog-master.xml:35-42`).

**Konsekwencje.** Dodanie modułu zawsze dopisuje zmiany na końcu — co chroni zakresy auto-increment. Zmiana kolejności lub edycja historycznych zestawów zmian jest zabroniona. Wdrożenia tylko "do przodu"; wycofywanie zmian odbywa się poprzez migrację danych, a nie rollbacks Liquibase.

---

## ADR-007 — Wtyczkowe procesory transakcji pożyczkowych

*(wnioskowane z przełączników `fineract.loan.transactionprocessor.*` w `application.properties:165-175`)*

**Decyzja.** Wiele strategii alokacji spłat (creocore, mifos-standard, rbi-india, advanced-payment-strategy, due-penalty-fee-interest-…, itp.) jest dostarczanych w kodzie źródłowym i wybieranych dla każdego produktu pożyczkowego.

**Konsekwencje.** Możliwość obsługi różnych jurysdykcji / modeli biznesowych bez forkowania kodu. Zwiększa kombinatoryczną powierzchnię testową; zachowanie zależy od tego, która strategia jest aktywna.

---

## ADR-008 — Quartz dla crona, Spring Batch dla partycjonowanych zadań

*(wnioskowane z enuma `JobName`, `application.properties:88-95`, schematu `0021_add_spring_batch_db_structure.xml`)*

**Decyzja.** Użycie Quartz do małych cyklicznych zadań (naliczanie odsetek, NPA, inkrementacja daty biznesowej) oraz Spring Batch (z metadanymi w bazie danych) do długotrwałych prac partycjonowanych, takich jak Loan COB (zamknięcie dnia dla pożyczek).

**Konsekwencje.** Dwie abstrakcje harmonogramowania do ogarnięcia. Partycjonowanie Spring Batch + wzorzec manager/worker czysto mapuje się na flagi `fineract.mode.batch-{manager,worker}-enabled`. Odzyskiwanie odbywa się na poziomie partycji; zablokowane partycje ponawiają próby do limitu `LOAN_COB_RETRY_LIMIT`.

---

## ADR-009 — Wtyczkowy transport asynchroniczny: Spring events / JMS / Kafka

*(wnioskowane z `application.properties:97-152`, profilu `activeMqEnabled`)*

**Decyzja.** Wspólna abstrakcja nad brokerami wiadomości (`fineract.remote-job-message-handler.*`, `fineract.events.external.producer.*`) pozwala operatorom wybrać transport dla danego środowiska. Domyślnie zdarzenia Spring wewnątrz procesu; opcje klasy produkcyjnej to ActiveMQ Classic (JMS) i Apache Kafka (w tym AWS MSK).

**Konsekwencje.** Brak sztywnego powiązania z konkretnym brokerem. Rozrost konfiguracji. Wymagana dbałość o to, by węzły manager i worker używały tego samego transportu.

---

## ADR-010 — Wzorzec Outbox dla zdarzeń zewnętrznych

*(wnioskowane z zadania `SEND_ASYNCHRONOUS_EVENTS` w `JobName`, `fineract.events.external.partition-size` i strojenia puli wątków producenta)*

**Decyzja.** Zdarzenia domenowe są zapisywane do trwałego outboxa w tej samej transakcji DB co zmiana biznesowa. Osobne zadanie Quartz przesyła strumieniowo zawartość outboxa do brokera po zatwierdzeniu transakcji (commit).

**Konsekwencje.** Brak problemu podwójnego zapisu (dual-write); konsumenci widzą tylko trwały stan. Dodaje opóźnienie end-to-end; zdarzenia są ostatecznie spójne (eventually consistent).

---

## ADR-011 — Rozszerzenia definiowane przez użytkownika via Datatables

*(wnioskowane z `x_registered_table`, `m_field_configuration`, punktów końcowych `/v1/datatables/*`)*

**Decyzja.** Najemcy rozszerzają encje domenowe o dowolne kolumny za pomocą "Datatables" — tabel rozszerzeń rejestrowanych przez API Datatables. Reguły walidacji znajdują się w `m_field_configuration`.

**Konsekwencje.** Dane specyficzne dla najemcy bez forkowania schematu. Raportowanie musi rozumieć dynamiczny schemat. Ścisła walidacja danych wejściowych (regex / mime / zabezpieczenia przed SQL-injection) kompensuje elastyczność rozwiązania.

---

## ADR-012 — Wtyczkowy magazyn treści: system plików lub S3

*(wnioskowane z `application.properties:186-194`, `software.amazon.awssdk:bom`, `io.awspring.cloud:spring-cloud-aws-dependencies`)*

**Decyzja.** Wspólny interfejs obsługuje przechowywanie dokumentów albo w lokalnym systemie plików (domyślnie, `${user.home}/.fineract`), albo w AWS S3.

**Konsekwencje.** Ta sama ścieżka kodu dla deweloperki (FS) i produkcji (S3). S3 wymaga jawnej konfiguracji danych uwierzytelniających. Ograniczenia: listy dozwolonych rozszerzeń plików/typów mime są wspólne dla obu rozwiązań.

---

## ADR-013 — Generowane klienty REST (`fineract-client`, `-feign`)

*(wnioskowane z wtyczki `org.openapi.generator`, `build.gradle:125`, modułów `fineract-client` i `fineract-client-feign`)*

**Decyzja.** Generowanie SDK Java z `fineract.json` (wynik Swaggera) w czasie budowania. Dwa warianty: klient w stylu Retrofit i klient w stylu Feign.

**Konsekwencje.** Wewnętrzne testy integracyjne ponownie wykorzystują SDK — co wymusza poprawność kontraktów API. Konsumenci SDK są powiązani z szablonami OpenAPI generatora; nietrywialne zmiany niosą ze sobą skutki uboczne.

---

## ADR-014 — Flagi trybu Maker pozwalają na wyspecjalizowane formy wdrożeń

*(wnioskowane z `application.properties:67-70` i wzorca manager/worker w plikach compose)*

**Decyzja.** Pojedynczy obraz może działać jako: węzeł API, węzeł tylko do zapisu, menedżer wsadowy (batch manager), worker wsadowy, replika tylko do odczytu — poprzez przełączanie zmiennych środowiskowych `FINERACT_MODE_*`.

**Konsekwencje.** Mniej artefaktów budowania. Operatorzy muszą synchronizować diagram topologii z plikami środowiskowymi. Przełączanie w trakcie działania (`/v1/instance-mode`) jest obsługiwane, ale rzadko bezpieczne pod aktywnym obciążeniem zapisami.

---

## ADR-015 — JAX-RS przez Jersey, nie Spring MVC

*(wnioskowane z `JerseyConfig @ApplicationPath("/api")` i Glassfish Jersey BOM)*

**Decyzja.** Mimo użycia Spring Boot, REST jest wystawiony przez JAX-RS (Jersey) zamiast `@RestController`ów.

**Konsekwencje.** Znajome dla praktyków Java EE; czyste oddzielenie frameworka HTTP od DI Springa. Niektóre funkcje webowe Springa (np. `@RestControllerAdvice`) nie mają zastosowania — maperzy wyjątków to `@Provider`y JAX-RS.

---

## ADR-016 — Apache Avro dla zdarzeń zewnętrznych

*(wnioskowane z modułu `fineract-avro-schemas` i `fineract.events.external.producer.kafka.topic.name=external-events`)*

**Decyzja.** Zdarzenia zewnętrzne są wysyłane ze schematem Avro; konsumenci mogą polegać na regułach ewolucji schematu.

**Konsekwencje.** Rejestr schematów (schema registry) zalecany dla wdrożeń produkcyjnych. Dodaje generowanie kodu w czasie kompilacji. Oddziela format przesyłu od wewnętrznych typów Java.

---

## ADR-017 — Pliki Compose na profil uruchomieniowy, nie flagi w jednym pliku

*(wnioskowane z kilkunastu plików `docker-compose-*.yml` w korzeniu repozytorium)*

**Decyzja.** Każda obsługiwana topologia uruchomieniowa ma swój własny plik `docker-compose-*.yml`. Wspólne usługi znajdują się w `config/docker/compose/` i są rozszerzane (`extends:`) przez pliki główne.

**Konsekwencje.** Łatwe dla nowych osób do wybrania stosu. Pliki główne z czasem się rozchodzą; osoba dodająca nową funkcję musi zaktualizować każdy wariant, na którym jej zależy (patrz np. `docker-compose-postgresql-test-activemq.yml` odzwierciedlający `docker-compose-postgresql-activemq.yml`).

---

## ADR-018 — Czysty YAML Kubernetes, brak wykresu Helm

*(wnioskowane z katalogu `kubernetes/` zawierającego tylko `*.yml` i skrypty pomocnicze powłoki)*

**Decyzja.** Dostarczenie minimalnego pakietu manifestów, aby początkujący mogli użyć `kubectl apply -f`. Oczekuje się, że użytkownicy produkcyjni sami przygotują Helm / Kustomize.

**Konsekwencje.** Mniejsze obciążenie utrzymaniem dla autorów. Wdrożenia produkcyjne odpowiadają za Ingress, TLS, zarządzanie sekretami, RBAC, autoskalowanie i dobór zasobów.

---

## ADR-019 — Aktualizacje zależności sterowane przez Renovate, przypięte wersje

*(wnioskowane z `renovate.json` i polityki jawnych wersji w `dependencies.gradle`)*

**Decyzja.** Przypięcie wersji każdej zależności. Pozostawienie Renovate Botowi podbijania PR-ów z aktualizacjami. Unikanie selektorów `+` / `latest`.

**Konsekwencje.** Stabilne, powtarzalne buildy. Wymagana dyscyplina w obsłudze kolejki PR. Poprawki wymagają aktywnych opiekunów.

---

## ADR-020 — Zabezpieczenie przed SQL-injection na warstwie właściwości

*(wnioskowane z `application.properties:228-319`, profilów `main`, `adhoc`, `column`)*

**Decyzja.** Ścieżki SQL o dowolnej formie (raporty Stretchy, zapytania ad-hoc, filtry datatables) przechodzą przez konfigurowalny silnik wzorców regex przed wykonaniem.

**Konsekwencje.** Wychwytuje klasę oczywistych prób wstrzyknięcia. Możliwe fałszywe trafienia (false positives) dla specyficznych, poprawnych zapytań SQL (np. złożone klauzule `ORDER BY`); operatorzy mogą dostroić profil na daną integrację. Nie zastępuje to zapytań parametryzowanych w innych miejscach — a większość zapytań domenowych już używa bindowania.

---

## Notatki dotyczące procesu (również wnioskowane)

- Gałęzie: długożyjące gałęzie `release/<x.y.z>` i `maintenance/<x.y>` zasilają `git-versioning` (`build.gradle:140-159`).
- Audyty wydań Apache (`org.nosphere.apache.rat`) sprawdzają każdy PR pod kątem nagłówków licencyjnych i zabronionych plików binarnych.
- Wszystkie źródła Java przechodzą przez formatowanie Spotless + Greclipse przed połączeniem (merge).


---

← Poprzedni: [Strategia testowania](11-testing-strategy.md) · ↑ [Indeks](../README.md) · Następny: [Dziennik zmian dokumentacji](../CHANGELOG-of-docs.md) →
