# Setmana 20 — Dimecres: Health Endpoints i Actuator

## Objectiu del Dia

Afegir health endpoints a tots els serveis per saber si estan sans i operatius. Configurar Docker per usar aquests endpoints com a health checks. Al final del dia, `docker-compose ps` mostra l'estat de salut de cada servei, i si un servei cau, ho detectes immediatament.

---

## Teoria

### Per Què Health Endpoints?

Sense health endpoints, l'única manera de saber si un servei funciona és provar-lo manualment. Amb ells:

```
Load Balancer → GET /actuator/health → 200 OK      → enviar trànsit
                                     → 503 Service Unavailable → no enviar trànsit

Docker        → GET /actuator/health → 200 OK      → contenidor sa
                                     → timeout/503 → reiniciar contenidor
```

**Què ha de comprovar un health endpoint?**
- La pròpia aplicació respon? (bàsic)
- La connexió a PostgreSQL funciona? (dependència crítica)
- Redis respon? (dependència important)
- RabbitMQ accessible? (dependència per events)
- Hi ha prou memòria/disc? (recursos)

### Spring Boot Actuator

Actuator és un mòdul de Spring Boot que exposa endpoints de gestió i monitorització:

| Endpoint | Funció |
|----------|--------|
| `/actuator/health` | Estat de salut (UP/DOWN) |
| `/actuator/info` | Informació de l'aplicació (versió, build) |
| `/actuator/metrics` | Mètriques (memòria, threads, peticions) |
| `/actuator/env` | Variables d'entorn (cuidado amb secrets!) |
| `/actuator/beans` | Tots els beans de Spring registrats |

**Per defecte, Actuator només exposa `/health` i `/info`.** Cal configurar explícitament quins endpoints es fan públics.

```properties
# Exposar endpoints concrets (mai tots en producció!)
management.endpoints.web.exposure.include=health,info,metrics
# Mostrar detalls del health check (dependències, estat)
management.endpoint.health.show-details=when-authorized
```

### Health Indicators Personalitzats

Actuator ja comprova PostgreSQL i Redis automàticament si les dependències estan al classpath. Però pots afegir comprovacions pròpies:

```java
// Comprova que RabbitMQ respon a AMQP
@Component
public class RabbitMQHealthIndicator implements HealthIndicator {

    private final RabbitTemplate rabbitTemplate;

    public RabbitMQHealthIndicator(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    @Override
    public Health health() {
        try {
            // Intentar obtenir informació de connexió
            rabbitTemplate.execute(channel -> {
                channel.queueDeclarePassive("llm-processing");
                return null;
            });
            return Health.up()
                .withDetail("queues", "accessible")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

### Health Endpoint a FastAPI

FastAPI no té Actuator, però crear un health endpoint és senzill:

```python
@app.get("/health")
async def health():
    """Health check que verifica totes les dependències."""
    checks = {}
    
    # Comprovar Redis
    try:
        redis_client.ping()
        checks["redis"] = "UP"
    except Exception as e:
        checks["redis"] = f"DOWN: {e}"
    
    # Comprovar Qdrant
    try:
        qdrant_client.get_collections()
        checks["qdrant"] = "UP"
    except Exception as e:
        checks["qdrant"] = f"DOWN: {e}"
    
    # Determinar estat global
    all_up = all(v == "UP" for v in checks.values())
    status_code = 200 if all_up else 503
    
    return JSONResponse(
        status_code=status_code,
        content={"status": "UP" if all_up else "DOWN", "checks": checks}
    )
```

> **Lectura recomanada (opcional, no bloquejant):**
> - [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html) — Documentació oficial
> - [Health Check Patterns](https://microservices.io/patterns/observability/health-check-api.html)

---

## Activitat

### 1. Configurar Spring Boot Actuator (20 min)

Afegeix la dependència al `pom.xml`:

```xml
<!-- Spring Boot Actuator — endpoints de gestió i monitorització -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Configura a `application.properties`:

```properties
# --- Actuator ---
# Endpoints exposats via HTTP (restringir en producció!)
management.endpoints.web.exposure.include=health,info,metrics

# Mostrar detalls del health check (dependències, estat de cada una)
# 'always' per desenvolupament; en producció usar 'when-authorized'
management.endpoint.health.show-details=always

# Mostrar components individuals del health check
management.endpoint.health.show-components=always

# Informació del build (es mostra a /actuator/info)
management.info.build.enabled=true
management.info.java.enabled=true
management.info.os.enabled=true

# Prefix per als endpoints d'Actuator (defecte: /actuator)
management.endpoints.web.base-path=/actuator
```

Verifica:

```bash
# Arrencar l'aplicació
mvn spring-boot:run

# Health check bàsic
curl -s http://localhost:8080/actuator/health | jq

# Resposta esperada:
# {
#   "status": "UP",
#   "components": {
#     "db": { "status": "UP", "details": { "database": "PostgreSQL" } },
#     "diskSpace": { "status": "UP" },
#     "redis": { "status": "UP" }
#   }
# }
```

### 2. Afegir health indicator personalitzat per RabbitMQ (20 min)

Crea `RabbitMQHealthIndicator.java`:

```java
// RabbitMQHealthIndicator.java — Comprova la connexió amb RabbitMQ
// S'integra automàticament amb Actuator gràcies a @Component

@Component
public class RabbitMQHealthIndicator implements HealthIndicator {

    private final RabbitTemplate rabbitTemplate;
    // Nom de la cua que comprovem (ha d'existir)
    private static final String CHECK_QUEUE = "llm-processing";

    public RabbitMQHealthIndicator(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    @Override
    public Health health() {
        try {
            // Intentar una operació passiva: verificar que la cua existeix
            // No modifica res, només comprova la connexió
            rabbitTemplate.execute(channel -> {
                // queueDeclarePassive llança excepció si la cua no existeix
                channel.queueDeclarePassive(CHECK_QUEUE);
                return null;
            });
            return Health.up()
                .withDetail("queue", CHECK_QUEUE)
                .withDetail("status", "accessible")
                .build();
        } catch (Exception e) {
            // Si falla, reportar DOWN amb el missatge d'error
            return Health.down()
                .withDetail("queue", CHECK_QUEUE)
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

### 3. Crear health endpoint a FastAPI (25 min)

Crea o modifica l'endpoint de health al servei Python:

```python
# ai-python/src/health.py — Health endpoint per al servei FastAPI

import time
from fastapi import APIRouter
from fastapi.responses import JSONResponse
import redis
from qdrant_client import QdrantClient

# Router per agrupar els endpoints de health
router = APIRouter(tags=["health"])

# Clients (injectats o creats aquí)
redis_client = redis.Redis(
    host=os.getenv("REDIS_HOST", "localhost"),
    port=int(os.getenv("REDIS_PORT", "6379")),
    decode_responses=True
)

qdrant_client = QdrantClient(
    host=os.getenv("QDRANT_HOST", "localhost"),
    port=int(os.getenv("QDRANT_PORT", "6333"))
)


@router.get("/health")
async def health_check():
    """
    Health endpoint que comprova totes les dependències.
    Retorna 200 si tot funciona, 503 si alguna dependència falla.
    """
    checks = {}
    start_time = time.time()

    # Comprovar Redis
    try:
        redis_client.ping()
        checks["redis"] = {"status": "UP", "responseTimeMs": _measure(redis_client.ping)}
    except Exception as e:
        checks["redis"] = {"status": "DOWN", "error": str(e)}

    # Comprovar Qdrant
    try:
        collections = qdrant_client.get_collections()
        checks["qdrant"] = {
            "status": "UP",
            "collections": len(collections.collections)
        }
    except Exception as e:
        checks["qdrant"] = {"status": "DOWN", "error": str(e)}

    # Determinar estat global
    all_healthy = all(c.get("status") == "UP" for c in checks.values())
    total_time = round((time.time() - start_time) * 1000, 1)

    response = {
        "status": "UP" if all_healthy else "DOWN",
        "checks": checks,
        "totalCheckTimeMs": total_time
    }

    return JSONResponse(
        status_code=200 if all_healthy else 503,
        content=response
    )


@router.get("/health/live")
async def liveness():
    """
    Liveness probe: l'aplicació està viva (pot respondre HTTP).
    No comprova dependències — només que el procés no està penjat.
    """
    return {"status": "UP"}


@router.get("/health/ready")
async def readiness():
    """
    Readiness probe: l'aplicació està preparada per rebre trànsit.
    Comprova les dependències crítiques.
    """
    # Reutilitzar la lògica del health check complet
    return await health_check()


def _measure(func) -> float:
    """Mesura el temps d'execució d'una funció en mil·lisegons."""
    start = time.time()
    func()
    return round((time.time() - start) * 1000, 1)
```

Registra el router a l'aplicació principal:

```python
# ai-python/src/main.py — afegir el router de health
from src.health import router as health_router

app = FastAPI(title="EsportsPulse AI Service")
app.include_router(health_router)
```

### 4. Configurar Docker healthchecks (20 min)

Actualitza el `docker-compose.yml` per usar els health endpoints:

```yaml
  # Backend Java — usa Actuator per health check
  backend-java:
    # ... configuració existent ...
    healthcheck:
      # curl crida l'endpoint d'Actuator dins el contenidor
      test: ["CMD-SHELL", "curl -f http://localhost:8080/actuator/health || exit 1"]
      interval: 30s       # Comprova cada 30 segons
      timeout: 10s        # Si no respon en 10s, es considera fallada
      retries: 3          # Després de 3 fallades, marcar com unhealthy
      start_period: 60s   # Donar 60s perquè l'app arrenqui (Spring Boot triga)

  # Servei Python — usa el health endpoint propi
  service-python:
    # ... configuració existent ...
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8000/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

  # Consumer Python — comprova que el procés està viu
  consumer-python:
    # ... configuració existent ...
    healthcheck:
      # El consumer no té endpoint HTTP; comprovem que el procés Python corre
      test: ["CMD-SHELL", "pgrep -f champion_consumer || exit 1"]
      interval: 30s
      timeout: 5s
      retries: 3

  # Streamlit
  streamlit:
    # ... configuració existent ...
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8501/_stcore/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
```

> **Nota:** Assegura't que les imatges Docker dels serveis Python incloguin `curl`. Si usen `python:3.12-slim`, afegeix al Dockerfile:
> ```dockerfile
> RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
> ```

### 5. Verificar tots els health checks (10 min)

```bash
# Arrencar tot
docker-compose up -d

# Esperar que tot estigui healthy (pot trigar 1-2 minuts)
# Observar l'estat amb:
docker-compose ps

# Resposta esperada:
# NAME                       STATUS          PORTS
# esportspulse-backend       Up (healthy)    0.0.0.0:8080->8080/tcp
# esportspulse-postgres      Up (healthy)    0.0.0.0:5432->5432/tcp
# esportspulse-redis         Up (healthy)    0.0.0.0:6379->6379/tcp
# esportspulse-rabbitmq      Up (healthy)    ...
# esportspulse-python        Up (healthy)    0.0.0.0:8000->8000/tcp
# esportspulse-consumer      Up (healthy)    
# esportspulse-streamlit     Up (healthy)    0.0.0.0:8501->8501/tcp

# Comprovar els health endpoints directament:
curl -s http://localhost:8080/actuator/health | jq
curl -s http://localhost:8000/health | jq
```

### 6. Commit (5 min)

```bash
git add .
git commit -m "feat(health): add health endpoints with Actuator and Docker healthchecks"
```

---

## Checklist de Lliurament

- [ ] Spring Boot Actuator configurat amb `/actuator/health`
- [ ] Health indicator personalitzat per RabbitMQ
- [ ] Health endpoint a FastAPI amb comprovació de Redis i Qdrant
- [ ] Endpoints `/health/live` i `/health/ready` a FastAPI
- [ ] Docker healthchecks configurats per a tots els serveis
- [ ] `docker-compose ps` mostra tots els serveis com "healthy"
- [ ] `start_period` configurat per donar temps d'arrencada
- [ ] Commit fet
