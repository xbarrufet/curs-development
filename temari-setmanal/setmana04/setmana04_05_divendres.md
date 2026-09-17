# Setmana 04 — Divendres: Entorn Python, Integracio Completa i Pull Request

## Objectiu del Dia

Configurar correctament l'entorn Python amb `venv` i `pyproject.toml`, executar el flux complet d'integracio (registrar campions, cercar, verificar) tant a Java com a Python, i crear la Pull Request de la setmana amb tot el treball integrat.

---

## Teoria

### Per que Necessitem Entorns Virtuals?

Sense entorn virtual, totes les llibreries Python s'instal·len globalment. Aixo causa conflictes:

```
Projecte A necessita pytest 7.4
Projecte B necessita pytest 8.1
→ ❌ Nomes pots tenir UNA versio instal·lada globalment

Amb venv:
Projecte A → .venv-A/ → pytest 7.4  ✅
Projecte B → .venv-B/ → pytest 8.1  ✅
Cada projecte te les seves propies dependencies, aillades.
```

### Crear i Activar un Entorn Virtual

```bash
# Naveguem al directori del projecte Python
cd esportspulse-engine/ai-python/

# Creem l'entorn virtual — .venv es el nom convencional
# python -m venv: executa el modul venv com a script
# .venv: directori on s'instal·laran les dependencies
python -m venv .venv

# Activem l'entorn — des d'ara, "python" i "pip" apunten a .venv/
# macOS / Linux:
source .venv/bin/activate

# Windows (PowerShell):
# .venv\Scripts\Activate.ps1

# Verificació: el prompt ha de mostrar (.venv) al principi
# (.venv) $ which python
# → /path/to/esportspulse-engine/ai-python/.venv/bin/python
```

**IMPORTANT:** Afegeix `.venv/` al `.gitignore`:

```gitignore
# .gitignore — afegeix aquestes linies
.venv/
__pycache__/
*.pyc
*.db
```

### pyproject.toml: Configuracio Moderna de Python

`pyproject.toml` es l'equivalent del `pom.xml` de Maven — defineix el projecte, dependencies i configuracio d'eines:

```toml
# pyproject.toml — a l'arrel de ai-python/

# --- Metadades del projecte ---
[project]
name = "esportspulse-ai"
version = "0.1.0"
description = "Modul Python d'EsportsPulse: analisi i persistencia de campions"
requires-python = ">=3.11"  # Versio minima de Python necessaria

# Dependencies de produccio (es necessiten per executar el codi)
dependencies = []  # De moment no tenim dependencies externes

# Dependencies opcionals agrupades per us
[project.optional-dependencies]
# Dependencies de desenvolupament (nomes per tests i eines)
dev = [
    "pytest>=8.0",       # Framework de tests
    "pytest-cov>=5.0",   # Cobertura de codi
    "ruff>=0.4.0",       # Linter i formatador (com Checkstyle per Java)
]

# --- Configuracio de pytest ---
[tool.pytest.ini_options]
# Directori on pytest busca els tests
testpaths = ["tests"]
# Opcions per defecte quan executem pytest
addopts = "-v --tb=short"  # -v: verbose, --tb=short: tracebacks curts

# --- Configuracio de ruff (linter) ---
[tool.ruff]
# Longitud maxima de linia
line-length = 100
# Versio de Python objectiu (per compatibilitat de sintaxi)
target-version = "py311"
```

### Instal·lacio: pip install vs pip install -e

```bash
# Activar l'entorn virtual primer!
source .venv/bin/activate

# Opcio 1: Instal·lar dependencies normals
# Descarrega i instal·la les dependencies llistades a pyproject.toml
pip install .

# Opcio 2: Instal·lar en mode editable (RECOMANAT per desenvolupament)
# -e: "editable" — els canvis al codi es reflecteixen immediatament
# sense necessitat de reinstal·lar cada cop
# [dev]: tambe instal·la les dependencies opcionals del grup "dev"
pip install -e ".[dev]"

# Verificació: pytest ha d'estar disponible
pytest --version
# → pytest 8.x.x
```

**Diferencia clau:**
- `pip install .` → copia el codi a `.venv/`. Si modifiques el codi, has de reinstal·lar.
- `pip install -e .` → crea un link. Els canvis es veuen immediatament. Ideal per desenvolupament.

### El Canvi de Repository: InMemory a SQLite

El moment de la veritat — canviem la implementacio sense tocar el servei:

```python
# main.py — punt d'entrada del programa

from champion_record import ChampionRecord
from champion_management_service import ChampionManagementService

# --- OPCIO A: Memoria (per desenvolupament rapid) ---
from in_memory_champion_repository import InMemoryChampionRepository
# repository = InMemoryChampionRepository()

# --- OPCIO B: SQLite (per persistencia) ---
from sqlite_champion_repository import SqliteChampionRepository
repository = SqliteChampionRepository("esportspulse.db")

# El servei es IDENTIC — nomes canviem quina implementacio injectem
# Aquesta es la gracia del patro Repository + Dependency Inversion
service = ChampionManagementService(repository)
```

El diagrama es el mateix que a Java:

```
┌─────────────────────────────────┐
│  ChampionManagementService      │
│  (NO CANVIA)                    │
└────────────┬────────────────────┘
             │ depèn de
             ▼
┌─────────────────────────────────┐
│  ChampionRepository (ABC)       │
└────────────┬────────────────────┘
             │ implementada per
        ┌────┴────┐
        ▼         ▼
┌────────────┐ ┌──────────────┐
│ InMemory   │ │ SQLite       │
│ (dilluns)  │ │ (dijous)     │
└────────────┘ └──────────────┘
     ↕                ↕
  Java:            Java:
  InMemory    →    JPA+H2

  Mateixa arquitectura, dos llenguatges.
```

### Test d'Integracio: Java amb @SpringBootTest

```java
package com.esportspulse.engine;

import com.esportspulse.engine.domain.ChampionRecord;
import com.esportspulse.engine.repository.ChampionRepository;
import com.esportspulse.engine.service.ChampionManagementService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.List;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Test d'integracio: verifica que tots els components funcionen junts.
 *
 * @SpringBootTest arrenca TOTA l'aplicacio Spring Boot, incloent:
 * - Connexio H2
 * - Entitats JPA
 * - Repositoris
 * - Serveis
 *
 * Es mes lent que un test unitari, pero verifica la integracio real.
 */
@SpringBootTest
class ChampionIntegrationTest {

    @Autowired // Spring injecta el servei amb el repositori JPA real
    private ChampionManagementService service;

    @Autowired // Podem injectar el repositori directament per verificar
    private ChampionRepository repository;

    @Test
    void fullFlow_registerSearchAndVerify() {
        // 1. Registrem campions
        service.registerChampion(
            new ChampionRecord("ahri-it-001", "Ahri", 1500, 52.3)
        );
        service.registerChampion(
            new ChampionRecord("jinx-it-002", "Jinx", 2300, 51.8)
        );
        service.registerChampion(
            new ChampionRecord("thresh-it-003", "Thresh", 3100, 49.5)
        );

        // 2. Cerquem per ID
        Optional<ChampionRecord> ahri = service.findChampion("ahri-it-001");
        assertTrue(ahri.isPresent(), "Ahri hauria d'existir");
        assertEquals("Ahri", ahri.get().name());

        // 3. Llistem tots
        List<ChampionRecord> all = service.listAllChampions();
        assertTrue(all.size() >= 3, "Hi hauria d'haver minim 3 campions");

        // 4. Eliminem i verifiquem
        service.removeChampion("thresh-it-003");
        Optional<ChampionRecord> deleted = service.findChampion("thresh-it-003");
        assertTrue(deleted.isEmpty(), "Thresh hauria d'estar eliminat");
    }

    @Test
    void duplicateRegistration_throws() {
        // Registrem un campió
        ChampionRecord lux = new ChampionRecord("lux-it-004", "Lux", 2800, 53.1);
        service.registerChampion(lux);

        // Intentem registrar-lo una altra vegada — ha de fallar
        assertThrows(IllegalArgumentException.class, () -> {
            service.registerChampion(lux);
        });
    }
}
```

### Test d'Integracio: Python amb pytest

```python
import pytest
from champion_record import ChampionRecord
from champion_management_service import ChampionManagementService
from sqlite_champion_repository import SqliteChampionRepository


@pytest.fixture
def service():
    """Fixture: crea un servei complet amb SQLite en memoria.

    Equivalent a @SpringBootTest — monta tots els components reals.
    La diferencia: en Python ho fem manualment (no hi ha "magic" de Spring).
    """
    repository = SqliteChampionRepository(db_path=":memory:")
    svc = ChampionManagementService(repository)
    yield svc
    repository.close()


def test_full_flow_register_search_verify(service):
    """Test d'integracio complet: registrar, cercar, verificar.

    Simula el flux real que faria un usuari de l'aplicacio.
    """
    # 1. Registrem campions
    service.register_champion(
        ChampionRecord("ahri-001", "Ahri", 1500, 52.3)
    )
    service.register_champion(
        ChampionRecord("jinx-002", "Jinx", 2300, 51.8)
    )
    service.register_champion(
        ChampionRecord("thresh-003", "Thresh", 3100, 49.5)
    )

    # 2. Cerquem per ID
    ahri = service.find_champion("ahri-001")
    assert ahri is not None, "Ahri hauria d'existir"
    assert ahri.name == "Ahri"

    # 3. Llistem tots
    all_champions = service.list_all_champions()
    assert len(all_champions) == 3

    # 4. Verifiquem noms
    names = [c.name for c in all_champions]
    assert "Ahri" in names
    assert "Jinx" in names
    assert "Thresh" in names


def test_duplicate_registration_raises(service):
    """Verifica que registrar un campió duplicat llanca ValueError."""
    champion = ChampionRecord("ahri-001", "Ahri", 1500, 52.3)
    service.register_champion(champion)

    # El segon registre ha de fallar
    with pytest.raises(ValueError, match="ja existeix"):
        service.register_champion(champion)
```

### Cicle Git: Branca, Commit i Pull Request

```bash
# 1. Crear branca des de main
git checkout main
git pull origin main
git checkout -b feature/week4-jpa-repository

# 2. Verificar que tot compila i passa tests
# Java:
mvn clean verify

# Python:
cd ai-python/
source .venv/bin/activate
pytest -v

# 3. Afegir fitxers nous i modificats
git add backend-java/src/main/java/com/esportspulse/engine/entity/ChampionEntity.java
git add backend-java/src/main/java/com/esportspulse/engine/repository/ChampionJpaRepository.java
git add backend-java/src/main/java/com/esportspulse/engine/repository/JpaChampionRepositoryAdapter.java
git add backend-java/src/main/resources/application.properties
git add backend-java/pom.xml
git add ai-python/src/sqlite_champion_repository.py
git add ai-python/tests/test_sqlite_repository.py
git add ai-python/pyproject.toml
git add ai-python/.gitignore

# 4. Commit amb missatge descriptiu (Conventional Commits)
git commit -m "feat(persistence): add JPA repository with H2 and SQLite mirror

- Add ChampionEntity with JPA annotations
- Create ChampionJpaRepository with derived queries
- Implement JpaChampionRepositoryAdapter for domain separation
- Add SqliteChampionRepository for Python side
- Configure H2 database with console access
- Add integration tests for both Java and Python"

# 5. Pujar branca i crear PR
git push -u origin feature/week4-jpa-repository

# 6. Crear Pull Request (amb GitHub CLI o via web)
gh pr create \
  --title "feat(persistence): JPA + SQLite repositories" \
  --body "## Resum
- Repository pattern implementat amb interficie + dues implementacions
- Java: Spring Data JPA amb H2 (entitat, repositori, adaptador)
- Python: sqlite3 amb SqliteChampionRepository
- Tests d'integracio a Java i Python

## Que s'ha afegit
- ChampionEntity amb @Entity, @Version
- ChampionJpaRepository amb consultes derivades
- JpaChampionRepositoryAdapter (patro Adapter)
- SqliteChampionRepository amb queries parametritzades
- Tests unitaris i d'integracio

## Com provar
1. Java: mvn clean verify
2. Python: cd ai-python && pytest -v
3. H2 Console: mvn spring-boot:run → http://localhost:8080/h2-console"
```

### Revisio d'Arquitectura: Les Capes

```
┌────────────────────────────────────────────────────────────────┐
│                    ARQUITECTURA ESPORTSPULSE                    │
│                                                                │
│  ┌──────────────────────┐     ┌──────────────────────┐        │
│  │    JAVA (Spring)     │     │    PYTHON             │        │
│  │                      │     │                       │        │
│  │  ┌────────────────┐  │     │  ┌─────────────────┐  │        │
│  │  │  Controller    │  │     │  │  main.py         │  │        │
│  │  │  (REST API)    │  │     │  │  (entry point)   │  │        │
│  │  └───────┬────────┘  │     │  └────────┬────────┘  │        │
│  │          │            │     │           │           │        │
│  │  ┌───────▼────────┐  │     │  ┌────────▼────────┐  │        │
│  │  │  Service       │  │     │  │  Service        │  │        │
│  │  │  (Logica)      │  │     │  │  (Logica)       │  │        │
│  │  └───────┬────────┘  │     │  └────────┬────────┘  │        │
│  │          │            │     │           │           │        │
│  │  ┌───────▼────────┐  │     │  ┌────────▼────────┐  │        │
│  │  │  Repository    │  │     │  │  Repository     │  │        │
│  │  │  (Interficie)  │  │     │  │  (ABC)          │  │        │
│  │  └───────┬────────┘  │     │  └────────┬────────┘  │        │
│  │     ┌────┴────┐      │     │      ┌────┴────┐      │        │
│  │     ▼         ▼      │     │      ▼         ▼      │        │
│  │  InMemory   JPA      │     │  InMemory  SQLite     │        │
│  │  (RAM)      (H2)     │     │  (dict)    (sqlite3)  │        │
│  └──────────────────────┘     └───────────────────────┘        │
│                                                                │
│  Regla d'or: les capes superiors MAI depenen de les inferiors  │
│  concretament — sempre a traves d'abstraccions (interficies).  │
└────────────────────────────────────────────────────────────────┘
```

---

## Activitat

### Exercici d'Integracio Complet

**Durada estimada:** 120 minuts

#### Part 1: Entorn Python (20 min)

1. Navega a `ai-python/`
2. Crea l'entorn virtual: `python -m venv .venv`
3. Activa'l: `source .venv/bin/activate`
4. Crea `pyproject.toml` amb la configuracio proporcionada
5. Instal·la en mode editable: `pip install -e ".[dev]"`
6. Verifica: `pytest --version` ha de funcionar
7. Afegeix `.venv/` al `.gitignore`

#### Part 2: Flux Complet Java (25 min)

1. Arrenca l'aplicacio: `mvn spring-boot:run`
2. Obre la consola H2: `http://localhost:8080/h2-console`
3. Registra 5 campions via l'API o directament al servei
4. Verifica a la consola H2: `SELECT * FROM champions;`
5. Prova consultes derivades:
   - `findByNameContaining("Ah")` → ha de retornar Ahri
   - `findByGamesPlayedGreaterThan(2000)` → campions veterans
6. Executa els tests d'integracio: `mvn test`

#### Part 3: Flux Complet Python (25 min)

1. Executa `pytest -v` per verificar tots els tests
2. Executa el flux d'integracio manualment:

```python
# script_integracio.py — executa manualment per verificar
from champion_record import ChampionRecord
from champion_management_service import ChampionManagementService
from sqlite_champion_repository import SqliteChampionRepository

# Creem el servei amb SQLite
repo = SqliteChampionRepository("integracio_test.db")
service = ChampionManagementService(repo)

# Registrem campions
campions = [
    ChampionRecord("ahri-001", "Ahri", 1500, 52.3),
    ChampionRecord("jinx-002", "Jinx", 2300, 51.8),
    ChampionRecord("thresh-003", "Thresh", 3100, 49.5),
    ChampionRecord("yasuo-004", "Yasuo", 4200, 48.7),
    ChampionRecord("lux-005", "Lux", 2800, 53.1),
]

for c in campions:
    service.register_champion(c)
    print(f"Registrat: {c.name}")

# Cerquem
print(f"\nTotal campions: {len(service.list_all_champions())}")

ahri = service.find_champion("ahri-001")
print(f"Trobat: {ahri.name} — WR: {ahri.win_rate}%")

# Verifiquem persistencia: tanquem i reobrim
repo.close()

# Reobrim la mateixa BD — les dades han de seguir alla!
repo2 = SqliteChampionRepository("integracio_test.db")
service2 = ChampionManagementService(repo2)
print(f"\nDespres de reobrir: {len(service2.list_all_champions())} campions")
# Ha d'imprimir 5 — les dades han sobreviscut al reinici!

repo2.close()
print("\nIntegracio completada amb exit!")
```

3. Executa: `python script_integracio.py`
4. Verifica que al reobrir la BD les dades persisteixen

#### Part 4: Pull Request (20 min)

1. Crea la branca: `git checkout -b feature/week4-jpa-repository`
2. Afegeix NOMES els fitxers rellevants (no `.DS_Store`, no `.db`)
3. Escriu un commit descriptiu seguint Conventional Commits
4. Puja la branca: `git push -u origin feature/week4-jpa-repository`
5. Crea la PR amb descripcio clara (usa el template de la teoria)

#### Part 5: Dibuixa l'Arquitectura (10 min)

1. Dibuixa (a ma o amb una eina) el diagrama de capes d'EsportsPulse
2. Inclou: Controller, Service, Repository (interficie), Implementacions (InMemory, JPA, SQLite)
3. Marca amb fletxes la direccio de les dependencies
4. Verifica que les fletxes van de dalt (Controller) cap a baix (Repository)
5. Verifica que CAP fletxa apunta a una implementacio concreta des del servei

---

## Checklist de Lliurament

- [ ] Entorn virtual creat i activat (`.venv/`)
- [ ] `pyproject.toml` configurat amb dependencies dev (pytest, ruff)
- [ ] `pip install -e ".[dev]"` executat sense errors
- [ ] `.venv/` afegit al `.gitignore`
- [ ] Flux Java complet: registrar, cercar, verificar a H2 console
- [ ] Flux Python complet: registrar, cercar, verificar persistencia al reobrir
- [ ] Tests Java: `mvn test` verd (unitaris + integracio)
- [ ] Tests Python: `pytest -v` verd (unitaris + integracio)
- [ ] Branca `feature/week4-jpa-repository` creada i pujada
- [ ] Pull Request creada amb descripcio clara (resum, que s'ha afegit, com provar)
- [ ] Diagrama d'arquitectura dibuixat amb les capes i dependencies
- [ ] El servei NO ha canviat en tot el proces (verifica amb `git diff` del servei)
