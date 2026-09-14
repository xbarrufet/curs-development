# Setmana 9 - Teoria: Error Handling Inter-Serveis, Logging Estructurat i MCP Server Personalitzat

## 1. Errors en Sistemes Distribuïts: Per Què try-catch No és Suficient

Quan tens un sol servei (S1-S6), els errors són locals: una excepció puja pel call stack, la captures, i el programa continua o es para. Tot passa dins del mateix procés, a la mateixa màquina.

```
Servei únic (S1-S6):

┌─────────────────────────┐
│         Java App         │
│                          │
│  Controller              │
│    └─→ Service           │
│          └─→ Repository  │
│                └─→ H2 DB │
│                          │
│  Si falla → Exception    │
│  → catch → error 500     │
│  Tot al mateix procés.   │
└─────────────────────────┘
```

Ara tens dos serveis (S7-S8): Java Spring Boot i Python FastAPI, parlant per HTTP. Els errors ja no són locals — són **remots**:

```
Sistema distribuït (S7+):

              XARXA (pot fallar!)
┌──────────┐     │        ┌──────────┐     │      ┌────┐
│  Client  │────→│────→   │  Python  │────→│───→  │Java│──→ H2
│  (curl)  │     │    X₁  │ FastAPI  │     │  X₃  │ API│
└──────────┘     │        └──────────┘     │      └────┘
                 │              │           │
                 │         X₂  │      X₄   │
                 │       (Python         (Java
                 │        crash)         crash)

X₁ = Client no pot connectar amb Python
X₂ = Python té un bug intern
X₃ = Python no pot connectar amb Java
X₄ = Java té un bug intern
```

**Tipus d'errors remots que no existien abans:**

- **Connection refused:** el servei Java no està arrencat.
- **Timeout:** Java triga massa a respondre (BD lenta, LLM lent).
- **Partial failure:** Java respon amb 500 però la dada s'ha escrit a la BD.
- **Network partition:** la xarxa entre Python i Java cau temporalment.
- **Cascading failure:** Java cau → Python es queda esperant → Python acumula requests → Python cau → els clients es queden sense res.

Cap d'aquests errors es pot capturar amb un `try-catch` tradicional que espera una excepció síncrona. Necessites **estratègies**:  timeout, retry, circuit breaker, fallback.

> El teu codi perfecte depèn de codi que no controles.

---

## 2. Timeout: La Primera Línia de Defensa

### El perill de no tenir timeout

Sense timeout, una crida HTTP espera **indefinidament**. Què passa?

```
Sense timeout:

Thread-1 ──→ requests.get(java_api) ──→ esperant... esperant... esperant...
Thread-2 ──→ requests.get(java_api) ──→ esperant... esperant... esperant...
Thread-3 ──→ requests.get(java_api) ──→ esperant... esperant... esperant...
...
Thread-N ──→ requests.get(java_api) ──→ esperant... esperant... esperant...

→ Thread pool exhaurit
→ El teu servei Python ja no pot acceptar cap request nova
→ El client rep... res. Ni error ni resposta.
→ Has mort sense saber-ho.
```

### Configurar timeout a Python

```python
import requests

# MAI fer això:
response = requests.get(f"{JAVA_API_URL}/champions/{champion_id}")  # timeout = infinit!

# SEMPRE fer això:
response = requests.get(
    f"{JAVA_API_URL}/champions/{champion_id}",
    timeout=5  # connect + read timeout en segons
)

# O separar connect i read:
response = requests.get(
    f"{JAVA_API_URL}/champions/{champion_id}",
    timeout=(3, 5)  # connect=3s, read=5s
)
```

### Configurar timeout a Java

```java
@Configuration
public class RestClientConfig {

    @Bean
    public RestTemplate restTemplate() {
        var factory = new SimpleClientHttpRequestFactory();
        factory.setConnectTimeout(Duration.ofSeconds(3));
        factory.setReadTimeout(Duration.ofSeconds(5));
        return new RestTemplate(factory);
    }
}
```

### Quins valors de timeout?

| Tipus de crida          | Timeout recomanat |
|-------------------------|-------------------|
| API REST interna        | 3-5 segons        |
| Base de dades           | 5-10 segons       |
| Crida a LLM (Claude)   | 30-60 segons      |
| Servei extern (3rd party) | 10-15 segons    |

Regla: si no saps quin timeout posar, **5 segons** és un bon punt de partida per APIs REST. Si necessites més, és que l'arquitectura té un problema que el timeout no resoldrà.

---

## 3. Retry amb Backoff Exponencial

### Quan fer retry — i quan NO

No tots els errors mereixen un retry. La distinció és simple:

| Status Code | Significat                | Retry? | Per què                                     |
|-------------|---------------------------|--------|---------------------------------------------|
| 400         | Bad Request               | No     | La request és incorrecta, repetir-la no canviarà res |
| 401/403     | Auth error                | No     | No tens permisos, repetir-la no en donarà   |
| 404         | Not found                 | No     | El recurs no existeix, repetir no el crearà |
| 409         | Conflict                  | No     | Conflicte de dades, cal intervenció         |
| 422         | Validation error          | No     | Dades invàlides, cal corregir-les           |
| 429         | Rate limited              | Si     | Massa requests, espera i torna a provar     |
| 500         | Server error              | Si     | Pot ser un error transitori                 |
| 502         | Bad gateway               | Si     | L'upstream pot recuperar-se                 |
| 503         | Service unavailable       | Si     | El servei pot tornar                        |
| 504         | Gateway timeout           | Si     | Pot ser un pic de latència temporal         |
| Connection  | Connection error/timeout  | Si     | Pot ser un problema de xarxa transitori     |

**Regla:** retry errors del servidor (5xx) i de xarxa. Mai retry errors del client (4xx excepte 429).

### Backoff exponencial

Si el servei Java està sobrecarregat i tu fas retry cada segon, estàs empitjorant el problema. Backoff exponencial: cada retry espera el doble que l'anterior.

```
Intent 1: falla → espera 1s
Intent 2: falla → espera 2s
Intent 3: falla → espera 4s
Intent 4: falla → espera 8s
Intent 5: falla → GIVE UP, retorna error al client

Timeline:
0s        1s        3s        7s        15s
|─────────|─────────|─────────|─────────|
try1      try2      try3      try4      try5
  fail      fail      fail      fail     GIVE UP
```

### Jitter: evitar el thundering herd

Si 100 clients fan retry tots al segon exacte, el servidor rep 100 requests simultànies just quan estava intentant recuperar-se. Afegim **jitter** — un random petit:

```
Sense jitter: tots retry a 1.000s, 2.000s, 4.000s (thundering herd)
Amb jitter:   retry a 0.8s-1.2s, 1.6s-2.4s, 3.2s-4.8s (distribuït)
```

### Implementació a Python (manual)

```python
import time
import random
import requests

def call_java_api_with_retry(champion_id: str, max_retries: int = 3) -> dict:
    """Crida l'API Java amb retry i backoff exponencial."""
    for attempt in range(max_retries):
        try:
            response = requests.get(
                f"{JAVA_API_URL}/champions/{champion_id}",
                timeout=5
            )
            response.raise_for_status()
            return response.json()
        except (requests.ConnectionError, requests.Timeout) as e:
            if attempt == max_retries - 1:
                raise  # Últim intent, propagar l'error
            wait = (2 ** attempt) + random.uniform(0, 1)  # backoff + jitter
            logger.warning(
                "retry_scheduled",
                champion_id=champion_id,
                attempt=attempt + 1,
                wait_seconds=round(wait, 2),
                error=str(e)
            )
            time.sleep(wait)
        except requests.HTTPError as e:
            if e.response.status_code < 500:
                raise  # 4xx: no retry
            if attempt == max_retries - 1:
                raise
            wait = (2 ** attempt) + random.uniform(0, 1)
            time.sleep(wait)
```

### Implementació a Python (tenacity)

```python
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential_jitter,
    retry_if_exception_type,
)
import requests

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential_jitter(initial=1, max=10, jitter=2),
    retry=retry_if_exception_type((requests.ConnectionError, requests.Timeout)),
)
def call_java_api(champion_id: str) -> dict:
    """Crida l'API Java. tenacity gestiona retry i backoff."""
    response = requests.get(
        f"{JAVA_API_URL}/champions/{champion_id}",
        timeout=5
    )
    response.raise_for_status()
    return response.json()
```

### Implementació a Java (Spring Retry)

```java
// pom.xml: spring-retry + spring-boot-starter-aop
@Service
public class PythonApiClient {

    private final RestTemplate restTemplate;

    @Retryable(
        retryFor = {ResourceAccessException.class, HttpServerErrorException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2, maxDelay = 8000)
    )
    public ChampionAnalysis fetchAnalysis(String championId) {
        return restTemplate.getForObject(
            pythonApiUrl + "/analyze/" + championId,
            ChampionAnalysis.class
        );
    }

    @Recover
    public ChampionAnalysis fallback(Exception e, String championId) {
        logger.error("All retries failed for champion {}", championId, e);
        throw new ExternalServiceUnavailableException(
            "Python analysis service unavailable after 3 retries"
        );
    }
}
```

---

## 4. Circuit Breaker (Conceptual)

### El problema que retry no resol

Retry està bé per errors transitoris (un pic de latència, un reinici ràpid). Però si el servei Java està **realment** mort (hardware failure, desplegament malament, BD corrupta), cada retry és un malbaratament:

```
Sense circuit breaker:

Request 1 → retry 3x → falla (15s perduts)
Request 2 → retry 3x → falla (15s perduts)
Request 3 → retry 3x → falla (15s perduts)
...
Request 100 → retry 3x → falla (15s perduts)

Has gastat 1500 segons de CPU intentant parlar amb un mort.
```

### El patró Circuit Breaker

Funciona com un interruptor elèctric: si detecta massa errors, **talla el circuit** i ni intenta la crida.

```
                    massa errors (> 5 en 60s)
         ┌──────────────────────────────────────┐
         │                                      v
    ┌─────────┐                           ┌─────────┐
    │ CLOSED  │                           │  OPEN   │
    │ (normal)│                           │ (tall)  │
    └─────────┘                           └─────────┘
         ^                                      │
         │            timer expira (60s)        │
         │                                      v
         │                              ┌────────────┐
         │         1 request OK         │ HALF-OPEN  │
         └──────────────────────────────│ (prova 1)  │
                                        └────────────┘
                    1 request FALLA            │
                    ┌──────────────────────────┘
                    v
              ┌─────────┐
              │  OPEN   │ (torna a esperar)
              └─────────┘
```

**Estats:**

- **CLOSED (normal):** les requests passen amb normalitat. Es compten els errors.
- **OPEN (tall):** si els errors superen un llindar (p.ex. 5 errors en 60 segons), el circuit s'obre. Totes les requests retornen immediatament amb error 503, sense intentar la crida. Això protegeix el teu servei i dona temps al servei caigut per recuperar-se.
- **HALF-OPEN (prova):** després d'un temps d'espera, es deixa passar **una sola** request de prova. Si funciona, el circuit torna a CLOSED. Si falla, torna a OPEN.

### Per què no l'implementem ara?

Aquesta setmana no implementem un circuit breaker complet (ho farem com a exercici avançat). Però és important entendre per què existeix:

- **Retry** protegeix contra errors transitoris.
- **Circuit breaker** protegeix contra fallades prolongades.
- Junts formen la base de la **resiliència** en sistemes distribuïts.

Eines reals: Resilience4j (Java), pybreaker (Python), Istio/Envoy (a nivell de xarxa).

---

## 5. HTTP Status Codes Semàntics

Quan el teu servei Python actua com a proxy de Java, els status codes han de reflectir **on** ha passat el problema:

| Code | Significat            | Exemple EsportsPulse                                              |
|------|-----------------------|----------------------------------------------------------------|
| 400  | Bad Request           | `POST /champions` amb JSON malformat o camp `name` buit           |
| 404  | Not Found             | `GET /champions/Zzzrot` — el campió no existeix a la BD              |
| 409  | Conflict              | `POST /champions` amb un `championId` que ja existeix                   |
| 422  | Unprocessable Entity  | `POST /champions` amb `winRate: 150.0` — format correcte, valor invàlid |
| 500  | Internal Server Error | Bug al teu codi Python (NullPointer, TypeError)                |
| 502  | Bad Gateway           | Java ha retornat un error 500 (el problema és upstream)        |
| 503  | Service Unavailable   | El teu servei està sobrecarregat o en mode manteniment         |
| 504  | Gateway Timeout       | Java no ha respost dins del timeout de 5 segons                |

### La regla d'or dels 5xx

```
Si el teu servei retorna 500 quan l'upstream falla, estàs mentint — és un 502.
Si el teu servei retorna 500 quan l'upstream no respon, estàs mentint — és un 504.
500 vol dir "JO he petat". 502/504 vol dir "UN ALTRE ha petat/no respon".
```

Aquesta distinció és crucial per al debugging: si veus un 502 als logs, saps que has d'anar a mirar el servei upstream. Si veus un 500, el bug és teu.

---

## 6. RFC 7807: Problem Details

### Per què un format estàndard d'error?

Sense estàndard, cada API inventa el seu format:

```json
// API A:
{"error": "not found"}

// API B:
{"message": "Champion not found", "code": 404}

// API C:
{"errors": [{"field": "name", "msg": "required"}]}
```

El client ha d'escriure lògica de parsing diferent per cada API. RFC 7807 defineix un format únic:

### Estructura Problem Details

```json
{
    "type": "https://esportspulse.dev/errors/champion-not-found",
    "title": "Champion Not Found",
    "status": 404,
    "detail": "No champion found with championId Zzzrot",
    "instance": "/champions/Zzzrot"
}
```

| Camp       | Descripció                                          | Obligatori |
|------------|-----------------------------------------------------|------------|
| `type`     | URI que identifica el tipus d'error (pot ser about:blank) | Si         |
| `title`    | Resum curt i llegible per humans                    | Si         |
| `status`   | HTTP status code                                    | Si         |
| `detail`   | Explicació detallada d'aquest error concret         | No         |
| `instance` | URI de la request que ha causat l'error             | No         |

Pots afegir camps extra:

```json
{
    "type": "https://esportspulse.dev/errors/validation-failed",
    "title": "Validation Failed",
    "status": 422,
    "detail": "One or more fields failed validation",
    "instance": "/champions",
    "errors": [
        {"field": "name", "message": "must not be blank"},
        {"field": "winRate", "message": "must be between 0 and 100"}
    ]
}
```

### Implementació a Java (Spring Boot 3)

Spring Boot 3 té suport natiu per Problem Details:

```java
@ControllerAdvice
public class EsportsPulseExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(ChampionNotFoundException.class)
    public ProblemDetail handleChampionNotFound(ChampionNotFoundException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND,
            ex.getMessage()
        );
        problem.setTitle("Champion Not Found");
        problem.setType(URI.create("https://esportspulse.dev/errors/champion-not-found"));
        problem.setProperty("championId", ex.getChampionId());
        return problem;
    }

    @ExceptionHandler(ExternalServiceUnavailableException.class)
    public ProblemDetail handleExternalServiceDown(ExternalServiceUnavailableException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.BAD_GATEWAY,
            ex.getMessage()
        );
        problem.setTitle("External Service Unavailable");
        problem.setType(URI.create("https://esportspulse.dev/errors/external-service-unavailable"));
        return problem;
    }
}
```

### Implementació a Python (FastAPI)

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

class ChampionNotFoundException(Exception):
    def __init__(self, champion_id: str):
        self.champion_id = champion_id

class ExternalServiceUnavailableException(Exception):
    def __init__(self, service: str, detail: str):
        self.service = service
        self.detail = detail

app = FastAPI()

@app.exception_handler(ChampionNotFoundException)
async def champion_not_found_handler(request: Request, exc: ChampionNotFoundException):
    return JSONResponse(
        status_code=404,
        content={
            "type": "https://esportspulse.dev/errors/champion-not-found",
            "title": "Champion Not Found",
            "status": 404,
            "detail": f"No champion found with championId {exc.champion_id}",
            "instance": str(request.url),
            "championId": exc.champion_id,
        },
        media_type="application/problem+json",
    )

@app.exception_handler(ExternalServiceUnavailableException)
async def external_service_handler(request: Request, exc: ExternalServiceUnavailableException):
    return JSONResponse(
        status_code=502,
        content={
            "type": "https://esportspulse.dev/errors/external-service-unavailable",
            "title": "External Service Unavailable",
            "status": 502,
            "detail": exc.detail,
            "instance": str(request.url),
            "service": exc.service,
        },
        media_type="application/problem+json",
    )
```

---

## 7. Logging Estructurat: Per Què JSON > Text

### El problema del log de text

Un log de text típic:

```
2024-03-15 10:23:45.123 ERROR c.g.ChampionService - Failed to fetch champion Yasuo from Java API: Connection refused
2024-03-15 10:23:45.456 WARN  c.g.RetryHandler - Retrying request to Java API, attempt 2/3
2024-03-15 10:23:47.789 ERROR c.g.ChampionService - Failed to fetch champion Yasuo from Java API: Connection refused
2024-03-15 10:23:47.801 ERROR c.g.ChampionController - Request failed for /champions/Yasuo: External service unavailable
```

Ara imagina que tens 50.000 requests per hora i el PM et diu: "el client amb IP 10.0.1.42 ha tingut un error fa 20 minuts, quin?". Bona sort fent grep per trobar-ho.

### La solució: logs JSON

```json
{"timestamp":"2024-03-15T10:23:45.123Z","level":"ERROR","service":"esportspulse-python","correlation_id":"f47ac10b-58cc","champion_id":"Yasuo","error_type":"ConnectionError","message":"champion_fetch_failed","target_service":"esportspulse-java","client_ip":"10.0.1.42"}
{"timestamp":"2024-03-15T10:23:45.456Z","level":"WARN","service":"esportspulse-python","correlation_id":"f47ac10b-58cc","attempt":2,"max_attempts":3,"message":"retry_scheduled","wait_seconds":2.3}
{"timestamp":"2024-03-15T10:23:47.789Z","level":"ERROR","service":"esportspulse-python","correlation_id":"f47ac10b-58cc","champion_id":"Yasuo","error_type":"ConnectionError","message":"champion_fetch_failed","target_service":"esportspulse-java"}
{"timestamp":"2024-03-15T10:23:47.801Z","level":"ERROR","service":"esportspulse-python","correlation_id":"f47ac10b-58cc","path":"/champions/Yasuo","status_code":502,"message":"request_completed"}
```

Ara pots:

```bash
# Trobar tots els logs d'un client concret:
cat logs.json | jq 'select(.client_ip == "10.0.1.42")'

# Trobar tots els errors d'un correlation_id:
cat logs.json | jq 'select(.correlation_id == "f47ac10b-58cc")'

# Comptar errors per servei en l'última hora:
cat logs.json | jq 'select(.level == "ERROR") | .service' | sort | uniq -c

# Trobar els champion_id que més fallen:
cat logs.json | jq 'select(.message == "champion_fetch_failed") | .champion_id' | sort | uniq -c | sort -rn
```

### Comparativa

| Criteri                    | Text pla                          | JSON estructurat                    |
|----------------------------|------------------------------------|------------------------------------|
| Llegibilitat per humans    | Bona (una línia = un event)       | Acceptable (menys intuïtiu)        |
| Cerca per camp             | `grep` amb regex (fràgil)         | `jq` amb camps tipats (robust)     |
| Filtrar per correlation_id | Molt difícil si no tens format fix | `jq '.correlation_id == "abc"'`    |
| Eines d'agregació          | Cal parsejar primer               | Datadog, ELK, Grafana Loki natiu  |
| Afegir context             | Concatenar strings                | Afegir un camp al dict             |
| Alerting automàtic         | Regex fràgils                     | Queries estructurades              |

---

## 8. Correlation IDs: Traçar una Request

### El problema

Una request pot travessar múltiples serveis. Sense un identificador comú, és impossible connectar els logs:

```
Python logs:
  10:23:45 ERROR - Failed to call Java API
  10:23:45 ERROR - Failed to call Java API    ← Quin és quin?
  10:23:46 ERROR - Failed to call Java API

Java logs:
  10:23:45 ERROR - Database query failed
  10:23:45 ERROR - NullPointerException at ChampionService.java:42  ← De quina request?
```

### La solució: Correlation ID

Un UUID generat al punt d'entrada que acompanya la request a tot arreu:

```
Request flow amb correlation_id:

┌────────┐    X-Correlation-ID: abc-123    ┌────────┐    X-Correlation-ID: abc-123    ┌────┐
│ Client │ ──────────────────────────────→ │ Python │ ──────────────────────────────→ │Java│
│ (curl) │                                 │FastAPI │                                 │ API│
└────────┘                                 └────────┘                                 └────┘
                                                │                                       │
                                          LOG:                                    LOG:
                                          correlation_id=abc-123           correlation_id=abc-123
                                          message=request_received         message=champion_fetched
                                          path=/champions/Jinx                champion_id=Jinx

                                          LOG:                                    LOG:
                                          correlation_id=abc-123           correlation_id=abc-123
                                          message=java_api_called          message=db_query_executed
                                          target=/champions/Jinx              query_time_ms=12

                                          LOG:
                                          correlation_id=abc-123
                                          message=request_completed
                                          status=200
                                          duration_ms=145
```

Amb el `correlation_id=abc-123` pots trobar **tota** la traça als dos serveis amb una sola query.

### Implementació a Python (FastAPI middleware)

```python
import uuid
import structlog
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request

class CorrelationIdMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Llegir o generar correlation_id
        correlation_id = request.headers.get(
            "X-Correlation-ID",
            str(uuid.uuid4())
        )

        # Bind al context de structlog (disponible a tot el request)
        structlog.contextvars.clear_contextvars()
        structlog.contextvars.bind_contextvars(
            correlation_id=correlation_id,
            service="esportspulse-python",
        )

        logger = structlog.get_logger()
        logger.info("request_received", method=request.method, path=str(request.url.path))

        response = await call_next(request)

        # Afegir correlation_id a la response (perquè el client el pugui veure)
        response.headers["X-Correlation-ID"] = correlation_id

        logger.info(
            "request_completed",
            status_code=response.status_code,
        )
        return response

# Quan Python crida Java, propagar el correlation_id:
def call_java_api(champion_id: str, correlation_id: str) -> dict:
    response = requests.get(
        f"{JAVA_API_URL}/champions/{champion_id}",
        headers={"X-Correlation-ID": correlation_id},
        timeout=5,
    )
    response.raise_for_status()
    return response.json()
```

### Implementació a Java (Spring filter)

```java
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {

    private static final String HEADER = "X-Correlation-ID";

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain chain) throws ServletException, IOException {

        String correlationId = request.getHeader(HEADER);
        if (correlationId == null || correlationId.isBlank()) {
            correlationId = UUID.randomUUID().toString();
        }

        // Afegir al MDC (SLF4J Mapped Diagnostic Context)
        MDC.put("correlation_id", correlationId);
        response.setHeader(HEADER, correlationId);

        try {
            chain.doFilter(request, response);
        } finally {
            MDC.clear();  // Netejar per evitar memory leaks
        }
    }
}
```

El MDC fa que el `correlation_id` aparegui automàticament a tots els logs dins d'aquell request, sense haver-lo de passar manualment a cada `logger.info()`.

> Quan el PM et diu "el client X ha tingut un error fa 10 minuts", el correlation_id et porta directament als logs.

---

## 9. structlog (Python) + SLF4J/Logback (Java)

### Configuració de structlog (Python)

```python
import structlog
import logging

def configure_logging():
    """Configurar structlog per emetre JSON."""
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,    # Agafa correlation_id del context
            structlog.processors.add_log_level,          # Afegeix camp "level"
            structlog.processors.TimeStamper(fmt="iso"), # Afegeix camp "timestamp"
            structlog.processors.StackInfoRenderer(),    # Stack traces si cal
            structlog.processors.format_exc_info,        # Formatar excepcions
            structlog.processors.JSONRenderer(),         # Output final: JSON
        ],
        wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(),
        cache_logger_on_first_use=True,
    )
```

Ús:

```python
logger = structlog.get_logger()

# Log simple
logger.info("server_started", port=8000, environment="development")
# → {"timestamp":"2024-03-15T10:00:00Z","level":"info","service":"esportspulse-python","event":"server_started","port":8000,"environment":"development"}

# Log amb error
try:
    champion = call_java_api(champion_id)
except requests.Timeout as e:
    logger.error("java_api_timeout", champion_id=champion_id, timeout_seconds=5)
    # → {"timestamp":"...","level":"error","correlation_id":"abc-123","event":"java_api_timeout","champion_id":"Jinx","timeout_seconds":5}

# Log amb context addicional
logger.info(
    "champion_analysis_completed",
    champion_id=champion_id,
    model="claude-sonnet-4-20250514",
    tokens_used=1523,
    duration_ms=3200,
)
```

### Configuració de Logback (Java)

Primer, afegir la dependència al `pom.xml`:

```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

Configurar `src/main/resources/logback-spring.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"service":"esportspulse-java"}</customFields>
            <includeMdcKeyName>correlation_id</includeMdcKeyName>
        </encoder>
    </appender>

    <!-- Per desenvolupament local: text pla (més llegible) -->
    <springProfile name="dev">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} [%X{correlation_id}] - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>

    <!-- Per producció/CI: JSON -->
    <springProfile name="!dev">
        <root level="INFO">
            <appender-ref ref="JSON"/>
        </root>
    </springProfile>
</configuration>
```

Ús a Java:

```java
@Service
public class ChampionManagementService {
    private static final Logger logger = LoggerFactory.getLogger(ChampionManagementService.class);

    public ChampionDTO findChampion(String championId) {
        logger.info("Fetching champion from database", kv("champion_id",championId));

        return championRepository.findByChampionId(championId)
            .map(champion -> {
                logger.info("Champion found", kv("champion_id",championId), kv("name", champion.getName()));
                return mapper.toDTO(champion);
            })
            .orElseThrow(() -> {
                logger.warn("Champion not found", kv("champion_id",championId));
                return new ChampionNotFoundException(championId);
            });
    }
}
```

Output JSON:

```json
{"@timestamp":"2024-03-15T10:23:45.123Z","level":"INFO","logger_name":"c.g.ChampionManagementService","message":"Champion found","service":"esportspulse-java","correlation_id":"abc-123","champion_id":"Jinx","name":"Jinx"}
```

---

## 10. MCP Server Personalitzat: De Consumidor a Creador

### El canvi de perspectiva

A S8 vas connectar MCP servers que altres havien creat (filesystem, brave-search). Ara **tu** ets el creador. El teu MCP server és la interfície entre un LLM (via Cursor) i el teu sistema EsportsPulse.

```
S8: Consumidor                         S9: Creador

┌────────┐    ┌──────────────┐         ┌────────┐    ┌──────────────────┐    ┌──────────┐
│ Cursor │───→│ MCP Server   │         │ Cursor │───→│ EsportsPulse MCP    │───→│ EsportsPulse│
│        │    │ (d'altri)    │         │        │    │ Server (teu!)    │    │ REST API │
└────────┘    └──────────────┘         └────────┘    └──────────────────┘    └──────────┘
                                                            │ stdio              │ HTTP
                                                            │                    │
                                                       Python script      Java Spring Boot
```

### Arquitectura del MCP server

```
┌─────────────────────────────────────────────────┐
│                  Cursor (IDE)                     │
│  "Quin és el campió més jugat?"                   │
└────────────────────┬────────────────────────────┘
                     │ stdio (JSON-RPC)
                     v
┌─────────────────────────────────────────────────┐
│            esportspulse_mcp_server.py                │
│                                                   │
│  FastMCP("esportspulse")                            │
│                                                   │
│  @mcp.tool()                                     │
│  ├── get_champion(champion_id) → ChampionRecord          │
│  ├── search_champions(query, limit) → list           │
│  ├── get_champion_analysis(champion_id) → Analysis       │
│  └── get_champion_stats() → Stats                  │
│                                                   │
│  @mcp.resource("esportspulse://champions/top10")        │
│  └── top10_champions() → str                         │
│                                                   │
│  @mcp.prompt()                                   │
│  └── analyze_champion(champion_id) → str                 │
└────────────────────┬────────────────────────────┘
                     │ HTTP (requests)
                     v
┌─────────────────────────────────────────────────┐
│         EsportsPulse REST API (Java, S7)            │
│         http://localhost:8080/champions/...           │
└─────────────────────────────────────────────────┘
```

### Codi complet del MCP server

```python
"""EsportsPulse MCP Server — exposa EsportsPulse com a tools per a LLMs."""
import requests
from mcp.server.fastmcp import FastMCP

JAVA_API_URL = "http://localhost:8080"

mcp = FastMCP(
    "esportspulse",
    description="Accés a les dades i anàlisis de EsportsPulse (plataforma d'anàlisi d'eSports - League of Legends)",
)

@mcp.tool()
async def get_champion(champion_id: str) -> dict:
    """Obté les dades completes d'un campió per ID (nom del campió).

    Retorna: nom, winRate, partides jugades.
    Exemple: get_champion("Jinx") → dades de Jinx.
    """
    response = requests.get(f"{JAVA_API_URL}/champions/{champion_id}", timeout=5)
    response.raise_for_status()
    return response.json()


@mcp.tool()
async def search_champions(query: str, limit: int = 10) -> list[dict]:
    """Cerca campions per nom (cerca parcial, case-insensitive).

    Args:
        query: Text a buscar al nom del campió.
        limit: Nombre màxim de resultats (per defecte 10).

    Retorna: Llista de campions que coincideixen.
    """
    response = requests.get(
        f"{JAVA_API_URL}/champions",
        params={"name": query, "size": limit},
        timeout=5,
    )
    response.raise_for_status()
    return response.json()


@mcp.tool()
async def get_champion_analysis(champion_id: str) -> dict:
    """Analitza les estadístiques d'un campió usant un LLM.

    Crida el servei d'anàlisi Python (FastAPI) que al seu torn
    usa Claude/OpenAI per generar l'anàlisi (configurat a S8).

    Pot trigar 10-30 segons per la crida al LLM.
    """
    response = requests.post(
        f"http://localhost:8001/analyze/{champion_id}",
        timeout=30,
    )
    response.raise_for_status()
    return response.json()


@mcp.tool()
async def get_champion_stats() -> dict:
    """Retorna estadístiques agregades de tots els campions:
    total de campions, mitjana de partides jugades, campió més jugat.
    """
    response = requests.get(f"{JAVA_API_URL}/champions", timeout=5)
    response.raise_for_status()
    champions = response.json()
    if not champions:
        return {"totalChampions": 0, "averagePlayers": 0, "mostPopular": None}

    total = len(champions)
    avg_players = sum(g["gamesPlayed"] for g in champions) / total
    top_champion = max(champions, key=lambda g: g["gamesPlayed"])
    return {
        "totalChampions": total,
        "averagePlayers": round(avg_players),
        "mostPopular": top_champion["name"],
    }


@mcp.resource("esportspulse://champions/top10")
async def top10_champions() -> str:
    """Els 10 campions més jugats per partides."""
    response = requests.get(
        f"{JAVA_API_URL}/champions",
        params={"sort": "gamesPlayed,desc", "size": 10},
        timeout=5,
    )
    response.raise_for_status()
    champions = response.json()
    lines = [f"{i+1}. {g['name']} — {g['gamesPlayed']:,} partides"
             for i, g in enumerate(champions)]
    return "\n".join(lines)


@mcp.prompt()
async def analyze_champion(champion_id: str) -> str:
    """Prompt pre-definit per analitzar les estadístiques d'un campió."""
    return f"""Analitza les estadístiques del campió amb ID {champion_id}.

Pas 1: Usa el tool get_champion per obtenir les dades bàsiques.
Pas 2: Usa el tool get_champion_analysis per obtenir l'anàlisi detallada.
Pas 3: Presenta els resultats amb:
  - Dades bàsiques del campió
  - Punts forts del campió
  - Punts febles o àrees de millora
  - Recomanació general"""


if __name__ == "__main__":
    mcp.run()
```

### Tools vs Resources vs Prompts

| Element    | Què és                              | Quan usar-lo                                    |
|------------|--------------------------------------|-------------------------------------------------|
| **Tool**   | Funció que el model pot cridar       | Quan cal una acció (get, search, create, analyze) |
| **Resource** | Dada estàtica o semi-estàtica      | Quan el model necessita context (dashboard, resum) |
| **Prompt** | Template de prompt pre-definit       | Quan vols guiar el model en tasques complexes    |

### Les descripcions dels tools són crucials

El model decideix quin tool cridar basant-se en la **descripció**. Si la descripció és vaga, el model no sabrà quan usar-lo:

```python
# Dolenta — el model no sap quan cridar-la
@mcp.tool()
async def get_data(id: str) -> dict:
    """Gets data."""

# Bona — el model entén exactament què fa i quan usar-la
@mcp.tool()
async def get_champion(champion_id: str) -> dict:
    """Obté les dades completes d'un campió per ID (nom del campió).
    Retorna: nom, winRate, partides jugades.
    Exemple: get_champion("Jinx") → dades de Jinx."""
```

Això connecta directament amb el que véiem a S8 sobre tool_use: el model necessita bons noms i bones descripcions per decidir quina eina usar.

---

## 11. MCP com a Spec Executable

### La idea clau

El conjunt de tools, resources i prompts del teu MCP server és, de facto, una **especificació** del que el teu sistema pot fer. Quan un LLM es connecta al teu MCP server, el que veu és:

```
Tools disponibles:
  - get_champion(champion_id: str) → dict
  - search_champions(query: str, limit: int) → list[dict]
  - get_champion_analysis(champion_id: str) → dict
  - get_champion_stats() → dict

Resources disponibles:
  - esportspulse://champions/top10

Prompts disponibles:
  - analyze_champion(champion_id: str)
```

Això és una spec. I a diferència d'un document Markdown, aquesta spec **s'executa**. Si canvies un tool, el comportament canvia immediatament.

### Iterar la spec

Quan l'LLM no fa el que esperes, el problema pot ser:

1. **Descripció del tool** — poc clara o ambigua.
2. **Nom del tool** — confús (`get_data` vs `get_champion`).
3. **Paràmetres** — el model no sap quin valor passar.
4. **Falta un tool** — la pregunta de l'usuari no encaixa amb cap tool existent.
5. **Falta context** — el model no té prou informació per decidir.

La solució no és millorar el prompt de l'usuari — és millorar el **server**:

```
Pregunta: "Quins campions tenen winRate superior al 55%?"
Model: crida search_champions(query="55")  ← No funciona, search_champions busca per nom

Solució: afegir un nou tool
@mcp.tool()
async def search_high_winrate_champions(max_results: int = 10) -> list[dict]:
    """Cerca campions amb winRate > 55%. Retorna fins a max_results."""
    ...
```

Aquesta iteració — pregunta real → el model falla → millorar el server → tornar a provar — és el workflow de **spec-driven development** que veurem amb més profunditat a S18.

### Connexió amb S8

A S8 vas veure com l'LLM decideix cridar tools (`tool_use`). Ara entens el costat contrari: tu defineixes les tools que l'LLM pot cridar. La qualitat de les teves definicions determina la qualitat de les respostes del model.

```
S8: Entens com l'LLM crida tools
    (tool_use, function calling)
         │
         v
S9: Tu defineixes les tools
    (MCP server, @mcp.tool)
         │
         v
S18: Spec-driven development complet
     (la spec governa tot el sistema)
```

---

## Resum

| Concepte                | Takeaway                                                                    |
|-------------------------|-----------------------------------------------------------------------------|
| Errors distribuïts      | try-catch no és suficient — necessites timeout, retry, circuit breaker     |
| Timeout                 | Mai cridar un servei extern sense timeout. 5s per APIs, 30s per LLMs      |
| Retry + backoff         | Retry 5xx i errors de xarxa. Mai retry 4xx. Backoff exponencial + jitter  |
| Circuit breaker         | Si l'upstream està mort, deixa de provar-ho. CLOSED → OPEN → HALF-OPEN   |
| HTTP status codes       | 500 = bug teu. 502 = upstream ha petat. 504 = upstream no respon          |
| RFC 7807                | Format estàndard d'error: type, title, status, detail, instance           |
| Logging estructurat     | JSON, no text. Camps tipats. Permet cerca, agregació i alerting           |
| Correlation ID          | UUID propagat entre serveis. Traça completa amb una sola query            |
| structlog + Logback     | Python: structlog amb JSONRenderer. Java: Logback amb LogstashEncoder     |
| MCP server              | Tu defineixes les tools — la qualitat de les descripcions = qualitat respostes |
| MCP com a spec          | El server és una spec executable. Itera el server, no el prompt           |
