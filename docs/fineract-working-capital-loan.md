# Moduł Kredytu Obrotowego (fineract-working-capital-loan)

[Powrót do dokumentacji głównej](README.md)

## Opis
Moduł `fineract-working-capital-loan` (Kredyty Obrotowe dla MŚP i Korporacji) implementuje obsługę elastycznych pożyczek celowych i linii kredytowych przeznaczonych dla wspierania płynności w przedsiębiorstwach. Kredyty tego typu znacznie odbiegają od statycznych umów ratalnych dla konsumentów. 

W przeciwieństwie do standardowego kredytu (`fineract-loan`), kredyt obrotowy to często rodzaj konta otwartego z określonym na umowie maksymalnym limitem kredytowym, gdzie klient (firma) dokonuje spłat i wypłat wedle bieżących potrzeb wynikających z luk finansowych i operacyjnych (ang. Working Capital). Odsetki i kary naliczane są na podstawie czasu trwania wypłaconego kapitału bez z góry narzuconej, jedynej słusznej formy ratalnej.

## Kluczowe mechanizmy

| Funkcja | Opis implementacji |
| :--- | :--- |
| **`WorkingCapitalLoanProduct`** | Zestaw definicji z predefiniowanymi parametrami dla obrotówki. Obejmuje inne flagi wymagalności aniżeli produkt standardowy (np. `LoanProduct`). Odmienna jest metoda rozliczeń z Księgą Główną i kontami tranzytowymi (GL Accounts). |
| **Wycofania (Drawdowns)** | Klient nie wypłaca pożyczki jednorazowo przy zawarciu umowy, lecz zleca "Tłoczenie/Tranche" (ang. Drawdown) pod limit. Jeśli limit to 50 000 zł, a klient zaciąga obecnie 10 000 zł, dostępne staje się 40 000 zł. Moduł pilnuje bilansu linii kredytowej w czasie rzeczywistym. |
| **Modyfikacja Harmonogramu** | Wycofywanie kapitału bądź jego dołożenie w środku cyklu zmienia kapitał resztkowy. Moduł współpracuje tu często w oparciu o silniki wyliczania stawek dziennych/średnio-dziennych (Daily Average Balance). |

## Zależności
Dzieli wspólną logikę obsługi komend CQRS i jest skrupulatnie wpięty pod interfejsy z `fineract-loan`, umożliwiając systemom księgowym bezkolizyjne odbieranie z tego modułu zdarzeń Business Events o spłatach i zaciągnięciu nowych transzy kredytu.
