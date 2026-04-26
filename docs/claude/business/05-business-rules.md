# Reguły biznesowe

<details>
<summary><strong>Przejdź do dowolnego dokumentu</strong></summary>

**[Indeks](../README.md)**

**Biznesowe:** [01 Podsumowanie menedżerskie](01-executive-summary.md) · [02 Przegląd produktu](02-product-overview.md) · [03 Procesy biznesowe](03-business-processes.md) · [04 Słownik domenowy](04-domain-glossary.md) · **05 Reguły biznesowe** · [06 Integracje i interesariusze](06-integrations-and-stakeholders.md) · [07 Ryzyka i luki](07-risks-and-gaps.md)

**Techniczne:** [01 Opis architektury](../technical/01-architecture-overview.md) · [02 Stos technologiczny](../technical/02-tech-stack.md) · [03 Mapa repozytorium](../technical/03-repository-map.md) · [04 Model danych](../technical/04-data-model.md) · [05 Dokumentacja API](../technical/05-api-reference.md) · [06 Przepływy uruchomieniowe](../technical/06-runtime-flows.md) · [07 Infrastruktura i wdrożenie](../technical/07-infrastructure-and-deployment.md) · [08 Konfiguracja i flagi funkcji](../technical/08-configuration-and-feature-flags.md) · [09 Model bezpieczeństwa](../technical/09-security-model.md) · [10 Podręcznik operacyjny](../technical/10-operational-runbook.md) · [11 Strategia testowania](../technical/11-testing-strategy.md) · [12 Rejestr decyzji](../technical/12-decision-log.md)

</details>

> Wyliczone reguły wraz z odniesieniami do kodu. Każda reguła jest oparta na właściwości, tabeli, migracji, kroku biznesowym lub zasobie API. Tam, gdzie reguła jest **wnioskowana** (nie ujęta dosłownie w jednej linii kodu), zostało to zaznaczone.

## Jak zakodowane są reguły

Fineract rozmieszcza reguły biznesowe w następujących warstwach:

1. **Konfiguracja produktów pożyczkowych / oszczędnościowych** — pola w `m_product_loan`, `m_savings_product` itp.
2. **Konfiguracja globalna** w `c_configuration` (przełączalna w czasie rzeczywistym przez `/v1/configurations`).
3. **Flagi funkcji w `application.properties`** — wymienione w [`docs/technical/08-configuration-and-feature-flags.md`](../technical/08-configuration-and-feature-flags.md).
4. **Migracje Liquibase**, które wprowadzają nowe kolumny z domyślną semantyką (np. `0006_product_loan_disallow_expected_disbursements.xml`).
5. **Encje JPA i usługi domenowe**, które rzucają `PlatformDataIntegrityException` / `PlatformApiDataValidationException` w przypadku naruszenia reguły.
6. **Kroki biznesowe COB** (np. tagowanie zaległości) — kodują one reguły behawioralne.
7. **Maker-checker** — reguły procesowe dotyczące tego, kto może zatwierdzić konkretne polecenie.

## Reguły walidacji (poziom wejściowy)

| Reguła | Lokalizacja |
| --- | --- |
| Nagłówek tenanta `Fineract-Platform-TenantId` jest wymagany w każdym wywołaniu API. | `TenantAwareBasicAuthenticationFilter`. *(wnioskowane: udokumentowane powszechnie w przykładach API i testach integracyjnych)* |
| Pola numeryczne muszą przestrzegać skonfigurowanego trybu zaokrąglania. | `fineract.tenant.config.rounding-mode=6` (HALF_EVEN, zaokrąglanie bankierskie). |
| Przesyłanie plików ograniczone przez rozszerzenia i listę dozwolonych typów MIME. | `fineract.content.regex-whitelist`, `fineract.content.mime-whitelist` (`application.properties:182-184`). |
| Limity multipart: 5 MB na plik, 10 MB na żądanie. | `application.properties:196-197`. |
| Dowolne wejścia SQL są czyszczone za pomocą profili regex (`main`, `adhoc`, `column`). | `fineract.sql-validation.*` (`application.properties:228-319`). |
| Idempotentne ponawianie: ten sam nagłówek `Idempotency-Key` odtwarza poprzednią odpowiedź. | `fineract.idempotency-key-header-name` (domyślnie `Idempotency-Key`). |

## Produkty pożyczkowe

| Reguła | Źródło |
| --- | --- |
| Produkt pożyczkowy należy do dokładnie jednej waluty. | `m_product_loan.currency_code` *(wnioskowane)*. |
| Produkty pożyczkowe mogą zabraniać wypłat powyżej **wnioskowanej** kwoty. | Migracja `0007_product_loan_higher_than_applied_loan_amount_management.xml`. |
| Produkty pożyczkowe mogą nie zezwalać na **oczekiwane** wypłaty (wielotranszowe). | Migracja `0006_product_loan_disallow_expected_disbursements.xml`. |
| Produkt pożyczkowy może deklarować strategię procesora transakcji; wybrana strategia musi być włączona na poziomie platformy. | Flagi `fineract.loan.transactionprocessor.*`. |
| Produkty pożyczkowe mogą być powiązane z definicją **zmiennej stopy procentowej**; zmiany stóp propagują się do aktywnych pożyczek. | `m_floating_rates_periods` i `LoanInterestRecalculationCOBBusinessStep`. |
| Produkty pożyczkowe mogą być powiązane z **kontami Księgi Głównej (GL)** dla naliczeń, kapitału, odsetek, opłat i kar. | `acc_product_mapping`. |

## Pożyczki (cykl życia)

| Reguła | Źródło |
| --- | --- |
| Pożyczka przechodzi przez statusy `SUBMITTED → APPROVED → ACTIVE → CLOSED` (z opcjonalnymi `WITHDRAWN`, `REJECTED`, `WRITTEN_OFF`, `OVERPAID`). | Początkowy schemat `m_loan.loan_status_id`; enum statusu w kodzie *(wnioskowane)*. |
| Zmiany statusów mogą być śledzone w historii, jeśli `fineract.loan.status-change-history-statuses` jest ustawione na `ALL` lub listę. | `application.properties:179`; tabela `m_loan_status_change_history`. |
| Zatwierdzone pożyczki nie mogą być zatwierdzone dwukrotnie. | Strażnicy poleceń API w `LoanWritePlatformService` *(wnioskowane)*. |
| Kwota wypłaty nie może przekroczyć kwoty zatwierdzonej. | Migracje `0006_*`, `0007_*`. |
| Spłaty są rozdzielane na kapitał / odsetki / opłaty / kary zgodnie ze strategią pożyczki. | `fineract.loan.transactionprocessor.*`. |
| Stornowana transakcja pożyczkowa tworzy przeciwstawny wpis w dzienniku zamiast usuwania oryginału. | Migracja `0019_refactor_loan_transaction.xml`. |
| Maker-checker może być wymagany dla zatwierdzenia pożyczki / wypłaty / umorzenia — konfigurowalne dla roli / uprawnienia. | `m_role_permission` *(wnioskowane)*; `/v1/makercheckers`. |
| Pożyczka może zostać zablokowana podczas COB; równoczesne aktualizacje są blokowane. | Wyjątek `LoanAccountWasAlreadyLockedOrProcessed` w `fineract-provider/.../cob/exceptions/`. |
| Data COB pożyczki jest przesuwana krok po kroku; luki wyzwalają przepływ nadrabiania zaległości (catch-up). | `/v1/internal/cob/oldest-cob-closed`, `/v1/internal/cob/catch-up`. |
| External ID może być używany zamiast wewnętrznego ID w wielu API pożyczkowych. | Zasoby `Path("external-id/{loanExternalId}/...")`. |

## Obsługa pożyczek (kroki biznesowe COB)

Skonfigurowany potok dotyczy portfela pożyczek (`m_batch_business_step_configuration`, migracja `0022_*`). Domyślne kroki obejmują:

| Krok | Reguła, którą egzekwuje |
| --- | --- |
| `CheckLoanRepaymentDueBusinessStep` | Oznacz raty płatne **dzisiaj** jako DUE; generuj zdarzenia. |
| `CheckLoanRepaymentOverdueBusinessStep` | Oznacz raty po terminie płatności jako OVERDUE. |
| `CheckDueInstallmentsBusinessStep` | Ogólne sprawdzenie i przejścia stanów dla należnych rat. |
| `ApplyChargeToOverdueLoansBusinessStep` | Naliczenie kar zgodnie z konfiguracją produktu. |
| `SetLoanDelinquencyTagsBusinessStep` | Przypisanie pożyczki do odpowiedniego przedziału zaległości (delinquency bucket). |
| `AccrualActivityPostingBusinessStep` | Księgowanie dziennych naliczeń (rozpoznany przychód odsetkowy). |
| `AddPeriodicAccrualEntriesBusinessStep` | Okresowe wpisy dziennika naliczeń. |
| `LoanInterestRecalculationCOBBusinessStep` | Ponowne przeliczenie odsetek, jeśli produkt na to pozwala. |
| `UpdateLoanArrearsAgingBusinessStep` | Aktualizacja liczników starzenia się zaległości (arrears-aging). |
| `CapitalizedIncomeAmortizationBusinessStep` | Rozłożenie skapitalizowanych opłat w harmonogramie. |
| `BuyDownFeeAmortizationBusinessStep` | Amortyzacja opłat typu buy-down przez pozostały okres. |

Tenanci włączają / ustalają kolejność kroków poprzez `m_batch_business_step_configuration`.

## Produkty i konta oszczędnościowe

| Reguła | Źródło |
| --- | --- |
| Produkt oszczędnościowy należy do jednej waluty. | `m_savings_product` *(wnioskowane)*. |
| Konto oszczędnościowe może założyć **blokadę (hold)** z podaniem powodu. | Migracja `0009_hold_reason_savings_account.xml`. |
| Produkty oszczędnościowe mogą pozwalać na **zastaw (lien)** (blokada prawna). | Migracja `0010_lien_allowed_on_savings_account_products.xml`. |
| Reguły uśpienia konta są zarządzane przez zadanie `UPDATE_SAVINGS_DORMANT_ACCOUNTS`. | Enum `JobName`. |
| Komponenty podatkowe mogą być dołączane do księgowań odsetek. | `m_savings_product_tax_group_mapping` *(wnioskowane z tabel podatkowych)*; logika zadania. |
| Transakcje oszczędnościowe mogą być stornowane — storno generuje wpis przeciwstawny zamiast usuwania. | Migracja `0005_savings_transaction_reversal.xml`. |
| Konto oszczędnościowe może należeć do użytkownika samoobsługowego (powiązane z `m_appuser`); tylko ten użytkownik może na nim operować przez `/v1/self/*`. | `SelfServiceUserAuthorizationManager`. |
| `is-savings-account-transaction-reversal-enabled` przełącza, czy stornowania są widoczne. | `c_configuration` *(wnioskowane z migracji)*. |

## Lokaty terminowe i wkłady systematyczne

| Reguła | Źródło |
| --- | --- |
| Konta lokat terminowych (FD) mają **datę zapadalności** obliczoną na podstawie okresu. | `m_deposit_account_term_and_preclosure`. Zadanie `UPDATE_DEPOSITS_ACCOUNT_MATURITY_DETAILS` dba o aktualność flagi `is_matured`. |
| Konta wkładów systematycznych (RD) generują **obowiązkowy harmonogram oszczędzania**. | Zadanie `GENERATE_RD_SCEHDULE`. |
| Wcześniejsze zamknięcie (wypłata przed terminem) jest dozwolone tylko wtedy, gdy produkt na to pozwala; zasady kar różnią się w zależności od produktu. | `m_deposit_product_term_and_preclosure`. |
| Oprocentowanie produktów FD/RD pochodzi z tabeli stóp procentowych (przedziały kwota × okres). | `m_deposit_product_interest_rate_chart`, `m_interest_rate_chart` *(wnioskowane)*. |

## Opłaty i kary

| Reguła | Źródło |
| --- | --- |
| Opłata ma **typ kalkulacji** (stała / procent od x / kwota na ratę). | `m_charge.charge_calculation_enum` *(wnioskowane)*. |
| Opłaty mogą być dodawane do pożyczek lub oszczędności; niektóre są nakładane automatycznie (np. opłata roczna). | Zadanie `APPLY_ANNUAL_FEE_FOR_SAVINGS`. |
| Opłaty pożyczkowe mogą mieć przypisane **zewnętrzne ID** dla integracji z partnerami. | Migracja `0008_loan_charge_add_external_id.xml`. |
| Kary dodane przez COB są oflagowane jako wygenerowane automatycznie. | `ApplyChargeToOverdueLoansBusinessStep`. |
| Opłaty mogą być opłacone z oszczędności klienta poprzez `TRANSFER_FEE_CHARGE_FOR_LOANS`. | `JobName`. |

## Księgowość

| Reguła | Źródło |
| --- | --- |
| Każdy ruch pieniężny wpływający na księgę tworzy wpis w dzienniku. | `acc_gl_journal_entry`. |
| Wpisy w dzienniku są niezmienne — korekty tworzą wpisy kompensujące. | Migracja `0025_add_audit_entries_to_journal_entry.xml` dodaje kolumny audytowe. |
| Zamknięcia Księgi Głównej zamrażają okres **dla danego biura**; księgowania z datą wsteczną przed datą zamknięcia dla tego biura są odrzucane. | `acc_gl_closure`. |
| Naliczenia odbywają się codziennie; niektóre produkty używają naliczeń okresowych na koniec miesiąca/okresu. | Zadania `ADD_ACCRUAL_ENTRIES`, `ADD_PERIODIC_ACCRUAL_ENTRIES`, `ACCRUAL_ACTIVITY_POSTING`. |
| Bilans próbny jest przeliczany przez zadanie `UPDATE_TRIAL_BALANCE_DETAILS`. | `JobName`. |
| Księgowania między biurami używają **kont aktywności finansowej** jako mostów. | `acc_gl_financial_activity_account`. |
| Agregacja wpisów w dzienniku podsumowuje wiersze starsze niż `fineract.job.journal-entry-aggregation.exclude-recent-N-days` (domyślnie 1). | `application.properties:83`. |

## Tworzenie rezerw

| Reguła | Źródło |
| --- | --- |
| Kategorie rezerw definiują przedziały zaległości (np. `0-30`, `31-60`). | `provisioning_category` *(wnioskowane)*. |
| Kryteria tworzenia rezerw definiują procent rezerwy na kategorię, opcjonalnie na produkt. | `provisioning_criteria` *(wnioskowane)*. |
| Zadanie `GENERATE_LOANLOSS_PROVISIONING` księguje wpisy w dzienniku dla wyliczonej rezerwy na pożyczkę. | `JobName`. |

## Tożsamość, dostęp i bezpieczeństwo

| Reguła | Źródło |
| --- | --- |
| Hasła są haszowane algorytmem bcrypt; poprzednie hasła są zapamiętywane. | `m_appuser_previous_password`. |
| Flaga wymuszenia zmiany hasła musi zostać usunięta przy pierwszym logowaniu. | Flaga `m_appuser` *(wnioskowane)*. |
| Role mogą mieć tylko kody uprawnień — brak nadpisywania wierszowego. | `m_role_permission` *(wnioskowane z początkowego schematu i migracji uprawnień `0011_*`, `0012_*`)*. |
| Użytkownicy samoobsługowi mogą działać tylko na własnych klientach. | `SelfServiceUserAuthorizationManager`. |
| Tokeny 2FA są cache'owane domyślnie przez 2 godziny. | `application.properties:325-326`. |
| Bezpieczeństwo na poziomie metod Spring Security wymusza uprawnienia dla każdej metody zasobu. | `SecurityConfig` `@EnableMethodSecurity`. |
| Hasła do baz danych tenantów są szyfrowane w spoczynku (at rest). | Migracje `tenant-store/0007_*`, `0008_*`, `0009_*`. |

## Reguły operacyjne

| Reguła | Źródło |
| --- | --- |
| Tylko jeden węzeł może być *menedżerem* wsadowym na tenanta dla zadania LOAN_COB. *(wnioskowane — równocześni menedżerowie podzieliby partycje)* | `fineract.mode.batch-manager-enabled`. |
| Zablokowane partycje są ponawiane do 5 razy domyślnie. | `application.properties:91-95` (`LOAN_COB_RETRY_LIMIT=5`). |
| Zdarzenia zewnętrzne korzystają ze **skrzynki nadawczej (outbox)** — są wstawiane w tej samej transakcji bazy danych, a następnie wysyłane przez `SEND_ASYNCHRONOUS_EVENTS`. | `JobName`, `fineract.events.external.partition-size`. |
| Sonda liveness musi odpowiedzieć w ciągu `initialDelaySeconds=90`; readiness w ciągu `60`. | `kubernetes/fineract-server-deployment.yml:73-86`. |
| Usuwanie wierszy `f_command_source` jest celowe — tylko poprzez zadanie `PURGE_PROCESSED_COMMANDS`. | `JobName`. |
| Zadanie czyszczenia zdarzeń zewnętrznych zachowuje konfigurowalne okno czasowe. | `JobName.PURGE_EXTERNAL_EVENTS`. |
| Bramka konfiguracji `enable-business-date` musi być włączona, aby `INCREASE_BUSINESS_DATE_BY_1_DAY` odniosło skutek. | `c_configuration` *(wnioskowane z migracji `0015_add_business_date.xml`)*. |

## Implikowane reguły zgodności / regulacyjne

| Reguła | Źródło |
| --- | --- |
| Wsparcie Maker-checker dla wrażliwych operacji jest dostępne od razu; instytucja decyduje, które uprawnienia go wymagają. | `f_command_source`, `/v1/makercheckers`. |
| Wszystkie wywołania API mogą być audytowane (tenant + użytkownik + URL + status) przez `request_audit_table`. | Początkowy schemat. |
| Wszystkie polecenia przechowują ładunek (payload) i wynik w magazynie poleceń. | Moduł `fineract-command`. |
| Tabele danych (datatables) pozwalają instytucji dodawać dane specyficzne dla jurysdykcji bez modyfikacji schematu. | `x_registered_table`, `m_field_configuration`. |
| Raportowanie MIX-XBRL jest wspierane dla raportowania do darczyńców. | Tabele `mix_taxonomy*`; `/v1/mixreport`. |

## Typowe pułapki (wnioskowane)

- Zapominanie o wypełnieniu `Fineract-Platform-TenantId` w każdym żądaniu.
- Edytowanie istniejących changesetów Liquibase po wdrożeniu — Liquibase odrzuci uruchomienie.
- Wyłączenie `FINERACT_JOB_LOAN_COB_ENABLED` podczas naliczania pożyczek — naliczenia, zaległości, starzenie się zaległości zostają zatrzymane.
- Uruchamianie z `fineract.insecure-http-client=true` na produkcji — pozwala to na certyfikaty samopodpisane w połączeniach wychodzących.
- Pozwalanie na działanie dwóch węzłów menedżera bez koordynacji przez brokera — duplikacja partycji.
- Używanie dołączonego magazynu kluczy (`keystore.jks`, hasło `openmf`) poza środowiskiem deweloperskim.

## Gdzie szukać dalej

- Dla rzeczywistych nazw właściwości i wartości domyślnych: [`docs/technical/08-configuration-and-feature-flags.md`](../technical/08-configuration-and-feature-flags.md).
- Dla szczegółów kroków COB: [`docs/technical/06-runtime-flows.md`](../technical/06-runtime-flows.md#3-loan-close-of-business-loan_cob).
- Dla tabel danych stojących za tymi regułami: [`docs/technical/04-data-model.md`](../technical/04-data-model.md).


---

← Poprzedni: [Słownik domenowy](04-domain-glossary.md) · ↑ [Indeks](../README.md) · Następny: [Integracje i interesariusze](06-integrations-and-stakeholders.md) →
