# Moduł Infrastrukturalny (fineract-core)

[Powrót do dokumentacji głównej](README.md)

## Opis
Moduł `fineract-core` to wysoce reużywalny, dolnowarstwowy kręgosłup (Backbone) całego systemu Apache Fineract. Nie zawiera on logiki bezpośrednio powiązanej z bankowością (takiej jak pożyczki czy oszczędności), ale dostarcza wszystkie krytyczne elementy infrastruktury IT i frameworku, na których opierają się wyższe warstwy domeny biznesowej. Gwarantuje unifikację zarządzania zdarzeniami asynchronicznymi (Event Brokering), czasem w architekturze rozproszonej (Date & Time utilities dla stref czasowych dzierżawców), wyjątkami błędu, obsługą błędów bazodanowych oraz warstwą trwałości danych (JPA/EclipseLink Base Classes).

## Kluczowe komponenty

| Komponent | Odpowiedzialność biznesowa i techniczna |
| :--- | :--- |
| **`ThreadLocalContextUtil` i Contexty** | Utrzymywanie wątkowego kontekstu bieżącego wykonania (np. `TenantContext`, w którym zapisane jest do jakiego najemcy obecnie "mówimy", strefa czasowa tegoż tenanta oraz data operacyjna w banku dla tego wątku). |
| **`BusinessEvent` i Publishery** | Interfejsy i klasy bazowe dla szyny zdarzeń biznesowych (np. `LoanApprovedBusinessEvent`). Pozwala asynchronicznie (lub synchronicznie) powiadamiać rozłączne moduły systemu o zmianach stanu za pomocą `ApplicationEventPublisher` (Spring) lub Message Brokera (ActiveMQ/Kafka). |
| **`DateUtils` i Serwisy Czasowe** | Apache Fineract jest wrażliwy na czas. Wszystkie operacje bankowe wykonują się w strefie czasowej konkretnego serwera bankowego dzierżawcy, a nie w strefie fizycznego serwera w chmurze AWS/GCP. Te biblioteki narzucają precyzyjne manipulacje datami. |
| **`PlatformApiDataValidationException`** | Standaryzacja wyrzucania błędów walidacyjnych REST. Zbiera informacje o wszystkich polach w Payloadzie JSON, które naruszyły warunki biznesowe i przekazuje je na zewnątrz jako jedna sformatowana kolekcja błędów z kodem 400 Bad Request. |
| **Klasy Bazowe encji JPA** | `AbstractPersistableCustom` dostarczający generowanie identyfikatorów z Long (dla optymalizacji), pola dla Audytu (CreatedBy, LastModifiedBy) dla każdego obiekty w całym repozytorium kodu. |

## Architektura modułu

Architektura nie posiada wyraźnego kierunku "od dołu do góry" - jest zestawem usług wstrzykiwanych wszędzie indziej. 

```plantuml
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml
title Model C4 - Komponenty modułu fineract-core

Component(all_modules, "Wszystkie moduły domenowe", "Spring Component", "Loan, Savings, Accounting, Client korzystają z metod Core do bazowych operacji.")
Component(event_publisher, "Event Publisher / Listener", "Spring Data Events", "Rozsyła powiadomienia zdarzeń domenowych. Abstrakcja pozwalająca wymienić implementation-detail na kafkę/rabbitmq")
Component(exception_handlers, "Exception Mapper (JAX-RS / Spring)", "Controller Advice", "Łapie wyjątki typu PlatformDomainRuleException i tłumaczy na globalny standard API błędów HTTP.")
Component(tenant_context, "TenantContextHolder", "ThreadLocal", "Abstrakcja nad mapowaniem bazodanowym Multi-Tenancy.")

Rel(all_modules, tenant_context, "Odczyt aktualnej strefy czasowej i daty")
Rel(all_modules, exception_handlers, "Wrzuca wyjątki biznesowe w locie")
Rel(all_modules, event_publisher, "Wysyła i odbiera Business Events")

@enduml
```

## Zależności wewnętrzne i Integracje

*   Moduł leży na samym dole piramidy zależności w repozytorium. Fineract-core nie importuje żadnych innych modułów domenowych typu pożyczki (Loans).
*   Jest silnie zintegrowany z dostawcą ORM (EclipseLink) z użyciem specyficznych konfiguracji `JPA Provider` pod Fineract, w celu radzenia sobie z odroczonym ładowaniem list (`LazyInitializationException`) w skomplikowanych połączeniach kont oszczędnościowych i kredytowych.
