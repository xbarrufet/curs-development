# Setmana 10 — Dimarts: Connecting to Claude API — System Prompts, Few-Shot, JSON Output

## Objectiu del Dia

Connectar el projecte EsportsPulse amb l'API de Claude (Anthropic) per obtenir analisis de campions en format JSON estructurat. Al final del dia tindras un script que crida Claude, forca output estructurat amb `tool_use`, i valida la resposta amb els models Pydantic de dilluns.

---

## Teoria

### L'API de Claude: Estructura Basica

L'API de Claude funciona amb el patro de missatges: envies una llista de missatges amb rols (`user`, `assistant`) i un `system` prompt opcional, i Claude respon.

```python
# Instal·la el SDK d'Anthropic
# pip install anthropic

from anthropic import Anthropic

# Crea el client — necessites una API key
# La clau es llegeix automaticament de la variable d'entorn ANTHROPIC_API_KEY
client = Anthropic()

# Primera crida: text lliure (per veure el problema)
response = client.messages.create(
    model="claude-sonnet-4-20250514",  # Model a fer servir
    max_tokens=1024,                    # Limit de tokens de resposta
    system="Ets un analista expert de League of Legends.",  # System prompt
    messages=[
        {
            "role": "user",
            "content": "Analitza el campio Jinx: punts forts, counters, tier actual."
        }
    ]
)

# La resposta es text lliure — impossible de parsejar de forma fiable
print(response.content[0].text)
# "Jinx es una ADC molt forta al meta actual. Els seus punts forts inclouen..."
# Com extreim champion_id? I el win_rate exacte? Impossible sense regex fragils.
```

### El Problema del Text Lliure

La resposta anterior es util per a humans pero inutilitzable per a codi:
- No podem garantir el format
- Cada crida pot retornar l'informacio en ordre diferent
- No hi ha garantia que inclogui tots els camps que necessitem
- Parsejar text natural es fragil i propens a errors

**La solucio: `tool_use` (Function Calling)**

### tool_use: Forcar Output Estructurat

El mecanisme de `tool_use` permet definir "eines" que el model pot cridar. L'enginy: no cal que l'eina existeixi realment — la fem servir per forcar el model a retornar JSON amb un esquema concret.

```python
import json
from anthropic import Anthropic
from models.champion_analysis import ChampionAnalysis

# Pas 1: Generem l'esquema JSON des del model Pydantic de dilluns
# Aixo garanteix que l'esquema esta sempre sincronitzat amb el model
schema = ChampionAnalysis.model_json_schema()

# Pas 2: Definim la "tool" amb l'esquema Pydantic
# El model pensara que ha de cridar aquesta funcio, i retornara JSON valid
tool_definition = {
    "name": "submit_champion_analysis",  # Nom de la funcio (el model el veura)
    "description": (
        "Envia l'analisi estructurada d'un campio de League of Legends. "
        "Has d'omplir TOTS els camps amb dades precises i actuals."
    ),
    "input_schema": schema  # L'esquema JSON generat per Pydantic
}

client = Anthropic()

# Pas 3: Fem la crida forcant l'us de la tool
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=2048,
    system=(
        "Ets un analista expert de League of Legends amb dades actualitzades. "
        "Quan analitzis un campio, proporciona dades precises i actuals. "
        "Respon SEMPRE en catala."
    ),
    tools=[tool_definition],        # Llista de tools disponibles
    tool_choice={
        "type": "tool",                    # Forca l'us d'una tool concreta
        "name": "submit_champion_analysis" # Nom de la tool a forcar
    },
    messages=[
        {
            "role": "user",
            "content": "Analitza el campio Jinx (champion_id: 222) al patch actual."
        }
    ]
)

# Pas 4: Extraiem el bloc tool_use de la resposta
# La resposta conte un ContentBlock de tipus tool_use amb l'input JSON
tool_use_block = next(
    block for block in response.content
    if block.type == "tool_use"
)

# tool_use_block.input es un dict amb l'estructura del nostre model
raw_data = tool_use_block.input
print(json.dumps(raw_data, indent=2, ensure_ascii=False))
```

### Validacio amb Pydantic

El JSON que retorna Claude pot contenir errors subtils (valors fora de rang, camps amb format incorrecte). Per aixo validem amb Pydantic:

```python
from pydantic import ValidationError

# Pas 5: Validem la resposta del LLM amb el model Pydantic
# Si Claude ha retornat dades incorrectes, Pydantic ho detectara
try:
    analysis = ChampionAnalysis(**raw_data)
    print(f"Validacio OK: {analysis.name} — Tier {analysis.patch_tier}")
    print(f"Win rate: {analysis.champion_trend.current_win_rate}%")
except ValidationError as e:
    # Si la validacio falla, tenim els errors detallats
    print(f"La resposta del LLM no es valida:")
    for error in e.errors():
        print(f"  Camp: {error['loc']} — Error: {error['msg']}")
```

### Few-Shot Prompting: Millorar la Qualitat

Few-shot vol dir donar exemples al model perque entengui exactament que volem. La diferencia de qualitat es dramatica:

```python
# Few-shot: incloem exemples dins del system prompt
# Aixo millora MOLT la qualitat i consistencia de les respostes
system_prompt = """Ets un analista expert de League of Legends. Analitza campions amb dades precises.

Exemple d'analisi correcta per Ahri:
- champion_id: 103
- role: MID
- patch_tier: A
- strengths: ["Mobilitat amb R", "CC fiable amb E", "Poke amb Q"]
- counters: ["Zed", "Fizz", "Kassadin"]
- summary: "Ahri es una maga versatil amb excel·lent mobilitat i pick potential."

Exemple d'analisi correcta per Thresh:
- champion_id: 412
- role: SUPPORT
- patch_tier: B
- strengths: ["Engage amb Q", "Peel amb E i R", "Llanternes salven vides"]
- counters: ["Morgana", "Braum", "Zyra"]
- summary: "Thresh es un suport amb alt skill ceiling que recompensa jugadors mecanics."

Segueix EXACTAMENT aquest nivell de detall i format. Respon en catala."""

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=2048,
    system=system_prompt,          # System prompt amb exemples few-shot
    temperature=0.2,               # Temperatura baixa = respostes mes consistents
    tools=[tool_definition],
    tool_choice={"type": "tool", "name": "submit_champion_analysis"},
    messages=[
        {"role": "user", "content": "Analitza Jinx (champion_id: 222)."}
    ]
)
```

### Temperatura: Controlar la Creativitat

| Temperatura | Comportament                            | Quan usar-la                      |
|------------|------------------------------------------|-----------------------------------|
| 0.0        | Determinista, sempre la mateixa resposta | Tests, dades molt precises        |
| 0.2        | Gairebe determinista, lleugeres variacions | Analisis, dades estructurades   |
| 0.7        | Creatiu, variacions significatives       | Text narratiu, descripcions       |
| 1.0        | Molt creatiu, impredictible              | Brainstorming, generacio d'idees  |

Per a EsportsPulse, farem servir `temperature=0.2` — volem dades precises i consistents, no creativitat.

### Tokens: Com Es Compta (i Es Paga) el que Envies a Claude

Cada crida a l'API de Claude es mesura en **tokens**. Un token no es exactament una paraula — es un fragment de text que el model processa internament:

```
Exemples de tokenitzacio:
  "Hola"          → 1 token
  "League of Legends" → 3 tokens
  "ChampionRecord" → 2 tokens  (Camel → 2 parts)
  "{"name": "jinx"}" → ~7 tokens (JSON es car en tokens!)

Regla practica:
  Catala/Castellà: ~1 token per cada 3-4 caràcters
  Angles: ~1 token per cada 4 caràcters (~0.75 paraules)
  JSON/codi: mes tokens del que sembla (claus, cometes, estructura)
```

Cada crida te dues parts que es cobren per separat:

| Part | Que inclou | Preu (Sonnet) |
|------|-----------|---------------|
| **Input tokens** | System prompt + tools + missatges anteriors + pregunta | $3 / milió tokens |
| **Output tokens** | La resposta generada per Claude | $15 / milió tokens |

> **Important:** Els output tokens son **5x mes cars** que els input tokens. Reduir la longitud de la resposta estalvia mes que reduir el prompt.

### Context Window: El Limit de Memoria

Cada model te un **context window** — el maxim de tokens que pot "veure" en una crida (input + output junts):

```
┌─────────────────────────────────────────────────────────────┐
│                    CONTEXT WINDOW                            │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  System prompt (500 tokens)                          │   │
│  │  + Definicio de tools (300 tokens)                   │   │
│  │  + Historial de missatges (2000 tokens)              │   │
│  │  + Pregunta actual (100 tokens)                      │   │
│  │  ─────────────────────────────────                   │   │
│  │  = 2900 tokens d'input                               │   │
│  │                                                      │   │
│  │  + Resposta del model (500 tokens d'output)          │   │
│  │  ─────────────────────────────────                   │   │
│  │  = 3400 tokens TOTALS (dins del limit)               │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  Limit Haiku:   200K tokens                                 │
│  Limit Sonnet:  200K tokens                                 │
│  Limit Opus:    200K tokens                                 │
└─────────────────────────────────────────────────────────────┘
```

Si superes el context window, la crida falla. Aixo es critic quan:
- Envies molts exemples few-shot (ocupen espai)
- L'historial de conversa creix (agents amb molts passos)
- Els resultats de tools son grans (respostes JSON extenses)

### Triar el Model: Haiku vs Sonnet vs Opus

No tot requereix el model mes potent. Triar be el model pot reduir costos 10-50x:

| Model | Velocitat | Qualitat | Preu (input/output per 1M tokens) | Quan usar-lo |
|-------|-----------|---------|----------------------------------|--------------|
| **Haiku 4.5** | Molt rapid | Bona per tasques simples | $0.80 / $4 | Classificar, extreure dades simples, validar formats |
| **Sonnet** | Rapid | Alta | $3 / $15 | Analisi, generacio de text, tool use — **el default** |
| **Opus** | Mes lent | Molt alta | $15 / $75 | Raonament complex, decisions critiques, codi complex |

```python
# Estrategia: usar Haiku per a tasques simples, Sonnet per a la resta

# Tasca simple: classificar un campio per tier → Haiku
response = client.messages.create(
    model="claude-haiku-4-5-20251001",  # 10x mes barat que Sonnet
    max_tokens=50,
    messages=[{"role": "user", "content": f"Classifica {name} en S/A/B/C tier. Respon NOMES amb la lletra."}]
)

# Tasca complexa: analisi detallada → Sonnet
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=2048,
    tools=[tool_definition],
    # ...
)
```

> **Regla practica per a EsportsPulse:** Usa **Sonnet** per defecte. Canvia a **Haiku** per a validacions simples o classificacions. Reserva **Opus** per a decisions on la qualitat es critica (i el cost no importa).

---

### Anatomia Completa d'una Resposta tool_use

```python
# La resposta de l'API te aquesta estructura:
# response.content = [
#     ContentBlock(
#         type="tool_use",
#         id="toolu_01ABC...",        # ID unic de la crida
#         name="submit_champion_analysis",  # Nom de la tool cridada
#         input={                     # El JSON estructurat que volem!
#             "champion_id": 222,
#             "name": "Jinx",
#             "role": "ADC",
#             ...
#         }
#     )
# ]
#
# response.stop_reason = "tool_use"   # Indica que el model ha cridat una tool
# response.usage.input_tokens = 1523  # Tokens consumits (entrada)
# response.usage.output_tokens = 312  # Tokens consumits (sortida)

# Preu aproximat per crida amb claude-sonnet-4-20250514:
# ~1500 input tokens + ~300 output tokens = ~$0.006 per crida
# Amb 100 campions: ~$0.60 total — molt assequible
```

---

## Activitat

### Pas 1: Configura la Clau API (5 min)

```bash
# Configura la clau API com a variable d'entorn
# Obtingues la clau a: https://console.anthropic.com/settings/keys
export ANTHROPIC_API_KEY="sk-ant-api03-..."

# Instal·la el SDK
pip install anthropic
```

### Pas 2: Script Basica — Text Lliure (15 min)

Crea `ai-python/src/llm/test_basic_call.py`:

```python
# test_basic_call.py — Primera crida a Claude sense estructura
# Objectiu: veure que el text lliure es impossible de parsejar
from anthropic import Anthropic

client = Anthropic()

# Crida basica sense tool_use — retorna text lliure
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=512,
    messages=[
        {"role": "user", "content": "Analitza breument Jinx de LoL."}
    ]
)

# Imprimeix la resposta — observa que es text lliure, no JSON
print("=== Resposta text lliure ===")
print(response.content[0].text)
print(f"\nTokens usats: {response.usage.input_tokens} in / {response.usage.output_tokens} out")
```

### Pas 3: Script amb tool_use — Output Estructurat (30 min)

Crea `ai-python/src/llm/analyze_champion.py`:

```python
# analyze_champion.py — Crida a Claude amb tool_use per obtenir JSON estructurat
# Objectiu: obtenir una ChampionAnalysis valida des de Claude
import json
from anthropic import Anthropic
from models.champion_analysis import ChampionAnalysis
from pydantic import ValidationError

def analyze_champion(champion_id: int, champion_name: str) -> ChampionAnalysis:
    """
    Crida Claude per obtenir una analisi estructurada d'un campio.
    Retorna un objecte ChampionAnalysis validat per Pydantic.
    Llanca ValueError si la resposta del LLM no es valida.
    """
    client = Anthropic()

    # Definim la tool amb l'esquema generat per Pydantic
    tool_def = {
        "name": "submit_champion_analysis",
        "description": "Envia l'analisi estructurada d'un campio de LoL.",
        "input_schema": ChampionAnalysis.model_json_schema()
    }

    # System prompt amb exemples few-shot per millorar la qualitat
    system = (
        "Ets un analista expert de League of Legends. "
        "Proporciona dades precises sobre campions. Respon en catala.\n\n"
        "Exemple: Ahri (103) — MID, Tier A, win_rate 51.2%, "
        "strengths: mobilitat, CC, poke. counters: Zed, Fizz."
    )

    # Fem la crida amb tool_choice forcat i temperatura baixa
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=2048,
        system=system,
        temperature=0.2,
        tools=[tool_def],
        tool_choice={"type": "tool", "name": "submit_champion_analysis"},
        messages=[
            {
                "role": "user",
                "content": f"Analitza {champion_name} (champion_id: {champion_id})."
            }
        ]
    )

    # Extraiem el bloc tool_use de la resposta
    tool_block = next(
        b for b in response.content if b.type == "tool_use"
    )

    # Validem amb Pydantic — si falla, llancem ValueError
    try:
        analysis = ChampionAnalysis(**tool_block.input)
        return analysis
    except ValidationError as e:
        raise ValueError(f"Resposta LLM invalida: {e}")

# Execucio directa del script
if __name__ == "__main__":
    result = analyze_champion(222, "Jinx")
    print(f"Campio: {result.name}")
    print(f"Rol: {result.role}")
    print(f"Tier: {result.patch_tier}")
    print(f"Win rate: {result.champion_trend.current_win_rate}%")
    print(f"Resum: {result.summary}")
    print(f"\nJSON complet:")
    print(result.model_dump_json(indent=2))
```

### Pas 4: Experimenta amb Few-Shot (15 min)

Modifica el `system` prompt per afegir 2-3 exemples complets i observa com millora la qualitat. Compara:
1. Sense exemples: respostes generiques
2. Amb 1 exemple: respostes mes consistents
3. Amb 2-3 exemples: respostes d'alta qualitat i format consistent

---

## Checklist de Lliurament

- [ ] Clau API configurada com a variable d'entorn `ANTHROPIC_API_KEY`
- [ ] SDK `anthropic` instal·lat i funcionant
- [ ] Script de crida basica (`test_basic_call.py`) executa correctament
- [ ] Script amb `tool_use` (`analyze_champion.py`) retorna JSON estructurat
- [ ] La resposta del LLM passa la validacio de Pydantic sense errors
- [ ] S'ha provat few-shot i s'ha observat la millora de qualitat
- [ ] Temperatura configurada a 0.2 per respostes consistents
- [ ] S'entenen els costos aproximats per crida (tokens in/out)
