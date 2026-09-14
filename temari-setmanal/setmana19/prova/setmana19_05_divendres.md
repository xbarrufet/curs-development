# Setmana 19 — Divendres: Tests d'Integració del Flux Sencer

## Objectiu del Dia

Escriure tests d'integració que verifiquin el flux asíncron complet: crear un campió → verificar event a la cua → verificar processament → verificar resultat a Redis. Al final del dia, tens una suite de tests que valida tota la pipeline asíncrona, i el treball setmanal està en un PR.

---

## Teoria

### Testejar Fluxos Asíncrons: El Repte

Tests unitaris són fàcils: crida una funció, comprova el resultat. Tests asíncrons són més complexos perquè:
- El resultat no existeix immediatament (cal esperar)
- Depenen d'infraestructura externa (RabbitMQ, Redis, PostgreSQL)
- L'ordre d'execució no és determinista

### Testcontainers: Infraestructura en Tests

**Testcontainers** arrencar contenidors Docker dins dels tests automàticament:

```python
# Python: testcontainers per RabbitMQ i Redis
from testcontainers.rabbitmq import RabbitMqContainer
from testcontainers.redis import RedisContainer

# Arrencar un RabbitMQ temporal per al test
with RabbitMqContainer("rabbitmq:3-management") as rabbitmq:
    # rabbitmq.get_connection_url() → URL per connectar-s'hi
    pass
```

```java
// Java: Testcontainers per PostgreSQL i RabbitMQ
@Container
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

@Container
static RabbitMQContainer rabbitmq = new RabbitMQContainer("rabbitmq:3-management");
```

### Alternativa Pràctica: docker-compose per Tests

Si Testcontainers és massa complex, una alternativa més senzilla és usar el `docker-compose` existent per als tests d'integració:

```bash
# Arrencar la infraestructura
docker-compose up -d postgres redis rabbitmq

# Executar els tests d'integració
mvn test -Pintegration   # Java
pytest tests/integration/ # Python

# Aturar
docker-compose down
```

### Polling per a Resultats Asíncrons

Com que el resultat no arriba instantàniament, cal "esperar" amb un límit:

```python
import time

def wait_for_condition(check_func, timeout=30, interval=0.5):
    """
    Espera fins que check_func() retorni True, o falla per timeout.
    Útil per tests asíncrons on el resultat no és immediat.
    """
    start = time.time()
    while time.time() - start < timeout:
        if check_func():
            return True
        time.sleep(interval)
    raise TimeoutError(f"Condició no complerta en {timeout} segons")
```

> **Lectura recomanada (opcional, no bloquejant):**
> - [Testcontainers for Python](https://testcontainers-python.readthedocs.io/) — Documentació oficial
> - [Testing Async Systems](https://www.rabbitmq.com/docs/consumer-prefetch#testing)

---

## Activitat

### 1. Preparar l'entorn de test Python (15 min)

Afegeix les dependències de test al `requirements.txt`:

```
# Testing
pytest==8.1.1
pytest-timeout==2.3.1
testcontainers[rabbitmq,redis]==4.4.0
```

```bash
pip install -r requirements.txt
```

Crea l'estructura de directoris:

```bash
mkdir -p ai-python/tests/integration
touch ai-python/tests/__init__.py
touch ai-python/tests/integration/__init__.py
```

### 2. Escriure el test d'integració Python (45 min)

Crea `ai-python/tests/integration/test_champion_flow.py`:

```python
"""
Tests d'integració per al flux asíncron de campions.
Validen que el flux complet funciona: event → consumer → LLM → Redis.

Requereix: docker-compose up -d rabbitmq redis (o Testcontainers)
"""

import json
import os
import time
import pytest
import pika
import redis

# Configuració per tests (usa els serveis de docker-compose)
RABBITMQ_HOST = os.getenv("TEST_RABBITMQ_HOST", "localhost")
RABBITMQ_PORT = int(os.getenv("TEST_RABBITMQ_PORT", "5672"))
RABBITMQ_USER = os.getenv("TEST_RABBITMQ_USER", "esports")
RABBITMQ_PASS = os.getenv("TEST_RABBITMQ_PASS", "esports_pwd")
REDIS_HOST = os.getenv("TEST_REDIS_HOST", "localhost")
REDIS_PORT = int(os.getenv("TEST_REDIS_PORT", "6379"))

# Timeout màxim per esperar resultats asíncrons (segons)
ASYNC_TIMEOUT = 30


@pytest.fixture(scope="module")
def rabbitmq_channel():
    """Fixture que proporciona un canal RabbitMQ connectat."""
    credentials = pika.PlainCredentials(RABBITMQ_USER, RABBITMQ_PASS)
    connection = pika.BlockingConnection(
        pika.ConnectionParameters(
            host=RABBITMQ_HOST,
            port=RABBITMQ_PORT,
            credentials=credentials
        )
    )
    channel = connection.channel()
    yield channel
    connection.close()


@pytest.fixture(scope="module")
def redis_client():
    """Fixture que proporciona un client Redis connectat."""
    client = redis.Redis(
        host=REDIS_HOST, port=REDIS_PORT, decode_responses=True
    )
    yield client
    client.close()


@pytest.fixture(autouse=True)
def clean_redis(redis_client):
    """Neteja les claus de test de Redis abans de cada test."""
    # Esborrar claus de test (patró champion:summary:test_*)
    for key in redis_client.scan_iter("champion:summary:test_*"):
        redis_client.delete(key)
    for key in redis_client.scan_iter("processed_events:test_*"):
        redis_client.delete(key)
    yield


def wait_for_redis_key(redis_client, key, timeout=ASYNC_TIMEOUT):
    """
    Espera fins que una clau existeixi a Redis.
    Retorna el valor si el troba, o llença TimeoutError.
    """
    start = time.time()
    while time.time() - start < timeout:
        value = redis_client.get(key)
        if value is not None:
            return json.loads(value)
        time.sleep(0.5)
    raise TimeoutError(f"Clau '{key}' no trobada a Redis en {timeout}s")


def create_test_event(champion_id: str, name: str) -> dict:
    """Crea un event de test amb ID prefixat per facilitar neteja."""
    return {
        "eventId": f"test_{champion_id}_{int(time.time())}",
        "eventType": "champion.created",
        "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
        "payload": {
            "championId": champion_id,
            "name": name,
            "role": "MID",
            "description": f"Test champion {name}"
        }
    }


class TestChampionEventFlow:
    """Tests d'integració per al flux asíncron de campions."""

    def test_event_reaches_queue(self, rabbitmq_channel):
        """Verificar que un event publicat arriba a la cua."""
        event = create_test_event("test_001", "TestChampion")

        # Publicar a l'exchange
        rabbitmq_channel.basic_publish(
            exchange="esportspulse.events",
            routing_key="champion.created",
            body=json.dumps(event),
            properties=pika.BasicProperties(
                content_type="application/json",
                delivery_mode=2  # Persistent
            )
        )

        # Verificar que el missatge arriba a la cua
        # (get amb auto_ack per netejar)
        time.sleep(1)  # Donar temps a RabbitMQ per routing
        method, properties, body = rabbitmq_channel.basic_get(
            queue="llm-processing", auto_ack=True
        )

        assert body is not None, "El missatge no ha arribat a la cua"
        received_event = json.loads(body)
        assert received_event["eventId"] == event["eventId"]
        assert received_event["payload"]["name"] == "TestChampion"

    @pytest.mark.timeout(ASYNC_TIMEOUT + 5)
    def test_full_flow_produces_redis_result(self, rabbitmq_channel,
                                              redis_client):
        """
        Test end-to-end: publicar event → consumer processa → resultat a Redis.
        NOTA: requereix que el consumer Python estigui en execució.
        """
        champion_id = "test_e2e_002"
        event = create_test_event(champion_id, "E2EChampion")

        # Publicar l'event
        rabbitmq_channel.basic_publish(
            exchange="esportspulse.events",
            routing_key="champion.created",
            body=json.dumps(event),
            properties=pika.BasicProperties(content_type="application/json")
        )

        # Esperar el resultat a Redis (el consumer ha de processar-lo)
        cache_key = f"champion:summary:{champion_id}"
        result = wait_for_redis_key(redis_client, cache_key)

        # Verificar el resultat
        assert result["championId"] == champion_id
        assert result["name"] == "E2EChampion"
        assert "summary" in result
        assert len(result["summary"]) > 0

    def test_idempotency_no_duplicate_processing(self, rabbitmq_channel,
                                                   redis_client):
        """
        Enviar el mateix event dues vegades.
        El segon no ha de re-processar (deduplicació per eventId).
        """
        event = create_test_event("test_idem_003", "IdempotentChampion")

        # Publicar el mateix event dues vegades
        for _ in range(2):
            rabbitmq_channel.basic_publish(
                exchange="esportspulse.events",
                routing_key="champion.created",
                body=json.dumps(event),
                properties=pika.BasicProperties(content_type="application/json")
            )

        # Esperar processament
        cache_key = f"champion:summary:test_idem_003"
        result = wait_for_redis_key(redis_client, cache_key)

        # Verificar que l'event està marcat com a processat
        assert redis_client.exists(f"processed_events:{event['eventId']}") > 0

    def test_invalid_json_goes_to_dlq(self, rabbitmq_channel):
        """Un missatge amb JSON invàlid ha d'anar al DLQ."""
        # Publicar un missatge invàlid
        rabbitmq_channel.basic_publish(
            exchange="esportspulse.events",
            routing_key="champion.created",
            body="THIS IS NOT JSON"
        )

        # Esperar que el consumer el rebutgi i vagi al DLQ
        time.sleep(5)
        method, properties, body = rabbitmq_channel.basic_get(
            queue="llm-processing.dlq", auto_ack=True
        )

        # Verificar que el missatge invàlid està al DLQ
        assert body is not None, "El missatge invàlid no ha anat al DLQ"
        assert body.decode() == "THIS IS NOT JSON"
```

### 3. Preparar el test d'integració Java (30 min)

Afegeix Testcontainers al `pom.xml`:

```xml
<!-- Testcontainers: contenidors Docker per tests d'integració -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>rabbitmq</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

Crea `ChampionEventIntegrationTest.java`:

```java
// Test d'integració que verifica la publicació d'events a RabbitMQ

@SpringBootTest
@Testcontainers  // Activa Testcontainers per aquesta classe
class ChampionEventIntegrationTest {

    // Contenidor PostgreSQL temporal per al test
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
        .withDatabaseName("esportspulse_test")
        .withUsername("test")
        .withPassword("test");

    // Contenidor RabbitMQ temporal per al test
    @Container
    static RabbitMQContainer rabbitmq = new RabbitMQContainer("rabbitmq:3-management")
        .withAdminPassword("test");

    // Configurar les propietats dinàmicament amb els ports dels contenidors
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.rabbitmq.host", rabbitmq::getHost);
        registry.add("spring.rabbitmq.port", rabbitmq::getAmqpPort);
    }

    @Autowired
    private ChampionService championService;

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Test
    void whenChampionCreated_thenEventPublishedToQueue() throws Exception {
        // Crear un campió
        Champion champion = new Champion();
        champion.setName("TestChampion");
        champion.setRole("MID");
        champion.setDescription("Campió de test");

        Champion saved = championService.create(champion);

        // Esperar i llegir el missatge de la cua
        Thread.sleep(1000);  // Donar temps a la publicació

        // Verificar que el missatge està a la cua
        Message message = rabbitTemplate.receive("llm-processing", 5000);
        assertNotNull(message, "No s'ha rebut cap missatge a la cua");

        // Deserialitzar i verificar el contingut
        String body = new String(message.getBody());
        assertTrue(body.contains("TestChampion"));
        assertTrue(body.contains("champion.created"));
    }
}
```

### 4. Executar els tests (15 min)

```bash
# Tests d'integració Python
# (assegura't que docker-compose up -d i el consumer estan actius)
cd ai-python
pytest tests/integration/ -v --timeout=60

# Tests d'integració Java (Testcontainers arrencarà contenidors automàticament)
cd backend-java
mvn test -Dtest=ChampionEventIntegrationTest
```

### 5. Consolidació setmanal i PR (15 min)

```bash
git add .
git commit -m "test(integration): add end-to-end tests for async champion flow"

# Push i crear PR
git push -u origin feature/week19-message-queues
```

Crea el PR amb un resum del treball setmanal:
- Dilluns: Disseny de fluxos síncrons vs asíncrons
- Dimarts: Productor Java amb Spring AMQP
- Dimecres: Consumer Python amb pika i LLM
- Dijous: Retry, idempotència i circuit breaker
- Divendres: Tests d'integració del flux complet

---

## Checklist de Lliurament

- [ ] Tests d'integració Python escrits i executats
- [ ] Test que verifica que l'event arriba a la cua
- [ ] Test end-to-end: event → consumer → Redis (requereix consumer actiu)
- [ ] Test d'idempotència: missatge duplicat no es re-processa
- [ ] Test de DLQ: missatge invàlid va al Dead Letter Queue
- [ ] Test d'integració Java amb Testcontainers (o docker-compose)
- [ ] Tots els tests passen
- [ ] PR creat amb resum del treball S19
