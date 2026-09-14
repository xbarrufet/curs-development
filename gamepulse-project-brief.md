# GamePulse - Project Brief

## Descripció General

**GamePulse** és un sistema multi-agent per analitzar i monitorizar el balanç i evolució de videojocs. El projecte integra:
- **Backend Java 21**: Serveis REST per ingesta de dades, cerca de jocs, i estadístiques
- **Python IA**: Agents RAG per interpretar notes de parches, agents analítics per trends de balanç
- **Observabilitat**: LangFuse per traçar el comportament dels agents
- **Interfície**: CLI (`gamepulse`) i dashboard web per consultar l'equip d'agents

## Objectiu Primari

**Formar un developer junior en el cicle complet AI-SDLC**: des de algorítmica bàsica fins a orquestració de múltiples agents autònoms, passant per RAG, testing d'IA, i desplegament en cloud. Cada setmana afegeix una capa nova sense trencar la precedent.

## Roadmap de Features per Bloc

---

### **Bloc 1: Fundaments, POO i Concurrència (Setmanes 1-6)**

**Objectiu:** Domini bàsic de Java 21, estructura de projecte, i noció de rendiment.

**Features:**
- ✅ **(S1)** Benchmark O(n) vs O(1): Search lineal vs HashMap per ID de joc
- ✅ **(S2)** Model `GameRecord` immutable (record); DTO per jocs amb preu, jugadors actius
- ✅ **(S3)** Concurrència virtual: Extractor paralel d'APIs Steam/IGDB
- ✅ **(S4)** Repositori git estructura + refactorització sense trencar tests
- ✅ **(S5)** Factory + Repository patterns per desacoblar ingesta de dades
- ✅ **(S6)** Suite JUnit 5 + Mockito; tests del nucli com defensiva contra regressions

**Lliuramento:** Codi Java netejat, amb tests, versionat a `v0.1`

---

### **Bloc 2: APIs REST i Output Estructurat amb IA (Setmanes 7-10)**

**Objectiu:** Contractes REST clars, validació automàtica de dades, integració amb LLMs.

**Features:**
- ✅ **(S7)** Endpoints REST Spring Boot 3: `GET /games/{id}`, `POST /games/search`
- ✅ **(S8)** Python Pydantic models; servei que converteix "quina és la jugabilitat de X?" en JSONs estructurats
- ✅ **(S9)** Resilience4j + Tenacity: reintentos automàtics inter-serveis Java-Python
- ✅ **(S10)** CLI gamepulse (`typer`): interfície terminal que fa queries al backend

**Lliurament:** Backend REST funcional + CLI consumidor

---

### **Bloc 3: Processament de Documents i RAG Lean (Setmanes 11-14)**

**Objectiu:** Implementar RAG híbrid per cercar informació dins patch notes.

**Features:**
- ✅ **(S11)** Parser de PDF + chunking intel·ligent de notes d'actualització (ex: League of Legends patches)
- ✅ **(S12)** Qdrant (Docker): base vectorial local per embeddings de patch notes
- ✅ **(S13)** RAG híbrid BM25 + semàntic: cercar "balanç archer" dins 5 anys de patches
- ✅ **(S14)** Anti-al·lucinació: prompts de citació estricta, traça de fonts

**Lliurament:** Motor RAG que respon "quins canvis de balanç ha tingut el heroi X?" amb citació de font + secció del patch

---

### **Bloc 4: Orquestració Multi-Agent, Observabilitat i EDD (Setmanes 15-18)**

**Objectiu:** Múltiples agents autònoms que col·laboren; testing d'IA a producció.

**Features:**
- ✅ **(S15)** Arquitectura d'agents: Agent Quantitatiu (stats Java) + Agent RAG (balanç). Comparativa frameworks (OpenAI Agents SDK vs Claude Code SDK vs LangGraph)
- ✅ **(S16)** LangFuse dashboard: traçabilitat completa (quins tokens gastats, latències per agent, cost total per query)
- ✅ **(S17)** Dataset d'evaluació: 25+ preguntes de balanç amb respostes de referència (ground truth)
- ✅ **(S18)** Ragas + Pytest: mètriques de fidelitat i precisió automàtiques al CI (gate de qualitat pre-push)

**Lliurament:** Comitè de 2 agents amb traçabilitat, dataset d'evals, i gate de qualitat

---

### **Bloc 5: AI-SDLC, UI i Desplegament (Setmanes 19-24)**

**Objectiu:** Producció: especificacions per agents, UI, infra as code, CI/CD.

**Features:**
- ✅ **(S19)** `CLAUDE.md` + `.cursorrules` final: arquitectura documentada per agents de codi
- ✅ **(S20)** Dashboard Streamlit: visualització de l'equip d'agents (state, historial de queries, mètrica de qualitat)
- ✅ **(S21)** Docker Compose multi-servei: Qdrant + Java + Python + Streamlit en una comanda
- ✅ **(S22)** GitHub Actions: CI per tests Java/Python + evals d'IA (bloqueja si fidelitat < 80%)
- ✅ **(S23)** Desplegament cloud: Render/Fly.io amb secrets CLI auto-injectats
- ✅ **(S24)** Diagrama C4 + Portfolio: README professional, demostracions en viu, GitHub públic

**Lliurament:** Plataforma íntegra en cloud, accessible via URL pública, amb documentació sòlida

---

## Evolució del Projecte per Setmana

| Setmana | Tema | GamePulse Milestone | Status |
|---------|------|-------------------|--------|
| 1-6 | Algoritmica + POO | `v0.1`: Backend Java amb tests | Backend Squad |
| 7-10 | APIs + Output estructurat | `v0.2`: REST + CLI consumidor | API Squad |
| 11-14 | RAG | `v0.3`: Motor RAG amb Qdrant + anti-al·lucinació | RAG Pipeline |
| 15-18 | Agents + EDD | `v0.4`: Comitè multi-agent + evals CI | Agent Team |
| 19-24 | AI-SDLC + Deploy | `v1.0`: Producció cloud + portfolio | Release Candidate |

---

## Hypothetical Query Lifecycle

**Setmana 1-6:** "Quants jugadors actius té League of Legends?"
→ Consulta Java pura; resposta SQL

**Setmana 7-10:** "Quants jugadors actius té LoL?" 
→ Query REST → Java → CLI imprimeix resultat

**Setmana 11-14:** "Ha canviat el balanç del heroi Yasuo en els últims 2 anys?"
→ Query → RAG busca al Qdrant → Retorna 3 patches amb citació

**Setmana 15-18:** "Analitza si Yasuo està overpowered i quins canvis caldrien"
→ Query → Agent Stats (pulls data) → Agent RAG (busca patches) → Orquestrador sintetitza → LangFuse traça tot

**Setmana 19-24:** "Projecta l'evolució de Yasuo si les comunitats demanen nerf a l'AP scaling"
→ Query via Dashboard Streamlit → Comitè d'agents → Resultat amb mètriques de confiança → Desplegat en `api.gamepulse.io`

---

## Stack Final (v1.0)

```
Frontend:        Streamlit (Python)
Backend:         Spring Boot 3 (Java 21)
IA Agents:       LangGraph / Claude Code SDK / OpenAI Agents (to be chosen S15)
Vector DB:       Qdrant (Docker)
Observabilitat:  LangFuse
Infra:           Docker Compose + GitHub Actions + Render/Fly.io
Testing IA:      Pytest + Ragas + Eval datasets
```

---

## Competències Adquirides al Final

✅ Algorítmica aplicada (Big-O, estructures)
✅ Arquitectura Java clean + patterns (Factory, Repository)
✅ APIs REST + output estructurat
✅ RAG i busca híbrida
✅ Orquestració multi-agent
✅ Observabilitat i costos d'IA
✅ Testing d'IA amb datasets + mètriques
✅ AI-SDLC: prompts com specs, `.cursorrules` com guidelines
✅ Docker + CI/CD
✅ Desplegament cloud + secrets management

**Resultat:** Un developer junior que pot construir sistemes complets AI-powered end-to-end.
