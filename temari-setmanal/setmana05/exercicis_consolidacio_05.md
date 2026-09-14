# Setmana 5 — Exercicis de Consolidació

---

## Bàsics (has de saber fer-ho)

### 1. SQL a mà: 5 queries sense JPA
Obre la consola H2 i escriu 5 queries SQL a mà (sense tocar Java):
1. Els 5 jocs més populars ordenats per jugadors actius (DESC).
2. El preu mitjà dels jocs de pagament (price > 0).
3. Quants jocs hi ha per rang de preu (free / <20€ / >=20€) — usa `CASE`.
4. Insereix 3 jocs nous amb `INSERT INTO`.
5. Actualitza el preu d'un joc amb `UPDATE` i verifica amb `SELECT`.

**Connexió S1:** Executa `EXPLAIN` sobre la query 1 amb i sense index a `active_player_count`. Quina diferència veus?

**Fet quan:** Les 5 queries copiades a un fitxer `queries.sql` dins el projecte. L'EXPLAIN mostra la diferència entre table scan i index scan.

### 2. Swap verification: els tests de S2 passen amb JPA
Executa tots els tests de S2 (`ChampionRepositoryTests`, `ChampionManagementServiceTests`) sense modificar-los. Han de passar ara que el repository és JPA en lloc d'InMemory. Si algun falla, identifica per què i corregeix **sense canviar el test** — el problema és a la implementació, no al test.

**Connexió S2 + SOLID:** Això demostra el poder del patró Repository i DIP. Si els tests no passen, és que l'abstracció té un forat.

**Fet quan:** `mvn test` passa al 100% incloent tots els tests de setmanes anteriors.

### 3. Python: SQLite repository
Implementa `SqlitePlayerRepository` en Python (per al `PlayerRecord` de l'exercici de consolidació S2). Mètodes: `save()`, `find_by_id()`, `find_all()`, `find_by_level_greater_than()`. Tests amb `pytest` usant una BD `:memory:`.

**Fet quan:** 4 tests que passen, usant `sqlite3` amb paràmetres vinculats (mai concatenació de strings).

---

## Avançats (si vas sobrat)

### 4. Query derivada custom
Afegeix a `ChampionJpaRepository` una query derivada que Spring Data no pot generar automàticament: "champions amb winRate entre X i Y, ordenats per partides jugades, limitant a N resultats". Usa `@Query` amb JPQL. Escriu el test corresponent.

**Connexió S7:** Aquesta mateixa query serà l'endpoint `GET /games?minPrice=X&maxPrice=Y&limit=N` a la setmana 7.

**Fet quan:** Query funcional amb `@Query`, test que verifica el filtratge i l'ordre.

### 5. Migrar un repository extern
Busca un projecte open source petit a GitHub que tingui un `InMemoryRepository` (o equivalent). Fes un fork, crea una branca, i migra'l a JPA + H2. Verifica que els tests originals passen. No cal fer PR — l'objectiu és practicar el swap en codi que no has escrit.

**Fet quan:** Fork amb branca on el repository és JPA i els tests originals passen.
