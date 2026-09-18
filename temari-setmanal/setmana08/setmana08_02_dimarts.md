# Setmana 08 — Dimarts: Mockito — Aïllar la Unitat Sota Test

## Objectiu del Dia

Entendre per què necessitem mocks, com funciona Mockito, i escriure tests unitaris que aïllen el `ChampionManagementService` de les seves dependències reals. Al final del dia sabràs usar `@Mock`, `@InjectMocks`, `when/thenReturn`, `verify` i `ArgumentCaptor`.

---

## Teoria

### Què és un Mock?

Un **mock** és un objecte fals que simula el comportament d'un objecte real en un test. En lloc d'usar una dependència real (base de dades, API externa, servei de mail), creem un objecte que **fa veure** que és real però que nosaltres controlem completament.

> **Analogia:** Imagina que vols testejar si un porter de futbol para bé els penals. No necessites un estadi de 80.000 persones ni un àrbitre FIFA — necessites algú que xuti pilotes. Aquesta persona que xuta és el "mock" del davanter: fa el mateix (xutar) però en un entorn controlat.

En el context de testing:
- **Mock** = objecte simulat que imita una dependència
- **Stub** = mock que retorna respostes predefinides (`when/thenReturn`)
- **Spy** = objecte real que registra les crides que rep

---

### Per Què Necessitem Mocks?

Quan testem el `ChampionManagementService`, aquest depèn d'un `ChampionRepository`. Si usem el repositori real (JPA + H2), el test:
- Necessita una base de dades activa
- Triga més (accés a disc/memòria)
- Pot fallar per problemes de la BD, no del servei

```
  Sense mocks (test d'integració):             Amb mocks (test unitari):
  ┌──────────┐    ┌──────────────┐    ┌────┐   ┌──────────┐    ┌──────────────┐
  │  Service  │───>│  Repository  │───>│ H2 │   │  Service  │───>│  Mock Repo   │
  └──────────┘    └──────────────┘    └────┘   └──────────┘    └──────────────┘
       ↑                                            ↑
  Lent (~100ms)                               Ràpid (~1ms)
  Pot fallar per BD                           Només falla si el servei té bug
```

**Regla:** Fes servir mocks per aïllar la unitat sota test. Només testeja **una cosa** a cada test.

---

### Configuració de Mockito

Primer, assegura't que tens la dependència a `pom.xml` (ja la tens des de setmanes anteriors):

```xml
<!-- Mockito: framework per crear objectes simulats (mocks) -->
<!-- Permet testejar el servei sense necessitar un repositori real -->
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>5.12.0</version>
    <scope>test</scope>
</dependency>

<!-- Integració Mockito + JUnit 5 -->
<!-- Permet usar @Mock i @InjectMocks amb anotacions -->
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-junit-jupiter</artifactId>
    <version>5.12.0</version>
    <scope>test</scope>
</dependency>
```

### Setup amb Anotacions

```java
// @ExtendWith activa la integració Mockito-JUnit 5
// Sense això, @Mock i @InjectMocks no funcionen
@ExtendWith(MockitoExtension.class)
class ChampionManagementServiceTest {

    // @Mock crea un objecte simulat del repositori
    // Totes les crides al mock retornen null/buit per defecte
    @Mock
    private ChampionRepository repository;

    // @InjectMocks crea el servei REAL i li injecta els mocks
    // Equivalent a: new ChampionManagementService(repository)
    @InjectMocks
    private ChampionManagementService service;
}
```

**Important:** El `service` és REAL. El `repository` és FALS (mock). Estem testejant el servei de veritat, però amb una dependència controlada.

---

### when/thenReturn: Definir Comportament del Mock

Amb `when/thenReturn` diem al mock: "Quan et cridin amb X, retorna Y".

```java
@Test
@DisplayName("ha de retornar un campió quan existeix al repositori")
void shouldReturnChampionWhenExists() {
    // Arrange: definim el comportament del mock
    // Quan algú cridi findById("jinx"), el mock retorna un Optional amb Jinx
    ChampionRecord jinx = new ChampionRecord("jinx", "Marksman", 51.5);
    when(repository.findById("jinx")).thenReturn(Optional.of(jinx));

    // Act: cridem el servei (que internament crida el mock)
    Optional<ChampionRecord> result = service.findById("jinx");

    // Assert: verifiquem que el servei retorna el que esperem
    assertTrue(result.isPresent());
    assertEquals("Marksman", result.get().role());
    assertEquals(51.5, result.get().winRate());
}

@Test
@DisplayName("ha de retornar buit quan el campió no existeix")
void shouldReturnEmptyWhenNotFound() {
    // Arrange: el mock retorna Optional.empty() per qualsevol ID
    // Això simula un repositori on el campió no existeix
    when(repository.findById("championInexistent")).thenReturn(Optional.empty());

    // Act
    Optional<ChampionRecord> result = service.findById("championInexistent");

    // Assert
    assertTrue(result.isEmpty(), "Hauria de retornar Optional buit");
}

@Test
@DisplayName("ha de retornar tots els campions")
void shouldReturnAllChampions() {
    // Arrange: el mock retorna una llista predefinida
    // Controlem exactament què "hi ha" al repositori fals
    List<ChampionRecord> champions = List.of(
        new ChampionRecord("jinx", "Marksman", 51.5),
        new ChampionRecord("lux", "Mage", 52.0),
        new ChampionRecord("thresh", "Support", 49.8)
    );
    when(repository.findAll()).thenReturn(champions);

    // Act
    List<ChampionRecord> result = service.findAll();

    // Assert
    assertEquals(3, result.size());
}
```

#### when/thenThrow: Simular Errors

```java
@Test
@DisplayName("ha de gestionar errors del repositori")
void shouldHandleRepositoryErrors() {
    // Arrange: simulem que el repositori llança una excepció
    // Això passa quan la BD està caiguda o hi ha errors de connexió
    when(repository.findAll()).thenThrow(
        new RuntimeException("Connexió a la BD fallida")
    );

    // Act + Assert: verifiquem que el servei gestiona l'error
    // (depenent de la implementació: pot relançar, retornar buit, etc.)
    assertThrows(RuntimeException.class, () -> service.findAll());
}
```

---

### verify: Confirmar Interaccions

`verify` comprova que el mock ha rebut una crida específica. No comprova el resultat, sinó **que la interacció ha passat**.

```java
@Test
@DisplayName("ha de guardar el campió al repositori quan el registrem")
void shouldSaveChampionToRepository() {
    // Arrange
    ChampionRecord jinx = new ChampionRecord("jinx", "Marksman", 51.5);

    // Act: registrem el campió
    service.register(jinx);

    // Assert: verifiquem que el servei HA CRIDAT repository.save()
    // Si el servei no crida save(), el test falla
    verify(repository).save(any(ChampionRecord.class));
}

@Test
@DisplayName("no ha d'esborrar res quan registrem un campió")
void shouldNotDeleteWhenRegistering() {
    // Arrange
    ChampionRecord jinx = new ChampionRecord("jinx", "Marksman", 51.5);

    // Act
    service.register(jinx);

    // Assert: verifiquem que delete() NO s'ha cridat MAI
    // Protegeix contra efectes secundaris no desitjats
    verify(repository, never()).delete(any());
}

@Test
@DisplayName("ha de cridar findAll exactament un cop")
void shouldCallFindAllOnce() {
    // Arrange
    when(repository.findAll()).thenReturn(List.of());

    // Act
    service.findAll();

    // Assert: verifiquem que findAll s'ha cridat exactament 1 cop
    // Protegeix contra crides duplicades accidentals
    verify(repository, times(1)).findAll();
}
```

**Modes de verify:**
- `verify(mock).method()` — cridat exactament 1 cop (per defecte)
- `verify(mock, times(3)).method()` — cridat exactament 3 cops
- `verify(mock, never()).method()` — mai cridat
- `verify(mock, atLeastOnce()).method()` — cridat 1 o més cops

---

### ArgumentCaptor: Inspeccionar Què s'ha Passat al Mock

Quan el servei transforma les dades abans de guardar-les, volem verificar **exactament** què s'ha passat al repositori.

```java
@Test
@DisplayName("ha de normalitzar el nom del campió a minúscules abans de guardar")
void shouldNormalizeChampionNameBeforeSaving() {
    // Arrange: registrem amb majúscules
    ChampionRecord jinxUpperCase = new ChampionRecord("JINX", "Marksman", 51.5);

    // Creem un ArgumentCaptor per capturar el que es passa a save()
    ArgumentCaptor<ChampionRecord> captor = ArgumentCaptor.forClass(ChampionRecord.class);

    // Act
    service.register(jinxUpperCase);

    // Assert: capturem l'argument passat a save()
    verify(repository).save(captor.capture());

    // Ara podem inspeccionar l'objecte capturat
    ChampionRecord savedChampion = captor.getValue();

    // Verifiquem que el servei ha normalitzat el nom a minúscules
    assertEquals("jinx", savedChampion.name(),
        "El nom s'hauria de guardar en minúscules");

    // Verifiquem que la resta de camps no han canviat
    assertEquals("Marksman", savedChampion.role());
    assertEquals(51.5, savedChampion.winRate());
}
```

---

### Anti-Patrons amb Mocks

#### 1. Testejar el Mock (Test Inútil)

```java
// MAL: Estem testejant que el mock retorna el que li hem dit
// Això no testeja RES del nostre codi
@Test
void testInutil() {
    when(repository.findById("jinx")).thenReturn(Optional.of(jinx));

    // Cridem el mock directament, no el servei!
    Optional<ChampionRecord> result = repository.findById("jinx");

    // Òbviament retorna el que hem configurat — no testeja res
    assertTrue(result.isPresent()); // SEMPRE passa, no demostra res
}
```

#### 2. Over-Mocking (Massa Mocks)

```java
// MAL: Si necessites 4+ mocks, potser el servei fa massa coses
// Considera dividir-lo (Single Responsibility Principle)
@Mock private ChampionRepository championRepo;
@Mock private MatchRepository matchRepo;
@Mock private PlayerRepository playerRepo;
@Mock private NotificationService notifier;
@Mock private AuditLogger auditLogger;
// Senyal d'alerta: probablement SRP violat
```

#### 3. Verificar Implementació, No Comportament

```java
// MAL: Verifiquem l'ordre intern de les crides
// Si refactoritzem el servei, el test es trenca
@Test
void testFragil() {
    service.register(jinx);
    InOrder inOrder = inOrder(repository);
    inOrder.verify(repository).existsById("jinx");
    inOrder.verify(repository).save(jinx);
    // Massa acoblat a la implementació interna
}

// BÉ: Verifiquem el comportament observable
@Test
void testRobust() {
    service.register(jinx);
    verify(repository).save(jinx);
    // No ens importa COM ho fa, sinó QUÈ fa
}
```

> **Regla:** "Mockeja dependències externes, mai el subjecte sota test."

---

### Exemple Complet: Test Suite amb Mockito

```java
@ExtendWith(MockitoExtension.class)
class ChampionManagementServiceTest {

    @Mock
    private ChampionRepository repository;

    @InjectMocks
    private ChampionManagementService service;

    // Dades de test reutilitzables
    private ChampionRecord jinx;
    private ChampionRecord lux;

    @BeforeEach
    void setUp() {
        // Creem campions de test que farem servir en múltiples tests
        jinx = new ChampionRecord("jinx", "Marksman", 51.5);
        lux = new ChampionRecord("lux", "Mage", 52.0);
    }

    @Nested
    @DisplayName("Registre de campions")
    class Registration {

        @Test
        @DisplayName("ha de guardar un campió vàlid")
        void shouldSaveValidChampion() {
            service.register(jinx);
            verify(repository).save(jinx);
        }

        @Test
        @DisplayName("ha de rebutjar un campió amb winRate negatiu")
        void shouldRejectNegativeWinRate() {
            ChampionRecord invalid = new ChampionRecord("bad", "Mage", -5.0);
            assertThrows(IllegalArgumentException.class,
                () -> service.register(invalid));
            // Verifiquem que NO s'ha intentat guardar
            verify(repository, never()).save(any());
        }
    }

    @Nested
    @DisplayName("Cerca de campions")
    class Search {

        @Test
        @DisplayName("ha de retornar campions filtrats per rol")
        void shouldFilterByRole() {
            // Arrange: el mock retorna una llista mixta
            when(repository.findAll()).thenReturn(List.of(jinx, lux));

            // Act: el servei filtra per rol
            List<ChampionRecord> marksmen = service.findByRole("Marksman");

            // Assert: només retorna els Marksman
            assertEquals(1, marksmen.size());
            assertEquals("jinx", marksmen.get(0).name());
        }

        @Test
        @DisplayName("ha de retornar llista buida si no hi ha campions")
        void shouldReturnEmptyWhenNoChampions() {
            when(repository.findAll()).thenReturn(List.of());

            List<ChampionRecord> result = service.findByRole("Mage");

            assertTrue(result.isEmpty());
        }
    }
}
```

---

## Activitat

### Exercici: Tests Unitaris amb Mockito

Escriu una suite de tests per al `ChampionManagementService` usant Mockito:

1. **Configura el test amb anotacions:**
   - `@ExtendWith(MockitoExtension.class)`
   - `@Mock` per al repositori
   - `@InjectMocks` per al servei

2. **Escriu tests amb `when/thenReturn`:**
   - `findById` quan el campió existeix
   - `findById` quan el campió no existeix
   - `findAll` amb repositori buit
   - `findAll` amb múltiples campions
   - `findByRole` filtrant correctament

3. **Escriu tests amb `verify`:**
   - `register` crida `save()` amb el campió correcte
   - `register` amb dades invàlides NO crida `save()`
   - `delete` crida `deleteById()` correctament

4. **Usa `ArgumentCaptor`:**
   - Captura el `ChampionRecord` passat a `save()` i verifica tots els camps

5. **Organitza amb `@Nested` i `@DisplayName`** (aplica el que vas aprendre ahir)

### Criteris d'Èxit

- Tots els tests usen `@Mock` i `@InjectMocks` (cap repositori real)
- Almenys 3 tests amb `when/thenReturn`
- Almenys 2 tests amb `verify`
- Almenys 1 test amb `ArgumentCaptor`
- Cap anti-patró (no testejar el mock, no over-mocking)
- `mvn test` passa al 100%

---

## Checklist de Lliurament

- [ ] `ChampionManagementServiceTest` reescrit amb Mockito
- [ ] `@Mock` per al repositori, `@InjectMocks` per al servei
- [ ] Tests amb `when/thenReturn` per definir comportament
- [ ] Tests amb `verify` per confirmar interaccions
- [ ] Almenys 1 test amb `ArgumentCaptor`
- [ ] Organitzat amb `@Nested` i `@DisplayName`
- [ ] Cap anti-patró de mocking
- [ ] Tots els tests passen (`mvn test` verd)
- [ ] Commit: `test(java): add Mockito-based unit tests for service layer`
