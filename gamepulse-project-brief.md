# GamePulse - Project Brief

## Descripció General

**GamePulse** és un sistema multi-agent per analitzar i monitoritzar el balanç i evolució de videojocs. El projecte integra:
- **Backend Java 21**: Serveis REST per ingesta de dades, cerca de jocs, i estadístiques
- **Python IA**: Agents per interpretar notes de parches, knowledge retrieval per trends de balanç
- **Observabilitat**: LangFuse per traçar el comportament dels agents
- **Interfície**: Dashboard Streamlit per consultar l'equip d'agents i visualitzar dades

## Objectiu Primari

**Formar un enginyer de software junior (no només un developer) en el cicle complet AI-SDLC**: des d'algorítmica bàsica fins a agents autònoms, passant per knowledge engineering, testing d'IA, i desplegament en cloud. El diferenciador: cada línia de codi es pensa per a producció — observabilitat, mantenibilitat, resiliència i escalabilitat són criteris des del dia 1, no un afegitó al final. Cada setmana afegeix una capa nova sense trencar la precedent.

## Scope: Backend + IA (No Fullstack)

Aquest curs **no inclou frontend web** (HTML/CSS/JavaScript/React). La decisió és conscient:
- El perfil objectiu és **junior backend + IA**, no fullstack.
- L'estudiant entén com un frontend consumiria la seva API (Swagger/OpenAPI, CORS, contractes JSON a S7) sense haver de construir-ne un.
- Si l'estudiant vol ampliar amb frontend, el projecte té una API REST documentada amb Swagger que qualsevol framework (React, Vue, etc.) pot consumir directament.

### Streamlit com a Vehicle de Spec-Driven Development

La interfície d'usuari del projecte és **Streamlit** (Python). No és "el frontend" — és un **vehicle per practicar el workflow spec → agent → review**:

- **Per què Streamlit:** Feedback visual immediat. Si la spec diu "taula amb filtres i gràfic de tendència", l'estudiant veu en 2 segons si l'agent ho ha generat correctament. Amb codi backend, verificar requereix tests o curl — més lent, menys intuïtiu per iterar specs.
- **Per què NO React/Vue:** Aprendre JS + JSX + state management + hooks + CSS requeriria 4+ setmanes que diluirien el focus backend+IA. No és el diferenciador d'aquest perfil.
- **Cas d'ús real:** A una empresa, al junior li diran "fes-me un dashboard per veure les mètriques dels agents" o "munta una demo per al client". Per a eines internes, dashboards i demos — Streamlit (o Gradio, Panel) és la resposta habitual en equips backend+IA.
- **El que Streamlit NO és:** Un framework de producció per UI d'usuari final. Per a això cal React/Vue/Angular — que és una formació separada. El curs no ho amaga.

**Progressió de Streamlit al curs:**
| Setmana | Exercici | Focus |
|---------|----------|-------|
| S10 | Dashboard bàsic que consumeix l'API REST | Aprendre Streamlit + spec amb wireframe ASCII |
| S19 | Dashboard avançat d'agents (state, historial, mètriques) | Generat per agent a partir de spec; l'estudiant audita |
| S24 | Demo final del sistema complet (5 min) | Streamlit com a capa de presentació del portfolio |

---

## APIs Públiques de Videojocs

GamePulse consumeix dades reals de videojocs. Les APIs recomanades (totes amb tier gratuït):

### Steam Web API (Principal)

| | |
|---|---|
| **URL base** | `https://api.steampowered.com` |
| **Documentació** | https://steamcommunity.com/dev |
| **Obtenir API key** | https://steamcommunity.com/dev/apikey (cal compte Steam gratuït) |
| **Rate limit** | 100.000 crides/dia (àmpliament suficient) |
| **Autenticació** | Query param `?key=YOUR_API_KEY` |

**Endpoints útils per GamePulse:**

```
# Obtenir detalls d'un joc per appId
GET https://store.steampowered.com/api/appdetails?appids=730
→ Retorna: nom, preu, descripció, categories, screenshots
→ No necessita API key (endpoint públic)

# Jugadors actius en temps real
GET https://api.steampowered.com/ISteamUserStats/GetNumberOfCurrentPlayers/v1/?appid=730
→ Retorna: { "player_count": 1234567 }
→ Necessita API key

# Llista de tots els jocs de Steam (~100.000)
GET https://api.steampowered.com/ISteamApps/GetAppList/v2/
→ Retorna: llista de { appid, name }
→ No necessita API key
```

**Limitacions:**
- `appdetails` té rate limit agressiu (~200 crides/5 minuts). Per ingesta massiva, usar `GetAppList` + caching.
- No retorna patch notes directament (les patch notes són a la Steam Community, no a l'API).

### RAWG Video Games Database API (Alternativa/Complement)

| | |
|---|---|
| **URL base** | `https://api.rawg.io/api` |
| **Documentació** | https://rawg.io/apidocs |
| **Obtenir API key** | https://rawg.io/login (registre gratuït) |
| **Rate limit** | 20.000 crides/mes (tier gratuït) |
| **Autenticació** | Query param `?key=YOUR_API_KEY` |

**Endpoints útils per GamePulse:**

```
# Cercar jocs per nom
GET https://api.rawg.io/api/games?key=YOUR_KEY&search=league+of+legends
→ Retorna: nom, rating, metacritic, platforms, released, screenshots

# Detalls d'un joc
GET https://api.rawg.io/api/games/{id}?key=YOUR_KEY
→ Retorna: descripció completa, tags, publishers, ESRB rating

# Llista de jocs per gènere/plataforma
GET https://api.rawg.io/api/games?key=YOUR_KEY&genres=action&platforms=4
→ Retorna: llista filtrada amb paginació
```

**Avantatge sobre Steam:** API més neta, millor documentada, retorna dades de TOTES les plataformes (no només PC). Ideal per a l'exercici de S3 (extractor concurrent) perquè té paginació clara.

### IGDB (Avançat — Opcional)

| | |
|---|---|
| **URL base** | `https://api.igdb.com/v4` |
| **Documentació** | https://api-docs.igdb.com |
| **Obtenir accés** | Requereix compte Twitch Developer (https://dev.twitch.tv) + OAuth Client Credentials |
| **Rate limit** | 4 crides/segon (tier gratuït) |
| **Autenticació** | Bearer token via OAuth2 (més complex que Steam/RAWG) |

**Per què és opcional:** L'autenticació OAuth2 és més complexa. Recomanat per S13 (quan l'estudiant ja sap auth) com a exercici d'integració amb API autenticada, no per S3.

### Patch Notes (Per al Bloc 3: Knowledge Engineering)

Les patch notes de videojocs no vénen d'una API — són documents publicats pels developers del joc. Fonts recomanades:

- **League of Legends:** https://www.leagueoflegends.com/en-us/news/tags/patch-notes/ (HTML scrapable, historial complet)
- **Dota 2:** https://www.dota2.com/patches (HTML)
- **Valorant:** https://playvalorant.com/en-us/news/tags/patch-notes/ (HTML)

**Exercici S11:** L'estudiant escull un joc, descarrega 10-20 patch notes (HTML o PDF), i les estructura en la wiki per al knowledge retrieval.

### Gestió de Claus API al Projecte

```
# .env (exclòs de Git via .gitignore)
STEAM_API_KEY=your_steam_key_here
RAWG_API_KEY=your_rawg_key_here

# Java: application.properties
steam.api.key=${STEAM_API_KEY}
rawg.api.key=${RAWG_API_KEY}

# Python: os.environ
steam_api_key = os.environ["STEAM_API_KEY"]
```

**Regla del curs:** Mai secrets al codi. Sempre `.env` + `.gitignore`. Això es practica des de S4 (anti-patró de secrets hardcodejats).

---

## Roadmap de Features per Bloc

### **Bloc 1: Fundaments, POO i Concurrència (Setmanes 1-6)**

**Objectiu:** Domini bàsic de Java 21 + Python, estructura de projecte, i concurrència pràctica.

**Features:**
- **(S1)** Benchmark O(n) vs O(1): Search lineal vs HashMap per ID de joc
- **(S2)** Model `GameRecord` immutable (record / dataclass); DTO per jocs amb preu, jugadors actius
- **(S3)** Concurrència pràctica: race conditions, @Transactional, CompletableFuture, extractor concurrent d'APIs Steam/RAWG
- **(S4)** Code review, refactorització d'anti-patrons IA, Git workflow, CI bàsic
- **(S5)** Factory + Repository patterns amb JPA + H2; SQL pur a consola
- **(S6)** Suite JUnit 5 + pytest; coverage gates al CI

**Lliurament:** Codi Java + Python netejat, amb tests, versionat a `v0.1`

### **Bloc 2: APIs REST i Output Estructurat amb IA (Setmanes 7-10)**

**Objectiu:** Contractes REST clars, integració amb LLMs, primera interfície.

**Features:**
- **(S7)** Endpoints REST Spring Boot 3: CRUD de jocs, query params, Swagger, Virtual Threads. API spec com a exercici de specs.
- **(S8)** Python Pydantic models + LLM output estructurat (Claude/OpenAI API). MCP basics: connectar l'agent a la BD.
- **(S9)** Error handling inter-serveis Java↔Python, logging estructurat. MCP server personalitzat.
- **(S10)** Dashboard Streamlit que consumeix l'API REST. Spec del dashboard.

**Lliurament:** Backend REST funcional + dashboard Streamlit + MCP servers

### **Bloc 3: Infraestructura, Knowledge, Seguretat i Integració (Setmanes 11-16)**

**Objectiu:** Contenidors, knowledge retrieval, autenticació, BD real, caching, message queues.

**Features:**
- **(S11)** Docker i Docker Compose: imatges, Dockerfile, volums, docker-compose amb PostgreSQL + Qdrant.
- **(S12)** Parser de patch notes + wiki estructurada + Qdrant (ja dockeritzat des de S11). Buscar i avaluar skills/MCP existents.
- **(S13)** Knowledge retrieval semàntic amb anti-al·lucinació i evals. Cache de respostes LLM amb Redis. Cost management d'APIs IA.
- **(S14)** Spring Security + JWT. Endpoints protegits. Session store amb Redis. Auth en Python (FastAPI).
- **(S15)** PostgreSQL (ja dockeritzat des de S11), Flyway migrations, JOINs, EXPLAIN ANALYZE, optimitzar N+1. Cache de queries amb Redis.
- **(S16)** Integració de sistemes: Redis consolidat (cache-aside, TTL, invalidació). RabbitMQ: events asíncrons (game.ingested → processament LLM → cache). Patterns: retry, idempotència, dead letter queue.

**Lliurament:** Infraestructura dockeritzada + knowledge retrieval amb citació + API protegida amb JWT + BD PostgreSQL + Redis cache + message queue funcional

### **Bloc 4: Agents i Spec-Driven Development (Setmanes 17-19)**

**Objectiu:** Agents autònoms, workflow de specs, consolidació.

**Features:**
- **(S17)** Agents via API directa amb tool use (sense framework): Agent Quantitatiu + Agent de Knowledge. Evals (15+ preguntes, pytest, CI gate). Observabilitat amb LangFuse (traces, cost). OpenSpec per documentar agents.
- **(S18)** Spec-Driven Development: feature completa implementada per agent a partir de spec. Skill personalitzat + hooks.
- **(S19)** Consolidació: Dashboard Streamlit avançat d'agents (state, historial, mètriques). `CLAUDE.md` + `.cursorrules` finals. MCP/skills documentats.

**Lliurament:** Agents amb evals al CI; dashboard d'agents; skill `/review-gamepulse`

### **Bloc 5: Producció, Portfolio i Entrevista (Setmanes 20-24)**

**Objectiu:** Producció: CI/CD, deploy cloud, portfolio, preparació professional.

**Features:**
- **(S20)** CI/CD consolidat (GitHub Actions). Dockeritzar tots els serveis al `docker-compose.yml` (Java, Python, Streamlit, RabbitMQ). Logging i monitoring bàsic.
- **(S21)** Desplegament cloud (Render/Fly.io) accessible via URL pública.
- **(S22)** Testing E2E complet + revisió seguretat OWASP + hardening.
- **(S23)** Diagrames C4 + documentació tècnica + Swagger consolidat + Portfolio GitHub.
- **(S24)** Preparació entrevista tècnica (system design bàsic, live coding, behavioral) + demo final 5 min.

**Lliurament:** Plataforma íntegra en cloud, amb documentació sòlida, demo i preparació per a entrevistes

---

## Evolució del Projecte per Setmana

| Setmana | Tema | GamePulse Milestone |
|---------|------|-------------------|
| 1-6 | Algorítmica + POO + Concurrència + Testing | `v0.1`: Backend Java + Python amb tests |
| 7-10 | APIs + LLMs + Dashboard + MCP | `v0.2`: REST + Streamlit + MCP servers |
| 11-16 | Docker + Knowledge + Auth + SQL + Redis + Queues | `v0.3`: Docker + Retrieval + JWT + PostgreSQL + Redis + RabbitMQ |
| 17-19 | Agents + Specs + Consolidació | `v0.4`: Agents + evals + dashboard d'agents |
| 20-24 | Producció + Portfolio + Entrevista | `v1.0`: Cloud + demo + portfolio + prep entrevista |

---

## Hypothetical Query Lifecycle

**Setmana 1-6:** "Quants jugadors actius té League of Legends?"
→ Consulta Java pura; resposta de la BD local

**Setmana 7-10:** "Quants jugadors actius té LoL?"
→ Query REST → Java → Dashboard Streamlit mostra resultat

**Setmana 11-16:** "Ha canviat el balanç del heroi Yasuo en els últims 2 anys?"
→ Query → Knowledge retrieval busca a Qdrant → Resposta cacheada a Redis → Retorna patches amb citació (autenticat amb JWT) → Tot dins Docker

**Setmana 17-19:** "Analitza si Yasuo està overpowered i quins canvis caldrien"
→ Query → Agent Stats (pulls data) → Agent Knowledge (busca patches) → Orquestrador sintetitza → LangFuse traça tot → Dashboard d'agents

**Setmana 20-24:** "Projecta l'evolució de Yasuo si les comunitats demanen nerf a l'AP scaling"
→ Query via Dashboard Streamlit → Comitè d'agents → Resultat amb mètriques de confiança → Desplegat a `gamepulse.fly.dev`

---

## Stack Final (v1.0)

```
Dashboard:       Streamlit (Python) — prototipatge ràpid, no frontend web
Backend:         Spring Boot 3 (Java 21)
IA/Agents:       Claude API / OpenAI API amb tool use directe (S17)
Knowledge:       Qdrant (Docker) — embeddings + cerca semàntica
BD:              PostgreSQL (Docker) — migracions amb Flyway
Cache:           Redis (Docker) — cache de queries, respostes LLM, sessions
Messaging:       RabbitMQ (Docker) — events asíncrons entre serveis
Observabilitat:  LangFuse — traces d'agents
Infra:           Docker Compose + GitHub Actions + Render/Fly.io
Testing IA:      pytest + eval datasets
APIs externes:   Steam Web API + RAWG API
```

---

## Competències Adquirides al Final

**Enginyeria de Software:**
- Algorítmica aplicada (Big-O, estructures de dades)
- Arquitectura Java clean + patterns (Factory, Repository, SOLID)
- APIs REST amb contractes clars (DTOs, Swagger/OpenAPI)
- SQL real (JOINs, indexes, EXPLAIN, migrations)
- Autenticació i seguretat (JWT, Spring Security, OWASP)
- Testing (JUnit, pytest, mocks, coverage, E2E)
- Git professional (rebase, bisect, code review, PRs)
- Caching (Redis: cache-aside, TTL, invalidació, session store)
- Message queues (RabbitMQ: events asíncrons, dead letter, retry, idempotència)
- Docker i Docker Compose
- CI/CD amb GitHub Actions
- Desplegament cloud

**Mindset de Codi Productiu (transversal):**
- Observabilitat: logging estructurat (JSON), correlation IDs, mètriques de negoci, traces
- Mantenibilitat: codi llegible, responsabilitats separades, tests com a documentació viva
- Resiliència: timeouts, retries amb backoff, graceful degradation, dead letter queues
- Operabilitat: health checks, configuració externalitzada, slow query logs, docker logs
- Escalabilitat: identificar colls d'ampolla (thread pools, N+1, cache miss), mesurar abans d'optimitzar
- Troubleshooting: diagnosticar problemes en producció amb logs i mètriques (sense accés al codi font)

**IA i AI-SDLC:**
- Prompt engineering progressiu (bàsic → specs)
- Output estructurat amb LLMs (Pydantic, few-shot)
- Knowledge engineering i retrieval (chunking, embeddings, Qdrant, anti-al·lucinació)
- Agents amb tool use via API directa (sense dependència de frameworks)
- Observabilitat d'agents (LangFuse)
- Eval-driven development (datasets + mètriques al CI)
- MCP servers (usar i crear)
- Skills i hooks (personalitzar eines IA)
- Spec-driven development (escriure specs → agent implementa → review)
- Escriptura de `CLAUDE.md` i `.cursorrules` com a specs d'agent

**Resultat:** Un enginyer de software junior que pot construir sistemes backend AI-powered end-to-end, operar-los en producció, i treballar amb agents de codi com a copilots. No només escriu codi que funciona — escriu codi que es pot diagnosticar, mantenir i escalar.
