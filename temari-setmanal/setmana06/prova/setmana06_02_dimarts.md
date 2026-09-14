# Setmana 06 — Dimarts: Spring Data JPA amb H2 — Entitats, @Repository i Consultes Derivades

## Objectiu del Dia

Connectar l'aplicacio EsportsPulse a una base de dades H2 mitjancant Spring Data JPA. Al final del dia, els campions es desaran en una taula SQL real (encara que en memoria), podras veure'ls a la consola H2 i tindras consultes derivades funcionals com `findByNameContaining`.

---

## Teoria

### De la RAM a la Base de Dades

Ahir vam crear `InMemoryChampionRepository` — funciona, pero te un problema fonamental:

```
┌─────────────────────────────────────────────┐
│  Problema: les dades viuen a la RAM          │
│                                              │
│  1. Arrenques l'aplicacio                    │
│  2. Registres 50 campions                    │
│  3. L'aplicacio es reinicia (deploy, error)  │
│  4. ❌ Tots els campions han desaparegut     │
└─────────────────────────────────────────────┘
```

**Solucio:** una base de dades. Les dades sobreviuen als reinicis perque es guarden a disc.

### Que es JPA?

**JPA** (Java Persistence API) es l'estandard de Java per mapejar objectes Java a taules SQL:

```
Java                          SQL
─────                         ─────
Classe (@Entity)         →    Taula (TABLE)
Camp (@Column)           →    Columna (COLUMN)
Instancia                →    Fila (ROW)
championId (@Id)         →    PRIMARY KEY
```

JPA no es una llibreria — es una **especificacio** (un contracte). **Hibernate** es la implementacio mes usada, i Spring Data JPA la integra automaticament.

### H2: Base de Dades per a Desenvolupament

H2 es una base de dades SQL escrita en Java que pot funcionar en dos modes:

| Mode | Descripcio | Us |
|---|---|---|
| **In-memory** | Les dades viuen a RAM, es perden al reiniciar | Tests, desenvolupament |
| **File-based** | Les dades es guarden a disc | Desenvolupament persistent |

**Avantatge d'H2:** zero configuracio. No cal instal·lar res — es una dependencia Maven.

### Pas 1: Afegir Dependencies

Al `pom.xml`, afegeix Spring Data JPA i H2:

```xml
<!-- pom.xml — seccio <dependencies> -->

<!-- Spring Data JPA: proporciona repositoris automatics i gestio d'entitats -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- H2: base de dades SQL en memoria per a desenvolupament -->
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope> <!-- Nomes necessaria en temps d'execucio, no de compilacio -->
</dependency>
```

### Pas 2: Configurar application.properties

```properties
# src/main/resources/application.properties

# --- Configuracio H2 ---
# URL de connexio: mem = en memoria, esportspulse = nom de la BD
spring.datasource.url=jdbc:h2:mem:esportspulse

# Driver JDBC per H2
spring.datasource.driver-class-name=org.h2.Driver

# Credencials (per defecte, H2 accepta qualsevol)
spring.datasource.username=sa
spring.datasource.password=

# --- Configuracio JPA ---
# update: JPA crea/modifica taules automaticament segons les entitats
# IMPORTANT: nomes per desenvolupament! En produccio usariem "validate" o "none"
spring.jpa.hibernate.ddl-auto=update

# Mostra les queries SQL al log — util per aprendre, desactivar en produccio
spring.jpa.show-sql=true

# Format les queries SQL per llegibilitat
spring.jpa.properties.hibernate.format_sql=true

# --- Consola H2 ---
# Activa la interficie web per veure les taules i executar SQL manualment
spring.h2.console.enabled=true

# URL de la consola: http://localhost:8080/h2-console
spring.h2.console.path=/h2-console
```

### Pas 3: Convertir ChampionRecord a @Entity

JPA necessita classes **mutables** amb constructor sense arguments. Aixo entra en conflicte amb els `record` de Java (immutables). Tenim dues opcions:

**Opcio A: Classe JPA separada (recomanada)**

```java
package com.esportspulse.engine.entity;

import jakarta.persistence.*;

/**
 * Entitat JPA que representa un campió a la base de dades.
 * Es una classe mutable perque JPA ho requereix (necessita constructor buit
 * i setters per hidratar objectes des de la BD).
 *
 * NOTA: Mantenim ChampionRecord com a objecte de domini immutable.
 * Aquesta classe es nomes per persistencia — separacio de responsabilitats.
 */
@Entity // Indica a JPA que aquesta classe correspon a una taula SQL
@Table(name = "champions") // Nom explicit de la taula (per defecte seria "champion_entity")
public class ChampionEntity {

    @Id // Clau primaria — identifica unicament cada fila
    @Column(name = "champion_id", nullable = false) // Nom de columna explicit
    private String championId;

    @Column(nullable = false) // NOT NULL a la BD — un campió SEMPRE te nom
    private String name;

    @Column(name = "games_played") // snake_case a SQL, camelCase a Java
    private int gamesPlayed;

    @Column(name = "win_rate")
    private double winRate;

    @Version // Control de concurrencia optimista
    // Si dos usuaris modifiquen el mateix campió, JPA detecta el conflicte
    private Long version;

    // Constructor buit OBLIGATORI per JPA
    // JPA crea instancies buides i despres omple els camps via reflexio
    protected ChampionEntity() {
    }

    // Constructor per crear noves instancies des del codi
    public ChampionEntity(String championId, String name, int gamesPlayed, double winRate) {
        this.championId = championId;
        this.name = name;
        this.gamesPlayed = gamesPlayed;
        this.winRate = winRate;
    }

    // --- Conversio entre domini i entitat ---

    /**
     * Converteix un objecte de domini (immutable) a entitat JPA (mutable).
     * Patro "factory method" — centralitza la conversio en un sol lloc.
     */
    public static ChampionEntity fromDomain(ChampionRecord record) {
        return new ChampionEntity(
            record.championId(),
            record.name(),
            record.gamesPlayed(),
            record.winRate()
        );
    }

    /**
     * Converteix l'entitat JPA a objecte de domini.
     * El servei treballa amb ChampionRecord, mai amb ChampionEntity.
     */
    public ChampionRecord toDomain() {
        return new ChampionRecord(championId, name, gamesPlayed, winRate);
    }

    // Getters i setters (necessaris per JPA)
    public String getChampionId() { return championId; }
    public void setChampionId(String championId) { this.championId = championId; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public int getGamesPlayed() { return gamesPlayed; }
    public void setGamesPlayed(int gamesPlayed) { this.gamesPlayed = gamesPlayed; }
    public double getWinRate() { return winRate; }
    public void setWinRate(double winRate) { this.winRate = winRate; }
}
```

### Pas 4: Crear el Repositori JPA

Aqui es on Spring Data JPA brilla — **no has d'escriure cap implementacio**:

```java
package com.esportspulse.engine.repository;

import com.esportspulse.engine.entity.ChampionEntity;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;

/**
 * Repositori JPA per a campions.
 * Spring Data genera AUTOMATICAMENT la implementacio en temps d'execucio.
 * Nomes cal definir la interficie — Spring crea les queries SQL per nosaltres.
 *
 * JpaRepository<ChampionEntity, String>:
 *   - ChampionEntity: el tipus d'entitat
 *   - String: el tipus de la clau primaria (@Id)
 */
@Repository
public interface ChampionJpaRepository extends JpaRepository<ChampionEntity, String> {

    // --- Metodes heretats de JpaRepository (no cal escriure'ls) ---
    // save(entity)        → INSERT o UPDATE
    // findById(id)        → SELECT WHERE champion_id = ?
    // findAll()           → SELECT * FROM champions
    // deleteById(id)      → DELETE WHERE champion_id = ?
    // count()             → SELECT COUNT(*) FROM champions

    // --- Consultes derivades: Spring genera SQL a partir del nom del metode ---

    /**
     * Cerca campions que continguin el text donat al nom.
     * Spring genera: SELECT * FROM champions WHERE name LIKE '%text%'
     * Exemple: findByNameContaining("Ah") → retorna Ahri
     */
    List<ChampionEntity> findByNameContaining(String text);

    /**
     * Cerca campions amb mes de X partides jugades.
     * Spring genera: SELECT * FROM champions WHERE games_played > ?
     */
    List<ChampionEntity> findByGamesPlayedGreaterThan(int minGames);

    /**
     * Cerca campions amb win rate entre dos valors.
     * Spring genera: SELECT * FROM champions WHERE win_rate BETWEEN ? AND ?
     */
    List<ChampionEntity> findByWinRateBetween(double min, double max);

    /**
     * Cerca campions ordenats per win rate descendent.
     * Spring genera: SELECT * FROM champions ORDER BY win_rate DESC
     */
    List<ChampionEntity> findAllByOrderByWinRateDesc();
}
```

### Com Funcionen les Consultes Derivades?

Spring Data analitza el nom del metode i genera la query SQL:

```
findByNameContaining("Ah")
│    │    │
│    │    └─ LIKE '%Ah%'
│    └────── WHERE name
└─────────── SELECT * FROM champions

findByGamesPlayedGreaterThan(1000)
│    │           │
│    │           └─ > 1000
│    └───────────── WHERE games_played
└────────────────── SELECT * FROM champions
```

**Paraules clau disponibles:**

| Paraula | SQL | Exemple |
|---|---|---|
| `Containing` | `LIKE '%x%'` | `findByNameContaining("ri")` |
| `GreaterThan` | `> x` | `findByGamesPlayedGreaterThan(100)` |
| `LessThan` | `< x` | `findByWinRateLessThan(50.0)` |
| `Between` | `BETWEEN x AND y` | `findByWinRateBetween(45.0, 55.0)` |
| `OrderBy...Desc` | `ORDER BY col DESC` | `findAllByOrderByWinRateDesc()` |
| `And` / `Or` | `AND` / `OR` | `findByNameAndGamesPlayedGreaterThan(...)` |

### Adaptador: Connectant JPA amb la Nostra Interficie

Per mantenir la separacio, creem un adaptador que implementa la nostra interficie `ChampionRepository` i delega al JPA:

```java
package com.esportspulse.engine.repository;

import com.esportspulse.engine.domain.ChampionRecord;
import com.esportspulse.engine.entity.ChampionEntity;
import org.springframework.context.annotation.Primary;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.Optional;

/**
 * Adaptador que connecta ChampionRepository (interficie de domini)
 * amb ChampionJpaRepository (interficie de Spring Data).
 * Patro Adapter: tradueix entre dos mons (domini i persistencia).
 *
 * @Primary: quan Spring troba dues implementacions de ChampionRepository,
 * tria aquesta per defecte (per sobre de InMemoryChampionRepository).
 */
@Component
@Primary // Prioritat sobre InMemoryChampionRepository
public class JpaChampionRepositoryAdapter implements ChampionRepository {

    private final ChampionJpaRepository jpaRepository;

    public JpaChampionRepositoryAdapter(ChampionJpaRepository jpaRepository) {
        this.jpaRepository = jpaRepository;
    }

    @Override
    public void save(ChampionRecord champion) {
        // Convertim domini → entitat JPA, i demanem a JPA que la desi
        jpaRepository.save(ChampionEntity.fromDomain(champion));
    }

    @Override
    public Optional<ChampionRecord> findById(String championId) {
        // Convertim entitat JPA → domini en retornar
        return jpaRepository.findById(championId)
            .map(ChampionEntity::toDomain); // .map() transforma el contingut de l'Optional
    }

    @Override
    public List<ChampionRecord> findAll() {
        // Stream: convertim cada entitat JPA a objecte de domini
        return jpaRepository.findAll().stream()
            .map(ChampionEntity::toDomain)
            .toList();
    }

    @Override
    public void delete(String championId) {
        jpaRepository.deleteById(championId);
    }
}
```

### La Consola H2

Un cop l'aplicacio esta arrencada, pots accedir a la consola H2:

```
URL:       http://localhost:8080/h2-console
JDBC URL:  jdbc:h2:mem:esportspulse
Username:  sa
Password:  (buit)
```

Des d'aqui pots:
- Veure les taules creades automaticament per JPA
- Executar queries SQL manualment
- Verificar que les dades s'han desat correctament

---

## Activitat

### Exercici: Integra JPA a EsportsPulse

**Durada estimada:** 90 minuts

#### Pas 1: Dependencies (10 min)

1. Afegeix `spring-boot-starter-data-jpa` i `h2` al `pom.xml`
2. Executa `mvn compile` per verificar que les dependencies es descarreguen

#### Pas 2: Configuracio (10 min)

1. Crea/modifica `application.properties` amb la configuracio H2
2. Activa `spring.jpa.show-sql=true` per veure les queries

#### Pas 3: Entitat (20 min)

1. Crea `ChampionEntity` amb les anotacions JPA
2. Implementa `fromDomain()` i `toDomain()`
3. Afegeix `@Version` per control de concurrencia

#### Pas 4: Repositori JPA (15 min)

1. Crea `ChampionJpaRepository extends JpaRepository`
2. Afegeix 3 consultes derivades:
   - `findByNameContaining`
   - `findByGamesPlayedGreaterThan`
   - `findByWinRateBetween`

#### Pas 5: Adaptador (15 min)

1. Crea `JpaChampionRepositoryAdapter` que implementa `ChampionRepository`
2. Marca amb `@Primary`
3. Verifica que `ChampionManagementService` no canvia ni una linia

#### Pas 6: Verifica amb H2 (20 min)

1. Arrenca l'aplicacio: `mvn spring-boot:run`
2. Obre `http://localhost:8080/h2-console`
3. Connecta amb `jdbc:h2:mem:esportspulse`
4. Executa: `SELECT * FROM champions;`
5. Insereix un campió manualment amb SQL i verifica que l'API el retorna

---

## Checklist de Lliurament

- [ ] Dependencies JPA i H2 afegides al `pom.xml`
- [ ] `application.properties` configurat amb H2 i consola activada
- [ ] `ChampionEntity` creada amb `@Entity`, `@Id`, `@Column`, `@Version`
- [ ] Metodes `fromDomain()` i `toDomain()` funcionals
- [ ] `ChampionJpaRepository` amb minim 3 consultes derivades
- [ ] `JpaChampionRepositoryAdapter` implementa `ChampionRepository` amb `@Primary`
- [ ] `ChampionManagementService` NO ha canviat (comprova amb `git diff`)
- [ ] Aplicacio arrenca sense errors: `mvn spring-boot:run`
- [ ] Consola H2 accessible i mostra la taula `champions`
- [ ] `mvn test` verd
