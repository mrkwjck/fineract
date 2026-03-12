# Moduł Stóp Procentowych (fineract-rates)

[Powrót do dokumentacji głównej](README.md)

## Opis
Moduł `fineract-rates` dostarcza scentralizowane usługi zarządzania stopami bazowymi i referencyjnymi (ang. Floating Rates, Base Rates) w systemie Apache Fineract. Obejmuje to popularne wskaźniki makroekonomiczne takie jak WIBOR, LIBOR, EURIBOR, czy stopy referencyjne lokalnych banków centralnych.

Zarządzanie zmiennym oprocentowaniem jest kluczowe w nowoczesnej bankowości. Zamiast określać "sztywne" (Flat) oprocentowanie w momencie podpisywania umowy pożyczkowej lub depozytowej (np. 10% w skali roku), instytucja finansowa może oprzeć umowę o stopę referencyjną i narzut marży (np. WIBOR 3M + 2% marży banku). Gdy bank centralny zmienia wskaźnik WIBOR, Fineract korzystając z tego modułu, propaguje nową wartość do powiązanych kont oszczędnościowych i kredytowych.

## Kluczowe komponenty biznesowe

| Komponent | Odpowiedzialność biznesowa i techniczna |
| :--- | :--- |
| **`FloatingRate`** | Definicja konkretnego wskaźnika zmiennego (np. "WIBOR_6M"). |
| **`FloatingRatePeriod`** | Zestawienie wartości wskaźnika (Rate Value) obowiązującego w określonym przedziale czasu (`from_date` do `to_date`). Pozwala systemowi na odtwarzanie pełnej historii zmian stopy i stosowanie historycznych stawek do kalkulacji zaległych odsetek. |
| **`Rate`** | Standardowa tabela definiująca opłaty procentowe powiązane z pożyczką, które mogą mieć zastosowanie w specyficznych produktach (niebędących stricte stopami referencyjnymi, np. specyficzne oprocentowanie promocyjne). |

## Architektura modułu

Architektura tego komponentu to w głównej mierze scentralizowany słownik danych zapytywany przez moduły wykonawcze (np. wyliczarkę rat).

```plantuml
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml
title Model C4 - Zależności modułu fineract-rates

Component(rate_api, "Floating Rates API", "Spring Web", "Rejestracja i aktualizacja stawek stóp bazowych przez menedżerów (np. /floatingrates)")
Component(rate_service, "FloatingRateWritePlatformService", "Serwis Zapisujący", "Tworzy nowe okresy dla wskaźnika (FloatingRatePeriod). Utrzymuje ciągłość dat bez nakładania się (no-overlap rule).")
Component(progressive_loan, "fineract-progressive-loan", "Pożyczki Zmienne", "Odpytuje moduł rat o obowiązującą stawkę WIBOR na dzień dzisiejszy w celu wygenerowania nowego harmonogramu")
Component(savings_module, "fineract-savings", "Oszczędności", "Przelicza kapitalizację dzienną korzystając z pobranej dziennej stawki referencyjnej")

SystemDb_Ext(db, "Relacyjna Baza Danych", "Model Tenanta")

Rel(rate_api, rate_service, "HTTP PUT (nowa stawka: 6.5%)")
Rel(rate_service, db, "Aktualizacja m_floating_rates / m_floating_rates_periods")
Rel(progressive_loan, db, "Odczyt stawek na potrzeby symulacji / przeliczenia rat")
Rel(savings_module, db, "Odczyt stawek dla kalkulacji Interest Posting")

@enduml
```

## Zarządzanie stanem i baza danych

Moduł wprowadza poniższe słowniki do relacyjnej bazy danych dzierżawcy:
*   `m_floating_rates`: Tabela nagłówkowa przechowująca m.in. nazwę stawki (np. "Krajowa Stopa Referencyjna NBP"), status aktywności (is_active) oraz bazę naliczania w dniach (np. czy dzielimy przez 360, czy 365 dni w roku).
*   `m_floating_rates_periods`: Tablica relacyjna powiązana `1:N` przechowująca historię wartości stopy referencyjnej. Posiada klucz obcy do `m_floating_rates`, pole `from_date` określające dzień wejścia w życie nowej wartości procentowej oraz samo pole `interest_rate`.
*   `m_rate`: Zwykła, historycznie starsza encja odsetkowa dla prostych konfiguracji marżowych, powiązana z określonym produktem.
