# Setmana 2 — Dimarts: Interficies, Repository Pattern i Python Equivalents

## Objectiu del Dia

Crear la capa de persistencia del projecte EsportsPulse usant el patro Repository amb interficies Java. Implementar una versio in-memory que mes endavant (Setmana 5) es podra canviar per SQL sense tocar el codi de negoci. Veure com Python fa el mateix amb `abc.ABC` i `@dataclass`. Al final del dia tens `GameRepository`, `InMemoryGameRepository`, i l'equivalent Python funcionant.

---

## Teoria

### Interficies: El Contracte que Desacobla

Una interficie en Java defineix QUE ha de fer una classe, pero no COM. Es un contracte: qualsevol classe que la implementi promet oferir certs metodes.

```java
// Interficie: defineix el contracte per a qualsevol repositori de jocs
// No te codi — nomes la signatura dels metodes que han d'existir
public interface GameRepository {

    // Guarda un joc (o l'actualitza si ja existeix)
    void save(GameRecord game);

    // Busca un joc per ID — retorna Optional per evitar nulls
    // Optional es una caixa que pot contenir un valor o estar buida
    Optional<GameRecord> findById(String appId);

    // Retorna tots els jocs com a llista immutable
    // Collections.unmodifiableList evita que qui rebi la llista la modifiqui
    List<GameRecord> findAll();

    // Elimina un joc per ID
    void delete(String appId);
}
```

**Per que Optional i no null?**

```java
// MAL — retorna null si no troba el joc
public GameRecord findById(String appId) {
    return storage.get(appId);  // Pot ser null!
}

// Qui crida el metode oblida comprovar null → NullPointerException
GameRecord g = repo.findById("APP-999");
System.out.println(g.title());  // CRASH si no existeix

// BE — retorna Optional, que OBLIGA a gestionar l'absencia
public Optional<GameRecord> findById(String appId) {
    return Optional.ofNullable(storage.get(appId));
}

// Qui crida el metode HA de decidir que fer si no existeix
Optional<GameRecord> result = repo.findById("APP-999");

// Opcio 1: valor per defecte
GameRecord g = result.orElse(defaultGame);

// Opcio 2: excepcio controlada
GameRecord g = result.orElseThrow(
    () -> new GameNotFoundException("APP-999 no existeix")
);

// Opcio 3: actuar nomes si existeix
result.ifPresent(game -> System.out.println(game.title()));
```

### Repository Pattern: Separar Dades de Negoci

El patro Repository es una abstraccio que amaga on i com es guarden les dades. El codi de negoci (`GameManagementService`) treballa amb la interficie `GameRepository` — no sap si les dades estan en memoria, en una base de dades SQL, o en un fitxer JSON.

```
                        GameManagementService
                               |
                     depèn de (interficie)
                               |
                        GameRepository       ← CONTRACTE
                        /           \
            InMemoryGameRepo    SqlGameRepo  ← IMPLEMENTACIONS
            (Setmana 2)         (Setmana 5)
```

Aixo es DIP (Dependency Inversion) i OCP (Open/Closed) en accio:
- El servei depèn d'una abstraccio, no d'una concrecio (DIP)
- Per afegir SQL, crees una nova classe, no modifiques les existents (OCP)

### Implementacio In-Memory amb ConcurrentHashMap

```java
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

// Implementa la interficie GameRepository guardant tot en memoria
// ConcurrentHashMap es thread-safe: multiples threads poden llegir/escriure sense problemes
public class InMemoryGameRepository implements GameRepository {

    // ConcurrentHashMap: clau = appId, valor = GameRecord
    // Es thread-safe perque permet lectures concurrents i escriptures atomiques
    // Un HashMap normal petaria si dos threads escriuen al mateix temps
    private final Map<String, GameRecord> storage = new ConcurrentHashMap<>();

    @Override
    public void save(GameRecord game) {
        // put() afegeix o substitueix — si l'appId ja existeix, actualitza
        storage.put(game.appId(), game);
    }

    @Override
    public Optional<GameRecord> findById(String appId) {
        // ofNullable: si get() retorna null, Optional estara buit
        // Si retorna un valor, Optional el contindra
        return Optional.ofNullable(storage.get(appId));
    }

    @Override
    public List<GameRecord> findAll() {
        // Retorna una COPIA immutable de tots els valors
        // Si qui rebi la llista intenta afegir-hi elements, petara (UnsupportedOperationException)
        // Aixo protegeix l'estat intern del repositori
        return Collections.unmodifiableList(
            new ArrayList<>(storage.values())
        );
    }

    @Override
    public void delete(String appId) {
        // remove() elimina l'entrada amb aquesta clau
        // Si no existeix, no fa res (no peta)
        storage.remove(appId);
    }
}
```

**Per que `ConcurrentHashMap` i no `HashMap`?**
- `HashMap` no es thread-safe: si dos threads escriuen al mateix temps, les dades es corrompen
- `ConcurrentHashMap` permet lectures concurrents sense bloqueig i escriptures segures
- A EsportsPulse, quan tinguem una API web (Setmana 5+), multiples requests accediran al repositori alhora

### Python Equivalent: `abc.ABC` i `@dataclass(frozen=True)`

Python no te `record` ni `interface` com a paraula clau, pero te equivalents funcionals.

**`@dataclass(frozen=True)` = Java Record:**

```python
from dataclasses import dataclass
from decimal import Decimal

# frozen=True fa que l'objecte sigui immutable — no pots canviar atributs despres de crear-lo
# Es l'equivalent de Java record: camps finals, sense setters, __eq__ i __hash__ automatics
@dataclass(frozen=True)
class GameRecord:
    app_id: str                    # Identificador unic del joc
    title: str                     # Nom del joc
    price: Decimal                 # Preu en euros
    active_player_count: int       # Jugadors actius

    def is_popular(self) -> bool:
        """Un joc es popular si te mes de 100.000 jugadors actius."""
        return self.active_player_count > 100_000

    def discounted_price(self, percentage: float) -> "GameRecord":
        """Retorna un NOU GameRecord amb el preu reduit.
        L'original no canvia — frozen=True ho impedeix."""
        new_price = self.price * Decimal(str(1 - percentage))
        # Creem un objecte nou perque l'original es immutable
        return GameRecord(
            app_id=self.app_id,
            title=self.title,
            price=new_price,
            active_player_count=self.active_player_count
        )

# Prova d'immutabilitat:
lol = GameRecord("APP-1", "LoL", Decimal("0"), 5_000_000)
# lol.price = Decimal("100")  # ERROR! FrozenInstanceError — no es pot mutar
```

**`abc.ABC` = Java Interface:**

```python
from abc import ABC, abstractmethod
from typing import Optional

# ABC = Abstract Base Class — equivalent a una interficie Java
# No es pot instanciar directament, nomes serveix com a contracte
class GameRepository(ABC):

    @abstractmethod  # Obliga les subclasses a implementar aquest metode
    def save(self, game: GameRecord) -> None:
        """Guarda un joc al repositori."""
        pass

    @abstractmethod
    def find_by_id(self, app_id: str) -> Optional[GameRecord]:
        """Busca un joc per ID. Retorna None si no existeix."""
        pass

    @abstractmethod
    def find_all(self) -> list[GameRecord]:
        """Retorna tots els jocs."""
        pass

    @abstractmethod
    def delete(self, app_id: str) -> None:
        """Elimina un joc per ID."""
        pass


# Implementacio concreta — equivalent a InMemoryGameRepository en Java
class InMemoryGameRepository(GameRepository):

    def __init__(self):
        # dict Python es similar a HashMap Java
        self._storage: dict[str, GameRecord] = {}

    def save(self, game: GameRecord) -> None:
        self._storage[game.app_id] = game

    def find_by_id(self, app_id: str) -> Optional[GameRecord]:
        # dict.get() retorna None si la clau no existeix (equivalent a Optional.empty())
        return self._storage.get(app_id)

    def find_all(self) -> list[GameRecord]:
        # Retorna una copia de la llista per protegir l'estat intern
        return list(self._storage.values())

    def delete(self, app_id: str) -> None:
        # pop amb default None: elimina si existeix, no peta si no
        self._storage.pop(app_id, None)
```

### Taula Comparativa Java vs Python

| Concepte | Java 21 | Python 3.12 |
|----------|---------|-------------|
| Model immutable | `record GameRecord(...)` | `@dataclass(frozen=True)` |
| Interficie | `interface GameRepository` | `class GameRepository(ABC)` |
| Metode abstracte | Implicit a interficie | `@abstractmethod` |
| Null-safe | `Optional<T>` | `Optional[T]` (typing) o `None` |
| Map thread-safe | `ConcurrentHashMap` | `dict` + `threading.Lock` |
| Llista immutable | `Collections.unmodifiableList()` | `tuple()` o `frozenset()` |

---

## Activitat

### 1. Crear la interficie `GameRepository` (15 min)

Crea el fitxer:
```
backend-java/src/main/java/com/esportspulse/engine/repository/GameRepository.java
```

Defineix els 4 metodes: `save`, `findById`, `findAll`, `delete`. Usa `Optional<GameRecord>` per a `findById`.

### 2. Implementar `InMemoryGameRepository` (30 min)

Crea el fitxer:
```
backend-java/src/main/java/com/esportspulse/engine/repository/InMemoryGameRepository.java
```

Implementa tots 4 metodes usant `ConcurrentHashMap`. `findAll()` ha de retornar una copia immutable.

### 3. Verificar a `main` (15 min)

```java
public class RepositoryDemo {
    public static void main(String[] args) {
        // Creem el repositori — notem que el tipus declarat es la INTERFICIE
        // Aixo es DIP: el codi depèn de GameRepository, no de InMemoryGameRepository
        GameRepository repo = new InMemoryGameRepository();

        // Guardem dos jocs
        GameRecord lol = new GameRecord("APP-1", "LoL", BigDecimal.ZERO, 5_000_000L);
        GameRecord dota = new GameRecord("APP-2", "Dota 2", BigDecimal.ZERO, 800_000L);
        repo.save(lol);
        repo.save(dota);

        // Busquem per ID — Optional ens obliga a gestionar l'absencia
        Optional<GameRecord> found = repo.findById("APP-1");
        found.ifPresent(g -> System.out.println("Trobat: " + g.title()));  // "Trobat: LoL"

        // Busquem un ID que no existeix
        Optional<GameRecord> notFound = repo.findById("APP-999");
        System.out.println("Existeix? " + notFound.isPresent());  // false

        // Llistem tots els jocs
        List<GameRecord> all = repo.findAll();
        System.out.println("Total jocs: " + all.size());  // 2

        // Verifiquem que la llista es immutable — aixo HA de petar
        try {
            all.add(new GameRecord("APP-3", "Hack", BigDecimal.ZERO, 0L));
        } catch (UnsupportedOperationException e) {
            System.out.println("Llista immutable OK — no es pot modificar des de fora");
        }
    }
}
```

### 4. Mirror Python (30 min)

Crea a `ai-python/src/`:
- `game_record.py` — `@dataclass(frozen=True)` amb `is_popular()` i `discounted_price()`
- `game_repository.py` — `GameRepository(ABC)` i `InMemoryGameRepository`

Prova amb un script:

```python
# ai-python/src/demo_repository.py
from decimal import Decimal
from game_record import GameRecord
from game_repository import InMemoryGameRepository

# Creem el repositori i uns quants jocs
repo = InMemoryGameRepository()
lol = GameRecord("APP-1", "LoL", Decimal("0"), 5_000_000)
dota = GameRecord("APP-2", "Dota 2", Decimal("0"), 800_000)

repo.save(lol)
repo.save(dota)

# Busquem per ID
found = repo.find_by_id("APP-1")
print(f"Trobat: {found.title}" if found else "No trobat")  # "Trobat: LoL"

# Verifiquem immutabilitat
try:
    lol.price = Decimal("100")  # HA de petar — frozen=True
except Exception as e:
    print(f"Immutabilitat OK: {e}")
```

### 5. Commit (5 min)

```bash
git add backend-java/src/main/java/com/esportspulse/engine/repository/
git add ai-python/src/
git commit -m "feat: GameRepository interface + InMemory impl (Java + Python mirror)"
```

---

## Checklist de Lliurament

- [ ] `GameRepository` interficie amb `save`, `findById`, `findAll`, `delete`
- [ ] `InMemoryGameRepository` implementa tots 4 metodes amb `ConcurrentHashMap`
- [ ] `findById` retorna `Optional<GameRecord>` — mai null
- [ ] `findAll` retorna una llista immutable (modificar-la peta)
- [ ] Python: `GameRecord` amb `@dataclass(frozen=True)` funcional
- [ ] Python: `GameRepository(ABC)` amb `InMemoryGameRepository` funcional
- [ ] Demo Java i Python executats sense errors
- [ ] Commit amb format Conventional Commits
