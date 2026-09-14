# Curs de Desenvolupament per a Juniors — AI-SDLC

Pla de formació de 24 setmanes per formar un enginyer de software junior (no només un developer). Combina fonaments d'enginyeria de software (algorítmica, SOLID, testing, APIs, SQL, seguretat, observabilitat) amb les noves formes de treballar amb IA (prompt engineering, MCP, agents, specs, eval-driven development). L'objectiu: codi que no només funciona, sinó que es pot operar, mantenir i escalar en producció.

**Projecte vehicle:** [EsportsPulse](esportspulse-project-brief.md) — un sistema multi-agent per analitzar champions, metes i estadístiques d'eSports de League of Legends (Java 21 + Python + Qdrant + Streamlit).

## Priorització Recomanada per Empleabilitat

Si l’objectiu és maximitzar la inserció laboral i la capacitat d’operar software real, la seqüència ideal és:

### Fase 1 — Base imprescindible (S1–S6)
- **Fundaments reals**: Big-O, estructures, POO, SOLID, testing, Git, CI.
- **Per què primer**: és la base que permet a un junior ser útil en qualsevol equip tecnològic.
- **Resultat buscats**: construir codi fiable, llegible i validable.

### Fase 2 — Productivitat i software operatiu (S7–S12)
- **REST + contractes + SQL + observabilitat + Docker**.
- **Per què ara**: són les habilitats més presents en projectes reals d’empresa i d’API/backend.
- **Resultat buscats**: servei que funciona, s’administra, es monitoritza i es desplega sense drama.

### Fase 3 — IA aplicada amb rigor (S13–S18)
- **Python per IA, prompt engineering, structured output, MCP, agents i specs**.
- **Per què després**: la IA és multiplicadora, no substitut de la base tècnica.
- **Resultat buscats**: treballar amb IA de manera productiva i segura, no com un chatbot.

### Fase 4 — Producció, portfolio i entrevista (S19–S24)
- **CI/CD, observabilitat, deploy, C4, docs, demo i preparació d’entrevista**.
- **Per què al final**: la demostració professional és la part que converteix el projecte en peça de portfolio.

### Resum executiu
La priorització ideal és:
1. **Construcció de software real**
2. **Infraestructura i operabilitat**
3. **IA aplicada amb disciplina**
4. **Presentació i portfolio**

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
| **Docker Desktop** | S11 (Docker, Qdrant, PostgreSQL) | [docker.com](https://www.docker.com/products/docker-desktop/) | Contenidors per BD, serveis, deploy |
| **PostgreSQL** | S15 | Via Docker (`docker run postgres:16`) | BD de producció (reemplaça H2) |
| **Streamlit** | S10 | `pip install streamlit` | Dashboard / spec-driven UI |
| **Qdrant** | S12 | Via Docker (`docker run qdrant/qdrant`) | Base vectorial per knowledge retrieval |
| **Postman** o **curl** | S7 | [postman.com](https://www.postman.com/downloads/) / ja instal·lat (macOS) | Testejar APIs REST manualment |
| **Compte PandaScore** | S3 (extractor) | [pandascore.co](https://pandascore.co) (gratuït) | API de tornejos i resultats d'eSports |
| **Redis** | S13 (cache LLM) | Via Docker (`docker run redis:7`) | Cache de respostes, sessions, queries |
| **RabbitMQ** | S16 (message queues) | Via Docker (`docker run rabbitmq:3-management`) | Events asíncrons entre serveis |
| **Claude Code CLI** | S17+ (agents) | [claude.ai/claude-code](https://claude.ai/claude-code) | Agent de codi per terminal |

### Configuració Python recomanada

```bash
# Crear entorn virtual (S5 formalment, però recomanat des de S1)
python3 -m venv .venv
source .venv/bin/activate  # macOS/Linux

# Dependències base (creixen al llarg del curs)
pip install pytest ruff requests           # S1-S6
pip install fastapi uvicorn pydantic       # S7-S9
pip install structlog tenacity             # S9
pip install streamlit plotly               # S10
pip install aiohttp qdrant-client          # S12-S13
pip install redis                          # S13
pip install python-jose bcrypt             # S14
pip install pika                           # S16
pip install langfuse openai anthropic      # S17
```

### Configuració Java recomanada

```bash
# Verificar versió
java --version   # Ha de ser 21+
mvn --version    # Ha de ser 3.9+

# El projecte usa Spring Boot 3 amb Maven
# Les dependències s'afegeixen al pom.xml progressivament:
# S1-S4: spring-boot-starter, junit-jupiter
# S5:    spring-boot-starter-data-jpa, h2
# S7:    spring-boot-starter-web, springdoc-openapi
# S14:   spring-boot-starter-security
# S15:   postgresql, flyway-core
```

### Claus API (fitxer `.env` a l'arrel, exclòs de Git)

```bash
# .env — MAI pujar a Git (afegir a .gitignore)
RIOT_API_KEY=your_riot_key_here        # developer.riotgames.com
PANDASCORE_API_KEY=your_key_here       # pandascore.co (tornejos)
OPENAI_API_KEY=your_key_here      # S8+
ANTHROPIC_API_KEY=your_key_here   # S8+ (alternativa)
```

## Estructura del Curs

| Bloc | Setmanes | Focus | Ratio mà/assistit |
|------|----------|-------|-------------------|
| **1. Fundaments** | S1–S6 | Algorítmica, POO, concurrència, testing, CI | 80% / 20% |
| **2. APIs i Integració** | S7–S10 | REST, Pydantic, LLMs, Streamlit, MCP basics | 50% / 50% |
| **3. Infraestructura, Knowledge i Integració** | S11–S16 | Docker, knowledge, auth (JWT), SQL, Redis, message queues | 30% / 70% |
| **4. Agents i Spec-Driven** | S17–S19 | Agents, tool use, eval-driven, spec-driven dev | 20% / 80% |
| **5. Producció, Portfolio i Entrevista** | S20–S24 | CI/CD, deploy cloud, C4, demo, entrevista tècnica | 10% / 90% |

## Competències Transversals

El curs té 5 fils que es treballen de forma progressiva cada setmana (no en blocs aïllats):

- **Prompt Engineering** — De prompts bàsics (S1) a specs per agents (S18).
- **Python com a Segon Llenguatge** — Exercicis mirall Java↔Python des de S1. A S8+ Python és co-protagonista.
- **Escriptura de Specs** — `.cursorrules` (S2) → API specs (S7) → specs d'agent (S17) → spec-driven development (S18).
- **Ecosistema d'Eines IA** — Hooks (S4) → MCP servers (S8-S9) → skills i plugins (S12, S18) → consolidació (S19).
- **Mindset de Codi Productiu** — Observabilitat, mantenibilitat, escalabilitat, resiliència i operabilitat. Des de S1 (benchmarks + logging bàsic) fins a S22 (troubleshooting amb logs i mètriques). L'objectiu no és formar un developer que escriu codi que funciona, sinó un enginyer de SW que escriu codi que es pot operar en producció.

## Contingut per Setmana

### Bloc 1: Fundaments

| Setmana | Tema | Material |
|---------|------|----------|
| **S1** | Rendiment i Big-O | [Pla](temari-setmanal/setmana01/setmana_01.md) · [Teoria](temari-setmanal/setmana01/teoria_setmana_01.md) · [Exercicis](temari-setmanal/setmana01/exercicis_consolidacio_01.md) · [.cursorrules template](temari-setmanal/setmana01/cursorrules-template-week1.md) |
| **S2** | POO, SOLID i Immutabilitat | [Pla](temari-setmanal/setmana02/setmana_02.md) · [Teoria](temari-setmanal/setmana02/teoria_setmana_02.md) · [Exercicis](temari-setmanal/setmana02/exercicis_consolidacio_02.md) |
| **S3** | Concurrència Pràctica Web (Race Conditions, CompletableFuture, asyncio) | [Pla](temari-setmanal/setmana03/setmana_03.md) · [Teoria](temari-setmanal/setmana03/teoria_setmana_03.md) · [Exercicis](temari-setmanal/setmana03/exercicis_consolidacio_03.md) |
| **S4** | Clean Code, Code Review, Git, CI | [Pla](temari-setmanal/setmana04/setmana_04.md) · [Teoria](temari-setmanal/setmana04/teoria_setmana_04.md) · [Exercicis](temari-setmanal/setmana04/exercicis_consolidacio_04.md) |
| **S5** | Factory, Repository, JPA | [Pla](temari-setmanal/setmana05/setmana_05.md) · [Teoria](temari-setmanal/setmana05/teoria_setmana_05.md) · [Exercicis](temari-setmanal/setmana05/exercicis_consolidacio_05.md) |
| **S6** | Testing, Mocks, Qualitat | [Pla](temari-setmanal/setmana06/setmana_06.md) · [Teoria](temari-setmanal/setmana06/teoria_setmana_06.md) · [Exercicis](temari-setmanal/setmana06/exercicis_consolidacio_06.md) |

### Bloc 2: APIs i Integració

| Setmana | Tema | Material |
|---------|------|----------|
| **S7** | APIs REST amb Spring Boot 3 | [Pla](temari-setmanal/setmana07/setmana_07.md) · [Teoria](temari-setmanal/setmana07/teoria_setmana_07.md) · [Exercicis](temari-setmanal/setmana07/exercicis_consolidacio_07.md) |
| **S8** | Python, Pydantic, LLMs, MCP basics | [Pla](temari-setmanal/setmana08/setmana_08.md) · [Teoria](temari-setmanal/setmana08/teoria_setmana_08.md) · [Exercicis](temari-setmanal/setmana08/exercicis_consolidacio_08.md) |
| **S9** | Error Handling, Logging, MCP pràctic | [Pla](temari-setmanal/setmana09/setmana_09.md) · [Teoria](temari-setmanal/setmana09/teoria_setmana_09.md) · [Exercicis](temari-setmanal/setmana09/exercicis_consolidacio_09.md) |
| **S10** | Streamlit, Spec-Driven UI, Testing E2E | [Pla](temari-setmanal/setmana10/setmana_10.md) · [Teoria](temari-setmanal/setmana10/teoria_setmana_10.md) · [Exercicis](temari-setmanal/setmana10/exercicis_consolidacio_10.md) |

### Bloc 3: Infraestructura, Knowledge i Integració

| Setmana | Tema | Material |
|---------|------|----------|
| **S11** | Docker i Docker Compose | *pendent* |
| **S12** | Knowledge Engineering, Qdrant | *pendent* |
| **S13** | Knowledge Retrieval, Anti-al·lucinació, Cache LLM (Redis) | *pendent* |
| **S14** | Autenticació (JWT), Seguretat Web | *pendent* |
| **S15** | SQL Avançat, PostgreSQL, Flyway | *pendent* |
| **S16** | Integració de Sistemes: Redis, Message Queues (RabbitMQ) | *pendent* |

### Bloc 4: Agents i Spec-Driven

| Setmana | Tema | Material |
|---------|------|----------|
| **S17** | Agents (API directa + tool use), Evals, LangFuse | *pendent* |
| **S18** | Spec-Driven Development, Skills, Hooks | *pendent* |
| **S19** | Consolidació: Dashboard Avançat, Specs Finals | *pendent* |

### Bloc 5: Producció, Portfolio i Entrevista

| Setmana | Tema | Material |
|---------|------|----------|
| **S20** | CI/CD Consolidació, Logging, Monitoring | *pendent* |
| **S21** | Desplegament Cloud (Render/Fly.io) | *pendent* |
| **S22** | Testing E2E, Hardening, Seguretat OWASP | *pendent* |
| **S23** | Arquitectura C4, Documentació, Portfolio | *pendent* |
| **S24** | Preparació Entrevista Tècnica, Demo Final | *pendent* |

## Estructura de Fitxers per Setmana

Cada setmana té fins a 4 fitxers:

```
temari-setmanal/setmanaXX/
├── setmana_XX.md                    # Pla dia a dia (dilluns-divendres)
├── teoria_setmana_XX.md             # Teoria amb exemples i diagrames
├── exercicis_consolidacio_XX.md     # 3 bàsics + 2 avançats (sense guia)
└── [recursos addicionals]           # Templates, snippets, etc.
```

- **Pla setmanal:** Activitats guiades pas a pas per cada dia, amb material de lectura i lliuraments.
- **Teoria:** Explicacions amb exemples aplicats a EsportsPulse, diagrames ASCII, codi Java + Python.
- **Exercicis de consolidació:** Reptes sense instruccions pas a pas. Bàsics (imprescindibles) i Avançats (opcionals). Cada exercici connecta amb setmanes anteriors.

## Documents Generals

- [Pla Global de 24 Setmanes](temari-global-24.md) — Visió completa amb competències transversals.
- [EsportsPulse Project Brief](esportspulse-project-brief.md) — Descripció del projecte vehicle, stack, roadmap.

## Principis del Curs

1. **Primer el dolor, després l'eina.** Les eines IA s'introdueixen quan l'estudiant ha viscut el problema que resolen.
2. **Empleabilitat sobre acadèmia.** Cada tema es justifica per "això ho faràs al primer mes de feina" o "això et preguntaran a l'entrevista", no per completesa teòrica.
3. **L'algorítmica necessària, no més.** Big-O i HashMap (S1) són suficients com a base. La resta d'algorítmica s'integra quan el context ho demana (indexes a S5, concurrència a S3).
4. **La IA és copilot, tu ets responsable.** L'estudiant aprèn a generar codi amb IA i a auditar-lo amb criteri: seguretat, rendiment, tests, mantenibilitat.
5. **Java + Python des del dia 1.** Exercicis mirall en Python cada setmana per demostrar que els conceptes són universals i per preparar la transició al Bloc 2.

## Llicència i Recursos Externs

El contingut original d'aquest curs (temaris, exercicis, teoria) es distribueix sota [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).

El material inclou links a recursos externs (articles, vídeos, documentació, llibres) com a referències educatives. Aquests recursos són propietat dels seus respectius autors i es referencien sense reproduir-ne el contingut:
- **Articles:** Baeldung, Atlassian, Google Engineering Practices (accés públic gratuït).
- **Vídeos:** YouTube (Midudev, MoureDev, i altres — contingut públic dels creadors).
- **Llibres:** *Pro Git* (Scott Chacon, CC BY-NC-SA 3.0), *Clean Code* (Robert C. Martin, Prentice Hall). Es referencien per títol i autor; no es reprodueix contingut.
- **Documentació:** Oracle Java, Spring, JUnit, GitHub (documentació oficial pública).

Si ets autor d'algun recurs referenciat i prefereixes que eliminem el link, obre un issue.
