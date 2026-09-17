# Setmana 08 — Dijous: Cobertura de Codi amb JaCoCo i CI amb pytest + ruff

## Objectiu del Dia

Configurar JaCoCo per mesurar la cobertura de codi a Java i pytest-cov a Python. Actualitzar el pipeline de GitHub Actions perquè falli si la cobertura baixa del 70%. Afegir ruff com a linter de Python. Al final del dia, el teu CI protegirà la qualitat del codi automàticament.

---

## Teoria

### Què és la Cobertura de Codi?

La cobertura de codi mesura **quin percentatge del teu codi s'executa durant els tests**. Hi ha dos tipus principals:

**Cobertura de línies (line coverage):** quantes línies s'han executat.

**Cobertura de branques (branch coverage):** quantes decisions if/else s'han explorat.

```java
// Exemple: mètode amb una branca if/else
// Per tenir 100% de cobertura de branques, necessitem 2 tests
public String classifyWinRate(double winRate) {
    if (winRate >= 52.0) {           // Branca 1: winRate alt
        return "META";               // Línia coberta si testem winRate >= 52
    } else {                         // Branca 2: winRate normal
        return "STANDARD";           // Línia coberta si testem winRate < 52
    }
}
```

| Test                          | Line coverage | Branch coverage |
|-------------------------------|--------------|-----------------|
| Només `classifyWinRate(55.0)` | 75%          | 50% (falta else) |
| `classifyWinRate(55.0)` + `classifyWinRate(48.0)` | 100% | 100% |

---

### Què Mesura la Cobertura (i Què NO)

**El que SÍ mesura:**
- Quines línies de codi s'han executat durant els tests
- Quines branques (if/else, switch) s'han explorat
- Quins mètodes s'han cridat

**El que NO mesura:**
- Si el codi és **correcte**
- Si els tests tenen **asserts** adequats
- Si la **lògica de negoci** funciona bé

#### Anti-patró: 100% Cobertura, 0% Valor

```java
// PERILL: Aquest test té 100% cobertura del mètode
// però NO VERIFICA RES — no té cap assert!
@Test
void testWithNoAssertions() {
    // Executem el mètode (cobertura de línia: 100%)
    service.register(new ChampionRecord("jinx", "Marksman", 51.5));
    // ... i ja? No comprovem si s'ha guardat correctament!
    // JaCoCo dirà 100% coverage, però el test és inútil
}

// BÉ: Menys cobertura potser, però verifica comportament
@Test
void shouldSaveAndRetrieveChampion() {
    service.register(new ChampionRecord("jinx", "Marksman", 51.5));

    // VERIFICAR que realment s'ha guardat
    Optional<ChampionRecord> found = service.findById("jinx");
    assertTrue(found.isPresent());
    assertEquals("Marksman", found.get().role());
}
```

> **Regla:** La cobertura és un **indicador**, no un **objectiu**. 70-80% és un bon llindar. 100% sol indicar tests artificials.

---

### JaCoCo: Configuració a pom.xml

JaCoCo (Java Code Coverage) s'integra amb Maven com un plugin. Afegeix-lo al `pom.xml`:

```xml
<build>
    <plugins>
        <!-- JaCoCo: mesura la cobertura de codi durant els tests -->
        <!-- S'activa automàticament amb 'mvn verify' -->
        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <version>0.8.12</version>
            <executions>
                <!-- 1. prepare-agent: instrumenta el codi abans dels tests -->
                <!-- Afegeix un agent JVM que registra quines línies s'executen -->
                <execution>
                    <id>prepare-agent</id>
                    <goals>
                        <goal>prepare-agent</goal>
                    </goals>
                </execution>

                <!-- 2. report: genera l'informe HTML després dels tests -->
                <!-- Es pot obrir a target/site/jacoco/index.html -->
                <execution>
                    <id>report</id>
                    <phase>test</phase>
                    <goals>
                        <goal>report</goal>
                    </goals>
                </execution>

                <!-- 3. check: falla el build si la cobertura és insuficient -->
                <!-- Aquesta és la part que integrem al CI -->
                <execution>
                    <id>check</id>
                    <goals>
                        <goal>check</goal>
                    </goals>
                    <configuration>
                        <rules>
                            <rule>
                                <!-- Aplica a tot el bundle (projecte) -->
                                <element>BUNDLE</element>
                                <limits>
                                    <limit>
                                        <!-- Mínim 70% de cobertura de línies -->
                                        <!-- Si baixa del 70%, 'mvn verify' FALLA -->
                                        <counter>LINE</counter>
                                        <value>COVEREDRATIO</value>
                                        <minimum>0.70</minimum>
                                    </limit>
                                </limits>
                            </rule>
                        </rules>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

#### Executar i Veure l'Informe

```bash
# Compila, executa tests i verifica cobertura
# Si la cobertura < 70%, el build FALLA
mvn verify

# L'informe HTML es genera a:
# target/site/jacoco/index.html
# Obre'l al navegador per veure detalls per classe i mètode
open target/site/jacoco/index.html
```

**Exemple de sortida quan falla:**

```
[ERROR] Rule violated for bundle esportspulse-engine:
  lines covered ratio is 0.58, but expected minimum is 0.70
[ERROR] BUILD FAILURE
```

---

### pytest-cov: Cobertura en Python

Instal·la el plugin de cobertura per a pytest:

```bash
# Instal·lar pytest-cov (afegir també a requirements.txt)
pip install pytest-cov
```

Afegeix-lo a `requirements.txt`:

```
pytest>=8.0.0
pytest-cov>=5.0.0
```

#### Executar amb Cobertura

```bash
# Executar tests amb cobertura del mòdul esportspulse
# --cov: quin mòdul mesurar
# --cov-report=html: generar informe HTML
# --cov-report=term-missing: mostrar línies no cobertes a la terminal
# --cov-fail-under=70: fallar si la cobertura < 70%
pytest --cov=esportspulse \
       --cov-report=html \
       --cov-report=term-missing \
       --cov-fail-under=70
```

**Exemple de sortida:**

```
---------- coverage: platform linux, python 3.12 ----------
Name                                Stmts   Miss  Cover   Missing
-----------------------------------------------------------------
esportspulse/__init__.py                0      0   100%
esportspulse/champion_record.py        12      0   100%
esportspulse/champion_service.py       35      4    89%   42-45
esportspulse/in_memory_repository.py   20      2    90%   31-32
esportspulse/sqlite_repository.py      45     15    67%   58-72
-----------------------------------------------------------------
TOTAL                                 112     21    81%

FAIL Required test coverage of 70% reached. Total coverage: 81.25%
```

#### Configuració a pyproject.toml

```toml
# pyproject.toml
# Configuració centralitzada per a pytest i cobertura

[tool.pytest.ini_options]
# Opcions per defecte de pytest
# Així no cal recordar els flags cada cop
testpaths = ["tests"]
addopts = """
    -v
    --cov=esportspulse
    --cov-report=term-missing
    --cov-fail-under=70
"""
```

---

### ruff: Linter de Python (com Checkstyle per a Java)

ruff és un linter ultra-ràpid per a Python. Detecta errors d'estil, imports no usats, variables mortes i problemes comuns:

```bash
# Instal·lar ruff
pip install ruff
```

#### Configuració a pyproject.toml

```toml
# pyproject.toml

[tool.ruff]
# Versió de Python del projecte
target-version = "py312"

# Amplada màxima de línia
line-length = 100

[tool.ruff.lint]
# Regles activades:
# E = errors d'estil (PEP 8)
# F = errors lògics (pyflakes)
# I = imports desordenats
# N = convencions de nomenclatura
# UP = suggeriments de modernització
select = ["E", "F", "I", "N", "UP"]

# Regles ignorades:
# E501 = línia massa llarga (ja controlat per line-length)
ignore = ["E501"]
```

#### Executar ruff

```bash
# Comprovar errors (sense corregir)
ruff check .

# Corregir errors automàticament (imports, format)
ruff check . --fix

# Exemple de sortida:
# esportspulse/champion_service.py:3:1: F401 'os' imported but unused
# esportspulse/sqlite_repository.py:15:5: N806 variable 'Champions' should be lowercase
```

---

### Actualitzar GitHub Actions CI

Actualitzem el workflow de CI per incloure cobertura i linting:

```yaml
# .github/workflows/ci.yml
name: CI - EsportsPulse

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # Job 1: Java — compilar, testejar, verificar cobertura
  java-build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Configurar Java 21
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      # mvn verify executa: compile → test → JaCoCo check
      # Si la cobertura < 70%, el step FALLA i el CI es posa vermell
      - name: Build and verify with Maven
        run: mvn verify --batch-mode
        working-directory: backend-java

      # Pujar l'informe JaCoCo com a artefacte
      # Permet descarregar-lo des de la pàgina del workflow
      - name: Upload JaCoCo report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: jacoco-report
          path: backend-java/target/site/jacoco/

  # Job 2: Python — testejar, cobertura, linting
  python-build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Configurar Python 3.12
      - name: Set up Python 3.12
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      # Instal·lar dependències
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
        working-directory: ai-python

      # Executar ruff primer: si l'estil és incorrecte, no cal executar tests
      # Falla ràpid: millor saber que tens un import no usat ABANS de córrer tests
      - name: Lint with ruff
        run: ruff check .
        working-directory: ai-python

      # Executar tests amb cobertura
      # --cov-fail-under=70: falla si la cobertura < 70%
      - name: Test with pytest and coverage
        run: |
          pytest --cov=esportspulse \
                 --cov-report=html \
                 --cov-report=term-missing \
                 --cov-fail-under=70
        working-directory: ai-python

      # Pujar l'informe de cobertura Python
      - name: Upload Python coverage report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: python-coverage
          path: ai-python/htmlcov/
```

---

### Mutation Testing: Concepte (Manual)

El mutation testing verifica que els tests realment **detecten errors**. La idea:

1. **Canvia** una línia del codi (crea un "mutant")
2. **Executa** els tests
3. Si els tests **fallen** → el mutant ha estat **detectat** (bé)
4. Si els tests **passen** → el mutant ha **sobreviscut** (els tests són febles)

#### Exemples de Mutacions Manuals

```java
// ORIGINAL: filtra campions amb winRate >= mínim
public List<ChampionRecord> findByMinWinRate(double minRate) {
    return repository.findAll().stream()
        .filter(c -> c.winRate() >= minRate)  // Original: >=
        .toList();
}

// MUTANT 1: canviar >= per >
// Si els tests no fallen, no testegem el cas límit (winRate == minRate)
        .filter(c -> c.winRate() > minRate)   // Mutant: >

// MUTANT 2: eliminar el filtre
// Si els tests no fallen, no estem comprovant que el filtre funciona
    return repository.findAll();              // Mutant: retorna tot

// MUTANT 3: canviar el return
// Si els tests no fallen, no comprovem el resultat
    return List.of();                         // Mutant: retorna buit
```

#### Com fer-ho manualment

```bash
# 1. Canvia una línia del codi (>= per >)
# 2. Executa els tests
mvn test

# 3. Si els tests FALLEN → El mutant ha estat detectat (els tests són bons)
# 4. Si els tests PASSEN → Tens un forat! Afegeix un test pel cas límit

# 5. IMPORTANT: reverteix el canvi!
git checkout -- src/main/java/com/esportspulse/engine/ChampionManagementService.java
```

> **Consell:** Fes 3-5 mutacions manuals avui. Si algun mutant sobreviu, afegeix un test que el mati.

---

## Activitat

### Exercici: Configurar Cobertura + CI

1. **Configura JaCoCo a `pom.xml`:**
   - Afegeix el plugin amb els 3 goals: `prepare-agent`, `report`, `check`
   - Estableix mínim 70% de cobertura de línies
   - Executa `mvn verify` i obre l'informe HTML

2. **Configura pytest-cov:**
   - Afegeix `pytest-cov` a `requirements.txt`
   - Configura `pyproject.toml` amb les opcions per defecte
   - Executa `pytest --cov-fail-under=70`

3. **Configura ruff:**
   - Afegeix configuració a `pyproject.toml`
   - Executa `ruff check .` i corregeix els errors

4. **Actualitza `.github/workflows/ci.yml`:**
   - Java job: `mvn verify` (inclou JaCoCo check)
   - Python job: `ruff check .` + `pytest --cov-fail-under=70`
   - Puja els informes com a artefactes

5. **Mutation testing manual:**
   - Fes 3 mutacions al codi Java (canvia `>=` per `>`, elimina un `null` check, canvia un return)
   - Per a cada mutació: executa tests, anota si el mutant sobreviu
   - Si sobreviu, escriu un test que el mati
   - **Reverteix** totes les mutacions!

### Criteris d'Èxit

- `mvn verify` passa amb cobertura >= 70%
- `pytest --cov-fail-under=70` passa
- `ruff check .` no reporta errors
- CI actualitzat amb ambdós jobs
- Almenys 3 mutants provats manualment

---

## Checklist de Lliurament

- [ ] JaCoCo configurat a `pom.xml` amb mínim 70%
- [ ] `mvn verify` passa i genera informe HTML
- [ ] `pytest-cov` configurat a `pyproject.toml`
- [ ] `pytest --cov-fail-under=70` passa
- [ ] `ruff` configurat i sense errors
- [ ] `.github/workflows/ci.yml` actualitzat amb ambdós jobs
- [ ] Almenys 3 mutacions manuals provades i documentades
- [ ] Commit: `ci: add JaCoCo coverage check and Python linting with ruff`
