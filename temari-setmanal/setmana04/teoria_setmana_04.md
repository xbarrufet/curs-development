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
2. Et donen un ticket Jira: "Bug: el preu es mostra amb decimals incorrectes".
3. Has de trobar on es calcula el preu, entendre la lògica, i corregir-ho.

Ningú t'explicarà el codi línia per línia. Has de saber llegir-lo sol.

---

## 2. Clean Code: Principis que Importen

### Noms Significatius

```java
// ❌ Què fa això?
public List<GameRecord> get(String s, int n) {
    return repo.findAll().stream()
        .filter(g -> g.title().contains(s))
        .limit(n)
        .toList();
}

// ✅ Ara s'entén sense llegir la implementació
public List<GameRecord> searchByTitle(String keyword, int maxResults) {
    return gameRepository.findAll().stream()
        .filter(game -> game.title().contains(keyword))
        .limit(maxResults)
        .toList();
}
```

**Regla:** Si has de llegir el cos d'un mètode per entendre què fa, el nom és dolent.

**En Python és igual:**

```python
# ❌
def get(s, n):
    return [g for g in games if s in g.title][:n]

# ✅
def search_by_title(keyword: str, max_results: int) -> list[GameRecord]:
    return [game for game in games if keyword in game.title][:max_results]
```

### Funcions Petites amb Una Sola Responsabilitat

```java
// ❌ Fa massa coses: valida, busca, transforma, i gestiona errors
public GameDTO getGameWithDiscount(String appId, double discount) {
    if (appId == null || appId.isBlank()) {
        throw new IllegalArgumentException("appId is required");
    }
    if (discount < 0 || discount > 1) {
        throw new IllegalArgumentException("discount must be between 0 and 1");
    }
    GameRecord game = repository.findById(appId).orElse(null);
    if (game == null) {
        throw new EntityNotFoundException("Game " + appId + " not found");
    }
    BigDecimal discountedPrice = game.price().multiply(
        BigDecimal.valueOf(1 - discount)
    ).setScale(2, RoundingMode.HALF_UP);
    return new GameDTO(game.appId(), game.title(), discountedPrice, game.activePlayerCount());
}

// ✅ Cada cosa al seu lloc
public GameDTO getGameWithDiscount(String appId, double discount) {
    GameRecord game = findGameOrThrow(appId);
    BigDecimal discountedPrice = calculateDiscount(game.price(), discount);
    return GameDTO.from(game, discountedPrice);
}

private GameRecord findGameOrThrow(String appId) {
    return repository.findById(appId)
        .orElseThrow(() -> new EntityNotFoundException("Game " + appId + " not found"));
}

private BigDecimal calculateDiscount(BigDecimal price, double discount) {
    return price.multiply(BigDecimal.valueOf(1 - discount))
        .setScale(2, RoundingMode.HALF_UP);
}
```

### Early Return (Evitar Nesting)

```java
// ❌ Piràmide de la mort
public void processGame(GameRecord game) {
    if (game != null) {
        if (game.price().compareTo(BigDecimal.ZERO) > 0) {
            if (game.activePlayerCount() > 0) {
                // La lògica real està enterrada a 3 nivells
                repository.save(game);
            }
        }
    }
}

// ✅ Guard clauses: surt aviat si les precondicions fallen
public void processGame(GameRecord game) {
    if (game == null) return;
    if (game.price().compareTo(BigDecimal.ZERO) <= 0) return;
    if (game.activePlayerCount() <= 0) return;
    
    repository.save(game);
}
```

---

## 3. Anti-Patrons en Codi Generat per IA

La IA (Claude, Copilot, ChatGPT) genera codi que **compila i funciona al happy path**. Però el happy path no és producció. Aquests són els anti-patrons que has de saber detectar:

### Anti-Patró 1: SQL Injection

```java
// La IA sovint genera queries concatenant strings
@Query("SELECT g FROM Game g WHERE g.title = '" + title + "'")
List<GameRecord> findByTitle(String title);
```

**Per què és perillós:**

```
Input normal:    title = "League of Legends"
Query:           SELECT g FROM Game g WHERE g.title = 'League of Legends'  ✅

Input maliciós:  title = "'; DROP TABLE games; --"
Query:           SELECT g FROM Game g WHERE g.title = ''; DROP TABLE games; --'
                 → ELIMINA TOTA LA TAULA
```

**Solució:**

```java
@Query("SELECT g FROM Game g WHERE g.title = :title")
List<GameRecord> findByTitle(@Param("title") String title);
// El paràmetre s'escapa automàticament → impossible injectar SQL
```

**En Python:**

```python
# ❌ SQL injection
cursor.execute(f"SELECT * FROM games WHERE title = '{title}'")

# ✅ Paràmetres vinculats
cursor.execute("SELECT * FROM games WHERE title = ?", (title,))
```

### Anti-Patró 2: Secrets al Codi Font

```java
// La IA genera secrets inline perquè no coneix el teu entorn
private static final String STEAM_API_KEY = "sk-abc123def456ghi789";
private static final String DB_PASSWORD = "admin123";
```

**Per què és perillós:** El secret acaba a Git. Qualsevol que tingui accés al repo (inclòs si es fa públic per accident) té les teves claus.

**Solució:**

```java
// application.properties (exclòs de Git via .gitignore)
steam.api.key=${STEAM_API_KEY}

// Codi
@Value("${steam.api.key}")
private String steamApiKey;
```

```python
# .env (exclòs de Git)
STEAM_API_KEY=sk-abc123def456ghi789

# Codi
import os
steam_api_key = os.environ["STEAM_API_KEY"]
```

### Anti-Patró 3: NullPointerException Amagat

```java
// La IA assumeix que tot existeix
GameRecord game = repository.findById(appId);
return game.title();  // ← Si el joc no existeix: NullPointerException
```

**Solució:**

```java
return repository.findById(appId)
    .orElseThrow(() -> new EntityNotFoundException("Game " + appId + " not found"))
    .title();
```

**En Python:**

```python
# ❌
game = repository.find_by_id(app_id)
return game.title  # AttributeError si game és None

# ✅
game = repository.find_by_id(app_id)
if game is None:
    raise GameNotFoundError(f"Game {app_id} not found")
return game.title
```

### Anti-Patró 4: Tests que No Testegen Res

```java
@Test
void testGetGame() {
    // Setup: diem al mock què ha de retornar
    when(mockService.findById("APP-1")).thenReturn(testGame);
    
    // Act: cridem el mock directament
    GameRecord result = mockService.findById("APP-1");
    
    // Assert: verifiquem que el mock retorna el que li hem dit
    assertNotNull(result);  // Sempre serà no-null perquè ho hem configurat!
    assertEquals("APP-1", result.appId());  // Verifica el mock, no el codi!
}
```

**Per què és dolent:** No testeja el teu codi. Testeja que Mockito funciona. Si el teu `GameController` té un bug, aquest test seguirà passant.

**Solució: Testejar el codi real, mockejar les dependències**

```java
@Test
void testGetGame() {
    // Mock de la dependència (repository)
    when(mockRepository.findById("APP-1")).thenReturn(Optional.of(testGame));
    
    // Crida al codi REAL (controller o service)
    GameDTO result = gameService.getGame("APP-1");
    
    // Verifica que el codi real transforma correctament
    assertEquals("APP-1", result.appId());
    assertEquals("League of Legends", result.title());
}
```

### Anti-Patró 5: Excepció Silenciada

```java
try {
    steamClient.fetchGameData(appId);
} catch (Exception e) {
    // TODO: handle later
}
// El codi continua com si no hagués passat res
// En producció: dades incompletes sense cap error visible
```

**Solució mínima:**

```java
try {
    steamClient.fetchGameData(appId);
} catch (SteamApiException e) {
    log.error("Failed to fetch game data for {}: {}", appId, e.getMessage());
    throw new GameEnrichmentException("Could not enrich game " + appId, e);
}
```

**En Python:**

```python
# ❌
try:
    steam_client.fetch_game_data(app_id)
except:  # Catch-all sense logging
    pass

# ✅
try:
    steam_client.fetch_game_data(app_id)
except SteamApiError as e:
    logger.error("Failed to fetch game %s: %s", app_id, e)
    raise GameEnrichmentError(f"Could not enrich {app_id}") from e
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

"Veig que concatenes el title al query SQL. Això obre un vector 
 d'SQL injection — caldria usar paràmetres vinculats (:title)."

"Bon refactoring separant la validació! Una cosa: el test 
 testGetGame() verifica el mock en lloc del controller — 
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
mvn test -pl :module -Dtest=GameSearchTest

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
GitHub Actions: "El test testGetGame falla a Ubuntu amb Java 21"
Developer: "Ah, tenia un path hardcodejat amb \\ en lloc de /"
→ Fix before merge
```

---

## 7. Checkstyle: Format Automàtic

### Per Què Formatatge Automàtic?

Sense estàndard:
```java
// Developer A
public void save(GameRecord game){
    repo.save( game );
}

// Developer B
public void save( GameRecord game )
{
    repo.save(game);
}
```

En una code review, acabes discutint espais en lloc de lògica.

Amb Checkstyle:
```java
// Tothom
public void save(GameRecord game) {
    repo.save(game);
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
