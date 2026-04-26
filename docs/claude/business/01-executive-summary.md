# Podsumowanie menedżerskie

<details>
<summary><strong>Skocz do dowolnego dokumentu</strong></summary>

**[Indeks](../README.md)**

**Biznes:** **01 Podsumowanie menedżerskie** · [02 Przegląd produktu](02-product-overview.md) · [03 Procesy biznesowe](03-business-processes.md) · [04 Słownik domenowy](04-domain-glossary.md) · [05 Reguły biznesowe](05-business-rules.md) · [06 Integracje i interesariusze](06-integrations-and-stakeholders.md) · [07 Ryzyka i luki](07-risks-and-gaps.md)

**Techniczne:** [01 Przegląd architektury](../technical/01-architecture-overview.md) · [02 Stos technologiczny](../technical/02-tech-stack.md) · [03 Mapa repozytorium](../technical/03-repository-map.md) · [04 Model danych](../technical/04-data-model.md) · [05 Dokumentacja API](../technical/05-api-reference.md) · [06 Przepływy uruchomieniowe](../technical/06-runtime-flows.md) · [07 Infrastruktura i wdrożenie](../technical/07-infrastructure-and-deployment.md) · [08 Konfiguracja i flagi funkcji](../technical/08-configuration-and-feature-flags.md) · [09 Model bezpieczeństwa](../technical/09-security-model.md) · [10 Podręcznik operacyjny](../technical/10-operational-runbook.md) · [11 Strategia testowania](../technical/11-testing-strategy.md) · [12 Dziennik decyzji](../technical/12-decision-log.md)

</details>

Apache Fineract to otwarta **platforma bankowości rdzeniowej (core banking)**. Zapewnia system typu back-office, którego instytucje finansowe — w szczególności instytucje mikrofinansowe (MFI), unie kredytowe i fintechy — używają do zarządzania klientami, depozytami, pożyczkami oraz księgą główną.

## Co robi

- Utrzymuje **księgę klientów** — osoby fizyczne, grupy i centra — wraz z ich danymi KYC, adresami i relacjami rodzinnymi.
- Obsługuje **sieć oddziałów** — biura, personel, dni wolne, dni robocze, kasjerów i transakcje kasowe.
- Prowadzi **produkty pożyczkowe** od początku do końca: inicjację, zatwierdzenie, wypłatę, spłatę, śledzenie zaległości, restrukturyzację, umorzenie. Obsługiwanych jest wiele strategii alokacji spłat (wcześniejsza spłata, RBI India, płatność z góry itp.).
- Prowadzi **produkty oszczędnościowe, lokaty terminowe i produkty cykliczne** z naliczaniem odsetek, opłatami i zarządzaniem uśpionymi kontami.
- Utrzymuje pełną **księgę główną podwójnego zapisu**: plan kont, wpisy do dziennika, zamknięcia okresów, rozliczenia międzyokresowe, tworzenie rezerw i bilans próbny.
- Wykonuje codzienne **przetwarzanie na koniec dnia (COB)**: naliczenia, oznaczanie zaległości, naliczanie kar, okresowe rozliczenia międzyokresowe, klasyfikacja NPA (kredyty zagrożone).
- Udostępnia **interfejs API REST HTTPS** (`/api/v1/**`) dla personelu, klientów (samoobsługa) i integracji partnerskich. Pakietowy punkt końcowy (batch endpoint) grupuje wiele podżądań w jedną transakcję.
- Emituje **zdarzenia zewnętrzne** (zakodowane w formacie Avro przez Kafka lub JMS), dzięki czemu systemy niższego szczebla mogą reagować na aktywność finansową.

## Komu służy

- **Personelowi banku / MFI** w oddziałach i back-office — poprzez klienta webowego Mifos.
- **Klientom** — poprzez interfejsy webowe i mobilne samoobsługi (self-service), które korzystają z interfejsów API `/v1/self/*`.
- **Operatorom** — administratorom baz danych, inżynierom SRE, inżynierom dyżurnym — poprzez migracje Liquibase, Spring Boot Actuator i administracyjne interfejsy API (`/v1/configurations`, `/v1/scheduler`, `/v1/internal/cob`).
- **Regulatorom i audytorom** — poprzez niezmienne tabele poleceń (commands), audytu i wpisów do dziennika.
- **Partnerom / integratorom fintech** — poprzez interfejsy API REST chronione OAuth2 i zdarzenia zewnętrzne.

## Dlaczego istnieje

Fineract kontynuuje misję MIFOS: **gotowy na chmurę system bankowości rdzeniowej, który umożliwia cyfrowe usługi finansowe dla każdego, w tym dla osób nieubankowionych i o ograniczonym dostępie do usług bankowych** (`CLAUDE.md`). Infrastruktura bankowa historycznie była dostępna tylko na licencji, a przez to nieosiągalna dla małych instytucji na rynkach wschodzących. Fineract zapewnia te same możliwości na licencji Apache 2, dzięki czemu MFI, fintech lub unia kredytowa mogą prowadzić regulowany produkt finansowy bez kosztów licencyjnych za każde konto.

## W skrócie

| Aspekt | Wartość |
| --- | --- |
| Etap | Produkcja (używany przez wiele MFI i rosnącą liczbę fintechów). |
| Stos technologiczny | Java 21, Spring Boot 3.5, EclipseLink JPA, Liquibase, Quartz, Spring Batch, opcjonalnie Kafka / ActiveMQ. |
| Baza danych | MariaDB, MySQL lub PostgreSQL — jeden obraz aplikacji, wiele baz danych na najemcę (tenant). |
| Wdrożenie | Docker Compose (dev / pilot), Kubernetes (produkcja). |
| Wielodostępność (Multi-tenancy) | Jedna baza danych na najemcę, dynamicznie kierowana w czasie żądania. |
| Rozszerzalność | Wtyczkowe procesory transakcji pożyczkowych, tabele danych (kolumny zdefiniowane przez użytkownika), webhooki, niestandardowe moduły Gradle. |
| Licencjonowanie | Apache License 2.0 (open source). |

## Co obejmuje ta dokumentacja

- Dokumenty **Biznesowe** (ten folder) — co system robi i dla kogo, procesy biznesowe, które wspiera, słownik domenowy oraz reguły, które je wiążą.
- Dokumenty **Techniczne** (`docs/technical/`) — architektura, mapa modułów, model danych, dokumentacja API, przepływy uruchomieniowe, wdrożenie, bezpieczeństwo, podręcznik operacyjny, testowanie i zrekonstruowane zapisy decyzji.

Aby uzyskać głęboki kontekst techniczny, zacznij od [`docs/technical/01-architecture-overview.md`](../technical/01-architecture-overview.md). Dla operatorów: [`docs/technical/10-operational-runbook.md`](../technical/10-operational-runbook.md).


---

↑ [Indeks](../README.md) · Dalej: [Przegląd produktu](02-product-overview.md) →
