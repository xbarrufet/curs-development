# Setmana 2 — Dimecres: Dependency Inversion en Practica i .cursorrules com a Spec

## Objectiu del Dia

Construir la capa de servei (`ChampionManagementService`) que aplica Dependency Inversion de forma real: rep el repositori per constructor, no crea res directament. Tambe escriure un `.cursorrules` que funcioni com a especificacio de comportament per a l'assistent IA, verificant que genera codi coherent amb el projecte. Al final del dia tens el servei funcionant amb el repositori d'ahir i un `.cursorrules` que produeix resultats consistents.

---

## Teoria

### Dependency Inversion en 3 Capes

Fins ara tens dues peces:
- `ChampionRecord` — el model immutable (dilluns)
- `InMemoryChampionRepository` — la persistencia en memoria (dimarts)

Avui afegim la tercera: el servei de negoci. L'arquitectura queda aixi:

```
┌──────────────────────────────────┐
│    ChampionManagementService     │  ← Logica de negoci: QUE fer
│  (orquestra, no crea ni guarda)  │
└────────────┬─────────────────────┘
             │ depèn de (interficie)
┌────────────▼─────────────────────┐
│       ChampionRepository         │  ← Contracte: quins metodes existeixen
│     (interficie abstracta)       │
└────────────┬─────────────────────┘
             │ implementa
┌────────────▼─────────────────────┐
│  InMemoryChampionRepository      │  ← COM es fa: detall d'implementacio
│   (ConcurrentHashMap)            │
└──────────────────────────────────┘
```

**La regla clau:** les fletxes de dependencia van cap AMUNT. El servei depèn de la interficie, no de la implementacio concreta. La implementacio concreta tambe depèn de la interficie (la implementa). Ningu depèn de ningu cap avall.

### Factory Pattern: Centralitzar la Creacio

El servei no hauria de saber com es crea un `ChampionRecord` — nomes que en vol un. Per aixo usem un Factory:

```java
// Factory: responsable de CREAR ChampionRecords amb logica centralitzada
// Si el format de creacio canvia, nomes toques aquesta classe
public class ChampionRecordFactory {

    // Crea un ChampionRecord amb valors per defecte (pickRate 0.0)
    // Util per quan registres un campio nou que encara no te dades de meta
    public ChampionRecord createDefault(String championId, String name, double winRate) {
        return new ChampionRecord(championId, name, winRate, 0.0);
    }

    // Crea un ChampionRecord a partir d'un JSON de la Riot API
    // Encapsula tota la logica de parsing — el servei no sap res de JSON
    public ChampionRecord createFromRiotAPI(String championId, String json) {
        // Parseja el JSON de la Riot API per extreure els camps necessaris
        String name = extractField(json, "name");
        double winRate = Double.parseDouble(extractField(json, "winRate"));
        double pickRate = Double.parseDouble(extractField(json, "pickRate"));

        // Crea el record — la validacio del compact constructor s'aplica automaticament
        return new ChampionRecord(championId, name, winRate, pickRate);
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
public class ChampionManagementService {

    // Dependencies declarades com a interficies (abstraccions)
    // El servei no sap si el repo es in-memory, SQL, o MongoDB
    private final ChampionRepository repo;
    private final ChampionRecordFactory factory;

    // Constructor injection: qui crea el servei decideix QUINES implementacions usar
    // Aixo es DIP pur: el servei depèn d'abstraccions, no de concrecions
    public ChampionManagementService(ChampionRepository repo, ChampionRecordFactory factory) {
        this.repo = repo;
        this.factory = factory;
    }

    // Registra un campio nou: delega creacio al factory, persistencia al repo
    public void registerChampion(String championId, String name, double winRate) {
        // El factory crea el record (amb validacio del compact constructor)
        ChampionRecord champion = factory.createDefault(championId, name, winRate);
        // El repo el guarda (no sabem on — in-memory? SQL? No importa)
        repo.save(champion);
    }

    // Retorna nomes els campions meta (amb pickRate i winRate alts)
    // Filtra usant el metode isMeta() del propi ChampionRecord
    public List<ChampionRecord> getMetaChampions() {
        return repo.findAll()        // Obte tots els campions del repo
            .stream()                // Converteix a stream per filtrar
            .filter(ChampionRecord::isMeta)  // Filtra els que son meta
            .toList();               // Converteix el resultat a llista
    }

    // Busca un campio per ID — delega al repo i retorna Optional
    public Optional<ChampionRecord> findChampion(String championId) {
        return repo.findById(championId);
    }
}
```

### Per Que Constructor Injection Importa

```java
// MAL — el servei CREA les seves dependencies
public class ChampionManagementService {
    // Acoblat a InMemoryChampionRepository — si vull SQL, he de modificar AQUESTA classe
    private ChampionRepository repo = new InMemoryChampionRepository();

    // Impossible de testejar amb un mock — sempre usa InMemory
}

// BE — el servei REP les seves dependencies
public class ChampionManagementService {
    private final ChampionRepository repo;

    // Qui crea el servei decideix la implementacio
    public ChampionManagementService(ChampionRepository repo, ChampionRecordFactory factory) {
        this.repo = repo;
        this.factory = factory;
    }
}

// Produccio: usa SQL
ChampionRepository sqlRepo = new SqlChampionRepository(dataSource);
ChampionManagementService prodService = new ChampionManagementService(sqlRepo, factory);

// Test: usa un mock o in-memory — SENSE tocar el servei
ChampionRepository testRepo = new InMemoryChampionRepository();
ChampionManagementService testService = new ChampionManagementService(testRepo, factory);
```

**El benefici real:** als tests de demà podras crear un `InMemoryChampionRepository`, injectar-lo al servei, i testejar la logica de negoci sense cap base de dades. Si el servei creés les seves dependencies, no podries fer-ho.

### `.cursorrules` com a Especificacio de Comportament

Un `.cursorrules` no es un fitxer de configuracio generic — es la primera especificacio que escrius per a un agent IA. Si les regles son vagues, l'agent genera codi inconsistent. Si son precises, el codi surt coherent amb el projecte.

**Diferencia entre vague i precis:**

```
# VAGUE — l'agent interpretara com vulgui
"Usa bons noms de variable"
"Segueix bones practiques"
"Escriu codi net"

# PRECIS — l'agent sap exactament que fer
"Variables Java en camelCase: championRecord, pickRate"
"Variables Python en snake_case: champion_record, pick_rate"
"Models de domini: record (Java), @dataclass(frozen=True) (Python). Mai setters."
"Cada classe publica necessita un test JUnit corresponent"
"Commits: Conventional Commits (feat/fix/test/docs/refactor)"
```

**La llico:** Escriure specs per a una IA es escriure specs per a un dev junior molt rapid pero amb zero context. Si no li dius com vols les coses, les fara a la seva manera — i no sera la teva.

---

## Activitat

### 1. Crear `ChampionRecordFactory` (20 min)

Crea el fitxer:
```
backend-java/src/main/java/com/esportspulse/engine/factory/ChampionRecordFactory.java
```

Implementa:
- `createDefault(String championId, String name, double winRate)` — crea un ChampionRecord amb pickRate 0.0
- `createFromRiotAPI(String championId, String json)` — parseja JSON simplificat i crea un ChampionRecord

### 2. Crear `ChampionManagementService` (30 min)

Crea el fitxer:
```
backend-java/src/main/java/com/esportspulse/engine/service/ChampionManagementService.java
```

Implementa:
- Constructor que rep `ChampionRepository` i `ChampionRecordFactory` (DIP)
- `registerChampion(String championId, String name, double winRate)`
- `getMetaChampions()` — filtra amb stream + `isMeta()`
- `findChampion(String championId)` — delega al repo

### 3. Integrar les 3 capes (20 min)

```java
public class ServiceDemo {
    public static void main(String[] args) {
        ChampionRepository repo = new InMemoryChampionRepository();
        ChampionRecordFactory factory = new ChampionRecordFactory();
        ChampionManagementService service = new ChampionManagementService(repo, factory);

        // Registrem campions usant el SERVEI
        service.registerChampion("CHAMP-1", "Ahri", 52.3);
        service.registerChampion("CHAMP-2", "Yasuo", 49.1);
        service.registerChampion("CHAMP-3", "Jinx", 51.5);

        // Nota: registerChampion crea campions amb pickRate 0.0 (createDefault)
        // Aixi que getMetaChampions() retornara llista buida
        List<ChampionRecord> meta = service.getMetaChampions();
        System.out.println("Meta: " + meta.size());  // 0 — tots tenen pickRate 0

        // Busquem un campio per ID
        service.findChampion("CHAMP-1").ifPresent(
            c -> System.out.println("Trobat: " + c.name())  // "Trobat: Ahri"
        );

        // Busquem un campio que no existeix
        boolean exists = service.findChampion("CHAMP-999").isPresent();
        System.out.println("CHAMP-999 existeix? " + exists);  // false
    }
}
```

### 4. Escriure `.cursorrules` com a Spec (30 min)

Crea o reescriu el fitxer `.cursorrules` a l'arrel del projecte. Ha de ser una especificacio precisa:

```
# EsportsPulse Engine — Especificacio per a l'Assistent

## Llenguatge i Convencions
- Java 21: variables en camelCase (championRecord, pickRate)
- Python 3.12: variables en snake_case (champion_record, pick_rate)
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
- Exemple: `feat(java): add ChampionManagementService with DI`
- Branques: `feature/weekN-description`
```

**Verificacio:** Demana a Cursor: "Genera un `PlayerRecord` seguint les convencions del projecte". L'assistent ha de generar un `record` (no una classe amb setters), amb compact constructor i validacio. Si no ho fa, ajusta les regles.

### 5. Commit (5 min)

```bash
git add backend-java/src/main/java/com/esportspulse/engine/factory/
git add backend-java/src/main/java/com/esportspulse/engine/service/
git add .cursorrules
git commit -m "feat(java): ChampionManagementService with DI + ChampionRecordFactory + .cursorrules spec"
```

---

## Checklist de Lliurament

- [ ] `ChampionRecordFactory` amb `createDefault` i `createFromRiotAPI`
- [ ] `ChampionManagementService` rep `ChampionRepository` i `ChampionRecordFactory` per constructor (DIP)
- [ ] `registerChampion()` usa el factory per crear i el repo per guardar
- [ ] `getMetaChampions()` filtra correctament amb streams
- [ ] Demo de les 3 capes integrades funciona sense errors
- [ ] `.cursorrules` escrit com a especificacio precisa (no generica)
- [ ] Cursor genera `PlayerRecord` com a `record` (no POJO) quan li demanes
- [ ] Commit amb format Conventional Commits
