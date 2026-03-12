# Moduł Raportowania MIX Market / XBRL (fineract-mix)

[Powrót do dokumentacji głównej](README.md)

## Opis
Moduł `fineract-mix` (Microfinance Information Exchange) to wysoce specjalistyczny podmoduł służący integracji platformy Apache Fineract ze znormalizowanymi standardami raportowania rynków mikrofinansowych. Rozwiązanie to odpowiada za tworzenie paczek raportowych w formacie **XBRL** (eXtensible Business Reporting Language).

Wielu dostawców oprogramowania i instytucji pozarządowych w sektorze mikrofinansów zgłasza swoje wyniki do centralnej ogólnoświatowej bazy danych ujednoliconego słownika raportowego (Taxonomy). Moduł wyciąga dane zagregowane z księgowości (fineract-accounting) i portfela (fineract-loan/savings) - np. sumy aktywnych kredytobiorców, wolumenu pożyczkowego na portfelach ryzykownych (PAR - Portfolio at Risk) - a następnie taguje je za pomocą definicji przypisanych z taksonomii MIX, konwertując wyjściowo na spójny biznesowo dokument `XML/XBRL`.

## Kluczowe komponenty techniczne

*   **`MixTaxonomy` / `MixMapping`**: Encje pozwalające zmapować istniejące wewnętrzne konta w planie kont Księgi Głównej (GL Account) z wymaganymi na zewnątrz tagami księgowymi dla raportów (np. Mapowanie Konta "Należności z kredytów mieszkaniowych" do tagu MIX: `GrossLoanPortfolio`).
*   **XBRL Builder**: Generator silnika szablonów produkujący kompletny, technicznie poprawny znacznikami `XML` raport finansowy nadający się do bezpośredniego audytu lub wysyłki na platformy raportowe agencji ratingowych.
