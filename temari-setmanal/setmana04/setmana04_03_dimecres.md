# Setmana 4 — Dimecres: Concurrencia en Python — threading, GIL i asyncio

## Objectiu del Dia

Traslladar els conceptes de concurrencia de Java a Python: veure que les race conditions existeixen exactament igual, entendre el GIL (Global Interpreter Lock) i per que NO et protegeix, i aprendre `asyncio` com l'equivalent de `CompletableFuture` per a I/O concurrent. Al final del dia tindras el mateix domini de concurrencia en els dos llenguatges i podras comparar els patrons de cadascun.

---

## Teoria

### Race Conditions en Python: El GIL No Et Salva

Python te el Global Interpreter Lock (GIL), un mecanisme que fa que **nomes un thread executi bytecode Python alhora**. Molts developers creuen que aixo elimina les race conditions. **Es fals.**

El GIL garanteix que la JVM de Python (CPython) no es corromp internament. Pero `count += 1` **no es un sol bytecode** — son tres instruccions:

```python
# count += 1 es compila a:
LOAD_GLOBAL  count    # 1. Llegeix el valor actual de count
BINARY_ADD   1        # 2. Suma 1
STORE_GLOBAL count    # 3. Escriu el nou valor

# El GIL pot canviar de thread ENTRE qualsevol d'aquestes instruccions
# Si Thread A llegeix count=42 i el GIL dona el torn a Thread B
# abans que A escrigui, Thread B també llegirà 42 → increment perdut
```

**Regla:** El GIL protegeix les estructures internes de CPython, **no el teu codi**. Qualsevol operacio composta (read-modify-write) es vulnerable a race conditions.

### threading: L'Equivalent a Java Threads

El modul `threading` de Python funciona gairebe identic als threads de Java. Els threads comparteixen memoria, i les mateixes trampes apliquen.

```python
import threading

# Crear i llançar un thread — idèntic al concepte de Java
# target és la funció que executarà el thread (com un Runnable)
# args són els arguments que rebrà la funció
t = threading.Thread(target=la_teva_funcio, args=("argument1",))
t.start()   # Llança el thread — comença l'execució en paral·lel
t.join()    # Espera que el thread acabi — com Thread.join() a Java
```

### threading.Lock(): L'Equivalent a synchronized

```python
import threading

lock = threading.Lock()

# El lock garanteix exclusió mútua — com synchronized a Java
# Només un thread pot tenir el lock alhora; la resta esperen
with lock:
    # Secció crítica — codi protegit
    # Només un thread pot estar aquí dins alhora
    count += 1
```

La construccio `with lock:` es l'equivalent de `synchronized` a Java. Adquireix el lock a l'entrada i l'allibera a la sortida (fins i tot si hi ha una excepcio).

### asyncio: Concurrencia per a I/O sense Threads

`asyncio` es el model de concurrencia modern de Python per a operacions d'I/O (crides a APIs, lectures de fitxers, queries a BD). A diferencia de `threading`, **no usa multiples threads**. Usa un sol thread amb un event loop que gestiona multiples tasques.

```
threading (Java i Python):
┌──────────────────────────────────┐
│ Thread-1: fetch(api_1) ──[espera 300ms]──→ resultat │
│ Thread-2: fetch(api_2) ──[espera 100ms]──→ resultat │
│ Thread-3: fetch(api_3) ──[espera 200ms]──→ resultat │
└──────────────────────────────────┘
3 threads, cadascun bloquejat esperant

asyncio (Python):
┌──────────────────────────────────┐
│ Event Loop (1 sol thread):                           │
│   Task-1: fetch(api_1) → [espera] → reprèn          │
│   Task-2: fetch(api_2) → [espera] → reprèn          │
│   Task-3: fetch(api_3) → [espera] → reprèn          │
│                                                      │
│   Mentre Task-1 espera la xarxa, l'event loop       │
│   executa Task-2 o Task-3. Ningú està bloquejat.    │
└──────────────────────────────────┘
1 thread, mai bloquejat
```

### Comparativa de Patrons Java vs Python

| Concepte | Java | Python |
|----------|------|--------|
| Thread basic | `new Thread(() -> {...}).start()` | `threading.Thread(target=fn).start()` |
| Lock | `synchronized` | `threading.Lock()` |
| Async I/O | `CompletableFuture.supplyAsync()` | `asyncio.create_task()` |
| Esperar multiples | `CompletableFuture.allOf()` | `asyncio.gather()` |
| Combinar resultats | `.thenCombine()` | `await asyncio.gather()` retorna llista |
| Gestio d'errors | `.exceptionally()` | `return_exceptions=True` a `gather()` |

### async/await: La Sintaxi

```python
import asyncio

# "async def" defineix una coroutine — una funció que pot ser pausada i repressa
# L'event loop gestiona quan s'executa cada part
async def fetch_champion(champion_id: str) -> dict:
    # "await" pausa la coroutine fins que l'operació I/O acaba
    # Mentre espera, l'event loop pot executar altres coroutines
    await asyncio.sleep(0.3)  # Simula 300ms de latència de xarxa
    return {"id": champion_id, "name": champion_id, "winRate": 52.3}

async def main():
    # create_task llança la coroutine — equivalent a supplyAsync()
    # La tasca comença immediatament però no bloqueja
    task1 = asyncio.create_task(fetch_champion("Ahri"))
    task2 = asyncio.create_task(fetch_champion("Zed"))

    # gather espera que totes les tasques acabin — equivalent a allOf()
    # Retorna una llista amb tots els resultats, en el mateix ordre
    results = await asyncio.gather(task1, task2)
    return results

# asyncio.run() crea l'event loop i executa la coroutine principal
results = asyncio.run(main())
```

---

## Activitat

### 1. Race Condition en Python (20 min)

Replica l'exercici de dilluns en Python per demostrar que el GIL no protegeix:

```python
# race_condition.py
# Demostra que el GIL de Python NO protegeix contra race conditions
# count += 1 és LOAD + ADD + STORE — el GIL pot canviar de thread entre elles

import threading
import time

# Variable global compartida entre tots els threads
count = 0

def increment_unsafe(iterations: int) -> None:
    """Incrementa el comptador sense protecció — race condition garantida."""
    global count
    for _ in range(iterations):
        count += 1  # NO és atòmic: LOAD_GLOBAL + BINARY_ADD + STORE_GLOBAL

def increment_safe(lock: threading.Lock, iterations: int) -> None:
    """Incrementa el comptador amb lock — equivalent a synchronized de Java."""
    global count
    for _ in range(iterations):
        # El lock garanteix que només un thread fa LOAD+ADD+STORE alhora
        with lock:
            count += 1

def run_test(name: str, target_fn, num_threads: int, iterations: int, **kwargs) -> None:
    """Llança N threads i mesura temps i correcció."""
    global count
    count = 0  # Reset

    start = time.perf_counter()

    threads = []
    for _ in range(num_threads):
        t = threading.Thread(target=target_fn, args=(iterations,), kwargs=kwargs)
        t.start()
        threads.append(t)

    # join() espera que cada thread acabi — idèntic a Java
    for t in threads:
        t.join()

    elapsed_ms = (time.perf_counter() - start) * 1000
    expected = num_threads * iterations

    status = "CORRECTE" if count == expected else f"INCORRECTE (perduts: {expected - count})"
    print(f"{name:20s} Temps: {elapsed_ms:7.1f}ms  Esperat: {expected:>10,}  Real: {count:>10,}  {status}")

if __name__ == "__main__":
    NUM_THREADS = 10
    ITERATIONS = 100_000

    print("=== Race Condition en Python ===")
    print(f"{NUM_THREADS} threads x {ITERATIONS:,} increments\n")

    # Test 1: Sense protecció — demostra la race condition
    run_test("Unsafe (no lock)", increment_unsafe, NUM_THREADS, ITERATIONS)

    # Test 2: Amb Lock — equivalent a synchronized
    lock = threading.Lock()
    # Passem el lock com a argument extra
    def safe_wrapper(iterations):
        increment_safe(lock, iterations)
    run_test("Lock (synchronized)", safe_wrapper, NUM_THREADS, ITERATIONS)

    print("\nNota: el GIL NO protegeix contra race conditions en operacions compostes.")
    print("count += 1 és LOAD + ADD + STORE — el GIL pot canviar entre elles.")
```

Executa-ho:

```bash
python3 race_condition.py
# La versió unsafe perdrà increments. La versió amb Lock serà correcta.
```

### 2. Concurrencia I/O amb asyncio (25 min)

Ara implementa l'equivalent de `CompletableFuture` de dimarts en Python:

```python
# async_fetcher.py
# Equivalent al ParallelFetcher.java de dimarts, però amb asyncio
# Un sol thread, múltiples tasques I/O concurrents

import asyncio
import time

async def fetch_riot_data(champion_id: str) -> dict:
    """Simula crida a Riot API — 300ms de latència."""
    # asyncio.sleep és la versió async de time.sleep
    # A diferència de time.sleep (que bloqueja el thread),
    # asyncio.sleep retorna el control a l'event loop
    await asyncio.sleep(0.3)
    return {"source": "riot", "champion": champion_id, "winRate": 52.3}

async def fetch_data_dragon(champion_id: str) -> dict:
    """Simula crida a Data Dragon — 100ms de latència."""
    await asyncio.sleep(0.1)
    return {"source": "dataDragon", "champion": champion_id, "imageUrl": f"https://dd.cdn/{champion_id}.png"}

async def fetch_sequential(champion_id: str) -> tuple:
    """Versió seqüencial — espera una crida abans de començar l'altra."""
    riot = await fetch_riot_data(champion_id)
    dd = await fetch_data_dragon(champion_id)
    return riot, dd

async def fetch_parallel(champion_id: str) -> tuple:
    """Versió paral·lela — llança les dues crides alhora amb gather."""
    # asyncio.gather és l'equivalent de CompletableFuture.allOf()
    # Llança totes les coroutines i espera que totes acabin
    # Retorna una llista amb els resultats en el MATEIX ordre
    riot, dd = await asyncio.gather(
        fetch_riot_data(champion_id),
        fetch_data_dragon(champion_id)
    )
    return riot, dd

async def main():
    champion_id = "Ahri"

    # --- Seqüencial ---
    start = time.perf_counter()
    riot, dd = await fetch_sequential(champion_id)
    seq_ms = (time.perf_counter() - start) * 1000
    print(f"Seqüencial:  {seq_ms:.0f}ms (esperat: ~400ms)")
    print(f"  Riot: {riot}")
    print(f"  DD:   {dd}")

    # --- Paral·lel ---
    start = time.perf_counter()
    riot, dd = await fetch_parallel(champion_id)
    par_ms = (time.perf_counter() - start) * 1000
    print(f"\nParal·lel:   {par_ms:.0f}ms (esperat: ~300ms)")
    print(f"  Riot: {riot}")
    print(f"  DD:   {dd}")

    print(f"\nDiferència:  {seq_ms - par_ms:.0f}ms estalviats")

# asyncio.run() crea l'event loop i executa la coroutine main
asyncio.run(main())
```

### 3. Extractor Concurrent de 50 Campions (20 min)

Escala a 50 campions per veure la diferencia real:

```python
# batch_extractor.py
# Extreu dades de 50 campions en paral·lel amb asyncio
# Seqüencial: 50 × 300ms = 15s. Paral·lel: ~300ms.
# Inclou gestió d'errors parcials — si algunes crides fallen, no perd les altres

import asyncio
import time
import random
from dataclasses import dataclass

@dataclass(frozen=True)
class ChampionData:
    """Dades d'un campió. frozen=True el fa immutable i thread-safe (com record a Java)."""
    champion_id: str
    riot_data: str
    dd_data: str

@dataclass(frozen=True)
class ExtractionResult:
    """Resultat complet: campions extrets + errors parcials."""
    champions: list   # Llista de ChampionData
    errors: list      # Llista de (champion_id, error_message)

async def fetch_riot_data(champion_id: str) -> str:
    """Simula crida a Riot API amb 10% de probabilitat de fallar."""
    await asyncio.sleep(0.3)  # 300ms de latència
    # Simula errors de xarxa aleatoris — a producció, les APIs fallen
    if random.random() < 0.1:
        raise ConnectionError(f"Timeout connectant a Riot API per {champion_id}")
    return f"RiotData({champion_id}, wr=52.3)"

async def fetch_data_dragon(champion_id: str) -> str:
    """Simula crida a Data Dragon."""
    await asyncio.sleep(0.1)
    return f"DDData({champion_id}, img=ok)"

async def extract_one(champion_id: str) -> ChampionData:
    """Extreu dades d'un campió (Riot + DD en paral·lel)."""
    # gather amb les dues fonts — paral·lel dins de cada campió
    riot, dd = await asyncio.gather(
        fetch_riot_data(champion_id),
        fetch_data_dragon(champion_id)
    )
    return ChampionData(champion_id=champion_id, riot_data=riot, dd_data=dd)

async def extract_all(champion_ids: list[str]) -> ExtractionResult:
    """
    Extreu dades de tots els campions en paral·lel.
    Gestiona errors parcials: si algunes crides fallen,
    retorna els resultats vàlids + la llista d'errors.
    """
    # Llançar TOTES les extraccions alhora
    # return_exceptions=True fa que els errors es retornin com a valors
    # en lloc de petar tot el gather — CLAU per gestió d'errors parcials
    tasks = [extract_one(cid) for cid in champion_ids]
    results = await asyncio.gather(*tasks, return_exceptions=True)

    champions = []
    errors = []

    for cid, result in zip(champion_ids, results):
        if isinstance(result, Exception):
            # Error parcial: anotem-lo i continuem amb la resta
            errors.append((cid, str(result)))
        else:
            champions.append(result)

    return ExtractionResult(champions=champions, errors=errors)

async def main():
    # Llista de 50 campions
    champion_ids = [f"Champion_{i:03d}" for i in range(50)]

    # --- Versió seqüencial (per comparar) ---
    print("=== Versió Seqüencial ===")
    start = time.perf_counter()
    seq_results = []
    for cid in champion_ids[:10]:  # Només 10, o trigaríem massa
        try:
            data = await extract_one(cid)
            seq_results.append(data)
        except Exception as e:
            pass
    seq_ms = (time.perf_counter() - start) * 1000
    print(f"10 campions seqüencials: {seq_ms:.0f}ms (esperat: ~3000ms)")

    # --- Versió paral·lela (tots 50) ---
    print("\n=== Versió Paral·lela ===")
    start = time.perf_counter()
    result = await extract_all(champion_ids)
    par_ms = (time.perf_counter() - start) * 1000

    print(f"50 campions paral·lels:  {par_ms:.0f}ms (esperat: ~300ms)")
    print(f"Campions extrets:        {len(result.champions)}")
    print(f"Errors parcials:         {len(result.errors)}")

    if result.errors:
        print("\nErrors:")
        for cid, msg in result.errors:
            print(f"  {cid}: {msg}")

    # Mostra els primers 3 resultats
    print("\nPrimers resultats:")
    for champ in result.champions[:3]:
        print(f"  {champ.champion_id} → {champ.riot_data}")

    # Verificació
    assert par_ms < 1000, f"L'extractor hauria de trigar <1s, ha trigat {par_ms:.0f}ms"
    print(f"\nVerificació OK: {par_ms:.0f}ms < 1000ms")

asyncio.run(main())
```

### 4. Comparativa de Patrons (10 min)

Crea un fitxer de notes comparant els dos llenguatges:

```
Patró Java vs Python — Notes personals

CompletableFuture.supplyAsync() ↔ asyncio.create_task()
  - Java: llança en un thread del ForkJoinPool
  - Python: registra la coroutine a l'event loop (1 sol thread)

CompletableFuture.allOf() ↔ asyncio.gather()
  - Java: espera N futures
  - Python: espera N coroutines, retorna llista de resultats

.thenCombine() ↔ await asyncio.gather() amb unpacking
  - Java: combina 2 resultats amb lambda
  - Python: retorna tupla que pots destructurar

.exceptionally() ↔ return_exceptions=True
  - Java: encadena handler d'error per future
  - Python: gather retorna excepcions com a valors dins la llista

Quina sintaxi prefereixes? Per què?
Quin codi és més fàcil de depurar? Per què?
```

> **Lectura recomanada (opcional, no bloquejant):**
> - Real Python: [Async IO in Python](https://realpython.com/async-io-python/)
> - Real Python: [An Intro to Threading in Python](https://realpython.com/intro-to-python-threading/)

---

## Checklist de Lliurament

- [ ] Has demostrat la race condition en Python amb `threading` i has vist que el GIL no protegeix
- [ ] Has implementat la solucio amb `threading.Lock()` i funciona correctament
- [ ] Has implementat l'extractor amb `asyncio` que fa 50 crides en paral-lel en menys d'1 segon
- [ ] La gestio d'errors parcials funciona: si algunes crides fallen, els resultats valids es conserven
- [ ] Has comparat els patrons Java vs Python i tens les teves notes amb les equivalencies
- [ ] Pots explicar: que es el GIL, per que `count += 1` no es atomic en Python, i la diferencia entre `threading` i `asyncio`
