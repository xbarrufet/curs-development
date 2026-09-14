**Setmana 7: APIs REST amb Spring Boot 3 i Consultes a BD**

---

### **Dilluns: DTOs, Contractes REST i Exceptions**

* **Cursos i Material de Lectura:**
* **Article:** Baeldung — [*Spring Boot REST Controllers*](https://www.baeldung.com/spring-boot-rest-controller).
* **Article:** Baeldung — [*DTO Pattern in Spring*](https://www.baeldung.com/entity-to-dto).
* **Article:** Baeldung — [*Spring Exception Handling*](https://www.baeldung.com/exception-handling-for-rest-with-spring).
* **Documentació:** Spring — [*Building REST Services*](https://spring.io/guides/gs/rest-service/).


* **Activitat i Què s'espera programar:**
* Entendre la separació: `@Entity` (BD) vs `DTO` (API contract).
* **Exercici d'Escriptura de Specs: API Spec en Markdown.**
  * Abans de programar res, escriu la spec de l'API en un fitxer `api-spec.md`:
    ```markdown
    ## GET /games/{appId}
    Retorna les dades d'un joc per ID.
    - **Response 200:**
      ```json
      { "appId": "APP-1", "title": "League of Legends", "price": 0.00, "activePlayerCount": 5000000 }
      ```
    - **Response 404:** `{ "error": "Game APP-999 not found", "status": 404 }`

    ## POST /games
    Crea un joc nou.
    - **Request body:**
      ```json
      { "title": "Nou Joc", "price": 29.99 }
      ```
    - **Validació:** `title` obligatori (no buit), `price` >= 0.
    - **Response 201:** Retorna el `GameDTO` creat amb `appId` generat.
    - **Response 400:** Si validació falla, retorna errors per camp.

    ## GET /games?title=X&minPlayers=Y
    Cerca amb filtres opcionals. Retorna llista de `GameDTO`.

    ## PUT /games/{appId}
    Actualitza camps d'un joc. Response 200 o 404.

    ## DELETE /games/{appId}
    Elimina un joc. Response 204 o 404.
    ```
  * Dona la spec a Cursor i demana: *"Genera els DTOs, Mapper, Exception Handler i Controller seguint exactament aquesta API spec."*
  * Verifica que el codi generat compleix la spec: fes requests amb curl/Postman i compara input/output amb els exemples de la spec.
  * Si el codi no compleix la spec, **itera la spec** (afegir detalls que faltaven) i regenera.
  * **Lliçó:** Definir el contracte ABANS de programar és el workflow professional. A una empresa, l'API spec (OpenAPI/Swagger) es pacta entre frontend i backend abans d'escriure una línia.
* **Crear DTOs (a mà o validant el generat):**
  * `GameDTO` (public response): `appId`, `title`, `price`, `activePlayerCount`.
  * `CreateGameRequest` (POST payload): `title`, `price`.
  * `UpdateGameRequest` (PUT payload): camps opcionals.
* **Mapper (manual o usar MapStruct):**
  * Mètode `toDTO(GameRecord entity)`: converte entity en DTO.
  * Mètode `toEntity(CreateGameRequest request)`: crea entity de request.
* **Global Exception Handler:**
  * `@ControllerAdvice` que captura `EntityNotFoundException`, `ValidationException`.
  * Retorna `ErrorResponse` amb HTTP status apropiat (404, 400, 500).


---

### **Dimarts: Controllers REST - CRUD Bàsic**

* **Cursos i Material de Lectura:**
* **Article:** Baeldung — [*Spring @RequestMapping*](https://www.baeldung.com/spring-requestmapping).
* **Article:** Baeldung — [*HTTP Status Codes*](https://www.baeldung.com/spring-boot-http-responses).


* **Activitat i Què s'espera programar:**
* **Crear `GameController`:**
  * Injecta `GameManagementService` (que a la seva vegada injecta `GameJpaRepository` de S5).
  * `@GetMapping("/games/{appId}")`: retorna `GameDTO` o 404 si no existeix.
  * `@GetMapping("/games")`: retorna `List<GameDTO>` amb paginació opcional (`?page=0&size=10`).
  * `@PostMapping("/games")`: accepta `CreateGameRequest`, persisten via service, retorna `GameDTO` + 201 Created.
  * `@PutMapping("/games/{appId}")`: actualitza i retorna `GameDTO` o 404.
  * `@DeleteMapping("/games/{appId}")`: elimina i retorna 204 No Content.
* **Query Parameters:**
  * `GET /games?title=Yasuo`: filtra per títol.
  * `GET /games?minPlayers=100000`: filtra per jugadors mínims.
* **Validació:**
  * `@Valid` a DTOs amb `@NotBlank`, `@Min`, etc.
  * Retorna 400 si validation fails.


---

### **Dimecres: Integració amb la BD i Testing**

* **Cursos i Material de Lectura:**
* **Article:** Baeldung — [*Testing Spring Boot REST*](https://www.baeldung.com/integration-testing-in-spring).
* **Article:** Baeldung — [*MockMvc vs WebTestClient*](https://www.baeldung.com/spring-boot-webflux-webclient).


* **Activitat i Què s'espera programar:**
* **Verificar que els endpoints consulten la BD (S5):**
  * El `@GetMapping("/games")` executa `gameRepository.findAll()` i retorna DTOs.
  * El `@PostMapping("/games")` executa `gameRepository.save()`.
  * Prova manualment amb Postman o curl.
* **Crear tests (`GameControllerTests`):**
  * Usar `@WebMvcTest` per testejar controladors isolats.
  * Mock `GameManagementService`.
  * Test: `GET /games/APP-123` retorna 200 + DTO.
  * Test: `GET /games/NONEXISTENT` retorna 404 + ErrorResponse.
  * Test: `POST /games` amb dades vàlides retorna 201.
  * Test: `POST /games` amb dades invàlides retorna 400.
* **Integració Tests (`GameControllerIT`):**
  * Usar `@SpringBootTest` + `MockMvc` o `TestRestTemplate`.
  * Test end-to-end: POST game → GET per appId → verifica que apareix.
  * Test: DELETE → GET retorna 404.


---

### **Dijous: Virtual Threads, CLI Consumidor i Documentació**

* **Cursos i Material de Lectura:**
* **Article:** Baeldung — [*Virtual Threads in Java 21*](https://www.baeldung.com/java-virtual-thread-vs-thread).
* **Article:** Inside Java — [*JEP 444: Virtual Threads*](https://openjdk.org/jeps/444).
* **Article:** Baeldung — [*Building a CLI with Spring Boot*](https://www.baeldung.com/spring-boot-cli).
* **Documentació:** [*Python Requests Library*](https://docs.python-requests.org/).


* **Activitat i Què s'espera programar:**
* **Virtual Threads (Java 21) — Millorar el rendiment dels endpoints:**
  * **El problema real:** L'endpoint `GET /games/{appId}` crida Steam API (300ms) i RAWG API (400ms). Amb `CompletableFuture` (S3) ho hem paral·lelitzat, però el thread pool de Tomcat (200 threads) limita la concurrència: 200 requests lentes → pool esgotat → les següents esperen.
  * **Teoria: Virtual Threads.** Threads gestionats per la JVM, no pel SO. Cada un ocupa ~1KB (vs ~1MB d'un platform thread). Pots tenir milions de threads "barats".
  * **Exercici pràctic — Extractor amb 3 versions:**
    * **Versió 1 — Thread pool clàssic:** `Executors.newFixedThreadPool(10)` per extreure dades de 50 jocs. Mesura temps.
    * **Versió 2 — Virtual Threads:** `Executors.newVirtualThreadPerTaskExecutor()`. Mesura temps.
    * **Versió 3 — Spring Boot integrat:** Afegir `spring.threads.virtual.enabled=true` a `application.properties`. Ara cada petició HTTP al teu endpoint usa un Virtual Thread automàticament. Mesura el throughput amb múltiples requests simultànies.
  * **Connexió amb S3:** Els problemes de race conditions i `@Transactional` **segueixen existint** amb Virtual Threads. No és una bala de plata — és una optimització d'I/O.
* **Ampliar CLI `gamepulse` (Python) per consumir l'API REST:**
  * Setup base de la CLI (basic `typer` setup).
  * Comandos: `gamepulse list-games`, `gamepulse get-game APP-123`, `gamepulse create-game --title "X" --price 29.99`.
  * Client HTTP: usar `requests` library en Python per fer requests a `http://localhost:8080/games/...`.
  * Formatar output: taules, colors amb `rich` (optional).
* **Documentació:**
  * Afegir a README: "API Reference" amb exemples curl.
  * Afegir a README: "CLI Quickstart" amb exemples.


---

### **Divendres: Swagger/OpenAPI i PR Final**

* **Cursos i Material de Lectura:**
* **Article:** Baeldung — [*Swagger 2 with Spring Boot*](https://www.baeldung.com/swagger-2-documentation-for-spring-rest-api).
* **Documentació:** [*SpringDoc OpenAPI*](https://springdoc.org/).


* **Activitat i Què s'espera programar:**
* **Afegir Swagger (OpenAPI):**
  * Dependència: `springdoc-openapi-starter-webmvc-ui`.
  * Afegir `@Operation`, `@Schema` anotations als endpoints.
  * Accedeix a `http://localhost:8080/swagger-ui.html` per veure API documentada automàticament.
* **Consolidació CI/CD:**
  * Tests REST passen (unit + integration).
  * GitHub Actions: `mvn test` + cobertura gates.
  * Commit message: `feat(java): REST API endpoints with database queries, DTOs, error handling, Swagger documentation`
* **Demo i Merge:**
  * Arranca l'app: `mvn spring-boot:run`.
  * Prova manualment: POST game, GET, DELETE.
  * Prova CLI: `python -m gamepulse list-games`.
  * Puja la branca `feature/week7-rest-api` a GitHub.
  * Verifica que tot és verd.
  * Fes merge a `main`.

---

## Nota sobre Progressió Persistència

- **S2/4:** Repository pattern abstracte (In-Memory).
- **S5:** JPA real amb H2 (swap implementation, no changes to `GameManagementService`).
- **S7:** REST endpoints que consulten la BD via `GameManagementService`.

Gràcies al pattern, el developer va de "conceptual" a "real DB" sense esforç de refactorització major.
