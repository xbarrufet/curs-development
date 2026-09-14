# Setmana 18 — Dimarts: Disseny Normalitzat: Relacions i Foreign Keys

## Objectiu del Dia

Dissenyar un esquema normalitzat per EsportsPulse amb múltiples taules relacionades (Champions, Patches, Players). Escriure les migracions Flyway V2 i V3 per crear-les. Al final del dia, la base de dades té un esquema relacional correcte amb foreign keys, constraints i dades de prova.

---

## Teoria

### Normalització: Per Què No Tot en Una Taula?

Imagina que guardes tota la informació en una sola taula:

```
| champion | role | patch | patch_date | player | player_team |
|----------|------|-------|------------|--------|-------------|
| Ahri     | MID  | 14.1  | 2024-01-10 | Faker  | T1          |
| Ahri     | MID  | 14.1  | 2024-01-10 | Caps   | G2          |
| Ahri     | MID  | 14.2  | 2024-01-24 | Faker  | T1          |
```

**Problemes:**
- **Redundància:** "Ahri, MID" es repeteix en cada fila
- **Anomalia d'actualització:** Si Ahri canvia de rol, has d'actualitzar N files
- **Anomalia d'eliminació:** Si esborres l'últim jugador d'un campió, perds les dades del campió

### Formes Normals (Versió Pràctica)

- **1NF (Primera Forma Normal):** Cada cel·la conté un sol valor atòmic. No llistes separades per comes.
  - Malament: `roles = "MID,SUPPORT"` → Bé: una fila per rol, o una taula `champion_roles`

- **2NF (Segona Forma Normal):** Cap columna depèn només d'una part de la clau primària.
  - Si la clau és `(champion_id, patch_id)`, el `role` del campió no depèn del patch → va en una altra taula.

- **3NF (Tercera Forma Normal):** Cap columna depèn d'una altra columna que no sigui la clau.
  - Si tens `player → team → team_region`, la regió depèn de l'equip, no del jugador → taula separada.

> **Regla pràctica:** Si una dada es pot deduir d'una altra columna que no és la PK, mou-la a una altra taula.

### Quan Desnormalitzar?

La normalització perfecta implica molts JOINs. En certs casos convé desnormalitzar:

- **Lectura intensiva:** Si una query s'executa 10.000 cops/segon i fa 5 JOINs, pot ser millor duplicar dades
- **Caches:** Guardar el resultat pre-calculat (ho veurem divendres amb Redis)
- **Reporting/analytics:** Taules materialitzades per a dashboards

> **Principi:** Normalitza per defecte. Desnormalitza quan tens dades de rendiment que ho justifiquin.

### Foreign Keys i Integritat Referencial

Una **foreign key** (FK) és una columna que referencia la clau primària d'una altra taula. PostgreSQL garanteix que:
- No pots inserir una FK que no existeixi a la taula referenciada
- No pots esborrar una fila si altres taules hi fan referència (o pots configurar `CASCADE`)

```sql
-- La columna champion_id NOMÉS pot contenir valors que existeixin a champions.id
ALTER TABLE player_champion_stats
    ADD CONSTRAINT fk_champion
    FOREIGN KEY (champion_id) REFERENCES champions(id);
```

> **Lectura recomanada (opcional, no bloquejant):**
> - [PostgreSQL Tutorial — Foreign Keys](https://www.postgresqltutorial.com/postgresql-tutorial/postgresql-foreign-key/)
> - [Database Normalization Explained](https://www.guru99.com/database-normalization.html)

---

## Activitat

### 1. Dissenyar l'esquema (30 min)

Dibuixa (en paper o eina) l'esquema relacional amb aquestes taules:

```
┌─────────────────┐       ┌─────────────────┐
│    champions     │       │     patches      │
├─────────────────┤       ├─────────────────┤
│ id (PK)         │       │ id (PK)         │
│ name            │       │ version         │
│ role            │       │ release_date    │
│ description     │       │ notes           │
│ created_at      │       └────────┬────────┘
└────────┬────────┘                │
         │                         │
         │    ┌────────────────────┘
         │    │
┌────────┴────┴───────┐
│ champion_patch_stats │     ┌─────────────────┐
├─────────────────────┤     │     players      │
│ id (PK)             │     ├─────────────────┤
│ champion_id (FK)    │     │ id (PK)         │
│ patch_id (FK)       │     │ summoner_name   │
│ win_rate            │     │ team            │
│ pick_rate           │     │ region          │
│ ban_rate            │     │ main_role       │
└─────────────────────┘     │ created_at      │
                            └─────────────────┘
```

**Decisions de disseny:**
- `champion_patch_stats` és una taula d'associació: les estadístiques d'un campió canvien amb cada patch
- `win_rate` ja no està a `champions` sinó a `champion_patch_stats` (depèn del patch, no del campió sol)
- `players` és independent per ara; la setmana 19 la connectarem amb events

### 2. Escriure la migració V2: Patches i Stats (30 min)

Crea `V2__add_patches_and_stats.sql`:

```sql
-- V2: Afegir taula de patches i estadístiques per patch
-- Les estadístiques d'un campió varien amb cada patch, per tant
-- les separem en una taula d'associació champion_patch_stats.

-- Taula de patches (versions del joc)
CREATE TABLE patches (
    id           BIGSERIAL PRIMARY KEY,
    -- Versió del patch (ex: "14.1", "14.2") — ha de ser única
    version      VARCHAR(20) NOT NULL UNIQUE,
    -- Data de publicació del patch
    release_date DATE        NOT NULL,
    -- Notes del patch (text lliure, pot ser molt llarg)
    notes        TEXT
);

COMMENT ON TABLE patches IS 'Versions (patches) del joc amb data de publicació';

-- Taula d'estadístiques per campió i patch
CREATE TABLE champion_patch_stats (
    id           BIGSERIAL PRIMARY KEY,
    -- Referència al campió — si s'esborra el campió, s'esborren les stats
    champion_id  BIGINT           NOT NULL REFERENCES champions(id) ON DELETE CASCADE,
    -- Referència al patch — si s'esborra el patch, s'esborren les stats
    patch_id     BIGINT           NOT NULL REFERENCES patches(id) ON DELETE CASCADE,
    -- Percentatge de victòries en aquest patch (0.0 a 1.0)
    win_rate     DOUBLE PRECISION NOT NULL,
    -- Percentatge de vegades que es tria (pick rate)
    pick_rate    DOUBLE PRECISION NOT NULL,
    -- Percentatge de vegades que es baneja (ban rate)
    ban_rate     DOUBLE PRECISION NOT NULL DEFAULT 0.0,
    -- Restricció: un campió només pot tenir unes stats per patch
    CONSTRAINT uq_champion_patch UNIQUE (champion_id, patch_id)
);

COMMENT ON TABLE champion_patch_stats IS 'Estadístiques de cada campió per a cada patch';

-- Eliminem win_rate de la taula champions (ara està a champion_patch_stats)
ALTER TABLE champions DROP COLUMN IF EXISTS win_rate;
```

### 3. Escriure la migració V3: Players (20 min)

Crea `V3__add_players_table.sql`:

```sql
-- V3: Afegir taula de jugadors professionals
-- Per ara independent; es connectarà amb campions en futures migracions.

CREATE TABLE players (
    id            BIGSERIAL PRIMARY KEY,
    -- Nom dins el joc (summoner name) — únic
    summoner_name VARCHAR(100) NOT NULL UNIQUE,
    -- Nom de l'equip professional
    team          VARCHAR(100),
    -- Regió competitiva (LEC, LCK, LCS, LPL, etc.)
    region        VARCHAR(20),
    -- Rol principal del jugador
    main_role     VARCHAR(50)  NOT NULL,
    -- Data de creació del registre
    created_at    TIMESTAMP    NOT NULL DEFAULT NOW()
);

COMMENT ON TABLE players IS 'Jugadors professionals de League of Legends';

-- Índex per buscar jugadors per equip (query freqüent)
CREATE INDEX idx_players_team ON players(team);
-- Índex per buscar jugadors per regió
CREATE INDEX idx_players_region ON players(region);
```

### 4. Inserir dades de prova (20 min)

Crea `V4__seed_data.sql` amb dades inicials:

```sql
-- V4: Dades de prova per a desenvolupament
-- Aquestes dades permeten testejar queries des del primer moment.

-- Campions
INSERT INTO champions (name, role, description) VALUES
    ('Ahri', 'MID', 'Maga amb mobilitat i ràfegues de dany'),
    ('Jinx', 'ADC', 'Tiradora amb dany explosiu a late game'),
    ('Thresh', 'SUPPORT', 'Suport amb CC i utilitat defensiva'),
    ('Lee Sin', 'JUNGLE', 'Jungla mecànicament exigent amb alta mobilitat'),
    ('Garen', 'TOP', 'Guerrer tanc amb kit senzill i efectiu');

-- Patches
INSERT INTO patches (version, release_date, notes) VALUES
    ('14.1', '2024-01-10', 'Primer patch de la temporada 2024'),
    ('14.2', '2024-01-24', 'Ajustos de balanceig generals'),
    ('14.3', '2024-02-07', 'Canvis importants a objectes de suport');

-- Estadístiques per patch (usem subqueries per obtenir els IDs)
INSERT INTO champion_patch_stats (champion_id, patch_id, win_rate, pick_rate, ban_rate)
SELECT c.id, p.id, stats.win_rate, stats.pick_rate, stats.ban_rate
FROM (VALUES
    ('Ahri',    '14.1', 0.52, 0.12, 0.08),
    ('Ahri',    '14.2', 0.51, 0.11, 0.07),
    ('Jinx',    '14.1', 0.50, 0.15, 0.05),
    ('Jinx',    '14.2', 0.53, 0.18, 0.10),
    ('Thresh',  '14.1', 0.49, 0.10, 0.03),
    ('Lee Sin', '14.1', 0.48, 0.20, 0.15),
    ('Garen',   '14.1', 0.54, 0.08, 0.02)
) AS stats(champ_name, patch_ver, win_rate, pick_rate, ban_rate)
JOIN champions c ON c.name = stats.champ_name
JOIN patches p ON p.version = stats.patch_ver;

-- Jugadors
INSERT INTO players (summoner_name, team, region, main_role) VALUES
    ('Faker', 'T1', 'LCK', 'MID'),
    ('Caps', 'G2 Esports', 'LEC', 'MID'),
    ('Gumayusi', 'T1', 'LCK', 'ADC'),
    ('Keria', 'T1', 'LCK', 'SUPPORT'),
    ('Jankos', 'Team Heretics', 'LEC', 'JUNGLE');
```

### 5. Verificar i commit (15 min)

```bash
# Reiniciar l'aplicació perquè Flyway apliqui V2, V3 i V4
mvn spring-boot:run

# Verificar que totes les taules existeixen
docker exec -it esportspulse-postgres psql -U esports -d esportspulse -c "\dt"

# Verificar les dades de prova
docker exec -it esportspulse-postgres psql -U esports -d esportspulse \
  -c "SELECT c.name, p.version, s.win_rate 
      FROM champion_patch_stats s 
      JOIN champions c ON s.champion_id = c.id 
      JOIN patches p ON s.patch_id = p.id;"

# Verificar historial Flyway
docker exec -it esportspulse-postgres psql -U esports -d esportspulse \
  -c "SELECT version, description, success FROM flyway_schema_history ORDER BY version;"
```

```bash
git add .
git commit -m "feat(db): add normalized schema with patches, stats and players (V2-V4)"
```

---

## Checklist de Lliurament

- [ ] Migracions V2, V3 i V4 creades i aplicades sense errors
- [ ] Taules `patches`, `champion_patch_stats` i `players` existeixen
- [ ] Foreign keys amb `ON DELETE CASCADE` configurades
- [ ] Restricció `UNIQUE (champion_id, patch_id)` a `champion_patch_stats`
- [ ] Dades de prova inserides i verificables amb queries
- [ ] `win_rate` eliminat de `champions` (ara és a `champion_patch_stats`)
- [ ] Commit fet amb les 3 migracions
