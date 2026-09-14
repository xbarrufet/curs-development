# Setmana 1 — Exercicis de Consolidació

Aquests exercicis es fan **sense instruccions pas a pas**. Si no saps per on començar, revisa la teoria i els exercicis de la setmana. Si segueixes encallat, demana ajuda — però intenta-ho primer sol.

---

## Bàsics (has de saber fer-ho)

### 1. Benchmark amb mida variable
Modifica el `BenchmarkRunner` perquè accepti la mida de dades com a argument de línia de comandes (`java BenchmarkRunner 500000`). Executa'l amb 10.000, 100.000, 500.000 i 1.000.000 d'elements. Imprimeix una taula amb els resultats. **Pregunta:** El ratio HashMap/ArrayList es manté constant o canvia amb la mida? Per què?

**Fet quan:** La taula mostra 4 files amb resultats coherents i la pregunta té resposta al codi (comentari d'1 línia).

### 2. Cerca per rang de preu
Afegeix un mètode `findByPriceRange(BigDecimal min, BigDecimal max)` al `GameSearchService` que retorni tots els jocs dins d'un rang de preu. Implementa'l per a `List` i per a `HashMap`. **Pregunta:** Pot el `HashMap` ser O(1) per a aquesta cerca? Per què no?

**Fet quan:** Ambdós mètodes funcionen, el test JUnit passa, i la resposta a la pregunta és correcta (una cerca per rang no pot ser O(1) perquè necessites recórrer valors, no claus).

### 3. Python mirror
Tradueix `GameSearchService` a Python amb `list` i `dict`. Executa el benchmark equivalent amb `timeit`. Compara els resultats amb Java. No cal que sigui idèntic — l'objectiu és que el codi Python funcioni i mesuris el ratio.

**Fet quan:** Script Python executable que imprimeix el ratio linear vs dict per a 100.000 elements.

---

## Avançats (si vas sobrat)

### 4. Detectar duplicats
Escriu un mètode que detecti jocs duplicats (mateix `title`, diferent `appId`) dins d'una llista de 100.000 jocs. Primer fes-ho amb doble bucle O(n²), després amb un `HashSet` O(n). Mesura la diferència.

**Fet quan:** Test que verifica que ambdós mètodes troben els mateixos duplicats; benchmark que demostra la diferència de temps.

### 5. El HashMap que col·lisiona
Crea un `GameRecord` custom amb un `hashCode()` que sempre retorna el mateix valor (ex: `return 1;`). Posa 10.000 d'aquests en un `HashMap`. Mesura el temps de `get()` i compara amb un `hashCode()` normal. Explica el resultat.

**Fet quan:** Benchmark que mostra la degradació; explicació d'1 frase de per què passa (totes les entrades van al mateix bucket → O(n)).
