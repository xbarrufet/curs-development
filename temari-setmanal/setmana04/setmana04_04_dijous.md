# Setmana 4 — Dijous: Correlation IDs i Tracabilitat en Sistemes Concurrents

## Objectiu del Dia

Entendre com depurar problemes quan tens desenes de peticions concurrents barrejades als logs. Aprendras a implementar Correlation IDs — identificadors unics que segueixen una peticio des que entra al sistema fins que surt, passant per tots els serveis que toca. Al final del dia sabras afegir tracabilitat al teu codi concurrent i tindras la mentalitat de "produccion first": no n'hi ha prou que el codi funcioni, ha de ser **diagnosticable** quan falla.

---

## Teoria

### El Problema: Logs Caotiques en Sistemes Concurrents

Ahir i abans d'ahir has fet crides paral-leles a APIs. Ara imagina que una d'elles falla en produccio. Obres els logs i veus aixo:

```
INFO  fetching champion data...
INFO  fetching champion data...
ERROR connection timeout
INFO  fetching champion data...
INFO  champion data received
ERROR null pointer at ChampionService:42
INFO  champion data received
INFO  fetching champion data...
```

**Preguntes impossibles de respondre:**
- Quin dels 50 campions ha causat el timeout?
- El `NullPointerException` es del mateix request que el timeout o d'un altre?
- Quin usuari esta afectat?

Sense tracabilitat, depurar sistemes concurrents es com buscar una agulla en un paller.

### Que es un Correlation ID?

Un **Correlation ID** (o Request ID, Trace ID) es un identificador unic que s'assigna a cada peticio quan entra al sistema. Totes les operacions d'aquella peticio — logs, crides a APIs externes, queries a BD — inclouen aquest ID.

```
ABANS (sense Correlation ID):
INFO  fetching champion data...
ERROR connection timeout

DESPRÉS (amb Correlation ID):
INFO  [req-a1b2c3] fetching champion data for Ahri
ERROR [req-a1b2c3] connection timeout calling Riot API (champion=Ahri, elapsed=5003ms)
INFO  [req-d4e5f6] fetching champion data for Zed
INFO  [req-d4e5f6] champion data received (champion=Zed, elapsed=287ms)
```

Ara pots filtrar per `req-a1b2c3` i veure TOT el recorregut d'aquella peticio fallida.

### Anatomia d'un Bon Log en Produccio

Un log util en un sistema concurrent ha de tenir:

```
[NIVELL] [TIMESTAMP] [CORRELATION_ID] [CONTEXT] missatge (detalls mesurables)
```

```
INFO  2024-03-15T10:23:45.123 [req-a1b2c3] [ChampionExtractor] Starting extraction (champion=Ahri)
INFO  2024-03-15T10:23:45.124 [req-a1b2c3] [RiotClient] Calling Riot API (url=https://api.riot/...)
WARN  2024-03-15T10:23:50.127 [req-a1b2c3] [RiotClient] Riot API slow response (elapsed=5003ms, threshold=2000ms)
ERROR 2024-03-15T10:23:50.128 [req-a1b2c3] [ChampionExtractor] Extraction failed (champion=Ahri, cause=ConnectionTimeout)
```

**Regles per a bons logs:**
1. **Sempre inclou el Correlation ID** — es l'unica manera de correlacionar events
2. **Inclou context mesurable** — temps, IDs, URLs, no missatges vagues
3. **Mai loggeges dades sensibles** — no tokens, no contrasenyes, no dades personals
4. **Usa nivells correctament** — ERROR per coses que requereixen accio, WARN per anomalies, INFO per flux normal

### Com s'Implementa: El Patro

#### En Java (amb `ThreadLocal`)

`ThreadLocal` es una variable que te un valor diferent per a cada thread — perfecta per a Correlation IDs en un model thread-per-request:

```java
// RequestContext.java
// Emmagatzema el Correlation ID del request actual
// ThreadLocal garanteix que cada thread (= cada request) té el seu propi valor
// Cap thread veu el Correlation ID d'un altre thread

import java.util.UUID;

public class RequestContext {
    // ThreadLocal: cada thread veu una còpia independent d'aquesta variable
    // Quan Thread-1 escriu "req-abc", Thread-2 encara veu null (o el seu propi valor)
    private static final ThreadLocal<String> correlationId = new ThreadLocal<>();

    // Genera un nou ID únic (UUID) i l'associa al thread actual
    public static String initCorrelationId() {
        String id = "req-" + UUID.randomUUID().toString().substring(0, 8);
        correlationId.set(id);
        return id;
    }

    // Retorna el Correlation ID del thread actual
    public static String getCorrelationId() {
        String id = correlationId.get();
        return id != null ? id : "no-correlation-id";
    }

    // IMPORTANT: netejar quan el request acaba
    // Si no ho fas, el thread pot ser reutilitzat pel pool
    // i el següent request heretarà el Correlation ID antic
    public static void clear() {
        correlationId.remove();
    }
}
```

#### En Python (amb `contextvars`)

Python 3.7+ te `contextvars`, que funciona com `ThreadLocal` pero tambe funciona correctament amb `asyncio`:

```python
# request_context.py
# Equivalent al RequestContext.java però per Python
# contextvars funciona tant amb threading com amb asyncio

import uuid
import contextvars

# ContextVar és l'equivalent de ThreadLocal en Python
# A diferència de threading.local(), funciona correctament amb asyncio
# Cada task async té la seva pròpia còpia del valor
correlation_id: contextvars.ContextVar[str] = contextvars.ContextVar(
    "correlation_id", default="no-correlation-id"
)

def init_correlation_id() -> str:
    """Genera i assigna un nou Correlation ID."""
    new_id = f"req-{uuid.uuid4().hex[:8]}"
    correlation_id.set(new_id)
    return new_id

def get_correlation_id() -> str:
    """Retorna el Correlation ID del context actual."""
    return correlation_id.get()
```

### Propagacio del Correlation ID a Crides Externes

Quan el teu servei crida una API externa, ha d'enviar el Correlation ID com a header HTTP. Aixi, si l'altre servei te logs, pots correlacionar les dues bandes:

```
El teu servei                       API externa
──────────────                     ───────────
[req-a1b2c3] crido Riot API
  → Header: X-Correlation-ID: req-a1b2c3
                                    [req-a1b2c3] request rebut
                                    [req-a1b2c3] processant...
                                    [req-a1b2c3] retornant resposta
[req-a1b2c3] resposta rebuda
```

---

## Activitat

### 1. Logger amb Correlation ID en Java (25 min)

Implementa un logger que inclou automaticament el Correlation ID:

```java
// CorrelatedLogger.java
// Logger senzill que inclou automàticament el Correlation ID en cada missatge
// A producció usaríem SLF4J + MDC, però el concepte és el mateix

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

public class CorrelatedLogger {
    private final String component;
    private static final DateTimeFormatter FMT =
        DateTimeFormatter.ofPattern("HH:mm:ss.SSS");

    // Cada instància del logger s'associa a un component (classe/servei)
    // Així sabem d'on ve cada línia de log
    public CorrelatedLogger(String component) {
        this.component = component;
    }

    // Mètode central: construeix la línia de log amb tota la informació
    private void log(String level, String message) {
        String timestamp = LocalDateTime.now().format(FMT);
        String corrId = RequestContext.getCorrelationId();
        // Format: NIVELL TIMESTAMP [CORR_ID] [COMPONENT] missatge
        System.out.printf("%-5s %s [%s] [%s] %s%n",
            level, timestamp, corrId, component, message);
    }

    public void info(String message) { log("INFO", message); }
    public void warn(String message) { log("WARN", message); }
    public void error(String message) { log("ERROR", message); }
}
```

### 2. Extractor amb Tracabilitat (25 min)

Ara integra el Correlation ID a l'extractor de campions:

```java
// TracedExtractor.java
// Extractor concurrent amb Correlation IDs — cada operació es pot traçar
// Demostra com la traçabilitat fa els errors diagnosticables

import java.util.List;
import java.util.UUID;
import java.util.concurrent.CompletableFuture;
import java.util.stream.Collectors;

public class TracedExtractor {
    private static final CorrelatedLogger log = new CorrelatedLogger("TracedExtractor");
    private static final CorrelatedLogger riotLog = new CorrelatedLogger("RiotClient");
    private static final CorrelatedLogger ddLog = new CorrelatedLogger("DataDragonClient");

    record ChampionData(String id, String riotData, String ddData) {}
    record ExtractionResult(List<ChampionData> champions, List<String> errors) {}

    static String fetchRiotData(String championId) {
        long start = System.nanoTime();
        riotLog.info("Calling Riot API (champion=" + championId + ")");
        try {
            Thread.sleep(300);
            // 15% de probabilitat de fallar
            if (Math.random() < 0.15) {
                throw new RuntimeException("Connection timeout");
            }
            long elapsed = (System.nanoTime() - start) / 1_000_000;
            riotLog.info("Riot API OK (champion=" + championId + ", elapsed=" + elapsed + "ms)");
            return "RiotData{" + championId + "}";
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }

    static String fetchDataDragon(String championId) {
        long start = System.nanoTime();
        ddLog.info("Calling Data Dragon (champion=" + championId + ")");
        try {
            Thread.sleep(100);
            long elapsed = (System.nanoTime() - start) / 1_000_000;
            ddLog.info("Data Dragon OK (champion=" + championId + ", elapsed=" + elapsed + "ms)");
            return "DDData{" + championId + "}";
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }

    static CompletableFuture<ChampionData> extractOne(String championId) {
        // IMPORTANT: capturem el Correlation ID ABANS de llançar l'async
        // Perquè supplyAsync s'executa en un thread diferent (del ForkJoinPool)
        // i el ThreadLocal del thread original no es propaga automàticament
        String corrId = RequestContext.getCorrelationId();

        CompletableFuture<String> riot = CompletableFuture.supplyAsync(() -> {
            // Propaguem el Correlation ID al thread del ForkJoinPool
            RequestContext.setCorrelationId(corrId);
            return fetchRiotData(championId);
        }).exceptionally(error -> {
            RequestContext.setCorrelationId(corrId);
            riotLog.error("FAILED (champion=" + championId + ", cause=" + error.getMessage() + ")");
            return null;  // Null indica error parcial
        });

        CompletableFuture<String> dd = CompletableFuture.supplyAsync(() -> {
            RequestContext.setCorrelationId(corrId);
            return fetchDataDragon(championId);
        }).exceptionally(error -> {
            RequestContext.setCorrelationId(corrId);
            ddLog.error("FAILED (champion=" + championId + ", cause=" + error.getMessage() + ")");
            return null;
        });

        return riot.thenCombine(dd, (r, d) -> new ChampionData(championId, r, d));
    }

    public static void main(String[] args) {
        // Simulem 3 requests concurrents, cadascun amb el seu Correlation ID
        List<String> requests = List.of("Request-A", "Request-B", "Request-C");

        List<Thread> requestThreads = requests.stream().map(reqName -> {
            Thread t = new Thread(() -> {
                // Cada "request" rep el seu propi Correlation ID
                String corrId = RequestContext.initCorrelationId();
                log.info("=== START " + reqName + " (corrId=" + corrId + ") ===");

                List<String> champions = List.of("Ahri", "Zed", "Lux", "Yasuo", "Jinx");

                List<CompletableFuture<ChampionData>> futures = champions.stream()
                    .map(TracedExtractor::extractOne)
                    .collect(Collectors.toList());

                List<ChampionData> results = futures.stream()
                    .map(CompletableFuture::join)
                    .collect(Collectors.toList());

                long ok = results.stream().filter(c -> c.riotData() != null).count();
                long failed = results.size() - ok;
                log.info("=== END " + reqName + " (ok=" + ok + ", errors=" + failed + ") ===");

                RequestContext.clear();  // Neteja el ThreadLocal
            });
            t.start();
            return t;
        }).collect(Collectors.toList());

        // Espera que tots els "requests" acabin
        requestThreads.forEach(t -> {
            try { t.join(); } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
    }
}
```

Per fer funcionar l'exemple, afegeix `setCorrelationId` al `RequestContext`:

```java
// Afegeix al RequestContext.java
public static void setCorrelationId(String id) {
    correlationId.set(id);
}
```

### 3. Logger amb Correlation ID en Python (15 min)

```python
# traced_extractor.py
# Versió Python de l'extractor amb Correlation IDs
# Usa contextvars, que funciona correctament amb asyncio

import asyncio
import time
import random
import uuid
import contextvars

# ContextVar: cada tasca async té la seva pròpia còpia
correlation_id: contextvars.ContextVar[str] = contextvars.ContextVar(
    "correlation_id", default="no-corr-id"
)

def log(level: str, component: str, message: str) -> None:
    """Log amb Correlation ID, timestamp i component."""
    corr = correlation_id.get()
    ts = time.strftime("%H:%M:%S")
    print(f"{level:5s} {ts} [{corr}] [{component}] {message}")

async def fetch_riot_data(champion_id: str) -> str:
    """Simula crida a Riot API amb traçabilitat."""
    start = time.perf_counter()
    log("INFO", "RiotClient", f"Calling Riot API (champion={champion_id})")
    await asyncio.sleep(0.3)

    if random.random() < 0.15:
        elapsed = (time.perf_counter() - start) * 1000
        log("ERROR", "RiotClient", f"Timeout (champion={champion_id}, elapsed={elapsed:.0f}ms)")
        raise ConnectionError(f"Riot API timeout for {champion_id}")

    elapsed = (time.perf_counter() - start) * 1000
    log("INFO", "RiotClient", f"OK (champion={champion_id}, elapsed={elapsed:.0f}ms)")
    return f"RiotData({champion_id})"

async def fetch_data_dragon(champion_id: str) -> str:
    """Simula crida a Data Dragon amb traçabilitat."""
    start = time.perf_counter()
    log("INFO", "DDClient", f"Calling Data Dragon (champion={champion_id})")
    await asyncio.sleep(0.1)
    elapsed = (time.perf_counter() - start) * 1000
    log("INFO", "DDClient", f"OK (champion={champion_id}, elapsed={elapsed:.0f}ms)")
    return f"DDData({champion_id})"

async def extract_champions(request_name: str, champion_ids: list[str]) -> None:
    """Extreu campions amb el seu propi Correlation ID."""
    # Assigna un Correlation ID únic per aquest "request"
    corr = f"req-{uuid.uuid4().hex[:8]}"
    correlation_id.set(corr)

    log("INFO", "Extractor", f"=== START {request_name} ({len(champion_ids)} champions) ===")

    tasks = []
    for cid in champion_ids:
        tasks.append(fetch_and_handle(cid))

    results = await asyncio.gather(*tasks)
    ok = sum(1 for r in results if r is not None)
    errors = len(results) - ok

    log("INFO", "Extractor", f"=== END {request_name} (ok={ok}, errors={errors}) ===")

async def fetch_and_handle(champion_id: str):
    """Extreu un campió amb gestió d'errors."""
    try:
        riot, dd = await asyncio.gather(
            fetch_riot_data(champion_id),
            fetch_data_dragon(champion_id)
        )
        return {"champion": champion_id, "riot": riot, "dd": dd}
    except Exception as e:
        log("WARN", "Extractor", f"Partial error (champion={champion_id}, cause={e})")
        return None

async def main():
    # Simula 3 requests concurrents, cadascun amb el seu Correlation ID
    await asyncio.gather(
        extract_champions("Request-A", ["Ahri", "Zed", "Lux"]),
        extract_champions("Request-B", ["Yasuo", "Jinx", "Thresh"]),
        extract_champions("Request-C", ["Katarina", "Ezreal", "Vayne"]),
    )

asyncio.run(main())
```

### 4. Analisi de Logs (10 min)

Executa el `TracedExtractor` (Java o Python) i redirigeix la sortida a un fitxer:

```bash
# Executa i guarda els logs
python3 traced_extractor.py > extraction_logs.txt 2>&1

# Filtra per un Correlation ID concret — veus tot el recorregut d'un request
grep "req-a1b2c3" extraction_logs.txt

# Busca tots els errors
grep "ERROR" extraction_logs.txt

# Quants requests han tingut errors?
grep "errors=" extraction_logs.txt

# Quin component falla més?
grep "ERROR" extraction_logs.txt | grep -oP '\[\K[^\]]+' | sort | uniq -c | sort -rn
```

Respon:
- Pots identificar quin request ha fallat nomes mirant els logs?
- Pots determinar quin campio ha causat cada error?
- Sense Correlation IDs, podries fer el mateix?

> **Lectura recomanada (opcional, no bloquejant):**
> - Martin Fowler: [Correlation ID](https://www.enterpriseintegrationpatterns.com/patterns/messaging/CorrelationIdentifier.html)
> - Baeldung: [MDC with SLF4J and Logback](https://www.baeldung.com/mdc-in-log4j-2-logback) (la versio "de veritat" per a projectes Spring)

---

## Checklist de Lliurament

- [ ] Has implementat `RequestContext` amb `ThreadLocal` (Java) o `contextvars` (Python)
- [ ] Has implementat un logger que inclou automaticament el Correlation ID
- [ ] Has executat l'extractor amb 3 requests concurrents i cada request te el seu propi Correlation ID als logs
- [ ] Pots filtrar els logs per Correlation ID i veure tot el recorregut d'un request
- [ ] Pots identificar errors parcials i el seu campio/request nomes mirant els logs
- [ ] Entens per que la tracabilitat es essencial en sistemes concurrents i en produccio
