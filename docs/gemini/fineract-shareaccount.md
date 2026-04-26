# Moduł Udziałów i Dywidend (Share Accounts)

[Powrót do dokumentacji głównej](README.md)

## Opis
Moduł Udziałów (dostępny fizycznie wewnątrz `fineract-provider/src/main/java/org/apache/fineract/portfolio/shareaccounts`) realizuje krytyczną funkcjonalność dla Spółdzielczych Kas Oszczędnościowo-Kredytowych (Credit Unions / SACCOs). W odróżnieniu od klasycznych banków komercyjnych, w instytucjach spółdzielczych klient nierzadko jest jednocześnie jej współwłaścicielem, nabywając "Udziały" (Shares). 

Moduł pozwala na konfigurowanie produktów udziałowych, emitowanie puli akcji/udziałów po danej cenie rynkowej, zakupywanie ich przez członków instytucji, a po zamknięciu roku obrotowego – na masowe i zautomatyzowane wyliczanie i księgowanie należnych dywidend (Zysków z udziałów) z powrotem na konta oszczędnościowe członków.

## Kluczowe komponenty biznesowe

| Komponent | Odpowiedzialność biznesowa i techniczna |
| :--- | :--- |
| **`ShareProduct`** | Zestaw zasad emisji: Całkowita liczba udziałów do wyemitowania na instytucję, cena nominalna (Nominal Price), mechanizm wyliczania dywidendy, okresy lock-up (zakaz sprzedaży) oraz minimalny/maksymalny wymóg udziałów na członka. |
| **`ShareAccount`** | Odpowiednik "Portfela Inwestycyjnego" danego klienta. Posiada powiązanie z `ShareProduct` i przechowuje rekordy zatwierdzonych, oczekujących (Pending) oraz sprzedanych/umorzonych transakcji udziałowych (`ShareAccountTransaction`). |
| **`ShareProductDividendPayOutDetails`** | Konfigurator Dywidendy: Zgłoszenie np. "Za rok 2023, dywidenda wyniesie 2.50 PLN na udział" i zlecenie masowej weryfikacji kto posiadał ile udziałów w wymaganym terminie i ile środków należy mu przelać. |
| **Post Dividends (Batch Job)** | Asynchroniczny krok Spring Batch (`PostDividentsForSharesTasklet`), który przechodzi przez dziesiątki tysięcy kont `ShareAccount`, zatwierdza dywidendę, tworzy `BusinessEvents` i wchodzi w interakcję z modułem `fineract-savings`, zasilając wyliczoną kwotą depozyty klientów. |

## Zależności i Księgowość

Udziały ściśle opierają się na fundamencie `fineract-accounting`. Moduł wysyła tam mapowania, gdyż emisja udziałów zasila bezpośrednio pozycję "Kapitał Własny" (Equity) na bilansie instytucji spółdzielczej (w przeciwieństwie do zwykłych oszczędności traktowanych jako Zobowiązania/Liabilities).

Moduł ten znajduje się w core ze względu na głębokie powiązania relacyjne encji z modelem Klienta i Oszczędności.
