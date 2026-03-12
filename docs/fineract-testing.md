# Moduły Testowe i SDK (Testing & Clients)

[Powrót do dokumentacji głównej](README.md)

## Opis SDK (fineract-client, fineract-client-feign)
Dla łatwej i bezbłędnej integracji dowolnej aplikacji Java z uruchomionym serwerem bankowym Fineract, projekt oferuje wbudowane, gotowe do skompilowania paczki SDK (Software Development Kit). W repozytorium generowane i obsługiwane są one w oparciu o specyfikację Swagger/OpenAPI.

*   **`fineract-client`**: Natywny klient Java (przy użyciu biblioteki Retrofit2/OkHttp). Definiuje pełen zestaw klas POJO dla Request/Response w standardzie JSON. Pozwala np. innemu systemowi bankowemu (w technologii Spring) na wysłanie jednej linijki w kodzie Java: `loanApi.createLoan(loanRequestData)`, pod maską kompilując to do w pełni bezpiecznego żądania HTTP REST i deserializując obiekty.
*   **`fineract-client-feign`**: Opcjonalny interfejs SDK dedykowany do użycia w infrastrukturze chmurowej (Spring Cloud). Używa technologii Declarative Web Client (OpenFeign), co daje rewelacyjne wyniki we wdrażaniu Fineractu w architekturę zdekomponowanych mikroserwisów.

## Środowisko Testowe End-To-End i Integracji (integration-tests)
Jako otwarte i wysoce krytyczne oprogramowanie finansowe, Apache Fineract posiada ogromną i niesamowicie pokrywającą pakiety pulę testów systemowych zapobiegających regresji. Moduły:
*   **`integration-tests`**: Zawiera tysiące scenariuszy E2E (End-To-End). Skrypty te budują kontekst testowy bazy danych w locie (In-Memory Database / Testcontainers), nakładają konfigurację biura i personelu, tworzą klienta z oszczędnościami i symulują całe procesy operacyjne z użyciem biblioteki REST Assured. To zautomatyzowane akceptacje (Acceptance Criteria) działania każdego API.
*   **`fineract-e2e-tests-core` i `fineract-e2e-tests-runner`**: Ekosystem izolujący konfigurację bazy, wstrzykiwanie mocków w celach odpalania skryptów wydajnościowych i biznesowych BDD (Behavior-Driven Development) za pomocą runnera bazującego m.in. na środowiskach takich jak Cucumber.
*   **`oauth2-tests` i `twofactor-tests`**: Odseparowane pule weryfikacji tożsamości. Sprawdzają poprawne blokowanie niezalogowanego personelu i przechwytywanie logowań wieloskładnikowych z włączonymi trybami restrykcyjnymi bez "zaśmiecania" ogólnych testów domenowych portfela kredytowego.

Architektura ta gwarantuje bezpieczne rozwijanie i unowocześnianie rdzenia (Continuous Integration) za pomocą GitHub Actions / Jenkins, natychmiastowo zrywając build aplikacji przy naruszeniu reguł dziedzinowych w `integration-tests`.
