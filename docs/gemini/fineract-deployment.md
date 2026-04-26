# Infrastruktura i Wdrożenie (Deployment / K8s / Docker)

[Powrót do dokumentacji głównej](README.md)

## Opis

Apache Fineract to system w klasie Enterprise, zaprojektowany jako Modularny Monolit. Chociaż z punktu widzenia budowy i kompilacji jest zrzeszany do jednej całości przez moduł `:fineract-provider` i dystrybuowany często jako klasyczny plik serwerowy `fineract.war` (Web Archive) za pośrednictwem modułu pomocniczego `:fineract-war`, współczesne wdrożenia tego systemu silnie opierają się na technologiach konteneryzacji.

Repozytorium Fineract posiada komplet plików konfiguracyjnych wspierających szybkie uruchomienie w celach deweloperskich (lokalnych) oraz produkcyjne skalowanie w infrastrukturze chmury (Cloud-Native). 

## Kluczowe Technologie Wdrożeniowe

| Nazwa Komponentu | Przeznaczenie |
| :--- | :--- |
| **`fineract-war`** | Moduł Gradle spinający aplikację i generujący standardowy plik `.war`, możliwy do wgrania na stand-alone serwer Apache Tomcat, omijający konteneryzację, wykorzystywany przez mniejsze instytucje. |
| **`Dockerfile` / `docker/` / `jib`** | Konfiguracja budowania obrazu Docker. Używa Jib do bez-demonowego pakowania skompilowanych warstw Javy (Dependencies, Resources, Classes) prosto do Registry. Wykorzystuje minimalny kontener JDK, optymalizując czas startu i wielkość. |
| **`docker-compose-*.yml`** | Kolekcja predefiniowanych architektur. Pliki gotowe do postawienia całego środowiska na jeden "klik". Umożliwiają wybór bazy danych (`docker-compose-mysql.yml` / `-postgresql.yml`), message brokera (`-kafka.yml` / `-activemq.yml`) z odpowiednimi zmiennymi środowiskowymi bez wchodzenia w kod. |
| **`kubernetes/`** | Katalog zawierający manifesty wdrożeniowe (Deployment, ConfigMap, Service) do łatwej implementacji Apache Fineract w klaster wysokiej dostępności (K8s) w oparciu o środowiska MFI (np. `fineract-mifoscommunity-deployment.yml`). |

## Architektura Wdrożeniowa (Model C4 - Deployment)

Architektura docelowa z produkcyjnym użyciem Kubernetes wymusza uruchomienie wielokrotnych replik kontenera, przy jednoczesnym współdzieleniu relacyjnej bazy danych i brokera wiadomości.

```plantuml
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Deployment.puml
title Model C4 Deployment - Instalacja K8s Fineract (Cloud-Native)

Deployment_Node(cloud, "Cloud Provider (np. AWS / GCP)", "Platform") {
    
    Deployment_Node(k8s, "Kubernetes Cluster", "EKS / GKE") {
        
        Deployment_Node(ingress, "Ingress Controller", "Nginx") {
            Container(gateway, "API Gateway / TLS Termination", "Reverse Proxy")
        }

        Deployment_Node(app_nodes, "Worker Nodes (Fineract Pods)", "Docker / containerd") {
            Container(pod1, "fineract-server", "Spring Boot / Java 17", "Replica 1 (Traffic & UI)")
            Container(pod2, "fineract-server", "Spring Boot / Java 17", "Replica 2 (Traffic)")
            Container(pod3, "fineract-server-cob", "Spring Boot / Java 17", "Replica 3 (Tylko procesowanie wsadów w nocy, wyłączony ruch web)")
        }

        Deployment_Node(config, "ConfigMaps & Secrets", "K8s State") {
            Container(env_vars, "fineractmysql-configmap.yml", "Env", "Trzyma JDBC URL, dane Kafki")
        }
    }

    Deployment_Node(rds, "Managed Database", "RDS / Cloud SQL") {
        SystemDb_Ext(default_db, "Baza fineract_default", "MySQL/PG", "Słownik Dzierżawców")
        SystemDb_Ext(tenant_db, "Baza fineract_tenants", "MySQL/PG", "Dane Banków")
    }
    
    Deployment_Node(msk, "Managed Streaming", "MSK / Confluent") {
        SystemQueue_Ext(kafka, "Apache Kafka", "Broker Zdarzeń", "Topiki: fineract.*")
    }
}

Rel(gateway, pod1, "Zbalansowany ruch HTTP/REST")
Rel(gateway, pod2, "Zbalansowany ruch HTTP/REST")
Rel(pod1, env_vars, "Odczytuje w locie")

Rel(pod1, default_db, "Połączenie JDBC (HikariCP)")
Rel(pod1, tenant_db, "Połączenie JDBC (HikariCP)")
Rel(pod2, tenant_db, "Połączenie JDBC (HikariCP)")
Rel(pod3, tenant_db, "Masowy Update (Spring Batch COB)")

Rel(pod1, kafka, "Produkcja zdarzeń Avro")
Rel(pod2, kafka, "Produkcja zdarzeń Avro")

@enduml
```

## Mechanizmy i Zarządzanie

*   **Bezstanowość**: Pody (kontenery `fineract-server`) w Kubernetes są całkowicie bezstanowe (Stateless). Każde wywołanie API niesie token uwierzytelniający i nagłówek tenanta (Dzierżawcy). Dzięki temu ruch sieciowy może w dowolnym momencie wylądować w `Pod 1` lub `Pod 2`.
*   **Multi-Tenancy i Bazy Danych**: Ze względu na to, iż instancje podnoszą się jedna za drugą, zarządzanie migracją schematów bazy danych (Flyway/Liquibase) musi być precyzyjne i powiązane z blokadami, tak by aplikacje nie uszkodziły sobie nawzajem schematu. Fineract przy starcie (Boot) wykonuje samodzielną migrację bazy dzierżawców na podstawie definicji w `fineract_default`.
*   **Docker Compose Opcje**: Poprzez flagi i predefiniowane `docker-compose.yml` można w dosłownie minutę wdrożyć Fineract lokalnie integrując go pod klucz z dedykowanymi interfejsami graficznymi społeczności Mifos, uruchamiając całą platformę z bazą danych komendą: `docker compose -f docker-compose-community-app.yml up -d`.
