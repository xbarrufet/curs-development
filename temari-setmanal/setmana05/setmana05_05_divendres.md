# Setmana 05 — Divendres: Tag v0.1, CI Badge, Code Review Final i PR

## Objectiu del Dia

Tancar la primera versio estable del projecte EsportsPulse: netejar el repositori, crear el tag v0.1, afegir el badge de CI al README, fer un exercici complet de code review, i practicar el cicle professional de PR. Al final del dia tindras un projecte publicable amb versionat semàntic.

---

## Teoria

### Neteja del Repositori — Checklist Pre-release

Abans de crear un tag (versió), el repositori ha d'estar impecable:

```
Checklist Pre-release:

✅ .gitignore complet (veure exemple a sota)
✅ Cap secret al codi (grep -r "API_KEY\|SECRET\|PASSWORD" src/)
✅ Tots els tests passen (mvn test && pytest)
✅ Checkstyle/ruff passen sense errors
✅ CI pipeline verd (GitHub Actions)
✅ README actualitzat amb instruccions de setup
✅ Cap fitxer innecessari (.DS_Store, .class, __pycache__)
```

#### `.gitignore` Complet per EsportsPulse

```gitignore
# === Java ===
# Fitxers compilats — es regeneren amb mvn compile
target/
*.class
*.jar
*.war

# === Python ===
# Bytecode i cache — es regeneren automàticament
__pycache__/
*.py[cod]
*.egg-info/
.venv/
venv/

# === IDEs ===
# Configuració local de cada developer — no compartir
.idea/
*.iml
.vscode/
.settings/
.project
.classpath

# === Secrets ===
# MAI pujar secrets al repositori
.env
.env.local
*.key
credentials/

# === Sistema Operatiu ===
# Fitxers que crea el SO automàticament
.DS_Store
Thumbs.db

# === Base de Dades ===
# La BD H2 és local, no la compartim
*.mv.db
*.trace.db
```

---

### Semantic Versioning (SemVer)

El versionat semàntic es l'estàndard de la indústria per numerar versions. Format: **MAJOR.MINOR.PATCH**

```
  v MAJOR . MINOR . PATCH
    │        │        │
    │        │        └── Correccions de bugs (backward compatible)
    │        │             Exemple: v1.0.1 → fix winRate decimal
    │        │
    │        └── Noves funcionalitats (backward compatible)
    │             Exemple: v1.1.0 → afegir cerca per rol
    │
    └── Canvis que TRENQUEN compatibilitat
         Exemple: v2.0.0 → canviar l'API de REST a GraphQL
```

**Exemples pràctics:**

```
v0.1.0 → Primera versió funcional (pre-release, l'API pot canviar)
v0.2.0 → Afegim persistència amb JPA
v0.2.1 → Fix: corregir NullPointerException al servei
v1.0.0 → Primera versió estable (l'API és fixa, backward compatible)
v1.1.0 → Afegim endpoint REST per cercar campions
v2.0.0 → Migrem de H2 a PostgreSQL (trenca la config existent)
```

**Per què el nostre primer tag es `v0.1.0`?** Perquè estem en desenvolupament (MAJOR = 0). Qualsevol cosa pot canviar. Quan MAJOR es 0, no hi ha garanties de compatibilitat.

> **Referència:** [semver.org](https://semver.org) — L'especificació completa en 5 minuts de lectura.

---

### Conventional Commits

Format estàndard per als missatges de commit. Permet generar changelogs automàticament.

```
Format: type(scope): description

Tipus obligatoris:
  feat:     Nova funcionalitat        → incrementa MINOR
  fix:      Correcció de bug          → incrementa PATCH

Tipus opcionals:
  docs:     Documentació
  test:     Afegir o corregir tests
  refactor: Canvi intern sense afectar funcionalitat
  chore:    Manteniment (dependències, CI, configs)
  ci:       Canvis al pipeline de CI
  style:    Format del codi (no afecta lògica)
  perf:     Millora de rendiment
```

**Exemples reals del projecte EsportsPulse:**

```bash
# Nova funcionalitat
git commit -m "feat(search): add champion search by role and winRate"

# Correcció de bug
git commit -m "fix(service): handle Optional.empty in findById"

# Refactoring
git commit -m "refactor(repository): extract query methods to interface"

# Tests
git commit -m "test(service): add edge cases for champion search"

# CI
git commit -m "ci: add checkstyle step to GitHub Actions pipeline"

# Documentació
git commit -m "docs: update README with setup instructions"

# BREAKING CHANGE (incrementa MAJOR)
git commit -m "feat(api)!: change REST response format to include metadata

BREAKING CHANGE: response now wraps data in {data: ..., meta: ...}"
```

---

### Crear un Tag amb Git

Un tag es una "etiqueta" permanent a un commit específic. Marca un punt de referencia (normalment una versió).

```bash
# Crear un tag anotat (recomanat — inclou missatge, data i autor)
git tag -a v0.1.0 -m "Primera versió funcional d'EsportsPulse

Inclou:
- Model de domini amb ChampionRecord (Java) i Champion dataclass (Python)
- Patró Repository amb implementacions InMemory i JPA
- Persistència amb H2 (JPA)
- Tests unitaris per a totes les capes
- Pipeline CI amb GitHub Actions
- Checkstyle i ruff configurats"

# Pujar el tag a GitHub (els tags no es pugen amb git push normal)
git push origin v0.1.0

# Veure tots els tags
git tag --list

# Veure informació d'un tag concret
git show v0.1.0

# Crear un tag lleuger (NO recomanat — sense missatge ni metadades)
git tag v0.1.0-light    # Evita això, sempre usa -a (anotat)
```

**On apareix el tag?** A GitHub, a la secció "Releases". Pots crear un Release a partir del tag amb notes de versió.

---

### CI Badge al README

Un badge es una imatge dinàmica que mostra l'estat actual del CI directament al README.

```markdown
<!-- README.md — Afegeix això al principi del fitxer -->
# EsportsPulse

<!-- Badge de CI — mostra l'estat de l'últim build -->
![CI Pipeline](https://github.com/EL_TEU_USUARI/esportspulse-engine/actions/workflows/ci.yml/badge.svg)

> Base de dades de campions de League of Legends.
> Projecte educatiu — Spring Boot (Java 21) + Python 3.12.
```

**Format del badge:**

```
https://github.com/{OWNER}/{REPO}/actions/workflows/{WORKFLOW_FILE}/badge.svg
```

El badge es actualitza automàticament:
- Verd (passing) quan l'últim CI ha passat
- Vermell (failing) quan l'últim CI ha fallat

---

### Estructura Professional d'un README

```markdown
# EsportsPulse

![CI](https://github.com/user/esportspulse-engine/actions/workflows/ci.yml/badge.svg)

> Breu descripció del projecte en 1-2 línies.

## Requisits

- Java 21 (Temurin)
- Maven 3.9+
- Python 3.12+
- Git 2.40+

## Setup

### Java

​```bash
cd java/
mvn compile
mvn test
​```

### Python

​```bash
cd python/
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
pytest
​```

## Estructura del Projecte

​```
esportspulse-engine/
├── java/
│   ├── src/main/java/com/esportspulse/
│   │   ├── domain/          # Records i entitats
│   │   ├── repository/      # Interfícies i implementacions
│   │   └── service/         # Lògica de negoci
│   └── src/test/java/       # Tests unitaris
├── python/
│   ├── domain/              # Dataclasses
│   ├── repository/          # Implementacions
│   └── tests/               # Tests amb pytest
└── .github/workflows/       # Pipeline CI
​```

## Versionat

Seguim [Semantic Versioning](https://semver.org/).
Seguim [Conventional Commits](https://www.conventionalcommits.org/).

## Llicència

MIT
```

---

## Activitat

### Exercici 1: Neteja del Repositori (20 min)

1. Executa la checklist pre-release:

```bash
# 1. Verifica que .gitignore és complet
cat .gitignore

# 2. Busca secrets al codi (no hauria de trobar res)
grep -r "API_KEY\|SECRET\|PASSWORD\|sk-" src/ --include="*.java" --include="*.py"

# 3. Executa tots els tests
cd java/ && mvn test --batch-mode
cd ../python/ && pytest --verbose

# 4. Executa checkstyle i ruff
cd java/ && mvn checkstyle:check --batch-mode
cd ../python/ && ruff check .

# 5. Verifica que no hi ha fitxers innecessaris
git status    # No hauria de mostrar .class, __pycache__, etc.
```

2. Corregeix qualsevol problema que trobis.

### Exercici 2: Crear Tag v0.1.0 (10 min)

```bash
# 1. Assegura't que tot està commitejat
git status

# 2. Crea el tag anotat amb un missatge descriptiu
git tag -a v0.1.0 -m "v0.1.0: Primera versió funcional

- Model de domini amb records/dataclasses
- Patró Repository (InMemory + JPA)
- Persistència H2
- Tests unitaris
- CI pipeline amb GitHub Actions
- Checkstyle i ruff configurats"

# 3. Puja el tag a GitHub
git push origin v0.1.0

# 4. Verifica a GitHub: pestanya Code → Releases/Tags
```

### Exercici 3: CI Badge i README Professional (20 min)

1. Afegeix el badge de CI al principi del README.md
2. Estructura el README seguint el model professional (veure teoria)
3. Fes commit i push:

```bash
git add README.md
git commit -m "docs: add CI badge and professional README structure"
git push origin main
```

4. Verifica que el badge apareix correctament a GitHub.

### Exercici 4: Code Review Complet (40 min)

Revisa el seguent Pull Request simulat. Identifica **tots** els problemes i escriu comentaris de review per a cadascun.

**Fitxer 1: ChampionController.java (NOU)**

```java
// Nou controller REST per a campions
@RestController
@RequestMapping("/api/champions")
public class ChampionController {

    @Autowired
    private ChampionManagementService service;

    @GetMapping("/{name}")
    public Champion getByName(@PathVariable String name) {
        // Retorna el campió directament, sense validar
        return service.findByName(name);
    }

    @PostMapping
    public void create(@RequestBody Champion champion) {
        // Password hardcoded per "autenticació"
        if (!champion.getAdminPassword().equals("admin123")) {
            throw new RuntimeException("Unauthorized");
        }
        service.create(champion);
    }
}
```

**Fitxer 2: ChampionControllerTest.java (NOU)**

```java
@SpringBootTest
class ChampionControllerTest {

    @MockBean
    private ChampionManagementService service;

    @Test
    void testGetByName() {
        // Configura el mock
        when(service.findByName("Jinx"))
            .thenReturn(new Champion("Jinx", "ADC", 51.2));

        // Crida al servei (no al controller!)
        var result = service.findByName("Jinx");

        // Comprova que el mock retorna el que li hem dit
        assertEquals("Jinx", result.getName());
    }
}
```

**Fitxer 3: .env (NOU, afegit al commit)**

```env
DB_PASSWORD=supersecret123
RIOT_API_KEY=rgapi-1234-5678-abcd
```

**Respon:**
1. Quants problemes has trobat? (Objectiu: almenys 6)
2. Escriu un comentari de review per a cadascun (Observació + Impacte + Suggeriment)
3. Classifica cada problema: Seguretat / Correcció / Tests / Mantenibilitat

### Exercici 5: Cicle Professional de PR (30 min)

Practica el cicle complet:

```bash
# 1. Crea una branca feature
git checkout -b feature/week5-cleanup

# 2. Fes els canvis (README, .gitignore, qualsevol millora)
# Fes commits petits amb Conventional Commits

# 3. Puja la branca
git push -u origin feature/week5-cleanup

# 4. Crea el PR a GitHub
gh pr create --title "chore: week 5 cleanup and CI badge" \
  --body "## Canvis
- Afegit badge de CI al README
- Actualitzat .gitignore
- Neteja general del repositori

## Tests
- mvn test: ✅
- pytest: ✅
- checkstyle: ✅"

# 5. Revisa el teu propi PR a GitHub (self-review)
#    - Mira el diff complet
#    - Hi ha algo que no hauria d'estar?

# 6. Merge el PR
gh pr merge --merge
```

---

### Reflexio: "5 Linies per a una Entrevista"

Si nomes poguessis dir 5 frases en una entrevista de feina sobre el que has après aquesta setmana, quines serien?

Exemple:

```
1. "Sé identificar anti-patrons de seguretat com SQL injection i secrets
    hardcoded, tant en codi humà com generat per IA."

2. "Tinc experiència amb Git professional: rebase, resolució de conflictes,
    interactive rebase per netejar l'historial."

3. "He configurat pipelines de CI amb GitHub Actions que executen tests,
    checkstyle i linting automàticament."

4. "Sé escriure especificacions de refactoring precises i fer code reviews
    constructius seguint la fórmula Observació-Impacte-Suggeriment."

5. "El meu projecte segueix Semantic Versioning, Conventional Commits,
    i té un README professional amb badge de CI."
```

Escriu les teves 5 frases personalitzades. Guarda-les — les faras servir al CV.

---

## Checklist de Lliurament

- [ ] El repositori passa la checklist pre-release (cap secret, tests verds, estil net)
- [ ] He creat el tag `v0.1.0` amb un missatge descriptiu
- [ ] El badge de CI apareix al README i mostra "passing"
- [ ] He completat el code review de l'Exercici 4 amb almenys 6 problemes trobats
- [ ] He practicat el cicle complet de PR (branca, push, PR, self-review, merge)
- [ ] He escrit les meves "5 linies per a una entrevista"
- [ ] Tots els commits segueixen Conventional Commits
- [ ] Commit final: `chore: complete week 5 — clean code, CI and code review`
