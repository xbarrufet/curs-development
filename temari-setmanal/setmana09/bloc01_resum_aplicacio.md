# Bloc 1 (Setmanes 1–9) — L'Aplicació EsportsPulse: D'On Veníem i On Som

## Visió General

Al final del Bloc 1, EsportsPulse és una aplicació multi-servei funcional amb:
- Un **backend Java** (Spring Boot) amb API REST, persistència JPA i tests complets
- Un **servei Python** amb models de domini equivalents, repositori SQLite i CLI
- Una **base de dades PostgreSQL** i un **cache Redis** executant-se en contenidors Docker
- Un **pipeline CI** amb GitHub Actions que valida tests, cobertura i linting automàticament
- Tot orquestrat amb **Docker Compose**: `docker-compose up` aixeca tota la plataforma

L'estudiant ha passat de zero a tenir un "Walking Skeleton" — una aplicació de punta a punta amb les bases d'arquitectura, testing, CI/CD i containerització que permetran escalar el projecte en els blocs següents.

---

## Estat de l'Aplicació al Final del Bloc 1

```
esportspulse-engine/
│
├── backend-java/                          ← Spring Boot + JPA + H2/PostgreSQL
│   ├── src/main/java/com/esportspulse/
│   │   ├── model/
│   │   │   ├── PlayerRecord.java          ← Entitat de domini (S01)
│   │   │   └── ChampionRecord.java        ← Entitat amb validació (S02)
│   │   ├── repository/
│   │   │   ├── ChampionRepository.java    ← Interfície (S02)
│   │   │   ├── InMemoryChampionRepo.java  ← Implementació RAM (S02)
│   │   │   └── ChampionJpaRepository.java ← Implementació JPA (S04)
│   │   ├── service/
│   │   │   └── ChampionManagementService.java  ← Lògica de negoci (S02)
│   │   └── controller/
│   │       └── ChampionController.java    ← API REST CRUD (S05)
│   ├── src/test/java/
│   │   ├── ChampionRecordTest.java        ← Tests model (S01)
│   │   ├── ChampionManagementServiceTest.java  ← Tests amb Mockito (S08)
│   │   ├── ChampionJpaRepositoryTest.java ← Tests integració (S04)
│   │   └── ...
│   ├── Dockerfile                         ← Multi-stage build (S09)
│   └── pom.xml                            ← JaCoCo configurat (S08)
│
├── ai-python/                             ← Python equivalent
│   ├── models/
│   │   └── champion_record.py             ← @dataclass (S02)
│   ├── repository/
│   │   ├── champion_repository.py         ← Interfície abstracta (S02)
│   │   └── sqlite_champion_repository.py  ← SQLite (S04)
│   ├── cli.py                             ← CLI amb argparse (S05)
│   ├── tests/
│   │   ├── conftest.py                    ← Fixtures (S08)
│   │   ├── test_champion_record.py        ← Tests model (S02)
│   │   ├── test_champion_service.py       ← Tests amb MagicMock (S08)
│   │   └── test_sqlite_repository.py      ← Tests integració (S04)
│   ├── Dockerfile                         ← (S09)
│   └── pyproject.toml                     ← ruff + pytest-cov (S08)
│
├── docker-compose.yml                     ← Orquestració completa (S09)
├── .env.example                           ← Variables d'entorn (S09)
├── .github/workflows/ci.yml              ← GitHub Actions (S07)
└── .gitignore
```

### Diagrama d'Arquitectura

```
┌─────────────────────────────────────────────────────────────────┐
│                     docker-compose up                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────┐      ┌──────────────────┐                │
│  │  Backend Java     │      │  Servei Python    │                │
│  │  (Spring Boot)    │      │  (CLI + tests)    │                │
│  │                   │      │                   │                │
│  │  Controller       │◄────►│  CLI (argparse)   │                │
│  │  Service          │ REST │  Repository       │                │
│  │  Repository (JPA) │      │  (SQLite)         │                │
│  │  Port 8080        │      │                   │                │
│  └────────┬──────────┘      └───────────────────┘                │
│           │                                                      │
│           ▼                                                      │
│  ┌──────────────────┐      ┌──────────────────┐                │
│  │  PostgreSQL 16    │      │  Redis 7          │                │
│  │  Port 5432        │      │  Port 6379        │                │
│  │  Volume: pgdata   │      │  (cache futur)    │                │
│  └──────────────────┘      └──────────────────┘                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Construcció Setmana a Setmana

### Setmana 01 — Les Bases: Entorn, Domini i Primers Tests

**Què tenim al final:** Un projecte Java configurat amb Git, una primera entitat de domini (`PlayerRecord`), comprensió de Big-O i rendiment, i els primers tests amb JUnit 5.

**Què hem afegit a l'aplicació:**
- Estructura del projecte Java (Maven)
- `PlayerRecord` — primera entitat de domini amb col·leccions
- Benchmarks de rendiment (JMH)
- Primers tests unitaris (JUnit 5)
- Primer Pull Request

**Habilitats adquirides:**
- Git bàsic (clone, commit, push, branch, PR)
- Modelat de domini (entitats, col·leccions)
- Anàlisi de complexitat (Big-O)
- Benchmarking (mesurar vs suposar)
- Testing bàsic (JUnit 5)

```
Aplicació al final de S01:
  Java project + PlayerRecord + JUnit tests
  ❌ Sense persistència (tot en memòria)
  ❌ Sense API
  ❌ Sense Python
```

---

### Setmana 02 — Disseny OO: Validació, Patrons i Python

**Què tenim al final:** Dues entitats de domini amb validació estricta, el Repository Pattern implementat, capa de servei amb SOLID i DI, i el model replicat en Python amb dataclasses.

**Què hem afegit a l'aplicació:**
- `ChampionRecord` amb validació al constructor (objectes sempre vàlids)
- `ChampionRepository` — interfície + implementació InMemory
- `ChampionManagementService` — lògica de negoci separada
- Python: `champion_record.py` amb `@dataclass`
- Python: `champion_repository.py` amb classe abstracta

**Habilitats adquirides:**
- Always Valid Objects (validació a la creació)
- Repository Pattern (abstraure accés a dades)
- SOLID, Dependency Injection
- Python dataclasses com equivalent de Java Records
- Tests unitaris i d'integració

```
Aplicació al final de S02:
  Java: PlayerRecord + ChampionRecord + Repository + Service
  Python: champion_record + champion_repository (in-memory)
  ❌ Sense persistència real
  ❌ Sense API
```

---

### Setmana 03 — El Terminal: SO, Permisos, Pipes i Xarxes

**Què tenim al final:** Les mateixes funcionalitats d'aplicació, però l'estudiant domina el terminal i pot automatitzar tasques.

**Què hem afegit a l'aplicació:**
- Res directament al codi — setmana d'habilitats transversals

**Habilitats adquirides:**
- Sistema operatiu (processos, memòria, fitxers)
- Permisos (rwx), variables d'entorn, PATH
- Pipes, redirecció, processament de text (grep, awk, sed)
- Bash scripting (automatitzar tasques)
- Xarxes bàsiques (IP, ports, DNS, SSH, curl)

```
Aplicació al final de S03:
  (Mateixa que S02 — setmana de fonaments)
  ✅ L'estudiant ja sap navegar per terminal, automatitzar i entendre xarxes
```

---

### Setmana 04 — Persistència Real: JPA, SQL i SQLite

**Què tenim al final:** L'aplicació guarda dades de veritat — en H2 (Java) i SQLite (Python). L'estudiant sap SQL bàsic.

**Què hem afegit a l'aplicació:**
- `ChampionJpaRepository` — repositori amb Spring Data JPA + H2
- Entitat JPA amb anotacions (`@Entity`, `@Id`)
- Consultes SQL (SELECT, WHERE, JOIN, INDEX)
- Python: `sqlite_champion_repository.py` — persistència amb SQLite
- Entorn virtual Python (venv, requirements.txt)

**Habilitats adquirides:**
- Spring Data JPA (entitats, repositoris, consultes derivades)
- SQL fonamental (SELECT, WHERE, JOIN, INDEX)
- SQLite des de Python
- Entorns virtuals Python

```
Aplicació al final de S04:
  Java: Domini + Repository (InMemory + JPA/H2) + Service
  Python: Domini + Repository (InMemory + SQLite)
  ✅ Persistència real en ambdós llenguatges
  ❌ Sense API (encara és una library)
```

---

### Setmana 05 — L'API REST: El Walking Skeleton

**Què tenim al final:** L'aplicació és accessible via HTTP. Tenim un CRUD complet, logging de peticions, un CLI Python que consumeix l'API, i un flux end-to-end funcional.

**Què hem afegit a l'aplicació:**
- `ChampionController` — API REST amb endpoints CRUD
- Middleware de logging (request ID, temps de resposta)
- Especificació de l'API
- Python: `cli.py` amb argparse que crida l'API Java
- Tests d'integració CLI → API
- **Walking Skeleton complet**: Domini → Repository → Service → REST → CLI

**Habilitats adquirides:**
- Disseny d'APIs REST (URIs, mètodes HTTP, codis de resposta)
- Spring Boot Controllers (@RestController, @GetMapping, etc.)
- Middleware/Filters per logging
- CLI Python amb argparse + requests
- Concepte de Walking Skeleton

```
Aplicació al final de S05:
  Java: Domini → Repository → Service → REST API (port 8080)
  Python: CLI → crida API Java via HTTP
  ✅ PRIMERA DEMO END-TO-END FUNCIONAL
  ❌ Sense concurrència
  ❌ Sense CI/CD
  ❌ Sense Docker
```

---

### Setmana 06 — Concurrència: Threads, Async i Traçabilitat

**Què tenim al final:** L'aplicació pot fer crides paral·leles a APIs externes i traçar peticions amb Correlation IDs.

**Què hem afegit a l'aplicació:**
- Crides paral·leles a APIs externes amb `CompletableFuture`
- Virtual Threads (Java 21) per escalabilitat
- Python: concurrència amb `threading` i `asyncio`
- Correlation IDs per traçar peticions entre serveis
- Exercici integrador: Parallel Champion Extractor

**Habilitats adquirides:**
- Threads, race conditions, operacions atòmiques
- CompletableFuture (composició asíncrona)
- Virtual Threads (Java 21)
- Python: GIL, threading, asyncio
- Correlation IDs i traçabilitat

```
Aplicació al final de S06:
  Java: API REST + crides paral·leles + Correlation IDs
  Python: Extractor paral·lel amb threading/asyncio
  ✅ L'aplicació escala i traça peticions
  ❌ Sense CI/CD
  ❌ Sense Docker
```

---

### Setmana 07 — Qualitat Professional: CI, Code Review i Release

**Què tenim al final:** El projecte té un pipeline CI automàtic, pràctiques de code review i la primera release versionada (v0.1).

**Què hem afegit a l'aplicació:**
- `.github/workflows/ci.yml` — pipeline amb GitHub Actions
- Pre-commit hooks per validar codi abans de commit
- Tag `v0.1` — primera release formal
- CI badge al README

**Habilitats adquirides:**
- Detectar anti-patrons en codi generat per IA
- Git rebase i resolució de conflictes
- GitHub Actions (CI pipeline)
- Code review com a pràctica professional
- Pre-commit hooks
- Versionat semàntic i releases

```
Aplicació al final de S07:
  Java: API REST + concurrència + CI/CD
  Python: CLI + concurrència + CI/CD
  ✅ Pipeline CI verd amb cada push
  ✅ Tag v0.1 publicat
  ❌ Tests encara bàsics
  ❌ Sense Docker
```

---

### Setmana 08 — Testing Avançat: Mocks, Cobertura i Filosofia

**Què tenim al final:** Una suite de tests completa i professional amb mocks, cobertura mesurada i linting automàtic.

**Què hem afegit a l'aplicació:**
- Tests avançats amb JUnit 5 (@Nested, @ParameterizedTest)
- Mockito per tests unitaris aïllats
- Python: pytest amb fixtures i MagicMock
- JaCoCo — cobertura mínima 70% al CI
- ruff — linting Python al CI
- CI actualitzat amb ambdós jobs (Java + Python)

**Habilitats adquirides:**
- JUnit 5 avançat (parametritzats, nested, cicle de vida)
- Mockito (@Mock, @InjectMocks, verify, ArgumentCaptor)
- pytest (fixtures, parametrize, MagicMock)
- Cobertura de codi (JaCoCo, pytest-cov)
- Linting (ruff)
- Filosofia del testing (què testejar, què no, anti-patrons)

```
Aplicació al final de S08:
  Java: API REST + concurrència + tests complets + CI verd
  Python: CLI + tests complets + linting + CI verd
  ✅ Cobertura > 70% en ambdós llenguatges
  ✅ CI valida tests, cobertura i estil automàticament
  ❌ Sense Docker
```

---

### Setmana 09 — Docker: Containerització Completa

**Què tenim al final:** Tota la plataforma containeritzada. `docker-compose up` aixeca tot: backend Java, PostgreSQL, Redis, amb health checks, volums i variables d'entorn externalitzades.

**Què hem afegit a l'aplicació:**
- `Dockerfile` per al backend Java (multi-stage build)
- `Dockerfile` per al servei Python
- `docker-compose.yml` amb tots els serveis
- Volums per persistir dades de PostgreSQL
- Health checks per a cada servei
- `.env` per externalitzar configuració
- Observabilitat: logs, stats, exec, inspect

**Habilitats adquirides:**
- Docker: imatges, contenidors, registres
- Dockerfile amb multi-stage builds
- Docker Compose: orquestrar múltiples serveis
- Volums (named, bind mount), health checks
- Variables d'entorn i .env
- Observabilitat de contenidors (logs, stats, exec, inspect)

```
Aplicació al final de S09 (FINAL BLOC 1):
  ┌─ docker-compose up ────────────────────────────┐
  │  Backend Java (Spring Boot, REST, JPA)         │
  │  Servei Python (CLI, SQLite)                   │
  │  PostgreSQL 16 (volum persistent)              │
  │  Redis 7 (cache)                               │
  └────────────────────────────────────────────────┘
  ✅ Tot containeritzat
  ✅ CI/CD amb GitHub Actions
  ✅ Tests complets amb cobertura > 70%
  ✅ Linting automàtic
  ✅ v0.1 released
  ✅ Walking Skeleton end-to-end
```

---

## Resum: Evolució de l'Aplicació en 9 Setmanes

| Setmana | Afegit | Resultat Acumulat |
|---------|--------|-------------------|
| **S01** | Entorn + PlayerRecord + JUnit | Projecte Java amb tests |
| **S02** | ChampionRecord + Repository + Service + Python | Arquitectura en capes, bilíngüe |
| **S03** | (Fonaments terminal) | Habilitats transversals |
| **S04** | JPA/H2 + SQL + SQLite | Persistència real |
| **S05** | REST API + CLI Python + Walking Skeleton | **Primera demo E2E** |
| **S06** | Concurrència + Correlation IDs | Escalabilitat i traçabilitat |
| **S07** | CI + Code Review + v0.1 | Pipeline professional |
| **S08** | Mocks + Cobertura + Linting | Testing complet |
| **S09** | Docker + Compose + Observabilitat | **Tot containeritzat** |

---

## Què Vindrà al Bloc 2

El Bloc 1 ha construït les bases: domini, persistència, API, tests, CI i Docker. El Bloc 2 (setmanes 10–16) afegirà:
- **IA**: Pydantic, API de Claude, FastAPI, MCP
- **Robustesa**: Exception handling, logging estructurat, resiliència
- **Seguretat**: Autenticació, JWT, Spring Security
- **Frontend**: Dashboard amb Streamlit
