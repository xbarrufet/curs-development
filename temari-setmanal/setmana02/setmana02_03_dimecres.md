# Setmana 2 — Dimecres: Dependency Inversion en Practica i .cursorrules com a Spec

## Objectiu del Dia

Construir la capa de servei (`GameManagementService`) que aplica Dependency Inversion de forma real: rep el repositori per constructor, no crea res directament. Tambe escriure un `.cursorrules` que funcioni com a especificacio de comportament per a l'assistent IA, verificant que genera codi coherent amb el projecte. Al final del dia tens el servei funcionant amb el repositori d'ahir i un `.cursorrules` que produeix resultats consistents.

---

## Teoria

### Dependency Inversion en 3 Capes

Fins ara tens dues peces:
- `GameRecord` — el model immutable (dilluns)
- `InMemoryGameRepository` — la persistencia en memoria (dimarts)

Avui afegim la tercera: el servei de negoci. L'arquitectura queda aixi:

```
┌──────────────────────────────────┐
│      GameManagementService       │  ← Logica de negoci: QUE fer
│  (orquestra, no crea ni guarda)  │
└────────────┬─────────────────────┘
             │ depèn de (interficie)
┌────────────▼─────────────────────┐
│        GameRepository            │  ← Contracte: quins metodes existeixen
│     (interficie abstracta)       │
└────────────┬─────────────────────┘
             │ implementa
┌────────────▼─────────────────────┐
│   InMemoryGameRepository         │  ← COM es fa: detall d'implementacio
│   (ConcurrentHashMap)            │
└──────────────────────────────────┘
```

**La regla clau:** les fletxes de dependencia van cap AMUNT. El servei depèn de la interficie, no de la implementacio concreta. La implementacio concreta tambe depèn de la interficie (la implementa). Ningu depèn de ningu cap avall.

### Factory Pattern: Centralitzar la Creacio

El servei no hauria de saber com es crea un `GameRecord` — nomes que en vol un. Per aixo usem un Factory:

```java
// Factory: responsable de CREAR GameRecords amb logica centralitzada
// Si el format de creacio canvia, nomes toques aquesta classe
public class GameRecordFactory {

    // Crea un GameRecord amb valors per defecte (preu 0, jugadors 0)
    // Util per quan registres un joc nou que encara no te dades de mercat
    public GameRecord createDefault(String appId, String title, BigDecimal price) {
        return new GameRecord(appId, title, price, 0L);
    }

    // Crea un GameRecord a partir d'un JSON de la Steam API
    // Encapsula tota la logica de parsing — el servei no sap res de JSON
    public GameRecord createFromSteamAPI(String appId, String json) {
        // Parseja el JSON per extreure els camps necessaris
        String title = extractField(json, "name");
        BigDecimal price = new BigDecimal(extractField(json, "price"));
        Long players = Long.parseLong(extractField(json, "players"));

        // Crea el record — la validacio del compact constructor s'aplica automaticament
        return new GameRecord(appId, title, price, players);
    }

    // Metode auxiliar per extreure un camp del JSON
    // (simplificat — en un projecte real usaries Jackson o Gson)
    private String extractField(String json, String field) {
        // Implementacio simplificada per a la demo
        // A la Setmana 5 usarem una llibreria de parsing real
        return "";  // Placeholder
    }
}
```

**Per que un Factory i no crear directament al servei?**
- SRP: el servei orquestra, el factory crea
- Si el format de l'API canvia, nomes toques el factory
- Testejar el factory es independent de testejar el servei

### Service Layer: L'Orquestrador

```java
// Servei de negoci: orquestra la logica sense saber detalls d'implementacio
// Rep les dependencies per CONSTRUCTOR — mai les crea ell (DIP)
public class GameManagementService {

    // Dependencies declarades com a interficies (abstraccions)
    // El servei no sap si el repo es in-memory, SQL, o MongoDB
    private final GameRepository repo;
    private final GameRecordFactory factory;

    // Constructor injection: qui crea el servei decideix QUINES implementacions usar
    // Aixo es DIP pur: el servei depèn d'abstraccions, no de concrecions
    public GameManagementService(GameRepository repo, GameRecordFactory factory) {
        this.repo = repo;
        this.factory = factory;
    }

    // Registra un joc nou: delega creacio al factory, persistencia al repo
    public void registerGame(String appId, String title, BigDecimal price) {
        // El factory crea el record (amb validacio del compact constructor)
        GameRecord game = factory.createDefault(appId, title, price);
        // El repo el guarda (no sabem on — in-memory? SQL? No importa)
        repo.save(game);
    }

    // Retorna nomes els jocs populars (mes de 100K jugadors)
    // Filtra usant el metode isPopular() del propi GameRecord
    public List<GameRecord> getPopularGames() {
        return repo.findAll()        // Obte tots els jocs del repo
            .stream()                // Converteix a stream per filtrar
            .filter(GameRecord::isPopular)  // Filtra els que son populars
            .toList();               // Converteix el resultat a llista
    }

    // Busca un joc per ID — delega al repo i retorna Optional
    public Optional<GameRecord> findGame(String appId) {
        return repo.findById(appId);
    }
}
```

### Per Que Constructor Injection Importa

```java
// MAL — el servei CREA les seves dependencies
public class GameManagementService {
    // Acoblat a InMemoryGameRepository — si vull SQL, he de modificar AQUESTA classe
    private GameRepository repo = new InMemoryGameRepository();

    // Impossible de testejar amb un mock — sempre usa InMemory
}

// BE — el servei REP les seves dependencies
public class GameManagementService {
    private final GameRepository repo;

    // Qui crea el servei decideix la implementacio
    public GameManagementService(GameRepository repo, GameRecordFactory factory) {
        this.repo = repo;
        this.factory = factory;
    }
}

// Produccio: usa SQL
GameRepository sqlRepo = new SqlGameRepository(dataSource);
GameManagementService prodService = new GameManagementService(sqlRepo, factory);

// Test: usa un mock o in-memory — SENSE tocar el servei
GameRepository testRepo = new InMemoryGameRepository();
GameManagementService testService = new GameManagementService(testRepo, factory);
```

**El benefici real:** als tests de demà podras crear un `InMemoryGameRepository`, injectar-lo al servei, i testejar la logica de negoci sense cap base de dades. Si el servei creés les seves dependencies, no podries fer-ho.

### `.cursorrules` com a Especificacio de Comportament

Un `.cursorrules` no es un fitxer de configuracio generic — es la primera especificacio que escrius per a un agent IA. Si les regles son vagues, l'agent genera codi inconsistent. Si son precises, el codi surt coherent amb el projecte.

**Diferencia entre vague i precis:**

```
# VAGUE — l'agent interpretara com vulgui
"Usa bons noms de variable"
"Segueix bones practiques"
"Escriu codi net"

# PRECIS — l'agent sap exactament que fer
"Variables Java en camelCase: gameRecord, activePlayerCount"
"Variables Python en snake_case: game_record, active_player_count"
"Models de domini: record (Java), @dataclass(frozen=True) (Python). Mai setters."
"Cada classe publica necessita un test JUnit corresponent"
"Commits: Conventional Commits (feat/fix/test/docs/refactor)"
```

**La llico:** Escriure specs per a una IA es escriure specs per a un dev junior molt rapid pero amb zero context. Si no li dius com vols les coses, les fara a la seva manera — i no sera la teva.

---

## Activitat

### 1. Crear `GameRecordFactory` (20 min)

Crea el fitxer:
```
backend-java/src/main/java/com/esportspulse/engine/factory/GameRecordFactory.java
```

Implementa:
- `createDefault(String appId, String title, BigDecimal price)` — crea un GameRecord amb 0 jugadors
- `createFromSteamAPI(String appId, String json)` — parseja JSON simplificat i crea un GameRecord

### 2. Crear `GameManagementService` (30 min)

Crea el fitxer:
```
backend-java/src/main/java/com/esportspulse/engine/service/GameManagementService.java
```

Implementa:
- Constructor que rep `GameRepository` i `GameRecordFactory` (DIP)
- `registerGame(String appId, String title, BigDecimal price)`
- `getPopularGames()` — filtra amb stream + `isPopular()`
- `findGame(String appId)` — delega al repo

### 3. Integrar les 3 capes (20 min)

```java
public class ServiceDemo {
    public static void main(String[] args) {
        // COMPOSICIO: aqui decidim quines implementacions usar
        // En una aplicacio real, aixo ho faria Spring Boot automaticament
        GameRepository repo = new InMemoryGameRepository();
        GameRecordFactory factory = new GameRecordFactory();
        GameManagementService service = new GameManagementService(repo, factory);

        // Registrem jocs usant el SERVEI — no el repo directament
        service.registerGame("APP-1", "League of Legends", BigDecimal.ZERO);
        service.registerGame("APP-2", "Dota 2", BigDecimal.ZERO);
        service.registerGame("APP-3", "Indie Game", BigDecimal.valueOf(19.99));

        // Nota: registerGame crea jocs amb 0 jugadors (createDefault)
        // Aixi que getPopularGames() retornara llista buida
        List<GameRecord> popular = service.getPopularGames();
        System.out.println("Populars: " + popular.size());  // 0 — tots tenen 0 jugadors

        // Busquem un joc per ID
        service.findGame("APP-1").ifPresent(
            g -> System.out.println("Trobat: " + g.title())  // "Trobat: League of Legends"
        );

        // Busquem un joc que no existeix
        boolean exists = service.findGame("APP-999").isPresent();
        System.out.println("APP-999 existeix? " + exists);  // false
    }
}
```

### 4. Escriure `.cursorrules` com a Spec (30 min)

Crea o reescriu el fitxer `.cursorrules` a l'arrel del projecte. Ha de ser una especificacio precisa:

```
# EsportsPulse Engine — Especificacio per a l'Assistent

## Llenguatge i Convencions
- Java 21: variables en camelCase (gameRecord, activePlayerCount)
- Python 3.12: variables en snake_case (game_record, active_player_count)
- Classes en PascalCase en ambdos llenguatges

## Models de Domini
- Java: SEMPRE usar `record`. Mai generar classes amb setters.
- Python: SEMPRE usar `@dataclass(frozen=True)`. Mai atributs mutables.
- Cada record/dataclass ha de tenir compact constructor/`__post_init__` amb validacio.

## Arquitectura
- Patrons: Repository (persistencia), Factory (creacio), Service (logica)
- Dependencies: injectar per constructor. Mai crear dependencies amb `new` dins un servei.
- Interficies: capa de dades sempre darrera d'una interficie.
- Packages Java: model/, repository/, factory/, service/
- Moduls Python: model/, repository/, factory/, service/

## Testing
- Cada classe publica ha de tenir un test JUnit 5 / pytest corresponent.
- Noms de test: `metode_comportament_condicio` (ex: `findById_returnsEmpty_whenNotFound`)
- Assercions: `assertEquals`, `assertNotNull`, `assertThrows` (Java); `assert`, `pytest.raises` (Python)

## Git
- Format: Conventional Commits (feat/fix/test/docs/refactor)
- Exemple: `feat(java): add GameManagementService with DI`
- Branques: `feature/weekN-description`
```

**Verificacio:** Demana a Cursor: "Genera un `PlayerRecord` seguint les convencions del projecte". L'assistent ha de generar un `record` (no una classe amb setters), amb compact constructor i validacio. Si no ho fa, ajusta les regles.

### 5. Commit (5 min)

```bash
git add backend-java/src/main/java/com/esportspulse/engine/factory/
git add backend-java/src/main/java/com/esportspulse/engine/service/
git add .cursorrules
git commit -m "feat(java): GameManagementService with DI + GameRecordFactory + .cursorrules spec"
```

---

## Checklist de Lliurament

- [ ] `GameRecordFactory` amb `createDefault` i `createFromSteamAPI`
- [ ] `GameManagementService` rep `GameRepository` i `GameRecordFactory` per constructor (DIP)
- [ ] `registerGame()` usa el factory per crear i el repo per guardar
- [ ] `getPopularGames()` filtra correctament amb streams
- [ ] Demo de les 3 capes integrades funciona sense errors
- [ ] `.cursorrules` escrit com a especificacio precisa (no generica)
- [ ] Cursor genera `PlayerRecord` com a `record` (no POJO) quan li demanes
- [ ] Commit amb format Conventional Commits
