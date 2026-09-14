**Setmana 2: POO, SOLID i Models Immutables amb Java 21 Records**

---

### **Dilluns: Principis SOLID i Immutabilitat**

* **Cursos i Material de Lectura:**
* **Article:** [*SOLID Principles in Java*](https://www.baeldung.com/solid-principles) — Baeldung.
* **Article:** [*Java 21 Records Deep Dive*](https://www.baeldung.com/java-record-keyword) — Baeldung.
* **Vídeo:** [*SOLID en 7 Minuts*](https://www.youtube.com/results?search_query=solid+principles+java+tutorial) — CodeWithMosh o equivalent.
* **Documentació:** Oracle — [*Records (Preview Feature)*](https://docs.oracle.com/en/java/javase/21/docs/api/java.lang.Record.html).


* **Activitat i Què s'espera programar:**
* Llegir els principis SOLID (Single Responsibility, Open/Closed, Liskov, Interface Segregation, Dependency Inversion).
* Entendre **per què `record` és millor que POJOs mutables**: elimina setters, força immutabilitat, evita bugs.
* **Exercici de Prompt Engineering:** Obre Cursor i demana: *"Explica amb un exemple Java 21 per què un record és immutable i com això evita bugs en aplicacions concurrent"*. Compara la teva intuïció amb la resposta de l'IA.
* Revisar el `.cursorrules` de la setmana 1: afegir regles sobre immutabilitat i SOLID.
* **Exercici d'Escriptura de Specs (competència transversal):**
  * Reescriu el `.cursorrules` des de zero, però ara amb intenció: no és un fitxer de configuració genèric, és una **especificació de comportament** per a l'assistent.
  * Ha d'incloure:
    * Convencions de noms: variables en camelCase (Java) i snake_case (Python).
    * Regla d'immutabilitat: "Tots els models de domini han de ser `record` (Java) o `@dataclass(frozen=True)` (Python). Mai generar setters."
    * Estructura de carpetes: on va cada tipus de fitxer.
    * Regles de testing: "Cada classe pública ha de tenir una classe de test corresponent."
    * Regles de Git: format de commit (Conventional Commits).
  * **Verificació:** Demana a Cursor que generi un nou model (`PlayerRecord`) seguint el context del projecte. L'assistent respecta les regles del `.cursorrules`? Si no, ajusta les regles fins que ho faci.
  * **Lliçó:** Un `.cursorrules` és la primera spec que escrius per a un agent. Si és vaga, l'agent genera codi inconsistent. Si és precisa, el codi surt coherent amb el projecte.


---

### **Dimarts: Model Immutable GameRecord i DTOs**

* **Cursos i Material de Lectura:**
* **Article:** Baeldung — [*Java Records Constructors*](https://www.baeldung.com/java-record-keyword).
* **Documentació:** Oracle — [*Records Tutorial*](https://docs.oracle.com/javase/tutorial/records/).


* **Activitat i Què s'espera programar:**
* **Model de domini (`GameRecord`):**
  * Defineix un `record` amb camps: `appId` (String, identifier únic), `title` (String), `price` (BigDecimal), `activePlayerCount` (Long).
  * Afegeix un mètode `isPopular()` que retorna true si `activePlayerCount > 100_000`.
  * Afegeix un mètode `discountedPrice(double percentage)` que retorna el preu amb descompte sense mutar l'original (retorna un nou record).
  * **Compact Constructor:** Programar validació al constructor per assegurar que `appId` no és null i `price` >= 0.
* **Prova manualment:** Crea 3 instàncies de `GameRecord` a main, prova `isPopular()`, i crea un record descomptat sense mutar l'original.
* **Lliçó:** Els records són immutables; la way de fer "canvis" és crear nous records.


---

### **Dimecres: Patrons Repository i Interfícies**

* **Cursos i Material de Lectura:**
* **Article:** Baeldung — [*Repository Pattern in Java*](https://www.baeldung.com/java-dao-pattern).
* **Article:** Baeldung — [*Interface Segregation Principle*](https://www.baeldung.com/solid-principles#4-interface-segregation-principle).


* **Activitat i Què s'espera programar:**
* **Interfícies per a la capa de dades (preparació per S5):**
  * `GameRepository<T>`: interfície genèrica amb mètodes `save(T)`, `findById(String)`, `findAll()`, `delete(String)`.
  * `GameDataSource`: interfície per abstraure la font de dades (pot ser BD, API, o memòria).
* **Implementació In-Memory (`InMemoryGameRepository`):**
  * Implementa `GameRepository` usant un `ConcurrentHashMap` per emmagatzemar `GameRecord`s.
  * `save()`: afegeix o actualitza el record.
  * `findById()`: retorna `Optional<GameRecord>` per evitar nulls.
  * `findAll()`: retorna una immutable copy de tots els records.
* **Lliçó:** L'abstracció mediante interfícies permet canviar fonts de dades sense tocar el negoci. A la S5 reemplazarem `InMemory` per `JpaRepository`.


---

### **Dijous: Disseny de l'Arquitectura amb Patterns**

* **Cursos i Material de Lectura:**
* **Article:** Baeldung — [*Builder Pattern in Java*](https://www.baeldung.com/creational-design-patterns#builder).
* **Documentació:** Oracle — [*Sealed Classes*](https://docs.oracle.com/en/java/javase/21/docs/api/java.lang.Class.html) (opcional avançat).


* **Activitat i Què s'espera programar:**
* **Factory Pattern (`GameRecordFactory`):**
  * Estàtica que crea `GameRecord`s amb validacions centralitzades.
  * Mètode `createFromSteamAPI(String appId, String json)`: parseja un JSON de Steam i crea un `GameRecord`.
  * Mètode `createDefault(String appId)`: crea un record amb valors per defecte (preu 0, jugadors 0).
* **Service Layer (`GameManagementService`):**
  * Injecta `GameRepository` via constructor.
  * Mètode `registerGame(String appId, String title, BigDecimal price)`: usa el factory i guarda al repo.
  * Mètode `getPopularGames()`: retorna només jocs amb `isPopular() = true`.
* **Lliçó:** Separació de responsabilitats: Factory crea, Repository guarda, Service orquesta.


---

### **Divendres: Proves Unitàries i Refactorització amb CI**

* **Cursos i Material de Lectura:**
* **Documentació:** [*JUnit 5 Parameterized Tests*](https://junit.org/junit5/docs/current/user-guide/#writing-tests-parameterized-tests).
* **Article:** Baeldung — [*Testing Immutable Objects*](https://www.baeldung.com/java-testing-immutable-objects).


* **Activitat i Què s'espera programar:**
* **Proves unitàries (`GameRecordTests`):**
  * Test que verifica que `GameRecord` no es pot modificar (intentar setter fail).
  * Test parameteritzat que valida `isPopular()` amb múltiples valors de jugadors (100K, 50K, 1M).
  * Test que verifica que `discountedPrice()` no muta l'original.
* **Proves unitàries (`GameRepositoryTests`):**
  * Test que `save` i `findById` retorna el mateix record.
  * Test que `findById` retorna `Optional.empty()` per ID que no existeix.
  * Test que `findAll()` retorna immutable copy (modificar la llista no afecta el repo).
* **Proves unitàries (`GameManagementServiceTests`):**
  * Test que `registerGame` crea i guarda correctament.
  * Test que `getPopularGames` filtra correctament.
* **GitHub Actions Update:**
  * Executar tots els tests nous amb `mvn test`.
  * Afegir coverage report: si coverage < 80%, PR block (extends S6).
  * Commit message format: `feat(java): Immutable GameRecord, Repository pattern, SOLID principles`


* **Finalització del cicle Git:**
* Executa `mvn test` localment i verifica que els 10+ tests passen.
* Realitza el *commit* amb format convencional.
* Puja la branca `feature/week2-oop-solid` a GitHub i crea una PR.
* Verifica que el workflow GitHub Actions passa (tests + coverage).
* Fes merge a `main` si tot està verd.
