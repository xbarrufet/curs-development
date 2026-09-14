# Setmana 1 — Dijous: Benchmarking de Temps i Memòria

## Objectiu del Dia

Mesurar la diferència real de rendiment entre cerca lineal i cerca per HashMap amb 100.000 elements. Entendre per què mesurar malament dona resultats enganyosos (warm-up JIT) i com fer-ho correctament. Al final del dia tens un `BenchmarkRunner` executable que imprimeix un quadre comparatiu.

---

## Teoria

### Per Què No Pots Mesurar amb un Cronòmetre Ingenu

La temptació és fer:
```java
long start = System.nanoTime();
searchLinear(players, "P-100000");
long elapsed = System.nanoTime() - start;
System.out.println("Triga: " + elapsed + " ns");
```

Això dona resultats **enganyosos** per tres motius:

**1. La JVM interpreta abans de compilar (JIT)**

Java no executa el teu codi directament — primer l'interpreta (lent), i quan detecta que un mètode s'executa moltes vegades, el compila a codi natiu de la CPU (ràpid). Les primeres execucions d'un mètode són molt més lentes que les posteriors.

```
Execució 1-10:    interpretat     → 500μs per cerca
Execució 11-100:  compilant JIT   → 200μs per cerca
Execució 101+:    codi natiu      → 50μs per cerca
```

Si mesurem la primera execució, estem mesurant l'intèrpret, no el nostre algorisme.

**2. Una sola execució té massa soroll**

El sistema operatiu pot pausar el teu programa per atendre altres processos. Si mesurem una sola cerca, un canvi de context de la CPU pot afegir mil·lisegons espuris. Solució: executar 1.000 vegades i mesurar el total.

**3. El Garbage Collector pot disparar-se**

Java allibera memòria automàticament (GC). Si es dispara a mig benchmark, els resultats es distorsionen. No ho podem evitar del tot, però el warm-up ajuda a estabilitzar.

### Com Mesurar Correctament

```
1. WARM-UP:  Executa cada mètode 100 vegades SENSE mesurar
             → Força la compilació JIT
             → Estabilitza la memòria

2. MESURA:   Executa cada mètode 1.000 vegades amb System.nanoTime()
             → Les 1.000 repeticions suavitzen el soroll
             → El total es converteix a mil·lisegons dividint per 1.000.000

3. MEMÒRIA:  Consulta Runtime.getRuntime().totalMemory() - freeMemory()
             → Dona una aproximació dels MB usats per la JVM
```

### `System.nanoTime()` vs `System.currentTimeMillis()`

- `nanoTime()` — precisió de nanosegons, ideal per mesurar durades curtes. No lligat al rellotge del sistema.
- `currentTimeMillis()` — precisió de mil·lisegons, massa gros per a microbenchmarks. Pot saltar si l'OS ajusta el rellotge.

Per a benchmarks, sempre `nanoTime()`.

> **Lectura recomanada (opcional, no bloquejant):**
> - Baeldung — [Microbenchmarking with System.nanoTime](https://www.baeldung.com/java-system-nanotime)
> - Oracle Java API — [Class Runtime](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Runtime.html) (`totalMemory()`, `freeMemory()`)

---

## Activitat

### 1. Crear `BenchmarkRunner` (60 min)

Crea el fitxer:
```
backend-java/src/main/java/com/esportspulse/engine/benchmark/BenchmarkRunner.java
```

Programa una classe amb mètode `main` que faci el següent, en aquest ordre:

1. **Genera les dades:** 100.000 jugadors en `ArrayList` i el `HashMap` equivalent (usant `PlayerDataGenerator`).

2. **Defineix l'objectiu de cerca:** `"P-100000"` — el pitjor cas per la cerca lineal (últim element).

3. **Warm-up:** Executa 100 cerques de cada tipus (lineal i hash) **sense mesurar**. Això força la compilació JIT.

4. **Mesura la cerca lineal:**
   - Pren el temps amb `System.nanoTime()` abans
   - Executa `searchLinear()` 1.000 vegades
   - Pren el temps després
   - Calcula el total en mil·lisegons: `(end - start) / 1_000_000.0`

5. **Mesura la cerca per HashMap:**
   - Mateix procés: 1.000 cerques amb `searchByKey()`
   - Calcula el total en mil·lisegons

6. **Mesura la memòria:**
   ```java
   Runtime rt = Runtime.getRuntime();
   long usedMB = (rt.totalMemory() - rt.freeMemory()) / (1024 * 1024);
   ```

7. **Imprimeix el quadre comparatiu:**
   ```
   === Benchmark: 100.000 elements, 1.000 cerques ===
   Linear search:  XX.XX ms
   HashMap search:  X.XX ms
   Ratio:          XXx
   RAM usada:      XX MB
   ```

### 2. Experiment sense warm-up (15 min)

Comenta les línies de warm-up i torna a executar el benchmark. Compara els resultats amb els anteriors.

Preguntes a respondre (en un comentari breu al codi o mentalment):
- La cerca lineal és més lenta o més ràpida sense warm-up? Per què?
- El ratio ha canviat? En quina direcció?

Descomenta el warm-up quan acabis — el benchmark final ha de ser correcte.

### 3. Commit (5 min)

```bash
git add backend-java/src/main/java/com/esportspulse/engine/benchmark/BenchmarkRunner.java
git commit -m "feat(java): add benchmark runner for O(n) vs O(1) comparison"
```

---

## Checklist de Lliurament

- [ ] `BenchmarkRunner` s'executa i imprimeix temps, ratio i memòria
- [ ] El warm-up està implementat (100 iteracions sense mesurar)
- [ ] La mesura usa `System.nanoTime()` amb 1.000 iteracions
- [ ] El ratio HashMap/lineal és d'almenys 10x (resultat típic: 30-50x)
- [ ] Has vist la diferència amb i sense warm-up
- [ ] Commit fet a `feature/week1-benchmarking`
