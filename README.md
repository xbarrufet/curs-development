# Curs de Desenvolupament per a Juniors — AI-SDLC

Pla de formació de 24 setmanes per a un developer junior. Combina fonaments d'enginyeria de software (algorítmica, SOLID, testing, APIs, SQL, seguretat) amb les noves formes de treballar amb IA (prompt engineering, MCP, agents, specs, eval-driven development).

**Projecte vehicle:** [GamePulse](gamepulse-project-brief.md) — un sistema multi-agent per analitzar el balanç i evolució de videojocs (Java 21 + Python + Qdrant + Streamlit).

## Estructura del Curs

| Bloc | Setmanes | Focus | Ratio mà/assistit |
|------|----------|-------|-------------------|
| **1. Fundaments** | S1–S6 | Algorítmica, POO, concurrència, testing, CI | 80% / 20% |
| **2. APIs i Integració** | S7–S10 | REST, Pydantic, LLMs, Streamlit, MCP basics | 50% / 50% |
| **3. Knowledge i Seguretat** | S11–S14 | Knowledge engineering, auth (JWT), SQL avançat | 30% / 70% |
| **4. Agents i Docker** | S15–S18 | Multi-agent, tool use, Docker, spec-driven dev | 20% / 80% |
| **5. Producció i Portfolio** | S19–S24 | CI/CD, deploy cloud, C4, demo | 10% / 90% |

## Competències Transversals

El curs té 4 fils que es treballen de forma progressiva cada setmana (no en blocs aïllats):

- **Prompt Engineering** — De prompts bàsics (S1) a specs per agents (S19).
- **Python com a Segon Llenguatge** — Exercicis mirall Java↔Python des de S1. A S8+ Python és co-protagonista.
- **Escriptura de Specs** — `.cursorrules` (S2) → API specs (S7) → specs d'agent (S15) → spec-driven development (S18).
- **Ecosistema d'Eines IA** — Hooks (S4) → MCP servers (S8-S9) → skills i plugins (S11, S18) → consolidació (S19).

## Contingut per Setmana

### Bloc 1: Fundaments

| Setmana | Tema | Material |
|---------|------|----------|
| **S1** | Rendiment i Big-O | [Pla](temari-setmanal/setmana01/setmana_01.md) · [Teoria](temari-setmanal/setmana01/teoria_setmana_01.md) · [Exercicis](temari-setmanal/setmana01/exercicis_consolidacio_01.md) · [.cursorrules template](temari-setmanal/setmana01/cursorrules-template-week1.md) |
| **S2** | POO, SOLID i Immutabilitat | [Pla](temari-setmanal/setmana02/setmana_02.md) · [Teoria](temari-setmanal/setmana02/teoria_setmana_02.md) · [Exercicis](temari-setmanal/setmana02/exercicis_consolidacio_02.md) |
| **S3** | Concurrència Pràctica Web | [Pla](temari-setmanal/setmana03/setmana_03.md) · [Teoria](temari-setmanal/setmana03/teoria_setmana_03.md) · [Exercicis](temari-setmanal/setmana03/exercicis_consolidacio_03.md) |
| **S4** | Clean Code, Code Review, Git, CI | [Pla](temari-setmanal/setmana04/setmana_04.md) · [Teoria](temari-setmanal/setmana04/teoria_setmana_04.md) · [Exercicis](temari-setmanal/setmana04/exercicis_consolidacio_04.md) |
| **S5** | Factory, Repository, JPA | [Pla](temari-setmanal/setmana05/setmana_05.md) · [Teoria](temari-setmanal/setmana05/teoria_setmana_05.md) · [Exercicis](temari-setmanal/setmana05/exercicis_consolidacio_05.md) |
| **S6** | Testing, Mocks, Qualitat | *pendent* |

### Bloc 2: APIs i Integració

| Setmana | Tema | Material |
|---------|------|----------|
| **S7** | APIs REST amb Spring Boot 3 | [Pla](temari-setmanal/setmana07/setmana_07.md) |
| **S8** | Python, Pydantic, LLMs, MCP basics | *pendent* |
| **S9** | Error Handling, Logging, MCP pràctic | *pendent* |
| **S10** | Streamlit, Testing E2E | *pendent* |

### Bloc 3: Knowledge i Seguretat

| Setmana | Tema | Material |
|---------|------|----------|
| **S11** | Knowledge Engineering, Skills/MCP discovery | *pendent* |
| **S12** | Knowledge Retrieval, Anti-al·lucinació | *pendent* |
| **S13** | Autenticació (JWT), Seguretat Web | *pendent* |
| **S14** | SQL Avançat, PostgreSQL | *pendent* |

### Bloc 4: Agents i Docker

| Setmana | Tema | Material |
|---------|------|----------|
| **S15** | Agents, Tool Use, OpenSpec | *pendent* |
| **S16** | Multi-Agent, Eval-Driven Development | *pendent* |
| **S17** | Docker, Docker Compose | *pendent* |
| **S18** | Spec-Driven Development, Skills, Hooks | *pendent* |

### Bloc 5: Producció i Portfolio

| Setmana | Tema | Material |
|---------|------|----------|
| **S19** | Specs Finals, Dashboard Avançat | *pendent* |
| **S20** | CI/CD Consolidació, Logging, Monitoring | *pendent* |
| **S21** | Desplegament Cloud | *pendent* |
| **S22** | Testing E2E, Hardening | *pendent* |
| **S23** | Arquitectura C4, Documentació | *pendent* |
| **S24** | Portfolio, Demo, Presentació | *pendent* |

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
- **Teoria:** Explicacions amb exemples aplicats a GamePulse, diagrames ASCII, codi Java + Python.
- **Exercicis de consolidació:** Reptes sense instruccions pas a pas. Bàsics (imprescindibles) i Avançats (opcionals). Cada exercici connecta amb setmanes anteriors.

## Documents Generals

- [Pla Global de 24 Setmanes](temari-global-24.md) — Visió completa amb competències transversals.
- [GamePulse Project Brief](gamepulse-project-brief.md) — Descripció del projecte vehicle, stack, roadmap.

## Principis del Curs

1. **Primer el dolor, després l'eina.** Les eines IA s'introdueixen quan l'estudiant ha viscut el problema que resolen.
2. **Empliabilitat sobre acadèmia.** Cada tema es justifica per "això ho faràs al primer mes de feina" o "això et preguntaran a l'entrevista", no per completesa teòrica.
3. **L'algorítmica necessària, no més.** Big-O i HashMap (S1) són suficients com a base. La resta d'algorítmica s'integra quan el context ho demana (indexes a S5, concurrència a S3).
4. **La IA és copilot, tu ets responsable.** L'estudiant aprèn a generar codi amb IA i a auditar-lo amb criteri: seguretat, rendiment, tests, mantenibilitat.
5. **Java + Python des del dia 1.** Exercicis mirall en Python cada setmana per demostrar que els conceptes són universals i per preparar la transició al Bloc 2.
