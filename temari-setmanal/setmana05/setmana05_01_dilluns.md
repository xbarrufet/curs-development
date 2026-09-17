# Setmana 05 — Dilluns: Principis de Disseny d'APIs REST

## Objectiu del Dia

Entendre els fonaments del disseny d'APIs REST i aplicar-los al projecte EsportsPulse. Al final del dia tindras DTOs separats de les entitats, un mapper funcional i un controlador Spring Boot que compili i respongui peticions HTTP bàsiques.

---

## Teoria

### Què és una API REST?

REST (Representational State Transfer) és un estil d'arquitectura per a serveis web. Les seves idees clau són:

1. **Recursos**: Tot és un recurs identificat per una URL (`/api/champions`, `/api/champions/42`)
2. **Mètodes HTTP**: Cada operació utilitza el verb HTTP adequat
3. **Sense estat**: Cada petició conté tota la informació necessària; el servidor no recorda peticions anteriors

### Mètodes HTTP i Significat

| Mètode   | Acció                  | Exemple                     | Cos de la Petició? |
|----------|------------------------|-----------------------------|---------------------|
| `GET`    | Llegir recurs(os)      | `GET /api/champions`        | No                  |
| `POST`   | Crear recurs nou       | `POST /api/champions`       | Sí                  |
| `PUT`    | Actualitzar recurs     | `PUT /api/champions/42`     | Sí                  |
| `DELETE` | Esborrar recurs        | `DELETE /api/champions/42`  | No                  |

### Codis d'Estat HTTP

Els codis d'estat comuniquen el resultat de l'operació al client:

```
// Codis d'èxit
200 OK            → La petició s'ha processat correctament (GET, PUT)
201 Created       → S'ha creat un recurs nou (POST)
204 No Content    → Operació correcta sense cos de resposta (DELETE)

// Codis d'error del client
400 Bad Request   → Les dades enviades no són vàlides (validació fallida)
404 Not Found     → El recurs sol·licitat no existeix

// Codis d'error del servidor
500 Internal Server Error → Error inesperat al servidor
```

> **Regla d'or**: El client mai ha d'endevinar què ha passat. El codi d'estat i el cos de la resposta han de ser suficients per entendre el resultat.

### DTOs: Separar l'Entitat del Contracte de l'API

Un error habitual és retornar directament l'entitat JPA com a resposta de l'API. Això crea un acoblament perillós: qualsevol canvi a la base de dades trenca els clients de l'API.

**Per què cal separar?**
- L'entitat pot tenir camps interns que no volem exposar (id tècnic, timestamps d'auditoria)
- El format de l'API pot ser diferent del de la base de dades
- Podem evolucionar l'API i la BD independentment

```java
// === Entitat JPA: representa la taula a la base de dades ===
// Aquesta classe mapeja directament a la taula "champions" de la BD
@Entity
@Table(name = "champions")
public class Champion {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;              // Clau primària autogenerada
    private String name;          // Nom del campió (ex: "Ahri")
    private String role;          // Rol principal (ex: "Mage")
    private double winRate;       // Percentatge de victòries (0.0 - 100.0)
    private int totalGames;       // Nombre total de partides jugades

    // Getters i setters omesos per brevetat
}
```

```java
// === DTO de resposta: el que el client rep ===
// Usem un "record" de Java 21 — immutable i concís
// Només exposem els camps que el client necessita
public record ChampionDTO(
    Long id,           // Identificador públic del campió
    String name,       // Nom del campió
    String role,       // Rol principal
    double winRate,    // Percentatge de victòries
    int totalGames    // Total de partides registrades
) {}
```

```java
// === DTO de petició: el que el client envia per crear un campió ===
// No inclou "id" perquè el servidor el genera automàticament
// No inclou "totalGames" perquè comença a 0
public record CreateChampionRequest(
    String name,       // Nom del campió a crear
    String role,       // Rol assignat
    double winRate     // Win rate inicial
) {}
```

### El Patró Mapper

El Mapper és la classe que converteix entre entitat i DTO. Centralitzar aquesta lògica evita duplicar codi de conversió a tot arreu.

```java
// === Mapper: converteix entre entitat i DTOs ===
// Classe utilitària amb mètodes estàtics per simplicitat
public class ChampionMapper {

    // Converteix una entitat JPA a un DTO de resposta
    // S'usa quan retornem dades al client
    public static ChampionDTO toDTO(Champion entity) {
        return new ChampionDTO(
            entity.getId(),
            entity.getName(),
            entity.getRole(),
            entity.getWinRate(),
            entity.getTotalGames()
        );
    }

    // Converteix un DTO de creació a una entitat JPA
    // S'usa quan el client envia dades per crear un campió
    public static Champion toEntity(CreateChampionRequest request) {
        Champion champion = new Champion();
        champion.setName(request.name());       // Assignem el nom del request
        champion.setRole(request.role());       // Assignem el rol del request
        champion.setWinRate(request.winRate()); // Assignem el win rate inicial
        champion.setTotalGames(0);              // Un campió nou comença amb 0 partides
        return champion;
    }
}
```

### Spring Boot Controllers

Spring Boot utilitza anotacions per definir endpoints HTTP:

```java
// === Controlador REST per a Champions ===
// @RestController indica que tots els mètodes retornen dades (JSON), no vistes HTML
// @RequestMapping estableix el prefix comú per a totes les rutes d'aquest controlador
@RestController
@RequestMapping("/api/champions")
public class ChampionController {

    // Injectem el servei que conté la lògica de negoci
    private final ChampionService service;

    // Constructor injection: Spring Boot injecta automàticament el servei
    public ChampionController(ChampionService service) {
        this.service = service;
    }

    // GET /api/champions → Retorna la llista de tots els campions
    // ResponseEntity ens permet controlar el codi d'estat HTTP
    @GetMapping
    public ResponseEntity<List<ChampionDTO>> getAll() {
        // Obtenim les entitats, les convertim a DTOs i retornem amb 200 OK
        List<ChampionDTO> champions = service.findAll()
            .stream()
            .map(ChampionMapper::toDTO)    // Converteix cada entitat a DTO
            .toList();                      // Recull en una llista
        return ResponseEntity.ok(champions); // 200 OK amb la llista
    }

    // GET /api/champions/{id} → Retorna un campió concret pel seu ID
    // @PathVariable extreu el valor de la URL (ex: /api/champions/42 → id=42)
    @GetMapping("/{id}")
    public ResponseEntity<ChampionDTO> getById(@PathVariable Long id) {
        return service.findById(id)
            .map(ChampionMapper::toDTO)                        // Si existeix, convertim a DTO
            .map(ResponseEntity::ok)                           // Emboliquem amb 200 OK
            .orElse(ResponseEntity.notFound().build());        // Si no existeix, 404
    }

    // POST /api/champions → Crea un campió nou
    // @RequestBody indica que el cos de la petició JSON es deserialitza al record
    @PostMapping
    public ResponseEntity<ChampionDTO> create(@RequestBody CreateChampionRequest request) {
        // Convertim el request a entitat, el guardem, i retornem el DTO creat
        Champion entity = ChampionMapper.toEntity(request);
        Champion saved = service.save(entity);
        ChampionDTO dto = ChampionMapper.toDTO(saved);
        // 201 Created és el codi correcte per a creació de recursos
        return ResponseEntity.status(HttpStatus.CREATED).body(dto);
    }
}
```

### Flux Complet d'una Petició

```
Client (Postman/curl)
    ↓ POST /api/champions  { "name": "Ahri", "role": "Mage", "winRate": 52.3 }
    ↓
ChampionController.create()
    ↓ Rep CreateChampionRequest
    ↓
ChampionMapper.toEntity()
    ↓ Converteix request → entitat JPA
    ↓
ChampionService.save()
    ↓ Lògica de negoci + persistència
    ↓
ChampionMapper.toDTO()
    ↓ Converteix entitat guardada → DTO de resposta
    ↓
Client rep: 201 Created { "id": 1, "name": "Ahri", "role": "Mage", "winRate": 52.3, "totalGames": 0 }
```

---

## Activitat

### Part 1: Escriu l'especificació de l'API (api-spec.md)

Abans d'escriure codi, documenta el que construiràs. Crea `docs/api-spec.md` amb:

```markdown
# Champions API — Especificació

## Endpoints

### GET /api/champions
- Descripció: Retorna tots els campions
- Resposta: 200 OK — Array de ChampionDTO

### GET /api/champions/{id}
- Descripció: Retorna un campió pel seu ID
- Resposta: 200 OK — ChampionDTO
- Error: 404 Not Found — si l'ID no existeix

### POST /api/champions
- Descripció: Crea un campió nou
- Cos: CreateChampionRequest (name, role, winRate)
- Resposta: 201 Created — ChampionDTO creat
- Error: 400 Bad Request — si les dades no són vàlides
```

### Part 2: Implementa els DTOs i el Mapper

1. Crea els fitxers `ChampionDTO.java`, `CreateChampionRequest.java` i `ChampionMapper.java`
2. Col·loca'ls al paquet `com.esportspulse.engine.dto` (els DTOs) i `com.esportspulse.engine.mapper` (el mapper)

### Part 3: Crea el Controlador

1. Crea `ChampionController.java` al paquet `com.esportspulse.engine.controller`
2. Implementa `GET /api/champions`, `GET /api/champions/{id}` i `POST /api/champions`
3. Comprova que `mvn compile` funciona sense errors

### Part 4: Verifica amb curl

```bash
# Arrenca l'aplicació
mvn spring-boot:run

# Crea un campió
curl -X POST http://localhost:8080/api/champions \
  -H "Content-Type: application/json" \
  -d '{"name": "Ahri", "role": "Mage", "winRate": 52.3}'

# Llista tots els campions
curl http://localhost:8080/api/champions
```

---

## Checklist de Lliurament

- [ ] Fitxer `docs/api-spec.md` escrit amb tots els endpoints documentats
- [ ] Records `ChampionDTO` i `CreateChampionRequest` creats al paquet `dto`
- [ ] Classe `ChampionMapper` amb mètodes `toDTO()` i `toEntity()`
- [ ] `ChampionController` amb `@RestController` i endpoints GET/POST
- [ ] `mvn compile` passa sense errors
- [ ] Commit: `feat(api): add champion DTOs, mapper and REST controller`
