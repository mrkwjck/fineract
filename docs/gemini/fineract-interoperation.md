# Moduł Interoperacyjności (Interoperation / GSMA / Mojaloop)

[Powrót do dokumentacji głównej](README.md)

## Opis
Moduł Interoperacyjności (dostępny z poziomu rdzenia w `fineract-provider/.../interoperation`) odpowiada za umożliwienie płynnej komunikacji i transferu środków finansowych między Fineractem, a ogólnoświatowymi lub krajowymi szynami płatności mobilnych i portfeli cyfrowych (Mobile Money). 

Wdrożenia w krajach rozwijających się często polegają na integracji platformy Fineract jako dostawcy systemu Core Banking, dla którego wejściem (kanałem płatności) są platformy telekomunikacyjne. Moduł ten posiada pre-definiowane, certyfikowane kontrolery i modele danych zgodne z rygorystycznymi standardami API stowarzyszenia **GSMA** (Global System for Mobile Communications) oraz open-source'owym hubem płatności **Mojaloop**. 

## Kluczowe komponenty techniczne

| Moduł Logiczny | Odpowiedzialność techniczna i biznesowa |
| :--- | :--- |
| **Identyfikatory (Interoperation Identifiers)** | W świecie Mobile Money klient nie używa numeru rachunku bankowego (IBAN). Jego kluczem głównym jest najczęściej numer MSISDN (Numer Telefonu Komórkowego). Ten moduł rejestruje powiązanie ("Rezerwuje Alias") numeru telefonu `+48 123 456 789` z konkretnym systemowym kontem `SavingsAccount` po stronie Fineractu. |
| **Kalkulacja Kosztów Transferu (Quotes)** | Zgodnie z protokołem Mojaloop, wysłanie pieniędzy to proces dwuetapowy. Etap pierwszy to zapytanie (Quote): "Ile wyniesie prowizja, jeśli wyślę 50 USD z Fineracta do sieci MTN?". Moduł ten odpowiada za obliczenie prowizji w czasie rzeczywistym bazując na `fineract-charge` bez potrącania jeszcze salda. |
| **Realizacja Transferu (Transfers)** | Etap drugi. Asynchroniczne potwierdzenie (Commit) lub odrzucenie (Rollback) środków u operatorów. Jeśli MTN zgłosi awarię, Fineract zwalnia blokadę kwoty i proces interoperation kończy się statusem FAIL. |
| **Zarządzanie Stanem Transakcji (`InteropActionState`)** | Wbudowana maszyna stanów, pozwalająca Fineractowi odrzucać nieprawidłowe przelewy (np. ktoś próbuje zacommitować przelew, który nie posiadał etapu Quote). |

## Zależności
Z racji wpięcia go pod API kompatybilne z firmami telekomunikacyjnymi, moduł ten udostępnia końcówki REST z prefiksami specyficznymi dla zewnętrznych usług. Mocno polega na module `fineract-savings` (wszelkie przelewy to de facto Debit/Credit na kontach oszczędnościowych lub wirtualnych portfelach depozytowych Fineractu).
