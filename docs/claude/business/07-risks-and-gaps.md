# Ryzyka i luki

<details>
<summary><strong>Skocz do dowolnego dokumentu</strong></summary>

**[Indeks](../README.md)**

**Biznesowe:** [01 Podsumowanie wykonawcze](01-executive-summary.md) · [02 Przegląd produktu](02-product-overview.md) · [03 Procesy biznesowe](03-business-processes.md) · [04 Słownik domenowy](04-domain-glossary.md) · [05 Reguły biznesowe](05-business-rules.md) · [06 Integracje i interesariusze](06-integrations-and-stakeholders.md) · **07 Ryzyka i luki**

**Techniczne:** [01 Przegląd architektury](../technical/01-architecture-overview.md) · [02 Stos technologiczny](../technical/02-tech-stack.md) · [03 Mapa repozytorium](../technical/03-repository-map.md) · [04 Model danych](../technical/04-data-model.md) · [05 Dokumentacja API](../technical/05-api-reference.md) · [06 Przepływy uruchomieniowe](../technical/06-runtime-flows.md) · [07 Infrastruktura i wdrożenie](../technical/07-infrastructure-and-deployment.md) · [08 Konfiguracja i flagi funkcji](../technical/08-configuration-and-feature-flags.md) · [09 Model bezpieczeństwa](../technical/09-security-model.md) · [10 Podręcznik operacyjny](../technical/10-operational-runbook.md) · [11 Strategia testowania](../technical/11-testing-strategy.md) · [12 Rejestr decyzji](../technical/12-decision-log.md)

</details>

> Zaobserwowane ryzyka, nieudokumentowane zachowania, przestarzałe ścieżki i kwestie zgodności. Użyj tego jako listy zadań dla ekspertów dziedzinowych (SME).

## Ryzyka operacyjne

### Domyślne poświadczenia i artefakty deweloperskie w kodzie źródłowym

- **Ryzyko.** Plik `keystore.jks` jest dostarczany wraz z buildem (`fineract-provider/src/main/resources/keystore.jks`); hasło `openmf` jest zapisane tekstem jawnym w `application.properties:391`.
- **Dlaczego to ma znaczenie.** Naiwne wdrożenie wystawia znany klucz prywatny przez HTTPS.
- **Łagodzenie.** Nadpisz `FINERACT_SERVER_SSL_KEY_STORE` i `FINERACT_SERVER_SSL_KEY_STORE_PASSWORD` w każdym środowisku innym niż deweloperskie. Lepiej: zakończ TLS na poziomie LB / Ingress i uruchom Fineract na czystym protokole HTTP za barierą.

### Niezabezpieczony wychodzący klient HTTP

- **Ryzyko.** `fineract.insecure-http-client=true` (domyślnie, `application.properties:221`). Połączenia wychodzące akceptują certyfikaty podpisane przez samego siebie.
- **Łagodzenie.** Ustaw `FINERACT_INSECURE_HTTP_CLIENT=false` na produkcji.

### Szeroko otwarty CORS

- **Ryzyko.** `fineract.security.cors.allowed-origin-patterns=*` i inne ustawienia CORS domyślnie przyjmują wartość `*` (`application.properties:30-35`).
- **Łagodzenie.** Ogranicz do znanych domen klienta WWW na produkcji.

### HSTS domyślnie wyłączony

- **Ryzyko.** `fineract.security.hsts.enabled=false`.
- **Łagodzenie.** Włącz za LB kończącym TLS.

### Dołączony klient demonstracyjny OAuth2

- **Ryzyko.** Rejestracja `frontend-client` z adresem zwrotnym `http://localhost:3000/callback` jest skonfigurowana w `application.properties:38-42`.
- **Łagodzenie.** Zastąp lub usuń przed uruchomieniem produkcyjnym; najlepiej skieruj do zewnętrznego dostawcy tożsamości (IdP).

### Manifesty Kubernetes są minimalne

- **Ryzyko.** Katalog `kubernetes/` zawiera proste pliki YAML skierowane na pojedynczą bazę MariaDB. Brak Ingress, brak charta Helm, brak autoskalowania zasobów, brak NetworkPolicy.
- **Łagodzenie.** Operatorzy muszą opakować te manifesty w Helm/Kustomize na potrzeby produkcji. Użyj zarządzanej bazy danych poza klastrem.

### Domyślne ustawienia bazy danych dewelopera w kodzie

- **Ryzyko.** `application.properties` domyślnie ustawia `root`/`mysql` dla bazy danych tenanta. Pliki env Compose zawierają podobnie oczywiste wartości domyślne.
- **Łagodzenie.** Zmienne środowiskowe nadpisują te wartości; upewnij się, że system zarządzania sekretami (np. K8s `Secret`, AWS Secrets Manager) je uzupełnia.

## Ryzyka behawioralne / danych

### Migracje Liquibase są tylko do przodu

- **Ryzyko.** Edycja lub zmiana kolejności historycznych części changelogu powoduje błąd uruchomienia (niezgodność sumy kontrolnej Liquibase).
- **Łagodzenie.** Nigdy nie edytuj wydanych migracji. Dodawaj nowe migracje na dole odpowiedniego pliku `module-changelog-master.xml` (`db.changelog-master.xml:35-42`).

### Szyfrowanie poświadczeń bazy danych tenanta używa statycznego hasła głównego

- **Ryzyko.** `fineract.tenant.master-password=fineract` jest wartością domyślną; musi ono zostać zmienione i przechowywane jako sekret.
- **Łagodzenie.** Wstrzyknij `FINERACT_DEFAULT_TENANTDB_MASTER_PASSWORD` przez menedżera sekretów wdrożenia.

### Opóźnienia COB powodują efekty kaskadowe

- **Ryzyko.** Jeśli proces LOAN_COB (Close of Business dla pożyczek) ma opóźnienia, naliczenia, zaległości, opłaty karne, NPA i rezerwy zostają wstrzymane. Długa przerwa może wymagać ręcznego nadrabiania zaległości.
- **Łagodzenie.** Monitoruj `/v1/internal/cob/oldest-cob-closed`; alarmuj, jeśli data jest starsza niż wynikająca z SLA. Użyj punktu końcowego do nadrabiania zaległości lub ponownych prób `LOAN_COB`.

### Długotrwałe migracje przy onboardingu tenanta

- **Ryzyko.** Dodanie nowego tenanta uruchamia Liquibase na pustej bazie danych przy starcie; wiele changelogów modułów sprawia, że jest to powolne.
- **Łagodzenie.** Użyj profilu `liquibase-only`, aby zastosować migracje jako krok wdrożeniowy, a nie w aplikacji przy pierwszym żądaniu.

### Okno retencji `Idempotency-Key`

- **Ryzyko.** Aplikacja przechowuje klucze idempotencji; jeśli czas retencji jest zbyt krótki, równoległe ponowne próby mogą zostać zastosowane dwukrotnie.
- **Łagodzenie.** *(wywnioskowane)* Upewnij się, że `PURGE_PROCESSED_COMMANDS` nie jest zbyt agresywne w stosunku do zachowania klienta w zakresie ponawiania prób.

### Dwóch menedżerów i problem split-brain przy COB

- **Ryzyko.** Uruchomienie dwóch węzłów z `FINERACT_MODE_BATCH_MANAGER_ENABLED=true` dla tego samego tenanta bez brokera prowadzi do zduplikowanego przetwarzania partycji.
- **Łagodzenie.** Uruchom albo pojedynczego menedżera, albo koordynuj działania przez Kafkę/ActiveMQ.

## Kwestie zgodności

### Kompletność audytu

- Platforma śledzi: źródło komendy (`f_command_source`), audyt HTTP (`request_audit_table`), kolumny audytowe poszczególnych encji (`createdby_id`, `lastmodifiedby_id` itp.) oraz dedykowane tabele historii (`m_loan_status_change_history`, `m_calendar_history`, `m_tax_component_history`).
- > TODO (wymaga potwierdzenia SME): czy poziom retencji audytu jest zgodny z wymaganiami lokalnego regulatora bankowego? Zdefiniuj politykę retencji dla każdej tabeli.

### Rezydencja danych PII

- Bazy danych dla poszczególnych tenantów ułatwiają zarządzanie rezydencją danych, ale **dokumenty** w S3 dziedziczą region bucketa.
- > TODO (wymaga potwierzenia SME): udokumentuj region bucketa dla każdego tenanta oraz politykę replikacji międzyregionowej (lub jej brak).

### Szyfrowanie w spoczynku

- Tylko **poświadczenia bazy danych tenanta** w magazynie tenantów są szyfrowane przez aplikację (AES/CBC/PKCS5Padding).
- Dane tenantów, dokumenty i ładunki zdarzeń polegają na **infrastrukturze** w zakresie szyfrowania (TDE bazy danych, S3 SSE, Kafka TLS).
- > TODO (wymaga potwierzenia SME): potwierdź konfigurację DB TDE / S3 SSE dla każdego środowiska.

### Szyfrowanie w transporcie

- HTTPS kończy się domyślnie na dołączonym magazynie kluczy (tylko dewelopersko).
- Kafka/JMS TLS: nie wymuszane w ustawieniach domyślnych; konfigurowalne przez `extra-properties`.

### Retencja i usuwanie danych

- Dane klientów nie są usuwane automatycznie; instytucja musi wdrożyć polityki retencji.
- Zadania `PURGE_EXTERNAL_EVENTS` i `PURGE_PROCESSED_COMMANDS` czyszczą wewnętrzne kolejki, ale **nie** anonimizują danych klientów / pożyczek / oszczędności.
- > TODO (wymaga potwierzenia SME): udokumentuj przepływy pracy dotyczące RODO / lokalnego prawa do bycia zapomnianym.

### Wymuszanie zasady maker-checker

- Zasada maker-checker jest **opcjonalna dla każdego uprawnienia**. Instytucja musi skonfigurować, które uprawnienia jej wymagają.
- > TODO (wymaga potwierzenia SME): opublikuj wybraną przez instytucję macierz maker-checker.

## Przestarzałe / legacy ścieżki *(zaobserwowane)*

- Kilka migracji Liquibase jawnie usuwa martwe artefakty:
  - `0013_remove_topics.xml` — usunięto starszy mechanizm tematów/powiadomień.
  - `0014_remove_unused_jobs.xml` — czyszczenie tabel zadań Quartz.
- Literówka w kodzie (`fineract.tenant.encrytion`, `application.properties:54`; oraz `GENERATE_RD_SCEHDULE` w `JobName`) jest zachowana dla kompatybilności wstecznej — jej poprawienie zmieniłoby klucze konfiguracji/nazwy zadań.
- Kilka komentarzy odnosi się do "starego/klasycznego Mifos Workspace 2.0" w `ServerApplication` — artefakt historyczny; nieszkodliwy.
- Komentarz `// TODO: this is work in progress, please follow FINERACT-1171` na górze głównego `build.gradle:19` wskazuje, że sam proces budowania podlega długofalowej modernizacji.

## Nieudokumentowane zachowania do zweryfikowania

> TODO (wymaga potwierzenia SME):

1. Dokładna wersja API Mojaloop zaimplementowana przez `/v1/interoperation/*` oraz czy rozliczenia (settlement) wchodzą w zakres.
2. Czy dołączony magazyn kluczy jest zastępowany w obrazach Docker budowanych przez CI, czy tylko we wdrożeniach u klienta.
3. Polityka retencji dla `request_audit_table` i `f_command_source` — czy istnieje jakiekolwiek zadanie czyszczące poza `PURGE_PROCESSED_COMMANDS`?
4. Lista kluczy `c_configuration` rozpoznawanych przez kod (obecnie rozproszone po migracjach i inicjalizacji modułów).
5. Zalecany wzorzec podłączania zewnętrznego IdP (Keycloak / Okta) — dołączony `AuthorizationServerConfig` jest zorientowany na deweloperów.
6. Rzeczywista lista zdarzeń Avro emitowanych z każdego modułu.
7. Czy rejestracja OAuth2 `frontend-client` jest tylko przykładem, czy jest w aktywnym użyciu przez klienta WWW Mifos.
8. Czy domyślne ustawienie `fineract.insecure-http-client=true` jest wymagane przez jakąkolwiek integrację własną (first-party).
9. Zalecany chart Helm (utrzymywany przez społeczność?) dla produkcyjnego K8s.
10. Bieżące pokrycie JaCoCo na moduł i które moduły znajdują się poniżej progu jakości zespołu.

## Ryzyka wydajności i skali

- Brak artefaktów do testów obciążeniowych w kodzie (brak JMeter/Gatling). Oczekiwania są anegdotyczne.
- `application.properties` nie określa rozmiarów puli Hikari — przyjmują one domyślne wartości Spring Boot; mogą zostać wyczerpane przy gwałtownych wzrostach ruchu.
- Metadane Spring Batch rosną w czasie (tabele `BATCH_*`); należy zdefiniować czas ich przechowywania.
- `request_audit_table` rośnie liniowo wraz z ruchem; partycjonowanie / archiwizacja leży w gestii operatora.

## Ryzyka odporności (resilience)

- Resilience4j znajduje się w ścieżce klas (`build.gradle` BOM); jawne testy wstrzykiwania błędów nie są obecne w CI (`build-postgresql.yml` itp. nie symulują awarii brokera).
- Magazyn tokenów OAuth2 znajduje się domyślnie w bazie danych — awaria bazy uniemożliwia nowe logowania.

## Luki w dokumentacji

- Przed powstaniem tego zestawu dokumentów nie wprowadzono formalnych ADR-ów; **`docs/technical/12-decision-log.md`** zostało **wywnioskowane** retrospektywnie. Zweryfikuj każdy wpis z senior inżynierem przed uznaniem go za kanoniczny.
- `messages*.properties` zawiera tylko język angielski i niemiecki; inne wersje językowe muszą być dostarczone przy każdym wdrożeniu.
- Definicje raportów Pentaho i SQL raportów Stretchy są zarządzane na poziomie tenanta; brak nadrzędnego katalogu dostarczanego z systemem.

## Pytania do SME

To jest bieżąca lista — kopiuj elementy do trackera projektu w miarę uzyskiwania odpowiedzi.

1. Właściciele każdego procesu biznesowego w [`03-business-processes.md`](03-business-processes.md#process-ownership-inferred).
2. Domyślna macierz maker-checker — które komendy zapisu zawsze jej wymagają?
3. Narzędzia do onboardingu tenanta — ręczny SQL + restart, czy oskryptowane?
4. Polityka retencji dla tabel audytu, magazynu komend i skrzynki nadawczej zdarzeń zewnętrznych.
5. Polityka rotacji kluczy kryptograficznych dla `fineract.tenant.master-password`.
6. Cele RPO/RTO dla odtwarzania po awarii (DR) na środowisko.
7. Budżety wydajności dla kluczowych API (np. `POST /v1/loans` p99 < 500 ms?).
8. Domyślne wartości dla `fineract.events.external.partition-size` (5000) — czy są zgodne z przepustowością systemów docelowych?
9. Produkcyjny stos obserwacji (observability) — Prometheus + Grafana + Loki + Tempo (wybór dołączony) czy rozwiązania SaaS?
10. Lista emitowanych zdarzeń Avro; mapowanie na procesy biznesowe.


---

← Poprzedni: [Integracje i interesariusze](06-integrations-and-stakeholders.md) · ↑ [Indeks](../README.md) · Następny: [Przegląd architektury](../technical/01-architecture-overview.md) →
