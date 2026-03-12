# Moduł: fineract-core

## Przegląd

Moduł `fineract-core` stanowi fundament całej platformy Apache Fineract, dostarczając zbiór globalnych klas pomocniczych, wspólnych wyjątków, konfiguracji, interfejsów API, funkcji uwierzytelniania, filtrowania, obsługi baz danych oraz integracji z pocztą elektroniczną. Jest to moduł infrastrukturalny, na którym opierają się wszystkie inne moduły biznesowe Fineract, zapewniając spójność, reusability i ułatwiając rozwój. Jego głównym zadaniem jest dostarczanie wspólnych narzędzi i mechanizmów, które są niezbędne do prawidłowego funkcjonowania systemu bankowego.

## Kluczowe komponenty

Moduł `fineract-core` zawiera szereg podpakietów, z których każdy odpowiada za specyficzny obszar infrastruktury lub wspólnych funkcjonalności:

*   **org.apache.fineract.accounting**: Zawiera podstawowe struktury danych i logikę związaną z księgowością, wykorzystywaną przez moduł `fineract-accounting` oraz inne moduły do inicjalizacji transakcji księgowych.
*   **org.apache.fineract.batch**: Wsparcie dla przetwarzania wsadowego (batch processing), umożliwiające wykonywanie grupowych operacji.
*   **org.apache.fineract.commands**: Definicje dla mechanizmu komend i zdarzeń, kluczowego dla spójności transakcyjnej i audytowalności w systemie Fineract.
*   **org.apache.fineract.infrastructure**: Pakiet zawierający szeroki zakres komponentów infrastrukturalnych, takich jak:
    *   **accountnumberformat**: Obsługa formatowania numerów kont.
    *   **bulkimport**: Funkcjonalności związane z masowym importem danych.
    *   **businessdate**: Zarządzanie datami biznesowymi w systemie.
    *   **cache**: Mechanizmy buforowania danych dla poprawy wydajności.
    *   **codes**: Zarządzanie kodami systemowymi i wartościami stałymi.
    *   **configuration**: Wspólne konfiguracje systemowe.
    *   **core**: Podstawowe narzędzia i klasy pomocnicze.
    *   **dataqueries**: Obsługa zapytań do danych.
    *   **documentmanagement**: Wsparcie dla zarządzania dokumentami.
    *   **event**: Definicje i mechanizmy obsługi zdarzeń systemowych.
    *   **hooks**: Punkty zaczepienia (hooks) dla niestandardowej logiki.
    *   **instancemode**: Określa tryb działania instancji Fineract (np. produkcyjny, deweloperski).
    *   **jobs**: Definicje i zarządzanie zadaniami cyklicznymi (cron jobs).
    *   **security**: Podstawowe mechanizmy bezpieczeństwa, w tym autoryzacji i uwierzytelniania.
    *   **springbatch**: Integracja ze Spring Batch dla złożonych operacji wsadowych.
    *   `DataIntegrityErrorHandler.java`: Klasa obsługująca błędy integralności danych.
*   **org.apache.fineract.interoperation**: Wsparcie dla interoperacyjności z innymi systemami.
*   **org.apache.fineract.notification**: Obsługa powiadomień systemowych.
*   **org.apache.fineract.organisation**: Podstawowe struktury związane z organizacją (np. oddziały, biura).
*   **org.apache.fineract.portfolio**: Wspólne struktury danych i logikę dla zarządzania portfelami (np. pożyczki, oszczędności).
*   **org.apache.fineract.useradministration**: Podstawowe funkcjonalności związane z administracją użytkowników.
*   **org.apache.fineract.util**: Zbiór ogólnych klas użytkowych.

## Przepływ danych

Moduł `fineract-core`, będąc modułem infrastrukturalnym, nie ma typowego "przepływu danych" w sensie realizacji konkretnej operacji biznesowej od początku do końca. Zamiast tego, dostarcza on komponenty, które są wykorzystywane przez inne moduły w ich przepływach danych.

Na przykład, gdy moduł `fineract-loan` przetwarza nową pożyczkę:
1.  Może wykorzystać komponenty z `fineract-core/infrastructure/security` do weryfikacji uprawnień użytkownika.
2.  Wywołać mechanizm komend z `fineract-core/commands` do zarejestrowania operacji udzielenia pożyczki.
3.  Użyć komponentów z `fineract-core/infrastructure/configuration` do pobrania globalnych ustawień.
4.  Wykorzystać `fineract-core/infrastructure/cache` do szybkiego dostępu do często używanych danych konfiguracyjnych.
5.  W przypadku błędów, `DataIntegrityErrorHandler.java` lub inne klasy z `fineract-core` mogą obsłużyć wyjątki i zapewnić spójność danych.

`fineract-core` działa więc jako dostawca usług dla innych modułów, umożliwiając im skuteczne i spójne przetwarzanie danych.

## Zależności wewnętrzne

`fineract-core` jest modułem, od którego zależy praktycznie każdy inny moduł w systemie Apache Fineract. Dostarcza on wspólne interfejsy, klasy bazowe, wyjątki i narzędzia, które są importowane i wykorzystywane w całym projekcie. Moduły takie jak `fineract-provider`, `fineract-loan`, `fineract-savings` i `fineract-accounting` w szerokim zakresie polegają na funkcjonalnościach dostarczanych przez `fineract-core` w celu realizacji swoich zadań biznesowych.

## Zależności zewnętrzne i integracje

Moduł `fineract-core` zawiera integracje z podstawowymi komponentami zewnętrznymi, które są kluczowe dla działania systemu:

*   **Baza Danych**: Poprzez komponenty z pakietu `infrastructure` (np. `DataIntegrityErrorHandler.java`) oraz wspólne mechanizmy dostępu do danych, `fineract-core` ułatwia interakcję z relacyjnymi bazami danych (MySQL/PostgreSQL).
*   **Spring Framework**: Fineract jest zbudowany na Spring Boot, a `fineract-core` wykorzystuje wiele funkcjonalności Springa (np. Spring Security, Spring Batch, Spring Cache) dostarczając ich ogólne konfiguracje i rozszerzenia.
*   **Usługi pocztowe**: Poprzez odpowiednie klasy i konfiguracje, `fineract-core` może integrować się z zewnętrznymi serwerami pocztowymi do wysyłania powiadomień.
*   **System plików/Magazyny dokumentów**: Pakiet `documentmanagement` w `infrastructure` sugeruje możliwość integracji z systemami plików lub innymi rozwiązaniami do przechowywania dokumentów.

## Zarządzanie stanem i baza Danych

`fineract-core` odgrywa kluczową rolę w zarządzaniu stanem i interakcjach z bazą danych poprzez:

*   **Mechanizmy Konfiguracji**: Przechowuje i udostępnia globalne ustawienia systemu, które mogą wpływać na zachowanie innych modułów. Te konfiguracje są często trwałe i zapisywane w bazie danych.
*   **Buforowanie (Cache)**: Pakiet `infrastructure/cache` implementuje mechanizmy buforowania, które zarządzają stanem danych w pamięci podręcznej, redukując obciążenie bazy danych i poprawiając czas odpowiedzi.
*   **Zarządzanie Zadaniami (Jobs)**: Pakiet `infrastructure/jobs` definiuje i zarządza zadaniami cyklicznymi, które modyfikują stan systemu i bazy danych (np. zadania COB).
*   **Obsługa Integralności Danych**: Klasy takie jak `DataIntegrityErrorHandler` zapewniają, że operacje na bazie danych są wykonywane w sposób bezpieczny i spójny, chroniąc przed naruszeniami integralności danych.
*   **Formatowanie Numerów Kont**: Moduł ten może zarządzać generatorami sekwencji numerów kont, które są trwale przechowywane i aktualizowane w bazie danych.
