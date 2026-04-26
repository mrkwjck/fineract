# Stos technologiczny

<details>
<summary><strong>Skocz do dowolnego dokumentu</strong></summary>

**[Indeks](../README.md)**

**Biznesowe:** [01 Podsumowanie menedżerskie](../business/01-executive-summary.md) · [02 Przegląd produktu](../business/02-product-overview.md) · [03 Procesy biznesowe](../business/03-business-processes.md) · [04 Glosariusz domenowy](../business/04-domain-glossary.md) · [05 Reguły biznesowe](../business/05-business-rules.md) · [06 Integracje i interesariusze](../business/06-integrations-and-stakeholders.md) · [07 Ryzyka i luki](../business/07-risks-and-gaps.md)

**Techniczne:** [01 Przegląd architektury](01-architecture-overview.md) · **02 Stos technologiczny** · [03 Mapa repozytorium](03-repository-map.md) · [04 Model danych](04-data-model.md) · [05 Dokumentacja API](05-api-reference.md) · [06 Przepływy uruchomieniowe](06-runtime-flows.md) · [07 Infrastruktura i wdrożenie](07-infrastructure-and-deployment.md) · [08 Konfiguracja i flagi funkcji](08-configuration-and-feature-flags.md) · [09 Model bezpieczeństwa](09-security-model.md) · [10 Podręcznik operacyjny](10-operational-runbook.md) · [11 Strategia testowania](11-testing-strategy.md) · [12 Rejestr decyzji](12-decision-log.md)

</details>

> Języki, frameworki, biblioteki i narzędzia faktycznie użyte w projekcie.

Wersje pochodzą z plików `build.gradle`, `settings.gradle` oraz centralnego katalogu zależności `buildSrc/src/main/groovy/org.apache.fineract.dependencies.gradle`. Są one przypięte (brak wersji dynamicznych); aktualizacje są proponowane przez bota Renovate (`renovate.json`).

## Środowisko uruchomieniowe

| Obszar | Wybór | Źródło |
| --- | --- | --- |
| Język | Java 21 | `build.gradle:393-399` (`languageVersion = JavaLanguageVersion.of(21)`) |
| Framework aplikacji | Spring Boot 3.5.6 | `build.gradle:113`, BOM zależności `org.springframework.boot:spring-boot-dependencies:3.5.6` |
| Kontener webowy | Embedded Tomcat 10.1.49 | dep `org.apache.tomcat.embed:tomcat-embed-core:10.1.49` |
| Framework REST | JAX-RS przez Jersey 3.1.10 (BOM) | `JerseyConfig` montuje `/api`; `mavenBom 'org.glassfish.jersey:jersey-bom:3.1.10'` |
| Serializacja JSON | Jackson 2.19.2 (BOM) i Gson 2.11.0 | `mavenBom 'com.fasterxml.jackson:jackson-bom:2.19.2'`, `dep com.google.code.gson:gson` |
| Walidacja | Jakarta Bean Validation 3.1.1 + Hibernate Validator 9.0.1.Final | dep `jakarta.validation:jakarta.validation-api:3.1.1`, `org.hibernate.validator:hibernate-validator:9.0.1.Final` |
| Persystencja | EclipseLink JPA 4.0.2 (ze statycznym weaveniem) | dep `org.eclipse.persistence:org.eclipse.persistence.jpa:4.0.2`, `static-weaving.gradle` aplikowane per subprojekt (`build.gradle:165-167`) |
| Migracje schematu | Liquibase (Spring Boot starter) + rozszerzenie `liquibase-postgresql:4.33.0` | `db/changelog/db.changelog-master.xml`, dep `org.liquibase.ext:liquibase-postgresql:4.33.0` |
| Pula połączeń | HikariCP (domyślna w Spring Boot) | właściwość `spring.datasource.hikari.*` w `application.properties` |
| Obsługiwane bazy danych | MariaDB / MySQL / PostgreSQL | sterowniki `com.mysql:mysql-connector-j:9.3.0`, `org.postgresql:postgresql:42.7.9`; `docker-compose-{mariadb,mysql,postgresql}.yml` |
| Harmonogram zadań | Quartz 2.5.0 + Spring Batch (Spring Boot starter) | dep `org.quartz-scheduler:quartz:2.5.0`; tabele Spring Batch zainicjowane w `db/changelog/tenant/parts/0021_add_spring_batch_db_structure.xml` |
| Komunikacja asynchroniczna | ActiveMQ Classic 5.18.x (domyślny JMS) i Apache Kafka (opcjonalnie) | `docker-compose-postgresql-activemq.yml`, `docker-compose-postgresql-kafka.yml`; właściwości `fineract.remote-job-message-handler.{jms,kafka}.*`, `fineract.events.external.producer.{jms,kafka}.*` |
| Cache | Ehcache 3.10.8 (JCache) | dep `org.ehcache:ehcache:3.10.8`, `javax.cache:cache-api:1.1.1` |
| Bezpieczeństwo | Spring Security (Boot starter), Nimbus JOSE+JWT 10.0.2 (OAuth2/JWT), Bouncy Castle 1.81 (krypto) | dep `com.nimbusds:nimbus-jose-jwt:10.0.2`, `org.bouncycastle:bcprov-jdk18on:1.81` |
| Autoryzacja dwuskładnikowa (2FA) | Niestandardowy przepływ OTP w `fineract.security.2fa.*` | przełącznik w `application.properties:26` |
| Szablony | Mustache (kompilator `0.9.14`) | dep `com.github.spullara.mustache.java:compiler:0.9.14` |
| OpenAPI / Swagger | Swagger 2.2.22, Springdoc 2.6.0 | `build.gradle:99,115`, dep `org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0` |
| AWS SDK | Spring Cloud AWS 3.2.1, AWS SDK 2.29.9 | BOMy `io.awspring.cloud:spring-cloud-aws-dependencies`, `software.amazon.awssdk:bom` |
| Obserwowalność | Micrometer 1.13.6 BOM, Spring Boot actuator, OpenTelemetry 1.44.1 | BOMy `io.micrometer:micrometer-bom`, `io.opentelemetry:opentelemetry-bom` |
| Logowanie | Logback 1.5.19 + enkoder JSON | dep `ch.qos.logback:logback-classic:1.5.19`, `logback-spring.xml` |
| Klient HTTP (serwerowy) | OkHttp 4.12.0 BOM, Retrofit 2.9.0 (klienci) | `build.gradle:23-24`, BOM `com.squareup.okhttp3:okhttp-bom:4.12.0` |
| Zdarzenia oparte na schemacie | Apache Avro (moduł `fineract-avro-schemas`) | plugin `com.github.davidmc24.gradle.plugin.avro-base`, `build.gradle:124` |
| Raportowanie | Raporty Pentaho (niestandardowa integracja), raporty Stretchy (szablony SQL) | tabele w `db/changelog/tenant/parts/0017_fix_stretchy_reports.xml`, `0018_pentaho_reports_to_table.xml` |
| Przechowywanie plików | Lokalny system plików (domyślnie) lub AWS S3 | `application.properties:186-194` (`fineract.content.filesystem.*`, `fineract.content.s3.*`) |

## Budowanie i narzędzia

| Obszar | Wybór | Źródło |
| --- | --- | --- |
| System budowania | Gradle wielomodułowy | `settings.gradle`, `build.gradle` |
| Budowanie obrazu kontenera | Google Jib 3.4.5, baza `azul/zulu-openjdk-alpine:21` | `build.gradle:119` |
| Cache budowania Develocity | włączony, instancja ASF Develocity | `settings.gradle:21-47` |
| Statyczne weavenie | Agent EclipseLink | `static-weaving.gradle` |
| Jakość kodu | Checkstyle (`config/checkstyle/`), SpotBugs 6.0.26, Modernizer, ErrorProne 4.1.0, SonarQube 6.0.1 | `build.gradle:108,114,121-122,433` |
| Formatowanie | Spotless 6.25.0 + greclipse, Apache RAT 0.8.1 | `build.gradle:108-109,228-247` |
| Zarządzanie licencjami | hierynomus license-gradle-plugin, license-report 2.9 | `build.gradle:110-111,251-275` |
| Wydania | git-versioning 6.4.4, niestandardowy `org.apache.fineract.release.gradle` | `build.gradle:104,132,140-159` |
| Generator klienta OpenAPI | `org.openapi.generator` 7.8.0 | `build.gradle:125` |
| Codegen Avro | `com.github.davidmc24.gradle.plugin.avro-base` 1.9.1 | `build.gradle:124` |
| Logowanie testów | `com.adarshr.test-logger` 4.0.0 | `build.gradle:107` |
| Runner Cucumber | `se.thinkcode.cucumber-runner` 0.0.11 | `build.gradle:123` |
| Mikrobenchmarki | JMH 1.37 (`me.champeau.jmh`) | `build.gradle:127` |
| SBOM | CycloneDX 3.1.0 | `build.gradle:128` |
| Wykrywanie zmian niszczących API | Swagger-Brake 2.7.0 | `build.gradle:129` |
| Aktualizacje zależności | Renovate Bot | `renovate.json` |
| CI | GitHub Actions (19 przepływów w `.github/workflows/`) | budowanie per baza danych, Docker, e2e, Cucumber, SonarCloud, itd. |

## Stos testowy

| Obszar | Wybór |
| --- | --- |
| Testy jednostkowe | JUnit 5 (BOM `org.junit:junit-bom:5.11.3`), Mockito 5.14.2 (BOM), AssertJ 3.27.7 |
| BDD / E2E | Cucumber 7.20.1 (`fineract-e2e-tests-core`/`-runner`) |
| Kontenery | Testcontainers 1.20.4 |
| HTTP / mockowanie | MockServer 5.15.0, OkHttp |
| Testowanie poczty | GreenMail 2.0.1 |
| Smoke testy API | Spring REST Docs 3.0.3, Awaitility 4.2.2 |

## Czego **nie ma** w stosie

Potwierdzone braki (brak zależności lub importów):

- Brak reaktywnego WebFlux — całe wejście/wyjście jest synchroniczne na wątkach Tomcata.
- Brak GraphQL.
- Brak magazynów NoSQL: tylko MariaDB / MySQL / PostgreSQL.
- Brak frontendu: to repozytorium to backend; interfejs użytkownika znajduje się w oddzielnych obrazach Mifos web-app, do których odwołują się pliki `docker-compose-community-app.yml` i `docker-compose-web-app.yml`.

> TODO (wymaga potwierdzenia SME): czy istnieje oficjalnie wspierany Helm chart? Katalog `kubernetes/` zawiera tylko zwykłe pliki YAML.


---

← Poprzedni: [Przegląd architektury](01-architecture-overview.md) · ↑ [Indeks](../README.md) · Następny: [Mapa repozytorium](03-repository-map.md) →
