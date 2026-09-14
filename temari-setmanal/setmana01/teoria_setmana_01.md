# Setmana 1 - Teoria: Big-O, Algorítmica i Benchmarking

## 1. Introducció: Per Què Importa el Rendiment?

Imagina que GamePulse tiene 100 milions de jocs indexats. Un usuario pregunta: "Quants jugadors actius té el joc amb ID APP-42857?"

**Escenari A (cercador lent):** Recorrem els 100M de jocs un per un fins trobar-lo.
- Si la BD tarda 1 microsegon per joc: **100 milions × 1μs = 100 segons**
- L'usuari espera... i al final es va a la competència.

**Escenari B (cercador ràpid):** Accedim directament a la posició de APP-42857 com si fos un diccionari.
- Temps: **1 microsegon (sempre, sigui 100M o 1M de jocs)**
- L'usuari rep la resposta instantània.

La diferència? **Algoritmes + estructures de dades correctes.**

---

## 2. Notació Big-O: Mesura de Complexitat

### Definició

Big-O ens diu **com escala el temps d'execució conforme creixen les dades** (n).

| Notació | Nom | Exemples | Escala |
|---------|-----|----------|--------|
| **O(1)** | Constant | HashMap lookup, array index | Temps fix, sempre igual |
| **O(log n)** | Logarítmic | Binary search, tree lookup | Creix lentament |
| **O(n)** | Linear | Recorrer un array, búsqueda lineal | Creix proporcionalment |
| **O(n log n)** | Lineal-logarítmic | Merge sort, bons ordenaments | Creix més ràpid que O(n) |
| **O(n²)** | Quadràtic | Bubble sort, nested loops | Creix molt ràpid |
| **O(2^n)** | Exponencial | Força bruta recursiva | **EVITAR A TOT COST** |

### Interpretació Pràctica

Si tens **n = 100,000 jocs**, quant tiempo trigarà?

```
O(1):       1 operació sempre
O(log n):   ~17 operacions (logaritme de 100K)
O(n):       100,000 operacions
O(n²):      10,000,000,000 operacions (10 mil milions!)
```

**Conclusió:** A escala, O(1) i O(n) són diferència entre 1ms i 100 segons.

---

## 3. ArrayList vs HashMap en GamePulse

### ArrayList (O(n) search)

```java
List<GameRecord> games = new ArrayList<>();
games.add(new GameRecord("APP-1", "LoL", 0, 5_000_000));
games.add(new GameRecord("APP-2", "Dota2", 0, 2_000_000));
// ... 100,000 jocs

// Buscar per appId
for (GameRecord g : games) {
    if (g.appId().equals("APP-50000")) {
        return g;  // Worst case: recorre 50,000 items
    }
}
```

**Característiques:**
- Accés per índex: O(1) — `games.get(0)` és ràpid
- Búsqueda per valor: O(n) — **hem de recórrer tots**
- Worst case: l'element és l'últim (o no existeix)

### HashMap (O(1) search promig)

```java
Map<String, GameRecord> games = new HashMap<>();
games.put("APP-1", new GameRecord("APP-1", "LoL", 0, 5_000_000));
games.put("APP-2", new GameRecord("APP-2", "Dota2", 0, 2_000_000));
// ... 100,000 jocs

// Buscar per appId (clau)
GameRecord g = games.get("APP-50000");  // O(1) promig!
```

**Característiques:**
- Basada en **hash function**: convierte la clau en índex
- Búsqueda: O(1) promig (cas normal), O(n) worst case (col·lisions)
- **Price:** usa més memòria (hash table internament)

### Taula Hash Internament

```
HashMap amb 4 slots:
┌──────────────────────┐
│ [0] → APP-1: LoL     │ (hash("APP-1") % 4 = 0)
│ [1] → APP-2: Dota2   │ (hash("APP-2") % 4 = 1)
│ [2] → null           │
│ [3] → APP-50000: ... │ (hash("APP-50000") % 4 = 3)
└──────────────────────┘

Lookup "APP-50000":
1. hash("APP-50000") = 299847 (número gran)
2. 299847 % 4 = 3
3. Accedeix slot [3] directament → TROBET!
```

Sense recórrer res — just calcular i accedir.

---

## 4. Microbenchmarking: Mesurant la Realitat

### Per Què Els Microbenchmarks Senzills Menteixen?

Quan fas:

```java
long start = System.nanoTime();
for (int i = 0; i < 1000; i++) {
    result = linearSearch(games, "APP-50000");
}
long elapsed = System.nanoTime() - start;
```

**Problemes:**
1. **Warm-up JIT:** La JVM no ha compilat el bytecode a native code; els primers iterations són **molts més lents**.
2. **Garbage Collector:** Pot disparar-se a mig benchmark i sesgar resultats.
3. **CPU Caching:** Després de 100 iteracions, els datos ja estan cached; més ràpid que la realitat.
4. **Branch Prediction:** La CPU adivina si entrarem a un `if`; si perd, és lent.

### Benchmark Correcte

```java
// 1. WARM-UP (sin medir)
for (int i = 0; i < 100; i++) {
    linearSearch(games, "APP-" + random());
}

// 2. MESURA REAL (amb iteracions)
long start = System.nanoTime();
for (int i = 0; i < 1000; i++) {
    result = linearSearch(games, "APP-50000");
}
long linearTime = System.nanoTime() - start;

// 3. COMPARACIÓ
start = System.nanoTime();
for (int i = 0; i < 1000; i++) {
    result = games.get("APP-50000");
}
long hashTime = System.nanoTime() - start;

System.out.println("Linear: " + linearTime + "ns");
System.out.println("Hash: " + hashTime + "ns");
System.out.println("Ratio: " + (linearTime / hashTime) + "x");
```

**Resultat típic:**
```
Linear: 50,000,000 ns (50ms per 1000 búsquedas)
Hash: 1,000,000 ns (1ms per 1000 búsquedas)
Ratio: 50x més ràpid amb HashMap!
```

---

## 5. Worst-Case vs Promig: Quan Falla HashMap?

### Worst Case: Col·lisions

```
HashMap amb mala hash function:
┌─────────────────────────────────────┐
│ [0] → APP-1 → APP-11 → APP-111 ...  │ (cadena de col·lisions!)
│ [1] → null                          │
│ [2] → null                          │
│ [3] → null                          │
└─────────────────────────────────────┘

Lookup "APP-111":
1. hash("APP-111") % 4 = 0
2. Accedeix slot [0]
3. Recorre la cadena: APP-1? No. APP-11? No. APP-111? Sí!
→ O(n) en worst case!
```

**Quan passa?** Quan hash function és boja o Hi ha moltes col·lisions degut a coincidencies (rara en pràctica amb Java's HashMap).

### Promig: O(1) real

Java's `HashMap` usa una bona hash function per defecte. Per tant, en pràctica:
- **Millió de jocs:** accés en ~1 microsegon
- **100 milions de jocs:** accés en ~1 microsegon
- La taula s'auto-redimensiona per mantenir aquesta garantia

---

## 6. Memòria: El Trade-Off

### ArrayList: Compact

```
ArrayList de 100K jocs:
┌──────────────────────────────┐
│ Joc 1                        │ ← índex 0
│ Joc 2                        │ ← índex 1
│ ...                          │
│ Joc 100,000                  │ ← índex 99,999
└──────────────────────────────┘
Memòria: ~1MB (compacte)
```

### HashMap: Espaiet (per velocitat)

```
HashMap de 100K jocs amb 200K slots (half-full per evitar col·lisions):
┌──────────────────────────────────────────────────────────────┐
│ [0] → LoL         │ [1] → null        │ [2] → Dota2          │
│ [3] → null        │ [4] → null        │ [5] → Valorant       │
│ ...               │ [200000] → null   │                      │
└──────────────────────────────────────────────────────────────┘
Memòria: ~2MB (més per hash table, però O(1))
```

**Trade-off:** 2x més memòria → 50x més ràpid. **Val la pena.**

---

## 7. A GamePulse: Aplicació Pràctica

### S1: Benchmark O(n) vs O(1)

L'exercici de setmana 1 comprova això en pràctica:
- **100,000 jocs** guardats en `ArrayList` i `HashMap`
- **Worst case búsqueda:** últim joc de la llista
- **Mesura:** Linear triga **40ms**, HashMap triga **0.8ms** → **50x difference**

### S5: HashMap → BD Indexada

A la setmana 5 afegim BD (H2/PostgreSQL). Una BD bem indexada és com un HashMap:
```sql
CREATE INDEX idx_games_appid ON games(appId);
```

Aquesta índex emmagatzema els appIds en una estructura d'arbre (similar a hash table):
- `SELECT * FROM games WHERE appId = 'APP-50000'` → O(log n) ≈ O(1) per a nostre cas

### S7: REST API Ràpida

Quan un client fa:
```
GET /api/games/APP-50000
```

El backend:
1. `gameRepository.findById("APP-50000")` → O(1) via BD index
2. Mapper `entity → DTO` → O(1) sempre
3. `ResponseEntity.ok(dto)` → retorna instantàneament

Total: **< 1ms** en lloc de **50ms** amb algoritme dolent.

---

## 8. Exercici Conceptual: Escull l'Estructura

Tens 1 milió de jocs. Escull l'estructura per a cada cas:

| Cas | Estructura | Per què? |
|-----|-----------|----------|
| Buscar per `appId` (clau única) | HashMap o BD Indexed | O(1) lookup |
| Llistar TOT els jocs | ArrayList o BD table scan | Accés sequential |
| Buscar jocs per rang de preu | ArrayList + filter o BD range query | O(n) però necessari |
| Jocs més populars (top 10) | ArrayList sorted o BD ORDER BY LIMIT | O(n log n) uma vegada, después cache |

---

## 9. Lectura Profunda

- **Big-O Analysis:** [Baeldung](https://www.baeldung.com/big-o-notation)
- **HashMap Internals:** [Baeldung HashMap](https://www.baeldung.com/java-hashmap)
- **Benchmarking Tools:** JMH (Java Microbenchmark Harness)

---

## Resum

| Concepte | Key Takeaway |
|----------|--------------|
| **O(1) vs O(n)** | Diferència de 1ms vs 50ms per 100K items |
| **HashMap** | Taula hash → lookup O(1) promig |
| **ArrayList** | Recorregut lineal → lookup O(n) |
| **Trade-off** | HashMap usa 2x memòria per guanyar 50x velocitat |
| **Warm-up** | JIT compilation: els primers iterations són més lents |
| **Aplicació** | GamePulse usarà HashMap (S1), DB indexed (S5), REST (S7) — tots O(1)/(log n) |

**Objectiu setmana:** Entendre per què les estructures importan; veure-ho en pràctica amb benchmark.
