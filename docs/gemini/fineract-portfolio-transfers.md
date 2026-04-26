# Zlecenia Stałe, Transfery i Arkusze Inkasowe (Portfolio Transfers / Collection Sheets)

[Powrót do dokumentacji głównej](README.md)

## Opis
Katalogi i mechanizmy ukryte w module głównym (`fineract-provider/src/main/java/org/apache/fineract/portfolio/...` m.in. `transfer`, `account`, `collectionsheet`, `meeting`) obejmują jedne z najbardziej skomplikowanych operacji ułatwiających zautomatyzowany przepływ pieniędzy w banku oraz unikalne dla Fineracta operacje zbiorcze ułatwiające pracę ankieterów mikro-finansowych w terenie.

## Kluczowe mechanizmy

### 1. Account Transfers & Standing Instructions (Zlecenia Stałe)
Zamiast wymagać od klienta comiesięcznej wizyty w placówce celem spłacenia kredytu gotówką, ten pod-moduł umożliwia tworzenie Transferów Wewnętrznych (Account Transfers).
*   **Transfer:** Pozwala przenieść środki pomiędzy dwoma dowolnymi rachunkami (np. z Savings Account na Loan Account) w obrębie tego samego banku, z pominięciem systemu międzybankowego.
*   **Standing Instructions (Zlecenia Stałe):** Klient ustawia harmonogram (np. 10. dzień każdego miesiąca), a Job COB (Close Of Business) w tle automatycznie wyzwala transfer kwoty raty z jego konta oszczędnościowego na konto pożyczki (Automatic Loan Repayment). System obsługuje również zasady traktowania porażek (np. brak wystarczających środków -> ponów jutro).

### 2. Collection Sheets & Meetings (Arkusze Inkasowe)
Szczególnie w instalacjach Fineract poza rynkiem europejskim (Ameryka Południowa, Azja, Afryka) stosuje się model mikrofinansowy "JLG" (Joint Liability Groups). Oficer kredytowy wyrusza do wioski na **Spotkanie (Meeting)**.
*   Większość klientów na takim spotkaniu wpłaca małe raty gotówkowe (np. 5 dolarów). Zamiast wklikiwać 100 osobnych transakcji w systemie, Oficer generuje tzw. **Collection Sheet (Arkusz Inkasowy)**.
*   Arkusz ten w jednym widoku (i wywołaniu API JSON) przyjmuje i masowo rozlicza wpłaty gotówkowe setek klientów przypisanych do danej grupy we wskazanej dacie, drastycznie przyspieszając pracę oddziałów terenowych.

## Architektura i Zależności
Oba moduły spinają ze sobą domeny Portfela (Klienta, Pożyczek i Oszczędności).
- `AccountTransfer` generuje podwójny ruch modyfikujący - Debit na `SavingsAccountTransaction` oraz Credit jako Repayment na `LoanTransaction`. Transakcja musi zachować spójność kwasową (ACID) - jeśli pożyczka zwróci błąd w momencie wpłaty, środki z konta oszczędnościowego muszą zostać natychmiast wycofane (Rollback).
