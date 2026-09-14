# Setmana 2 — Divendres: Tests Unitaris, Integracio i Pull Request

## Objectiu del Dia

Escriure tests unitaris per totes les classes creades aquesta setmana: `GameRecord`, `PlayerRecord`, `InMemoryGameRepository`, `InMemoryPlayerRepository` i `GameManagementService`. Tancar la setmana amb tots els tests verds, un commit final i una Pull Request a GitHub. Al final del dia tens 12+ tests verds, codi fusionat a `main`, i la branca tancada.

---

## Teoria

### Testejar Models Immutables

Testejar un record immutable es mes senzill que testejar un objecte mutable. Per que? Perque no tens estat canviant — el que crees es el que tens.

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;
import static org.junit.jupiter.api.Assertions.*;

class GameRecordTest {

    // Test 1: Verificar que el compact constructor rebutja dades invalides
    // assertThrows comprova que es llança l'excepcio correcta
    @Test
    void constructor_throwsException_whenAppIdIsNull() {
        // arrange + act + assert en una sola linia
        // Intentem crear un GameRecord amb appId null — HA de petar
        assertThrows(IllegalArgumentException.class, () ->
            new GameRecord(null, "Test", BigDecimal.ZERO, 0L)
        );
    }

    @Test
    void constructor_throwsException_whenPriceIsNegative() {
        // Preu negatiu no te sentit — el compact constructor ho valida
        assertThrows(IllegalArgumentException.class, () ->
            new GameRecord("APP-1", "Test", BigDecimal.valueOf(-1), 0L)
        );
    }

    // Test 2: Parameteritzat — prova isPopular amb multiples valors
    // En lloc d'escriure 3 tests iguals amb valors diferents,
    // @ParameterizedTest executa el MATEIX test amb cada valor
    @ParameterizedTest
    @ValueSource(longs = {100_001L, 500_000L, 1_000_000L, 5_000_000L})
    void isPopular_returnsTrue_whenPlayersAboveThreshold(long players) {
        GameRecord game = new GameRecord("APP-1", "Test", BigDecimal.ZERO, players);
        // assertTrue: verifica que la condicio es certa
        assertTrue(game.isPopular(),
            "Joc amb " + players + " jugadors hauria de ser popular");
    }

    @ParameterizedTest
    @ValueSource(longs = {0L, 50_000L, 99_999L, 100_000L})
    void isPopular_returnsFalse_whenPlayersBelowOrEqualThreshold(long players) {
        GameRecord game = new GameRecord("APP-1", "Test", BigDecimal.ZERO, players);
        // assertFalse: verifica que la condicio es falsa
        assertFalse(game.isPopular(),
            "Joc amb " + players + " jugadors NO hauria de ser popular");
    }

    // Test 3: Verificar que discountedPrice NO muta l'original
    @Test
    void discountedPrice_doesNotMutateOriginal() {
        BigDecimal originalPrice = BigDecimal.valueOf(59.99);
        GameRecord original = new GameRecord("APP-1", "Test", originalPrice, 0L);

        // Creem un record descomptat
        GameRecord discounted = original.discountedPrice(0.50);

        // L'original NO ha canviat — segueix amb el preu original
        assertEquals(originalPrice, original.price(),
            "El preu original no hauria de canviar despres de discountedPrice");

        // El descomptat te el preu reduit
        // compareTo == 0 vol dir "son iguals" (BigDecimal no usa equals per comparar valors)
        assertTrue(discounted.price().compareTo(BigDecimal.valueOf(29.995)) == 0,
            "El preu descomptat hauria de ser la meitat");

        // Son objectes DIFERENTS
        assertNotSame(original, discounted,
            "discountedPrice ha de retornar un objecte NOU, no el mateix");
    }
}
```

### Testejar Repositoris

```java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class InMemoryGameRepositoryTest {

    private InMemoryGameRepository repo;
    private GameRecord sampleGame;

    // S'executa ABANS de cada test — cada test comenca amb un repo buit
    // Aixo garanteix que els tests son independents entre ells
    @BeforeEach
    void setUp() {
        repo = new InMemoryGameRepository();
        sampleGame = new GameRecord("APP-1", "LoL", BigDecimal.ZERO, 5_000_000L);
    }

    @Test
    void save_andFindById_returnsSameRecord() {
        // Guardem un joc
        repo.save(sampleGame);

        // El busquem per ID
        Optional<GameRecord> found = repo.findById("APP-1");

        // Ha de ser present i ser el MATEIX objecte
        assertTrue(found.isPresent(), "El joc guardat hauria d'existir");
        assertEquals(sampleGame, found.get(),
            "El joc trobat hauria de ser igual al guardat");
    }

    @Test
    void findById_returnsEmpty_whenNotFound() {
        // Busquem un ID que no existeix — ha de retornar Optional buit
        Optional<GameRecord> result = repo.findById("APP-999");

        assertTrue(result.isEmpty(),
            "findById hauria de retornar buit per un ID que no existeix");
    }

    @Test
    void findAll_returnsImmutableCopy() {
        repo.save(sampleGame);

        // Obtenim la llista
        List<GameRecord> all = repo.findAll();

        // Intentem modificar-la — HA de petar
        // Aixo verifica que findAll retorna una copia immutable
        assertThrows(UnsupportedOperationException.class, () ->
            all.add(new GameRecord("APP-2", "Hack", BigDecimal.ZERO, 0L))
        );
    }

    @Test
    void delete_removesRecord() {
        repo.save(sampleGame);

        // Eliminem el joc
        repo.delete("APP-1");

        // Ja no hauria d'existir
        assertTrue(repo.findById("APP-1").isEmpty(),
            "El joc eliminat no hauria d'existir");
    }
}
```

### Testejar el Servei amb Dependencies Injectades

Aqui es on DIP brilla: podem testejar `GameManagementService` amb un `InMemoryGameRepository` real, sense necessitat de mocks complexos. El servei no sap ni li importa que el repo es in-memory.

```java
class GameManagementServiceTest {

    private InMemoryGameRepository repo;
    private GameRecordFactory factory;
    private GameManagementService service;

    @BeforeEach
    void setUp() {
        // Creem dependencies reals — no calen mocks perque InMemory es lleuger
        repo = new InMemoryGameRepository();
        factory = new GameRecordFactory();
        // Injectem per constructor — exactament com a produccio, pero amb InMemory
        service = new GameManagementService(repo, factory);
    }

    @Test
    void registerGame_savesGameToRepository() {
        // Registrem un joc a traves del servei
        service.registerGame("APP-1", "League of Legends", BigDecimal.ZERO);

        // Verifiquem que el repo el te
        Optional<GameRecord> found = repo.findById("APP-1");
        assertTrue(found.isPresent(), "El joc registrat hauria d'existir al repo");
        assertEquals("League of Legends", found.get().title());
    }

    @Test
    void getPopularGames_filtersCorrectly() {
        // Registrem jocs — pero registerGame usa createDefault que posa 0 jugadors
        // Necessitem guardar directament al repo per controlar activePlayerCount
        repo.save(new GameRecord("APP-1", "LoL", BigDecimal.ZERO, 5_000_000L));
        repo.save(new GameRecord("APP-2", "Indie", BigDecimal.valueOf(19.99), 500L));
        repo.save(new GameRecord("APP-3", "Valorant", BigDecimal.ZERO, 200_000L));

        // Filtrem populars (> 100K jugadors)
        List<GameRecord> popular = service.getPopularGames();

        // Nomes LoL i Valorant son populars
        assertEquals(2, popular.size(),
            "Nomes 2 jocs tenen > 100K jugadors");
    }

    @Test
    void findGame_returnsEmpty_whenGameDoesNotExist() {
        Optional<GameRecord> result = service.findGame("APP-999");
        assertTrue(result.isEmpty());
    }
}
```

### Tests Python amb pytest

```python
import pytest
from decimal import Decimal
from game_record import GameRecord
from player_record import PlayerRecord
from game_repository import InMemoryGameRepository

# pytest detecta automaticament funcions que comencen per test_

def test_game_record_immutability():
    """Verificar que no es pot mutar un GameRecord."""
    game = GameRecord("APP-1", "LoL", Decimal("0"), 5_000_000)
    # Intentar canviar un atribut HA de llançar FrozenInstanceError
    with pytest.raises(AttributeError):
        game.price = Decimal("100")

def test_game_record_is_popular():
    """Verificar que isPopular funciona correctament."""
    popular = GameRecord("APP-1", "LoL", Decimal("0"), 5_000_000)
    not_popular = GameRecord("APP-2", "Indie", Decimal("19.99"), 500)
    assert popular.is_popular() is True
    assert not_popular.is_popular() is False

def test_game_record_validation():
    """Verificar que el constructor rebutja dades invalides."""
    with pytest.raises(ValueError):
        GameRecord("", "Bad", Decimal("0"), 0)  # app_id buit

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
    repo = InMemoryGameRepository()
    game = GameRecord("APP-1", "LoL", Decimal("0"), 5_000_000)
    repo.save(game)
    found = repo.find_by_id("APP-1")
    assert found == game

def test_repository_find_returns_none_when_not_found():
    """Verificar que find retorna None si no existeix."""
    repo = InMemoryGameRepository()
    assert repo.find_by_id("APP-999") is None
```

---

## Activitat

### 1. Tests Java: `GameRecordTest` (20 min)

Crea el fitxer:
```
backend-java/src/test/java/com/esportspulse/engine/model/GameRecordTest.java
```

Tests minims:
- Compact constructor rebutja `appId` null i preu negatiu (2 tests)
- `isPopular()` parameteritzat amb valors per sobre i per sota del llindar (2 tests)
- `discountedPrice()` no muta l'original (1 test)

### 2. Tests Java: `InMemoryGameRepositoryTest` (20 min)

Crea el fitxer:
```
backend-java/src/test/java/com/esportspulse/engine/repository/InMemoryGameRepositoryTest.java
```

Tests minims:
- `save` + `findById` retorna el mateix record (1 test)
- `findById` retorna `Optional.empty()` per ID inexistent (1 test)
- `findAll()` retorna copia immutable (1 test)
- `delete` elimina correctament (1 test)

### 3. Tests Java: `GameManagementServiceTest` (20 min)

Crea el fitxer:
```
backend-java/src/test/java/com/esportspulse/engine/service/GameManagementServiceTest.java
```

Tests minims:
- `registerGame` guarda al repositori (1 test)
- `getPopularGames` filtra correctament (1 test)
- `findGame` retorna buit per ID inexistent (1 test)

### 4. Tests Python amb pytest (20 min)

Crea el fitxer:
```
ai-python/src/test_models.py
```

Tests minims:
- Immutabilitat de GameRecord (1 test)
- `is_popular()` correcte (1 test)
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

- [ ] `GameRecordTest`: 5 tests (validacio constructor x2, isPopular x2, discountedPrice immutable)
- [ ] `InMemoryGameRepositoryTest`: 4 tests (save+find, find buit, findAll immutable, delete)
- [ ] `GameManagementServiceTest`: 3 tests (register, getPopular, findGame buit)
- [ ] Python `test_models.py`: 7 tests (immutabilitat, is_popular, validacio, is_veteran, level_up, repo save+find, repo find None)
- [ ] `mvn test` passa amb 12+ tests verds, 0 errors
- [ ] `pytest` passa amb 7 tests verds
- [ ] Commit amb format Conventional Commits
- [ ] Pull Request creada, revisada i fusionada a `main`
- [ ] Branca `feature/week2-oop-solid` eliminada
