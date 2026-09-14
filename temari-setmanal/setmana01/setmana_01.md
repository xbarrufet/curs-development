**Setmana 1: Rendiment Real, Algorísmia i Benchmarking ($O(n)$ vs $O(1)$) amb Java 21**

---

### **Dilluns: Entorn, Git i Estructura Políglota**

* **Cursos i Material de Lectura:**
* **Llibre:** [*Pro Git*](https://git-scm.com/book/en/v2) (Scott Chacon) — Capítols 1 (Getting Started) i 2 (Git Basics).
* **Vídeo:** [*Curso de Git y GitHub desde cero*](https://www.youtube.com/watch?v=niPExbK8lSw) — Midudev (YouTube).
* **Documentació:** [Guia d'instal·lació d'Oracle OpenJDK 21](https://docs.oracle.com/en/java/javase/21/install/).
* **Guia de Cursor:** [Cursor IDE Docs](https://cursor.sh) — Instal·lació i Claude integration.


* **Activitat i Què s'espera programar:**
* Instal·la OpenJDK 21 i [Cursor IDE](https://cursor.sh) (o VS Code + extensió Claude).
* **Setup de Cursor + Claude:**
  * Configura Cursor per usar Claude com a model per defecte.
  * Prova una conversa simple amb l'agent: "Genera un `.gitignore` per un projecte Java + Python políglota."
  * Obres Cursor Settings → Features → activar "Tab autocomplete" i "Agent mode".
* Inicialitza des de la terminal el repositori `esportspulse-engine`.
* Crea l'estructura de directoris aïllant la carpeta `/backend-java` de la carpeta `/ai-python`.
* **Exercici de Prompt Engineering amb Cursor (competència transversal):**
  * Obre Cursor, navega a la carpeta del projecte i crea un `.cursorrules` buit.
  * Obres la seva finestra de chat i demana: *"Escriu un `.cursorrules` per a un projecte Java 21 + Python amb Spring Boot i Pydantic. Inclou regles per a noms de variable, format de commit, i structure de carpetes."*
  * Deixa que Cursor generi el fitxer. (Alternativa: usa el [template inicial](../cursorrules-template-week1.md) de referència).
  * Revisa el contingut generat. **Ajusta a mà** les regles que no quadrin amb EsportsPulse (ex: noms de serveis, convencions de API).
  * **Lliçó:** L'IA pot generar boilerplate, però **l'arquitectura la dibuixes tu**.
* Configura el fitxer `.gitignore` a l'arrel per excloure brossa de compilació de Java (`target/`, `.class`), la configuració de l'IDE (`.idea/`) i els entorns virtuals de Python.
* Fes el primer *commit* d'estructura i crea la branca `feature/week1-benchmarking`.



---

### **Dimarts: Modelat de Domini Immutable i Col·leccions Bàsiques**

* **Cursos i Material de Lectura:**
* **Curs de NeetCode:** [*Algorithms & Data Structures for Beginners*](https://neetcode.io/courses/dsa-for-beginners/0) — Secció: *Arrays & Dynamic Arrays*.
* **Article:** Baeldung — [*Java 21 Record Keyword*](https://www.baeldung.com/java-record-keyword).
* **Documentació:** Oracle Java Tutorial — [*Collections Framework (List & Map Interfaces)*](https://docs.oracle.com/javase/tutorial/collections/interfaces/index.html).


* **Activitat i Què s'espera programar:**
* **Model de domini (`GameRecord`):** Defineix un tipus immutable usant la sintaxi `record` de Java 21 amb els camps `appId`, nom del joc, preu i jugadors actius.
* **Exploració a petita escala:** Crea manualment una `ArrayList` amb 10-20 jocs i un `HashMap` equivalent. Imprimeix per consola el contingut de les dues estructures per entendre com s'emmagatzemen i s'accedeixen les dades. Experimenta amb `.get()`, `.contains()`, iteració amb `for-each`, i observa la diferència conceptual entre accés per índex i accés per clau.





---

### **Dimecres: Generador de Dades Massives i Lògica de Cerca ($O(n)$ vs $O(1)$)**

* **Cursos i Material de Lectura:**
* **Curs de NeetCode:** [*Algorithms & Data Structures for Beginners*](https://neetcode.io/courses/dsa-for-beginners/0) — Seccions: *Hash Maps* i *Big-O Notation*.
* **Article:** Baeldung — [*Guide to Java HashMap*](https://www.baeldung.com/java-hashmap).
* **Vídeo:** [*Estructuras y Algoritmos de Manera Visual*](https://www.youtube.com/watch?v=2LZanU8UC_A) — MoureDev (YouTube).


* **Activitat i Què s'espera programar:**
* **Generador de dades (`GameDataGenerator`):** Escala el que es va fer dimarts a volum real.
  * Escriu un mètode estàtic que rebi un enter i generi un `ArrayList` amb 100.000 instàncies sintètiques de jocs amb IDs únics i predictibles (ex. `"APP-1"`, `"APP-2"`...).
  * Escriu un segon mètode que converteixi la llista en un `HashMap`, utilitzant l'`appId` com a clau.
* **Servei de cerca (`GameSearchService`):** Crea la classe que contindrà la lògica de comparació.
  * **Mètode lineal ($O(n)$):** Programa una cerca que rebi la llista (`List`) i recorri element per element amb un bucle avaluant si l'ID coincideix fins a trobar-lo o retornar `null`.
  * **Mètode Hash ($O(1)$):** Programa una cerca que rebi el mapa (`Map`) i utilitzi l'accés directe per clau (`.get()`) a la taula de dispersió.



---

### **Dijous: Executable de Benchmarking de Temps i Memòria**

* **Cursos i Material de Lectura:**
* **Article:** Baeldung — [*Microbenchmarking with System.nanoTime*](https://www.baeldung.com/java-system-nanotime).
* **Documentació:** Oracle Java API — [Class `Runtime`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Runtime.html) (`totalMemory()` i `freeMemory()`).


* **Activitat i Què s'espera programar:**
* **Executable (`BenchmarkRunner`):** Crea la classe amb mètode `main` per mesurar l'impacte en producció:
1. Genera les 100.000 instàncies en memòria.
2. Estableix com a objectiu cerca el pitjor cas: l'últim element de la llista (`"APP-100000"`).
3. **Warm-up (important!):** Executa 100 cerques de cada tipus **sense mesurar** per forçar la compilació JIT de la JVM. Explica en un comentari al codi per què cal aquest pas (la JVM interpreta el bytecode les primeres vegades i els resultats serien enganyosos).
4. **Mesura lineal:** Pren el temps inicial en nanosegons (`System.nanoTime()`), executa la cerca lineal 1.000 vegades consecutives per magnificar el resultat i calcula el temps total en mil·lisegons.
5. **Mesura Hash:** Executa exactament les mateixes 1.000 iteracions amb el mètode de mapa i calcula el temps en mil·lisegons.
6. **Mesura de RAM:** Consulta la memòria utilitzada per la JVM mitjançant la classe `Runtime`.
7. Imprimeix per la terminal el quadre comparatiu de ms en lineal vs fracció de ms en Hash Map i els MBs consumits.
8. **Reflexió:** Executa el benchmark sense el warm-up i compara els resultats. Anota en un breu comentari al codi quina diferència hi ha i per què els microbenchmarks ingenus poden ser enganyosos.





---

### **Divendres: Proves Unitàries amb JUnit 5 i Pull Request**

* **Cursos i Material de Lectura:**
* **Documentació:** [*JUnit 5 User Guide*](https://junit.org/junit5/docs/current/user-guide/) — Seccions d'Assercions (`assertEquals`, `assertNull`) i cicle de vida (`@BeforeEach`).
* **Especificació:** [*Conventional Commits Standard*](https://www.conventionalcommits.org/).


* **Activitat i Què s'espera programar:**
* **Proves unitàries (`GameSearchServiceTest`):**
* Programar un test que verifiqui que ambdós mètodes de cerca retornen exactament el mateix objecte quan l'ID existeix.
* Programar un test que verifiqui que ambdós mètodes retornen `null` si l'ID no existeix.
* **Test de rendiment senzill:** Programar un test que mesuri el temps de les dues cerques sobre 100.000 elements i faci un `assertTrue` que la cerca per HashMap és almenys 10x més ràpida que la lineal. Això connecta el testing amb el tema central de la setmana i ensenya que un test pot verificar propietats de rendiment, no només de correcció.


* **Finalització del cicle Git:**
* Executa la suite de proves des de la terminal (`mvn test`).
* Realitza el *commit* final amb format convencional (ex. `feat(java): O(1) vs O(n) benchmarking implementation and JUnit5 tests`).
* Puja la branca a GitHub i realitza la fusió (*merge*) cap a la branca principal `main`.