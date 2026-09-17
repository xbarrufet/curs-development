# Setmana 1 — Exercicis de Consolidació

Aquests exercicis repassen els conceptes clau de la setmana. No cal lliurar-los — són per verificar que has entès la teoria i la pràctica abans de passar a la setmana 2. Intenta resoldre'ls sense mirar els apunts; si et quedes encallat, revisa el dia corresponent.

---

## Bloc 1: Git i Estructura de Projecte (Dilluns)

**Exercici 1.1 — Conventional Commits**

Dels següents missatges de commit, indica quins segueixen el format Conventional Commits correctament i quins no. Per als incorrectes, explica què falla:

1. `feat(java): add PlayerRecord domain entity`
2. `he afegit la classe de jugadors`
3. `fix(search): correct null handling in linear search`
4. `FIX: arreglat bug`
5. `docs: add README with setup instructions`
6. `feat: coses noves`

**Exercici 1.2 — Estructura Maven**

Sense mirar el projecte, respon:
1. En quin directori va el codi font Java d'una aplicació Maven?
2. En quin directori van els tests?
3. Si la teva classe té el paquet `com.esportspulse.engine.model`, quin és el path complet del fitxer dins el projecte?

---

## Bloc 2: Entitats de Domini i Immutabilitat (Dimarts)

**Exercici 2.1 — Record vs Classe Mutable**

Donat aquest codi:

```java
public record PlayerRecord(String playerId, String name, int ranking) {}
```

1. Com accediries al nom d'un jugador `p`? Escriu la crida.
2. Pots fer `p.setName("NouNom")`? Per què sí o per què no?
3. Si necessites un `PlayerRecord` amb un ranking diferent, com ho fas?

**Exercici 2.2 — Detecta el Problema**

Analitza aquest codi i explica quin problema pot causar:

```java
class MutablePlayer {
    private String name;
    private int ranking;

    public void setRanking(int r) { this.ranking = r; }
}

void processPlayer(MutablePlayer player) {
    player.setRanking(1);  // "només un ajust temporal"
}

// En un altre lloc del codi:
MutablePlayer player = new MutablePlayer("Faker", 5);
processPlayer(player);
System.out.println(player.ranking);  // Quin valor imprimeix? Per què és problemàtic?
```

---

## Bloc 3: Col·leccions — ArrayList vs HashMap (Dimarts / Dimecres)

**Exercici 3.1 — Tria l'Estructura Correcta**

Per a cadascun d'aquests casos, indica si usaries `ArrayList` o `HashMap` i per què:

1. Vols guardar els 10 últims resultats d'un jugador en ordre cronològic.
2. Vols trobar ràpidament un jugador pel seu nom.
3. Vols recórrer tots els jugadors i calcular el nivell mitjà.
4. Vols saber si el playerId `"P-4521"` existeix sense recórrer res.

**Exercici 3.2 — Completa el Codi**

Escriu el codi que falta (sense mirar els apunts):

```java
// 1. Crea un HashMap buit que mapeja String -> PlayerRecord
________ players = ________;

// 2. Afegeix un jugador al mapa amb playerId com a clau
PlayerRecord faker = new PlayerRecord("P-1", "Faker", 500, 3500.0);
________;

// 3. Recupera el jugador amb playerId "P-1"
PlayerRecord found = ________;

// 4. Comprova si existeix la clau "P-99"
boolean exists = ________;
```

---

## Bloc 4: Big-O i Ordre de Magnitud (Dimecres)

**Exercici 4.1 — Classifica la Complexitat**

Quina complexitat Big-O té cadascuna d'aquestes operacions?

1. `map.get("P-500")` sobre un HashMap de 100.000 elements.
2. Recórrer una llista de 100.000 elements amb un `for-each` per trobar un element concret.
3. `list.get(42)` sobre un ArrayList.
4. Per cada element d'una llista de 1.000 elements, recórrer una altra llista de 1.000 elements.

**Exercici 4.2 — Predicció**

Un codi amb complexitat O(n) triga 2ms per processar 10.000 elements. Aproximadament, quant trigarà amb:

1. 100.000 elements?
2. 1.000.000 elements?

I si la complexitat fos O(n²), quant trigaria amb 100.000 elements si amb 10.000 triga 2ms?

---

## Bloc 5: Benchmarking (Dijous)

**Exercici 5.1 — Warm-up**

Explica amb les teves paraules:
1. Per què les primeres execucions d'un mètode Java són més lentes que les posteriors?
2. Quin problema causa això si mesurem el temps de la primera execució?
3. Com ho resolem al `BenchmarkRunner`?

**Exercici 5.2 — Tria l'Eina Correcta**

Vols mesurar quant triga una cerca que dura uns microsegons. Quina crida fas servir i per què?

- `System.currentTimeMillis()`
- `System.nanoTime()`

---

## Bloc 6: Tests amb JUnit 5 (Divendres)

**Exercici 6.1 — Noms de Test**

Reescriu aquests noms de test perquè descriguin el comportament verificat:

1. `void test1()`
2. `void testSearch()`
3. `void testNullReturn()`

**Exercici 6.2 — Escriu un Test**

Sense mirar els apunts, escriu un test JUnit 5 complet que verifiqui el següent:

> Donat un `HashMap` amb 1.000 jugadors generats per `PlayerDataGenerator`, quan busques un playerId que existeix (`"P-500"`), el resultat no és null i té el playerId correcte.

Inclou: imports, `@BeforeEach`, `@Test`, i les assercions necessàries.

**Exercici 6.3 — Detecta l'Error**

Aquest test sempre passa, però conté un error de disseny. Quin és?

```java
@Test
void searchFindsPlayer() {
    List<PlayerRecord> players = PlayerDataGenerator.generateList(1_000);
    Map<String, PlayerRecord> map = PlayerDataGenerator.toMap(players);
    PlayerRecord result = map.get("P-500");
    assertNotNull(result);
}
```

Pista: pensa en `@BeforeEach` i per què existeix.

---

## Bloc 7: Clean Code (Divendres)

**Exercici 7.1 — Detecta el Problema**

Aquest mètode fa el que ha de fer, però té almenys dos problemes de clean code. Identifica'ls:

```java
public PlayerRecord get(List<PlayerRecord> l, String s) {
    if (l != null) {
        if (!l.isEmpty()) {
            for (var p : l) {
                if (p.playerId().equals(s)) {
                    return p;
                }
            }
        }
    }
    return null;
}
```

**Exercici 7.2 — Reescriu'l**

Reescriu el mètode de l'exercici 7.1 aplicant noms significatius i early return.

---

## Exercici Final: Integració

Crea un nou record `PlayerRecord` amb els camps: `playerId` (String), `username` (String), `level` (int), `hoursPlayed` (double).

1. Crea el fitxer al paquet correcte del projecte.
2. Escriu un `main` que creï 5 jugadors, els guardi en un `ArrayList` i un `HashMap` (clau: `playerId`), i imprimeixi el jugador amb més hores jugades (recorrent la llista).
3. Escriu un test JUnit 5 que verifiqui que la cerca per `playerId` al HashMap retorna el jugador correcte.
4. Fes commit amb un missatge Conventional Commits adequat.

> **Nota:** Aquest exercici és preparatori — a la setmana 2 treballarem amb dades reals de l'API de Riot i introduirem `ChampionRecord`.

---
---

# Solucions

> **Atenció:** Intenta resoldre els exercicis abans de mirar les solucions. Aprens molt més si primer t'equivoques i després compares.

---

## Bloc 1: Git i Estructura de Projecte

**Solució 1.1 — Conventional Commits**

1. `feat(java): add PlayerRecord domain entity` — **Correcte.** Tipus `feat`, scope `java`, descripció clara en imperatiu.
2. `he afegit la classe de jugadors` — **Incorrecte.** No segueix el format. Falta el type (`feat`, `fix`...), no té scope, i la descripció no està en imperatiu anglès. Hauria de ser: `feat(java): add PlayerRecord domain entity`.
3. `fix(search): correct null handling in linear search` — **Correcte.** Tipus `fix`, scope `search`, descripció clara.
4. `FIX: arreglat bug` — **Incorrecte.** El type ha d'anar en minúscules (`fix`, no `FIX`). La descripció "arreglat bug" és massa vaga — quin bug? On? Hauria de ser algo com: `fix(search): return null when id not found`.
5. `docs: add README with setup instructions` — **Correcte.** Tipus `docs`, sense scope (correcte si afecta el projecte en general), descripció clara.
6. `feat: coses noves` — **Incorrecte.** La descripció no diu res útil. Quines coses? A on? Un bon missatge seria: `feat(java): add PlayerRecord domain entity`.

**Solució 1.2 — Estructura Maven**

1. `src/main/java/`
2. `src/test/java/`
3. `backend-java/src/main/java/com/esportspulse/engine/model/NomDeLaClasse.java`

---

## Bloc 2: Entitats de Domini i Immutabilitat

**Solució 2.1 — Record vs Classe Mutable**

1. `p.name()` — els records generen accessors sense el prefix `get`.
2. No. Els records són immutables — no generen setters. El codi no compilarà.
3. Crees un objecte nou: `PlayerRecord updated = new PlayerRecord(p.playerId(), "NouNom", p.ranking());`

**Solució 2.2 — Detecta el Problema**

Imprimeix `1`, no `5`. El mètode `processPlayer` modifica l'objecte original perquè és mutable. Qui ha creat el `player` amb ranking 5 no espera que una altra funció li canviï el valor. Això és un **efecte lateral** — el codi es torna imprevisible perquè qualsevol mètode que rebi l'objecte el pot modificar sense que el codi que l'ha creat ho sàpiga. Amb un `record` immutable, això no pot passar.

---

## Bloc 3: Col·leccions — ArrayList vs HashMap

**Solució 3.1 — Tria l'Estructura Correcta**

1. **ArrayList** — necessites ordre cronològic (posició 0 = més antic, posició 9 = més recent). Les llistes mantenen l'ordre d'inserció.
2. **HashMap** — necessites cerca ràpida per clau (el nom). `map.get("Faker")` és O(1).
3. **ArrayList** (o qualsevol de les dues) — has de recórrer tots els elements igualment. Ambdues serveixen, però si no necessites cerca per clau, una llista és més senzilla.
4. **HashMap** — `map.containsKey("P-4521")` és O(1), sense recórrer res.

**Solució 3.2 — Completa el Codi**

```java
// 1.
Map<String, PlayerRecord> players = new HashMap<>();

// 2.
players.put(faker.playerId(), faker);

// 3.
PlayerRecord found = players.get("P-1");

// 4.
boolean exists = players.containsKey("P-99");
```

---

## Bloc 4: Big-O i Ordre de Magnitud

**Solució 4.1 — Classifica la Complexitat**

1. **O(1)** — accés directe per clau al HashMap.
2. **O(n)** — en el pitjor cas, recorres tots els 100.000 elements.
3. **O(1)** — accés directe per índex a l'ArrayList (un array intern).
4. **O(n²)** — per cada element (1.000) recorres una altra llista (1.000) = 1.000 × 1.000 = 1.000.000 operacions.

**Solució 4.2 — Predicció**

Amb O(n) — creix proporcionalment:
1. 100.000 elements = 10× més dades → **~20ms**
2. 1.000.000 elements = 100× més dades → **~200ms**

Amb O(n²) — creix quadràticament:
- 10.000 elements → 2ms
- 100.000 elements = 10× més dades → 10² = 100× més temps → **~200ms**

---

## Bloc 5: Benchmarking

**Solució 5.1 — Warm-up**

1. Perquè la JVM primer **interpreta** el bytecode (lent) i després, quan detecta que un mètode s'executa moltes vegades, el **compila a codi natiu** amb el compilador JIT (ràpid). Les primeres execucions usen l'intèrpret.
2. Estàs mesurant el temps de l'intèrpret, no del teu algorisme. Els resultats són enganyosament lents i no representen el rendiment real.
3. Executem cada mètode 100 vegades **sense mesurar** (warm-up) per forçar la compilació JIT. Després mesurem 1.000 execucions reals.

**Solució 5.2 — Tria l'Eina Correcta**

**`System.nanoTime()`** — té precisió de nanosegons, ideal per mesurar durades curtes (microsegons). `currentTimeMillis()` té precisió de mil·lisegons, massa gros per a microbenchmarks, i pot saltar si l'OS ajusta el rellotge del sistema.

---

## Bloc 6: Tests amb JUnit 5

**Solució 6.1 — Noms de Test**

Respostes possibles (hi ha variacions vàlides):

1. `void searchLinear_findsExistingPlayer()`
2. `void searchByKey_returnsCorrectPlayer_whenIdExists()`
3. `void searchLinear_returnsNull_whenIdDoesNotExist()`

**Solució 6.2 — Escriu un Test**

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import static org.junit.jupiter.api.Assertions.*;

import java.util.Map;

class PlayerSearchByKeyTest {

    private Map<String, PlayerRecord> map;

    @BeforeEach
    void setUp() {
        var list = PlayerDataGenerator.generateList(1_000);
        map = PlayerDataGenerator.toMap(list);
    }

    @Test
    void searchByKey_findsExistingPlayer() {
        PlayerRecord result = map.get("P-500");
        assertNotNull(result);
        assertEquals("P-500", result.playerId());
    }
}
```

**Solució 6.3 — Detecta l'Error**

El test funciona, però **crea les dades dins del propi test** en comptes d'usar `@BeforeEach`. Això significa que:
- Si afegeixes un segon test, hauràs de copiar les mateixes línies de creació de dades.
- Si un test modifiqués l'estat (en casos amb objectes mutables), podria contaminar altres tests.

La preparació de dades hauria d'anar a `@BeforeEach` perquè cada test arrenqui amb un estat fresc i compartit, sense duplicar codi.

---

## Bloc 7: Clean Code

**Solució 7.1 — Detecta el Problema**

1. **Noms críptics** — `get`, `l`, `s` no diuen res sobre què fa el mètode ni què representen els paràmetres.
2. **Nesting excessiu** — dos `if` niats abans d'arribar al bucle són innecessaris; es poden convertir en early returns.

**Solució 7.2 — Reescriu'l**

```java
public PlayerRecord searchLinear(List<PlayerRecord> players, String playerId) {
    if (players == null || players.isEmpty()) return null;

    for (var p : players) {
        if (p.playerId().equals(playerId)) return p;
    }
    return null;
}
```

---

## Exercici Final: Integració

**Solució — PlayerRecord**

1. Fitxer: `backend-java/src/main/java/com/esportspulse/engine/model/PlayerRecord.java`

```java
package com.esportspulse.engine.model;

public record PlayerRecord(
    String playerId,
    String username,
    int level,
    double hoursPlayed
) {}
```

2. Main amb cerca del jugador amb més hores jugades:

```java
package com.esportspulse.engine;

import com.esportspulse.engine.model.PlayerRecord;
import java.util.*;

public class PlayerExplorer {
    public static void main(String[] args) {
        List<PlayerRecord> list = new ArrayList<>();
        list.add(new PlayerRecord("P-1", "Faker", 500, 3500.0));
        list.add(new PlayerRecord("P-2", "Caps", 350, 2800.5));
        list.add(new PlayerRecord("P-3", "Rekkles", 420, 3100.0));
        list.add(new PlayerRecord("P-4", "Jankos", 380, 2950.0));
        list.add(new PlayerRecord("P-5", "Perkz", 400, 3200.5));

        Map<String, PlayerRecord> map = new HashMap<>();
        for (PlayerRecord p : list) {
            map.put(p.playerId(), p);
        }

        PlayerRecord best = list.get(0);
        for (PlayerRecord p : list) {
            if (p.hoursPlayed() > best.hoursPlayed()) {
                best = p;
            }
        }

        System.out.println("Jugador amb més hores jugades: " + best);
    }
}
```

3. Test:

```java
package com.esportspulse.engine;

import com.esportspulse.engine.model.PlayerRecord;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import static org.junit.jupiter.api.Assertions.*;

import java.util.*;

class PlayerExplorerTest {

    private Map<String, PlayerRecord> map;

    @BeforeEach
    void setUp() {
        map = new HashMap<>();
        map.put("P-1", new PlayerRecord("P-1", "Faker", 500, 3500.0));
        map.put("P-2", new PlayerRecord("P-2", "Caps", 350, 2800.5));
        map.put("P-3", new PlayerRecord("P-3", "Rekkles", 420, 3100.0));
    }

    @Test
    void searchByKey_returnsCorrectPlayer() {
        PlayerRecord result = map.get("P-2");
        assertNotNull(result);
        assertEquals("Caps", result.username());
    }
}
```

4. Commit: `feat(java): add PlayerRecord entity with explorer and test`
