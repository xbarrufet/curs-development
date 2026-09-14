# Setmana 11 — Divendres: Patrons de Resiliència i Tests d'Integració

## Objectiu del Dia

Consolidar tot el que hem fet aquesta setmana amb **tests d'integració** que verifiquen el comportament del sistema quan les coses fallen: timeouts, serveis caiguts, retries i propagació de correlation IDs. Al final del dia, tindràs un sistema robust i testejat, amb una PR creada.

---

## Teoria

### Timeouts: La Primera Línia de Defensa

**Regla absoluta:** SEMPRE configura un timeout en cada crida HTTP. Si no ho fas, un servei que no respon pot bloquejar el teu sistema indefinidament.

```python
import httpx

# MALAMENT: Sense timeout. Si Java no respon, Python queda bloquejat PER SEMPRE.
async with httpx.AsyncClient() as client:
    response = await client.get("http://localhost:8080/api/champions/1")

# BÉ: Timeout de 5 segons per APIs normals.
# Si Java no respon en 5s, llençarà httpx.TimeoutException.
async with httpx.AsyncClient(timeout=5.0) as client:
    response = await client.get("http://localhost:8080/api/champions/1")

# BÉ: Timeout de 30 segons per crides a LLM (són més lentes).
# Els LLMs poden trigar 10-20s en generar una resposta llarga.
async with httpx.AsyncClient(timeout=30.0) as client:
    response = await client.post("https://api.openai.com/v1/chat/completions", ...)
```

**Valors recomanats:**

| Tipus de Crida | Timeout | Per Què |
|---------------|---------|---------|
| API interna (Java ↔ Python) | 5 segons | Si triga més, alguna cosa va malament |
| API externa (Riot API) | 10 segons | Xarxa pública, latència variable |
| LLM (OpenAI, Anthropic) | 30 segons | Generació de text és inherentment lenta |
| Base de dades | 3 segons | Queries optimitzades no haurien de trigar més |

### Què Passa Quan Python No Respon: Cascading Failure

Escenari: El servei Python crida a Java, que triga 60s per un deadlock a la DB.

```
Minut 0: Petició 1 → Python → Java (bloquejat)
Minut 0: Petició 2 → Python → Java (bloquejat)
Minut 0: Petició 3 → Python → Java (bloquejat)
...
Minut 1: Python ha esgotat tots els threads/connexions
Minut 1: Noves peticions → 503 Service Unavailable
Minut 2: El client rep timeouts massius
```

**Amb timeout de 5s:**

```
Segon 0: Petició 1 → Python → Java (esperant)
Segon 5: Python talla la connexió → retorna 504 al client
Segon 5: Thread alliberat → pot servir noves peticions
→ El sistema continua funcionant per a peticions que NO depenen de Java
```

### Demostració: Aturar Java i Observar el Comportament

```bash
# 1. Arrencar ambdós serveis normalment.
# Terminal 1:
cd backend-java && mvn spring-boot:run

# Terminal 2:
cd ai-python && uvicorn src.main:app --reload

# 2. Verificar que funciona:
curl http://localhost:8000/api/champions/1
# → 200 OK amb dades del campió

# 3. ATURAR Java (Ctrl+C al Terminal 1).

# 4. Cridar Python:
curl -v http://localhost:8000/api/champions/1
# Observar als logs de Python:
# Intent 1: ConnectionError → retry (espera 1s)
# Intent 2: ConnectionError → retry (espera 2s)
# Intent 3: ConnectionError → retorna 502 Bad Gateway
# Total: ~3s en lloc de bloqueig infinit
```

### Tests d'Integració: Simular Errors

No podem dependre de "aturar Java manualment" per testejar. Usem **mocks** per simular errors de xarxa.

#### Python: pytest amb `respx` (mock per httpx)

```bash
# respx és el mock oficial per httpx (el client HTTP que usem).
pip install pytest respx pytest-asyncio
```

```python
# test_resilience.py
import pytest
import httpx
import respx
from httpx import Response

# Importem les funcions que volem testejar.
from src.services import get_champion_from_java

# pytest-asyncio ens permet testejar funcions async.
@pytest.mark.asyncio
class TestResilience:

    @respx.mock
    async def test_retry_on_connection_error(self):
        """
        Verifica que el sistema reintenta 3 vegades quan Java no respon.
        Simulem un ConnectionError: Java no està arrencat.
        """
        # Configurem el mock per llençar ConnectionError les 2 primeres vegades
        # i respondre OK la tercera.
        route = respx.get("http://localhost:8080/api/champions/1")
        route.side_effect = [
            httpx.ConnectError("Connection refused"),    # Intent 1: falla
            httpx.ConnectError("Connection refused"),    # Intent 2: falla
            Response(200, json={"name": "Jinx", "role": "ADC"}),  # Intent 3: OK
        ]

        # La funció hauria de retornar les dades del tercer intent.
        result = await get_champion_from_java(1)
        assert result["name"] == "Jinx"
        # Verifiquem que s'han fet exactament 3 crides.
        assert route.call_count == 3

    @respx.mock
    async def test_returns_502_after_max_retries(self):
        """
        Verifica que retorna 502 quan s'esgoten tots els intents.
        Simulem que Java MAI respon.
        """
        # Totes les crides fallen amb ConnectionError.
        respx.get("http://localhost:8080/api/champions/1").mock(
            side_effect=httpx.ConnectError("Connection refused")
        )

        # Esperem que llenci JavaServiceUnavailableError després de 3 intents.
        with pytest.raises(JavaServiceUnavailableError):
            await get_champion_from_java(1)

    @respx.mock
    async def test_no_retry_on_404(self):
        """
        Verifica que NO reintentem quan Java retorna 404.
        Un 404 és un error del client, reintentar no canviarà el resultat.
        """
        route = respx.get("http://localhost:8080/api/champions/999").mock(
            return_value=Response(404, json={
                "type": "https://esportspulse.dev/errors/champion-not-found",
                "title": "Champion No Trobat",
                "status": 404,
                "detail": "Champion amb ID 999 no trobat",
            })
        )

        # Ha de llençar ChampionNotFoundError immediatament, sense retry.
        with pytest.raises(ChampionNotFoundError):
            await get_champion_from_java(999)

        # IMPORTANT: només 1 crida, NO 3. No hem reintentat.
        assert route.call_count == 1

    @respx.mock
    async def test_timeout_triggers_retry(self):
        """
        Verifica que un timeout provoca retry (és un error transitori).
        """
        route = respx.get("http://localhost:8080/api/champions/1")
        route.side_effect = [
            httpx.ReadTimeout("Read timed out"),         # Intent 1: timeout
            Response(200, json={"name": "Jinx"}),        # Intent 2: OK
        ]

        result = await get_champion_from_java(1)
        assert result["name"] == "Jinx"
        assert route.call_count == 2

    @respx.mock
    async def test_problem_details_format(self):
        """
        Verifica que les respostes d'error segueixen el format Problem Details.
        """
        respx.get("http://localhost:8080/api/champions/1").mock(
            side_effect=httpx.ConnectError("Connection refused")
        )

        # Simulem una petició HTTP completa a l'endpoint FastAPI.
        from httpx import AsyncClient, ASGITransport
        from src.main import app

        # ASGITransport permet testejar FastAPI sense arrencar el servidor.
        transport = ASGITransport(app=app)
        async with AsyncClient(transport=transport, base_url="http://test") as client:
            response = await client.get("/api/champions/1")

        # Verifiquem el format Problem Details.
        assert response.status_code == 502
        body = response.json()
        assert "type" in body      # URI del tipus d'error
        assert "title" in body     # Títol per a humans
        assert "status" in body    # Codi HTTP
        assert "detail" in body    # Explicació detallada
```

### Test de Propagació de Correlation ID

```python
@pytest.mark.asyncio
class TestCorrelationId:

    @respx.mock
    async def test_correlation_id_propagated_to_java(self):
        """
        Verifica que el correlation_id del client es propaga a Java.
        Quan Python rep X-Correlation-ID, l'ha d'enviar a Java.
        """
        # Capturem els headers que Python envia a Java.
        captured_headers = {}

        def capture_request(request):
            # Guardem els headers de la petició sortint.
            captured_headers.update(dict(request.headers))
            return Response(200, json={"name": "Jinx"})

        respx.get("http://localhost:8080/api/champions/1").mock(
            side_effect=capture_request
        )

        # Enviem petició a Python amb un correlation_id específic.
        from httpx import AsyncClient, ASGITransport
        from src.main import app

        transport = ASGITransport(app=app)
        async with AsyncClient(transport=transport, base_url="http://test") as client:
            response = await client.get(
                "/api/champions/1",
                headers={"X-Correlation-ID": "test-corr-123"},
            )

        # Verifiquem que Python ha propagat el correlation_id a Java.
        assert captured_headers.get("x-correlation-id") == "test-corr-123"

        # Verifiquem que la resposta conté el correlation_id.
        assert response.headers.get("x-correlation-id") == "test-corr-123"

    @respx.mock
    async def test_generates_correlation_id_if_missing(self):
        """
        Si el client NO envia X-Correlation-ID, Python n'ha de generar un.
        """
        respx.get("http://localhost:8080/api/champions/1").mock(
            return_value=Response(200, json={"name": "Jinx"})
        )

        from httpx import AsyncClient, ASGITransport
        from src.main import app

        transport = ASGITransport(app=app)
        async with AsyncClient(transport=transport, base_url="http://test") as client:
            # NO enviem X-Correlation-ID.
            response = await client.get("/api/champions/1")

        # La resposta HA de tenir un correlation_id generat automàticament.
        corr_id = response.headers.get("x-correlation-id")
        assert corr_id is not None
        # Ha de ser un UUID vàlid (36 caràcters amb guions).
        assert len(corr_id) == 36
```

### Test End-to-End (E2E)

El test E2E verifica tot el flux complet amb els serveis reals arrencat:

```bash
#!/bin/bash
# test_e2e.sh — Test end-to-end complet d'EsportsPulse.
# Requereix: Java i Python arrencat.

echo "=== Test E2E EsportsPulse ==="

# 1. Petició normal amb correlation_id.
echo "--- Test 1: Flux complet ---"
CORR_ID="e2e-test-$(date +%s)"
RESPONSE=$(curl -s -w "\n%{http_code}" \
    -H "X-Correlation-ID: $CORR_ID" \
    http://localhost:8000/api/champions/1)

HTTP_CODE=$(echo "$RESPONSE" | tail -1)
BODY=$(echo "$RESPONSE" | head -1)

if [ "$HTTP_CODE" == "200" ]; then
    echo "PASS: Petició OK (200)"
else
    echo "FAIL: Esperava 200, rebut $HTTP_CODE"
    exit 1
fi

# 2. Verificar que el correlation_id apareix als logs.
echo "--- Test 2: Correlation ID als logs ---"
# Busquem el correlation_id als logs de Python i Java.
sleep 1  # Esperem que els logs s'escriguin.

PYTHON_LOG=$(grep "$CORR_ID" logs/python.json 2>/dev/null | head -1)
JAVA_LOG=$(grep "$CORR_ID" logs/java.json 2>/dev/null | head -1)

if [ -n "$PYTHON_LOG" ] && [ -n "$JAVA_LOG" ]; then
    echo "PASS: Correlation ID trobat als dos serveis"
else
    echo "WARN: Verifica manualment els logs per $CORR_ID"
fi

# 3. Petició a campió inexistent → 404 Problem Details.
echo "--- Test 3: Error 404 Problem Details ---"
RESPONSE=$(curl -s http://localhost:8000/api/champions/99999)
if echo "$RESPONSE" | python3 -c "import sys,json; d=json.load(sys.stdin); assert d['status']==404" 2>/dev/null; then
    echo "PASS: 404 amb Problem Details"
else
    echo "FAIL: Resposta 404 no segueix Problem Details"
fi

# 4. Verificar timeouts configurats.
echo "--- Test 4: Timeout configurat ---"
# Aquesta verificació és estàtica (revisem el codi).
if grep -r "timeout" ai-python/src/ | grep -q "5.0\|30.0"; then
    echo "PASS: Timeouts configurats al codi"
else
    echo "WARN: Verifica que tots els clients HTTP tenen timeout"
fi

echo "=== Tests E2E completats ==="
```

### Afegir Tests Python a GitHub Actions

Actualitza el workflow de CI per executar els tests de Python:

```yaml
# .github/workflows/ci.yml
name: CI EsportsPulse

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # Job existent per Java.
  java-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configurar Java 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Executar tests Java
        run: cd backend-java && mvn test

  # NOU: Job per tests Python.
  python-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configurar Python 3.12
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      # Instal·lem les dependències del projecte.
      - name: Instal·lar dependències
        run: |
          cd ai-python
          pip install -r requirements.txt
          pip install pytest pytest-asyncio respx

      # Executem tots els tests amb pytest.
      # -v: mode verbose per veure cada test individualment.
      # --tb=short: traceback curt per errors (llegible a CI).
      - name: Executar tests Python
        run: |
          cd ai-python
          python -m pytest tests/ -v --tb=short
```

---

## Activitat

### Pas 1: Verificar Timeouts

Revisa tot el codi i assegura't que **cada** crida HTTP té un timeout configurat:

```python
# Buscar crides sense timeout:
# grep -rn "httpx.AsyncClient()" ai-python/src/
# grep -rn "RestTemplate" backend-java/src/

# Totes han de tenir timeout explícit.
```

### Pas 2: Escriure Tests de Resiliència

1. Crea `ai-python/tests/test_resilience.py` amb els tests de la teoria
2. Executa: `cd ai-python && python -m pytest tests/ -v`
3. Tots els tests han de passar

### Pas 3: Escriure Tests de Correlation ID

1. Crea `ai-python/tests/test_correlation.py` amb els tests de propagació
2. Executa i verifica que passen

### Pas 4: Executar Test E2E (Manual)

1. Arrencar Java i Python
2. Executa `bash test_e2e.sh`
3. Verifica que tots els tests passen

### Pas 5: Actualitzar CI

1. Afegeix el job `python-tests` a `.github/workflows/ci.yml`
2. Fes push i verifica que CI passa

### Pas 6: Crear PR

```bash
# Crea una branca amb tot el treball de la setmana.
git checkout -b feature/week11-error-handling

# Afegeix tots els fitxers nous i modificats.
git add .

# Commit amb missatge descriptiu.
git commit -m "feat(week11): error handling, structured logging, correlation IDs, MCP server and resilience tests"

# Puja la branca.
git push -u origin feature/week11-error-handling

# Crea la PR.
gh pr create \
    --title "feat: Week 11 — Error Handling, Logging i Integració" \
    --body "## Canvis
- Gestió centralitzada d'errors amb Problem Details (RFC 7807)
- Structured logging JSON a Java i Python
- Correlation IDs entre serveis
- Servidor MCP amb 3 eines
- Retry amb backoff exponencial
- Tests de resiliència i integració
- CI actualitzat amb tests Python"
```

---

## Checklist de Lliurament

- [ ] Timeout configurat en **totes** les crides HTTP (5s per APIs, 30s per LLMs)
- [ ] Tests de retry: verificat que reintenta per 5xx/ConnectionError i NO per 4xx
- [ ] Tests de timeout: verificat que TimeoutException provoca retry
- [ ] Tests de Problem Details: verificat format RFC 7807 a les respostes d'error
- [ ] Tests de Correlation ID: verificat propagació entre Python i Java
- [ ] Tests de generació automàtica de correlation_id quan el client no n'envia
- [ ] Test E2E executat manualment amb èxit
- [ ] GitHub Actions actualitzat amb job `python-tests`
- [ ] PR creada amb tots els canvis de la setmana
- [ ] Commit amb missatge: `feat(week11): error handling, structured logging, correlation IDs, MCP server and resilience tests`
