**Setmana 3: Concurrència Pràctica per a Developers Web (Java 21)**

---

### **Dilluns: Com Funciona un Servidor Web Per Dins**

* **Cursos i Material de Lectura:**
* **Article:** Baeldung — [*How Spring Boot Handles Requests*](https://www.baeldung.com/spring-boot-start).
* **Article:** Baeldung — [*Introduction to Java Threads*](https://www.baeldung.com/java-thread-lifecycle).
* **Vídeo:** [*Concurrency in 15 Minutes*](https://www.youtube.com/results?search_query=java+concurrency+basics+for+beginners) — qualsevol vídeo introductori clar.
* **Documentació:** Oracle — [*Concurrency Lesson*](https://docs.oracle.com/javase/tutorial/essential/concurrency/).


* **Activitat i Què s'espera programar:**
* **Teoria: El model thread-per-request.**
  * Quan un usuari fa `GET /games/APP-123`, Spring Boot assigna un thread del pool a aquella request. Si arriben 200 requests simultànies, hi ha 200 threads treballant en paral·lel.
  * Dibuixa el diagrama: `Client → Tomcat thread pool → Controller → Service → Repository → BD`.
  * Pregunta clau: *"Què passa si el Service modifica una variable compartida entre threads?"*
* **Exercici pràctic: Demostrar el problema.**
  * Crea una classe `UnsafeCounter` amb un camp `int count` i un mètode `increment()` que fa `count++`.
  * Llança 10 threads que criden `increment()` 10.000 vegades cadascun.
  * Resultat esperat: el comptador **no** arriba a 100.000. Imprimeix el valor real.
  * Explica per què: `count++` no és atòmic (read-modify-write), dos threads poden llegir el mateix valor.
* **Exercici: Solucions bàsiques.**
  * Versió amb `synchronized`: funciona, però bloqueja.
  * Versió amb `AtomicInteger`: funciona sense bloquejar.
  * Mesura el temps de les dues solucions amb 10 threads × 1M increments. Quina és més ràpida? Per què?
* **Connexió amb S2:** Per què `GameRecord` és un `record` immutable? Perquè si fos mutable i compartit entre requests, tindríem exactament el problema de l'`UnsafeCounter`.
* **Exercici de Prompt Engineering:** Demana a Cursor: *"Genera un exemple Java 21 de race condition amb ArrayList compartida entre 4 threads."* Executa'l 5 vegades — el resultat canvia? L'IA t'ha avisat del problema? Revisa si l'explicació de l'assistent és correcta.


---

### **Dimarts: Race Conditions a la Vida Real — BD i @Transactional**

* **Cursos i Material de Lectura:**
* **Article:** Baeldung — [*Spring @Transactional*](https://www.baeldung.com/transaction-configuration-with-jpa-and-spring).
* **Article:** Baeldung — [*Optimistic vs Pessimistic Locking*](https://www.baeldung.com/jpa-optimistic-locking).
* **Article:** Vlad Mihalcea — [*Lost Update Problem*](https://vladmihalcea.com/lost-update-problem/).


* **Activitat i Què s'espera programar:**
* **Teoria: El "Lost Update" Problem.**
  * Dos usuaris obren la fitxa del mateix joc. Un canvia el preu, l'altre canvia el títol. Tots dos fan "Save" al mateix temps. Resultat: un dels canvis es perd.
  * Això és el que passa a qualsevol aplicació web amb BD. No és teòric — és el bug #1 de producció.
* **Exercici amb codi:**
  * Simula el problema: dos threads llegeixen el mateix `GameRecord` de la BD, el modifiquen, i el guarden. Verifica que un canvi es perd.
  * **Solució 1 — `@Transactional`:** Entendre que Spring obre una transacció per request i fa rollback si falla. Programar un `@Transactional` al `GameManagementService` i veure com la BD protegeix la integritat.
  * **Solució 2 — Optimistic Locking:** Afegir un camp `@Version` a l'entity `GameRecord`. Ara si dos threads intenten guardar la mateixa versió, un rep `OptimisticLockException`. Programar el handler.
* **Reflexió:** A una empresa, aquest problema apareix amb qualsevol formulari d'edició. Saber diagnosticar-lo i resoldre'l és el que diferencia un junior que funciona d'un que crea bugs en producció.


---

### **Dimecres: Crides a APIs Externes sense Bloquejar**

* **Cursos i Material de Lectura:**
* **Article:** Baeldung — [*Guide to CompletableFuture*](https://www.baeldung.com/java-completablefuture).
* **Article:** Baeldung — [*Spring @Async*](https://www.baeldung.com/spring-async).
* **Documentació:** Oracle — [*CompletableFuture API*](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/CompletableFuture.html).


* **Activitat i Què s'espera programar:**
* **Problema real:** GamePulse ha de consultar l'API de Steam (300ms) i l'API d'IGDB (400ms) per obtenir dades d'un joc. Si ho fem seqüencial: 700ms. Si ho fem en paral·lel: ~400ms (el màxim de les dues).
* **Exercici seqüencial:**
  * Crea `SteamApiClient` amb un mètode `fetchGameData(String appId)` que simula una crida HTTP amb `Thread.sleep(300)` i retorna un `SteamGameData` record.
  * Crea `IgdbApiClient` amb un mètode similar amb `Thread.sleep(400)`.
  * Crida ambdós seqüencialment i mesura el temps total (~700ms).
* **Exercici paral·lel amb `CompletableFuture`:**
  * Usa `CompletableFuture.supplyAsync()` per llançar les dues crides en paral·lel.
  * `CompletableFuture.allOf()` per esperar que ambdues acabin.
  * Mesura el temps total (~400ms).
  * Combina els resultats en un sol `GameRecord` amb `thenCombine()`.
* **Exercici amb `@Async` de Spring:**
  * Anota els mètodes de fetch amb `@Async` i retorna `CompletableFuture<SteamGameData>`.
  * Configura `@EnableAsync` a l'aplicació.
  * Crida des del service i combina resultats.
* **Lliçó pràctica:** Cada cop que un service ha de cridar més d'una API externa, has de pensar: "Puc fer-ho en paral·lel?" Això és el que es fa cada dia a una empresa amb microserveis.


---

### **Dijous: Virtual Threads (Java 21) i l'Extractor de Dades**

* **Cursos i Material de Lectura:**
* **Article:** Baeldung — [*Virtual Threads in Java 21*](https://www.baeldung.com/java-virtual-thread-vs-thread).
* **Article:** Inside Java — [*JEP 444: Virtual Threads*](https://openjdk.org/jeps/444).
* **Vídeo:** [*Project Loom Explained*](https://www.youtube.com/results?search_query=java+virtual+threads+explained) — qualsevol vídeo curt i clar.


* **Activitat i Què s'espera programar:**
* **Teoria: Per què existeixen els Virtual Threads?**
  * Problema: el thread pool de Tomcat té ~200 threads. Si cada request tarda 500ms esperant una API externa, només pots gestionar 400 requests/segon.
  * Amb Virtual Threads: pots tenir milions de threads "barats" (sense overhead de memòria). El thread que espera una resposta HTTP no bloqueja un thread real del sistema operatiu.
  * Reescriu l'exercici de dimecres amb `Thread.ofVirtual().start()` i compara la simplicitat del codi.
* **Exercici: Extractor de GamePulse.**
  * Crea `GameDataExtractor` que rep una llista de 50 `appId`s.
  * **Versió 1 — Seqüencial:** Crida `SteamApiClient.fetchGameData()` per cada appId un per un. Mesura temps.
  * **Versió 2 — Thread pool clàssic:** Usa `Executors.newFixedThreadPool(10)` per paral·lelitzar. Mesura temps.
  * **Versió 3 — Virtual Threads:** Usa `Executors.newVirtualThreadPerTaskExecutor()`. Mesura temps.
  * Compara els 3 resultats en una taula: temps total, simplicitat del codi, memòria usada.
* **Reflexió:** Virtual Threads simplifica el codi concurrent, però els problemes de race conditions i transaccions (dilluns-dimarts) **segueixen existint**. No és una bala de plata.
* **Connexió amb l'empresa:** Spring Boot 3.2+ ja suporta Virtual Threads amb un sol flag (`spring.threads.virtual.enabled=true`). A la feina, probablement només hauràs d'activar-lo — però has d'entendre *per què* existeix.


---

### **Divendres: Tests de Concurrència, Integració i Pull Request**

* **Cursos i Material de Lectura:**
* **Documentació:** JUnit 5 — [*Parallel Test Execution*](https://junit.org/junit5/docs/current/user-guide/#writing-tests-parallel-execution).
* **Article:** Baeldung — [*Testing Concurrent Code*](https://www.baeldung.com/java-testing-multithreaded).


* **Activitat i Què s'espera programar:**
* **Tests de concurrència (`ConcurrencyTests`):**
  * Test que demostra que `UnsafeCounter` falla amb múltiples threads (el test ha de fallar si el comptador no és atòmic).
  * Test que `AtomicInteger` versió passa amb múltiples threads.
  * Test que l'extractor amb Virtual Threads retorna exactament el mateix nombre de resultats que la versió seqüencial (consistència).
* **Tests d'integració de l'extractor (`GameDataExtractorTests`):**
  * Mock de `SteamApiClient` amb delays simulats.
  * Test que la versió paral·lela és almenys 3x més ràpida que la seqüencial per a 20 jocs.
  * Test que gestiona errors parcials: si 2 de 20 crides fallen, l'extractor retorna 18 resultats + 2 errors (no es perd tot).
* **Test d'Optimistic Locking (`GameTransactionTests`):**
  * Test que dos threads intentant actualitzar el mateix `GameRecord` simultàniament resulta en un `OptimisticLockException` per a un d'ells.

* **Finalització del cicle Git:**
* Executa `mvn test` i verifica que la suite sencera passa.
* Commit: `feat(java): concurrent data extraction with CompletableFuture and Virtual Threads`
* Puja branca `feature/week3-concurrency` i crea PR.

---

## Nota sobre Progressió

- **S1:** Big-O i estructures → *entens per què una query és lenta*
- **S2:** SOLID i immutabilitat → *entens per què el codi ha de ser desacoblat i thread-safe*
- **S3:** Concurrència pràctica → *entens per què dues requests simultànies poden corrompre dades, i com paral·lelitzar I/O*
- **S4:** Clean Code i CI → *entens com detectar problemes en codi d'altri (inclòs IA)*

Cada setmana resol un problema real que el developer trobarà al primer mes de feina.
