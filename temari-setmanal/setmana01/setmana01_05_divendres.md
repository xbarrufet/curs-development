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

class GameSearchServiceTest {

    private List<GameRecord> list;
    private Map<String, GameRecord> map;
    private GameSearchService service;

    @BeforeEach  // S'executa ABANS de cada test — estat fresc cada vegada
    void setUp() {
        list = GameDataGenerator.generateList(1_000);
        map = GameDataGenerator.toMap(list);
        service = new GameSearchService();
    }

    @Test  // Marca el mètode com a test
    void linearSearch_findsExistingGame() {
        GameRecord result = service.searchLinear(list, "APP-500");
        assertNotNull(result);                    // No és null
        assertEquals("APP-500", result.appId());  // Té l'ID correcte
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

### Noms de Test: Què Verifiquen, No Com

Un bon nom de test descriu el comportament que verifica:
```java
// BÉ — descriu el comportament
void searchLinear_returnsNull_whenIdDoesNotExist()
void searchByKey_returnsSameResult_asLinearSearch()

// MALAMENT — descriu la implementació
void testSearchMethod()
void test1()
```

> **Lectura recomanada (opcional, no bloquejant):**
> - [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/) — Seccions d'Assercions i cicle de vida
> - [Conventional Commits](https://www.conventionalcommits.org/)

---

## Activitat

### 1. Crear els tests — `GameSearchServiceTest` (45 min)

Crea el fitxer:
```
backend-java/src/test/java/com/esportspulse/engine/search/GameSearchServiceTest.java
```

Escriu els tests següents:

**Test 1: Ambdós mètodes troben el mateix objecte**
- Genera 1.000 jocs (no cal 100.000 per tests de correcció — han de ser ràpids)
- Busca un appId que existeix (ex: `"APP-500"`)
- Verifica amb `assertEquals` que ambdós mètodes retornen un objecte amb el mateix `appId`

**Test 2: Ambdós mètodes retornen null si l'ID no existeix**
- Busca un appId que no existeix (ex: `"APP-999999"`)
- Verifica amb `assertNull` que ambdós retornen `null`

**Test 3: HashMap és almenys 10x més ràpid que la cerca lineal**
- Genera 100.000 jocs (aquí sí cal volum per veure la diferència)
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
git add backend-java/src/test/java/com/esportspulse/engine/search/GameSearchServiceTest.java
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
- [ ] Commit amb format Conventional Commits
- [ ] Pull Request creada, revisada i fusionada a `main`
- [ ] Branca `feature/week1-benchmarking` eliminada
