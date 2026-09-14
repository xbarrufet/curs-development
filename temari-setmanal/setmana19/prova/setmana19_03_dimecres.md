# Setmana 19 — Dimecres: Consumidor Python: Processar amb LLM

## Objectiu del Dia

Implementar el consumer Python que escolta events de RabbitMQ, genera un resum del campió amb l'LLM i el guarda a Redis. Al final del dia, el flux complet funciona: POST campió a Java → event a RabbitMQ → consumer Python → LLM → resultat a Redis.

---

## Teoria

### El Consumer: Escoltar i Processar

Un consumer és un procés que es queda escoltant una cua de RabbitMQ. Quan arriba un missatge:

```
1. RabbitMQ lliura el missatge al consumer
2. El consumer el processa (crida l'LLM, guarda dades, etc.)
3. El consumer envia un ACK (acknowledge) a RabbitMQ
4. RabbitMQ esborra el missatge de la cua

Si el consumer falla ABANS de l'ACK:
→ RabbitMQ re-envia el missatge a un altre consumer (o al mateix quan torni)
→ El missatge NO es perd
```

### Pika: Client Python per RabbitMQ

`pika` és la llibreria estàndard de Python per connectar-se a RabbitMQ:

```python
import pika

# Connectar a RabbitMQ
connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost', port=5672,
        credentials=pika.PlainCredentials('esports', 'esports_pwd'))
)
channel = connection.channel()

# Definir la funció que processa cada missatge
def callback(ch, method, properties, body):
    event = json.loads(body)
    # ... processar l'event ...
    ch.basic_ack(delivery_tag=method.delivery_tag)  # Confirmar processament

# Subscriure's a la cua
channel.basic_consume(queue='llm-processing', on_message_callback=callback)
channel.start_consuming()  # Bloquejar i esperar missatges
```

### ACK Manual vs Automàtic

```python
# AUTO ACK (perillós): RabbitMQ esborra el missatge immediatament
# Si el consumer falla, el missatge es PERD
channel.basic_consume(queue='llm-processing',
                      on_message_callback=callback,
                      auto_ack=True)  # ← MAI en producció!

# MANUAL ACK (correcte): el consumer confirma quan ha acabat
channel.basic_consume(queue='llm-processing',
                      on_message_callback=callback,
                      auto_ack=False)  # ← Per defecte i correcte

# Dins el callback:
ch.basic_ack(delivery_tag=method.delivery_tag)    # Tot OK
ch.basic_nack(delivery_tag=method.delivery_tag,
              requeue=False)                        # Error → Dead Letter Queue
```

### Dead Letter Queue (DLQ)

Quan un missatge no es pot processar (error permanent, format invàlid, etc.), no volem que es re-intenti infinitament. El DLQ és una cua especial on van els missatges "morts":

```
Queue "llm-processing"
   │
   ├── Missatge processat correctament → ACK → esborrat
   │
   └── Missatge falla 3 vegades → NACK (requeue=false) → DLQ
                                                          ↓
                                          Queue "llm-processing.dlq"
                                          (inspeccionar manualment)
```

> **Lectura recomanada (opcional, no bloquejant):**
> - [Pika Documentation](https://pika.readthedocs.io/) — Documentació oficial
> - [RabbitMQ Dead Letter Exchanges](https://www.rabbitmq.com/docs/dlx)

---

## Activitat

### 1. Configurar el DLQ a Java (15 min)

Modifica `RabbitMQConfig.java` per afegir el Dead Letter Exchange:

```java
// Afegir al RabbitMQConfig.java existent

// Dead Letter Exchange — on van els missatges que fallen
public static final String DLX_EXCHANGE = "esportspulse.events.dlx";
public static final String LLM_DLQ = "llm-processing.dlq";

@Bean
public DirectExchange deadLetterExchange() {
    // Exchange directe per al DLQ (no cal patrons, els missatges van directament)
    return new DirectExchange(DLX_EXCHANGE);
}

@Bean
public Queue llmDeadLetterQueue() {
    return QueueBuilder.durable(LLM_DLQ).build();
}

@Bean
public Binding dlqBinding(Queue llmDeadLetterQueue, DirectExchange deadLetterExchange) {
    return BindingBuilder
        .bind(llmDeadLetterQueue)
        .to(deadLetterExchange)
        .with(LLM_QUEUE);  // Routing key = nom de la cua original
}

// Modificar la cua LLM per configurar el DLQ
@Bean
public Queue llmQueue() {
    return QueueBuilder.durable(LLM_QUEUE)
        // Quan un missatge és rebutjat (NACK sense requeue), va al DLX
        .withArgument("x-dead-letter-exchange", DLX_EXCHANGE)
        // Routing key per al DLX
        .withArgument("x-dead-letter-routing-key", LLM_QUEUE)
        .build();
}
```

> **Nota:** Si la cua ja existeix sense arguments DLQ, esborra-la des de la UI de RabbitMQ i reinicia l'aplicació per recrear-la amb els nous arguments.

### 2. Instal·lar dependències Python (10 min)

Afegeix al `requirements.txt`:

```
pika==1.3.2
redis==5.0.1
httpx==0.27.0
python-dotenv==1.0.1
```

```bash
# Activar l'entorn virtual i instal·lar
cd ai-python
source .venv/bin/activate
pip install -r requirements.txt
```

### 3. Implementar el consumer Python (45 min)

Crea `ai-python/src/consumers/champion_consumer.py`:

```python
"""
Consumer de RabbitMQ per processar events de campions.
Escolta la cua 'llm-processing', genera resums amb l'LLM
i guarda els resultats a Redis.
"""

import json
import logging
import os
import time
from datetime import timedelta

import pika
import redis
import httpx
from dotenv import load_dotenv

# Carregar variables d'entorn des de .env
load_dotenv()

# Configurar logging per veure què fa el consumer
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s"
)
logger = logging.getLogger(__name__)

# --- Configuració ---
RABBITMQ_HOST = os.getenv("RABBITMQ_HOST", "localhost")
RABBITMQ_PORT = int(os.getenv("RABBITMQ_PORT", "5672"))
RABBITMQ_USER = os.getenv("RABBITMQ_USER", "esports")
RABBITMQ_PASS = os.getenv("RABBITMQ_PASS", "esports_pwd")
QUEUE_NAME = "llm-processing"

REDIS_HOST = os.getenv("REDIS_HOST", "localhost")
REDIS_PORT = int(os.getenv("REDIS_PORT", "6379"))
# TTL per al cache del resum generat per l'LLM (1 hora)
CACHE_TTL = timedelta(hours=1)

# URL del servei LLM (el FastAPI que ja tenim de setmanes anteriors)
LLM_SERVICE_URL = os.getenv("LLM_SERVICE_URL", "http://localhost:8000")


def get_redis_client() -> redis.Redis:
    """Crea i retorna un client Redis."""
    return redis.Redis(host=REDIS_HOST, port=REDIS_PORT, decode_responses=True)


def generate_champion_summary(champion_data: dict) -> str:
    """
    Crida al servei LLM per generar un resum del campió.
    Utilitza el FastAPI existent de les setmanes anteriors.
    """
    prompt = (
        f"Genera un resum breu (2-3 frases) del campió de League of Legends "
        f"'{champion_data['name']}' que juga al rol '{champion_data['role']}'. "
        f"Descripció base: {champion_data.get('description', 'No disponible')}. "
        f"Inclou consells bàsics de gameplay."
    )

    # Cridar al endpoint /generate del servei FastAPI
    response = httpx.post(
        f"{LLM_SERVICE_URL}/api/generate",
        json={"prompt": prompt},
        timeout=60.0  # Timeout generós perquè l'LLM pot trigar
    )
    response.raise_for_status()
    return response.json()["response"]


def process_champion_event(event: dict, redis_client: redis.Redis) -> None:
    """
    Processa un event de campió:
    1. Genera un resum amb l'LLM
    2. Guarda el resum a Redis
    """
    event_type = event.get("eventType", "unknown")
    payload = event.get("payload", {})
    champion_name = payload.get("name", "unknown")

    logger.info("Processant event '%s' per campió '%s'", event_type, champion_name)

    # Generar resum amb l'LLM
    start_time = time.time()
    summary = generate_champion_summary(payload)
    elapsed = time.time() - start_time
    logger.info("Resum generat en %.2f segons per '%s'", elapsed, champion_name)

    # Guardar a Redis amb un TTL
    cache_key = f"champion:summary:{payload.get('championId', 'unknown')}"
    cache_value = json.dumps({
        "championId": payload.get("championId"),
        "name": champion_name,
        "summary": summary,
        "generatedAt": event.get("timestamp"),
        "processingTimeMs": int(elapsed * 1000)
    })
    redis_client.setex(cache_key, CACHE_TTL, cache_value)
    logger.info("Resum guardat a Redis amb clau '%s' (TTL: %s)", cache_key, CACHE_TTL)


def on_message(channel, method, properties, body):
    """
    Callback que s'executa per cada missatge rebut de la cua.
    Processa l'event i envia ACK o NACK segons el resultat.
    """
    redis_client = get_redis_client()

    try:
        # Deserialitzar el missatge JSON
        event = json.loads(body)
        event_id = event.get("eventId", "unknown")
        logger.info("Rebut event: %s (ID: %s)", event.get("eventType"), event_id)

        # Processar l'event
        process_champion_event(event, redis_client)

        # ACK: confirmar que el missatge s'ha processat correctament
        channel.basic_ack(delivery_tag=method.delivery_tag)
        logger.info("Event %s processat correctament. ACK enviat.", event_id)

    except json.JSONDecodeError as e:
        # Missatge amb format invàlid — enviar al DLQ (no requeue)
        logger.error("Error descodificant JSON: %s", e)
        channel.basic_nack(delivery_tag=method.delivery_tag, requeue=False)

    except httpx.HTTPError as e:
        # Error cridant l'LLM — requeue per reintentar
        logger.error("Error cridant l'LLM: %s. Re-enqueuant missatge.", e)
        channel.basic_nack(delivery_tag=method.delivery_tag, requeue=True)

    except Exception as e:
        # Error inesperat — enviar al DLQ per inspeccionar manualment
        logger.error("Error inesperat processant event: %s", e)
        channel.basic_nack(delivery_tag=method.delivery_tag, requeue=False)


def main():
    """Punt d'entrada del consumer. Connecta a RabbitMQ i comença a escoltar."""
    logger.info("Iniciant consumer LLM. Connectant a RabbitMQ (%s:%s)...",
                RABBITMQ_HOST, RABBITMQ_PORT)

    # Configurar credencials
    credentials = pika.PlainCredentials(RABBITMQ_USER, RABBITMQ_PASS)
    parameters = pika.ConnectionParameters(
        host=RABBITMQ_HOST,
        port=RABBITMQ_PORT,
        credentials=credentials,
        # Heartbeat per mantenir la connexió viva durant processaments llargs
        heartbeat=600
    )

    # Connectar a RabbitMQ
    connection = pika.BlockingConnection(parameters)
    channel = connection.channel()

    # Prefetch: només processar 1 missatge a la vegada
    # (evita que un consumer lent acumuli massa missatges)
    channel.basic_qos(prefetch_count=1)

    # Subscriure's a la cua amb ACK manual
    channel.basic_consume(
        queue=QUEUE_NAME,
        on_message_callback=on_message,
        auto_ack=False  # ACK manual — el consumer confirma quan ha acabat
    )

    logger.info("Consumer LLM escoltant la cua '%s'. Ctrl+C per aturar.", QUEUE_NAME)
    try:
        channel.start_consuming()
    except KeyboardInterrupt:
        logger.info("Aturant consumer...")
        channel.stop_consuming()
    finally:
        connection.close()
        logger.info("Connexió tancada.")


if __name__ == "__main__":
    main()
```

### 4. Testejar el flux complet (20 min)

Obre tres terminals:

**Terminal 1 — Backend Java:**
```bash
mvn spring-boot:run
```

**Terminal 2 — Consumer Python:**
```bash
cd ai-python
source .venv/bin/activate
python src/consumers/champion_consumer.py
```

**Terminal 3 — Enviar peticions:**
```bash
# Crear un campió
curl -X POST http://localhost:8080/api/champions \
  -H "Content-Type: application/json" \
  -d '{"name":"Yasuo","role":"MID","description":"Samurai del vent amb alta mobilitat"}'

# Esperar uns segons i verificar el resum a Redis
docker exec -it esportspulse-redis redis-cli GET "champion:summary:*"

# Llistar totes les claus de resum
docker exec -it esportspulse-redis redis-cli KEYS "champion:summary:*"

# Llegir un resum concret (substitueix l'ID pel real)
docker exec -it esportspulse-redis redis-cli GET "champion:summary:42"
```

**Verificar a la UI de RabbitMQ (http://localhost:15672):**
- La cua `llm-processing` ha de tenir 0 missatges pendents (processats)
- El comptador "Total" ha d'haver pujat

### 5. Commit (5 min)

```bash
git add .
git commit -m "feat(consumer): implement Python consumer for LLM processing via RabbitMQ"
```

---

## Checklist de Lliurament

- [ ] Consumer Python implementat amb `pika`
- [ ] ACK manual configurat (no auto_ack)
- [ ] Dead Letter Queue configurada per a missatges fallits
- [ ] Consumer escolta la cua `llm-processing` i processa events
- [ ] Resum generat per l'LLM i guardat a Redis amb TTL
- [ ] Flux complet testat: POST → event → consumer → LLM → Redis
- [ ] Errors d'LLM fan requeue; errors de format van al DLQ
- [ ] `prefetch_count=1` configurat
- [ ] Commit fet
