**Setmana 6: Testing, Mocks i Qualitat — Tancament de Bloc 1**

---

### **Dilluns: JUnit 5 en Profunditat**

* **Cursos i Material de Lectura:**
* **Article:** Baeldung — [*A Guide to JUnit 5*](https://www.baeldung.com/junit-5).
* **Documentació:** JUnit — [*JUnit 5 User Guide*](https://junit.org/junit5/docs/current/user-guide/).
* **Article:** Baeldung — [*JUnit 5 Parameterized Tests*](https://www.baeldung.com/parameterized-tests-junit-5).


* **Activitat i Què s'espera programar:**
* **Més enllà de @Test — Organització i cicle de vida:**
  * `@BeforeEach`: preparar dades de test (crear un `GameManagementService` fresh amb dependències).
  * `@AfterEach`: netejar si cal (normalment no amb tests unitaris, però sí amb recursos externs).
  * `@BeforeAll` / `@AfterAll` (static): per a setup costós que es comparteix entre tests (ex: carregar un fitxer de configuració).
* **@Nested per organitzar tests per context:**
  ```java
  class GameManagementServiceTest {
      @Nested
      class WhenRegisteringAGame {
          @Test void shouldSaveToRepository() { ... }
          @Test void shouldRejectNullTitle() { ... }
      }
      @Nested
      class WhenSearchingGames {
          @Test void shouldReturnMatchingGames() { ... }
          @Test void shouldReturnEmptyForUnknownTitle() { ... }
      }
  }
  ```
* **@ParameterizedTest per testejar múltiples inputs:**
  * `@CsvSource` per inputs simples: testejar validació de preu amb valors límit.
    ```java
    @ParameterizedTest
    @CsvSource({
        "0.00, true",    // Free-to-play és vàlid
        "59.99, true",   // Preu normal
        "-1.00, false",  // Preu negatiu no és vàlid
        "999.99, true"   // Preu alt però vàlid
    })
    void shouldValidatePrice(BigDecimal price, boolean expected) { ... }
    ```
  * `@MethodSource` per inputs complexos: testejar amb objectes `GameRecord` complets.
* **@DisplayName per output llegible:** `@DisplayName("Hauria de rebutjar un joc amb títol buit")` transforma l'output de Maven en documentació viva.
* **Lliçó clau:** "Un test ha de ser una documentació viva del teu codi. Si algú llegeix els noms dels tests, ha d'entendre què fa el servei."


---

### **Dimarts: Mocks amb Mockito — Quan i Per Què**

* **Cursos i Material de Lectura:**
* **Article:** Baeldung — [*Mockito Tutorial*](https://www.baeldung.com/mockito-series).
* **Article:** Martin Fowler — [*Mocks Aren't Stubs*](https://martinfowler.com/articles/mocksArentStubs.html).
* **Article:** Baeldung — [*Mockito ArgumentCaptor*](https://www.baeldung.com/mockito-argumentcaptor).


* **Activitat i Què s'espera programar:**
* **Setup de Mockito amb JUnit 5:**
  ```java
  @ExtendWith(MockitoExtension.class)
  class GameManagementServiceTest {
      @Mock GameJpaRepository repository;
      @InjectMocks GameManagementService service;
  }
  ```
* **when/thenReturn — Simular comportament:**
  * Quan el repository rep `findById("APP-1")`, retorna un `GameRecord` predefinit.
  * Quan el repository rep `findAll()`, retorna una llista de 3 jocs.
  * Testejar que `getPopularGames()` filtra correctament (la lògica del servei, no del repository).
* **verify — Confirmar que es va cridar el que tocava:**
  * `verify(repository).save(any(GameRecord.class))` — el servei ha guardat al repo?
  * `verify(repository, never()).delete(any())` — el servei NO ha esborrat res?
* **ArgumentCaptor per verificacions complexes:**
  ```java
  ArgumentCaptor<GameRecord> captor = ArgumentCaptor.forClass(GameRecord.class);
  verify(repository).save(captor.capture());
  GameRecord saved = captor.getValue();
  assertThat(saved.getTitle()).isEqualTo("League of Legends");
  ```
* **Anti-patró de S4 (Snippet 4): el test que testeja el mock.** Recorda: si el teu test fa `when(repo.findAll()).thenReturn(list)` i després asserta `assertEquals(list, service.getAll())`, no estàs testejant res. Estàs verificant que Mockito funciona.
* **Regla d'or:** "Mock les dependències externes, no la lògica que vols testejar."


---

### **Dimecres: pytest Mirall — Fixtures, Parametrize, Monkeypatch**

* **Cursos i Material de Lectura:**
* **Documentació:** pytest — [*How to use fixtures*](https://docs.pytest.org/en/stable/how-to/fixtures.html).
* **Article:** Real Python — [*Effective Python Testing With pytest*](https://realpython.com/pytest-python-testing/).
* **Documentació:** pytest — [*Parametrize*](https://docs.pytest.org/en/stable/how-to/parametrize.html).


* **Activitat i Què s'espera programar:**
* **Mirall Python del que s'ha fet dilluns i dimarts en Java:**
* **`@pytest.fixture` per setup:**
  ```python
  @pytest.fixture
  def repo():
      return SqliteGameRepository(":memory:")

  @pytest.fixture
  def sample_games(repo):
      games = [
          GameRecord("APP-1", "League of Legends", 0.0, 5_000_000),
          GameRecord("APP-2", "Stardew Valley", 14.99, 90_000),
      ]
      for g in games:
          repo.save(g)
      return games
  ```
* **`@pytest.mark.parametrize` per múltiples inputs:**
  ```python
  @pytest.mark.parametrize("price,valid", [
      (0.0, True), (59.99, True), (-1.0, False),
  ])
  def test_validate_price(price, valid):
      assert validate_game_price(price) == valid
  ```
* **`monkeypatch` per simular errors externs:**
  * Substituir `sqlite3.connect` per llançar un error i verificar que el servei gestiona l'error correctament.
  * Equivalent conceptual de `@Mock` en Mockito: aïllar dependències.
* **`conftest.py` per fixtures compartides:** Col·loca les fixtures comunes (repo, sample_games) a `conftest.py` perquè tots els fitxers de test les puguin usar.
* **Comparativa d'estructures:**

  | Concepte | JUnit 5 | pytest |
  |----------|---------|--------|
  | Setup per test | `@BeforeEach` | `@pytest.fixture` |
  | Múltiples inputs | `@ParameterizedTest` + `@CsvSource` | `@pytest.mark.parametrize` |
  | Mock extern | `@Mock` + Mockito | `monkeypatch` |
  | Organitzar per context | `@Nested` | Classes dins el fitxer |
  | Fixtures globals | `@BeforeAll` | `conftest.py` |


---

### **Dijous: Coverage, CI i Qualitat**

* **Cursos i Material de Lectura:**
* **Documentació:** JaCoCo — [*Maven Plugin*](https://www.jacoco.org/jacoco/trunk/doc/maven.html).
* **Documentació:** pytest-cov — [*pytest-cov Documentation*](https://pytest-cov.readthedocs.io/).
* **Article:** Martin Fowler — [*Test Coverage*](https://martinfowler.com/bliki/TestCoverage.html).


* **Activitat i Què s'espera programar:**
* **Configurar JaCoCo al `pom.xml`:**
  * Plugin `jacoco-maven-plugin` amb goals `prepare-agent` i `report`.
  * Regla `check` amb mínim 70% de line coverage.
  * Executar `mvn verify` i obrir `target/site/jacoco/index.html`.
  * Si el coverage és <70%, escriure els tests que falten (no codi sense sentit per augmentar-lo).
* **Configurar pytest-cov:**
  * `pip install pytest-cov`
  * Executar `pytest --cov=gamepulse --cov-report=html`
  * Revisar l'informe: quines línies no estan cobertes? Són importants?
* **Afegir coverage gates al CI de S4:**
  * Al `ci.yml` de GitHub Actions: `mvn verify` (que inclou JaCoCo check) en lloc de `mvn test`.
  * Afegir step per Python: `pytest --cov=gamepulse --cov-fail-under=70`.
  * El CI ha de fallar si el coverage baixa del 70%.
* **Mutation testing conceptual (sense eines):**
  * Modifica manualment 3 línies del codi de `GameManagementService`:
    1. Canvia un `>` per `>=` en `getPopularGames()`.
    2. Elimina un `null` check.
    3. Canvia un return value.
  * Executa els tests. Si tots passen, els tests son febles i cal millorar-los.
* **Lliçó clau:** "100% coverage no vol dir zero bugs. 0% coverage sí vol dir molts bugs."


---

### **Divendres: Consolidació, Suite Completa, Tag v0.1**

* **Cursos i Material de Lectura:**
* **Especificació:** [*Semantic Versioning*](https://semver.org/).
* **Article:** Google Testing Blog — [*What Makes a Good Test Suite*](https://testing.googleblog.com/).


* **Activitat i Què s'espera programar:**
* **Assegurar que GamePulse té una suite de tests completa:**
  * **Unit tests (servei amb mocks):** `GameManagementServiceTest` amb `@Mock` per `GameJpaRepository`. Tests: registrar joc, buscar per ID, buscar per títol, llistar populars, gestió d'errors.
  * **Integration tests:** `GameManagementServiceIT` amb `@SpringBootTest` i H2. Tests: el flux complet registrar → buscar → verificar.
  * **Repository tests:** `GameJpaRepositoryTest` amb `@DataJpaTest`. Tests: queries derivades, JPQL custom.
* **Tots els tests passen, CI verd:**
  * `mvn verify` passa (tests + JaCoCo check).
  * `pytest --cov-fail-under=70` passa.
  * GitHub Actions workflow completament verd.
* **Tag v0.1 — Final de Bloc 1:**
  * `git tag -a v0.1 -m "Bloc 1 complete: domain, patterns, concurrency, CI, JPA, testing"`.
  * `git push origin v0.1`.
* **Reflexió de Bloc 1 — "5 línies per a una entrevista":**
  Escriu 5 frases que podries dir en una entrevista tècnica:
  * "Sé per què HashMap és O(1) i quan ArrayList O(n) importa per rendiment." (S1)
  * "Sé implementar Repository pattern per desacoblar persistència i testejar amb mocks." (S2, S5, S6)
  * "Sé detectar race conditions i entenc @Transactional i Virtual Threads." (S3)
  * "Sé fer code review, configurar CI amb GitHub Actions, i detectar anti-patrons en codi IA." (S4)
  * "Sé escriure tests unitaris amb Mockito, tests d'integració amb Spring, i configurar coverage gates." (S6)

* **Finalització del cicle Git:**
* Commit final amb tests complets i coverage gates.
* Puja branca `feature/week6-testing-quality`, crea PR.
* Merge a `main`.

---

## Vídeos Recomanats

- **JUnit 5:** Cerca "TodoCode JUnit 5 tutorial" o "MitoCode JUnit 5" (castellà). En anglès: "Java Brains JUnit 5" (sèrie completa).
- **Mockito:** Cerca "TodoCode Mockito tutorial" o "Java Brains Mockito" (anglès). Mockito és la llibreria de mocking estàndard — entendre `when/thenReturn` i `verify` és essencial.
- **pytest (Python):** Cerca "MoureDev pytest tutorial" o "ArjanCodes pytest" (anglès, molt pràctic).
- **TDD:** Cerca "CodelyTV TDD" (castellà, expliquen el cicle red-green-refactor aplicat a projectes reals).

---

## Nota sobre Testing i Empliabilitat

Setmana 6 tanca el Bloc 1 amb la skill que defineix un desenvolupador professional: **saber testejar el propi codi**.

| Skill | On es practica | Per què importa |
|-------|---------------|-----------------|
| Tests unitaris amb mocks | Dimarts | Cada empresa espera que un junior sàpiga escriure unit tests |
| Organitzar tests per context | Dilluns | Un test suite llegible estalvia hores de debugging |
| Coverage com a safety net | Dijous | El CI t'avisa quan deixes codi sense testejar |
| Tests d'integració | Divendres | Verificar que les capes funcionen juntes, no només aïllades |
| pytest mirror | Dimecres | Demostrar versatilitat Java + Python en entrevistes |

Cap d'aquestes skills requereix coneixement avançat. Totes requereixen **disciplina** — que és el que diferencia un junior que "fa tests perquè li diuen" d'un que **entén per què els tests li estalvien temps**.
