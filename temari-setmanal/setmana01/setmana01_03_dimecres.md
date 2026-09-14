# Setmana 1 — Dimecres: Big-O, Hash Internals i Cerca a Escala

## Objectiu del Dia

Entendre per què amb 20 jocs no es nota la diferència entre `ArrayList` i `HashMap`, però amb 100.000 sí. Construir un generador de dades massives i un servei de cerca amb els dos enfocaments. Al final del dia tens `GameDataGenerator` i `GameSearchService` funcionant amb 100.000 elements.

---

## Teoria

### Notació Big-O: Com Escala el Teu Codi

Ahir vas veure que buscar per `appId` en una llista requereix un bucle, i en un mapa una sola crida. Amb 15 jocs, les dues opcions triguen menys d'un mil·lisegon — no hi ha diferència perceptible.

Però què passa quan les dades creixen?

Big-O és una notació que descriu **com escala el temps d'execució quan creix el volum de dades (n)**. No mesura mil·lisegons concrets — mesura la tendència.

| Notació | Nom | Què fa | Exemple |
|---------|-----|--------|---------|
| **O(1)** | Constant | Sempre el mateix temps, doni igual quants elements hi hagi | `map.get("APP-50000")` |
| **O(log n)** | Logarítmic | Creix molt lentament | Cerca binària en llista ordenada |
| **O(n)** | Lineal | Creix proporcionalment amb les dades | Recórrer una llista sencera |
| **O(n²)** | Quadràtic | Creix molt ràpidament | Bucle dins de bucle |

**Amb 100.000 jocs (n = 100.000):**

```
O(1):       1 operació           →  instantani
O(log n):   ~17 operacions       →  instantani
O(n):       100.000 operacions   →  notable
O(n²):      10.000.000.000 ops   →  inacceptable
```

La cerca per bucle que vas fer ahir és O(n) — en el pitjor cas recorre tots els elements. L'accés per clau al HashMap és O(1) — temps constant independentment de quants jocs hi hagi. Amb 100.000 elements, la diferència entre O(n) i O(1) pot ser de 1ms vs 50ms. Amb 10 milions, de 1ms vs 5 segons.

### Com Funciona un HashMap Per Dins

Ahir vas veure que `map.get("APP-7")` troba l'objecte sense recórrer res. Com ho fa?

Un HashMap internament és un array de "caselles" (buckets). Quan guardes un objecte amb una clau, Java:

1. **Calcula un número a partir de la clau** — això es diu *hash*. Per exemple, `"APP-7"` → `299847` (un número gran).
2. **Converteix el número en una posició** — fa el residu: `299847 % nombre_de_caselles`. Si hi ha 200.000 caselles, `299847 % 200000 = 99847`.
3. **Guarda l'objecte a la casella 99847.**

Quan fas `get("APP-7")`, repeteix el càlcul: hash → posició → accedeix directament a la casella. Sense recórrer res.

```
HashMap amb 4 caselles (simplificat):

Guardar "APP-1" → hash("APP-1") % 4 = 0 → casella [0]
Guardar "APP-2" → hash("APP-2") % 4 = 1 → casella [1]
Guardar "APP-7" → hash("APP-7") % 4 = 3 → casella [3]

┌───────────────────────┐
│ [0] → APP-1: LoL      │
│ [1] → APP-2: Dota2    │
│ [2] → (buida)         │
│ [3] → APP-7: Valorant │
└───────────────────────┘

Buscar "APP-7":
  1. hash("APP-7") % 4 = 3
  2. Accedeix casella [3] → Trobat!
```

**I si dues claus cauen a la mateixa casella?** Això es diu *col·lisió*. Java encadena els objectes en aquella casella i els recorre fins trobar la clau exacta. Per això el cas pitjor teòric de HashMap és O(n) — però en pràctica, Java's HashMap distribueix bé les claus i manté un promig de O(1).

**El preu:** un HashMap usa més memòria que una llista, perquè necessita l'array de caselles (moltes buides). Amb 100.000 jocs, típicament un HashMap usa ~2x la memòria d'un ArrayList. **2x memòria per 50x velocitat és un bon intercanvi.**

> **Lectura recomanada (opcional, no bloquejant):**
> - NeetCode — [Hash Maps](https://neetcode.io/courses/dsa-for-beginners/0) i [Big-O Notation](https://neetcode.io/courses/dsa-for-beginners/0)
> - Baeldung — [Guide to Java HashMap](https://www.baeldung.com/java-hashmap)
> - Vídeo — [Estructuras y Algoritmos de Manera Visual](https://www.youtube.com/watch?v=2LZanU8UC_A) — MoureDev (YouTube)

---

## Activitat

### 1. Generador de dades massives — `GameDataGenerator` (30 min)

Crea el fitxer:
```
backend-java/src/main/java/com/esportspulse/engine/data/GameDataGenerator.java
```

Programa dos mètodes estàtics:

1. **`generateList(int size)`** — Retorna un `ArrayList<GameRecord>` amb `size` jocs sintètics. Cada joc té un appId predictible (`"APP-1"`, `"APP-2"`, ..., `"APP-100000"`), un títol inventat (pot ser el mateix per tots, no importa), preu aleatori i jugadors aleatoris.

2. **`toMap(List<GameRecord> games)`** — Rep la llista i retorna un `HashMap<String, GameRecord>` usant `appId` com a clau.

Exemple d'ús:
```java
List<GameRecord> list = GameDataGenerator.generateList(100_000);
Map<String, GameRecord> map = GameDataGenerator.toMap(list);
```

### 2. Servei de cerca — `GameSearchService` (30 min)

Crea el fitxer:
```
backend-java/src/main/java/com/esportspulse/engine/search/GameSearchService.java
```

Programa dos mètodes:

1. **`searchLinear(List<GameRecord> games, String appId)`** — Recorre la llista element per element amb un bucle. Retorna el `GameRecord` si el troba, `null` si no.

2. **`searchByKey(Map<String, GameRecord> games, String appId)`** — Usa `map.get(appId)` directament.

### 3. Verificació ràpida (15 min)

Crea un petit `main` temporal (o afegeix-lo a `CollectionExplorer`) que:
1. Generi 100.000 jocs
2. Busqui `"APP-100000"` (el pitjor cas per cerca lineal — l'últim element)
3. Imprimeixi el resultat d'ambdós mètodes per verificar que troben el mateix objecte

No mesuris temps encara — demà ho faràs amb rigor. Avui l'objectiu és que el codi funcioni.

### 4. Commit (5 min)

```bash
git add backend-java/src/main/java/com/esportspulse/engine/data/GameDataGenerator.java
git add backend-java/src/main/java/com/esportspulse/engine/search/GameSearchService.java
git commit -m "feat(java): add data generator and search service (linear vs hash)"
```

---

## Checklist de Lliurament

- [ ] `GameDataGenerator.generateList(100_000)` retorna 100.000 jocs amb IDs predictibles
- [ ] `GameDataGenerator.toMap(list)` converteix la llista a HashMap correctament
- [ ] `GameSearchService.searchLinear()` troba l'element recorrent la llista
- [ ] `GameSearchService.searchByKey()` troba l'element per clau de mapa
- [ ] Ambdós mètodes retornen el mateix objecte per al mateix appId
- [ ] Commit fet a `feature/week1-benchmarking`
