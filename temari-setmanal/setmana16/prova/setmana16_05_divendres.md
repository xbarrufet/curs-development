# Setmana 16 — Divendres: OpenSpec: Documentar els Agents Formalment

## Objectiu del Dia

Escriure una especificacio formal (OpenSpec) per a cada agent: system prompt, eines amb schemas, guardrails, format de resposta i exemples. Al final del dia, un altre desenvolupador (o Claude Code) ha de poder recrear l'agent nomes llegint l'spec. Farem la prova: donaras l'spec a Claude Code i veuras si genera un agent equivalent.

---

## Teoria

### Per Que Documentar un Agent Formalment

Fins ara, l'agent "viu" dins el codi. El system prompt esta dins un string Python, les eines estan definides com a diccionaris, i els guardrails estan escampats pel codi. Problemes:

1. **Un nou dev no sap que fa l'agent** sense llegir tot el codi
2. **No es clar que pot i que NO pot fer** — els limits estan implicits
3. **No es pot revisar sense executar** — has de provar-lo per entendre'l
4. **No es pot versionar facilment** — canvis al prompt es perden entre commits

### OpenSpec: Format d'Especificacio d'Agents

Un OpenSpec es un document YAML/Markdown que descriu un agent de forma completa i independent del codi:

```yaml
# Estructura d'un OpenSpec
agent:
  name: "Nom de l'agent"
  version: "1.0.0"
  description: "Que fa l'agent en una frase"

role_prompt: |
  El system prompt complet de l'agent.
  Inclou personalitat, regles, estrategia.

tools:
  - name: "nom_eina"
    description: "Que fa"
    input_schema:
      type: object
      properties:
        param1:
          type: string
          description: "Que es"
      required: ["param1"]
    output_schema:
      type: object
      properties:
        result:
          type: array
          description: "Que retorna"
    examples:
      - input: {"param1": "valor"}
        output: [{"key": "value"}]

guardrails:
  - id: "GR-001"
    rule: "Mai inventar dades"
    enforcement: "system_prompt + eval test"
  - id: "GR-002"
    rule: "Maxim 10 crides a eines per conversa"
    enforcement: "code (max_steps=10)"

response_format:
  language: "catala"
  style: "informative, objectiu"
  citations: "sempre citar la font (versio de patch)"

examples:
  - question: "Pregunta d'exemple"
    expected_response: "Resposta esperada"
    tools_used: ["eina1"]
    notes: "Observacions sobre el comportament esperat"
```

### Beneficis de l'OpenSpec

| Benefici | Sense OpenSpec | Amb OpenSpec |
|---|---|---|
| **Onboarding** | "Llegeix el codi" | "Llegeix l'spec (2 min)" |
| **Revisio** | Cal executar l'agent | Es pot revisar el document |
| **Reproduibilitat** | Depen de l'entorn | Qualsevol pot recrear-lo |
| **Versionat** | Diffs de codi criptics | Diffs clars i llegibles |
| **Testing** | Inventes preguntes | Els exemples de l'spec son tests |
| **Comunicacio** | "L'agent fa coses" | Contracte formal amb el stakeholder |

### Connexio amb MCP

Recorda de S10-S11: un servidor MCP exposa eines amb `name`, `description` i `input_schema`. L'OpenSpec fa el mateix pero a nivell d'agent complet — no nomes les eines, sino tambe el comportament, els limits i els exemples.

```
MCP Server Spec:  Eines disponibles (QUE pot fer)
OpenSpec Agent:   Eines + Comportament + Limits + Exemples (QUE, COM, QUAN, PER QUE)
```

> **Lectura recomanada (no bloquejant):**
> - [Anthropic: Tool use best practices](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/best-practices) — Com descriure eines
> - [OpenAPI Specification](https://swagger.io/specification/) — Inspiracio per a formats d'especificacio

---

## Activitat

### 1. Escriure l'OpenSpec de l'Agent Quantitatiu (25 min)

Crea el fitxer d'especificacio:

```yaml
# fitxer: ai-python/src/agents/specs/quantitative_agent_spec.yaml
# OpenSpec v1 — Agent Quantitatiu d'EsportsPulse

agent:
  name: "Agent Quantitatiu"
  version: "1.0.0"
  description: >
    Agent especialitzat en respondre preguntes sobre estadistiques
    de champions de League of Legends. Treballa amb dades estructurades
    (win rate, pick rate, ban rate, KDA, gold per minut).
  model: "claude-sonnet-4-20250514"
  max_tokens: 1024

role_prompt: |
  Ets l'Agent Quantitatiu d'EsportsPulse. El teu rol es respondre
  preguntes sobre estadistiques de champions de League of Legends.

  REGLES:
  1. Nomes respons sobre estadistiques i dades objectives
  2. Si no tens dades suficients, digues-ho clarament — NO invents dades
  3. Usa SEMPRE les eines per obtenir informacio abans de respondre
  4. Respon en catala
  5. Quan facis comparacions, mostra les dades de cada champion
  6. Si l'usuari pregunta sobre temes fora del teu ambit, indica que nomes
     gestiones estadistiques de champions

  ESTRATEGIA:
  - Preguntes generals → query_champions amb filtres apropiats
  - Preguntes sobre un champion concret → get_champion_stats
  - Comparacions → get_champion_stats per cada champion

tools:
  - name: "query_champions"
    description: >
      Consulta la llista de champions amb filtres opcionals.
      Retorna nom, rol i estadistiques basiques.
      Usa-la per preguntes generals o per trobar champions amb certs criteris.
    input_schema:
      type: object
      properties:
        role:
          type: string
          description: "Filtra per rol"
          enum: ["top", "jungle", "mid", "adc", "support"]
        min_win_rate:
          type: number
          description: "Win rate minim (0-100)"
        limit:
          type: integer
          description: "Nombre maxim de resultats (defecte 10)"
      required: []
    output_schema:
      type: array
      items:
        type: object
        properties:
          id: { type: string }
          name: { type: string }
          role: { type: string }
          win_rate: { type: number }
          pick_rate: { type: number }
          ban_rate: { type: number }
          kda: { type: number }
          gold_per_min: { type: integer }
    examples:
      - input: { "role": "mid", "min_win_rate": 52 }
        output:
          - { "id": "lux", "name": "Lux", "role": "mid", "win_rate": 53.1 }
          - { "id": "ahri", "name": "Ahri", "role": "mid", "win_rate": 52.3 }
      - input: { "limit": 3 }
        output:
          - { "id": "lux", "name": "Lux", "win_rate": 53.1 }
          - { "id": "ahri", "name": "Ahri", "win_rate": 52.3 }
          - { "id": "jinx", "name": "Jinx", "win_rate": 51.8 }

  - name: "get_champion_stats"
    description: >
      Obte estadistiques detallades d'un champion especific:
      win rate, pick rate, ban rate, KDA, gold per minut.
      Usa-la quan l'usuari pregunta per un champion concret.
    input_schema:
      type: object
      properties:
        champion_id:
          type: string
          description: "ID del champion en minuscules (ex: 'ahri', 'jinx')"
      required: ["champion_id"]
    output_schema:
      type: object
      properties:
        id: { type: string }
        name: { type: string }
        role: { type: string }
        win_rate: { type: number }
        pick_rate: { type: number }
        ban_rate: { type: number }
        kda: { type: number }
        gold_per_min: { type: integer }
    examples:
      - input: { "champion_id": "ahri" }
        output: { "id": "ahri", "name": "Ahri", "role": "mid", "win_rate": 52.3, "pick_rate": 8.1, "ban_rate": 5.2, "kda": 2.8, "gold_per_min": 412 }
      - input: { "champion_id": "champion_inexistent" }
        output: { "error": "Champion 'champion_inexistent' no trobat" }

guardrails:
  - id: "GR-Q001"
    rule: "No inventar estadistiques — sempre usar eines"
    enforcement: "system_prompt (regla 2 i 3) + eval test (quant-005)"
  - id: "GR-Q002"
    rule: "Rebutjar preguntes fora d'ambit"
    enforcement: "system_prompt (regla 6) + eval test (quant-007)"
  - id: "GR-Q003"
    rule: "Maxim 10 crides a eines per conversa"
    enforcement: "code (max_steps=10 al bucle)"
  - id: "GR-Q004"
    rule: "Respondre sempre en catala"
    enforcement: "system_prompt (regla 4)"

response_format:
  language: "catala"
  style: "informatiu, objectiu, amb dades numèriques"
  structure: >
    Per comparacions: taula o llista amb dades de cada champion.
    Per consultes simples: frase amb la dada i context.

examples:
  - question: "Quin champion te el win rate mes alt?"
    expected_behavior: "Crida query_champions sense filtres, identifica Lux amb 53.1%"
    expected_tools: ["query_champions"]
    expected_response_contains: ["Lux", "53.1"]

  - question: "Compara Ahri i Lux al mid"
    expected_behavior: "Crida get_champion_stats per Ahri i per Lux, compara"
    expected_tools: ["get_champion_stats", "get_champion_stats"]
    expected_response_contains: ["Ahri", "Lux", "52.3", "53.1"]

  - question: "Quin es el millor restaurant?"
    expected_behavior: "Rebutja la pregunta educadament, no usa eines"
    expected_tools: []
    expected_response_contains: ["estadistiques", "champions"]
```

### 2. Escriure l'OpenSpec de l'Agent de Knowledge (25 min)

```yaml
# fitxer: ai-python/src/agents/specs/knowledge_agent_spec.yaml
# OpenSpec v1 — Agent de Knowledge d'EsportsPulse

agent:
  name: "Agent de Knowledge"
  version: "1.0.0"
  description: >
    Agent especialitzat en respondre preguntes sobre patch notes,
    canvis de meta i histories de champions. Treballa amb informacio
    no estructurada provinent del sistema de retrieval (Qdrant + embeddings).
  model: "claude-sonnet-4-20250514"
  max_tokens: 1024

role_prompt: |
  Ets l'Agent de Knowledge d'EsportsPulse. El teu rol es respondre
  preguntes sobre patch notes, canvis de meta i histories de champions
  de League of Legends.

  REGLES CRITIQUES:
  1. MAI invents informacio. Si no la trobes amb les eines, digues:
     "No tinc informacio sobre aixo als patch notes indexats"
  2. SEMPRE cita la font: indica la versio del patch i la data
  3. Si els resultats de la cerca NO son rellevants, digues-ho
  4. Distingeix FETS (dades del patch) d'INTERPRETACIO (la teva analisi)
  5. Per comparacions entre patches, fes cerques separades per cada un
  6. Respon en catala

  ESTRATEGIA:
  - Pregunta sobre un champion → search_patches amb el nom del champion
  - Pregunta comparativa → multiples search_patches (un per element)
  - Pregunta sobre un patch concret → get_patch_detail amb l'ID
  - Pregunta d'evolucio temporal → multiples cerques per cada patch

tools:
  - name: "search_patches"
    description: >
      Cerca semantica als patch notes de League of Legends.
      Retorna fragments rellevants amb la versio del patch i data.
      Usa-la per trobar canvis de champions, nerfs, buffs i ajustos.
    input_schema:
      type: object
      properties:
        query:
          type: string
          description: "Cerca en llenguatge natural"
        top_k:
          type: integer
          description: "Nombre de resultats (defecte 3)"
          default: 3
      required: ["query"]
    output_schema:
      type: array
      items:
        type: object
        properties:
          patch_version: { type: string }
          patch_date: { type: string }
          champion: { type: string }
          change_type: { type: string, enum: ["buff", "nerf", "adjust"] }
          detail: { type: string }
          relevance_score: { type: number }
    examples:
      - input: { "query": "canvis Ahri patch 14.8", "top_k": 2 }
        output:
          - { "patch_version": "14.8", "champion": "Ahri", "change_type": "nerf", "detail": "Charm (E): durada reduida..." }
      - input: { "query": "Yasuo patch 14.15" }
        output: { "message": "Cap resultat trobat per aquesta cerca" }

  - name: "get_patch_detail"
    description: >
      Obte el detall complet d'un patch especific amb tots els canvis.
      Usa-la quan l'usuari pregunta per un patch concret.
    input_schema:
      type: object
      properties:
        patch_id:
          type: string
          description: "ID del patch (ex: 'patch-14.10')"
      required: ["patch_id"]
    output_schema:
      type: object
      properties:
        id: { type: string }
        version: { type: string }
        date: { type: string }
        changes: { type: array }
    examples:
      - input: { "patch_id": "patch-14.10" }
        output: { "id": "patch-14.10", "version": "14.10", "date": "2024-05-15", "changes": ["..."] }

guardrails:
  - id: "GR-K001"
    rule: "Mai inventar informacio sobre patches — anti-hallucinacio"
    enforcement: "system_prompt (regla 1) + eval test (know-005)"
  - id: "GR-K002"
    rule: "Sempre citar la font (versio del patch)"
    enforcement: "system_prompt (regla 2)"
  - id: "GR-K003"
    rule: "Indicar quan els resultats no son rellevants"
    enforcement: "system_prompt (regla 3)"
  - id: "GR-K004"
    rule: "Maxim 10 crides a eines per conversa"
    enforcement: "code (max_steps=10 al bucle)"
  - id: "GR-K005"
    rule: "Distingir fets d'interpretacio"
    enforcement: "system_prompt (regla 4)"

response_format:
  language: "catala"
  style: "informatiu, amb citacions de fonts"
  citations: "Sempre indicar 'Segons el patch X.Y (data)...'"
  structure: >
    Per histories d'un champion: cronologic, de mes antic a mes recent.
    Per comparacions: llista paralela amb les dades de cada patch.
    Per resums de patch: agrupat per champion amb tipus de canvi.

examples:
  - question: "Quins canvis ha tingut Ahri recentment?"
    expected_behavior: "Cerca 'canvis Ahri', retorna resultats de multiples patches"
    expected_tools: ["search_patches"]
    expected_response_contains: ["Ahri", "patch"]

  - question: "Com ha canviat Ahri entre el patch 14.8 i el 14.12?"
    expected_behavior: "Dues cerques separades, una per cada patch, sintetitza"
    expected_tools: ["search_patches", "search_patches"]
    min_tool_calls: 2
    expected_response_contains: ["14.8", "14.12"]

  - question: "Quins canvis ha tingut Yasuo al patch 14.15?"
    expected_behavior: "Cerca, no troba resultats, informa que no te dades"
    expected_tools: ["search_patches"]
    expected_response_contains: ["no tinc informacio"]
    expected_response_not_contains: ["Yasuo va rebre"]
```

### 3. La Prova de Foc: Recrear l'Agent des de l'Spec (20 min)

Obre Claude Code i dona-li l'spec:

> "Llegeix el fitxer `ai-python/src/agents/specs/quantitative_agent_spec.yaml` i implementa l'agent complet en Python seguint exactament l'especificacio. Usa l'API de Claude amb tool use. Guarda'l a `ai-python/src/agents/quantitative_agent_v2.py`."

**Compara el resultat:**
1. El system prompt es equivalent al teu?
2. Les eines estan definides correctament?
3. Els guardrails estan implementats?
4. El bucle agent funciona?

Si Claude Code pot recrear un agent funcional nomes amb l'spec, l'spec es bona. Si no, identifica que falta i millora-la.

### 4. Revisio Creuada (15 min)

Opcio A — **Amb un company:** Intercanvieu els OpenSpec. Cada un intenta entendre l'agent de l'altre NOMES llegint l'spec. Preguntes a respondre:
- Que fa l'agent?
- Quines eines te?
- Quins limits te?
- Com gestiona errors?

Opcio B — **Sol:** Dona l'spec de l'Agent de Knowledge a Claude Code amb el prompt:

> "Actua com un nou desenvolupador que acaba d'entrar al projecte. Llegeix l'spec a `specs/knowledge_agent_spec.yaml`. Sense mirar cap altre fitxer, respon: 1) Que fa aquest agent? 2) Quines eines te? 3) Que NO pot fer? 4) Com sabem si funciona be?"

Si Claude Code pot respondre les 4 preguntes nomes amb l'spec, esta ben escrita.

### 5. Commit i PR de Setmana (10 min)

```bash
# Afegeix tots els fitxers de la setmana
git add ai-python/src/agents/
git add docs/agent-trace-s16.md
git add .github/workflows/agent-evals.yml

# Commit final de la setmana
git commit -m "feat(agents): add OpenSpec for both agents, complete S16 agent module"

# Crea la PR
git push -u origin feature/week16-agents
```

Crea la PR amb un resum que inclogui:
- Que s'ha construit (2 agents, evals, observabilitat, specs)
- Decisions de disseny (per que tool use directe sense framework)
- Metriques (fidelitat dels evals, cost per consulta)
- Seguents passos (S17: multi-agent, S18: agent de produccio)

---

## Checklist de Lliurament

- [ ] `specs/quantitative_agent_spec.yaml` complet amb: role_prompt, 2 eines, 3+ guardrails, 3+ exemples
- [ ] `specs/knowledge_agent_spec.yaml` complet amb: role_prompt, 2 eines, 5+ guardrails, 3+ exemples
- [ ] Claude Code ha pogut recrear un agent funcional nomes amb l'spec
- [ ] La revisio creuada (company o Claude Code) ha validat que l'spec es comprensible
- [ ] PR creada amb tot el treball de la setmana 16
- [ ] Commit: `feat(agents): add OpenSpec for both agents, complete S16 agent module`
