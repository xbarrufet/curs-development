# Setmana 6 - Teoria: Testing, Mocks i Qualitat

## 1. Per Què Testejar? El Cost del Bug

Un bug no costa el mateix en tot moment. Com més tard el detectes, més car és de corregir:

```
Cost de corregir un bug segons quan es detecta:

Fase                   Cost relatiu       Qui el troba
─────────────────────────────────────────────────────────
Mentre escrius codi    1x                 Tu, al moment
En un test unitari     2x                 El test suite
En code review         5x                 Un company
Al CI (integració)     10x                El pipeline
A staging/QA           50x                El tester
A producció            100x-1000x         L'usuari final

        Cost
          ▲
   1000x  │                                          ╱
          │                                        ╱
    100x  │                                     ╱
          │                                  ╱
     50x  │                              ╱
          │                          ╱
     10x  │                     ╱
          │                ╱
      5x  │           ╱
      2x  │       ╱
      1x  │  ╱
          └──────────────────────────────────────────→ Temps
           Local  Test   Review  CI   Staging  Prod
```

**Exemple concret amb GamePulse:**

Imagina un bug a `getPopularGames()` que usa `>` en lloc de `>=`:

```java
// Bug: retorna jocs amb MÉS de 100.000 jugadors
// Hauria de retornar jocs amb 100.000 O MÉS
return repository.findAll().stream()
    .filter(g -> g.getActivePlayerCount() > 100_000)  // Bug: > en lloc de >=
    .toList();
```

- **Detectat en un test unitari:** 2 minuts. Canvies `>` per `>=`, test passa, commit.
- **Detectat en producció:** Un analista es queixa que falten jocs al dashboard. L'equip de suport investiga. Un dev busca al codi. Es fa hotfix, deploy, validació. Total: hores o dies de treball de múltiples persones.

El cost no és només el temps de fix. És el temps de **trobar** el bug, **comunicar-lo**, **prioritzar-lo**, i **validar** que el fix no trenca res més.

---

## 2. Anatomia d'un Test: AAA / GWT

Tot bon test segueix una estructura de 3 fases. Hi ha dues formes d'expressar-ho:

### AAA: Arrange - Act - Assert

```java
@Test
void shouldReturnPopularGamesOnly() {
    // ARRANGE — Prepara l'escenari
    GameRecord popular = new GameRecord("APP-1", "League of Legends", 
        BigDecimal.ZERO, 5_000_000L);
    GameRecord unpopular = new GameRecord("APP-2", "Indie Gem", 
        new BigDecimal("9.99"), 500L);
    when(repository.findAll()).thenReturn(List.of(popular, unpopular));

    // ACT — Executa l'acció que vols testejar
    List<GameRecord> result = service.getPopularGames();

    // ASSERT — Verifica el resultat
    assertThat(result).containsExactly(popular);
    assertThat(result).doesNotContain(unpopular);
}
```

### GWT: Given - When - Then (estil BDD)

```java
@Test
@DisplayName("Donat un mix de jocs populars i no populars, " +
             "quan demano els populars, " +
             "llavors només retorna els de >100k jugadors")
void shouldReturnPopularGamesOnly() {
    // GIVEN — Donat un estat inicial
    GameRecord popular = new GameRecord("APP-1", "League of Legends", 
        BigDecimal.ZERO, 5_000_000L);
    GameRecord unpopular = new GameRecord("APP-2", "Indie Gem", 
        new BigDecimal("9.99"), 500L);
    when(repository.findAll()).thenReturn(List.of(popular, unpopular));

    // WHEN — Quan passa una acció
    List<GameRecord> result = service.getPopularGames();

    // THEN — Llavors espero un resultat
    assertThat(result).containsExactly(popular);
}
```

### Quan usar cadascun?

AAA és el format estàndard per a tests tècnics. GWT va bé quan el test descriu **comportament de negoci** que un product owner podria entendre.

En la pràctica, la majoria d'equips usen AAA amb noms de test descriptius. L'important no és el format sinó que cada test tingui les 3 fases clarament separades.

**Anti-patró:** El test que barreja Act i Assert en un bucle, o que fa múltiples Acts. Si el test falla, no saps quina acció ha causat l'error.

---

## 3. La Piràmide de Tests

La piràmide de tests defineix quants tests de cada tipus hauries de tenir:

```
                    ╱╲
                   ╱  ╲
                  ╱ E2E╲          Pocs (5-10%)
                 ╱______╲         Lents, fràgils, costosos
                ╱        ╲        Ex: Controller → Service → Repo → H2
               ╱Integration╲     Moderats (15-25%)
              ╱______________╲    Velocitat mitjana
             ╱                ╲   Ex: Service + Repository + H2
            ╱    Unit Tests    ╲  Molts (70-80%)
           ╱____________________╲ Ràpids, estables, barats
                                  Ex: Service amb mocks
```

### Cada nivell amb GamePulse

**Unit Tests (base de la piràmide):**
```java
// Testeja NOMÉS la lògica de GameManagementService
// Repository és un mock — no toca BD
@ExtendWith(MockitoExtension.class)
class GameManagementServiceTest {
    @Mock GameJpaRepository repository;
    @InjectMocks GameManagementService service;

    @Test
    void shouldFilterPopularGames() {
        when(repository.findAll()).thenReturn(List.of(
            new GameRecord("APP-1", "LoL", BigDecimal.ZERO, 5_000_000L),
            new GameRecord("APP-2", "Indie", BigDecimal.TEN, 100L)
        ));
        
        List<GameRecord> result = service.getPopularGames();
        
        assertThat(result).hasSize(1);
        assertThat(result.get(0).getTitle()).isEqualTo("LoL");
    }
}
// Temps: ~50ms — no arrenca Spring, no toca BD
```

**Integration Tests (mig de la piràmide):**
```java
// Testeja Service + Repository + H2 junts
@SpringBootTest
class GameManagementServiceIT {
    @Autowired GameManagementService service;
    @Autowired GameJpaRepository repository;

    @BeforeEach
    void setUp() {
        repository.deleteAll();
    }

    @Test
    void shouldRegisterAndFindGame() {
        service.registerGame("APP-1", "League of Legends", BigDecimal.ZERO);
        
        Optional<GameRecord> found = repository.findById("APP-1");
        
        assertThat(found).isPresent();
        assertThat(found.get().getTitle()).isEqualTo("League of Legends");
    }
}
// Temps: ~2-5s — arrenca Spring context, crea BD H2
```

**E2E Tests (punta de la piràmide):**
```java
// Testeja tot el stack: HTTP → Controller → Service → Repository → H2
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class GameControllerE2ETest {
    @Autowired TestRestTemplate restTemplate;

    @Test
    void shouldCreateAndRetrieveGame() {
        GameDTO dto = new GameDTO("APP-1", "League of Legends", BigDecimal.ZERO);
        restTemplate.postForEntity("/api/games", dto, Void.class);
        
        ResponseEntity<GameDTO> response = 
            restTemplate.getForEntity("/api/games/APP-1", GameDTO.class);
        
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody().title()).isEqualTo("League of Legends");
    }
}
// Temps: ~5-10s — arrenca servidor HTTP complet
// Nota: GameController és de S7. Aquí és per il·lustrar el concepte.
```

### Speed vs Confidence

| Tipus | Velocitat | Confiança | Cost manteniment | Quantitat |
|-------|-----------|-----------|------------------|-----------|
| Unit | Molt ràpid (~ms) | Lògica aïllada | Baix | Molts (70-80%) |
| Integration | Mitjà (~s) | Capes juntes | Mitjà | Moderats (15-25%) |
| E2E | Lent (~s a min) | Sistema complet | Alt | Pocs (5-10%) |

**Per què la piràmide?** Si tots els tests fossin E2E, un canvi petit podria trigar 30 minuts a validar. Si tots fossin unitaris, podries tenir tots verds i el sistema no funcionar perquè les capes no connecten bé. La piràmide equilibra velocitat i confiança.

---

## 4. JUnit 5: Més Enllà de @Test

### @ParameterizedTest amb @CsvSource

Ideal per testejar la mateixa lògica amb molts inputs:

```java
@ParameterizedTest(name = "Preu {0} hauria de ser vàlid={1}")
@CsvSource({
    "0.00,   true",     // Free-to-play
    "14.99,  true",     // Preu normal
    "59.99,  true",     // AAA
    "-1.00,  false",    // Negatiu: invàlid
    "-0.01,  false"     // Just sota zero: invàlid
})
void shouldValidateGamePrice(BigDecimal price, boolean expectedValid) {
    if (expectedValid) {
        assertDoesNotThrow(() -> service.registerGame("APP-1", "Test Game", price));
    } else {
        assertThrows(IllegalArgumentException.class, 
            () -> service.registerGame("APP-1", "Test Game", price));
    }
}
```

Output de Maven:
```
shouldValidateGamePrice(BigDecimal, boolean)
  ├─ Preu 0.00 hauria de ser vàlid=true       ✓
  ├─ Preu 14.99 hauria de ser vàlid=true      ✓
  ├─ Preu 59.99 hauria de ser vàlid=true      ✓
  ├─ Preu -1.00 hauria de ser vàlid=false     ✓
  └─ Preu -0.01 hauria de ser vàlid=false     ✓
```

### @MethodSource per inputs complexos

Quan `@CsvSource` no és suficient (objectes complexos, llistes):

```java
@ParameterizedTest
@MethodSource("provideGamesForPopularityCheck")
void shouldCorrectlyClassifyPopularity(GameRecord game, boolean expectedPopular) {
    when(repository.findAll()).thenReturn(List.of(game));
    
    List<GameRecord> popular = service.getPopularGames();
    
    if (expectedPopular) {
        assertThat(popular).contains(game);
    } else {
        assertThat(popular).isEmpty();
    }
}

static Stream<Arguments> provideGamesForPopularityCheck() {
    return Stream.of(
        Arguments.of(
            new GameRecord("APP-1", "LoL", BigDecimal.ZERO, 5_000_000L), true),
        Arguments.of(
            new GameRecord("APP-2", "Indie", BigDecimal.TEN, 500L), false),
        Arguments.of(
            new GameRecord("APP-3", "Edge Case", BigDecimal.ZERO, 100_000L), false)
            // Atenció: 100_000 és exactament el límit. > o >=?
    );
}
```

### @Nested per organitzar

Agrupa tests per escenari. Fa que l'output de Maven sigui una documentació:

```java
@DisplayName("GameManagementService")
class GameManagementServiceTest {
    
    @Nested
    @DisplayName("quan registra un joc")
    class WhenRegistering {
        @Test
        @DisplayName("hauria de guardar-lo al repository")
        void shouldSaveToRepository() { ... }

        @Test
        @DisplayName("hauria de rebutjar un títol buit")
        void shouldRejectEmptyTitle() { ... }

        @Test
        @DisplayName("hauria de rebutjar un preu negatiu")
        void shouldRejectNegativePrice() { ... }
    }

    @Nested
    @DisplayName("quan busca jocs populars")
    class WhenSearchingPopular {
        @Test
        @DisplayName("hauria de retornar només jocs amb >100k jugadors")
        void shouldReturnOnlyPopular() { ... }

        @Test
        @DisplayName("hauria de retornar llista buida si cap compleix")
        void shouldReturnEmptyIfNonePopular() { ... }
    }
}
```

Output:
```
GameManagementService
  quan registra un joc
    ✓ hauria de guardar-lo al repository
    ✓ hauria de rebutjar un títol buit
    ✓ hauria de rebutjar un preu negatiu
  quan busca jocs populars
    ✓ hauria de retornar només jocs amb >100k jugadors
    ✓ hauria de retornar llista buida si cap compleix
```

Això és **documentació viva**: si el test passa, el comportament descrit existeix.

### Cicle de vida

```
┌──────────────────────────────────────────────────┐
│ @BeforeAll (1 cop, static)                       │
│   Ex: carregar fitxer config, crear connexió     │
│                                                  │
│   ┌────────────────────────────────────────────┐ │
│   │ @BeforeEach (abans de cada @Test)          │ │
│   │   Ex: crear service, resetar mocks         │ │
│   │                                            │ │
│   │   @Test shouldDoSomething()                │ │
│   │                                            │ │
│   │ @AfterEach (després de cada @Test)         │ │
│   │   Ex: tancar recursos temporals            │ │
│   └────────────────────────────────────────────┘ │
│                                                  │
│   ┌────────────────────────────────────────────┐ │
│   │ @BeforeEach                                │ │
│   │   @Test shouldDoSomethingElse()            │ │
│   │ @AfterEach                                 │ │
│   └────────────────────────────────────────────┘ │
│                                                  │
│ @AfterAll (1 cop, static)                        │
│   Ex: tancar connexió compartida                 │
└──────────────────────────────────────────────────┘
```

---

## 5. Mockito: L'Art de Simular Dependències

### Per Què Mock?

`GameManagementService` depèn de `GameJpaRepository`. En un unit test, no volem arrancar Spring ni H2 — volem testejar **només** la lògica del servei:

```
Sense mock (integration test):        Amb mock (unit test):

GameManagementService                 GameManagementService
        │                                     │
        ▼                                     ▼
GameJpaRepository                     Mock<GameJpaRepository>
        │                                     │
        ▼                                     ▼
H2 Database                           Respostes predefinides
(arrenca Spring, crea BD,             (instantani, 0 I/O,
 executa SQL, 2-5 segons)              50 mil·lisegons)
```

El mock intercepta les crides al repository i retorna el que tu li dius. Així pots testejar la lògica del servei en aïllament.

### when/thenReturn: Definir Comportament

```java
@ExtendWith(MockitoExtension.class)
class GameManagementServiceTest {
    @Mock GameJpaRepository repository;
    @InjectMocks GameManagementService service;

    @Test
    void shouldReturnGameById() {
        // Definim: quan algú cridi findById("APP-1"), retorna aquest game
        GameRecord game = new GameRecord("APP-1", "LoL", BigDecimal.ZERO, 5_000_000L);
        when(repository.findById("APP-1")).thenReturn(Optional.of(game));

        // Ara cridem el servei — que internament crida repository.findById
        Optional<GameRecord> result = service.findById("APP-1");

        assertThat(result).isPresent();
        assertThat(result.get().getTitle()).isEqualTo("LoL");
    }

    @Test
    void shouldThrowWhenGameNotFound() {
        when(repository.findById("NOPE")).thenReturn(Optional.empty());

        assertThrows(GameNotFoundException.class, 
            () -> service.findByIdOrThrow("NOPE"));
    }
}
```

### verify: Confirmar Interaccions

A vegades no importa el return, sinó que el servei **ha cridat** el que tocava:

```java
@Test
void shouldSaveGameToRepository() {
    service.registerGame("APP-1", "League of Legends", BigDecimal.ZERO);

    // Verifica que el servei ha cridat save() exactament 1 cop
    verify(repository).save(any(GameRecord.class));
}

@Test
void shouldNotDeleteWhenRegistering() {
    service.registerGame("APP-1", "League of Legends", BigDecimal.ZERO);

    // Verifica que el servei NO ha cridat delete()
    verify(repository, never()).delete(any());
}
```

### ArgumentCaptor: Inspecció Detallada

Quan vols verificar **què exactament** s'ha passat al mock:

```java
@Test
void shouldCreateGameRecordWithCorrectFields() {
    service.registerGame("APP-1", "League of Legends", new BigDecimal("0.00"));

    ArgumentCaptor<GameRecord> captor = ArgumentCaptor.forClass(GameRecord.class);
    verify(repository).save(captor.capture());

    GameRecord saved = captor.getValue();
    assertThat(saved.getAppId()).isEqualTo("APP-1");
    assertThat(saved.getTitle()).isEqualTo("League of Legends");
    assertThat(saved.getPrice()).isEqualByComparingTo(BigDecimal.ZERO);
    assertThat(saved.getActivePlayerCount()).isEqualTo(0L);  // Nou joc = 0 jugadors
}
```

### Anti-Patrons amb Mocks

**1. Testejar el mock (Snippet 4 de S4):**
```java
// MAL: Això testeja que Mockito funciona, no que el servei funciona
@Test
void badTest() {
    List<GameRecord> games = List.of(someGame);
    when(repository.findAll()).thenReturn(games);
    
    List<GameRecord> result = service.getAllGames();
    
    assertEquals(games, result);  // Obvi! Has dit al mock que retorni 'games'!
}

// BÉ: Testeja la LÒGICA del servei (el filtratge)
@Test
void goodTest() {
    when(repository.findAll()).thenReturn(List.of(popularGame, unpopularGame));
    
    List<GameRecord> result = service.getPopularGames();
    
    assertThat(result).containsOnly(popularGame);  // El servei filtra!
}
```

**2. Over-mocking (mock everything, test nothing):**
```java
// MAL: Mocking el propi servei
@Mock GameManagementService service;  // NO! Vols testejar AQUEST servei!

// BÉ: Mock les dependències, no el subject under test
@Mock GameJpaRepository repository;
@InjectMocks GameManagementService service;  // service és REAL
```

**3. Verificar implementació interna:**
```java
// MAL: Si refactoritzes el servei, el test trenca
verify(repository).findAll();  // Què importa si ha cridat findAll?
                                // El que importa és el RESULTAT

// BÉ: Verifica comportament, no implementació
assertThat(result).containsOnly(popularGame);
```

---

## 6. @DataJpaTest vs @SpringBootTest vs @WebMvcTest

Spring Boot ofereix "test slices" que carreguen només la part del context que necessites. Escollir el correcte accelera els tests:

```
┌─────────────────────────────────────────────────────────────────┐
│                    @SpringBootTest                              │
│  Carrega TOT: Controllers, Services, Repositories, Config      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                 @WebMvcTest                              │   │
│  │  Carrega: Controllers + MVC config                      │   │
│  │  NO carrega: Services, Repositories                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                 @DataJpaTest                             │   │
│  │  Carrega: Repositories + JPA + BD                       │   │
│  │  NO carrega: Controllers, Services                      │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

| Anotació | Què carrega | Velocitat | Cas d'ús |
|----------|------------|-----------|----------|
| `@DataJpaTest` | JPA, Repositories, H2 | Ràpid (~1s) | Testejar queries i persistència |
| `@WebMvcTest` | Controllers, MVC | Ràpid (~1s) | Testejar endpoints (S7) |
| `@SpringBootTest` | Tot el context | Lent (~3-5s) | Tests d'integració complets |

### @DataJpaTest: Testejar GameJpaRepository (connexió S5)

```java
@DataJpaTest
class GameJpaRepositoryTest {

    @Autowired
    private GameJpaRepository repository;

    @Autowired
    private TestEntityManager entityManager;

    @BeforeEach
    void setUp() {
        entityManager.persist(new GameRecord("APP-1", "League of Legends", 
            BigDecimal.ZERO, 5_000_000L));
        entityManager.persist(new GameRecord("APP-2", "Stardew Valley", 
            new BigDecimal("14.99"), 90_000L));
        entityManager.persist(new GameRecord("APP-3", "Elden Ring", 
            new BigDecimal("49.99"), 300_000L));
        entityManager.flush();
    }

    @Test
    void shouldFindByTitleContaining() {
        List<GameRecord> result = repository.findByTitleContaining("Legend");
        
        assertThat(result).hasSize(1);
        assertThat(result.get(0).getAppId()).isEqualTo("APP-1");
    }

    @Test
    void shouldFindGamesWithManyPlayers() {
        List<GameRecord> result = 
            repository.findByActivePlayerCountGreaterThan(100_000L);
        
        assertThat(result).hasSize(2);  // LoL (5M) + Elden Ring (300k)
    }

    @Test
    void shouldReturnEmptyForNonExistentTitle() {
        List<GameRecord> result = repository.findByTitleContaining("Cyberpunk");
        
        assertThat(result).isEmpty();
    }
}
```

`@DataJpaTest` fa rollback automàtic de cada test: cada `@Test` comença amb una BD neta.

### @SpringBootTest: Test d'integració complet

```java
@SpringBootTest
class GameManagementServiceIT {

    @Autowired
    private GameManagementService service;

    @Autowired
    private GameJpaRepository repository;

    @BeforeEach
    void setUp() {
        repository.deleteAll();
    }

    @Test
    void shouldRegisterAndRetrieveGame() {
        service.registerGame("APP-1", "League of Legends", BigDecimal.ZERO);

        List<GameRecord> all = service.getAllGames();

        assertThat(all).hasSize(1);
        assertThat(all.get(0).getTitle()).isEqualTo("League of Legends");
    }

    @Test
    void shouldFindPopularGamesAfterRegistration() {
        service.registerGame("APP-1", "LoL", BigDecimal.ZERO);
        // Simulem que el joc ja té jugadors (normalment vindria d'un update)
        GameRecord game = repository.findById("APP-1").orElseThrow();
        game.setActivePlayerCount(5_000_000L);
        repository.save(game);

        List<GameRecord> popular = service.getPopularGames();

        assertThat(popular).hasSize(1);
    }
}
```

### @WebMvcTest: Preview de S7

A S7 crearem `GameController` amb endpoints REST. Aleshores usarem `@WebMvcTest`:

```java
// Això és S7 — aquí és només un preview conceptual
@WebMvcTest(GameController.class)
class GameControllerTest {
    @Autowired MockMvc mockMvc;
    @MockBean GameManagementService service;

    @Test
    void shouldReturnGameById() throws Exception {
        when(service.findByIdOrThrow("APP-1"))
            .thenReturn(new GameRecord("APP-1", "LoL", BigDecimal.ZERO, 5_000_000L));

        mockMvc.perform(get("/api/games/APP-1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.title").value("LoL"));
    }
}
```

---

## 7. pytest: El Mirror Python

### Fixtures

Les fixtures de pytest són l'equivalent de `@BeforeEach` + factory methods, però més flexibles:

```python
# conftest.py — fixtures compartides per a tots els tests
import pytest
from game_repository import SqliteGameRepository, GameRecord

@pytest.fixture
def repo():
    """Crea un repository amb BD en memòria. Cada test en rep un de nou."""
    return SqliteGameRepository(":memory:")

@pytest.fixture
def sample_game():
    return GameRecord(
        app_id="APP-1",
        title="League of Legends",
        price=0.0,
        active_player_count=5_000_000
    )

@pytest.fixture
def populated_repo(repo):
    """Repository amb 3 jocs predefinits."""
    games = [
        GameRecord("APP-1", "League of Legends", 0.0, 5_000_000),
        GameRecord("APP-2", "Stardew Valley", 14.99, 90_000),
        GameRecord("APP-3", "Elden Ring", 49.99, 300_000),
    ]
    for game in games:
        repo.save(game)
    return repo
```

```python
# test_game_repository.py

def test_save_and_find(repo, sample_game):
    repo.save(sample_game)
    found = repo.find_by_id("APP-1")
    assert found == sample_game

def test_find_nonexistent(repo):
    assert repo.find_by_id("NOPE") is None

def test_find_all_returns_all(populated_repo):
    games = populated_repo.find_all()
    assert len(games) == 3

def test_find_all_empty(repo):
    assert repo.find_all() == []
```

### Parametrize

```python
@pytest.mark.parametrize("price,expected_valid", [
    (0.0, True),       # Free-to-play
    (14.99, True),     # Normal
    (59.99, True),     # AAA
    (-1.0, False),     # Negatiu
    (-0.01, False),    # Just sota zero
])
def test_validate_game_price(price, expected_valid):
    if expected_valid:
        # No hauria de llançar excepció
        validate_game_price(price)
    else:
        with pytest.raises(ValueError):
            validate_game_price(price)
```

### Monkeypatch: L'Equivalent de Mockito

`monkeypatch` reemplaça objectes, funcions o variables d'entorn durant un test:

```python
def test_handles_database_error(monkeypatch):
    """Verifica que el servei gestiona errors de BD correctament."""
    repo = SqliteGameRepository(":memory:")

    def broken_save(game):
        raise sqlite3.OperationalError("database is locked")

    monkeypatch.setattr(repo, "save", broken_save)

    with pytest.raises(sqlite3.OperationalError):
        repo.save(GameRecord("APP-1", "Test", 0.0, 0))
```

```python
def test_uses_environment_variable(monkeypatch):
    """Verifica que el servei llegeix la config correcta."""
    monkeypatch.setenv("GAMEPULSE_DB_PATH", "/tmp/test.db")
    
    config = load_config()
    
    assert config.db_path == "/tmp/test.db"
```

### Comparativa JUnit 5 vs pytest

| Concepte | JUnit 5 | pytest |
|----------|---------|--------|
| Executar tests | `mvn test` | `pytest` |
| Setup per test | `@BeforeEach` | `@pytest.fixture` |
| Setup global | `@BeforeAll` (static) | `conftest.py` + `scope="session"` |
| Múltiples inputs | `@ParameterizedTest` + `@CsvSource` | `@pytest.mark.parametrize` |
| Mock objectes | `@Mock` + `when().thenReturn()` | `monkeypatch.setattr()` |
| Mock framework | Mockito (`@ExtendWith`) | `unittest.mock` o `monkeypatch` |
| Grups de tests | `@Nested` classes | Classes dins el fitxer |
| Nom llegible | `@DisplayName("...")` | Nom de la funció (PEP) |
| Assercions | AssertJ: `assertThat(x).isEqualTo(y)` | `assert x == y` |
| Expected exception | `assertThrows(Ex.class, () -> ...)` | `with pytest.raises(Ex):` |

---

## 8. Coverage: Què Mesura i Què NO Mesura

### Line Coverage i Branch Coverage

```java
public BigDecimal calculateDiscount(GameRecord game) {
    if (game.getPrice().compareTo(new BigDecimal("50")) > 0) {  // Branca A
        return game.getPrice().multiply(new BigDecimal("0.10"));  // Línia 1
    } else if (game.getActivePlayerCount() > 1_000_000) {        // Branca B
        return game.getPrice().multiply(new BigDecimal("0.05"));  // Línia 2
    }
    return BigDecimal.ZERO;                                       // Línia 3
}
```

- **Line coverage:** Percentatge de línies executades pels tests.
- **Branch coverage:** Percentatge de branques (if/else) executades.

Un test que només passa un joc de 60 euros tindria:
- Line coverage: 66% (Línia 1 i 3, però no Línia 2)
- Branch coverage: 33% (Branca A, però no B ni el cas per defecte)

### Què Coverage SÍ Mesura

Coverage garanteix que el codi **s'ha executat** durant els tests. Si una línia té 0% coverage, cap test l'ha executat mai. Això vol dir que:
- Si aquella línia té un bug, cap test el detectarà.
- Si algú la canvia, cap test fallarà.

### Què Coverage NO Mesura

Coverage no garanteix que el codi sigui **correcte**. Exemple:

```java
@Test
void terribleTestWithFullCoverage() {
    // Executa totes les línies... però no asserteja res!
    service.getPopularGames();
    service.registerGame("APP-1", "Test", BigDecimal.ZERO);
    service.calculateDiscount(someGame);
    // Coverage: 100%. Valor del test: 0%.
}
```

Això passa més del que sembla. El "test que no asserteja" és un dels anti-patrons més perillosos perquè inflà la mètrica sense aportar seguretat.

### Mutation Testing: La Prova de Foc

El mutation testing canvia el codi i verifica que els tests fallen:

```
Codi original:                      Mutant:
if (players > 100_000)      →      if (players >= 100_000)
                                    
Si el test NO falla, el test és feble.
```

Manualment (exercici de dijous):
1. Canvia `>` per `>=` a `getPopularGames()`.
2. Elimina un `null` check a `findByIdOrThrow()`.
3. Canvia `return BigDecimal.ZERO` per `return null` a `calculateDiscount()`.
4. Executa `mvn test`. Cada mutació hauria de fer fallar almenys un test.
5. Si alguna passa desapercebuda, el test suite té un forat.

### Configuració JaCoCo (Maven)

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution>
            <id>prepare-agent</id>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals><goal>report</goal></goals>
        </execution>
        <execution>
            <id>check</id>
            <goals><goal>check</goal></goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.70</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

Amb això, `mvn verify` fallarà si el coverage baixa del 70%.

### Configuració pytest-cov

```ini
# pytest.ini (o pyproject.toml)
[pytest]
addopts = --cov=gamepulse --cov-report=html --cov-report=term-missing
```

```bash
# Execució manual
pytest --cov=gamepulse --cov-fail-under=70

# Output:
# Name                          Stmts   Miss  Cover
# -------------------------------------------------
# gamepulse/game_repository.py     45      3    93%
# gamepulse/game_service.py        30      8    73%
# -------------------------------------------------
# TOTAL                            75     11    85%
```

---

## 9. Anti-Patrons de Testing

### 1. Test que no asserteja res d'útil

```java
// MAL: Executa codi però no verifica res
@Test
void testRegisterGame() {
    service.registerGame("APP-1", "LoL", BigDecimal.ZERO);
    // ... i ja? Què estem comprovant?
}

// BÉ: Verifica el comportament esperat
@Test
void shouldSaveGameWhenRegistering() {
    service.registerGame("APP-1", "LoL", BigDecimal.ZERO);
    
    verify(repository).save(argThat(game -> 
        game.getAppId().equals("APP-1") && 
        game.getTitle().equals("LoL")
    ));
}
```

**Pista:** Si pots eliminar l'`assert` i el test segueix passant, el test no serveix de res.

### 2. Test que testeja el mock

```java
// MAL: Estàs testejant que Mockito funciona
@Test
void testGetAllGames() {
    List<GameRecord> games = List.of(someGame);
    when(repository.findAll()).thenReturn(games);
    
    assertEquals(games, service.getAllGames());
    // Literalment li has dit que retorni 'games' i ara comproves que retorna 'games'
}

// BÉ: Testeja lògica que transforma el resultat
@Test
void shouldReturnOnlyPopularGames() {
    when(repository.findAll()).thenReturn(List.of(popularGame, unpopularGame));
    
    List<GameRecord> result = service.getPopularGames();
    
    assertThat(result).containsOnly(popularGame);
    // Aquí SÍ testem la lògica de filtratge del servei
}
```

Recorda el Snippet 4 de S4: si el test no testeja transformació, validació, o lògica de negoci, probablement testeja el mock.

### 3. Test fràgil

```java
// MAL: Depèn de l'ordre dels resultats
@Test
void testFindAll() {
    List<GameRecord> result = service.getAllGames();
    assertEquals("League of Legends", result.get(0).getTitle());  // Per què el primer?
    assertEquals("Dota 2", result.get(1).getTitle());  // I si la BD retorna en ordre diferent?
}

// BÉ: Verifica contingut sense dependre de l'ordre
@Test
void shouldReturnAllRegisteredGames() {
    List<GameRecord> result = service.getAllGames();
    assertThat(result)
        .extracting(GameRecord::getTitle)
        .containsExactlyInAnyOrder("League of Legends", "Dota 2");
}
```

```java
// MAL: Depèn del temps
@Test
void testTimestamp() {
    GameRecord game = service.registerGame("APP-1", "LoL", BigDecimal.ZERO);
    assertEquals(LocalDateTime.now(), game.getCreatedAt());  // Pot fallar per mil·lisegons!
}

// BÉ: Tolerància temporal
@Test
void shouldSetCreatedAtToCurrentTime() {
    LocalDateTime before = LocalDateTime.now();
    GameRecord game = service.registerGame("APP-1", "LoL", BigDecimal.ZERO);
    LocalDateTime after = LocalDateTime.now();
    
    assertThat(game.getCreatedAt()).isBetween(before, after);
}
```

### 4. Over-mocking

```java
// MAL: Tot mockejat, no testem res real
@Mock GameJpaRepository repository;
@Mock GameValidator validator;
@Mock GameMapper mapper;
@Mock PriceCalculator calculator;
@Mock NotificationService notifier;
@InjectMocks GameManagementService service;

@Test
void testRegister() {
    when(validator.validate(any())).thenReturn(true);
    when(mapper.toEntity(any())).thenReturn(someEntity);
    when(calculator.calculate(any())).thenReturn(BigDecimal.TEN);
    when(repository.save(any())).thenReturn(someEntity);
    // Has definit tot el comportament manualment.
    // El test passa sempre, independentment de la implementació real.
}
```

**Regla:** Si tens >3 mocks en un test, potser el test hauria de ser d'integració, o el servei fa massa coses (responsabilitat excessiva, SRP de S2).

### 5. Test massa gran

```java
// MAL: Fa 5 coses; si falla, quina ha fallat?
@Test
void testEverything() {
    service.registerGame("APP-1", "LoL", BigDecimal.ZERO);
    service.registerGame("APP-2", "Dota", BigDecimal.ZERO);
    List<GameRecord> all = service.getAllGames();
    assertEquals(2, all.size());
    List<GameRecord> popular = service.getPopularGames();
    assertEquals(0, popular.size());
    service.updatePlayerCount("APP-1", 5_000_000L);
    popular = service.getPopularGames();
    assertEquals(1, popular.size());
    service.deleteGame("APP-1");
    all = service.getAllGames();
    assertEquals(1, all.size());
}

// BÉ: Un test, un concepte
@Test void shouldRegisterGame() { ... }
@Test void shouldReturnAllGames() { ... }
@Test void shouldReturnNoPopularGamesInitially() { ... }
@Test void shouldReturnPopularAfterPlayerUpdate() { ... }
@Test void shouldRemoveDeletedGame() { ... }
```

**Regla:** Un test hauria de tenir un sol motiu per fallar. Si falla, el nom del test t'ha de dir què ha anat malament sense obrir el codi.

---

## 10. Aplicació a GamePulse: La Suite Completa

Al final de S6 (i del Bloc 1), GamePulse hauria de tenir aquesta estructura de tests:

```
gamepulse-engine/
├── src/
│   ├── main/java/com/gamepulse/
│   │   ├── GameRecord.java              (@Entity, de S5)
│   │   ├── GameJpaRepository.java       (JpaRepository, de S5)
│   │   ├── GameManagementService.java   (Service, de S2-S5)
│   │   └── GameDTO.java                 (record, de S2)
│   └── test/java/com/gamepulse/
│       ├── unit/
│       │   └── GameManagementServiceTest.java    ← @Mock + @InjectMocks
│       │       ├── shouldRegisterGame()
│       │       ├── shouldRejectNegativePrice()
│       │       ├── shouldReturnPopularGamesOnly()
│       │       ├── shouldReturnEmptyWhenNoPopular()
│       │       ├── shouldThrowWhenGameNotFound()
│       │       └── shouldValidatePrice() [parametrized]
│       ├── repository/
│       │   └── GameJpaRepositoryTest.java        ← @DataJpaTest
│       │       ├── shouldFindByTitleContaining()
│       │       ├── shouldFindByActivePlayerCountGreaterThan()
│       │       └── shouldReturnEmptyForNonExistent()
│       └── integration/
│           └── GameManagementServiceIT.java       ← @SpringBootTest
│               ├── shouldRegisterAndRetrieveGame()
│               └── shouldFindPopularGamesEndToEnd()
│
├── python/
│   ├── gamepulse/
│   │   ├── game_repository.py            (SqliteGameRepository, de S5)
│   │   └── game_service.py
│   ├── tests/
│   │   ├── conftest.py                    (fixtures compartides)
│   │   ├── test_game_repository.py        (fixtures + parametrize)
│   │   └── test_game_service.py           (monkeypatch)
│   └── pytest.ini                         (coverage config)
│
├── pom.xml                                (JaCoCo plugin amb 70% mínim)
└── .github/workflows/ci.yml              (mvn verify + pytest --cov)
```

### Com CI Executa els Tests

```yaml
# .github/workflows/ci.yml (actualitzat des de S4)
name: CI
on: [push, pull_request]
jobs:
  java:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      - run: mvn verify           # Tests + JaCoCo coverage check (70%)
      - run: mvn checkstyle:check  # Format del codi (S4)

  python:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install pytest pytest-cov
      - run: pytest --cov=gamepulse --cov-fail-under=70
```

Si un test falla o el coverage baixa del 70%, el CI es posa vermell i no es pot fer merge.

---

## Resum

| Concepte | Key Takeaway |
|----------|--------------|
| **Cost del bug** | Cada fase on avança un bug multiplica el cost de corregir-lo |
| **AAA / GWT** | Tot test té Arrange-Act-Assert; GWT per tests de negoci |
| **Piràmide de tests** | Molts unitaris (ràpids), alguns d'integració, pocs E2E |
| **JUnit 5** | @Nested, @ParameterizedTest, @DisplayName → tests com a documentació |
| **Mockito** | Mock les dependències, no el subject under test; verify comportament |
| **Test slices** | @DataJpaTest (BD), @WebMvcTest (HTTP), @SpringBootTest (tot) |
| **pytest** | Fixtures, parametrize, monkeypatch — mirror de JUnit 5 en Python |
| **Coverage** | Mesura execució, no correcció; 70% mínim, mai l'objectiu final |
| **Mutation testing** | Canvia el codi, mira si els tests ho detecten — la prova de foc |
| **Anti-patrons** | Test sense assert, test del mock, test fràgil, over-mocking, test gegant |

**Objectiu setmana:** Que cada línia de GamePulse tingui un test que la protegeix, que el CI ho verifiqui, i que sàpigues distingir un test que aporta valor d'un que només fa bulto.
