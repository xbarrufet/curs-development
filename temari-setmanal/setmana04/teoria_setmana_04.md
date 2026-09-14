# Setmana 4 - Teoria: Clean Code, Code Review i Git Professional

## 1. Introducció: El 80% del Teu Temps és Llegir Codi

Un developer junior espera escriure codi tot el dia. La realitat:

```
Distribució real del temps d'un developer:
┌────────────────────────────────────────────────────┐
│ ████████████████████████████████████████  80% Llegir│  Entendre codi existent, reviews, debugging
│ ████████  20% Escriure                             │  Codi nou, refactoring
└────────────────────────────────────────────────────┘
```

El teu primer dia a una empresa:
1. Et donen accés a un repositori de 200.000 línies.
2. Et donen un ticket Jira: "Bug: el winRate es mostra amb decimals incorrectes".
3. Has de trobar on es calcula el winRate, entendre la lògica, i corregir-ho.

Ningú t'explicarà el codi línia per línia. Has de saber llegir-lo sol.

---

## 2. Clean Code: Principis que Importen

### Noms Significatius

```java
// ❌ Què fa això?
public List<ChampionRecord> get(String s, int n) {
    return repo.findAll().stream()
        .filter(g -> g.name().contains(s))
        .limit(n)
        .toList();
}

// ✅ Ara s'entén sense llegir la implementació
public List<ChampionRecord> searchByName(String keyword, int maxResults) {
    return championRepository.findAll().stream()
        .filter(champion -> champion.name().contains(keyword))
        .limit(maxResults)
        .toList();
}
```

**Regla:** Si has de llegir el cos d'un mètode per entendre què fa, el nom és dolent.

**En Python és igual:**

```python
# ❌
def get(s, n):
    return [g for g in champions if s in g.name][:n]

# ✅
def search_by_name(keyword: str, max_results: int) -> list[ChampionRecord]:
    return [champion for champion in champions if keyword in champion.name][:max_results]
```

### Funcions Petites amb Una Sola Responsabilitat

```java
// ❌ Fa massa coses: valida, busca, transforma, i gestiona errors
public ChampionDTO getChampionWithAdjustedStats(String championId, double patchModifier) {
    if (championId == null || championId.isBlank()) {
        throw new IllegalArgumentException("championId is required");
    }
    if (patchModifier < -1 || patchModifier > 1) {
        throw new IllegalArgumentException("patchModifier must be between -1 and 1");
    }
    ChampionRecord champion = repository.findById(championId).orElse(null);
    if (champion == null) {
        throw new EntityNotFoundException("Champion " + championId + " not found");
    }
    BigDecimal adjustedWinRate = champion.winRate().multiply(
        BigDecimal.valueOf(1 + patchModifier)
    ).setScale(2, RoundingMode.HALF_UP);
    return new ChampionDTO(champion.championId(), champion.name(), adjustedWinRate, champion.gamesPlayed());
}

// ✅ Cada cosa al seu lloc
public ChampionDTO getChampionWithAdjustedStats(String championId, double patchModifier) {
    ChampionRecord champion = findChampionOrThrow(championId);
    BigDecimal adjustedWinRate = calculateAdjustedWinRate(champion.winRate(), patchModifier);
    return ChampionDTO.from(champion, adjustedWinRate);
}

private ChampionRecord findChampionOrThrow(String championId) {
    return repository.findById(championId)
        .orElseThrow(() -> new EntityNotFoundException("Champion " + championId + " not found"));
}

private BigDecimal calculateAdjustedWinRate(BigDecimal winRate, double patchModifier) {
    return winRate.multiply(BigDecimal.valueOf(1 + patchModifier))
        .setScale(2, RoundingMode.HALF_UP);
}
```

### Early Return (Evitar Nesting)

```java
// ❌ Piràmide de la mort
public void processChampion(ChampionRecord champion) {
    if (champion != null) {
        if (champion.winRate().compareTo(BigDecimal.ZERO) > 0) {
            if (champion.gamesPlayed() > 0) {
                // La lògica real està enterrada a 3 nivells
                repository.save(champion);
            }
        }
    }
}

// ✅ Guard clauses: surt aviat si les precondicions fallen
public void processChampion(ChampionRecord champion) {
    if (champion == null) return;
    if (champion.winRate().compareTo(BigDecimal.ZERO) <= 0) return;
    if (champion.gamesPlayed() <= 0) return;
    
    repository.save(champion);
}
```

---

## 3. Anti-Patrons en Codi Generat per IA

La IA (Claude, Copilot, ChatGPT) genera codi que **compila i funciona al happy path**. Però el happy path no és producció. Aquests són els anti-patrons que has de saber detectar:

### Anti-Patró 1: SQL Injection

```java
// La IA sovint genera queries concatenant strings
@Query("SELECT g FROM Champion g WHERE g.name = '" + name + "'")
List<ChampionRecord> findByName(String name);
```

**Per què és perillós:**

```
Input normal:    name = "Jinx"
Query:           SELECT g FROM Champion g WHERE g.name = 'Jinx'  ✅

Input maliciós:  name = "'; DROP TABLE champions; --"
Query:           SELECT g FROM Champion g WHERE g.name = ''; DROP TABLE champions; --'
                 → ELIMINA TOTA LA TAULA
```

**Solució:**

```java
@Query("SELECT g FROM Champion g WHERE g.name = :name")
List<ChampionRecord> findByName(@Param("name") String name);
// El paràmetre s'escapa automàticament → impossible injectar SQL
```

**En Python:**

```python
# ❌ SQL injection
cursor.execute(f"SELECT * FROM champions WHERE name = '{name}'")

# ✅ Paràmetres vinculats
cursor.execute("SELECT * FROM champions WHERE name = ?", (name,))
```

### Anti-Patró 2: Secrets al Codi Font

```java
// La IA genera secrets inline perquè no coneix el teu entorn
private static final String RIOT_API_KEY = "RGAPI-abc123def456ghi789";
private static final String DB_PASSWORD = "admin123";
```

**Per què és perillós:** El secret acaba a Git. Qualsevol que tingui accés al repo (inclòs si es fa públic per accident) té les teves claus.

**Solució:**

```java
// application.properties (exclòs de Git via .gitignore)
riot.api.key=${RIOT_API_KEY}

// Codi
@Value("${riot.api.key}")
private String riotApiKey;
```

```python
# .env (exclòs de Git)
RIOT_API_KEY=RGAPI-abc123def456ghi789

# Codi
import os
riot_api_key = os.environ["RIOT_API_KEY"]
```

### Anti-Patró 3: NullPointerException Amagat

```java
// La IA assumeix que tot existeix
ChampionRecord champion = repository.findById(championId);
return champion.name();  // ← Si el campió no existeix: NullPointerException
```

**Solució:**

```java
return repository.findById(championId)
    .orElseThrow(() -> new EntityNotFoundException("Champion " + championId + " not found"))
    .name();
```

**En Python:**

```python
# ❌
champion = repository.find_by_id(champion_id)
return champion.name  # AttributeError si champion és None

# ✅
champion = repository.find_by_id(champion_id)
if champion is None:
    raise ChampionNotFoundError(f"Champion {champion_id} not found")
return champion.name
```

### Anti-Patró 4: Tests que No Testegen Res

```java
@Test
void testGetChampion() {
    // Setup: diem al mock què ha de retornar
    when(mockService.findById("Jinx")).thenReturn(testChampion);
    
    // Act: cridem el mock directament
    ChampionRecord result = mockService.findById("Jinx");
    
    // Assert: verifiquem que el mock retorna el que li hem dit
    assertNotNull(result);  // Sempre serà no-null perquè ho hem configurat!
    assertEquals("Jinx", result.championId());  // Verifica el mock, no el codi!
}
```

**Per què és dolent:** No testeja el teu codi. Testeja que Mockito funciona. Si el teu `ChampionController` té un bug, aquest test seguirà passant.

**Solució: Testejar el codi real, mockejar les dependències**

```java
@Test
void testGetChampion() {
    // Mock de la dependència (repository)
    when(mockRepository.findById("Jinx")).thenReturn(Optional.of(testChampion));
    
    // Crida al codi REAL (controller o service)
    ChampionDTO result = championService.getChampion("Jinx");
    
    // Verifica que el codi real transforma correctament
    assertEquals("Jinx", result.championId());
    assertEquals("Jinx", result.name());
}
```

### Anti-Patró 5: Excepció Silenciada

```java
try {
    riotApiClient.fetchChampionData(championId);
} catch (Exception e) {
    // TODO: handle later
}
// El codi continua com si no hagués passat res
// En producció: dades incompletes sense cap error visible
```

**Solució mínima:**

```java
try {
    riotApiClient.fetchChampionData(championId);
} catch (RiotApiException e) {
    log.error("Failed to fetch champion data for {}: {}", championId, e.getMessage());
    throw new ChampionEnrichmentException("Could not enrich champion " + championId, e);
}
```

**En Python:**

```python
# ❌
try:
    riot_client.fetch_champion_data(champion_id)
except:  # Catch-all sense logging
    pass

# ✅
try:
    riot_client.fetch_champion_data(champion_id)
except RiotApiError as e:
    logger.error("Failed to fetch champion %s: %s", champion_id, e)
    raise ChampionEnrichmentError(f"Could not enrich {champion_id}") from e
```

---

## 4. Code Review: La Skill Professional Invisible

### Per Què Code Review?

A una empresa, **cap línia de codi arriba a producció sense review**. El procés:

```
Developer escriu codi
    │
    ▼
Crea Pull Request (PR) a GitHub
    │
    ▼
Reviewer(s) llegeixen el codi
    │
    ├── Comentaris: preguntes, suggeriments, problemes
    │
    ▼
Developer adreça els comentaris
    │
    ▼
Reviewer aprova ("LGTM" — Looks Good To Me)
    │
    ▼
Merge a main
```

### Què Buscar en una Code Review

```
1. CORRECCIÓ         Fa el que hauria de fer? Edge cases?
2. SEGURETAT         SQL injection? Secrets? Input no validat?
3. TESTS             Els canvis estan testejats? Tests edge cases?
4. MANTENIBILITAT    Ho entendrà algú dins 6 mesos?
5. RENDIMENT         Hi ha loops innecessaris? N+1 queries?
```

### Com Escriure Comentaris de Review

```
❌ MAL:
"Això està malament."
"No m'agrada."
"Per què has fet això?"

✅ BÉ:
"Això podria llançar NullPointerException si findById retorna empty. 
 Suggereixo usar orElseThrow() amb un missatge descriptiu."

"Veig que concatenes el name al query SQL. Això obre un vector 
 d'SQL injection — caldria usar paràmetres vinculats (:name)."

"Bon refactoring separant la validació! Una cosa: el test 
 testGetChampion() verifica el mock en lloc del controller — 
 mockeja el repository i crida el controller real."
```

**Format d'un bon comentari:**
1. **Observació:** Què veus (objectiu, no subjectiu)
2. **Impacte:** Per què és un problema (o per què és bo)
3. **Suggeriment:** Com millorar-ho (si és un problema)

### Saber Dir "LGTM"

No tot és un problema. Si un fitxer està bé, diga-ho:

```
✅ "LGTM — la separació entre DTO i Entity és neta."
✅ "Bon ús d'Optional aquí, consistent amb la resta del codebase."
```

Trobar problemes és important. Reconèixer codi bo també ho és.

---

## 5. Git Professional: Més Enllà de Commit i Push

### El Workflow de PR

```
main ──●──●──●──●──●──●──●──●──●──●──
              │                    ▲
              │  git checkout -b   │ merge PR
              ▼                    │
feature/xxx ──●──●──●─────────────●
              (commits de la feature)
```

### Rebase: Historial Net

Quan treballes en una branca, `main` pot avançar:

```
Abans del rebase:

main     ──A──B──C──D──E      (altres han mergejat coses)
                │
feature  ──────F──G──H         (el teu treball)

Després de git rebase main:

main     ──A──B──C──D──E
                         │
feature  ────────────────F'──G'──H'   (els teus commits "rereplicats" sobre main actual)
```

**Per què rebase i no merge?**

```
Merge crea un historial confús:
──A──B──C──D──E──────────M──   (M = merge commit)
        │               /
        └──F──G──H─────┘

Rebase crea un historial lineal:
──A──B──C──D──E──F'──G'──H'──  (net, fàcil de llegir)
```

### Resolució de Conflictes

Quan fas `git rebase main` i dos fitxers han canviat al mateix lloc:

```
<<<<<<< HEAD
    BigDecimal price;
=======
    BigDecimal price;
    String currency;
>>>>>>> feature/price-format
```

**Passos per resoldre:**
1. Obre el fitxer amb conflicte
2. Entén què vol cada costat (HEAD = main, feature = el teu)
3. Decideix el resultat final (potser vols els dos canvis!)
4. Elimina els marcadors `<<<<`, `====`, `>>>>`
5. `git add <fitxer>` → `git rebase --continue`
6. Executa els tests per verificar que no has trencat res

### Comandes Git Essencials

| Comanda | Quan usar-la |
|---------|-------------|
| `git rebase main` | Posar la teva branca al dia amb main |
| `git rebase -i HEAD~3` | Combinar 3 commits en 1 (squash) abans de PR |
| `git stash` / `git stash pop` | Guardar treball temporal quan has de canviar de branca |
| `git log --oneline --graph` | Visualitzar l'historial amb branques |
| `git bisect start` / `git bisect bad` / `git bisect good` | Trobar quin commit va introduir un bug |
| `git cherry-pick <sha>` | Portar un commit específic d'una altra branca |
| `git reflog` | Recuperar treball "perdut" (commits orfes, resets accidentals) |

### git bisect: Trobar un Bug en 10 Segons

Tens 100 commits i un test que falla. Quin commit va trencar-ho?

```bash
git bisect start
git bisect bad            # El commit actual és dolent
git bisect good v0.1      # El tag v0.1 era bo

# Git fa binary search! Salta a un commit del mig:
# "Bisecting: 50 revisions left to test"
# Executes el test:
mvn test -pl :module -Dtest=ChampionSearchTest

# Si falla:
git bisect bad
# Si passa:
git bisect good

# Després de ~7 passos (log2(100)):
# "abc123 is the first bad commit"
# Trobat! En 7 passos en lloc de 100.
```

**Per què és útil:** A la feina, algú diu "Ahir funcionava, avui no". `git bisect` troba el commit culpable en minuts.

---

## 6. GitHub Actions: CI Bàsic

### Què és CI (Continuous Integration)?

Cada cop que fas push, un servidor executa els teus tests automàticament:

```
Developer fa push
    │
    ▼
GitHub detecta el push
    │
    ▼
GitHub Actions arrenca una màquina Ubuntu
    │
    ├── Instal·la Java 21
    ├── Executa mvn test
    ├── Executa mvn checkstyle:check
    │
    ▼
Resultat: ✅ Pass o ❌ Fail
    │
    ▼
Badge al PR: "All checks passed" o "Checks failed"
```

### Anatomia d'un Workflow

```yaml
# .github/workflows/ci.yml

name: CI                          # Nom visible a GitHub

on: [push, pull_request]          # Quan s'executa

jobs:
  build-and-test:                 # Nom del job
    runs-on: ubuntu-latest        # Màquina on corre
    
    steps:
      - uses: actions/checkout@v4           # 1. Descarrega el codi
      
      - uses: actions/setup-java@v4         # 2. Instal·la Java
        with:
          java-version: '21'
          distribution: 'temurin'
      
      - run: mvn test                       # 3. Executa tests
      
      - run: mvn checkstyle:check           # 4. Verifica format
```

**Cada `run` és una comanda de terminal.** Si qualsevol retorna un codi d'error (exit code != 0), el workflow falla i el PR es marca amb ❌.

### Per Què Importa?

Sense CI:
```
Developer: "A mi em funciona" (en el seu portàtil)
Reviewer: "A mi no" (diferent versió de Java)
Producció: 💥 (diferent sistema operatiu)
```

Amb CI:
```
GitHub Actions: "El test testGetChampion falla a Ubuntu amb Java 21"
Developer: "Ah, tenia un path hardcodejat amb \\ en lloc de /"
→ Fix before merge
```

---

## 7. Checkstyle: Format Automàtic

### Per Què Formatatge Automàtic?

Sense estàndard:
```java
// Developer A
public void save(ChampionRecord champion){
    repo.save( champion );
}

// Developer B
public void save( ChampionRecord champion )
{
    repo.save(champion);
}
```

En una code review, acabes discutint espais en lloc de lògica.

Amb Checkstyle:
```java
// Tothom
public void save(ChampionRecord champion) {
    repo.save(champion);
}
// Checkstyle falla si no segueixes el format → no es pot mergejar
```

### Configuració Bàsica

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-checkstyle-plugin</artifactId>
    <version>3.3.1</version>
    <configuration>
        <configLocation>google_checks.xml</configLocation>
        <failOnViolation>true</failOnViolation>
    </configuration>
</plugin>
```

`google_checks.xml` és un estàndard raonable. Regles principals:
- Indentació de 2 espais (o 4, configurable)
- Imports ordenats
- Noms de variables en camelCase
- Llargada de línia màxima

---

## Resum

| Concepte | Key Takeaway |
|----------|--------------|
| **Llegir codi** | El 80% del temps; la skill que ningú ensenya |
| **Noms significatius** | Si has de llegir el cos per entendre el nom, el nom és dolent |
| **Early return** | Evita nesting amb guard clauses |
| **SQL injection** | Mai concatenar strings en queries; usar paràmetres vinculats |
| **Secrets** | Mai al codi; variables d'entorn o fitxers exclosos de Git |
| **NPE / None** | Usar `Optional` (Java) o comprovació explícita (Python) |
| **Tests reals** | Mockeja les dependències, testeja el teu codi |
| **Code review** | Observació + Impacte + Suggeriment; saber dir LGTM |
| **git rebase** | Historial net; sempre rebase sobre main abans de PR |
| **git bisect** | Binary search per trobar el commit que va trencar algo |
| **GitHub Actions** | Tests automàtics a cada push; mai mergejar sense CI |

**Objectiu setmana:** Saber llegir, criticar, i millorar codi d'altri (inclòs generat per IA). Dominar el workflow Git professional. Tenir CI funcional des del dia 1.
