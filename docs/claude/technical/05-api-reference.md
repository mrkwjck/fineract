# Dokumentacja API

<details>
<summary><strong>Skocz do dowolnego dokumentu</strong></summary>

**[Indeks](../README.md)**

**Biznes:** [01 Podsumowanie wykonawcze](../business/01-executive-summary.md) · [02 Przegląd produktu](../business/02-product-overview.md) · [03 Procesy biznesowe](../business/03-business-processes.md) · [04 Słownik domenowy](../business/04-domain-glossary.md) · [05 Reguły biznesowe](../business/05-business-rules.md) · [06 Integracje i interesariusze](../business/06-integrations-and-stakeholders.md) · [07 Ryzyka i luki](../business/07-risks-and-gaps.md)

**Techniczne:** [01 Przegląd architektury](01-architecture-overview.md) · [02 Stos technologiczny](02-tech-stack.md) · [03 Mapa repozytorium](03-repository-map.md) · [04 Model danych](04-data-model.md) · **05 Dokumentacja API** · [06 Przepływy uruchomieniowe](06-runtime-flows.md) · [07 Infrastruktura i wdrożenie](07-infrastructure-and-deployment.md) · [08 Konfiguracja i flagi funkcji](08-configuration-and-feature-flags.md) · [09 Model bezpieczeństwa](09-security-model.md) · [10 Podręcznik operacyjny](10-operational-runbook.md) · [11 Strategia testowania](11-testing-strategy.md) · [12 Dziennik decyzji](12-decision-log.md)

</details>

> Mapa punktów końcowych REST, uwierzytelniania i obsługi błędów.

Pełna lista API jest generowana z adnotacji JAX-RS i udostępniana jako plik JSON OpenAPI („Swagger”). Ten dokument indeksuje interfejs; szczegóły dotyczące parametrów można znaleźć uruchamiając serwer i czytając wygenerowany plik `swagger.json` (ścieżka wyjściowa skonfigurowana w `build.gradle:162`: `$rootDir/fineract-provider/build/resources/main/static/fineract.json`).

## Punkt montowania

Katalog główny aplikacji JAX-RS: `/api` (`org.apache.fineract.infrastructure.core.jersey.JerseyConfig` z `@ApplicationPath("/api")`). Wszystkie poniższe punkty końcowe mają prefiks `/api`.

## Wersjonowanie

Obecnie używana jest jedna główna wersja (`/v1/...`). Kompatybilność wsteczna jest wymuszana przez `swagger-brake` (`.github/workflows/verify-api-backward-compatibility.yml`).

## Uwierzytelnianie i wymagane nagłówki

| Nagłówek | Cel |
| --- | --- |
| `Authorization: Basic <base64>` | Domyślne; włączone przez `FINERACT_SECURITY_BASICAUTH_ENABLED=true`. |
| `Authorization: Bearer <jwt>` | Gdy `FINERACT_SECURITY_OAUTH_ENABLED=true`. |
| `Fineract-Platform-TenantId: <id>` | Wymagane przy każdym żądaniu w celu rozwiązania bazy danych najemcy. *(wywnioskowane z `TenantAwareBasicAuthenticationFilter` i `application.properties:50` `fineract.tenant.identifier=default`)* |
| `Idempotency-Key: <uuid>` | Opcjonalne, API zapisu; konfigurowalna nazwa nagłówka `fineract.idempotency-key-header-name` (`application.properties:163`). |
| `X-Correlation-ID` | Opcjonalne; włączone przez `fineract.correlation.enabled` (`application.properties:76-77`). |
| Nagłówek tokena dwuskładnikowego | Gdy 2FA jest włączone (`fineract.security.2fa.enabled`). Patrz `09-security-model.md`. |

## Katalog zasobów

### Uwierzytelnianie

| Ścieżka | Zasób |
| --- | --- |
| `/v1/authentication` | Tworzy sesję dla uwierzytelniania podstawowego (login + najemca). |
| `/v1/twofactor`, `/v1/twofactor/configure` | Wydawanie / konfiguracja OTP. |
| `/v1/userdetails`, `/v1/users`, `/v1/permissions`, `/v1/roles` | Administracja użytkownikami i uprawnieniami. |

### Klienci, grupy, organizacja

| Ścieżka | Cel |
| --- | --- |
| `/v1/clients`, `/v1/client`, `/v1/clients/{clientId}/...` | CRUD klienta (odbiorcy); zasoby podrzędne dla `charges`, `collaterals`, `familymembers`, `identifiers`, `addresses`, `transactions`. |
| `/v1/groups`, `/v1/centers`, `/v1/grouplevels` | Hierarchie klientów. |
| `/v1/offices`, `/v1/staff`, `/v1/holidays`, `/v1/workingdays`, `/v1/officetransactions` | Sieć oddziałów i kalendarz. |
| `/v1/tellers`, `/v1/cashiers`, `/v1/cashiersjournal` | Zarządzanie gotówką przez kasjerów. |
| `/v1/funds`, `/v1/codes`, `/v1/codes/{codeId}/codevalues`, `/v1/currencies`, `/v1/paymenttypes` | Dane podstawowe. |

### Pożyczki

| Ścieżka | Cel |
| --- | --- |
| `/v1/loans`, `/v1/loans/{loanId}/...` | Cykl życia konta pożyczkowego: tworzenie, zatwierdzanie, wypłata, spłata, umorzenie, zamknięcie itp. |
| `/v1/loans/external-id/{loanExternalId}/...` | To samo co powyżej, ale adresowane przez identyfikator zewnętrzny. |
| `/v1/loans/{loanId}/charges`, `/v1/loans/{loanId}/disbursements`, `/v1/loans/{loanId}/guarantors`, `/v1/loans/{loanId}/postdatedchecks`, `/v1/loans/{loanId}/schedule` | Zasoby podrzędne pożyczki. |
| `/v1/loanproducts`, `/v1/loanproducts/{productId}/productmix` | Konfiguracja produktów pożyczkowych i miksu produktów. |
| `/v1/loans/loanreassignment`, `/v1/loans/at-date` | Zmiana przypisania opiekuna pożyczki, widok na dany moment. |
| `/v1/rescheduleloans` | Wnioski / zatwierdzenia zmiany harmonogramu. |
| `/v1/loan-collateral-management`, `/v1/collateral-management` | Administracja zabezpieczeniami. |
| `/v1/loan-originators` | Rejestr opiekunów pożyczek. |
| `/v1/working-capital-loan-products`, `/v1/working-capital-loans` | Specjalizacja pożyczek na kapitał obrotowy. |
| `/v1/delinquency` | Konfiguracja koszyków zaległości. |
| `/v1/runaccruals` | Uruchomienie naliczania memoriałowego. |

### Oszczędności, depozyty, udziały

| Ścieżka | Cel |
| --- | --- |
| `/v1/savingsaccounts`, `/v1/savingsaccounts/{savingsAccountId}/charges`, `/v1/savingsaccounts/{savingsId}/transactions`, `/v1/savingsaccounts/{savingsId}/onholdtransactions` | Zwykłe oszczędności. |
| `/v1/savingsproducts` | Konfiguracja produktów oszczędnościowych. |
| `/v1/fixeddepositaccounts`, `/v1/fixeddepositaccounts/{fixedDepositAccountId}/transactions`, `/v1/fixeddepositproducts` | Konta lokat terminowych. |
| `/v1/recurringdepositaccounts`, `/v1/recurringdepositaccounts/{recurringDepositAccountId}/transactions`, `/v1/recurringdepositproducts` | Konta depozytów cyklicznych. |
| `/v1/interestratecharts`, `/v1/interestratecharts/{chartId}/chartslabs` | Tabele stóp procentowych. |
| `/v1/products/{type}` | Ogólny router produktów (pożyczki/oszczędności itp.). |
| `/v1/shareproduct/{productId}/dividend` | Dywidendy z udziałów. |
| `/v1/accounts/{type}` | Ogólny widok konta. |

### Księgowość

| Ścieżka | Cel |
| --- | --- |
| `/v1/glaccounts` | Plan kont. |
| `/v1/glclosures` | Zamknięcia okresów księgi głównej. |
| `/v1/journalentries`, `/v1/journalentries/openingbalance` | Wpisy do dziennika (ręczne + bilanse otwarcia). |
| `/v1/financialactivityaccounts` | Mapowania aktywności finansowej na konta KG. |
| `/v1/accountingrules` | Silnik reguł księgowych. |
| `/v1/provisioningcategory`, `/v1/provisioningcriteria`, `/v1/provisioningentries` | Tworzenie rezerw na straty pożyczkowe. |

### Opłaty, podatki, stawki

| Ścieżka | Cel |
| --- | --- |
| `/v1/charges` | Opłaty (prowizje / kary). |
| `/v1/taxes/component`, `/v1/taxes/group` | Definicje komponentów / grup podatkowych. |
| `/v1/floatingrates` | Definicje zmiennych stóp procentowych. |
| `/v1/rates` | Plany stawek. |

### Przelewy i SI

| Ścieżka | Cel |
| --- | --- |
| `/v1/accounttransfers` | Przelewy międzykontowe. |
| `/v1/standinginstructions`, `/v1/standinginstructionrunhistory` | Zlecenia stałe i historia ich uruchomień. |

### Operacyjne / administracyjne

| Ścieżka | Cel |
| --- | --- |
| `/v1/jobs`, `/v1/jobs/{jobId}/jobrunhistory` | Zadania harmonogramu Quartz (odczyt i ręczne uruchamianie). |
| `/v1/scheduler` | Sterowanie harmonogramem (start/stop). |
| `/v1/configurations`, `/v1/configurations/name/{configName}` | Globalne wpisy konfiguracyjne. |
| `/v1/caches` | Zarządzanie pamięcią podręczną (Ehcache). |
| `/v1/businessdate` | Zarządzanie „datą biznesową” systemu (oddzielna od czasu rzeczywistego, używana do testów COB). |
| `/v1/audits`, `/v1/makercheckers` | Ścieżka audytu i kolejka zatwierdzania „na cztery oczy”. |
| `/v1/instance-mode` | Przełączanie trybów odczyt/zapis/menedżer/pracownik w czasie rzeczywistym. |
| `/v1/echo` | Echo stanu (health check). |
| `/v1/externalevents/configuration` | Przełączanie funkcji zdarzeń zewnętrznych (na typ zdarzenia). |
| `/v1/externalservice` | Rejestr usług zewnętrznych (np. poświadczenia bramki SMS/e-mail). |
| `/v1/imports` | Zadania masowego importu. |
| `/v1/templates`, `/v1/template`, `/v1/templates/{resourceId}` | Magazyn szablonów Mustache (powiadomienia, haki). |
| `/v1/datatables`, `/v1/datatables/{datatable}/...` | Tabele rozszerzeń zdefiniowane przez użytkownika. |
| `/v1/entityDatatableChecks` | Reguły walidacji tabel danych. |
| `/v1/entitytoentitymapping`, `/v1/fieldconfiguration/{entity}` | Relacje encji i konfiguracja na poziomie pól. |
| `/v1/hooks` | Subskrypcje webhooków (`m_hook_configuration`). |
| `/v1/notifications` | Preferencje powiadomień i skrzynka odbiorcza. |
| `/v1/sms`, `/v1/smscampaigns`, `/v1/email`, `/v1/email/campaign`, `/v1/email/configuration` | Wychodzące wiadomości SMS / e-mail. |
| `/v1/reports`, `/v1/runreports`, `/v1/availableExports/{reportName}` | Raporty Stretchy / Pentaho. |
| `/v1/adhocquery` | Zapytania SQL ad-hoc (tylko administrator). |
| `/v1/search` | Wyszukiwanie międzyencyjne. |

### Zadania wsadowe i wewnętrzne

| Ścieżka | Cel |
| --- | --- |
| `/v1/batches` | Batch API: łączenie wielu żądań podrzędnych w ramach jednej transakcji (`BatchApiResource`). |
| `/v1/internal/cob` | Wewnętrzne punkty końcowe sterowania COB (`LoanCOBCatchUpApiResource`, `WorkingCapitalLoanCOBCatchUpApiResource`). Zawiera `catch-up`, `is-catch-up-running`, `oldest-cob-closed`, `fast-forward-cob-date-of-loan/{loanId}`. |
| `/v1/internal/loan`, `/v1/internal/loans`, `/v1/internal/savingsaccounts`, `/v1/internal/client`, `/v1/internal/configurations`, `/v1/internal/externalevents` | Wewnętrzne narzędzia pomocnicze (przeznaczone dla operacji, zablokowane za jawnymi rolami). |

### Samoobsługa klienta

Rodzina `/v1/self/*` to zestaw funkcji **skierowanych do klienta** (Mifos web UI / mobile). Zasoby odzwierciedlają API personelu, ale stosują autoryzację samoobsługową (`SelfServiceUserAuthorizationManager`).

| Ścieżka |
| --- |
| `/v1/self/authentication` |
| `/v1/self/clients` |
| `/v1/self/loans`, `/v1/self/loanproducts` |
| `/v1/self/savingsaccounts`, `/v1/self/savingsproducts` |
| `/v1/self/shareaccounts`, `/v1/self/products/share` |
| `/v1/self/accounttransfers` |
| `/v1/self/beneficiaries/tpt` |
| `/v1/self/pockets` |
| `/v1/self/registration` |
| `/v1/self/runreports` |
| `/v1/self/surveys`, `/v1/self/surveys/scorecards` |
| `/v1/self/user`, `/v1/self/userdetails`, `/v1/self/device/registration` |

### Interoperacyjność Mojaloop

| Ścieżka | Cel |
| --- | --- |
| `/v1/interoperation` | Interoperacyjność między dostawcami usług finansowych (FSP): `parties/{idType}/{idValue}`, `quotes`, `transfers`, `requests`, `transactions/{transactionCode}/quotes/{quoteCode}` itp. *(wywnioskowane z `org.apache.fineract.interoperation`)* |

## Model błędów

Fineract używa kodów statusu HTTP oraz otoczki JSON tworzonej przez mapery wyjątków JAX-RS. Otoczka zawiera:

- `userMessageGlobalisationCode` — klucz i18n odwołujący się do `messages_*.properties`.
- `developerMessage` — komunikat techniczny (angielski).
- `parameterName` i `args` — powiązane z polem powodującym błąd, tam gdzie ma to zastosowanie.
- `httpStatusCode` — powtórzone dla wygody klienta.

Mapery wyjątków znajdują się w `org.apache.fineract.infrastructure.core.exceptionmapper` (oraz pakietach `exceptionmapper` specyficznych dla modułów, takich jak `org.apache.fineract.cob.exceptionmapper`).

> TODO (wymaga potwierdzenia eksperta): opublikować tabelę referencyjną kodów błędów po uzgodnieniu kluczy `messages.properties` między modułami.

## Generowanie OpenAPI

- Kompilacja dostawcy uruchamia `swagger-jaxrs2-jakarta` w celu wygenerowania `fineract.json`.
- Dwa zestawy SDK klienta są generowane automatycznie: `fineract-client` i `fineract-client-feign` przez `org.openapi.generator` (`build.gradle:125`).
- Kolekcja Postman społeczności Mifos jest tworzona na podstawie tego pliku `fineract.json` (poza zakresem tego repozytorium).

## Szczegóły Batch API

`/v1/batches` (`BatchApiResource`) pozwala wywołującemu połączyć listę żądań podrzędnych, z których każde posiada `relativeUrl`, `method`, `headers`, `body`, opcjonalne `reference`, w jedną transakcję. Żądania podrzędne mogą odwoływać się do wyników wcześniejszych żądań za pomocą odniesień ścieżki JSON `$.`. Jest to kanoniczny wzorzec dla przepływów pracy między encjami (utwórz klienta + otwórz konto oszczędnościowe + złóż wniosek o pożyczkę w jednym wywołaniu).

## Idempotentność

Jeśli skonfigurowany nagłówek (domyślnie `Idempotency-Key`) jest obecny w wywołaniu zapisu, serwer przechowuje hash żądania oraz treść pierwszej odpowiedzi i odtwarza ją przy każdej próbie ponowienia w oknie retencji. Magazyn poleceń / mechanizm maker-checker stanowi podstawę tego zachowania.


---

← Poprzedni: [Model danych](04-data-model.md) · ↑ [Indeks](../README.md) · Następny: [Przepływy uruchomieniowe](06-runtime-flows.md) →
