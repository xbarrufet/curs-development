# Setmana 21 — Dilluns: CLAUDE.md Definitiu: L'Especificacio del Projecte

## Objectiu del Dia

Escriure el fitxer `CLAUDE.md` definitiu del projecte EsportsPulse. Aquest fitxer sera la font unica de veritat per a qualsevol eina d'IA (Claude Code, Cursor, Copilot) que treballi amb el repositori. Al final del dia, un agent nou ha de poder entendre el projecte sense cap explicacio oral.

---

## Teoria

### Que es CLAUDE.md?

`CLAUDE.md` es un fitxer que descriu el projecte de manera que una IA el pugui entendre i contribuir-hi de forma efectiva. Es l'equivalent a un "onboarding document" pero optimitzat per a agents d'IA.

**Per que es important?**

Fins ara has treballat amb Claude en sessions individuals, explicant el context cada vegada. Amb un `CLAUDE.md` ben escrit, qualsevol sessio nova (teva o d'un company) comenca amb tot el context necessari.

```markdown
# Estructura d'un bon CLAUDE.md
# 1. Descripcio del projecte (que fa, per a qui)
# 2. Arquitectura (serveis, com es comuniquen)
# 3. Convencions de codi (naming, format, commits)
# 4. Com executar el projecte (comandes exactes)
# 5. Servidors MCP configurats (si n'hi ha)
# 6. Skills/Workflows disponibles
# 7. Com contribuir (branques, PR, tests)
```

### Les Seccions Clau

**1. Descripcio del Projecte**

No es un README generic. Ha de respondre:
- Que fa el sistema exactament?
- Quins problemes resol?
- Quins son els fluxos principals de l'usuari?

```markdown
# EsportsPulse Engine

## Descripcio
# Plataforma d'analisi d'esports electronics que combina:
# - API REST (Java/Spring Boot) per a dades d'equips i jugadors
# - Servei d'IA (Python/FastAPI) amb agents per a analisi quantitatiu
#   i knowledge retrieval
# - Dashboard (Streamlit) per a visualitzacio i interaccio
# - Infraestructura completa: PostgreSQL, Redis, RabbitMQ, Prometheus
```

**2. Arquitectura**

L'agent necessita saber quins serveis existeixen i com es connecten:

```markdown
## Arquitectura
# Serveis (docker-compose):
# - backend-java (port 8080): API REST Spring Boot, JWT auth
# - ai-python (port 8000): FastAPI, agents LangChain
# - streamlit (port 8501): Dashboard de visualitzacio
# - postgres (port 5432): Base de dades principal
# - redis (port 6379): Cache i sessions
# - rabbitmq (port 5672/15672): Cua de missatges async
# - prometheus (port 9090): Metriques
# - grafana (port 3000): Dashboards de monitoritzacio
```

**3. Convencions**

Sense convencions explicites, l'IA generara codi inconsistent:

```markdown
## Convencions
# Java: camelCase per variables, PascalCase per classes
# Python: snake_case per funcions i variables
# Commits: Conventional Commits (feat/fix/docs/test/refactor)
# Branques: feature/weekXX-descripcio, fix/descripcio
# Tests: cada endpoint ha de tenir test unitari + integracio
```

**4. Comandes d'Execucio**

L'agent ha de poder executar el projecte sense preguntar:

```markdown
## Execucio
# Arrencar tot: docker-compose up -d
# Només backend: docker-compose up backend-java
# Tests Java: mvn test -pl backend-java
# Tests Python: cd ai-python && pytest
# Lint: mvn checkstyle:check && cd ai-python && ruff check .
```

**5. Servidors MCP**

Si fas servir Model Context Protocol, documenta'ls:

```markdown
## MCP Servers
# - filesystem: acces al sistema de fitxers del projecte
# - postgres: consultes directes a la BD
# - docker: gestio de contenidors
```

**6. Com Contribuir**

Regles per a l'IA (i per a humans):

```markdown
## Contribucio
# 1. Crea branca des de main: git checkout -b feature/descripcio
# 2. Escriu tests ABANS del codi (TDD)
# 3. Tots els tests han de passar: mvn test && pytest
# 4. Commit amb Conventional Commits
# 5. PR amb descripcio del canvi i checklist
# 6. NO hardcodejar secrets — usa variables d'entorn
```

### Principis d'un Bon CLAUDE.md

1. **Especific, no generic**: "Usa Java 21 amb Spring Boot 3.2" es millor que "Usa Java".
2. **Executable**: Cada comanda ha de funcionar copy-paste.
3. **Actualitzat**: Si canvia l'arquitectura, actualitza el fitxer.
4. **Honest**: Si hi ha deute tecnic, documenta'l. L'IA treballara millor si sap les limitacions.

---

## Activitat

### 1. Auditar l'Estat Actual (20 min)

Abans d'escriure, revisa que tens i que falta:

```bash
# Llista tots els serveis del docker-compose
docker-compose config --services

# Revisa les variables d'entorn necessaries
grep -r "os.environ\|System.getenv\|process.env" --include="*.java" --include="*.py" .

# Llista els endpoints de l'API
grep -rn "@GetMapping\|@PostMapping\|@PutMapping\|@DeleteMapping" backend-java/

# Llista les rutes de FastAPI
grep -rn "@app.get\|@app.post\|@router.get\|@router.post" ai-python/
```

Apunta tot el que trobes. Sera el contingut del teu CLAUDE.md.

### 2. Escriure el CLAUDE.md Definitiu (60 min)

Crea el fitxer `CLAUDE.md` a l'arrel del projecte. Ha de contenir:

```markdown
# EsportsPulse Engine

## Descripcio del Projecte
<!-- Explica que fa el sistema, per a qui, i quins problemes resol.
     Sigues especific: menciona els agents, el knowledge retrieval,
     i el dashboard. -->

## Arquitectura
<!-- Diagrama textual dels serveis i les seves connexions.
     Inclou ports, protocols, i fluxos de dades principals. -->

## Stack Tecnologic
<!-- Llista cada tecnologia amb la seva versio exacta.
     Java 21, Spring Boot 3.2, Python 3.12, FastAPI, etc. -->

## Estructura del Repositori
<!-- Arbre de directoris simplificat amb explicacio de cada carpeta. -->

## Execucio
<!-- Comandes exactes per arrencar, testejar, i desplegar.
     Ha de funcionar amb copy-paste. -->

## Variables d'Entorn
<!-- Totes les variables necessaries amb valors d'exemple.
     MAI posar secrets reals aqui. -->

## Convencions de Codi
<!-- Naming, format, commits, branques, PRs. -->

## Servidors MCP
<!-- Si en tens de configurats, documenta'ls. -->

## Tests
<!-- Com executar tests, quina cobertura es espera,
     quins tipus de test hi ha (unitari, integracio, E2E). -->

## Com Contribuir
<!-- Flux de treball complet: branca -> codi -> test -> PR. -->

## Deute Tecnic Conegut
<!-- Coses que saps que caldria millorar. Ser honest ajuda l'IA. -->
```

**Important:** No copïs plantilles generiques. Cada seccio ha de reflectir el TEU projecte real.

### 3. Validar el CLAUDE.md (20 min)

Obre una sessio nova de Claude Code (o Cursor) i posa el `CLAUDE.md` com a context:

```bash
# Obre Claude Code al directori del projecte
# Claude Code llegira automaticament el CLAUDE.md
claude

# Demana-li alguna cosa que requereixi context del projecte:
# "Afegeix un endpoint GET /api/v1/teams/{id}/stats que retorni
#  estadistiques agregades de l'equip"
```

Observa:
- Genera codi amb les convencions correctes?
- Usa les tecnologies correctes?
- Sap on posar el fitxer?

Si falla en alguna d'aquestes, el CLAUDE.md necessita mes detall en aquella seccio.

### 4. Commit (10 min)

```bash
# Afegeix el CLAUDE.md i fes commit
git add CLAUDE.md
git commit -m "docs: add definitive CLAUDE.md as project specification"
```

---

## Checklist de Lliurament

- [ ] `CLAUDE.md` creat a l'arrel del projecte amb totes les seccions
- [ ] Descripcio especifica del projecte (no generica)
- [ ] Arquitectura amb tots els serveis i ports
- [ ] Comandes d'execucio que funcionen amb copy-paste
- [ ] Variables d'entorn documentades (sense secrets reals)
- [ ] Convencions de codi explicitades
- [ ] Validat amb una sessio nova de Claude Code
- [ ] Commit amb missatge descriptiu
