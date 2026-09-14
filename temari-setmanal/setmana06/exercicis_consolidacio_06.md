# Setmana 6 — Exercicis de Consolidació

---

## Bàsics (has de saber fer-ho)

### 1. Suite de tests per GameManagementService
Escriu una suite de tests completa per a `GameManagementService`. Com a mínim 3 unit tests amb Mockito (`@Mock` per `GameJpaRepository`, `@InjectMocks` pel servei) i 2 integration tests amb `@SpringBootTest` i H2. Els tests han de cobrir: registrar un joc nou, buscar per ID inexistent (verificar que llança excepció), i llistar jocs populars amb un mix de jocs que compleixen i no compleixen el criteri. Usa `@Nested` per organitzar i `@DisplayName` per fer l'output llegible.

**Connexió S2-S5:** El servei és el de S2, el repository és el JPA de S5, i l'organització de tests segueix les bones pràctiques de clean code de S4.

**Fet quan:** `mvn verify` passa, JaCoCo reporta >70% de line coverage a `GameManagementService`, i l'output de Maven es llegeix com una documentació del servei.

### 2. Mirror Python amb pytest
Escriu els tests equivalents per al mòdul Python de GamePulse. Usa `@pytest.fixture` (a `conftest.py`) per crear el `SqliteGameRepository` amb BD `:memory:` i dades de prova. Usa `@pytest.mark.parametrize` per testejar validació de preus amb múltiples inputs (positiu, zero, negatiu). Usa `monkeypatch` per simular un error de base de dades i verificar que el codi el gestiona correctament.

**Connexió S5:** El `SqliteGameRepository` és el que vas crear a S5. Ara li poses tests de veritat.

**Fet quan:** `pytest --cov=gamepulse --cov-fail-under=70` passa. L'informe HTML mostra quines línies estan cobertes.

### 3. Coverage gate al CI
Afegeix el plugin JaCoCo al `pom.xml` amb la regla de 70% mínim de line coverage (goal `check`). Afegeix `pytest-cov` al job de Python del CI. Actualitza el `ci.yml` de GitHub Actions (S4) perquè el job Java executi `mvn verify` (en lloc de `mvn test`) i el job Python executi `pytest --cov-fail-under=70`. El CI ha de fallar si el coverage baixa del llindar.

**Connexió S4:** El workflow de GitHub Actions és el que vas crear a S4. Ara li afegeixes quality gates.

**Fet quan:** CI completament verd amb coverage gates actius. Si elimines un test i el coverage baixa del 70%, el CI ha de fallar.

---

## Avançats (si vas sobrat)

### 4. Test de la race condition de S3
Reprodueix la race condition del comptador concurrent de S3 dins d'un test JUnit 5. Crea un test que llança 100 threads (o Virtual Threads) que incrementen un comptador compartit. Sense protecció (`int` normal), el resultat ha de ser inconsistent. Amb protecció (`AtomicInteger` o `synchronized`), el resultat ha de ser exactament 100. El test ha de fallar SENSE la solució i passar AMB ella.

**Connexió S3:** Demostrar dins d'un test el que vas aprendre sobre concurrència. Això és un test que serveix com a prova de concepte.

**Fet quan:** Un test que demostra el bug (falla amb `int`) i un test que demostra la solució (passa amb `AtomicInteger`). Bonus: usa `@RepeatedTest(10)` per augmentar la probabilitat de detectar la race condition.

### 5. Mutation testing manual
Modifica 5 línies del codi de GamePulse (una per una, executant tests entre cada canvi): canvia un `>` per `<` a `getPopularGames()`, elimina un null check a `findByIdOrThrow()`, canvia un return value a `calculateDiscount()`, inverteix una condició a la validació de preu, i elimina un `@Transactional`. Per cada mutació, executa `mvn test`. Si els tests segueixen passant, aquell és un forat a la suite: escriu el test que detectaria la mutació.

**Fet quan:** 5 mutacions documentades (en un comentari o fitxer). Totes detectades pels tests originals, o tests nous escrits per les que no ho eren. El codi torna a l'estat original al final.
