# Apache Fineract — Indeks dokumentacji

<details>
<summary><strong>Skocz do dokumentu</strong></summary>

**[Indeks](README.md)**

**Biznes:** [01 Podsumowanie wykonawcze](business/01-executive-summary.md) · [02 Przegląd produktu](business/02-product-overview.md) · [03 Procesy biznesowe](business/03-business-processes.md) · [04 Słownik domenowy](business/04-domain-glossary.md) · [05 Reguły biznesowe](business/05-business-rules.md) · [06 Integracje i interesariusze](business/06-integrations-and-stakeholders.md) · [07 Ryzyka i luki](business/07-risks-and-gaps.md)

**Techniczne:** [01 Przegląd architektury](technical/01-architecture-overview.md) · [02 Stos technologiczny](technical/02-tech-stack.md) · [03 Mapa repozytorium](technical/03-repository-map.md) · [04 Model danych](technical/04-data-model.md) · [05 Dokumentacja API](technical/05-api-reference.md) · [06 Przepływy wykonawcze](technical/06-runtime-flows.md) · [07 Infrastruktura i wdrażanie](technical/07-infrastructure-and-deployment.md) · [08 Konfiguracja i flagi funkcji](technical/08-configuration-and-feature-flags.md) · [09 Model bezpieczeństwa](technical/09-security-model.md) · [10 Podręcznik operacyjny](technical/10-operational-runbook.md) · [11 Strategia testowania](technical/11-testing-strategy.md) · [12 Rejestr decyzji](technical/12-decision-log.md)

</details>

Wygenerowany zestaw dokumentacji dla bazy kodu Apache Fineract w katalogu głównym tego repozytorium. Zestaw jest podzielony na dokumentację **biznesową** i **techniczną**.

Zacznij tutaj:

- Jesteś nowy w Fineract? → [`business/01-executive-summary.md`](business/01-executive-summary.md)
- Inżynier dołączający do projektu? → [`technical/01-architecture-overview.md`](technical/01-architecture-overview.md) i [`technical/03-repository-map.md`](technical/03-repository-map.md)
- Operator uruchamiający system? → [`technical/10-operational-runbook.md`](technical/10-operational-runbook.md)
- Ryzyka / otwarte pytania? → [`business/07-risks-and-gaps.md`](business/07-risks-and-gaps.md)

## Dokumentacja biznesowa

| Plik | Co obejmuje |
| --- | --- |
| [`business/01-executive-summary.md`](business/01-executive-summary.md) | Jednostronicowy przegląd: co robi system, komu służy, dlaczego istnieje. |
| [`business/02-product-overview.md`](business/02-product-overview.md) | Mapa możliwości, persony, główne ścieżki użytkownika. |
| [`business/03-business-processes.md`](business/03-business-processes.md) | Przepływy end-to-end (onboarding, pożyczki, COB, zamknięcie GL, …). |
| [`business/04-domain-glossary.md`](business/04-domain-glossary.md) | Definicje w prostym języku i indeks akronimów. |
| [`business/05-business-rules.md`](business/05-business-rules.md) | Reguły walidacji, kwalifikowalności, cyklu życia i księgowości z odniesieniami do kodu. |
| [`business/06-integrations-and-stakeholders.md`](business/06-integrations-and-stakeholders.md) | Integracje przychodzące/wychodzące, własność, wymieniane dane. |
| [`business/07-risks-and-gaps.md`](business/07-risks-and-gaps.md) | Ryzyka, nieudokumentowane zachowania i bieżąca lista pytań do ekspertów (SME). |

## Dokumentacja techniczna

| Plik | Co obejmuje |
| --- | --- |
| [`technical/01-architecture-overview.md`](technical/01-architecture-overview.md) | Diagramy kontekstu, kontenerów i komponentów w stylu C4; kluczowe wzorce. |
| [`technical/02-tech-stack.md`](technical/02-tech-stack.md) | Języki, frameworki, biblioteki i narzędzia wraz z wersjami. |
| [`technical/03-repository-map.md`](technical/03-repository-map.md) | Przewodnik moduł po module po wielomodułowym projekcie Gradle. |
| [`technical/04-data-model.md`](technical/04-data-model.md) | Schematy, kluczowe tabele, konwencje nazewnictwa, szybka mapa ER. |
| [`technical/05-api-reference.md`](technical/05-api-reference.md) | Katalog zasobów REST, nagłówki uwierzytelniania, model błędów, API wsadowe. |
| [`technical/06-runtime-flows.md`](technical/06-runtime-flows.md) | Diagramy sekwencji dla 10 najważniejszych przepływów wykonawczych. |
| [`technical/07-infrastructure-and-deployment.md`](technical/07-infrastructure-and-deployment.md) | Budowanie kontenerów, stosy Compose, Kubernetes, CI/CD, obserwowalność. |
| [`technical/08-configuration-and-feature-flags.md`](technical/08-configuration-and-feature-flags.md) | Katalog `application.properties` i parametry konfiguracyjne czasu wykonania. |
| [`technical/09-security-model.md`](technical/09-security-model.md) | Uwierzytelnianie, autoryzacja, sekrety, powierzchnia ataku. |
| [`technical/10-operational-runbook.md`](technical/10-operational-runbook.md) | Operacje dnia drugiego (Day-2), typowe incydenty, rozwiązywanie problemów. |
| [`technical/11-testing-strategy.md`](technical/11-testing-strategy.md) | Warstwy testowe, co jest objęte testami, a co nie. |
| [`technical/12-decision-log.md`](technical/12-decision-log.md) | Wnioskowane ADR-y (Architectural Decision Records) zrekonstruowane z kodu i historii. |

## Konwencje

- Cytowania używają formy `ścieżka/do/pliku.ext:Lstart-Lend`.
- Diagramy to PlantUML w blokach kodu (` ```plantuml `), dzięki czemu renderują się w dowolnej przeglądarce Markdown wspierającej bloki `plantuml`.
- `*(wnioskowane z <file>:<line>)*` oznacza twierdzenia, które nie są dosłownie stwierdzone w kodzie, ale wynikają z obserwacji.
- `> TODO (wymaga potwierdzenia SME): ...` oznacza otwarte pytania; skonsolidowana lista znajduje się w [`business/07-risks-and-gaps.md`](business/07-risks-and-gaps.md#questions-for-smes).

## Utrzymanie

Gdy baza kodu ulegnie znaczącej zmianie (nowy moduł, nowa migracja schematu, nowa integracja, nowa flaga funkcji):

1. Zaktualizuj odpowiednie dokumenty.
2. Dodaj wpis do [`CHANGELOG-of-docs.md`](CHANGELOG-of-docs.md).

## Poza zakresem

Ten zestaw dokumentacji obejmuje repozytorium **fineract** w tej wersji. Klient webowy społeczności Mifos, aplikacje mobilne i powiązane mikroserwisy znajdują się w oddzielnych repozytoriach i są tutaj jedynie przywoływane.


---

Dalej: [Podsumowanie wykonawcze](business/01-executive-summary.md) →
