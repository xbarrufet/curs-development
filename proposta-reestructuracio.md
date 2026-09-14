
# Proposta de Reestructuració — Pla de 24 Setmanes

**Canvis aplicats respecte al temari original:**

| Feedback | Canvi aplicat |
|----------|--------------|
| S11 (Docker) a Fundaments | Docker passa a S8, tanca el Bloc 1 |
| Autenticació (S14) al Bloc 2 | Auth s'agrupa amb APIs a S12 |
| Bloc standalone d'Agents | Nou Bloc 3: Knowledge + Agents + Spec-Driven |
| Juntar S17 amb S12+S13 | Agents, Knowledge Eng. i Retrieval van junts |
| Un sol bloc AI dev + spec-driven | Bloc 3 unifica tot el flux IA |
| GitHub Pages | S24 inclou creació de pàgina personal |
| Linux i HW bàsics | Nova S3 de Linux, terminal i scripting |

---

## Bloc 1: Fundaments, POO, Infra Bàsica (80% a mà / 20% assistit) — S1-S8

### Setmana 1 — Rendiment i Big-O
*Igual que l'original.*
Programació a mà de cerques O(n) en `List`/`list` vs O(1) en `HashMap`/`dict`. Benchmark Java + Python. Logging bàsic (`java.util.logging`) al benchmark. Primer contacte amb instrumentació.

**Lliurament:** Executable de benchmarking en Java 21 + script equivalent en Python.

### Setmana 2 — POO i SOLID
*Igual que l'original.*
Models immutables (`records` / `@dataclass(frozen=True)`), regles d'arquitectura a `.cursorrules`. Exercici Python: `ChampionRecord` com a `dataclass` frozen. Primer `.cursorrules` complet per al projecte.

**Lliurament:** Domini `ChampionRecord` i interfícies `Repository` en Java + equivalents Python.

### Setmana 3 — Linux, Terminal i Conceptes de Sistema (NOU)
Objectiu: que l'estudiant es mogui amb comoditat per un terminal i entengui els conceptes bàsics del sistema on corre el seu codi.

- **Sistema operatiu:** Què és un procés, un thread, memòria RAM vs disc, CPU. Suficient perquè Big-O (S1) i concurrència (S4) tinguin context físic.
- **Terminal i shell:** Navegació (`cd`, `ls`, `pwd`, `find`, `grep`), permisos (`chmod`, `chown`), variables d'entorn, `$PATH`. Pipes i redirecció (`|`, `>`, `>>`, `2>&1`).
- **Bash scripting:** Escriure un script que automatitzi una tasca repetitiva del projecte (ex: compilar, executar tests, netejar artifacts). Condicionals, bucles, funcions bàsiques.
- **SSH i xarxes bàsiques:** Què és una IP, un port, DNS. Connexió SSH a una màquina remota (preparació per al desplegament de S22). `curl` per fer peticions HTTP des del terminal (preparació per APIs de S9).
- **Gestió de paquets:** `apt`/`brew`, entendre què fa un package manager (connexió conceptual amb `pip`, `npm`, `mvn`).
- **Exercici integrador:** Script bash que descarrega dades d'una API pública (ex: Riot Data Dragon), les processa amb `jq`, i les guarda en un fitxer. L'estudiant veu que el terminal és una eina de productivitat, no un obstacle.

**Lliurament:** Script bash funcional amb documentació; l'estudiant pot navegar, editar fitxers, i executar comandes sense dependre d'una GUI.

### Setmana 4 — Concurrència Pràctica per a Developers Web
*Era S3.*
Race conditions, `synchronized`, `AtomicInteger`, `CompletableFuture` (Java). Python: `threading`, GIL, `asyncio` + `aiohttp`. Traçabilitat: `requestId` per cada crida concurrent. Primer exercici de correlation ID.

**Lliurament:** Extractor paral·lel d'APIs Riot/Data Dragon en Java + versió asyncio en Python.

### Setmana 5 — Clean Code, Code Review, Git Workflow & CI
*Era S4.*
Refactorització d'anti-patrons en codi generat per IA. Code review com a skill. Git rebase, resolució de conflictes, GitHub Actions bàsic. Hook `pre-commit` amb Checkstyle/ruff. Primer exercici de spec de refactorització.

**Lliurament:** Tag `v0.1` en repositori estructurat, badge CI al README.

### Setmana 6 — Patrons Factory, Repository i Persistència JPA
*Era S5.*
Jerarquia d'interfícies per desacoblar accés a dades. Spring Data JPA amb H2. SQL pur a la consola H2. Python: `ChampionRepository` amb SQLite.

**Lliurament:** Capa d'abstracció d'ingesta amb persistència real en BD local.

### Setmana 7 — Testing, Mocks i Qualitat de Codi
*Era S6.*
JUnit 5 + Mockito. GitHub Actions amb cobertura mínima 70%. Python: `pytest` + `unittest.mock` + `ruff` al CI.

**Lliurament:** Suite de tests Java + Python amb cobertura mesurable; CI que valida els dos llenguatges.

### Setmana 8 — Docker i Docker Compose
*Era S11. Mogut a Fundaments perquè la infraestructura base estigui llesta abans de les APIs.*

Introducció a Docker: `Dockerfile`, `docker build`, `docker run`. Dockeritzar el backend Java i el servei Python. `docker-compose.yml` amb PostgreSQL + Qdrant. Networking, volums, health checks, environment variables.

**Connexió amb S3 (Linux):** L'estudiant ja sap navegar el terminal, entén processos i ports — Docker és una extensió natural.

**Mindset productiu:** `docker logs`, `docker-compose logs -f`. L'estudiant veu per què el format JSON i els correlation IDs (S4) importen quan 4 contenidors barregen els seus logs.

**Lliurament:** `docker-compose up` arrenca els serveis d'infra (BD, Qdrant) amb una sola comanda; logs llegibles i health checks configurats.

---

## Bloc 2: APIs REST, Integració i Seguretat (50% a mà / 50% assistit) — S9-S13

### Setmana 9 — APIs REST amb Spring Boot 3, Virtual Threads i Persistència
*Era S7.*
DTOs, contractes, controladors HTTP. Virtual Threads amb `spring.threads.virtual.enabled=true`. Python: CLI que consumeix l'API amb `requests`. Spec d'API en markdown. Request logging middleware.

**Lliurament:** Endpoints REST en Java amb Virtual Threads; request logging; CLI Python funcional.

### Setmana 10 — Python, Pydantic i Output Estructurat amb LLMs
*Era S8.*
Pydantic models per validar sortides d'LLM. API de Claude/OpenAI. System prompts i few-shot. MCP basics: instal·lar `mcp-server-sqlite`/`mcp-server-postgres` per connectar l'agent a la BD.

**Lliurament:** Servei Python (FastAPI) que converteix consultes en JSONs via LLM; MCP server connectat a la BD.

### Setmana 11 — Error Handling, Logging i Integració entre Serveis
*Era S9.*
Gestió d'excepcions, retry amb backoff. Logging estructurat (JSON) a Java i Python (`structlog`). Correlation IDs propagats entre serveis. MCP server propi: `search_logs(query, severity, last_minutes)`.

**Lliurament:** Serveis Java i Python comunicant-se amb gestió d'errors robusta; MCP server de logs funcional.

### Setmana 12 — Autenticació, Autorització i Seguretat Web
*Era S14. Mogut al Bloc 2 per agrupar-ho amb les APIs.*

Spring Security: login, JWT tokens, `@PreAuthorize`, roles (USER/ADMIN). CORS, CSRF, headers de seguretat. Bcrypt. Session store amb Redis. Python: protegir endpoints FastAPI amb JWT.

**Raó del moviment:** L'autenticació és part natural del desenvolupament d'APIs — l'estudiant acaba de crear endpoints (S9) i ara els protegeix. No té sentit esperar fins al Bloc 3.

**Lliurament:** API REST protegida amb JWT; endpoints accessibles només per usuaris autenticats.

### Setmana 13 — Interfície Streamlit, Integració i Testing End-to-End
*Era S10.*
Dashboard Streamlit amb login (aprofitant l'auth de S12). Formularis, taules, visualització. Spec del dashboard en wireframe ASCII. CI ampliat per testejar Python.

**Lliurament:** Dashboard Streamlit funcional amb login que opera sobre l'API REST; CI complet.

---

## Bloc 3: Knowledge, Agents i Spec-Driven Development (30% a mà / 70% assistit) — S14-S17

> **Bloc unificat:** Agrupa knowledge engineering, retrieval, agents i spec-driven en un sol flux coherent. L'estudiant passa d'estructurar informació → recuperar-la → fer-la servir amb agents → especificar agents formalment. Sense salts entre blocs.

### Setmana 14 — Knowledge Engineering: Estructurar Informació per a IA
*Era S12.*
Per què la IA falla amb informació mal estructurada. Patch notes reals → wiki markdown amb metadades. Chunking. Indexar a Qdrant (ja dockeritzat des de S8). Parser PDF → markdown. Buscar i avaluar skills, MCP servers, `.cursorrules` existents per al projecte.

**Lliurament:** Wiki estructurada indexada a Qdrant; skills i MCP servers documentats al `CLAUDE.md`.

### Setmana 15 — Knowledge Retrieval, Anti-al·lucinació i Cache
*Era S13.*
Retrieval semàntic. Prompts de restricció amb citació obligatòria. Anti-al·lucinació. Evals: 10 preguntes amb ground truth, pytest. Cache Redis per respostes d'LLM. Cost management d'APIs IA.

**Lliurament:** Sistema de retrieval amb citació, anti-al·lucinació, evals i cache Redis.

### Setmana 16 — Agents: Configurar, Avaluar i Observar
*Era S17. Ara va just després de knowledge — l'agent utilitza directament el que s'ha construït a S14-S15.*

Patró agent (prompt → LLM → eina → resultat → decisió). Configurar agents d'EsportsPulse amb **system prompts + tool schemas** via API directa (sense framework):
- Agent Quantitatiu: `query_champions(filters)`, `get_stats(championId)`
- Agent de Knowledge: `search_patches(query)`, `get_patch_detail(id)` — connectat al retrieval de S15

Eval dataset 15+ preguntes. Observabilitat amb LangFuse. OpenSpec per documentar agents.

**Connexió natural:** L'agent de Knowledge reutilitza la wiki (S14) i el retrieval (S15) directament. No hi ha salt temporal entre construir el knowledge i usar-lo.

**Lliurament:** Agents via API directa amb tool use, spec documentada, evals al CI, traces a LangFuse.

### Setmana 17 — Spec-Driven Development: Feature Completa amb Agent
*Era S18.*
Exercici integrador: requisit nou → spec completa → agent implementa → code review → tests → iteració. Crear skill `/review-esportspulse`. Hook `pre-push` amb evals automàtics.

**Avaluació:** Qualitat de la spec, no del codi generat. Bona spec = codi correcte en 1-2 intents.

**Lliurament:** Feature implementada per agent amb spec; skill funcional; informe d'iteracions.

---

## Bloc 4: Infraestructura Avançada i Integració (20% a mà / 80% assistit) — S18-S20

### Setmana 18 — SQL Avançat i PostgreSQL
*Era S15.*
Migrar d'H2 a PostgreSQL (ja dockeritzat des de S8). Disseny normalitzat, Flyway, JOINs, GROUP BY, window functions. Indexes, `EXPLAIN ANALYZE`. Cache-aside amb Redis. Trobar i optimitzar query N+1.

**Lliurament:** BD PostgreSQL normalitzada, migracions, queries optimitzades, cache Redis.

### Setmana 19 — Integració de Sistemes: Redis i Message Queues
*Era S16.*
Consolidació Redis. RabbitMQ en Docker. Event `champion.ingested` → processament asíncron amb LLM → cache. Dead letter queue, retry, idempotència.

**Lliurament:** Flux complet POST → event → processament asíncron → cache; tests d'integració.

### Setmana 20 — Consolidació CI/CD, Observabilitat i Monitoring
*Era S20.*
Refactorització dels workflows CI acumulats. Dockeritzar tots els serveis restants. `docker-compose up` arrenca tota la plataforma. Health endpoints, dashboard de mètriques.

**Lliurament:** Pipeline CI/CD consolidat; plataforma dockeritzada; dashboard operacional.

---

## Bloc 5: Producció i Portfolio (10% a mà / 90% assistit — Rol d'Auditor) — S21-S24

### Setmana 21 — Especificacions Finals i Dashboard Avançat
*Era S19.*
`CLAUDE.md` definitiu, `.cursorrules` finals, specs d'agents. Dashboard Streamlit avançat. Exercici: donar el `CLAUDE.md` a un agent "nou" i verificar que pot contribuir sense ajuda.

**Lliurament:** Repositori amb specs completes; ecosistema IA documentat; dashboard complet.

### Setmana 22 — Desplegament al Núvol
*Era S21.*
Seguretat de claus API. Desplegament a Render/Fly.io. Variables d'entorn en producció. Domini i HTTPS.

**Connexió amb S3 (Linux):** L'estudiant ja sap SSH, ports, DNS — el desplegament és aplicar-ho en un servidor real.

**Lliurament:** Desplegament real accessible via URL pública.

### Setmana 23 — Testing E2E, Hardening i Arquitectura
*Era S22 + S23 (comprimides).*
Tests d'integració del flux complet. Load testing bàsic. OWASP checklist, rate limiting. Diagrames C4 (Context, Container, Component) amb fluxos d'observabilitat. Exercici capstone: troubleshooting amb logs i mètriques (sense codi font).

**Lliurament:** Suite E2E; informe de seguretat; diagrames C4; exercici troubleshooting resolt.

### Setmana 24 — GitHub Pages, Portfolio i Presentació
*Era S24 + GitHub Pages (NOU).*

**GitHub Pages (NOU):**
- Crear pàgina personal amb Jekyll/HTML a `username.github.io`.
- Seccions: sobre mi, projectes, skills tècnics, contacte.
- Desplegar amb GitHub Actions (reutilitzant coneixements de CI de S5).
- Enllaçar el projecte EsportsPulse com a peça principal del portfolio.

**Portfolio i Demo:**
- Repositori pinned a GitHub amb description i topics.
- README professional amb API docs (Swagger/OpenAPI).
- Demo de 5 minuts del sistema complet.
- Assajar explicar decisions tècniques per entrevistes.

**Lliurament:** Pàgina personal a GitHub Pages; repositori final amb demo en viu; l'estudiant pot explicar qualsevol decisió del projecte.

---

## Resum de la redistribució

| Bloc | Setmanes | Tema | Ratio mà/assistit |
|------|----------|------|-------------------|
| 1 | S1-S8 (8) | Fundaments + Linux + Docker | 80/20 |
| 2 | S9-S13 (5) | APIs + Integració + Seguretat | 50/50 |
| 3 | S14-S17 (4) | Knowledge + Agents + Spec-Driven | 30/70 |
| 4 | S18-S20 (3) | Infra Avançada + CI/CD | 20/80 |
| 5 | S21-S24 (4) | Producció + Portfolio + GitHub Pages | 10/90 |

### Competències transversals — actualitzacions
- **Prompt Engineering:** Sense canvis, la progressió es manté.
- **Python:** Sense canvis, la introducció gradual continua.
- **Eines IA (MCP, Skills, Hooks):** L'ordre es manté coherent amb la nova numeració.
- **Specs per Agents:** La progressió ara culmina a S16-S17 dins d'un sol bloc.
- **Mindset Productiu:** Afegir a S3 (Linux): "Entendre el sistema on corre el teu codi és el primer pas per operar-lo."
