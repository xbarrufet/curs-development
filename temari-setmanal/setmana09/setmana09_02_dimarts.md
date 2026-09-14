# Setmana 09 — Dimarts: Endpoints CRUD Complets amb JPA

## Objectiu del Dia

Completar tots els endpoints CRUD (Create, Read, Update, Delete) per a l'API de Champions, integrats amb JPA i base de dades H2. Al final del dia, els 5 endpoints han de funcionar amb validació, filtres i paginació.

---

## Teoria

### Arquitectura en Capes

Spring Boot segueix una arquitectura en capes on cada capa té una responsabilitat clara:

```
Controller (Rep peticions HTTP, retorna respostes)
    ↓
Service (Lògica de negoci, validacions complexes)
    ↓
Repository (Accés a base de dades via JPA)
    ↓
Database (H2 en desenvolupament, PostgreSQL en producció)
```

> **Per què capes?** Si demà canviem de H2 a PostgreSQL, només cal tocar la configuració. Si canviem la lògica de negoci, el controller no es veu afectat. Cada capa es pot testejar independentment.

### Configuració de H2 i JPA

```properties
# application.properties — Configuració de la base de dades H2 en memòria
# H2 és una BD lleugera que viu dins la JVM, ideal per desenvolupament
spring.datasource.url=jdbc:h2:mem:esportspulse
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=

# JPA: Hibernate crea les taules automàticament a partir de les entitats
spring.jpa.hibernate.ddl-auto=create-drop
# Mostrem les queries SQL al log per depurar
spring.jpa.show-sql=true

# Consola web de H2: permet veure les taules des del navegador
spring.h2-console.enabled=true
# Accessible a: http://localhost:8080/h2-console
```

### Repository JPA

```java
// === Repository: interfície que JPA implementa automàticament ===
// Extenem JpaRepository, que ens dona CRUD complet sense escriure SQL
// Els tipus genèrics són: <Entitat, Tipus de la clau primària>
@Repository
public interface ChampionRepository extends JpaRepository<Champion, Long> {

    // JPA genera la query automàticament a partir del nom del mètode!
    // findByRole("Mage") → SELECT * FROM champions WHERE role = 'Mage'
    List<Champion> findByRole(String role);

    // Podem combinar condicions amb "And"
    // Busca campions amb un nom que contingui el text i un mínim de partides
    List<Champion> findByNameContainingIgnoreCaseAndTotalGamesGreaterThanEqual(
        String name, int minGames
    );
}
```

### Validació amb Bean Validation

Les anotacions de validació comproven automàticament les dades d'entrada:

```java
// === Request DTO amb validació ===
// Cada camp té anotacions que defineixen les regles de validació
// Si una regla falla, Spring retorna 400 Bad Request automàticament
public record CreateChampionRequest(

    @NotBlank(message = "El nom del campió és obligatori")
    // @NotBlank: no pot ser null, ni buit, ni només espais
    String name,

    @NotBlank(message = "El rol és obligatori")
    String role,

    @Min(value = 0, message = "El win rate no pot ser negatiu")
    @Max(value = 100, message = "El win rate no pot superar 100")
    // @Min/@Max: defineixen el rang de valors acceptats
    double winRate
) {}
```

```java
// === Request DTO per a actualitzacions (PUT) ===
// Similar al de creació, però pot incloure camps addicionals
public record UpdateChampionRequest(

    @NotBlank(message = "El nom del campió és obligatori")
    String name,

    @NotBlank(message = "El rol és obligatori")
    String role,

    @Min(value = 0, message = "El win rate no pot ser negatiu")
    @Max(value = 100, message = "El win rate no pot superar 100")
    double winRate,

    @Min(value = 0, message = "El total de partides no pot ser negatiu")
    int totalGames
) {}
```

### Controller CRUD Complet

```java
@RestController
@RequestMapping("/api/champions")
public class ChampionController {

    private final ChampionService service;

    public ChampionController(ChampionService service) {
        this.service = service;
    }

    // GET /api/champions?name=X&role=Y&minGames=Z
    // @RequestParam amb required=false fa que els filtres siguin opcionals
    // El client pot combinar filtres com vulgui
    @GetMapping
    public ResponseEntity<List<ChampionDTO>> getAll(
            @RequestParam(required = false) String name,
            @RequestParam(required = false) String role,
            @RequestParam(required = false) Integer minGames) {
        List<ChampionDTO> result = service.findWithFilters(name, role, minGames)
            .stream()
            .map(ChampionMapper::toDTO)
            .toList();
        return ResponseEntity.ok(result);
    }

    // GET /api/champions/{id}
    // Retorna 200 si existeix, 404 si no
    @GetMapping("/{id}")
    public ResponseEntity<ChampionDTO> getById(@PathVariable Long id) {
        return service.findById(id)
            .map(ChampionMapper::toDTO)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    // POST /api/champions
    // @Valid activa les validacions del DTO — sense @Valid, les anotacions s'ignoren!
    @PostMapping
    public ResponseEntity<ChampionDTO> create(
            @Valid @RequestBody CreateChampionRequest request) {
        Champion entity = ChampionMapper.toEntity(request);
        Champion saved = service.save(entity);
        // 201 Created amb la URL del recurs creat a la capçalera Location
        URI location = URI.create("/api/champions/" + saved.getId());
        return ResponseEntity.created(location).body(ChampionMapper.toDTO(saved));
    }

    // PUT /api/champions/{id}
    // Actualitza un campió existent. Si no existeix, retorna 404.
    @PutMapping("/{id}")
    public ResponseEntity<ChampionDTO> update(
            @PathVariable Long id,
            @Valid @RequestBody UpdateChampionRequest request) {
        return service.findById(id)
            .map(existing -> {
                // Actualitzem els camps de l'entitat existent amb els valors del request
                existing.setName(request.name());
                existing.setRole(request.role());
                existing.setWinRate(request.winRate());
                existing.setTotalGames(request.totalGames());
                Champion updated = service.save(existing);
                return ResponseEntity.ok(ChampionMapper.toDTO(updated));
            })
            .orElse(ResponseEntity.notFound().build());  // 404 si no existeix
    }

    // DELETE /api/champions/{id}
    // 204 No Content indica èxit sense cos de resposta
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        try {
            service.deleteById(id);
            return ResponseEntity.noContent().build();  // 204: esborrat correctament
        } catch (EntityNotFoundException e) {
            return ResponseEntity.notFound().build();   // 404: no existia
        }
    }
}
```

### Paginació amb Pageable

Quan tens milers de registres, no vols retornar-los tots de cop:

```java
// Endpoint paginat: GET /api/champions/paged?page=0&size=10&sort=name,asc
@GetMapping("/paged")
public ResponseEntity<Page<ChampionDTO>> getAllPaged(
        // Pageable s'injecta automàticament des dels query params
        // page: número de pàgina (0-based)
        // size: elements per pàgina
        // sort: camp i direcció d'ordenació
        Pageable pageable) {

    // repository.findAll(pageable) retorna un Page<Champion>
    Page<ChampionDTO> page = service.findAllPaged(pageable)
        .map(ChampionMapper::toDTO);   // Transforma cada element de la pàgina

    return ResponseEntity.ok(page);
    // La resposta inclou metadades:
    // { "content": [...], "totalElements": 150, "totalPages": 15, "number": 0 }
}
```

```java
// Al Service:
public Page<Champion> findAllPaged(Pageable pageable) {
    return repository.findAll(pageable);  // JpaRepository ja suporta Pageable
}
```

### Gestió d'Errors Global

```java
// === Handler global d'excepcions ===
// @RestControllerAdvice intercepta excepcions de qualsevol controller
// Centralitza la gestió d'errors en un sol lloc
@RestControllerAdvice
public class GlobalExceptionHandler {

    // Captura errors de validació (@Valid que falla)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidation(
            MethodArgumentNotValidException ex) {
        // Recollim tots els errors de validació en un mapa camp → missatge
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error ->
            errors.put(error.getField(), error.getDefaultMessage())
        );
        return ResponseEntity.badRequest().body(errors);  // 400 amb detalls
    }

    // Captura quan no es troba un recurs
    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<Map<String, String>> handleNotFound(
            EntityNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(Map.of("error", ex.getMessage()));
    }
}
```

---

## Activitat

### Part 1: Implementa el CRUD Complet

1. Actualitza `CreateChampionRequest` amb validacions `@Valid`
2. Crea `UpdateChampionRequest` amb validacions
3. Afegeix els endpoints PUT i DELETE al controller
4. Afegeix l'endpoint paginat

### Part 2: Testa amb curl

```bash
# Crea campions, llista, filtra, actualitza, esborra i verifica errors
curl -X POST http://localhost:8080/api/champions \
  -H "Content-Type: application/json" \
  -d '{"name": "Ahri", "role": "Mage", "winRate": 52.3}'

curl http://localhost:8080/api/champions                           # Llista tots
curl "http://localhost:8080/api/champions?role=Mage"               # Filtra per rol
curl -X DELETE http://localhost:8080/api/champions/1               # Esborra
curl -v http://localhost:8080/api/champions/1                      # 404 esperat
curl -X POST http://localhost:8080/api/champions \
  -H "Content-Type: application/json" \
  -d '{"name": "", "role": "Mage", "winRate": 150}'               # 400 esperat
```

### Part 3: Implementa el GlobalExceptionHandler

Crea la classe `GlobalExceptionHandler` per gestionar errors de forma uniforme.

---

## Checklist de Lliurament

- [ ] Els 5 endpoints funcionen: GET (llista), GET (per id), POST, PUT, DELETE
- [ ] La validació retorna 400 amb missatges clars quan les dades no són vàlides
- [ ] GET d'un ID inexistent retorna 404
- [ ] DELETE d'un ID inexistent retorna 404
- [ ] L'endpoint paginat retorna metadades (`totalElements`, `totalPages`)
- [ ] `GlobalExceptionHandler` gestiona errors de validació i entity not found
- [ ] Commit: `feat(api): complete CRUD endpoints with validation and pagination`
