# Setmana 4 — Dilluns: Threads, Race Conditions i Operacions Atomiques

## Objectiu del Dia

Entendre com un servidor web gestiona centenars de peticions simultanees: el model thread-per-request. Veure amb els teus propis ulls com dos threads accedint a la mateixa variable produeixen resultats incorrectes (race condition), i aprendre les dues solucions fonamentals — `synchronized` i `AtomicInteger`. Al final del dia sabras per que les aplicacions web tenen bugs de concurrencia i com evitar-los amb mecanismes de Java.

---

## Teoria

### Per Que la Concurrencia es el Teu Problema

A les setmanes anteriors has escrit codi Java i Python, has apres SOLID i has treballat amb el terminal. Tot aixo era codi que s'executava en un sol fil. Pero a la vida real, quan el teu backend rep 200 peticions simultanees, **el teu codi s'executa 200 vegades alhora**. I aqui comencen els problemes.

```
500 usuaris simultanis:
Usuari A → GET  /champions/Ahri
Usuari B → POST /champions           (crear campió)
Usuari C → PUT  /champions/Ahri      (canviar estadístiques)
Usuari D → GET  /champions/Ahri
...500 peticions al mateix temps
```

**Pregunta:** Com gestiona Spring Boot 500 peticions alhora si el teu codi es un sol fitxer `ChampionController.java`?

**Resposta:** Amb **threads**. Cada peticio s'executa en un thread diferent, en paral-lel.

### Que es un Thread?

Un **thread** (fil d'execucio) es la unitat minima d'execucio dins d'un proces. Imagina un restaurant:

- **El proces** es la cuina sencera: forn, nevera, estris, ingredients
- **Cada thread** es un cuiner dins d'aquesta cuina
- Tots els cuiners comparteixen els mateixos ingredients (memoria), pero cadascun segueix la seva propia recepta (pila d'execucio)

```
Procés Java (JVM):
┌──────────────────────────────────────────────────────┐
│                                                      │
│  HEAP (compartit per tots els threads):               │
│  ┌──────────────────────────────────────────────┐    │
│  │ ChampionService (singleton)                   │    │
│  │ ConcurrentHashMap<String, Champion>            │    │
│  │ Objectes, instàncies, dades compartides        │    │
│  └──────────────────────────────────────────────┘    │
│                                                      │
│  STACKS (privats per cada thread):                    │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │Thread-1 │ │Thread-2 │ │Thread-3 │ │Thread-4 │   │
│  │ locals  │ │ locals  │ │ locals  │ │ locals  │   │
│  │ criades │ │ criades │ │ criades │ │ criades │   │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘   │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**La clau:** Les variables locals (stack) son privades. Pero el heap (objectes, services, repositoris) es **compartit**. Quan dos threads modifiquen el mateix objecte al heap, tens una race condition.

### El Model Thread-per-Request

Quan arrenques Spring Boot, Tomcat crea un **pool de 200 threads** (per defecte). Cada peticio HTTP agafa un thread del pool, executa el teu codi (Controller - Service - Repository - BD), i retorna el thread al pool.

```
                    ┌──────────────────────────────────┐
                    │         Tomcat (Spring Boot)      │
                    │                                   │
Client A ────────── │ ──→ Thread-1 → Controller →       │
Client B ────────── │ ──→ Thread-2 → Controller →       │ ──→ Base de Dades
Client C ────────── │ ──→ Thread-3 → Controller →       │
                    │         ...                       │
Client N ────────── │ ──→ Thread-N → Controller →       │
                    │                                   │
                    │  Thread Pool: 200 threads (defecte)│
                    └──────────────────────────────────┘
```

**Implicacio practica:** El teu `ChampionService` es un singleton — una sola instancia compartida per tots els threads. Si el service te estat mutable (una variable que es modifica), tens un bug esperant a passar.

### Race Conditions: El Bug Invisible

Una race condition passa quan dos threads accedeixen a les mateixes dades al mateix temps i almenys un les modifica. El resultat depen de l'ordre d'execucio, que es **impredictible**.

L'operacio `count++` sembla una sola instruccio, pero internament son **tres**:

```
1. LLEGIR:   registre = count       (llegeix 42)
2. CALCULAR: registre = registre + 1  (calcula 43)
3. ESCRIURE: count = registre       (escriu 43)
```

Si dos threads fan `count++` alhora:

```
Thread A                    Thread B
────────                    ────────
LLEGIR count = 42
                            LLEGIR count = 42    ← Llegeix el MATEIX valor!
CALCULAR 42 + 1 = 43
                            CALCULAR 42 + 1 = 43
ESCRIURE count = 43
                            ESCRIURE count = 43  ← Sobreescriu amb 43!

Resultat: count = 43 (hauria de ser 44). Un increment s'ha perdut.
```

### Solucio 1: `synchronized` (Exclusio Mutua)

`synchronized` posa un lock al metode: quan un thread hi entra, la resta esperen fora.

```java
public class SafeCounter {
    // Camp compartit entre tots els threads
    private int count = 0;

    // synchronized garanteix que només un thread executa aquest mètode alhora
    // La resta de threads esperen en cua fins que el lock s'allibera
    public synchronized void increment() {
        count++;  // Ara és segur: cap altre thread pot interrompre
    }

    public synchronized int getCount() {
        return count;
    }
}
```

**Problema:** Si tots els threads esperen el lock, perdem paral-lelisme. En un servidor web, aixo pot convertir 200 threads en efectivament 1.

### Solucio 2: `AtomicInteger` (Operacions Atomiques)

`AtomicInteger` utilitza instruccions especials del processador (CAS — Compare-And-Swap) que garanteixen atomicitat **sense lock**:

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicCounter {
    // AtomicInteger encapsula un int amb operacions thread-safe
    // Usa instruccions CAS del processador: compara el valor actual amb l'esperat,
    // i només escriu si coincideixen. Si no, reintenta automàticament.
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        // incrementAndGet() és atòmic a nivell de CPU — no necessita lock
        count.incrementAndGet();
    }

    public int getCount() {
        return count.get();
    }
}
```

**Avantatge:** Molt mes rapid que `synchronized` perque no bloqueja threads. Quan hi ha poca contencio (pocs threads competint pel mateix recurs), es ordres de magnitud mes eficient.

### Comparativa de Rendiment

| Solucio | 10 threads x 1M increments | Correcte? |
|---------|---------------------------|-----------|
| `count++` (unsafe) | ~50ms | No — perd increments |
| `synchronized` | ~800ms | Si, pero lent |
| `AtomicInteger` | ~200ms | Si, i rapid |

### Connexio amb S2: Per Que la Immutabilitat Importa

A S2 vam insistir que `GameRecord` fos un `record` (immutable). Ara entens per que: si un objecte no es pot modificar despres de crear-lo, no hi ha race condition possible. **La immutabilitat es una estrategia de concurrencia.**

```java
// record és immutable per disseny → thread-safe automàticament
// Cap thread pot modificar els camps després de la creació
public record Champion(String id, String name, int attack, int defense) {}

// Si necessites "modificar", crees un nou objecte
// Això és segur perquè cap thread veu un objecte a mig canviar
Champion updated = new Champion(champ.id(), champ.name(), newAttack, champ.defense());
```

---

## Activitat

### 1. Demostrar la Race Condition (20 min)

Crea una classe `UnsafeCounter` i demostra que `count++` no es atomic:

```java
// UnsafeCounter.java
// Aquesta classe demostra el problema fonamental de la concurrència:
// una operació que SEMBLA atòmica (count++) en realitat NO ho és

public class UnsafeCounter {
    private int count = 0;

    public void increment() {
        count++;  // PERILL: read-modify-write NO és atòmic
    }

    public int getCount() {
        return count;
    }

    public static void main(String[] args) throws InterruptedException {
        UnsafeCounter counter = new UnsafeCounter();
        int numThreads = 10;
        int incrementsPerThread = 100_000;

        // Creem 10 threads, cadascun farà 100.000 increments
        Thread[] threads = new Thread[numThreads];
        for (int i = 0; i < numThreads; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < incrementsPerThread; j++) {
                    counter.increment();
                }
            });
        }

        // Llançem tots els threads
        for (Thread t : threads) {
            t.start();
        }

        // Esperem que tots acabin
        for (Thread t : threads) {
            t.join();
        }

        // Resultat esperat: 1.000.000. Resultat real: menys!
        int expected = numThreads * incrementsPerThread;
        System.out.println("Esperat: " + expected);
        System.out.println("Real:    " + counter.getCount());
        System.out.println("Perduts: " + (expected - counter.getCount()));
    }
}
```

Compila i executa **5 vegades**. Observa que el resultat canvia cada vegada — es impredictible.

```bash
# Compila i executa
javac UnsafeCounter.java
java UnsafeCounter
# Executa diverses vegades — el valor real mai és 1.000.000
```

### 2. Implementar les Dues Solucions (25 min)

Implementa `SafeCounter` amb `synchronized` i `AtomicCounter` amb `AtomicInteger`. Mesura el temps de cadascuna:

```java
// CounterBenchmark.java
// Compara les tres aproximacions: unsafe, synchronized, i AtomicInteger
// Mesura tant la correcció com el rendiment

import java.util.concurrent.atomic.AtomicInteger;

public class CounterBenchmark {

    // Versió 1: Unsafe — count++ sense protecció
    static class UnsafeCounter {
        private int count = 0;
        public void increment() { count++; }
        public int getCount() { return count; }
    }

    // Versió 2: synchronized — exclusió mútua amb lock implícit
    // Només un thread pot executar increment() alhora
    static class SyncCounter {
        private int count = 0;
        public synchronized void increment() { count++; }
        public synchronized int getCount() { return count; }
    }

    // Versió 3: AtomicInteger — operacions CAS sense lock
    // El processador garanteix l'atomicitat amb instruccions especials
    static class AtomicCounter {
        private final AtomicInteger count = new AtomicInteger(0);
        public void increment() { count.incrementAndGet(); }
        public int getCount() { return count.get(); }
    }

    // Mètode genèric que llança N threads fent M increments cadascun
    // Accepta qualsevol Runnable, així podem testejar les tres versions
    static long benchmark(String name, Runnable incrementFn, int numThreads,
                          int incrementsPerThread, java.util.function.IntSupplier getCount)
                          throws InterruptedException {

        long start = System.nanoTime();

        Thread[] threads = new Thread[numThreads];
        for (int i = 0; i < numThreads; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < incrementsPerThread; j++) {
                    incrementFn.run();
                }
            });
        }

        for (Thread t : threads) t.start();
        for (Thread t : threads) t.join();

        long elapsed = (System.nanoTime() - start) / 1_000_000;  // ms
        int expected = numThreads * incrementsPerThread;
        int actual = getCount.getAsInt();

        System.out.printf("%-15s Temps: %4dms  Esperat: %,d  Real: %,d  %s%n",
            name, elapsed, expected, actual,
            actual == expected ? "CORRECTE" : "INCORRECTE (perduts: " + (expected - actual) + ")");

        return elapsed;
    }

    public static void main(String[] args) throws InterruptedException {
        int threads = 10;
        int increments = 1_000_000;

        System.out.println("=== Benchmark de Concurrència ===");
        System.out.println(threads + " threads x " + increments + " increments cadascun\n");

        // Unsafe
        UnsafeCounter unsafe = new UnsafeCounter();
        benchmark("Unsafe", unsafe::increment, threads, increments, unsafe::getCount);

        // Synchronized
        SyncCounter sync = new SyncCounter();
        benchmark("Synchronized", sync::increment, threads, increments, sync::getCount);

        // AtomicInteger
        AtomicCounter atomic = new AtomicCounter();
        benchmark("AtomicInteger", atomic::increment, threads, increments, atomic::getCount);
    }
}
```

### 3. Demostrar Race Condition amb ArrayList (15 min)

Els `@Service` de Spring son singletons. Si tenen estat mutable, el bug es inevitable:

```java
// SharedStateDemo.java
// Demostra per què MAI has de tenir estat mutable en un @Service
// ArrayList NO és thread-safe — amb múltiples threads, es corromp

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class SharedStateDemo {
    // Simula un @Service amb estat mutable — ANTIPATRÓ
    private static final List<String> recentSearches = new ArrayList<>();

    public static void main(String[] args) throws InterruptedException {
        int numThreads = 10;
        int operationsPerThread = 1_000;

        Thread[] threads = new Thread[numThreads];
        for (int i = 0; i < numThreads; i++) {
            final int threadId = i;
            threads[i] = new Thread(() -> {
                for (int j = 0; j < operationsPerThread; j++) {
                    // Afegir a un ArrayList des de múltiples threads
                    // pot llançar ConcurrentModificationException o perdre elements
                    try {
                        recentSearches.add("search-" + threadId + "-" + j);
                    } catch (Exception e) {
                        System.out.println("EXCEPCIÓ: " + e.getClass().getSimpleName());
                    }
                }
            });
        }

        for (Thread t : threads) t.start();
        for (Thread t : threads) t.join();

        int expected = numThreads * operationsPerThread;
        System.out.println("Esperat: " + expected);
        System.out.println("Real:    " + recentSearches.size());
        // Sovint el resultat és diferent, o fins i tot hi ha excepcions
    }
}
```

Solucio: usa `Collections.synchronizedList()` o `CopyOnWriteArrayList`. Pero la millor solucio es **no tenir estat mutable en services**.

### 4. Exercici de Prompt Engineering (10 min)

Demana a Cursor o ChatGPT:

> "Genera un exemple Java 21 de race condition amb ArrayList compartida entre 4 threads."

Executa'l 5 vegades. Respon:
- El resultat canvia entre execucions?
- L'IA t'ha avisat del problema?
- L'explicacio de l'assistent es correcta i completa?

> **Lectura recomanada (opcional, no bloquejant):**
> - Baeldung: [Introduction to Java Threads](https://www.baeldung.com/java-thread-lifecycle)
> - Oracle: [Concurrency Lesson](https://docs.oracle.com/javase/tutorial/essential/concurrency/)

---

## Checklist de Lliurament

- [ ] Has executat `UnsafeCounter` 5 vegades i has vist que el resultat varia
- [ ] Has implementat `SafeCounter` (synchronized) i `AtomicCounter` (AtomicInteger) i ambdos donen el resultat correcte
- [ ] Has mesurat el rendiment de les tres versions i pots explicar per que AtomicInteger es mes rapid
- [ ] Has demostrat la race condition amb ArrayList i entens per que els @Service no han de tenir estat mutable
- [ ] Has fet l'exercici de Prompt Engineering i has evaluat la resposta de l'IA
- [ ] Pots explicar: que es un thread, que es el model thread-per-request, i per que `count++` no es atomic
