# Setmana 2 — Dimecres: Dependency Injection, Factory i Capa de Servei

## Objectiu del Dia

Construir la capa de servei (`ChampionManagementService`) que aplica Dependency Injection de forma real: rep el repositori per constructor, no crea res directament. Usar el Factory Pattern per centralitzar la creació d'entitats. Al final del dia tens les 3 capes integrades (model → repository → servei) funcionant juntes amb injecció de dependències manual.

---

## Teoria

### Principis SOLID: Per Què el Codi s'Organitza Així

Dilluns vas crear `ChampionRecord` amb validació. Dimarts vas crear `ChampionRepository` i `InMemoryChampionRepository`. Avui afegirem un servei. Però **per què** separem el codi en tantes classes? Per què no ho posem tot en una sola?

Imagina que trobes aquest codi a un projecte real:

```java
// Classe que fa MASSA coses
public class ChampionService {
    private List<ChampionRecord> champions;  // Emmagatzema campions
    private Database db;                     // Connexio a base de dades
    private EmailService emailService;       // Envia correus
    private APIClient riotAPI;               // Crida a API externa

    public void registerChampion(String championId, String name) {
        if (championId == null) throw new Exception("Invalid");
        ChampionRecord g = new ChampionRecord(championId, name, 0, 0);
        champions.add(g);
        db.save(g);
        emailService.notifyAdmins("Nou campió: " + name);
    }
}
```

**Problemes concrets:**
- Fa 4 coses: valida, crea, guarda, notifica
- Si vols canviar com envies emails, toques una classe que tambe guarda a la BD
- Per testejar `registerChampion` necessites mock de 4 serveis — un test senzill es converteix en 40 linies de setup

**SOLID** son 5 principis que resolen exactament aquests problemes. No son teoria abstracta — son la diferencia entre codi que pots mantenir i codi que ningú vol tocar.

| Principi | Regla | Aplicat avui |
|----------|-------|-------------|
| **S** — Single Responsibility | Cada classe fa una sola cosa | El servei orquestra, el factory crea, el repo guarda |
| **O** — Open/Closed | Afegir funcionalitat sense modificar codi existent | Nova implementació de repo (SQL) sense tocar el servei |
| **L** — Liskov Substitution | Tota implementació d'una interfície és intercanviable | InMemory i SQL compleixen el mateix contracte |
| **I** — Interface Segregation | Interfícies petites, no gegants | `ChampionRepository` té 4 mètodes, no 20 |
| **D** — Dependency Inversion | Dependre d'interfícies, no de classes concretes | El servei rep `ChampionRepository`, no `InMemoryChampionRepository` |

Avui aplicaràs els 5 de forma pràctica: el servei que crearàs demostra SRP (només orquestra), OCP (funciona amb qualsevol repo), i DIP (rep dependències per constructor).

> **Lectura i vídeos recomanats sobre SOLID (opcional, no bloquejant):**
> - Vídeo — [Los principios SOLID, ¡explicados!](https://www.youtube.com/watch?v=2X50sKeBAcQ) — Explicació visual i clara dels 5 principis en castellà
> - Baeldung — [A Solid Guide to SOLID Principles](https://www.baeldung.com/solid-principles) — Exemples en Java per a cada principi

### Que es la Injecció de Dependències i Per Què Importa

Quan una classe necessita una altra per funcionar, diem que en **depèn**. Per exemple, un servei que guarda campions depèn d'un repositori. La pregunta és: **qui decideix quin repositori s'utilitza?**

```java
// SENSE injecció: el servei decideix (i queda acoblat)
public class ChampionManagementService {
    private ChampionRepository repo = new InMemoryChampionRepository();  // Hardcodejat!
}
```

Això crea tres problemes concrets:

1. **No pots testejar fàcilment.** Si vols testejar el servei sense base de dades, no pots — sempre crea un `InMemoryChampionRepository`. Per usar un mock o un repositori de test, hauries de modificar el codi del servei.

2. **No pots canviar la implementació sense tocar el servei.** Si demà vols guardar a PostgreSQL en comptes de memòria, has d'obrir el servei i canviar la línia. Cada canvi d'infraestructura toca la lògica de negoci.

3. **No pots reutilitzar el servei en contextos diferents.** Si un endpoint REST vol un repositori SQL i un test vol un repositori en memòria, necessites dues versions del servei.

La **injecció de dependències** resol això: el servei **no crea** les seves dependències, les **rep** pel constructor.

```java
// AMB injecció: qui crea el servei decideix (i el servei queda lliure)
public class ChampionManagementService {
    private final ChampionRepository repo;

    public ChampionManagementService(ChampionRepository repo) {
        this.repo = repo;  // Rep el que li passin — no sap ni li importa quina implementació és
    }
}

// A producció:
new ChampionManagementService(new SqlChampionRepository(dataSource));

// Als tests:
new ChampionManagementService(new InMemoryChampionRepository());
```

**El mateix servei, zero canvis, dos contextos diferents.** Això és DIP (Dependency Inversion Principle) aplicat — el principi que vas veure dilluns a la teoria de SOLID.

### Dependency Inversion en 3 Capes

Fins ara tens dues peces:
- `ChampionRecord` — el model immutable amb validació (dilluns)
- `InMemoryChampionRepository` — la persistència en memòria (dimarts)

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

### 4. Commit (5 min)

```bash
git add backend-java/src/main/java/com/esportspulse/engine/factory/
git add backend-java/src/main/java/com/esportspulse/engine/service/
git commit -m "feat(java): ChampionManagementService with DI + ChampionRecordFactory"
```

---

## Checklist de Lliurament

- [ ] `ChampionRecordFactory` amb `createDefault` i `createFromRiotAPI`
- [ ] `ChampionManagementService` rep `ChampionRepository` i `ChampionRecordFactory` per constructor (DIP)
- [ ] `registerChampion()` usa el factory per crear i el repo per guardar
- [ ] `getMetaChampions()` filtra correctament amb streams
- [ ] Demo de les 3 capes integrades funciona sense errors
- [ ] Commit amb format Conventional Commits
