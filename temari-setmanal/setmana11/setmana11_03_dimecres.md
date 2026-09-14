# Setmana 11 — Dimecres: Correlation IDs Entre Serveis Java i Python

## Objectiu del Dia

Implementar **Correlation IDs** per traçar una petició des del client fins a Java i Python, i afegir **retry amb backoff exponencial** per gestionar errors transitoris. Al final del dia, podràs buscar un `correlation_id` als logs i veure tot el recorregut de la petició entre serveis.

---

## Teoria

### El Problema: Traçar Peticions Entre Serveis

Imagina que un usuari reporta: "He demanat les estadístiques de Jinx i m'ha donat error". Obres els logs i veus:

```json
{"service_name": "ai-python", "level": "error", "message": "Error cridant Java"}
{"service_name": "ai-python", "level": "error", "message": "Error cridant Java"}
{"service_name": "ai-python", "level": "error", "message": "Error cridant Java"}
{"service_name": "backend-java", "level": "error", "message": "NullPointerException"}
{"service_name": "backend-java", "level": "info", "message": "Champion carregat"}
```

**Quina línia de Java correspon a quina de Python?** Sense un identificador comú, és impossible saber-ho.

### Solució: X-Correlation-ID

Cada petició rep un **UUID únic** que viatja entre tots els serveis:

```
Client → [X-Correlation-ID: abc-123] → Python → [X-Correlation-ID: abc-123] → Java
```

```json
{"service_name": "ai-python", "correlation_id": "abc-123", "message": "Petició rebuda"}
{"service_name": "backend-java", "correlation_id": "abc-123", "message": "Champion carregat"}
{"service_name": "ai-python", "correlation_id": "abc-123", "message": "Resposta enviada"}
```

Ara pots buscar `correlation_id == "abc-123"` i veure **tota la traça**, en ordre, entre serveis.

### FastAPI Middleware: Generar i Propagar Correlation ID

Un **middleware** intercepta TOTES les peticions abans d'arribar als endpoints:

```python
import uuid
import structlog
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request

# Variable de context per compartir el correlation_id dins de la petició.
# contextvars és thread-safe i async-safe (funciona amb asyncio).
from contextvars import ContextVar

# ContextVar: cada petició concurrent té el seu propi valor.
# Sense això, peticions concurrents compartirien el mateix correlation_id.
correlation_id_ctx: ContextVar[str] = ContextVar("correlation_id", default="")

class CorrelationIdMiddleware(BaseHTTPMiddleware):
    """
    Middleware que:
    1. Llegeix X-Correlation-ID del header (si el client l'envia).
    2. Si no existeix, genera un UUID nou.
    3. El guarda al context per a structlog.
    4. L'afegeix a la resposta (perquè el client pugui referenciar-lo).
    """

    async def dispatch(self, request: Request, call_next):
        # Si el client envia correlation_id, el reutilitzem.
        # Si no, en generem un de nou (és el punt d'entrada).
        corr_id = request.headers.get(
            "X-Correlation-ID",
            str(uuid.uuid4())  # UUID v4: aleatori, pràcticament únic.
        )

        # Guardem al ContextVar per a que qualsevol codi async pugui accedir-hi.
        correlation_id_ctx.set(corr_id)

        # Vinculem el correlation_id a structlog.
        # A partir d'aquí, TOTS els logs d'aquesta petició inclouran aquest camp.
        structlog.contextvars.bind_contextvars(correlation_id=corr_id)

        # Processem la petició normalment.
        response = await call_next(request)

        # Afegim el correlation_id a la resposta.
        # El client pot usar-lo per reportar problemes: "Error amb ID abc-123".
        response.headers["X-Correlation-ID"] = corr_id

        # Netejem el context per a la pròxima petició.
        structlog.contextvars.unbind_contextvars("correlation_id")

        return response
```

**Registrar el middleware a FastAPI:**

```python
from fastapi import FastAPI

app = FastAPI()

# Afegim el middleware. S'executarà per a CADA petició, automàticament.
app.add_middleware(CorrelationIdMiddleware)
```

### Spring Boot OncePerRequestFilter: Llegir i Propagar Correlation ID

L'equivalent Java del middleware FastAPI. `OncePerRequestFilter` garanteix que s'executa exactament una vegada per petició:

```java
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.MDC;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.UUID;

/**
 * Filtre que llegeix o genera un correlation_id per a cada petició.
 * El guarda al MDC (Mapped Diagnostic Context) de SLF4J perquè
 * aparegui automàticament a TOTS els logs d'aquesta petició.
 */
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {

    // Nom del header HTTP estàndard per correlation IDs.
    public static final String CORRELATION_ID_HEADER = "X-Correlation-ID";
    // Clau del MDC. Ha de coincidir amb el nom a logback-spring.xml.
    public static final String CORRELATION_ID_MDC_KEY = "correlation_id";

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {

        // 1. Llegim el header. Si Python ens envia X-Correlation-ID, el reutilitzem.
        String correlationId = request.getHeader(CORRELATION_ID_HEADER);

        // 2. Si no hi ha header, generem un UUID nou.
        if (correlationId == null || correlationId.isBlank()) {
            correlationId = UUID.randomUUID().toString();
        }

        // 3. Guardem al MDC. A partir d'aquí, TOTS els logs d'aquesta petició
        //    inclouran "correlation_id": "abc-123" automàticament.
        MDC.put(CORRELATION_ID_MDC_KEY, correlationId);

        // 4. Afegim a la resposta (per traçabilitat).
        response.setHeader(CORRELATION_ID_HEADER, correlationId);

        try {
            // 5. Continuem amb el processament normal de la petició.
            filterChain.doFilter(request, response);
        } finally {
            // 6. Netejem el MDC. IMPORTANT: si no ho fem, el thread
            //    reutilitzat podria tenir el correlation_id d'una altra petició.
            MDC.remove(CORRELATION_ID_MDC_KEY);
        }
    }
}
```

### Propagar el Correlation ID Quan Python Crida Java

El punt clau: quan Python fa una petició HTTP a Java, **ha d'enviar el correlation_id** com a header:

```python
import httpx
import structlog
from contextvars import ContextVar

# Reutilitzem el ContextVar del middleware.
from middleware import correlation_id_ctx

logger = structlog.get_logger().bind(service_name="ai-python")

async def get_champion_from_java(champion_id: int) -> dict:
    """
    Crida al servei Java per obtenir un campió.
    Propaga el correlation_id perquè Java pugui traçar la petició.
    """
    # Recuperem el correlation_id del context actual.
    corr_id = correlation_id_ctx.get()

    logger.info("Cridant servei Java", champion_id=champion_id)

    async with httpx.AsyncClient(timeout=5.0) as client:
        response = await client.get(
            f"http://localhost:8080/api/champions/{champion_id}",
            # CLAU: enviem el correlation_id com a header.
            # Java el llegirà al CorrelationIdFilter i l'afegirà als seus logs.
            headers={"X-Correlation-ID": corr_id},
        )

    response.raise_for_status()
    logger.info("Resposta de Java rebuda",
        champion_id=champion_id,
        status=response.status_code,
    )
    return response.json()
```

### Retry amb Backoff Exponencial

Quan un servei extern falla de forma transitòria (sobrecàrrega, xarxa inestable), reintentar pot funcionar. Però reintentar immediatament **empitjora el problema** (thundering herd). El **backoff exponencial** espera cada vegada més temps:

```
Intent 1: espera 0s → falla
Intent 2: espera 1s → falla
Intent 3: espera 2s → falla
Intent 4: espera 4s → OK!
```

**Quan fer retry i quan NO:**

| Codi HTTP | Retry? | Per Què |
|-----------|--------|---------|
| **5xx** (500, 502, 503) | SÍ | Error temporal del servidor. Pot funcionar en reintentar. |
| **4xx** (400, 404, 422) | NO | Error del client. Reintentar donarà el mateix resultat. |
| **ConnectionError** | SÍ | El servei pot estar arrencant o la xarxa tallada temporalment. |
| **Timeout** | SÍ (amb precaució) | Si el servei estava sobrecarregat, pot respondre després. |

### Python: tenacity per Retry

`tenacity` és la llibreria de retry més popular de Python. Suporta backoff exponencial, condicions de retry i logging.

```bash
pip install tenacity
```

```python
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type
import httpx
import structlog

logger = structlog.get_logger().bind(service_name="ai-python")

# @retry configura el comportament de reintents:
# - stop_after_attempt(3): màxim 3 intents (1 original + 2 retries).
# - wait_exponential: espera 1s, 2s, 4s... entre intents.
# - retry_if_exception_type: NOMÉS reintentem errors de connexió i timeout.
#   NO reintentem si Java retorna 404 (reintentar no canviarà res).
@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10),
    retry=retry_if_exception_type((httpx.ConnectError, httpx.TimeoutException)),
    before_sleep=lambda retry_state: logger.warning(
        "Reintentant crida a Java",
        attempt=retry_state.attempt_number,
        wait_seconds=retry_state.next_action.sleep,
    ),
)
async def call_java_with_retry(url: str, headers: dict) -> httpx.Response:
    """
    Crida HTTP a Java amb retry automàtic per errors transitoris.
    El backoff exponencial evita sobrecarregar el servei.
    """
    async with httpx.AsyncClient(timeout=5.0) as client:
        response = await client.get(url, headers=headers)
        # Si Java retorna 5xx, llencem excepció per forçar retry.
        if response.status_code >= 500:
            raise httpx.HTTPStatusError(
                f"Java ha retornat {response.status_code}",
                request=response.request,
                response=response,
            )
        return response
```

### Java: @Retryable de Spring Retry

```xml
<!-- Dependència per Spring Retry (pom.xml). -->
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aspects</artifactId>
</dependency>
```

```java
import org.springframework.retry.annotation.Backoff;
import org.springframework.retry.annotation.Retryable;
import org.springframework.web.client.ResourceAccessException;

@Service
public class ExternalApiService {

    private static final Logger log = LoggerFactory.getLogger(ExternalApiService.class);

    /**
     * Crida a una API externa amb retry automàtic.
     * @Retryable configura:
     * - retryFor: només reintentem per errors de connexió (5xx, timeout).
     * - maxAttempts: 3 intents com a màxim.
     * - backoff: delay inicial 1s, es multiplica x2 cada intent.
     */
    @Retryable(
        retryFor = { ResourceAccessException.class },
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2)
    )
    public String callExternalService(String url) {
        log.info("Cridant servei extern", kv("url", url));
        // RestTemplate o WebClient fan la crida HTTP.
        return restTemplate.getForObject(url, String.class);
    }
}
```

Activar retry a la configuració de Spring:

```java
import org.springframework.retry.annotation.EnableRetry;

// @EnableRetry activa el processament de @Retryable a tota l'aplicació.
// Sense això, l'anotació @Retryable és ignorada.
@SpringBootApplication
@EnableRetry
public class EsportsPulseApplication {
    public static void main(String[] args) {
        SpringApplication.run(EsportsPulseApplication.class, args);
    }
}
```

---

## Activitat

### Pas 1: Implementar Middleware a Python

1. Crea `CorrelationIdMiddleware` com es mostra a la teoria
2. Registra'l a l'aplicació FastAPI amb `app.add_middleware()`
3. Verifica que els logs inclouen `correlation_id`

### Pas 2: Implementar Filter a Java

1. Crea `CorrelationIdFilter` al paquet `com.esportspulse.engine.filter`
2. Verifica que `logback-spring.xml` inclou el camp `correlation_id` del MDC
3. Arrencar i verificar que els logs de Java mostren el `correlation_id`

### Pas 3: Propagar Correlation ID Entre Serveis

1. Modifica la funció Python que crida a Java per enviar `X-Correlation-ID`
2. Fes una petició i verifica que el **mateix** `correlation_id` apareix als logs de Python i Java

```bash
# Fes una petició i observa els logs dels dos serveis:
curl -H "X-Correlation-ID: test-manual-123" http://localhost:8000/api/champions/1

# Als logs de Python has de veure: "correlation_id": "test-manual-123"
# Als logs de Java has de veure:   "correlation_id": "test-manual-123"
```

### Pas 4: Implementar Retry amb tenacity

1. Instal·la `tenacity`: `pip install tenacity`
2. Implementa `call_java_with_retry()` com a la teoria
3. Prova: apaga Java, crida Python, verifica que reintenta 3 vegades

```bash
# Apaga Java i crida Python:
curl http://localhost:8000/api/champions/1

# Als logs de Python has de veure:
# {"level": "warning", "message": "Reintentant crida a Java", "attempt": 1, ...}
# {"level": "warning", "message": "Reintentant crida a Java", "attempt": 2, ...}
# {"level": "error", "message": "Servei Java no disponible", ...}
```

---

## Checklist de Lliurament

- [ ] `CorrelationIdMiddleware` implementat a FastAPI
- [ ] `CorrelationIdFilter` implementat a Spring Boot amb MDC
- [ ] El `correlation_id` es propaga de Python a Java via header `X-Correlation-ID`
- [ ] El **mateix** `correlation_id` apareix als logs dels dos serveis per a una mateixa petició
- [ ] Retry amb `tenacity`: 3 intents amb backoff exponencial per errors de connexió
- [ ] NO es reintenta per errors 4xx (404, 400)
- [ ] Logs de retry mostren el número d'intent i el temps d'espera
- [ ] Commit amb missatge: `feat(observability): add correlation IDs and retry with backoff`
