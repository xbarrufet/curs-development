# Setmana 07 — Dimecres: GitHub Actions — CI Pipeline Automàtic

## Objectiu del Dia

Configurar un pipeline de CI (Continuous Integration) amb GitHub Actions que executi els tests i el checkstyle automàticament a cada push. Al final del dia, cada cop que pugis codi a GitHub, els tests s'executaran sols i veuràs si el codi passa o falla sense haver d'executar res manualment.

---

## Teoria

### Què és CI (Continuous Integration)?

CI es la pràctica d'integrar el codi de tots els developers al repositori compartit **diverses vegades al dia**, amb verificació automàtica.

**Sense CI:**

```
Developer 1: "Al meu ordinador funciona" ✅
Developer 2: "Al meu també" ✅
Servidor de producció: Error 500 💥

Per què? Perquè les versions de Java, les dependències,
o les configuracions eren diferents.
```

**Amb CI:**

```
Developer 1: push → GitHub Actions executa tests → ✅ Passa
Developer 2: push → GitHub Actions executa tests → ❌ Falla!
  → Es veu immediatament quins tests fallen
  → Es corregeix ABANS de fer merge a main
```

**Regla d'or:** Si el CI no passa, el codi NO es pot fer merge a main. Mai.

---

### Anatomia d'un Workflow de GitHub Actions

Un workflow és un fitxer YAML a `.github/workflows/` que defineix **què** s'executa, **quan** i **on**.

```yaml
# .github/workflows/ci.yml
# Fitxer de configuració de CI — s'executa automàticament a cada push

# 1. NOM — Descriptiu, apareix a la pestanya "Actions" de GitHub
name: CI Pipeline

# 2. TRIGGERS — Quan s'executa aquest workflow?
on:
  push:
    branches: [ main ]          # A cada push a main
  pull_request:
    branches: [ main ]          # A cada PR que apunti a main

# 3. JOBS — Què s'executa? Pot tenir múltiples jobs en paral·lel
jobs:
  # Nom del job — pot ser qualsevol cosa descriptiva
  build-and-test:
    # 4. RUNNER — On s'executa? Ubuntu és l'estàndard per CI
    runs-on: ubuntu-latest

    # 5. STEPS — Passos seqüencials dins del job
    steps:
      # Pas 1: Descarregar el codi del repositori
      # 'uses' indica una "Action" pre-feta (com una llibreria)
      - name: Checkout del codi
        uses: actions/checkout@v4

      # Pas 2: Instal·lar Java 21
      - name: Configurar JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'    # Distribució OpenJDK gratuïta

      # Pas 3: Executar els tests amb Maven
      # 'run' executa una comanda de terminal
      - name: Executar tests
        run: mvn test --batch-mode
        # --batch-mode evita output interactiu (no hi ha terminal al CI)
```

---

### Cada Element en Detall

#### `name` — El nom del workflow

```yaml
# Apareix a la pestanya Actions de GitHub
# Usa un nom descriptiu que expliqui QUÈ fa
name: CI Pipeline              # ✅ Clar
name: Build                    # ❌ Massa vague — build de què?
```

#### `on` — Triggers (quan s'executa)

```yaml
# Opció 1: A cada push i PR a main
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

# Opció 2: A QUALSEVOL push (totes les branques)
on: push

# Opció 3: Programat (cron) — per exemple, cada nit a les 2AM
on:
  schedule:
    - cron: '0 2 * * *'        # Minuts Hores DiaDelMes Mes DiaDeLaSetmana

# Opció 4: Manual (botó a GitHub)
on:
  workflow_dispatch:           # Afegeix un botó "Run workflow" a la UI
```

#### `runs-on` — El sistema operatiu del runner

```yaml
runs-on: ubuntu-latest         # Linux (el més comú per CI)
runs-on: windows-latest        # Windows (si necessites .NET, per exemple)
runs-on: macos-latest          # macOS (si necessites Xcode)
```

#### `uses` vs `run` — Actions pre-fetes vs comandes

```yaml
steps:
  # 'uses' — Usa una Action del Marketplace de GitHub
  # Format: organització/nom-action@versió
  - name: Checkout
    uses: actions/checkout@v4           # Descarrega el codi del repo

  - name: Setup Java
    uses: actions/setup-java@v4         # Instal·la Java
    with:                                # Paràmetres de l'Action
      java-version: '21'
      distribution: 'temurin'

  # 'run' — Executa comandes de terminal directament
  - name: Compilar
    run: mvn compile --batch-mode

  # Múltiples comandes amb '|' (pipe YAML)
  - name: Tests i cobertura
    run: |
      mvn test --batch-mode
      echo "Tests completats!"
```

---

### Workflow Complet per EsportsPulse

```yaml
# .github/workflows/ci.yml
# Pipeline de CI complet per al projecte EsportsPulse
# Executa: compilació, tests, checkstyle i (opcionalment) cobertura

name: EsportsPulse CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  # JOB 1: Compilar i executar tests de Java
  java-build:
    name: Java Build & Test
    runs-on: ubuntu-latest

    steps:
      # Descarreguem el codi del repositori
      - name: Checkout del codi
        uses: actions/checkout@v4

      # Configurem Java 21 (Temurin és una distribució gratuïta d'OpenJDK)
      - name: Configurar JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      # Cache de Maven — evita descarregar dependències cada vegada
      # Estalvia 2-3 minuts per execució
      - name: Cache de dependències Maven
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-

      # Compilar el projecte (sense executar tests encara)
      - name: Compilar
        run: mvn compile --batch-mode
        working-directory: ./java     # Si el projecte Java està en un subdirectori

      # Executar tots els tests
      - name: Executar tests
        run: mvn test --batch-mode
        working-directory: ./java

      # Executar Checkstyle per validar l'estil del codi
      - name: Checkstyle
        run: mvn checkstyle:check --batch-mode
        working-directory: ./java

  # JOB 2: Lint de Python (s'executa en paral·lel amb java-build)
  python-lint:
    name: Python Lint & Test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout del codi
        uses: actions/checkout@v4

      # Configurem Python 3.12
      - name: Configurar Python 3.12
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      # Instal·lem dependències de Python
      - name: Instal·lar dependències
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install ruff pytest
        working-directory: ./python

      # Ruff — linter ultra-ràpid per Python (substitueix flake8, isort, etc.)
      - name: Lint amb ruff
        run: ruff check .
        working-directory: ./python

      # Executar tests de Python
      - name: Executar tests
        run: pytest --verbose
        working-directory: ./python
```

---

### Checkstyle — Estil de Codi Automàtic

Checkstyle valida que el codi Java segueix un estàndard d'estil (indentació, noms, imports).

#### Configuració al `pom.xml`

```xml
<!-- pom.xml — Secció de plugins -->
<build>
    <plugins>
        <!-- Plugin de Checkstyle — valida l'estil del codi automàticament -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-checkstyle-plugin</artifactId>
            <version>3.3.1</version>
            <configuration>
                <!-- Usem les regles de Google (estàndard de la indústria) -->
                <configLocation>google_checks.xml</configLocation>
                <!-- Si hi ha violacions, el build falla -->
                <failOnViolation>true</failOnViolation>
                <!-- Nivell de severitat mínim per fallar -->
                <violationSeverity>warning</violationSeverity>
            </configuration>
        </plugin>
    </plugins>
</build>
```

Per executar-lo:

```bash
# Verificar l'estil del codi
mvn checkstyle:check

# Si falla, veuràs missatges com:
# [ERROR] src/main/java/com/esportspulse/ChampionService.java:15:
#   Whitespace: 'if' is not followed by whitespace. [WhitespaceAround]
# [ERROR] src/main/java/com/esportspulse/ChampionService.java:23:
#   Naming: Name 'winrate' must match pattern '^[a-z][a-zA-Z0-9]*$' [LocalVariableName]
```

**Per què importa l'estil automàtic?**
- Elimina discussions inútils al code review ("hauries de posar espai aquí")
- Tothom escriu codi amb el mateix format
- El CI ho comprova automàticament, no cal que ho revisi un humà

---

### Entendre els Resultats del CI

Quan fas push, GitHub mostra l'estat del CI:

```
  Commit abc1234: "feat: add champion search"

  ✅ Java Build & Test — Passed (2m 15s)
     ✅ Checkout del codi
     ✅ Configurar JDK 21
     ✅ Compilar
     ✅ Executar tests (15 tests passed)
     ✅ Checkstyle

  ❌ Python Lint & Test — Failed (45s)
     ✅ Checkout del codi
     ✅ Configurar Python 3.12
     ❌ Lint amb ruff
        Error: champion_service.py:12: F841 local variable 'x' is assigned but never used
```

**Quan el CI falla:**
1. Clica al job que ha fallat
2. Llegeix el missatge d'error (sol ser clar)
3. Corregeix al teu ordinador
4. Fes commit i push — el CI es torna a executar automàticament

---

## Activitat

### Exercici 1: Crear el Pipeline CI des de Zero (45 min)

1. Crea el directori per al workflow:

```bash
# Crea el directori (ha d'estar exactament aquí, GitHub el busca aquí)
mkdir -p .github/workflows
```

2. Crea el fitxer `.github/workflows/ci.yml` amb el contingut del workflow complet (veure secció anterior).

3. Adapta els `working-directory` a l'estructura del teu projecte.

4. Fes commit i push:

```bash
git add .github/workflows/ci.yml
git commit -m "ci: add GitHub Actions pipeline for Java and Python"
git push origin main
```

5. Ves a la pestanya **Actions** del teu repositori a GitHub i observa l'execució.

### Exercici 2: Configurar Checkstyle (30 min)

1. Afegeix el plugin de Checkstyle al `pom.xml` (veure secció anterior).

2. Executa localment:

```bash
mvn checkstyle:check
```

3. Corregeix les violacions d'estil que trobi.

4. Torna a executar fins que passi sense errors.

### Exercici 3: Provocar un Error al CI (15 min)

1. Introdueix un error intencionat (per exemple, un test que falla).
2. Fes push i observa com el CI detecta l'error.
3. Corregeix l'error, fes push de nou, i verifica que el CI passa.
4. Reflexiona: **Quant de temps t'ha estalviat el CI** respecte a trobar l'error manualment?

### Exercici 4: Interpretar YAML (20 min)

Llegeix el seguent workflow i respon les preguntes:

```yaml
name: Mystery Workflow
on:
  schedule:
    - cron: '0 3 * * 1'
  workflow_dispatch:

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run security audit
        run: mvn dependency:tree | grep -i "vulnerability"
      - name: Check for secrets
        run: |
          if grep -r "API_KEY\|SECRET\|PASSWORD" src/ --include="*.java"; then
            echo "::error::Secrets trobats al codi font!"
            exit 1
          fi
```

**Preguntes:**
1. Quan s'executa aquest workflow? (pista: tradueix el cron)
2. Es pot executar manualment? Per què?
3. Què fa el segon step? Què passaria si trobés un secret al codi?
4. Per què l'exit code `1` és important?

---

## Checklist de Lliurament

- [ ] He creat `.github/workflows/ci.yml` amb el pipeline complet
- [ ] El CI s'executa automàticament quan faig push
- [ ] He configurat Checkstyle al `pom.xml` i passa localment
- [ ] He provocat un error al CI i l'he corregit
- [ ] He respost les preguntes de l'Exercici 4
- [ ] Entenc la diferència entre `uses` (Actions) i `run` (comandes)
- [ ] Commit amb missatge: `ci: add GitHub Actions pipeline with checkstyle`
