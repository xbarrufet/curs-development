# Setmana 09 — Dimecres: Virtual Threads (Java 21)

## Objectiu del Dia

Entendre el problema de l'escalabilitat amb threads de plataforma, com els Virtual Threads de Java 21 el resolen, i activar-los a Spring Boot. Al final del dia tindràs Virtual Threads habilitats al projecte i un benchmark senzill que demostra la diferència.

---

## Teoria

### El Problema: Threads de Plataforma

Quan un servidor web rep una petició, assigna un thread per processar-la. El problema és que els threads de plataforma (els "normals" de Java) són cars:

```
Thread de Plataforma:
├── ~1 MB de memòria d'stack per thread
├── Gestionat pel sistema operatiu (context switch costós)
├── Limitat a ~200-500 threads en un pool típic
└── Si el thread està bloquejat (esperant BD, HTTP), la memòria es malgasta
```

**Exemple pràctic**: Si el teu servidor té 200 threads i cada petició triga 100ms (50ms de BD + 50ms de lògica), pots servir ~2000 peticions/segon. Però si la BD va lenta (500ms), baixes a ~400 peticions/segon perquè els threads estan bloquejats esperant.

### El Diagrama del Problema

```
=== Model Tradicional: Thread Pool Limitat ===

Peticions entrants:          Thread Pool (200 threads):
    [P1] ──────────────→     [T1] █████░░░░░ (50% esperant BD)
    [P2] ──────────────→     [T2] █████░░░░░ (50% esperant BD)
    [P3] ──────────────→     [T3] █████░░░░░ (50% esperant BD)
    ...                       ...
    [P200] ────────────→     [T200] █████░░░░░
    [P201] ─── ESPERA! ──→   ⛔ Pool ple! El client espera...
    [P202] ─── ESPERA! ──→   ⛔ Pool ple!


=== Model Virtual Threads: Sense Límit Pràctic ===

Peticions entrants:          Virtual Threads (milions possibles):
    [P1] ──────────────→     [VT1] █░ (2KB, allibera carrier quan espera BD)
    [P2] ──────────────→     [VT2] █░
    [P3] ──────────────→     [VT3] █░
    ...                       ...
    [P1000] ───────────→     [VT1000] █░
    [P1001] ───────────→     [VT1001] █░  ← Cap problema!
    [P5000] ───────────→     [VT5000] █░  ← Encara bé!
```

### Com Funcionen els Virtual Threads

Els Virtual Threads són threads lleugers gestionats per la JVM (no pel sistema operatiu):

```
Virtual Thread:
├── ~2 KB de memòria (vs ~1 MB dels de plataforma → 500x menys)
├── Gestionat per la JVM, no pel SO
├── Milions possibles en una sola JVM
├── Quan es bloqueja (I/O), la JVM el "desmunta" del carrier thread
└── El carrier thread queda lliure per executar un altre virtual thread
```

**Concepte clau — Carrier Thread**: La JVM manté un petit pool de threads reals (carrier threads). Quan un virtual thread es bloqueja (per exemple, esperant una resposta de la BD), la JVM el desmunta del carrier i hi munta un altre virtual thread. Això maximitza l'ús dels threads reals.

```
Carrier Thread [CT1]:
    temps 0ms:   executa VT1 (processant)
    temps 10ms:  VT1 fa query a BD → JVM desmunta VT1, munta VT2
    temps 15ms:  executa VT2 (processant)
    temps 25ms:  VT2 fa HTTP call → JVM desmunta VT2, munta VT3
    temps 30ms:  resposta BD de VT1 arriba → JVM munta VT1 en CT2
    ...
    // Un sol carrier thread serveix desenes de virtual threads!
```

### Creació de Virtual Threads amb Java 21

```java
// === Exemple bàsic: crear virtual threads manualment ===
public class VirtualThreadDemo {

    public static void main(String[] args) throws Exception {

        // Opció 1: Crear un virtual thread directament
        // Thread.ofVirtual() és la nova API de Java 21
        Thread vt = Thread.ofVirtual()
            .name("el-meu-virtual-thread")  // Nom per depuració
            .start(() -> {
                // Aquest codi s'executa en un virtual thread
                System.out.println("Hola des de: " + Thread.currentThread());
                // El thread és virtual — ocupa ~2KB, no 1MB
            });
        vt.join();  // Esperem que acabi

        // Opció 2: Executor amb virtual threads
        // Crea un thread nou per cada tasca — però són virtuals, així que és barat!
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            // Llancem 10.000 tasques simultànies
            // Amb threads de plataforma, això necessitaria ~10 GB de memòria
            // Amb virtual threads, ~20 MB
            for (int i = 0; i < 10_000; i++) {
                final int taskId = i;
                executor.submit(() -> {
                    // Simulem una operació de I/O (query BD, HTTP call)
                    Thread.sleep(Duration.ofMillis(100));
                    System.out.println("Tasca " + taskId + " completada");
                    return null;
                });
            }
        }
        // L'executor es tanca automàticament (try-with-resources)
        // Totes 10.000 tasques han acabat en ~100ms (no 10000 * 100ms!)
    }
}
```

### Integració amb Spring Boot 3

Activar virtual threads a Spring Boot és extraordinàriament senzill:

```properties
# application.properties
# Aquesta sola línia fa que Spring Boot faci servir virtual threads
# per a TOTES les peticions HTTP del servidor Tomcat
spring.threads.virtual.enabled=true
```

Amb aquesta línia:
- Tomcat crea un virtual thread per cada petició HTTP entrant
- No cal thread pool fix: cada petició té el seu propi virtual thread lleuger
- Les operacions bloquejants (JPA queries, HTTP calls) no malgasten recursos
- El rendiment sota càrrega millorarà significativament

### Quan els Virtual Threads NO Ajuden

```java
// ❌ Operacions intensives de CPU: no milloren amb virtual threads
// Exemple: càlcul matemàtic pur, compressió, encriptació
public double calcularEstadistiques(List<Match> matches) {
    // Això usa la CPU al 100%, no fa I/O
    // Virtual threads no ajuden perquè no hi ha bloqueig
    return matches.stream()
        .mapToDouble(Match::getDuration)
        .average()
        .orElse(0.0);
}

// ✅ Operacions de I/O: milloren molt amb virtual threads
// Exemple: queries BD, crides HTTP, lectura de fitxers
public List<ChampionStats> getStatsFromMultipleSources() {
    // Cada crida bloqueja esperant resposta → virtual threads brillen
    var riotData = riotApiClient.getChampionStats();    // ~200ms esperant
    var localData = championRepository.findAll();        // ~50ms esperant BD
    return mergeStats(riotData, localData);
}
```

### Benchmark: Platform Threads vs Virtual Threads

```java
// === Benchmark per comparar ambdós models ===
// Simula peticions concurrents amb operacions de I/O
public class ThreadBenchmark {

    // Simula una operació que bloqueja el thread (com una query a BD)
    static void simulateIOWork() {
        try {
            Thread.sleep(Duration.ofMillis(100)); // Simula 100ms de I/O
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    public static void main(String[] args) throws Exception {
        int totalTasks = 10_000;  // 10.000 "peticions" simultànies

        // --- Benchmark amb Platform Threads (pool de 200) ---
        System.out.println("=== Platform Threads (pool 200) ===");
        long start = System.currentTimeMillis();

        // Pool fix de 200 threads — el màxim habitual en producció
        try (var executor = Executors.newFixedThreadPool(200)) {
            var futures = new ArrayList<Future<?>>();
            for (int i = 0; i < totalTasks; i++) {
                futures.add(executor.submit(() -> simulateIOWork()));
            }
            // Esperem que acabin totes les tasques
            for (var f : futures) f.get();
        }

        long platformTime = System.currentTimeMillis() - start;
        System.out.println("Temps: " + platformTime + "ms");
        // Resultat esperat: ~5000ms (10000 tasques / 200 threads * 100ms)

        // --- Benchmark amb Virtual Threads ---
        System.out.println("=== Virtual Threads ===");
        start = System.currentTimeMillis();

        // Un virtual thread per tasca — no cal pool fix!
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            var futures = new ArrayList<Future<?>>();
            for (int i = 0; i < totalTasks; i++) {
                futures.add(executor.submit(() -> simulateIOWork()));
            }
            for (var f : futures) f.get();
        }

        long virtualTime = System.currentTimeMillis() - start;
        System.out.println("Temps: " + virtualTime + "ms");
        // Resultat esperat: ~100-200ms (totes 10000 corren quasi en paral·lel!)

        // --- Comparació ---
        System.out.println("\n=== Resultats ===");
        System.out.println("Platform Threads: " + platformTime + "ms");
        System.out.println("Virtual Threads:  " + virtualTime + "ms");
        System.out.println("Speedup: " + (platformTime / virtualTime) + "x");
        // Speedup esperat: ~25-50x per a operacions de I/O
    }
}
```

### Resultat Esperat del Benchmark

```
=== Platform Threads (pool 200) ===
Temps: 5124ms

=== Virtual Threads ===
Temps: 187ms

=== Resultats ===
Platform Threads: 5124ms
Virtual Threads:  187ms
Speedup: 27x
```

La diferència és brutal perquè els virtual threads no malbaraten temps esperant: quan un es bloqueja, un altre ocupa el seu lloc al carrier thread immediatament.

---

## Activitat

### Part 1: Activa Virtual Threads a Spring Boot

1. Afegeix la propietat al teu `application.properties`:
   ```properties
   spring.threads.virtual.enabled=true
   ```
2. Arrenca l'aplicació i verifica que funciona igual que abans

### Part 2: Crea el Benchmark

1. Crea la classe `ThreadBenchmark.java` al paquet `com.esportspulse.engine.benchmark`
2. Executa-la amb `mvn exec:java` o directament des de l'IDE
3. Anota els resultats

### Part 3: Verifica amb un Endpoint de Prova

```java
// Afegeix temporalment aquest endpoint al controller per veure
// que el thread que processa la petició és virtual
@GetMapping("/thread-info")
public ResponseEntity<Map<String, Object>> threadInfo() {
    Thread current = Thread.currentThread();
    return ResponseEntity.ok(Map.of(
        "threadName", current.getName(),
        "isVirtual", current.isVirtual(),  // Ha de ser true!
        "threadClass", current.getClass().getSimpleName()
    ));
}
```

```bash
# Verifica que el thread és virtual
curl http://localhost:8080/api/champions/thread-info
# Resposta esperada: {"threadName":"tomcat-handler-0","isVirtual":true,...}
```

---

## Checklist de Lliurament

- [ ] `spring.threads.virtual.enabled=true` afegit a `application.properties`
- [ ] L'aplicació arrenca correctament amb virtual threads
- [ ] Benchmark `ThreadBenchmark.java` creat i executat
- [ ] Els resultats del benchmark mostren una diferència significativa (>10x)
- [ ] L'endpoint `/thread-info` confirma `"isVirtual": true`
- [ ] Commit: `feat(threads): enable virtual threads and add benchmark`
