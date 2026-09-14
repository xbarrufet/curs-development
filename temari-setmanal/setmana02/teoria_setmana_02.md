# Setmana 2 - Teoria: POO, SOLID i Immutabilitat

## 1. Introducció: Per Què SOLID?

Imagina el codi de GamePulse sense SOLID:

```java
// ❌ SENSE SOLID - pesadilla de manteniment
public class GameService {
    private List<GameRecord> games;
    private Database db;
    private EmailService emailService;
    private APIClient steamAPI;
    private AnalyticsTracker tracker;
    
    public void registerGame(String appId, String title) {
        // Valida
        if (appId == null) throw new Exception("Invalid");
        
        // Crea
        GameRecord g = new GameRecord(appId, title, 0, 0);
        
        // Guarda a llocs múltiples
        games.add(g);
        db.save(g);
        
        // Envia email
        emailService.notifyAdmins("New game: " + title);
        
        // Trackeja
        tracker.log("game_registered", g.appId());
        
        // Si Steam API falla... que passa? Tot falla.
    }
}
```

**Problemes:**
- `GameService` fa 5 coses: validació, creació, persistència, notificació, tracking
- Una change a qualsivolol comportament afecta tot
- Test `registerGame` necessita mock de 5 serveis
- Reutilitzar lógica de creació a altre lloc? Imposible, està acoblada.

---

## 2. SOLID Principles: Els 5 Pilars

### S - Single Responsibility Principle (SRP)

**Definició:** Una classe hauria de tenir una **única raó per canviar**.

**Aplicat a GamePulse:**

```java
// ✅ BIEN - Cada classe té una responsabilitat

// 1. Creació de GameRecords
public class GameRecordFactory {
    public GameRecord createFromSteamAPI(String appId, String json) {
        // Solo parse + validate + create
        return new GameRecord(appId, title, price, players);
    }
}

// 2. Persistència
public interface GameRepository {
    void save(GameRecord g);
    GameRecord findById(String appId);
}

// 3. Lógica de negoci
public class GameManagementService {
    private GameRepository repo;
    private GameRecordFactory factory;
    
    public void registerGame(String appId, String title, BigDecimal price) {
        GameRecord g = factory.createDefault(appId, title, price);
        repo.save(g);
    }
}

// 4. Notificació (separate)
public class GameRegistrationNotifier {
    public void notifyAdmins(GameRecord g) {
        // Envía email, Slack, etc.
    }
}
```

**Raó per canviar:**
- `GameRecordFactory`: si la lógica de parsing Steam cambia
- `GameRepository`: si canviem BD (in-memory → SQL)
- `GameManagementService`: si la regla de negoci canvia
- `GameRegistrationNotifier`: si canviem el canal de notificació

**Cada classe canvia per una única raó.**

### O - Open/Closed Principle (OCP)

**Definició:** Una classe hauria d'estar **oberta per extensió, tancada per modificació**.

**Mal (violar OCP):**

```java
public class GameRepository {
    public void save(GameRecord g) {
        if (this.type.equals("memory")) {
            this.list.add(g);
        } else if (this.type.equals("sql")) {
            this.db.insert(g);
        } else if (this.type.equals("mongodb")) {
            this.mongo.insert(g);
        }
    }
}
// Cada vegada que afegim un backend, hem de modificar GameRepository!
```

**Bien (complir OCP):**

```java
// Interfície (abstracció)
public interface GameRepository {
    void save(GameRecord g);
}

// Implementacions concretes (closed per modifications)
public class InMemoryGameRepository implements GameRepository {
    private List<GameRecord> list = new ArrayList<>();
    public void save(GameRecord g) { list.add(g); }
}

public class SqlGameRepository implements GameRepository {
    private Database db;
    public void save(GameRecord g) { db.insert(g); }
}

public class MongoGameRepository implements GameRepository {
    private MongoClient mongo;
    public void save(GameRecord g) { mongo.insert(g); }
}

// GameManagementService no canvia NUNCA
public class GameManagementService {
    private GameRepository repo;  // Accepta qualsevol implementació
    
    public void registerGame(String appId, String title, BigDecimal price) {
        GameRecord g = factory.create(appId, title, price);
        repo.save(g);  // Funciona igual amb In-Memory, SQL, MongoDB
    }
}
```

**Aplicat a GamePulse:**
- S2-4: `InMemoryGameRepository` implementa `GameRepository`
- S5: `SqlGameRepository` implementa `GameRepository` (sense tocar `GameManagementService`)
- Futura: `MongoGameRepository` → **extensió sense modificació**

### L - Liskov Substitution Principle (LSP)

**Definició:** Subclasses hauria de poder substituir-se per la classe pare sense trencar la lógica.

**Mal (violar LSP):**

```java
public class GameRepository {
    public void save(GameRecord g) { /* guarda */ }
    public GameRecord findById(String id) { /* cerca */ }
}

public class ReadOnlyGameRepository extends GameRepository {
    public void save(GameRecord g) {
        throw new UnsupportedOperationException("Read-only!");
    }
    public GameRecord findById(String id) { /* funciona */ }
}

// A GameManagementService:
GameRepository repo = new ReadOnlyGameRepository();
repo.save(game);  // ¡CRASH! Violació de contract.
```

**Bien (complir LSP):**

```java
public interface GameRepository {
    void save(GameRecord g);
    GameRecord findById(String id);
}

// Si necessitem read-only, separa les interfícies
public interface GameReadRepository {
    GameRecord findById(String id);
}

public interface GameWriteRepository {
    void save(GameRecord g);
}

public class ReadOnlyGameRepository implements GameReadRepository {
    // Implementa només findById
}

// GameManagementService sabia que necessita escriure:
public class GameManagementService {
    private GameWriteRepository writeRepo;  // Espera poder guardar
    private GameReadRepository readRepo;    // Espera poder cercar
}
```

**A GamePulse:** `GameRepository` sempre pot `save()` i `findById()`. No fem subclasses que trenquin el contract.

### I - Interface Segregation Principle (ISP)

**Definició:** Els clients no hauria de dependre d'interfícies que no usen.

**Mal (violar ISP):**

```java
public interface GameService {
    GameRecord findById(String id);
    void save(GameRecord g);
    void delete(String id);
    List<GameRecord> getAllGames();
    void updatePrice(String id, BigDecimal price);
    void applyDiscount(String id, double percentage);
    void ban(String id);
    void unban(String id);
    List<GameRecord> getRecommendations(String userId);
    // ... 20 més mètodes
}

// Quan vull fer un simple "get game", he de dependre de TODO
public class SimpleGameViewer {
    private GameService service;  // Depèn de ban(), unban(), applyDiscount(), etc.
    
    public void viewGame(String id) {
        GameRecord g = service.findById(id);  // Necessito solo esto
    }
}
```

**Bien (complir ISP):**

```java
// Interfícies petites, específiques
public interface GameReadService {
    GameRecord findById(String id);
    List<GameRecord> getAllGames();
}

public interface GameWriteService {
    void save(GameRecord g);
    void delete(String id);
}

public interface GameManagementService {
    void updatePrice(String id, BigDecimal price);
    void applyDiscount(String id, double percentage);
}

public interface GameModerationService {
    void ban(String id);
    void unban(String id);
}

public interface GameRecommendationService {
    List<GameRecord> getRecommendations(String userId);
}

// Cada classe depèn SOLO de les interfícies que necessita
public class SimpleGameViewer {
    private GameReadService readService;  // Minim necessari
    
    public void viewGame(String id) {
        GameRecord g = readService.findById(id);
    }
}
```

**A GamePulse (S2):** `GameRepository` té només `save()`, `findById()`, `delete()`. No afegim mètodes de moderació o recomanacions aquí.

### D - Dependency Inversion Principle (DIP)

**Definició:** Depèn d'abstraccions, no de concrecions.

**Mal (violar DIP):**

```java
public class GameManagementService {
    private InMemoryGameRepository repo = new InMemoryGameRepository();  // ❌ Concrete
    private GameRecordFactory factory = new GameRecordFactory();
    
    public void registerGame(String appId, String title) {
        GameRecord g = factory.create(appId, title);
        repo.save(g);  // Acopla a InMemoryGameRepository
    }
}
// Si vull canviar a BD, he de modificar GameManagementService
```

**Bien (complir DIP):**

```java
public class GameManagementService {
    private GameRepository repo;  // ✅ Abstracció (interfície)
    private GameRecordFactory factory;
    
    // Constructor injection
    public GameManagementService(GameRepository repo, GameRecordFactory factory) {
        this.repo = repo;
        this.factory = factory;
    }
    
    public void registerGame(String appId, String title) {
        GameRecord g = factory.create(appId, title);
        repo.save(g);  // Funciona amb qualsevol GameRepository
    }
}

// Ús:
GameRepository inMemoryRepo = new InMemoryGameRepository();
GameManagementService service = new GameManagementService(inMemoryRepo, factory);

// Més tard (S5): canvia a SQL sense tocar GameManagementService
GameRepository sqlRepo = new SqlGameRepository(dataSource);
GameManagementService service = new GameManagementService(sqlRepo, factory);
```

**A GamePulse:** `GameManagementService` rep `GameRepository` per constructor. No crea ni tria implementació.

---

## 3. Immutabilitat: Java Records (S2)

### Per Què Immutabilitat?

```java
// ❌ MUTABLE - problemes
public class GameRecord {
    private String appId;
    private String title;
    private BigDecimal price;
    
    // Setters permeten mudança
    public void setPrice(BigDecimal price) { this.price = price; }
    public void setTitle(String title) { this.title = title; }
}

// Ús:
GameRecord g = new GameRecord("APP-1", "LoL", BigDecimal.valueOf(0));
g.setPrice(BigDecimal.valueOf(100));  // Ha canviat!

// Problema: Si passem 'g' a un altre thread, pot ser que canviï mentre l'estem usant
// Race condition!
```

**Immutabilitat protegeix contra concurrència i errors:**

```java
// ✅ IMMUTABLE - segur
public record GameRecord(
    String appId,
    String title,
    BigDecimal price,
    Long activePlayerCount
) {}

// Ús:
GameRecord g = new GameRecord("APP-1", "LoL", BigDecimal.valueOf(0), 5_000_000L);
// g.setPrice(...)  ← ¡ERROR! Records no tienen setters

// Per fer un "canvi":
GameRecord updatedG = new GameRecord(
    g.appId(),
    g.title(),
    BigDecimal.valueOf(100),  // novo precio
    g.activePlayerCount()
);
// Original 'g' no canvia mai
```

### Avantatges de Records

1. **Thread-safe:** No hi ha race conditions
2. **Predictible:** Els datos no canvien
3. **Cache-friendly:** El compilador pot optimizar

### Records a GamePulse

```java
// Immutable DTO
public record GameDTO(
    String appId,
    String title,
    BigDecimal price,
    Long activePlayerCount
) {}

// Immutable Entity (amb JPA, S5)
@Entity
public record GameEntity(
    @Id String appId,
    String title,
    BigDecimal price,
    Long activePlayerCount
) {}

// Métode que NO muta
public GameRecord discountedPrice(double percentage) {
    BigDecimal discounted = this.price.multiply(
        BigDecimal.valueOf(1 - percentage)
    );
    return new GameRecord(appId, title, discounted, activePlayerCount);
    // Retorna nou record, l'original no canvia
}
```

---

## 4. Compact Constructors (Records)

Records permeten compact constructors per validació:

```java
public record GameRecord(
    String appId,
    String title,
    BigDecimal price,
    Long activePlayerCount
) {
    // Compact constructor - auto-assigna fields
    public GameRecord {
        if (appId == null || appId.isBlank()) {
            throw new IllegalArgumentException("appId cannot be blank");
        }
        if (price.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("price must be >= 0");
        }
        if (activePlayerCount < 0) {
            throw new IllegalArgumentException("activePlayerCount must be >= 0");
        }
    }
}
```

**Avantatge:** Validació centralitzada a la creació. Nunca es pot crear un `GameRecord` invàlid.

---

## 5. Patterns at GamePulse

### Factory Pattern (S2)

```java
public class GameRecordFactory {
    public GameRecord createDefault(String appId, String title) {
        return new GameRecord(appId, title, BigDecimal.ZERO, 0L);
    }
    
    public GameRecord createFromSteamAPI(String json) {
        // Parse JSON, valida, crea
        String appId = extractAppId(json);
        String title = extractTitle(json);
        BigDecimal price = extractPrice(json);
        Long players = extractPlayers(json);
        
        return new GameRecord(appId, title, price, players);
    }
}
```

**Responsabilitat:** Crear `GameRecords` vàlides. Encapsula lógica de parsing.

### Repository Pattern (S2, S5)

**S2 (In-Memory):**
```java
public class InMemoryGameRepository implements GameRepository {
    private Map<String, GameRecord> storage = new ConcurrentHashMap<>();
    
    public void save(GameRecord g) {
        storage.put(g.appId(), g);
    }
    
    public Optional<GameRecord> findById(String appId) {
        return Optional.ofNullable(storage.get(appId));
    }
}
```

**S5 (SQL):**
```java
public interface GameJpaRepository extends JpaRepository<GameEntity, String> {}
```

**Aplicació:** `GameManagementService` no sap on guardamos. S'ajusta automàticament.

---

## 6. Immutability + SOLID en Acció

```java
// Immutable record
public record GameRecord(String appId, String title, BigDecimal price, Long players) {
    public GameRecord {
        if (appId == null) throw new IllegalArgumentException("appId required");
    }
}

// Single Responsibility: crear
public class GameRecordFactory {
    public GameRecord create(String appId, String title, BigDecimal price) {
        return new GameRecord(appId, title, price, 0L);
    }
}

// Single Responsibility: guardar
public interface GameRepository {
    void save(GameRecord g);
    Optional<GameRecord> findById(String id);
}

// Single Responsibility: lógica
public class GameManagementService {
    private final GameRepository repo;
    private final GameRecordFactory factory;
    
    public GameManagementService(GameRepository repo, GameRecordFactory factory) {
        this.repo = repo;
        this.factory = factory;
    }
    
    public void registerGame(String appId, String title, BigDecimal price) {
        GameRecord g = factory.create(appId, title, price);
        repo.save(g);  // Immutable, thread-safe, persisten
    }
}
```

**Resultats:**
- ✅ Fàcil de testejar (mock `GameRepository`)
- ✅ Thread-safe (records immutables)
- ✅ Extensible (implementa altres `GameRepository`)
- ✅ Mantenible (cada classe una raó)

---

## 7. Lectura Profunda

- **SOLID Principles:** [Baeldung SOLID](https://www.baeldung.com/solid-principles)
- **Java Records:** [Baeldung Records](https://www.baeldung.com/java-record-keyword)
- **Design Patterns:** [Refactoring.Guru](https://refactoring.guru/design-patterns/java)

---

## Resum

| Concepte | Problemes que soluciona | Exemple GamePulse |
|----------|-------------------------|-------------------|
| **SRP** | Classes amb moltes responsabilitats | `GameRecordFactory` (crear), `GameRepository` (guardar), `GameManagementService` (lógica) |
| **OCP** | Modificacions constants | Interfície `GameRepository` + múltiples implementacions (In-Memory, SQL, MongoDB) |
| **LSP** | Comportament impredictible de subclasses | Totes les `GameRepository` impl. cumpleixen el contract |
| **ISP** | Dependències innecessàries | `GameRepository` és mínima (save, findById) |
| **DIP** | Acoplament a concrecions | `GameManagementService` depèn de `GameRepository` (interfície) |
| **Immutability** | Race conditions en multithreading | Records de Java 21: `GameRecord` no es pot modificar |

**Objectiu setmana:** Implementar SOLID amb records immutables; veure que els tests vells passen quan canviem persistència.
