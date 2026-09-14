# Setmana 19 — Dijous: Retry, Idempotència i Resiliència

## Objectiu del Dia

Fer el consumer robust: reintentar quan un servei extern falla, evitar processar el mateix missatge dos cops, i implementar el concepte de circuit breaker. Al final del dia, el consumer aguanta caigudes del servei LLM sense perdre missatges.

---

## Teoria

### Retry amb Backoff Exponencial

Quan un servei extern falla (l'LLM no respon, timeout, error 500), reintentar immediatament sol empitjorar les coses: si el servei està sobrecarregat, 1000 retries simultanis l'acaben de matar.

**Backoff exponencial:** Cada reintent espera més temps que l'anterior.

```
Intent 1: falla → espera 1 segon
Intent 2: falla → espera 2 segons
Intent 3: falla → espera 4 segons
Intent 4: falla → espera 8 segons
Intent 5: falla → enviar a DLQ (prou, és un error permanent)
```

**Amb jitter (aleatorietat):** Afegir un component aleatori per evitar que tots els consumers reinviïn al mateix instant.

```python
import random
import time

def retry_with_backoff(func, max_retries=5, base_delay=1.0):
    """
    Executa una funció amb reintentos i backoff exponencial.
    base_delay: temps base en segons (es duplica a cada intent)
    """
    for attempt in range(1, max_retries + 1):
        try:
            return func()
        except Exception as e:
            if attempt == max_retries:
                # Últim intent fallit — relançar l'excepció
                raise
            # Calcular delay amb jitter (±25%)
            delay = base_delay * (2 ** (attempt - 1))
            jitter = delay * 0.25 * random.random()
            wait_time = delay + jitter
            logger.warning(
                "Intent %d/%d fallit: %s. Reintentant en %.1fs...",
                attempt, max_retries, e, wait_time
            )
            time.sleep(wait_time)
```

> **Connexió amb S11:** A la Setmana 11 vam veure backoff al servei Python quan cridava APIs externes. El patró és el mateix, ara aplicat a consumers de cues.

### Idempotència: Processar Sense Duplicats

En sistemes distribuïts, un missatge pot arribar **més d'un cop**:
- El consumer processa el missatge però falla abans d'enviar l'ACK
- RabbitMQ reenvia el missatge (creu que no s'ha processat)
- El consumer el torna a processar → **duplicat**

**Solució: Event ID deduplication**

```python
def is_already_processed(event_id: str, redis_client) -> bool:
    """Comprova si ja hem processat aquest event."""
    key = f"processed_events:{event_id}"
    return redis_client.exists(key) > 0

def mark_as_processed(event_id: str, redis_client):
    """Marca l'event com a processat (amb TTL per no acumular infinitament)."""
    key = f"processed_events:{event_id}"
    # Guardar durant 24h — prou per detectar duplicats
    redis_client.setex(key, timedelta(hours=24), "1")
```

**Flux idempotent complet:**
```
1. Rebre missatge amb eventId = "evt-abc123"
2. Comprovar a Redis: existeix "processed_events:evt-abc123"?
   → SÍ: skip, enviar ACK (ja processat)
   → NO: processar, marcar com processat, enviar ACK
```

### Circuit Breaker

El **circuit breaker** és un patró que evita cridar un servei que sabem que està caigut:

```
CLOSED (normal)  → El servei funciona, enviem peticions
                    Si falla 5 vegades seguides → passa a OPEN

OPEN (protecció) → Rebutgem peticions immediatament (no cridem el servei)
                    Esperem 30 segons → passa a HALF-OPEN

HALF-OPEN (prova)→ Deixem passar 1 petició
                    Si funciona → torna a CLOSED
                    Si falla → torna a OPEN
```

```python
class CircuitBreaker:
    """
    Implementació simple de circuit breaker.
    Impedeix cridar un servei que sabem que està caigut.
    """
    def __init__(self, failure_threshold=5, recovery_timeout=30):
        self.failure_threshold = failure_threshold  # Fallos per obrir el circuit
        self.recovery_timeout = recovery_timeout    # Segons abans de provar de nou
        self.failure_count = 0
        self.state = "CLOSED"       # CLOSED, OPEN, HALF_OPEN
        self.last_failure_time = 0

    def can_execute(self) -> bool:
        """Retorna True si podem cridar el servei."""
        if self.state == "CLOSED":
            return True
        if self.state == "OPEN":
            # Comprovar si ha passat prou temps per provar de nou
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = "HALF_OPEN"
                return True
            return False
        # HALF_OPEN: permetre un sol intent
        return True

    def record_success(self):
        """Registrar un èxit — tancar el circuit."""
        self.failure_count = 0
        self.state = "CLOSED"

    def record_failure(self):
        """Registrar un error — potencialment obrir el circuit."""
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = "OPEN"
            logger.warning("Circuit OBERT — el servei LLM ha fallat %d vegades",
                          self.failure_count)
```

> **Lectura recomanada (opcional, no bloquejant):**
> - [Circuit Breaker Pattern](https://martinfowler.com/bliki/CircuitBreaker.html) — Martin Fowler
> - [RabbitMQ Reliability Guide](https://www.rabbitmq.com/docs/reliability)

---

## Activitat

### 1. Afegir retry amb backoff al consumer (30 min)

Modifica el consumer per afegir retries a la crida LLM:

```python
# ai-python/src/consumers/champion_consumer.py — afegir retry

def generate_champion_summary_with_retry(champion_data: dict,
                                          max_retries: int = 3) -> str:
    """
    Genera un resum amb l'LLM amb reintentos i backoff exponencial.
    Si tots els intents fallen, llença l'excepció original.
    """
    last_exception = None

    for attempt in range(1, max_retries + 1):
        try:
            return generate_champion_summary(champion_data)
        except (httpx.HTTPError, httpx.TimeoutException) as e:
            last_exception = e
            if attempt == max_retries:
                logger.error("Tots els %d intents han fallat per '%s'",
                            max_retries, champion_data.get("name"))
                raise

            # Backoff exponencial amb jitter
            delay = (2 ** (attempt - 1)) + random.uniform(0, 1)
            logger.warning(
                "Intent %d/%d fallit per '%s': %s. Reintentant en %.1fs...",
                attempt, max_retries, champion_data.get("name"), e, delay
            )
            time.sleep(delay)

    # Punt mai assolit, però per seguretat
    raise last_exception
```

Actualitza `process_champion_event` per usar la versió amb retry:

```python
def process_champion_event(event: dict, redis_client: redis.Redis) -> None:
    """Processa un event de campió amb retry i idempotència."""
    payload = event.get("payload", {})
    champion_name = payload.get("name", "unknown")

    # Generar resum amb reintentos
    start_time = time.time()
    summary = generate_champion_summary_with_retry(payload)
    elapsed = time.time() - start_time

    logger.info("Resum generat en %.2f segons per '%s'", elapsed, champion_name)

    # Guardar a Redis
    cache_key = f"champion:summary:{payload.get('championId', 'unknown')}"
    cache_value = json.dumps({
        "championId": payload.get("championId"),
        "name": champion_name,
        "summary": summary,
        "generatedAt": event.get("timestamp"),
        "processingTimeMs": int(elapsed * 1000)
    })
    redis_client.setex(cache_key, CACHE_TTL, cache_value)
    logger.info("Resum guardat a Redis: '%s'", cache_key)
```

### 2. Afegir deduplicació per event ID (20 min)

```python
# Afegir al champion_consumer.py

def is_already_processed(event_id: str, redis_client: redis.Redis) -> bool:
    """
    Comprova si l'event ja s'ha processat.
    Usa Redis com a registre d'events processats.
    """
    return redis_client.exists(f"processed_events:{event_id}") > 0


def mark_as_processed(event_id: str, redis_client: redis.Redis) -> None:
    """
    Marca l'event com a processat a Redis amb un TTL de 24 hores.
    El TTL evita acumular claus infinitament.
    """
    redis_client.setex(f"processed_events:{event_id}", timedelta(hours=24), "1")
```

Actualitza el callback `on_message`:

```python
def on_message(channel, method, properties, body):
    """Callback amb deduplicació i retry."""
    redis_client = get_redis_client()

    try:
        event = json.loads(body)
        event_id = event.get("eventId", "unknown")

        # Comprovar duplicats ABANS de processar
        if is_already_processed(event_id, redis_client):
            logger.info("Event %s ja processat. Skipping.", event_id)
            channel.basic_ack(delivery_tag=method.delivery_tag)
            return

        # Processar l'event
        process_champion_event(event, redis_client)

        # Marcar com a processat DESPRÉS de l'èxit
        mark_as_processed(event_id, redis_client)

        # ACK
        channel.basic_ack(delivery_tag=method.delivery_tag)
        logger.info("Event %s processat i marcat com completat.", event_id)

    except json.JSONDecodeError as e:
        logger.error("JSON invàlid: %s. Enviant a DLQ.", e)
        channel.basic_nack(delivery_tag=method.delivery_tag, requeue=False)

    except (httpx.HTTPError, httpx.TimeoutException) as e:
        # Després dels retries interns, si encara falla → DLQ
        logger.error("Error LLM persistent: %s. Enviant a DLQ.", e)
        channel.basic_nack(delivery_tag=method.delivery_tag, requeue=False)

    except Exception as e:
        logger.error("Error inesperat: %s. Enviant a DLQ.", e)
        channel.basic_nack(delivery_tag=method.delivery_tag, requeue=False)
```

### 3. Implementar el circuit breaker (20 min)

Crea `ai-python/src/consumers/circuit_breaker.py`:

```python
"""
Implementació del patró Circuit Breaker.
Impedeix cridar un servei extern que sabem que està caigut,
evitant sobrecàrrega i timeouts innecessaris.
"""

import time
import logging

logger = logging.getLogger(__name__)


class CircuitBreaker:
    def __init__(self, name: str, failure_threshold: int = 5,
                 recovery_timeout: int = 30):
        # Nom per identificar el circuit als logs
        self.name = name
        # Nombre de fallos consecutius per obrir el circuit
        self.failure_threshold = failure_threshold
        # Segons que el circuit roman obert abans de provar de nou
        self.recovery_timeout = recovery_timeout
        # Comptador de fallos consecutius
        self.failure_count = 0
        # Estat actual: CLOSED (normal), OPEN (bloquejat), HALF_OPEN (provant)
        self.state = "CLOSED"
        # Moment de l'últim error (per calcular quan provar de nou)
        self.last_failure_time = 0

    def can_execute(self) -> bool:
        """Retorna True si és segur cridar el servei extern."""
        if self.state == "CLOSED":
            return True
        if self.state == "OPEN":
            elapsed = time.time() - self.last_failure_time
            if elapsed > self.recovery_timeout:
                logger.info("[%s] Circuit HALF_OPEN — provant una petició",
                           self.name)
                self.state = "HALF_OPEN"
                return True
            return False
        return True  # HALF_OPEN: permetre un intent

    def record_success(self):
        """Registrar un èxit — tancar el circuit si estava obert."""
        if self.state == "HALF_OPEN":
            logger.info("[%s] Circuit TANCAT — servei recuperat", self.name)
        self.failure_count = 0
        self.state = "CLOSED"

    def record_failure(self):
        """Registrar un error — obrir el circuit si supera el llindar."""
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = "OPEN"
            logger.warning(
                "[%s] Circuit OBERT — %d errors consecutius. "
                "Esperant %ds abans de reintentar.",
                self.name, self.failure_count, self.recovery_timeout
            )


class CircuitOpenError(Exception):
    """Excepció llançada quan el circuit està obert."""
    pass
```

Integra'l al consumer:

```python
# Al champion_consumer.py — afegir circuit breaker

from consumers.circuit_breaker import CircuitBreaker, CircuitOpenError

# Circuit breaker global per al servei LLM
llm_circuit = CircuitBreaker(name="LLM-Service", failure_threshold=5, recovery_timeout=30)

def generate_champion_summary_with_retry(champion_data: dict,
                                          max_retries: int = 3) -> str:
    """Genera resum amb retry i circuit breaker."""
    # Comprovar si el circuit està obert ABANS d'intentar
    if not llm_circuit.can_execute():
        raise CircuitOpenError("Circuit obert — servei LLM no disponible")

    try:
        result = _do_generate_with_retry(champion_data, max_retries)
        llm_circuit.record_success()
        return result
    except Exception as e:
        llm_circuit.record_failure()
        raise
```

### 4. Mesurar latència publish-to-consume (15 min)

Afegeix mesures de latència al consumer:

```python
# Al on_message, després de parsejar l'event:
from datetime import datetime

def on_message(channel, method, properties, body):
    """Callback amb mesures de latència."""
    redis_client = get_redis_client()

    try:
        event = json.loads(body)
        event_id = event.get("eventId", "unknown")
        
        # Mesurar latència end-to-end (publish → consume start)
        event_timestamp = event.get("timestamp")
        if event_timestamp:
            published_at = datetime.fromisoformat(event_timestamp.replace("Z", "+00:00"))
            latency_ms = (datetime.now(published_at.tzinfo) - published_at).total_seconds() * 1000
            logger.info("Latència publish→consume: %.0f ms (event: %s)", latency_ms, event_id)

        # ... resta del processament (deduplicació, etc.)
```

### 5. Testejar els escenaris de fallada (15 min)

```bash
# Test 1: Aturar el servei LLM i enviar un event
# El consumer ha de reintentar 3 vegades amb backoff i enviar a DLQ

# Test 2: Enviar el mateix event dos cops (simular duplicat)
# El segon processament ha de ser skipped

# Test 3: Aturar el servei LLM durant 30s i tornar-lo a arrencar
# El circuit breaker ha d'obrir-se i tancar-se quan el servei torni
```

### 6. Commit (5 min)

```bash
git add .
git commit -m "feat(resilience): add retry, idempotency and circuit breaker to consumer"
```

---

## Checklist de Lliurament

- [ ] Retry amb backoff exponencial implementat (3 intents per defecte)
- [ ] Deduplicació per event ID funciona (el segon processament és skipped)
- [ ] Circuit breaker implementat amb 3 estats (CLOSED, OPEN, HALF_OPEN)
- [ ] Missatges que fallen tots els retries van al DLQ
- [ ] Latència publish-to-consume mesurada als logs
- [ ] Testat amb servei LLM caigut (retry + DLQ funcionen)
- [ ] Testat amb missatge duplicat (deduplicació funciona)
- [ ] Commit fet
