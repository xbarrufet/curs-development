# Setmana 1 — Dimecres: Big-O, Hash Internals i Cerca a Escala

## Objectiu del Dia

Entendre per què amb 20 jugadors no es nota la diferència entre `ArrayList` i `HashMap`, però amb 100.000 sí. Construir un generador de dades massives i un servei de cerca amb els dos enfocaments. Al final del dia tens `PlayerDataGenerator` i `PlayerSearchService` funcionant amb 100.000 elements.

---

## Teoria

### Per Què Importa l'Ordre de Magnitud

Ahir vas veure que buscar per `playerId` en una llista requereix un bucle, i en un mapa una sola crida. Amb 15 jugadors, les dues opcions triguen menys d'un mil·lisegon — no hi ha diferència perceptible. Quan desenvolupes, normalment proves amb pocs elements: 5 jugadors, 10 registres, 20 files. Tot funciona ràpid i sembla correcte.

**El problema arriba a producció.** Quan el teu codi serveix usuaris reals, les dades ja no són 15 jugadors de prova — són milers, desenes de milers o milions de registres. Una cerca que amb 15 elements triga 0.001ms, amb 500.000 pot trigar mig segon. Si es crida dins un bucle o per cada petició d'un usuari, el sistema es torna inutilitzable. Per això necessites una eina per predir com es comportarà el teu codi **abans** que arribi a producció, sense haver de provar amb dades reals cada vegada.

Aquesta eina és la notació **Big-O**.

### Notació Big-O: Com Escala el Teu Codi

Big-O descriu **com escala el temps d'execució quan creix el volum de dades (n)**. No mesura mil·lisegons concrets — mesura la tendència.

| Notació | Nom | Què fa | Exemple |
|---------|-----|--------|---------|
| **O(1)** | Constant | Sempre el mateix temps, doni igual quants elements hi hagi | `map.get("P-50000")` |
| **O(log n)** | Logarítmic | Creix molt lentament | Cerca binària en llista ordenada |
| **O(n)** | Lineal | Creix proporcionalment amb les dades | Recórrer una llista sencera |
| **O(n²)** | Quadràtic | Creix molt ràpidament | Bucle dins de bucle |

**Amb 100.000 jugadors (n = 100.000):**

```
O(1):       1 operació           →  instantani
O(log n):   ~17 operacions       →  instantani
O(n):       100.000 operacions   →  notable
O(n²):      10.000.000.000 ops   →  inacceptable
```

La cerca per bucle que vas fer ahir és O(n) — en el pitjor cas recorre tots els elements. L'accés per clau al HashMap és O(1) — temps constant independentment de quants jugadors hi hagi. Amb 100.000 elements, la diferència entre O(n) i O(1) pot ser de 1ms vs 50ms. Amb 10 milions, de 1ms vs 5 segons.

### Com Funciona un HashMap Per Dins

Ahir vas veure que `map.get("P-7")` troba l'objecte sense recórrer res. Com ho fa?

Un HashMap internament és un array de "caselles" (buckets). Quan guardes un objecte amb una clau, Java:

1. **Calcula un número a partir de la clau** — això es diu *hash*. Per exemple, `"P-7"` → `299847` (un número gran).
2. **Converteix el número en una posició** — fa el residu: `299847 % nombre_de_caselles`. Si hi ha 200.000 caselles, `299847 % 200000 = 99847`.
3. **Guarda l'objecte a la casella 99847.**

Quan fas `get("P-7")`, repeteix el càlcul: hash → posició → accedeix directament a la casella. Sense recórrer res.

```
HashMap amb 4 caselles (simplificat):

Guardar "P-1" → hash("P-1") % 4 = 0 → casella [0]
Guardar "P-2" → hash("P-2") % 4 = 1 → casella [1]
Guardar "P-7" → hash("P-7") % 4 = 3 → casella [3]

┌─────────────────────────────┐
│ [0] → P-1: Faker            │
│ [1] → P-2: Caps             │
│ [2] → (buida)               │
│ [3] → P-7: Rekkles          │
└─────────────────────────────┘

Buscar "P-7":
  1. hash("P-7") % 4 = 3
  2. Accedeix casella [3] → Trobat!
```

**I si dues claus cauen a la mateixa casella?** Això es diu *col·lisió*. Java encadena els objectes en aquella casella i els recorre fins trobar la clau exacta. Per això el cas pitjor teòric de HashMap és O(n) — però en pràctica, Java's HashMap distribueix bé les claus i manté un promig de O(1).

**El preu:** un HashMap usa més memòria que una llista, perquè necessita l'array de caselles (moltes buides). Amb 100.000 jugadors, típicament un HashMap usa ~2x la memòria d'un ArrayList. **2x memòria per 50x velocitat és un bon intercanvi.**

> **Lectura recomanada (opcional, no bloquejant):**
> - NeetCode — [Hash Maps](https://neetcode.io/courses/dsa-for-beginners/0) i [Big-O Notation](https://neetcode.io/courses/dsa-for-beginners/0)
> - Baeldung — [Guide to Java HashMap](https://www.baeldung.com/java-hashmap)
> - Vídeo — [Estructuras y Algoritmos de Manera Visual](https://www.youtube.com/watch?v=2LZanU8UC_A) — MoureDev (YouTube)

---

## Activitat

### 1. Generador de dades massives — `PlayerDataGenerator` (30 min)

Crea el fitxer:
```
backend-java/src/main/java/com/esportspulse/engine/data/PlayerDataGenerator.java
```

Programa dos mètodes estàtics:

> **Per què "estàtics"?** Un mètode `static` pertany a la classe, no a un objecte concret. El crides directament amb `PlayerDataGenerator.generateList(100_000)` — sense necessitar un `new PlayerDataGenerator()` previ. S'usa quan el mètode és una operació d'utilitat que rep un input i retorna un output, sense dependre d'estat interna de l'objecte. Exemples coneguts: `Math.max()`, `Integer.parseInt()`. En canvi, un mètode no estàtic (d'instància) pertany a un objecte i pot accedir als seus camps — s'usa quan l'objecte manté configuració o estat que afecta el comportament. Aquí, `generateList` i `toMap` són funcions pures: entra input, surt output, res més. Per tant, `static` és l'elecció correcta.
> Si vols aprofundir: [Programiz — Java Static Keyword](https://www.programiz.com/java-programming/static-keyword) — exemples clars comparant mètodes estàtics vs d'instància.

> **Què són dades sintètiques?** Són dades inventades pel teu codi, no reals. Encara no tenim connexió a cap API ni base de dades, però necessitem 100.000 jugadors per provar el rendiment. La solució és generar-los automàticament amb valors ficticis: IDs seqüencials, noms inventats, level i hoursPlayed aleatoris. No importen els valors concrets — el que importa és el volum. Això és una pràctica habitual en enginyeria: generar dades sintètiques per testejar com es comporta el codi a escala abans de connectar-lo a dades reals. En el cas d'EsportsPulse, 100.000 jugadors és molt realista — League of Legends té milions de jugadors actius.

1. **`generateList(int size)`** — Retorna un `ArrayList<PlayerRecord>` amb `size` jugadors sintètics. Cada jugador té un playerId seqüencial (`"P-1"`, `"P-2"`, ..., `"P-100000"`), un nom d'usuari inventat (pot ser "Player-1", "Player-2", etc.), level aleatori entre 1 i 500, i hoursPlayed aleatori entre 0 i 5000.

2. **`toMap(List<PlayerRecord> players)`** — Rep la llista i retorna un `HashMap<String, PlayerRecord>` usant `playerId` com a clau.

Exemple d'ús:
```java
List<PlayerRecord> list = PlayerDataGenerator.generateList(100_000);
Map<String, PlayerRecord> map = PlayerDataGenerator.toMap(list);
```

### 2. Servei de cerca — `PlayerSearchService` (30 min)

Crea el fitxer:
```
backend-java/src/main/java/com/esportspulse/engine/search/PlayerSearchService.java
```

Programa dos mètodes:

1. **`searchLinear(List<PlayerRecord> players, String playerId)`** — Recorre la llista element per element amb un bucle. Retorna el `PlayerRecord` si el troba, `null` si no.

2. **`searchByKey(Map<String, PlayerRecord> players, String playerId)`** — Usa `map.get(playerId)` directament.

### 3. Verificació ràpida (15 min)

Crea un petit `main` temporal (o afegeix-lo a `PlayerExplorer`) que:
1. Generi 100.000 jugadors
2. Busqui `"P-100000"` (el pitjor cas per cerca lineal — l'últim element)
3. Imprimeixi el resultat d'ambdós mètodes per verificar que troben el mateix objecte

No mesuris temps encara — demà ho faràs amb rigor. Avui l'objectiu és que el codi funcioni.

### 4. Commit (5 min)

```bash
git add backend-java/src/main/java/com/esportspulse/engine/data/PlayerDataGenerator.java
git add backend-java/src/main/java/com/esportspulse/engine/search/PlayerSearchService.java
git commit -m "feat(java): add player data generator and search service (linear vs hash)"
```

---

## Checklist de Lliurament

- [ ] `PlayerDataGenerator.generateList(100_000)` retorna 100.000 jugadors amb IDs predictibles
- [ ] `PlayerDataGenerator.toMap(list)` converteix la llista a HashMap correctament
- [ ] `PlayerSearchService.searchLinear()` troba l'element recorrent la llista
- [ ] `PlayerSearchService.searchByKey()` troba l'element per clau de mapa
- [ ] Ambdós mètodes retornen el mateix objecte per al mateix playerId
- [ ] Commit fet a `feature/week1-benchmarking`
