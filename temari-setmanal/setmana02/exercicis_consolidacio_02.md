# Setmana 2 — Exercicis de Consolidació

---

## Bàsics (has de saber fer-ho)

### 1. Nou model immutable: PlayerRecord
Crea un `record` `PlayerRecord` amb camps: `playerId` (String), `username` (String), `level` (int), `hoursPlayed` (double). Afegeix validació al compact constructor (playerId no null, level >= 1, hoursPlayed >= 0). Afegeix un mètode `isVeteran()` que retorna true si `hoursPlayed > 1000`.

**Connexió S1:** Crea un `HashMap<String, PlayerRecord>` amb 50.000 jugadors i mesura el temps de cerca per `playerId`. Ha de ser O(1).

**Fet quan:** Record creat, 3 tests JUnit (validació, isVeteran true, isVeteran false), benchmark de cerca funcional.

### 2. Repository per a PlayerRecord
Crea `PlayerRepository` (interfície) i `InMemoryPlayerRepository` (implementació amb `ConcurrentHashMap`). Mètodes: `save()`, `findById()`, `findAll()`, `findByLevelGreaterThan(int level)`.

**Connexió SOLID:** `GameManagementService` depèn de `GameRepository`, `PlayerManagementService` depèn de `PlayerRepository`. Mateixa estructura, diferent domini. Això és OCP i DIP en acció.

**Fet quan:** Interfície + implementació + 4 tests (save+find, find nonexistent, findAll, findByLevel).

### 3. Python mirror: dataclass frozen
Tradueix `PlayerRecord` a Python amb `@dataclass(frozen=True)`. Implementa `PlayerRepository` amb `abc.ABC` i `InMemoryPlayerRepository` amb un `dict`. Tests amb `pytest`.

**Fet quan:** Codi Python executable amb 3+ tests que passen.

---

## Avançats (si vas sobrat)

### 4. Sealed interface per a events
Crea una `sealed interface GameEvent` amb tres implementacions: `GameCreated(GameRecord game)`, `PriceChanged(String appId, BigDecimal oldPrice, BigDecimal newPrice)`, `GameDeleted(String appId)`. Totes com a `record`. Escriu un mètode `processEvent(GameEvent event)` que usi `switch` amb pattern matching (Java 21).

**Fet quan:** Compila, els 3 tipus d'event es processen correctament, i si afegeixes un nou event el compilador t'obliga a gestionar-lo (exhaustive switch).

### 5. Audita el teu propi codi de S1
Revisa el codi de la setmana 1 i aplica els principis SOLID que has après aquesta setmana. Hi ha alguna classe que fa massa coses? Es podria extreure una interfície? El `BenchmarkRunner` depèn de concrecions? Fes un commit amb els canvis i un missatge que expliqui quins principis has aplicat.

**Fet quan:** Commit amb missatge descriptiu; cap test de S1 es trenca.
