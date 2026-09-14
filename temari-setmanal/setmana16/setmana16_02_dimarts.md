# Setmana 16 — Dimarts: Tool Use amb l'API de Claude: L'Agent Quantitatiu

## Objectiu del Dia

Construir un agent des de zero usant l'API de Claude amb tool use — sense frameworks, sense magia. Al final del dia tindras un "Agent Quantitatiu" que pot respondre preguntes sobre estadistiques de champions consultant les dades de l'EsportsPulse a traves d'eines que tu defineixes.

---

## Teoria

### Tool Use: Com Funciona Per Dins

Quan envies un missatge a l'API de Claude amb eines definides, el model pot decidir cridar-ne una. El flux es:

```
1. Tu envies:    messages + tools (definicions JSON)
2. Claude respon: "Vull cridar l'eina X amb parametres Y"
                  (stop_reason: "tool_use")
3. Tu executes:  L'eina X amb parametres Y → obtens resultat Z
4. Tu envies:    El resultat Z com a tool_result
5. Claude respon: Resposta final (o una altra tool_use)
```

**Important:** Claude NO executa les eines. Nomes diu QUINA vol cridar i amb QUINS parametres. Tu (el teu codi) ets qui les executa. Aixo et dona control total.

### Definir Eines com a JSON Schema

Cada eina es defineix amb un diccionari Python que segueix el format JSON Schema:

```python
# Definicio d'una eina per consultar champions
# El format es identic al que vas veure a MCP (S10-S11)
TOOL_QUERY_CHAMPIONS = {
    "name": "query_champions",
    "description": (
        "Consulta la llista de champions amb filtres opcionals. "
        "Retorna nom, rol i estadistiques basiques de cada champion."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "role": {
                "type": "string",
                "description": "Filtra per rol (top, jungle, mid, adc, support)",
                "enum": ["top", "jungle", "mid", "adc", "support"]
            },
            "min_win_rate": {
                "type": "number",
                "description": "Win rate minim (0-100). Ex: 52.5"
            },
            "limit": {
                "type": "integer",
                "description": "Nombre maxim de resultats (per defecte 10)",
                "default": 10
            }
        },
        "required": []
    }
}

TOOL_GET_STATS = {
    "name": "get_champion_stats",
    "description": (
        "Obte estadistiques detallades d'un champion especific: "
        "win rate, pick rate, ban rate, KDA promig, gold per minut."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "champion_id": {
                "type": "string",
                "description": "Identificador del champion (ex: 'ahri', 'jinx')"
            }
        },
        "required": ["champion_id"]
    }
}
```

**Clau:** La `description` es el que Claude llegeix per decidir QUAN usar l'eina. Si la descripcio es vaga, Claude no sabra quan cridar-la. Si es massa amplia, la cridara massa sovint.

### El Bucle Agent en Codi

El bucle es sorprenentment simple — uns 30 linia de Python:

```python
import anthropic

def run_agent_loop(user_question: str, tools: list, system_prompt: str) -> str:
    """Executa el bucle agent fins que Claude dona una resposta final."""

    # Inicialitzem el client de l'API
    client = anthropic.Anthropic()  # Usa ANTHROPIC_API_KEY del entorn

    # Historial de missatges — comenca amb la pregunta de l'usuari
    messages = [{"role": "user", "content": user_question}]

    while True:
        # 1. Enviem a Claude amb les eines disponibles
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            system=system_prompt,
            tools=tools,
            messages=messages
        )

        # 2. Comprovem si Claude vol cridar una eina
        if response.stop_reason == "tool_use":
            # Afegim la resposta de Claude al historial
            messages.append({"role": "assistant", "content": response.content})

            # Processem cada tool_use block
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    # 3. Executem l'eina localment
                    result = execute_tool(block.name, block.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result
                    })

            # 4. Enviem els resultats a Claude
            messages.append({"role": "user", "content": tool_results})

        else:
            # 5. Claude ha acabat — retornem la resposta final
            return response.content[0].text
```

**Observa:** El bucle es exactament el diagrama de dilluns traduit a codi. `while True` es el bucle. `stop_reason == "tool_use"` es la decisio. `execute_tool()` es l'accio. El resultat enviat de tornada es l'observacio.

### System Prompt: El Cervell de l'Agent

El system prompt defineix QUI es l'agent, QUE pot fer, i QUINS limits te:

```python
SYSTEM_PROMPT = """Ets l'Agent Quantitatiu d'EsportsPulse. El teu rol es respondre
preguntes sobre estadistiques de champions de League of Legends.

REGLES:
1. Nomes respons sobre estadistiques i dades objectives
2. Si no tens dades suficients, digues-ho clarament
3. No inventes dades — usa SEMPRE les eines per obtenir informacio
4. Respon en catala
5. Si l'usuari pregunta sobre temes fora del teu ambit, redirigeix educadament

EINES DISPONIBLES:
- query_champions: Per buscar champions amb filtres
- get_champion_stats: Per obtenir detalls d'un champion concret

ESTRATEGIA:
- Per preguntes generals ("quin champion te mes win rate?") → usa query_champions
- Per preguntes concretes ("quines stats te Ahri?") → usa get_champion_stats
- Per comparacions → crida get_champion_stats per cada champion i compara
"""
```

> **Lectura recomanada (no bloquejant):**
> - [Anthropic Tool Use Guide](https://docs.anthropic.com/en/docs/build-with-claude/tool-use) — Referencia oficial
> - [Tool use examples](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/overview#example-tool-use-flow) — Exemples pas a pas

---

## Activitat

### 1. Preparar les Dades de Prova (10 min)

Crea un fitxer amb dades simulades que faran de "base de dades":

```python
# fitxer: ai-python/src/agents/champion_data.py
# Dades simulades de champions per a l'Agent Quantitatiu
# En un sistema real, aixo vindria de la BD o de l'API de Riot

CHAMPIONS_DB = [
    {
        "id": "ahri",
        "name": "Ahri",
        "role": "mid",
        "win_rate": 52.3,
        "pick_rate": 8.1,
        "ban_rate": 5.2,
        "kda": 2.8,
        "gold_per_min": 412
    },
    {
        "id": "jinx",
        "name": "Jinx",
        "role": "adc",
        "win_rate": 51.8,
        "pick_rate": 12.4,
        "ban_rate": 3.1,
        "kda": 2.5,
        "gold_per_min": 438
    },
    {
        "id": "thresh",
        "name": "Thresh",
        "role": "support",
        "win_rate": 49.7,
        "pick_rate": 10.2,
        "ban_rate": 7.8,
        "kda": 3.1,
        "gold_per_min": 285
    },
    {
        "id": "darius",
        "name": "Darius",
        "role": "top",
        "win_rate": 50.9,
        "pick_rate": 6.5,
        "ban_rate": 12.3,
        "kda": 1.9,
        "gold_per_min": 405
    },
    {
        "id": "leesin",
        "name": "Lee Sin",
        "role": "jungle",
        "win_rate": 48.2,
        "pick_rate": 14.7,
        "ban_rate": 4.5,
        "kda": 2.4,
        "gold_per_min": 378
    },
    {
        "id": "lux",
        "name": "Lux",
        "role": "mid",
        "win_rate": 53.1,
        "pick_rate": 9.8,
        "ban_rate": 2.1,
        "kda": 2.6,
        "gold_per_min": 395
    },
    {
        "id": "kaisa",
        "name": "Kai'Sa",
        "role": "adc",
        "win_rate": 50.4,
        "pick_rate": 15.2,
        "ban_rate": 8.9,
        "kda": 2.7,
        "gold_per_min": 445
    },
    {
        "id": "leona",
        "name": "Leona",
        "role": "support",
        "win_rate": 51.5,
        "pick_rate": 7.3,
        "ban_rate": 4.2,
        "kda": 2.2,
        "gold_per_min": 265
    }
]


def query_champions(role: str = None, min_win_rate: float = None,
                    limit: int = 10) -> list:
    """Filtra champions segons els criteris donats.

    Args:
        role: Filtra per rol (top, jungle, mid, adc, support)
        min_win_rate: Win rate minim (0-100)
        limit: Nombre maxim de resultats

    Returns:
        Llista de champions que compleixen els filtres
    """
    results = CHAMPIONS_DB

    # Aplica filtre per rol si s'ha especificat
    if role:
        results = [c for c in results if c["role"] == role]

    # Aplica filtre per win rate minim
    if min_win_rate is not None:
        results = [c for c in results if c["win_rate"] >= min_win_rate]

    # Ordena per win rate descendent i limita resultats
    results = sorted(results, key=lambda c: c["win_rate"], reverse=True)
    return results[:limit]


def get_champion_stats(champion_id: str) -> dict | None:
    """Obte les estadistiques detallades d'un champion.

    Args:
        champion_id: Identificador del champion (ex: 'ahri')

    Returns:
        Diccionari amb totes les stats, o None si no existeix
    """
    for champion in CHAMPIONS_DB:
        if champion["id"] == champion_id:
            return champion
    return None
```

### 2. Implementar l'Agent Quantitatiu (40 min)

Crea l'agent complet:

```python
# fitxer: ai-python/src/agents/quantitative_agent.py
# Agent Quantitatiu: respon preguntes sobre estadistiques de champions
# Usa l'API de Claude amb tool use — sense frameworks

import json
import anthropic
from champion_data import query_champions, get_champion_stats

# --- Definicio d'eines (el que Claude "veu") ---

TOOLS = [
    {
        "name": "query_champions",
        "description": (
            "Consulta la llista de champions amb filtres opcionals. "
            "Retorna nom, rol i estadistiques basiques. "
            "Usa-la per preguntes generals o per trobar champions amb certs criteris."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "role": {
                    "type": "string",
                    "description": "Filtra per rol",
                    "enum": ["top", "jungle", "mid", "adc", "support"]
                },
                "min_win_rate": {
                    "type": "number",
                    "description": "Win rate minim (0-100)"
                },
                "limit": {
                    "type": "integer",
                    "description": "Nombre maxim de resultats (defecte 10)"
                }
            },
            "required": []
        }
    },
    {
        "name": "get_champion_stats",
        "description": (
            "Obte estadistiques detallades d'un champion especific: "
            "win rate, pick rate, ban rate, KDA, gold per minut. "
            "Usa-la quan l'usuari pregunta per un champion concret."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "champion_id": {
                    "type": "string",
                    "description": "ID del champion en minuscules (ex: 'ahri', 'jinx')"
                }
            },
            "required": ["champion_id"]
        }
    }
]

# --- System prompt: defineix el comportament de l'agent ---

SYSTEM_PROMPT = """Ets l'Agent Quantitatiu d'EsportsPulse. El teu rol es respondre
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
- Comparacions → get_champion_stats per cada champion"""


def execute_tool(tool_name: str, tool_input: dict) -> str:
    """Executa una eina localment i retorna el resultat com a string JSON.

    Aquesta funcio es el "pont" entre el que Claude demana i les funcions reals.
    Es on tu tens el control — pots afegir logging, validacio, limits, etc.
    """
    if tool_name == "query_champions":
        # Crida la funcio real amb els parametres que Claude ha triat
        result = query_champions(
            role=tool_input.get("role"),
            min_win_rate=tool_input.get("min_win_rate"),
            limit=tool_input.get("limit", 10)
        )
        return json.dumps(result, ensure_ascii=False)

    elif tool_name == "get_champion_stats":
        result = get_champion_stats(tool_input["champion_id"])
        if result is None:
            return json.dumps({"error": f"Champion '{tool_input['champion_id']}' no trobat"})
        return json.dumps(result, ensure_ascii=False)

    else:
        # Guardrail: si Claude intenta cridar una eina que no existeix
        return json.dumps({"error": f"Eina desconeguda: {tool_name}"})


def run_agent(user_question: str, verbose: bool = True) -> str:
    """Executa el bucle agent complet.

    Args:
        user_question: La pregunta de l'usuari
        verbose: Si True, mostra cada pas del bucle (util per depurar)

    Returns:
        La resposta final de l'agent
    """
    # Inicialitza el client — necessita ANTHROPIC_API_KEY al entorn
    client = anthropic.Anthropic()

    # Historial de la conversa
    messages = [{"role": "user", "content": user_question}]

    # Comptador de passos (guardrail contra bucles infinits)
    max_steps = 10
    step = 0

    while step < max_steps:
        step += 1

        if verbose:
            print(f"\n--- Pas {step} ---")

        # Envia a Claude amb les eines disponibles
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            system=SYSTEM_PROMPT,
            tools=TOOLS,
            messages=messages
        )

        if verbose:
            print(f"Stop reason: {response.stop_reason}")
            # Mostra quant costa cada crida (tokens)
            print(f"Tokens — input: {response.usage.input_tokens}, "
                  f"output: {response.usage.output_tokens}")

        # Si Claude vol usar eines
        if response.stop_reason == "tool_use":
            # Afegim la resposta completa al historial
            messages.append({"role": "assistant", "content": response.content})

            # Processem cada crida a eina
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    if verbose:
                        print(f"Eina: {block.name}({json.dumps(block.input)})")

                    # Executem l'eina
                    result = execute_tool(block.name, block.input)

                    if verbose:
                        print(f"Resultat: {result[:200]}...")  # Truncat per llegibilitat

                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result
                    })

            # Enviem els resultats de les eines a Claude
            messages.append({"role": "user", "content": tool_results})

        else:
            # Claude ha decidit que ja pot respondre — fi del bucle
            final_text = ""
            for block in response.content:
                if hasattr(block, "text"):
                    final_text += block.text
            return final_text

    # Si arriba aqui, ha superat el maxim de passos (guardrail)
    return "ERROR: L'agent ha superat el maxim de passos permesos."


# --- Punt d'entrada ---
if __name__ == "__main__":
    # Preguntes de prova per validar l'agent
    preguntes = [
        "Quin champion te el win rate mes alt?",
        "Dona'm les stats de Jinx",
        "Compara Ahri i Lux — quina es millor al mid?",
        "Quins supports tenen mes del 50% de win rate?",
    ]

    for pregunta in preguntes:
        print(f"\n{'='*60}")
        print(f"PREGUNTA: {pregunta}")
        print('='*60)
        resposta = run_agent(pregunta)
        print(f"\nRESPOSTA:\n{resposta}")
```

### 3. Provar l'Agent (15 min)

Executa l'agent:

```bash
# Assegura't que tens la clau de l'API configurada
export ANTHROPIC_API_KEY="la-teva-clau"

cd esportspulse-engine/ai-python/src/agents
python3 quantitative_agent.py
```

**Observa el mode `verbose`:** Per cada pregunta veuras:
- Quantes vegades Claude crida eines
- Quins parametres tria
- Quants tokens gasta per pas
- Com decideix que ja pot respondre

**Preguntes de prova addicionals:**

```python
# Preguntes que haurien de funcionar be
"Quin es el champion mes bannejat?"
"Quins champions de jungle hi ha?"

# Preguntes que haurien d'activar guardrails
"Quin es el millor restaurant de Barcelona?"  # Fora d'ambit
"Inventa estadistiques per un champion nou"   # Hauria de negar-se
```

### 4. Mesurar Costos i Eficiencia (10 min)

Afegeix un resum de costos al final de `run_agent`:

```python
# Afegeix aixo dins el bucle, despres de cada response
# per acumular costos totals
total_input_tokens = 0
total_output_tokens = 0

# Dins el while:
total_input_tokens += response.usage.input_tokens
total_output_tokens += response.usage.output_tokens

# Al final, abans del return:
if verbose:
    # Preus aproximats per Claude Sonnet (maig 2025)
    cost_input = (total_input_tokens / 1_000_000) * 3.0
    cost_output = (total_output_tokens / 1_000_000) * 15.0
    print(f"\nCost total: ${cost_input + cost_output:.6f}")
    print(f"  Input:  {total_input_tokens} tokens (${cost_input:.6f})")
    print(f"  Output: {total_output_tokens} tokens (${cost_output:.6f})")
```

---

## Checklist de Lliurament

- [ ] `champion_data.py` funciona: `query_champions()` i `get_champion_stats()` retornen dades correctes
- [ ] `quantitative_agent.py` executa el bucle agent complet amb tool use
- [ ] L'agent respon correctament a les 4 preguntes de prova
- [ ] El mode verbose mostra les crides a eines, parametres i tokens per pas
- [ ] L'agent gestiona preguntes fora d'ambit sense inventar dades
- [ ] Has mesurat el cost per consulta (en tokens i dolars aproximats)
- [ ] Commit: `feat(agents): implement quantitative agent with tool use`
