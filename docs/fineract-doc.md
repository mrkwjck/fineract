# Moduł Dokumentacji (fineract-doc)

[Powrót do dokumentacji głównej](README.md)

## Opis
Moduł `fineract-doc` to oficjalny projekt wchodzący w skład repozytorium zajmujący się kompilowaniem i udostępnianiem publicznej, oficjalnej dokumentacji projektu Apache Fineract na portalach The Apache Software Foundation. Nie zawiera on kodu Java modyfikującego działania systemu, a jedynie silnik budowania bazy wiedzy.

## Technologia

Dokumentacja Fineract, z racji bycia potężnym, wielodziedzinowym systemem, odeszła od standardowych plików Wiki/Markdown wpiętych bez struktury. System korzysta z technologii **Antora** oraz **Asciidoctor**, co niesie za sobą wymierne korzyści:
*   **Docs-as-Code:** Koncepcja trzymania architektury dokumentacji wraz z jej wersjonowaniem na równi z kodem źródłowym Javy, by pull request dodający nową funkcjonalność mógł od razu zawierać opis w doc.
*   **Modularność (Antora):** Konfiguracja zawarta w pliku `antora.yml` pozwala na kompilowanie struktury wielu modułów w jedną spójną i estetyczną witrynę internetową, z dynamicznie renderowanym menu bocznym.
*   **Asciidoctor (`.adoc`):** Potężniejszy standard od klasycznego Markdown, umożliwiający wstrzykiwanie zmiennych, generowanie w locie spisów treści zagnieżdżonych głęboko oraz includowanie (dołączanie) zewnętrznych diagramów PlantUML i urywków kodu bez jego duplikacji.

Projekt kompilowany jest komendami Gradle, które odpalają wewnątrz wtyczkę Node.js uruchamiającą kompilator stron statycznych do folderu `build/site`.
