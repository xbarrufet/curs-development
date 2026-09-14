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
    ## GET /champions/{championId}
    Retorna les dades d'un champion per ID.
    - **Response 200:**
      ```json
      { "championId": "CHAMP-1", "name": "Jinx", "winRate": 52.3, "gamesPlayed": 150000 }
      ```
    - **Response 404:** `{ "error": "Champion CHAMP-999 not found", "status": 404 }`

    ## POST /champions
    Crea un champion nou.
    - **Request body:**
      ```json
      { "name": "Yasuo", "winRate": 49.5 }
      ```
    - **Validació:** `name` obligatori (no buit), `winRate` entre 0 i 100.
    - **Response 201:** Retorna el `ChampionDTO` creat amb `championId` generat.
    - **Response 400:** Si validació falla, retorna errors per camp.

    ## GET /champions?name=X&minGames=Y
    Cerca amb filtres opcionals. Retorna llista de `ChampionDTO`.

    ## PUT /champions/{championId}
    Actualitza camps d'un champion. Response 200 o 404.

    ## DELETE /champions/{championId}
    Elimina un champion. Response 204 o 404.
    ```
  * Dona la spec a Cursor i demana: *"Genera els DTOs, Mapper, Exception Handler i Controller seguint exactament aquesta API spec."*
  * Verifica que el codi generat compleix la spec: fes requests amb curl/Postman i compara input/output amb els exemples de la spec.
  * Si el codi no compleix la spec, **itera la spec** (afegir detalls que faltaven) i regenera.
  * **Lliçó:** Definir el contracte ABANS de programar és el workflow professional. A una empresa, l'API spec (OpenAPI/Swagger) es pacta entre frontend i backend abans d'escriure una línia.
* **Crear DTOs (a mà o validant el generat):**
  * `ChampionDTO` (public response): `championId`, `name`, `winRate`, `gamesPlayed`.
  * `CreateChampionRequest` (POST payload): `name`, `winRate`.
  * `UpdateChampionRequest` (PUT payload): camps opcionals.
* **Mapper (manual o usar MapStruct):**
  * Mètode `toDTO(ChampionRecord entity)`: converte entity en DTO.
  * Mètode `toEntity(CreateChampionRequest request)`: crea entity de request.
* **Global Exception Handler:**
  * `@ControllerAdvice` que captura `EntityNotFoundException`, `ValidationException`.
  * Retorna `ErrorResponse` amb HTTP status apropiat (404, 400, 500).


---

### **Dimarts: Controllers REST - CRUD Bàsic**

* **Cursos i Material de Lectura:**
* **Article:** Baeldung — [*Spring @RequestMapping*](https://www.baeldung.com/spring-requestmapping).
* **Article:** Baeldung — [*HTTP Status Codes*](https://www.baeldung.com/spring-boot-http-responses).


* **Activitat i Què s'espera programar:**
* **Crear `ChampionController`:**
  * Injecta `ChampionManagementService` (que a la seva vegada injecta `ChampionJpaRepository` de S5).
  * `@GetMapping("/champions/{championId}")`: retorna `ChampionDTO` o 404 si no existeix.
  * `@GetMapping("/champions")`: retorna `List<ChampionDTO>` amb paginació opcional (`?page=0&size=10`).
  * `@PostMapping("/champions")`: accepta `CreateChampionRequest`, persisten via service, retorna `ChampionDTO` + 201 Created.
  * `@PutMapping("/champions/{championId}")`: actualitza i retorna `ChampionDTO` o 404.
  * `@DeleteMapping("/champions/{championId}")`: elimina i retorna 204 No Content.
* **Query Parameters:**
  * `GET /champions?name=Yasuo`: filtra per nom.
  * `GET /champions?minGames=100000`: filtra per partides mínimes.
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
  * El `@GetMapping("/champions")` executa `championRepository.findAll()` i retorna DTOs.
  * El `@PostMapping("/champions")` executa `championRepository.save()`.
  * Prova manualment amb Postman o curl.
* **Crear tests (`ChampionControllerTests`):**
  * Usar `@WebMvcTest` per testejar controladors isolats.
  * Mock `ChampionManagementService`.
  * Test: `GET /champions/CHAMP-123` retorna 200 + DTO.
  * Test: `GET /champions/NONEXISTENT` retorna 404 + ErrorResponse.
  * Test: `POST /champions` amb dades vàlides retorna 201.
  * Test: `POST /champions` amb dades invàlides retorna 400.
* **Integració Tests (`ChampionControllerIT`):**
  * Usar `@SpringBootTest` + `MockMvc` o `TestRestTemplate`.
  * Test end-to-end: POST champion → GET per championId → verifica que apareix.
  * Test: DELETE → GET retorna 404.


---

### **Dijous: CLI Consumidor (Python) i Documentació**

* **Cursos i Material de Lectura:**
* **Documentació:** [*Python Requests Library*](https://docs.python-requests.org/).
* **Article:** Baeldung — [*Building a CLI with Spring Boot*](https://www.baeldung.com/spring-boot-cli).


* **Activitat i Què s'espera programar:**
* **Ampliar CLI `esportspulse` (Python) per consumir l'API REST:**
  * Setup base de la CLI (basic `typer` setup).
  * Comandos: `esportspulse list-champions`, `esportspulse get-champion CHAMP-123`, `esportspulse create-champion --name "Jinx" --winRate 52.3`.
  * Client HTTP: usar `requests` library en Python per fer requests a `http://localhost:8080/champions/...`.
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
  * Prova manualment: POST champion, GET, DELETE.
  * Prova CLI: `python -m esportspulse list-champions`.
  * Puja la branca `feature/week7-rest-api` a GitHub.
  * Verifica que tot és verd.
  * Fes merge a `main`.

---

## Vídeos Recomanats

- **REST amb Spring Boot:** Cerca "TodoCode Spring Boot REST API" o "MitoCode Spring Boot CRUD" (castellà). En anglès: "Amigoscode Spring Boot REST API" (curs complet, molt clar).
- **DTOs i Mappers:** Cerca "CodelyTV DTO pattern" (castellà) o "Bouali Ali Spring Boot DTO tutorial" (anglès).
- **Swagger/OpenAPI:** Cerca "Spring Boot Swagger tutorial" o "Daily Code Buffer Spring Boot OpenAPI" (anglès, curt i directe).
- **Testing REST:** Cerca "Java Brains Spring Boot MockMvc" o "Amigoscode Spring Boot testing" (anglès).

---

## Nota sobre Progressió Persistència

- **S2/4:** Repository pattern abstracte (In-Memory).
- **S5:** JPA real amb H2 (swap implementation, no changes to `ChampionManagementService`).
- **S7:** REST endpoints que consulten la BD via `ChampionManagementService`.

Gràcies al pattern, el developer va de "conceptual" a "real DB" sense esforç de refactorització major.
