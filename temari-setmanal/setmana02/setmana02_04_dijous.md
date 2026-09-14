# Setmana 2 — Dijous: Python Dataclasses i Modelatge Bilinguee

## Objectiu del Dia

Aprofundir en Python `@dataclass` com a eina de modelatge i construir un segon model de domini (`PlayerRecord`) en ambdos llenguatges. Veure les diferencies i similituds entre Java records i Python dataclasses quan modeles el mateix domini. Al final del dia tens `PlayerRecord` funcionant en Java i Python, amb repositori i validacio en ambdos.

---

## Teoria

### Python `@dataclass`: Mes que un Record

A dimarts vas veure el basic de `@dataclass(frozen=True)`. Avui aprofundim en les opcions i com Python gestiona la validacio i els valors per defecte.

```python
from dataclasses import dataclass, field
from decimal import Decimal

# @dataclass genera automaticament __init__, __repr__, __eq__, __hash__
# frozen=True fa l'objecte immutable (com un Java record)
# order=True genera __lt__, __le__, __gt__, __ge__ — permet ordenar objectes
@dataclass(frozen=True, order=True)
class PlayerRecord:
    player_id: str                 # Identificador unic del jugador
    username: str                  # Nom d'usuari
    level: int                     # Nivell actual (minim 1)
    hours_played: float            # Hores jugades (minim 0)

    def __post_init__(self):
        """Equivalent al compact constructor de Java.
        S'executa automaticament despres de __init__.
        Amb frozen=True, hem d'usar object.__setattr__ per validar
        perque l'objecte ja esta 'congelat' quan arriba aqui."""

        # Validacio: player_id no pot ser buit
        if not self.player_id or not self.player_id.strip():
            raise ValueError("player_id no pot ser buit")

        # Validacio: level minim 1 (un jugador sempre te almenys nivell 1)
        if self.level < 1:
            raise ValueError(f"level ha de ser >= 1, rebut: {self.level}")

        # Validacio: hores jugades no poden ser negatives
        if self.hours_played < 0:
            raise ValueError(f"hours_played ha de ser >= 0, rebut: {self.hours_played}")

    def is_veteran(self) -> bool:
        """Un jugador es veterà si ha jugat mes de 1000 hores."""
        return self.hours_played > 1000

    def level_up(self) -> "PlayerRecord":
        """Retorna un NOU PlayerRecord amb el nivell incrementat.
        L'original no canvia — frozen=True ho impedeix.
        Aixo es el patro immutable: 'modificar' = crear un objecte nou."""
        # object.__setattr__ no serveix aqui — creem un objecte nou
        return PlayerRecord(
            player_id=self.player_id,
            username=self.username,
            level=self.level + 1,       # Nivell incrementat
            hours_played=self.hours_played
        )
```

### Valors per Defecte i Fields

```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass(frozen=True)
class PlayerRecord:
    player_id: str
    username: str
    level: int = 1                  # Valor per defecte: nivell 1
    hours_played: float = 0.0       # Valor per defecte: 0 hores
    # field(default_factory=...) per a valors mutables (llistes, dicts, dates)
    # No pots posar [] directament — Python compartiria la mateixa llista entre objectes
    created_at: str = field(
        default_factory=lambda: datetime.now().isoformat()
    )

# Creacio amb i sense valors per defecte:
p1 = PlayerRecord("P-1", "Faker")                    # level=1, hours=0.0
p2 = PlayerRecord("P-2", "Caps", level=50, hours_played=3500.0)
```

**Diferencia amb Java:** Java records no tenen valors per defecte als camps — has de passar-los tots o crear un Factory. Python dataclasses si que els permeten, cosa que simplifica la creacio per a casos comuns.

### Comparativa Detallada: Java Record vs Python Dataclass

**Model `PlayerRecord` en Java:**

```java
// Java record: immutable per disseny
// El compilador genera automaticament: constructor, getters, equals(), hashCode(), toString()
public record PlayerRecord(
    String playerId,          // Accedit amb .playerId() — es un metode, no un camp public
    String username,
    int level,
    double hoursPlayed
) {
    // Compact constructor: validacio centralitzada
    // No cal "this.playerId = playerId" — Java ho fa automaticament
    public PlayerRecord {
        if (playerId == null || playerId.isBlank()) {
            throw new IllegalArgumentException("playerId no pot ser buit");
        }
        if (level < 1) {
            throw new IllegalArgumentException("level ha de ser >= 1");
        }
        if (hoursPlayed < 0) {
            throw new IllegalArgumentException("hoursPlayed ha de ser >= 0");
        }
    }

    // Metode de negoci: determina si el jugador es veterà
    public boolean isVeteran() {
        return hoursPlayed > 1000;
    }

    // Patro immutable: retorna un NOU record amb el nivell incrementat
    public PlayerRecord levelUp() {
        return new PlayerRecord(playerId, username, level + 1, hoursPlayed);
    }
}
```

**Model `PlayerRecord` en Python:**

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class PlayerRecord:
    player_id: str        # Accedit amb .player_id — es un atribut, no un metode
    username: str
    level: int
    hours_played: float

    def __post_init__(self):
        """Validacio equivalent al compact constructor Java."""
        if not self.player_id or not self.player_id.strip():
            raise ValueError("player_id no pot ser buit")
        if self.level < 1:
            raise ValueError("level ha de ser >= 1")
        if self.hours_played < 0:
            raise ValueError("hours_played ha de ser >= 0")

    def is_veteran(self) -> bool:
        return self.hours_played > 1000

    def level_up(self) -> "PlayerRecord":
        return PlayerRecord(
            player_id=self.player_id,
            username=self.username,
            level=self.level + 1,
            hours_played=self.hours_played
        )
```

### Taula de Diferencies Clau

| Aspecte | Java 21 Record | Python @dataclass(frozen) |
|---------|----------------|---------------------------|
| Declaracio | `public record X(camps) {}` | `@dataclass(frozen=True) class X:` |
| Accedir camps | `x.playerId()` (metode) | `x.player_id` (atribut) |
| Validacio | Compact constructor `public X {}` | `__post_init__(self)` |
| Valors per defecte | No als camps (usa Factory) | Si: `level: int = 1` |
| Herencia | Records no poden heretar | Dataclasses si (pero amb frozen es complicat) |
| Noms | `camelCase` | `snake_case` |
| Null handling | `Optional<T>` | `Optional[T]` o `None` |
| Serialitzacio | Necessita Jackson/Gson | `dataclasses.asdict()` nativa |

### Repository Bilinguee: Mateixa Estructura, Dos Llenguatges

**Java:**
```java
// Interficie generica — funciona per a qualsevol tipus de domini
// T es un parametre de tipus: quan creem ChampionRepository, T = ChampionRecord
public interface Repository<T> {
    void save(T entity);
    Optional<T> findById(String id);
    List<T> findAll();
    void delete(String id);
}

// Implementacio especifica per a PlayerRecord
public class InMemoryPlayerRepository implements Repository<PlayerRecord> {
    private final Map<String, PlayerRecord> storage = new ConcurrentHashMap<>();

    @Override
    public void save(PlayerRecord player) {
        storage.put(player.playerId(), player);
    }

    @Override
    public Optional<PlayerRecord> findById(String playerId) {
        return Optional.ofNullable(storage.get(playerId));
    }

    @Override
    public List<PlayerRecord> findAll() {
        return Collections.unmodifiableList(new ArrayList<>(storage.values()));
    }

    @Override
    public void delete(String playerId) {
        storage.remove(playerId);
    }
}
```

**Python:**
```python
from abc import ABC, abstractmethod
from typing import Optional

# Interficie generica (abc.ABC) — mateix patro que Java
class PlayerRepository(ABC):

    @abstractmethod
    def save(self, player: PlayerRecord) -> None:
        pass

    @abstractmethod
    def find_by_id(self, player_id: str) -> Optional[PlayerRecord]:
        pass

    @abstractmethod
    def find_all(self) -> list[PlayerRecord]:
        pass

    @abstractmethod
    def delete(self, player_id: str) -> None:
        pass


class InMemoryPlayerRepository(PlayerRepository):

    def __init__(self):
        self._storage: dict[str, PlayerRecord] = {}

    def save(self, player: PlayerRecord) -> None:
        self._storage[player.player_id] = player

    def find_by_id(self, player_id: str) -> Optional[PlayerRecord]:
        return self._storage.get(player_id)

    def find_all(self) -> list[PlayerRecord]:
        return list(self._storage.values())

    def delete(self, player_id: str) -> None:
        self._storage.pop(player_id, None)

    # Metode extra: cerca per nivell minim
    def find_by_level_greater_than(self, min_level: int) -> list[PlayerRecord]:
        """Retorna jugadors amb nivell superior al minim indicat.
        Usa list comprehension — l'equivalent Pythonic de streams Java."""
        return [p for p in self._storage.values() if p.level > min_level]
```

---

## Activitat

### 1. Crear `PlayerRecord` en Java (20 min)

Crea el fitxer:
```
backend-java/src/main/java/com/esportspulse/engine/model/PlayerRecord.java
```

Implementa:
- Camps: `playerId` (String), `username` (String), `level` (int), `hoursPlayed` (double)
- Compact constructor amb validacio (playerId no null/buit, level >= 1, hoursPlayed >= 0)
- `isVeteran()` — retorna true si `hoursPlayed > 1000`
- `levelUp()` — retorna un NOU record amb `level + 1`

### 2. Crear `PlayerRecord` en Python (20 min)

Crea el fitxer:
```
ai-python/src/player_record.py
```

Implementa l'equivalent amb `@dataclass(frozen=True)`:
- Camps: `player_id`, `username`, `level`, `hours_played`
- `__post_init__` amb les mateixes validacions
- `is_veteran()` i `level_up()` amb el mateix comportament

### 3. Crear Repository per a `PlayerRecord` en ambdos (30 min)

**Java:**
```
backend-java/src/main/java/com/esportspulse/engine/repository/PlayerRepository.java
backend-java/src/main/java/com/esportspulse/engine/repository/InMemoryPlayerRepository.java
```
- Afegeix `findByLevelGreaterThan(int level)` a la interficie

**Python:**
```
ai-python/src/player_repository.py
```
- Mateixa estructura, amb `find_by_level_greater_than(min_level)`

### 4. Connexio S1: Benchmark de Cerca (15 min)

Crea un `HashMap<String, PlayerRecord>` amb 50.000 jugadors i mesura el temps de cerca. Ha de ser O(1) — exactament el que vas aprendre la Setmana 1.

```java
public class PlayerBenchmark {
    public static void main(String[] args) {
        // Genera 50.000 jugadors
        Map<String, PlayerRecord> map = new HashMap<>();
        for (int i = 0; i < 50_000; i++) {
            String id = "P-" + i;
            PlayerRecord p = new PlayerRecord(id, "Player" + i, 1, 0.0);
            map.put(id, p);
        }

        // Mesura temps de cerca per clau — ha de ser quasi instantani (O(1))
        long start = System.nanoTime();
        for (int i = 0; i < 10_000; i++) {
            map.get("P-25000");  // Busca sempre el del mig
        }
        long elapsed = System.nanoTime() - start;

        // 10.000 cerques en menys de 1ms — O(1) confirmat
        System.out.printf("10.000 cerques: %d ns (%.2f ms)%n",
            elapsed, elapsed / 1_000_000.0);
    }
}
```

### 5. Verificacio creuada (15 min)

Crea un petit script que demostri que Java i Python fan exactament el MATEIX:

**Python:**
```python
# ai-python/src/demo_bilingual.py
from player_record import PlayerRecord
from player_repository import InMemoryPlayerRepository

# Crea jugadors amb les mateixes dades que en Java
repo = InMemoryPlayerRepository()
faker = PlayerRecord("P-1", "Faker", 50, 3500.0)
caps = PlayerRecord("P-2", "Caps", 30, 800.0)
rookie = PlayerRecord("P-3", "Rookie", 5, 50.0)

repo.save(faker)
repo.save(caps)
repo.save(rookie)

# Verifica is_veteran
print(f"Faker veterà: {faker.is_veteran()}")    # True (3500 > 1000)
print(f"Caps veterà: {caps.is_veteran()}")      # False (800 < 1000)

# Verifica level_up immutable
faker_up = faker.level_up()
print(f"Faker original: level {faker.level}")    # 50 — no ha canviat
print(f"Faker level_up: level {faker_up.level}") # 51 — objecte NOU

# Cerca per nivell
pros = repo.find_by_level_greater_than(20)
print(f"Jugadors nivell > 20: {len(pros)}")     # 2 (Faker i Caps)

# Verifica immutabilitat
try:
    faker.level = 100  # HA de petar
except Exception as e:
    print(f"Immutabilitat OK: {type(e).__name__}")
```

### 6. Escriure `.cursorrules` (30 min)

Ara que tens dues entitats (`PlayerRecord`, `ChampionRecord`), un repositori, un servei i codi en dos llenguatges, tens prou context per escriure regles útils per a l'assistent IA.

Un `.cursorrules` no és un fitxer de configuració genèric — és una **especificació de comportament** per a l'LLM. Li dius exactament com vols que generi codi al teu projecte. Si les regles són vagues ("usa bons noms"), l'agent farà el que vulgui. Si són precises, el codi surt coherent.

Crea el fitxer `.cursorrules` a l'arrel del projecte:

```
# EsportsPulse Engine — Especificació per a l'Assistent

## Llenguatge i Convencions
- Java 21: variables en camelCase (championRecord, pickRate)
- Python 3.12: variables en snake_case (champion_record, pick_rate)
- Classes en PascalCase en ambdós llenguatges

## Models de Domini
- Java: SEMPRE usar `record`. Mai generar classes amb setters.
- Python: SEMPRE usar `@dataclass(frozen=True)`. Mai atributs mutables.
- Cada record/dataclass ha de tenir compact constructor/`__post_init__` amb validació.

## Arquitectura
- Patrons: Repository (persistència), Factory (creació), Service (lògica)
- Dependències: injectar per constructor. Mai crear dependències amb `new` dins un servei.
- Interfícies: capa de dades sempre darrere d'una interfície.
- Packages Java: model/, repository/, factory/, service/
- Mòduls Python: model/, repository/, factory/, service/

## Testing
- Cada classe pública ha de tenir un test JUnit 5 / pytest corresponent.
- Noms de test: `metode_comportament_condicio` (ex: `findById_returnsEmpty_whenNotFound`)

## Git
- Format: Conventional Commits (feat/fix/test/docs/refactor)
- Branques: `feature/weekN-description`
```

**Verificació:** Demana a Cursor: "Genera un `MatchRecord` seguint les convencions del projecte". L'assistent ha de generar un `record` (no una classe amb setters), amb compact constructor i validació. Si no ho fa, ajusta les regles fins que ho faci.

### 7. Commit (5 min)

```bash
git add backend-java/src/main/java/com/esportspulse/engine/model/PlayerRecord.java
git add backend-java/src/main/java/com/esportspulse/engine/repository/PlayerRepository.java
git add backend-java/src/main/java/com/esportspulse/engine/repository/InMemoryPlayerRepository.java
git add ai-python/src/player_record.py
git add ai-python/src/player_repository.py
git add .cursorrules
git commit -m "feat: bilingual PlayerRecord + repository + .cursorrules project spec"
```

---

## Checklist de Lliurament

- [ ] Java `PlayerRecord` amb compact constructor, `isVeteran()`, `levelUp()`
- [ ] Python `PlayerRecord` amb `@dataclass(frozen=True)`, `__post_init__`, `is_veteran()`, `level_up()`
- [ ] Java `InMemoryPlayerRepository` amb `findByLevelGreaterThan`
- [ ] Python `InMemoryPlayerRepository` amb `find_by_level_greater_than`
- [ ] Benchmark: 50.000 jugadors en HashMap, cerca O(1) verificada
- [ ] Demo bilinguee: Java i Python produeixen els mateixos resultats
- [ ] Immutabilitat verificada en ambdos llenguatges (setter peta)
- [ ] `.cursorrules` escrit amb regles precises; Cursor genera `MatchRecord` correctament
- [ ] Commit amb format Conventional Commits
