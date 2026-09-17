# Setmana 04 — Dijous: Python i SQLite — ChampionRepository amb Base de Dades

## Objectiu del Dia

Implementar `SqliteChampionRepository` en Python utilitzant el modul estandard `sqlite3`. Al final del dia tindras un repositori persistent que guarda campions en una base de dades SQLite, amb tests automatitzats amb pytest que usen una base de dades en memoria.

---

## Teoria

### SQLite: La Base de Dades Integrada de Python

SQLite es una base de dades SQL completa que ve inclosa amb Python — no cal instal·lar res. Es l'equivalent del H2 de Java:

| Caracteristica | Java (H2) | Python (SQLite) |
|---|---|---|
| Inclosa al llenguatge | No (dependencia Maven) | Si (modul `sqlite3`) |
| Configuracio | `application.properties` | Cap — nomes `import sqlite3` |
| Mode en memoria | `jdbc:h2:mem:nom` | `sqlite3.connect(":memory:")` |
| Mode fitxer | `jdbc:h2:file:./data/nom` | `sqlite3.connect("champions.db")` |
| Tipus SQL | Complet | Complet (amb algunes limitacions) |

### Connexio Basica amb sqlite3

```python
import sqlite3

# Connectem a una base de dades en fitxer
# Si el fitxer no existeix, SQLite el crea automaticament
conn = sqlite3.connect("champions.db")

# El cursor es l'objecte que executa queries SQL
cursor = conn.cursor()

# Executem una query SQL — identica a la que escriuriem a H2
cursor.execute("""
    CREATE TABLE IF NOT EXISTS champions (
        champion_id TEXT PRIMARY KEY,
        name TEXT NOT NULL,
        games_played INTEGER DEFAULT 0,
        win_rate REAL
    )
""")

# IMPORTANT: commit() desa els canvis a disc
# Sense commit, els canvis es perden al tancar la connexio
conn.commit()

# SEMPRE tanquem la connexio quan acabem
conn.close()
```

### El Model de Dades en Python

Recordem el nostre `ChampionRecord` (creat a la setmana 1-2):

```python
from dataclasses import dataclass

@dataclass(frozen=True)  # frozen=True: immutable, com un record de Java
class ChampionRecord:
    """Registre d'un campió de League of Legends.

    frozen=True garanteix que un cop creat, no es pot modificar.
    Aixo es important per seguretat: si passes un ChampionRecord a una funcio,
    tens la garantia que no el modificara.
    """
    champion_id: str
    name: str
    games_played: int
    win_rate: float
```

### Implementacio: SqliteChampionRepository

```python
import sqlite3
from typing import Optional
from champion_record import ChampionRecord
from champion_repository import ChampionRepository


class SqliteChampionRepository(ChampionRepository):
    """Repositori de campions que usa SQLite com a backend.

    Equivalent a JpaChampionRepositoryAdapter de Java, pero sense ORM.
    Escrivim les queries SQL directament — mes control, mes responsabilitat.
    """

    def __init__(self, db_path: str = "champions.db"):
        """Inicialitza la connexio i crea la taula si no existeix.

        Args:
            db_path: ruta al fitxer de base de dades.
                     Usa ":memory:" per tests (base de dades en RAM).
        """
        # Guardem el path per poder crear connexions noves si cal
        self._db_path = db_path
        self._conn = sqlite3.connect(db_path)

        # row_factory = sqlite3.Row permet accedir a columnes per nom
        # Sense aixo, els resultats son tuples (index numeric)
        self._conn.row_factory = sqlite3.Row

        # Creem la taula al inicialitzar — IF NOT EXISTS evita errors si ja existeix
        self._create_table()

    def _create_table(self) -> None:
        """Crea la taula de campions si no existeix.

        Metode privat (prefix _) — nomes s'usa internament.
        Equivalent al ddl-auto=update de JPA.
        """
        self._conn.execute("""
            CREATE TABLE IF NOT EXISTS champions (
                champion_id TEXT PRIMARY KEY,
                name        TEXT NOT NULL,
                games_played INTEGER DEFAULT 0,
                win_rate    REAL
            )
        """)
        self._conn.commit()

    def save(self, champion: ChampionRecord) -> None:
        """Desa un campió. Si ja existeix, l'actualitza.

        INSERT OR REPLACE: equivalent a un "upsert"
        - Si el champion_id no existeix → INSERT
        - Si el champion_id ja existeix → DELETE + INSERT (replace)

        IMPORTANT: usem parametres (?) en comptes de concatenar strings.
        MAI feu: f"INSERT INTO champions VALUES ('{champion.name}')"
        Aixo es vulnerable a SQL Injection!
        """
        self._conn.execute(
            """
            INSERT OR REPLACE INTO champions (champion_id, name, games_played, win_rate)
            VALUES (?, ?, ?, ?)
            """,
            # Tupla de parametres — sqlite3 els escapa automaticament
            # Preveniu SQL Injection sense esforc
            (champion.champion_id, champion.name,
             champion.games_played, champion.win_rate)
        )
        # Commit per persistir els canvis
        self._conn.commit()

    def find_by_id(self, champion_id: str) -> ChampionRecord | None:
        """Cerca un campió per ID.

        Retorna None si no existeix — equivalent a Optional.empty() de Java.
        Python no te Optional, pero | None (union type) es la convencio.
        """
        cursor = self._conn.execute(
            "SELECT * FROM champions WHERE champion_id = ?",
            (champion_id,)  # NOTA: la coma es necessaria per fer-ho tupla d'un element
        )
        row = cursor.fetchone()

        if row is None:
            return None

        # Convertim la fila SQL a objecte de domini
        # row["column_name"] funciona gracies a row_factory = sqlite3.Row
        return ChampionRecord(
            champion_id=row["champion_id"],
            name=row["name"],
            games_played=row["games_played"],
            win_rate=row["win_rate"]
        )

    def find_all(self) -> list[ChampionRecord]:
        """Retorna tots els campions.

        fetchall() retorna una llista de totes les files.
        Usem list comprehension per convertir cada fila a ChampionRecord.
        """
        cursor = self._conn.execute("SELECT * FROM champions ORDER BY name")
        rows = cursor.fetchall()

        # List comprehension: equivalent a .stream().map().toList() de Java
        return [
            ChampionRecord(
                champion_id=row["champion_id"],
                name=row["name"],
                games_played=row["games_played"],
                win_rate=row["win_rate"]
            )
            for row in rows
        ]

    def find_by_name_containing(self, text: str) -> list[ChampionRecord]:
        """Cerca campions que continguin el text donat al nom.

        Equivalent a findByNameContaining de Spring Data JPA.
        LIKE amb % a banda i banda: cerca en qualsevol posicio.
        """
        cursor = self._conn.execute(
            "SELECT * FROM champions WHERE name LIKE ? ORDER BY name",
            (f"%{text}%",)  # f-string NOMES per construir el patró LIKE, NO per la query
        )
        return [
            ChampionRecord(
                champion_id=row["champion_id"],
                name=row["name"],
                games_played=row["games_played"],
                win_rate=row["win_rate"]
            )
            for row in cursor.fetchall()
        ]

    def delete(self, champion_id: str) -> None:
        """Elimina un campió per ID.

        Si el champion_id no existeix, no passa res (comportament idempotent).
        """
        self._conn.execute(
            "DELETE FROM champions WHERE champion_id = ?",
            (champion_id,)
        )
        self._conn.commit()

    def count(self) -> int:
        """Retorna el nombre total de campions.

        fetchone()[0]: la primera columna de la primera fila del resultat.
        """
        cursor = self._conn.execute("SELECT COUNT(*) FROM champions")
        return cursor.fetchone()[0]

    def close(self) -> None:
        """Tanca la connexio a la base de dades.

        IMPORTANT: sempre tancar connexions per alliberar recursos.
        Alternativa: usar context manager (with statement).
        """
        self._conn.close()
```

### SQL Injection: Per que MAI Concatenar Strings

```python
# ❌ PERILLOS — Vulnerable a SQL Injection
def find_by_id_UNSAFE(self, champion_id: str):
    # Un atacant podria passar: "'; DROP TABLE champions; --"
    # La query resultant seria: SELECT * FROM champions WHERE champion_id = ''; DROP TABLE champions; --'
    query = f"SELECT * FROM champions WHERE champion_id = '{champion_id}'"
    self._conn.execute(query)

# ✅ SEGUR — Queries parametritzades
def find_by_id_SAFE(self, champion_id: str):
    # sqlite3 escapa automaticament els parametres
    # L'atacant nomes pot cercar pel string literal, no injectar SQL
    self._conn.execute(
        "SELECT * FROM champions WHERE champion_id = ?",
        (champion_id,)
    )
```

### Comparacio: Java (JPA + H2) vs Python (sqlite3)

| Aspecte | Java (JPA) | Python (sqlite3) |
|---|---|---|
| ORM | Hibernate (automatic) | Manual (escrivim SQL) |
| Queries | Derivades del nom del metode | SQL escrit a ma |
| Entities | `@Entity`, `@Column` | No cal — mapegem manualment |
| Transaccions | `@Transactional` | `conn.commit()` / `conn.rollback()` |
| Migracions | `ddl-auto` | `CREATE TABLE IF NOT EXISTS` |
| Avantatge | Menys codi, mes magic | Mes control, menys abstraccio |
| Inconvenient | Corba d'aprenentatge, magic | Mes codi repetitiu |

### Tests amb pytest

```python
import pytest
from champion_record import ChampionRecord
from sqlite_champion_repository import SqliteChampionRepository


@pytest.fixture
def repository():
    """Fixture: crea un repositori amb base de dades en memoria.

    Fixtures de pytest son l'equivalent de @BeforeEach de JUnit.
    ":memory:" crea una BD nova per cada test — aïllament total.
    yield permet executar codi de "cleanup" despres del test.
    """
    repo = SqliteChampionRepository(db_path=":memory:")
    yield repo  # El test s'executa aqui
    repo.close()  # Cleanup: tanquem la connexio


@pytest.fixture
def sample_champion():
    """Fixture: un campió de mostra per reusar als tests."""
    return ChampionRecord(
        champion_id="ahri-001",
        name="Ahri",
        games_played=1500,
        win_rate=52.3
    )


def test_save_and_find_by_id(repository, sample_champion):
    """Verifica que un campió desat es pot recuperar per ID.

    Equivalent al test registerAndFind_withInMemoryRepository de Java.
    """
    # Act: desem el campió
    repository.save(sample_champion)

    # Assert: el recuperem i comprovem que es el mateix
    found = repository.find_by_id("ahri-001")
    assert found is not None
    assert found.name == "Ahri"
    assert found.games_played == 1500
    assert found.win_rate == pytest.approx(52.3, abs=0.01)  # approx per floats


def test_find_by_id_not_found(repository):
    """Verifica que cercar un ID inexistent retorna None."""
    found = repository.find_by_id("no-existeix")
    assert found is None  # Equivalent a assertTrue(optional.isEmpty()) de Java


def test_find_all_returns_all_champions(repository):
    """Verifica que find_all retorna tots els campions desats."""
    # Arrange: desem 3 campions
    champions = [
        ChampionRecord("ahri-001", "Ahri", 1500, 52.3),
        ChampionRecord("jinx-002", "Jinx", 2300, 51.8),
        ChampionRecord("thresh-003", "Thresh", 3100, 49.5),
    ]
    for champ in champions:
        repository.save(champ)

    # Act
    all_champions = repository.find_all()

    # Assert
    assert len(all_champions) == 3
    # Comprovem que els noms estan presents (ordenats per nom)
    names = [c.name for c in all_champions]
    assert "Ahri" in names
    assert "Jinx" in names
    assert "Thresh" in names


def test_find_by_name_containing(repository):
    """Verifica la cerca parcial per nom (LIKE %text%)."""
    repository.save(ChampionRecord("ahri-001", "Ahri", 1500, 52.3))
    repository.save(ChampionRecord("thresh-003", "Thresh", 3100, 49.5))

    # Cerquem campions que continguin "hr"
    results = repository.find_by_name_containing("hr")

    # "Ahri" i "Thresh" contenen "hr"
    assert len(results) == 2


def test_delete_removes_champion(repository, sample_champion):
    """Verifica que delete elimina el campió correctament."""
    repository.save(sample_champion)
    assert repository.find_by_id("ahri-001") is not None

    # Act: eliminem
    repository.delete("ahri-001")

    # Assert: ja no existeix
    assert repository.find_by_id("ahri-001") is None


def test_save_updates_existing_champion(repository):
    """Verifica que save sobre un ID existent actualitza (upsert)."""
    # Desem un campió
    original = ChampionRecord("ahri-001", "Ahri", 1500, 52.3)
    repository.save(original)

    # Desem un campió amb el MATEIX ID pero dades diferents
    updated = ChampionRecord("ahri-001", "Ahri", 2000, 55.0)
    repository.save(updated)

    # Ha d'haver-hi nomes 1 campió, amb les dades actualitzades
    found = repository.find_by_id("ahri-001")
    assert found is not None
    assert found.games_played == 2000
    assert found.win_rate == pytest.approx(55.0, abs=0.01)
    assert repository.count() == 1  # Nomes 1, no 2


def test_count_returns_correct_number(repository):
    """Verifica que count() retorna el nombre correcte de campions."""
    assert repository.count() == 0  # Inicialment buit

    repository.save(ChampionRecord("ahri-001", "Ahri", 1500, 52.3))
    assert repository.count() == 1

    repository.save(ChampionRecord("jinx-002", "Jinx", 2300, 51.8))
    assert repository.count() == 2
```

### El Patro Repository: Mateixa Interficie, Diferent Implementacio

```python
# El servei nomes coneix la interficie — identic a Java
class ChampionManagementService:
    """Servei de gestio de campions.

    Depèn de ChampionRepository (ABC), no de cap implementacio concreta.
    Podem passar InMemoryChampionRepository o SqliteChampionRepository
    sense canviar ni una linia d'aquest codi.
    """

    def __init__(self, repository: ChampionRepository):
        # Type hint amb la classe abstracta — el servei no sap quina implementacio es
        self._repository = repository

    def register_champion(self, champion: ChampionRecord) -> None:
        existing = self._repository.find_by_id(champion.champion_id)
        if existing is not None:
            raise ValueError(f"El campió amb ID {champion.champion_id} ja existeix")
        self._repository.save(champion)

    def find_champion(self, champion_id: str) -> ChampionRecord | None:
        return self._repository.find_by_id(champion_id)

    def list_all_champions(self) -> list[ChampionRecord]:
        return self._repository.find_all()


# --- Dos usos identics, implementacio diferent ---

# Opcio 1: En memoria (per tests rapids)
in_memory_service = ChampionManagementService(InMemoryChampionRepository())

# Opcio 2: SQLite (per persistencia real)
sqlite_service = ChampionManagementService(SqliteChampionRepository("champions.db"))

# El codi del servei es EXACTAMENT el mateix — aixo es el poder del patro
```

---

## Activitat

### Exercici: Implementa SqliteChampionRepository

**Durada estimada:** 90 minuts

#### Pas 1: Estructura de fitxers (5 min)

```
ai-python/
└── src/
    ├── champion_record.py            ← Ja existeix (setmanes anteriors)
    ├── champion_repository.py        ← ABC (dilluns)
    ├── in_memory_champion_repository.py  ← Dilluns
    ├── sqlite_champion_repository.py     ← NOU: implementacio SQLite
    └── tests/
        ├── test_in_memory_repository.py  ← Ja existeix
        └── test_sqlite_repository.py     ← NOU: tests SQLite
```

#### Pas 2: Implementa SqliteChampionRepository (30 min)

1. Crea `sqlite_champion_repository.py`
2. Implementa els 4 metodes CRUD + `find_by_name_containing` + `count`
3. Usa queries parametritzades (?) — MAI concatenacio de strings
4. Recorda fer `conn.commit()` despres de cada escriptura

#### Pas 3: Escriu 4 tests amb pytest (30 min)

1. `test_save_and_find_by_id` — desa i recupera
2. `test_find_all_returns_all_champions` — desa 3, recupera 3
3. `test_find_by_name_containing` — cerca parcial funciona
4. `test_delete_removes_champion` — elimina i verifica absencia

Usa `":memory:"` com a `db_path` per aïllar cada test.

#### Pas 4: Verifica (15 min)

1. Executa els tests: `python -m pytest tests/test_sqlite_repository.py -v`
2. Tots han de passar en verd
3. Prova tambe amb un fitxer real: `SqliteChampionRepository("test_champions.db")`
4. Obre el fitxer `.db` amb una eina SQLite per verificar les dades

#### Pas 5: Integra amb el servei (10 min)

1. Modifica el punt d'entrada per usar `SqliteChampionRepository` en comptes d'`InMemoryChampionRepository`
2. Verifica que el servei funciona igual — cap canvi al codi del servei

---

## Checklist de Lliurament

- [ ] `SqliteChampionRepository` implementat amb 4 metodes CRUD + `find_by_name_containing`
- [ ] Queries parametritzades (?) usades a TOTS els metodes — cap concatenacio de strings
- [ ] `CREATE TABLE IF NOT EXISTS` al constructor
- [ ] `conn.commit()` despres de cada operacio d'escriptura
- [ ] Fixture pytest amb `":memory:"` per aïllament de tests
- [ ] Minim 4 tests passant: save+find, find_all, find_by_name, delete
- [ ] Test d'upsert: save dos cops amb el mateix ID actualitza (no duplica)
- [ ] `python -m pytest -v` tot en verd
- [ ] El servei funciona amb `SqliteChampionRepository` sense canviar codi del servei
