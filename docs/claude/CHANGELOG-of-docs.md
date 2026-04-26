# Dziennik zmian dokumentacji

<details>
<summary><strong>Skocz do dowolnego dokumentu</strong></summary>

**[Indeks](README.md)**

**Biznesowe:** [01 Podsumowanie menedżerskie](business/01-executive-summary.md) · [02 Przegląd produktu](business/02-product-overview.md) · [03 Procesy biznesowe](business/03-business-processes.md) · [04 Słownik domenowy](business/04-domain-glossary.md) · [05 Reguły biznesowe](business/05-business-rules.md) · [06 Integracje i interesariusze](business/06-integrations-and-stakeholders.md) · [07 Ryzyka i luki](business/07-risks-and-gaps.md)

**Techniczne:** [01 Przegląd architektury](technical/01-architecture-overview.md) · [02 Stos technologiczny](technical/02-tech-stack.md) · [03 Mapa repozytorium](technical/03-repository-map.md) · [04 Model danych](technical/04-data-model.md) · [05 Referencja API](technical/05-api-reference.md) · [06 Przepływy wykonawcze](technical/06-runtime-flows.md) · [07 Infrastruktura i wdrożenie](technical/07-infrastructure-and-deployment.md) · [08 Konfiguracja i flagi funkcji](technical/08-configuration-and-feature-flags.md) · [09 Model bezpieczeństwa](technical/09-security-model.md) · [10 Instrukcja operacyjna (Runbook)](technical/10-operational-runbook.md) · [11 Strategia testowania](technical/11-testing-strategy.md) · [12 Rejestr decyzji](technical/12-decision-log.md)

</details>

> Śledzi, kiedy i dlaczego zestaw dokumentacji został wygenerowany lub odświeżony.

## 2026-04-26 — Początkowe wygenerowanie

Wygenerowano pełny zestaw dokumentacji zgodnie z metodologią opisaną w `/CLAUDE.md`.

- Odbiorcy: inżynierowie i interesariusze biznesowi nowi w Apache Fineract.
- Wersja źródłowa (source-of-truth): gałąź `ai-documentaton-poc` w momencie generowania.
- Autor: automatyczny przebieg inżynierii wstecznej; wpisy oznaczone jako `*(inferred …)*` lub `> TODO (needs SME confirmation): …` oznaczają twierdzenia, które powinny zostać zatwierdzone przez eksperta dziedzinowego (SME) przed uznaniem ich za kanoniczne.
- Zakres: wszystkie 13 plików technicznych i 7 plików biznesowych określonych w `/CLAUDE.md §3`, plus ten dziennik zmian oraz indeks `docs/README.md`.
- Lista otwartych pytań: zobacz `business/07-risks-and-gaps.md`.

### Status pokrycia

| Obszar | Status |
| --- | --- |
| Mapa repozytorium, stos technologiczny | Kompletne; oparte na cytatach. |
| Przegląd architektury, przepływy wykonawcze | Kompletne; diagramy wykorzystują PlantUML. |
| Model danych | Tylko przegląd na poziomie schematu; szczegółowe informacje o kolumnach dla poszczególnych tabel **nie** wchodzą w zakres tego początkowego przebiegu. |
| Referencja API | Katalog zasobów kompletny; szczegóły parametrów poszczególnych punktów końcowych odłożone do wygenerowanego pliku Swagger `fineract.json`. |
| Konfiguracja | Objęto wszystkie główne grupy właściwości; lista kluczy uruchomieniowych `c_configuration` jest niekompletna (TODO). |
| Bezpieczeństwo | Tryby, authentykacja/autoryzacja (authn/z), zagrożenia objęte; głębsze modelowanie zagrożeń odłożone. |
| Strategia testowania | Podział na warstwy gotowy; konkretne liczby dotyczące pokrycia TODO. |
| Rejestr decyzji | Dwadzieścia wywnioskowanych ADR-ów; wymagają zatwierdzenia. |
| Procesy i reguły biznesowe | Główne procesy i reguły zacytowane; niuanse specyficzne dla poszczególnych jurysdykcji odłożone. |
| Ryzyka i luki | Bieżąca lista jest utrzymywana. |

### Znane działania następcze

- Zweryfikować wywnioskowane ADR-y w `technical/12-decision-log.md` z inżynierami seniorami.
- Uzupełnić tabelę kluczy `c_configuration` wymienioną w `technical/08-configuration-and-feature-flags.md`.
- Opracować katalog zdarzeń Avro na podstawie `fineract-avro-schemas/src/main/resources/avro/**/*.avsc`.
- Zdecydować, czy dołączony klient demonstracyjny OAuth2 jest tylko przykładowy, czy aktywny produkcyjnie.
- Udokumentować strategię wykresów Helm (społecznościowy wykres Helm kontra zwykłe manifesty).
- Zebrać liczby pokrycia JaCoCo dla poszczególnych modułów (`technical/11-testing-strategy.md`).


---

← Poprzedni: [Rejestr decyzji](technical/12-decision-log.md) · ↑ [Indeks](README.md)
