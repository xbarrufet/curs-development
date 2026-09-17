# Setmana 1 — Divendres: Tests amb JUnit 5 i Pull Request

## Objectiu del Dia

Escriure tests unitaris que verifiquin que el servei de cerca funciona correctament i que el HashMap és significativament més ràpid que la cerca lineal. Tancar la setmana amb una Pull Request a GitHub. Al final del dia tens tests verds, codi fusionat a `main`, i la branca tancada.

---

## Teoria

### JUnit 5: Què és i Per Què

Un test unitari és un tros de codi que verifica que un mètode fa el que ha de fer. JUnit és el framework estàndard de Java per escriure'ls.

**Per què testejar?**
- Detectes bugs abans que arribin a producció
- Pots canviar codi amb confiança — si els tests passen, no has trencat res
- A qualsevol feina de programador, els tests són obligatoris. Codi sense tests no es fusiona.

### Anatomia d'un Test JUnit 5

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import static org.junit.jupiter.api.Assertions.*;

class PlayerSearchServiceTest {

    private List<PlayerRecord> list;
    private Map<String, PlayerRecord> map;
    private PlayerSearchService service;

    @BeforeEach  // S'executa ABANS de cada test — estat fresc cada vegada
    void setUp() {
        list = PlayerDataGenerator.generateList(1_000);
        map = PlayerDataGenerator.toMap(list);
        service = new PlayerSearchService();
    }

    @Test  // Marca el mètode com a test
    void linearSearch_findsExistingPlayer() {
        PlayerRecord result = service.searchLinear(list, "P-500");
        assertNotNull(result);                    // No és null
        assertEquals("P-500", result.playerId());  // Té l'ID correcte
    }
}
```

**Conceptes clau:**
- **`@BeforeEach`** — Prepara l'estat abans de cada test. Cada test arrenca net (recorda: immutabilitat i tests, de dimarts).
- **`@Test`** — Diu a JUnit que aquest mètode és un test.
- **`assertEquals(expected, actual)`** — Verifica que dos valors són iguals. Si no ho són, el test falla amb un missatge clar.
- **`assertNotNull(value)`** — Verifica que el valor no és null.
- **`assertNull(value)`** — Verifica que el valor és null.
- **`assertTrue(condition)`** — Verifica que una condició és certa.

### Estructura d'un Test: Given / When / Then

Tot test segueix el mateix patró de tres passos, conegut com **Given / When / Then**:

- **GIVEN** (Donat) — L'estat inicial. Quines dades o objectes necessites preparats abans d'actuar. Exemple: "donada una llista amb 1.000 jugadors".
- **WHEN** (Quan) — L'acció que vols provar. La crida al mètode que estàs testejant. Exemple: "quan busco el playerId P-500 amb cerca lineal".
- **THEN** (Llavors) — La verificació. Què esperes que passi després de l'acció. Exemple: "llavors el resultat no és null i té el playerId correcte".

```java
@Test
void linearSearch_findsPlayerById() {
    // GIVEN: una llista amb 1.000 jugadors
    List<PlayerRecord> players = PlayerDataGenerator.generateList(1_000);

    // WHEN: busquem un playerId que existeix
    PlayerRecord result = PlayerSearchService.searchLinear(players, "P-500");

    // THEN: el resultat no és null i té el playerId correcte
    assertNotNull(result);
    assertEquals("P-500", result.playerId());
}
```

No cal escriure els comentaris `// GIVEN`, `// WHEN`, `// THEN` a cada test — però sí que has de pensar en aquests tres passos cada vegada. Si no saps què posar a un dels tres, és que el test no està ben definit.

### Noms de Test: Què Verifiquen, No Com

Un bon nom de test descriu el comportament que verifica:
```java
// BÉ — descriu el comportament
void searchLinear_returnsNull_whenPlayerIdDoesNotExist()
void searchByKey_returnsSameResult_asLinearSearch()

// MALAMENT — descriu la implementació
void testSearchMethod()
void test1()
```

> **Lectura recomanada (opcional, no bloquejant):**
> - [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/) — Seccions d'Assercions i cicle de vida

---

### Clean Code: Escriure Codi que Altres Puguin Llegir

Ja has escrit codi tota la setmana (`PlayerSearchService`, generadors de dades, tests). Abans de la PR d'avui, tres regles que has d'aplicar **des d'ara i sempre**, no només quan algú t'ho recordi:

#### 1. Noms Significatius

```java
// ❌ Críptic — què fa això sense llegir el cos?
public PlayerRecord get(List<PlayerRecord> l, String id) { ... }

// ✅ Descriptiu — s'entén sense llegir el cos del mètode
public PlayerRecord searchLinear(List<PlayerRecord> players, String playerId) { ... }
```

#### 2. Funcions Petites (Una Sola Responsabilitat)

```java
// ❌ Una funció que genera dades I calcula temps I compara resultats
public void runBenchmark() {
    var players = PlayerDataGenerator.generateList(100_000);
    long start = System.nanoTime();
    // ... cerca lineal ...
    long linearTime = System.nanoTime() - start;
    // ... cerca per mapa ...
    // ... comparació i print ...
}

// ✅ Cada mètode fa una sola cosa, es pot testejar per separat
private long measureLinearSearch(List<PlayerRecord> players, String id) { ... }
private long measureHashSearch(Map<String, PlayerRecord> map, String id) { ... }
```

#### 3. Early Return (Evitar Niuament Excessiu)

```java
// ❌ Piràmide de la mort
public PlayerRecord searchLinear(List<PlayerRecord> players, String id) {
    if (players != null) {
        if (!players.isEmpty()) {
            for (var p : players) {
                if (p.playerId().equals(id)) {
                    return p;
                }
            }
        }
    }
    return null;
}

// ✅ Early return — el cas d'error surt de seguida, sense nesting
public PlayerRecord searchLinear(List<PlayerRecord> players, String id) {
    if (players == null || players.isEmpty()) return null;

    for (var p : players) {
        if (p.playerId().equals(id)) return p;
    }
    return null;
}
```

**Aquestes tres regles formen part del checklist de PR d'avui i de totes les properes setmanes.** No és teoria per llegir un cop — és com escrius codi cada dia, des d'ara.

---

## Activitat

### 1. Crear els tests — `PlayerSearchServiceTest` (45 min)

Crea el fitxer:
```
backend-java/src/test/java/com/esportspulse/engine/search/PlayerSearchServiceTest.java
```

Comença copiant aquest test resolt. Llegeix-lo línia per línia, executa'l amb `mvn test`, i verifica que passa (verd). Un cop l'entenguis, escriu els tests 1, 2 i 3 tu sol seguint el mateix patró.

**Test 0 (resolt): `linearSearch_findsPlayerById()`**

```java
package com.esportspulse.engine.search;

import com.esportspulse.engine.data.PlayerDataGenerator;
import com.esportspulse.engine.model.PlayerRecord;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import static org.junit.jupiter.api.Assertions.*;

import java.util.List;
import java.util.Map;

class PlayerSearchServiceTest {

    private List<PlayerRecord> list;
    private Map<String, PlayerRecord> map;

    @BeforeEach  // S'executa ABANS de cada test — cada test arrenca amb dades fresques
    void setUp() {
        list = PlayerDataGenerator.generateList(1_000);
        map = PlayerDataGenerator.toMap(list);
    }

    @Test  // Marca aquest mètode com a test — JUnit el trobarà i l'executarà
    void linearSearch_findsPlayerById() {
        // GIVEN: una llista amb 1.000 jugadors (creada a setUp)
        // WHEN: busquem un playerId que sabem que existeix
        PlayerRecord result = PlayerSearchService.searchLinear(list, "P-500");

        // THEN: el resultat no és null i té el playerId correcte
        assertNotNull(result);                     // Si és null, el test falla aquí
        assertEquals("P-500", result.playerId());   // Si el playerId no coincideix, falla aquí
    }
}
```

Executa'l: `mvn test`. Has de veure `Tests run: 1, Failures: 0`. Si falla, llegeix el missatge de JUnit — et diu exactament què esperava i què ha rebut.

Ara escriu els tests següents **dins la mateixa classe** (afegeix-los sota el test 0):

**Test 1: `bothMethods_findSamePlayer()`**
- Genera 1.000 jugadors (no cal 100.000 per tests de correcció — han de ser ràpids)
- Busca un playerId que existeix (ex: `"P-500"`)
- Verifica amb `assertEquals` que ambdós mètodes retornen un objecte amb el mateix `playerId`

**Test 2: `bothMethods_returnNull_forMissingId()`**
- Busca un playerId que no existeix (ex: `"P-999999"`)
- Verifica amb `assertNull` que ambdós retornen `null`

**Test 3: `hashMap_isAtLeast10xFaster()`**
- Genera 100.000 jugadors (aquí sí cal volum per veure la diferència)
- Warm-up: 50 cerques de cada sense mesurar
- Mesura 1.000 cerques de cada amb `System.nanoTime()`
- `assertTrue(linearTime > hashTime * 10)` — verifica que la lineal és almenys 10x més lenta

Executar un test de rendiment dins JUnit ensenya que un test pot verificar **propietats de rendiment**, no només de correcció.

### 2. Executar els tests (10 min)

Des de la terminal:
```bash
mvn test
```

Has de veure:
```
Tests run: 3, Failures: 0, Errors: 0
BUILD SUCCESS
```

Si algun test falla, llegeix el missatge d'error de JUnit — et diu exactament què esperava i què ha rebut.

### 3. Commit final i Pull Request (20 min)

Fes el commit dels tests:
```bash
git add backend-java/src/test/java/com/esportspulse/engine/search/PlayerSearchServiceTest.java
git commit -m "test(java): add search service tests (correctness + performance)"
```

Puja la branca a GitHub:
```bash
git push -u origin feature/week1-benchmarking
```

Crea la Pull Request a GitHub:
- **Títol:** `feat: O(1) vs O(n) benchmarking with JUnit5 tests`
- **Descripció:** Explica breument què fa el codi (2-3 frases), el ratio obtingut al benchmark, i que els tests passen.

Fusiona la PR a `main` des de GitHub (botó "Merge pull request").

### 4. Tancar la branca (5 min)

Un cop fusionada:
```bash
git checkout main
git pull origin main
git branch -d feature/week1-benchmarking  # Elimina la branca local
```

---

## Checklist de Lliurament

- [ ] `mvn test` passa amb 3 tests verds (correcció + null + rendiment)
- [ ] El test de rendiment verifica que HashMap és almenys 10x més ràpid
- [ ] Noms clars, funcions petites i sense nesting excessiu (early return)
- [ ] Commit amb format Conventional Commits
- [ ] Pull Request creada, revisada i fusionada a `main`
- [ ] Branca `feature/week1-benchmarking` eliminada
