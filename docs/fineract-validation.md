# Moduł Walidacji Scentralizowanej (fineract-validation)

[Powrót do dokumentacji głównej](README.md)

## Opis
Moduł `fineract-validation` dostarcza ujednolicony silnik do przeprowadzania złożonych walidacji obiektów w Apache Fineract. O ile proste weryfikacje (np. długość imienia klienta nie przekraczająca 255 znaków) dokonywane są na poziomie samych klas modelu lub w warstwie API (dzięki Java Bean Validation / Hibernate Validator), o tyle bankowość generuje zasady obejmujące kilka krzyżowych domen jednocześnie.

Ten moduł odpowiada za implementację sprawdzania kondycji danych pomiędzy różnymi usługami. Jest to wysoce reużywalna "biblioteka reguł biznesowych", zapobiegająca duplikowaniu logiki if-else we wszystkich modułach z osobna.

## Przykłady Logiki Biznesowej (Walidacji)

*   **Złożone Daty Zapadalności:** Upewnienie się, czy zadeklarowana we wniosku kredytowym Data Wypłaty (Disbursement Date) nie występuje chronologicznie przed oficjalną Datą Dołączenia (Activation Date) klienta do Banku, uwzględniając jednocześnie strefy czasowe odpowiednie dla danego dzierżawcy (Multi-Tenancy context).
*   **Limity Kwotowe i Koszykowe:** Weryfikacja reguł poprawności (Sanity Checks) w procesie tworzenia `LoanProduct` - np. minimalny przedział pożyczki nie może być wyższy niż maksymalny dopuszczalny kapitał (Min Principal > Max Principal = Błąd).
*   **Integralność Przejść Stanów (State Machine Integrity):** Centralne komponenty upewniające się, że nikt przez interfejs API nie odrzuci (Reject) kredytu, który został już wcześniej z sukcesem zamknięty (Closed) lub odpisany w straty (Written-Off).

Moduł ten przy napotkaniu naruszenia reguł, współpracuje z `fineract-core` w celu złączenia ich (Aggregation) i natychmiastowego zgłoszenia wyjątku `PlatformApiDataValidationException` na zewnątrz z listą konkretnych kodów błędu (Error Codes).
