# Setmana 06 — Dimarts: CompletableFuture i I/O Paral-lel

## Objectiu del Dia

Aprendre a fer crides a APIs externes en paral-lel amb `CompletableFuture`. Ahir vas veure que la concurrencia pot causar problemes (race conditions). Avui veurem el costat positiu: la concurrencia permet que el teu backend sigui **molt mes rapid** quan ha de consultar multiples serveis externs. Al final del dia sabras transformar crides sequencials en paral-leles, combinar resultats de multiples fonts, i entendre per que aixo es el que fas cada dia a una empresa amb microserveis.

---

## Teoria

### El Problema: APIs Externes Lentes

El teu backend no viu aillat. Necessita dades d'altres serveis: Riot API per a estadistiques de campions, Data Dragon per a imatges, potser un servei intern de recomanacions. Cada crida triga temps perque viatge per la xarxa.

```
Seqüencial (el que faries sense pensar-hi):
─────────────────────────────────────────
Riot API     ──────[300ms]───────→
                                  Data Dragon ──[100ms]──→
                                                          Total: 400ms

Paral·lel (el que has de fer):
─────────────────────────────
Riot API     ──────[300ms]───────→
Data Dragon  ──[100ms]──→
                          Total: 300ms (el màxim de les dues)
```

**100ms de diferencia?** Sembla poc, pero:
- Amb 10 crides sequencials: 3 segons vs 300ms
- Amb 50 crides: 15 segons vs ~600ms
- L'usuari nota qualsevol resposta >200ms

### El Thread Pool i el Seu Limit

Recorda d'ahir: Tomcat te 200 threads. Si cada thread espera 500ms per una API externa, el maxim es 400 peticions per segon. Despres, els nous clients esperen en cua fins que algun thread s'allibera.

```
| Temps per request | Threads | Peticions/segon |
|-------------------|---------|-----------------|
| 5ms (cache local) | 200     | 40.000 req/s    |
| 50ms (BD)         | 200     | 4.000 req/s     |
| 500ms (API ext.)  | 200     | 400 req/s       |
| Pool esgotat      | 0       | Timeout!        |
```

**Conclusio:** Reduir el temps d'espera per I/O es critic. Si pots fer dues crides de 300ms en paral-lel en lloc de sequencial, has guanyat 300ms per request — i el thread torna al pool mes aviat.

### CompletableFuture: Futures en Java

Un `CompletableFuture<T>` representa un valor que **encara no existeix** pero existira en el futur. Es com un tiquet de recollida: el dones al cuiner i continues fent altres coses. Quan el plat esta llest, el reculls.

```java
// supplyAsync llança una tasca en un thread del ForkJoinPool
// Retorna immediatament un CompletableFuture — la promesa d'un resultat futur
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    // Això s'executa en un thread separat
    return crida_lenta_a_api();
});

// El thread principal pot fer altres coses mentre espera...

// .join() bloqueja fins que el resultat està disponible
String resultat = future.join();
```

### Operacions Clau de CompletableFuture

```java
// supplyAsync: llança una tasca que retorna un valor
// El ForkJoinPool.commonPool() assigna un thread automàticament
CompletableFuture<RiotData> riotFuture =
    CompletableFuture.supplyAsync(() -> riotClient.fetch(championId));

// thenApply: transforma el resultat quan arribi (com .map() en streams)
// No bloqueja — encadena la transformació per quan el resultat estigui llest
CompletableFuture<String> nameFuture =
    riotFuture.thenApply(data -> data.name().toUpperCase());

// thenCombine: combina els resultats de dos futures independents
// Espera que AMBDÓS acabin i aplica la funció de combinació
CompletableFuture<ChampionView> combined =
    riotFuture.thenCombine(ddFuture, (riot, dd) ->
        new ChampionView(riot.name(), riot.winRate(), dd.imageUrl()));

// allOf: espera que TOTS els futures d'un array acabin
// Útil quan tens N crides i vols esperar-les totes
CompletableFuture.allOf(future1, future2, future3).join();

// exceptionally: gestiona errors sense que tot peti
// Si la crida falla, retorna un valor per defecte en lloc de propagar l'excepció
CompletableFuture<RiotData> safeFuture =
    riotFuture.exceptionally(error -> {
        System.err.println("Error cridant Riot API: " + error.getMessage());
        return RiotData.empty();  // Valor per defecte
    });
```

### Flux d'Execucio Visual

```
Thread principal (Tomcat)
    │
    ├── supplyAsync() → ForkJoinPool thread-1 → riotClient.fetch()
    │                                              ↓ (300ms)
    │                                           RiotData
    │
    ├── supplyAsync() → ForkJoinPool thread-2 → dataDragonClient.fetch()
    │                                              ↓ (100ms)
    │                                           DataDragonData
    │
    └── thenCombine(riot, dd) → ChampionView
         ↓
       Retorna al client (temps total: ~300ms, no 400ms)
```

El thread principal NO espera. Llanca les dues tasques i continua. Nomes quan necessita el resultat final (al `join()` o al retornar la resposta HTTP), espera que tot estigui llest.

---

## Activitat

### 1. Versio Sequencial — El Punt de Partida (15 min)

Primer, implementa la versio lenta per veure el problema:

```java
// SequentialFetcher.java
// Versió seqüencial: crida una API darrere l'altra
// Serveix com a baseline per comparar amb la versió paral·lela

public class SequentialFetcher {

    // Simula una crida a Riot API (300ms de latència de xarxa)
    static String fetchRiotData(String championId) {
        try {
            Thread.sleep(300);  // Simula latència de xarxa
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        return "RiotData{champion=" + championId + ", winRate=52.3}";
    }

    // Simula una crida a Data Dragon (100ms de latència)
    static String fetchDataDragon(String championId) {
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        return "DDData{champion=" + championId + ", imageUrl=https://dd.cdn/ahri.png}";
    }

    public static void main(String[] args) {
        String championId = "Ahri";

        long start = System.nanoTime();

        // Crida seqüencial: primer Riot, després Data Dragon
        // El thread espera 300ms, després espera 100ms més
        String riotData = fetchRiotData(championId);
        String ddData = fetchDataDragon(championId);

        long elapsed = (System.nanoTime() - start) / 1_000_000;

        System.out.println("Riot:   " + riotData);
        System.out.println("DD:     " + ddData);
        System.out.println("Temps:  " + elapsed + "ms (esperat: ~400ms)");
    }
}
```

### 2. Versio Paral-lela amb CompletableFuture (25 min)

Ara transforma-ho en paral-lel:

```java
// ParallelFetcher.java
// Versió paral·lela: llança les dues crides alhora amb CompletableFuture
// El temps total és el màxim de les dues, no la suma

import java.util.concurrent.CompletableFuture;

public class ParallelFetcher {

    // Record immutable per al resultat combinat
    // Usar records garanteix thread-safety (S2 + S6 dilluns)
    record ChampionView(String riotData, String ddData, long fetchTimeMs) {}

    static String fetchRiotData(String championId) {
        try { Thread.sleep(300); } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        return "RiotData{champion=" + championId + ", winRate=52.3}";
    }

    static String fetchDataDragon(String championId) {
        try { Thread.sleep(100); } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        return "DDData{champion=" + championId + ", imageUrl=https://dd.cdn/ahri.png}";
    }

    public static void main(String[] args) {
        String championId = "Ahri";

        long start = System.nanoTime();

        // supplyAsync llança cada crida en un thread separat del ForkJoinPool
        // Retorna immediatament — no bloqueja el thread principal
        CompletableFuture<String> riotFuture =
            CompletableFuture.supplyAsync(() -> fetchRiotData(championId));

        CompletableFuture<String> ddFuture =
            CompletableFuture.supplyAsync(() -> fetchDataDragon(championId));

        // thenCombine espera que AMBDÓS futures acabin i combina els resultats
        // La funció lambda rep els dos valors i crea l'objecte final
        ChampionView view = riotFuture.thenCombine(ddFuture, (riot, dd) -> {
            long elapsed = (System.nanoTime() - start) / 1_000_000;
            return new ChampionView(riot, dd, elapsed);
        }).join();  // join() bloqueja fins que tot està llest

        System.out.println("Riot:   " + view.riotData());
        System.out.println("DD:     " + view.ddData());
        System.out.println("Temps:  " + view.fetchTimeMs() + "ms (esperat: ~300ms)");
    }
}
```

Compila i compara els temps:

```bash
javac SequentialFetcher.java && java SequentialFetcher
# Temps: ~400ms

javac ParallelFetcher.java && java ParallelFetcher
# Temps: ~300ms — 25% més ràpid amb només 2 crides
```

### 3. Extractor de Multiples Campions (25 min)

Ara escala: en lloc de 1 campio, extreu dades de 20 campions en paral-lel:

```java
// BatchExtractor.java
// Extreu dades de múltiples campions en paral·lel
// Demostra l'escalabilitat de CompletableFuture: 20 crides en ~300ms vs ~6s seqüencial

import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.stream.Collectors;

public class BatchExtractor {

    record ChampionData(String id, String riotData, String ddData) {}

    static String fetchRiotData(String championId) {
        try { Thread.sleep(300); } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        return "RiotData{" + championId + ", wr=52.3}";
    }

    static String fetchDataDragon(String championId) {
        try { Thread.sleep(100); } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        return "DDData{" + championId + ", img=ok}";
    }

    // Extreu dades d'UN campió en paral·lel (Riot + DD simultàniament)
    static CompletableFuture<ChampionData> extractOne(String championId) {
        CompletableFuture<String> riot =
            CompletableFuture.supplyAsync(() -> fetchRiotData(championId));
        CompletableFuture<String> dd =
            CompletableFuture.supplyAsync(() -> fetchDataDragon(championId));

        // thenCombine combina els dos resultats quan ambdós estan llestos
        return riot.thenCombine(dd, (r, d) -> new ChampionData(championId, r, d));
    }

    public static void main(String[] args) {
        List<String> champions = List.of(
            "Ahri", "Zed", "Lux", "Yasuo", "Jinx",
            "Thresh", "Lee Sin", "Katarina", "Ezreal", "Vayne",
            "Darius", "Morgana", "Garen", "Ashe", "Blitzcrank",
            "Teemo", "Miss Fortune", "Jhin", "Kai'Sa", "Viego"
        );

        // --- Versió seqüencial ---
        long startSeq = System.nanoTime();
        for (String champ : champions) {
            fetchRiotData(champ);
            fetchDataDragon(champ);
        }
        long seqMs = (System.nanoTime() - startSeq) / 1_000_000;

        // --- Versió paral·lela ---
        long startPar = System.nanoTime();

        // Llança TOTES les extraccions en paral·lel
        // Cada extractOne ja fa Riot+DD en paral·lel internament
        List<CompletableFuture<ChampionData>> futures = champions.stream()
            .map(BatchExtractor::extractOne)
            .collect(Collectors.toList());

        // Espera que totes acabin i recull els resultats
        List<ChampionData> results = futures.stream()
            .map(CompletableFuture::join)  // join() espera cada futur individualment
            .collect(Collectors.toList());

        long parMs = (System.nanoTime() - startPar) / 1_000_000;

        System.out.println("=== Resultats ===");
        System.out.println("Campions extrets: " + results.size());
        System.out.println("Seqüencial:       " + seqMs + "ms");
        System.out.println("Paral·lel:        " + parMs + "ms");
        System.out.printf("Speedup:          %.1fx més ràpid%n", (double) seqMs / parMs);

        // Mostra els primers 3 resultats com a verificació
        results.stream().limit(3).forEach(c ->
            System.out.println("  " + c.id() + " → " + c.riotData()));
    }
}
```

### 4. Gestio d'Errors Parcials (15 min)

A la vida real, algunes crides fallen. L'extractor no ha de petar sencer:

```java
// Afegeix al BatchExtractor o crea un fitxer nou

// Versió robusta que gestiona errors parcials
// Si una crida falla, la resta continuen — retorna resultats vàlids + llista d'errors
static CompletableFuture<ChampionData> extractOneSafe(String championId) {
    CompletableFuture<String> riot =
        CompletableFuture.supplyAsync(() -> fetchRiotData(championId))
            .exceptionally(error -> {
                // Si Riot API falla, retornem un valor per defecte
                // L'error queda loggejat però no trenca l'extracció
                System.err.println("WARN: Riot API error per " + championId + ": " + error.getMessage());
                return "RiotData{UNAVAILABLE}";
            });

    CompletableFuture<String> dd =
        CompletableFuture.supplyAsync(() -> fetchDataDragon(championId))
            .exceptionally(error -> {
                System.err.println("WARN: DataDragon error per " + championId + ": " + error.getMessage());
                return "DDData{UNAVAILABLE}";
            });

    return riot.thenCombine(dd, (r, d) -> new ChampionData(championId, r, d));
}
```

Prova afegint un error aleatori al `fetchRiotData`:

```java
static String fetchRiotData(String championId) {
    try { Thread.sleep(300); } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
    // 10% de probabilitat de fallar — simula errors de xarxa reals
    if (Math.random() < 0.1) {
        throw new RuntimeException("Connection timeout to Riot API");
    }
    return "RiotData{" + championId + ", wr=52.3}";
}
```

> **Lectura recomanada (opcional, no bloquejant):**
> - Baeldung: [Guide to CompletableFuture](https://www.baeldung.com/java-completablefuture)
> - Oracle: [CompletableFuture API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/CompletableFuture.html)

---

## Checklist de Lliurament

- [ ] Has implementat la versio sequencial i veus que triga ~400ms per 1 campio
- [ ] Has implementat la versio paral-lela amb CompletableFuture i triga ~300ms (25% menys)
- [ ] Has escalat a 20 campions i la versio paral-lela es almenys 5x mes rapida
- [ ] Has afegit gestio d'errors parcials amb `exceptionally()` — si 2 de 20 fallen, tens 18 resultats
- [ ] Pots explicar: que fa `supplyAsync`, que fa `thenCombine`, que fa `join()`
- [ ] Entens per que paral-lelitzar I/O extern es critic en un servidor web amb thread pool limitat
