# Setmana 8 — Exercicis de Consolidació

Aquests exercicis repassen els conceptes clau de la setmana. No cal lliurar-los — són per verificar que has entès la teoria i la pràctica abans de passar a la setmana 9. Intenta resoldre'ls sense mirar els apunts; si et quedes encallat, revisa el dia corresponent.

---

## Bloc 1: JUnit 5 Avançat (Dilluns)

**Exercici 1.1 — @ParameterizedTest**

Donat el mètode:

```java
public String classifyChampion(double winRate) {
    if (winRate >= 54.0) return "S-TIER";
    if (winRate >= 51.0) return "A-TIER";
    if (winRate >= 48.0) return "B-TIER";
    return "C-TIER";
}
```

1. Escriu un `@ParameterizedTest` amb `@CsvSource` que testegi els 4 casos: un valor per a cada tier.
2. Afegeix 4 valors extra que testegin els **límits exactes**: 54.0, 51.0, 48.0 i 47.9.
3. Per què és millor un test parametritzat que 8 tests `@Test` separats?

**Exercici 1.2 — @Nested i @DisplayName**

Sense escriure codi, dissenya l'estructura de tests (només noms) per a un `PlayerService` amb aquestes funcionalitats:
- Registrar jugadors (amb validació de nom i nivell)
- Buscar jugadors per nivell mínim
- Esborrar jugadors

Organitza-ho amb `@Nested` i `@DisplayName` perquè la sortida del test sigui llegible com a documentació.

**Exercici 1.3 — Cicle de Vida**

Respon sense executar:

1. En quin ordre s'executen `@BeforeAll`, `@BeforeEach`, `@Test`, `@AfterEach`, `@AfterAll`?
2. Si tens 3 mètodes `@Test` dins una classe, quantes vegades s'executa `@BeforeEach`?
3. Per què `@BeforeAll` ha de ser `static`?

---

## Bloc 2: Mockito (Dimarts)

**Exercici 2.1 — Conceptes**

Respon:

1. Quina diferència hi ha entre un **mock** i un **stub**?
2. Quina diferència hi ha entre `@Mock` i `@InjectMocks`?
3. Per què diem que amb mocks testem la unitat "aïllada"?

**Exercici 2.2 — Llegeix el Test**

```java
@ExtendWith(MockitoExtension.class)
class MatchServiceTest {

    @Mock private MatchRepository repository;
    @Mock private NotificationService notifier;
    @InjectMocks private MatchService service;

    @Test
    void shouldNotifyWhenMatchEnds() {
        Match match = new Match("game-1", "jinx", "lux", 35);
        when(repository.save(any())).thenReturn(match);

        service.recordMatch(match);

        verify(repository).save(match);
        verify(notifier).sendMatchResult(match);
        verify(notifier, never()).sendReminder(any());
    }
}
```

1. Quants mocks hi ha? Quins són?
2. Quin objecte és REAL?
3. Què verifica `verify(notifier, never()).sendReminder(any())`?
4. Hi ha un potencial anti-patró d'over-mocking aquí? Per què sí o per què no?

**Exercici 2.3 — Escriu el Test**

Tens un `ChampionStatsService` amb aquest mètode:

```java
public ChampionStats calculateStats(String championId) {
    List<Match> matches = matchRepository.findByChampion(championId);
    if (matches.isEmpty()) {
        throw new NoDataException("No matches found for " + championId);
    }
    double avgKills = matches.stream()
        .mapToDouble(Match::kills)
        .average().orElse(0);
    return new ChampionStats(championId, matches.size(), avgKills);
}
```

Escriu 3 tests amb Mockito:
1. Cas normal: retorna estadístiques correctes amb 3 partides.
2. Cas buit: llança `NoDataException` quan no hi ha partides.
3. Usa `ArgumentCaptor` per verificar que el repositori rep l'ID correcte.

---

## Bloc 3: pytest i unittest.mock (Dimecres)

**Exercici 3.1 — Equivalències Java ↔ Python**

Completa la taula d'equivalències:

| Java (JUnit 5 + Mockito)       | Python (pytest + unittest.mock) |
|--------------------------------|---------------------------------|
| `@Test`                        | ?                               |
| `@BeforeEach`                  | ?                               |
| `@Mock`                        | ?                               |
| `when(mock.method()).thenReturn(x)` | ?                          |
| `verify(mock).method()`        | ?                               |
| `@ParameterizedTest`           | ?                               |
| `assertEquals(expected, actual)` | ?                             |
| `assertThrows(Exception.class, () -> ...)` | ?                  |

**Exercici 3.2 — Llegeix el Test Python**

```python
def test_should_return_champion_analysis(mocker):
    mock_repo = mocker.patch("services.champion_service.ChampionRepository")
    mock_repo.return_value.find_by_id.return_value = {
        "name": "jinx", "role": "Marksman", "win_rate": 51.5
    }

    service = ChampionService()
    result = service.analyze("jinx")

    assert result["tier"] == "A"
    mock_repo.return_value.find_by_id.assert_called_once_with("jinx")
```

1. Què fa `mocker.patch`?
2. Quin és l'equivalent Java de `assert_called_once_with`?
3. Si `find_by_id` retornés `None`, què hauria de testejar un segon test?

**Exercici 3.3 — Fixtures**

Escriu un `conftest.py` amb:
1. Una fixture `sample_champions` que retorni una llista de 3 campions.
2. Una fixture `mock_repository` que retorni un `MagicMock` amb `find_all` configurat per retornar els `sample_champions`.
3. Una fixture `tmp_db` amb `tmp_path` que crei un SQLite temporal i retorni el path.

---

## Bloc 4: Cobertura i Linting (Dijous)

**Exercici 4.1 — Interpreta la Cobertura**

Donat aquest report de JaCoCo:

```
Class                          | Line Coverage | Branch Coverage
-------------------------------|-------------- |----------------
ChampionRecord                 | 100%          | 100%
ChampionManagementService      | 85%           | 70%
InMemoryChampionRepository     | 92%           | 80%
ChampionController             | 40%           | 20%
ApplicationConfig              | 10%           | 0%
```

1. Quina classe hauries de prioritzar per millorar la cobertura? Per què?
2. `ApplicationConfig` té 10% de cobertura. Hauries de preocupar-te? Per què sí o no?
3. `ChampionManagementService` té 85% de línies però 70% de branques. Què vol dir això? Quin tipus de test falta?

**Exercici 4.2 — ruff: Detecta els Problemes**

Sense executar ruff, identifica tots els problemes d'aquest codi Python:

```python
import os
import json
import sys
from typing import List, Optional
import requests

def get_champion_data(name:str)->dict:
    data=requests.get(f"https://api.example.com/{name}")
    result = data.json()
    temp = result["stats"]
    return result

class ChampionAnalyzer:
    def __init__(self):
        self.cache = {}

    def analyze(self, champion_name, role):
        if champion_name == None:
            raise ValueError("Name cannot be None")
        x = self.cache.get(champion_name)
        if x != None:
            return x
        return None
```

Llista cada problema que ruff detectaria (imports no usats, variables mortes, estil, comparacions amb None, etc.).

**Exercici 4.3 — CI Pipeline**

Escriu un workflow de GitHub Actions (`.yml`) que:
1. S'executi a cada push i pull request.
2. Tingui dos jobs: `java-tests` i `python-tests`.
3. El job Java: compili amb Maven, executi tests i verifiqui cobertura >= 70%.
4. El job Python: instal·li dependències, executi `ruff check .`, executi `pytest --cov-fail-under=70`.
5. El job Python depengui del job Java (només s'executa si Java passa).

---

## Bloc 5: Filosofia del Testing (Divendres)

**Exercici 5.1 — Classifica els Tests**

Per a cadascun, indica si és un bon test, un test innecessari o un anti-patró. Justifica:

1. Un test que verifica que `new ChampionRecord("jinx", "Marksman", 51.5).name()` retorna `"jinx"`.
2. Un test que verifica que `register(null)` llança `IllegalArgumentException`.
3. Un test que verifica l'ordre exacte de les crides internes dins de `register()`.
4. Un test que verifica que `findByRole("Mage")` retorna llista buida quan no hi ha campions Mage.
5. Un test que verifica que Spring `@Autowired` injecta correctament el repository.

**Exercici 5.2 — Dissenya la Suite**

Tens un nou servei `MatchHistoryService` amb:
- `recordMatch(Match match)` — guarda una partida
- `getHistory(String playerId)` — retorna les últimes 20 partides d'un jugador
- `getWinRate(String playerId)` — calcula el % de victòries
- `getStreak(String playerId)` — retorna la ratxa actual (victòries o derrotes consecutives)

Dissenya la suite de tests:
1. Quins tests unitaris (amb mocks) escriuries? Llista'ls amb `@DisplayName`.
2. Quins edge cases testejaries?
3. Quin test d'integració necessitaries?
4. Hi ha algun test que NO escriuries? Per què?

**Exercici 5.3 — Refactoring amb Confiança**

Imagina que has de refactoritzar `ChampionManagementService.findByMinWinRate()`:

Versió actual:
```java
public List<ChampionRecord> findByMinWinRate(double minRate) {
    return repository.findAll().stream()
        .filter(c -> c.winRate() >= minRate)
        .toList();
}
```

Nova versió (optimitzada, amb query directa):
```java
public List<ChampionRecord> findByMinWinRate(double minRate) {
    return repository.findByWinRateGreaterThanEqual(minRate);
}
```

1. Quins tests existents haurien de seguir passant després del refactoring?
2. Quins tests es trencaran? Per què?
3. Quins tests nous necessitaràs?
4. Explica per què els tests permeten fer aquest refactoring amb confiança.

---

## Autoavaluació

Abans de passar a la setmana 9, hauries de poder respondre "sí" a tot:

- [ ] Sé escriure tests parametritzats amb `@CsvSource` i `@MethodSource`
- [ ] Sé organitzar tests amb `@Nested` i `@DisplayName`
- [ ] Sé la diferència entre mock, stub i spy
- [ ] Sé usar `@Mock`, `@InjectMocks`, `when/thenReturn` i `verify`
- [ ] Sé usar `ArgumentCaptor` per inspeccionar arguments
- [ ] Sé escriure tests equivalents en pytest amb `MagicMock`
- [ ] Sé interpretar un report de cobertura de JaCoCo
- [ ] Sé què és un linter i com configurar ruff
- [ ] Sé distingir entre tests que aporten valor i tests innecessaris
- [ ] El meu CI executa tests, cobertura i linting automàticament
