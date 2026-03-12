# Moduł Schematów Avro i Zdarzeń (fineract-avro-schemas)

[Powrót do dokumentacji głównej](README.md)

## Opis
Moduł `fineract-avro-schemas` stanowi fundamentalny element nowoczesnej, zdarzeniowej (Event-Driven) ewolucji systemu Apache Fineract. Umożliwia on bezkolizyjną integrację platformy z architekturą mikroserwisów banku poprzez system komunikacji oparty na strumieniowaniu (np. Apache Kafka). 

Zamiast przekazywać luźne, nietypowane obiekty JSON, Fineract wykorzystuje standard **Apache Avro**. Moduł ten to de facto rejestr schematów (Schema Registry), który udostępnia "Kontrakt Zdarzenia" (Event Contract). Definiuje on formalną strukturę komunikatów, co upewnia odbiorców w sieci banku (np. zewnętrzne systemy Anti-Money-Laundering, wysyłkę e-maili, SMS-ów, księgowość korporacyjną), że Payload (dane zdarzenia) zawsze będzie zgodny ze specyfikacją.

## Kluczowe komponenty techniczne

| Pliki Schematów (avsc) | Odpowiedzialność techniczna |
| :--- | :--- |
| **`*.avsc` (Pliki definicji)** | Fizyczne pliki definiujące strukturę JSON Schema z mocnym typowaniem danych (np. `LoanStatusChangedV1.avsc` definiuje: `id: long`, `status: string`, `timestamp: long`). |
| **Klasy Generowane (Generated Java)** | Proces budowy aplikacji (Gradle plugin `com.github.davidmc24.gradle.plugin.avro`) konwertuje pliki `.avsc` na pełnoprawne obiekty POJO (np. `LoanStatusChangedV1.java`) wykorzystywane bezpiecznie na produkcji. |
| **Payload Avro** | Skompresowany, binarny transfer wiadomości (mniejszy i szybszy od standardowego tekstowego JSON) rozsyłany w EventBrokerach za pomocą infrastruktury z `fineract-core`. |

## Zastosowanie i Architektura

Fineract wysyła te wygenerowane wiadomości Avro podczas kluczowych operacji domenowych.

```plantuml
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml
title Model C4 - Rola formatu Avro w komunikacji zewnętrznej

Component(loan_svc, "LoanWritePlatformService", "Moduł Pożyczek", "Wykrywa zmianę statusu umowy i rzuca BusinessEvent (zgodny ze schematem Avro)")
Component(message_broker, "Apache Kafka / ActiveMQ", "Infrastruktura Zewnętrzna", "Kolejka FIFO dla zdarzeń bankowych")
Component(external_crm, "Zewnętrzny CRM / SMS Gateway", "Aplikacja Niezależna", "Nasłuchuje Kafka Topic `fineract.loan.status` i weryfikuje Payload z Schema Registry")

System_Boundary(schemas, "fineract-avro-schemas") {
    Component(schema_registry, "Avro Schemas (.avsc)", "Kontrakt Typów", "Zapewnia zgodność formatu miedzy Producentem a Konsumentem")
}

Rel(loan_svc, message_broker, "Publikuje binarny Payload (zgodny z .avsc)")
Rel(message_broker, external_crm, "Konsumpcja wiadomości")
Rel(loan_svc, schema_registry, "Kompiluje i wykorzystuje klasy Java z Avro")
Rel(external_crm, schema_registry, "Opiera się na tym samym schemacie")

@enduml
```
