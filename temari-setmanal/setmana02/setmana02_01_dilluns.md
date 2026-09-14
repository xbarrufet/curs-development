# Setmana 2 — Dilluns: Principis SOLID i Immutabilitat amb Records

## Objectiu del Dia

Entendre els 5 principis SOLID i per que son fonamentals per escriure codi mantenible. Crear el primer model immutable amb Java 21 Records i veure com un `record` elimina bugs que un POJO mutable permet. Al final del dia tens `ChampionRecord` creat, validat amb compact constructor, i 3 instancies provades a `main`.

---

## Teoria

### Per Que SOLID? El Problema Real

Imagina que ets al teu primer dia de feina i trobes aquest codi:

```java
// Classe que fa MASSA coses — et trobaras aixo a projectes reals
public class ChampionService {
    private List<ChampionRecord> champions;  // Emmagatzema campions
    private Database db;                     // Connexio a base de dades
    private EmailService emailService;       // Envia correus
    private APIClient riotAPI;               // Crida a API externa
    private AnalyticsTracker tracker;        // Trackeja events

    public void registerChampion(String championId, String name) {
        // 1. Valida (responsabilitat de validacio)
        if (championId == null) throw new Exception("Invalid");

        // 2. Crea l'objecte (responsabilitat de creacio)
        ChampionRecord g = new ChampionRecord(championId, name, 0, 0);

        // 3. Guarda a multiples llocs (responsabilitat de persistencia)
        champions.add(g);
        db.save(g);

        // 4. Envia notificacio (responsabilitat de notificacio)
        emailService.notifyAdmins("Nou campió: " + name);

        // 5. Trackeja (responsabilitat d'analytics)
        tracker.log("champion_registered", g.championId());
    }
}
```

**Problemes concrets d'aquest codi:**
- Fa 5 coses: validacio, creacio, persistencia, notificacio, tracking
- Si vols canviar com envies emails, toques una classe que tambe guarda a la BD
- Per testejar `registerChampion` necessites mock de 5 serveis — un test que hauria de ser senzill es converteix en 40 linies de setup
- Reutilitzar la logica de creacio a un altre lloc? Impossible, esta acoblada

SOLID son 5 principis que resolen exactament aquests problemes. No son teoria abstracta — son la diferencia entre codi que pots mantenir i codi que ningú vol tocar.

### S — Single Responsibility Principle (SRP)

**Definicio:** Una classe ha de tenir una unica rao per canviar.

Dit d'una altra manera: cada classe fa **una sola cosa** i la fa be. Si necessites canviar com es guarda a la base de dades, nomes hauries de tocar la classe de persistencia — no una classe que tambe valida dades, envia emails i trackeja events.

**Com detectar que estàs violant SRP:** Si per descriure el que fa una classe necessites dir "i" mes de dues vegades ("valida **i** guarda **i** notifica **i** trackeja"), la classe fa massa coses.

Aplicat al problema anterior, la solucio es separar responsabilitats:

```java
// RESPONSABILITAT 1: Crear ChampionRecords
// Aquesta classe nomes canvia si canvia la logica de creacio
public class ChampionRecordFactory {
    public ChampionRecord createFromRiotAPI(String championId, String json) {
        // Parseja el JSON i crea un record valid
        return new ChampionRecord(championId, name, winRate, pickRate);
    }
}

// RESPONSABILITAT 2: Guardar i recuperar dades
// Aquesta classe nomes canvia si canvia la font de dades
public interface ChampionRepository {
    void save(ChampionRecord g);
    ChampionRecord findById(String championId);
}

// RESPONSABILITAT 3: Orquestrar la logica de negoci
// Aquesta classe nomes canvia si canvien les regles de negoci
public class ChampionManagementService {
    private ChampionRepository repo;          // Delega persistencia
    private ChampionRecordFactory factory;    // Delega creacio

    public void registerChampion(String championId, String name, double winRate) {
        // Nomes fa una cosa: coordinar el flux de negoci
        ChampionRecord g = factory.createDefault(championId, name, winRate);
        repo.save(g);
    }
}

// RESPONSABILITAT 4: Notificacions (completament separada)
public class ChampionRegistrationNotifier {
    public void notifyAdmins(ChampionRecord g) {
        // Envia email, Slack, o el que sigui
    }
}
```

**Per que funciona:** Cada classe canvia per una unica rao. Si canvia el format de l'API de Riot, nomes toques `ChampionRecordFactory`. Si canvies de base de dades, nomes toques la implementacio del `ChampionRepository`. El servei de negoci no sap ni li importa.

### O — Open/Closed Principle (OCP)

**Definicio:** Obert per extensio, tancat per modificacio.

Aixo vol dir que hauries de poder **afegir funcionalitat nova** sense haver de **modificar el codi que ja funciona**. Com? Utilitzant interficies. Defineixes un contracte (interficie) i cada nova funcionalitat es una nova classe que implementa aquell contracte. El codi existent mai es toca.

**Analogia:** Pensa en un endoll de paret. L'endoll (interficie) no canvia — pots connectar-hi una lampada, un carregador o una nevera (implementacions). Per afegir un electrodomestic nou, no has de recablar la paret.

```java
// MAL — cada vegada que afegim un backend, modifiquem aquesta classe
public class ChampionRepository {
    public void save(ChampionRecord g) {
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
public interface ChampionRepository {
    void save(ChampionRecord g);
}

// Implementacio 1: memoria (Setmana 2)
public class InMemoryChampionRepository implements ChampionRepository {
    private List<ChampionRecord> list = new ArrayList<>();

    public void save(ChampionRecord g) {
        list.add(g);  // Guarda en memoria — per desenvolupament i tests
    }
}

// Implementacio 2: SQL (Setmana 5 — no cal tocar InMemory!)
public class SqlChampionRepository implements ChampionRepository {
    private Database db;

    public void save(ChampionRecord g) {
        db.insert(g);  // Guarda a base de dades
    }
}
```

**Per que funciona:** Per afegir MongoDB, crees una nova classe. Mai toques les existents. `ChampionManagementService` no canvia mai perque depèn de la interficie, no de la implementacio.

### L — Liskov Substitution Principle (LSP)

**Definicio:** Qualsevol implementacio d'una interficie ha de poder substituir una altra sense trencar el programa.

Si tens una interficie `ChampionRepository` amb un metode `save()`, **totes** les implementacions han de fer realment un `save()`. Si una implementacio llança una excepcio perque "no suporta" aquella operacio, estas trencant el contracte — qualsevol codi que confiï en `save()` petara de forma inesperada.

**Analogia:** Si un restaurant diu que te menú del dia, esperes poder demanar-lo. Si arribeixes i et diuen "tenim menú, però no el servim", el contracte esta trencat. Millor no posar-lo a la carta.

```java
// MAL — ReadOnlyChampionRepository trenca el contracte de save()
public class ReadOnlyChampionRepository extends ChampionRepository {
    public void save(ChampionRecord g) {
        // Llança excepcio! El codi que crida save() petara
        throw new UnsupportedOperationException("Read-only!");
    }
}

// BE — si necessites read-only, separa les interficies
public interface ChampionReadRepository {
    Optional<ChampionRecord> findById(String id);  // Nomes lectura
}

public interface ChampionWriteRepository {
    void save(ChampionRecord g);  // Nomes escriptura
}
// Cada servei demana nomes el que necessita
```

### I — Interface Segregation Principle (ISP)

**Definicio:** Els clients no han de dependre d'interficies que no usen.

Si una classe nomes necessita llegir dades, no l'obliguis a dependre d'una interficie que tambe te metodes per esborrar, bannejar i enviar notificacions. Com mes gran sigui la interficie, mes acoblat queda el codi: qualsevol canvi a un metode que ni uses et pot afectar.

**Analogia:** Imagina un comandament a distancia amb 200 botons. Tu nomes vols canviar de canal i pujar el volum. Seria millor un comandament amb 5 botons que fa el que necessites, que un de gegant on la majoria de botons no serveixen per a res.

```java
// MAL — una interficie gegant amb 20 metodes
public interface ChampionService {
    ChampionRecord findById(String id);
    void save(ChampionRecord g);
    void delete(String id);
    void ban(String id);
    void unban(String id);
    List<ChampionRecord> getRecommendations(String userId);
    // ... 15 metodes mes
}

// Si nomes vull llegir un campió, depenc de ban(), unban(), etc.

// BE — interficies petites i especifiques
public interface ChampionReadService {
    ChampionRecord findById(String id);       // Nomes lectura
    List<ChampionRecord> getAllChampions();
}

public interface ChampionWriteService {
    void save(ChampionRecord g);              // Nomes escriptura
    void delete(String id);
}

// Cada classe depèn NOMES del minim que necessita
public class SimpleChampionViewer {
    private ChampionReadService readService;  // No sap que existeix ban() o delete()

    public void viewChampion(String id) {
        ChampionRecord g = readService.findById(id);
    }
}
```

### D — Dependency Inversion Principle (DIP)

**Definicio:** Depèn d'abstraccions (interficies), mai de concrecions (implementacions concretes).

Quan una classe crea directament les seves dependencies amb `new`, queda **acoblada** a aquella implementacio concreta. Si demà vols canviar de base de dades, has de modificar la classe de negoci — que no hauria de saber ni importar-li on es guarden les dades. La solucio es **injectar** la dependencia pel constructor: la classe demana una interficie, i qui la crea decideix quina implementacio li passa.

**Analogia:** Un cotxe no fabrica les seves pròpies rodes — les rep muntades. Si vols posar rodes d'hivern, les canvies sense modificar el motor. El cotxe (servei) depèn del concepte "roda" (interficie), no d'una marca concreta (implementacio).

```java
// MAL — acoblat a InMemoryChampionRepository
public class ChampionManagementService {
    // Crea directament la implementacio concreta
    private InMemoryChampionRepository repo = new InMemoryChampionRepository();

    public void registerChampion(String championId, String name) {
        repo.save(factory.create(championId, name));
        // Si vull canviar a SQL, he de MODIFICAR aquesta classe
    }
}

// BE — depèn de la interficie, rep la implementacio per constructor
public class ChampionManagementService {
    private ChampionRepository repo;  // Interficie — no sap quina implementacio es

    // Constructor injection: qui crea el servei decideix quina implementacio usa
    public ChampionManagementService(ChampionRepository repo) {
        this.repo = repo;
    }

    public void registerChampion(String championId, String name) {
        repo.save(factory.create(championId, name));
        // Funciona igual amb InMemory, SQL, MongoDB... sense canviar res
    }
}

// Us: qui monta l'aplicacio decideix la implementacio
ChampionRepository repo = new InMemoryChampionRepository();  // Setmana 2
ChampionManagementService service = new ChampionManagementService(repo);

// Setmana 5: canvies una sola linia
ChampionRepository repo = new SqlChampionRepository(dataSource);  // Canvi sense tocar el servei
ChampionManagementService service = new ChampionManagementService(repo);
```

> **Lectura i vídeos recomanats sobre SOLID (opcional, no bloquejant):**
> - Vídeo — [Los principios SOLID, ¡explicados!](https://www.youtube.com/watch?v=2X50sKeBAcQ) — Explicació visual i clara dels 5 principis en castellà
> - Baeldung — [A Solid Guide to SOLID Principles](https://www.baeldung.com/solid-principles) — Exemples en Java per a cada principi

### Exercicis rapids de SOLID (10 min)

Abans de continuar, verifica que has entes els principis. Per a cada cas, indica **quin principi SOLID es viola** i com ho resoldries en una frase:

1. Una classe `ReportService` genera el PDF, l'envia per email i el guarda a la base de dades.

2. Tens aquest codi i cada cop que afegeixes un format nou, modifiques la classe:
   ```java
   public class ExportService {
       public void export(String format, Data data) {
           if (format.equals("csv")) { ... }
           else if (format.equals("json")) { ... }
           else if (format.equals("xml")) { ... }  // i creixent...
       }
   }
   ```

3. Una classe `DashboardController` depèn directament de `MySqlDatabase` i no pots testejar-la sense una base de dades real.

4. Una interficie `UserService` te 15 metodes (create, delete, ban, unban, sendEmail, generateReport...) i una classe que nomes necessita llegir usuaris ha d'implementar-los tots.

> **Solucions:** 1) SRP — separar en `PdfGenerator`, `EmailSender` i `ReportRepository`. 2) OCP — crear una interficie `Exporter` amb implementacions `CsvExporter`, `JsonExporter`, etc. 3) DIP — fer que `DashboardController` depengui d'una interficie `Database`, no de `MySqlDatabase`. 4) ISP — separar en interficies petites: `UserReadService`, `UserWriteService`, `UserAdminService`.

---

### SOLID Aplicat als Records

Els principis SOLID no nomes s'apliquen a serveis i repositoris — tambe guien com dissenyem les entitats de domini. Segons el principi de **Responsabilitat Unica (S — Single Responsibility)**, un record te una responsabilitat clara: **representar una entitat i garantir que sempre sigui valida**. Aixo vol dir que la validacio de dades pertany al propi record (compact constructor), i la logica de negoci que depèn exclusivament de les dades de l'entitat tambe hi pot viure (metodes com `isMeta()`). En canvi, operacions com guardar a la base de dades o enviar notificacions **no** pertanyen al record — son responsabilitats d'altres classes (repositoris, serveis), tal com has vist als exemples de SOLID.

A la setmana 1 vas crear `ChampionRecord` com un record senzill amb camps basics. Ara anem un pas mes enlla: afegim **validacio a la creacio** perque mai pugui existir un objecte invalid, i **metodes de negoci** dins el record.

### Compact Constructor: Validacio a la Creacio

```java
public record ChampionRecord(
    String championId,
    String name,
    double winRate,
    double pickRate
) {
    // Compact constructor — s'executa automaticament quan es crea el record
    // No cal "this.championId = championId" — Java ho fa sol amb records
    public ChampionRecord {
        // Validacio: si les dades son invalides, llança excepcio
        // Aixi MAI pot existir un ChampionRecord invalid al sistema
        if (championId == null || championId.isBlank()) {
            throw new IllegalArgumentException("championId no pot ser buit");
        }
        if (winRate < 0 || winRate > 100) {
            throw new IllegalArgumentException("winRate ha de ser entre 0 i 100");
        }
        if (pickRate < 0 || pickRate > 100) {
            throw new IllegalArgumentException("pickRate ha de ser entre 0 i 100");
        }
    }
}
```

### Metodes de Negoci al Record

```java
public record ChampionRecord(
    String championId,
    String name,
    double winRate,
    double pickRate
) {
    // Compact constructor (validacio)
    public ChampionRecord {
        if (championId == null || championId.isBlank()) {
            throw new IllegalArgumentException("championId no pot ser buit");
        }
        if (winRate < 0 || winRate > 100) {
            throw new IllegalArgumentException("winRate ha de ser entre 0 i 100");
        }
        if (pickRate < 0 || pickRate > 100) {
            throw new IllegalArgumentException("pickRate ha de ser entre 0 i 100");
        }
    }

    // Metode de negoci: un campió es "meta" si te winRate > 52 i pickRate > 10
    public boolean isMeta() {
        return winRate > 52.0 && pickRate > 10.0;
    }

    // Metode que "modifica" sense mutar — retorna un NOU record amb el winRate ajustat
    // L'original no canvia MAI
    public ChampionRecord withPatchAdjustment(double modifier) {
        // Calcula el nou winRate sumant el modifier (pot ser negatiu per nerfs)
        double adjustedWinRate = this.winRate + modifier;
        // Crea i retorna un record NOU — l'original segueix intacte
        return new ChampionRecord(championId, name, adjustedWinRate, pickRate);
    }
}
```

---

## Activitat

### 1. Crear `ChampionRecord` amb validacio (30 min)

Crea el fitxer:
```
backend-java/src/main/java/com/esportspulse/engine/model/ChampionRecord.java
```

Implementa el `record` amb:
- Camps: `championId` (String), `name` (String), `winRate` (double), `pickRate` (double)
- Compact constructor amb validacio (championId no null/buit, winRate entre 0 i 100, pickRate entre 0 i 100)
- Metode `isMeta()` que retorna `true` si `winRate > 52.0 && pickRate > 10.0`
- Metode `withPatchAdjustment(double modifier)` que retorna un NOU record amb el winRate ajustat

### 2. Provar a `main` (20 min)

Crea 3 instancies de `ChampionRecord` a un `main` temporal:

```java
public class ChampionRecordDemo {
    public static void main(String[] args) {
        // Campió meta (winRate > 52 i pickRate > 10)
        ChampionRecord ahri = new ChampionRecord("CHAMP-1", "Ahri", 52.3, 8.1);

        // Campió no meta
        ChampionRecord yasuo = new ChampionRecord("CHAMP-2", "Yasuo", 49.1, 12.5);

        // Campió amb ajust de patch
        ChampionRecord ahriNerfed = ahri.withPatchAdjustment(-2.5);

        // Verifica que isMeta funciona
        System.out.println(ahri.isMeta());     // false — winRate > 52 pero pickRate < 10
        System.out.println(yasuo.isMeta());    // false — winRate < 52

        // Verifica que withPatchAdjustment NO muta l'original
        System.out.println(ahri.winRate());           // 52.3 — no ha canviat
        System.out.println(ahriNerfed.winRate());     // 49.8 — reduit 2.5 punts

        // Prova amb un campió clarament meta
        ChampionRecord broken = new ChampionRecord("CHAMP-3", "Broken Champ",
            55.0, 15.0);
        ChampionRecord brokenBuffed = broken.withPatchAdjustment(1.5);
        System.out.println(broken.winRate());          // 55.0
        System.out.println(brokenBuffed.winRate());    // 56.5

        // Prova que la validacio funciona (ha de petar)
        try {
            new ChampionRecord(null, "Bad Champ", 50.0, 5.0);
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
git add backend-java/src/main/java/com/esportspulse/engine/model/ChampionRecord.java
git add .cursorrules
git commit -m "feat(java): immutable ChampionRecord with compact constructor + SOLID .cursorrules"
```

---

## Checklist de Lliurament

- [ ] `ChampionRecord` creat com a `record` amb 4 camps
- [ ] Compact constructor valida `championId` no null, `winRate` entre 0 i 100, `pickRate` entre 0 i 100
- [ ] `isMeta()` retorna `true` si `winRate > 52.0 && pickRate > 10.0`
- [ ] `withPatchAdjustment()` retorna un NOU record sense mutar l'original
- [ ] 3 instancies creades i provades a `main` — tot imprimeix el que s'espera
- [ ] `.cursorrules` escrit amb regles d'immutabilitat, noms i testing
- [ ] Commit amb format Conventional Commits
