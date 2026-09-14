# Curs de Desenvolupament per a Juniors — AI-SDLC

Pla de formació de 24 setmanes per formar un enginyer de software junior (no només un developer). Combina fonaments d'enginyeria de software (algorítmica, SOLID, testing, Linux, Docker, APIs, SQL, seguretat, observabilitat) amb les noves formes de treballar amb IA (prompt engineering, MCP, agents, specs, eval-driven development). L'objectiu: codi que no només funciona, sinó que es pot operar, mantenir i escalar en producció.

**Projecte vehicle:** [EsportsPulse](esportspulse-project-brief.md) — un sistema multi-agent per analitzar champions, metes i estadístiques d'eSports de League of Legends (Java 21 + Python + Qdrant + Streamlit).

## Priorització Recomanada per Empleabilitat

Si l'objectiu és maximitzar la inserció laboral i la capacitat d'operar software real, la seqüència ideal és:

### Fase 1 — Base imprescindible (S1–S8)
- **Fundaments reals**: Big-O, estructures, POO, SOLID, Linux/terminal, concurrència, testing, Git, CI, Docker.
- **Per què primer**: és la base que permet a un junior ser útil en qualsevol equip tecnològic.
- **Resultat buscats**: construir codi fiable, llegible i validable sobre infraestructura que controles.

### Fase 2 — Productivitat i software operatiu (S9–S13)
- **REST + contractes + SQL + observabilitat + autenticació + Streamlit**.
- **Per què ara**: són les habilitats més presents en projectes reals d'empresa i d'API/backend.
- **Resultat buscats**: servei que funciona, s'administra, es monitoritza i es desplega sense drama.

### Fase 3 — IA aplicada amb rigor (S14–S17)
- **Knowledge engineering, retrieval, agents amb tool use, spec-driven development**.
- **Per què després**: la IA és multiplicadora, no substitut de la base tècnica.
- **Resultat buscats**: treballar amb IA de manera productiva i segura, no com un chatbot.

### Fase 4 — Infraestructura avançada i producció (S18–S24)
- **SQL avançat, message queues, CI/CD, deploy cloud, C4, docs, GitHub Pages, demo i preparació d'entrevista**.
- **Per què al final**: la demostració professional és la part que converteix el projecte en peça de portfolio.

### Resum executiu
La priorització ideal és:
1. **Construcció de software real + infraestructura bàsica**
2. **APIs, integració i seguretat**
3. **IA aplicada amb disciplina**
4. **Producció, portfolio i presentació**

> En una frase: primer solidesa tècnica, després productivitat amb IA, i finalment demostració professional.

## Software Necessari

### Obligatori des de S1

| Eina | Versió | Instal·lació | Per què |
|------|--------|-------------|---------|
| **OpenJDK** | 21+ | [Oracle](https://docs.oracle.com/en/java/javase/21/install/) o `brew install openjdk@21` | Llenguatge principal del backend |
| **Maven** | 3.9+ | `brew install maven` o inclòs amb IDE | Build i dependències Java |
| **Python** | 3.11+ | [python.org](https://www.python.org/downloads/) o `brew install python@3.12` | Llenguatge secundari, IA, dashboard |
| **Git** | 2.40+ | `brew install git` (macOS) / [git-scm.com](https://git-scm.com) | Control de versions |
| **Cursor IDE** | Latest | [cursor.sh](https://cursor.sh) | IDE amb IA integrada (alternativa: VS Code + extensió Claude) |
| **Compte GitHub** | — | [github.com](https://github.com) | Repositori, CI/CD, PRs |
| **Compte Riot Developer** | — | [developer.riotgames.com](https://developer.riotgames.com) (gratuït) | API key per a dades de League of Legends |

### S'afegeix al llarg del curs

| Eina | Quan s'introdueix | Instal·lació | Per què |
|------|-------------------|-------------|---------|
| **Docker Desktop** | S8 (Docker, Qdrant, PostgreSQL) | [docker.com](https://www.docker.com/products/docker-desktop/) | Contenidors per BD, serveis, deploy |
| **Postman** o **curl** | S3 / S9 | [postman.com](https://www.postman.com/downloads/) / ja instal·lat (macOS) | Testejar APIs REST manualment |
| **Streamlit** | S13 | `pip install streamlit` | Dashboard / spec-driven UI |
| **Qdrant** | S14 | Via Docker (`docker run qdrant/qdrant`) | Base vectorial per knowledge retrieval |
| **Redis** | S12 (sessions), S15 (cache LLM) | Via Docker (`docker run redis:7`) | Cache de respostes, sessions, queries |
| **PostgreSQL** | S18 | Via Docker (`docker run postgres:16`) | BD de producció (reemplaça H2) |
| **RabbitMQ** | S19 (message queues) | Via Docker (`docker run rabbitmq:3-management`) | Events asíncrons entre serveis |
| **Claude Code CLI** | S16+ (agents) | [claude.ai/claude-code](https://claude.ai/claude-code) | Agent de codi per terminal |
| **Compte PandaScore** | S4 (extractor) | [pandascore.co](https://pandascore.co) (gratuït) | API de tornejos i resultats d'eSports |

### Configuració Python recomanada

```bash
# Crear entorn virtual (S6 formalment, però recomanat des de S1)
python3 -m venv .venv
source .venv/bin/activate  # macOS/Linux

# Dependències base (creixen al llarg del curs)
pip install pytest ruff requests           # S1-S7
pip install fastapi uvicorn pydantic       # S10-S11
pip install structlog tenacity             # S11
pip install streamlit plotly               # S13
pip install aiohttp qdrant-client          # S14-S15
pip install python-jose bcrypt             # S12
pip install redis                          # S12, S15
pip install pika                           # S19
pip install langfuse openai anthropic      # S16
```

### Configuració Java recomanada

```bash
# Verificar versió
java --version   # Ha de ser 21+
mvn --version    # Ha de ser 3.9+

# El projecte usa Spring Boot 3 amb Maven
# Les dependències s'afegeixen al pom.xml progressivament:
# S1-S5: spring-boot-starter, junit-jupiter
# S6:    spring-boot-starter-data-jpa, h2
# S9:    spring-boot-starter-web, springdoc-openapi
# S12:   spring-boot-starter-security
# S18:   postgresql, flyway-core
```

### Claus API (fitxer `.env` a l'arrel, exclòs de Git)

```bash
# .env — MAI pujar a Git (afegir a .gitignore)
RIOT_API_KEY=your_riot_key_here        # developer.riotgames.com
PANDASCORE_API_KEY=your_key_here       # pandascore.co (tornejos)
OPENAI_API_KEY=your_key_here      # S10+
ANTHROPIC_API_KEY=your_key_here   # S10+ (alternativa)
```

## Estructura del Curs

| Bloc | Setmanes | Focus | Ratio mà/assistit |
|------|----------|-------|-------------------|
| **1. Fundaments** | S1–S8 | Algorítmica, POO, Linux/terminal, concurrència, testing, CI, Docker | 80% / 20% |
| **2. APIs, Integració i Seguretat** | S9–S13 | REST, Pydantic, LLMs, error handling, auth (JWT), Streamlit, MCP | 50% / 50% |
| **3. Knowledge, Agents i Spec-Driven** | S14–S17 | Knowledge engineering, retrieval, agents amb tool use, spec-driven dev | 30% / 70% |
| **4. Infraestructura Avançada** | S18–S20 | SQL avançat, PostgreSQL, Redis, message queues, CI/CD, monitoring | 20% / 80% |
| **5. Producció i Portfolio** | S21–S24 | Specs finals, deploy cloud, E2E, hardening, GitHub Pages, demo | 10% / 90% |

## Competències Transversals

El curs té 5 fils que es treballen de forma progressiva cada setmana (no en blocs aïllats):

- **Prompt Engineering** — De prompts bàsics (S1) a specs per agents (S17).
- **Python com a Segon Llenguatge** — Exercicis mirall Java↔Python des de S1. A S10+ Python és co-protagonista.
- **Escriptura de Specs** — `.cursorrules` (S2) → API specs (S9) → specs d'agent (S16) → spec-driven development (S17).
- **Ecosistema d'Eines IA** — Hooks (S5) → MCP servers (S10-S11) → skills i plugins (S14, S17) → consolidació (S21).
- **Mindset de Codi Productiu** — Observabilitat, mantenibilitat, escalabilitat, resiliència i operabilitat. Des de S1 (benchmarks + logging bàsic) fins a S23 (troubleshooting amb logs i mètriques). L'objectiu no és formar un developer que escriu codi que funciona, sinó un enginyer de SW que escriu codi que es pot operar en producció.

## Contingut per Setmana

### Bloc 1: Fundaments (S1–S8)

| Setmana | Tema | Dilluns | Dimarts | Dimecres | Dijous | Divendres |
|---------|------|---------|---------|----------|--------|-----------|
| **S1** | Rendiment i Big-O | [Entorn i Git](temari-setmanal/setmana01/setmana01_01_dilluns.md) | [Domini i Col·leccions](temari-setmanal/setmana01/setmana01_02_dimarts.md) | [Benchmark 100K](temari-setmanal/setmana01/setmana01_03_dimecres.md) | [Python Mirror](temari-setmanal/setmana01/setmana01_04_dijous.md) | [Tests i PR](temari-setmanal/setmana01/setmana01_05_divendres.md) |
| **S2** | POO, SOLID i Immutabilitat | [SOLID i Records](temari-setmanal/setmana02/setmana02_01_dilluns.md) | [Interfaces i Repository](temari-setmanal/setmana02/setmana02_02_dimarts.md) | [Dependency Inversion](temari-setmanal/setmana02/setmana02_03_dimecres.md) | [Python Dataclasses](temari-setmanal/setmana02/setmana02_04_dijous.md) | [Integració i PR](temari-setmanal/setmana02/setmana02_05_divendres.md) |
| **S3** | Linux, Terminal i Sistema | [SO i Terminal](temari-setmanal/setmana03/setmana03_01_dilluns.md) | [Permisos i PATH](temari-setmanal/setmana03/setmana03_02_dimarts.md) | [Pipes i Redirecció](temari-setmanal/setmana03/setmana03_03_dimecres.md) | [Bash Scripting](temari-setmanal/setmana03/setmana03_04_dijous.md) | [Xarxes, SSH i curl](temari-setmanal/setmana03/setmana03_05_divendres.md) |
| **S4** | Concurrència Pràctica | [Threads i Race Conditions](temari-setmanal/setmana04/setmana04_01_dilluns.md) | [CompletableFuture](temari-setmanal/setmana04/setmana04_02_dimarts.md) | [Python asyncio](temari-setmanal/setmana04/setmana04_03_dimecres.md) | [Correlation IDs](temari-setmanal/setmana04/setmana04_04_dijous.md) | [Extractor Paral·lel](temari-setmanal/setmana04/setmana04_05_divendres.md) |
| **S5** | Clean Code, Git, CI | [Anti-patrons IA](temari-setmanal/setmana05/setmana05_01_dilluns.md) | [Git Rebase](temari-setmanal/setmana05/setmana05_02_dimarts.md) | [GitHub Actions](temari-setmanal/setmana05/setmana05_03_dimecres.md) | [Specs i Hooks](temari-setmanal/setmana05/setmana05_04_dijous.md) | [Tag v0.1 i PR](temari-setmanal/setmana05/setmana05_05_divendres.md) |
| **S6** | Patrons, Repository, JPA | [Repository Pattern](temari-setmanal/setmana06/setmana06_01_dilluns.md) | [Spring Data JPA](temari-setmanal/setmana06/setmana06_02_dimarts.md) | [SQL Fonamental](temari-setmanal/setmana06/setmana06_03_dimecres.md) | [Python SQLite](temari-setmanal/setmana06/setmana06_04_dijous.md) | [Entorn Python i PR](temari-setmanal/setmana06/setmana06_05_divendres.md) |
| **S7** | Testing, Mocks, Qualitat | [JUnit 5 Avançat](temari-setmanal/setmana07/setmana07_01_dilluns.md) | [Mockito](temari-setmanal/setmana07/setmana07_02_dimarts.md) | [pytest i mock](temari-setmanal/setmana07/setmana07_03_dimecres.md) | [Cobertura i CI](temari-setmanal/setmana07/setmana07_04_dijous.md) | [Filosofia de Testing](temari-setmanal/setmana07/setmana07_05_divendres.md) |
| **S8** | Docker i Docker Compose | [Què és Docker](temari-setmanal/setmana08/setmana08_01_dilluns.md) | [Dockerfile](temari-setmanal/setmana08/setmana08_02_dimarts.md) | [Docker Compose](temari-setmanal/setmana08/setmana08_03_dimecres.md) | [Volums i Health](temari-setmanal/setmana08/setmana08_04_dijous.md) | [Observabilitat](temari-setmanal/setmana08/setmana08_05_divendres.md) |

### Bloc 2: APIs, Integració i Seguretat (S9–S13)

| Setmana | Tema | Dilluns | Dimarts | Dimecres | Dijous | Divendres |
|---------|------|---------|---------|----------|--------|-----------|
| **S9** | APIs REST, Spring Boot 3 | [Disseny REST](temari-setmanal/setmana09/setmana09_01_dilluns.md) | [CRUD i Validació](temari-setmanal/setmana09/setmana09_02_dimarts.md) | [Virtual Threads](temari-setmanal/setmana09/setmana09_03_dimecres.md) | [Request Logging](temari-setmanal/setmana09/setmana09_04_dijous.md) | [CLI Python i PR](temari-setmanal/setmana09/setmana09_05_divendres.md) |
| **S10** | Pydantic, LLMs, MCP | [Pydantic Models](temari-setmanal/setmana10/setmana10_01_dilluns.md) | [API Claude/OpenAI](temari-setmanal/setmana10/setmana10_02_dimarts.md) | [FastAPI](temari-setmanal/setmana10/setmana10_03_dimecres.md) | [MCP Basics](temari-setmanal/setmana10/setmana10_04_dijous.md) | [Integració i PR](temari-setmanal/setmana10/setmana10_05_divendres.md) |
| **S11** | Error Handling, Logging | [Excepcions](temari-setmanal/setmana11/setmana11_01_dilluns.md) | [Logging Estructurat](temari-setmanal/setmana11/setmana11_02_dimarts.md) | [Correlation IDs](temari-setmanal/setmana11/setmana11_03_dimecres.md) | [MCP Server Propi](temari-setmanal/setmana11/setmana11_04_dijous.md) | [Resiliència i PR](temari-setmanal/setmana11/setmana11_05_divendres.md) |
| **S12** | Autenticació i Seguretat | [Auth i JWT](temari-setmanal/setmana12/setmana12_01_dilluns.md) | [Spring Security](temari-setmanal/setmana12/setmana12_02_dimarts.md) | [JWT Filter i Roles](temari-setmanal/setmana12/setmana12_03_dimecres.md) | [FastAPI JWT + Redis](temari-setmanal/setmana12/setmana12_04_dijous.md) | [Headers i OWASP](temari-setmanal/setmana12/setmana12_05_divendres.md) |
| **S13** | Streamlit i Testing E2E | [Intro Streamlit](temari-setmanal/setmana13/setmana13_01_dilluns.md) | [Dashboard i Login](temari-setmanal/setmana13/setmana13_02_dimarts.md) | [Wireframe-Driven](temari-setmanal/setmana13/setmana13_03_dimecres.md) | [Testing E2E](temari-setmanal/setmana13/setmana13_04_dijous.md) | [Consolidació i PR](temari-setmanal/setmana13/setmana13_05_divendres.md) |

### Bloc 3: Knowledge, Agents i Spec-Driven (S14–S17)

| Setmana | Tema | Dilluns | Dimarts | Dimecres | Dijous | Divendres |
|---------|------|---------|---------|----------|--------|-----------|
| **S14** | Knowledge Engineering | [Per Què la IA Falla](temari-setmanal/setmana14/setmana14_01_dilluns.md) | [Chunking i Metadades](temari-setmanal/setmana14/setmana14_02_dimarts.md) | [Embeddings i Qdrant](temari-setmanal/setmana14/setmana14_03_dimecres.md) | [Skills i MCP](temari-setmanal/setmana14/setmana14_04_dijous.md) | [@docs i Consolidació](temari-setmanal/setmana14/setmana14_05_divendres.md) |
| **S15** | Retrieval i Anti-al·lucinació | [Retrieval Semàntic](temari-setmanal/setmana15/setmana15_01_dilluns.md) | [Anti-al·lucinació](temari-setmanal/setmana15/setmana15_02_dimarts.md) | [Evals amb pytest](temari-setmanal/setmana15/setmana15_03_dimecres.md) | [Cache Redis i Cost](temari-setmanal/setmana15/setmana15_04_dijous.md) | [Spec i Consolidació](temari-setmanal/setmana15/setmana15_05_divendres.md) |
| **S16** | Agents: Tool Use i Evals | [Patró Agent](temari-setmanal/setmana16/setmana16_01_dilluns.md) | [Agent Quantitatiu](temari-setmanal/setmana16/setmana16_02_dimarts.md) | [Agent Knowledge](temari-setmanal/setmana16/setmana16_03_dimecres.md) | [Evals i LangFuse](temari-setmanal/setmana16/setmana16_04_dijous.md) | [OpenSpec](temari-setmanal/setmana16/setmana16_05_divendres.md) |
| **S17** | Spec-Driven Development | [Anatomia d'una Spec](temari-setmanal/setmana17/setmana17_01_dilluns.md) | [Escriure la Spec](temari-setmanal/setmana17/setmana17_02_dimarts.md) | [Generació i Review](temari-setmanal/setmana17/setmana17_03_dimecres.md) | [Skills i Hooks](temari-setmanal/setmana17/setmana17_04_dijous.md) | [Mètriques i Retro](temari-setmanal/setmana17/setmana17_05_divendres.md) |

### Bloc 4: Infraestructura Avançada (S18–S20)

| Setmana | Tema | Dilluns | Dimarts | Dimecres | Dijous | Divendres |
|---------|------|---------|---------|----------|--------|-----------|
| **S18** | SQL Avançat, PostgreSQL | [Migració i Flyway](temari-setmanal/setmana18/setmana18_01_dilluns.md) | [Normalització i FK](temari-setmanal/setmana18/setmana18_02_dimarts.md) | [JOINs i Window](temari-setmanal/setmana18/setmana18_03_dimecres.md) | [EXPLAIN ANALYZE](temari-setmanal/setmana18/setmana18_04_dijous.md) | [Cache-Aside Redis](temari-setmanal/setmana18/setmana18_05_divendres.md) |
| **S19** | Redis i Message Queues | [Sync vs Async](temari-setmanal/setmana19/setmana19_01_dilluns.md) | [RabbitMQ Productor](temari-setmanal/setmana19/setmana19_02_dimarts.md) | [Consumidor Python](temari-setmanal/setmana19/setmana19_03_dimecres.md) | [Retry i Idempotència](temari-setmanal/setmana19/setmana19_04_dijous.md) | [Tests Integració](temari-setmanal/setmana19/setmana19_05_divendres.md) |
| **S20** | CI/CD i Monitoring | [Audit CI Workflows](temari-setmanal/setmana20/setmana20_01_dilluns.md) | [docker-compose Complet](temari-setmanal/setmana20/setmana20_02_dimarts.md) | [Health Endpoints](temari-setmanal/setmana20/setmana20_03_dimecres.md) | [Dashboard Mètriques](temari-setmanal/setmana20/setmana20_04_dijous.md) | [Consolidació Bloc 4](temari-setmanal/setmana20/setmana20_05_divendres.md) |

### Bloc 5: Producció i Portfolio (S21–S24)

| Setmana | Tema | Dilluns | Dimarts | Dimecres | Dijous | Divendres |
|---------|------|---------|---------|----------|--------|-----------|
| **S21** | Specs Finals i Dashboard | [CLAUDE.md Definitiu](temari-setmanal/setmana21/setmana21_01_dilluns.md) | [.cursorrules Finals](temari-setmanal/setmana21/setmana21_02_dimarts.md) | [Dashboard Avançat](temari-setmanal/setmana21/setmana21_03_dimecres.md) | [Test Agent Nou](temari-setmanal/setmana21/setmana21_04_dijous.md) | [Consolidació i PR](temari-setmanal/setmana21/setmana21_05_divendres.md) |
| **S22** | Desplegament al Núvol | [Auditoria Seguretat](temari-setmanal/setmana22/setmana22_01_dilluns.md) | [Deploy Backend](temari-setmanal/setmana22/setmana22_02_dimarts.md) | [Deploy Serveis](temari-setmanal/setmana22/setmana22_03_dimecres.md) | [Domini i HTTPS](temari-setmanal/setmana22/setmana22_04_dijous.md) | [Verificació Producció](temari-setmanal/setmana22/setmana22_05_divendres.md) |
| **S23** | E2E, Hardening, Arquitectura | [Tests E2E](temari-setmanal/setmana23/setmana23_01_dilluns.md) | [Load Test i OWASP](temari-setmanal/setmana23/setmana23_02_dimarts.md) | [Diagrames C4](temari-setmanal/setmana23/setmana23_03_dimecres.md) | [Troubleshooting](temari-setmanal/setmana23/setmana23_04_dijous.md) | [Documentació Final](temari-setmanal/setmana23/setmana23_05_divendres.md) |
| **S24** | GitHub Pages i Portfolio | [GitHub Pages](temari-setmanal/setmana24/setmana24_01_dilluns.md) | [Integrar Projecte](temari-setmanal/setmana24/setmana24_02_dimarts.md) | [Polir Repositori](temari-setmanal/setmana24/setmana24_03_dimecres.md) | [Demo 5 Minuts](temari-setmanal/setmana24/setmana24_04_dijous.md) | [Presentació Final](temari-setmanal/setmana24/setmana24_05_divendres.md) |

## Estructura de Fitxers per Setmana

Cada setmana té 5 fitxers diaris que combinen teoria i pràctica:

```
temari-setmanal/setmanaXX/
├── setmanaXX_01_dilluns.md      # Dilluns — Teoria + Activitat
├── setmanaXX_02_dimarts.md      # Dimarts — Teoria + Activitat
├── setmanaXX_03_dimecres.md     # Dimecres — Teoria + Activitat
├── setmanaXX_04_dijous.md       # Dijous — Teoria + Activitat
└── setmanaXX_05_divendres.md    # Divendres — Integració + PR
```

Cada fitxer diari segueix el format:
- **Objectiu del Dia** — Què sabrà fer l'estudiant al final del dia.
- **Teoria** — Conceptes amb exemples de codi comentats (Java + Python).
- **Activitat** — Exercicis guiats pas a pas amb temps estimats.
- **Checklist de Lliurament** — Verificació que tot funciona.

Tots els code snippets inclouen comentaris que expliquen **què fa** cada línia i **per què**.

## Documents Generals

- [Pla Global de 24 Setmanes](temari-global-24.md) — Visió completa amb competències transversals.
- [EsportsPulse Project Brief](esportspulse-project-brief.md) — Descripció del projecte vehicle, stack, roadmap.
- [Proposta de Reestructuració](proposta-reestructuracio.md) — Detall dels canvis aplicats respecte la versió original.

## Principis del Curs

1. **Primer el dolor, després l'eina.** Les eines IA s'introdueixen quan l'estudiant ha viscut el problema que resolen.
2. **Empleabilitat sobre acadèmia.** Cada tema es justifica per "això ho faràs al primer mes de feina" o "això et preguntaran a l'entrevista", no per completesa teòrica.
3. **L'algorítmica necessària, no més.** Big-O i HashMap (S1) són suficients com a base. La resta s'integra quan el context ho demana (indexes a S6, concurrència a S4).
4. **La IA és copilot, tu ets responsable.** L'estudiant aprèn a generar codi amb IA i a auditar-lo amb criteri: seguretat, rendiment, tests, mantenibilitat.
5. **Java + Python des del dia 1.** Exercicis mirall en Python cada setmana per demostrar que els conceptes són universals.
6. **Entendre la màquina.** L'estudiant aprèn Linux, terminal i conceptes de sistema (S3) perquè saber on corre el teu codi és el primer pas per operar-lo.

## Llicència i Recursos Externs

El contingut original d'aquest curs (temaris, exercicis, teoria) es distribueix sota [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).

El material inclou links a recursos externs (articles, vídeos, documentació, llibres) com a referències educatives. Aquests recursos són propietat dels seus respectius autors i es referencien sense reproduir-ne el contingut:
- **Articles:** Baeldung, Atlassian, Google Engineering Practices (accés públic gratuït).
- **Vídeos:** YouTube (Midudev, MoureDev, i altres — contingut públic dels creadors).
- **Llibres:** *Pro Git* (Scott Chacon, CC BY-NC-SA 3.0), *Clean Code* (Robert C. Martin, Prentice Hall). Es referencien per títol i autor; no es reprodueix contingut.
- **Documentació:** Oracle Java, Spring, JUnit, GitHub (documentació oficial pública).

Si ets autor d'algun recurs referenciat i prefereixes que eliminem el link, obre un issue.
