# Setmana 2 — Dimarts: Interficies, Repository Pattern i Python Equivalents

## Objectiu del Dia

Crear la capa de persistencia del projecte EsportsPulse usant el patro Repository amb interficies Java. Implementar una versio in-memory que mes endavant (Setmana 5) es podra canviar per SQL sense tocar el codi de negoci. Veure com Python fa el mateix amb `abc.ABC` i `@dataclass`. Al final del dia tens `ChampionRepository`, `InMemoryChampionRepository`, i l'equivalent Python funcionant.

---

## Teoria

### Interficies: El Contracte que Desacobla

Una interficie en Java defineix QUE ha de fer una classe, pero no COM. Es un contracte: qualsevol classe que la implementi promet oferir certs metodes.

```java
// Interficie: defineix el contracte per a qualsevol repositori de campions
// No te codi — nomes la signatura dels metodes que han d'existir
public interface ChampionRepository {

    // Guarda un campió (o l'actualitza si ja existeix)
    void save(ChampionRecord champion);

    // Busca un campió per ID — retorna Optional per evitar nulls
    // Optional es una caixa que pot contenir un valor o estar buida
    Optional<ChampionRecord> findById(String championId);

    // Retorna tots els campions com a llista immutable
    // Collections.unmodifiableList evita que qui rebi la llista la modifiqui
    List<ChampionRecord> findAll();

    // Elimina un campió per ID
    void delete(String championId);
}
```

**Per que Optional i no null?**

```java
// MAL — retorna null si no trova el campió
public ChampionRecord findById(String championId) {
    return storage.get(championId);  // Pot ser null!
}

// Qui crida el metode oblida comprovar null → NullPointerException
ChampionRecord g = repo.findById("CHAMP-999");
System.out.println(g.name());  // CRASH si no existeix

// BE — retorna Optional, que OBLIGA a gestionar l'absencia
public Optional<ChampionRecord> findById(String championId) {
    return Optional.ofNullable(storage.get(championId));
}

// Qui crida el metode HA de decidir que fer si no existeix
Optional<ChampionRecord> result = repo.findById("CHAMP-999");

// Opcio 1: valor per defecte
ChampionRecord g = result.orElse(defaultChampion);

// Opcio 2: excepcio controlada
ChampionRecord g = result.orElseThrow(
    () -> new ChampionNotFoundException("CHAMP-999 no existeix")
);

// Opcio 3: actuar nomes si existeix
result.ifPresent(champion -> System.out.println(champion.name()));
```

### Repository Pattern: Separar Dades de Negoci

El patro Repository es una abstraccio que amaga on i com es guarden les dades. El codi de negoci (`ChampionManagementService`) treballa amb la interficie `ChampionRepository` — no sap si les dades estan en memoria, en una base de dades SQL, o en un fitxer JSON.

```
                     ChampionManagementService
                               |
                     depèn de (interficie)
                               |
                     ChampionRepository       ← CONTRACTE
                        /           \
         InMemoryChampionRepo  SqlChampionRepo  ← IMPLEMENTACIONS
            (Setmana 2)         (Setmana 5)
```

Aixo es DIP (Dependency Inversion) i OCP (Open/Closed) en accio:
- El servei depèn d'una abstraccio, no d'una concrecio (DIP)
- Per afegir SQL, crees una nova classe, no modifiques les existents (OCP)

### Implementacio In-Memory amb ConcurrentHashMap

```java
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

// Implementa la interficie ChampionRepository guardant tot en memoria
// ConcurrentHashMap es thread-safe: multiples threads poden llegir/escriure sense problemes
public class InMemoryChampionRepository implements ChampionRepository {

    // ConcurrentHashMap: clau = championId, valor = ChampionRecord
    // Es thread-safe perque permet lectures concurrents i escriptures atomiques
    // Un HashMap normal petaria si dos threads escriuen al mateix temps
    private final Map<String, ChampionRecord> storage = new ConcurrentHashMap<>();

    @Override
    public void save(ChampionRecord champion) {
        // put() afegeix o substitueix — si el championId ja existeix, actualitza
        storage.put(champion.championId(), champion);
    }

    @Override
    public Optional<ChampionRecord> findById(String championId) {
        // ofNullable: si get() retorna null, Optional estara buit
        // Si retorna un valor, Optional el contindra
        return Optional.ofNullable(storage.get(championId));
    }

    @Override
    public List<ChampionRecord> findAll() {
        // Retorna una COPIA immutable de tots els valors
        // Si qui rebi la llista intenta afegir-hi elements, petara (UnsupportedOperationException)
        // Aixo protegeix l'estat intern del repositori
        return Collections.unmodifiableList(
            new ArrayList<>(storage.values())
        );
    }

    @Override
    public void delete(String championId) {
        // remove() elimina l'entrada amb aquesta clau
        // Si no existeix, no fa res (no peta)
        storage.remove(championId);
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

# frozen=True fa que l'objecte sigui immutable — no pots canviar atributs despres de crear-lo
# Es l'equivalent de Java record: camps finals, sense setters, __eq__ i __hash__ automatics
@dataclass(frozen=True)
class ChampionRecord:
    champion_id: str               # Identificador unic del campió
    name: str                      # Nom del campió
    win_rate: float                # Percentatge de victories (0-100)
    pick_rate: float               # Percentatge de seleccio (0-100)

    def is_meta(self) -> bool:
        """Un campió es meta si te win rate > 52% i pick rate > 10%."""
        return self.win_rate > 52.0 and self.pick_rate > 10.0

    def with_patch_adjustment(self, modifier: float) -> "ChampionRecord":
        """Retorna un NOU ChampionRecord amb el win_rate ajustat.
        L'original no canvia — frozen=True ho impedeix."""
        new_win_rate = self.win_rate * modifier
        # Creem un objecte nou perque l'original es immutable
        return ChampionRecord(
            champion_id=self.champion_id,
            name=self.name,
            win_rate=new_win_rate,
            pick_rate=self.pick_rate
        )

# Prova d'immutabilitat:
ahri = ChampionRecord("CHAMP-1", "Ahri", 53.2, 12.5)
# ahri.win_rate = 60.0  # ERROR! FrozenInstanceError — no es pot mutar
```

**`abc.ABC` = Java Interface:**

```python
from abc import ABC, abstractmethod
from typing import Optional

# ABC = Abstract Base Class — equivalent a una interficie Java
# No es pot instanciar directament, nomes serveix com a contracte
class ChampionRepository(ABC):

    @abstractmethod  # Obliga les subclasses a implementar aquest metode
    def save(self, champion: ChampionRecord) -> None:
        """Guarda un campió al repositori."""
        pass

    @abstractmethod
    def find_by_id(self, champion_id: str) -> Optional[ChampionRecord]:
        """Busca un campió per ID. Retorna None si no existeix."""
        pass

    @abstractmethod
    def find_all(self) -> list[ChampionRecord]:
        """Retorna tots els campions."""
        pass

    @abstractmethod
    def delete(self, champion_id: str) -> None:
        """Elimina un campió per ID."""
        pass


# Implementacio concreta — equivalent a InMemoryChampionRepository en Java
class InMemoryChampionRepository(ChampionRepository):

    def __init__(self):
        # dict Python es similar a HashMap Java
        self._storage: dict[str, ChampionRecord] = {}

    def save(self, champion: ChampionRecord) -> None:
        self._storage[champion.champion_id] = champion

    def find_by_id(self, champion_id: str) -> Optional[ChampionRecord]:
        # dict.get() retorna None si la clau no existeix (equivalent a Optional.empty())
        return self._storage.get(champion_id)

    def find_all(self) -> list[ChampionRecord]:
        # Retorna una copia de la llista per protegir l'estat intern
        return list(self._storage.values())

    def delete(self, champion_id: str) -> None:
        # pop amb default None: elimina si existeix, no peta si no
        self._storage.pop(champion_id, None)
```

### Taula Comparativa Java vs Python

| Concepte | Java 21 | Python 3.12 |
|----------|---------|-------------|
| Model immutable | `record ChampionRecord(...)` | `@dataclass(frozen=True)` |
| Interficie | `interface ChampionRepository` | `class ChampionRepository(ABC)` |
| Metode abstracte | Implicit a interficie | `@abstractmethod` |
| Null-safe | `Optional<T>` | `Optional[T]` (typing) o `None` |
| Map thread-safe | `ConcurrentHashMap` | `dict` + `threading.Lock` |
| Llista immutable | `Collections.unmodifiableList()` | `tuple()` o `frozenset()` |

---

## Activitat

### 1. Crear la interficie `ChampionRepository` (15 min)

Crea el fitxer:
```
backend-java/src/main/java/com/esportspulse/engine/repository/ChampionRepository.java
```

Defineix els 4 metodes: `save`, `findById`, `findAll`, `delete`. Usa `Optional<ChampionRecord>` per a `findById`.

### 2. Implementar `InMemoryChampionRepository` (30 min)

Crea el fitxer:
```
backend-java/src/main/java/com/esportspulse/engine/repository/InMemoryChampionRepository.java
```

Implementa tots 4 metodes usant `ConcurrentHashMap`. `findAll()` ha de retornar una copia immutable.

### 3. Verificar a `main` (15 min)

```java
public class RepositoryDemo {
    public static void main(String[] args) {
        // Creem el repositori — notem que el tipus declarat es la INTERFICIE
        // Aixo es DIP: el codi depèn de ChampionRepository, no de InMemoryChampionRepository
        ChampionRepository repo = new InMemoryChampionRepository();

        // Guardem dos campions
        ChampionRecord ahri = new ChampionRecord("CHAMP-1", "Ahri", 53.2, 12.5);
        ChampionRecord yasuo = new ChampionRecord("CHAMP-2", "Yasuo", 49.8, 15.3);
        repo.save(ahri);
        repo.save(yasuo);

        // Busquem per ID — Optional ens obliga a gestionar l'absencia
        Optional<ChampionRecord> found = repo.findById("CHAMP-1");
        found.ifPresent(c -> System.out.println("Trobat: " + c.name()));  // "Trobat: Ahri"

        // Busquem un ID que no existeix
        Optional<ChampionRecord> notFound = repo.findById("CHAMP-999");
        System.out.println("Existeix? " + notFound.isPresent());  // false

        // Llistem tots els campions
        List<ChampionRecord> all = repo.findAll();
        System.out.println("Total campions: " + all.size());  // 2

        // Verifiquem que la llista es immutable — aixo HA de petar
        try {
            all.add(new ChampionRecord("CHAMP-3", "Hack", 0.0, 0.0));
        } catch (UnsupportedOperationException e) {
            System.out.println("Llista immutable OK — no es pot modificar des de fora");
        }
    }
}
```

### 4. Mirror Python (30 min)

Crea a `ai-python/src/`:
- `champion_record.py` — `@dataclass(frozen=True)` amb `is_meta()` i `with_patch_adjustment()`
- `champion_repository.py` — `ChampionRepository(ABC)` i `InMemoryChampionRepository`

Prova amb un script:

```python
# ai-python/src/demo_repository.py
from champion_record import ChampionRecord
from champion_repository import InMemoryChampionRepository

# Creem el repositori i uns quants campions
repo = InMemoryChampionRepository()
ahri = ChampionRecord("CHAMP-1", "Ahri", 53.2, 12.5)
yasuo = ChampionRecord("CHAMP-2", "Yasuo", 49.8, 15.3)

repo.save(ahri)
repo.save(yasuo)

# Busquem per ID
found = repo.find_by_id("CHAMP-1")
print(f"Trobat: {found.name}" if found else "No trobat")  # "Trobat: Ahri"

# Verifiquem immutabilitat
try:
    ahri.win_rate = 60.0  # HA de petar — frozen=True
except Exception as e:
    print(f"Immutabilitat OK: {e}")
```

### 5. Commit (5 min)

```bash
git add backend-java/src/main/java/com/esportspulse/engine/repository/
git add ai-python/src/
git commit -m "feat: ChampionRepository interface + InMemory impl (Java + Python mirror)"
```

---

## Checklist de Lliurament

- [ ] `ChampionRepository` interficie amb `save`, `findById`, `findAll`, `delete`
- [ ] `InMemoryChampionRepository` implementa tots 4 metodes amb `ConcurrentHashMap`
- [ ] `findById` retorna `Optional<ChampionRecord>` — mai null
- [ ] `findAll` retorna una llista immutable (modificar-la peta)
- [ ] Python: `ChampionRecord` amb `@dataclass(frozen=True)` funcional
- [ ] Python: `ChampionRepository(ABC)` amb `InMemoryChampionRepository` funcional
- [ ] Demo Java i Python executats sense errors
- [ ] Commit amb format Conventional Commits
