# Słownik pojęć domenowych

<details>
<summary><strong>Przejdź do dowolnego dokumentu</strong></summary>

**[Indeks](../README.md)**

**Biznesowe:** [01 Podsumowanie wykonawcze](01-executive-summary.md) · [02 Przegląd produktu](02-product-overview.md) · [03 Procesy biznesowe](03-business-processes.md) · **04 Słownik pojęć domenowych** · [05 Reguły biznesowe](05-business-rules.md) · [06 Integracje i interesariusze](06-integrations-and-stakeholders.md) · [07 Ryzyka i luki](07-risks-and-gaps.md)

**Techniczne:** [01 Przegląd architektury](../technical/01-architecture-overview.md) · [02 Stos technologiczny](../technical/02-tech-stack.md) · [03 Mapa repozytorium](../technical/03-repository-map.md) · [04 Model danych](../technical/04-data-model.md) · [05 Dokumentacja API](../technical/05-api-reference.md) · [06 Przepływy uruchomieniowe](../technical/06-runtime-flows.md) · [07 Infrastruktura i wdrożenie](../technical/07-infrastructure-and-deployment.md) · [08 Konfiguracja i flagi funkcji](../technical/08-configuration-and-feature-flags.md) · [09 Model bezpieczeństwa](../technical/09-security-model.md) · [10 Podręcznik operacyjny](../technical/10-operational-runbook.md) · [11 Strategia testowania](../technical/11-testing-strategy.md) · [12 Dziennik decyzji](../technical/12-decision-log.md)

</details>

> Definicje pojęć, encji i akronimów występujących w kodzie, interfejsach REST API i schemacie bazy danych sformułowane w prostym języku.

W przypadkach, gdy termin mapuje się bezpośrednio na tabelę bazy danych lub zasób REST, podano odpowiednie odniesienie. Szczegółowe informacje o schemacie znajdują się w [`docs/technical/04-data-model.md`](../technical/04-data-model.md). Ścieżki API można znaleźć w [`docs/technical/05-api-reference.md`](../technical/05-api-reference.md).

## Klienci i organizacja

- **Klient (Client)** — Klient banku, osoba fizyczna lub jednostka prawna niebędąca osobą fizyczną. Odpowiada tabeli `m_client` (oraz `m_client_non_person` dla jednostek prawnych). API: `/v1/clients`.
- **Grupa (Group)** — Zbiór klientów (np. grupa samopomocy, grupa wspólnej odpowiedzialności). `m_group`, `m_group_client`. API: `/v1/groups`.
- **Centrum (Centre)** — Wyższy poziom grupowania grup (zazwyczaj stosowany w mikrofinansach). Wiersze `m_group` z `level` o wartości „Center”. API: `/v1/centers`, `/v1/grouplevels`.
- **GLIM** — *Group Loan Individual Monitoring*: produkt pożyczki grupowej, w którym każdy członek grupy ma śledzony indywidualny udział. `glim_accounts`. Segment ścieżki API: `glim`.
- **GSIM** — *Group Savings Individual Monitoring*: odpowiednik dla oszczędności. `gsim_accounts`.
- **Biuro / Oddział (Office)** — Oddział lub jednostka w strukturze organizacyjnej banku. `m_office`. API: `/v1/offices`.
- **Personel / Pracownik ds. kredytów (Staff / Loan officer)** — Pracownik banku przypisany do klientów/pożyczek. `m_staff`. API: `/v1/staff`.
- **Kasjer (Teller / Cashier)** — Rola personelu obsługująca gotówkę. `m_tellers`, `m_cashiers`, `m_cashier_transactions`. API: `/v1/tellers`, `/v1/cashiers`.
- **Dzień wolny (Holiday)** — Data wolna od pracy dla danego biura. `m_holiday`, `m_holiday_office`. API: `/v1/holidays`.
- **Dzień roboczy (Working day)** — Konfiguracja dni tygodnia, które są dniami roboczymi dla danego biura. `m_working_days`. API: `/v1/workingdays`.
- **Kalendarz (Calendar)** — Reguła cykliczności używana przez grupy, spotkania, harmonogramy spłat. `m_calendar`, `m_calendar_history`, `m_calendar_instance`.

## Tożsamość i dostęp

- **AppUser** — Użytkownik systemu (pracownik lub klient korzystający z samoobsługi). `m_appuser`.
- **Rola (Role)** — Nazwany zestaw uprawnień. `m_role`, `m_appuser_role`.
- **Uprawnienie (Permission)** — Kod reprezentujący prawo do wykonania akcji (np. `CREATE_LOAN`, `APPROVE_LOAN`). `m_permission`.
- **Maker / Checker** — Role w procesie zatwierdzania („zasada czterech oczu”). Maker przesyła zapis; Checker zatwierdza go lub odrzuca przez `/v1/makercheckers`.
- **Użytkownik samoobsługowy (Self-service user)** — `m_appuser` powiązany z jednym lub wieloma wierszami `m_client`; ograniczony do `/v1/self/*`.
- **2FA** — Uwierzytelnianie dwuskładnikowe. `twofactor_access_token`, `twofactor_configuration`.
- **Klient OAuth2 (OAuth2 client)** — Zarejestrowana aplikacja kliencka, identyfikowana przez `client_id`. Przechowywana w `oauth_client_details`.

## Pożyczki (Lending)

- **Produkt pożyczkowy (Loan product)** — Szablon definiujący zachowanie pożyczki: waluta, odsetki, częstotliwość spłat, opłaty, zasady księgowania. `m_product_loan`. API: `/v1/loanproducts`.
- **Pożyczka (Loan)** — Konkretna instancja pożyczki zaciągniętej na podstawie produktu. `m_loan`. API: `/v1/loans`.
- **Konto pożyczkowe (Loan account)** — Synonim *pożyczki*; bieżący rekord z saldami i harmonogramem.
- **Transakcja pożyczkowa (Loan transaction)** — Pojedyncza zmiana w pożyczce: wypłata, spłata, opłata, naliczenie odsetek, umorzenie, zwrot. `m_loan_transaction`.
- **Harmonogram spłat (Repayment schedule)** — Lista rat z terminami płatności i kwotami. `m_loan_repayment_schedule`.
- **Wypłata (Disbursement)** — Przekazanie kapitału klientowi.
- **Spłata (Repayment)** — Płatność klienta zmniejszająca zaległe saldo pożyczki.
- **Opłata (Charge)** — Opłata lub kara nałożona na transakcję pożyczkową lub oszczędnościową. `m_charge`. API: `/v1/charges`.
- **Kara (Penalty)** — Specyficzny rodzaj opłaty nakładany za zwłokę lub naruszenie zasad.
- **Opłata pożyczkowa (Loan charge)** — Instancja `m_charge` nałożona na konkretną pożyczkę. `m_loan_charge`.
- **Zaległość (Delinquency)** — Stan bycia po terminie spłaty raty pożyczki.
- **Koszyk zaległości (Delinquency bucket)** — Przedział dni po terminie (np. 30, 60, 90). `m_loan_delinquency_action`, `m_loan_installment_delinquency_tag_history`. API: `/v1/delinquency`.
- **Zmiana harmonogramu (Reschedule)** — Modyfikacja warunków istniejącej pożyczki (okresu, rat). `/v1/rescheduleloans`.
- **Zmiana wieku (Re-age)** — Resetowanie terminów spłat rat po zmianie planu płatności. `m_loan_reage_parameter`.
- **Reamortyzacja (Re-amortise)** — Ponowne przeliczenie harmonogramu po częściowej przedpłacie. `m_loan_reamortization_parameter`.
- **Umorzenie / Spisanie (Write-off)** — Zamknięcie pożyczki jako nieściągalnej; kapitał przeniesiony do kosztów za pomocą zapisu w dzienniku.
- **Zamknięcie przedterminowe (Foreclose)** — Klient spłaca pozostałe saldo wcześniej, aby zamknąć pożyczkę.
- **Gwarant (Guarantor)** — Osoba/jednostka gwarantująca spłatę pożyczki innej osoby. `m_guarantor`, `m_guarantor_funding_details`, `m_guarantor_transaction`.
- **Zabezpieczenie (Collateral)** — Aktywa zastawione pod pożyczkę. `m_collateral_management`, `m_client_collateral_management`. API: `/v1/collateral-management`, `/v1/loan-collateral-management`.
- **Ponowne przypisanie pracownika (Loan officer reassignment)** — Przypisanie pożyczki do innego pracownika. `/v1/loans/loanreassignment`.
- **Strategia procesora transakcji (Transaction processor strategy)** — Wymienny algorytm dzielący płatność między kapitał, odsetki, opłaty i kary. Strategie obejmują m.in. `creocore`, `early-repayment`, `mifos-standard`, `heavensfamily`, `interest-principal-penalties-fees`, `principal-interest-penalties-fees`, `rbi-india`. Zobacz `application.properties:165-179`.
- **Reguła alokacji płatności (Payment allocation rule)** — W ramach strategii, sposób mapowania pojedynczej transakcji na pozycje harmonogramu. `m_loan_payment_allocation_rule`, `m_loan_product_payment_allocation_rule`.
- **Reguła alokacji uznania (Credit allocation rule)** — Sposób rozdzielania uznania (np. zrzeczenia się części spłaty) względem pożyczki. `m_loan_credit_allocation_rule`, `m_loan_product_credit_allocation_rule`.
- **Opłata za obniżenie stopy (Buy-down fee)** — Opłata, która „wykupuje” niższą efektywną stopę procentową, amortyzowana w okresie trwania pożyczki. Krok biznesowy: `BuyDownFeeAmortizationBusinessStep`.
- **Dochód skapitalizowany (Capitalised income)** — Dochód (np. opłaty) dodany do kapitału i amortyzowany. Krok: `CapitalizedIncomeAmortizationBusinessStep`.
- **Zmienna stopa procentowa (Floating rate)** — Zmienna stopa składająca się ze stopy bazowej i marży. `m_floating_rates`, `m_floating_rates_periods`. API: `/v1/floatingrates`.
- **Pożyczka progresywna (Progressive loan)** — Model pożyczki, w którym harmonogram jest przeliczany dynamicznie (powtarzające się przeliczenia). Moduł `fineract-progressive-loan`.
- **Pożyczka na kapitał obrotowy (Working-capital loan)** — Specjalistyczny produkt pożyczkowy na krótkoterminowy kapitał obrotowy. Moduł `fineract-working-capital-loan`.

## Oszczędności i depozyty

- **Produkt oszczędnościowy (Savings product)** — Szablon dla konta oszczędnościowego: waluta, metoda naliczania odsetek, opłaty, zasady. `m_savings_product` *(w schemacie początkowym)*.
- **Konto oszczędnościowe (Savings account)** — Aktywna instancja oszczędności. `m_savings_account`. API: `/v1/savingsaccounts`.
- **Lokata terminowa (Fixed deposit)** — Produkt oszczędnościowy zablokowany na określony termin. `m_deposit_account_term_and_preclosure`, `m_deposit_product_term_and_preclosure`. API: `/v1/fixeddepositaccounts`.
- **Lokata cykliczna (Recurring deposit)** — Produkt oszczędnościowy z okresowymi wpłatami. `m_deposit_account_recurring_detail`. API: `/v1/recurringdepositaccounts`.
- **Tabela stóp procentowych (Interest rate chart)** — Harmonogram stóp zależny od kwoty i okresu. `m_deposit_product_interest_rate_chart`. API: `/v1/interestratecharts`, `/v1/interestratecharts/{chartId}/chartslabs`.
- **Blokada / zastaw (Hold / lien)** — Zablokowana kwota na koncie oszczędnościowym, której nie można wypłacić. `m_deposit_account_on_hold_transaction`. Migracje `0009_hold_reason_savings_account.xml`, `0010_lien_allowed_on_savings_account_products.xml`.
- **Uśpienie (Dormancy)** — Status nieaktywnego konta oszczędnościowego. Zadanie: `UPDATE_SAVINGS_DORMANT_ACCOUNTS`.
- **Harmonogram obowiązkowych oszczędności (Mandatory savings schedule)** — Dla produktów RD, przewidywany harmonogram wpłat. Zadanie: `GENERATE_RD_SCEHDULE` (sic, literówka w kodzie).
- **Zlecenie stałe (Standing instruction - SI)** — Cykliczny przelew między kontami. `m_account_transfer_standing_instructions`. API: `/v1/standinginstructions`.
- **Przelew między kontami (Account transfer)** — Jednorazowy przelew międzykontowy. `m_account_transfer_transaction`. API: `/v1/accounttransfers`.
- **TPT** — *Third-Party Transfer*: przelew w ramach samoobsługi do zarejestrowanego beneficjenta. `/v1/self/beneficiaries/tpt`.

## Księgowość (Accounting)

- **KG (GL)** — Księga Główna (General Ledger).
- **Konto KG (GL account)** — Pozycja w planie kont (aktywa / pasywa / kapitał / przychody / koszty). `acc_gl_account`. API: `/v1/glaccounts`.
- **Zapis w dzienniku (Journal entry)** — Podwójny zapis księgowy w KG. `acc_gl_journal_entry`. API: `/v1/journalentries`.
- **Zamknięcie KG (GL closure)** — Odcięcie okresu dla biura, po którym księgowanie z datą wsteczną jest blokowane. `acc_gl_closure`. API: `/v1/glclosures`.
- **Reguła księgowa (Accounting rule)** — Możliwe do wielokrotnego wykorzystania mapowanie zdarzenia biznesowego (np. „wypłata pożyczki”) na konta KG Winien/Ma. `acc_accounting_rule`, `acc_rule_tags`. API: `/v1/accountingrules`.
- **Mapowanie produktu na KG (Product-to-GL mapping)** — Konfiguracja na poziomie produktu określająca, na które konta KG trafiają transakcje. `acc_product_mapping`.
- **Konto aktywności finansowej (Financial activity account)** — Specjalne „systemowe” konta KG do przelewów międzyoddziałowych. `acc_gl_financial_activity_account`. API: `/v1/financialactivityaccounts`.
- **Zestawienie obrotów i sald (Trial balance)** — Tabela sald KG na dany okres. `m_trial_balance`. Zadanie: `UPDATE_TRIAL_BALANCE_DETAILS`.
- **Naliczenie (Accrual)** — Rozpoznawanie odsetek/opłat jako przychodu zgodnie z harmonogramem, a nie w momencie otrzymania wpłaty. Zadania: `ADD_ACCRUAL_ENTRIES`, `ADD_PERIODIC_ACCRUAL_ENTRIES`, `ACCRUAL_ACTIVITY_POSTING`. Krok biznesowy: `AccrualActivityPostingBusinessStep`.
- **Tworzenie rezerw (Provisioning)** — Rozpoznawanie oczekiwanych strat z tytułu pożyczek według kategorii (koszyk zaległości). Tabele `provisioning_*`, zadania: `GENERATE_LOANLOSS_PROVISIONING`. API: `/v1/provisioningcategory`, `/v1/provisioningcriteria`, `/v1/provisioningentries`.
- **Agregacja zapisów w dzienniku (Journal entry aggregation)** — Nocna agregacja podsumowująca zapisy w dzienniku w celu przyspieszenia raportowania. Zadanie: `JOURNAL_ENTRY_AGGREGATION`.
- **NPA** — *Non-Performing Asset*: pożyczka sklasyfikowana jako utracona zgodnie z zasadami określonymi przez regulatora. Zadanie: `UPDATE_NPA`.

## Opłaty, podatki, waluta

- **Waluta (Currency)** — Kod waluty ISO z atrybutami wyświetlania. `m_currency`. API: `/v1/currencies`.
- **Składnik podatkowy (Tax component)** — Jednostka podatku (np. 18% VAT). `m_tax_component`, `m_tax_component_history`. API: `/v1/taxes/component`.
- **Grupa podatkowa (Tax group)** — Pakiet składników podatkowych. `m_tax_group`, `m_tax_group_mappings`. API: `/v1/taxes/group`.
- **Podatek u źródła (Withholding tax)** — Podatek potrącany od odsetek wypłacanych deponentom.
- **Kara (Penalty)** — Patrz *opłata*.
- **Fundusz (Fund)** — Źródło finansowania pożyczek (np. program darczyńców). `m_fund`. API: `/v1/funds`.

## Operacje i platforma

- **COB** — *Close-of-Business*: zestaw zadań wsadowych na zakończenie dnia. Moduły: `fineract-cob`, kroki biznesowe w `cob/loan/`, `cob/savings/`. API: `/v1/internal/cob`.
- **Data biznesowa (Business date)** — Logiczne „dziś” systemu, potencjalnie inne niż czas rzeczywisty (używane do odtwarzania/nadganiania zaległości). Klucze `c_configuration`, API `/v1/businessdate`.
- **Nadganianie (Catch-up)** — Ponowne uruchamianie COB dla dni, w których dany najemca (tenant) ma zaległości. `/v1/internal/cob/catch-up`.
- **Zadanie (Job)** — Zaplanowane zadanie Quartz. Wymienione w enumie `JobName`. `job`, `job_run_history`, `job_parameters`. API: `/v1/jobs`, `/v1/scheduler`.
- **Zadanie partycjonowane (Partitioned job)** — Zadanie Spring Batch podzielone na partycje robocze (obecnie `LOAN_COB`).
- **Kolejka Maker-checker** — Lista oczekujących poleceń wymagających zatwierdzenia. API: `/v1/makercheckers`.
- **Audyt (Audit)** — Niezmienny zapis wywołań API i wykonań poleceń. `request_audit_table`, `f_command_source`. API: `/v1/audits`.
- **Hook / webhook** — Skonfigurowane wywołanie zwrotne (callback) dla określonych zdarzeń. `m_hook_configuration`, `m_hook_registered_events`. API: `/v1/hooks`.
- **Powiadomienie (Notification)** — Wiadomość w aplikacji / e-mail / SMS do użytkownika. `notification_generator`, `notification_mapper`, `topic`, `topic_subscriber`. API: `/v1/notifications`, `/v1/sms`, `/v1/email`.
- **Tabela danych (Datatable)** — Zdefiniowana przez użytkownika tabela rozszerzeń dołączona do zarejestrowanej encji. `x_registered_table`. API: `/v1/datatables/...`.
- **Raport elastyczny (Stretchy report)** — Raport na bazie szablonu SQL z parametrami przekazywanymi w czasie wykonywania. `stretchy_report`, `stretchy_report_parameter`. API: `/v1/runreports/{name}`.
- **Raport Pentaho (Pentaho report)** — Raport `.prpt` Pentaho przechowywany i serwowany przez Fineract. Migracja `pentaho_reports_to_table`.
- **MIX** — *MicroFinance Information eXchange*: taksonomia danych rynkowych używana do raportowania dla organów regulacyjnych/darczyńców. `mix_taxonomy`, `mix_xbrl_namespace`. API: `/v1/mixmapping`, `/v1/mixreport`, `/v1/mixtaxonomy`.
- **Zdarzenie zewnętrzne (External event)** — Zdarzenie domenowe zakodowane w formacie Avro, publikowane do Kafki lub JMS dla systemów zewnętrznych. Moduł: `fineract-avro-schemas`.
- **Klucz idempotentności (Idempotency key)** — Nagłówek (domyślnie `Idempotency-Key`) zapewniający, że zduplikowany zapis nie zostanie zastosowany dwukrotnie.
- **Hooki a zdarzenia** — *Hooki* to webhooki HTTP; *zdarzenia zewnętrzne* to komunikaty Avro publikowane przez brokera. Oba są konfigurowalne dla poszczególnych typów zdarzeń.

## Samoobsługa i ankiety

- **Portal samoobsługowy (Self-service portal)** — Aplikacja dla klienta korzystająca z `/v1/self/*`.
- **Kieszeń (Pocket)** — Wirtualne subkonto w ramach konta oszczędnościowego służące do budżetowania. `/v1/self/pockets`.
- **Beneficjent (Beneficiary)** — Zarejestrowany odbiorca przelewów samoobsługowych. `/v1/self/beneficiaries/tpt`.
- **PPI** — *Progress out of Poverty Index*: model scoringowy używany do oceny prawdopodobieństwa ubóstwa klienta. `ppi_likelihoods`, `ppi_likelihoods_ppi`, `ppi_scores`. API: `/v1/likelihood`, `/v1/povertyLine`.
- **Ankieta / karta wyników (Survey / scorecard)** — Kwestionariusz i wynik końcowy przypisany do klientów (często do celów KYC / śledzenia wpływu). API: `/v1/surveys`, `/v1/surveys/scorecards`.

## Inwestorzy / zewnętrzni właściciele aktywów

- **Zewnętrzny właściciel aktywów (External asset owner)** — Strona trzecia, która kupuje / posiada część portfela pożyczek. API: `/v1/external-asset-owners`, `/v1/external-asset-owners/loan-product`. Moduł: `fineract-investor`.

## Interoperacyjność (Mojaloop)

- **FSP** — *Financial Service Provider* (Dostawca usług finansowych). Każda instancja Fineract może być jednym FSP w sieci Mojaloop.
- **Kwotowanie (Quote)** — Wycena przelewu między dostawcami FSP. `/v1/interoperation/quotes`.
- **Transfer (Transfer)** — Ruch pieniędzy między dostawcami FSP. `/v1/interoperation/transfers`.
- **Strona (Party)** — Identyfikator (numer telefonu, numer konta) używany do znalezienia kontrahenta. `/v1/interoperation/parties/{idType}/{idValue}`.

## Raportowanie i i18n

- **i18n** — Internacjonalizacja. Paczki komunikatów `messages*.properties` (obecnie `en` i `de`).
- **Kod globalizacji (Globalisation code)** — Klucz i18n w kopercie błędu.

## Indeks akronimów

| Akronim | Rozwinięcie |
| --- | --- |
| AML | Anti-Money Laundering (Przeciwdziałanie praniu pieniędzy). |
| API | Application Programming Interface (Interfejs programistyczny aplikacji). |
| CRM | Customer Relationship Management (Zarządzanie relacjami z klientami). |
| CSV | Comma-Separated Values (Wartości rozdzielane przecinkami). |
| CIB | Credit Information Bureau (Biuro informacji kredytowej). |
| COB | Close-of-Business (Zamknięcie dnia). |
| FSP | Financial Service Provider (Dostawca usług finansowych). |
| GL | General Ledger (Księga Główna). |
| GLIM | Group Loan Individual Monitoring. |
| GSIM | Group Savings Individual Monitoring. |
| KYC | Know Your Customer (Poznaj swojego klienta). |
| MFI | Microfinance Institution (Instytucja mikrofinansowa). |
| MIX | MicroFinance Information eXchange. |
| NPA | Non-Performing Asset (Aktywa niepracujące). |
| OTP | One-Time Password (Hasło jednorazowe). |
| PPI | Progress out of Poverty Index. |
| RBI | Reserve Bank of India. |
| RD | Recurring Deposit (Lokata cykliczna). |
| SI | Standing Instruction (Zlecenie stałe). |
| SLA | Service Level Agreement (Umowa o gwarantowanym poziomie usług). |
| SME | Subject-Matter Expert (Ekspert dziedzinowy). |
| SMS | Short Message Service. |
| SSO | Single Sign-On (Jednokrotne logowanie). |
| TPT | Third-Party Transfer (Przelew zewnętrzny). |
| WAR | Web ARchive. |
| XBRL | eXtensible Business Reporting Language. |

> DO ZROBIENIA (wymaga potwierdzenia przez eksperta): Fineract obsługuje wiele regionalnych wariantów terminologii specyficznych dla organów regulacyjnych (np. „BSA” w MFIs w Ameryce Łacińskiej). Aneks ze słowniczkiem regionalnym wykracza poza zakres tej wersji.


---

← Poprzedni: [Procesy biznesowe](03-business-processes.md) · ↑ [Indeks](../README.md) · Następny: [Reguły biznesowe](05-business-rules.md) →
