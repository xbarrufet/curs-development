# Setmana 2 - Teoria: POO, SOLID i Immutabilitat

## 1. Introducció: Per Què SOLID?

Imagina el codi de EsportsPulse sense SOLID:

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

**Aplicat a EsportsPulse:**

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
public interface ChampionRepository {
    void save(GameRecord g);
    GameRecord findById(String appId);
}

// 3. Lógica de negoci
public class ChampionManagementService {
    private ChampionRepository repo;
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
- `ChampionRepository`: si canviem BD (in-memory → SQL)
- `ChampionManagementService`: si la regla de negoci canvia
- `GameRegistrationNotifier`: si canviem el canal de notificació

**Cada classe canvia per una única raó.**

### O - Open/Closed Principle (OCP)

**Definició:** Una classe hauria d'estar **oberta per extensió, tancada per modificació**.

**Mal (violar OCP):**

```java
public class ChampionRepository {
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
// Cada vegada que afegim un backend, hem de modificar ChampionRepository!
```

**Bien (complir OCP):**

```java
// Interfície (abstracció)
public interface ChampionRepository {
    void save(GameRecord g);
}

// Implementacions concretes (closed per modifications)
public class InMemoryChampionRepository implements ChampionRepository {
    private List<GameRecord> list = new ArrayList<>();
    public void save(GameRecord g) { list.add(g); }
}

public class SqlChampionRepository implements ChampionRepository {
    private Database db;
    public void save(GameRecord g) { db.insert(g); }
}

public class MongoChampionRepository implements ChampionRepository {
    private MongoClient mongo;
    public void save(GameRecord g) { mongo.insert(g); }
}

// ChampionManagementService no canvia NUNCA
public class ChampionManagementService {
    private ChampionRepository repo;  // Accepta qualsevol implementació
    
    public void registerGame(String appId, String title, BigDecimal price) {
        GameRecord g = factory.create(appId, title, price);
        repo.save(g);  // Funciona igual amb In-Memory, SQL, MongoDB
    }
}
```

**Aplicat a EsportsPulse:**
- S2-4: `InMemoryChampionRepository` implementa `ChampionRepository`
- S5: `SqlChampionRepository` implementa `ChampionRepository` (sense tocar `ChampionManagementService`)
- Futura: `MongoChampionRepository` → **extensió sense modificació**

### L - Liskov Substitution Principle (LSP)

**Definició:** Subclasses hauria de poder substituir-se per la classe pare sense trencar la lógica.

**Mal (violar LSP):**

```java
public class ChampionRepository {
    public void save(GameRecord g) { /* guarda */ }
    public GameRecord findById(String id) { /* cerca */ }
}

public class ReadOnlyChampionRepository extends ChampionRepository {
    public void save(GameRecord g) {
        throw new UnsupportedOperationException("Read-only!");
    }
    public GameRecord findById(String id) { /* funciona */ }
}

// A ChampionManagementService:
ChampionRepository repo = new ReadOnlyChampionRepository();
repo.save(game);  // ¡CRASH! Violació de contract.
```

**Bien (complir LSP):**

```java
public interface ChampionRepository {
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

public class ReadOnlyChampionRepository implements GameReadRepository {
    // Implementa només findById
}

// ChampionManagementService sabia que necessita escriure:
public class ChampionManagementService {
    private GameWriteRepository writeRepo;  // Espera poder guardar
    private GameReadRepository readRepo;    // Espera poder cercar
}
```

**A EsportsPulse:** `ChampionRepository` sempre pot `save()` i `findById()`. No fem subclasses que trenquin el contract.

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

public interface ChampionManagementService {
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

**A EsportsPulse (S2):** `ChampionRepository` té només `save()`, `findById()`, `delete()`. No afegim mètodes de moderació o recomanacions aquí.

### D - Dependency Inversion Principle (DIP)

**Definició:** Depèn d'abstraccions, no de concrecions.

**Mal (violar DIP):**

```java
public class ChampionManagementService {
    private InMemoryChampionRepository repo = new InMemoryChampionRepository();  // ❌ Concrete
    private GameRecordFactory factory = new GameRecordFactory();
    
    public void registerGame(String appId, String title) {
        GameRecord g = factory.create(appId, title);
        repo.save(g);  // Acopla a InMemoryChampionRepository
    }
}
// Si vull canviar a BD, he de modificar ChampionManagementService
```

**Bien (complir DIP):**

```java
public class ChampionManagementService {
    private ChampionRepository repo;  // ✅ Abstracció (interfície)
    private GameRecordFactory factory;
    
    // Constructor injection
    public ChampionManagementService(ChampionRepository repo, GameRecordFactory factory) {
        this.repo = repo;
        this.factory = factory;
    }
    
    public void registerGame(String appId, String title) {
        GameRecord g = factory.create(appId, title);
        repo.save(g);  // Funciona amb qualsevol ChampionRepository
    }
}

// Ús:
ChampionRepository inMemoryRepo = new InMemoryChampionRepository();
ChampionManagementService service = new ChampionManagementService(inMemoryRepo, factory);

// Més tard (S5): canvia a SQL sense tocar ChampionManagementService
ChampionRepository sqlRepo = new SqlChampionRepository(dataSource);
ChampionManagementService service = new ChampionManagementService(sqlRepo, factory);
```

**A EsportsPulse:** `ChampionManagementService` rep `ChampionRepository` per constructor. No crea ni tria implementació.

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

### Records a EsportsPulse

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

## 5. Patterns at EsportsPulse

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
public class InMemoryChampionRepository implements ChampionRepository {
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

**Aplicació:** `ChampionManagementService` no sap on guardamos. S'ajusta automàticament.

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
public interface ChampionRepository {
    void save(GameRecord g);
    Optional<GameRecord> findById(String id);
}

// Single Responsibility: lógica
public class ChampionManagementService {
    private final ChampionRepository repo;
    private final GameRecordFactory factory;
    
    public ChampionManagementService(ChampionRepository repo, GameRecordFactory factory) {
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
- ✅ Fàcil de testejar (mock `ChampionRepository`)
- ✅ Thread-safe (records immutables)
- ✅ Extensible (implementa altres `ChampionRepository`)
- ✅ Mantenible (cada classe una raó)

---

## 7. Lectura Profunda

- **SOLID Principles:** [Baeldung SOLID](https://www.baeldung.com/solid-principles)
- **Java Records:** [Baeldung Records](https://www.baeldung.com/java-record-keyword)
- **Design Patterns:** [Refactoring.Guru](https://refactoring.guru/design-patterns/java)

---

## Resum

| Concepte | Problemes que soluciona | Exemple EsportsPulse |
|----------|-------------------------|-------------------|
| **SRP** | Classes amb moltes responsabilitats | `GameRecordFactory` (crear), `ChampionRepository` (guardar), `ChampionManagementService` (lógica) |
| **OCP** | Modificacions constants | Interfície `ChampionRepository` + múltiples implementacions (In-Memory, SQL, MongoDB) |
| **LSP** | Comportament impredictible de subclasses | Totes les `ChampionRepository` impl. cumpleixen el contract |
| **ISP** | Dependències innecessàries | `ChampionRepository` és mínima (save, findById) |
| **DIP** | Acoplament a concrecions | `ChampionManagementService` depèn de `ChampionRepository` (interfície) |
| **Immutability** | Race conditions en multithreading | Records de Java 21: `GameRecord` no es pot modificar |

**Objectiu setmana:** Implementar SOLID amb records immutables; veure que els tests vells passen quan canviem persistència.
