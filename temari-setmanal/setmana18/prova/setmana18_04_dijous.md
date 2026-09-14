# Setmana 18 — Dijous: Indexes, EXPLAIN ANALYZE i Optimització

## Objectiu del Dia

Entendre per què les queries poden ser lentes, com els índexs les acceleren, i com llegir un pla d'execució amb `EXPLAIN ANALYZE`. Detectar i resoldre el problema N+1 de JPA. Al final del dia, has optimitzat almenys una query real amb un índex i has corregit un N+1.

---

## Teoria

### Per Què Les Queries Són Lentes?

Sense índexs, PostgreSQL fa un **Sequential Scan** (Seq Scan): llegeix TOTA la taula fila per fila per trobar les que coincideixen amb el `WHERE`.

```
Seq Scan: llegeix 1.000.000 files per trobar-ne 3
           → O(n) — temps lineal

Index Scan: va directament a les 3 files
           → O(log n) — temps logarítmic
```

Amb 100 files no es nota. Amb 1.000.000, un Seq Scan pot trigar segons.

### Índexs: Què Són i Com Funcionen

Un índex és una estructura de dades (normalment un **B-tree**) que permet trobar files ràpidament sense llegir tota la taula.

**Analogia:** L'índex d'un llibre. Per trobar "PostgreSQL" no llegeixes tot el llibre — vas a l'índex, trobes la pàgina i hi vas directament.

```sql
-- Crear un índex a la columna 'name' de champions
CREATE INDEX idx_champions_name ON champions(name);

-- Crear un índex compost (dues columnes) — útil per queries que filtren per ambdues
CREATE INDEX idx_stats_champion_patch ON champion_patch_stats(champion_id, patch_id);
```

**Quan crear índexs:**
- Columnes que apareixen sovint a `WHERE`, `JOIN ON`, `ORDER BY`
- Columnes amb alta cardinalitat (molts valors diferents)
- Foreign keys (PostgreSQL NO crea índexs automàtics a les FK!)

**Quan NO crear índexs:**
- Taules molt petites (el Seq Scan és més ràpid que consultar l'índex)
- Columnes amb pocs valors diferents (ex: `boolean`)
- Taules amb moltes escriptures i poques lectures (els índexs alenteixen els INSERTs)

### EXPLAIN ANALYZE: Llegir el Pla d'Execució

`EXPLAIN ANALYZE` executa la query i mostra exactament com PostgreSQL l'ha resolt:

```sql
EXPLAIN ANALYZE
SELECT * FROM champions WHERE name = 'Ahri';
```

Resultat (sense índex):
```
Seq Scan on champions  (cost=0.00..1.06 rows=1 width=200) (actual time=0.015..0.016 rows=1 loops=1)
  Filter: ((name)::text = 'Ahri'::text)
  Rows Removed by Filter: 4
Planning Time: 0.080 ms
Execution Time: 0.035 ms
```

Resultat (amb índex):
```
Index Scan using idx_champions_name on champions  (cost=0.14..8.15 rows=1 width=200) (actual time=0.025..0.026 rows=1 loops=1)
  Index Cond: ((name)::text = 'Ahri'::text)
Planning Time: 0.100 ms
Execution Time: 0.045 ms
```

**Claus per llegir el pla:**

| Element | Significat |
|---------|-----------|
| `Seq Scan` | Lectura seqüencial de tota la taula (potencialment lent) |
| `Index Scan` | Usa un índex per anar directament a les files |
| `cost=X..Y` | Estimació del cost (unitats arbitràries, relatiu) |
| `rows=N` | Nombre de files que PostgreSQL estima que retornarà |
| `actual time` | Temps real d'execució en mil·lisegons |
| `Rows Removed by Filter` | Files llegides però descartades (ineficiència) |
| `Nested Loop` | Un JOIN implementat amb bucles niats (pot ser N+1!) |

### El Problema N+1 amb JPA/Hibernate

El **problema N+1** passa quan JPA genera una query per obtenir una llista i després N queries addicionals per carregar les relacions de cada element:

```
1 query:  SELECT * FROM champions                         → 5 resultats
5 queries: SELECT * FROM champion_patch_stats WHERE champion_id = 1
           SELECT * FROM champion_patch_stats WHERE champion_id = 2
           SELECT * FROM champion_patch_stats WHERE champion_id = 3
           SELECT * FROM champion_patch_stats WHERE champion_id = 4
           SELECT * FROM champion_patch_stats WHERE champion_id = 5
Total: 1 + 5 = 6 queries (amb 1000 campions serien 1001!)
```

**Com detectar-lo:**
1. Activar el log de queries SQL a `application.properties`:
```properties
# Mostra les queries SQL generades per Hibernate al log
spring.jpa.show-sql=true
# Format les queries perquè siguin llegibles
spring.jpa.properties.hibernate.format_sql=true
# Mostra estadístiques de sessions Hibernate (nombre de queries)
spring.jpa.properties.hibernate.generate_statistics=true
# Log de queries lentes (> 100ms)
spring.jpa.properties.hibernate.session.events.log.LOG_QUERIES_SLOWER_THAN_MS=100
```

2. Observar als logs: si veus la mateixa query repetida amb IDs diferents, tens un N+1.

**Solucions:**

```java
// Solució 1: JOIN FETCH a la query JPQL
// Una sola query amb JOIN que carrega campions + stats d'un cop
@Query("SELECT c FROM Champion c JOIN FETCH c.patchStats WHERE c.role = :role")
List<Champion> findByRoleWithStats(@Param("role") String role);

// Solució 2: @EntityGraph a la declaració del mètode
// Indica a JPA que carregui les relacions especificades amb un JOIN
@EntityGraph(attributePaths = {"patchStats"})
List<Champion> findByRole(String role);
```

Ambdues solucions generen **una sola query SQL** amb JOIN en comptes de N+1.

> **Lectura recomanada (opcional, no bloquejant):**
> - [Use The Index, Luke](https://use-the-index-luke.com/) — Guia visual d'índexs SQL
> - [Hibernate N+1 Problem](https://vladmihalcea.com/n-plus-1-query-problem/) — Vlad Mihalcea

---

## Activitat

### 1. Generar dades per notar la diferència (20 min)

Les 5 files actuals són massa poques per veure l'impacte dels índexs. Crea una migració que generi dades massives:

Crea `V5__generate_bulk_data.sql`:

```sql
-- V5: Generar dades massives per practicar optimització
-- 1000 campions ficticis i estadístiques per mesurar l'impacte dels índexs

-- Generar 1000 campions ficticis amb generate_series
INSERT INTO champions (name, role, description)
SELECT
    -- Nom únic per cada campió fictici
    'Champion_' || i,
    -- Repartir entre 5 rols de forma cíclica
    CASE (i % 5)
        WHEN 0 THEN 'TOP'
        WHEN 1 THEN 'JUNGLE'
        WHEN 2 THEN 'MID'
        WHEN 3 THEN 'ADC'
        WHEN 4 THEN 'SUPPORT'
    END,
    'Campió generat automàticament per a tests de rendiment'
FROM generate_series(1, 1000) AS i;

-- Generar estadístiques per cada campió i patch existent
INSERT INTO champion_patch_stats (champion_id, patch_id, win_rate, pick_rate, ban_rate)
SELECT
    c.id,
    p.id,
    -- Win rate aleatori entre 0.40 i 0.60
    0.40 + (random() * 0.20),
    -- Pick rate aleatori entre 0.01 i 0.30
    0.01 + (random() * 0.29),
    -- Ban rate aleatori entre 0.00 i 0.15
    random() * 0.15
FROM champions c
CROSS JOIN patches p
-- Excloure combinacions que ja existeixen (dades seed de V4)
WHERE NOT EXISTS (
    SELECT 1 FROM champion_patch_stats s
    WHERE s.champion_id = c.id AND s.patch_id = p.id
);
```

### 2. Analitzar queries sense índexs (20 min)

```sql
-- Activa el temporitzador
\timing

-- Query 1: Buscar per nom sense índex
EXPLAIN ANALYZE
SELECT * FROM champions WHERE name = 'Champion_500';

-- Anota: Seq Scan? Quant triga?

-- Query 2: JOIN sense índex a FK
EXPLAIN ANALYZE
SELECT c.name, s.win_rate
FROM champion_patch_stats s
JOIN champions c ON s.champion_id = c.id
WHERE c.role = 'MID'
ORDER BY s.win_rate DESC
LIMIT 10;

-- Anota: quin tipus de scan fa? Hash Join? Nested Loop?
```

### 3. Crear índexs i mesurar l'impacte (20 min)

```sql
-- Índex a champions.name (busques per nom)
CREATE INDEX idx_champions_name ON champions(name);

-- Índex a champions.role (filtratge per rol)
CREATE INDEX idx_champions_role ON champions(role);

-- Índex a la FK champion_id de champion_patch_stats
-- (PostgreSQL NO crea automàticament índexs a les FK!)
CREATE INDEX idx_stats_champion_id ON champion_patch_stats(champion_id);

-- Índex a la FK patch_id
CREATE INDEX idx_stats_patch_id ON champion_patch_stats(patch_id);
```

Repeteix les mateixes queries:

```sql
-- Mateixa query que abans — ara ha de fer Index Scan
EXPLAIN ANALYZE
SELECT * FROM champions WHERE name = 'Champion_500';

-- Mateixa query amb JOIN — observa si el pla ha canviat
EXPLAIN ANALYZE
SELECT c.name, s.win_rate
FROM champion_patch_stats s
JOIN champions c ON s.champion_id = c.id
WHERE c.role = 'MID'
ORDER BY s.win_rate DESC
LIMIT 10;
```

Compara els temps d'execució i el tipus de scan (Seq Scan vs Index Scan).

### 4. Detectar i corregir un N+1 (30 min)

Activa els logs SQL a `application.properties` (veure secció teoria).

Crea o modifica un endpoint que carregui campions amb les seves estadístiques:

```java
// A ChampionService.java — aquest mètode provoca un N+1
// perquè JPA carrega les stats amb lazy loading (una query per campió)
public List<Champion> getChampionsByRole(String role) {
    // Això genera: 1 SELECT champions + N SELECT champion_patch_stats
    List<Champion> champions = championRepository.findByRole(role);
    // Accedir a les stats força el lazy loading per cada campió
    champions.forEach(c -> c.getPatchStats().size());
    return champions;
}
```

Corregeix-ho al repository:

```java
// A ChampionRepository.java — solució amb @EntityGraph
// @EntityGraph indica a JPA que carregui patchStats amb un JOIN (una sola query)
@EntityGraph(attributePaths = {"patchStats"})
List<Champion> findByRole(String role);
```

Verifica als logs que ara només es genera **una query** amb JOIN.

### 5. Crear la migració d'índexs i commitar (10 min)

Crea `V6__add_indexes.sql` amb els índexs que has creat manualment:

```sql
-- V6: Índexs per optimitzar les queries més freqüents
-- Basats en l'anàlisi amb EXPLAIN ANALYZE

-- Índex per buscar campions per nom (query freqüent des de l'API)
CREATE INDEX IF NOT EXISTS idx_champions_name ON champions(name);
-- Índex per filtrar campions per rol
CREATE INDEX IF NOT EXISTS idx_champions_role ON champions(role);
-- Índexs a les foreign keys (PostgreSQL no els crea automàticament)
CREATE INDEX IF NOT EXISTS idx_stats_champion_id ON champion_patch_stats(champion_id);
CREATE INDEX IF NOT EXISTS idx_stats_patch_id ON champion_patch_stats(patch_id);
```

```bash
git add .
git commit -m "perf(db): add indexes and fix N+1 query with EntityGraph"
```

---

## Checklist de Lliurament

- [ ] Dades massives generades (V5) — almenys 1000 campions
- [ ] `EXPLAIN ANALYZE` executat abans i després dels índexs
- [ ] Diferència de rendiment documentada (Seq Scan vs Index Scan)
- [ ] Índexs creats a columnes de FK i columnes de filtratge freqüent
- [ ] N+1 detectat als logs de Hibernate
- [ ] N+1 corregit amb `@EntityGraph` o `JOIN FETCH`
- [ ] Migració V6 amb els índexs creada
- [ ] Commit fet
