# Moduł Inwestycyjny i P2P (fineract-investor)

[Powrót do dokumentacji głównej](README.md)

## Opis
Moduł `fineract-investor` realizuje w Apache Fineract obsługę rynku Private-to-Private (P2P Lending), Peer-to-Peer bądź szerszego pozyskiwania kapitału zewnętrznego zwanego sekurytyzacją portfela. Głównym założeniem modułu jest to, że MFI lub Bank udzielający bezpośrednio pożyczki (Loan), może nie posiadać własnego, wolnego kapitału.

Aby zmniejszyć ryzyko bądź pozyskać natychmiastową gotówkę do udzielania nowych kredytów, instytucja może połączyć inwestora (Investor - np. firmę zewnętrzną, czy majętnego klienta) i odsprzedać mu aktywną umowę pożyczkową (częściowo lub w całości). Jeśli spłaty klienta w pożyczce wpłyną poprawnie do Fineractu, w tle moduł inwestora przeksięguje te wpłaty na konto inwestora, pomniejszając je często o procentową prowizję na rzecz Banku za administrowanie obsługi pożyczki.

## Kluczowe komponenty

| Komponent | Odpowiedzialność biznesowa i techniczna |
| :--- | :--- |
| **`Client` jako `Investor`** | W Fineract, "Inwestor" nie jest całkowicie nową encją biznesową na zewnątrz. Jest to tradycyjny profil `Client` (osoba bądź fundusz powierniczy), co ułatwia przejście KYC/AML. Powstają jednak na nim dedykowane parametry. |
| **`AccountTransfer` i Mapowanie Kredytów** | W Fineract zakupienie portfela polega na przelaniu fizycznych pieniędzy pomiędzy zasilającym depozytem (`SavingsAccount` inwesotra), a portfelem kredytowym dłużnika. Moduł `fineract-investor` orkiestruje te wywołania transferowe. |

## Architektura modułu

Architektura nie ingeruje mocno w samą modyfikację encji kredytowych. Utrzymuje logiczne tabele z powiązaniami kto tak naprawdę sfinansował daną pulę (portfolio) pożyczkową, i do kogo należą zyski odsetkowe z bieżącej spłaty.

```plantuml
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml
title Model C4 - Zależności modułu fineract-investor

Component(investor_api, "Investor REST API", "Spring Web", "Służy udostępnianiu pożyczek dla zewnętrznych portali crowdfundingowych P2P")
Component(portfolio_mapper, "InvestorPortfolioMapper", "Service", "Rejestruje jaka pożyczka została przypisana (odsprzedana) jakiej frakcji zysków Inwestora")
Component(loan_module, "fineract-loan", "Moduł Pożyczek", "Wysyła zdarzenie po każdorazowej spłacie Klienta (MakeRepaymentEvent)")
Component(savings_module, "fineract-savings", "Moduł Oszczędności", "Odbiera polecenia z InvestorModule aby zaksięgować zyski z powrotem na rachunek depozytowy Inwestora")

Rel(loan_module, portfolio_mapper, "Zgłasza: Otrzymano 100 zł z raty klienta A")
Rel(portfolio_mapper, portfolio_mapper, "Wykrywa, że 50% raty należy do Inwestora B, a 50% do Banku")
Rel(portfolio_mapper, savings_module, "Wysyła 50 zł na podpięty rachunek oszczędnościowy Inwestora B (Zysk dla inwestora)")
Rel(investor_api, portfolio_mapper, "Mapowanie nowej pożyczki z funduszem inwestora")

@enduml
```

## Przepływ danych (Zasada Peer-To-Peer)

```plantuml
@startuml
title Sekwencja - Dystrybucja funduszy w ramach P2P Lending ze zwrotem z rat

actor Użytkownik (Pożyczkobiorca) as client
actor Inwestor as investor
participant "Fineract Loan (Kredyt)" as loan
participant "Fineract Investor (Logika P2P)" as p2p
participant "Fineract Savings (Konto Inwestora)" as inv_account

investor -> inv_account: Inwestor wpłaca 5000 PLN kapitału początkowego (Deposit) na konto Savings
inv_account --> investor: Saldo = 5000 PLN

client -> loan: Prośba o kredyt 5000 PLN
loan -> p2p: Inicjacja finansowania pożyczki
p2p -> inv_account: Transfer 5000 PLN ze środków inwestora by wypłacić kwotę klientowi
inv_account --> investor: Saldo = 0 PLN

note over client, loan
 Miesiąc później...
end note

client -> loan: Klient wpłaca ratę w wys. 500 PLN (kapitał + odsetki)
loan -> loan: Aktualizacja Salda Kredytu i rejestracja Transakcji
loan -> p2p: LoanTransactionMakeRepaymentPostBusinessEvent (Zdarzenie spłaty 500 PLN)

p2p -> p2p: Kalkulacja i cięcie zwrotu (np. 10 PLN za obsługę dla banku, 490 PLN zysk Inwestora)
p2p -> inv_account: Transfer (Deposit) na kwotę 490 PLN do inwestora
inv_account --> investor: Nowe Saldo Oszczędności = 490 PLN (Kapitał wraca do inwestora z odsetkami)
@enduml
```

## Zarządzanie stanem i baza danych

System ten buduje dodatkowy wskaźnik alokacji wpłat w bazie:

*   Tabele łącznikowe i śledzące (na przykład w starszych standardach lub zewnętrznych customowych projektach implementowane jako `m_account_transfer_transaction` w powiązaniu z grupami klienckimi i flagami sekurytyzacji u Klienta) trzymają informacje o udziale funduszu, z którego kredyt został przelany (rozdzielenie źródła w `m_loan` - `fund_id`). Pozwala to oddzielić pożyczki wyemitowane z bezpośredniego budżetu MFI od tych z puli zewnętrznych "Funduszy" Inwestorskich.
