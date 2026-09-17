# Setmana 04 — Dilluns: Repository Pattern — Per que Abstraure l'Acces a Dades

## Objectiu del Dia

Entendre per que el patró Repository es fonamental per desacoblar la logica de negoci del mecanisme d'emmagatzematge. Al final del dia tindras una interficie `ChampionRepository`, una implementacio `InMemoryChampionRepository` i el servei `ChampionManagementService` injectat amb la interficie — sense dependre de cap implementacio concreta.

---

## Teoria

### El Problema: Acoblament Directe

Fins ara, el nostre `ChampionManagementService` utilitza directament un `ConcurrentHashMap` per guardar els campions. Aixo funciona, pero te un problema greu:

```java
// PROBLEMA: El servei coneix el mecanisme d'emmagatzematge
// Si volem canviar a SQL, hem de modificar TOT el servei
public class ChampionManagementService {

    // Acoblament directe: el servei sap COM es guarden les dades
    private final ConcurrentHashMap<String, ChampionRecord> champions =
        new ConcurrentHashMap<>();

    public void registerChampion(ChampionRecord champion) {
        // Logica de negoci barrejada amb logica d'emmagatzematge
        if (champions.containsKey(champion.championId())) {
            throw new IllegalArgumentException("El campió ja existeix");
        }
        champions.put(champion.championId(), champion);
    }

    public Optional<ChampionRecord> findChampion(String championId) {
        // Si canviem a base de dades, hem de reescriure TOTS els metodes
        return Optional.ofNullable(champions.get(championId));
    }
}
```

**Que passa si volem canviar a una base de dades SQL?** Hem de reescriure tot el servei. I si tenim 10 serveis que fan servir el `ConcurrentHashMap`? 10 refactoritzacions.

### La Solucio: El Patro Repository

El patró Repository defineix un **contracte** (interficie) que descriu QUE operacions es poden fer amb les dades, sense dir COM es fan:

```
┌─────────────────────────────────┐
│  ChampionManagementService      │
│  (Logica de Negoci)             │
│                                 │
│  Nomes coneix la INTERFICIE     │
└────────────┬────────────────────┘
             │ depèn de
             ▼
┌─────────────────────────────────┐
│  <<interface>>                  │
│  ChampionRepository             │
│                                 │
│  save(), findById(),            │
│  findAll(), delete()            │
└────────────┬────────────────────┘
             │ implementada per
        ┌────┴────┐
        ▼         ▼
┌────────────┐ ┌────────────┐
│ InMemory   │ │ JPA/SQL    │
│ Repository │ │ Repository │
└────────────┘ └────────────┘
```

**Avantatge clau:** Podem canviar d'`InMemory` a `JPA` sense tocar ni una linia del servei.

### Disseny de la Jerarquia d'Interficies

La interficie `ChampionRepository` defineix les operacions CRUD (Create, Read, Update, Delete):

```java
package com.esportspulse.engine.repository;

import com.esportspulse.engine.domain.ChampionRecord;
import java.util.List;
import java.util.Optional;

/**
 * Contracte per a l'acces a dades de campions.
 * Qualsevol implementacio (memoria, SQL, fitxer) ha de complir aquest contracte.
 * Principi DIP: els moduls d'alt nivell (servei) depenen d'abstraccions, no de detalls.
 */
public interface ChampionRepository {

    /**
     * Desa un campió. Si ja existeix (mateix championId), el sobreescriu.
     * @param champion el registre del campió a desar
     */
    void save(ChampionRecord champion);

    /**
     * Cerca un campió pel seu identificador unic.
     * @param championId l'identificador del campió
     * @return Optional amb el campió si existeix, buit altrament
     */
    Optional<ChampionRecord> findById(String championId);

    /**
     * Retorna tots els campions emmagatzemats.
     * @return llista immutable de tots els campions
     */
    List<ChampionRecord> findAll();

    /**
     * Elimina un campió pel seu identificador.
     * @param championId l'identificador del campió a eliminar
     */
    void delete(String championId);
}
```

### Implementacio InMemory

La primera implementacio utilitza el `ConcurrentHashMap` que ja teniem, pero ara encapsulat darrere la interficie:

```java
package com.esportspulse.engine.repository;

import com.esportspulse.engine.domain.ChampionRecord;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Implementacio en memoria del repositori de campions.
 * Ideal per a tests i desenvolupament rapid.
 * Les dades es perden quan l'aplicacio es reinicia.
 */
@Repository // Spring detecta aquesta classe i la registra com a bean
public class InMemoryChampionRepository implements ChampionRepository {

    // ConcurrentHashMap: thread-safe sense sincronitzacio explicita
    // La clau es el championId, el valor es el registre complet
    private final Map<String, ChampionRecord> store = new ConcurrentHashMap<>();

    @Override
    public void save(ChampionRecord champion) {
        // put() sobreescriu si la clau ja existeix — comportament "upsert"
        store.put(champion.championId(), champion);
    }

    @Override
    public Optional<ChampionRecord> findById(String championId) {
        // Optional evita NullPointerException — forcem el codi client a gestionar l'absencia
        return Optional.ofNullable(store.get(championId));
    }

    @Override
    public List<ChampionRecord> findAll() {
        // Retornem una copia immutable per evitar modificacions externes
        return List.copyOf(store.values());
    }

    @Override
    public void delete(String championId) {
        // remove() no llanca excepcio si la clau no existeix — comportament idempotent
        store.remove(championId);
    }
}
```

### El Servei Refactoritzat

Ara el servei nomes depèn de la **interficie**, no de la implementacio:

```java
package com.esportspulse.engine.service;

import com.esportspulse.engine.domain.ChampionRecord;
import com.esportspulse.engine.repository.ChampionRepository;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Optional;

/**
 * Servei de gestio de campions.
 * Depèn de ChampionRepository (interficie), NO de InMemoryChampionRepository.
 * Gracies a aixo, podem canviar la implementacio sense tocar aquest codi.
 */
@Service
public class ChampionManagementService {

    // Declarat com a interficie — el servei no sap si es InMemory, SQL o fitxer
    private final ChampionRepository repository;

    // Spring injecta automaticament la implementacio disponible (@Repository)
    public ChampionManagementService(ChampionRepository repository) {
        this.repository = repository;
    }

    public void registerChampion(ChampionRecord champion) {
        // Validacio de negoci — aixo SI es responsabilitat del servei
        repository.findById(champion.championId()).ifPresent(existing -> {
            throw new IllegalArgumentException(
                "El campió amb ID " + champion.championId() + " ja existeix"
            );
        });
        // Delegacio a la interficie — el COM es responsabilitat del repositori
        repository.save(champion);
    }

    public Optional<ChampionRecord> findChampion(String championId) {
        return repository.findById(championId);
    }

    public List<ChampionRecord> listAllChampions() {
        return repository.findAll();
    }

    public void removeChampion(String championId) {
        repository.delete(championId);
    }
}
```

### Principis SOLID en Accio

| Principi | Com s'aplica |
|---|---|
| **D — Dependency Inversion** | El servei depèn de l'abstraccio (`ChampionRepository`), no del detall (`InMemoryChampionRepository`) |
| **O — Open/Closed** | Podem afegir noves implementacions (SQL, fitxer, API) sense modificar el servei |
| **L — Liskov Substitution** | Qualsevol implementacio de `ChampionRepository` es intercanviable — el servei funciona igual |

### Equivalent en Python

Python no te interficies com Java, pero te `Protocol` (duck typing estructural) i `ABC` (classes abstractes):

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass

# Classe de dades equivalent al record de Java
@dataclass(frozen=True)  # frozen=True fa la classe immutable, com un record
class ChampionRecord:
    champion_id: str
    name: str
    games_played: int
    win_rate: float

# Interficie amb ABC — forces les subclasses a implementar els metodes
class ChampionRepository(ABC):
    """Contracte per a l'acces a dades de campions."""

    @abstractmethod
    def save(self, champion: ChampionRecord) -> None:
        """Desa un campió."""
        ...

    @abstractmethod
    def find_by_id(self, champion_id: str) -> ChampionRecord | None:
        """Cerca un campió per ID. Retorna None si no existeix."""
        ...

    @abstractmethod
    def find_all(self) -> list[ChampionRecord]:
        """Retorna tots els campions."""
        ...

    @abstractmethod
    def delete(self, champion_id: str) -> None:
        """Elimina un campió per ID."""
        ...


# Implementacio en memoria — mateixa estructura que Java
class InMemoryChampionRepository(ChampionRepository):
    """Repositori en memoria. Les dades es perden al tancar."""

    def __init__(self):
        # Diccionari Python — equivalent al HashMap de Java
        self._store: dict[str, ChampionRecord] = {}

    def save(self, champion: ChampionRecord) -> None:
        self._store[champion.champion_id] = champion

    def find_by_id(self, champion_id: str) -> ChampionRecord | None:
        # .get() retorna None si la clau no existeix — com Optional.empty()
        return self._store.get(champion_id)

    def find_all(self) -> list[ChampionRecord]:
        return list(self._store.values())

    def delete(self, champion_id: str) -> None:
        # pop amb default evita KeyError — comportament idempotent
        self._store.pop(champion_id, None)
```

### El Poder del Patro: Canviar Sense Tocar

```
Avui:           Service → InMemoryRepository (RAM)
Dema:           Service → JpaRepository (H2/SQL)
La setmana que ve: Service → MongoRepository (NoSQL)

El servei NO canvia. Mai. Nomes canviem quina implementacio injectem.
```

---

## Activitat

### Exercici: Implementa el Patró Repository a EsportsPulse

**Durada estimada:** 90 minuts

#### Pas 1: Crea la interficie (15 min)

1. Crea el paquet `com.esportspulse.engine.repository`
2. Crea la interficie `ChampionRepository` amb els 4 metodes CRUD
3. Documenta cada metode amb Javadoc en catala

#### Pas 2: Crea la implementacio InMemory (20 min)

1. Crea `InMemoryChampionRepository` al mateix paquet
2. Implementa els 4 metodes amb `ConcurrentHashMap`
3. Anota amb `@Repository`
4. Afegeix un metode extra: `int count()` que retorna el nombre de campions

#### Pas 3: Refactoritza el servei (20 min)

1. Modifica `ChampionManagementService` per dependre de `ChampionRepository`
2. Elimina qualsevol referencia directa a `ConcurrentHashMap`
3. Utilitza injeccio per constructor

#### Pas 4: Tests unitaris (25 min)

```java
// Test que demostra que el servei funciona amb qualsevol implementacio
@Test
void registerAndFind_withInMemoryRepository() {
    // Arrange: creem el repositori i el servei
    ChampionRepository repo = new InMemoryChampionRepository();
    ChampionManagementService service = new ChampionManagementService(repo);

    // Act: registrem un campió
    ChampionRecord ahri = new ChampionRecord("ahri-001", "Ahri", 1500, 52.3);
    service.registerChampion(ahri);

    // Assert: el campió es recuperable
    Optional<ChampionRecord> found = service.findChampion("ahri-001");
    assertTrue(found.isPresent());
    assertEquals("Ahri", found.get().name());
}

@Test
void registerDuplicate_throwsException() {
    // Arrange
    ChampionRepository repo = new InMemoryChampionRepository();
    ChampionManagementService service = new ChampionManagementService(repo);
    ChampionRecord ahri = new ChampionRecord("ahri-001", "Ahri", 1500, 52.3);

    // Act
    service.registerChampion(ahri);

    // Assert: registrar el mateix campió ha de fallar
    assertThrows(IllegalArgumentException.class, () -> {
        service.registerChampion(ahri);
    });
}
```

#### Pas 5: Python mirror (10 min)

1. Crea `champion_repository.py` amb la classe abstracta
2. Crea `in_memory_champion_repository.py` amb la implementacio
3. Escriu un test basic amb pytest

---

## Checklist de Lliurament

- [ ] Interficie `ChampionRepository` creada amb 4 metodes CRUD documentats
- [ ] `InMemoryChampionRepository` implementa la interficie amb `ConcurrentHashMap`
- [ ] `ChampionManagementService` refactoritzat per dependre de la interficie
- [ ] Injeccio per constructor funcionant (Spring `@Service` + `@Repository`)
- [ ] Minim 3 tests unitaris passant (register, find, delete)
- [ ] Python: classe abstracta + implementacio InMemory creades
- [ ] Tot compila: `mvn compile` sense errors
- [ ] Tests passen: `mvn test` verd
