# Setmana 10 — Divendres: Integracio — FastAPI + LLM + Java API

## Objectiu del Dia

Integrar tots els components de la setmana en un flux complet: FastAPI rep peticions, consulta la Java API, crida Claude per generar analisis estructurades, valida amb Pydantic i retorna la resposta. Escriurem tests amb pytest que no gasten tokens de l'API. Al final del dia tindras el sistema funcionant end-to-end amb 3 tests passant.

---

## Teoria

### El Flux Complet d'EsportsPulse AI

```
┌──────────┐    ┌─────────────┐    ┌──────────────┐    ┌─────────────┐
│  Client   │    │   FastAPI    │    │   Java API   │    │   Claude    │
│  (curl/   │───►│   (Python)   │───►│  (Spring     │    │   (LLM)     │
│   front)  │    │   Port 8001  │    │   Boot)      │    │             │
│           │◄───│              │◄───│   Port 8080  │    │             │
└──────────┘    │              │    └──────────────┘    │             │
                │              │                        │             │
                │              │───────────────────────►│             │
                │              │◄───────────────────────│             │
                │              │                        └─────────────┘
                │   Pydantic   │
                │   Validation │
                └─────────────┘

Flux:
1. Client envia GET /analyze/222
2. FastAPI crida Java API: GET /api/champions/222
3. Java API retorna dades basiques del campio (nom, rol)
4. FastAPI envia prompt a Claude amb tool_use
5. Claude retorna JSON estructurat (ChampionAnalysis)
6. Pydantic valida la resposta de Claude
7. FastAPI retorna l'analisi valida al client
```

### agent_analyst.py: El Script Complet

Aquest script combina tots els conceptes de la setmana en un sol fitxer cohesiu:

```python
# agent_analyst.py — Agent analista que integra Java API + Claude + Pydantic
# Combina els conceptes de dilluns (models), dimarts (LLM) i dimecres (FastAPI)

import json
import httpx
from anthropic import Anthropic
from pydantic import ValidationError
from models.champion_analysis import ChampionAnalysis

# Configuracio — en produccio, usar variables d'entorn
JAVA_API_URL = "http://localhost:8080/api"
LLM_MODEL = "claude-sonnet-4-20250514"
LLM_TEMPERATURE = 0.2  # Baixa per consistencia en dades estructurades


def fetch_champion_from_java_api(champion_id: int) -> dict:
    """
    Obte les dades basiques d'un campio de la Java API (Spring Boot).
    Retorna un dict amb les dades del campio.
    Llanca ValueError si el campio no existeix (404).
    Llanca ConnectionError si la Java API no respon.
    """
    try:
        # httpx.get es la versio sincrona — mes senzilla per scripts
        response = httpx.get(
            f"{JAVA_API_URL}/champions/{champion_id}",
            timeout=10.0
        )
        response.raise_for_status()
        return response.json()
    except httpx.HTTPStatusError as e:
        if e.response.status_code == 404:
            raise ValueError(f"Campio {champion_id} no trobat a la Java API")
        raise ConnectionError(f"Error Java API: {e.response.status_code}")
    except httpx.ConnectError:
        raise ConnectionError("No es pot connectar amb la Java API")


def call_llm_for_analysis(champion_id: int, champion_name: str) -> dict:
    """
    Crida Claude amb tool_use per obtenir una analisi estructurada.
    Retorna el dict raw de la resposta (abans de validar amb Pydantic).
    """
    client = Anthropic()

    # Definim la tool amb l'esquema Pydantic
    tool_def = {
        "name": "submit_champion_analysis",
        "description": "Envia l'analisi estructurada d'un campio de LoL.",
        "input_schema": ChampionAnalysis.model_json_schema()
    }

    # System prompt amb context i exemples few-shot
    system = (
        "Ets un analista expert de League of Legends amb acces a dades "
        "actualitzades de patches i estadistiques. Analitza campions amb "
        "precisio. Respon sempre en catala.\n\n"
        "Exemple: Ahri (103) — MID, Tier A, win_rate 51.2%, "
        "tendencia estable, strengths: mobilitat, CC, poke. "
        "counters: Zed, Fizz, Kassadin.\n\n"
        "Exemple: Thresh (412) — SUPPORT, Tier B, win_rate 50.5%, "
        "tendencia down, strengths: engage, peel, llanternes. "
        "counters: Morgana, Braum, Zyra."
    )

    # Crida a Claude amb tool_use forcat
    response = client.messages.create(
        model=LLM_MODEL,
        max_tokens=2048,
        system=system,
        temperature=LLM_TEMPERATURE,
        tools=[tool_def],
        tool_choice={"type": "tool", "name": "submit_champion_analysis"},
        messages=[
            {
                "role": "user",
                "content": (
                    f"Analitza el campio {champion_name} "
                    f"(champion_id: {champion_id}) al patch actual."
                )
            }
        ]
    )

    # Extraiem el bloc tool_use de la resposta
    tool_block = next(b for b in response.content if b.type == "tool_use")
    return tool_block.input


def analyze_champion_full(champion_id: int) -> ChampionAnalysis:
    """
    Flux complet d'analisi d'un campio:
    1. Obte dades de la Java API
    2. Crida Claude per generar l'analisi
    3. Valida la resposta amb Pydantic
    Retorna un ChampionAnalysis validat.
    """
    # Pas 1: Obtenim dades del campio de la Java API
    champion_data = fetch_champion_from_java_api(champion_id)
    champion_name = champion_data.get("name", f"Champion_{champion_id}")

    # Pas 2: Cridem Claude per generar l'analisi
    raw_analysis = call_llm_for_analysis(champion_id, champion_name)

    # Pas 3: Validem amb Pydantic — si falla, llancem ValueError
    try:
        analysis = ChampionAnalysis(**raw_analysis)
        return analysis
    except ValidationError as e:
        raise ValueError(f"Resposta LLM invalida:\n{e}")


# Execucio directa per provar el flux complet
if __name__ == "__main__":
    print("=== EsportsPulse AI Analyst ===\n")
    result = analyze_champion_full(222)
    print(f"Campio: {result.name}")
    print(f"Rol: {result.role} | Tier: {result.patch_tier}")
    print(f"Win rate: {result.champion_trend.current_win_rate}%")
    print(f"Tendencia: {result.champion_trend.trend_direction}")
    print(f"Punts forts: {', '.join(result.strengths)}")
    print(f"Counters: {', '.join(result.counters)}")
    print(f"Resum: {result.summary}")
    print(f"\n=== JSON complet ===")
    print(result.model_dump_json(indent=2))
```

### Testing amb pytest: Sense Gastar Tokens

Cridar Claude en cada test seria car i lent. La solucio: **monkeypatch** per substituir les crides externes amb respostes controlades.

```python
# test_agent_analyst.py — Tests del flux complet sense crides reals a APIs
# Utilitzem monkeypatch de pytest per substituir les funcions que criden APIs externes

import pytest
from pydantic import ValidationError
from agent_analyst import (
    analyze_champion_full,
    fetch_champion_from_java_api,
    call_llm_for_analysis,
)
from models.champion_analysis import ChampionAnalysis


# --- FIXTURES: Dades de prova reutilitzables ---

@pytest.fixture
def valid_java_response():
    """Simula una resposta correcta de la Java API."""
    return {
        "id": 222,
        "name": "Jinx",
        "role": "ADC",
        "win_rate": 52.3,
        "pick_rate": 15.2
    }


@pytest.fixture
def valid_llm_response():
    """Simula una resposta correcta de Claude (tool_use output)."""
    return {
        "champion_id": 222,
        "name": "Jinx",
        "role": "ADC",
        "strengths": ["Late game scaling", "AOE damage", "Tower pushing"],
        "counters": ["Zed", "Fizz", "Katarina"],
        "patch_tier": "S",
        "champion_trend": {
            "current_win_rate": 52.3,
            "previous_win_rate": 50.1,
            "trend_direction": "up",
            "games_analyzed": 15234
        },
        "summary": "Jinx es una ADC devastadora al late game amb DPS excepcional i AOE."
    }


@pytest.fixture
def invalid_llm_response():
    """Simula una resposta incorrecta de Claude (camps invalids)."""
    return {
        "champion_id": -1,         # Invalid: ha de ser > 0
        "name": "jinx",            # Invalid: ha de comencar amb majuscula
        "role": "ASSASSIN",        # Invalid: no es un Literal valid
        "strengths": [],           # Invalid: min_length=1
        "counters": ["Zed"],
        "patch_tier": "S",
        "champion_trend": {
            "current_win_rate": 150.0,  # Invalid: max 100
            "previous_win_rate": 50.0,
            "trend_direction": "up",
            "games_analyzed": 0         # Invalid: ha de ser > 0
        },
        "summary": "Curt"          # Invalid: min_length=20
    }


# --- TEST 1: Resposta LLM valida ---

def test_valid_llm_response(monkeypatch, valid_java_response, valid_llm_response):
    """
    Comprova que el flux complet funciona quan la Java API i Claude
    retornen dades correctes. El resultat ha de ser un ChampionAnalysis valid.
    """
    # Substituim fetch_champion_from_java_api per retornar dades controlades
    # Aixi no necessitem la Java API activa durant els tests
    monkeypatch.setattr(
        "agent_analyst.fetch_champion_from_java_api",
        lambda cid: valid_java_response
    )

    # Substituim call_llm_for_analysis per retornar JSON controlat
    # Aixi no gastem tokens de Claude durant els tests
    monkeypatch.setattr(
        "agent_analyst.call_llm_for_analysis",
        lambda cid, name: valid_llm_response
    )

    # Executem el flux complet — ha de funcionar sense errors
    result = analyze_champion_full(222)

    # Verifiquem que el resultat es correcte
    assert isinstance(result, ChampionAnalysis)
    assert result.name == "Jinx"
    assert result.role == "ADC"
    assert result.patch_tier == "S"
    assert result.champion_trend.current_win_rate == 52.3
    assert result.champion_trend.trend_direction == "up"
    assert len(result.strengths) == 3
    assert len(result.counters) == 3


# --- TEST 2: Resposta LLM invalida (ValidationError) ---

def test_invalid_llm_response(monkeypatch, valid_java_response, invalid_llm_response):
    """
    Comprova que quan Claude retorna dades invalides, el sistema
    llanca ValueError (que encapsula el ValidationError de Pydantic).
    """
    # Java API funciona correctament
    monkeypatch.setattr(
        "agent_analyst.fetch_champion_from_java_api",
        lambda cid: valid_java_response
    )

    # Claude retorna dades invalides — multiples camps incorrectes
    monkeypatch.setattr(
        "agent_analyst.call_llm_for_analysis",
        lambda cid, name: invalid_llm_response
    )

    # El flux ha de fallar amb ValueError (que conte el ValidationError)
    with pytest.raises(ValueError, match="Resposta LLM invalida"):
        analyze_champion_full(222)


# --- TEST 3: Campio no trobat (404 de la Java API) ---

def test_champion_not_found(monkeypatch):
    """
    Comprova que quan la Java API retorna 404 (campio no existeix),
    el sistema llanca ValueError amb un missatge clar.
    """
    # Simulem que la Java API retorna 404
    def mock_fetch_404(champion_id):
        raise ValueError(f"Campio {champion_id} no trobat a la Java API")

    monkeypatch.setattr(
        "agent_analyst.fetch_champion_from_java_api",
        mock_fetch_404
    )

    # El flux ha de fallar amb ValueError indicant que no s'ha trobat
    with pytest.raises(ValueError, match="no trobat"):
        analyze_champion_full(9999)  # ID que no existeix
```

### Executar els Tests

```bash
# Instal·la pytest si no el tens
pip install pytest

# Executa els tests des del directori del projecte
cd esportspulse-engine/ai-python/src
pytest test_agent_analyst.py -v

# Sortida esperada:
# test_agent_analyst.py::test_valid_llm_response PASSED
# test_agent_analyst.py::test_invalid_llm_response PASSED
# test_agent_analyst.py::test_champion_not_found PASSED
# =================== 3 passed in 0.12s ===================
```

### Integrar amb FastAPI (main.py actualitzat)

```python
# main.py — Versio final amb agent_analyst integrat
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from models.champion_analysis import ChampionAnalysis
from agent_analyst import analyze_champion_full

app = FastAPI(
    title="EsportsPulse AI Service",
    version="1.0.0"
)

# Model de request per batch
class BatchRequest(BaseModel):
    champion_ids: list[int]

# Model de resposta per batch
class BatchResponse(BaseModel):
    results: list[ChampionAnalysis]
    errors: list[dict]

@app.get("/health")
async def health():
    """Health check del servei."""
    return {"status": "healthy", "service": "esportspulse-ai"}

@app.get("/analyze/{champion_id}", response_model=ChampionAnalysis)
async def analyze(champion_id: int):
    """
    Analitza un campio: Java API -> Claude -> Pydantic -> Resposta.
    """
    try:
        # analyze_champion_full encapsula tot el flux
        return analyze_champion_full(champion_id)
    except ValueError as e:
        # Campio no trobat o resposta LLM invalida
        raise HTTPException(status_code=404, detail=str(e))
    except ConnectionError as e:
        # Java API no disponible
        raise HTTPException(status_code=502, detail=str(e))

@app.post("/batch-analyze", response_model=BatchResponse)
async def batch_analyze(request: BatchRequest):
    """
    Analitza multiples campions. Continua si un falla.
    """
    results = []
    errors = []

    for cid in request.champion_ids:
        try:
            analysis = analyze_champion_full(cid)
            results.append(analysis)
        except (ValueError, ConnectionError) as e:
            errors.append({"champion_id": cid, "error": str(e)})

    return BatchResponse(results=results, errors=errors)
```

### Resum de la Setmana: Que Hem Apres

```
Dilluns:   Pydantic → Models validats amb type safety real
Dimarts:   Claude API → tool_use per forcar JSON estructurat
Dimecres:  FastAPI → Servei web amb docs auto-generades
Dijous:    MCP → Protocol per connectar IA amb eines
Divendres: Integracio → Tot connectat amb tests sense gastar tokens

     ┌─────────┐
     │ Pydantic │ ← Validacio i esquema (dilluns)
     └────┬────┘
          │
     ┌────▼────┐
     │  Claude  │ ← tool_use amb esquema Pydantic (dimarts)
     │   API    │
     └────┬────┘
          │
     ┌────▼────┐
     │ FastAPI  │ ← Endpoints amb models Pydantic (dimecres)
     └────┬────┘
          │
     ┌────▼────┐
     │   MCP   │ ← Connexio IA-eines (dijous)
     └────┬────┘
          │
     ┌────▼────┐
     │  Tests  │ ← monkeypatch sense tokens (divendres)
     └─────────┘
```

---

## Activitat

### Pas 1: Crea agent_analyst.py (20 min)

Crea el fitxer `ai-python/src/agent_analyst.py` amb les tres funcions de la teoria:
1. `fetch_champion_from_java_api()` — crida la Java API
2. `call_llm_for_analysis()` — crida Claude amb tool_use
3. `analyze_champion_full()` — combina tot el flux

### Pas 2: Escriu els 3 Tests (20 min)

Crea `ai-python/src/test_agent_analyst.py` amb:
1. `test_valid_llm_response` — flux complet amb dades correctes
2. `test_invalid_llm_response` — Claude retorna dades invalides
3. `test_champion_not_found` — campio no existeix (404)

### Pas 3: Executa els Tests (5 min)

```bash
# Executa i verifica que els 3 tests passen
cd esportspulse-engine/ai-python/src
pytest test_agent_analyst.py -v

# Si algun falla, llegeix l'error de Pydantic i corregeix les dades del fixture
```

### Pas 4: Prova el Flux End-to-End (10 min)

Si tens la Java API activa i la clau de Claude configurada:

```bash
# Arrenca el servei
uvicorn main:app --reload --port 8001

# Prova el flux complet
curl http://localhost:8001/analyze/222 | python -m json.tool
```

### Pas 5: Crea el PR (5 min)

```bash
# Crea una branca per la feina de la setmana
git checkout -b feature/week10-python-ai

# Afegeix tots els fitxers nous
git add ai-python/

# Commit amb missatge descriptiu
git commit -m "feat(ai): add Python AI service with Pydantic, Claude API and FastAPI

- Pydantic models for ChampionAnalysis with full validation
- Claude API integration with tool_use for structured output
- FastAPI service with /analyze and /batch-analyze endpoints
- Tests with monkeypatch (no API tokens needed)"

# Puja la branca i crea el PR
git push -u origin feature/week10-python-ai
```

---

## Checklist de Lliurament

- [ ] `agent_analyst.py` creat amb les 3 funcions (fetch, LLM, flux complet)
- [ ] `test_agent_analyst.py` creat amb 3 tests (valid, invalid, not found)
- [ ] Els 3 tests passen amb `pytest -v`
- [ ] Cap test fa crides reals a APIs externes (tot amb monkeypatch)
- [ ] `main.py` actualitzat amb `analyze_champion_full` integrat
- [ ] El flux end-to-end funciona (si la Java API i Claude estan disponibles)
- [ ] PR creada amb tots els fitxers de la setmana
- [ ] Entens el flux complet: Client -> FastAPI -> Java API -> Claude -> Pydantic -> Resposta
