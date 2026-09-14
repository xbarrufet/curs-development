# Setmana 11 — Dimarts: Structured Logging (JSON)

## Objectiu del Dia

Reemplaçar tots els `print()` de Python i `System.out.println()` de Java per **logging estructurat en format JSON**. Al final del dia, cada línia de log serà un objecte JSON amb camps estandarditzats que es poden buscar, filtrar i agregar de forma automàtica.

---

## Teoria

### Per Què els Logs de Text Fallen a Escala

Quan tens un sol servei i 10 peticions per minut, llegir logs amb `grep` funciona. Quan tens dos serveis (Java + Python) i centenars de peticions, els logs de text es converteixen en un malson:

```
# Log de text típic — com trobar què ha fallat?
2024-03-15 10:23:45 INFO Starting request processing
2024-03-15 10:23:45 ERROR Something went wrong with champion 42
2024-03-15 10:23:46 INFO Request completed in 234ms
```

**Problemes:**
- `grep "ERROR"` et dona la línia però no el context (quin usuari? quina petició?)
- No pots filtrar per servei, endpoint o temps de resposta
- No pots agregar (quantes peticions per minut? quin és el temps mitjà?)
- Cada servei formata els logs diferent → impossible unificar

### Logging Estructurat en JSON

Cada línia de log és un objecte JSON amb camps consistents:

```json
{
    "timestamp": "2024-03-15T10:23:45.123Z",
    "level": "ERROR",
    "service_name": "ai-python",
    "correlation_id": "abc-123-def",
    "message": "Champion no trobat",
    "champion_id": 42,
    "http_status": 404,
    "duration_ms": 234
}
```

**Avantatges:**
- Cada camp és buscable: `jq '.[] | select(.level == "ERROR")'`
- Pots agregar: quants errors per minut? quin endpoint és el més lent?
- Tools com Elasticsearch, Datadog, Grafana Loki els processen automàticament
- Consistència entre Java i Python

### Taula Comparativa: Text vs JSON

| Aspecte | Logs de Text | Logs JSON |
|---------|-------------|-----------|
| Llegibilitat humana | Alta (directa) | Baixa (cal formatador) |
| Cerca per camp | `grep` amb regex fràgil | `jq`, Elasticsearch: precís |
| Agregació | Manual, propensa a errors | Automàtica amb eines |
| Alertes | Difícil: regex sobre text | Fàcil: query sobre camps |
| Multi-servei | Cada servei format diferent | Format unificat |
| Cost d'implementació | Zero (print) | Baix (una vegada) |

### Camps Obligatoris per Línia de Log

Tots els logs d'EsportsPulse han de tenir aquests camps:

| Camp | Descripció | Exemple |
|------|------------|---------|
| `timestamp` | Moment exacte en ISO 8601 | `2024-03-15T10:23:45.123Z` |
| `level` | Gravetat del log | `INFO`, `WARN`, `ERROR` |
| `service_name` | Quin servei ha generat el log | `ai-python`, `backend-java` |
| `correlation_id` | ID únic per traçar la petició entre serveis | `abc-123-def` |
| `message` | Descripció de l'event | `Champion carregat correctament` |

### Python: structlog amb JSONRenderer

`structlog` és la llibreria de logging estructurat més popular de Python. Converteix cada log en JSON automàticament.

**Instal·lació:**

```bash
# Afegim structlog al projecte Python.
pip install structlog
```

**Configuració inicial** (`ai-python/src/logging_config.py`):

```python
import structlog
import logging
import sys

def configure_logging():
    """
    Configura structlog per emetre logs en format JSON.
    Cridar aquesta funció UNA SOLA VEGADA a l'inici de l'aplicació.
    """

    # Processadors: cada log passa per aquesta cadena de transformacions.
    # L'ordre importa: primer afegim camps, després formatem.
    shared_processors = [
        # Afegeix el nivell de log (INFO, ERROR...) com a camp.
        structlog.stdlib.add_log_level,
        # Afegeix el timestamp en format ISO 8601 (estàndard internacional).
        structlog.processors.TimeStamper(fmt="iso"),
        # Afegeix el nom del servei a TOTS els logs automàticament.
        # Així no cal recordar-ho cada vegada que fem log.
        structlog.processors.CallsiteParameterAdder(
            parameters=[structlog.processors.CallsiteParameter.FUNC_NAME]
        ),
    ]

    structlog.configure(
        processors=shared_processors + [
            # Formatem com a JSON. Cada camp serà una clau del JSON.
            structlog.processors.JSONRenderer(),
        ],
        # Integració amb el logging estàndard de Python.
        wrapper_class=structlog.stdlib.BoundLogger,
        # Fàbrica de loggers: caché per eficiència.
        logger_factory=structlog.stdlib.LoggerFactory(),
        cache_logger_on_first_use=True,
    )

    # Configurem el logging estàndard de Python per redirigir a structlog.
    # Així, llibreries que usen logging.getLogger() també emeten JSON.
    logging.basicConfig(
        format="%(message)s",
        stream=sys.stdout,
        level=logging.INFO,
    )
```

**Ús al codi:**

```python
import structlog

# Creem un logger vinculat al servei.
# 'bind' afegeix camps permanents a TOTES les línies d'aquest logger.
logger = structlog.get_logger().bind(service_name="ai-python")

# Abans (MAL - text pla, sense context):
# print(f"Champion {champion_id} carregat")

# Ara (BÉ - JSON estructurat amb context):
logger.info(
    "Champion carregat",                    # Missatge clar i concís.
    champion_id=champion_id,                # Camp buscable per ID.
    source="java_api",                      # D'on ve la dada.
    duration_ms=elapsed,                    # Temps de resposta.
)
# Sortida: {"timestamp": "2024-...", "level": "info", "service_name": "ai-python",
#           "message": "Champion carregat", "champion_id": 42, "source": "java_api",
#           "duration_ms": 123}

# Per errors, afegim informació de l'excepció:
logger.error(
    "Error cridant servei Java",
    error_type=type(exc).__name__,           # Tipus d'error (ConnectionError, etc).
    error_detail=str(exc),                   # Detall de l'error.
    champion_id=champion_id,                 # Context: què intentàvem fer.
)
```

### Java: SLF4J + Logback amb LogstashEncoder

A Java, SLF4J és la façana de logging i Logback és la implementació. Amb `logstash-logback-encoder`, els logs surten en JSON.

**Dependència Maven** (`pom.xml`):

```xml
<!-- Encoder que converteix els logs de Logback a format JSON.
     Sense això, Logback emet text pla. -->
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

**Configuració** (`src/main/resources/logback-spring.xml`):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <!-- Appender que escriu logs a la consola (stdout).
         Usem LogstashEncoder per emetre JSON en lloc de text. -->
    <appender name="JSON_CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <!-- Afegim el nom del servei a cada línia de log.
                 Igual que al Python, per identificar l'origen. -->
            <customFields>{"service_name":"backend-java"}</customFields>
            <!-- Incloem camps del MDC (Mapped Diagnostic Context).
                 El MDC és on guardarem el correlation_id (demà). -->
            <includeMdcKeyName>correlation_id</includeMdcKeyName>
        </encoder>
    </appender>

    <!-- Nivell mínim: INFO. Logs de DEBUG no es mostren en producció. -->
    <root level="INFO">
        <appender-ref ref="JSON_CONSOLE" />
    </root>

</configuration>
```

**Ús al codi Java:**

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import static net.logstash.logback.argument.StructuredArguments.kv;

@Service
public class ChampionService {

    // Creem el logger una vegada per classe.
    // getLogger(classe) fa que el camp 'logger_name' tingui el nom de la classe.
    private static final Logger log = LoggerFactory.getLogger(ChampionService.class);

    public ChampionDTO findById(Long id) {
        // kv() crea parells clau-valor que apareixen com a camps JSON.
        // Sense kv(), el log seria text pla dins del camp 'message'.
        log.info("Buscant champion", kv("champion_id", id));

        Optional<Champion> champion = championRepository.findById(id);

        if (champion.isEmpty()) {
            // WARN perquè no és un error del sistema, però cal investigar
            // si passa sovint (potser el client envia IDs invàlids).
            log.warn("Champion no trobat", kv("champion_id", id));
            throw new ChampionNotFoundException(id);
        }

        log.info("Champion trobat",
            kv("champion_id", id),
            kv("champion_name", champion.get().getName())
        );
        return toDTO(champion.get());
    }
}
```

**Exemple de sortida JSON de Java:**

```json
{
    "@timestamp": "2024-03-15T10:23:45.123+00:00",
    "level": "INFO",
    "logger_name": "com.esportspulse.engine.service.ChampionService",
    "message": "Champion trobat",
    "service_name": "backend-java",
    "champion_id": 42,
    "champion_name": "Jinx"
}
```

### Bones Pràctiques de Logging

```python
# BÉ: Missatge descriptiu + camps estructurats
logger.info("Petició completada", endpoint="/champions", status=200, duration_ms=45)

# MALAMENT: Tot dins del missatge (no es pot buscar per camp)
logger.info(f"Petició a /champions completada amb status 200 en 45ms")

# BÉ: Nivell correcte per a cada situació
logger.debug("Dades crus de Java", raw_response=data)       # Debug: per desenvolupament
logger.info("Champion carregat", champion_id=42)             # Info: operació normal
logger.warning("Retry necessari", attempt=2, max_attempts=3) # Warning: no és error però atenció
logger.error("Servei Java no disponible", error=str(exc))    # Error: alguna cosa ha fallat

# MALAMENT: Tot com a ERROR (impossible filtrar)
logger.error("Champion carregat")   # Això NO és un error!
```

---

## Activitat

### Pas 1: Configurar structlog a Python

1. Instal·la `structlog`: `pip install structlog`
2. Crea `ai-python/src/logging_config.py` amb la configuració de la teoria
3. Crida `configure_logging()` a l'inici de `main.py`
4. Reemplaça tots els `print()` per crides a structlog

### Pas 2: Configurar LogstashEncoder a Java

1. Afegeix la dependència `logstash-logback-encoder` al `pom.xml`
2. Crea `logback-spring.xml` a `src/main/resources/`
3. Reemplaça tots els `System.out.println()` per `log.info()` / `log.error()` amb `kv()`

### Pas 3: Verificar el Format

```bash
# Arrencar els dos serveis i fer una petició.
curl http://localhost:8000/api/champions/1

# Verificar que els logs són JSON vàlid:
# Python:
python -c "import json; json.loads('{...}')"  # Ha de funcionar

# Java — filtrar amb jq:
# Si tens jq instal·lat:
curl http://localhost:8080/api/champions/1 2>&1 | jq .
```

### Pas 4: Buscar un Error als Logs

Simula un error (apaga Java, crida Python) i després busca'l:

```bash
# Filtrar només errors del servei Python:
cat logs.json | jq 'select(.level == "error" and .service_name == "ai-python")'
```

---

## Checklist de Lliurament

- [ ] `structlog` instal·lat i configurat a Python amb `JSONRenderer`
- [ ] `logstash-logback-encoder` configurat a Java amb `logback-spring.xml`
- [ ] Zero `print()` o `System.out.println()` al codi (tots reemplaçats)
- [ ] Cada línia de log conté: `timestamp`, `level`, `service_name`, `message`
- [ ] Logs de Python i Java surten en format JSON vàlid
- [ ] Camps addicionals de context (champion_id, duration_ms, etc.) presents
- [ ] Commit amb missatge: `feat(logging): replace print/sysout with structured JSON logging`
