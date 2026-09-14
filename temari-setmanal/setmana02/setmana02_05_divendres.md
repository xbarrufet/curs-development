# Setmana 2 — Divendres: Tests Unitaris, Integracio i Pull Request

## Objectiu del Dia

Escriure tests unitaris per totes les classes creades aquesta setmana: `ChampionRecord`, `PlayerRecord`, `InMemoryChampionRepository`, `InMemoryPlayerRepository` i `ChampionManagementService`. Tancar la setmana amb tots els tests verds, un commit final i una Pull Request a GitHub. Al final del dia tens 12+ tests verds, codi fusionat a `main`, i la branca tancada.

---

## Teoria

### Testejar Models Immutables

Testejar un record immutable es mes senzill que testejar un objecte mutable. Per que? Perque no tens estat canviant — el que crees es el que tens.

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class ChampionRecordTest {

    // Test 1: Verificar que el compact constructor rebutja dades invalides
    // assertThrows comprova que es llanca l'excepcio correcta
    @Test
    void constructor_throwsException_whenChampionIdIsNull() {
        // arrange + act + assert en una sola linia
        // Intentem crear un ChampionRecord amb championId null — HA de petar
        assertThrows(IllegalArgumentException.class, () ->
            new ChampionRecord(null, "Test", 50.0, 5.0)
        );
    }

    @Test
    void constructor_throwsException_whenWinRateIsNegative() {
        // WinRate negatiu no te sentit — el compact constructor ho valida
        assertThrows(IllegalArgumentException.class, () ->
            new ChampionRecord("CHAMP-1", "Test", -1.0, 5.0)
        );
    }

    // Test 2: isMeta requereix DUES condicions: winRate > 52.0 AND pickRate > 10.0
    // Usem tests regulars per cobrir els casos significatius
    @Test
    void isMeta_returnsTrue_whenWinRateAndPickRateAboveThreshold() {
        ChampionRecord champion = new ChampionRecord("CHAMP-1", "Ahri", 53.0, 12.0);
        // assertTrue: verifica que la condicio es certa
        assertTrue(champion.isMeta(),
            "Campio amb winRate 53.0 i pickRate 12.0 hauria de ser meta");
    }

    @Test
    void isMeta_returnsFalse_whenBelowThreshold() {
        // winRate alta pero pickRate baixa — NO es meta
        ChampionRecord lowPick = new ChampionRecord("CHAMP-1", "Aurelion Sol", 54.0, 3.0);
        assertFalse(lowPick.isMeta(),
            "Campio amb pickRate 3.0 NO hauria de ser meta");

        // pickRate alta pero winRate baixa — NO es meta
        ChampionRecord lowWin = new ChampionRecord("CHAMP-2", "Yasuo", 49.0, 15.0);
        assertFalse(lowWin.isMeta(),
            "Campio amb winRate 49.0 NO hauria de ser meta");

        // Ambdues al llindar exacte — NO es meta (cal superar, no igualar)
        ChampionRecord atThreshold = new ChampionRecord("CHAMP-3", "Ezreal", 52.0, 10.0);
        assertFalse(atThreshold.isMeta(),
            "Campio amb winRate 52.0 i pickRate 10.0 NO hauria de ser meta (cal superar el llindar)");
    }

    // Test 3: Verificar que withPatchAdjustment NO muta l'original
    @Test
    void withPatchAdjustment_doesNotMutateOriginal() {
        ChampionRecord original = new ChampionRecord("CHAMP-1", "Ahri", 55.0, 12.0);

        // Creem un record ajustat pel patch
        ChampionRecord adjusted = original.withPatchAdjustment(-2.5);

        // L'original NO ha canviat — segueix amb el winRate original
        assertEquals(55.0, original.winRate(),
            "El winRate original no hauria de canviar despres de withPatchAdjustment");

        // L'ajustat te el winRate reduit
        assertEquals(52.5, adjusted.winRate(),
            "El winRate ajustat hauria de ser 52.5");

        // Son objectes DIFERENTS
        assertNotSame(original, adjusted,
            "withPatchAdjustment ha de retornar un objecte NOU, no el mateix");
    }
}
```

### Testejar Repositoris

```java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class InMemoryChampionRepositoryTest {

    private InMemoryChampionRepository repo;
    private ChampionRecord sampleChampion;

    // S'executa ABANS de cada test — cada test comenca amb un repo buit
    // Aixo garanteix que els tests son independents entre ells
    @BeforeEach
    void setUp() {
        repo = new InMemoryChampionRepository();
        sampleChampion = new ChampionRecord("CHAMP-1", "Ahri", 52.3, 8.1);
    }

    @Test
    void save_andFindById_returnsSameRecord() {
        // Guardem un campio
        repo.save(sampleChampion);

        // El busquem per ID
        Optional<ChampionRecord> found = repo.findById("CHAMP-1");

        // Ha de ser present i ser el MATEIX objecte
        assertTrue(found.isPresent(), "El campio guardat hauria d'existir");
        assertEquals(sampleChampion, found.get(),
            "El campio trobat hauria de ser igual al guardat");
    }

    @Test
    void findById_returnsEmpty_whenNotFound() {
        // Busquem un ID que no existeix — ha de retornar Optional buit
        Optional<ChampionRecord> result = repo.findById("CHAMP-999");

        assertTrue(result.isEmpty(),
            "findById hauria de retornar buit per un ID que no existeix");
    }

    @Test
    void findAll_returnsImmutableCopy() {
        repo.save(sampleChampion);

        // Obtenim la llista
        List<ChampionRecord> all = repo.findAll();

        // Intentem modificar-la — HA de petar
        // Aixo verifica que findAll retorna una copia immutable
        assertThrows(UnsupportedOperationException.class, () ->
            all.add(new ChampionRecord("CHAMP-2", "Zed", 51.0, 7.0))
        );
    }

    @Test
    void delete_removesRecord() {
        repo.save(sampleChampion);

        // Eliminem el campio
        repo.delete("CHAMP-1");

        // Ja no hauria d'existir
        assertTrue(repo.findById("CHAMP-1").isEmpty(),
            "El campio eliminat no hauria d'existir");
    }
}
```

### Testejar el Servei amb Dependencies Injectades

Aqui es on DIP brilla: podem testejar `ChampionManagementService` amb un `InMemoryChampionRepository` real, sense necessitat de mocks complexos. El servei no sap ni li importa que el repo es in-memory.

```java
class ChampionManagementServiceTest {

    private InMemoryChampionRepository repo;
    private ChampionRecordFactory factory;
    private ChampionManagementService service;

    @BeforeEach
    void setUp() {
        // Creem dependencies reals — no calen mocks perque InMemory es lleuger
        repo = new InMemoryChampionRepository();
        factory = new ChampionRecordFactory();
        // Injectem per constructor — exactament com a produccio, pero amb InMemory
        service = new ChampionManagementService(repo, factory);
    }

    @Test
    void registerChampion_savesChampionToRepository() {
        // Registrem un campio a traves del servei
        service.registerChampion("CHAMP-1", "Ahri", 53.0);

        // Verifiquem que el repo el te
        Optional<ChampionRecord> found = repo.findById("CHAMP-1");
        assertTrue(found.isPresent(), "El campio registrat hauria d'existir al repo");
        assertEquals("Ahri", found.get().name());
    }

    @Test
    void getMetaChampions_filtersCorrectly() {
        // Guardem campions directament al repo per controlar winRate i pickRate
        repo.save(new ChampionRecord("CHAMP-1", "Ahri", 53.0, 12.0));
        repo.save(new ChampionRecord("CHAMP-2", "Yasuo", 49.1, 15.0));
        repo.save(new ChampionRecord("CHAMP-3", "Jinx", 54.0, 11.0));

        // Filtrem campions meta (winRate > 52.0 AND pickRate > 10.0)
        List<ChampionRecord> meta = service.getMetaChampions();

        // Nomes Ahri i Jinx son meta (Yasuo te winRate massa baixa)
        assertEquals(2, meta.size(),
            "Nomes 2 campions tenen winRate > 52.0 i pickRate > 10.0");
    }

    @Test
    void findChampion_returnsEmpty_whenChampionDoesNotExist() {
        Optional<ChampionRecord> result = service.findChampion("CHAMP-999");
        assertTrue(result.isEmpty());
    }
}
```

### Tests Python amb pytest

```python
import pytest
from champion_record import ChampionRecord
from player_record import PlayerRecord
from champion_repository import InMemoryChampionRepository

# pytest detecta automaticament funcions que comencen per test_

def test_champion_record_immutability():
    """Verificar que no es pot mutar un ChampionRecord."""
    champion = ChampionRecord("CHAMP-1", "Ahri", 52.3, 8.1)
    # Intentar canviar un atribut HA de llancar FrozenInstanceError
    with pytest.raises(AttributeError):
        champion.win_rate = 99.0

def test_champion_record_is_meta():
    """Verificar que is_meta funciona correctament."""
    meta = ChampionRecord("CHAMP-1", "Ahri", 53.0, 12.0)
    not_meta = ChampionRecord("CHAMP-2", "Yasuo", 49.1, 15.0)
    assert meta.is_meta() is True
    assert not_meta.is_meta() is False

def test_champion_record_validation():
    """Verificar que el constructor rebutja dades invalides."""
    with pytest.raises(ValueError):
        ChampionRecord("", "Bad", 50.0, 5.0)  # champion_id buit

def test_player_record_is_veteran():
    """Verificar que is_veteran funciona correctament."""
    veteran = PlayerRecord("P-1", "Faker", 50, 3500.0)
    newbie = PlayerRecord("P-2", "Rookie", 1, 50.0)
    assert veteran.is_veteran() is True
    assert newbie.is_veteran() is False

def test_player_record_level_up_immutable():
    """Verificar que level_up retorna un objecte NOU sense mutar l'original."""
    player = PlayerRecord("P-1", "Faker", 50, 3500.0)
    leveled = player.level_up()
    assert player.level == 50      # Original no ha canviat
    assert leveled.level == 51     # Nou objecte amb nivell incrementat
    assert player is not leveled   # Son objectes DIFERENTS

def test_repository_save_and_find():
    """Verificar que save + find retorna el mateix objecte."""
    repo = InMemoryChampionRepository()
    champion = ChampionRecord("CHAMP-1", "Ahri", 52.3, 8.1)
    repo.save(champion)
    found = repo.find_by_id("CHAMP-1")
    assert found == champion

def test_repository_find_returns_none_when_not_found():
    """Verificar que find retorna None si no existeix."""
    repo = InMemoryChampionRepository()
    assert repo.find_by_id("CHAMP-999") is None
```

---

## Activitat

### 1. Tests Java: `ChampionRecordTest` (20 min)

Crea el fitxer:
```
backend-java/src/test/java/com/esportspulse/engine/model/ChampionRecordTest.java
```

Tests minims:
- Compact constructor rebutja `championId` null i winRate negatiu (2 tests)
- `isMeta()` amb valors per sobre i per sota del llindar (2 tests)
- `withPatchAdjustment()` no muta l'original (1 test)

### 2. Tests Java: `InMemoryChampionRepositoryTest` (20 min)

Crea el fitxer:
```
backend-java/src/test/java/com/esportspulse/engine/repository/InMemoryChampionRepositoryTest.java
```

Tests minims:
- `save` + `findById` retorna el mateix record (1 test)
- `findById` retorna `Optional.empty()` per ID inexistent (1 test)
- `findAll()` retorna copia immutable (1 test)
- `delete` elimina correctament (1 test)

### 3. Tests Java: `ChampionManagementServiceTest` (20 min)

Crea el fitxer:
```
backend-java/src/test/java/com/esportspulse/engine/service/ChampionManagementServiceTest.java
```

Tests minims:
- `registerChampion` guarda al repositori (1 test)
- `getMetaChampions` filtra correctament (1 test)
- `findChampion` retorna buit per ID inexistent (1 test)

### 4. Tests Python amb pytest (20 min)

Crea el fitxer:
```
ai-python/src/test_models.py
```

Tests minims:
- Immutabilitat de ChampionRecord (1 test)
- `is_meta()` correcte (1 test)
- Validacio rebutja dades invalides (1 test)
- `is_veteran()` correcte (1 test)
- `level_up()` immutable (1 test)
- Repository save + find (1 test)
- Repository find retorna None quan no existeix (1 test)

### 5. Executar tots els tests (10 min)

**Java:**
```bash
# Executa tots els tests Java — han de sortir tots verds
mvn test
```

Has de veure:
```
Tests run: 12+, Failures: 0, Errors: 0
BUILD SUCCESS
```

**Python:**
```bash
# Executa tots els tests Python — han de sortir tots verds
cd ai-python/src
python -m pytest test_models.py -v
```

Has de veure:
```
7 passed
```

Si algun test falla, llegeix el missatge d'error — et dira exactament que esperava i que ha rebut.

### 6. Commit final i Pull Request (15 min)

```bash
# Afegeix tots els tests
git add backend-java/src/test/java/
git add ai-python/src/test_models.py
git commit -m "test: add unit tests for models, repositories and service (Java + Python)"
```

Puja la branca a GitHub:
```bash
git push -u origin feature/week2-oop-solid
```

Crea la Pull Request a GitHub:
- **Titol:** `feat: SOLID principles, immutable models, Repository pattern (Java + Python)`
- **Descripcio:** Explica breument:
  - Models immutables amb Java records i Python dataclasses
  - Repository pattern amb interficies i implementacio in-memory
  - Service layer amb Dependency Injection
  - 12+ tests unitaris (Java JUnit + Python pytest)

Fusiona la PR a `main` des de GitHub (boto "Merge pull request").

### 7. Tancar la branca (5 min)

```bash
git checkout main
git pull origin main
git branch -d feature/week2-oop-solid
```

---

## Checklist de Lliurament

- [ ] `ChampionRecordTest`: 5 tests (validacio constructor x2, isMeta x2, withPatchAdjustment immutable)
- [ ] `InMemoryChampionRepositoryTest`: 4 tests (save+find, find buit, findAll immutable, delete)
- [ ] `ChampionManagementServiceTest`: 3 tests (register, getMetaChampions, findChampion buit)
- [ ] Python `test_models.py`: 7 tests (immutabilitat, is_meta, validacio, is_veteran, level_up, repo save+find, repo find None)
- [ ] `mvn test` passa amb 12+ tests verds, 0 errors
- [ ] `pytest` passa amb 7 tests verds
- [ ] Commit amb format Conventional Commits
- [ ] Pull Request creada, revisada i fusionada a `main`
- [ ] Branca `feature/week2-oop-solid` eliminada
