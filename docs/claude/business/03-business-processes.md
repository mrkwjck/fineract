# Procesy biznesowe

<details>
<summary><strong>Skocz do dowolnego dokumentu</strong></summary>

**[Indeks](../README.md)**

**Biznesowe:** [01 Podsumowanie menedżerskie](01-executive-summary.md) · [02 Przegląd produktu](02-product-overview.md) · **03 Procesy biznesowe** · [04 Słownik domenowy](04-domain-glossary.md) · [05 Reguły biznesowe](05-business-rules.md) · [06 Integracje i interesariusze](06-integrations-and-stakeholders.md) · [07 Ryzyka i luki](07-risks-and-gaps.md)

**Techniczne:** [01 Przegląd architektury](../technical/01-architecture-overview.md) · [02 Stos technologiczny](../technical/02-tech-stack.md) · [03 Mapa repozytorium](../technical/03-repository-map.md) · [04 Model danych](../technical/04-data-model.md) · [05 Dokumentacja API](../technical/05-api-reference.md) · [06 Przepływy uruchomieniowe](../technical/06-runtime-flows.md) · [07 Infrastruktura i wdrożenie](../technical/07-infrastructure-and-deployment.md) · [08 Konfiguracja i flagi funkcji](../technical/08-configuration-and-feature-flags.md) · [09 Model bezpieczeństwa](../technical/09-security-model.md) · [10 Podręcznik operacyjny](../technical/10-operational-runbook.md) · [11 Strategia testowania](../technical/11-testing-strategy.md) · [12 Dziennik decyzji](../technical/12-decision-log.md)

</details>

> Kompleksowe (end-to-end) przepływy pracy obsługiwane przez system, z opisami w stylu swimlane. Wszystkie odniesienia do interfejsów API i koncepcji domenowych prowadzą do dokumentacji technicznej.

## 1. Onboarding klienta

```plantuml
@startuml
|Klient lub pracownik|
start
:Przechwycenie profilu klienta\n(imię, KYC, adres);
|Doradca kredytowy|
:Walidacja dokumentów;
if (Czy włączono zasadę maker-checker?) then (tak)
  :Prześlij do zatwierdzenia\nstatus=PENDING;
  |Kierownik oddziału|
  :Zatwierdź w /v1/makercheckers;
else (nie)
  :Automatyczne zatwierdzenie;
endif
|System|
:Utwórz m_client\n(aktywny);
:Opcjonalnie utwórz wiersze w datatable\n(dane rozszerzone);
stop
@enduml
```

Wywołane API: `/v1/clients`, `/v1/clients/{clientId}/identifiers`, `/v1/clients/{clientId}/familymembers`, `/v1/clients/{clientId}/addresses`, `/v1/datatables/...`. Zobacz [`docs/technical/05-api-reference.md`](../technical/05-api-reference.md).

## 2. Udzielanie pożyczek, zatwierdzanie i wypłata

```plantuml
@startuml
|Doradca kredytowy|
start
:Wybierz produkt pożyczkowy;
:Wprowadź wniosek\n(kapitał, okres, harmonogram);
|System|
:Wylicz harmonogram za pomocą\nwybranego procesora transakcji;
|Doradca kredytowy|
:Prześlij wniosek;
|Kierownik oddziału|
if (Maker-checker?) then (tak)
  :Zatwierdź w /v1/makercheckers;
else (nie)
endif
:POST /v1/loans/{id}?command=approve;
:POST /v1/loans/{id}?command=disburse;
|System|
:Utwórz transakcję wypłaty;
:Księguj wpisy do dziennika (KG);
:Wyemituj zdarzenie zewnętrzne LoanDisbursed;
:Zaplanuj pierwszą spłatę;
stop
@enduml
```

Uwagi:

- Harmonogram zależy od **konfiguracji produktu pożyczkowego** (metoda odsetkowa, ponowne przeliczanie, opłaty) oraz **strategii procesora transakcji**. Zobacz [`docs/technical/08-configuration-and-feature-flags.md`](../technical/08-configuration-and-feature-flags.md#loan-transaction-processors-applicationproperties165-179).
- Pożyczki grupowe (GLIM) i oszczędności grupowe (GSIM) postępują zgodnie z tym samym przepływem, ale na poziomie konta grupowego.

## 3. Spłata

```plantuml
@startuml
|Klient|
start
:Zapłać w oddziale / przez mobile money / zlecenie stałe;
|System (API zapisu)|
:POST /v1/loans/{id}/transactions?command=repayment;
:Przydziel płatność zgodnie z\nskonfigurowaną strategią:\nkapitał / odsetki / opłaty / kary;
:Zaktualizuj harmonogram spłat (opłacona / częściowo opłacona);
:Przelicz odsetki, jeśli są dynamiczne;
:Księguj wpisy do dziennika;
:Wyemituj zdarzenie LoanRepayment;
stop
@enduml
```

Jeśli spłata w pełni pokrywa kapitał + naliczone odsetki, system oznacza pożyczkę jako **zamkniętą** (closed). Jeśli opłata lub kara została umorzona, używane są dedykowane polecenia (`waive_charge`, `waive_interest`).

## 4. Codzienne zamknięcie dnia (COB)

```plantuml
@startuml
|Harmonogram Quartz|
start
:Wyzwalacz INCREASE_BUSINESS_DATE_BY_1_DAY;
|System|
:Przesuń datę biznesową w c_configuration;
|Harmonogram Quartz|
:Wyzwalacz partycjonowanego zadania LOAN_COB;
|Węzeł zarządcy|
:Pobierz pożyczki dla daty COB,\nzablokuj je;
:Dodaj partycje do kolejki brokera;
|Węzeły wykonawcze|
:Przetwarzaj pożyczki w paczkach po 100;
:Uruchom skonfigurowane kroki biznesowe:\n  - sprawdź termin spłaty/zaległość\n  - nałóż karę\n  - ustaw tagi zaległości\n  - księguj naliczenia\n  - amortyzuj skapitalizowany dochód / buy-down\n  - przelicz odsetki;
:Zwolnij blokady; przesuń datę COB pożyczki;
|Węzeł zarządcy|
:Agreguj, ponów zablokowane partycje;
|System|
:Wyemituj zdarzenia zewnętrzne dla każdej dotkniętej pożyczki;
stop
@enduml
```

Scenariusz nadrabiania zaległości: gdy najemca (tenant) ma opóźnienia (np. awaria bazy danych), wywołaj `/v1/internal/cob/catch-up`, aby przetworzyć zaległe dni.

## 5. Zatwierdzanie Maker-checker (zasada czterech oczu)

Dotyczy każdej operacji zapisu, dla której instytucja włączyła zasadę czterech oczu.

```plantuml
@startuml
|Maker (rola ograniczona)|
start
:Prześlij operację zapisu;
|System|
:Zapisz polecenie\nstate=PENDING;
:Zwróć identyfikator audytu (audit id);
|Checker (inny użytkownik)|
:GET /v1/makercheckers (lista oczekujących);
if (zatwierdzić?) then (tak)
  :POST /v1/makercheckers/{id};
  |System|
  :Wykonaj zapisane polecenie;
  :Zwróć ostateczną odpowiedź;
else (nie)
  :DELETE /v1/makercheckers/{id};
  |System|
  :Oznacz polecenie jako REJECTED;
endif
@enduml
```

Decyzje i znaczniki czasu są zapisywane; stanowi to ścieżkę audytu instytucji (`/v1/audits` i `request_audit_table`).

## 6. Księgowe zamknięcie okresu

```plantuml
@startuml
|Księgowy|
start
:Uzgodnij wpisy do dziennika\nz księgami pomocniczymi;
:POST /v1/glclosures (closingDate);
|System|
:Odrzucaj nowe wpisy\nw dniu lub przed closingDate\ndla tego oddziału;
|Księgowy|
:Uruchom bilans próbny;
:Uruchom księgowanie rezerw;
:Uruchom eksport raportów;
stop
@enduml
```

Zamknięcie KG (Księgi Głównej) odbywa się **na oddział**: centrala może zamknąć okres, podczas gdy oddziały nadal mają otwarte okresy (lub odwrotnie, w zależności od polityki).

## 7. Tworzenie rezerw (na straty pożyczkowe)

```plantuml
@startuml
|Specjalista ds. ryzyka|
start
:Zdefiniuj kategorie rezerw\n(0-30, 31-60, 61-90, ...);
:Zdefiniuj kryteria rezerw\n(procentowo według kategorii, według produktu);
|Zadanie Quartz\nGENERATE_LOANLOSS_PROVISIONING|
:Uruchamiaj zgodnie z harmonogramem;
|System|
:Agreguj zaległe salda\nw każdym koszyku zaległości;
:Oblicz rezerwę na pożyczkę;
:Zapisz wpis dotyczący rezerwy;
:Księguj wpis do dziennika w KG\n(poprzez mapowanie produkt-do-KG);
stop
@enduml
```

## 8. Zlecenia stałe i przelewy między kontami

```plantuml
@startuml
|Klient / pracownik|
start
:Skonfiguruj zlecenie stałe\n(z konta, na konto, kwota, częstotliwość);
|Zadanie Quartz EXECUTE_STANDING_INSTRUCTIONS|
:W zaplanowanym terminie;
|System|
:Walidacja salda;
if (wystarczające?) then (tak)
  :Utwórz m_account_transfer_transaction;
  :Księguj wpisy do dziennika;
  :Zaktualizuj m_account_transfer_standing_instructions_history\n(SUCCESS);
else (nie)
  :Wiersz historii\n(INSUFFICIENT_BALANCE);
endif
stop
@enduml
```

## 9. Import masowy

```plantuml
@startuml
|Operator|
start
:Pobierz szablon Excel\n/v1/imports/downloadtemplate;
:Wypełnij wiersze (klienci / pożyczki / spłaty);
:POST /v1/imports;
|Systemowy pracownik asynchroniczny|
:Parsuj, waliduj każdy wiersz;
:Utwórz encje lub zapisz błędy;
:Zaktualizuj m_import_document.status;
|Operator|
:Odpytuj /v1/imports/{id};
stop
@enduml
```

Import masowy jest kanoniczną ścieżką migracji z arkuszy kalkulacyjnych / systemów spuścizny (legacy).

## 10. Integracja ze zdarzeniami zewnętrznymi

```plantuml
@startuml
|Usługa domenowa|
start
:Zmodyfikuj encję domenową;
:Wstaw wiersz zdarzenia do outboxa\n(ta sama transakcja DB);
|Zadanie Quartz\nSEND_ASYNCHRONOUS_EVENTS|
:Czytaj outbox seryjnie (batch);
|Broker (Kafka / JMS)|
:Odbierz ładunek (payload) zakodowany w Avro;
|Konsument niższego szczebla|
:Zaktualizuj CRM / data lake / powiadomienia push;
stop
@enduml
```

Wzorzec outbox gwarantuje, że zdarzenia zostaną opublikowane tylko wtedy, gdy zmiana w bazowej bazie danych zostanie utrwalona.

## 11. Przepływ samoobsługowy klienta

```plantuml
@startuml
|Klient|
start
:Otwórz aplikację mobilną lub webową;
:POST /v1/self/authentication;
:Widok kont / pożyczek;
if (Złożyć wniosek o produkt?) then (tak)
  :POST /v1/self/loans (lub /shareaccounts, /savingsaccounts);
  |Zaplecze bankowe|
  :Przegląd poprzez interfejsy API dla personelu;
else (przesłać pieniądze)
  :POST /v1/self/accounttransfers;
endif
:Opcjonalnie odpowiedz na ankiety;
stop
@enduml
```

## 12. Raportowanie i ujawnianie danych

```plantuml
@startuml
|Interesariusz|
start
:Wybierz raport\n(stretchy / Pentaho / MIX);
:GET /v1/reports;
:GET /v1/runreports/{name};
|System|
:Renderuj przez Stretchy / Pentaho;
:Opcjonalnie eksportuj do S3\n(fineract.report.export.s3);
stop
@enduml
```

Raporty Stretchy to szablony SQL przechowywane w `stretchy_report` i parametryzowane przez `stretchy_report_parameter`. Raporty MIX-Market mapują tagi taksonomii XBRL poprzez `mix_taxonomy_mapping`.

## 13. Onboarding najemcy (operacyjny)

```plantuml
@startuml
|Zespół platformy|
start
:Przygotuj bazę danych najemcy (tenant);
:Wstaw wiersz najemcy w tenant-store\n(zaszyfrowane dane uwierzytelniające);
:Zrestartuj Fineract\n(lub zastosuj migracje za pomocą\nprofilu liquibase-only);
|System|
:Uruchom listy zmian (changelogs) modułów\nwzględem nowego najemcy;
:Najemca gotowy;
|Operator bankowy|
:Wprowadź dane podstawowe (master data)\n(waluty, kody, oddziały, produkty);
stop
@enduml
```

Zobacz [`docs/technical/10-operational-runbook.md#adding-a-tenant`](../technical/10-operational-runbook.md#adding-a-tenant).

## Własność procesów *(wnioskowana)*

| Proces | Prawdopodobna rola właściciela |
| --- | --- |
| Onboarding klienta, KYC | Operacje bankowe / compliance. |
| Udzielanie i zatwierdzanie pożyczek | Dział kredytowy / doradcy kredytowi. |
| Codzienne COB | SRE platformy + back office. |
| Zamknięcie KG | Finanse. |
| Tworzenie rezerw | Ryzyko. |
| Integracja zdarzeń zewnętrznych | Integracja / platforma danych. |
| Onboarding najemcy | Administrator baz danych (DBA) platformy. |

> TODO (wymaga potwierdzenia od MŚP/SME): przypisz imiennych ekspertów (SME) (lub zespoły) do każdego procesu przed opublikowaniem tego dokumentu interesariuszom nietechnicznym.


---

← Poprzedni: [Przegląd produktu](02-product-overview.md) · ↑ [Indeks](../README.md) · Następny: [Słownik domenowy](04-domain-glossary.md) →
