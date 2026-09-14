# Setmana 2 — Dilluns: Principis SOLID i Immutabilitat amb Records

## Objectiu del Dia

Entendre els 5 principis SOLID i per que son fonamentals per escriure codi mantenible. Crear el primer model immutable amb Java 21 Records i veure com un `record` elimina bugs que un POJO mutable permet. Al final del dia tens `GameRecord` creat, validat amb compact constructor, i 3 instancies provades a `main`.

---

## Teoria

### Per Que SOLID? El Problema Real

Imagina que ets al teu primer dia de feina i trobes aquest codi:

```java
// Classe que fa MASSA coses — et trobaras aixo a projectes reals
public class GameService {
    private List<GameRecord> games;       // Emmagatzema jocs
    private Database db;                  // Connexio a base de dades
    private EmailService emailService;    // Envia correus
    private APIClient steamAPI;           // Crida a API externa
    private AnalyticsTracker tracker;     // Trackeja events

    public void registerGame(String appId, String title) {
        // 1. Valida (responsabilitat de validacio)
        if (appId == null) throw new Exception("Invalid");

        // 2. Crea l'objecte (responsabilitat de creacio)
        GameRecord g = new GameRecord(appId, title, 0, 0);

        // 3. Guarda a multiples llocs (responsabilitat de persistencia)
        games.add(g);
        db.save(g);

        // 4. Envia notificacio (responsabilitat de notificacio)
        emailService.notifyAdmins("New game: " + title);

        // 5. Trackeja (responsabilitat d'analytics)
        tracker.log("game_registered", g.appId());
    }
}
```

**Problemes concrets d'aquest codi:**
- Fa 5 coses: validacio, creacio, persistencia, notificacio, tracking
- Si vols canviar com envies emails, toques una classe que tambe guarda a la BD
- Per testejar `registerGame` necessites mock de 5 serveis — un test que hauria de ser senzill es converteix en 40 linies de setup
- Reutilitzar la logica de creacio a un altre lloc? Impossible, esta acoblada

SOLID son 5 principis que resolen exactament aquests problemes. No son teoria abstracta — son la diferencia entre codi que pots mantenir i codi que ningú vol tocar.

### S — Single Responsibility Principle (SRP)

**Definicio:** Una classe ha de tenir una unica rao per canviar.

Aplicat al problema anterior, la solucio es separar responsabilitats:

```java
// RESPONSABILITAT 1: Crear GameRecords
// Aquesta classe nomes canvia si canvia la logica de creacio
public class GameRecordFactory {
    public GameRecord createFromSteamAPI(String appId, String json) {
        // Parseja el JSON i crea un record valid
        return new GameRecord(appId, title, price, players);
    }
}

// RESPONSABILITAT 2: Guardar i recuperar dades
// Aquesta classe nomes canvia si canvia la font de dades
public interface GameRepository {
    void save(GameRecord g);
    GameRecord findById(String appId);
}

// RESPONSABILITAT 3: Orquestrar la logica de negoci
// Aquesta classe nomes canvia si canvien les regles de negoci
public class GameManagementService {
    private GameRepository repo;          // Delega persistencia
    private GameRecordFactory factory;    // Delega creacio

    public void registerGame(String appId, String title, BigDecimal price) {
        // Nomes fa una cosa: coordinar el flux de negoci
        GameRecord g = factory.createDefault(appId, title, price);
        repo.save(g);
    }
}

// RESPONSABILITAT 4: Notificacions (completament separada)
public class GameRegistrationNotifier {
    public void notifyAdmins(GameRecord g) {
        // Envia email, Slack, o el que sigui
    }
}
```

**Per que funciona:** Cada classe canvia per una unica rao. Si canvia el format de l'API de Steam, nomes toques `GameRecordFactory`. Si canvies de base de dades, nomes toques la implementacio del `GameRepository`. El servei de negoci no sap ni li importa.

### O — Open/Closed Principle (OCP)

**Definicio:** Obert per extensio, tancat per modificacio.

```java
// MAL — cada vegada que afegim un backend, modifiquem aquesta classe
public class GameRepository {
    public void save(GameRecord g) {
        if (this.type.equals("memory")) {
            this.list.add(g);              // Backend 1
        } else if (this.type.equals("sql")) {
            this.db.insert(g);             // Backend 2
        } else if (this.type.equals("mongodb")) {
            this.mongo.insert(g);          // Backend 3... i creixent
        }
    }
}

// BE — definim una interficie i cada backend es una implementacio nova
// Interficie: el contracte que tothom ha de complir
public interface GameRepository {
    void save(GameRecord g);
}

// Implementacio 1: memoria (Setmana 2)
public class InMemoryGameRepository implements GameRepository {
    private List<GameRecord> list = new ArrayList<>();

    public void save(GameRecord g) {
        list.add(g);  // Guarda en memoria — per desenvolupament i tests
    }
}

// Implementacio 2: SQL (Setmana 5 — no cal tocar InMemory!)
public class SqlGameRepository implements GameRepository {
    private Database db;

    public void save(GameRecord g) {
        db.insert(g);  // Guarda a base de dades
    }
}
```

**Per que funciona:** Per afegir MongoDB, crees una nova classe. Mai toques les existents. `GameManagementService` no canvia mai perque depèn de la interficie, no de la implementacio.

### L — Liskov Substitution Principle (LSP)

**Definicio:** Qualsevol implementacio d'una interficie ha de poder substituir una altra sense trencar el programa.

```java
// MAL — ReadOnlyGameRepository trenca el contracte de save()
public class ReadOnlyGameRepository extends GameRepository {
    public void save(GameRecord g) {
        // Llança excepcio! El codi que crida save() petara
        throw new UnsupportedOperationException("Read-only!");
    }
}

// BE — si necessites read-only, separa les interficies
public interface GameReadRepository {
    Optional<GameRecord> findById(String id);  // Nomes lectura
}

public interface GameWriteRepository {
    void save(GameRecord g);  // Nomes escriptura
}
// Cada servei demana nomes el que necessita
```

### I — Interface Segregation Principle (ISP)

**Definicio:** Els clients no han de dependre d'interficies que no usen.

```java
// MAL — una interficie gegant amb 20 metodes
public interface GameService {
    GameRecord findById(String id);
    void save(GameRecord g);
    void delete(String id);
    void ban(String id);
    void unban(String id);
    List<GameRecord> getRecommendations(String userId);
    // ... 15 metodes mes
}

// Si nomes vull llegir un joc, depenc de ban(), unban(), etc.

// BE — interficies petites i especifiques
public interface GameReadService {
    GameRecord findById(String id);       // Nomes lectura
    List<GameRecord> getAllGames();
}

public interface GameWriteService {
    void save(GameRecord g);              // Nomes escriptura
    void delete(String id);
}

// Cada classe depèn NOMES del minim que necessita
public class SimpleGameViewer {
    private GameReadService readService;  // No sap que existeix ban() o delete()

    public void viewGame(String id) {
        GameRecord g = readService.findById(id);
    }
}
```

### D — Dependency Inversion Principle (DIP)

**Definicio:** Depèn d'abstraccions (interficies), mai de concrecions (implementacions concretes).

```java
// MAL — acoblat a InMemoryGameRepository
public class GameManagementService {
    // Crea directament la implementacio concreta
    private InMemoryGameRepository repo = new InMemoryGameRepository();

    public void registerGame(String appId, String title) {
        repo.save(factory.create(appId, title));
        // Si vull canviar a SQL, he de MODIFICAR aquesta classe
    }
}

// BE — depèn de la interficie, rep la implementacio per constructor
public class GameManagementService {
    private GameRepository repo;  // Interficie — no sap quina implementacio es

    // Constructor injection: qui crea el servei decideix quina implementacio usa
    public GameManagementService(GameRepository repo) {
        this.repo = repo;
    }

    public void registerGame(String appId, String title) {
        repo.save(factory.create(appId, title));
        // Funciona igual amb InMemory, SQL, MongoDB... sense canviar res
    }
}

// Us: qui monta l'aplicacio decideix la implementacio
GameRepository repo = new InMemoryGameRepository();  // Setmana 2
GameManagementService service = new GameManagementService(repo);

// Setmana 5: canvies una sola linia
GameRepository repo = new SqlGameRepository(dataSource);  // Canvi sense tocar el servei
GameManagementService service = new GameManagementService(repo);
```

---

### Immutabilitat: Java 21 Records

Un `record` de Java 21 es un objecte que no es pot modificar despres de crear-lo. Aixo resol problemes reals de concurrencia i bugs subtils.

```java
// MAL — objecte mutable: permet canviar l'estat despres de crear-lo
public class GameRecord {
    private String appId;
    private String title;
    private BigDecimal price;

    // Setters permeten mutacio — qualsevol codi pot canviar el preu
    public void setPrice(BigDecimal price) { this.price = price; }
}

GameRecord g = new GameRecord("APP-1", "LoL", BigDecimal.valueOf(0));
g.setPrice(BigDecimal.valueOf(100));  // Ha canviat! Si un altre thread llegia g...

// BE — record immutable: un cop creat, mai canvia
public record GameRecord(
    String appId,            // Identificador unic del joc
    String title,            // Nom del joc
    BigDecimal price,        // Preu en euros
    Long activePlayerCount   // Jugadors actius
) {}

GameRecord g = new GameRecord("APP-1", "LoL", BigDecimal.valueOf(0), 5_000_000L);
// g.setPrice(...)  — ERROR! No existeix cap setter. L'objecte es immutable.
```

**Per que immutabilitat importa:**
- **Thread-safe:** Si dos threads llegeixen `g` al mateix temps, cap dels dos pot canviar-lo. Zero race conditions.
- **Predictible:** Si passes `g` a un metode, saps que quan torni el metode, `g` no ha canviat.
- **Debuggable:** L'objecte que veus al log es exactament l'objecte que existia — ningu l'ha mutat entremig.

### Compact Constructor: Validacio a la Creacio

```java
public record GameRecord(
    String appId,
    String title,
    BigDecimal price,
    Long activePlayerCount
) {
    // Compact constructor — s'executa automaticament quan es crea el record
    // No cal "this.appId = appId" — Java ho fa sol amb records
    public GameRecord {
        // Validacio: si les dades son invalides, llança excepcio
        // Aixi MAI pot existir un GameRecord invalid al sistema
        if (appId == null || appId.isBlank()) {
            throw new IllegalArgumentException("appId no pot ser buit");
        }
        if (price.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("price ha de ser >= 0");
        }
        if (activePlayerCount < 0) {
            throw new IllegalArgumentException("activePlayerCount ha de ser >= 0");
        }
    }
}
```

### Metodes de Negoci al Record

```java
public record GameRecord(
    String appId,
    String title,
    BigDecimal price,
    Long activePlayerCount
) {
    // Compact constructor (validacio)
    public GameRecord {
        if (appId == null || appId.isBlank()) {
            throw new IllegalArgumentException("appId no pot ser buit");
        }
        if (price.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("price ha de ser >= 0");
        }
    }

    // Metode de negoci: un joc es "popular" si te mes de 100.000 jugadors
    public boolean isPopular() {
        return activePlayerCount > 100_000;
    }

    // Metode que "modifica" sense mutar — retorna un NOU record amb el preu descomptat
    // L'original no canvia MAI
    public GameRecord discountedPrice(double percentage) {
        // Calcula el nou preu multiplicant per (1 - percentatge)
        BigDecimal discounted = this.price.multiply(
            BigDecimal.valueOf(1 - percentage)
        );
        // Crea i retorna un record NOU — l'original segueix intacte
        return new GameRecord(appId, title, discounted, activePlayerCount);
    }
}
```

---

## Activitat

### 1. Crear `GameRecord` amb validacio (30 min)

Crea el fitxer:
```
backend-java/src/main/java/com/esportspulse/engine/model/GameRecord.java
```

Implementa el `record` amb:
- Camps: `appId` (String), `title` (String), `price` (BigDecimal), `activePlayerCount` (Long)
- Compact constructor amb validacio (appId no null/buit, price >= 0, activePlayerCount >= 0)
- Metode `isPopular()` que retorna `true` si `activePlayerCount > 100_000`
- Metode `discountedPrice(double percentage)` que retorna un NOU record amb el preu reduit

### 2. Provar a `main` (20 min)

Crea 3 instancies de `GameRecord` a un `main` temporal:

```java
public class GameRecordDemo {
    public static void main(String[] args) {
        // Joc popular (mes de 100K jugadors)
        GameRecord lol = new GameRecord("APP-1", "League of Legends",
            BigDecimal.ZERO, 5_000_000L);

        // Joc no popular (menys de 100K jugadors)
        GameRecord indie = new GameRecord("APP-2", "Indie Game",
            BigDecimal.valueOf(19.99), 500L);

        // Joc amb descompte
        GameRecord lolDescompte = lol.discountedPrice(0.20);

        // Verifica que isPopular funciona
        System.out.println(lol.isPopular());    // true — te 5M jugadors
        System.out.println(indie.isPopular());  // false — te 500 jugadors

        // Verifica que discountedPrice NO muta l'original
        System.out.println(lol.price());           // 0 — no ha canviat
        System.out.println(lolDescompte.price());  // 0 — 20% de 0 es 0

        // Prova amb un joc de pagament
        GameRecord premium = new GameRecord("APP-3", "Premium Game",
            BigDecimal.valueOf(59.99), 200_000L);
        GameRecord premiumDescompte = premium.discountedPrice(0.50);
        System.out.println(premium.price());           // 59.99
        System.out.println(premiumDescompte.price());  // ~29.995

        // Prova que la validacio funciona (ha de petar)
        try {
            new GameRecord(null, "Bad Game", BigDecimal.ZERO, 0L);
        } catch (IllegalArgumentException e) {
            System.out.println("Validacio OK: " + e.getMessage());
        }
    }
}
```

### 3. Escriure `.cursorrules` amb intencio (30 min)

Escriu un `.cursorrules` des de zero. No es un fitxer de configuracio generic — es una **especificacio de comportament** per a l'assistent IA:

Ha d'incloure:
- Convencions de noms: `camelCase` (Java) i `snake_case` (Python)
- Regla d'immutabilitat: "Tots els models de domini han de ser `record` (Java) o `@dataclass(frozen=True)` (Python). Mai generar setters."
- Estructura de carpetes: on va cada tipus de fitxer
- Regles de testing: "Cada classe publica ha de tenir una classe de test corresponent."
- Regles de Git: format Conventional Commits

**Verificacio:** Demana a Cursor que generi un nou model (`PlayerRecord`) seguint el context del projecte. L'assistent respecta les regles? Si no, ajusta fins que ho faci.

### 4. Commit (5 min)

```bash
git add backend-java/src/main/java/com/esportspulse/engine/model/GameRecord.java
git add .cursorrules
git commit -m "feat(java): immutable GameRecord with compact constructor + SOLID .cursorrules"
```

---

## Checklist de Lliurament

- [ ] `GameRecord` creat com a `record` amb 4 camps
- [ ] Compact constructor valida `appId` no null, `price >= 0`, `activePlayerCount >= 0`
- [ ] `isPopular()` retorna `true` si `activePlayerCount > 100_000`
- [ ] `discountedPrice()` retorna un NOU record sense mutar l'original
- [ ] 3 instancies creades i provades a `main` — tot imprimeix el que s'espera
- [ ] `.cursorrules` escrit amb regles d'immutabilitat, noms i testing
- [ ] Commit amb format Conventional Commits
