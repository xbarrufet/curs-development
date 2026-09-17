# Setmana 10 — Dimecres: FastAPI Basics — Building the Python Service

## Objectiu del Dia

Construir el servei Python d'EsportsPulse amb FastAPI, exposant endpoints que consumeixen la Java API i criden Claude per generar analisis estructurades. Al final del dia tindras un servei amb `/health`, `/analyze/{champion_id}` i `/batch-analyze`, amb documentacio auto-generada a `/docs`.

---

## Teoria

### Per que FastAPI?

FastAPI es el framework web modern de Python. Comparat amb Flask o Django REST, te avantatges clau per a EsportsPulse:

| Caracteristica        | FastAPI                              | Spring Boot (Java)                    |
|----------------------|--------------------------------------|---------------------------------------|
| Routing              | Decoradors `@app.get("/path")`       | Anotacions `@GetMapping("/path")`     |
| Validacio            | Pydantic integrat nativament         | Bean Validation (`@Valid`)            |
| Documentacio API     | Auto-generada a `/docs` (Swagger UI) | SpringDoc (`/swagger-ui.html`)        |
| Servidor             | Uvicorn (ASGI, async)                | Tomcat (Servlet, sync per defecte)    |
| Type hints           | Obligatoris (part del framework)     | Opcionals (Java es tipat estatic)     |
| Velocitat de dev     | Molt rapida (menys boilerplate)      | Mes boilerplate pero mes estructura   |

```python
# Comparativa rapida de routing:

# FastAPI (Python)
@app.get("/champions/{champion_id}")
async def get_champion(champion_id: int) -> ChampionAnalysis:
    ...

# Spring Boot (Java) — equivalent
# @GetMapping("/champions/{championId}")
# public ChampionAnalysis getChampion(@PathVariable int championId) { ... }
```

### Instal·lacio i Primer Endpoint

```python
# Instal·la FastAPI i Uvicorn (el servidor ASGI)
# pip install fastapi uvicorn httpx

# main.py — Punt d'entrada del servei Python d'EsportsPulse
from fastapi import FastAPI

# Crea la instancia de l'aplicacio FastAPI
# title i version apareixen a la documentacio auto-generada (/docs)
app = FastAPI(
    title="EsportsPulse AI Service",
    description="Servei d'analisi de campions amb IA per a EsportsPulse",
    version="1.0.0"
)

# Endpoint de salut — per verificar que el servei esta actiu
# Equivalent a @GetMapping("/health") en Spring Boot
@app.get("/health")
async def health_check():
    """Retorna l'estat del servei. Util per a health checks de Docker/K8s."""
    return {
        "status": "healthy",
        "service": "esportspulse-ai",
        "version": "1.0.0"
    }
```

Per executar el servei:

```bash
# Arrenca el servidor amb recarga automatica (com nodemon o Spring DevTools)
# --reload detecta canvis als fitxers i reinicia automaticament
uvicorn main:app --reload --port 8001

# El servei arrenca a http://localhost:8001
# La documentacio interactiva esta a http://localhost:8001/docs
# L'esquema OpenAPI esta a http://localhost:8001/openapi.json
```

### Pydantic com a Request/Response Schema

FastAPI utilitza Pydantic per validar tant les entrades com les sortides. El parametre `response_model` defineix l'esquema de la resposta i apareix a la documentacio.

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

# Model de request per al batch — llista d'IDs a analitzar
class BatchAnalyzeRequest(BaseModel):
    """Peticio per analitzar multiples campions de cop."""
    champion_ids: list[int]  # Llista d'IDs de campions a analitzar
```

### GET /analyze/{champion_id}: Analisi Individual

```python
import httpx
from fastapi import FastAPI, HTTPException
from models.champion_analysis import ChampionAnalysis
from llm.analyze_champion import analyze_champion

# URL base del servei Java — en produccio seria una variable d'entorn
JAVA_API_URL = "http://localhost:8080/api"

@app.get(
    "/analyze/{champion_id}",
    response_model=ChampionAnalysis,  # Defineix l'esquema de la resposta a /docs
    summary="Analitza un campio amb IA",
    description="Obte dades del campio de la Java API i genera una analisi amb Claude."
)
async def analyze_single_champion(champion_id: int):
    """
    Flux complet:
    1. Crida la Java API per obtenir dades basiques del campio
    2. Envia les dades a Claude per generar l'analisi
    3. Valida la resposta amb Pydantic
    4. Retorna l'analisi estructurada
    """
    # Pas 1: Obtenim dades del campio de la Java API (Spring Boot)
    # httpx es el client HTTP async de Python (equivalent a RestTemplate/WebClient)
    async with httpx.AsyncClient() as client:
        try:
            response = await client.get(
                f"{JAVA_API_URL}/champions/{champion_id}",
                timeout=10.0  # Timeout de 10 segons per evitar bloquejos
            )
            response.raise_for_status()  # Llanca excepcio si status >= 400
        except httpx.HTTPStatusError as e:
            if e.response.status_code == 404:
                # El campio no existeix a la Java API
                raise HTTPException(
                    status_code=404,
                    detail=f"Campio amb ID {champion_id} no trobat a la base de dades"
                )
            # Qualsevol altre error de la Java API
            raise HTTPException(
                status_code=502,
                detail=f"Error comunicant amb la Java API: {e.response.status_code}"
            )
        except httpx.ConnectError:
            # La Java API no esta disponible
            raise HTTPException(
                status_code=502,
                detail="No es pot connectar amb la Java API. Esta el servei actiu?"
            )

    # Extraiem el nom del campio de la resposta Java
    champion_data = response.json()
    champion_name = champion_data.get("name", f"Champion_{champion_id}")

    # Pas 2: Cridem Claude per generar l'analisi estructurada
    try:
        analysis = analyze_champion(champion_id, champion_name)
        return analysis
    except ValueError as e:
        # La resposta de Claude no ha passat la validacio Pydantic
        raise HTTPException(
            status_code=502,
            detail=f"L'analisi generada per la IA no es valida: {str(e)}"
        )
```

### POST /batch-analyze: Analisi en Lot

```python
from pydantic import BaseModel

# Model de request — defineix l'estructura del body JSON
class BatchAnalyzeRequest(BaseModel):
    """Peticio per analitzar multiples campions."""
    champion_ids: list[int]  # Llista d'IDs (Pydantic valida que siguin enters)

# Model de resposta — inclou resultats i possibles errors
class BatchAnalyzeResponse(BaseModel):
    """Resposta amb les analisis generades i els errors trobats."""
    results: list[ChampionAnalysis]  # Analisis completades amb exit
    errors: list[dict]               # Errors per campio (id + missatge)

@app.post(
    "/batch-analyze",
    response_model=BatchAnalyzeResponse,
    summary="Analitza multiples campions",
    description="Envia una llista d'IDs i retorna les analisis generades per IA."
)
async def batch_analyze(request: BatchAnalyzeRequest):
    """
    Analitza multiples campions. Si un falla, continua amb els altres.
    Retorna tant els resultats exitosos com els errors.
    """
    results = []
    errors = []

    for cid in request.champion_ids:
        try:
            # Reutilitzem la logica d'analisi individual
            analysis = await analyze_single_champion(cid)
            results.append(analysis)
        except HTTPException as e:
            # Guardem l'error pero continuem amb els altres campions
            errors.append({
                "champion_id": cid,
                "error": e.detail
            })

    return BatchAnalyzeResponse(results=results, errors=errors)
```

### HTTPException: Gestio d'Errors

FastAPI utilitza `HTTPException` per retornar errors HTTP amb codis i missatges clars:

```python
from fastapi import HTTPException

# 404 — Recurs no trobat (equivalent a ResponseEntity.notFound() en Spring)
raise HTTPException(status_code=404, detail="Campio no trobat")

# 502 — Bad Gateway (la Java API o Claude ha fallat)
raise HTTPException(status_code=502, detail="Error al servei extern")

# 422 — Validation Error (automatic si Pydantic falla al parsejar el request)
# FastAPI retorna automaticament un 422 amb detalls dels camps invalids
```

### Documentacio Auto-generada

Quan arranques el servei i vas a `http://localhost:8001/docs`, FastAPI genera automaticament:

- Tots els endpoints amb metodes HTTP
- Esquemes de request i response (des dels models Pydantic)
- Formulari interactiu per provar cada endpoint
- Descripcions dels camps (des de `Field(description=...)`)

Aixo es equivalent a SpringDoc/Swagger en Spring Boot, pero sense cap configuracio addicional.

### Estructura del Projecte

```
esportspulse-engine/ai-python/
├── src/
│   ├── main.py                     # Punt d'entrada FastAPI
│   ├── models/
│   │   ├── __init__.py
│   │   └── champion_analysis.py    # Models Pydantic (dilluns)
│   └── llm/
│       ├── __init__.py
│       └── analyze_champion.py     # Logica de crida a Claude (dimarts)
├── requirements.txt
└── .venv/
```

```bash
# requirements.txt — Dependencies del servei Python
fastapi==0.115.0
uvicorn==0.30.0
httpx==0.27.0
anthropic==0.39.0
pydantic==2.9.0
```

---

## Activitat

### Pas 1: Crea l'Estructura del Projecte (10 min)

```bash
# Des del directori ai-python
cd esportspulse-engine/ai-python

# Crea els directoris necessaris
mkdir -p src/models src/llm

# Crea els fitxers __init__.py perque Python tracti els directoris com a moduls
touch src/__init__.py src/models/__init__.py src/llm/__init__.py

# Copia els models de dilluns i l'script de dimarts als seus directoris
# (si no els tens, crea'ls seguint els passos dels dies anteriors)

# Instal·la les dependencies
pip install fastapi uvicorn httpx
```

### Pas 2: Implementa main.py (30 min)

Crea `ai-python/src/main.py` amb:
1. L'endpoint `/health`
2. L'endpoint `GET /analyze/{champion_id}`
3. L'endpoint `POST /batch-analyze`

Segueix el codi de la teoria, adaptant-lo al teu projecte.

### Pas 3: Arrenca i Prova (15 min)

```bash
# Arrenca el servidor
cd esportspulse-engine/ai-python/src
uvicorn main:app --reload --port 8001

# Prova /health amb curl
curl http://localhost:8001/health
# Resposta esperada: {"status":"healthy","service":"esportspulse-ai","version":"1.0.0"}

# Obre la documentacio interactiva al navegador
# http://localhost:8001/docs

# Prova /analyze/222 (necessita la Java API activa i API key configurada)
curl http://localhost:8001/analyze/222

# Prova /batch-analyze
curl -X POST http://localhost:8001/batch-analyze \
  -H "Content-Type: application/json" \
  -d '{"champion_ids": [222, 103, 412]}'
```

### Pas 4: Compara amb Spring Boot (10 min)

Obre el codi del teu controlador Java de la setmana 5 i compara:
- Com es defineixen les rutes
- Com es gestionen els errors
- Com es genera la documentacio
- Quantes linies de codi fan falta per la mateixa funcionalitat

---

## Checklist de Lliurament

- [ ] `main.py` creat amb tots tres endpoints (`/health`, `/analyze`, `/batch-analyze`)
- [ ] El servei arrenca amb `uvicorn main:app --reload --port 8001`
- [ ] `/health` retorna status healthy
- [ ] `/analyze/{champion_id}` crida la Java API i Claude, retorna `ChampionAnalysis`
- [ ] `/batch-analyze` processa multiples campions i retorna resultats + errors
- [ ] `/docs` mostra la documentacio auto-generada amb tots els esquemes
- [ ] `HTTPException` gestiona correctament els errors 404 i 502
- [ ] `requirements.txt` actualitzat amb totes les dependencies
