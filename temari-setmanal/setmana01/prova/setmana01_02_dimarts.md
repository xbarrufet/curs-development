# Setmana 1 — Dimarts: Modelat de Domini i Col·leccions Bàsiques

## Objectiu del Dia

Crear la primera entitat de domini del projecte (`GameRecord`) i entendre la diferència entre `ArrayList` i `HashMap` manipulant dades a petita escala. Al final del dia tens una classe Java funcional al lloc correcte del projecte i un `main` que la utilitza.

---

## Teoria

### Immutabilitat: Objectes que No Canvien

Un objecte **mutable** es pot modificar després de crear-lo. Un objecte **immutable** no — un cop creat, els seus valors són definitius. Si necessites valors diferents, crees un objecte nou.

```java
// MUTABLE — pots canviar l'estat després de crear-lo
class GameMutable {
    private String title;
    private double price;

    public void setPrice(double price) { this.price = price; }
}

GameMutable game = new GameMutable("LoL", 0);
game.setPrice(9.99);  // Canvies l'objecte original!
```

**Quin problema té això?**
- **Efectes laterals inesperats:** Imagina que passes `game` a tres mètodes diferents. Si un d'ells crida `setPrice()`, els altres dos veuen el canvi sense esperar-ho. Això causa bugs difícils de rastrejar, especialment si el codi s'executa en paral·lel (concurrència, que veuràs a la setmana 3).
- **Tests fràgils:** Per testejar un mètode que rep un objecte mutable, has de preparar l'estat exacte de l'objecte abans de cridar-lo i verificar que no t'ha canviat camps que no tocava. Si el test anterior ha modificat el mateix objecte, el teu test pot fallar o passar per motius incorrectes — l'ordre d'execució dels tests afecta el resultat. Amb objectes immutables, cada test crea els seus propis objectes i cap test pot contaminar un altre.

```java
// IMMUTABLE — no pots canviar res
record GameRecord(String appId, String title, double price, long activePlayers) {}

GameRecord game = new GameRecord("APP-1", "LoL", 0, 5_000_000);
// game.setPrice(9.99);  ← NO EXISTEIX. No hi ha setters.
// Si vols un preu diferent, crees un objecte nou:
GameRecord updated = new GameRecord("APP-1", "LoL", 9.99, 5_000_000);
```

**Per què és bo:**
- **Predictible:** Si passes un `GameRecord` a un mètode, saps que no te'l canviarà. El que crees és el que tens, sempre.
- **Segur en concurrència:** Si dos threads llegeixen el mateix objecte, cap dels dos el pot modificar → zero conflictes.
- **Funciona bé amb col·leccions:** Demà veuràs que `HashMap` utilitza una tècnica interna per localitzar objectes ràpidament. Aquesta tècnica depèn de que els camps de l'objecte no canviïn — si canvien, el mapa "perd" l'objecte. Amb immutables això no pot passar perquè els valors són fixos des de la creació.

### Records de Java 21: Immutabilitat Sense Boilerplate

Per crear una classe immutable "a mà" necessites: camps `final`, constructor, getters, `equals()`, `hashCode()`, `toString()` — unes 40 línies. Java 21 ho resol amb `record`:

```java
// 1 línia — Java genera constructor, getters, equals, hashCode i toString
public record GameRecord(String appId, String title, double price, long activePlayers) {}
```

Accedir als camps: `game.appId()`, `game.title()`, etc. (sense `get` — és un record, no un bean).

A EsportsPulse, les dades de champions i partides són lectures — les consultes, no les modifiques. Per tant, `record` és l'elecció natural.

### ArrayList vs HashMap: Primera Intuïció

Imagina que tens 20 jocs i vols trobar el que té `appId = "APP-15"`.

**Amb `ArrayList`** — recorres un per un fins trobar-lo:
```java
List<GameRecord> games = new ArrayList<>();
// ... 20 jocs afegits

for (GameRecord g : games) {
    if (g.appId().equals("APP-15")) {
        return g;  // Potser al primer, potser al joc 20
    }
}
```
Amb 20 jocs no es nota. Amb 100.000 sí.

**Amb `HashMap`** — accés directe per clau:
```java
Map<String, GameRecord> gamesMap = new HashMap<>();
// ... 20 jocs afegits amb appId com a clau

GameRecord g = gamesMap.get("APP-15");  // Instantani, sempre
```

Demà escalarem a 100.000 elements i mesurarem la diferència real. Avui l'objectiu és familiaritzar-te amb les dues estructures.

> **Lectura recomanada (opcional, no bloquejant):**
> - NeetCode — [Arrays & Dynamic Arrays](https://neetcode.io/courses/dsa-for-beginners/0)
> - Baeldung — [Java 21 Record Keyword](https://www.baeldung.com/java-record-keyword)
> - Oracle — [Collections Framework (List & Map)](https://docs.oracle.com/javase/tutorial/collections/interfaces/index.html)

---

## Activitat

### 1. Crear l'entitat `GameRecord` (15 min)

Crea el fitxer al lloc correcte dins el projecte Maven:

```
backend-java/src/main/java/com/esportspulse/engine/model/GameRecord.java
```

```java
package com.esportspulse.engine.model;

public record GameRecord(
    String appId,
    String title,
    double price,
    long activePlayers
) {}
```

Verifica que compila: `mvn compile`

### 2. Explorar ArrayList i HashMap a petita escala (45 min)

Crea una classe executable per experimentar:

```
backend-java/src/main/java/com/esportspulse/engine/CollectionExplorer.java
```

Programa el següent:

1. **Crea una `ArrayList` amb 10-15 jocs** inventats (noms de jocs reals o ficticis, preus, jugadors).
2. **Crea un `HashMap`** amb els mateixos jocs, usant `appId` com a clau.
3. **Imprimeix per consola** respostes a aquestes preguntes:
   - Quin joc hi ha a la posició 5 de la llista? (`list.get(5)`)
   - Quin joc té l'appId `"APP-7"`? (`map.get("APP-7")`)
   - La llista conté un joc amb appId `"APP-99"`? (recorre amb `for-each` i comprova)
   - El mapa conté la clau `"APP-99"`? (`map.containsKey(...)`)
4. **Observa la diferència:** Per buscar per appId, la llista requereix un bucle. El mapa, una sola crida.

Executa amb: `mvn exec:java -Dexec.mainClass="com.esportspulse.engine.CollectionExplorer"` (o directament des de Cursor amb Run).

### 3. Commit (5 min)

```bash
git add backend-java/src/main/java/com/esportspulse/engine/model/GameRecord.java
git add backend-java/src/main/java/com/esportspulse/engine/CollectionExplorer.java
git commit -m "feat(java): add GameRecord domain entity and collection exploration"
```

---

## Checklist de Lliurament

- [ ] `GameRecord.java` compila correctament dins `com.esportspulse.engine.model`
- [ ] `CollectionExplorer` s'executa i imprimeix resultats per consola
- [ ] Entens la diferència entre accés per índex (List) i accés per clau (Map)
- [ ] Commit fet a `feature/week1-benchmarking`
