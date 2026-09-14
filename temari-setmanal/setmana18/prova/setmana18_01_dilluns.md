# Setmana 18 — Dilluns: De H2 a PostgreSQL: Migració i Flyway

## Objectiu del Dia

Migrar la base de dades d'EsportsPulse d'H2 (in-memory, només per desenvolupament) a PostgreSQL (producció). Configurar Flyway per gestionar migracions de forma versionada i automàtica. Al final del dia, l'aplicació Spring Boot arrenca amb PostgreSQL i la taula `champions` es crea via migració Flyway.

---

## Teoria

### Per Què Migrar d'H2 a PostgreSQL?

H2 és una base de dades en memòria que vam usar a les primeres setmanes perquè és fàcil d'arrencar: no cal instal·lar res, es reinicia amb cada execució. Però té limitacions greus per a producció:

| Característica | H2 (in-memory) | PostgreSQL |
|---|---|---|
| Persistència | Les dades es perden al reiniciar | Dades permanents a disc |
| Concurrència | Limitada (un sol procés) | Milers de connexions simultànies |
| SQL estàndard | Dialecte propi amb diferències | SQL estàndard complet |
| Ecosistema | Sense extensions | PostGIS, pg_trgm, vectors... |
| Ús real | Tests i prototips | Producció a escala |

PostgreSQL ja el tenim al `docker-compose.yml` des de la Setmana 8. Avui el connectarem de debò.

### Flyway: Migracions Versionades

Fins ara, Spring Boot creava les taules automàticament amb `spring.jpa.hibernate.ddl-auto=update`. Això és perillós en producció perquè:
- No saps exactament què ha canviat
- No pots revertir canvis
- Dos devs poden acabar amb esquemes diferents

**Flyway** resol això amb **migracions versionades**: fitxers SQL numerats que s'executen en ordre.

```
src/main/resources/db/migration/
├── V1__create_champions_table.sql    ← Primera migració
├── V2__add_players_table.sql         ← Segona migració
└── V3__add_index_on_name.sql         ← Tercera migració
```

**Regles de Flyway:**
1. Cada fitxer comença amb `V` + número + `__` (doble guió baix) + descripció
2. Un cop aplicada, una migració **mai es modifica** — se'n crea una de nova
3. Flyway porta un registre intern (`flyway_schema_history`) amb les migracions aplicades
4. A l'arrencar l'aplicació, Flyway detecta quines falten i les aplica automàticament

```
V1 → V2 → V3 (aplicades)
                 V4 (nova — Flyway l'aplica)
```

### Configuració de Spring Boot per PostgreSQL

El canvi principal és a `application.properties`:

```properties
# --- Connexió a PostgreSQL (substitueix la configuració d'H2) ---
# URL JDBC: jdbc:postgresql://host:port/nom_base_dades
spring.datasource.url=jdbc:postgresql://localhost:5432/esportspulse
# Usuari i contrasenya (els mateixos que al docker-compose)
spring.datasource.username=esports
spring.datasource.password=esports_pwd
# Driver JDBC per PostgreSQL
spring.datasource.driver-class-name=org.postgresql.Driver

# --- JPA/Hibernate ---
# Canviem de 'update' a 'validate': Hibernate comprova que les entitats
# coincideixin amb l'esquema, però NO crea ni modifica taules (això ho fa Flyway)
spring.jpa.hibernate.ddl-auto=validate
# Dialecte específic per PostgreSQL (optimitza les queries generades)
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
```

> **Lectura recomanada (opcional, no bloquejant):**
> - [Flyway Documentation](https://documentation.red-gate.com/fd) — Getting Started
> - [Spring Boot Database Migrations with Flyway](https://www.baeldung.com/database-migrations-with-flyway)

---

## Activitat

### 1. Verificar que PostgreSQL funciona al Docker (10 min)

Assegura't que el servei PostgreSQL del `docker-compose.yml` està actiu:

```bash
# Arrencar els contenidors (si no estan ja en marxa)
docker-compose up -d postgres

# Verificar que PostgreSQL respon
docker exec -it esportspulse-postgres psql -U esports -d esportspulse -c "SELECT version();"
```

Has de veure la versió de PostgreSQL. Si el contenidor no existeix, afegeix-lo al `docker-compose.yml`:

```yaml
services:
  # Servei de base de dades PostgreSQL
  postgres:
    image: postgres:16            # Versió LTS actual
    container_name: esportspulse-postgres
    environment:
      POSTGRES_DB: esportspulse   # Nom de la base de dades que es crea automàticament
      POSTGRES_USER: esports      # Usuari per connectar-s'hi
      POSTGRES_PASSWORD: esports_pwd  # Contrasenya (en producció, usar secrets!)
    ports:
      - "5432:5432"               # Mapeja el port per accedir des de fora del contenidor
    volumes:
      - postgres_data:/var/lib/postgresql/data  # Persistència de dades entre reinicis

volumes:
  postgres_data:                  # Volum persistent per no perdre dades
```

### 2. Afegir dependències al `pom.xml` (10 min)

Substitueix la dependència d'H2 per PostgreSQL i afegeix Flyway:

```xml
<!-- Driver JDBC per PostgreSQL — permet que Java es connecti a la base de dades -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- Flyway — gestiona les migracions SQL de forma automàtica -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>

<!-- Mòdul extra de Flyway per dialecte PostgreSQL (requerit des de Flyway 10) -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```

> **Nota:** No esborris H2 del tot — mou-la a `<scope>test</scope>` per usar-la als tests unitaris.

### 3. Configurar `application.properties` (10 min)

Actualitza el fitxer amb la configuració de PostgreSQL i Flyway:

```properties
# --- PostgreSQL ---
spring.datasource.url=jdbc:postgresql://localhost:5432/esportspulse
spring.datasource.username=esports
spring.datasource.password=esports_pwd
spring.datasource.driver-class-name=org.postgresql.Driver

# --- JPA: només valida, no crea taules ---
spring.jpa.hibernate.ddl-auto=validate

# --- Flyway: actiu per defecte, busca migracions a db/migration ---
spring.flyway.enabled=true
# Ubicació dels fitxers SQL de migració dins src/main/resources
spring.flyway.locations=classpath:db/migration
```

### 4. Escriure la primera migració Flyway (30 min)

Crea el directori i el primer fitxer de migració:

```bash
# Crea la carpeta on Flyway busca les migracions
mkdir -p backend-java/src/main/resources/db/migration
```

Crea `V1__create_champions_table.sql`:

```sql
-- V1: Creació de la taula champions
-- Aquesta migració recrea l'esquema que fins ara Hibernate generava automàticament.
-- A partir d'ara, tot canvi d'esquema es fa via migracions Flyway.

-- Taula principal de campions de League of Legends
CREATE TABLE champions (
    -- Identificador únic generat automàticament per PostgreSQL
    id          BIGSERIAL PRIMARY KEY,
    -- Nom del campió (únic, no pot ser nul)
    name        VARCHAR(100) NOT NULL UNIQUE,
    -- Rol principal del campió (TOP, JUNGLE, MID, ADC, SUPPORT)
    role        VARCHAR(50)  NOT NULL,
    -- Descripció o lore del campió
    description TEXT,
    -- Percentatge de victòries (win rate) — pot ser nul si no tenim dades
    win_rate    DOUBLE PRECISION,
    -- Data de creació del registre
    created_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Comentari a la taula per documentar-la dins PostgreSQL
COMMENT ON TABLE champions IS 'Campions de League of Legends amb estadístiques bàsiques';
```

> **Diferències clau amb H2:**
> - `BIGSERIAL` en comptes de `BIGINT AUTO_INCREMENT`
> - `DOUBLE PRECISION` en comptes de `DOUBLE`
> - `TIMESTAMP` amb `DEFAULT NOW()` (funció de PostgreSQL)

### 5. Arrencar i verificar (20 min)

```bash
# Arrencar l'aplicació Spring Boot
mvn spring-boot:run
```

Als logs has de veure:
```
Flyway Community Edition ...
Successfully validated 1 migration
Current version of schema "public": << Empty Schema >>
Migrating schema "public" to version "1 - create champions table"
Successfully applied 1 migration to schema "public"
```

Verifica a PostgreSQL:

```bash
# Connectar-se a la base de dades i llistar les taules
docker exec -it esportspulse-postgres psql -U esports -d esportspulse -c "\dt"

# Veure l'esquema de la taula champions
docker exec -it esportspulse-postgres psql -U esports -d esportspulse -c "\d champions"

# Veure l'historial de migracions Flyway
docker exec -it esportspulse-postgres psql -U esports -d esportspulse \
  -c "SELECT version, description, success FROM flyway_schema_history;"
```

### 6. Commit (5 min)

```bash
git add .
git commit -m "feat(db): migrate from H2 to PostgreSQL with Flyway V1 migration"
```

---

## Checklist de Lliurament

- [ ] PostgreSQL funciona dins Docker i accepta connexions
- [ ] Dependències de PostgreSQL i Flyway afegides al `pom.xml`
- [ ] `application.properties` configurat per PostgreSQL (no H2)
- [ ] Migració `V1__create_champions_table.sql` creada i aplicada
- [ ] `spring.jpa.hibernate.ddl-auto=validate` (no `update`)
- [ ] L'aplicació arrenca sense errors amb `mvn spring-boot:run`
- [ ] La taula `flyway_schema_history` existeix i mostra la V1 aplicada
- [ ] Commit fet amb missatge descriptiu
