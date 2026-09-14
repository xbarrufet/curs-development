# Setmana 4 — Divendres: Exercici Integrador — Parallel Champion Extractor

## Objectiu del Dia

Integrar tot el que has apres aquesta setmana en un projecte complet: un extractor de dades de campions de League of Legends que crida l'API publica de Riot Data Dragon en paral-lel, tant en Java com en Python. El codi inclou Correlation IDs, gestio d'errors parcials, i mesures de rendiment. Al final del dia tindras dues implementacions funcionals, les hauras comparat, i hauras fet commit i Pull Request.

---

## Teoria

### Arquitectura de l'Exercici

Avui construeixes el projecte complet que combina:

1. **Dilluns:** Consciencia de threads i race conditions (no faras `count++` compartit)
2. **Dimarts:** `CompletableFuture` per crides paral-leles (Java)
3. **Dimecres:** `asyncio.gather()` per crides paral-leles (Python)
4. **Dijous:** Correlation IDs i logs estructurats per tracabilitat

```
Arquitectura de l'extractor:
┌────────────────────────────────────────────────────────┐
│                  ChampionExtractor                      │
│                                                        │
│  1. Descarrega la llista de campions (1 crida GET)     │
│  2. Per cada campió, extreu detalls (N crides GET)     │
│  3. Tot en paral·lel amb Correlation IDs               │
│  4. Resultat: dades + errors parcials + mètriques      │
│                                                        │
│  Entrada: URL de Data Dragon                           │
│  Sortida: JSON/CSV amb dades de campions               │
└────────────────────────────────────────────────────────┘
```

### L'API de Riot Data Dragon

Data Dragon es l'API publica de Riot Games amb dades estatiques de League of Legends. No requereix autenticacio ni clau API.

```
URL base: https://ddragon.leagueoflegends.com

Endpoints que usarem:
1. Llista de campions:
   GET /cdn/14.10.1/data/en_US/champion.json
   → Retorna tots els campions amb dades bàsiques

2. Detall d'un campió:
   GET /cdn/14.10.1/data/en_US/champion/{ChampionName}.json
   → Retorna dades detallades (skins, spells, lore)

3. Imatge d'un campió:
   GET /cdn/14.10.1/img/champion/{ChampionName}.png
   → Retorna la imatge splash
```

**Important:** Aquesta API es publica i gratuita, pero te limits de velocitat. No facis centenars de crides per segon. Les nostres 50-100 crides estan dins el limit.

---

## Activitat

### 1. Implementacio Java: `ChampionExtractor.java` (35 min)

Crea el fitxer al teu directori de treball:

```java
// ChampionExtractor.java
// Extractor paral·lel de dades de campions de League of Legends
// Usa CompletableFuture per crides concurrents a Riot Data Dragon API
// Inclou Correlation IDs, gestió d'errors parcials, i mètriques de rendiment

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.*;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.stream.Collectors;

public class ChampionExtractor {

    // === CONFIGURACIÓ ===

    // URL base de l'API de Data Dragon — pública, sense autenticació
    private static final String BASE_URL = "https://ddragon.leagueoflegends.com/cdn/14.10.1/data/en_US";
    // Temps màxim d'espera per crida (evita que el programa es pengi)
    private static final Duration TIMEOUT = Duration.ofSeconds(10);

    // HttpClient de Java 11+ — reutilitzable i thread-safe
    // Crear un sol client per tota l'aplicació és la pràctica recomanada
    private static final HttpClient httpClient = HttpClient.newBuilder()
        .connectTimeout(TIMEOUT)
        .build();

    // Comptadors atòmics per mètriques (thread-safe, vist a dilluns)
    private static final AtomicInteger successCount = new AtomicInteger(0);
    private static final AtomicInteger errorCount = new AtomicInteger(0);

    // === CORRELATION ID (de dijous) ===

    private static final ThreadLocal<String> correlationId = new ThreadLocal<>();
    private static final DateTimeFormatter FMT = DateTimeFormatter.ofPattern("HH:mm:ss.SSS");

    static void log(String level, String component, String message) {
        String ts = LocalDateTime.now().format(FMT);
        String corr = correlationId.get() != null ? correlationId.get() : "no-corr";
        System.out.printf("%-5s %s [%s] [%s] %s%n", level, ts, corr, component, message);
    }

    // === RECORDS IMMUTABLES (de S2 — thread-safe per disseny) ===

    record ChampionSummary(String id, String name, String title, List<String> tags) {}

    record ChampionDetail(
        String id, String name, String title,
        int attack, int defense, int magic, int difficulty,
        List<String> tags, String lore
    ) {}

    record ExtractionResult(
        List<ChampionDetail> champions,
        List<String> errors,
        long elapsedMs,
        int totalAttempted
    ) {}

    // === CRIDES HTTP ===

    // Fa una petició GET i retorna el cos de la resposta com a String
    // Inclou el Correlation ID al header per traçabilitat extrema-a-extrem
    static String httpGet(String url) throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(url))
            .timeout(TIMEOUT)
            // Propagació del Correlation ID a les crides HTTP
            .header("X-Correlation-ID", correlationId.get() != null ? correlationId.get() : "none")
            .GET()
            .build();

        HttpResponse<String> response = httpClient.send(request, HttpResponse.BodyHandlers.ofString());

        if (response.statusCode() != 200) {
            throw new RuntimeException("HTTP " + response.statusCode() + " per " + url);
        }
        return response.body();
    }

    // === PARSEIG JSON SIMPLIFICAT ===
    // Nota: a producció usaríem Jackson o Gson. Aquí parsegem manualment
    // per no afegir dependències externes al programa standalone

    // Extreu la llista de noms de campions del JSON principal
    static List<String> parseChampionNames(String json) {
        List<String> names = new ArrayList<>();
        // El JSON té la forma: "data": { "Aatrox": {...}, "Ahri": {...}, ... }
        // Busquem els noms entre cometes dins de l'objecte "data"
        int dataStart = json.indexOf("\"data\":{");
        if (dataStart == -1) dataStart = json.indexOf("\"data\": {");
        if (dataStart == -1) return names;

        String dataSection = json.substring(dataStart);
        // Busquem claus que tenen un objecte com a valor (els noms dels campions)
        int pos = 0;
        while (true) {
            int keyStart = dataSection.indexOf("\"", pos);
            if (keyStart == -1) break;
            int keyEnd = dataSection.indexOf("\"", keyStart + 1);
            if (keyEnd == -1) break;
            String key = dataSection.substring(keyStart + 1, keyEnd);

            // Saltar les claus internes (id, key, name, etc.)
            int afterKey = dataSection.indexOf(":", keyEnd);
            if (afterKey == -1) break;
            int nextChar = afterKey + 1;
            while (nextChar < dataSection.length() && dataSection.charAt(nextChar) == ' ') nextChar++;
            if (nextChar < dataSection.length() && dataSection.charAt(nextChar) == '{') {
                // És un objecte — probablement un campió
                if (!key.equals("data") && !key.equals("info") && !key.equals("image")
                    && !key.equals("stats") && key.length() > 1 && Character.isUpperCase(key.charAt(0))) {
                    names.add(key);
                }
            }
            pos = keyEnd + 1;
        }
        return names;
    }

    // Extreu un valor string del JSON per clau
    static String extractJsonString(String json, String key) {
        String search = "\"" + key + "\":\"";
        int start = json.indexOf(search);
        if (start == -1) return "unknown";
        start += search.length();
        int end = json.indexOf("\"", start);
        return end == -1 ? "unknown" : json.substring(start, end);
    }

    // Extreu un valor numèric del JSON per clau
    static int extractJsonInt(String json, String key) {
        String search = "\"" + key + "\":";
        int start = json.indexOf(search);
        if (start == -1) return 0;
        start += search.length();
        StringBuilder num = new StringBuilder();
        for (int i = start; i < json.length(); i++) {
            char c = json.charAt(i);
            if (Character.isDigit(c)) num.append(c);
            else if (num.length() > 0) break;
        }
        return num.length() > 0 ? Integer.parseInt(num.toString()) : 0;
    }

    // === LÒGICA D'EXTRACCIÓ ===

    // Descarrega la llista completa de campions
    static List<String> fetchChampionList() throws Exception {
        log("INFO", "Extractor", "Fetching champion list from Data Dragon...");
        long start = System.nanoTime();

        String json = httpGet(BASE_URL + "/champion.json");
        List<String> names = parseChampionNames(json);

        long elapsed = (System.nanoTime() - start) / 1_000_000;
        log("INFO", "Extractor", "Champion list OK (count=" + names.size() + ", elapsed=" + elapsed + "ms)");
        return names;
    }

    // Descarrega les dades detallades d'un campió
    static ChampionDetail fetchChampionDetail(String championName) {
        long start = System.nanoTime();
        log("INFO", "DetailFetch", "Fetching detail (champion=" + championName + ")");

        try {
            String json = httpGet(BASE_URL + "/champion/" + championName + ".json");

            // Parsejar les dades del campió
            String name = extractJsonString(json, "name");
            String title = extractJsonString(json, "title");
            String lore = extractJsonString(json, "lore");
            int attack = extractJsonInt(json, "attack");
            int defense = extractJsonInt(json, "defense");
            int magic = extractJsonInt(json, "magic");
            int difficulty = extractJsonInt(json, "difficulty");

            long elapsed = (System.nanoTime() - start) / 1_000_000;
            log("INFO", "DetailFetch", "OK (champion=" + championName + ", elapsed=" + elapsed + "ms)");
            successCount.incrementAndGet();

            return new ChampionDetail(championName, name, title, attack, defense, magic, difficulty, List.of(), lore);

        } catch (Exception e) {
            long elapsed = (System.nanoTime() - start) / 1_000_000;
            log("ERROR", "DetailFetch", "FAILED (champion=" + championName
                + ", cause=" + e.getMessage() + ", elapsed=" + elapsed + "ms)");
            errorCount.incrementAndGet();
            return null;  // null indica error parcial
        }
    }

    // Extracció paral·lela de tots els campions
    static ExtractionResult extractAll(List<String> championNames, int maxParallel) {
        log("INFO", "Extractor", "Starting parallel extraction (total=" + championNames.size()
            + ", maxParallel=" + maxParallel + ")");

        long start = System.nanoTime();
        String corrId = correlationId.get();

        // Llançar totes les extraccions en paral·lel
        // Cada supplyAsync s'executa en un thread del ForkJoinPool
        List<CompletableFuture<ChampionDetail>> futures = championNames.stream()
            .map(name -> CompletableFuture.supplyAsync(() -> {
                // Propagar el Correlation ID al thread del pool
                correlationId.set(corrId);
                return fetchChampionDetail(name);
            }))
            .collect(Collectors.toList());

        // Esperar que tots acabin i recollir resultats
        List<ChampionDetail> allResults = futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());

        // Separar resultats vàlids i errors
        List<ChampionDetail> champions = allResults.stream()
            .filter(Objects::nonNull)
            .collect(Collectors.toList());

        List<String> errors = new ArrayList<>();
        for (int i = 0; i < allResults.size(); i++) {
            if (allResults.get(i) == null) {
                errors.add(championNames.get(i));
            }
        }

        long elapsed = (System.nanoTime() - start) / 1_000_000;
        return new ExtractionResult(champions, errors, elapsed, championNames.size());
    }

    // === INFORME ===

    static void printReport(ExtractionResult result) {
        System.out.println();
        System.out.println("══════════════════════════════════════════════════");
        System.out.println("  CHAMPION EXTRACTION REPORT");
        System.out.println("══════════════════════════════════════════════════");
        System.out.println();
        System.out.printf("  Total attempted:  %d%n", result.totalAttempted());
        System.out.printf("  Successful:       %d%n", result.champions().size());
        System.out.printf("  Failed:           %d%n", result.errors().size());
        System.out.printf("  Elapsed time:     %dms%n", result.elapsedMs());
        System.out.println();

        if (!result.errors().isEmpty()) {
            System.out.println("  Failed champions:");
            result.errors().forEach(e -> System.out.println("    - " + e));
            System.out.println();
        }

        // Top 5 per atac
        System.out.println("  Top 5 per atac:");
        result.champions().stream()
            .sorted(Comparator.comparingInt(ChampionDetail::attack).reversed())
            .limit(5)
            .forEach(c -> System.out.printf("    %-20s ATK:%d DEF:%d MAG:%d DIF:%d%n",
                c.name(), c.attack(), c.defense(), c.magic(), c.difficulty()));

        System.out.println();
        // Top 5 per dificultat
        System.out.println("  Top 5 per dificultat:");
        result.champions().stream()
            .sorted(Comparator.comparingInt(ChampionDetail::difficulty).reversed())
            .limit(5)
            .forEach(c -> System.out.printf("    %-20s DIF:%d [%s]%n",
                c.name(), c.difficulty(), c.title()));

        System.out.println();
        System.out.println("══════════════════════════════════════════════════");
    }

    // === MAIN ===

    public static void main(String[] args) {
        // Inicialitzar Correlation ID per a tot l'extractor
        String corrId = "extract-" + UUID.randomUUID().toString().substring(0, 8);
        correlationId.set(corrId);

        log("INFO", "Main", "=== Champion Extractor (Java + CompletableFuture) ===");

        try {
            // Pas 1: Obtenir la llista de campions
            List<String> allChampions = fetchChampionList();

            // Limitar a 30 campions per no sobrecarregar l'API
            int limit = Math.min(30, allChampions.size());
            List<String> selected = allChampions.subList(0, limit);
            log("INFO", "Main", "Selected " + limit + " champions for extraction");

            // Pas 2: Extreure detalls en paral·lel
            ExtractionResult result = extractAll(selected, limit);

            // Pas 3: Mostrar informe
            printReport(result);

            log("INFO", "Main", "Extraction complete (total=" + result.elapsedMs() + "ms)");

        } catch (Exception e) {
            log("ERROR", "Main", "Fatal error: " + e.getMessage());
            e.printStackTrace();
        }
    }
}
```

Compila i executa:

```bash
# Compila (Java 11+, no necessita dependències externes)
javac ChampionExtractor.java

# Executa — observa els logs amb Correlation IDs
java ChampionExtractor
```

### 2. Implementacio Python: `champion_extractor.py` (30 min)

```python
# champion_extractor.py
# Extractor paral·lel de dades de campions de League of Legends
# Usa asyncio + aiohttp per crides concurrents a Riot Data Dragon API
# Inclou Correlation IDs, gestió d'errors parcials, i mètriques de rendiment
#
# Requisits: pip install aiohttp

import asyncio
import aiohttp
import time
import uuid
import json
import contextvars
from dataclasses import dataclass

# === CONFIGURACIÓ ===

BASE_URL = "https://ddragon.leagueoflegends.com/cdn/14.10.1/data/en_US"
TIMEOUT = aiohttp.ClientTimeout(total=10)

# === CORRELATION ID (de dijous) ===

correlation_id: contextvars.ContextVar[str] = contextvars.ContextVar(
    "correlation_id", default="no-corr-id"
)

def log(level: str, component: str, message: str) -> None:
    """Log estructurat amb Correlation ID i component."""
    corr = correlation_id.get()
    ts = time.strftime("%H:%M:%S")
    print(f"{level:5s} {ts} [{corr}] [{component}] {message}")

# === DATACLASSES IMMUTABLES (frozen=True → thread-safe, com record a Java) ===

@dataclass(frozen=True)
class ChampionDetail:
    """Dades detallades d'un campió."""
    id: str
    name: str
    title: str
    attack: int
    defense: int
    magic: int
    difficulty: int
    tags: list
    lore: str

@dataclass(frozen=True)
class ExtractionResult:
    """Resultat complet de l'extracció."""
    champions: list          # Llista de ChampionDetail
    errors: list             # Llista de (champion_id, error_message)
    elapsed_ms: float        # Temps total en ms
    total_attempted: int     # Total de campions intentats

# === CRIDES HTTP AMB AIOHTTP ===

async def fetch_champion_list(session: aiohttp.ClientSession) -> list[str]:
    """Descarrega la llista de noms de campions de Data Dragon."""
    log("INFO", "Extractor", "Fetching champion list from Data Dragon...")
    start = time.perf_counter()

    url = f"{BASE_URL}/champion.json"
    # El header X-Correlation-ID permet traçar la petició si l'API el suporta
    headers = {"X-Correlation-ID": correlation_id.get()}

    async with session.get(url, headers=headers) as response:
        if response.status != 200:
            raise RuntimeError(f"HTTP {response.status} fetching champion list")
        data = await response.json()

    # data["data"] conté un diccionari amb els noms dels campions com a claus
    names = list(data["data"].keys())
    elapsed = (time.perf_counter() - start) * 1000
    log("INFO", "Extractor", f"Champion list OK (count={len(names)}, elapsed={elapsed:.0f}ms)")
    return names

async def fetch_champion_detail(session: aiohttp.ClientSession, champion_name: str) -> ChampionDetail:
    """Descarrega les dades detallades d'un campió."""
    start = time.perf_counter()
    log("INFO", "DetailFetch", f"Fetching detail (champion={champion_name})")

    url = f"{BASE_URL}/champion/{champion_name}.json"
    headers = {"X-Correlation-ID": correlation_id.get()}

    async with session.get(url, headers=headers) as response:
        if response.status != 200:
            raise RuntimeError(f"HTTP {response.status} for {champion_name}")
        data = await response.json()

    # Les dades del campió estan dins data["data"][champion_name]
    champ = data["data"][champion_name]
    info = champ.get("info", {})

    detail = ChampionDetail(
        id=champ.get("id", champion_name),
        name=champ.get("name", champion_name),
        title=champ.get("title", ""),
        attack=info.get("attack", 0),
        defense=info.get("defense", 0),
        magic=info.get("magic", 0),
        difficulty=info.get("difficulty", 0),
        tags=champ.get("tags", []),
        lore=champ.get("lore", "")[:100]  # Només els primers 100 chars del lore
    )

    elapsed = (time.perf_counter() - start) * 1000
    log("INFO", "DetailFetch", f"OK (champion={champion_name}, elapsed={elapsed:.0f}ms)")
    return detail

async def extract_one_safe(session: aiohttp.ClientSession, champion_name: str):
    """Extreu un campió amb gestió d'errors — no trenca l'extracció si falla."""
    try:
        return await fetch_champion_detail(session, champion_name)
    except Exception as e:
        log("ERROR", "DetailFetch", f"FAILED (champion={champion_name}, cause={e})")
        return (champion_name, str(e))  # Retorna tupla per indicar error

# === EXTRACCIÓ PARAL·LELA ===

async def extract_all(champion_names: list[str], max_concurrent: int = 30) -> ExtractionResult:
    """
    Extreu dades de tots els campions en paral·lel.
    max_concurrent limita les connexions simultànies per no sobrecarregar l'API.
    """
    log("INFO", "Extractor", f"Starting parallel extraction (total={len(champion_names)})")
    start = time.perf_counter()

    # Semaphore limita el nombre de crides concurrents
    # Sense això, 100+ crides simultànies podrien saturar l'API o el sistema
    semaphore = asyncio.Semaphore(max_concurrent)

    async def limited_extract(session, name):
        async with semaphore:  # Espera si ja hi ha max_concurrent crides actives
            return await extract_one_safe(session, name)

    # aiohttp.ClientSession gestiona el pool de connexions HTTP
    # Crear-la amb "async with" garanteix que es tanca correctament
    async with aiohttp.ClientSession(timeout=TIMEOUT) as session:
        tasks = [limited_extract(session, name) for name in champion_names]
        # gather llança totes les tasques i espera que totes acabin
        results = await asyncio.gather(*tasks)

    # Separar resultats vàlids i errors
    champions = []
    errors = []
    for result in results:
        if isinstance(result, ChampionDetail):
            champions.append(result)
        elif isinstance(result, tuple):
            errors.append(result)

    elapsed = (time.perf_counter() - start) * 1000
    return ExtractionResult(
        champions=champions,
        errors=errors,
        elapsed_ms=elapsed,
        total_attempted=len(champion_names)
    )

# === INFORME ===

def print_report(result: ExtractionResult) -> None:
    """Imprimeix un informe formatat dels resultats."""
    print()
    print("=" * 55)
    print("  CHAMPION EXTRACTION REPORT (Python + asyncio)")
    print("=" * 55)
    print()
    print(f"  Total attempted:  {result.total_attempted}")
    print(f"  Successful:       {len(result.champions)}")
    print(f"  Failed:           {len(result.errors)}")
    print(f"  Elapsed time:     {result.elapsed_ms:.0f}ms")
    print()

    if result.errors:
        print("  Failed champions:")
        for name, msg in result.errors:
            print(f"    - {name}: {msg}")
        print()

    # Top 5 per atac
    sorted_by_attack = sorted(result.champions, key=lambda c: c.attack, reverse=True)
    print("  Top 5 per atac:")
    for c in sorted_by_attack[:5]:
        print(f"    {c.name:20s} ATK:{c.attack} DEF:{c.defense} MAG:{c.magic} DIF:{c.difficulty}")

    print()

    # Top 5 per dificultat
    sorted_by_diff = sorted(result.champions, key=lambda c: c.difficulty, reverse=True)
    print("  Top 5 per dificultat:")
    for c in sorted_by_diff[:5]:
        tags = "/".join(c.tags) if c.tags else "?"
        print(f"    {c.name:20s} DIF:{c.difficulty} [{tags}]")

    print()
    print("=" * 55)

# === MAIN ===

async def main():
    # Inicialitzar Correlation ID
    corr = f"extract-{uuid.uuid4().hex[:8]}"
    correlation_id.set(corr)

    log("INFO", "Main", "=== Champion Extractor (Python + asyncio + aiohttp) ===")

    try:
        async with aiohttp.ClientSession(timeout=TIMEOUT) as session:
            # Pas 1: Obtenir la llista de campions
            all_champions = await fetch_champion_list(session)

        # Limitar a 30 campions
        limit = min(30, len(all_champions))
        selected = all_champions[:limit]
        log("INFO", "Main", f"Selected {limit} champions for extraction")

        # Pas 2: Extreure detalls en paral·lel
        result = await extract_all(selected, max_concurrent=15)

        # Pas 3: Mostrar informe
        print_report(result)

        log("INFO", "Main", f"Extraction complete (total={result.elapsed_ms:.0f}ms)")

    except Exception as e:
        log("ERROR", "Main", f"Fatal error: {e}")
        raise

asyncio.run(main())
```

Instal-la la dependencia i executa:

```bash
# Instal·la aiohttp (si no el tens)
pip install aiohttp

# Executa — observa els logs amb Correlation IDs
python3 champion_extractor.py
```

### 3. Comparativa Java vs Python (10 min)

Despres d'executar els dos extractors, omple aquesta taula:

```
| Mètrica                    | Java (CompletableFuture) | Python (asyncio) |
|----------------------------|--------------------------|------------------|
| Temps total (30 campions)  |              ms          |            ms    |
| Campions extrets           |                          |                  |
| Errors parcials            |                          |                  |
| Línies de codi             |                          |                  |
| Llegibilitat (1-5)         |                          |                  |
| Facilitat de depuració     |                          |                  |
```

Respon:
- Quin codi es mes facil de llegir?
- Quin seria mes facil de depurar si falla en produccio?
- Per a quin cas usaries cadascun?

### 4. Commit i Pull Request (10 min)

Finalitza el cicle Git de la setmana:

```bash
# Assegura't que ets a la branca de la setmana
git checkout -b feature/week4-concurrency

# Afegeix els fitxers de la setmana
git add UnsafeCounter.java CounterBenchmark.java SharedStateDemo.java
git add SequentialFetcher.java ParallelFetcher.java BatchExtractor.java
git add TracedExtractor.java RequestContext.java CorrelatedLogger.java
git add ChampionExtractor.java
git add race_condition.py async_fetcher.py batch_extractor.py
git add traced_extractor.py champion_extractor.py

# Commit amb format Conventional Commits
git commit -m "feat(concurrency): parallel champion extraction with CompletableFuture (Java) and asyncio (Python)

- Thread safety demos: UnsafeCounter, synchronized, AtomicInteger
- CompletableFuture for parallel I/O with error handling
- Python asyncio equivalent with aiohttp
- Correlation IDs for request traceability
- Integration: Riot Data Dragon parallel extractor in both languages"

# Puja la branca
git push -u origin feature/week4-concurrency
```

Crea la Pull Request a GitHub:
- **Titol:** `feat: Week 4 — Concurrent champion extraction (Java + Python)`
- **Descripcio:** Explica que has implementat un extractor paral-lel que crida l'API de Data Dragon, amb gestio d'errors parcials i Correlation IDs. Inclou la taula comparativa Java vs Python.

Fusiona la PR quan estigui llesta.

> **Lectura recomanada (opcional, no bloquejant):**
> - Baeldung: [Testing Concurrent Code](https://www.baeldung.com/java-testing-multithreaded)
> - JUnit 5: [Parallel Test Execution](https://junit.org/junit5/docs/current/user-guide/#writing-tests-parallel-execution)

---

## Checklist de Lliurament

- [ ] `ChampionExtractor.java` s'executa i extreu dades reals de Data Dragon en paral-lel
- [ ] `champion_extractor.py` s'executa i extreu les mateixes dades amb asyncio + aiohttp
- [ ] Ambdos extractors tenen Correlation IDs als logs i gestio d'errors parcials
- [ ] La versio paral-lela es significativament mes rapida que la sequencial (>5x per a 20+ campions)
- [ ] Has omplert la taula comparativa Java vs Python amb les teves metriques reals
- [ ] Commit amb format Conventional Commits i PR creada/fusionada a `main`
- [ ] Branca `feature/week4-concurrency` eliminada despres de fusionar
