# Setmana 4 — Exercicis de Consolidació

Aquests exercicis repassen els conceptes clau de la setmana. No cal lliurar-los — són per verificar que has entès la teoria i la pràctica abans de passar a la setmana 5. Intenta resoldre'ls sense mirar els apunts; si et quedes encallat, revisa el dia corresponent.

---

## Bloc 1: Threads i Race Conditions (Dilluns)

**Exercici 1.1 — Conceptes Bàsics**

Respon sense mirar els apunts:

1. Què és un **thread**? Quina diferència hi ha amb un procés (que vas veure a S3)?
2. Què comparteixen els threads d'un mateix procés? Què és privat de cada thread?
3. Per què `count++` no és atòmic? Quantes operacions realment fa?

**Exercici 1.2 — Detecta la Race Condition**

Analitza aquest codi i explica quin és el problema:

```java
public class PlayerCounter {
    private int onlinePlayers = 0;

    public void playerJoined() {
        onlinePlayers++;
    }

    public void playerLeft() {
        onlinePlayers--;
    }

    public int getOnlinePlayers() {
        return onlinePlayers;
    }
}
```

Si 1.000 jugadors entren i 500 surten simultàniament des de múltiples threads, el resultat serà sempre 500? Per què?

**Exercici 1.3 — Tria la Solució**

Per a cada cas, indica si usaries `synchronized`, `AtomicInteger`, o cap dels dos (i per què):

1. Un comptador de descàrregues que s'incrementa des de múltiples threads.
2. Un objecte `PlayerRecord` (record immutable) que es llegeix des de 100 threads.
3. Un mètode que modifica una llista compartida i envia un email.
4. Un comptador de rendiment que necessita ser el més ràpid possible.

---

## Bloc 2: CompletableFuture i I/O Paral·lel (Dimarts)

**Exercici 2.1 — Seqüencial vs Paral·lel**

Tens 5 crides a una API que cadascuna triga 200ms:

1. Quant trigarà en total si les fas **seqüencialment** (una darrere l'altra)?
2. Quant trigarà si les fas en **paral·lel** amb `CompletableFuture`?
3. Per què la versió paral·lela no triga 0ms? Què limita la velocitat?

**Exercici 2.2 — Llegeix el Codi**

Explica què fa cada línia:

```java
List<CompletableFuture<String>> futures = championIds.stream()
    .map(id -> CompletableFuture.supplyAsync(() -> fetchChampion(id)))
    .toList();

List<String> results = futures.stream()
    .map(CompletableFuture::join)
    .toList();
```

1. Què fa `supplyAsync`?
2. Què fa `join`?
3. En quin moment es llancen les crides? Quan `supplyAsync` es crida o quan `join` es crida?

**Exercici 2.3 — Gestió d'Errors**

Què passa si una de les 5 crides falla amb una excepció? Escriu el codi per gestionar-ho: si una crida falla, retorna un valor per defecte en comptes de fer petar tot el programa.

---

## Bloc 3: Concurrència en Python (Dimecres)

**Exercici 3.1 — GIL**

1. Què és el GIL (Global Interpreter Lock)?
2. Per què el GIL **no** et protegeix de race conditions?
3. En quin cas el GIL sí que limita el rendiment? (CPU-bound o I/O-bound?)

**Exercici 3.2 — Java vs Python**

Completa la taula d'equivalències:

| Concepte | Java | Python |
|----------|------|--------|
| Crear un thread | `new Thread(() -> ...)` | ________ |
| Lock / exclusió mútua | `synchronized` | ________ |
| Futures per I/O paral·lel | `CompletableFuture.supplyAsync()` | ________ |
| Esperar que acabin tots | `.join()` | ________ |
| Operació atòmica en comptador | `AtomicInteger` | ________ |

**Exercici 3.3 — asyncio**

Donat aquest codi, indica si funciona correctament i per què:

```python
import asyncio
import aiohttp

async def fetch_champion(session, champion_id):
    url = f"https://ddragon.leagueoflegends.com/cdn/15.1.1/data/en_US/champion/{champion_id}.json"
    async with session.get(url) as response:
        return await response.json()

async def main():
    async with aiohttp.ClientSession() as session:
        result1 = await fetch_champion(session, "Ahri")
        result2 = await fetch_champion(session, "Yasuo")
        result3 = await fetch_champion(session, "Jinx")

asyncio.run(main())
```

Problema: les crides són paral·leles o seqüencials? Com les faries paral·leles?

---

## Bloc 4: Correlation IDs i Traçabilitat (Dijous)

**Exercici 4.1 — Llegir Logs**

Donats aquests logs, quantes peticions independents hi ha? Agrupa les línies per petició:

```
2024-09-15 10:00:01 [req-abc123] INFO  Rebuda petició: fetch champion Ahri
2024-09-15 10:00:01 [req-def456] INFO  Rebuda petició: fetch champion Yasuo
2024-09-15 10:00:02 [req-abc123] INFO  Crida a Data Dragon: Ahri
2024-09-15 10:00:02 [req-def456] ERROR Data Dragon timeout: Yasuo
2024-09-15 10:00:03 [req-abc123] INFO  Resposta rebuda: Ahri (200 OK)
2024-09-15 10:00:03 [req-def456] INFO  Retry 1/3: Yasuo
```

**Exercici 4.2 — Per Què Importa**

1. Si no tinguéssim Correlation IDs als logs anteriors, com sabríem quina línia d'error pertany a quina petició?
2. En un sistema amb 200 peticions simultànies, per què no n'hi ha prou amb el timestamp per agrupar logs?
3. Qui genera el Correlation ID? Quan es genera?

**Exercici 4.3 — Disseny**

Escriu (en pseudocodi o Java) el patró per afegir un Correlation ID a una funció que fa una crida HTTP. Ha d'aparèixer a tots els logs d'aquella operació.

---

## Bloc 5: Exercici Integrador (Divendres)

**Exercici 5.1 — Arquitectura**

Dibuixa (text ASCII) el flux de dades del Parallel Champion Extractor:

1. D'on vénen les dades?
2. Com es paral·lelitzen les crides?
3. On s'apliquen els Correlation IDs?
4. Què passa si una crida falla?

**Exercici 5.2 — Comparativa**

Sense mirar els apunts, llista 3 diferències pràctiques entre la implementació Java (CompletableFuture) i la Python (asyncio) de l'extractor.

**Exercici 5.3 — Càlcul de Rendiment**

L'API de Data Dragon té 170 campions. Cada crida per obtenir els detalls d'un campió triga ~150ms. Calcula:

1. Temps total si fas les 170 crides **seqüencialment**.
2. Temps total si fas les 170 crides en **paral·lel** amb 10 threads/tasks simultanis (cada thread fa ~17 crides seqüencials).
3. Temps total amb 50 threads/tasks simultanis.

---

## Exercici Final: Integració

Crea un programa `PlayerStatsFetcher` (en Java o Python, tria un) que:

1. Llegeixi una llista de 20 IDs de jugador d'un fitxer `players.txt` (un ID per línia, pots inventar-los).
2. Per cada jugador, simuli una crida a una API que triga 100ms (usa `Thread.sleep(100)` / `asyncio.sleep(0.1)` per simular-ho).
3. Faci totes les crides en **paral·lel** (CompletableFuture o asyncio).
4. Cada crida tingui un **Correlation ID** únic que aparegui als logs.
5. Si una crida falla (simula fallo aleatori al 20% dels casos), logui l'error amb el Correlation ID i continuï amb la resta.
6. Al final, imprimeixi:
   - Quants jugadors processats correctament
   - Quants han fallat
   - Temps total
   - Comparació amb el temps que hauria trigat seqüencialment

> **Pista:** Reutilitza patrons del ChampionExtractor de divendres. La diferència principal és que llegeixes IDs d'un fitxer en comptes de l'API de Riot.

---
---

# Solucions

> **Atenció:** Intenta resoldre els exercicis abans de mirar les solucions.

---

## Bloc 1: Threads i Race Conditions

**Solució 1.1 — Conceptes Bàsics**

1. Un **thread** és un fil d'execució dins d'un procés. Diferència: un procés té la seva pròpia memòria aïllada (S3); els threads dins d'un procés comparteixen la memòria (heap) però cadascun té el seu propi stack.
2. **Compartit:** heap (objectes, instàncies, dades). **Privat:** stack (variables locals, crides de funcions en curs).
3. `count++` internament són 3 operacions: llegir el valor, sumar-hi 1, escriure el resultat. Si un thread és interromput entre qualsevol d'aquests passos, un altre thread pot llegir un valor desactualitzat.

**Solució 1.2 — Detecta la Race Condition**

No, el resultat **no** serà sempre 500. Tant `onlinePlayers++` com `onlinePlayers--` són operacions no atòmiques (read-modify-write). Si dos threads executen `playerJoined()` i `playerLeft()` simultàniament, poden llegir el mateix valor i sobreescriure's mútuament. El resultat serà impredictible.

**Solució 1.3 — Tria la Solució**

1. **`AtomicInteger`** — és un comptador simple que necessita velocitat i correcció. No cal lock.
2. **Cap dels dos** — un record immutable no es pot modificar. Llegir-lo des de 100 threads és segur sense cap protecció. La immutabilitat és la millor estratègia de concurrència.
3. **`synchronized`** — el mètode fa múltiples operacions (modificar llista + enviar email) que han de ser atòmiques com a bloc. `AtomicInteger` no serveix per a operacions compostes.
4. **`AtomicInteger`** — és molt més ràpid que `synchronized` perquè usa instruccions CAS del processador sense bloquejar threads.

---

## Bloc 2: CompletableFuture i I/O Paral·lel

**Solució 2.1 — Seqüencial vs Paral·lel**

1. 5 × 200ms = **1.000ms** (1 segon).
2. ~**200ms** — totes es llancen alhora i triguen el temps de la més lenta.
3. Hi ha overhead de creació de threads, coordinació, i el temps de xarxa no és exactament 200ms. A més, si el thread pool té menys de 5 threads, algunes crides hauran d'esperar.

**Solució 2.2 — Llegeix el Codi**

1. `supplyAsync` llança la funció `fetchChampion(id)` en un thread del pool en segon pla. Retorna un `CompletableFuture` immediatament.
2. `join` bloqueja fins que el Future completa i retorna el resultat.
3. Les crides es llancen quan es crida `supplyAsync` (al primer `stream`). Quan arriba el `join`, moltes ja han acabat o estan en curs.

**Solució 2.3 — Gestió d'Errors**

```java
CompletableFuture.supplyAsync(() -> fetchChampion(id))
    .exceptionally(error -> {
        System.out.println("Error fetching " + id + ": " + error.getMessage());
        return "DEFAULT_VALUE";
    });
```

---

## Bloc 3: Concurrència en Python

**Solució 3.1 — GIL**

1. El GIL és un lock global que només permet a un thread executar bytecode Python alhora.
2. No protegeix de race conditions perquè operacions com `count += 1` encara es poden interrompre entre instruccions. El GIL protegeix l'intèrpret, no les teves dades.
3. **CPU-bound** — si tens càlculs pesants, el GIL impedeix que múltiples threads aprofitin múltiples nuclis. Per I/O-bound (crides HTTP, BD), el GIL no és un problema perquè els threads esperen I/O i l'alliberen.

**Solució 3.2 — Java vs Python**

| Concepte | Java | Python |
|----------|------|--------|
| Crear un thread | `new Thread(() -> ...)` | `threading.Thread(target=...)` |
| Lock / exclusió mútua | `synchronized` | `threading.Lock()` |
| Futures per I/O paral·lel | `CompletableFuture.supplyAsync()` | `asyncio.create_task()` |
| Esperar que acabin tots | `.join()` | `asyncio.gather()` |
| Operació atòmica en comptador | `AtomicInteger` | `threading.Lock()` (Python no té equivalent directe) |

**Solució 3.3 — asyncio**

Les crides són **seqüencials** — cada `await` espera que la crida anterior acabi abans de començar la següent. Per fer-les paral·leles:

```python
async def main():
    async with aiohttp.ClientSession() as session:
        results = await asyncio.gather(
            fetch_champion(session, "Ahri"),
            fetch_champion(session, "Yasuo"),
            fetch_champion(session, "Jinx"),
        )
```

`asyncio.gather()` llança totes les crides alhora i espera que totes completin.

---

## Bloc 4: Correlation IDs i Traçabilitat

**Solució 4.1 — Llegir Logs**

**2 peticions:**

Petició `req-abc123` (Ahri):
```
10:00:01 Rebuda petició: fetch champion Ahri
10:00:02 Crida a Data Dragon: Ahri
10:00:03 Resposta rebuda: Ahri (200 OK)
```

Petició `req-def456` (Yasuo):
```
10:00:01 Rebuda petició: fetch champion Yasuo
10:00:02 Data Dragon timeout: Yasuo
10:00:03 Retry 1/3: Yasuo
```

**Solució 4.2 — Per Què Importa**

1. No podríem — les línies estan intercalades. L'error de timeout podria ser de qualsevol de les dues peticions.
2. Perquè múltiples peticions poden generar logs al mateix segon. El timestamp no és únic per petició.
3. Es genera al principi de cada operació (quan la petició arriba). Típicament és un UUID aleatori. Es propaga a totes les sub-operacions.

**Solució 4.3 — Disseny**

```java
public String fetchChampion(String championId) {
    String correlationId = UUID.randomUUID().toString().substring(0, 8);

    log(correlationId, "Iniciant fetch: " + championId);

    try {
        String result = httpClient.get(url);
        log(correlationId, "Resposta rebuda: " + championId);
        return result;
    } catch (Exception e) {
        log(correlationId, "ERROR: " + championId + " - " + e.getMessage());
        throw e;
    }
}

private void log(String correlationId, String message) {
    System.out.printf("[%s] %s %s%n",
        correlationId, Instant.now(), message);
}
```

---

## Bloc 5: Exercici Integrador

**Solució 5.1 — Arquitectura**

```
players.txt                  API (simulada)
    │                            │
    ▼                            │
┌──────────────┐                 │
│ Llegir IDs   │                 │
│ del fitxer   │                 │
└──────┬───────┘                 │
       │ 20 IDs                  │
       ▼                         │
┌──────────────────────────┐     │
│  Paral·lelitzar          │     │
│  (CompletableFuture /    │     │
│   asyncio.gather)        │     │
│                          │     │
│  ID-1 ──[corr-id-a]──▶ fetch ─┤
│  ID-2 ──[corr-id-b]──▶ fetch ─┤  ← Correlation ID per crida
│  ID-3 ──[corr-id-c]──▶ fetch ─┤
│  ...                     │     │
│                          │     │
│  Si falla → log error    │     │
│  amb corr-id + continua  │     │
└──────────┬───────────────┘     │
           │                     │
           ▼                     │
┌──────────────────────────┐
│ Resum: OK/KO/temps       │
└──────────────────────────┘
```

**Solució 5.2 — Comparativa**

1. **Threading model:** Java usa threads reals del SO (un thread per Future); Python asyncio usa un sol thread amb event loop (cooperative multitasking).
2. **Sintaxi:** Java encadena amb `.thenApply()/.exceptionally()`; Python usa `async/await` que sembla codi seqüencial.
3. **Gestió d'errors:** Java usa `.exceptionally()` als futures; Python usa `try/except` normal dins funcions `async`.

**Solució 5.3 — Càlcul de Rendiment**

1. Seqüencial: 170 × 150ms = **25.500ms** (~25,5 segons)
2. 10 threads: cada thread fa 17 crides seqüencials → 17 × 150ms = **2.550ms** (~2,5 segons)
3. 50 threads: cada thread fa ~3-4 crides → 4 × 150ms = **600ms** (~0,6 segons)
