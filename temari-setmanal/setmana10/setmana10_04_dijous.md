# Setmana 10 — Dijous: MCP Basics — El Protocol Universal per Eines d'IA

## Objectiu del Dia

Entendre MCP (Model Context Protocol) com a estandard de connexio entre eines i models d'IA, instal·lar un servidor MCP existent, i connectar-lo a Cursor o Claude Code per interactuar amb la base de dades del projecte EsportsPulse directament des del chat.

---

## Teoria

### Que es MCP?

MCP (Model Context Protocol) es un protocol obert creat per Anthropic que estandaritza com les aplicacions d'IA es connecten amb fonts de dades i eines externes. La millor analogia: **MCP es el USB de la IA**.

Abans d'USB, cada periferic tenia el seu propi connector. Ara, qualsevol dispositiu es connecta amb el mateix cable. MCP fa el mateix per a la IA: qualsevol eina que implementi MCP es pot connectar a qualsevol client compatible (Claude, Cursor, etc.).

```
Sense MCP:                           Amb MCP:
┌─────────┐                          ┌─────────┐
│ Claude   │──connector_A──→ BD      │ Claude   │
│          │──connector_B──→ Git     │          │──MCP──→ Qualsevol eina
│          │──connector_C──→ API     │ Cursor   │
│          │──connector_D──→ FS      │          │──MCP──→ Qualsevol eina
└─────────┘                          └─────────┘
```

### Arquitectura MCP

```
┌──────────────────┐     stdio / JSON-RPC 2.0     ┌──────────────────┐
│   MCP CLIENT     │ ◄──────────────────────────► │   MCP SERVER     │
│                  │                               │                  │
│  - Cursor IDE    │    El client envia requests   │  - El teu codi   │
│  - Claude Code   │    El server respon amb       │  - Servidor de   │
│  - Claude.ai     │    resultats o errors         │    BD, fitxers,  │
│  - Qualsevol app │                               │    APIs, etc.    │
│    compatible     │                               │                  │
└──────────────────┘                               └──────────────────┘
```

**Flux de comunicacio:**
1. El client MCP (Cursor, Claude Code) es connecta al servidor via `stdio` (entrada/sortida estandard)
2. El protocol es JSON-RPC 2.0 — missatges JSON amb `method`, `params` i `id`
3. El servidor exposa eines, recursos i prompts que el client pot utilitzar
4. L'usuari interactua pel chat, i el client decideix quines eines cridar

### Els Tres Primitius MCP

MCP defineix tres tipus de capacitats que un servidor pot exposar:

```
┌─────────────────────────────────────────────────────────────┐
│                    MCP SERVER                                │
│                                                              │
│  ┌──────────┐   ┌──────────────┐   ┌──────────────────┐    │
│  │  TOOLS   │   │  RESOURCES   │   │     PROMPTS      │    │
│  │          │   │              │   │                  │    │
│  │ Accions  │   │  Dades       │   │  Plantilles      │    │
│  │ que el   │   │  que el      │   │  de prompts      │    │
│  │ model    │   │  model pot   │   │  predefinides    │    │
│  │ pot      │   │  llegir      │   │  per a tasques   │    │
│  │ executar │   │              │   │  comunes         │    │
│  └──────────┘   └──────────────┘   └──────────────────┘    │
│                                                              │
│  Exemple:       Exemple:           Exemple:                  │
│  query_db()     schema.sql         "Analitza taula X"       │
│  insert_row()   config.json        "Genera migracioY"       │
│  run_migration  logs/app.log       "Revisa codi Z"          │
└─────────────────────────────────────────────────────────────┘
```

| Primitiu   | Que es                                     | Analogia                            |
|-----------|---------------------------------------------|--------------------------------------|
| **Tools** | Funcions que el model pot executar           | Metodes d'un controller REST         |
| **Resources** | Dades que el model pot llegir            | Endpoints GET d'una API              |
| **Prompts** | Plantilles de prompts predefinides        | Templates de missatges               |

### JSON-RPC 2.0: El Protocol de Comunicacio

MCP utilitza JSON-RPC 2.0 sobre `stdio`. Aixi es com es veuen els missatges:

```json
// Request del client al servidor (el client demana la llista de tools)
{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/list",
    "params": {}
}

// Response del servidor (retorna les tools disponibles)
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "tools": [
            {
                "name": "query",
                "description": "Executa una consulta SQL a la base de dades",
                "inputSchema": {
                    "type": "object",
                    "properties": {
                        "sql": {"type": "string", "description": "Consulta SQL"}
                    },
                    "required": ["sql"]
                }
            }
        ]
    }
}

// Request del client per executar una tool
{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/call",
    "params": {
        "name": "query",
        "arguments": {
            "sql": "SELECT name, win_rate FROM champions ORDER BY win_rate DESC LIMIT 5"
        }
    }
}

// Response del servidor amb el resultat de la query
{
    "jsonrpc": "2.0",
    "id": 2,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "name | win_rate\nJinx | 52.3\nAhri | 51.8\n..."
            }
        ]
    }
}
```

### Instal·lar un Servidor MCP Existent

Hi ha molts servidors MCP ja creats per la comunitat. Per a EsportsPulse, farem servir `mcp-server-sqlite` per connectar-nos a la base de dades del projecte.

```bash
# Opcio 1: mcp-server-sqlite — per a bases de dades SQLite
# Instal·la globalment amb npx (no cal instal·lar res permanentment)
npx -y @anthropic-ai/mcp-server-sqlite --db-path ./esportspulse.db

# Opcio 2: mcp-server-filesystem — per a accedir a fitxers del projecte
npx -y @anthropic-ai/mcp-server-filesystem /ruta/al/projecte
```

### Configuracio a Cursor IDE

Per connectar un servidor MCP a Cursor, crea el fitxer `.cursor/mcp.json` a l'arrel del projecte:

```json
{
    "mcpServers": {
        "esportspulse-db": {
            "command": "npx",
            "args": [
                "-y",
                "@anthropic-ai/mcp-server-sqlite",
                "--db-path",
                "./data/esportspulse.db"
            ]
        },
        "project-files": {
            "command": "npx",
            "args": [
                "-y",
                "@anthropic-ai/mcp-server-filesystem",
                "./esportspulse-engine"
            ]
        }
    }
}
```

### Configuracio a Claude Code

Per connectar un servidor MCP a Claude Code, edita `.claude/settings.json`:

```json
{
    "mcpServers": {
        "esportspulse-db": {
            "command": "npx",
            "args": [
                "-y",
                "@anthropic-ai/mcp-server-sqlite",
                "--db-path",
                "./data/esportspulse.db"
            ]
        }
    }
}
```

### Que Pots Fer amb MCP Connectat?

Un cop configurat, pots parlar amb la base de dades directament des del chat:

```
# Al chat de Cursor o Claude Code, simplement pregunta:

Tu: "Quins campions tenen un win rate superior al 52%?"

Claude (usa la tool query automaticament):
→ Executa: SELECT name, win_rate FROM champions WHERE win_rate > 52
→ Resposta: "Hi ha 3 campions amb win rate superior al 52%:
   - Jinx: 52.3%
   - Ahri: 52.1%
   - Thresh: 52.0%"

Tu: "Crea una taula per guardar les analisis generades per IA"

Claude (usa la tool query):
→ Executa: CREATE TABLE ai_analyses (
     id INTEGER PRIMARY KEY,
     champion_id INTEGER,
     analysis_json TEXT,
     created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   )
→ Resposta: "He creat la taula ai_analyses amb els camps..."
```

### Servidors MCP Populars

| Servidor                         | Que fa                                          |
|----------------------------------|--------------------------------------------------|
| `@anthropic-ai/mcp-server-sqlite`     | Consultes SQL a bases de dades SQLite      |
| `@anthropic-ai/mcp-server-filesystem` | Llegir i escriure fitxers del projecte     |
| `@anthropic-ai/mcp-server-github`     | Interactuar amb repos, issues, PRs         |
| `@anthropic-ai/mcp-server-postgres`   | Consultes a bases de dades PostgreSQL      |
| `@anthropic-ai/mcp-server-brave`      | Cerques web amb Brave Search               |

### MCP vs REST API: Per que un Nou Protocol?

```
REST API (el que ja coneixes):
- Dissenyat per a comunicacio entre serveis
- HTTP + JSON
- Cada API te el seu propi format
- L'humà escriu el codi que crida l'API

MCP (el que estem aprenent):
- Dissenyat per a comunicacio entre IA i eines
- stdio + JSON-RPC
- Format estandaritzat (tools/resources/prompts)
- La IA decideix quan i com cridar les eines
```

La diferencia clau: amb REST, TU escrius el codi que crida l'API. Amb MCP, la IA decideix quines eines cridar basant-se en la teva pregunta. Tu simplement parles en llenguatge natural.

---

## Activitat

### Pas 1: Prepara la Base de Dades (10 min)

```bash
# Crea una base de dades SQLite amb dades de prova per al projecte
# sqlite3 s'instal·la per defecte a macOS i la majoria de distribucions Linux
cd esportspulse-engine

sqlite3 data/esportspulse.db << 'EOF'
-- Crea la taula de campions (simplificada per a la prova)
CREATE TABLE IF NOT EXISTS champions (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    role TEXT NOT NULL,
    win_rate REAL,
    pick_rate REAL,
    patch TEXT
);

-- Insereix dades de prova — campions populars amb estadistiques
INSERT OR REPLACE INTO champions VALUES (222, 'Jinx', 'ADC', 52.3, 15.2, '14.10');
INSERT OR REPLACE INTO champions VALUES (103, 'Ahri', 'MID', 51.8, 12.1, '14.10');
INSERT OR REPLACE INTO champions VALUES (412, 'Thresh', 'SUPPORT', 50.5, 18.7, '14.10');
INSERT OR REPLACE INTO champions VALUES (238, 'Zed', 'MID', 49.8, 10.3, '14.10');
INSERT OR REPLACE INTO champions VALUES (157, 'Yasuo', 'MID', 48.2, 14.5, '14.10');
INSERT OR REPLACE INTO champions VALUES (39, 'Irelia', 'TOP', 49.1, 8.9, '14.10');
INSERT OR REPLACE INTO champions VALUES (236, 'Lucian', 'ADC', 50.1, 11.4, '14.10');
INSERT OR REPLACE INTO champions VALUES (267, 'Nami', 'SUPPORT', 52.7, 9.8, '14.10');

-- Verifica que les dades s'han inserit correctament
SELECT name, role, win_rate FROM champions ORDER BY win_rate DESC;
EOF
```

### Pas 2: Configura MCP al teu IDE (15 min)

**Opcio A — Cursor:**

Crea `.cursor/mcp.json` a l'arrel del projecte amb la configuracio de la teoria.

**Opcio B — Claude Code:**

Edita `.claude/settings.json` amb la configuracio de la teoria.

Reinicia l'IDE despres de guardar la configuracio.

### Pas 3: Prova la Connexio (20 min)

Obre el chat del teu IDE i fes les seguents consultes:

```
1. "Mostra'm tots els campions de la base de dades"
2. "Quins campions tenen un win rate superior al 51%?"
3. "Quin es el campio mes popular (pick rate mes alt)?"
4. "Crea una taula ai_analyses per guardar les analisis generades per IA"
5. "Descriu l'esquema complet de la base de dades"
```

Observa com l'IA:
- Detecta automaticament quina tool cridar
- Genera la SQL correcta
- Interpreta els resultats en llenguatge natural

### Pas 4: Reflexio (10 min)

Compara l'experiencia de:
1. Escriure SQL manualment al terminal
2. Preguntar en llenguatge natural al chat amb MCP

Pensa en com MCP pot accelerar el teu flux de treball diari: debug de dades, exploracio d'esquemes, generacio de queries complexes.

---

## Checklist de Lliurament

- [ ] Base de dades SQLite creada amb dades de prova de campions
- [ ] Servidor MCP instal·lat (`mcp-server-sqlite`)
- [ ] Configuracio MCP afegida a l'IDE (`.cursor/mcp.json` o `.claude/settings.json`)
- [ ] Connexio funciona: pots fer consultes SQL des del chat
- [ ] Has provat almenys 3 consultes diferents des del chat
- [ ] Entens la diferencia entre Tools, Resources i Prompts
- [ ] Entens el flux de comunicacio: Client <-> stdio/JSON-RPC <-> Server
