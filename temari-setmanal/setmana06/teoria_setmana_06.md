# Setmana 5 - Teoria: Persistència, JPA i SQL Real

## 1. Introducció: De la RAM a la Base de Dades

Fins ara, EsportsPulse guarda els champions en un `ConcurrentHashMap` dins de `InMemoryChampionRepository`:

```
Aplicació s'inicia → HashMap buit
Afegeixes 50 champions → HashMap amb 50 entries
Aplicació es para → TOT DESAPAREIX
```

A producció, les dades han de sobreviure reinicis, desplegaments, i crashes. Necessitem una **base de dades**.

```
Sense BD (S2-S4):                    Amb BD (S5+):
┌───────────────┐                    ┌───────────────┐
│  Java App     │                    │  Java App     │
│  ┌─────────┐  │                    │  ┌─────────┐  │
│  │ HashMap │  │                    │  │   JPA   │──│──→ Base de Dades (H2)
│  │ (RAM)   │  │                    │  └─────────┘  │     ┌──────────────────┐
│  └─────────┘  │                    └───────────────┘     │ Taula: champions │
└───────────────┘                                          │  championId (PK) │
   Dades en RAM                                            │  name            │
   → es perden                                             │  winRate         │
                                                           │  gamesPlayed     │
                                                           └──────────────────┘
                                                           Dades a disc
                                                           → sobreviuen
```

---

## 2. SQL: El Llenguatge de les Bases de Dades

### Què és SQL?

SQL (Structured Query Language) és el llenguatge per parlar amb bases de dades relacionals. Porta 50 anys funcionant i no té substitut.

### Les 4 Operacions Bàsiques (CRUD)

```sql
-- CREATE: Inserir dades
INSERT INTO champions (champion_id, name, role, win_rate, games_played)
VALUES ('jinx', 'Jinx', 'marksman', 52.30, 500000);

-- READ: Consultar dades
SELECT * FROM champions WHERE champion_id = 'jinx';
SELECT name, win_rate FROM champions WHERE win_rate > 52.00 ORDER BY win_rate;

-- UPDATE: Modificar dades
UPDATE champions SET win_rate = 51.50 WHERE champion_id = 'yasuo';

-- DELETE: Eliminar dades
DELETE FROM champions WHERE champion_id = 'yasuo';
```

### Crear la Taula

```sql
CREATE TABLE champions (
    champion_id VARCHAR(50) PRIMARY KEY,  -- Clau única, no es repeteix
    name VARCHAR(255) NOT NULL,           -- Obligatori
    role VARCHAR(50) NOT NULL,            -- Rol del champion
    win_rate DECIMAL(5, 2) NOT NULL,      -- 2 decimals
    games_played BIGINT DEFAULT 0
);
```

**PRIMARY KEY:** Com el `championId` del HashMap — identifica cada fila de forma única. La BD crea automàticament un **index** per buscar ràpidament per `champion_id` (equivalent al hash del HashMap).

### Filtrar i Ordenar

```sql
-- Champions marksman ordenats per partides jugades
SELECT name, games_played 
FROM champions 
WHERE role = 'marksman' 
ORDER BY games_played DESC;

-- Els 10 champions amb més partides
SELECT name, games_played 
FROM champions 
ORDER BY games_played DESC 
LIMIT 10;

-- Champions que contenen "Jinx" al nom
SELECT * FROM champions 
WHERE name LIKE '%Jinx%';
```

### Funcions d'Agregació

```sql
-- Quants champions tenim?
SELECT COUNT(*) FROM champions;

-- WinRate mitjà dels champions amb winRate positiu
SELECT AVG(win_rate) FROM champions WHERE win_rate > 0;

-- Total de partides jugades per rang de winRate
SELECT 
    CASE 
        WHEN win_rate < 48 THEN 'Low'
        WHEN win_rate <= 52 THEN 'Average'
        ELSE 'High'
    END AS tier,
    COUNT(*) AS champion_count,
    SUM(games_played) AS total_games
FROM champions
GROUP BY tier;
```

### JOINs: Relacionar Taules

Quan la BD creixi (S15 amb PostgreSQL), tindrem múltiples taules:

```sql
-- Taules
CREATE TABLE champions (
    champion_id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);

CREATE TABLE patches (
    id SERIAL PRIMARY KEY,
    champion_id VARCHAR(50) REFERENCES champions(champion_id),  -- Foreign key
    patch_version VARCHAR(20),
    release_date DATE,
    description TEXT
);

-- JOIN: Obtenir champions amb els seus patches de balance
SELECT g.name, p.patch_version, p.release_date
FROM champions g
JOIN patches p ON g.champion_id = p.champion_id
WHERE g.name = 'Jinx'
ORDER BY p.release_date DESC;
```

```
Resultat:
┌──────┬───────────────┬────────────┐
│ name │ patch_version │ release_date│
├──────┼───────────────┼────────────┤
│ Jinx │ 14.5          │ 2024-03-06 │
│ Jinx │ 14.4          │ 2024-02-22 │
│ Jinx │ 14.3          │ 2024-02-07 │
└──────┴───────────────┴────────────┘
```

**Per què importa:** A les entrevistes de backend, et demanaran escriure JOINs. JPA els amaga, però has de saber què passa per sota.

---

## 3. Indexes: Per Què les Queries Són Ràpides (o Lentes)

### Sense Index

```sql
SELECT * FROM champions WHERE name = 'Jinx';
```

Sense index, la BD recorre **tota la taula** fila per fila:

```
Taula champions (100.000 files):
Fila 1: ahri, "Ahri"                  ← No
Fila 2: yasuo, "Yasuo"                ← No
Fila 3: thresh, "Thresh"              ← No
...
Fila 42857: jinx, "Jinx"              ← TROBAT! (però ha mirat 42.857 files)
...continua fins al final per si n'hi ha més...
Fila 100000: sona, "Sona"

→ Full Table Scan: O(n) — exactament el problema de S1 amb ArrayList!
```

### Amb Index

```sql
CREATE INDEX idx_champions_name ON champions(name);
```

La BD crea una estructura auxiliar (B-Tree) que permet trobar files per `name` en O(log n):

```
Index B-Tree per name:
                    ┌─────────────────┐
                    │  "J" < "Jinx"   │
                    │  → branca dreta  │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ "Jinx"         │
                    │ → Fila 42857   │
                    └─────────────────┘

→ Index Scan: O(log n) — 17 passos per a 100.000 files (en lloc de 100.000)
```

### EXPLAIN: Veure Què Fa la BD

```sql
EXPLAIN SELECT * FROM champions WHERE name = 'Jinx';
```

```
Sense index:
Seq Scan on champions  (cost=0.00..1850.00 rows=1 width=120)
  Filter: (name = 'Jinx')
→ "Seq Scan" = recorregut seqüencial = LENT

Amb index:
Index Scan using idx_champions_name on champions  (cost=0.00..8.27 rows=1 width=120)
  Index Cond: (name = 'Jinx')
→ "Index Scan" = usa l'index = RÀPID
```

**Connexió amb S1:** Un index de BD és l'equivalent d'un HashMap per a dades a disc. La PRIMARY KEY ja crea un index automàticament — per això `findById` sempre és ràpid.

### Quan Crear Indexes

| Cas | Index? | Per què |
|-----|--------|---------|
| `WHERE champion_id = ?` | Ja existeix (PK) | Primary Key és index automàtic |
| `WHERE name = ?` | ✅ Crear | Cerques freqüents per nom |
| `WHERE win_rate > 52` | Depèn | Només si la query és freqüent |
| `ORDER BY games_played DESC` | ✅ Crear | Rankings/top lists |
| `WHERE name LIKE '%Jinx%'` | ❌ No serveix | LIKE amb % al principi no pot usar index B-Tree |

### Trade-off dels Indexes

```
Sense index:
- SELECT: Lent (full scan)
- INSERT/UPDATE: Ràpid (no cal actualitzar index)
- Espai: Mínim

Amb index:
- SELECT: Ràpid (index scan)
- INSERT/UPDATE: Una mica més lent (cal actualitzar index)
- Espai: Index ocupa disc extra
```

**Regla pràctica:** Crea indexes per a columnes que apareixen en `WHERE`, `JOIN ON`, i `ORDER BY` en queries freqüents. No indexis tot.

---

## 4. ACID: Les Garanties d'una Base de Dades

### Què és ACID?

```
A - Atomicitat:    Tot o res (si falla a mig camí, es desfà tot)
C - Consistència:  La BD sempre queda en estat vàlid
I - Isolament:     Transaccions concurrents no es trepitgen
D - Durabilitat:   Un cop fet COMMIT, les dades sobreviuen un crash
```

### Atomicitat en Pràctica

```java
@Transactional
public void transferGamesPlayed(String fromChampionId, String toChampionId, long count) {
    ChampionRecord from = repository.findById(fromChampionId).orElseThrow();
    ChampionRecord to = repository.findById(toChampionId).orElseThrow();
    
    from.setGamesPlayed(from.getGamesPlayed() - count);
    to.setGamesPlayed(to.getGamesPlayed() + count);
    
    repository.save(from);
    // Si aquí falla (excepció, crash, timeout)...
    repository.save(to);  
}
```

**Sense @Transactional:** Si falla entre els dos `save()`, `from` ha perdut partides però `to` no les ha guanyat. Partides desaparegudes.

**Amb @Transactional:** Si falla, Spring fa ROLLBACK → ambdós canvis es desfan. Les dades queden com estaven.

```
Sense @Transactional:          Amb @Transactional:
BEGIN                          BEGIN
UPDATE from: -1000 ✅          UPDATE from: -1000 ✅
💥 Error!                      💥 Error!
UPDATE to: +1000 ❌            ROLLBACK → from torna a l'original
→ 1000 partides perdudes       → Tot queda com estava
```

### Isolament: Concurrent DB Access (Connexió amb S3)

A S3 vam veure el "Lost Update":

```
Thread A: READ champion.winRate = 51.20
Thread B: READ champion.winRate = 51.20
Thread A: WRITE champion.winRate = 49.80  ✅
Thread B: WRITE champion.winRate = 53.10  ✅  ← Sobreescriu el canvi de A!
```

`@Transactional` amb isolation level controla això:

| Isolation Level | Permet Lost Update? | Rendiment |
|----------------|-------------------|-----------|
| READ_UNCOMMITTED | Sí ❌ | Molt ràpid |
| READ_COMMITTED (defecte) | Possible ⚠️ | Bo |
| REPEATABLE_READ | No ✅ | Acceptable |
| SERIALIZABLE | No ✅ | Lent (serialitza tot) |

**En pràctica:** `READ_COMMITTED` + Optimistic Locking (`@Version`) és la combinació més comuna.

---

## 5. JPA: L'Abstracció sobre SQL

### Què és JPA?

JPA (Java Persistence API) mapeja classes Java a taules SQL:

```
Java                           SQL
────                           ───
@Entity                   →    CREATE TABLE
class ChampionRecord      →    champions
@Id String championId     →    champion_id VARCHAR PRIMARY KEY
String name               →    name VARCHAR
BigDecimal winRate        →    win_rate DECIMAL

repository.save(champion) →    INSERT INTO champions VALUES (...)
repository.findById(id)   →    SELECT * FROM champions WHERE champion_id = ?
repository.findAll()      →    SELECT * FROM champions
repository.delete(champion)→   DELETE FROM champions WHERE champion_id = ?
```

### @Entity: Convertir una Classe en Taula

```java
@Entity
@Table(name = "champions")
public class ChampionRecord {
    
    @Id
    private String championId;
    
    @Column(nullable = false)
    private String name;
    
    @Column(nullable = false, length = 50)
    private String role;
    
    @Column(precision = 5, scale = 2)
    private BigDecimal winRate;
    
    @Column(name = "games_played")
    private Long gamesPlayed;
    
    @Version
    private Long version;  // Per Optimistic Locking (S3)
    
    // JPA necessita constructor buit
    protected ChampionRecord() {}
    
    public ChampionRecord(String championId, String name, String role, BigDecimal winRate, Long gamesPlayed) {
        this.championId = championId;
        this.name = name;
        this.role = role;
        this.winRate = winRate;
        this.gamesPlayed = gamesPlayed;
    }
    
    // Getters (i setters si necessaris per JPA)
}
```

**Nota sobre immutabilitat:** A S2 vam usar `record` (immutable). JPA entities necessiten setters per actualitzar camps. El trade-off:
- `record` → immutable, thread-safe, ideal per DTOs i transferir dades
- `@Entity class` → mutable (JPA ho requereix), però protegit per `@Transactional`
- Solució: Entity mutable a la capa de persistència, DTO immutable (`record`) a la capa d'API

### JpaRepository: CRUD Automàtic

```java
public interface ChampionJpaRepository extends JpaRepository<ChampionRecord, String> {
    
    // Spring genera la implementació SQL automàticament!
    // No has d'escriure cap línia de SQL ni d'implementació.
    
    // findAll()    → SELECT * FROM champions
    // findById()   → SELECT * FROM champions WHERE champion_id = ?
    // save()       → INSERT/UPDATE
    // delete()     → DELETE FROM champions WHERE champion_id = ?
    
    // Queries derivades del nom del mètode:
    List<ChampionRecord> findByNameContaining(String keyword);
    // → SELECT * FROM champions WHERE name LIKE '%keyword%'
    
    List<ChampionRecord> findByGamesPlayedGreaterThan(Long count);
    // → SELECT * FROM champions WHERE games_played > ?
    
    List<ChampionRecord> findByWinRateBetween(BigDecimal min, BigDecimal max);
    // → SELECT * FROM champions WHERE win_rate BETWEEN ? AND ?
}
```

**Com funciona?** Spring Data llegeix el nom del mètode i genera el SQL:

```
findByNameContaining
  │   │     │
  │   │     └── LIKE '%...%'
  │   └──────── WHERE name
  └──────────── SELECT * FROM champions
```

### @Query: SQL Explícit

Quan el nom del mètode no és suficient:

```java
@Query("SELECT g FROM ChampionRecord g WHERE g.winRate > :minWinRate ORDER BY g.gamesPlayed DESC")
List<ChampionRecord> findHighWinRateMetaChampions(@Param("minWinRate") BigDecimal minWinRate);

// SQL natiu (per queries complexes)
@Query(value = "SELECT * FROM champions WHERE name ILIKE %:keyword%", nativeQuery = true)
List<ChampionRecord> searchIgnoreCase(@Param("keyword") String keyword);
```

---

## 6. H2: La Base de Dades de Desenvolupament

### Què és H2?

H2 és una BD relacional que corre **dins de la JVM** (in-memory o a fitxer):

```
Producció:                          Desenvolupament:
┌─────────────┐    xarxa    ┌───────────┐    ┌─────────────┐   dins la JVM   ┌────┐
│ Java App    │ ──────────→ │ PostgreSQL│    │ Java App    │ ─────────────→ │ H2 │
└─────────────┘             └───────────┘    │ ┌────┐      │               └────┘
                                             │ │ H2 │      │
                                             │ └────┘      │
                                             └─────────────┘
```

**Avantatges per desenvolupament:**
- Zero configuració: només una dependència Maven
- Ràpida: tot a RAM
- Consola web: `http://localhost:8080/h2-console`
- Compatible amb SQL estàndard

### Configuració

```properties
# application.properties
spring.datasource.url=jdbc:h2:mem:esportspulse
spring.datasource.driver-class-name=org.h2.Driver
spring.jpa.hibernate.ddl-auto=create-drop
spring.h2.console.enabled=true
```

`ddl-auto=create-drop`: JPA crea les taules automàticament a partir de les `@Entity`. Pràctic per desenvolupament; **mai en producció** (per això a S15 introduirem Flyway per migracions controlades).

---

## 7. El Poder del Pattern Repository: El Swap

El moment clau de S5: reemplaçar `InMemoryChampionRepository` per `ChampionJpaRepository` **sense canviar cap línia de `ChampionManagementService`**:

```
Setmanes 2-4:                        Setmana 5:

ChampionManagementService                 ChampionManagementService
    │                                     │
    ▼                                     ▼
ChampionRepository (interfície)           ChampionRepository (interfície)
    │                                     │
    ▼                                     ▼
InMemoryChampionRepository               ChampionJpaRepository
    │                                     │
    ▼                                     ▼
ConcurrentHashMap (RAM)               H2 Database (SQL)
```

El codi de `ChampionManagementService` és **exactament el mateix**:

```java
@Service
public class ChampionManagementService {
    private final ChampionRepository repository;  // No sap si és In-Memory o SQL!
    
    public ChampionManagementService(ChampionRepository repository) {
        this.repository = repository;
    }
    
    public void registerChampion(String championId, String name, String role, BigDecimal winRate) {
        ChampionRecord champion = new ChampionRecord(championId, name, role, winRate, 0L);
        repository.save(champion);  // Funciona amb HashMap o SQL
    }
    
    public List<ChampionRecord> getMetaChampions() {
        return repository.findAll().stream()
            .filter(g -> g.getGamesPlayed() > 100_000)
            .toList();
    }
}
```

**Això és SOLID en acció:**
- **D (Dependency Inversion):** Depèn de l'abstracció (`ChampionRepository`), no de la implementació
- **O (Open/Closed):** Canviem la persistència sense modificar la lògica de negoci
- **L (Liskov):** `ChampionJpaRepository` és substituïble per `InMemoryChampionRepository`

**I tots els tests de S2-S4 segueixen passant** perquè testejaven contra la interfície.

---

## 8. Python: Persistència amb SQLite

### L'Equivalent a H2 en Python

```python
import sqlite3
from dataclasses import dataclass

@dataclass(frozen=True)
class ChampionRecord:
    champion_id: str
    name: str
    role: str
    win_rate: float
    games_played: int

class SqliteChampionRepository:
    def __init__(self, db_path: str = ":memory:"):
        self.conn = sqlite3.connect(db_path)
        self.conn.execute("""
            CREATE TABLE IF NOT EXISTS champions (
                champion_id TEXT PRIMARY KEY,
                name TEXT NOT NULL,
                role TEXT NOT NULL,
                win_rate REAL NOT NULL,
                games_played INTEGER DEFAULT 0
            )
        """)
    
    def save(self, champion: ChampionRecord) -> None:
        self.conn.execute(
            "INSERT OR REPLACE INTO champions VALUES (?, ?, ?, ?, ?)",
            (champion.champion_id, champion.name, champion.role, champion.win_rate, champion.games_played)
        )
        self.conn.commit()
    
    def find_by_id(self, champion_id: str) -> ChampionRecord | None:
        row = self.conn.execute(
            "SELECT * FROM champions WHERE champion_id = ?", (champion_id,)
        ).fetchone()
        if row is None:
            return None
        return ChampionRecord(*row)
    
    def find_all(self) -> list[ChampionRecord]:
        rows = self.conn.execute("SELECT * FROM champions").fetchall()
        return [ChampionRecord(*row) for row in rows]
```

**Comparativa:**

| Concepte | Java (JPA + H2) | Python (sqlite3) |
|----------|-----------------|-------------------|
| BD lleugera | H2 | SQLite |
| ORM | JPA (`@Entity`, `JpaRepository`) | Manual (o SQLAlchemy a S8+) |
| Queries | Derivades del nom del mètode | SQL explícit |
| Paràmetres | `@Param("name")` | `?` o `:name` |
| Transaccions | `@Transactional` | `conn.commit()` / `conn.rollback()` |

### Tests amb pytest

```python
import pytest

@pytest.fixture
def repo():
    return SqliteChampionRepository(":memory:")

@pytest.fixture
def sample_champion():
    return ChampionRecord("jinx", "Jinx", "marksman", 52.3, 500_000)

def test_save_and_find(repo, sample_champion):
    repo.save(sample_champion)
    found = repo.find_by_id("jinx")
    assert found == sample_champion

def test_find_nonexistent(repo):
    assert repo.find_by_id("NOPE") is None

def test_find_all(repo, sample_champion):
    repo.save(sample_champion)
    repo.save(ChampionRecord("yasuo", "Yasuo", "fighter", 49.5, 400_000))
    assert len(repo.find_all()) == 2
```

---

## 9. Exercici SQL a la Consola H2

Un dels exercicis de la setmana és obrir la consola H2 (`http://localhost:8080/h2-console`) i escriure queries SQL a mà. Exemples:

```sql
-- 1. Inserir dades de prova
INSERT INTO champions VALUES ('jinx', 'Jinx', 'marksman', 52.30, 500000);
INSERT INTO champions VALUES ('yasuo', 'Yasuo', 'fighter', 49.50, 400000);
INSERT INTO champions VALUES ('ahri', 'Ahri', 'mage', 51.80, 350000);
INSERT INTO champions VALUES ('lux', 'Lux', 'mage', 50.20, 300000);
INSERT INTO champions VALUES ('thresh', 'Thresh', 'support', 48.90, 250000);

-- 2. Consultes bàsiques
SELECT * FROM champions ORDER BY games_played DESC;
SELECT name, win_rate FROM champions WHERE role = 'marksman';
SELECT COUNT(*) AS total, AVG(win_rate) AS avg_win_rate FROM champions;

-- 3. Verificar que JPA ha creat el que esperem
SHOW TABLES;
SHOW COLUMNS FROM champions;

-- 4. Veure el pla d'execució
EXPLAIN SELECT * FROM champions WHERE name = 'Lux';
-- Sense index: TABLE SCAN
CREATE INDEX idx_champions_name ON champions(name);
EXPLAIN SELECT * FROM champions WHERE name = 'Lux';
-- Amb index: INDEX SCAN
```

**Per què fer-ho a mà?** JPA genera SQL automàticament. Però quan una query és lenta o retorna dades incorrectes, necessites saber SQL per diagnosticar. "L'abstracció no substitueix el coneixement."

---

## Resum

| Concepte | Key Takeaway |
|----------|--------------|
| **SQL** | El llenguatge universal de BD; 50 anys i comptant |
| **CRUD** | INSERT, SELECT, UPDATE, DELETE — les 4 operacions bàsiques |
| **INDEX** | Equivalent al HashMap per a BD; O(log n) en lloc de O(n) |
| **EXPLAIN** | Eina per veure si una query usa index o fa full scan |
| **ACID** | Atomicitat, Consistència, Isolament, Durabilitat — les garanties |
| **@Transactional** | Tot o res; si falla, ROLLBACK automàtic |
| **JPA** | Mapeja classes Java a taules SQL; genera queries automàticament |
| **JpaRepository** | CRUD automàtic + queries derivades del nom del mètode |
| **H2** | BD in-memory per desenvolupament; zero configuració |
| **Repository Pattern** | Swap de In-Memory a SQL sense canviar la lògica de negoci |
| **SQLite (Python)** | L'equivalent a H2; BD lleugera integrada al llenguatge |

**Objectiu setmana:** Entendre SQL, persistir dades reals, i veure la força del pattern Repository quan canviem d'implementació sense tocar el negoci.
