# Model zabezpieczeń

<details>
<summary><strong>Skocz do dowolnego dokumentu</strong></summary>

**[Indeks](../README.md)**

**Biznes:** [01 Podsumowanie menedżerskie](../business/01-executive-summary.md) · [02 Przegląd produktu](../business/02-product-overview.md) · [03 Procesy biznesowe](../business/03-business-processes.md) · [04 Słownik domenowy](../business/04-domain-glossary.md) · [05 Reguły biznesowe](../business/05-business-rules.md) · [06 Integracje i interesariusze](../business/06-integrations-and-stakeholders.md) · [07 Ryzyka i luki](../business/07-risks-and-gaps.md)

**Techniczne:** [01 Omówienie architektury](01-architecture-overview.md) · [02 Stos technologiczny](02-tech-stack.md) · [03 Mapa repozytorium](03-repository-map.md) · [04 Model danych](04-data-model.md) · [05 Dokumentacja API](05-api-reference.md) · [06 Przepływy uruchomieniowe](06-runtime-flows.md) · [07 Infrastruktura i wdrożenie](07-infrastructure-and-deployment.md) · [08 Konfiguracja i flagi funkcji](08-configuration-and-feature-flags.md) · **09 Model zabezpieczeń** · [10 Instrukcja operacyjna](10-operational-runbook.md) · [11 Strategia testowania](11-testing-strategy.md) · [12 Dziennik decyzji](12-decision-log.md)

</details>

> Uwierzytelnianie, autoryzacja, sekrety i znane zagrożenia.

## Opcje uwierzytelniania

Fineract obsługuje trzy tryby uwierzytelniania, które można łączyć:

| Tryb | Przełącznik | Domyślnie |
| --- | --- | --- |
| HTTP basic auth | `fineract.security.basicauth.enabled` (`FINERACT_SECURITY_BASICAUTH_ENABLED`) | włączone |
| Serwer zasobów OAuth2 (JWT bearer) | `fineract.security.oauth2.enabled` (`FINERACT_SECURITY_OAUTH_ENABLED`) | wyłączone |
| Uwierzytelnianie dwuskładnikowe (OTP) | `fineract.security.2fa.enabled` (`FINERACT_SECURITY_2FA_ENABLED`) | wyłączone |

Konfiguracja znajduje się w `fineract-provider/.../infrastructure/security/config/SecurityConfig.java` oraz `…/AuthorizationServerConfig.java`. `SecurityConfig` włącza `@EnableMethodSecurity` i rejestruje filtry uwzględniające tenantów (`TenantAwareBasicAuthenticationFilter`, `TenantAwareJpaPlatformUserDetailsService`).

### Nagłówek Tenant

Każde uwierzytelnione żądanie musi zawierać nagłówek `Fineract-Platform-TenantId: <id>`. Nagłówek jest wyodrębniany w łańcuchu filtrów i przekazywany do `ThreadLocalContextUtil`; kolejne filtry oraz `RoutingDataSource` korzystają z niego, aby wybrać bazę danych właściwą dla danego tenanta.

### Dane logowania Basic auth

- Przechowywane w tabeli `m_appuser` (dla każdego tenanta) z hasłami haszowanymi w formacie bcrypt.
- Polityka haseł jest wymuszana przez moduł `fineract-security` (długość, złożoność, historia poprzez `m_appuser_previous_password`).
- Flaga "wymuś zmianę hasła" w wierszu użytkownika wyzwala `/v1/users/{id}/changepassword` przed powodzeniem jakiegokolwiek innego wywołania API.

### Konfiguracja OAuth2

- Klient demonstracyjny zarejestrowany w `application.properties:38-42` (`frontend-client`, zakresy `read,write`, typy uprawnień `authorization_code,refresh_token`, przekierowanie `http://localhost:3000/callback`).
- `AuthorizationServerConfig` konfiguruje osadzony serwer autoryzacji Spring (pamięć podręczna klientów + podpisywanie JWT).
- Tokeny przechowywane po stronie serwera w tabelach `oauth_access_token`, `oauth_refresh_token`, `oauth_client_details` (schemat początkowy).
- W środowisku produkcyjnym należy zastąpić osadzony serwer zewnętrznym dostawcą tożsamości (IdP, np. Keycloak, Okta), wskazując punkt końcowy JWKS w `spring.security.oauth2.resourceserver.*`.

### Uwierzytelnianie dwuskładnikowe

- Kody OTP są generowane i wysyłane e-mailem/SMS-em za pośrednictwem systemu powiadomień `fineract-provider`.
- Tokeny są zapisywane w `twofactor_access_token`; konfiguracja w `twofactor_configuration`.
- Szablon pamięci podręcznej `userTFAccessToken` przechowuje wydane tokeny domyślnie przez 2 godziny (`application.properties:325-326`).
- Autonomiczne testy znajdują się w module `twofactor-tests`.

## Autoryzacja

- **Model ról / uprawnień.** Każdy użytkownik posiada jedną lub więcej ról (`m_appuser_role`); każda rola grupuje uprawnienia (`m_role_permission` *(wnioskowane)*); uprawnienia są wprowadzane przez Liquibase i poszczególne części dziennika zmian (np. `0011_add_credit_balance_refund_permission.xml`, `0012_add_merchantissuedrefund_payoutrefund_goodwillcredit_permissions.xml`).
- **Sprawdzanie na poziomie metod.** Zasoby oznaczają metody adnotacjami `@HasPermission` / `@PreAuthorize`; faktyczne sprawdzenie jest delegowane do `PlatformSecurityContext` i niestandardowego handlera wyrażeń (rozwiązuje kody w oparciu o uprawnienia aktywnego użytkownika).
- **Autoryzacja samoobsługowa.** Oddzielny `SelfServiceUserAuthorizationManager` (wywoływany w `SecurityConfig`) ogranicza ścieżki `/v1/self/*` wyłącznie do własnych zasobów dzwoniącego klienta.
- **Maker-checker.** Krytyczne operacje zapisu są wstępnie przygotowywane w magazynie poleceń (moduł `fineract-command`) i finalizowane przez innego użytkownika. Baza danych śledzi każde polecenie w tabelach typu `f_command_source` (`fineract-command/.../module/command/`).

## CORS, HSTS, CSRF

- CORS jest domyślnie włączony, zezwalając na wszystkie źródła / metody / nagłówki (`application.properties:30-35`). Należy go ograniczyć w produkcji.
- HSTS jest **domyślnie wyłączony** (`fineract.security.hsts.enabled=false`). Należy go włączyć za modułem równoważenia obciążenia (LB) kończącym połączenie TLS.
- CSRF nie ma zastosowania w przypadku API JSON (bezstanowe tokeny / uwierzytelnianie podstawowe) i nie jest włączony.

## Zarządzanie sekretami

- Załączony plik `keystore.jks` (`fineract-provider/src/main/resources/keystore.jks`) służy **wyłącznie do celów deweloperskich**; hasło `openmf` jest podane otwartym tekstem w `application.properties:391`.
- Dane logowania do bazy danych tenanta w tabeli magazynu tenantów są szyfrowane w spoczynku przy użyciu AES/CBC/PKCS5Padding z hasłem `fineract.tenant.master-password`. Migracje `0007_*`, `0008_*`, `0009_*` wstecznie szyfrują istniejące wiersze zapisane otwartym tekstem.
- Manifesty Kubernetes (`kubernetes/fineract-server-deployment.yml:94-117`) odczytują dane logowania do bazy danych z sekretu Kubernetes `fineract-tenants-db-secret`.
- Poświadczenia AWS mogą pochodzić z profilu instancji lub statycznych kluczy (`spring.cloud.aws.credentials.*`).

## Utwardzanie wejścia

- **Zabezpieczenia przed SQL injection** — wzorce `fineract.sql-validation.*` oczyszczają fragmenty SQL dostarczane przez użytkownika (raporty Stretchy, zapytania ad-hoc, wyrażenia filtrów tabel danych). Zobacz `application.properties:228-319`.
- **Lista dozwolonych treści** — `fineract.content.regex-whitelist` oraz `fineract.content.mime-whitelist` ograniczają dozwolone rozszerzenia plików / typy MIME (domyślnie PDF, MS Office, JPEG, PNG).
- **Limity multipart** — `spring.servlet.multipart.max-file-size=5MB`, `max-request-size=10MB` (`application.properties:196-197`).
- **OWASP ESAPI** znajduje się w ścieżce klasy (`org.owasp.esapi:esapi:2.7.0.0`) i jest skonfigurowany przez `ESAPI.properties`.
- **Walidacja wyjścia** — `validation.properties` definiuje wzorce walidacji wejścia oparte na wyrażeniach regularnych.
- **Flaga niezabezpieczonego klienta HTTP** — `fineract.insecure-http-client=true` domyślnie (zezwala na certyfikaty samopodpisane dla integracji wychodzących). **W produkcji należy ustawić na false.**

## Audytowanie

- Każde polecenie jest rejestrowane wraz z użytkownikiem, tenantem i znacznikiem czasu w magazynie poleceń (`fineract-command`).
- Tabela audytu na poziomie HTTP `request_audit_table` przechowuje dane o tenancie, użytkowniku, adresie URL i statusie; wypełniana przez interceptor.
- Tabele historii zmian domenowych: `m_loan_status_change_history`, `m_calendar_history`, `m_tax_component_history` itp.
- Krytyczne encje posiadają kolumny audytowe (`createdby_id`, `created_date`, `lastmodifiedby_id`, `lastmodified_date`); migracje `0020_add_audit_entries.xml`, `0024_add_audit_entries.xml`, `0025_add_audit_entries_to_journal_entry.xml` rozszerzają zakres audytu.

## Powierzchnia ataku (obserwowana)

| Powierzchnia | Ryzyko | Istniejące środki łagodzące |
| --- | --- | --- |
| Publiczne REST `/api/v1/**` | Próby nieuwierzytelnione; brute force na `/v1/authentication`. | Łańcuch filtrów Spring Security; blokada konta (kolumny blokady w `m_appuser` *(wnioskowane)*); ograniczanie liczby żądań przez Resilience4j (`management.health.ratelimiters.enabled`). |
| Dowolne dane wejściowe SQL | Injection poprzez Stretchy / Pentaho / Datatables / zapytania ad-hoc. | Profile `fineract.sql-validation.*`. |
| Przesyłanie plików | Złośliwe oprogramowanie, DoS dużymi plikami. | Lista dozwolonych typów treści i rozszerzeń, limity rozmiaru. |
| Punkt końcowy OAuth2 | Ujawnienie tokena, jeśli logi nie są czyszczone. | Wzorzec Logback pomija nagłówek Authorization zgodnie z konwencją; należy upewnić się, że wysyłka logów to respektuje. |
| Podszywanie się pod nagłówek Tenant | Użytkownik uwierzytelniony dla tenanta A próbuje uzyskać dostęp do tenanta B. | Uwierzytelnianie odbywa się **po** rozwiązaniu tenanta; wyszukiwanie użytkownika trafia do tabeli `m_appuser` rozwiązanego tenanta, więc poświadczenia z innego tenanta nie będą pasować. |
| Powierzchnia samoobsługowa | Klient uzyskujący dostęp do danych innych klientów. | `SelfServiceUserAuthorizationManager`. |
| Sekrety webhooków / hooków | Przechowywanie otwartym tekstem. | Przechowywane w `m_hook_configuration` (zaszyfrowane *(wnioskowane — do zweryfikowania)*). |
| Publiczny klient demo OAuth2 | Rejestracja "frontend-client" jest widoczna, jeśli używana bez zmian. | Należy go zastąpić przed uruchomieniem produkcyjnym. |

## Znane słabości i lista zadań

- **`fineract.insecure-http-client=true`** domyślnie — wychodzące połączenia HTTPS akceptują certyfikaty samopodpisane.
- **`server.ssl.key-store-password=openmf`** znajduje się w `application.properties` dla celów deweloperskich. Należy to nadpisać w każdym środowisku innym niż deweloperskie.
- **CORS szeroko otwarty** w konfiguracji domyślnej. Należy ograniczyć źródła w produkcji.
- **HSTS wyłączony** domyślnie.
- **Brak dołączonego WAF**; należy polegać na nadrzędnym LB / ingress.
- Literówka w kluczu `fineract.tenant.encrytion` (brak litery `p`) jest kluczem kanonicznym — należy zachować ostrożność podczas wyszukiwania.

> DO ZROBIENIA (wymaga potwierdzenia eksperta): czy tokeny autoryzacyjne są usuwane z `request_audit_table`? Należy to zweryfikować przed udostępnieniem tabeli dla systemów BI / monitoringu.


---

← Poprzedni: [Konfiguracja i flagi funkcji](08-configuration-and-feature-flags.md) · ↑ [Indeks](../README.md) · Następny: [Instrukcja operacyjna](10-operational-runbook.md) →
