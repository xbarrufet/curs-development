# Setmana 6 - Teoria: Testing, Mocks i Qualitat

## 1. Per Que Testejar? El Cost del Bug

Un bug no costa el mateix en tot moment. Com mes tard el detectes, mes car es de corregir:

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

**Exemple concret amb EsportsPulse:**

Imagina un bug a `getMetaChampions()` que usa `>` en lloc de `>=`:

```java
// Bug: retorna champions amb MÉS de 100.000 partides jugades
// Hauria de retornar champions amb 100.000 O MÉS
return repository.findAll().stream()
    .filter(c -> c.getGamesPlayed() > 100_000)  // Bug: > en lloc de >=
    .toList();
```

- **Detectat en un test unitari:** 2 minuts. Canvies `>` per `>=`, test passa, commit.
- **Detectat en producció:** Un analista es queixa que falten champions al dashboard. L'equip de suport investiga. Un dev busca al codi. Es fa hotfix, deploy, validació. Total: hores o dies de treball de múltiples persones.

El cost no és només el temps de fix. És el temps de **trobar** el bug, **comunicar-lo**, **prioritzar-lo**, i **validar** que el fix no trenca res més.

---

## 2. Anatomia d'un Test: AAA / GWT

Tot bon test segueix una estructura de 3 fases. Hi ha dues formes d'expressar-ho:

### AAA: Arrange - Act - Assert

```java
@Test
void shouldReturnMetaChampionsOnly() {
    // ARRANGE — Prepara l'escenari
    ChampionRecord popular = new ChampionRecord("jinx", "Jinx", 
        new BigDecimal("52.30"), 5_000_000L);
    ChampionRecord unpopular = new ChampionRecord("sona", "Sona", 
        new BigDecimal("48.50"), 500L);
    when(repository.findAll()).thenReturn(List.of(popular, unpopular));

    // ACT — Executa l'acció que vols testejar
    List<ChampionRecord> result = service.getMetaChampions();

    // ASSERT — Verifica el resultat
    assertThat(result).containsExactly(popular);
    assertThat(result).doesNotContain(unpopular);
}
```

### GWT: Given - When - Then (estil BDD)

```java
@Test
@DisplayName("Donat un mix de meta champions i no meta, " +
             "quan demano els meta champions, " +
             "llavors només retorna els de >100k partides jugades")
void shouldReturnMetaChampionsOnly() {
    // GIVEN — Donat un estat inicial
    ChampionRecord popular = new ChampionRecord("jinx", "Jinx", 
        new BigDecimal("52.30"), 5_000_000L);
    ChampionRecord unpopular = new ChampionRecord("sona", "Sona", 
        new BigDecimal("48.50"), 500L);
    when(repository.findAll()).thenReturn(List.of(popular, unpopular));

    // WHEN — Quan passa una acció
    List<ChampionRecord> result = service.getMetaChampions();

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

### Cada nivell amb EsportsPulse

**Unit Tests (base de la piràmide):**
```java
// Testeja NOMÉS la lògica de ChampionManagementService
// Repository és un mock — no toca BD
@ExtendWith(MockitoExtension.class)
class ChampionManagementServiceTest {
    @Mock ChampionJpaRepository repository;
    @InjectMocks ChampionManagementService service;

    @Test
    void shouldFilterMetaChampions() {
        when(repository.findAll()).thenReturn(List.of(
            new ChampionRecord("jinx", "Jinx", new BigDecimal("52.30"), 5_000_000L),
            new ChampionRecord("sona", "Sona", new BigDecimal("48.50"), 100L)
        ));
        
        List<ChampionRecord> result = service.getMetaChampions();
        
        assertThat(result).hasSize(1);
        assertThat(result.get(0).getName()).isEqualTo("Jinx");
    }
}
// Temps: ~50ms — no arrenca Spring, no toca BD
```

**Integration Tests (mig de la piràmide):**
```java
// Testeja Service + Repository + H2 junts
@SpringBootTest
class ChampionManagementServiceIT {
    @Autowired ChampionManagementService service;
    @Autowired ChampionJpaRepository repository;

    @BeforeEach
    void setUp() {
        repository.deleteAll();
    }

    @Test
    void shouldRegisterAndFindChampion() {
        service.registerChampion("jinx", "Jinx", new BigDecimal("52.30"));
        
        Optional<ChampionRecord> found = repository.findById("jinx");
        
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("Jinx");
    }
}
// Temps: ~2-5s — arrenca Spring context, crea BD H2
```

**E2E Tests (punta de la piràmide):**
```java
// Testeja tot el stack: HTTP → Controller → Service → Repository → H2
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ChampionControllerE2ETest {
    @Autowired TestRestTemplate restTemplate;

    @Test
    void shouldCreateAndRetrieveChampion() {
        ChampionDTO dto = new ChampionDTO("jinx", "Jinx", new BigDecimal("52.30"));
        restTemplate.postForEntity("/api/champions", dto, Void.class);
        
        ResponseEntity<ChampionDTO> response = 
            restTemplate.getForEntity("/api/champions/jinx", ChampionDTO.class);
        
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody().name()).isEqualTo("Jinx");
    }
}
// Temps: ~5-10s — arrenca servidor HTTP complet
// Nota: ChampionController és de S7. Aquí és per il·lustrar el concepte.
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
@ParameterizedTest(name = "WinRate {0} hauria de ser vàlid={1}")
@CsvSource({
    "0.00,   true",     // WinRate zero (no data)
    "52.30,  true",     // WinRate normal
    "99.99,  true",     // WinRate alt
    "-1.00,  false",    // Negatiu: invàlid
    "101.00, false"     // >100: invàlid
})
void shouldValidateChampionWinRate(BigDecimal winRate, boolean expectedValid) {
    if (expectedValid) {
        assertDoesNotThrow(() -> service.registerChampion("jinx", "Test Champion", winRate));
    } else {
        assertThrows(IllegalArgumentException.class, 
            () -> service.registerChampion("jinx", "Test Champion", winRate));
    }
}
```

Output de Maven:
```
shouldValidateChampionWinRate(BigDecimal, boolean)
  ├─ WinRate 0.00 hauria de ser vàlid=true       ✓
  ├─ WinRate 52.30 hauria de ser vàlid=true      ✓
  ├─ WinRate 99.99 hauria de ser vàlid=true      ✓
  ├─ WinRate -1.00 hauria de ser vàlid=false     ✓
  └─ WinRate 101.00 hauria de ser vàlid=false    ✓
```

### @MethodSource per inputs complexos

Quan `@CsvSource` no és suficient (objectes complexos, llistes):

```java
@ParameterizedTest
@MethodSource("provideChampionsForMetaCheck")
void shouldCorrectlyClassifyMeta(ChampionRecord champion, boolean expectedMeta) {
    when(repository.findAll()).thenReturn(List.of(champion));
    
    List<ChampionRecord> meta = service.getMetaChampions();
    
    if (expectedMeta) {
        assertThat(meta).contains(champion);
    } else {
        assertThat(meta).isEmpty();
    }
}

static Stream<Arguments> provideChampionsForMetaCheck() {
    return Stream.of(
        Arguments.of(
            new ChampionRecord("jinx", "Jinx", new BigDecimal("52.30"), 5_000_000L), true),
        Arguments.of(
            new ChampionRecord("sona", "Sona", new BigDecimal("48.50"), 500L), false),
        Arguments.of(
            new ChampionRecord("lux", "Lux", new BigDecimal("51.20"), 100_000L), false)
            // Atenció: 100_000 és exactament el límit. > o >=?
    );
}
```

### @Nested per organitzar

Agrupa tests per escenari. Fa que l'output de Maven sigui una documentació:

```java
@DisplayName("ChampionManagementService")
class ChampionManagementServiceTest {
    
    @Nested
    @DisplayName("quan registra un champion")
    class WhenRegistering {
        @Test
        @DisplayName("hauria de guardar-lo al repository")
        void shouldSaveToRepository() { ... }

        @Test
        @DisplayName("hauria de rebutjar un champion amb nom buit")
        void shouldRejectEmptyName() { ... }

        @Test
        @DisplayName("hauria de rebutjar un winRate invàlid")
        void shouldRejectInvalidWinRate() { ... }
    }

    @Nested
    @DisplayName("quan busca meta champions")
    class WhenSearchingMeta {
        @Test
        @DisplayName("hauria de retornar només champions amb >100k partides jugades")
        void shouldReturnOnlyMeta() { ... }

        @Test
        @DisplayName("hauria de retornar llista buida si cap compleix")
        void shouldReturnEmptyIfNoneMeta() { ... }
    }
}
```

Output:
```
ChampionManagementService
  quan registra un champion
    ✓ hauria de guardar-lo al repository
    ✓ hauria de rebutjar un champion amb nom buit
    ✓ hauria de rebutjar un winRate invàlid
  quan busca meta champions
    ✓ hauria de retornar només champions amb >100k partides jugades
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

`ChampionManagementService` depèn de `ChampionJpaRepository`. En un unit test, no volem arrancar Spring ni H2 — volem testejar **només** la lògica del servei:

```
Sense mock (integration test):        Amb mock (unit test):

ChampionManagementService             ChampionManagementService
        │                                     │
        ▼                                     ▼
ChampionJpaRepository                 Mock<ChampionJpaRepository>
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
class ChampionManagementServiceTest {
    @Mock ChampionJpaRepository repository;
    @InjectMocks ChampionManagementService service;

    @Test
    void shouldReturnChampionById() {
        // Definim: quan algú cridi findById("jinx"), retorna aquest champion
        ChampionRecord champion = new ChampionRecord("jinx", "Jinx", new BigDecimal("52.30"), 5_000_000L);
        when(repository.findById("jinx")).thenReturn(Optional.of(champion));

        // Ara cridem el servei — que internament crida repository.findById
        Optional<ChampionRecord> result = service.findById("jinx");

        assertThat(result).isPresent();
        assertThat(result.get().getName()).isEqualTo("Jinx");
    }

    @Test
    void shouldThrowWhenChampionNotFound() {
        when(repository.findById("NOPE")).thenReturn(Optional.empty());

        assertThrows(ChampionNotFoundException.class, 
            () -> service.findByIdOrThrow("NOPE"));
    }
}
```

### verify: Confirmar Interaccions

A vegades no importa el return, sinó que el servei **ha cridat** el que tocava:

```java
@Test
void shouldSaveChampionToRepository() {
    service.registerChampion("jinx", "Jinx", new BigDecimal("52.30"));

    // Verifica que el servei ha cridat save() exactament 1 cop
    verify(repository).save(any(ChampionRecord.class));
}

@Test
void shouldNotDeleteWhenRegistering() {
    service.registerChampion("jinx", "Jinx", new BigDecimal("52.30"));

    // Verifica que el servei NO ha cridat delete()
    verify(repository, never()).delete(any());
}
```

### ArgumentCaptor: Inspecció Detallada

Quan vols verificar **què exactament** s'ha passat al mock:

```java
@Test
void shouldCreateChampionRecordWithCorrectFields() {
    service.registerChampion("jinx", "Jinx", new BigDecimal("52.30"));

    ArgumentCaptor<ChampionRecord> captor = ArgumentCaptor.forClass(ChampionRecord.class);
    verify(repository).save(captor.capture());

    ChampionRecord saved = captor.getValue();
    assertThat(saved.getChampionId()).isEqualTo("jinx");
    assertThat(saved.getName()).isEqualTo("Jinx");
    assertThat(saved.getWinRate()).isEqualByComparingTo(new BigDecimal("52.30"));
    assertThat(saved.getGamesPlayed()).isEqualTo(0L);  // Nou champion = 0 partides
}
```

### Anti-Patrons amb Mocks

**1. Testejar el mock (Snippet 4 de S4):**
```java
// MAL: Això testeja que Mockito funciona, no que el servei funciona
@Test
void badTest() {
    List<ChampionRecord> champions = List.of(someChampion);
    when(repository.findAll()).thenReturn(champions);
    
    List<ChampionRecord> result = service.getAllChampions();
    
    assertEquals(champions, result);  // Obvi! Has dit al mock que retorni 'champions'!
}

// BÉ: Testeja la LÒGICA del servei (el filtratge)
@Test
void goodTest() {
    when(repository.findAll()).thenReturn(List.of(popularChampion, unpopularChampion));
    
    List<ChampionRecord> result = service.getMetaChampions();
    
    assertThat(result).containsOnly(popularChampion);  // El servei filtra!
}
```

**2. Over-mocking (mock everything, test nothing):**
```java
// MAL: Mocking el propi servei
@Mock ChampionManagementService service;  // NO! Vols testejar AQUEST servei!

// BÉ: Mock les dependències, no el subject under test
@Mock ChampionJpaRepository repository;
@InjectMocks ChampionManagementService service;  // service és REAL
```

**3. Verificar implementació interna:**
```java
// MAL: Si refactoritzes el servei, el test trenca
verify(repository).findAll();  // Què importa si ha cridat findAll?
                                // El que importa és el RESULTAT

// BÉ: Verifica comportament, no implementació
assertThat(result).containsOnly(popularChampion);
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

### @DataJpaTest: Testejar ChampionJpaRepository (connexió S5)

```java
@DataJpaTest
class ChampionJpaRepositoryTest {

    @Autowired
    private ChampionJpaRepository repository;

    @Autowired
    private TestEntityManager entityManager;

    @BeforeEach
    void setUp() {
        entityManager.persist(new ChampionRecord("jinx", "Jinx", 
            new BigDecimal("52.30"), 5_000_000L));
        entityManager.persist(new ChampionRecord("thresh", "Thresh", 
            new BigDecimal("51.20"), 90_000L));
        entityManager.persist(new ChampionRecord("lux", "Lux", 
            new BigDecimal("54.10"), 300_000L));
        entityManager.flush();
    }

    @Test
    void shouldFindByNameContaining() {
        List<ChampionRecord> result = repository.findByNameContaining("Jin");
        
        assertThat(result).hasSize(1);
        assertThat(result.get(0).getChampionId()).isEqualTo("jinx");
    }

    @Test
    void shouldFindChampionsWithManyGamesPlayed() {
        List<ChampionRecord> result = 
            repository.findByGamesPlayedGreaterThan(100_000L);
        
        assertThat(result).hasSize(2);  // Jinx (5M) + Lux (300k)
    }

    @Test
    void shouldReturnEmptyForNonExistentName() {
        List<ChampionRecord> result = repository.findByNameContaining("Zed");
        
        assertThat(result).isEmpty();
    }
}
```

`@DataJpaTest` fa rollback automàtic de cada test: cada `@Test` comença amb una BD neta.

### @SpringBootTest: Test d'integració complet

```java
@SpringBootTest
class ChampionManagementServiceIT {

    @Autowired
    private ChampionManagementService service;

    @Autowired
    private ChampionJpaRepository repository;

    @BeforeEach
    void setUp() {
        repository.deleteAll();
    }

    @Test
    void shouldRegisterAndRetrieveChampion() {
        service.registerChampion("jinx", "Jinx", new BigDecimal("52.30"));

        List<ChampionRecord> all = service.getAllChampions();

        assertThat(all).hasSize(1);
        assertThat(all.get(0).getName()).isEqualTo("Jinx");
    }

    @Test
    void shouldFindMetaChampionsAfterRegistration() {
        service.registerChampion("jinx", "Jinx", new BigDecimal("52.30"));
        // Simulem que el champion ja té partides jugades (normalment vindria d'un update)
        ChampionRecord champion = repository.findById("jinx").orElseThrow();
        champion.setGamesPlayed(5_000_000L);
        repository.save(champion);

        List<ChampionRecord> meta = service.getMetaChampions();

        assertThat(meta).hasSize(1);
    }
}
```

### @WebMvcTest: Preview de S7

A S7 crearem `ChampionController` amb endpoints REST. Aleshores usarem `@WebMvcTest`:

```java
// Això és S7 — aquí és només un preview conceptual
@WebMvcTest(ChampionController.class)
class ChampionControllerTest {
    @Autowired MockMvc mockMvc;
    @MockBean ChampionManagementService service;

    @Test
    void shouldReturnChampionById() throws Exception {
        when(service.findByIdOrThrow("jinx"))
            .thenReturn(new ChampionRecord("jinx", "Jinx", new BigDecimal("52.30"), 5_000_000L));

        mockMvc.perform(get("/api/champions/jinx"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("Jinx"));
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
from champion_repository import SqliteChampionRepository, ChampionRecord

@pytest.fixture
def repo():
    """Crea un repository amb BD en memòria. Cada test en rep un de nou."""
    return SqliteChampionRepository(":memory:")

@pytest.fixture
def sample_champion():
    return ChampionRecord(
        champion_id="jinx",
        name="Jinx",
        win_rate=52.30,
        games_played=5_000_000
    )

@pytest.fixture
def populated_repo(repo):
    """Repository amb 3 champions predefinits."""
    champions = [
        ChampionRecord("jinx", "Jinx", 52.30, 5_000_000),
        ChampionRecord("thresh", "Thresh", 51.20, 90_000),
        ChampionRecord("lux", "Lux", 54.10, 300_000),
    ]
    for champion in champions:
        repo.save(champion)
    return repo
```

```python
# test_champion_repository.py

def test_save_and_find(repo, sample_champion):
    repo.save(sample_champion)
    found = repo.find_by_id("jinx")
    assert found == sample_champion

def test_find_nonexistent(repo):
    assert repo.find_by_id("NOPE") is None

def test_find_all_returns_all(populated_repo):
    champions = populated_repo.find_all()
    assert len(champions) == 3

def test_find_all_empty(repo):
    assert repo.find_all() == []
```

### Parametrize

```python
@pytest.mark.parametrize("win_rate,expected_valid", [
    (0.0, True),       # WinRate zero (no data)
    (52.30, True),     # Normal
    (99.99, True),     # WinRate alt
    (-1.0, False),     # Negatiu
    (101.0, False),    # >100: invàlid
])
def test_validate_win_rate(win_rate, expected_valid):
    if expected_valid:
        # No hauria de llançar excepció
        validate_win_rate(win_rate)
    else:
        with pytest.raises(ValueError):
            validate_win_rate(win_rate)
```

### Monkeypatch: L'Equivalent de Mockito

`monkeypatch` reemplaça objectes, funcions o variables d'entorn durant un test:

```python
def test_handles_database_error(monkeypatch):
    """Verifica que el servei gestiona errors de BD correctament."""
    repo = SqliteChampionRepository(":memory:")

    def broken_save(champion):
        raise sqlite3.OperationalError("database is locked")

    monkeypatch.setattr(repo, "save", broken_save)

    with pytest.raises(sqlite3.OperationalError):
        repo.save(ChampionRecord("jinx", "Test", 0.0, 0))
```

```python
def test_uses_environment_variable(monkeypatch):
    """Verifica que el servei llegeix la config correcta."""
    monkeypatch.setenv("ESPORTSPULSE_DB_PATH", "/tmp/test.db")
    
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
public BigDecimal calculateAdjustedWinRate(ChampionRecord champion) {
    if (champion.getWinRate().compareTo(new BigDecimal("54")) > 0) {  // Branca A
        return champion.getWinRate().multiply(new BigDecimal("0.95"));  // Línia 1
    } else if (champion.getGamesPlayed() > 1_000_000) {               // Branca B
        return champion.getWinRate().multiply(new BigDecimal("1.02"));  // Línia 2
    }
    return champion.getWinRate();                                       // Línia 3
}
```

- **Line coverage:** Percentatge de línies executades pels tests.
- **Branch coverage:** Percentatge de branques (if/else) executades.

Un test que només passa un champion amb winRate de 55 tindria:
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
    service.getMetaChampions();
    service.registerChampion("jinx", "Test", new BigDecimal("52.30"));
    service.calculateAdjustedWinRate(someChampion);
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
1. Canvia `>` per `>=` a `getMetaChampions()`.
2. Elimina un `null` check a `findByIdOrThrow()`.
3. Canvia `return champion.getWinRate()` per `return null` a `calculateAdjustedWinRate()`.
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
addopts = --cov=esportspulse --cov-report=html --cov-report=term-missing
```

```bash
# Execució manual
pytest --cov=esportspulse --cov-fail-under=70

# Output:
# Name                                Stmts   Miss  Cover
# -------------------------------------------------------
# esportspulse/champion_repository.py    45      3    93%
# esportspulse/champion_service.py       30      8    73%
# -------------------------------------------------------
# TOTAL                                  75     11    85%
```

---

## 9. Anti-Patrons de Testing

### 1. Test que no asserteja res d'útil

```java
// MAL: Executa codi però no verifica res
@Test
void testRegisterChampion() {
    service.registerChampion("jinx", "Jinx", new BigDecimal("52.30"));
    // ... i ja? Què estem comprovant?
}

// BÉ: Verifica el comportament esperat
@Test
void shouldSaveChampionWhenRegistering() {
    service.registerChampion("jinx", "Jinx", new BigDecimal("52.30"));
    
    verify(repository).save(argThat(champion -> 
        champion.getChampionId().equals("jinx") && 
        champion.getName().equals("Jinx")
    ));
}
```

**Pista:** Si pots eliminar l'`assert` i el test segueix passant, el test no serveix de res.

### 2. Test que testeja el mock

```java
// MAL: Estàs testejant que Mockito funciona
@Test
void testGetAllChampions() {
    List<ChampionRecord> champions = List.of(someChampion);
    when(repository.findAll()).thenReturn(champions);
    
    assertEquals(champions, service.getAllChampions());
    // Literalment li has dit que retorni 'champions' i ara comproves que retorna 'champions'
}

// BÉ: Testeja lògica que transforma el resultat
@Test
void shouldReturnOnlyMetaChampions() {
    when(repository.findAll()).thenReturn(List.of(popularChampion, unpopularChampion));
    
    List<ChampionRecord> result = service.getMetaChampions();
    
    assertThat(result).containsOnly(popularChampion);
    // Aquí SÍ testem la lògica de filtratge del servei
}
```

Recorda el Snippet 4 de S4: si el test no testeja transformació, validació, o lògica de negoci, probablement testeja el mock.

### 3. Test fràgil

```java
// MAL: Depèn de l'ordre dels resultats
@Test
void testFindAll() {
    List<ChampionRecord> result = service.getAllChampions();
    assertEquals("Jinx", result.get(0).getName());  // Per què el primer?
    assertEquals("Yasuo", result.get(1).getName());  // I si la BD retorna en ordre diferent?
}

// BÉ: Verifica contingut sense dependre de l'ordre
@Test
void shouldReturnAllRegisteredChampions() {
    List<ChampionRecord> result = service.getAllChampions();
    assertThat(result)
        .extracting(ChampionRecord::getName)
        .containsExactlyInAnyOrder("Jinx", "Yasuo");
}
```

```java
// MAL: Depèn del temps
@Test
void testTimestamp() {
    ChampionRecord champion = service.registerChampion("jinx", "Jinx", new BigDecimal("52.30"));
    assertEquals(LocalDateTime.now(), champion.getCreatedAt());  // Pot fallar per mil·lisegons!
}

// BÉ: Tolerància temporal
@Test
void shouldSetCreatedAtToCurrentTime() {
    LocalDateTime before = LocalDateTime.now();
    ChampionRecord champion = service.registerChampion("jinx", "Jinx", new BigDecimal("52.30"));
    LocalDateTime after = LocalDateTime.now();
    
    assertThat(champion.getCreatedAt()).isBetween(before, after);
}
```

### 4. Over-mocking

```java
// MAL: Tot mockejat, no testem res real
@Mock ChampionJpaRepository repository;
@Mock ChampionValidator validator;
@Mock ChampionMapper mapper;
@Mock WinRateCalculator calculator;
@Mock NotificationService notifier;
@InjectMocks ChampionManagementService service;

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
    service.registerChampion("jinx", "Jinx", new BigDecimal("52.30"));
    service.registerChampion("yasuo", "Yasuo", new BigDecimal("48.50"));
    List<ChampionRecord> all = service.getAllChampions();
    assertEquals(2, all.size());
    List<ChampionRecord> meta = service.getMetaChampions();
    assertEquals(0, meta.size());
    service.updateGamesPlayed("jinx", 5_000_000L);
    meta = service.getMetaChampions();
    assertEquals(1, meta.size());
    service.deleteChampion("jinx");
    all = service.getAllChampions();
    assertEquals(1, all.size());
}

// BÉ: Un test, un concepte
@Test void shouldRegisterChampion() { ... }
@Test void shouldReturnAllChampions() { ... }
@Test void shouldReturnNoMetaChampionsInitially() { ... }
@Test void shouldReturnMetaAfterGamesPlayedUpdate() { ... }
@Test void shouldRemoveDeletedChampion() { ... }
```

**Regla:** Un test hauria de tenir un sol motiu per fallar. Si falla, el nom del test t'ha de dir què ha anat malament sense obrir el codi.

---

## 10. Aplicació a EsportsPulse: La Suite Completa

Al final de S6 (i del Bloc 1), EsportsPulse hauria de tenir aquesta estructura de tests:

```
esportspulse-engine/
├── src/
│   ├── main/java/com/esportspulse/
│   │   ├── ChampionRecord.java              (@Entity, de S5)
│   │   ├── ChampionJpaRepository.java       (JpaRepository, de S5)
│   │   ├── ChampionManagementService.java   (Service, de S2-S5)
│   │   └── ChampionDTO.java                 (record, de S2)
│   └── test/java/com/esportspulse/
│       ├── unit/
│       │   └── ChampionManagementServiceTest.java    ← @Mock + @InjectMocks
│       │       ├── shouldRegisterChampion()
│       │       ├── shouldRejectInvalidWinRate()
│       │       ├── shouldReturnMetaChampionsOnly()
│       │       ├── shouldReturnEmptyWhenNoMeta()
│       │       ├── shouldThrowWhenChampionNotFound()
│       │       └── shouldValidateWinRate() [parametrized]
│       ├── repository/
│       │   └── ChampionJpaRepositoryTest.java        ← @DataJpaTest
│       │       ├── shouldFindByNameContaining()
│       │       ├── shouldFindByGamesPlayedGreaterThan()
│       │       └── shouldReturnEmptyForNonExistent()
│       └── integration/
│           └── ChampionManagementServiceIT.java       ← @SpringBootTest
│               ├── shouldRegisterAndRetrieveChampion()
│               └── shouldFindMetaChampionsEndToEnd()
│
├── python/
│   ├── esportspulse/
│   │   ├── champion_repository.py            (SqliteChampionRepository, de S5)
│   │   └── champion_service.py
│   ├── tests/
│   │   ├── conftest.py                    (fixtures compartides)
│   │   ├── test_champion_repository.py    (fixtures + parametrize)
│   │   └── test_champion_service.py       (monkeypatch)
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
      - run: pytest --cov=esportspulse --cov-fail-under=70
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

**Objectiu setmana:** Que cada línia d'EsportsPulse tingui un test que la protegeix, que el CI ho verifiqui, i que sàpigues distingir un test que aporta valor d'un que només fa bulto.
