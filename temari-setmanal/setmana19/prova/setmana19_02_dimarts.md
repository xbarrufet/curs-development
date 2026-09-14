# Setmana 19 — Dimarts: RabbitMQ: Productor Java

## Objectiu del Dia

Afegir RabbitMQ al projecte i implementar el productor Java que publica events `champion.created` quan es crea un campió. Al final del dia, quan fas POST d'un campió, un missatge apareix a la cua de RabbitMQ visible des de la UI de gestió.

---

## Teoria

### RabbitMQ: Conceptes Bàsics

RabbitMQ és un **message broker**: rep missatges dels productors i els lliura als consumers.

```
Producer → Exchange → Binding → Queue → Consumer
   Java      (router)   (regla)   (cua)    Python
```

**Components:**

| Component | Funció | Analogia |
|-----------|--------|----------|
| **Producer** | Envia missatges | Tu envies una carta |
| **Exchange** | Decideix a quina cua va | L'oficina de correus |
| **Binding** | Regla que connecta exchange amb queue | La ruta de distribució |
| **Queue** | Emmagatzema missatges fins que un consumer els llegeix | La bústia |
| **Consumer** | Llegeix i processa missatges | El destinatari |

**Tipus d'Exchange:**
- **Direct:** Envia a la cua que coincideixi exactament amb la routing key
- **Topic:** Envia a les cues que coincideixin amb un patró (ex: `champion.*` rep `champion.created` i `champion.updated`)
- **Fanout:** Envia a totes les cues bindejades (broadcast)

Nosaltres usarem **Topic Exchange** perquè permet que múltiples consumers escoltin events diferents amb patrons.

```
Exchange "esportspulse.events" (topic)
   │
   ├── Routing key: champion.created
   │   └── Binding: champion.# → Queue "llm-processing"
   │   └── Binding: champion.# → Queue "qdrant-indexing"
   │
   └── Routing key: champion.updated
       └── Binding: champion.# → Queue "llm-processing"
```

### Spring AMQP

Spring Boot integra RabbitMQ amb `spring-boot-starter-amqp` (AMQP = Advanced Message Queuing Protocol):

```java
// RabbitTemplate és l'equivalent de RedisTemplate però per RabbitMQ
// Permet enviar missatges a un exchange amb una routing key
@Autowired
private RabbitTemplate rabbitTemplate;

// Enviar un missatge
rabbitTemplate.convertAndSend(
    "esportspulse.events",    // Nom de l'exchange
    "champion.created",        // Routing key
    eventJson                  // Contingut del missatge (JSON string)
);
```

### RabbitMQ Management UI

RabbitMQ inclou una interfície web de gestió on pots:
- Veure cues, exchanges i bindings
- Inspeccionar missatges pendents
- Monitoritzar el throughput (missatges/segon)
- Purgar cues per a debugging

URL per defecte: `http://localhost:15672` (user: guest, password: guest)

> **Lectura recomanada (opcional, no bloquejant):**
> - [RabbitMQ Tutorial — Topics](https://www.rabbitmq.com/tutorials/tutorial-five-java) — Tutorial oficial
> - [Spring AMQP Reference](https://docs.spring.io/spring-amqp/reference/) — Documentació oficial

---

## Activitat

### 1. Afegir RabbitMQ al Docker Compose (10 min)

Afegeix el servei al `docker-compose.yml`:

```yaml
  # Message broker per comunicació asíncrona entre serveis
  rabbitmq:
    image: rabbitmq:3-management    # Versió amb UI de gestió inclosa
    container_name: esportspulse-rabbitmq
    environment:
      # Credencials per defecte (en producció, canviar!)
      RABBITMQ_DEFAULT_USER: esports
      RABBITMQ_DEFAULT_PASS: esports_pwd
    ports:
      - "5672:5672"                 # Port AMQP (comunicació de missatges)
      - "15672:15672"               # Port de la UI de gestió
    # Healthcheck: verifica que RabbitMQ respon
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_port_connectivity"]
      interval: 15s
      timeout: 10s
      retries: 5
```

```bash
# Arrencar RabbitMQ
docker-compose up -d rabbitmq

# Esperar uns 30 segons perquè arrenqui completament
# Verificar que funciona obrint http://localhost:15672
# User: esports / Password: esports_pwd
```

### 2. Afegir dependència Spring AMQP (5 min)

Al `pom.xml`:

```xml
<!-- Spring AMQP — integració amb RabbitMQ -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

Afegeix a `application.properties`:

```properties
# --- RabbitMQ ---
# Host, port i credencials per connectar amb RabbitMQ
spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672
spring.rabbitmq.username=esports
spring.rabbitmq.password=esports_pwd
```

### 3. Configurar Exchange, Queues i Bindings (20 min)

Crea la configuració de RabbitMQ a Spring:

```java
// RabbitMQConfig.java — Declara l'exchange, les cues i els bindings
// Spring AMQP crea automàticament aquests elements a RabbitMQ si no existeixen

@Configuration
public class RabbitMQConfig {

    // Nom de l'exchange on es publiquen tots els events
    public static final String EXCHANGE_NAME = "esportspulse.events";
    // Nom de la cua per al processament LLM
    public static final String LLM_QUEUE = "llm-processing";
    // Nom de la cua per a la indexació a Qdrant
    public static final String QDRANT_QUEUE = "qdrant-indexing";
    // Patró de routing: qualsevol event de champion
    public static final String CHAMPION_ROUTING_PATTERN = "champion.#";

    // Exchange de tipus topic: permet routing amb patrons
    @Bean
    public TopicExchange eventsExchange() {
        // durable=true: l'exchange sobreviu a reinicis de RabbitMQ
        return new TopicExchange(EXCHANGE_NAME, true, false);
    }

    // Cua per al processament LLM
    @Bean
    public Queue llmQueue() {
        // durable=true: la cua i els seus missatges sobreviuen a reinicis
        return QueueBuilder.durable(LLM_QUEUE).build();
    }

    // Cua per a la indexació Qdrant
    @Bean
    public Queue qdrantQueue() {
        return QueueBuilder.durable(QDRANT_QUEUE).build();
    }

    // Binding: connecta la cua LLM amb l'exchange pel patró champion.#
    // Això vol dir que champion.created, champion.updated, etc. arriben a aquesta cua
    @Bean
    public Binding llmBinding(Queue llmQueue, TopicExchange eventsExchange) {
        return BindingBuilder
            .bind(llmQueue)
            .to(eventsExchange)
            .with(CHAMPION_ROUTING_PATTERN);
    }

    // Binding: connecta la cua Qdrant amb l'exchange pel mateix patró
    @Bean
    public Binding qdrantBinding(Queue qdrantQueue, TopicExchange eventsExchange) {
        return BindingBuilder
            .bind(qdrantQueue)
            .to(eventsExchange)
            .with(CHAMPION_ROUTING_PATTERN);
    }

    // Configura el converter de missatges per usar JSON
    @Bean
    public Jackson2JsonMessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }

    // Configura el RabbitTemplate per usar el converter JSON
    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory,
                                         Jackson2JsonMessageConverter converter) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(converter);
        return template;
    }
}
```

### 4. Implementar el productor (30 min)

Crea el servei que publica events:

```java
// ChampionEventPublisher.java — Publica events de campions a RabbitMQ

@Service
@Slf4j  // Anotació de Lombok per generar automàticament el logger
public class ChampionEventPublisher {

    private final RabbitTemplate rabbitTemplate;

    // Injecció via constructor
    public ChampionEventPublisher(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    /**
     * Publica un event champion.created quan es crea un campió.
     * El missatge arriba a totes les cues bindejades amb el patró champion.#
     */
    public void publishChampionCreated(Champion champion) {
        // Crear l'event amb ID únic i timestamp
        ChampionEvent event = ChampionEvent.created(champion);

        // Enviar a l'exchange amb routing key "champion.created"
        rabbitTemplate.convertAndSend(
            RabbitMQConfig.EXCHANGE_NAME,   // Exchange destí
            "champion.created",              // Routing key
            event                            // Objecte que es serialitza a JSON automàticament
        );

        log.info("Event publicat: {} per campió '{}' (eventId: {})",
            event.getEventType(), champion.getName(), event.getEventId());
    }

    /**
     * Publica un event champion.updated quan es modifica un campió.
     */
    public void publishChampionUpdated(Champion champion) {
        ChampionEvent event = ChampionEvent.updated(champion);

        rabbitTemplate.convertAndSend(
            RabbitMQConfig.EXCHANGE_NAME,
            "champion.updated",
            event
        );

        log.info("Event publicat: {} per campió '{}' (eventId: {})",
            event.getEventType(), champion.getName(), event.getEventId());
    }
}
```

### 5. Integrar el publisher al servei existent (15 min)

Modifica `ChampionService` per publicar events després de guardar:

```java
// ChampionService.java — Afegir publicació d'events

@Service
public class ChampionService {

    private final ChampionRepository repository;
    private final ChampionEventPublisher eventPublisher;

    public ChampionService(ChampionRepository repository,
                           ChampionEventPublisher eventPublisher) {
        this.repository = repository;
        this.eventPublisher = eventPublisher;
    }

    public Champion create(Champion champion) {
        // 1. Guardar a PostgreSQL (operació síncrona)
        Champion saved = repository.save(champion);

        // 2. Publicar event a RabbitMQ (no-bloquejant, molt ràpid)
        // Si RabbitMQ cau, l'excepció NO ha de fer fallar el POST
        try {
            eventPublisher.publishChampionCreated(saved);
        } catch (Exception e) {
            // Log de l'error però NO relançar — el campió ja està guardat
            log.error("Error publicant event champion.created: {}", e.getMessage());
        }

        return saved;
    }
}
```

> **Nota important:** El try-catch al publisher és una decisió de disseny. Si el POST falla perquè RabbitMQ no funciona, l'usuari no podria crear campions. Millor guardar el campió i processar l'event més tard (amb un mecanisme de recuperació que veurem dijous).

### 6. Verificar a la UI de RabbitMQ (15 min)

```bash
# Arrencar l'aplicació
mvn spring-boot:run

# Crear un campió via API
curl -X POST http://localhost:8080/api/champions \
  -H "Content-Type: application/json" \
  -d '{"name":"Zed","role":"MID","description":"Assassí amb ombres"}'
```

Obre http://localhost:15672 i verifica:
1. **Exchanges:** Existeix `esportspulse.events` (tipus topic)
2. **Queues:** Existeixen `llm-processing` i `qdrant-indexing`
3. **Missatges:** Cada cua té 1 missatge pendent (Ready: 1)
4. Clica a la cua → **Get Message** per veure el contingut JSON de l'event

### 7. Commit (5 min)

```bash
git add .
git commit -m "feat(events): add RabbitMQ producer for champion events with Spring AMQP"
```

---

## Checklist de Lliurament

- [ ] RabbitMQ funciona dins Docker amb la UI de gestió accessible
- [ ] Exchange `esportspulse.events` creat (tipus topic)
- [ ] Cues `llm-processing` i `qdrant-indexing` creades i bindejades
- [ ] `ChampionEventPublisher` implementat amb `convertAndSend`
- [ ] Events es publiquen quan es crea un campió via POST
- [ ] Missatge JSON visible a la cua des de la UI de RabbitMQ
- [ ] El POST no falla si RabbitMQ està caigut (try-catch)
- [ ] Commit fet
