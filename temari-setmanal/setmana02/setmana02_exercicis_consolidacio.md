# Setmana 2 — Exercicis de Consolidació

Aquests exercicis repassen els conceptes clau de la setmana. No cal lliurar-los — són per verificar que has entès la teoria i la pràctica abans de passar a la setmana 3. Intenta resoldre'ls sense mirar els apunts; si et quedes encallat, revisa el dia corresponent.

---

## Bloc 1: Validació i Compact Constructor (Dilluns)

**Exercici 1.1 — Objectes Sempre Vàlids**

Quin d'aquests codis és correcte i per què?

```java
// Opció A
ChampionRecord champ = new ChampionRecord("CHAMP-1", "Ahri", 52.3, 8.1);
if (champ.winRate() < 0 || champ.winRate() > 100) {
    System.out.println("ERROR: winRate invàlid");
}

// Opció B
// El compact constructor ja valida — si arribes aquí, l'objecte és vàlid
ChampionRecord champ = new ChampionRecord("CHAMP-1", "Ahri", 52.3, 8.1);
```

**Exercici 1.2 — Escriu un Compact Constructor**

Un **compact constructor** és un bloc de codi dins un `record` que s'executa automàticament quan es crea l'objecte. Serveix per validar que les dades siguin correctes — si no ho són, llança una excepció i l'objecte no arriba a existir. A diferència d'un constructor normal, no cal escriure `this.camp = camp` perquè Java ho fa sol.

Escriu el compact constructor per a un `MatchRecord` amb els camps: `matchId` (String), `duration` (int, en segons), `blueTeamWon` (boolean). Validacions:
- `matchId` no pot ser null ni buit
- `duration` ha de ser entre 600 (10 min) i 5400 (90 min)

**Exercici 1.3 — Patró With**

Els records són immutables — no tenen setters. El patró **`with*()`** és la convenció per "modificar" un objecte immutable: en comptes de canviar l'original, retorna un **objecte nou** amb el canvi aplicat. L'original queda intacte.

Donat:
```java
ChampionRecord ahri = new ChampionRecord("CHAMP-1", "Ahri", 52.3, 8.1);
ChampionRecord ahriBuffed = ahri.withPatchAdjustment(3.0);
```

1. Quin és el winRate de `ahri` després de la crida?
2. Quin és el winRate de `ahriBuffed`?
3. Per què `withPatchAdjustment` no es diu `setPatchAdjustment`?

---

## Bloc 2: Interfícies i Repository Pattern (Dimarts)

**Exercici 2.1 — Interfície vs Implementació**

Indica si cada afirmació és certa o falsa:

1. Una interfície defineix QUÈ fan els mètodes, no COM.
2. Una classe pot implementar múltiples interfícies.
3. Pots crear una instància d'una interfície amb `new ChampionRepository()`.
4. Si canvies la implementació de `InMemoryChampionRepository`, has de canviar també `ChampionRepository`.

**Exercici 2.2 — Optional**

Quin és el problema d'aquest codi?

```java
ChampionRecord champ = repo.findById("CHAMP-999");
System.out.println(champ.name());
```

Reescriu-lo usant `Optional` de forma segura (dues maneres: amb `ifPresent` i amb `orElseThrow`).

**Exercici 2.3 — Disseny d'Interfície**

Escriu una interfície `PlayerRepository` amb els mètodes necessaris per:
- Guardar un jugador
- Buscar un jugador per ID (pot no existir)
- Obtenir tots els jugadors
- Eliminar un jugador per ID

---

## Bloc 3: SOLID, Dependency Injection i Capa de Servei (Dimecres)

**Exercici 3.1 — Identifica el Principi SOLID**

Per a cada cas, indica quin principi SOLID es viola:

1. Un servei crea les seves dependències amb `new InMemoryChampionRepository()` directament.
2. Una classe `GameService` té mètodes per guardar jocs, enviar emails, generar PDFs i fer analytics.
3. Cada cop que afegeixes un nou tipus d'exportació (CSV, JSON, XML), modifiques la mateixa classe.
4. Una interfície `DataService` té 20 mètodes i una classe que només necessita llegir n'ha d'implementar tots 20.

**Exercici 3.2 — Constructor Injection**

Transforma aquest codi per usar injecció de dependències:

```java
public class ChampionManagementService {
    private InMemoryChampionRepository repo = new InMemoryChampionRepository();
    private ChampionRecordFactory factory = new ChampionRecordFactory();

    public void registerChampion(String id, String name, double winRate) {
        ChampionRecord champ = factory.createDefault(id, name, winRate);
        repo.save(champ);
    }
}
```

**Exercici 3.3 — Avantatge Real**

Amb el codi de l'exercici anterior transformat, escriu el codi que:
1. Crea el servei per a producció (amb `SqlChampionRepository`)
2. Crea el servei per a tests (amb `InMemoryChampionRepository`)

Quantes línies del servei has de canviar entre els dos casos?

---

## Bloc 4: Python Dataclasses i .cursorrules (Dijous)

**Exercici 4.1 — Java Record vs Python Dataclass**

Completa la taula:

| Concepte | Java | Python |
|----------|------|--------|
| Classe immutable | `record` | ________ |
| Validació al constructor | compact constructor | ________ |
| Mètode que retorna nou objecte | `withPatchAdjustment()` | ________ |
| Interfície | `interface` | ________ |

**Exercici 4.2 — Escriu un Dataclass**

Escriu un `MatchRecord` en Python amb `@dataclass(frozen=True)`:
- Camps: `match_id` (str), `duration` (int), `blue_team_won` (bool)
- Validació a `__post_init__`: `match_id` no buit, `duration` entre 600 i 5400
- Mètode `is_long_game()` que retorna `True` si la durada supera els 2400 segons (40 min)

**Exercici 4.3 — .cursorrules**

Dels següents fragments de `.cursorrules`, indica quins són prou precisos i quins són massa vagues:

1. `"Escriu bon codi"`
2. `"Models de domini: record (Java), @dataclass(frozen=True) (Python). Mai setters."`
3. `"Usa bons noms de variable"`
4. `"Variables Java en camelCase: championRecord, pickRate. Variables Python en snake_case: champion_record, pick_rate"`
5. `"Segueix bones pràctiques de testing"`
6. `"Noms de test: metode_comportament_condicio (ex: findById_returnsEmpty_whenNotFound)"`

---

## Bloc 5: Tests (Divendres)

**Exercici 5.1 — Given / When / Then**

Per a cada test, escriu els tres passos (no cal codi, només la descripció en text):

1. Test que verifica que `ChampionRecord` amb `championId` null llança `IllegalArgumentException`.
2. Test que verifica que `InMemoryChampionRepository.findById` retorna buit si l'ID no existeix.
3. Test que verifica que `ChampionManagementService.registerChampion` guarda el campió al repositori.

**Exercici 5.2 — Escriu un Test**

Sense mirar els apunts, escriu un test JUnit 5 complet que verifiqui:

> Donat un `InMemoryChampionRepository` buit, quan guardes un `ChampionRecord` i el busques per ID, el resultat no és buit i té el nom correcte.

Inclou: `@BeforeEach`, `@Test`, i les assercions necessàries.

**Exercici 5.3 — Per Què DI Ajuda als Tests**

Explica amb les teves paraules per què podem testejar `ChampionManagementService` sense una base de dades real. Quina classe usem en comptes de la BD i per què funciona?

---

## Exercici Final: Integració

Crea una nova entitat `MatchRecord` de zero, en ambdós llenguatges:

1. **Java record** amb camps: `matchId` (String), `duration` (int), `blueTeamWon` (boolean). Compact constructor amb validació. Mètode `isLongGame()`.

2. **Python dataclass** equivalent amb `@dataclass(frozen=True)`, `__post_init__` i `is_long_game()`.

3. **Interfície** `MatchRepository` en Java amb `save`, `findById`, `findAll`.

4. **InMemoryMatchRepository** que implementa la interfície.

5. **Un test JUnit 5** que verifica que guardar i buscar funciona correctament.

6. **Commit** amb missatge Conventional Commits.

> **Nota:** Aquesta entitat s'utilitzarà en setmanes posteriors quan connectem amb l'API de Riot per obtenir dades de partides reals.

---
---

# Solucions

> **Atenció:** Intenta resoldre els exercicis abans de mirar les solucions.

---

## Bloc 1: Validació i Compact Constructor

**Solució 1.1 — Objectes Sempre Vàlids**

L'**Opció B** és correcta. Si el compact constructor valida els camps, qualsevol `ChampionRecord` que existeixi ja és vàlid per definició. L'Opció A comprova després de crear — això és redundant si el constructor ja ho fa, i perillós si algú s'oblida de fer la comprovació.

**Solució 1.2 — Compact Constructor**

```java
public record MatchRecord(String matchId, int duration, boolean blueTeamWon) {
    public MatchRecord {
        if (matchId == null || matchId.isBlank()) {
            throw new IllegalArgumentException("matchId no pot ser buit");
        }
        if (duration < 600 || duration > 5400) {
            throw new IllegalArgumentException("duration ha de ser entre 600 i 5400 segons");
        }
    }
}
```

**Solució 1.3 — Patró With**

1. `ahri.winRate()` = **52.3** — l'original no canvia mai.
2. `ahriBuffed.winRate()` = **55.3** — el nou objecte té el valor ajustat (+3.0).
3. Perquè `set*` implica mutació (canviar l'objecte existent). `with*` indica que retorna un objecte **nou** — l'original queda intacte. És una convenció de Java modern per treballar amb objectes immutables.

---

## Bloc 2: Interfícies i Repository Pattern

**Solució 2.1 — Interfície vs Implementació**

1. **Certa** — la interfície és el contracte (QUÈ), la implementació és el detall (COM).
2. **Certa** — una classe pot implementar múltiples interfícies (`class X implements A, B`).
3. **Falsa** — no pots instanciar una interfície. Necessites una implementació: `new InMemoryChampionRepository()`.
4. **Falsa** — la interfície és el contracte estable. Canviar la implementació no afecta la interfície.

**Solució 2.2 — Optional**

El problema: si `findById` retorna `null` (l'ID no existeix), `champ.name()` llança `NullPointerException`.

```java
// Amb ifPresent:
repo.findById("CHAMP-999").ifPresent(
    champ -> System.out.println(champ.name())
);

// Amb orElseThrow:
ChampionRecord champ = repo.findById("CHAMP-999")
    .orElseThrow(() -> new RuntimeException("Campió no trobat: CHAMP-999"));
System.out.println(champ.name());
```

**Solució 2.3 — Disseny d'Interfície**

```java
public interface PlayerRepository {
    void save(PlayerRecord player);
    Optional<PlayerRecord> findById(String playerId);
    List<PlayerRecord> findAll();
    void delete(String playerId);
}
```

---

## Bloc 3: SOLID, DI i Capa de Servei

**Solució 3.1 — Identifica el Principi SOLID**

1. **DIP** (Dependency Inversion) — hauria de rebre la dependència per constructor, no crear-la.
2. **SRP** (Single Responsibility) — fa massa coses, s'hauria de separar en classes especialitzades.
3. **OCP** (Open/Closed) — s'hauria de poder afegir un nou exportador sense modificar la classe existent (usar interfície `Exporter`).
4. **ISP** (Interface Segregation) — s'hauria de dividir en interfícies petites (`ReadService`, `WriteService`).

**Solució 3.2 — Constructor Injection**

```java
public class ChampionManagementService {
    private final ChampionRepository repo;
    private final ChampionRecordFactory factory;

    public ChampionManagementService(ChampionRepository repo, ChampionRecordFactory factory) {
        this.repo = repo;
        this.factory = factory;
    }

    public void registerChampion(String id, String name, double winRate) {
        ChampionRecord champ = factory.createDefault(id, name, winRate);
        repo.save(champ);
    }
}
```

**Solució 3.3 — Avantatge Real**

```java
// Producció:
ChampionManagementService prodService = new ChampionManagementService(
    new SqlChampionRepository(dataSource), new ChampionRecordFactory());

// Tests:
ChampionManagementService testService = new ChampionManagementService(
    new InMemoryChampionRepository(), new ChampionRecordFactory());
```

Zero línies del servei canvien. Només canvia la línia on es crea el servei — la implementació del repo és diferent, però el servei no ho sap ni li importa.

---

## Bloc 4: Python Dataclasses i .cursorrules

**Solució 4.1 — Java Record vs Python Dataclass**

| Concepte | Java | Python |
|----------|------|--------|
| Classe immutable | `record` | `@dataclass(frozen=True)` |
| Validació al constructor | compact constructor | `__post_init__` |
| Mètode que retorna nou objecte | `withPatchAdjustment()` | `with_patch_adjustment()` |
| Interfície | `interface` | `abc.ABC` amb `@abstractmethod` |

**Solució 4.2 — Escriu un Dataclass**

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class MatchRecord:
    match_id: str
    duration: int
    blue_team_won: bool

    def __post_init__(self):
        if not self.match_id or not self.match_id.strip():
            raise ValueError("match_id no pot ser buit")
        if self.duration < 600 or self.duration > 5400:
            raise ValueError("duration ha de ser entre 600 i 5400 segons")

    def is_long_game(self) -> bool:
        return self.duration > 2400
```

**Solució 4.3 — .cursorrules**

1. `"Escriu bon codi"` — **Vague.** Què és "bon"? Cada persona interpreta diferent.
2. `"Models de domini: record (Java)..."` — **Precís.** L'agent sap exactament què generar.
3. `"Usa bons noms de variable"` — **Vague.** Quina convenció? camelCase? snake_case?
4. `"Variables Java en camelCase..."` — **Precís.** Exemples concrets inclosos.
5. `"Segueix bones pràctiques de testing"` — **Vague.** Quines pràctiques? Quin format de noms?
6. `"Noms de test: metode_comportament_condicio..."` — **Precís.** Format clar amb exemple.

---

## Bloc 5: Tests

**Solució 5.1 — Given / When / Then**

1. **GIVEN:** Res (creació directa). **WHEN:** Creem un `ChampionRecord` amb `championId = null`. **THEN:** Es llança `IllegalArgumentException`.

2. **GIVEN:** Un `InMemoryChampionRepository` buit. **WHEN:** Busquem per un ID que no existeix. **THEN:** El resultat és `Optional.empty()`.

3. **GIVEN:** Un servei amb un `InMemoryChampionRepository` buit. **WHEN:** Cridem `registerChampion("CHAMP-1", "Ahri", 52.3)`. **THEN:** El repositori conté un campió amb ID "CHAMP-1".

**Solució 5.2 — Escriu un Test**

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import static org.junit.jupiter.api.Assertions.*;

class InMemoryChampionRepositoryTest {

    private InMemoryChampionRepository repo;

    @BeforeEach
    void setUp() {
        repo = new InMemoryChampionRepository();
    }

    @Test
    void save_andFindById_returnsChampion() {
        ChampionRecord ahri = new ChampionRecord("CHAMP-1", "Ahri", 52.3, 8.1);
        repo.save(ahri);

        var result = repo.findById("CHAMP-1");

        assertTrue(result.isPresent());
        assertEquals("Ahri", result.get().name());
    }
}
```

**Solució 5.3 — Per Què DI Ajuda als Tests**

Podem testejar `ChampionManagementService` sense BD real perquè el servei **no sap** quina implementació de `ChampionRepository` està usant — rep una interfície pel constructor. Als tests li passem un `InMemoryChampionRepository` que guarda tot en memòria (un `ConcurrentHashMap`). El servei funciona exactament igual perquè els mètodes (`save`, `findById`, etc.) compleixen el mateix contracte. Això és possible gràcies a la injecció de dependències (DIP).

---

## Exercici Final: Integració

**Java Record:**

```java
package com.esportspulse.engine.model;

public record MatchRecord(String matchId, int duration, boolean blueTeamWon) {
    public MatchRecord {
        if (matchId == null || matchId.isBlank()) {
            throw new IllegalArgumentException("matchId no pot ser buit");
        }
        if (duration < 600 || duration > 5400) {
            throw new IllegalArgumentException("duration ha de ser entre 600 i 5400");
        }
    }

    public boolean isLongGame() {
        return duration > 2400;
    }
}
```

**Python Dataclass:**

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class MatchRecord:
    match_id: str
    duration: int
    blue_team_won: bool

    def __post_init__(self):
        if not self.match_id or not self.match_id.strip():
            raise ValueError("match_id no pot ser buit")
        if self.duration < 600 or self.duration > 5400:
            raise ValueError("duration ha de ser entre 600 i 5400")

    def is_long_game(self) -> bool:
        return self.duration > 2400
```

**Interfície i Implementació:**

```java
public interface MatchRepository {
    void save(MatchRecord match);
    Optional<MatchRecord> findById(String matchId);
    List<MatchRecord> findAll();
}

public class InMemoryMatchRepository implements MatchRepository {
    private final Map<String, MatchRecord> storage = new ConcurrentHashMap<>();

    public void save(MatchRecord match) { storage.put(match.matchId(), match); }
    public Optional<MatchRecord> findById(String matchId) { return Optional.ofNullable(storage.get(matchId)); }
    public List<MatchRecord> findAll() { return List.copyOf(storage.values()); }
}
```

**Test:**

```java
class InMemoryMatchRepositoryTest {

    private InMemoryMatchRepository repo;

    @BeforeEach
    void setUp() {
        repo = new InMemoryMatchRepository();
    }

    @Test
    void save_andFindById_returnsMatch() {
        MatchRecord match = new MatchRecord("MATCH-1", 1800, true);
        repo.save(match);

        var result = repo.findById("MATCH-1");
        assertTrue(result.isPresent());
        assertEquals("MATCH-1", result.get().matchId());
        assertTrue(result.get().blueTeamWon());
    }
}
```

**Commit:** `feat: add MatchRecord entity with repository, tests, and Python equivalent`
