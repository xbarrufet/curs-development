# Setmana 8 - Teoria: Python com a Co-Protagonista, Pydantic v2, LLMs i MCP

## 1. Python com a Segon Llenguatge: Per Què Java + Python

### La realitat del mercat

A moltes empreses, els equips de backend escriuen APIs en Java/Spring. Però quan arriba el moment de connectar un LLM, processar dades o construir un pipeline d'anàlisi, el llenguatge canvia a Python. No per capritx — per ecosistema.

El SDK d'Anthropic, el d'OpenAI, FastAPI, pandas, scikit-learn, langchain, la majoria de tooling IA... tot és Python-first. Si només saps Java, dependràs sempre d'algú altre per la part d'IA. Si només saps Python, no podràs tocar el backend core.

### Què ha passat fins ara

De S1 a S6, has fet exercicis "mirall" en Python: el mateix concepte implementat en ambdós llenguatges. Has après la sintaxi, els tests amb pytest, els patrons bàsics. A S7, Python va fer de consumidor: una CLI amb typer que cridava l'API Java.

Ara Python puja de categoria. Deixa de ser el mirall i passa a tenir les seves pròpies responsabilitats dins GamePulse.

### Qui fa què a GamePulse

```
┌──────────────────────────────────────────────────────────┐
│                     GamePulse                            │
│                                                          │
│  ┌─────────────────────┐    ┌──────────────────────────┐ │
│  │   Java / Spring     │    │   Python / FastAPI       │ │
│  │                     │    │                          │ │
│  │ - REST API (CRUD)   │    │ - Anàlisi amb LLMs      │ │
│  │ - Persistència (JPA)│    │ - Validació (Pydantic)  │ │
│  │ - Lògica de negoci  │◄──┤ - Endpoints d'anàlisi   │ │
│  │ - Transaccions      │    │ - MCP servers           │ │
│  │ - Tests (JUnit)     │    │ - Tests (pytest)        │ │
│  └─────────────────────┘    └──────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

| Àrea                    | Java + Spring                         | Python + FastAPI                 |
|:------------------------|:--------------------------------------|:---------------------------------|
| API REST (CRUD)         | ✅ Spring Boot, JPA, H2              | ❌ No cal duplicar              |
| Validació de dades      | ✅ Bean Validation                   | ✅ Pydantic v2                  |
| Connexió a LLMs         | ❌ SDKs limitats                     | ✅ anthropic, openai            |
| Anàlisi de dades        | ❌ Verbose, sense ecosistema          | ✅ pandas, numpy                |
| Concurrència            | ✅ Virtual Threads (S3)              | ✅ asyncio + FastAPI            |
| Tests                   | ✅ JUnit 5, Mockito (S6)            | ✅ pytest, monkeypatch (S6)    |
| Documentació automàtica | ✅ SpringDoc/Swagger (S7)            | ✅ FastAPI /docs                |

La combinació és complementària, no redundant. Cada llenguatge fa el que fa millor.

---

## 2. Pydantic v2: Models amb Validació

### Què és Pydantic?

Pydantic és una llibreria de Python que fa validació de dades usant type hints. Definiries un model, i Pydantic s'encarrega de:

1. Validar que les dades compleixen els tipus.
2. Convertir tipus automàticament (coercion).
3. Generar schemas JSON.
4. Serialitzar/deserialitzar a/des de JSON.

### El primer model: GameAnalysis

```python
from pydantic import BaseModel, Field, field_validator
from typing import Literal
from datetime import date

class PlayerTrend(BaseModel):
    current_players: int = Field(ge=0, description="Jugadors actius ara")
    peak_players: int = Field(ge=0, description="Pic de jugadors històric")
    trend_direction: Literal["up", "down", "stable"] = Field(
        description="Direcció de la tendència"
    )

class GameAnalysis(BaseModel):
    game_id: str = Field(description="Identificador del joc", examples=["APP-123"])
    title: str = Field(min_length=1, description="Nom del joc")
    sentiment: Literal["positive", "negative", "mixed"] = Field(
        description="Sentiment general dels jugadors"
    )
    player_trend: PlayerTrend
    summary: str = Field(min_length=10, description="Resum de l'anàlisi")

    @field_validator("game_id")
    @classmethod
    def game_id_must_start_with_app(cls, v: str) -> str:
        if not v.startswith("APP-"):
            raise ValueError("game_id ha de començar per 'APP-'")
        return v

class PatchSummary(BaseModel):
    version: str = Field(description="Versió del patch", examples=["1.2.3"])
    date: date = Field(description="Data del patch")
    changes: list[str] = Field(min_length=1, description="Llista de canvis")
    impact_score: float = Field(ge=0, le=10, description="Impacte del 0 al 10")
```

### Validació en acció

```python
# Dades vàlides
analysis = GameAnalysis(
    game_id="APP-42", title="League of Legends", sentiment="positive",
    player_trend=PlayerTrend(current_players=5_000_000, peak_players=8_000_000, trend_direction="up"),
    summary="Joc free-to-play amb tendència positiva i comunitat activa."
)
print(analysis.model_dump_json(indent=2))

# Dades invàlides → ValidationError (mostra TOTS els errors de cop)
try:
    bad = GameAnalysis(game_id="WRONG-1", title="", sentiment="meh",
        player_trend=PlayerTrend(current_players=-1, peak_players=0, trend_direction="up"),
        summary="curt")
except ValidationError as e:
    print(e)
```

### Generar JSON Schema

```python
import json
schema = GameAnalysis.model_json_schema()
print(json.dumps(schema, indent=2))
```

Això genera un schema JSON estàndard que pots donar a un LLM, a un frontend, o a qualsevol eina que entengui JSON Schema. Es el pont entre Pydantic i la resta del món.

### Comparativa: Java records (S2) vs Pydantic

**Java (S2 + S7):**
```java
public record GameAnalysis(@NotBlank String gameId, @NotBlank String title,
    @NotNull Sentiment sentiment, @Valid PlayerTrend playerTrend, @Size(min=10) String summary) {
    public GameAnalysis { if (!gameId.startsWith("APP-")) throw new IllegalArgumentException("..."); }
}
public enum Sentiment { POSITIVE, NEGATIVE, MIXED }
```

**Pydantic (Python):**
```python
class GameAnalysis(BaseModel):
    game_id: str = Field(pattern=r"^APP-\d+$")
    title: str = Field(min_length=1)
    sentiment: Literal["positive", "negative", "mixed"]
    player_trend: PlayerTrend
    summary: str = Field(min_length=10)
```

| Concepte              | Java record + Bean Validation      | Pydantic v2                      |
|:----------------------|:-----------------------------------|:---------------------------------|
| Declaració            | `record` + annotations             | `BaseModel` + `Field()`         |
| Validació             | `@Valid`, `@NotBlank`, `@Min`      | `Field(ge=, le=, min_length=)`  |
| Custom validation     | Compact constructor                | `@field_validator`               |
| Immutabilitat         | ✅ Per defecte                     | ✅ Amb `model_config = {"frozen": True}` |
| JSON schema           | ❌ Necessita lib addicional        | ✅ `model_json_schema()` built-in |
| Serialització JSON    | Jackson (configurable)             | `.model_dump_json()` built-in    |
| Errors                | Un a la vegada (per defecte)       | Tots de cop                      |

**Takeaway:** Pydantic fa en 3 línies el que Java necessita en 15, però el concepte és idèntic: definir un contracte de dades i validar-lo.

---

## 3. LLMs com a Funcions: Input Text, Output Estructurat

### El model mental

Un LLM és una funció: `f(prompt) → text`. Li passes text, retorna text. El problema: "text" és massa ambigu. Amb un schema, es transforma: `f(prompt, schema) → structured_data`.

### El flux visual

```
Sense schema:                           Amb schema:
┌──────────┐     ┌─────┐               ┌──────────┐     ┌─────┐     ┌──────────┐
│  Prompt   │────►│ LLM │────► text     │  Prompt   │────►│ LLM │────►│  JSON    │
│  (text)   │     └─────┘   lliure      │  (text)   │     └─────┘     │  validat │
└──────────┘                            │ + schema  │                  └──────────┘
                                        └──────────┘
"El LoL té molts                        { "game_id": "APP-1",
 jugadors i és                            "sentiment": "positive",
 bastant popular..."                      "player_trend": { ... } }
```

### Per què importa l'output estructurat?

Parsing (camps concrets), validació (Pydantic verifica el contracte), type safety (el codi sap què espera), composabilitat (sortida d'un LLM com a entrada d'un altre servei), i testing (verificar l'estructura). Sense estructura, cada resposta és una sorpresa. Amb estructura, és una funció predictible.

---

## 4. La API de Claude/OpenAI: Messages, Rols, Paràmetres

### Estructura de la Messages API

Totes les APIs de LLM modernes segueixen el patró de "conversa amb rols":

```python
messages = [
    {"role": "system", "content": "Ets un analista de videojocs expert."},
    {"role": "user", "content": "Analitza League of Legends."},
    {"role": "assistant", "content": "D'acord, analitzaré el LoL..."},
    {"role": "user", "content": "Ara fes-ho per Valorant."},
]
```

**Rols:**
- **system:** Instruccions globals per al model. Defineix el comportament, el format, les restriccions.
- **user:** El que tu (o el teu programa) li dius.
- **assistant:** El que el model ha respost (en converses multi-torn, inclou respostes anteriors).

### Paràmetres clau

```python
import anthropic

client = anthropic.Anthropic()  # Usa ANTHROPIC_API_KEY del entorn

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    temperature=0.2,
    system="Ets un analista de videojocs. Respon sempre en JSON.",
    messages=[
        {"role": "user", "content": "Analitza el joc League of Legends."}
    ]
)

print(response.content[0].text)
```

| Paràmetre      | Què fa                                    | Valor típic per GamePulse |
|:---------------|:------------------------------------------|:--------------------------|
| `model`        | Quin model usar                           | `claude-sonnet-4-20250514` |
| `max_tokens`   | Màxim de tokens a la resposta             | 1024                      |
| `temperature`  | Creativitat (0 = determinista, 1 = creatiu) | 0.2 (volem precisió)    |
| `system`       | Instruccions globals                       | Rol d'analista            |

### Gestió d'errors

Gestiona tres tipus d'error: `RateLimitError` (massa requests -- espera i reintenta), `APIStatusError` (error HTTP -- logeja status i message), `APIConnectionError` (xarxa -- reintenta o falla).

**Cost:** Cada crida costa tokens. Per a tests, usa sempre mocks (S6). Un prompt de GameAnalysis amb few-shot costa aprox. 500-800 tokens d'input i 200-400 de output.

---

## 5. Output Estructurat: tool_use i function_calling

### Com forçar JSON del LLM

El LLM, per defecte, retorna text lliure. Hi ha dues maneres de forçar estructura:

1. **Claude: tool_use** — defineixes un "tool" amb un schema, i el model "crida" el tool amb dades estructurades.
2. **OpenAI: function_calling** — conceptualment idèntic, diferent API.

El patró és el mateix: dones un schema, el model omplena les dades.

### Exemple complet amb Claude tool_use

```python
import anthropic
from pydantic import BaseModel

# 1. Definir el model Pydantic
class GameAnalysis(BaseModel):
    game_id: str
    title: str
    sentiment: str
    summary: str

# 2. Crear la definició del tool a partir del schema
tool_definition = {
    "name": "submit_game_analysis",
    "description": "Retorna l'anàlisi estructurada d'un videojoc.",
    "input_schema": GameAnalysis.model_json_schema()
}

# 3. Cridar l'API amb el tool
client = anthropic.Anthropic()
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=[tool_definition],
    tool_choice={"type": "tool", "name": "submit_game_analysis"},
    messages=[
        {
            "role": "user",
            "content": "Analitza el joc League of Legends (APP-1) amb 5M jugadors actius i preu 0€."
        }
    ]
)

# 4. Extreure i validar la resposta
tool_use_block = next(
    block for block in response.content if block.type == "tool_use"
)
analysis = GameAnalysis.model_validate(tool_use_block.input)
print(analysis.model_dump_json(indent=2))
```

### El flux de validació

```
┌──────────┐    ┌────────┐    ┌──────────┐    ┌──────────┐
│ Prompt + │───►│  LLM   │───►│  JSON    │───►│ Pydantic │───► GameAnalysis
│ Schema   │    │        │    │  (raw)   │    │ validate │    (objecte vàlid)
└──────────┘    └────────┘    └──────────┘    └──────────┘
                                                   │
                                                   ▼ (si falla)
                                              ValidationError
                                              → retry o error
```

### Quan la validació falla

El LLM no és perfecte. De vegades retorna dades que no compleixen el schema. Pydantic les captura:

```python
try:
    analysis = GameAnalysis.model_validate(tool_use_block.input)
except ValidationError as e:
    # Opcions:
    # 1. Reintentar la crida (amb el missatge d'error al prompt)
    # 2. Retornar un error al client
    # 3. Usar valors per defecte
    print(f"LLM ha retornat dades invàlides: {e}")
```

El bucle de retry és un patró comú: crida el LLM → valida → si falla, torna a cridar amb l'error com a context. Normalment 1-2 retries són suficients.

---

## 6. Few-Shot Prompting

### Què és?

Few-shot prompting és incloure 2-3 exemples complets al prompt perquè el LLM segueixi el patró. El model aprèn del format i l'estil dels exemples.

### Per què funciona?

Els LLMs són excel·lents imitant patrons. Si li dones exemples del que esperes, la resposta s'ajustarà molt millor al teu format.

### Exemple per GamePulse

```python
prompt = """Ets un analista de videojocs. Analitza el joc que et doni l'usuari.

## Exemples

Joc: League of Legends | Jugadors: 5,000,000 | Preu: 0€
Anàlisi: {
  "game_id": "APP-1",
  "title": "League of Legends",
  "sentiment": "positive",
  "summary": "Free-to-play MOBA amb base de jugadors massiva i creixent. Model de monetització basat en skins no afecta el gameplay."
}

Joc: Cyberpunk 2077 | Jugadors: 200,000 | Preu: 59.99€
Anàlisi: {
  "game_id": "APP-42",
  "title": "Cyberpunk 2077",
  "sentiment": "mixed",
  "summary": "RPG amb llançament problemàtic però recuperat amb patches. Base de jugadors estable post-DLC."
}

## Ara analitza:
Joc: {title} | Jugadors: {player_count} | Preu: {price}
"""
```

### Amb i sense exemples

Sense few-shot, el LLM retorna text narratiu ("Elden Ring és un joc d'acció RPG desenvolupat per...") -- impossible de parsejar. Amb 2 exemples, segueix el format exacte i retorna JSON estructurat.

### Regles pràctiques

1. **2-3 exemples** són suficients. Més de 5 rarament millora la qualitat i augmenta el cost.
2. **Exemples diversos:** inclou cas positiu, negatiu i mixt.
3. **Exemples realistes:** usa dades plausibles, no "test123".
4. **Format consistent:** tots els exemples han de seguir exactament el mateix format.

Few-shot és la manera més senzilla de millorar la qualitat sense complicar el codi. Abans de tocar paràmetres o canviar el model, prova afegir exemples.

---

## 7. FastAPI: L'Express de Python

### Per què FastAPI?

FastAPI destaca per quatre raons: Pydantic integrat (validació automàtica), docs auto-generades a `/docs`, async natiu, i type hints per tot. La frase clau: "FastAPI força Pydantic, Flask no força res."

### Anatomia d'un endpoint FastAPI

```python
from fastapi import FastAPI, HTTPException
import httpx

app = FastAPI(title="GamePulse Analysis Service")

@app.get("/analyze/{game_id}", response_model=GameAnalysis)
async def analyze_game(game_id: str):
    # 1. Fetch del joc des de l'API Java (S7)
    async with httpx.AsyncClient() as client:
        resp = await client.get(f"http://localhost:8080/games/{game_id}")
        if resp.status_code == 404:
            raise HTTPException(status_code=404, detail=f"Game {game_id} not found")
        game_data = resp.json()

    # 2. Cridar LLM amb tool_use
    analysis_data = await call_llm_analysis(game_data)

    # 3. Validar amb Pydantic i retornar
    return GameAnalysis.model_validate(analysis_data)

@app.post("/batch-analyze", response_model=list[GameAnalysis])
async def batch_analyze(game_ids: list[str]):
    results = []
    for gid in game_ids:
        analysis = await analyze_game(gid)
        results.append(analysis)
    return results

@app.get("/health")
def health():
    return {"status": "ok"}
```

### Comparativa Spring Boot (S7) vs FastAPI

| Aspecte                | Spring Boot                                 | FastAPI                              |
|:-----------------------|:--------------------------------------------|:-------------------------------------|
| Definir ruta           | `@GetMapping("/games/{id}")`                | `@app.get("/games/{id}")`            |
| Path parameter         | `@PathVariable String id`                   | `def f(id: str)`                     |
| Request body           | `@RequestBody @Valid CreateGameRequest req`  | `def f(req: CreateGameRequest)`      |
| Response model         | Retorna el DTO                              | `response_model=GameDTO`             |
| Validació              | `@Valid` + Bean Validation                   | Pydantic automàtic                   |
| Error handling         | `@ControllerAdvice`                          | `HTTPException`                      |
| Docs                   | SpringDoc + `@Operation`                     | Automàtic a `/docs`                  |
| Server                 | `mvn spring-boot:run` (Tomcat)               | `uvicorn main:app --reload`          |
| Port per defecte       | 8080                                         | 8000                                 |

### Executar i provar

```bash
pip install fastapi uvicorn httpx anthropic pydantic
uvicorn main:app --reload
curl http://localhost:8000/health
curl http://localhost:8000/analyze/APP-1
open http://localhost:8000/docs    # Swagger auto-generat amb "Try it out"
```

---

## 8. MCP: Model Context Protocol

### El problema que resol

Cada eina d'IA (Cursor, ChatGPT, Copilot, Claude Desktop) té el seu propi format de plugins. Cada desenvolupador ha de fer integracions separades per cada plataforma. MCP proposa un estàndard: un protocol únic perquè qualsevol eina es connecti a qualsevol model.

### Arquitectura

```
┌─────────────────────────────────────────────────────────────────┐
│  MCP Client (Cursor, Claude Desktop, IDE...)                    │
│                                                                 │
│  L'usuari fa una pregunta:                                      │
│  "Quants jugadors té el League of Legends?"                     │
│                                                                 │
│  El client detecta que necessita dades                          │
│  → crida el MCP Server via stdio                                │
└──────────────────┬──────────────────────────────────────────────┘
                   │  JSON-RPC 2.0 (stdin/stdout)
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│  MCP Server (el teu codi Python)                                │
│                                                                 │
│  Exposa:                                                        │
│  ┌──────────┐  ┌──────────────┐  ┌──────────────────┐          │
│  │  Tools   │  │  Resources   │  │  Prompts         │          │
│  │          │  │              │  │                  │          │
│  │ get_game │  │ games://list │  │ analyze_game     │          │
│  │ search   │  │ games://top  │  │ compare_patches  │          │
│  └──────────┘  └──────────────┘  └──────────────────┘          │
│                                                                 │
│  El server consulta l'API Java, la BD, fitxers...               │
└─────────────────────────────────────────────────────────────────┘
```

### Els tres primitius

**1. Tools (accions):**
Funcions que la IA pot cridar. Equivalen a les funcions de tool_use/function_calling, però exposades com a servei independent.

```python
@server.tool()
async def get_game(game_id: str) -> str:
    """Obté les dades d'un joc per ID."""
    resp = httpx.get(f"http://localhost:8080/games/{game_id}")
    return resp.text
```

**2. Resources (dades):**
Dades que la IA pot llegir, com fitxers o taules. Són read-only.

```python
@server.resource("gamepulse://games/top10")
async def top_games() -> str:
    """Els 10 jocs amb més jugadors actius."""
    resp = httpx.get("http://localhost:8080/games?sort=players&limit=10")
    return resp.text
```

**3. Prompts (plantilles):**
Templates reutilitzables amb paràmetres per tasques comunes.

### Configurar MCP a Cursor

Crea `.cursor/mcp.json` al projecte. Cursor detectarà el server i podrà cridar els tools des del chat:

```json
{ "mcpServers": { "gamepulse": { "command": "python", "args": ["mcp_server.py"] } } }
```

### L'analogia REST

```
REST (S7):                              MCP (S8):
Web App ←→ REST API ←→ BD              IA Client ←→ MCP Server ←→ Dades
   HTTP    endpoints   SQL                stdio    tools/resources  API/BD

Contracte: OpenAPI/Swagger              Contracte: MCP Protocol
Format: JSON                            Format: JSON-RPC
```

MCP és per a eines IA el que REST és per a apps web: un contracte estàndard que separa qui demana de qui serveix.

---

## 9. El Bridge Java i Python

Amb S8, GamePulse passa de ser una app monolítica Java a tenir dos serveis comunicats per HTTP:

```
  Usuari → curl /analyze/APP-1
              │
              ▼
  ┌─────────────────────┐     GET /games/APP-1     ┌───────────────┐
  │ Python (FastAPI)    │─────────────────────────►│ Java (Spring) │
  │ :8000               │◄─────────────────────────│ :8080         │
  │ /analyze, /batch    │     JSON response         │ /games (CRUD) │
  └────────┬────────────┘                           │ H2 DB         │
           │ anthropic SDK                          └───────────────┘
           ▼
  ┌─────────────────────┐
  │ LLM (tool_use)      │
  └─────────────────────┘
```

El flux complet: FastAPI rep la request, fa fetch a l'API Java, construeix el prompt amb les dades del joc, crida el LLM amb tool_use, valida amb Pydantic, i retorna el `GameAnalysis`.

### Per què dos serveis?

1. **Separació de responsabilitats:** Java fa CRUD i persistència. Python fa anàlisi IA.
2. **Desplegament independent:** Pots escalar el servei Python sense tocar el Java.
3. **El millor de cada món:** JPA+H2 per a dades, Pydantic+FastAPI per a IA.

---

## 10. Aplicació a GamePulse: L'Agent Analista

Aquesta setmana, l'"agent" és un script lineal que: rep un `game_id`, fa fetch a l'API Java, crida el LLM amb tool_use, valida amb Pydantic, i retorna un `GameAnalysis`. No és un agent autònom (això vindrà a S16) -- es un pipeline determinista amb un pas no-determinista (el LLM).

### El codi complet

```python
# agent_analyst.py
import httpx
import anthropic
from models import GameAnalysis

JAVA_API = "http://localhost:8080"
client = anthropic.Anthropic()

def analyze_game(game_id: str) -> GameAnalysis:
    # 1. Fetch dades del joc
    resp = httpx.get(f"{JAVA_API}/games/{game_id}")
    resp.raise_for_status()
    game = resp.json()

    # 2. Prompt amb few-shot + tool_use
    tool_def = {
        "name": "submit_analysis",
        "description": "Retorna l'anàlisi del joc.",
        "input_schema": GameAnalysis.model_json_schema()
    }
    prompt = f"""Analitza {game["title"]} ({game["activePlayerCount"]} jugadors, {game["price"]}€).
    Exemples: LoL, 5M, free → positive, up | Cyberpunk, 200K, 59.99€ → mixed, stable"""

    response = client.messages.create(
        model="claude-sonnet-4-20250514", max_tokens=1024, temperature=0.2,
        tools=[tool_def],
        tool_choice={"type": "tool", "name": "submit_analysis"},
        messages=[{"role": "user", "content": prompt}]
    )

    # 3. Validar amb Pydantic
    tool_block = next(b for b in response.content if b.type == "tool_use")
    return GameAnalysis.model_validate(tool_block.input)
```

### Testing (sense gastar tokens)

```python
# test_agent.py
from unittest.mock import MagicMock

def test_analyze_returns_valid_game_analysis(monkeypatch):
    # Mock LLM
    mock_block = MagicMock(type="tool_use", input={
        "game_id": "APP-1", "title": "LoL", "sentiment": "positive",
        "player_trend": {"current_players": 5000000, "peak_players": 8000000, "trend_direction": "up"},
        "summary": "Free-to-play MOBA amb tendència positiva."
    })
    mock_client = MagicMock()
    mock_client.messages.create.return_value = MagicMock(content=[mock_block])
    monkeypatch.setattr("agent_analyst.client", mock_client)

    # Mock API Java
    mock_resp = MagicMock(status_code=200)
    mock_resp.json.return_value = {"appId": "APP-1", "title": "LoL", "price": 0.0, "activePlayerCount": 5000000}
    mock_resp.raise_for_status = MagicMock()
    monkeypatch.setattr("httpx.get", lambda url: mock_resp)

    from agent_analyst import analyze_game
    result = analyze_game("APP-1")
    assert result.game_id == "APP-1"
    assert result.sentiment == "positive"
```

### Cap a agents reals (S16)

Ara tens un pipeline lineal. A S16 afegiràs decisions (quines eines cridar), memòria (anàlisis anteriors), bucles (reintentar si falta info), i multi-tool. Però el fonament -- fetch + LLM + validació -- ja el tens.

---

## Resum

| Concepte                     | Key Takeaway                                                              |
|:-----------------------------|:--------------------------------------------------------------------------|
| Java + Python                | Backend core en Java, IA/anàlisi en Python. Complementaris, no redundants.|
| Pydantic v2                  | Models amb validació en 3 línies. El JSON Schema bridge entre llenguatges.|
| LLM com a funció             | Sense schema: text lliure. Amb schema: funció amb signatura.              |
| Messages API                 | system/user/assistant. Temperature baixa per a precisió.                  |
| tool_use / function_calling  | Força el LLM a retornar JSON que compleix un schema.                      |
| Few-shot prompting           | 2-3 exemples al prompt milloren la qualitat sense complicar el codi.      |
| FastAPI                      | Pydantic integrat, docs automàtiques, async natiu.                        |
| MCP                          | L'USB de la IA: protocol estàndard per connectar eines a models.          |
| Bridge Java-Python           | Dos serveis HTTP, cada un fent el que el seu llenguatge fa millor.        |
| Agent analista               | Fetch + LLM + validació = la base de qualsevol agent.                     |
