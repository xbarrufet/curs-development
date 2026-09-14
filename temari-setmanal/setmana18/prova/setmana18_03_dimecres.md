# Setmana 18 — Dimecres: Queries SQL Reals: JOINs, GROUP BY, Subqueries

## Objectiu del Dia

Dominar les queries SQL que un desenvolupador backend fa servir cada dia: JOINs entre taules, agregacions amb GROUP BY, subqueries i funcions finestra. Al final del dia, has escrit 10 queries reals contra les dades d'EsportsPulse i saps connectar-te a PostgreSQL des de la terminal.

---

## Teoria

### Connectar-se a PostgreSQL des de la Terminal

La terminal `psql` és l'eina de línia de comandes de PostgreSQL. Connexió bàsica:

```bash
# Connectar des del host (amb el port mapejat a Docker)
psql -h localhost -p 5432 -U esports -d esportspulse

# O des de dins el contenidor Docker
docker exec -it esportspulse-postgres psql -U esports -d esportspulse
```

**Comandes útils dins psql:**
```
\dt          -- Llista de taules
\d champions -- Esquema d'una taula
\x           -- Mode expandit (vertical) per a resultats amples
\timing      -- Mostra el temps d'execució de cada query
\q           -- Sortir
```

### JOINs: Combinar Taules

Un JOIN combina files de dues o més taules basant-se en una condició (normalment una FK).

**Tipus de JOINs (diagrama ASCII):**

```
INNER JOIN          LEFT JOIN           RIGHT JOIN
┌───┐ ┌───┐        ┌───┐ ┌───┐        ┌───┐ ┌───┐
│ A │ │ B │        │ A │ │ B │        │ A │ │ B │
│   ├─┤   │        │   ├─┤   │        │   ├─┤   │
│   │█│   │        │███│█│   │        │   │█│███│
│   ├─┤   │        │   ├─┤   │        │   ├─┤   │
└───┘ └───┘        └───┘ └───┘        └───┘ └───┘
  Només files        Totes les files    Totes les files
  que coincideixen   d'A + les de B     de B + les d'A
  a AMBDUES taules   que coincideixin   que coincideixin
```

```sql
-- INNER JOIN: només campions que tenen estadístiques
SELECT c.name, p.version, s.win_rate
FROM champions c
INNER JOIN champion_patch_stats s ON s.champion_id = c.id
INNER JOIN patches p ON s.patch_id = p.id;

-- LEFT JOIN: tots els campions, tinguin o no estadístiques
-- (els que no en tenen mostren NULL a les columnes de stats)
SELECT c.name, s.win_rate
FROM champions c
LEFT JOIN champion_patch_stats s ON s.champion_id = c.id;
```

**Quan usar cada un:**
- `INNER JOIN`: "Vull només les dades que tenen relació a ambdues taules"
- `LEFT JOIN`: "Vull tots els elements de la taula esquerra, tinguin o no correspondència"
- `RIGHT JOIN`: Poc comú — normalment es reescriu com a LEFT JOIN canviant l'ordre

### GROUP BY i Funcions d'Agregació

`GROUP BY` agrupa files per una o més columnes i permet aplicar funcions d'agregació:

| Funció | Descripció | Exemple |
|--------|-----------|---------|
| `COUNT(*)` | Nombre de files | Quants campions per rol? |
| `AVG(col)` | Mitjana | Win rate mitjà per patch |
| `MAX(col)` | Valor màxim | Millor win rate |
| `MIN(col)` | Valor mínim | Pitjor win rate |
| `SUM(col)` | Suma total | Pick rate total per patch |

```sql
-- Nombre de campions per rol
SELECT role, COUNT(*) AS total
FROM champions
GROUP BY role
ORDER BY total DESC;

-- Win rate mitjà per patch
SELECT p.version, AVG(s.win_rate) AS avg_win_rate
FROM champion_patch_stats s
JOIN patches p ON s.patch_id = p.id
GROUP BY p.version
ORDER BY p.version;
```

**`HAVING` vs `WHERE`:**
- `WHERE` filtra files **abans** de l'agregació
- `HAVING` filtra grups **després** de l'agregació

```sql
-- Rols que tenen més de 1 campió (HAVING filtra el resultat de COUNT)
SELECT role, COUNT(*) AS total
FROM champions
GROUP BY role
HAVING COUNT(*) > 1;
```

### Subqueries

Una subquery és una query dins d'una altra query:

```sql
-- Campions amb win rate superior a la mitjana global
SELECT c.name, s.win_rate
FROM champions c
JOIN champion_patch_stats s ON s.champion_id = c.id
WHERE s.win_rate > (
    -- Subquery: calcula la mitjana global de win rate
    SELECT AVG(win_rate) FROM champion_patch_stats
);
```

**Subquery vs JOIN:** Sovint es pot reescriure una subquery com a JOIN i viceversa. En general, els JOINs són més llegibles i PostgreSQL els optimitza millor.

### Funcions Finestra (Window Functions)

Les funcions finestra operen sobre un conjunt de files relacionades amb la fila actual, **sense col·lapsar-les** (a diferència de GROUP BY):

```sql
-- ROW_NUMBER: assigna un número seqüencial dins cada grup
-- Exemple: classificar campions per win_rate dins de cada patch
SELECT
    c.name,
    p.version,
    s.win_rate,
    -- Dins de cada patch (PARTITION BY), ordena per win_rate descendent
    ROW_NUMBER() OVER (PARTITION BY p.version ORDER BY s.win_rate DESC) AS ranking
FROM champion_patch_stats s
JOIN champions c ON s.champion_id = c.id
JOIN patches p ON s.patch_id = p.id;

-- LAG: accedeix al valor de la fila anterior
-- Exemple: comparar el win_rate d'un campió entre patches consecutius
SELECT
    c.name,
    p.version,
    s.win_rate,
    -- win_rate del patch anterior per al mateix campió
    LAG(s.win_rate) OVER (PARTITION BY c.id ORDER BY p.release_date) AS prev_win_rate,
    -- Diferència amb el patch anterior
    s.win_rate - LAG(s.win_rate) OVER (PARTITION BY c.id ORDER BY p.release_date) AS delta
FROM champion_patch_stats s
JOIN champions c ON s.champion_id = c.id
JOIN patches p ON s.patch_id = p.id;
```

> **Lectura recomanada (opcional, no bloquejant):**
> - [PostgreSQL Window Functions Tutorial](https://www.postgresqltutorial.com/postgresql-window-function/)
> - [Mode SQL Tutorial](https://mode.com/sql-tutorial/) — Interactive SQL exercises

---

## Activitat

### Les 10 Queries (90 min)

Connecta't a PostgreSQL i escriu cada query. Guarda-les en un fitxer `queries_practice.sql` al projecte.

```bash
# Activa el temporitzador per veure quant triga cada query
\timing
```

**Query 1 — Llistat bàsic amb JOIN:**
```sql
-- Mostra campió, patch i win_rate ordenat per win_rate descendent
SELECT c.name AS champion, p.version AS patch, s.win_rate
FROM champion_patch_stats s
JOIN champions c ON s.champion_id = c.id
JOIN patches p ON s.patch_id = p.id
ORDER BY s.win_rate DESC;
```

**Query 2 — LEFT JOIN per trobar buits:**
```sql
-- Campions que NO tenen estadístiques a cap patch
-- (LEFT JOIN + WHERE IS NULL = "anti-join")
SELECT c.name
FROM champions c
LEFT JOIN champion_patch_stats s ON s.champion_id = c.id
WHERE s.id IS NULL;
```

**Query 3 — Agregació per rol:**
```sql
-- Win rate mitjà per rol (quin rol és més fort?)
SELECT c.role, ROUND(AVG(s.win_rate)::numeric, 4) AS avg_wr
FROM champions c
JOIN champion_patch_stats s ON s.champion_id = c.id
GROUP BY c.role
ORDER BY avg_wr DESC;
```

**Query 4 — Filtrar grups amb HAVING:**
```sql
-- Campions amb més d'1 entrada de patch i win_rate mitjà > 0.50
SELECT c.name, COUNT(*) AS patches_played, AVG(s.win_rate) AS avg_wr
FROM champions c
JOIN champion_patch_stats s ON s.champion_id = c.id
GROUP BY c.name
HAVING COUNT(*) > 1 AND AVG(s.win_rate) > 0.50;
```

**Query 5 — Subquery: per sobre de la mitjana:**
```sql
-- Campions amb win_rate per sobre de la mitjana global en qualsevol patch
SELECT DISTINCT c.name, s.win_rate, p.version
FROM champions c
JOIN champion_patch_stats s ON s.champion_id = c.id
JOIN patches p ON s.patch_id = p.id
WHERE s.win_rate > (SELECT AVG(win_rate) FROM champion_patch_stats);
```

**Query 6 — ROW_NUMBER: top campió per patch:**
```sql
-- El campió amb millor win_rate a cada patch
SELECT champion, patch, win_rate FROM (
    SELECT
        c.name AS champion,
        p.version AS patch,
        s.win_rate,
        ROW_NUMBER() OVER (PARTITION BY p.id ORDER BY s.win_rate DESC) AS rn
    FROM champion_patch_stats s
    JOIN champions c ON s.champion_id = c.id
    JOIN patches p ON s.patch_id = p.id
) ranked
WHERE rn = 1;
```

**Query 7 — LAG: evolució entre patches:**
```sql
-- Com ha canviat el win_rate de cada campió entre patches
SELECT
    c.name,
    p.version,
    s.win_rate AS current_wr,
    LAG(s.win_rate) OVER (PARTITION BY c.id ORDER BY p.release_date) AS prev_wr,
    ROUND((s.win_rate - COALESCE(LAG(s.win_rate) OVER (
        PARTITION BY c.id ORDER BY p.release_date), s.win_rate))::numeric, 4) AS change
FROM champion_patch_stats s
JOIN champions c ON s.champion_id = c.id
JOIN patches p ON s.patch_id = p.id
ORDER BY c.name, p.release_date;
```

**Query 8 — Estadístiques combinades:**
```sql
-- Resum per patch: quants campions, win_rate mitjà i màxim
SELECT
    p.version,
    COUNT(s.id) AS num_champions,
    ROUND(AVG(s.win_rate)::numeric, 4) AS avg_wr,
    MAX(s.win_rate) AS max_wr,
    MIN(s.win_rate) AS min_wr
FROM patches p
JOIN champion_patch_stats s ON s.patch_id = p.id
GROUP BY p.version
ORDER BY p.version;
```

**Query 9 — Jugadors per regió:**
```sql
-- Nombre de jugadors per regió i rol principal
SELECT region, main_role, COUNT(*) AS total
FROM players
GROUP BY region, main_role
ORDER BY region, total DESC;
```

**Query 10 — RANK amb empats:**
```sql
-- Classificació de campions per pick_rate al patch 14.1 (amb empats)
SELECT
    c.name,
    s.pick_rate,
    -- RANK deixa forats si hi ha empats (1, 2, 2, 4)
    RANK() OVER (ORDER BY s.pick_rate DESC) AS rank_pick
FROM champion_patch_stats s
JOIN champions c ON s.champion_id = c.id
JOIN patches p ON s.patch_id = p.id
WHERE p.version = '14.1';
```

### Guardar i commitar (10 min)

Guarda les queries en un fitxer per referència futura:

```bash
# Crea el directori si no existeix
mkdir -p backend-java/src/main/resources/sql

# Mou o crea el fitxer queries_practice.sql amb les 10 queries
git add backend-java/src/main/resources/sql/queries_practice.sql
git commit -m "docs(sql): add 10 practice queries with JOINs, aggregations and window functions"
```

---

## Checklist de Lliurament

- [ ] Connectat a PostgreSQL amb `psql` des de la terminal
- [ ] 10 queries escrites i executades sense errors
- [ ] Almenys 2 queries amb JOIN (INNER i LEFT)
- [ ] Almenys 2 queries amb GROUP BY i funcions d'agregació
- [ ] Almenys 1 subquery
- [ ] Almenys 2 queries amb funcions finestra (ROW_NUMBER, LAG, RANK)
- [ ] Fitxer `queries_practice.sql` guardat al projecte
- [ ] Commit fet
