**Setmana 5: Factory, Repository Patterns i Spring Data JPA amb H2**

---

### **Dilluns: Spring Data JPA i H2 Basics**

* **Cursos i Material de Lectura:**
* **Article:** Baeldung — [*Spring Data JPA Tutorial*](https://www.baeldung.com/the-persistence-layer-with-spring-data-jpa).
* **Article:** Baeldung — [*H2 Database*](https://www.baeldung.com/h2-database).
* **Documentació:** Spring — [*Spring Data JPA Reference*](https://spring.io/projects/spring-data-jpa).
* **Vídeo:** [*JPA & Hibernate Basics*](https://www.youtube.com/results?search_query=spring+data+jpa+tutorial).


* **Activitat i Què s'espera programar:**
* Afegir dependències Maven: `spring-boot-starter-data-jpa`, `com.h2database:h2`.
* Configurar `application.properties`: `spring.datasource.url=jdbc:h2:mem:esportspulse`, `spring.h2.console.enabled=true`.
* Entendre la diferència entre In-Memory (S2) i BD relacional: transaccions, ACID, queries SQL.
* **Exercici de Prompt Engineering:** Pregunta a Cursor: *"Explica quan és millor JPA que In-Memory i quins são els overhead"*. Reflexiona sobre trade-offs.


---

### **Dimarts: ChampionRecord com a Entity i JpaRepository**

* **Cursos i Material de Lectura:**
* **Article:** Baeldung — [*JPA Entities*](https://www.baeldung.com/jpa-entities).
* **Article:** Baeldung — [*Spring Data JPA Repository*](https://www.baeldung.com/spring-data-jpa-query).


* **Activitat i Què s'espera programar:**
* **Convertir `ChampionRecord` en `@Entity`:**
  * Afegir anotacions: `@Entity`, `@Table(name = "champions")`.
  * `@Id` a `championId` (String, primary key).
  * `@Column` annotations si necessari (ex: `@Column(nullable = false)` a `name`).
  * Mantenir el constructor (per a JPA cal un no-arg constructor; usar Lombok `@NoArgsConstructor` si es necessita).
  * **Important:** Discussió sobre mutabilitat: entities de JPA són mutables per defecte. Alternativi: usar `@Immutable` si vols mantenir immutabilitat.
* **Crear `ChampionJpaRepository` extends `JpaRepository<ChampionRecord, String>`:**
  * Hereda `save()`, `findById()`, `findAll()`, `delete()`.
  * Afegir query derivada: `List<ChampionRecord> findByNameContaining(String title)`.
  * Afegir query derivada: `List<ChampionRecord> findByGamesPlayedGreaterThan(Long count)`.
* **Prova manualment amb H2 Console:** Accedeix a `http://localhost:8080/h2-console`, verifica que la taula `champions` es va crear.


---

### **Dimecres: Service Layer i Queries Bàsiques**

* **Cursos i Material de Lectura:**
* **Article:** Baeldung — [*@Query Annotation*](https://www.baeldung.com/spring-data-jpa-query).
* **Documentació:** Oracle — [*SQL Basics*](https://docs.oracle.com/cd/B19306_01/server.102/b14200/sqlyntax.htm) (opcional).


* **Activitat i Què s'espera programar:**
* **Ampliar `ChampionManagementService` (de S4):**
  * Injecta `ChampionJpaRepository` en lloc de `ChampionRepository` in-memory.
  * Mètode `registerChampion()`: persisten al `@Entity`.
  * Mètode `searchByName(String keyword)`: usa `findByNameContaining()`.
  * Mètode `getMetaChampions()`: usa `findByGamesPlayedGreaterThan(100_000)`.
  * Mètode `getAllChampions()`: retorna `findAll()`.
* **Consultes JPQL opcionals:**
  * Afegir a `ChampionJpaRepository`: `@Query("SELECT c FROM ChampionRecord c WHERE c.winRate > :minWinRate")` per a queries més complexes.
* **Transaccions:**
  * Afegir `@Transactional` a mètodes que escriben (per seguretat ACID).
  * Discutió: Rollback automàtic en excepcions.


---

### **Dijous: Migració de In-Memory a JPA i Testing**

* **Cursos i Material de Lectura:**
* **Article:** Baeldung — [*Testing Spring Data JPA*](https://www.baeldung.com/spring-boot-testing-h2-database).
* **Article:** Baeldung — [*@DataJpaTest*](https://www.baeldung.com/spring-boot-testing-h2-database).


* **Activitat i Què s'espera programar:**
* **Refactorització del codi S4:**
  * Reemplaçar `InMemoryChampionRepository` per `ChampionJpaRepository` a `ChampionManagementService`.
  * Assegurar que els tests S2/S4 segueixen passant (interfaces are key).
* **Nous tests per a la capa JPA (`ChampionJpaRepositoryTests`):**
  * Usar `@DataJpaTest` per testejar només la capa de dades.
  * Test: `save()` i `findById()` retorna el mateix.
  * Test: `findByNameContaining()` amb wildcards.
  * Test: `findByGamesPlayedGreaterThan()` filtra correctament.
  * Test: `findAll()` retorna tots els records.
* **Tots els tests vell (S2/S4) han de passar sense canvis gràcies al pattern Repository.**


---

### **Divendres: Integració End-to-End i PR**

* **Cursos i Material de Lectura:**
* **Article:** Baeldung — [*Spring Boot Integration Tests*](https://www.baeldung.com/spring-boot-testing-h2-database).


* **Activitat i Què s'espera programar:**
* **Test d'integració (`ChampionManagementServiceIT`):**
  * Inicia context Spring sencer (`@SpringBootTest`).
  * Test: `registerChampion()` → `searchByName()` → verifica que apareix.
  * Test: `getMetaChampions()` amb dades que cumplen/no cumplen criteris.
* **Cobertura i CI:**
  * Executar `mvn test` i verificar cobertura >= 80%.
  * GitHub Actions workflow actualitzat: passa tots els tests (JPA + In-Memory legacy).
  * Commit message: `feat(java): Spring Data JPA with H2, ChampionRecord @Entity, Repository pattern integration`
* **Revisió d'arquitectura:**
  * Dibuixa en comentaris: `InMemoryChampionRepository` → `ChampionJpaRepository` swap sense canviar `ChampionManagementService`. Això és la **força de patterns**.
  * Afegir a `.cursorrules`: regles per a entities (immutability trade-offs, `@Transactional`, indices).


* **Finalització del cicle Git:**
* Puja la branca `feature/week5-jpa-h2` a GitHub.
* Verifica que el workflow passa (tests + coverage).
* Fes merge a `main`.

---

## Vídeos Recomanats

- **Spring Data JPA:** Cerca "TodoCode Spring Data JPA" o "MitoCode JPA Hibernate" (castellà). En anglès: "Amigoscode Spring Data JPA tutorial" (complet, pas a pas).
- **Repository Pattern:** Cerca "CodelyTV Repository Pattern" (castellà, explica bé la motivació darrere el patró).
- **H2 Database:** Cerca "Spring Boot H2 database tutorial" — qualsevol vídeo curt que mostri la consola H2 i com inspeccionar dades.
- **Factory Pattern:** Cerca "Refactoring Guru Factory Pattern" o "CodelyTV patrones de diseño Factory" (castellà).

---

## Nota sobre Persistència

Setmana 5 és **la darrera del Bloc 1**. Després (S7+), els endpoints REST consultaran aquesta BD. Per tant:
- **S2-4:** Patterns + In-Memory
- **S5:** JPA real
- **S6:** Tests de qualitat (cobertura gates)
- **S7:** REST endpoints que usen la BD
