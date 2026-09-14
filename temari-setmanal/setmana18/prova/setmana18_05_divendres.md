# Setmana 18 — Divendres: Cache-Aside amb Redis i Consolidació

## Objectiu del Dia

Implementar el patró cache-aside amb Redis per accelerar les queries més costoses. Entendre quan té sentit fer cache i quan no. Al final del dia, l'endpoint més lent respon des de cache en menys de 5ms i tot el treball de la setmana està commistat en un PR.

---

## Teoria

### El Patró Cache-Aside

El patró **cache-aside** (o "lazy loading") funciona així:

```
Client → Servei → Cache (Redis)
                    ├── HIT  → Retorna directament (ràpid, ~1ms)
                    └── MISS → Query a PostgreSQL → Guarda a Redis → Retorna
```

**Flux complet:**

```
1. El client fa GET /api/champions/stats?role=MID
2. El servei busca la clau "champions:stats:MID" a Redis
3a. Si EXISTEIX (cache hit):
    → Retorna el valor des de Redis (~1ms)
3b. Si NO EXISTEIX (cache miss):
    → Fa la query a PostgreSQL (~50-200ms)
    → Guarda el resultat a Redis amb TTL de 5 minuts
    → Retorna el resultat al client
4. Quan algú modifica un campió (POST/PUT/DELETE):
    → Invalida les claus de cache afectades
```

### TTL (Time To Live)

El **TTL** és el temps que una entrada de cache és vàlida. Passat el TTL, Redis l'esborra automàticament.

```
TTL curt (30s-1min):  Dades que canvien sovint. Més misses, més fresc.
TTL mitjà (5-15min):  Bon equilibri per a la majoria de casos.
TTL llarg (1h-24h):   Dades que quasi no canvien (ex: llista de rols).
```

**Triar el TTL correcte:**
- Pregunta't: "Quant de temps poden els usuaris veure dades velles sense problemes?"
- Si la resposta és "ni un segon" → no facis cache
- Si la resposta és "uns minuts" → TTL de 5 minuts

### Quan Fer Cache (i Quan No)

| Fer cache | NO fer cache |
|-----------|-------------|
| Queries costoses (JOINs complexos) | Dades que canvien cada segon |
| Dades llegides molt sovint | Dades personalitzades per usuari (difícil invalidar) |
| Dades que canvien poc | Operacions d'escriptura |
| Respostes d'APIs externes (LLM!) | Dades que han de ser 100% fresques |

### Redis amb Spring Boot

Spring Boot integra Redis de forma nativa amb `spring-boot-starter-data-redis`:

```java
// RedisTemplate permet operacions directes amb Redis
// Tipus: <String, String> — clau i valor són strings (el valor és JSON serialitzat)
@Autowired
private RedisTemplate<String, String> redisTemplate;

// Operacions bàsiques:
// Escriure amb TTL
redisTemplate.opsForValue().set("clau", "valor", Duration.ofMinutes(5));

// Llegir
String valor = redisTemplate.opsForValue().get("clau");

// Esborrar (invalidar)
redisTemplate.delete("clau");
```

> **Lectura recomanada (opcional, no bloquejant):**
> - [Caching Strategies](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Strategies.html) — Amazon (cache-aside, write-through, etc.)
> - [Spring Boot Redis Cache](https://www.baeldung.com/spring-boot-redis-cache)

---

## Activitat

### 1. Afegir Redis al Docker Compose (10 min)

Afegeix el servei Redis al `docker-compose.yml`:

```yaml
  # Servei de cache Redis
  redis:
    image: redis:7-alpine          # Versió lleugera (Alpine Linux)
    container_name: esportspulse-redis
    ports:
      - "6379:6379"                # Port per defecte de Redis
    # Healthcheck per verificar que Redis respon
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
```

```bash
# Arrencar Redis
docker-compose up -d redis

# Verificar que funciona
docker exec -it esportspulse-redis redis-cli ping
# Ha de respondre: PONG
```

### 2. Configurar Spring Boot per Redis (15 min)

Afegeix la dependència al `pom.xml`:

```xml
<!-- Spring Data Redis — integració nativa amb Redis -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<!-- Jackson per serialitzar objectes Java a JSON per guardar a Redis -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>
```

Afegeix a `application.properties`:

```properties
# --- Redis ---
# Host i port on corre Redis (el contenidor Docker)
spring.data.redis.host=localhost
spring.data.redis.port=6379
```

Crea la configuració de Redis:

```java
// RedisConfig.java — Configura com Spring Boot es connecta a Redis
// i com serialitza/deserialitza els objectes

@Configuration  // Indica a Spring que aquesta classe conté beans de configuració
public class RedisConfig {

    @Bean  // Registra un RedisTemplate personalitzat al context de Spring
    public RedisTemplate<String, String> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, String> template = new RedisTemplate<>();
        // Connecta el template a la fàbrica de connexions (configurat via application.properties)
        template.setConnectionFactory(factory);
        // Usa StringRedisSerializer per claus i valors (més llegible que el serialitzador per defecte)
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new StringRedisSerializer());
        return template;
    }
}
```

### 3. Implementar cache-aside al servei (40 min)

Identifica la query més costosa (probablement l'estadística per rol amb JOINs):

```java
// ChampionStatsService.java — Servei amb cache-aside

@Service
public class ChampionStatsService {

    // Template per operar amb Redis
    private final RedisTemplate<String, String> redisTemplate;
    // Repository per accedir a PostgreSQL
    private final ChampionRepository championRepository;
    // ObjectMapper per convertir objectes Java <-> JSON
    private final ObjectMapper objectMapper;

    // Prefix per a les claus de cache (facilita la invalidació)
    private static final String CACHE_PREFIX = "champions:stats:";
    // Temps de vida de la cache: 5 minuts
    private static final Duration CACHE_TTL = Duration.ofMinutes(5);

    // Injecció de dependències via constructor (millor que @Autowired als camps)
    public ChampionStatsService(RedisTemplate<String, String> redisTemplate,
                                 ChampionRepository championRepository,
                                 ObjectMapper objectMapper) {
        this.redisTemplate = redisTemplate;
        this.championRepository = championRepository;
        this.objectMapper = objectMapper;
    }

    /**
     * Obté estadístiques de campions per rol amb cache-aside.
     * Flux: Redis → si miss → PostgreSQL → guarda a Redis → retorna
     */
    public List<ChampionStatsDto> getStatsByRole(String role) {
        String cacheKey = CACHE_PREFIX + role.toUpperCase();

        // 1. Intentar llegir de cache
        String cached = redisTemplate.opsForValue().get(cacheKey);

        if (cached != null) {
            // Cache HIT — deserialitzar i retornar
            log.info("Cache HIT per clau: {}", cacheKey);
            return deserializeList(cached);
        }

        // 2. Cache MISS — query a PostgreSQL
        log.info("Cache MISS per clau: {}. Consultant PostgreSQL...", cacheKey);
        List<ChampionStatsDto> stats = championRepository.findStatsByRole(role);

        // 3. Guardar a Redis amb TTL
        String json = serialize(stats);
        redisTemplate.opsForValue().set(cacheKey, json, CACHE_TTL);

        return stats;
    }

    /**
     * Invalida la cache quan es modifiquen dades de campions.
     * Cridat des del controller quan es fa POST/PUT/DELETE.
     */
    public void invalidateCache(String role) {
        String cacheKey = CACHE_PREFIX + role.toUpperCase();
        redisTemplate.delete(cacheKey);
        log.info("Cache invalidada per clau: {}", cacheKey);
    }

    // Serialitza una llista d'objectes a JSON string
    private String serialize(List<ChampionStatsDto> stats) {
        try {
            return objectMapper.writeValueAsString(stats);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Error serialitzant a JSON", e);
        }
    }

    // Deserialitza un JSON string a llista d'objectes
    private List<ChampionStatsDto> deserializeList(String json) {
        try {
            return objectMapper.readValue(json,
                new TypeReference<List<ChampionStatsDto>>() {});
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Error deserialitzant JSON", e);
        }
    }
}
```

### 4. Invalidar cache a les escriptures (15 min)

Al controller, invalida la cache quan es modifiquen dades:

```java
// ChampionController.java — invalidar cache a les operacions d'escriptura

@PostMapping
public ResponseEntity<Champion> createChampion(@RequestBody Champion champion) {
    Champion saved = championService.save(champion);
    // Invalida la cache del rol afectat perquè les dades han canviat
    statsService.invalidateCache(saved.getRole());
    return ResponseEntity.status(HttpStatus.CREATED).body(saved);
}

@PutMapping("/{id}")
public ResponseEntity<Champion> updateChampion(@PathVariable Long id,
                                                @RequestBody Champion champion) {
    Champion updated = championService.update(id, champion);
    // Invalida la cache del rol anterior i del nou (per si ha canviat de rol)
    statsService.invalidateCache(champion.getRole());
    return ResponseEntity.ok(updated);
}
```

### 5. Mesurar l'impacte (15 min)

```bash
# Primera crida (cache miss) — mesurar temps amb curl
time curl -s http://localhost:8080/api/champions/stats?role=MID | jq

# Segona crida (cache hit) — ha de ser molt més ràpid
time curl -s http://localhost:8080/api/champions/stats?role=MID | jq

# Verificar que la clau existeix a Redis
docker exec -it esportspulse-redis redis-cli GET "champions:stats:MID"

# Verificar el TTL restant de la clau
docker exec -it esportspulse-redis redis-cli TTL "champions:stats:MID"
```

Hauries de veure una diferència significativa entre la primera crida (50-200ms) i la segona (~1-5ms).

### 6. Consolidació setmanal i PR (15 min)

Revisa tot el treball de la setmana 18:
- V1: Creació taula champions (migració d'H2)
- V2: Patches i stats normalitzades
- V3: Players
- V4: Dades seed
- V5: Dades massives
- V6: Índexs
- Cache-aside amb Redis

```bash
git add .
git commit -m "feat(cache): implement cache-aside pattern with Redis for champion stats"

# Crear el PR setmanal
git push -u origin feature/week18-sql-postgres
# Crea el PR a GitHub amb un resum del que has fet aquesta setmana
```

---

## Checklist de Lliurament

- [ ] Redis funciona dins Docker i respon a `PING`
- [ ] Dependència `spring-boot-starter-data-redis` afegida
- [ ] Patró cache-aside implementat al servei d'estadístiques
- [ ] Cache hit retorna en menys de 5ms
- [ ] Cache s'invalida quan es creen o modifiquen campions
- [ ] TTL configurat (5 minuts)
- [ ] Mesures de latència documentades (amb cache vs sense cache)
- [ ] Tots els commits de la setmana fets
- [ ] PR creat amb resum del treball S18
