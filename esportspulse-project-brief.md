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

## Roadmap de Features d'EsportsPulse

Cada bloc afegeix capacitats reals a l'aplicació. Al final de cada bloc, EsportsPulse **fa coses noves** que abans no podia fer.

### **Bloc 1: Motor de Dades Local (Setmanes 1-6)**

**L'aplicació pot:**
- Emmagatzemar jugadors i campions en memòria amb cerca instantània per ID (HashMap)
- Validar que cap entitat invàlida entri al sistema (compact constructors)
- Ingerir dades de múltiples fonts en paral·lel (extractor concurrent de Riot API + PandaScore)
- Persistir campions i jugadors a una base de dades local (H2) amb repositoris
- Executar una suite de tests automatitzats (JUnit 5 + pytest) integrada al CI

**Lliurament:** `v0.1` — Motor de dades local amb ingesta concurrent, persistència i tests

### **Bloc 2: API REST i Dashboard (Setmanes 7-10)**

**L'aplicació pot:**
- Exposar una API REST per consultar campions per rol, win rate i pick rate (endpoints Swagger documentats)
- Generar anàlisis estructurades de campions via LLM (output Pydantic validat)
- Connectar agents IA a la base de dades d'EsportsPulse via MCP
- Mostrar un dashboard interactiu amb tier list de campions, filtres per rol, i imatges de Data Dragon

**Lliurament:** `v0.2` — API REST + dashboard Streamlit + MCP servers

### **Bloc 3: Infraestructura i Knowledge Base (Setmanes 11-16)**

**L'aplicació pot:**
- Córrer tots els serveis amb un sol `docker-compose up` (Java, Python, PostgreSQL, Qdrant, Redis, RabbitMQ)
- Respondre "quan van nerfar Yasuo?" cercant semànticament entre patch notes indexades a Qdrant, amb citació de la font
- Cachear respostes de l'LLM i queries freqüents a Redis (reducció de latència i cost)
- Protegir endpoints amb autenticació JWT (login, tokens, endpoints protegits)
- Persistir dades a PostgreSQL amb migracions versionades (Flyway) i queries optimitzades
- Reaccionar a events asíncrons: quan es publica un patch nou → parsejar notes → indexar a Qdrant → invalidar cache (RabbitMQ)

**Lliurament:** `v0.3` — Infraestructura dockeritzada + knowledge base de patch notes + auth + cache + events asíncrons

### **Bloc 4: Agents Intel·ligents (Setmanes 17-19)**

**L'aplicació pot:**
- Respondre preguntes complexes combinant dos agents: l'Agent Estadístic (consulta stats de campions a la BD) i l'Agent de Knowledge (cerca patch notes a Qdrant)
- Recomanar picks i bans per a una partida concreta (Draft Assistant) basant-se en stats actuals i canvis recents del meta
- Mesurar la qualitat de les respostes dels agents amb evals automatitzats al CI (20+ preguntes de referència)
- Traçar cada interacció dels agents amb LangFuse (cost, latència, qualitat)
- Mostrar un dashboard d'agents amb historial de preguntes, qualitat i cost acumulat

**Lliurament:** `v0.4` — Agents amb evals al CI + Draft Assistant + dashboard d'agents

### **Bloc 5: Producció (Setmanes 20-24)**

**L'aplicació pot:**
- Desplegar-se automàticament a cloud (Render/Fly.io) via CI/CD amb cada push a `main`
- Ser accessible via URL pública (`esportspulse.fly.dev`) amb tota la funcionalitat operativa
- Passar tests E2E complets (login → cerca de campió → anàlisi d'agent → resultat)
- Resistir auditoria de seguretat bàsica (OWASP top 10)
- Presentar-se en una demo de 5 minuts amb documentació tècnica (diagrames C4, Swagger, portfolio GitHub)

**Lliurament:** `v1.0` — Plataforma completa en cloud, documentada i presentable en entrevista

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
