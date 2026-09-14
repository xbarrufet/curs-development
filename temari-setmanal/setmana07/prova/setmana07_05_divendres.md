# Setmana 07 — Divendres: Filosofia del Testing, Integració i PR

## Objectiu del Dia

Reflexionar sobre **per què** testem, entendre què val la pena testejar i què no, consolidar la suite de tests completa d'EsportsPulse, i tancar el Bloc 1 amb un PR que demostri tot el que has après. Al final del dia tindràs un projecte amb CI verd, cobertura adequada i tests que documenten el comportament del sistema.

---

## Teoria

### Filosofia del Testing: Per Què Testem?

Els tests no existeixen per "complir un requisit" o "arribar al 100% de cobertura". Tenen tres propòsits reals:

1. **Protecció contra regressions:** Quan canvies codi, els tests et diuen si has trencat alguna cosa.
2. **Documentació viva:** Els noms dels tests expliquen què fa el sistema. A diferència dels comentaris, els tests no es queden obsolets perquè fallen si el comportament canvia.
3. **Habilitar el refactoring:** Sense tests, refactoritzar dóna por. Amb tests, pots reestructurar amb confiança.

> "Els tests no són per demostrar que el codi funciona. Són per avisar-te quan deixa de funcionar."

---

### Què Testejar

#### 1. Lògica de Negoci

La part més valuosa de testejar: les regles del teu domini.

```java
// TESTEJAR: el filtratge per winRate és lògica de negoci
// Si canviem el llindar o la fórmula, volem saber-ho
@Test
@DisplayName("ha de filtrar campions amb winRate per sobre del llindar")
void shouldFilterChampionsByWinRate() {
    // Arrange: creem campions amb winRates diversos
    when(repository.findAll()).thenReturn(List.of(
        new ChampionRecord("jinx", "Marksman", 55.0),
        new ChampionRecord("lux", "Mage", 48.0),
        new ChampionRecord("thresh", "Support", 52.0)
    ));

    // Act: filtrem per mínim 50%
    List<ChampionRecord> result = service.findByMinWinRate(50.0);

    // Assert: només jinx (55%) i thresh (52%) passen el filtre
    assertEquals(2, result.size());
    assertTrue(result.stream().noneMatch(c -> c.winRate() < 50.0),
        "Cap campió hauria de tenir winRate per sota del llindar");
}
```

#### 2. Gestió d'Errors

Què passa quan les coses van malament? Aquesta és una font habitual de bugs en producció.

```java
// TESTEJAR: com gestiona el servei un repositori que falla
// En producció, les connexions a BD cauen, els discs es queden sense espai...
@Test
@DisplayName("ha de llançar ServiceException quan el repositori falla")
void shouldWrapRepositoryExceptions() {
    when(repository.findAll()).thenThrow(
        new RuntimeException("Connexió a BD perduda")
    );

    // Verifiquem que el servei transforma l'excepció tècnica
    // en una excepció de negoci amb un missatge entenedor
    ServiceException ex = assertThrows(ServiceException.class,
        () -> service.findAll());
    assertTrue(ex.getMessage().contains("campions"),
        "L'error hauria de mencionar el context de negoci");
}
```

#### 3. Casos Límit (Edge Cases)

Els bugs viuen als extrems: llistes buides, valors null, limits de rang.

```java
// TESTEJAR: límits i casos especials
// Els bugs més comuns apareixen en condicions límit
@Nested
@DisplayName("Casos límit")
class EdgeCases {

    @Test
    @DisplayName("ha de retornar llista buida quan no hi ha campions")
    void shouldReturnEmptyListWhenNoChampions() {
        when(repository.findAll()).thenReturn(List.of());

        List<ChampionRecord> result = service.findAll();

        assertNotNull(result, "Mai hauria de retornar null");
        assertTrue(result.isEmpty());
    }

    @Test
    @DisplayName("ha de gestionar nom amb espais en blanc")
    void shouldHandleWhitespaceName() {
        // Un nom amb espais no hauria de ser vàlid
        assertThrows(IllegalArgumentException.class,
            () -> service.register(
                new ChampionRecord("   ", "Mage", 50.0)
            ));
    }

    @Test
    @DisplayName("ha de gestionar winRate exactament 0")
    void shouldHandleZeroWinRate() {
        // winRate 0 és vàlid (campió sense dades de partides)
        assertDoesNotThrow(
            () -> service.register(
                new ChampionRecord("newChamp", "Tank", 0.0)
            ));
    }

    @Test
    @DisplayName("ha de gestionar winRate exactament 100")
    void shouldHandleMaxWinRate() {
        // winRate 100 és vàlid (cas teòric: 100% victòries)
        assertDoesNotThrow(
            () -> service.register(
                new ChampionRecord("perfecto", "Fighter", 100.0)
            ));
    }
}
```

---

### Què NO Testejar

#### 1. Codi del Framework

```java
// NO TESTEJAR: que Spring @Autowired funciona
// Això ja ho testa l'equip de Spring. Si @Autowired falla, el problema
// és de configuració, no de lògica de negoci
@Test
void springAutowiredWorks() {
    // Inútil: estem testejant que Spring funciona, no el nostre codi
    assertNotNull(applicationContext.getBean(ChampionRepository.class));
}
```

#### 2. Getters i Setters (sense lògica)

```java
// NO TESTEJAR: getters/setters generats o trivials
// Un record de Java no té getters que puguin fallar
@Test
void testGetName() {
    ChampionRecord c = new ChampionRecord("jinx", "Marksman", 51.5);
    assertEquals("jinx", c.name());
    // Testejar això no aporta valor: el record és generat pel compilador
}
```

#### 3. Mètodes Privats Directament

```java
// NO TESTEJAR: mètodes privats directament
// Testa'ls a través de la interfície pública del servei
// Si un mètode privat falla, un test públic hauria de detectar-ho

// MAL: accedir a mètodes privats via reflection
@Test
void testPrivateNormalizeName() throws Exception {
    Method method = ChampionManagementService.class
        .getDeclaredMethod("normalizeName", String.class);
    method.setAccessible(true);
    // No facis això! Si necessites testejar-lo, potser hauria de ser públic
    // o estar en una classe Helper separada
}

// BÉ: testejar a través de la interfície pública
@Test
void shouldNormalizeNameWhenRegistering() {
    // Si normalizeName() funciona, register() retornarà el nom normalitzat
    service.register(new ChampionRecord("JINX", "Marksman", 51.5));
    Optional<ChampionRecord> found = service.findById("jinx");
    assertTrue(found.isPresent(), "El nom hauria d'estar normalitzat a minúscules");
}
```

---

### Tests com a Documentació

Si algú llegeix **només els noms dels tests**, hauria d'entendre què fa el servei:

```
ChampionManagementServiceTest
  Registre de campions
    ✓ ha de guardar un campió vàlid al repositori
    ✓ ha de rebutjar un nom null
    ✓ ha de rebutjar un nom buit
    ✓ ha de rebutjar un winRate negatiu
    ✓ ha de normalitzar el nom a minúscules
  Cerca de campions
    ✓ ha de retornar tots els campions
    ✓ ha de filtrar per rol
    ✓ ha de filtrar per winRate mínim
    ✓ ha de retornar llista buida per rol desconegut
  Gestió d'errors
    ✓ ha de gestionar errors del repositori
    ✓ ha de llançar excepció per dades invàlides
  Casos límit
    ✓ ha de retornar llista buida quan no hi ha campions
    ✓ ha de gestionar winRate exactament 0
    ✓ ha de gestionar winRate exactament 100
```

Això és documentació millor que qualsevol wiki, perquè **si el comportament canvia, els tests fallen**.

---

### La Suite Completa de Tests d'EsportsPulse

A aquest punt del curs, el teu projecte hauria de tenir:

```
esportspulse-engine/
├── backend-java/
│   └── src/test/java/com/esportspulse/engine/
│       ├── ChampionRecordTest.java             ← Tests del model (S2)
│       ├── ChampionManagementServiceTest.java  ← Tests unitaris amb mocks (S7)
│       ├── ChampionJpaRepositoryTest.java      ← Tests integració amb H2 (S6)
│       └── InMemoryChampionRepositoryTest.java ← Tests del repositori en memòria (S2)
│
└── ai-python/
    └── tests/
        ├── conftest.py                         ← Fixtures compartides (S7)
        ├── test_champion_service.py            ← Tests unitaris amb mocks (S7)
        ├── test_champion_record.py             ← Tests del model (S2)
        ├── test_in_memory_repository.py        ← Tests del repositori (S2)
        └── test_sqlite_repository.py           ← Tests integració SQLite (S6-S7)
```

#### Tipus de Tests per Capa

| Capa         | Tipus de Test    | Anotació/Eina          | Velocitat | Què Testeja?                     |
|--------------|-----------------|------------------------|-----------|----------------------------------|
| Servei       | Unitari + Mock  | `@Mock` + `@InjectMocks` | ~1ms     | Lògica de negoci aïllada        |
| Repository   | Integració      | `@DataJpaTest`         | ~100ms    | Queries JPA amb H2 real          |
| Tot el Stack | Integració Full | `@SpringBootTest`      | ~1-2s     | Tot connectat, de punta a punta  |
| Python       | Unitari + Mock  | `MagicMock` + `pytest` | ~1ms      | Lògica Python aïllada           |
| Python SQLite| Integració      | `tmp_path` + `pytest`  | ~10ms     | Persistència SQLite real         |

---

### Spring Test Slices: Quan Usar Cada Un

Spring Boot ofereix anotacions que carreguen **només una part** del context:

```java
// @DataJpaTest: carrega NOMÉS la capa JPA (repositoris + BD)
// Ús: testejar queries, mapping d'entitats
// NO carrega: controladors, serveis, seguretat
@DataJpaTest
class ChampionJpaRepositoryTest {

    @Autowired
    private ChampionJpaRepository repository;

    @Test
    void shouldSaveAndRetrieveChampion() {
        // Treballa amb H2 en memòria per defecte
        // Ràpid perquè no carrega tot Spring
        ChampionEntity entity = new ChampionEntity("jinx", "Marksman", 51.5);
        repository.save(entity);

        Optional<ChampionEntity> found = repository.findById("jinx");
        assertTrue(found.isPresent());
    }
}

// @WebMvcTest: carrega NOMÉS la capa web (controladors)
// Ús: testejar endpoints HTTP, validació de requests
// NO carrega: repositoris, BD, serveis (cal mockjar-los)
@WebMvcTest(ChampionController.class)
class ChampionControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean  // Mock de Spring (no Mockito directe)
    private ChampionManagementService service;

    @Test
    void shouldReturnChampionsList() throws Exception {
        when(service.findAll()).thenReturn(List.of(
            new ChampionRecord("jinx", "Marksman", 51.5)
        ));

        mockMvc.perform(get("/api/champions"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$[0].name").value("jinx"));
    }
}

// @SpringBootTest: carrega TOT el context
// Ús: tests E2E, verificar que tot connecta bé
// Lent: només per a tests crítics de punta a punta
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class EsportsPulseIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void shouldCreateAndRetrieveChampion() {
        // Test complet: HTTP → Controller → Service → Repository → BD
        restTemplate.postForEntity("/api/champions",
            new ChampionRecord("jinx", "Marksman", 51.5),
            Void.class);

        ChampionRecord[] champions = restTemplate.getForObject(
            "/api/champions", ChampionRecord[].class);

        assertEquals(1, champions.length);
        assertEquals("jinx", champions[0].name());
    }
}
```

| Anotació          | Què carrega           | Velocitat | Quan usar-la                    |
|-------------------|-----------------------|-----------|----------------------------------|
| `@DataJpaTest`    | JPA + BD              | Ràpid     | Testejar repositoris i queries  |
| `@WebMvcTest`     | Web + Controllers     | Ràpid     | Testejar endpoints HTTP         |
| `@SpringBootTest` | Tot                   | Lent      | Tests E2E i d'integració final  |

---

### Anti-Patrons Recap

| Anti-Patró             | Problema                                     | Solució                                    |
|------------------------|----------------------------------------------|--------------------------------------------|
| Test sense assert      | Cobertura falsa, no verifica res             | Sempre tenir almenys un assert explícit    |
| Testejar el mock       | Proves que el mock retorna el que li hem dit | Testejar el servei, no el mock             |
| Test fràgil            | Trenca quan refactoritzem sense canviar comportament | Verificar comportament, no implementació  |
| Over-mocking           | >3 mocks indica SRP violat                  | Dividir el servei o fer test d'integració  |
| Test gegant            | 50 línies amb 10 asserts                     | Un test per comportament, dividir          |
| Testejar getters       | No aporta valor, el record és trivial        | Testejar lògica real, no boilerplate       |
| Testejar mètodes privats | Acobla test a implementació               | Testejar a través de la interfície pública |

---

## Activitat

### Exercici 1: Completar la Suite de Tests

Assegura't que la teva suite de tests és completa:

1. **Java — Tests unitaris:**
   - `ChampionManagementServiceTest` amb `@Mock` i `@InjectMocks`
   - Organitzats amb `@Nested` i `@DisplayName`
   - Tests parametritzats per a validació

2. **Java — Tests d'integració:**
   - `ChampionJpaRepositoryTest` amb `@DataJpaTest`
   - CRUD complet: save, findById, findAll, delete

3. **Python — Tests:**
   - `test_champion_service.py` amb `MagicMock`
   - `test_sqlite_repository.py` amb `tmp_path`
   - Fixtures a `conftest.py`

4. **Verifica el CI:**
   ```bash
   # Java: ha de passar amb cobertura >= 70%
   mvn verify

   # Python: ha de passar amb cobertura >= 70% i sense errors de lint
   ruff check .
   pytest --cov=esportspulse --cov-fail-under=70
   ```

### Exercici 2: Cicle Git Complet

```bash
# 1. Crear branca per la setmana
git checkout -b feature/week7-testing-quality

# 2. Afegir tots els fitxers nous i modificats
git add backend-java/src/test/
git add ai-python/tests/
git add pom.xml                    # Canvis JaCoCo
git add ai-python/pyproject.toml   # Canvis pytest-cov + ruff
git add .github/workflows/ci.yml   # CI actualitzat

# 3. Commit amb missatge descriptiu
git commit -m "test: add comprehensive test suite with mocks, coverage and CI

- JUnit 5: @Nested, @ParameterizedTest, @DisplayName
- Mockito: @Mock, verify, ArgumentCaptor
- pytest: fixtures, parametrize, MagicMock
- JaCoCo: 70% minimum line coverage
- pytest-cov: 70% minimum coverage
- ruff: Python linting
- CI: updated GitHub Actions with both jobs"

# 4. Pujar la branca
git push -u origin feature/week7-testing-quality

# 5. Crear Pull Request
gh pr create \
  --title "feat: week 7 - testing, mocks and code quality" \
  --body "## Resum
- Suite de tests completa per Java i Python
- Mockito per aïllar tests unitaris
- JaCoCo i pytest-cov per cobertura > 70%
- ruff per linting Python
- CI actualitzat amb ambdós jobs

## Tests
- [ ] mvn verify passa
- [ ] pytest --cov-fail-under=70 passa
- [ ] ruff check . net
- [ ] CI verd"
```

### Exercici 3: Reflexió de Bloc 1 — "5 Línies per a una Entrevista"

Has completat el Bloc 1 del curs. Escriu 5 línies que podries dir en una entrevista de feina sobre el que has après:

```
1. "He construït un projecte amb Clean Architecture separant domini, repositori i servei."
2. "He implementat el patró Repository amb dues implementacions: InMemory per tests i JPA per producció."
3. "He escrit tests unitaris amb JUnit 5 i Mockito, aïllant cada capa."
4. "He configurat un pipeline CI amb GitHub Actions que verifica cobertura i estil."
5. "He treballat amb Java i Python en paral·lel, aplicant els mateixos patrons en ambdós."
```

Personalitza-les amb detalls del **teu** projecte. Practica dir-les en veu alta.

---

## Checklist de Lliurament

- [ ] Suite de tests Java completa (unitaris + integració)
- [ ] Suite de tests Python completa (unitaris + integració SQLite)
- [ ] `mvn verify` passa amb cobertura >= 70%
- [ ] `pytest --cov-fail-under=70` passa
- [ ] `ruff check .` net (sense errors)
- [ ] CI de GitHub Actions verd
- [ ] Branca `feature/week7-testing-quality` creada i pujada
- [ ] Pull Request creat amb descripció completa
- [ ] Reflexió "5 línies per a una entrevista" escrita
- [ ] Commit final: `feat: complete week 7 - testing, mocks and code quality`
