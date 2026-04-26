# Zarządzanie Schematem Bazy Danych (fineract-db)

[Powrót do dokumentacji głównej](README.md)

## Opis
Moduł `fineract-db` to dedykowany pod-projekt odpowiedzialny za utrzymanie definicji, struktury i bezkolizyjnych aktualizacji schematów relacyjnych baz danych dla systemu Apache Fineract. Biorąc pod uwagę fakt, że Fineract jest systemem ERP/Core Banking, zmiany w modelach danych są skrajnie wrażliwe. W architekturze Multi-Tenant, każda wdrożona baza musi przejść identyczną, niezawodną ewolucję po uaktualnieniu wersji serwera.

Moduł ten porzucił czyste, nakładane ręcznie skrypty SQL na rzecz w pełni zautomatyzowanych silników wersjonowania schematu bazy danych (np. Flyway / Liquibase).

## Kluczowe mechanizmy

| Funkcja | Opis implementacji |
| :--- | :--- |
| **Główny schemat startowy** | System pozwala na inicjalizację bazy z całkowitego "zera" (Clean Install) za pomocą skompilowanych skryptów startowych DDL wprowadzających całe dziesiątki tabel `m_*` (portfolio) i `acc_*` (accounting). |
| **Migracje Wersjonowane (Flyway)** | Każda nowa tabela lub kolumna wprowadzana w kodzie przez programistę posiada odpowiadający jej plik migracji `V___opis_zmiany.sql`. Podczas startu aplikacji (Spring Boot Startup), mechanizm Flyway skanuje bazę dzierżawcy i zaciąga / wykonuje brakujące skrypty po kolei, aktualizując ją np. z wersji 1.7 do 1.8. |
| **Multi-Tenant DB Initialization** | Oddzielne pule skryptów definiują bazę konfiguracji nadrzędnej `fineract_default` (w której zarejestrowani są nowi dzierżawcy), od izolowanych skryptów dla konkretnego dzierżawcy `fineract_tenants`. |

## Procedura aktualizacji

```plantuml
@startuml
title Sekwencja - Start Aplikacji i migracja bazy danych (fineract-db)

participant "Serwer Fineract (Boot)" as server
participant "TenantDetailsService" as default_db
participant "Flyway Engine" as flyway
participant "Baza Klienta A (Tenant 1)" as db1
participant "Baza Klienta B (Tenant 2)" as db2

server -> default_db: Zbuduj pulę połączeń główną i pobierz Dzierżawców
activate default_db
default_db --> server: Lista: Tenant 1 (host1:3306), Tenant 2 (host2:5432)
deactivate default_db

loop Dla każdego dzierżawcy
    server -> flyway: Wywołaj migrate() dla bazy dzierżawcy
    activate flyway
    
    flyway -> db1: Odczytaj tabelę 'schema_version'
    db1 --> flyway: Ostatnia wgrana migracja: V320
    
    flyway -> flyway: Wykryto nowe pliki w folderze fineract-db: V321, V322
    flyway -> db1: Uruchom ALTER TABLE ... (Z pliku V321.sql)
    flyway -> db1: Zapisz log w schema_version (V321 SUCCESS)
    flyway -> db1: Uruchom ALTER TABLE ... (Z pliku V322.sql)
    flyway -> db1: Zapisz log w schema_version (V322 SUCCESS)
    
    flyway --> server: Baza dzierżawcy gotowa
    deactivate flyway
end

server -> server: Wystaw końcówki REST API. Aplikacja gotowa.
@enduml
```

## Znaczenie architektoniczne
Oddzielenie kodu DDL (Data Definition Language) do tego modułu pozwala również inżynierom DevOps na niezależne podnoszenie baz danych bez angażowania aplikacji Fineract (np. uruchamiając jedynie specjalny obraz Docker Flyway w procesach CI/CD przed startem rzeczywistych kontenerów serwerowych).
