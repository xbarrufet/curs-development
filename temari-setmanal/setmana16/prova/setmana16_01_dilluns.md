# Setmana 16 — Dilluns: El Patro Agent: Entendre el que Ja Fas Servir

## Objectiu del Dia

Entendre el patro agent — el bucle prompt -> LLM decideix eina -> executa -> resultat -> LLM decideix si ha acabat — i adonar-te que ja el fas servir cada dia amb Claude Code i Cursor. Al final del dia has de poder dibuixar el diagrama del bucle agent, identificar les parts (system prompt, eines, raonament, guardrails) i connectar-ho amb MCP (S10-S11).

---

## Teoria

### Que es un Agent?

Un agent es un programa on un LLM pren decisions en bucle. No es un simple "pregunta -> resposta". El model:

1. Rep una instruccio (prompt de l'usuari)
2. **Decideix** quina eina necessita (o cap)
3. Executa l'eina i rep el resultat
4. **Decideix** si ja pot respondre o necessita mes informacio
5. Si necessita mes, torna al pas 2

```
┌─────────────────────────────────────────────────┐
│                  BUCLE AGENT                     │
│                                                  │
│   Usuari: "Crea un endpoint GET /champions"      │
│       │                                          │
│       ▼                                          │
│   ┌─────────┐                                    │
│   │   LLM   │ ◄── System Prompt + Eines          │
│   │ (raona) │     disponibles                    │
│   └────┬────┘                                    │
│        │                                         │
│        ▼                                         │
│   Decideix: "Necessito llegir l'estructura"      │
│        │                                         │
│        ▼                                         │
│   ┌──────────────┐                               │
│   │ Eina: Read   │ → Llegeix fitxers del projecte│
│   └──────┬───────┘                               │
│          │ resultat                               │
│          ▼                                        │
│   ┌─────────┐                                    │
│   │   LLM   │ "Ara se l'estructura, puc escriure"│
│   │ (raona) │                                    │
│   └────┬────┘                                    │
│        │                                         │
│        ▼                                         │
│   ┌──────────────┐                               │
│   │ Eina: Write  │ → Crea ChampionController.java│
│   └──────┬───────┘                               │
│          │ resultat                               │
│          ▼                                        │
│   ┌─────────┐                                    │
│   │   LLM   │ "Fet. Retorno resposta a l'usuari" │
│   └─────────┘                                    │
│                                                  │
└─────────────────────────────────────────────────┘
```

**La diferencia clau amb un chatbot classic:**
- Chatbot: pregunta -> resposta (1 pas)
- Agent: pregunta -> [raona -> actua -> observa] x N -> resposta

### Tu Ja Fas Servir Agents Cada Dia

Quan demanes a Claude Code "crea un endpoint GET /champions", passa exactament aixo:

1. **System prompt:** Claude Code te instruccions internes (quines eines pot usar, en quin directori treballa, quines regles seguir)
2. **Eines disponibles:** `Read` (llegir fitxers), `Write` (escriure fitxers), `Edit` (modificar fitxers), `Bash` (executar comandes), `Search` (buscar al codi)
3. **Raonament:** El model decideix "primer llegire l'estructura del projecte per entendre on va l'endpoint"
4. **Execucio:** Crida l'eina `Read` amb el path del fitxer
5. **Avaluacio:** Amb el resultat, decideix el seguent pas
6. **Guardrails:** Hi ha limits — no pot esborrar fitxers sense permis, no pot executar comandes perilloses sense confirmacio

### Anatomia d'una Eina (Tool)

Cada eina que un agent pot usar te exactament la mateixa estructura — i es la mateixa que vas veure a MCP (S10-S11):

```json
{
  "name": "read_file",
  "description": "Llegeix el contingut d'un fitxer del projecte",
  "input_schema": {
    "type": "object",
    "properties": {
      "path": {
        "type": "string",
        "description": "Ruta absoluta al fitxer"
      }
    },
    "required": ["path"]
  }
}
```

**Connexio amb MCP:** Quan vas crear el teu servidor MCP a la setmana 10, vas definir eines amb `name`, `description` i `input_schema`. Un agent consumeix eines exactament amb aquest format — siguin eines locals o eines MCP remotes. El protocol es el mateix.

```
Agent = LLM + System Prompt + [Eina1, Eina2, ..., EinaN] + Bucle
                                 │
                    Poden ser eines locals O eines MCP
```

### Guardrails: Limits de l'Agent

Un agent sense limits es perillós. Els guardrails son regles que controlen que pot fer:

| Guardrail | Exemple a Claude Code | Per que existeix |
|---|---|---|
| **Confirmacio** | "Vols que executi `rm -rf`?" | Evitar accions destructives |
| **Llista blanca** | Nomes pot usar eines definides | Evitar acces no autoritzat |
| **Timeout** | Comanda max 2 minuts | Evitar bucles infinits |
| **Scope** | Nomes fitxers dins el projecte | Evitar acces a dades externes |

> **Lectura recomanada (no bloquejant):**
> - [Anthropic: Building effective agents](https://docs.anthropic.com/en/docs/build-with-claude/agentic) — El document de referencia
> - [Tool use (function calling)](https://docs.anthropic.com/en/docs/build-with-claude/tool-use) — Com funciona tool use a l'API

---

## Activitat

### 1. Tracejar una Sessio de Claude Code (30 min)

Obre Claude Code al projecte `esportspulse-engine` i demana:

> "Crea un endpoint GET /api/agents/health que retorni `{"status":"ok","module":"agents"}`"

**Mentre Claude Code treballa, observa i apunta:**

1. **Quines eines crida?** (Read, Write, Edit, Bash...)
2. **En quin ordre?** (Primer llegeix? Primer escriu?)
3. **Quantes vegades "raona" abans d'actuar?**
4. **Hi ha algun moment on demana confirmacio?** (guardrail)
5. **Com decideix que ha acabat?**

Crea un fitxer `docs/agent-trace-s16.md` amb les teves observacions:

```markdown
# Traca Agent — Sessio Claude Code

## Peticio
"Crea un endpoint GET /api/agents/health..."

## Passos Observats
1. [Eina: Read] Llegeix `src/main/java/...` — Per entendre l'estructura
2. [Raonament] Decideix on crear el controller
3. [Eina: Write] Crea `AgentHealthController.java`
4. ...

## Eines Utilitzades
- Read: X vegades
- Write: X vegades
- Edit: X vegades
- Bash: X vegades

## Guardrails Observats
- Moment on ha demanat confirmacio: ...
- Limits que ha respectat: ...
```

### 2. Dibuixar el Diagrama del Bucle Agent (15 min)

Amb el que has observat, dibuixa (en paper, Excalidraw, o ASCII art) el diagrama del bucle agent amb:

- **Entrada:** Prompt de l'usuari
- **Decisio:** LLM raona sobre que fer
- **Accio:** Crida a una eina
- **Observacio:** Rep el resultat
- **Bucle:** Torna a decidir fins que acaba
- **Sortida:** Resposta final a l'usuari

Inclou els guardrails com a "portes" al diagrama (punts on el flux pot ser aturat o redirigit).

### 3. Mapejar Eines Agent ↔ Eines MCP (15 min)

Crea una taula comparativa al mateix document:

```markdown
## Comparacio Eines Agent vs MCP

| Concepte | Agent (Claude Code) | MCP (S10-S11) |
|---|---|---|
| Definicio d'eina | JSON schema intern | JSON schema al servidor MCP |
| Qui decideix usar-la | El LLM | El LLM |
| Qui l'executa | El runtime de Claude Code | El servidor MCP |
| Format input | JSON amb schema | JSON amb schema |
| Format output | Text/JSON | Text/JSON |
| Descobriment | Configurades al system prompt | `tools/list` del protocol MCP |
```

### 4. Reflexio: Per Que Importa Entendre Aixo (10 min)

Afegeix una seccio final al document responent:

1. **Si Claude Code ja funciona be, per que he d'entendre com funciona per dins?**
   (Pista: per poder construir els teus propis agents adaptats al teu domini)
2. **Quina relacio hi ha entre les eines MCP que vas crear i les eines d'un agent?**
   (Pista: un agent pot usar eines MCP com a eines propies)
3. **Quins problemes podrien sorgir si un agent no te guardrails?**
   (Pista: pensa en bucles infinits, accions destructives, costos descontrolats)

---

## Checklist de Lliurament

- [ ] L'endpoint `/api/agents/health` funciona (creat per Claude Code)
- [ ] Tens el document `docs/agent-trace-s16.md` amb la traca completa de la sessio
- [ ] El diagrama del bucle agent inclou: entrada, decisio, accio, observacio, bucle, sortida i guardrails
- [ ] La taula comparativa Agent vs MCP esta completa amb 6+ files
- [ ] Has respost les 3 preguntes de reflexio amb exemples concrets
- [ ] Commit: `feat(agents): add agent pattern documentation and health endpoint`
