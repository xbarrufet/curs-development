# Setmana 1 — Dimarts: Modelat de Domini i Col·leccions Bàsiques

## Objectiu del Dia

Crear la primera **entitat de domini** del projecte (`PlayerRecord`) i entendre la diferència entre `ArrayList` i `HashMap` manipulant dades a petita escala. Al final del dia tens una classe Java funcional al lloc correcte del projecte i un `main` que la utilitza.

> **Què vol dir "domini"?** En enginyeria de software, el **domini** és el problema del món real que el teu software resol. Si programes una app de banca, el domini és "diners, comptes, transferències". Si programes EsportsPulse, el domini és "jugadors, campions, partides, estadístiques". Les **entitats de domini** són les classes que representen aquests conceptes — no són classes tècniques (com un `DatabaseConnection` o un `HttpClient`), sinó classes que un expert del negoci reconeixeria: `PlayerRecord`, `ChampionRecord`, `MatchRecord`. Quan diem "modelar el domini", volem dir decidir quines dades té cada entitat i com es relacionen entre elles. Avui comencem amb la més senzilla: un jugador (`PlayerRecord`), perquè és l'entitat més directa i generar 100.000 jugadors és totalment realista — League of Legends té milions de jugadors actius. `ChampionRecord` apareixerà a la setmana 2 quan connectem amb l'API de Riot, i `MatchRecord` quan afegim cerca de partides.

---

## Teoria

### Immutabilitat: Objectes que No Canvien

Un objecte **mutable** es pot modificar després de crear-lo. Un objecte **immutable** no — un cop creat, els seus valors són definitius. Si necessites valors diferents, crees un objecte nou.

```java
// MUTABLE — pots canviar l'estat després de crear-lo
class PlayerMutable {
    private String username;
    private int level;

    public void setLevel(int level) { this.level = level; }
}

PlayerMutable player = new PlayerMutable("Faker", 500);
player.setLevel(1);  // Canvies l'objecte original!
```

**Quin problema té això?**
- **Efectes laterals inesperats:** Imagina que passes `player` a tres mètodes diferents. Si un d'ells crida `setLevel()`, els altres dos veuen el canvi sense esperar-ho. Això causa bugs difícils de rastrejar, especialment si el codi s'executa en paral·lel (concurrència, que veuràs a la setmana 3).
- **Tests fràgils:** Per testejar un mètode que rep un objecte mutable, has de preparar l'estat exacte de l'objecte abans de cridar-lo i verificar que no t'ha canviat camps que no tocava. Si el test anterior ha modificat el mateix objecte, el teu test pot fallar o passar per motius incorrectes — l'ordre d'execució dels tests afecta el resultat. Amb objectes immutables, cada test crea els seus propis objectes i cap test pot contaminar un altre.

```java
// IMMUTABLE — no pots canviar res
record PlayerRecord(String playerId, String username, int level, double hoursPlayed) {}

PlayerRecord player = new PlayerRecord("P-1", "Faker", 500, 3500.0);
// player.setLevel(1);  ← NO EXISTEIX. No hi ha setters.
// Si vols un level diferent, crees un objecte nou:
PlayerRecord updated = new PlayerRecord("P-1", "Faker", 300, 3500.0);
```

**Per què és bo:**
- **Predictible:** Si passes un `PlayerRecord` a un mètode, saps que no te'l canviarà. El que crees és el que tens, sempre.
- **Segur en concurrència:** Si dos threads llegeixen el mateix objecte, cap dels dos el pot modificar → zero conflictes.
- **Funciona bé amb col·leccions:** Demà veuràs que `HashMap` utilitza una tècnica interna per localitzar objectes ràpidament. Aquesta tècnica depèn de que els camps de l'objecte no canviïn — si canvien, el mapa "perd" l'objecte. Amb immutables això no pot passar perquè els valors són fixos des de la creació.

### Records de Java 21: Immutabilitat Sense Boilerplate

Per crear una classe immutable "a mà" necessites: camps `final`, constructor, getters, `equals()`, `hashCode()`, `toString()` — unes 40 línies. Java 21 ho resol amb `record`:

```java
// 1 línia — Java genera constructor, getters, equals, hashCode i toString
public record PlayerRecord(String playerId, String username, int level, double hoursPlayed) {}
```

Accedir als camps: `player.playerId()`, `player.username()`, etc. (sense `get` — és un record, no un bean).

A EsportsPulse, les dades de jugadors i partides són lectures — les consultes, no les modifiques. Per tant, `record` és l'elecció natural.

### ArrayList vs HashMap: Primera Intuïció

Imagina que tens 20 jugadors i vols trobar el que té `playerId = "P-15"`.

**Amb `ArrayList`** — recorres un per un fins trobar-lo:
```java
List<PlayerRecord> players = new ArrayList<>();
// ... 20 jugadors afegits

for (PlayerRecord p : players) {
    if (p.playerId().equals("P-15")) {
        return p;  // Potser al primer, potser al jugador 20
    }
}
```
Amb 20 jugadors no es nota. Amb 100.000 sí.

**Amb `HashMap`** — accés directe per clau:
```java
Map<String, PlayerRecord> playersMap = new HashMap<>();
// ... 20 jugadors afegits amb playerId com a clau

PlayerRecord p = playersMap.get("P-15");  // Instantani, sempre
```

Demà escalarem a 100.000 elements i mesurarem la diferència real. Avui l'objectiu és familiaritzar-te amb les dues estructures.

> **Lectura recomanada (opcional, no bloquejant):**
> - NeetCode — [Arrays & Dynamic Arrays](https://neetcode.io/courses/dsa-for-beginners/0)
> - Baeldung — [Java 21 Record Keyword](https://www.baeldung.com/java-record-keyword)
> - Oracle — [Collections Framework (List & Map)](https://docs.oracle.com/javase/tutorial/collections/interfaces/index.html)

---

## Activitat

### 1. Crear l'entitat `PlayerRecord` (15 min)

Crea el fitxer dins el paquet `model` del projecte:

```
backend-java/src/main/java/com/esportspulse/engine/model/PlayerRecord.java
```

**Per què dins `model/`?** En una aplicació Java ben estructurada, les entitats de domini viuen separades del codi tècnic. El paquet `model` (o `domain`) és on poses les classes que representen conceptes del negoci — dades pures, sense lògica de bases de dades ni de HTTP. Així, quan el projecte creixi, qualsevol desenvolupador sap que a `model/` hi trobarà les entitats i a altres paquets (`service/`, `controller/`, `repository/`) la lògica que les utilitza.

> **I la resta d'entitats?** A la teoria hem esmentat `ChampionRecord` i `MatchRecord`. Avui només creem `PlayerRecord` perquè és la més senzilla i ens permet centrar-nos en les col·leccions. `ChampionRecord` apareixerà a la setmana 2 quan connectem amb l'API de Riot per obtenir dades de campions, i `MatchRecord` quan afegim cerca de partides. No cal crear-les ara — cada entitat es crea quan té un ús concret, no per endavant.

`PlayerRecord` representa un **jugador/invocador de League of Legends** dins el sistema EsportsPulse. Pensa en un jugador com Faker, Caps o Rekkles: té un identificador, un nom d'usuari, un nivell i les hores jugades. Aquests són els camps mínims que necessitem per treballar amb col·leccions avui i amb benchmarks més endavant.

```java
package com.esportspulse.engine.model;

public record PlayerRecord(
    String playerId,      // Identificador únic del jugador (ex: "P-1", "P-5432")
    String username,      // Nom d'usuari del jugador (ex: "Faker", "Caps", "Rekkles")
    int level,            // Nivell del jugador (ex: 1, 30, 500)
    double hoursPlayed    // Hores jugades (ex: 0.0, 1500.5, 3500.0)
) {}
```

Verifica que compila: `mvn compile`

### 2. Explorar ArrayList i HashMap a petita escala (45 min)

Crea una classe executable per experimentar:

```
backend-java/src/main/java/com/esportspulse/engine/PlayerExplorer.java
```

**Com escriure un missatge a pantalla en Java?** Utilitza `System.out.println()` — imprimeix el text que li passis i salta a la línia següent. És l'eina bàsica per veure què fa el teu codi:

```java
System.out.println("Hola, món!");                          // Text fix
System.out.println("El jugador és: " + player.username()); // Text + valor d'un objecte
System.out.println(player);                                // Imprimeix l'objecte sencer (els records generen un toString() automàtic)
```

Si vols imprimir sense salt de línia, usa `System.out.print()` (sense `ln`). Durant tot el curs faràs servir `println` constantment per verificar que el codi fa el que esperes.

Programa el següent:

1. **Crea una `ArrayList` amb 10-15 jugadors** inventats (noms de jugadors de LoL reals o ficticis, level, hoursPlayed).
2. **Crea un `HashMap`** amb els mateixos jugadors, usant `playerId` com a clau.
3. **Imprimeix per consola** respostes a aquestes preguntes:
   - Quin jugador hi ha a la posició 5 de la llista? (`list.get(5)`)
   - Quin jugador té el playerId `"P-7"`? (`map.get("P-7")`)
   - La llista conté un jugador amb playerId `"P-99"`? (recorre amb `for-each` i comprova)
   - El mapa conté la clau `"P-99"`? (`map.containsKey(...)`)
4. **Observa la diferència:** Per buscar per playerId, la llista requereix un bucle. El mapa, una sola crida.

> **Amb 15 jugadors, ambdues cerques són instantànies** — no notaràs cap diferència de velocitat. Dijous escalarem a 100.000 elements i mesurarem quant triga cadascuna amb `System.nanoTime()`. La diferència et sorprendrà.

Executa amb: `mvn exec:java -Dexec.mainClass="com.esportspulse.engine.PlayerExplorer"` (o directament des de Cursor amb Run).

### 3. Commit (5 min)

```bash
git add backend-java/src/main/java/com/esportspulse/engine/model/PlayerRecord.java
git add backend-java/src/main/java/com/esportspulse/engine/PlayerExplorer.java
git commit -m "feat(java): add PlayerRecord domain entity and collection exploration"
```

---

## Checklist de Lliurament

- [ ] `PlayerRecord.java` compila correctament dins `com.esportspulse.engine.model`
- [ ] `PlayerExplorer` s'executa i imprimeix resultats per consola
- [ ] Entens la diferència entre accés per índex (List) i accés per clau (Map)
- [ ] Commit fet a `feature/week1-benchmarking`
