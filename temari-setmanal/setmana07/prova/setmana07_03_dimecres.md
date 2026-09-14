# Setmana 07 — Dimecres: pytest i unittest.mock — Testing en Python

## Objectiu del Dia

Traslladar els patrons de testing que hem après amb JUnit 5 i Mockito al món Python amb pytest. Al final del dia tindràs una suite de tests completa per al mòdul Python d'EsportsPulse, usant fixtures, tests parametritzats i mocks.

---

## Teoria

### pytest vs JUnit: Mateixa Filosofia, Diferent Sintaxi

Python i Java comparteixen els mateixos principis de testing, però les eines tenen personalitats molt diferents. pytest és minimalista: menys boilerplate, més convencions.

### Fixtures: L'Equivalent de @BeforeEach

Una fixture és una funció decorada amb `@pytest.fixture` que prepara dades o objectes per als tests. pytest les injecta automàticament com a paràmetres:

```python
# test_champion_service.py

import pytest
from esportspulse.champion_record import ChampionRecord
from esportspulse.champion_service import ChampionManagementService
from esportspulse.in_memory_repository import InMemoryChampionRepository


# Fixture: crea un repositori buit per a cada test
# S'executa automàticament abans de cada test que la requereixi
@pytest.fixture
def repository():
    """Repositori buit, equivalent a @BeforeEach en JUnit."""
    return InMemoryChampionRepository()


# Fixture: crea el servei injectant el repositori
# Demostra composició de fixtures: depèn de 'repository'
@pytest.fixture
def service(repository):
    """Servei amb repositori buit, llest per testejar."""
    return ChampionManagementService(repository)


# Fixture: repositori amb dades predefinides
# Útil per als tests de cerca que necessiten dades existents
@pytest.fixture
def populated_repository(repository):
    """Repositori amb 3 campions per a tests de cerca."""
    repository.save(ChampionRecord("jinx", "Marksman", 51.5))
    repository.save(ChampionRecord("lux", "Mage", 52.0))
    repository.save(ChampionRecord("thresh", "Support", 49.8))
    return repository


# Fixture: servei amb dades
# Composició: depèn de populated_repository
@pytest.fixture
def populated_service(populated_repository):
    """Servei amb 3 campions registrats."""
    return ChampionManagementService(populated_repository)
```

#### conftest.py: Fixtures Compartides

Quan múltiples fitxers de test necessiten les mateixes fixtures, les posem a `conftest.py`:

```python
# tests/conftest.py
# pytest detecta automàticament aquest fitxer
# Les fixtures definides aquí estan disponibles a TOTS els tests del directori

import pytest
from esportspulse.champion_record import ChampionRecord


@pytest.fixture
def sample_jinx():
    """Campió de test: Jinx (Marksman)."""
    return ChampionRecord("jinx", "Marksman", 51.5)


@pytest.fixture
def sample_lux():
    """Campió de test: Lux (Mage)."""
    return ChampionRecord("lux", "Mage", 52.0)


@pytest.fixture
def sample_champions(sample_jinx, sample_lux):
    """Llista de campions de test per a proves de col·lecció."""
    return [
        sample_jinx,
        sample_lux,
        ChampionRecord("thresh", "Support", 49.8),
    ]
```

#### Ús en Tests

```python
# test_champion_service.py

def test_should_register_new_champion(service, sample_jinx):
    """Registrar un campió vàlid ha de guardar-lo al repositori."""
    # pytest injecta automàticament 'service' i 'sample_jinx'
    # No cal instanciar res manualment
    service.register(sample_jinx)

    found = service.find_by_id("jinx")
    assert found is not None
    assert found.role == "Marksman"


def test_should_return_all_champions(populated_service):
    """Ha de retornar tots els campions registrats."""
    # 'populated_service' ja té 3 campions gràcies a la fixture
    champions = populated_service.find_all()
    assert len(champions) == 3
```

---

### @pytest.mark.parametrize: L'Equivalent de @ParameterizedTest

```python
# Testem validació de winRate amb múltiples valors
# Cada tupla (winRate, expected_valid) és un cas de test independent
@pytest.mark.parametrize(
    "win_rate, expected_valid",
    [
        (0.0, True),      # Límit inferior: winRate zero és vàlid
        (52.3, True),     # Cas normal: winRate típic
        (100.0, True),    # Límit superior: winRate màxim
        (-1.0, False),    # Fora de rang: negatiu no és vàlid
        (101.0, False),   # Fora de rang: supera 100%
    ],
)
def test_should_validate_win_rate(service, win_rate, expected_valid):
    """El servei ha de validar que el winRate està entre 0 i 100."""
    champion = ChampionRecord("test", "Mage", win_rate)

    if expected_valid:
        # No ha de llançar excepció per a valors vàlids
        service.register(champion)
        assert service.find_by_id("test") is not None
    else:
        # Ha de llançar ValueError per a valors invàlids
        with pytest.raises(ValueError):
            service.register(champion)


# Parametritzar amb objectes complexos
# Cada campió és un cas de test complet
@pytest.mark.parametrize(
    "champion",
    [
        ChampionRecord("jinx", "Marksman", 51.5),
        ChampionRecord("lux", "Mage", 52.0),
        ChampionRecord("thresh", "Support", 49.8),
        ChampionRecord("garen", "Fighter", 50.1),
    ],
    # ids personalitzats per a la sortida de pytest
    ids=["jinx-marksman", "lux-mage", "thresh-support", "garen-fighter"],
)
def test_should_register_valid_champions(service, champion):
    """Tots els campions vàlids s'han de poder registrar correctament."""
    service.register(champion)
    found = service.find_by_id(champion.name)
    assert found is not None
    assert found.role == champion.role
```

**Sortida de pytest:**

```
test_champion_service.py::test_should_validate_win_rate[0.0-True]     PASSED
test_champion_service.py::test_should_validate_win_rate[52.3-True]    PASSED
test_champion_service.py::test_should_validate_win_rate[100.0-True]   PASSED
test_champion_service.py::test_should_validate_win_rate[-1.0-False]   PASSED
test_champion_service.py::test_should_validate_win_rate[101.0-False]  PASSED
test_champion_service.py::test_should_register_valid_champions[jinx-marksman]    PASSED
test_champion_service.py::test_should_register_valid_champions[lux-mage]        PASSED
```

---

### monkeypatch: L'Equivalent Lleuger de Mockito

`monkeypatch` és una fixture built-in de pytest que permet substituir atributs, mètodes o variables d'entorn temporalment. Els canvis es reverteixen automàticament després de cada test.

```python
def test_should_handle_broken_save(service, monkeypatch, sample_jinx):
    """Si el repositori falla al guardar, el servei ha de gestionar l'error."""

    # Definim una funció que simula un error
    def broken_save(champion):
        raise IOError("Disc ple — no es pot guardar")

    # Substituïm el mètode save() del repositori per la versió trencada
    # monkeypatch reverteix el canvi automàticament després del test
    monkeypatch.setattr(service.repository, "save", broken_save)

    # Verifiquem que el servei gestiona l'error correctament
    with pytest.raises(IOError):
        service.register(sample_jinx)


def test_should_use_test_database_path(monkeypatch):
    """Verificar que podem canviar el path de la BD via variable d'entorn."""

    # Substituïm la variable d'entorn DB_PATH
    # Útil per testejar que el codi llegeix la configuració correctament
    monkeypatch.setenv("DB_PATH", "/tmp/test_esportspulse.db")

    import os
    assert os.environ["DB_PATH"] == "/tmp/test_esportspulse.db"
    # Després del test, DB_PATH torna al seu valor original
```

---

### unittest.mock: Per a Mocking Més Complex

Quan necessitem funcionalitats equivalents a Mockito (`verify`, `ArgumentCaptor`), usem `unittest.mock`:

```python
from unittest.mock import MagicMock, patch, call


def test_should_save_champion_to_repository():
    """Equivalent a verify(repository).save() de Mockito."""
    # MagicMock crea un objecte que accepta qualsevol crida
    # Equivalent a @Mock de Mockito
    mock_repository = MagicMock()
    service = ChampionManagementService(mock_repository)

    champion = ChampionRecord("jinx", "Marksman", 51.5)
    service.register(champion)

    # Verifiquem que save() s'ha cridat amb el campió correcte
    # Equivalent a verify(repository).save(champion) de Mockito
    mock_repository.save.assert_called_once_with(champion)


def test_should_not_delete_when_registering():
    """Equivalent a verify(repository, never()).delete() de Mockito."""
    mock_repository = MagicMock()
    service = ChampionManagementService(mock_repository)

    service.register(ChampionRecord("jinx", "Marksman", 51.5))

    # Verifiquem que delete() NO s'ha cridat
    mock_repository.delete.assert_not_called()


def test_should_call_find_all_once():
    """Equivalent a verify(repository, times(1)).findAll() de Mockito."""
    mock_repository = MagicMock()
    mock_repository.find_all.return_value = []
    service = ChampionManagementService(mock_repository)

    service.find_all()

    # Verifiquem el nombre exacte de crides
    assert mock_repository.find_all.call_count == 1
```

#### side_effect: Simular Comportament Dinàmic

```python
def test_should_handle_intermittent_errors():
    """side_effect permet definir comportaments dinàmics per a cada crida."""
    mock_repository = MagicMock()

    # Primera crida: error. Segona crida: èxit.
    # Simula un error transitori de connexió a la BD
    mock_repository.find_all.side_effect = [
        IOError("Connexió perduda"),   # Primera crida: falla
        [ChampionRecord("jinx", "Marksman", 51.5)],  # Segona: funciona
    ]

    service = ChampionManagementService(mock_repository)

    # Primera crida: error
    with pytest.raises(IOError):
        service.find_all()

    # Segona crida: èxit (si el servei implementa retry)
    result = service.find_all()
    assert len(result) == 1
```

#### @patch: Substituir Mòduls Sencers

```python
# @patch substitueix un objecte durant el test
# Útil per a dependències que s'importen dins del mòdul
@patch("esportspulse.sqlite_repository.sqlite3")
def test_should_handle_sqlite_connection_error(mock_sqlite3):
    """Simular que SQLite no pot connectar."""
    # Quan algú cridi sqlite3.connect(), llançarà un error
    mock_sqlite3.connect.side_effect = Exception("BD corrupta")

    with pytest.raises(Exception):
        SqliteChampionRepository("/path/to/broken.db")
```

---

### Taula Comparativa: JUnit 5 vs pytest

| Concepte          | JUnit 5                          | pytest                                |
|-------------------|----------------------------------|---------------------------------------|
| Setup per test    | `@BeforeEach`                    | `@pytest.fixture`                     |
| Setup global      | `@BeforeAll`                     | `@pytest.fixture(scope="session")`    |
| Parametritzar     | `@ParameterizedTest + @CsvSource`| `@pytest.mark.parametrize`            |
| Grups             | `@Nested`                        | Classes dins del fitxer de test       |
| Mock              | `@Mock` (Mockito)                | `MagicMock` / `monkeypatch`           |
| Verify            | `verify(mock).method()`          | `mock.method.assert_called_once()`    |
| Excepcions        | `assertThrows(Ex.class, ()→...)` | `with pytest.raises(Ex):`             |
| Noms descriptius  | `@DisplayName("...")`           | Docstrings o noms de funcions clars   |
| Shared fixtures   | Herència de classes              | `conftest.py`                         |

---

### Testejar el SqliteChampionRepository

```python
# test_sqlite_repository.py
import os
import pytest
from esportspulse.sqlite_repository import SqliteChampionRepository
from esportspulse.champion_record import ChampionRecord


@pytest.fixture
def db_path(tmp_path):
    """Crea un path temporal per a la BD de test.
    tmp_path és una fixture built-in de pytest que crea un directori temporal.
    Es neteja automàticament després dels tests."""
    return str(tmp_path / "test_champions.db")


@pytest.fixture
def sqlite_repo(db_path):
    """Repositori SQLite amb BD temporal.
    Cada test treballa amb una BD buida i aïllada."""
    repo = SqliteChampionRepository(db_path)
    return repo


def test_should_save_and_retrieve_champion(sqlite_repo):
    """Guardar un campió i recuperar-lo ha de retornar les mateixes dades."""
    champion = ChampionRecord("jinx", "Marksman", 51.5)

    sqlite_repo.save(champion)
    found = sqlite_repo.find_by_id("jinx")

    assert found is not None
    assert found.name == "jinx"
    assert found.role == "Marksman"
    assert found.win_rate == 51.5


def test_should_return_none_for_unknown_champion(sqlite_repo):
    """Buscar un campió que no existeix ha de retornar None."""
    found = sqlite_repo.find_by_id("champion_inexistent")
    assert found is None


def test_should_persist_across_repository_instances(db_path):
    """Les dades han de persistir entre instàncies del repositori.
    Això verifica que realment estem guardant a disc, no a memòria."""
    # Primera instància: guardem
    repo1 = SqliteChampionRepository(db_path)
    repo1.save(ChampionRecord("jinx", "Marksman", 51.5))

    # Segona instància: recuperem (simula reiniciar l'aplicació)
    repo2 = SqliteChampionRepository(db_path)
    found = repo2.find_by_id("jinx")

    assert found is not None
    assert found.name == "jinx"


def test_should_delete_champion(sqlite_repo):
    """Esborrar un campió ha d'eliminar-lo de la BD."""
    sqlite_repo.save(ChampionRecord("jinx", "Marksman", 51.5))

    sqlite_repo.delete("jinx")

    assert sqlite_repo.find_by_id("jinx") is None
```

---

## Activitat

### Exercici: Suite de Tests Completa en Python

Escriu tests per al mòdul Python d'EsportsPulse:

1. **Configura les fixtures a `conftest.py`:**
   - `repository` — repositori buit
   - `service` — servei amb repositori buit
   - `sample_champions` — llista de campions de test

2. **Escriu tests parametritzats:**
   - Validació de `win_rate` amb `@pytest.mark.parametrize` (5+ valors)
   - Registre de campions vàlids parametritzat

3. **Escriu tests amb mocks:**
   - Usa `MagicMock` per aïllar el servei del repositori
   - Verifica interaccions amb `assert_called_once_with`
   - Usa `monkeypatch` per simular errors del repositori

4. **Testa el `SqliteChampionRepository`:**
   - CRUD complet: save, find, find_all, delete
   - Persistència entre instàncies (usa `tmp_path`)

5. **Executa:**
   ```bash
   # Executar tots els tests amb sortida detallada
   pytest -v

   # Executar només tests d'un fitxer
   pytest tests/test_champion_service.py -v
   ```

### Criteris d'Èxit

- `conftest.py` amb fixtures compartides
- Almenys 3 tests amb `@pytest.mark.parametrize`
- Almenys 2 tests amb `MagicMock` i verificació d'interaccions
- Tests del `SqliteChampionRepository` amb `tmp_path`
- `pytest -v` passa al 100%

---

## Checklist de Lliurament

- [ ] `conftest.py` amb fixtures compartides
- [ ] `test_champion_service.py` amb tests unitaris i parametritzats
- [ ] `test_sqlite_repository.py` amb tests d'integració
- [ ] Tests amb `MagicMock` per aïllar el servei
- [ ] Tests amb `monkeypatch` per simular errors
- [ ] Tests parametritzats amb `@pytest.mark.parametrize`
- [ ] `pytest -v` passa al 100%
- [ ] Commit: `test(python): add pytest suite with fixtures, mocks and parametrize`
