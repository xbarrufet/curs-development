# Setmana 2 — Dilluns: ChampionRecord amb Validació i Mètodes de Negoci

## Objectiu del Dia

Crear la segona entitat de domini del projecte (`ChampionRecord`) amb validació al constructor i mètodes de negoci. Entendre per què un objecte ha de ser **sempre vàlid** des del moment en què es crea, i com afegir lògica de negoci a un record sense trencar la immutabilitat. Al final del dia tens `ChampionRecord` creat, validat amb compact constructor, i 3 instàncies provades a `main`.

---

## Teoria

### Objectes Sempre Vàlids: Validar a la Creació

A la setmana 1 vas crear `PlayerRecord` com un record senzill — qualsevol combinació de valors era acceptada. Però en un sistema real, no tot és vàlid: un campió no pot tenir un winRate de -50%, ni un championId buit.

La pregunta és: **on valides?**

```java
// OPCIÓ 1: Validar després de crear — MALAMENT
ChampionRecord champ = new ChampionRecord("", "Ahri", -50.0, 200.0);  // Objecte invàlid existeix!
if (!isValid(champ)) { ... }  // Potser algú s'oblida de cridar isValid()
```

```java
// OPCIÓ 2: Validar al constructor — BÉ
ChampionRecord champ = new ChampionRecord("", "Ahri", -50.0, 200.0);
// ↑ PETA amb IllegalArgumentException — l'objecte invàlid MAI arriba a existir
```

**El principi:** si un `ChampionRecord` existeix al sistema, és garantidament vàlid. No cal mai comprovar-ho després. Això elimina una categoria sencera de bugs — cap mètode rep mai un objecte amb dades impossibles.

Java 21 ofereix el **compact constructor** per fer aquesta validació de forma neta, com veuràs a la secció de més avall.

### Lògica de Negoci: Dins l'Entitat o Fora?

Un record pot tenir mètodes. La pregunta és **quins** mètodes hi pertanyen:

- **Dins el record** → mètodes que depenen **exclusivament** dels camps de l'objecte. No necessiten res extern (ni bases de dades, ni APIs, ni altres serveis). Exemples: `isMeta()` (mira winRate i pickRate), `withPatchAdjustment()` (calcula un nou winRate).

- **Fora del record** (en un servei) → operacions que necessiten **coses externes**: guardar a la BD, enviar notificacions, cridar APIs, orquestrar múltiples objectes. Exemple: `registerChampion()` necessita un repositori per guardar.

**Regla pràctica:** si el mètode necessita un `import` que no sigui del propi paquet `model`, probablement pertany a un servei, no al record.

### Patró "With": Modificar Sense Mutar

A la setmana 1 vas aprendre que els records són immutables — no tenen setters. Llavors, com "modifiques" un campió? Retornant un **objecte nou**:

```java
// Imagina que Riot nerfeja Ahri en un patch: li baixa el winRate 2.5 punts
ChampionRecord ahri = new ChampionRecord("CHAMP-1", "Ahri", 52.3, 8.1);
ChampionRecord ahriNerfed = ahri.withPatchAdjustment(-2.5);

System.out.println(ahri.winRate());        // 52.3 — l'original NO canvia
System.out.println(ahriNerfed.winRate());  // 49.8 — el nou objecte té el valor ajustat
```

El nom `with*()` és una convenció: indica que retorna un nou objecte amb un canvi aplicat. En comptes de `setPrice()` (que muta), tens `withPrice()` (que crea). Trobaràs aquest patró a tot Java modern: `LocalDate.withMonth()`, `String.replace()`, etc.

> **Lectura recomanada (opcional, no bloquejant):**
> - Baeldung — [Java 21 Record Keyword](https://www.baeldung.com/java-record-keyword)
> - Baeldung — [Java Record with Custom Constructor](https://www.baeldung.com/java-record-custom-constructor)

### Compact Constructor: Validacio a la Creacio

```java
public record ChampionRecord(
    String championId,
    String name,
    double winRate,
    double pickRate
) {
    // Compact constructor — s'executa automaticament quan es crea el record
    // No cal "this.championId = championId" — Java ho fa sol amb records
    public ChampionRecord {
        // Validacio: si les dades son invalides, llança excepcio
        // Aixi MAI pot existir un ChampionRecord invalid al sistema
        if (championId == null || championId.isBlank()) {
            throw new IllegalArgumentException("championId no pot ser buit");
        }
        if (winRate < 0 || winRate > 100) {
            throw new IllegalArgumentException("winRate ha de ser entre 0 i 100");
        }
        if (pickRate < 0 || pickRate > 100) {
            throw new IllegalArgumentException("pickRate ha de ser entre 0 i 100");
        }
    }
}
```

### Metodes de Negoci al Record

```java
public record ChampionRecord(
    String championId,
    String name,
    double winRate,
    double pickRate
) {
    // Compact constructor (validacio)
    public ChampionRecord {
        if (championId == null || championId.isBlank()) {
            throw new IllegalArgumentException("championId no pot ser buit");
        }
        if (winRate < 0 || winRate > 100) {
            throw new IllegalArgumentException("winRate ha de ser entre 0 i 100");
        }
        if (pickRate < 0 || pickRate > 100) {
            throw new IllegalArgumentException("pickRate ha de ser entre 0 i 100");
        }
    }

    // Metode de negoci: un campió es "meta" si te winRate > 52 i pickRate > 10
    public boolean isMeta() {
        return winRate > 52.0 && pickRate > 10.0;
    }

    // Metode que "modifica" sense mutar — retorna un NOU record amb el winRate ajustat
    // L'original no canvia MAI
    public ChampionRecord withPatchAdjustment(double modifier) {
        // Calcula el nou winRate sumant el modifier (pot ser negatiu per nerfs)
        double adjustedWinRate = this.winRate + modifier;
        // Crea i retorna un record NOU — l'original segueix intacte
        return new ChampionRecord(championId, name, adjustedWinRate, pickRate);
    }
}
```

---

## Activitat

### 1. Crear `ChampionRecord` amb validacio (30 min)

Crea el fitxer:
```
backend-java/src/main/java/com/esportspulse/engine/model/ChampionRecord.java
```

Implementa el `record` amb:
- Camps: `championId` (String), `name` (String), `winRate` (double), `pickRate` (double)
- Compact constructor amb validacio (championId no null/buit, winRate entre 0 i 100, pickRate entre 0 i 100)
- Metode `isMeta()` que retorna `true` si `winRate > 52.0 && pickRate > 10.0`
- Metode `withPatchAdjustment(double modifier)` que retorna un NOU record amb el winRate ajustat

### 2. Provar a `main` (20 min)

Crea 3 instancies de `ChampionRecord` a un `main` temporal:

```java
public class ChampionRecordDemo {
    public static void main(String[] args) {
        // Campió meta (winRate > 52 i pickRate > 10)
        ChampionRecord ahri = new ChampionRecord("CHAMP-1", "Ahri", 52.3, 8.1);

        // Campió no meta
        ChampionRecord yasuo = new ChampionRecord("CHAMP-2", "Yasuo", 49.1, 12.5);

        // Campió amb ajust de patch
        ChampionRecord ahriNerfed = ahri.withPatchAdjustment(-2.5);

        // Verifica que isMeta funciona
        System.out.println(ahri.isMeta());     // false — winRate > 52 pero pickRate < 10
        System.out.println(yasuo.isMeta());    // false — winRate < 52

        // Verifica que withPatchAdjustment NO muta l'original
        System.out.println(ahri.winRate());           // 52.3 — no ha canviat
        System.out.println(ahriNerfed.winRate());     // 49.8 — reduit 2.5 punts

        // Prova amb un campió clarament meta
        ChampionRecord broken = new ChampionRecord("CHAMP-3", "Broken Champ",
            55.0, 15.0);
        ChampionRecord brokenBuffed = broken.withPatchAdjustment(1.5);
        System.out.println(broken.winRate());          // 55.0
        System.out.println(brokenBuffed.winRate());    // 56.5

        // Prova que la validacio funciona (ha de petar)
        try {
            new ChampionRecord(null, "Bad Champ", 50.0, 5.0);
        } catch (IllegalArgumentException e) {
            System.out.println("Validacio OK: " + e.getMessage());
        }
    }
}
```

### 3. Commit (5 min)

```bash
git add backend-java/src/main/java/com/esportspulse/engine/model/ChampionRecord.java
git commit -m "feat(java): ChampionRecord with compact constructor validation and business methods"
```

---

## Checklist de Lliurament

- [ ] `ChampionRecord` creat com a `record` amb 4 camps
- [ ] Compact constructor valida `championId` no null, `winRate` entre 0 i 100, `pickRate` entre 0 i 100
- [ ] `isMeta()` retorna `true` si `winRate > 52.0 && pickRate > 10.0`
- [ ] `withPatchAdjustment()` retorna un NOU record sense mutar l'original
- [ ] 3 instancies creades i provades a `main` — tot imprimeix el que s'espera
- [ ] Commit amb format Conventional Commits
