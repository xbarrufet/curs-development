# Setmana 08 — Dilluns: JUnit 5 Avançat — Tests Parametritzats, Nested i Cicle de Vida

## Objectiu del Dia

Dominar les eines avançades de JUnit 5 per escriure tests que serveixin com a documentació viva del projecte. Al final del dia, el teu `ChampionManagementServiceTest` estarà organitzat amb `@Nested`, tindrà tests parametritzats per a validacions, i faràs servir el cicle de vida complet (`@BeforeEach`, `@AfterEach`, `@BeforeAll`, `@AfterAll`).

---

## Teoria

### Més Enllà de @Test: Tests com a Documentació Viva

Fins ara hem escrit tests amb `@Test` simples. Però quan un servei té 15-20 tests, la llista es fa il·legible. JUnit 5 ens dona eines per **organitzar** i **documentar** els tests de manera que qualsevol persona pugui entendre què fa el servei només llegint els noms dels tests.

### Cicle de Vida dels Tests

Cada test s'executa en un entorn controlat. JUnit 5 segueix aquest cicle de vida:

```
┌─────────────────────────────────────────────────────────┐
│  @BeforeAll (1 cop, static)                             │
│    Inicialitzar recursos costosos: fitxers, config...   │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │  @BeforeEach                                      │  │
│  │    Crear estat fresc per a cada test               │  │
│  │                                                    │  │
│  │  @Test (execució del test)                         │  │
│  │                                                    │  │
│  │  @AfterEach                                        │  │
│  │    Netejar estat després de cada test              │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  └─── Es repeteix per a CADA mètode @Test               │
│                                                         │
│  @AfterAll (1 cop, static)                              │
│    Alliberar recursos costosos                          │
└─────────────────────────────────────────────────────────┘
```

**Quan fer servir cadascun:**

| Anotació      | Quan?                                    | Exemple a EsportsPulse                     |
|---------------|------------------------------------------|--------------------------------------------|
| `@BeforeAll`  | Setup costós compartit entre tots els tests | Carregar fitxer de configuració            |
| `@BeforeEach` | Estat fresc per a cada test              | Crear un `Service` nou amb repositori buit |
| `@AfterEach`  | Netejar efectes secundaris               | Tancar connexions, esborrar fitxers temp   |
| `@AfterAll`   | Alliberar recursos compartits            | Tancar pool de connexions                  |

```java
// Exemple: cicle de vida complet a EsportsPulse
// Demostrem quan s'executa cada part del lifecycle
class ChampionManagementServiceLifecycleTest {

    // S'executa UN COP abans de tots els tests
    // Útil per a recursos costosos que no canvien entre tests
    @BeforeAll
    static void setupOnce() {
        System.out.println(">>> Inicialitzant recursos compartits");
    }

    // Variable d'instància: es reinicialitza cada test gràcies a @BeforeEach
    private ChampionManagementService service;
    private InMemoryChampionRepository repository;

    // S'executa ABANS de cada @Test
    // Garanteix que cada test comença amb estat net
    @BeforeEach
    void setUp() {
        // Creem un repositori buit per a cada test
        // Així cap test depèn del resultat d'un altre
        repository = new InMemoryChampionRepository();
        service = new ChampionManagementService(repository);
    }

    @Test
    void shouldRegisterNewChampion() {
        // Aquest test treballa amb un repositori BUIT
        // independentment de què facin els altres tests
        service.register(new ChampionRecord("jinx", "Marksman", 51.5));
        assertEquals(1, repository.count());
    }

    @Test
    void shouldNotAffectOtherTests() {
        // Gràcies a @BeforeEach, el repositori torna a estar BUIT
        assertEquals(0, repository.count());
    }

    // S'executa DESPRÉS de cada @Test
    // Normalment no cal en tests unitaris, però sí amb recursos externs
    @AfterEach
    void tearDown() {
        System.out.println(">>> Netejant després del test");
    }

    // S'executa UN COP després de tots els tests
    @AfterAll
    static void cleanupOnce() {
        System.out.println(">>> Alliberant recursos compartits");
    }
}
```

> **Regla d'or:** Fes servir `@BeforeEach` per defecte. Només usa `@BeforeAll` si la inicialització és realment costosa (> 100ms) i l'estat no es modifica entre tests.

---

### @Nested: Organitzar Tests per Context

`@Nested` permet agrupar tests dins de classes internes. Cada grup representa un **context** o **escenari** diferent:

```java
// Organitzem els tests per escenari de negoci
// La sortida de Maven es llegirà com documentació
class ChampionManagementServiceTest {

    private ChampionManagementService service;
    private InMemoryChampionRepository repository;

    // Estat compartit per a tots els @Nested
    @BeforeEach
    void setUp() {
        repository = new InMemoryChampionRepository();
        service = new ChampionManagementService(repository);
    }

    // Grup 1: Tests de registre de campions
    @Nested
    @DisplayName("Quan registrem un campió")
    class WhenRegisteringAChampion {

        @Test
        @DisplayName("ha de guardar-lo al repositori")
        void shouldSaveToRepository() {
            // Arrange: preparem les dades
            ChampionRecord jinx = new ChampionRecord("jinx", "Marksman", 51.5);

            // Act: executem l'acció
            service.register(jinx);

            // Assert: verifiquem el resultat
            Optional<ChampionRecord> found = repository.findById("jinx");
            assertTrue(found.isPresent(), "El campió hauria d'existir al repositori");
            assertEquals("Marksman", found.get().role());
        }

        @Test
        @DisplayName("ha de rebutjar un nom null")
        void shouldRejectNullName() {
            // Verifiquem que el servei valida les dades d'entrada
            assertThrows(IllegalArgumentException.class,
                () -> service.register(new ChampionRecord(null, "Mage", 50.0)));
        }

        @Test
        @DisplayName("ha de rebutjar un nom buit")
        void shouldRejectEmptyName() {
            assertThrows(IllegalArgumentException.class,
                () -> service.register(new ChampionRecord("", "Tank", 48.0)));
        }
    }

    // Grup 2: Tests de cerca de campions
    @Nested
    @DisplayName("Quan cerquem campions")
    class WhenSearchingChampions {

        // @BeforeEach dins de @Nested afegeix setup addicional
        // S'executa DESPRÉS del @BeforeEach del pare
        @BeforeEach
        void populateRepository() {
            // Per als tests de cerca, necessitem dades al repositori
            service.register(new ChampionRecord("jinx", "Marksman", 51.5));
            service.register(new ChampionRecord("lux", "Mage", 52.0));
            service.register(new ChampionRecord("thresh", "Support", 49.8));
        }

        @Test
        @DisplayName("ha de retornar campions pel seu rol")
        void shouldReturnChampionsByRole() {
            List<ChampionRecord> marksmen = service.findByRole("Marksman");
            assertEquals(1, marksmen.size());
            assertEquals("jinx", marksmen.get(0).name());
        }

        @Test
        @DisplayName("ha de retornar llista buida per un rol desconegut")
        void shouldReturnEmptyForUnknownRole() {
            List<ChampionRecord> result = service.findByRole("Assassin");
            assertTrue(result.isEmpty(), "No hi ha assassins al repositori");
        }
    }
}
```

**Sortida de Maven (`mvn test`):**

```
ChampionManagementServiceTest
  Quan registrem un campió
    ✓ ha de guardar-lo al repositori
    ✓ ha de rebutjar un nom null
    ✓ ha de rebutjar un nom buit
  Quan cerquem campions
    ✓ ha de retornar campions pel seu rol
    ✓ ha de retornar llista buida per un rol desconegut
```

Fixa't: **la sortida es llegeix com a documentació**. Qualsevol persona pot entendre què fa el servei.

---

### @ParameterizedTest: Un Test, Múltiples Inputs

En lloc d'escriure 5 tests gairebé idèntics que només canvien el valor d'entrada, podem usar `@ParameterizedTest`:

#### Amb @CsvSource (inputs simples)

```java
// Testem la validació de winRate amb valors límit
// Cada fila del CSV és un cas de test independent
@Nested
@DisplayName("Validació de winRate")
class WinRateValidation {

    @ParameterizedTest(name = "winRate={0} hauria de ser vàlid={1}")
    @CsvSource({
        "0.00,  true",    // Límit inferior: winRate zero és vàlid
        "52.30, true",    // Cas normal: winRate típic
        "100.0, true",    // Límit superior: winRate màxim
        "-1.00, false",   // Fora de rang: negatiu no és vàlid
        "101.0, false"    // Fora de rang: supera 100%
    })
    void shouldValidateWinRate(double winRate, boolean expectedValid) {
        if (expectedValid) {
            // Si és vàlid, no ha de llançar excepció
            assertDoesNotThrow(
                () -> service.register(
                    new ChampionRecord("test", "Mage", winRate)
                )
            );
        } else {
            // Si no és vàlid, ha de llançar IllegalArgumentException
            assertThrows(IllegalArgumentException.class,
                () -> service.register(
                    new ChampionRecord("test", "Mage", winRate)
                )
            );
        }
    }
}
```

#### Amb @MethodSource (inputs complexos)

```java
// Quan els inputs són objectes complexos, usem @MethodSource
// El mètode estàtic retorna un Stream d'Arguments
@ParameterizedTest(name = "Registrar campió: {0}")
@MethodSource("provideValidChampions")
@DisplayName("ha de registrar campions vàlids")
void shouldRegisterValidChampions(ChampionRecord champion) {
    // Act: registrem el campió
    service.register(champion);

    // Assert: verifiquem que s'ha guardat
    Optional<ChampionRecord> found = repository.findById(champion.name());
    assertTrue(found.isPresent());
    assertEquals(champion.role(), found.get().role());
}

// Mètode que proporciona els casos de test
// Ha de ser static i retornar Stream<Arguments>
static Stream<Arguments> provideValidChampions() {
    return Stream.of(
        // Cada Arguments.of() és un cas de test
        Arguments.of(new ChampionRecord("jinx", "Marksman", 51.5)),
        Arguments.of(new ChampionRecord("lux", "Mage", 52.0)),
        Arguments.of(new ChampionRecord("thresh", "Support", 49.8)),
        Arguments.of(new ChampionRecord("garen", "Fighter", 50.1))
    );
}
```

---

### @DisplayName: Noms Llegibles per a Humans

```java
// Sense @DisplayName: shouldReturnEmptyListWhenNoChampionsMatchRole
// Amb @DisplayName: "ha de retornar llista buida quan cap campió coincideix amb el rol"
@Test
@DisplayName("ha de retornar llista buida quan cap campió coincideix amb el rol")
void shouldReturnEmptyListWhenNoChampionsMatchRole() {
    // El nom del mètode segueix sent en anglès (convenció Java)
    // Però @DisplayName mostra un text llegible a la sortida de Maven
    List<ChampionRecord> result = service.findByRole("Assassin");
    assertTrue(result.isEmpty());
}
```

---

### Piràmide de Tests

```
          /\
         /  \        E2E (pocs, lents, alta confiança)
        / E2E\       Ex: Arrancar Spring Boot + fer peticions HTTP
       /______\
      /        \     Integració (alguns, velocitat mitjana)
     / Integr.  \    Ex: @DataJpaTest amb H2 real
    /____________\
   /              \  Unitaris (molts, ràpids, baixa confiança individual)
  /   Unitaris     \ Ex: Service amb mock del Repository
 /__________________\
```

| Tipus       | Velocitat    | Confiança | Quantitat | Exemple EsportsPulse                    |
|-------------|-------------|-----------|-----------|------------------------------------------|
| Unitari     | ~1ms/test   | Baixa     | Molts     | `ChampionManagementServiceTest` amb mock |
| Integració  | ~100ms/test | Mitjana   | Alguns    | `ChampionJpaRepositoryTest` amb H2       |
| E2E         | ~1s/test    | Alta      | Pocs      | Arrancar tot Spring Boot + HTTP          |

---

### Anatomia d'un Test: AAA / GWT

Tot test ben escrit segueix una estructura de tres parts:

```java
@Test
void shouldFilterChampionsByMinimumWinRate() {
    // ARRANGE (Given): Preparar l'escenari
    // Registrem campions amb diferents winRates
    service.register(new ChampionRecord("jinx", "Marksman", 55.0));
    service.register(new ChampionRecord("lux", "Mage", 48.0));
    service.register(new ChampionRecord("thresh", "Support", 52.0));

    // ACT (When): Executar l'acció que volem testejar
    // Filtrem per winRate mínim de 50%
    List<ChampionRecord> result = service.findByMinWinRate(50.0);

    // ASSERT (Then): Verificar el resultat
    // Esperem 2 campions: jinx (55%) i thresh (52%)
    assertEquals(2, result.size());
    assertTrue(result.stream().allMatch(c -> c.winRate() >= 50.0));
}
```

> **Consell:** Si no pots separar clarament les tres parts, potser el test fa massa coses. Divideix-lo.

---

## Activitat

### Exercici: Reescriure ChampionManagementServiceTest

Pren el teu `ChampionManagementServiceTest` existent i reestructura'l amb les eines d'avui:

1. **Organitza amb `@Nested`:** Crea almenys 3 grups:
   - `WhenRegisteringAChampion` — tests de registre
   - `WhenSearchingChampions` — tests de cerca
   - `WhenValidatingInputs` — tests de validació

2. **Afegeix `@ParameterizedTest`:**
   - Usa `@CsvSource` per testejar validació de `winRate` (mínim 5 valors)
   - Usa `@MethodSource` per testejar registre amb diferents campions vàlids

3. **Usa `@DisplayName`** a tots els tests i grups `@Nested`

4. **Implementa el cicle de vida:**
   - `@BeforeEach` al nivell superior per crear `service` i `repository`
   - `@BeforeEach` dins de `WhenSearchingChampions` per popular el repositori

5. **Executa `mvn test`** i verifica que la sortida es llegeix com a documentació

### Criteris d'Èxit

- Almenys 10 tests organitzats en 3+ grups `@Nested`
- Almenys 1 `@ParameterizedTest` amb `@CsvSource`
- Almenys 1 `@ParameterizedTest` amb `@MethodSource`
- Tots els tests tenen `@DisplayName` en català o anglès descriptiu
- `mvn test` passa al 100%

---

## Checklist de Lliurament

- [ ] `ChampionManagementServiceTest` refactoritzat amb `@Nested`
- [ ] Tests parametritzats amb `@CsvSource` i `@MethodSource`
- [ ] `@DisplayName` a tots els tests i grups
- [ ] `@BeforeEach` per a estat fresc a cada nivell
- [ ] Sortida de `mvn test` llegible com a documentació
- [ ] Tots els tests passen (`mvn test` verd)
- [ ] Commit: `test(java): refactor tests with nested, parameterized and lifecycle`
