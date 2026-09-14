# Setmana 20 — Dijous: Dashboard de Mètriques Operacionals

## Objectiu del Dia

Instrumentar els serveis per exposar mètriques operacionals i construir un dashboard Streamlit que les mostri. Al final del dia, tens visibilitat en temps real sobre: latència d'endpoints, cache hit ratio, cost LLM, errors per servei i profunditat de cues.

---

## Teoria

### Què Mesurar?

Les mètriques es divideixen en categories:

**RED (Rate, Errors, Duration) — Per a serveis:**
- **Rate:** Peticions per segon
- **Errors:** Percentatge de respostes amb error (4xx, 5xx)
- **Duration:** Latència de cada endpoint (p50, p95, p99)

**USE (Utilization, Saturation, Errors) — Per a recursos:**
- **Utilization:** Quant de CPU/memòria/disc s'utilitza
- **Saturation:** Cua de treball pendent (queue depth)
- **Errors:** Errors de connexió, timeouts

**Mètriques específiques d'EsportsPulse:**

| Mètrica | Per què importa |
|---------|----------------|
| Latència d'endpoints | L'usuari nota si triga > 500ms |
| Cache hit ratio | Si és < 50%, el cache no serveix |
| Cost LLM per query | Cada crida a l'LLM costa diners |
| Queue depth | Si creix, el consumer no processa prou ràpid |
| Errors per servei | Detectar serveis degradats |

### Actuator Metrics a Spring Boot

Actuator ja exposa mètriques JVM i HTTP automàticament:

```bash
# Llista de totes les mètriques disponibles
curl -s http://localhost:8080/actuator/metrics | jq '.names[]' | head -20

# Mètrica concreta: temps de resposta HTTP
curl -s http://localhost:8080/actuator/metrics/http.server.requests | jq

# Mètrica de memòria JVM
curl -s http://localhost:8080/actuator/metrics/jvm.memory.used | jq
```

### Mètriques Personalitzades amb Micrometer

**Micrometer** és la llibreria de mètriques de Spring Boot (equivalent a `prometheus-client` en Python):

```java
// Registrar un comptador (counter) — incrementa amb cada event
@Autowired
private MeterRegistry meterRegistry;

// Comptar quantes vegades es crida l'LLM
Counter llmCalls = meterRegistry.counter("llm.calls.total", "service", "summary");
llmCalls.increment();

// Mesurar la duració d'una operació
Timer.Sample sample = Timer.start(meterRegistry);
// ... operació costosa ...
sample.stop(meterRegistry.timer("llm.processing.duration"));

// Gauge: valor actual (ex: profunditat de la cua)
meterRegistry.gauge("rabbitmq.queue.depth", queueDepth);
```

### Mètriques a Python

```python
# Guardar mètriques a Redis per compartir-les entre serveis
import time

def track_metric(redis_client, name: str, value: float):
    """Guarda una mètrica a Redis com a sorted set amb timestamp."""
    timestamp = time.time()
    # Sorted set: score = timestamp, member = valor
    redis_client.zadd(f"metrics:{name}", {f"{timestamp}:{value}": timestamp})
    # Mantenir només les últimes 1000 entrades
    redis_client.zremrangebyrank(f"metrics:{name}", 0, -1001)
```

> **Lectura recomanada (opcional, no bloquejant):**
> - [Micrometer Documentation](https://micrometer.io/docs) — Documentació oficial
> - [The RED Method](https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/)

---

## Activitat

### 1. Instrumentar el backend Java (30 min)

Afegeix mètriques personalitzades als punts clau:

```java
// MetricsService.java — Servei centralitzat de mètriques

@Service
public class MetricsService {

    private final MeterRegistry registry;

    // Comptadors
    private final Counter cacheHits;
    private final Counter cacheMisses;
    private final Counter eventsPublished;

    public MetricsService(MeterRegistry registry) {
        this.registry = registry;

        // Comptador de cache hits i misses (per calcular hit ratio)
        this.cacheHits = Counter.builder("cache.hits")
            .description("Nombre de cache hits")
            .tag("service", "champion-stats")
            .register(registry);

        this.cacheMisses = Counter.builder("cache.misses")
            .description("Nombre de cache misses")
            .tag("service", "champion-stats")
            .register(registry);

        // Comptador d'events publicats
        this.eventsPublished = Counter.builder("events.published.total")
            .description("Nombre total d'events publicats a RabbitMQ")
            .register(registry);
    }

    public void recordCacheHit() { cacheHits.increment(); }
    public void recordCacheMiss() { cacheMisses.increment(); }
    public void recordEventPublished() { eventsPublished.increment(); }

    /**
     * Mesura el temps d'execució d'una operació.
     * Ús: metricsService.timeOperation("db.query", () -> repository.findAll());
     */
    public <T> T timeOperation(String name, java.util.function.Supplier<T> operation) {
        Timer.Sample sample = Timer.start(registry);
        try {
            T result = operation.get();
            sample.stop(registry.timer(name, "status", "success"));
            return result;
        } catch (Exception e) {
            sample.stop(registry.timer(name, "status", "error"));
            throw e;
        }
    }
}
```

Integra les mètriques al servei de cache:

```java
// Modificar ChampionStatsService.java per registrar mètriques

public List<ChampionStatsDto> getStatsByRole(String role) {
    String cacheKey = CACHE_PREFIX + role.toUpperCase();
    String cached = redisTemplate.opsForValue().get(cacheKey);

    if (cached != null) {
        metricsService.recordCacheHit();  // Registrar cache hit
        return deserializeList(cached);
    }

    metricsService.recordCacheMiss();  // Registrar cache miss

    // Mesurar el temps de la query a PostgreSQL
    List<ChampionStatsDto> stats = metricsService.timeOperation(
        "db.query.championStats",
        () -> championRepository.findStatsByRole(role)
    );

    redisTemplate.opsForValue().set(cacheKey, serialize(stats), CACHE_TTL);
    return stats;
}
```

### 2. Instrumentar el consumer Python (20 min)

Afegeix mètriques al consumer:

```python
# ai-python/src/consumers/metrics.py — Mètriques del consumer

import time
import redis
import json

class ConsumerMetrics:
    """Registra mètriques del consumer a Redis per ser llegides pel dashboard."""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.prefix = "metrics:consumer"

    def record_processing_time(self, champion_name: str, duration_ms: float):
        """Registrar el temps de processament d'un event."""
        self.redis.lpush(f"{self.prefix}:processing_times",
                         json.dumps({"champion": champion_name,
                                     "durationMs": duration_ms,
                                     "timestamp": time.time()}))
        # Mantenir només els últims 100 registres
        self.redis.ltrim(f"{self.prefix}:processing_times", 0, 99)

    def record_llm_cost(self, tokens_used: int, cost_usd: float):
        """Registrar el cost d'una crida LLM."""
        self.redis.incrbyfloat(f"{self.prefix}:llm_cost_total", cost_usd)
        self.redis.incrby(f"{self.prefix}:llm_tokens_total", tokens_used)

    def record_event_processed(self):
        """Incrementar el comptador d'events processats."""
        self.redis.incr(f"{self.prefix}:events_processed")

    def record_event_failed(self):
        """Incrementar el comptador d'events fallits."""
        self.redis.incr(f"{self.prefix}:events_failed")

    def get_summary(self) -> dict:
        """Obtenir un resum de totes les mètriques."""
        return {
            "eventsProcessed": int(self.redis.get(f"{self.prefix}:events_processed") or 0),
            "eventsFailed": int(self.redis.get(f"{self.prefix}:events_failed") or 0),
            "llmCostTotal": float(self.redis.get(f"{self.prefix}:llm_cost_total") or 0),
            "llmTokensTotal": int(self.redis.get(f"{self.prefix}:llm_tokens_total") or 0),
        }
```

### 3. Crear l'endpoint d'agregació de mètriques (15 min)

Crea un endpoint que combini les mètriques de Java i Python:

```python
# ai-python/src/metrics_api.py — Endpoint d'agregació de mètriques

from fastapi import APIRouter
import httpx
import redis
import os

router = APIRouter(prefix="/api/metrics", tags=["metrics"])

BACKEND_URL = os.getenv("BACKEND_URL", "http://localhost:8080")
redis_client = redis.Redis(
    host=os.getenv("REDIS_HOST", "localhost"),
    port=int(os.getenv("REDIS_PORT", "6379")),
    decode_responses=True
)


@router.get("/overview")
async def metrics_overview():
    """Resum de mètriques de tota la plataforma."""
    metrics = {}

    # Mètriques de Java (Actuator)
    try:
        async with httpx.AsyncClient() as client:
            # Cache hit ratio
            hits_resp = await client.get(f"{BACKEND_URL}/actuator/metrics/cache.hits")
            misses_resp = await client.get(f"{BACKEND_URL}/actuator/metrics/cache.misses")
            hits = hits_resp.json().get("measurements", [{}])[0].get("value", 0)
            misses = misses_resp.json().get("measurements", [{}])[0].get("value", 0)
            total = hits + misses
            metrics["cacheHitRatio"] = round(hits / total, 2) if total > 0 else 0
            metrics["cacheHits"] = hits
            metrics["cacheMisses"] = misses
    except Exception as e:
        metrics["javaError"] = str(e)

    # Mètriques del consumer (Redis)
    try:
        metrics["eventsProcessed"] = int(redis_client.get("metrics:consumer:events_processed") or 0)
        metrics["eventsFailed"] = int(redis_client.get("metrics:consumer:events_failed") or 0)
        metrics["llmCostTotal"] = float(redis_client.get("metrics:consumer:llm_cost_total") or 0)
        metrics["llmTokensTotal"] = int(redis_client.get("metrics:consumer:llm_tokens_total") or 0)

        # Últims temps de processament
        raw_times = redis_client.lrange("metrics:consumer:processing_times", 0, 9)
        metrics["recentProcessingTimes"] = [
            json.loads(t) for t in raw_times
        ] if raw_times else []
    except Exception as e:
        metrics["consumerError"] = str(e)

    return metrics
```

### 4. Construir el dashboard Streamlit (40 min)

Crea `ai-python/src/pages/monitoring.py`:

```python
"""
Dashboard de mètriques operacionals d'EsportsPulse.
Mostra l'estat dels serveis, cache hit ratio, cost LLM i events processats.
"""

import streamlit as st
import httpx
import time

# Configuració de la pàgina
st.set_page_config(page_title="EsportsPulse — Monitoring", layout="wide")
st.title("Monitoring Dashboard — EsportsPulse")

# URL dels serveis
BACKEND_URL = "http://localhost:8080"
PYTHON_URL = "http://localhost:8000"

# Auto-refresh cada 10 segons
refresh_interval = st.sidebar.slider("Refresh interval (s)", 5, 60, 10)

# --- Secció 1: Estat dels serveis ---
st.header("Estat dels Serveis")

col1, col2, col3, col4 = st.columns(4)

# Comprovar cada servei
def check_service(url: str, name: str) -> dict:
    """Comprova si un servei respon i retorna el seu estat."""
    try:
        resp = httpx.get(url, timeout=5.0)
        return {"name": name, "status": "UP" if resp.status_code == 200 else "DOWN",
                "responseTime": resp.elapsed.total_seconds() * 1000}
    except Exception as e:
        return {"name": name, "status": "DOWN", "error": str(e)}

# Comprovar serveis en paral·lel
services = [
    check_service(f"{BACKEND_URL}/actuator/health", "Java Backend"),
    check_service(f"{PYTHON_URL}/health", "Python Service"),
    check_service("http://localhost:15672/api/overview", "RabbitMQ"),
]

for i, svc in enumerate(services):
    with [col1, col2, col3][i]:
        if svc["status"] == "UP":
            st.metric(svc["name"], "UP", f"{svc.get('responseTime', 0):.0f}ms")
        else:
            st.metric(svc["name"], "DOWN", delta="Error", delta_color="inverse")

# --- Secció 2: Mètriques de cache ---
st.header("Cache (Redis)")

try:
    metrics = httpx.get(f"{PYTHON_URL}/api/metrics/overview", timeout=10.0).json()

    col1, col2, col3 = st.columns(3)
    with col1:
        ratio = metrics.get("cacheHitRatio", 0)
        # Mostrar el hit ratio amb color segons el valor
        st.metric("Cache Hit Ratio",
                  f"{ratio:.0%}",
                  delta="Bo" if ratio > 0.7 else "Baix")
    with col2:
        st.metric("Cache Hits", metrics.get("cacheHits", 0))
    with col3:
        st.metric("Cache Misses", metrics.get("cacheMisses", 0))
except Exception as e:
    st.error(f"Error obtenint mètriques: {e}")

# --- Secció 3: Processament asíncron ---
st.header("Processament Asíncron (RabbitMQ + LLM)")

try:
    col1, col2, col3, col4 = st.columns(4)
    with col1:
        st.metric("Events Processats", metrics.get("eventsProcessed", 0))
    with col2:
        st.metric("Events Fallits", metrics.get("eventsFailed", 0))
    with col3:
        cost = metrics.get("llmCostTotal", 0)
        st.metric("Cost LLM Total", f"${cost:.4f}")
    with col4:
        tokens = metrics.get("llmTokensTotal", 0)
        st.metric("Tokens Totals", f"{tokens:,}")

    # Taula amb els últims temps de processament
    recent_times = metrics.get("recentProcessingTimes", [])
    if recent_times:
        st.subheader("Últims processaments")
        st.dataframe(recent_times, use_container_width=True)
except Exception:
    pass

# --- Secció 4: Latència d'endpoints ---
st.header("Latència d'Endpoints")

try:
    # Obtenir mètrica de latència HTTP des d'Actuator
    resp = httpx.get(
        f"{BACKEND_URL}/actuator/metrics/http.server.requests",
        timeout=5.0
    )
    data = resp.json()
    measurements = {m["statistic"]: m["value"] for m in data.get("measurements", [])}

    col1, col2, col3 = st.columns(3)
    with col1:
        st.metric("Peticions totals", f"{measurements.get('COUNT', 0):.0f}")
    with col2:
        total_time = measurements.get("TOTAL_TIME", 0)
        count = measurements.get("COUNT", 1)
        avg_ms = (total_time / count) * 1000 if count > 0 else 0
        st.metric("Latència mitjana", f"{avg_ms:.1f}ms")
    with col3:
        max_ms = measurements.get("MAX", 0) * 1000
        st.metric("Latència màxima", f"{max_ms:.1f}ms")
except Exception as e:
    st.warning(f"No es poden obtenir mètriques de latència: {e}")

# Auto-refresh
st.markdown(f"*Actualització automàtica cada {refresh_interval} segons*")
time.sleep(refresh_interval)
st.rerun()
```

### 5. Testejar el dashboard (10 min)

```bash
# Arrencar el dashboard
cd ai-python
streamlit run src/pages/monitoring.py

# Generar activitat per veure mètriques canviar:
# Fer varies crides a l'API per generar cache hits/misses
for i in $(seq 1 10); do
    curl -s http://localhost:8080/api/champions/stats?role=MID > /dev/null
done

# Crear un campió per generar un event asíncron
curl -X POST http://localhost:8080/api/champions \
  -H "Content-Type: application/json" \
  -d '{"name":"Lux","role":"SUPPORT","description":"Maga de llum amb shield i CC"}'
```

### 6. Commit (5 min)

```bash
git add .
git commit -m "feat(monitoring): add metrics instrumentation and Streamlit monitoring dashboard"
```

---

## Checklist de Lliurament

- [ ] Mètriques personalitzades afegides al backend Java (cache hits/misses, event counts)
- [ ] Mètriques del consumer registrades a Redis
- [ ] Endpoint `/api/metrics/overview` funciona i retorna dades
- [ ] Dashboard Streamlit mostra: estat serveis, cache ratio, cost LLM, latència
- [ ] Auto-refresh configurat al dashboard
- [ ] Mètriques canvien en temps real quan es generen peticions
- [ ] Commit fet
