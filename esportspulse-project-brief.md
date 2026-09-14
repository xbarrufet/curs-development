# EsportsPulse - Project Brief

## Descripció General

**EsportsPulse** és un sistema multi-agent per analitzar i monitoritzar l'evolució competitiva de campiòns a League of Legends. El projecte integra:
- **Backend Java 21**: Serveis REST per ingesta de dades de la Riot API, cerca de campiòns, i estadístiques competitives
- **Python IA**: Agents per interpretar patch notes, knowledge retrieval per trends de meta, anàlisi de campiòns
- **Observabilitat**: LangFuse per traçar el comportament dels agents
- **Interfície**: Dashboard Streamlit per consultar l'equip d'agents i visualitzar meta game

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

## Model de Domini

EsportsPulse treballa amb les entitats del món competitiu de League of Legends. Aquests són els objectes de negoci del sistema.

### Campions per Rol

El joc té 170+ campions, cadascun amb un o més rols. Exemples reals (carregats de l'API Data Dragon, sense clau):

| Rol | Campions d'exemple | Descripció del rol |
|-----|-------------------|-------------------|
| **Assassin** | Ahri, Akali, Zed | Eliminen objectius prioritaris ràpidament |
| **Mage** | Lux, Anivia, Syndra | Dany màgic a distància, control de zona |
| **Fighter** | Yasuo, Aatrox, Darius | Cos a cos amb resistència i dany sostingut |
| **Marksman** | Jinx, Aphelios, Caitlyn | Dany físic a distància, principals portadors de partida |
| **Support** | Thresh, Lulu, Nautilus | Protegeixen i potencien els aliats |
| **Tank** | Amumu, Alistar, Ornn | Absorbeixen dany i inicien lluites d'equip |

### Entitats de Domini

**ChampionRecord** — Un campió de LoL. Conté la identitat del campió (nom, rol) i les seves estadístiques competitives: percentatge de victòries (win rate) i percentatge de selecció (pick rate). Un campió amb win rate alt i pick rate alt es considera "meta" — dominant en el joc competitiu actual. És l'entitat central del sistema.
- *Font de dades:* Data Dragon (dades estàtiques: nom, rol, stats base) + Riot API match-v5 (estadístiques calculades a partir de partides)

**PlayerRecord** — Un jugador/invocador de LoL. Representa el perfil d'un jugador amb el seu progrés i experiència de joc. No és un usuari del sistema EsportsPulse — és una entitat del domini de LoL.
- *Font de dades:* Riot API summoner-v4

**MatchRecord** — Una partida jugada. Cada partida enfronta 2 equips de 5 jugadors, on cada jugador ha seleccionat un campió. Conté els campions que hi van participar, la durada, el resultat, i la versió del patch en què es va jugar. Connecta campions amb jugadors en un moment concret del joc.
- *Font de dades:* Riot API match-v5

**PatchNote** — Les notes d'una actualització del joc. Cada 2 setmanes, Riot Games modifica l'equilibri dels campions (buffs i nerfs). Conté la versió del patch, la data, i els canvis per campió. És la base del knowledge retrieval — permet respondre preguntes com "quan van nerfar Yasuo?" o "quin campió ha rebut més buffs recentment?".
- *Font de dades:* Web scraping de leagueoflegends.com/patch-notes

**User** — Usuari del sistema EsportsPulse (no un jugador de LoL). Per autenticació i autorització via JWT.

### Relacions entre Entitats

```
                    ┌─────────────┐
                    │  PatchNote  │
                    └──────┬──────┘
                  modifica │ (buff/nerf)
                           ▼
┌──────────────┐    ┌─────────────────┐
│ PlayerRecord │    │ ChampionRecord  │
└──────┬───────┘    └────────┬────────┘
       │                     │
       │  juga com a         │ seleccionat a
       │                     │
       └────────►┌───────────┴──┐
                 │ MatchRecord  │
                 └──────────────┘
```

Una **partida** connecta jugadors amb campions: cada jugador selecciona un campió per jugar. Les **patch notes** modifiquen les estadístiques dels campions, alterant el "meta" — el conjunt de campions dominants en cada moment. L'anàlisi d'EsportsPulse creua aquestes dades per respondre preguntes com "Yasuo està overpowered des de l'últim patch?".

### Font de Dades per Entitat

| Entitat | API | Autenticació | Primer ús al curs |
|---------|-----|-------------|-------------------|
| PlayerRecord | Riot API summoner-v4 | API key gratuïta | S1 (sintètic), S7 (real) |
| ChampionRecord | Data Dragon (CDN públic) | Cap | S2 (sintètic), S7 (real) |
| MatchRecord | Riot API match-v5 | API key gratuïta | S3 (extractor concurrent) |
| PatchNote | Web scraping LoL patch notes | Cap | S12 (knowledge retrieval) |
| User | Intern (BD pròpia) | — | S14 (JWT auth) |

### Evolució de les Entitats al Curs

| Setmana | Entitat | Què passa |
|---------|---------|-----------|
| S1 | PlayerRecord | Es crea amb dades sintètiques (100K jugadors) per practicar col·leccions i benchmarking |
| S2 | PlayerRecord | S'amplia amb validació (compact constructor) i mètodes de negoci |
| S2 | ChampionRecord | Es crea quan es connecta amb APIs reals (Data Dragon, Riot API) |
| S3 | MatchRecord | Es crea per a l'extractor concurrent de partides via Riot API |
| S5-S6 | ChampionRecord, PlayerRecord | Persistència amb JPA + H2, repositoris SQL |
| S7 | Totes | Endpoints REST, dades reals de Data Dragon i Riot API |
| S12-S13 | PatchNote | Parser de patch notes + indexació a Qdrant per knowledge retrieval |
| S14 | User | Autenticació JWT, endpoints protegits |
| S15 | Totes | PostgreSQL, migracions Flyway, JOINs entre entitats |

---

## APIs Públiques de League of Legends

EsportsPulse consumeix dades reals de League of Legends. Les APIs recomanades (totes amb tier gratuït):

### Riot Games API (Principal)

| | |
|---|---|
| **URL base** | `https://{region}.api.riotgames.com/` |
| **Documentació** | https://developer.riotgames.com |
| **Obtenir API key** | https://developer.riotgames.com (cal compte Riot gratuït) |
| **Rate limit** | 20 req/s + 100 req/2min (Personal Key, gratuït) |
| **Autenticació** | Header `X-Riot-Token: YOUR_API_KEY` |

**Endpoints útils per EsportsPulse:**

```
# Historial complet de partida (match timeline amb dades per minut)
GET https://{routing}/lol/match/v5/matches/{matchId}
→ Retorna: timeline per minut, stats, kills, items, gold per jugador
→ Necessita API key

# Perfil de jugador
GET https://{region}/lol/summoner/v4/summoners/by-name/{summonerName}
→ Retorna: summonerId, tier, rank, leaguePoints
→ Necessita API key

# Dades de campiò (rol, stats base, abilities)
GET https://ddragon.leagueoflegends.com/cdn/{version}/data/en_US/champion.json
→ Retorna: ~170 campiòns amb stats, abilities, skins
→ No necessita API key (CDN públic, Data Dragon)

# Dades de patch actual
GET https://ddragon.leagueoflegends.com/api/versions.json
→ Retorna: versió actual del joc i historial de versions
→ No necessita API key
```

**Limitacions:**
- Rate limit estricte (20 req/s). Per ingesta massiva de partides, usar caching + backoff exponencial.
- Match-v5 retorna historial complet però no prediccions — les prediccions venen de knowledge retrieval sobre patch notes.

### Data Dragon (Estàtic + Gratuït)

| | |
|---|---|
| **URL base** | `https://ddragon.leagueoflegends.com` |
| **Documentació** | Part de la Riot API docs (secció static data) |
| **Autenticació** | Cap (CDN públic) |
| **Rate limit** | Cap (CDN estàndard) |

**Endpoints útils per EsportsPulse:**

```
# Tots els campiòns (nom, role, stats, abilities)
GET https://ddragon.leagueoflegends.com/cdn/{version}/data/en_US/champion.json
→ Retorna: 170+ campiòns amb dades completes

# Ítems (estadístiques, noms, icones)
GET https://ddragon.leagueoflegends.com/cdn/{version}/data/en_US/item.json
→ Retorna: ítems amb preus, stats, descripcions

# Imatges de campiòns (splash art, loading screen)
https://ddragon.leagueoflegends.com/cdn/img/champion/splash/{championKey}_0.jpg
→ Retorna: imatge de splash art per visualitzar al dashboard
```

**Avantatge:** Zero autenticació, zero rate limit, dades completes per a 170+ campiòns. S'integra a partir de S7 quan es connecta amb dades reals (a S1-S2 les dades són sintètiques).

### PandaScore API (Alternativa/Complement — Multi-Game)

| | |
|---|---|
| **URL base** | `https://api.pandascore.co/` |
| **Documentació** | https://developers.pandascore.co |
| **Obtenir API key** | https://developers.pandascore.co/login (registre gratuït) |
| **Rate limit** | 1.000 req/hora (tier gratuït) |
| **Autenticació** | Bearer token `Authorization: Bearer YOUR_TOKEN` |

**Endpoints útils per EsportsPulse:**

```
# Calendari de torneigs
GET https://api.pandascore.co/lol/tournaments
→ Retorna: torneigs actuals, calendari, equips, jugadors

# Resultats de partides profesionals
GET https://api.pandascore.co/lol/matches
→ Retorna: partides recents de l'escena professional
```

**Avantatge:** Dades de la escena professional (torneigs, equips, resultats) que Riot API no retorna directament. Ideal per a l'exercici de S3 (extractor concurrent) com a complemento a Riot API.

### Patch Notes (Per al Bloc 3: Knowledge Engineering)

Les patch notes de LoL no vénen d'una API — són documents publicats pels developers. Fonts:

- **League of Legends:** https://www.leagueoflegends.com/en-us/news/tags/patch-notes/ (HTML scrapable, historial complet)
- **Valorant:** https://playvalorant.com/en-us/news/tags/patch-notes/ (HTML)

**Exercici S12:** L'estudiant descarrega 10-20 patch notes reals de LoL (HTML), les estructura en markdown amb metadades (patch version, data, canvis de campiò), i les indexa a Qdrant per a knowledge retrieval.

### Gestió de Claus API al Projecte

```
# .env (exclòs de Git via .gitignore)
RIOT_API_KEY=RGAPI-your_riot_key_here
PANDASCORE_API_KEY=your_pandascore_key_here

# Java: application.properties
riot.api.key=${RIOT_API_KEY}
pandascore.api.key=${PANDASCORE_API_KEY}

# Python: os.environ
riot_api_key = os.environ["RIOT_API_KEY"]
pandascore_api_key = os.environ["PANDASCORE_API_KEY"]
```

**Regla del curs:** Mai secrets al codi. Sempre `.env` + `.gitignore`. Això es practica des de S4 (anti-patró de secrets hardcodejats).

---

## Roadmap de Features per Bloc

### **Bloc 1: Fundaments, POO i Concurrència (Setmanes 1-6)**

**Objectiu:** Domini bàsic de Java 21 + Python, estructura de projecte, i concurrència pràctica.

**Features:**
- **(S1)** Benchmark O(n) vs O(1): Search lineal vs HashMap amb 100.000 jugadors sintètics (`PlayerRecord`)
- **(S2)** Ampliar `PlayerRecord` amb validació; crear `ChampionRecord` immutable (record / dataclass) connectat a APIs
- **(S3)** Concurrència pràctica: race conditions, @Transactional, CompletableFuture, extractor concurrent d'APIs Riot/PandaScore
- **(S4)** Code review, refactorització d'anti-patrons IA, Git workflow, CI bàsic
- **(S5)** Factory + Repository patterns amb JPA + H2; SQL pur a consola
- **(S6)** Suite JUnit 5 + pytest; coverage gates al CI

**Lliurament:** Codi Java + Python netejat, amb tests, versionat a `v0.1`

### **Bloc 2: APIs REST i Output Estructurat amb IA (Setmanes 7-10)**

**Objectiu:** Contractes REST clars, integració amb LLMs, primera interfície.

**Features:**
- **(S7)** Endpoints REST Spring Boot 3: CRUD de campiòns, query params per role/winRate, Swagger, Virtual Threads. API spec com a exercici de specs.
- **(S8)** Python Pydantic models + LLM output estructurat (Claude/OpenAI API). MCP basics: connectar l'agent a la BD d'EsportsPulse.
- **(S9)** Error handling inter-serveis Java↔Python, logging estructurat, gestió de rate limit de Riot API. MCP server personalitzat.
- **(S10)** Dashboard Streamlit que consumeix l'API REST: llistat de campiòns, filtres per rol/tier, tier list visual. Spec del dashboard.

**Lliurament:** Backend REST funcional + dashboard Streamlit + MCP servers

### **Bloc 3: Infraestructura, Knowledge, Seguretat i Integració (Setmanes 11-16)**

**Objectiu:** Contenidors, knowledge retrieval de patch notes, autenticació, BD real, caching, message queues.

**Features:**
- **(S11)** Docker i Docker Compose: imatges, Dockerfile, volums, docker-compose amb PostgreSQL + Qdrant.
- **(S12)** Parser de patch notes LoL + wiki estructurada (markdown amb metadades de campiò/patch) + Qdrant (ja dockeritzat des de S11). Buscar i avaluar skills/MCP existents.
- **(S13)** Knowledge retrieval semàntic sobre patch notes amb anti-al·lucinació i evals. Cache de respostes LLM amb Redis. Cost management d'APIs IA (Riot API + Claude API).
- **(S14)** Spring Security + JWT. Endpoints protegits. Session store amb Redis. Auth en Python (FastAPI).
- **(S15)** PostgreSQL (ja dockeritzat des de S11), Flyway migrations, JOINs per campiò+patch+meta, EXPLAIN ANALYZE, optimitzar N+1. Cache de queries amb Redis.
- **(S16)** Integració de sistemes: Redis consolidat (cache-aside, TTL, invalidació). RabbitMQ: events asíncrons (patch.released → parse notes → indexa Qdrant → invalida cache). Patterns: retry amb backoff, idempotència, dead letter queue.

**Lliurament:** Infraestructura dockeritzada + knowledge retrieval de patch notes amb citació + API protegida amb JWT + BD PostgreSQL + Redis cache + message queue funcional

### **Bloc 4: Agents i Spec-Driven Development (Setmanes 17-19)**

**Objectiu:** Agents autònoms per anàlisi competitiva, workflow de specs, consolidació.

**Features:**
- **(S17)** Agents via API directa amb tool use (sense framework): Agent Estadístic (queries de campiò stats) + Agent de Knowledge (retrieval de patch notes). Evals (20+ preguntes, pytest, CI gate). Observabilitat amb LangFuse (traces, cost). OpenSpec per documentar agents.
- **(S18)** Spec-Driven Development: feature completa (Draft Assistant per recomanar picks/bans) implementada per agent a partir de spec. Skill personalitzat `/review-esportspulse` + hooks.
- **(S19)** Consolidació: Dashboard Streamlit avançat d'agents (historial de preguntes, qualitat de respostes, cost acumulat). `CLAUDE.md` + `.cursorrules` finals. MCP/skills documentats.

**Lliurament:** Agents amb evals al CI; dashboard d'agents; skill `/review-esportspulse`

### **Bloc 5: Producció, Portfolio i Entrevista (Setmanes 20-24)**

**Objectiu:** Producció: CI/CD, deploy cloud, portfolio, preparació professional.

**Features:**
- **(S20)** CI/CD consolidat (GitHub Actions). Dockeritzar tots els serveis al `docker-compose.yml` (Java, Python, Streamlit, PostgreSQL, Qdrant, Redis, RabbitMQ). Logging i monitoring bàsic.
- **(S21)** Desplegament cloud (Render/Fly.io) accessible via URL pública amb suport per a Riot API key segura.
- **(S22)** Testing E2E complet (login → cerca de campiò → anàlisi d'agent) + revisió seguretat OWASP + hardening.
- **(S23)** Diagrames C4 (context, containers, components) + documentació tècnica + Swagger consolidat + Portfolio GitHub amb pinned repo.
- **(S24)** Preparació entrevista tècnica (system design d'EsportsPulse, live coding, behavioral) + demo final 5 min de la plataforma completa.

**Lliurament:** Plataforma íntegra en cloud (esportspulse.fly.dev), amb documentació sòlida, demo profesional i preparació per a entrevistes

---

## Evolució del Projecte per Setmana

| Setmana | Tema | EsportsPulse Milestone |
|---------|------|-------------------|
| 1-6 | Algorítmica + POO + Concurrència + Testing | `v0.1`: Backend Java + Python amb tests, model de domini amb dades sintètiques |
| 7-10 | APIs + LLMs + Dashboard + MCP | `v0.2`: REST + Streamlit (tier list, campiò search) + MCP servers |
| 11-16 | Docker + Knowledge + Auth + SQL + Redis + Queues | `v0.3`: Docker + Retrieval de patch notes + JWT + PostgreSQL + Redis + RabbitMQ |
| 17-19 | Agents + Specs + Consolidació | `v0.4`: Agents (Estadístic + Knowledge) + evals + dashboard d'agents + Draft Assistant |
| 20-24 | Producció + Portfolio + Entrevista | `v1.0`: Cloud (esportspulse.fly.dev) + demo + portfolio + prep entrevista |

---

## Hypothetical Query Lifecycle

**Setmana 1-6:** "Quant es juega Yasuo?"
→ Consulta Java pura; resposta de la BD local (gamesPlayed index)

**Setmana 7-10:** "Quants campiòns marksman hi ha amb winRate > 52%?"
→ Query REST + Streamlit → Tier list visual, imatges de Data Dragon

**Setmana 11-16:** "Quan van nerfar Yasuo per últim cop i quant li van baixar el dany?"
→ Query → Knowledge retrieval busca patch notes a Qdrant → Resposta cacheada a Redis → Retorna patch amb citació (autenticat amb JWT) → Tot dins Docker

**Setmana 17-19:** "Analitza si Yasuo està overpowered comparat amb Ahri"
→ Query → Agent Stats (pulls winrates, pickrates) → Agent Knowledge (busca patches recents) → Combina análisis → LangFuse traça tot → Dashboard d'agents

**Setmana 20-24:** "Recomana el millor pick per countre Yasuo en el patx actual"
→ Query via Dashboard Streamlit → Draft Assistant (combina stats + meta conocimiento) → Resultat amb mètriques de confiança → Desplegat a `esportspulse.fly.dev`

---

## Stack Final (v1.0)

```
Dashboard:       Streamlit (Python) — prototipatge ràpid, visualització meta game
Backend:         Spring Boot 3 (Java 21)
IA/Agents:       Claude API (tool use directe, sense frameworks)
Knowledge:       Qdrant (Docker) — embeddings de patch notes, cerca semàntica
BD:              PostgreSQL (Docker) — migracions amb Flyway, champions + meta stats
Cache:           Redis (Docker) — cache de queries, respostes LLM, sessions
Messaging:       RabbitMQ (Docker) — events asíncrons (patch.released, match.ingested)
Observabilitat:  LangFuse — traces d'agents, cost management
Infra:           Docker Compose + GitHub Actions + Render/Fly.io
Testing IA:      pytest + eval datasets (20+ preguntes per agent)
APIs externes:   Riot Games API (match-v5, summoner-v4) + Data Dragon (static data)
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

**Resultat:** Un enginyer de software junior que pot construir sistemes backend AI-powered end-to-end centrats en eSports (League of Legends), operar-los en producció, i treballar amb agents de codi com a copilots. No només escriu codi que funciona — escriu codi que es pot diagnosticar, mantenir i escalar. Porta a casa un portfolio amb EsportsPulse a cloud, demostrant arquitectura multi-agent, integració amb APIs reals (Riot, Data Dragon), i knowledge retrieval sobre dades complexes (patch notes LoL).
