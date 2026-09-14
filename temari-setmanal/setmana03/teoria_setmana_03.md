# Setmana 3 - Teoria: Concurrència Pràctica per a Developers Web

## 1. Introducció: Per Què la Concurrència és el Teu Problema?

Imagina que GamePulse té 500 usuaris simultanis. Cada un fa una petició:

```
Usuari A → GET /games/APP-123
Usuari B → POST /games (crear joc)
Usuari C → PUT /games/APP-123 (canviar preu)
Usuari D → GET /games/APP-123
...500 peticions al mateix temps
```

**Pregunta:** Com gestiona Spring Boot 500 peticions alhora si el teu codi és un sol fitxer `GameController.java`?

**Resposta:** Amb **threads**. Cada petició s'executa en un thread diferent, en paral·lel. I aquí és on comencen els problemes.

---

## 2. Què és un Thread?

### L'Analogia de la Cuina

Imagina un restaurant. El **procés** és la cuina sencera: forn, nevera, estris, ingredients. Un **thread** és un cuiner dins d'aquesta cuina.

- **1 cuiner (1 thread):** Pot preparar un sol plat alhora. Mentre espera que l'aigua bulli, no fa res.
- **4 cuiners (4 threads):** Poden preparar 4 plats alhora. Comparteixen la mateixa cuina (memòria), els mateixos estris (recursos). Si dos cuiners intenten usar la mateixa paella al mateix temps... conflicte.

```
Procés (la cuina):
┌──────────────────────────────────────────────────┐
│  Memòria compartida (ingredients, receptes)       │
│                                                  │
│  Thread 1 (cuiner A): Prepara amanida            │
│  Thread 2 (cuiner B): Prepara sopa               │
│  Thread 3 (cuiner C): Espera que el forn s'escalfi│
│  Thread 4 (cuiner D): Talla verdures             │
│                                                  │
│  Tots comparteixen el mateix espai                │
└──────────────────────────────────────────────────┘
```

### Definició Tècnica

Un **thread** (fil d'execució) és la unitat mínima d'execució dins d'un procés. Cada thread:

- Té el seu propi **comptador de programa** (on està executant)
- Té la seva pròpia **pila (stack)** (variables locals, crides a funcions)
- **Comparteix** la memòria del procés amb tots els altres threads (heap, variables globals)

```
Procés Java (JVM):
┌──────────────────────────────────────────────────────┐
│                                                      │
│  HEAP (compartit):                                   │
│  ┌──────────────────────────────────────────────┐    │
│  │ GameManagementService (singleton)             │    │
│  │ ConcurrentHashMap<String, GameRecord>         │    │
│  │ Objectes, instàncies, dades compartides       │    │
│  └──────────────────────────────────────────────┘    │
│                                                      │
│  STACKS (privats):                                   │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │Thread-1 │ │Thread-2 │ │Thread-3 │ │Thread-4 │   │
│  │ locals  │ │ locals  │ │ locals  │ │ locals  │   │
│  │ criades │ │ criades │ │ criades │ │ criades │   │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘   │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**La clau del problema:** Les variables locals (stack) són privades — cap thread veu les d'un altre. Però el heap (objectes, services, repositoris) és **compartit**. Quan dos threads modifiquen el mateix objecte al heap... race condition.

### Threads vs Processos

| | Procés | Thread |
|---|---|---|
| **Memòria** | Pròpia (aïllada) | Compartida amb altres threads |
| **Crear** | Lent (~100ms) | Ràpid (~1ms) |
| **Comunicació** | Costosa (IPC, sockets) | Directa (memòria compartida) |
| **Si falla** | No afecta altres processos | Pot corrompre tot el procés |
| **Exemple** | Cada pestanya de Chrome | Cada petició HTTP a Spring Boot |

**Per què no usar processos en lloc de threads?** Perquè un servidor web necessita compartir dades (cache, connexions a BD, estat de sessió) entre peticions. Amb processos aïllats, cada petició hauria de reconnectar-se a la BD, reconstruir la cache, etc. Massa lent.

### En Python

```python
import threading

def say_hello(name):
    print(f"Hola des del thread {name}!")

# Crear i llançar 3 threads
threads = []
for i in range(3):
    t = threading.Thread(target=say_hello, args=(f"T-{i}",))
    t.start()
    threads.append(t)

# Esperar que tots acabin
for t in threads:
    t.join()
```

El model és idèntic a Java: cada thread comparteix la memòria del procés, amb el mateix risc de race conditions.

---

## 3. El Model Thread-per-Request

### Com Funciona un Servidor Web

```
                    ┌─────────────────────────────────┐
                    │         Tomcat (Spring Boot)     │
                    │                                  │
Client A ────────── │ ──→ Thread-1 → Controller →      │
Client B ────────── │ ──→ Thread-2 → Controller →      │ ──→ Base de Dades
Client C ────────── │ ──→ Thread-3 → Controller →      │
                    │         ...                      │
Client N ────────── │ ──→ Thread-N → Controller →      │
                    │                                  │
                    │  Thread Pool: 200 threads (defecte)│
                    └─────────────────────────────────┘
```

Quan fas `mvn spring-boot:run`, Tomcat crea un **pool de 200 threads**. Cada petició HTTP agafa un thread del pool, executa el teu codi (Controller → Service → Repository → BD), i retorna el thread al pool.

**Implicació:** El teu `GameManagementService` **s'executa simultàniament en múltiples threads**. Si el service modifica alguna variable compartida... tens un problema.

### Thread Pool en Números

| Situació | Threads disponibles | Peticions/segon |
|----------|-------------------|-----------------|
| API ràpida (5ms/request) | 200 | 40.000 req/s |
| API amb BD (50ms/request) | 200 | 4.000 req/s |
| API que crida API externa (500ms/request) | 200 | 400 req/s |
| Pool esgotat | 0 | Peticions en cua → timeout |

**Conclusió:** Si cada thread espera 500ms per una API externa, 200 threads només poden servir 400 req/s. Després, els clients esperen. Això és el que resolen `CompletableFuture` i Virtual Threads.

---

## 4. Race Conditions: El Bug Invisible

### Què és una Race Condition?

Una race condition passa quan **dos threads accedeixen a les mateixes dades al mateix temps** i almenys un les modifica. El resultat depèn de l'ordre d'execució — que és **impredictible**.

### Exemple: El Comptador Trencat

```java
public class UnsafeCounter {
    private int count = 0;
    
    public void increment() {
        count++;  // Sembla atòmic, però NO ho és
    }
    
    public int getCount() {
        return count;
    }
}
```

`count++` sembla una operació, però internament són **tres**:

```
1. LLEGIR: registre = count       (llegeix 42)
2. CALCULAR: registre = registre + 1  (calcula 43)
3. ESCRIURE: count = registre     (escriu 43)
```

Amb dos threads:

```
Thread A                    Thread B
────────                    ────────
LLEGIR count = 42
                            LLEGIR count = 42    ← Llegeix el MATEIX valor!
CALCULAR 42 + 1 = 43
                            CALCULAR 42 + 1 = 43
ESCRIURE count = 43
                            ESCRIURE count = 43  ← Sobreescriu amb 43!

Resultat: count = 43 (hauria de ser 44)
```

Un increment s'ha perdut. Amb milers de threads, el comptador pot perdre el 20-30% dels increments.

### Per Què Afecta GamePulse?

Imagina un camp `viewCount` que compta quantes vegades s'ha vist un joc:

```java
public void viewGame(String appId) {
    GameRecord game = repository.findById(appId);
    game.setViewCount(game.getViewCount() + 1);  // ← Race condition!
    repository.save(game);
}
```

Si 100 usuaris visiten el joc al mateix temps, podríem comptar-ne 70 en lloc de 100.

---

## 5. Solucions a les Race Conditions

### Solució 1: `synchronized` (Exclusió Mútua)

```java
public class SafeCounter {
    private int count = 0;
    
    public synchronized void increment() {
        count++;  // Ara només un thread pot executar això alhora
    }
}
```

`synchronized` posa un **lock**: quan un thread entra al mètode, la resta esperen fora.

```
Thread A                    Thread B
────────                    ────────
AGAFA LOCK
LLEGIR count = 42
CALCULAR 43
ESCRIURE count = 43
ALLIBERA LOCK
                            AGAFA LOCK
                            LLEGIR count = 43    ← Ara llegeix 43!
                            CALCULAR 44
                            ESCRIURE count = 44
                            ALLIBERA LOCK

Resultat: count = 44 ✅
```

**Problema:** Si tots els threads esperen el lock, perdem paral·lelisme. En un servidor web, això pot convertir 200 threads en efectivament 1.

### Solució 2: `AtomicInteger` (Operacions Atòmiques)

```java
public class AtomicCounter {
    private AtomicInteger count = new AtomicInteger(0);
    
    public void increment() {
        count.incrementAndGet();  // Operació atòmica a nivell de CPU
    }
}
```

`AtomicInteger` utilitza instruccions especials del processador (CAS — Compare-And-Swap) que garanteixen atomicitat **sense lock**:

```
Thread A                    Thread B
────────                    ────────
CAS(count, 42, 43) → OK!
                            CAS(count, 42, 43) → FAIL! (count ja és 43)
                            CAS(count, 43, 44) → OK!  (reintenta amb valor actual)

Resultat: count = 44 ✅ (sense bloquejar ningú)
```

**Avantatge:** Molt més ràpid que `synchronized` perquè no bloqueja threads.

### Comparativa de Rendiment

| Solució | 10 threads × 1M increments | Correcte? |
|---------|---------------------------|-----------|
| `count++` (unsafe) | 50ms | ❌ Perd increments |
| `synchronized` | 800ms | ✅ Correcte però lent |
| `AtomicInteger` | 200ms | ✅ Correcte i ràpid |

### Solució 3: Immutabilitat (La Millor Solució)

Si les dades **no es poden modificar**, no hi ha race condition possible:

```java
// GameRecord és un record (immutable) → thread-safe per disseny
public record GameRecord(String appId, String title, BigDecimal price, Long players) {}

// Mai modifiquem, sempre creem nous
GameRecord updated = new GameRecord(game.appId(), game.title(), newPrice, game.players());
```

**Per això S2 va insistir en immutabilitat.** No és un capritx acadèmic — és una estratègia de concurrència.

---

## 6. Race Conditions a la Base de Dades: Lost Update

### El Problema

El `synchronized` de Java protegeix dins d'una JVM. Però a producció, pots tenir **múltiples instàncies** de l'aplicació:

```
Instància 1 (JVM)        Instància 2 (JVM)         Base de Dades
──────────────────        ──────────────────         ─────────────
Thread-A llegeix          Thread-B llegeix
  game.price = 29.99       game.price = 29.99       price = 29.99

Thread-A: price = 19.99
Thread-A guarda
                                                     price = 19.99

                          Thread-B: price = 39.99
                          Thread-B guarda
                                                     price = 39.99

Resultat: El descompte de Thread-A s'ha perdut!
```

`synchronized` no serveix aquí perquè els dos threads estan en JVMs diferents.

### Solució: @Transactional

```java
@Service
public class GameManagementService {
    
    @Transactional
    public void updatePrice(String appId, BigDecimal newPrice) {
        GameRecord game = repository.findById(appId)
            .orElseThrow(() -> new EntityNotFoundException("Game not found"));
        game.setPrice(newPrice);
        repository.save(game);
    }
}
```

`@Transactional` fa que Spring:
1. Obre una **transacció SQL** (BEGIN)
2. Executa el mètode
3. Si tot va bé: **COMMIT**
4. Si llança excepció: **ROLLBACK** (desfà tot)

La BD garanteix **ACID** — les transaccions no es trepitgen.

### Solució Avançada: Optimistic Locking

```java
@Entity
public class GameRecord {
    @Id
    private String appId;
    private String title;
    private BigDecimal price;
    
    @Version
    private Long version;  // ← Això és la clau
}
```

Cada vegada que es guarda un `GameRecord`, JPA incrementa `version`. Si dos threads llegeixen la versió 5 i intenten guardar:

```
Thread A: UPDATE games SET price=19.99, version=6 WHERE appId='APP-1' AND version=5
  → 1 fila afectada ✅

Thread B: UPDATE games SET price=39.99, version=6 WHERE appId='APP-1' AND version=5
  → 0 files afectades → OptimisticLockException!
```

El segon thread rep una excepció i pot reintentar amb les dades actualitzades.

**Quan usar-ho:**
- Formularis d'edició (dos usuaris editant el mateix registre)
- Qualsevol operació read-modify-write

---

## 7. I/O Concurrent: CompletableFuture

### El Problema: APIs Externes Lentes

GamePulse ha de consultar Steam (300ms) i IGDB (400ms) per cada joc:

```
Seqüencial:
Steam ─────────[300ms]───────→
                               IGDB ─────────[400ms]──────────→
                                                                 Total: 700ms

Paral·lel:
Steam ─────────[300ms]───────→
IGDB ─────────────[400ms]────────────→
                                       Total: 400ms (el més lent dels dos)
```

### Solució amb CompletableFuture

```java
public GameRecord enrichGame(String appId) {
    // Llançar les dues crides en paral·lel
    CompletableFuture<SteamData> steamFuture = 
        CompletableFuture.supplyAsync(() -> steamClient.fetch(appId));
    
    CompletableFuture<IgdbData> igdbFuture = 
        CompletableFuture.supplyAsync(() -> igdbClient.fetch(appId));
    
    // Esperar que ambdues acabin i combinar
    return steamFuture.thenCombine(igdbFuture, (steam, igdb) -> 
        new GameRecord(appId, steam.title(), steam.price(), igdb.playerCount())
    ).join();  // Bloqueja fins que els dos acaben
}
```

**Flux:**

```
Thread principal
    │
    ├── supplyAsync → ForkJoinPool thread-1 → steamClient.fetch() ──→ SteamData
    │
    ├── supplyAsync → ForkJoinPool thread-2 → igdbClient.fetch() ──→ IgdbData
    │
    └── thenCombine(steam, igdb) → GameRecord
```

> **Nota:** Virtual Threads (Java 21, Project Loom) es treballen a la **Setmana 7**, quan l'estudiant té endpoints REST que criden APIs externes i el problema d'escalabilitat del thread pool es fa evident de forma natural.

---

## 8. Python: El Mirall Concurrent

### Race Condition en Python

```python
import threading

count = 0

def increment():
    global count
    for _ in range(100_000):
        count += 1  # NO és atòmic tampoc en Python!

threads = [threading.Thread(target=increment) for _ in range(10)]
for t in threads: t.start()
for t in threads: t.join()

print(f"Esperat: 1.000.000, Real: {count}")
# Real: ~850.000 (perd increments!)
```

**Nota sobre el GIL:** Python té el Global Interpreter Lock (GIL), que fa que només un thread executi bytecode Python alhora. Però `count += 1` NO és un sol bytecode — és LOAD, ADD, STORE — i el GIL pot canviar de thread entre ells. El GIL **no protegeix contra race conditions** en operacions compostes.

### asyncio: L'Equivalent a CompletableFuture

```python
import asyncio
import aiohttp

async def fetch_game(session, app_id):
    async with session.get(f"https://api.steam.com/game/{app_id}") as resp:
        return await resp.json()

async def fetch_all(app_ids):
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_game(session, aid) for aid in app_ids]
        return await asyncio.gather(*tasks)  # Equivalent a CompletableFuture.allOf()

results = asyncio.run(fetch_all(["APP-1", "APP-2", "APP-3"]))
```

**Comparativa de patrons:**

| Concepte | Java | Python |
|----------|------|--------|
| Thread bàsic | `new Thread(() -> {...}).start()` | `threading.Thread(target=fn).start()` |
| Lock | `synchronized` | `threading.Lock()` |
| Async I/O | `CompletableFuture.supplyAsync()` | `asyncio.create_task()` |
| Esperar múltiples | `CompletableFuture.allOf()` | `asyncio.gather()` |
| Combinar resultats | `.thenCombine()` | `await asyncio.gather()` retorna llista |

---

## 9. Aplicació a GamePulse

### S3: Extractor Concurrent

```
50 jocs per extreure de Steam API (300ms cada crida)

Seqüencial:          50 × 300ms = 15.000ms (15 segons!)
CompletableFuture:   ~300-600ms (paral·lel, limitat pel ForkJoinPool)
Python asyncio:      ~300ms (equivalent, tot en paral·lel)
```

> **A S7** veurem Virtual Threads, que permeten escalar a milers de crides I/O simultànies amb codi tan simple com la versió seqüencial.

---

## 10. Errors Comuns de Concurrència (Que Veuràs a la Feina)

### Deadlock

```java
// Thread A agafa Lock 1, després vol Lock 2
synchronized(lock1) {
    synchronized(lock2) { /* ... */ }
}

// Thread B agafa Lock 2, després vol Lock 1
synchronized(lock2) {
    synchronized(lock1) { /* ... */ }  // ← DEADLOCK! Ambdós esperen l'altre
}
```

Cada thread té un lock i espera l'altre. Cap pot avançar. L'aplicació es congela.

### N+1 Queries (Concurrent + BD)

```java
// Sembla innocent...
List<GameRecord> games = repository.findAll();  // 1 query
for (GameRecord game : games) {
    List<PatchNote> patches = patchRepository.findByGameId(game.getId());  // N queries!
}
// Si tens 1000 jocs: 1 + 1000 = 1001 queries a la BD
```

Amb 200 threads fent això, la BD rep 200.000 queries. Solució: `JOIN FETCH` o `@EntityGraph`.

### Shared Mutable State en un @Service

```java
@Service
public class GameService {
    private List<String> recentSearches = new ArrayList<>();  // ← COMPARTIT entre requests!
    
    public List<GameRecord> search(String query) {
        recentSearches.add(query);  // ← Race condition! ArrayList no és thread-safe
        return repository.findByTitle(query);
    }
}
```

Els `@Service` de Spring són **singletons** — una sola instància compartida per tots els threads. Si tens estat mutable en un service, tens un bug.

**Solució:** No tenir estat mutable en services. Si necessites cache, usa `ConcurrentHashMap` o un servei de cache extern (Redis).

---

## Resum

| Concepte | Key Takeaway |
|----------|--------------|
| **Thread-per-request** | Spring Boot assigna un thread per petició; el teu codi s'executa en paral·lel |
| **Race condition** | Dos threads accedint a les mateixes dades → resultats impredictibles |
| **synchronized** | Exclusió mútua; segur però lent (serialitza threads) |
| **AtomicInteger** | Operacions atòmiques sense lock; ràpid i segur per comptadors |
| **Immutabilitat** | La millor defensa: si no es modifica, no hi ha race condition |
| **@Transactional** | Protecció a nivell de BD; ACID garantit |
| **Optimistic Locking** | `@Version` per detectar conflictes sense bloquejar |
| **CompletableFuture** | Crides I/O en paral·lel; combinar resultats |
| **asyncio (Python)** | Equivalent a CompletableFuture per I/O concurrent |

**Objectiu setmana:** Entendre per què la concurrència és un problema real en aplicacions web, saber diagnosticar-lo, i conèixer les solucions bàsiques (locks, atomics, `CompletableFuture`, `asyncio`) que faràs servir cada dia a la feina. Virtual Threads (Java 21) es treballen a S7, quan el context d'APIs REST fa el problema d'escalabilitat més evident.
