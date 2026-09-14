# Setmana 14 — Dijous: Trobar i Avaluar Skills, MCP Servers i Plugins

## Objectiu del Dia

Aprendre a trobar, avaluar i configurar eines existents (MCP servers, skills, plugins) en lloc de construir-ho tot des de zero. Al final del dia tindràs configurat almenys un MCP server funcional i documentat al teu `CLAUDE.md`.

---

## Teoria

### Per Què Buscar Eines Existents és una Habilitat Crítica

Una de les diferències entre un desenvolupador junior i un senior no és quant codi escriuen, sinó quant codi **no** escriuen. Abans de programar res, un bon enginyer:

1. **Busca** si algú ja ho ha resolt
2. **Avalua** si la solució existent és prou bona
3. **Integra** en lloc de reinventar

```
┌─────────────────────────────────────────────────────────────┐
│  La Piràmide de Productivitat amb IA                        │
│                                                              │
│                    ┌───────────┐                             │
│                    │ Codi propi│  ← Últim recurs             │
│                  ┌─┴───────────┴─┐                           │
│                  │  MCP Servers   │  ← Eines composables     │
│                ┌─┴───────────────┴─┐                         │
│                │  Skills / Plugins  │  ← Regles per a IA     │
│              ┌─┴───────────────────┴─┐                       │
│              │ @docs / Context extern │  ← Documentació       │
│            ┌─┴───────────────────────┴─┐                     │
│            │  Coneixement base del model │  ← El que ja sap   │
│            └─────────────────────────────┘                   │
│                                                              │
│  Comença per baix. Només puja si cal.                       │
└─────────────────────────────────────────────────────────────┘
```

### MCP (Model Context Protocol): Què És

MCP és un protocol estàndard per connectar eines externes a un LLM. Un MCP server exposa **tools** (funcions) que la IA pot cridar:

```json
// Exemple: un MCP server per a PostgreSQL exposa tools com:
{
  "tools": [
    {
      "name": "query",
      "description": "Execute a SQL query against the database",
      "parameters": {
        "sql": "string"
      }
    },
    {
      "name": "list_tables",
      "description": "List all tables in the database"
    }
  ]
}
// La IA pot decidir "necessito saber quines taules hi ha"
// i cridar list_tables automàticament
```

### On Buscar Eines

| Recurs | Què Trobaràs | URL |
|--------|-------------|-----|
| modelcontextprotocol.io | Directori oficial de MCP servers | modelcontextprotocol.io |
| GitHub MCP servers | Repositori oficial amb servers verificats | github.com/modelcontextprotocol/servers |
| mcp.so | Directori comunitari de MCP servers | mcp.so |
| Cursor directory | Rules i configuracions per a Cursor | cursor.directory |
| npm / PyPI | Paquets publicats (busca "mcp-server-") | npmjs.com / pypi.org |

### Skills i Rules: Context per a la IA

Els skills i rules files (.cursorrules, CLAUDE.md, .github/copilot-instructions.md) donen context i instruccions a la IA sobre el teu projecte:

```markdown
<!-- Exemple de .cursorrules per a un projecte Spring Boot -->
# Project Rules

## Architecture
- Use Spring Boot 3.x with Java 21
- Follow hexagonal architecture: domain/ adapters/ ports/
- DTOs live in adapters, never in domain

## Naming
- REST endpoints: kebab-case (/champion-stats)
- Java classes: PascalCase (ChampionStatsService)
- Database tables: snake_case (champion_stats)

## Testing
- Unit tests with JUnit 5 + Mockito
- Integration tests with @SpringBootTest + Testcontainers
- Minimum 80% coverage on domain layer
```

```markdown
<!-- Exemple de CLAUDE.md per a EsportsPulse -->
# EsportsPulse — Project Context

## Stack
- Java 21 + Spring Boot 3 (backend REST API)
- Python 3.12 + FastAPI (AI services)
- PostgreSQL 16 (data store)
- Qdrant (vector store)
- Docker Compose for all services

## MCP Servers
- @modelcontextprotocol/server-postgres: DB queries
- (afegeix aquí els que instal·lis avui)

## Conventions
- Conventional Commits
- Feature branches: feature/week14-knowledge
```

---

## Activitat

### 1. Explorar el Directori de MCP Servers (20 min)

Visita els recursos i busca MCP servers rellevants per al teu projecte:

```bash
# Busca MCP servers per a PostgreSQL
# (ja tens PostgreSQL al docker-compose)

# Opció 1: El servidor oficial de PostgreSQL per a MCP
# https://github.com/modelcontextprotocol/servers/tree/main/src/postgres
npm install -g @modelcontextprotocol/server-postgres

# Opció 2: Busca a npm
npm search mcp-server-postgres
```

Documenta cada eina que trobis amb aquesta plantilla:

```markdown
<!-- Apunts de la recerca — guarda-ho en un fitxer temporal -->

## MCP Server: @modelcontextprotocol/server-postgres

- **URL:** https://github.com/modelcontextprotocol/servers/tree/main/src/postgres
- **Què fa:** Permet a la IA fer queries SQL, llistar taules, veure esquemes
- **Manteniment:** Oficial (ModelContextProtocol org) — actiu
- **Stars:** [mira-ho a GitHub]
- **Seguretat:** Només read? Read-write? Cal revisar permisos
- **Veredicte:** ✅ Instal·lar / ⚠️ Provar / ❌ Descartar
- **Notes:** [apunts personals]
```

### 2. Instal·lar i Configurar un MCP Server (25 min)

Configura el MCP server de PostgreSQL per a Claude Code o Cursor:

```json5
// Per a Claude Code: .claude/settings.json (o settings.local.json)
// Això permet que Claude accedeixi directament a la teva BD
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-postgres",
        "postgresql://esportspulse:password@localhost:5432/esportspulse"
      ]
    }
  }
}
```

```json5
// Per a Cursor: .cursor/mcp.json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": [
        "-y", 
        "@modelcontextprotocol/server-postgres",
        "postgresql://esportspulse:password@localhost:5432/esportspulse"
      ]
    }
  }
}
```

Verifica que funciona:
```bash
# Arrenca la BD si no està corrent
docker compose up -d postgres

# Prova el MCP server manualment (hauria de connectar)
npx @modelcontextprotocol/server-postgres \
  "postgresql://esportspulse:password@localhost:5432/esportspulse"
# Ctrl+C per sortir — si no dona error de connexió, funciona
```

### 3. Avaluar un MCP Server per a Qdrant (15 min)

Busca si existeix un MCP server per a Qdrant:

```bash
# Busca a npm
npm search mcp qdrant

# Busca a GitHub
# https://github.com/search?q=mcp+qdrant&type=repositories
```

```markdown
<!-- Documenta la recerca -->

## MCP Server: Qdrant

### Opcions trobades:
1. **[nom del paquet]**
   - URL: [...]
   - Stars: [...]
   - Últim commit: [...]
   - Mantingut? [sí/no]

2. **[alternativa, si n'hi ha]**
   - ...

### Avaluació:
- Si n'hi ha un de bo → instal·la'l i configura'l
- Si no n'hi ha cap de fiable → documenta que no existeix
  i que usarem el client Python directament (que ja funciona)

### Criteris d'avaluació aplicats:
- [ ] Manteniment actiu (commits recents, issues resoltes)
- [ ] Stars/downloads (indica adopció)
- [ ] Seguretat (no demana permisos excessius)
- [ ] Scope adequat (fa el que necessitem, no massa més)
- [ ] Documentació clara
```

### 4. Buscar Rules per al Projecte (15 min)

Busca regles i configuracions per a les tecnologies del teu projecte:

```bash
# Visita cursor.directory i busca:
# - Spring Boot rules
# - FastAPI rules  
# - Docker rules
# - Python best practices

# Si trobes regles útils, afegeix-les al projecte
```

Crea o actualitza el fitxer de regles del projecte:

```markdown
<!-- .cursorrules (a l'arrel del projecte) -->
# EsportsPulse — Project Rules

## Architecture
- Backend Java: Spring Boot 3.x, Java 21, Maven
- AI Services: Python 3.12, FastAPI/Pydantic
- Data: PostgreSQL 16 + Qdrant (vector store)
- Infrastructure: Docker Compose

## Java Conventions
- Use records for DTOs
- Use Optional for nullable returns, never null
- @RestController for REST endpoints
- @Service for business logic
- @Repository for data access (Spring Data JPA)

## Python Conventions  
- Use dataclasses or Pydantic models for data structures
- Type hints everywhere
- Virtual environment in ai-python/venv/

## API Design
- REST endpoints: kebab-case paths (/champion-stats)
- JSON responses with consistent error format
- Pagination with page/size parameters

## Testing
- Java: JUnit 5 + Mockito + @SpringBootTest
- Python: pytest + pytest-asyncio
- Integration tests use Testcontainers

## Git
- Conventional Commits (feat/fix/docs/test/refactor)
- Feature branches: feature/weekNN-description
- PR required for merge to main
```

### 5. Documentar Tot al CLAUDE.md (10 min)

Actualitza (o crea) el `CLAUDE.md` del projecte amb tot el que has configurat:

```markdown
<!-- CLAUDE.md a l'arrel del projecte -->
# EsportsPulse

## Project Overview
Full-stack esports analytics platform.
- Java 21 + Spring Boot 3 (REST API)
- Python 3.12 (AI services, knowledge engineering)
- PostgreSQL 16 + Qdrant (data + vector store)

## Quick Start
```bash
docker compose up -d          # Arrenca PostgreSQL + Qdrant
cd backend-java && mvn spring-boot:run   # Backend
cd ai-python && python -m uvicorn main:app  # AI service
```

## MCP Servers Configured
- **postgres**: @modelcontextprotocol/server-postgres
  - Accés directe a la BD per queries i exploració d'esquema
- **qdrant**: [el que hagis trobat, o "client Python directe"]

## Key Directories
- `backend-java/` — Spring Boot REST API
- `ai-python/src/knowledge/` — Knowledge engineering pipeline (S14)
- `ai-python/data/raw/` — Dades en brut per a ingestió
- `docker-compose.yml` — Infraestructura local

## Conventions
- See `.cursorrules` for detailed coding conventions
- Conventional Commits for git messages
```

---

## Checklist de Lliurament

- [ ] Almenys 3 MCP servers avaluats amb la plantilla (documentats)
- [ ] MCP server de PostgreSQL instal·lat i configurat (`.claude/settings.json` o `.cursor/mcp.json`)
- [ ] Recerca documentada sobre MCP server per a Qdrant
- [ ] Fitxer `.cursorrules` creat amb les convencions del projecte
- [ ] `CLAUDE.md` actualitzat amb MCP servers, estructura i quick start
- [ ] Pots explicar els criteris per avaluar si una eina/plugin val la pena o no
