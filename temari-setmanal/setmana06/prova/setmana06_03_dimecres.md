# Setmana 06 — Dimecres: Fonaments SQL — SELECT, WHERE, JOIN i Indexos

## Objectiu del Dia

Dominar les operacions SQL fonamentals que JPA genera per nosaltres entre bastidors. Al final del dia sabras escriure queries SQL a ma, entendras com funcionen els indexos i podras analitzar el rendiment de les consultes amb EXPLAIN.

---

## Teoria

### SQL: El Llenguatge de les Bases de Dades

SQL (Structured Query Language) te mes de 50 anys i continua sent l'estandard per treballar amb dades relacionals. Quan JPA genera `findByNameContaining("Ahri")`, per sota executa SQL. Entendre SQL et dona **control total** sobre les teves dades.

### CRUD: Les 4 Operacions Basiques

#### CREATE TABLE — Definir l'Estructura

```sql
-- Creem la taula de campions
-- PRIMARY KEY: identifica unicament cada fila (no pot repetir-se)
-- NOT NULL: el camp es obligatori (no pot ser buit)
-- VARCHAR(100): text amb maxim 100 caracters
CREATE TABLE champions (
    champion_id VARCHAR(50) PRIMARY KEY,   -- Clau primaria unica
    name        VARCHAR(100) NOT NULL,     -- Nom obligatori
    games_played INT DEFAULT 0,            -- Partides jugades, per defecte 0
    win_rate    DOUBLE,                    -- Percentatge de victories
    version     BIGINT DEFAULT 0           -- Control de concurrencia (JPA @Version)
);
```

#### INSERT — Afegir Dades

```sql
-- Inserim campions a la taula
-- L'ordre dels valors ha de coincidir amb l'ordre de les columnes
INSERT INTO champions (champion_id, name, games_played, win_rate)
VALUES ('ahri-001', 'Ahri', 1500, 52.3);

INSERT INTO champions (champion_id, name, games_played, win_rate)
VALUES ('jinx-002', 'Jinx', 2300, 51.8);

INSERT INTO champions (champion_id, name, games_played, win_rate)
VALUES ('thresh-003', 'Thresh', 3100, 49.5);

INSERT INTO champions (champion_id, name, games_played, win_rate)
VALUES ('yasuo-004', 'Yasuo', 4200, 48.7);

INSERT INTO champions (champion_id, name, games_played, win_rate)
VALUES ('lux-005', 'Lux', 2800, 53.1);

INSERT INTO champions (champion_id, name, games_played, win_rate)
VALUES ('zed-006', 'Zed', 1900, 50.2);

INSERT INTO champions (champion_id, name, games_played, win_rate)
VALUES ('leona-007', 'Leona', 1200, 51.5);
```

#### SELECT — Consultar Dades

```sql
-- Seleccionar TOTS els campions amb TOTES les columnes
-- * significa "totes les columnes" — evita-ho en produccio (selecciona nomes el que necessitis)
SELECT * FROM champions;

-- Seleccionar nomes nom i win_rate — mes eficient que SELECT *
SELECT name, win_rate FROM champions;

-- Comptar quants campions tenim
-- COUNT(*) es una funcio d'agregacio — retorna un sol valor
SELECT COUNT(*) AS total_champions FROM champions;
```

#### UPDATE — Modificar Dades

```sql
-- Actualitzar el win_rate d'Ahri
-- WHERE es OBLIGATORI — sense WHERE, actualitzaries TOTES les files!
-- Error comu i catastrofic: UPDATE champions SET win_rate = 55.0; (sense WHERE)
UPDATE champions SET win_rate = 55.0 WHERE champion_id = 'ahri-001';

-- Incrementar partides jugades
-- Pots fer operacions aritmetiques directament
UPDATE champions SET games_played = games_played + 100 WHERE champion_id = 'jinx-002';
```

#### DELETE — Eliminar Dades

```sql
-- Eliminar un campió concret
-- SEMPRE amb WHERE — sense WHERE, elimines TOTA la taula!
DELETE FROM champions WHERE champion_id = 'yasuo-004';

-- Eliminar campions amb menys de 50% de win rate
DELETE FROM champions WHERE win_rate < 50.0;
```

### Filtratge i Ordenacio

#### WHERE — Filtrar Resultats

```sql
-- Campions amb win rate superior al 50%
SELECT name, win_rate FROM champions
WHERE win_rate > 50.0;

-- Campions amb mes de 2000 partides I win rate positiu
-- AND: les DUES condicions han de ser certes
SELECT name, games_played, win_rate FROM champions
WHERE games_played > 2000 AND win_rate > 50.0;

-- Campions que es diguin Ahri O Jinx
-- OR: alguna de les condicions ha de ser certa
SELECT * FROM champions
WHERE name = 'Ahri' OR name = 'Jinx';

-- Equivalent mes net amb IN
-- IN: comprova si el valor esta dins d'una llista
SELECT * FROM champions
WHERE name IN ('Ahri', 'Jinx', 'Lux');

-- Cercar per patrons amb LIKE
-- %: qualsevol seqüencia de caracters
-- _: exactament un caracter
SELECT * FROM champions WHERE name LIKE 'A%';     -- Comenca per A
SELECT * FROM champions WHERE name LIKE '%x';      -- Acaba en x
SELECT * FROM champions WHERE name LIKE '%re%';    -- Conte "re"
```

#### ORDER BY i LIMIT

```sql
-- Ordenar per win rate descendent (millors primer)
-- DESC: descendent (de mes gran a mes petit)
-- ASC: ascendent (per defecte, de mes petit a mes gran)
SELECT name, win_rate FROM champions
ORDER BY win_rate DESC;

-- Top 3 campions amb millor win rate
-- LIMIT: restringeix el nombre de resultats (molt util per paginacio)
SELECT name, win_rate FROM champions
ORDER BY win_rate DESC
LIMIT 3;

-- Paginacio: pagina 2 amb 3 resultats per pagina
-- OFFSET: salta els primers N resultats
SELECT name, win_rate FROM champions
ORDER BY win_rate DESC
LIMIT 3 OFFSET 3;
```

### Funcions d'Agregacio

```sql
-- Recompte total de campions
SELECT COUNT(*) AS total FROM champions;

-- Mitjana de win rate de tots els campions
-- AVG: calcula la mitjana aritmetica
SELECT AVG(win_rate) AS avg_win_rate FROM champions;

-- Sumatori total de partides jugades
SELECT SUM(games_played) AS total_games FROM champions;

-- Maxim i minim de win rate
SELECT MAX(win_rate) AS best, MIN(win_rate) AS worst FROM champions;
```

#### GROUP BY — Agrupar Resultats

Per veure GROUP BY, afegim una columna `role`:

```sql
-- Afegim columna de rol als campions
ALTER TABLE champions ADD COLUMN role VARCHAR(50);

-- Actualitzem els rols
UPDATE champions SET role = 'Mid' WHERE champion_id IN ('ahri-001', 'zed-006', 'lux-005');
UPDATE champions SET role = 'ADC' WHERE champion_id = 'jinx-002';
UPDATE champions SET role = 'Support' WHERE champion_id IN ('thresh-003', 'leona-007');

-- Comptar campions per rol
-- GROUP BY agrupa files amb el mateix valor i aplica la funcio d'agregacio a cada grup
SELECT role, COUNT(*) AS champions_per_role, AVG(win_rate) AS avg_wr
FROM champions
GROUP BY role
ORDER BY champions_per_role DESC;
```

#### CASE — Logica Condicional

```sql
-- Classificar campions per nivell de win rate
-- CASE funciona com un if/else dins de SQL
SELECT name, win_rate,
    CASE
        WHEN win_rate >= 53.0 THEN 'S-Tier'
        WHEN win_rate >= 51.0 THEN 'A-Tier'
        WHEN win_rate >= 49.0 THEN 'B-Tier'
        ELSE 'C-Tier'
    END AS tier
FROM champions
ORDER BY win_rate DESC;
```

### JOINs: Relacionar Taules

En una base de dades real, les dades es reparteixen en multiples taules. Els JOINs les connecten:

```sql
-- Creem una taula de patches (actualitzacions del joc)
CREATE TABLE patches (
    patch_id VARCHAR(20) PRIMARY KEY,
    patch_version VARCHAR(10) NOT NULL,
    release_date DATE NOT NULL
);

-- Taula intermedia: canvis de campions per patch
-- Cada fila relaciona un campió amb un patch
CREATE TABLE champion_patches (
    champion_id VARCHAR(50) NOT NULL,
    patch_id VARCHAR(20) NOT NULL,
    win_rate_change DOUBLE,  -- Canvi de win rate en aquest patch
    PRIMARY KEY (champion_id, patch_id), -- Clau composta: unica combinacio campió+patch
    FOREIGN KEY (champion_id) REFERENCES champions(champion_id),
    FOREIGN KEY (patch_id) REFERENCES patches(patch_id)
);

-- Inserim dades de patches
INSERT INTO patches VALUES ('patch-14.1', '14.1', '2024-01-10');
INSERT INTO patches VALUES ('patch-14.2', '14.2', '2024-01-24');

-- Inserim canvis de campions per patch
INSERT INTO champion_patches VALUES ('ahri-001', 'patch-14.1', 2.5);
INSERT INTO champion_patches VALUES ('ahri-001', 'patch-14.2', -1.0);
INSERT INTO champion_patches VALUES ('jinx-002', 'patch-14.1', -0.5);

-- INNER JOIN: retorna nomes files amb coincidencia a les DUES taules
-- Si un campió no te canvis en cap patch, NO apareix
SELECT c.name, p.patch_version, cp.win_rate_change
FROM champions c
INNER JOIN champion_patches cp ON c.champion_id = cp.champion_id
INNER JOIN patches p ON cp.patch_id = p.patch_id
ORDER BY c.name, p.patch_version;

-- LEFT JOIN: retorna TOTS els campions, tinguin o no canvis
-- Els campions sense canvis tindran NULL a les columnes del patch
SELECT c.name, p.patch_version, cp.win_rate_change
FROM champions c
LEFT JOIN champion_patches cp ON c.champion_id = cp.champion_id
LEFT JOIN patches p ON cp.patch_id = p.patch_id
ORDER BY c.name;
```

### Indexos: Per que les Queries son Rapides o Lentes

Un index es com l'index d'un llibre — et porta directament a la pagina que busques sense llegir tot el llibre:

```
Sense index (full table scan):
┌──────────────────────────────────────────┐
│ Fila 1 → Fila 2 → Fila 3 → ... → Fila N │  O(n)
│ Ha de llegir TOTES les files              │
└──────────────────────────────────────────┘

Amb index (B-Tree scan):
         ┌───┐
         │ M │         O(log n)
        ╱     ╲
    ┌───┐     ┌───┐
    │ D │     │ T │
   ╱     ╲   ╱     ╲
  A-C   E-L  N-S   U-Z
```

```sql
-- Creem un index al camp "name" per accelerar cerques per nom
-- Sense index: O(n) — ha de llegir totes les files
-- Amb index: O(log n) — va directe al valor
CREATE INDEX idx_champion_name ON champions(name);

-- Index compost: util quan filtrem per dos camps alhora
CREATE INDEX idx_role_winrate ON champions(role, win_rate);

-- EXPLAIN mostra COM la BD executa la query
-- Permet veure si usa un index o fa un full table scan
EXPLAIN SELECT * FROM champions WHERE name = 'Ahri';

-- Comparacio: amb i sense index
-- Primer, sense index (ja que champion_id te index per ser PK)
EXPLAIN SELECT * FROM champions WHERE games_played > 2000;

-- Creem index i tornem a mirar
CREATE INDEX idx_games_played ON champions(games_played);
EXPLAIN SELECT * FROM champions WHERE games_played > 2000;
```

**Quan crear indexos:**
- Columnes que uses sovint en WHERE, JOIN o ORDER BY
- Columnes amb alta cardinalitat (molts valors diferents)

**Quan NO crear indexos:**
- Taules petites (menys de 1000 files — el full scan es suficient)
- Columnes amb poca variabilitat (ex: un camp boolea amb 50/50)
- Taules amb moltes escriptures (cada INSERT/UPDATE ha d'actualitzar l'index)

### Propietats ACID

Les bases de dades relacionals garanteixen 4 propietats que fan les dades fiables:

| Propietat | Significat | Exemple |
|---|---|---|
| **Atomicity** | Tot o res — si una part falla, es desfà tot | Transferencia bancaria: treure d'un compte i posar a l'altre |
| **Consistency** | La BD sempre esta en un estat valid | NOT NULL, FOREIGN KEY — la BD rebutja dades invalides |
| **Isolation** | Transaccions concurrents no interfereixen | Dos usuaris comprant l'ultim producte alhora |
| **Durability** | Un cop confirmat (COMMIT), no es perd | Encara que el servidor es reinicii |

### @Transactional a Spring

L'anotacio `@Transactional` aplica ACID als nostres metodes:

```java
import org.springframework.transaction.annotation.Transactional;

/**
 * @Transactional: Spring obre una transaccio al inici del metode
 * i fa COMMIT si tot va be, o ROLLBACK si hi ha una excepcio.
 *
 * Analogia: transferencia bancaria
 * 1. Treure 100EUR del compte A
 * 2. Posar 100EUR al compte B
 * Si el pas 2 falla, el pas 1 es desfà automaticament.
 */
@Transactional
public void transferChampionStats(String fromId, String toId) {
    // Si qualsevol operacio falla, TOTES es desfan (ROLLBACK)
    ChampionEntity from = jpaRepository.findById(fromId)
        .orElseThrow(() -> new RuntimeException("Campió origen no trobat"));
    ChampionEntity to = jpaRepository.findById(toId)
        .orElseThrow(() -> new RuntimeException("Campió destí no trobat"));

    // Transferim partides d'un campió a l'altre
    int gamesToTransfer = from.getGamesPlayed() / 2;
    from.setGamesPlayed(from.getGamesPlayed() - gamesToTransfer);
    to.setGamesPlayed(to.getGamesPlayed() + gamesToTransfer);

    // JPA detecta els canvis automaticament ("dirty checking")
    // No cal cridar save() explicitament dins una @Transactional
}
```

---

## Activitat

### Exercici: Practica SQL a la Consola H2

**Durada estimada:** 90 minuts

#### Preparacio (5 min)

1. Arrenca l'aplicacio: `mvn spring-boot:run`
2. Obre la consola H2: `http://localhost:8080/h2-console`
3. Connecta amb `jdbc:h2:mem:esportspulse`

#### Exercici 1: Insercions (10 min)

Insereix manualment 7 campions a la taula `champions` amb les sentencies INSERT proporcionades a la teoria.

#### Exercici 2: Consultes basiques (15 min)

Escriu i executa:
1. `SELECT` de tots els campions amb win rate > 50%
2. `SELECT` amb `ORDER BY win_rate DESC LIMIT 3`
3. `SELECT` amb `LIKE` per trobar campions que continguin "a" al nom
4. `UPDATE` per modificar el win rate d'un campió
5. `DELETE` d'un campió concret

#### Exercici 3: Agregacions (15 min)

1. Calcula la mitjana de win rate de tots els campions
2. Compta quants campions tenen mes de 2000 partides
3. Afegeix la columna `role`, actualitza els rols i fes un `GROUP BY role`
4. Usa `CASE` per classificar campions en tiers

#### Exercici 4: JOINs (20 min)

1. Crea la taula `patches` i `champion_patches`
2. Insereix dades de patches
3. Escriu un `INNER JOIN` per veure canvis per campió i patch
4. Escriu un `LEFT JOIN` per veure TOTS els campions (amb o sense canvis)

#### Exercici 5: Indexos i EXPLAIN (15 min)

1. Executa `EXPLAIN SELECT * FROM champions WHERE name = 'Ahri';` **sense** index
2. Crea l'index: `CREATE INDEX idx_champion_name ON champions(name);`
3. Executa el mateix `EXPLAIN` i compara
4. Crea un index a `games_played` i repeteix l'experiment

#### Exercici 6: @Transactional (10 min)

1. Llegeix el codi d'exemple de `transferChampionStats`
2. Respon: que passaria si NO posem `@Transactional` i el segon `findById` falla?
3. Escriu la resposta com a comentari al codi

---

## Checklist de Lliurament

- [ ] 7 campions inserits a la consola H2 amb INSERT
- [ ] 5 queries SELECT executades (filtre, ordenacio, LIKE, UPDATE, DELETE)
- [ ] Agregacions amb COUNT, AVG, GROUP BY i CASE funcionals
- [ ] Taules `patches` i `champion_patches` creades amb JOINs funcionals
- [ ] EXPLAIN executat abans i despres de crear un index — diferencia documentada
- [ ] Pregunta sobre @Transactional resposta com a comentari
- [ ] Captures de pantalla o notes de les queries i resultats
