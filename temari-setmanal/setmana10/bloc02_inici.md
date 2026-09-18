# Bloc 2 (Setmanes 10–16) — IA, Seguretat, Frontend i Agents

## D'On Veníem (Final del Bloc 1)

Al Bloc 1 vas construir les bases d'EsportsPulse:
- Backend Java amb API REST, persistència JPA i tests complets
- Servei Python amb CLI, models de domini i SQLite
- CI/CD amb GitHub Actions, cobertura > 70% i linting
- Tot containeritzat amb Docker Compose

**El que falta:** L'aplicació funciona, però no pensa, no es protegeix, no es veu i no aprèn. El Bloc 2 ho canvia tot.

---

## Objectiu Global del Bloc 2

Transformar EsportsPulse d'una aplicació CRUD en una **plataforma intel·ligent, segura i visual**:

1. **Intel·ligència** — Connectar amb l'API de Claude per analitzar campions, construir un pipeline RAG amb cerca semàntica, i crear agents especialitzats
2. **Robustesa** — Gestió d'errors professional, logging estructurat, traçabilitat entre serveis i patrons de resiliència
3. **Seguretat** — Autenticació JWT, rols, Spring Security, FastAPI protegit i headers de seguretat
4. **Interfície** — Dashboard Streamlit que visualitza dades, permet interacció i connecta amb tot el stack
5. **Agents** — Construir agents IA amb tool use, retrieval i observabilitat, documentats formalment

---

## Objectius d'Aprenentatge

Al final del Bloc 2, l'estudiant serà capaç de:

### IA i LLMs
- Validar dades Python amb Pydantic (type safety real)
- Connectar-se a l'API de Claude amb system prompts, few-shot i JSON output
- Construir serveis amb FastAPI (el framework Python modern)
- Entendre i configurar MCP (Model Context Protocol)
- Preparar documents per a IA: chunking, metadades, embeddings
- Implementar cerca semàntica amb Qdrant
- Construir un pipeline RAG complet amb anti-al·lucinació
- Mesurar qualitat amb evals quantitatius
- Construir agents amb tool use (agent quantitatiu i agent de knowledge)
- Documentar agents amb OpenSpec

### Robustesa i Observabilitat
- Dissenyar una jerarquia d'excepcions de domini
- Implementar logging estructurat (JSON) per a anàlisi automatitzada
- Propagar Correlation IDs entre serveis Java i Python
- Construir eines MCP pròpies (de consumidor a creador)
- Aplicar patrons de resiliència: timeouts, retries, circuit breakers

### Seguretat
- Entendre autenticació vs autorització
- Configurar Spring Security amb JWT
- Implementar rols i @PreAuthorize
- Protegir FastAPI amb JWT i sessions Redis
- Aplicar headers de seguretat (CORS, CSRF, CSP)

### Frontend i UX
- Construir dashboards interactius amb Streamlit
- Dissenyar guiat per wireframes
- Testejar end-to-end el full stack
- Polir un producte fins a qualitat de demo

---

## Features que Afegirem a l'Aplicació

### Setmana 10 — El Servei Python amb IA

L'aplicació passa de "guardar dades" a "entendre dades".

```
ABANS (Bloc 1):                      DESPRÉS (S10):
CLI Python → API Java → DB           FastAPI Python → Claude API → anàlisi
(només CRUD)                          (intel·ligència artificial)
```

**Features noves:**
- Servei FastAPI independent (port 8000) que substitueix el CLI
- Models Pydantic per validar requests/responses
- Endpoint `/analyze/{champion}` que crida l'API de Claude per generar anàlisi
- MCP server bàsic configurat
- Integració FastAPI ↔ API Java ↔ Claude API

---

### Setmana 11 — Robustesa i Observabilitat

L'aplicació passa de "funciona si tot va bé" a "funciona encara que les coses fallin".

```
ABANS:                                DESPRÉS (S11):
try-catch genèric                     Excepcions de domini + codis HTTP
System.out.println                    Logs JSON estructurats
Sense traçabilitat                    Correlation IDs entre serveis
```

**Features noves:**
- Excepcions de domini (`ChampionNotFoundException` → 404) amb respostes consistents
- Logging JSON en tots els serveis (indexable per ELK/Grafana)
- Header `X-Correlation-Id` propagat de Java a Python
- MCP server `search_logs` per buscar logs amb IA
- Timeouts i retries amb backoff exponencial

---

### Setmana 12 — Seguretat

L'aplicació passa de "oberta al món" a "protegida amb autenticació i rols".

```
ABANS:                                DESPRÉS (S12):
GET /api/champions → qualsevol        GET /api/champions → token JWT requerit
POST /api/champions → qualsevol       POST /api/champions → rol ADMIN requerit
```

**Features noves:**
- Endpoint `/auth/login` que retorna un JWT token
- Spring Security filter chain que valida JWT a cada petició
- Rols (USER, ADMIN) amb `@PreAuthorize` per endpoint
- FastAPI protegit amb el mateix JWT
- Sessions Redis per invalidar tokens (logout, blacklist)
- Headers de seguretat (CORS, CSP, X-Frame-Options)

---

### Setmana 13 — Dashboard Streamlit

L'aplicació passa de "invisible" a "visual i interactiva".

```
ABANS:                                DESPRÉS (S13):
CLI al terminal                       Dashboard web amb taules, gràfics i login
curl per testejar                     Interfície visual per a stakeholders
```

**Features noves:**
- Dashboard Streamlit (port 8501) amb login
- Pàgina de llista de campions (taula ordenable, filtres)
- Gràfics interactius (win rate per rol, distribució de tiers)
- Formulari per afegir/editar campions
- Pàgina d'anàlisi IA (crida al servei Python)
- Tests E2E del full stack

---

### Setmana 14 — Knowledge Base per a IA

L'aplicació passa de "la IA respon del que sap" a "la IA respon amb les nostres dades".

```
ABANS:                                DESPRÉS (S14):
Claude respon amb coneixement general  Claude respon amb la wiki del projecte
(pot al·lucinar)                       (grounded en documents reals)
```

**Features noves:**
- Wiki del projecte en Markdown (guies, decisions, ADRs)
- Pipeline de chunking: documents → chunks amb metadades
- Base de dades vectorial Qdrant amb embeddings dels chunks
- Endpoint `/search` per buscar a la wiki semànticament
- Configuració de @docs per a l'editor

---

### Setmana 15 — RAG Robust i Avaluable

L'aplicació passa de "cerca bàsica" a "retrieval semàntic de qualitat mesurable".

```
ABANS:                                DESPRÉS (S15):
Cerca per paraules exactes            Cerca per significat
L'LLM sempre respon (pot inventar)    L'LLM diu "No ho sé" quan no té info
Sense mètriques de qualitat           Evals amb precision/recall
Cada query costa diners               Cache Redis per queries repetides
```

**Features noves:**
- Retrieval semàntic (buscar per significat, no per paraules)
- Anti-al·lucinació: threshold de confiança, resposta "No tinc aquesta informació"
- Framework d'evals amb test suite de preguntes + respostes esperades
- Cache Redis per respostes LLM (reducció de cost 80%+)
- Behavior spec documentant el comportament esperat del RAG

---

### Setmana 16 — Agents d'IA

L'aplicació passa de "la IA respon preguntes" a "la IA executa accions".

```
ABANS:                                DESPRÉS (S16):
Endpoint que retorna text             Agent que raona i usa eines
Una sola capacitat (anàlisi)          Dos agents especialitzats
Sense observabilitat                  Traces, costos i mètriques amb LangFuse
```

**Features noves:**
- **Agent Quantitatiu**: usa tools per fer càlculs (compare champions, stats)
- **Agent de Knowledge**: usa el RAG per respondre preguntes de la wiki
- Evals d'agents (mesuren qualitat de les respostes i ús correcte de tools)
- LangFuse per monitoritzar traces, costos i latència
- OpenSpec: documentació formal de cada agent (inputs, tools, comportament)

---

## Diagrama de l'Aplicació al Final del Bloc 2

```
┌─────────────────────────────────────────────────────────────────────┐
│                       docker-compose up                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐   ┌──────────────────┐   ┌──────────────────────┐ │
│  │  Streamlit   │──►│  FastAPI Python   │──►│  Backend Java        │ │
│  │  Dashboard   │   │  (IA Service)     │   │  (Spring Boot)       │ │
│  │  :8501       │   │  :8000            │   │  :8080               │ │
│  │              │   │                    │   │                      │ │
│  │  Login       │   │  /analyze          │   │  REST CRUD           │ │
│  │  Taules      │   │  /search           │   │  JWT Auth            │ │
│  │  Gràfics     │   │  /agents/query     │   │  Spring Security     │ │
│  │  Formularis  │   │  Agent Quantitatiu │   │  Structured Logging  │ │
│  └─────────────┘   │  Agent Knowledge   │   │  Correlation IDs     │ │
│                     └────────┬───────────┘   └──────────┬───────────┘ │
│                              │                           │             │
│                     ┌────────▼───────────┐   ┌──────────▼───────────┐ │
│                     │  Claude API        │   │  PostgreSQL 16       │ │
│                     │  (LLM extern)      │   │  :5432               │ │
│                     └────────────────────┘   └──────────────────────┘ │
│                                                                       │
│  ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐ │
│  │  Qdrant           │   │  Redis 7          │   │  LangFuse         │ │
│  │  (Vector DB)      │   │  (Cache + Sessions)│  │  (Observabilitat) │ │
│  │  :6333            │   │  :6379            │   │                    │ │
│  └──────────────────┘   └──────────────────┘   └──────────────────┘ │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

---

## Resum: Què Aporta Cada Setmana

| Setmana | Tema Principal | Feature Clau |
|---------|----------------|--------------|
| **S10** | IA + FastAPI + MCP | Servei Python intel·ligent amb Claude API |
| **S11** | Robustesa + Observabilitat | Logging JSON, Correlation IDs, resiliència |
| **S12** | Seguretat | JWT, Spring Security, rols, headers |
| **S13** | Frontend | Dashboard Streamlit complet amb login |
| **S14** | Knowledge Base | Chunking, embeddings, Qdrant |
| **S15** | RAG robust | Cerca semàntica, anti-al·lucinació, evals, cache |
| **S16** | Agents | Tool use, agents especialitzats, LangFuse, OpenSpec |
