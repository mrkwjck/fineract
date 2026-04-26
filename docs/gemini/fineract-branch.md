# Moduł Ograniczeń i Limitów Oddziałowych (fineract-branch)

[Powrót do dokumentacji głównej](README.md)

## Opis
Moduł `fineract-branch` (wcześniej w architekturze określane jako Cross-Branch, Teller/Branch management) poszerza standardowe funkcje hierarchii organizacyjnej (zawartej w domenie `organisation/office` w `fineract-provider`).
W zaawansowanych instalacjach bankowych lub rozproszonych geograficznie mikrofinansach (MFI), pojawia się wymóg kontroli przepływu gotówki i transferów pomiędzy poszczególnymi Oddziałami (Branches/Offices), bądź limitów obrotu na kasach (Tellers).

Głównym zadaniem tego sub-modułu jest zabezpieczanie procedur operacyjnych na poziomie oddziału banku - upewnienie się, że dany oddział posiada wystarczającą ilość przydzielonej mu gotówki do wyemitowania pożyczki oraz raportowanie per-Oddział.

## Kluczowe komponenty biznesowe

| Komponent | Odpowiedzialność biznesowa i techniczna |
| :--- | :--- |
| **Gotówka Oddziału (Branch Cash)** | Wyodrębnienie transakcji księgowych pokazujące przesunięcia bilansowe między centralą, a oddziałami (np. samochód opancerzony dostarczający gotówkę do sejfu placówki). |
| **Limity Kredytowe Placówki** | Funkcjonalność ograniczająca np. "Oddział we Wrocławiu może w tym miesiącu wydać na pożyczki maksymalnie 5 milionów PLN". Jeżeli kwota pożyczki przekroczy ten limit, walidacja uniemożliwi jej wypłatę (Disbursement). |
| **`BranchBuilder` / `BranchData`** | Abstrakcje ułatwiające zestawienia raportowe (Stretchy Reports) wyliczające "Performance" konkretnych oddziałów na tle regionu. |

## Zależności i Architektura

*   Zależy ściśle od encji `m_office` przechowywanej w `fineract-provider`.
*   Wywołuje Business Events do `fineract-accounting`, upewniając się, że przesunięcia środków finansowych między kasjerem z jednego oddziału na rzecz spłaty raty u klienta z innego oddziału banku zostaną prawidłowo zrekompensowane na kontach tranzytowych.
