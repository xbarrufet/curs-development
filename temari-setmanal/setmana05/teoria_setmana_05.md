# Setmana 5 - Teoria: Persistència, JPA i SQL Real

## 1. Introducció: De la RAM a la Base de Dades

Fins ara, GamePulse guarda els jocs en un `ConcurrentHashMap` dins de `InMemoryGameRepository`:

```
Aplicació s'inicia → HashMap buit
Afegeixes 50 jocs → HashMap amb 50 entries
Aplicació es para → TOT DESAPAREIX
```

A producció, les dades han de sobreviure reinicis, desplegaments, i crashes. Necessitem una **base de dades**.

```
Sense BD (S2-S4):                    Amb BD (S5+):
┌───────────────┐                    ┌───────────────┐
│  Java App     │                    │  Java App     │
│  ┌─────────┐  │                    │  ┌─────────┐  │
│  │ HashMap │  │                    │  │   JPA   │──│──→ Base de Dades (H2)
│  │ (RAM)   │  │                    │  └─────────┘  │     ┌──────────────┐
│  └─────────┘  │                    └───────────────┘     │ Taula: games │
└───────────────┘                                          │  appId (PK)  │
   Dades en RAM                                            │  title       │
   → es perden                                             │  price       │
                                                           │  players     │
                                                           └──────────────┘
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
INSERT INTO games (app_id, title, price, active_player_count)
VALUES ('APP-1', 'League of Legends', 0.00, 5000000);

-- READ: Consultar dades
SELECT * FROM games WHERE app_id = 'APP-1';
SELECT title, price FROM games WHERE price > 20.00 ORDER BY price;

-- UPDATE: Modificar dades
UPDATE games SET price = 19.99 WHERE app_id = 'APP-42';

-- DELETE: Eliminar dades
DELETE FROM games WHERE app_id = 'APP-42';
```

### Crear la Taula

```sql
CREATE TABLE games (
    app_id VARCHAR(50) PRIMARY KEY,    -- Clau única, no es repeteix
    title VARCHAR(255) NOT NULL,        -- Obligatori
    price DECIMAL(10, 2) NOT NULL,      -- 2 decimals
    active_player_count BIGINT DEFAULT 0
);
```

**PRIMARY KEY:** Com l'`appId` del HashMap — identifica cada fila de forma única. La BD crea automàticament un **index** per buscar ràpidament per `app_id` (equivalent al hash del HashMap).

### Filtrar i Ordenar

```sql
-- Jocs gratuïts ordenats per popularitat
SELECT title, active_player_count 
FROM games 
WHERE price = 0.00 
ORDER BY active_player_count DESC;

-- Els 10 jocs més populars
SELECT title, active_player_count 
FROM games 
ORDER BY active_player_count DESC 
LIMIT 10;

-- Jocs que contenen "Legend" al títol
SELECT * FROM games 
WHERE title LIKE '%Legend%';
```

### Funcions d'Agregació

```sql
-- Quants jocs tenim?
SELECT COUNT(*) FROM games;

-- Preu mitjà dels jocs de pagament
SELECT AVG(price) FROM games WHERE price > 0;

-- Total de jugadors actius per rang de preu
SELECT 
    CASE 
        WHEN price = 0 THEN 'Free'
        WHEN price < 20 THEN 'Budget'
        ELSE 'Premium'
    END AS tier,
    COUNT(*) AS game_count,
    SUM(active_player_count) AS total_players
FROM games
GROUP BY tier;
```

### JOINs: Relacionar Taules

Quan la BD creixi (S17 amb PostgreSQL), tindrem múltiples taules:

```sql
-- Taules
CREATE TABLE games (
    app_id VARCHAR(50) PRIMARY KEY,
    title VARCHAR(255) NOT NULL
);

CREATE TABLE patches (
    id SERIAL PRIMARY KEY,
    game_app_id VARCHAR(50) REFERENCES games(app_id),  -- Foreign key
    patch_version VARCHAR(20),
    release_date DATE,
    description TEXT
);

-- JOIN: Obtenir jocs amb els seus patches
SELECT g.title, p.patch_version, p.release_date
FROM games g
JOIN patches p ON g.app_id = p.game_app_id
WHERE g.title = 'League of Legends'
ORDER BY p.release_date DESC;
```

```
Resultat:
┌─────────────────────┬───────────────┬────────────┐
│ title               │ patch_version │ release_date│
├─────────────────────┼───────────────┼────────────┤
│ League of Legends   │ 14.5          │ 2024-03-06 │
│ League of Legends   │ 14.4          │ 2024-02-22 │
│ League of Legends   │ 14.3          │ 2024-02-07 │
└─────────────────────┴───────────────┴────────────┘
```

**Per què importa:** A les entrevistes de backend, et demanaran escriure JOINs. JPA els amaga, però has de saber què passa per sota.

---

## 3. Indexes: Per Què les Queries Són Ràpides (o Lentes)

### Sense Index

```sql
SELECT * FROM games WHERE title = 'League of Legends';
```

Sense index, la BD recorre **tota la taula** fila per fila:

```
Taula games (100.000 files):
Fila 1: APP-1, "Dota 2"               ← No
Fila 2: APP-2, "Counter-Strike"        ← No
Fila 3: APP-3, "Valorant"              ← No
...
Fila 42857: APP-42857, "League of Legends"  ← TROBAT! (però ha mirat 42.857 files)
...continua fins al final per si n'hi ha més...
Fila 100000: APP-100000, "Tetris"

→ Full Table Scan: O(n) — exactament el problema de S1 amb ArrayList!
```

### Amb Index

```sql
CREATE INDEX idx_games_title ON games(title);
```

La BD crea una estructura auxiliar (B-Tree) que permet trobar files per `title` en O(log n):

```
Index B-Tree per title:
                    ┌─────────────────┐
                    │  "L" < "League"  │
                    │  → branca dreta  │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ "League of L..."│
                    │ → Fila 42857   │
                    └─────────────────┘

→ Index Scan: O(log n) — 17 passos per a 100.000 files (en lloc de 100.000)
```

### EXPLAIN: Veure Què Fa la BD

```sql
EXPLAIN SELECT * FROM games WHERE title = 'League of Legends';
```

```
Sense index:
Seq Scan on games  (cost=0.00..1850.00 rows=1 width=120)
  Filter: (title = 'League of Legends')
→ "Seq Scan" = recorregut seqüencial = LENT

Amb index:
Index Scan using idx_games_title on games  (cost=0.00..8.27 rows=1 width=120)
  Index Cond: (title = 'League of Legends')
→ "Index Scan" = usa l'index = RÀPID
```

**Connexió amb S1:** Un index de BD és l'equivalent d'un HashMap per a dades a disc. La PRIMARY KEY ja crea un index automàticament — per això `findById` sempre és ràpid.

### Quan Crear Indexes

| Cas | Index? | Per què |
|-----|--------|---------|
| `WHERE app_id = ?` | Ja existeix (PK) | Primary Key és index automàtic |
| `WHERE title = ?` | ✅ Crear | Cerques freqüents per títol |
| `WHERE price > 20` | Depèn | Només si la query és freqüent |
| `ORDER BY active_player_count DESC` | ✅ Crear | Rankings/top lists |
| `WHERE title LIKE '%Legend%'` | ❌ No serveix | LIKE amb % al principi no pot usar index B-Tree |

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
public void transferPlayers(String fromGameId, String toGameId, long count) {
    GameRecord from = repository.findById(fromGameId).orElseThrow();
    GameRecord to = repository.findById(toGameId).orElseThrow();
    
    from.setActivePlayerCount(from.getActivePlayerCount() - count);
    to.setActivePlayerCount(to.getActivePlayerCount() + count);
    
    repository.save(from);
    // Si aquí falla (excepció, crash, timeout)...
    repository.save(to);  
}
```

**Sense @Transactional:** Si falla entre els dos `save()`, `from` ha perdut jugadors però `to` no els ha guanyat. Jugadors desapareguts.

**Amb @Transactional:** Si falla, Spring fa ROLLBACK → ambdós canvis es desfan. Les dades queden com estaven.

```
Sense @Transactional:          Amb @Transactional:
BEGIN                          BEGIN
UPDATE from: -1000 ✅          UPDATE from: -1000 ✅
💥 Error!                      💥 Error!
UPDATE to: +1000 ❌            ROLLBACK → from torna a l'original
→ 1000 jugadors perduts        → Tot queda com estava
```

### Isolament: Concurrent DB Access (Connexió amb S3)

A S3 vam veure el "Lost Update":

```
Thread A: READ game.price = 29.99
Thread B: READ game.price = 29.99
Thread A: WRITE game.price = 19.99  ✅
Thread B: WRITE game.price = 39.99  ✅  ← Sobreescriu el canvi de A!
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
class GameRecord          →    games
@Id String appId          →    app_id VARCHAR PRIMARY KEY
String title              →    title VARCHAR
BigDecimal price          →    price DECIMAL

repository.save(game)     →    INSERT INTO games VALUES (...)
repository.findById(id)   →    SELECT * FROM games WHERE app_id = ?
repository.findAll()      →    SELECT * FROM games
repository.delete(game)   →    DELETE FROM games WHERE app_id = ?
```

### @Entity: Convertir una Classe en Taula

```java
@Entity
@Table(name = "games")
public class GameRecord {
    
    @Id
    private String appId;
    
    @Column(nullable = false)
    private String title;
    
    @Column(precision = 10, scale = 2)
    private BigDecimal price;
    
    @Column(name = "active_player_count")
    private Long activePlayerCount;
    
    @Version
    private Long version;  // Per Optimistic Locking (S3)
    
    // JPA necessita constructor buit
    protected GameRecord() {}
    
    public GameRecord(String appId, String title, BigDecimal price, Long activePlayerCount) {
        this.appId = appId;
        this.title = title;
        this.price = price;
        this.activePlayerCount = activePlayerCount;
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
public interface GameJpaRepository extends JpaRepository<GameRecord, String> {
    
    // Spring genera la implementació SQL automàticament!
    // No has d'escriure cap línia de SQL ni d'implementació.
    
    // findAll()    → SELECT * FROM games
    // findById()   → SELECT * FROM games WHERE app_id = ?
    // save()       → INSERT/UPDATE
    // delete()     → DELETE FROM games WHERE app_id = ?
    
    // Queries derivades del nom del mètode:
    List<GameRecord> findByTitleContaining(String keyword);
    // → SELECT * FROM games WHERE title LIKE '%keyword%'
    
    List<GameRecord> findByActivePlayerCountGreaterThan(Long count);
    // → SELECT * FROM games WHERE active_player_count > ?
    
    List<GameRecord> findByPriceBetween(BigDecimal min, BigDecimal max);
    // → SELECT * FROM games WHERE price BETWEEN ? AND ?
}
```

**Com funciona?** Spring Data llegeix el nom del mètode i genera el SQL:

```
findByTitleContaining
  │   │     │
  │   │     └── LIKE '%...%'
  │   └──────── WHERE title
  └──────────── SELECT * FROM games
```

### @Query: SQL Explícit

Quan el nom del mètode no és suficient:

```java
@Query("SELECT g FROM GameRecord g WHERE g.price > :minPrice ORDER BY g.activePlayerCount DESC")
List<GameRecord> findExpensivePopularGames(@Param("minPrice") BigDecimal minPrice);

// SQL natiu (per queries complexes)
@Query(value = "SELECT * FROM games WHERE title ILIKE %:keyword%", nativeQuery = true)
List<GameRecord> searchIgnoreCase(@Param("keyword") String keyword);
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
spring.datasource.url=jdbc:h2:mem:gamepulse
spring.datasource.driver-class-name=org.h2.Driver
spring.jpa.hibernate.ddl-auto=create-drop
spring.h2.console.enabled=true
```

`ddl-auto=create-drop`: JPA crea les taules automàticament a partir de les `@Entity`. Pràctic per desenvolupament; **mai en producció** (per això a S17 introduirem Flyway per migracions controlades).

---

## 7. El Poder del Pattern Repository: El Swap

El moment clau de S5: reemplaçar `InMemoryGameRepository` per `GameJpaRepository` **sense canviar cap línia de `GameManagementService`**:

```
Setmanes 2-4:                        Setmana 5:

GameManagementService                 GameManagementService
    │                                     │
    ▼                                     ▼
GameRepository (interfície)           GameRepository (interfície)
    │                                     │
    ▼                                     ▼
InMemoryGameRepository               GameJpaRepository
    │                                     │
    ▼                                     ▼
ConcurrentHashMap (RAM)               H2 Database (SQL)
```

El codi de `GameManagementService` és **exactament el mateix**:

```java
@Service
public class GameManagementService {
    private final GameRepository repository;  // No sap si és In-Memory o SQL!
    
    public GameManagementService(GameRepository repository) {
        this.repository = repository;
    }
    
    public void registerGame(String appId, String title, BigDecimal price) {
        GameRecord game = new GameRecord(appId, title, price, 0L);
        repository.save(game);  // Funciona amb HashMap o SQL
    }
    
    public List<GameRecord> getPopularGames() {
        return repository.findAll().stream()
            .filter(g -> g.getActivePlayerCount() > 100_000)
            .toList();
    }
}
```

**Això és SOLID en acció:**
- **D (Dependency Inversion):** Depèn de l'abstracció (`GameRepository`), no de la implementació
- **O (Open/Closed):** Canviem la persistència sense modificar la lògica de negoci
- **L (Liskov):** `GameJpaRepository` és substituïble per `InMemoryGameRepository`

**I tots els tests de S2-S4 segueixen passant** perquè testejaven contra la interfície.

---

## 8. Python: Persistència amb SQLite

### L'Equivalent a H2 en Python

```python
import sqlite3
from dataclasses import dataclass

@dataclass(frozen=True)
class GameRecord:
    app_id: str
    title: str
    price: float
    active_player_count: int

class SqliteGameRepository:
    def __init__(self, db_path: str = ":memory:"):
        self.conn = sqlite3.connect(db_path)
        self.conn.execute("""
            CREATE TABLE IF NOT EXISTS games (
                app_id TEXT PRIMARY KEY,
                title TEXT NOT NULL,
                price REAL NOT NULL,
                active_player_count INTEGER DEFAULT 0
            )
        """)
    
    def save(self, game: GameRecord) -> None:
        self.conn.execute(
            "INSERT OR REPLACE INTO games VALUES (?, ?, ?, ?)",
            (game.app_id, game.title, game.price, game.active_player_count)
        )
        self.conn.commit()
    
    def find_by_id(self, app_id: str) -> GameRecord | None:
        row = self.conn.execute(
            "SELECT * FROM games WHERE app_id = ?", (app_id,)
        ).fetchone()
        if row is None:
            return None
        return GameRecord(*row)
    
    def find_all(self) -> list[GameRecord]:
        rows = self.conn.execute("SELECT * FROM games").fetchall()
        return [GameRecord(*row) for row in rows]
```

**Comparativa:**

| Concepte | Java (JPA + H2) | Python (sqlite3) |
|----------|-----------------|-------------------|
| BD lleugera | H2 | SQLite |
| ORM | JPA (`@Entity`, `JpaRepository`) | Manual (o SQLAlchemy a S8+) |
| Queries | Derivades del nom del mètode | SQL explícit |
| Paràmetres | `@Param("title")` | `?` o `:title` |
| Transaccions | `@Transactional` | `conn.commit()` / `conn.rollback()` |

### Tests amb pytest

```python
import pytest

@pytest.fixture
def repo():
    return SqliteGameRepository(":memory:")

@pytest.fixture
def sample_game():
    return GameRecord("APP-1", "League of Legends", 0.0, 5_000_000)

def test_save_and_find(repo, sample_game):
    repo.save(sample_game)
    found = repo.find_by_id("APP-1")
    assert found == sample_game

def test_find_nonexistent(repo):
    assert repo.find_by_id("NOPE") is None

def test_find_all(repo, sample_game):
    repo.save(sample_game)
    repo.save(GameRecord("APP-2", "Dota 2", 0.0, 2_000_000))
    assert len(repo.find_all()) == 2
```

---

## 9. Exercici SQL a la Consola H2

Un dels exercicis de la setmana és obrir la consola H2 (`http://localhost:8080/h2-console`) i escriure queries SQL a mà. Exemples:

```sql
-- 1. Inserir dades de prova
INSERT INTO games VALUES ('APP-1', 'League of Legends', 0.00, 5000000);
INSERT INTO games VALUES ('APP-2', 'Counter-Strike 2', 0.00, 1200000);
INSERT INTO games VALUES ('APP-3', 'Baldurs Gate 3', 59.99, 800000);
INSERT INTO games VALUES ('APP-4', 'Elden Ring', 49.99, 300000);
INSERT INTO games VALUES ('APP-5', 'Stardew Valley', 14.99, 90000);

-- 2. Consultes bàsiques
SELECT * FROM games ORDER BY active_player_count DESC;
SELECT title, price FROM games WHERE price = 0.00;
SELECT COUNT(*) AS total, AVG(price) AS avg_price FROM games;

-- 3. Verificar que JPA ha creat el que esperem
SHOW TABLES;
SHOW COLUMNS FROM games;

-- 4. Veure el pla d'execució
EXPLAIN SELECT * FROM games WHERE title = 'Elden Ring';
-- Sense index: TABLE SCAN
CREATE INDEX idx_title ON games(title);
EXPLAIN SELECT * FROM games WHERE title = 'Elden Ring';
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
