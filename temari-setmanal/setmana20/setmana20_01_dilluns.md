# Setmana 20 — Dilluns: Audit dels Workflows CI: Netejar i Unificar

## Objectiu del Dia

Revisar tots els workflows de GitHub Actions acumulats durant el curs (S5, S7, S13, S16), eliminar redundàncies, consolidar en workflows ben estructurats i afegir caching de dependències. Al final del dia, el CI és més ràpid, consistent i mantenible.

---

## Teoria

### Per Què Cal Auditar el CI?

Durant 19 setmanes hem anat afegint workflows a mesura que els necessitàvem. El resultat típic:

```
.github/workflows/
├── build.yml           ← S5: primer CI bàsic
├── test-java.yml       ← S7: tests Java separats
├── test-python.yml     ← S7: tests Python separats
├── lint.yml            ← S13: linting
├── docker-build.yml    ← S16: build Docker
└── deploy.yml          ← S16: deploy
```

**Problemes comuns:**
- **Duplicació:** Cada workflow fa `checkout` + instal·lar dependències per separat
- **Inconsistència:** Un workflow usa Java 21, un altre no especifica versió
- **Lentitud:** Sense cache, cada execució descarrega les mateixes dependències
- **Confusió:** 6 workflows difícils de saber quin fa què i quan s'executa

### Principis d'un Bon CI

1. **DRY (Don't Repeat Yourself):** Usa `jobs` dins un sol workflow en comptes de múltiples workflows que fan coses similars
2. **Caching:** Guarda dependències (Maven, pip) entre execucions
3. **Paral·lelisme:** Jobs independents s'executen en paral·lel
4. **Fail fast:** Si un job falla, no cal esperar els altres

### Caching a GitHub Actions

Cada cop que el CI s'executa, descarrega totes les dependències Maven (~200MB) i pip (~100MB). Amb cache:

```yaml
# Cache de Maven: guarda el directori ~/.m2/repository entre execucions
- uses: actions/cache@v4
  with:
    # Directori a guardar
    path: ~/.m2/repository
    # Clau de cache: canvia quan canvia el pom.xml
    key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
    # Si no troba la clau exacta, usa un prefix (cache parcial)
    restore-keys: |
      ${{ runner.os }}-maven-

# Cache de pip: guarda els paquets Python
- uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-
```

**Impacte típic:**
- Sense cache: 3-5 minuts per instal·lar dependències
- Amb cache: 10-30 segons

### Estructura Recomanada

En comptes de 6 workflows petits, consolidar en 2-3 workflows clars:

```
.github/workflows/
├── ci.yml              ← Tests + lint + build (s'executa a cada push/PR)
└── deploy.yml          ← Deploy (només quan es merja a main)
```

> **Lectura recomanada (opcional, no bloquejant):**
> - [GitHub Actions Caching](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows)
> - [GitHub Actions Best Practices](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)

---

## Activitat

### 1. Inventariar els workflows actuals (15 min)

Revisa tots els fitxers a `.github/workflows/`:

```bash
# Llistar tots els workflows
ls -la .github/workflows/

# Per cada workflow, veure quan s'executa (triggers)
grep -l "on:" .github/workflows/*.yml | xargs -I {} sh -c 'echo "--- {} ---"; head -20 {}'
```

Crea una taula d'inventari:

| Fitxer | Trigger | Què fa | Temps mitjà | Problemes |
|--------|---------|--------|-------------|-----------|
| build.yml | push | Compila Java | ~4min | No cache, duplicat amb test |
| test-java.yml | PR | Tests JUnit | ~5min | No cache |
| test-python.yml | PR | Tests pytest | ~3min | No cache |
| lint.yml | push | Linting | ~2min | Poc útil sol |
| docker-build.yml | PR | Build imatge | ~6min | No cache layers |

### 2. Consolidar en un workflow únic de CI (45 min)

Crea `.github/workflows/ci.yml`:

```yaml
# Workflow de CI unificat: tests, lint i build per a Java i Python
# S'executa a cada push i a cada Pull Request contra main

name: CI

on:
  push:
    branches: [main]        # Push directe a main (merge de PR)
  pull_request:
    branches: [main]        # Cada PR contra main

# Cancel·lar execucions anteriors si arriba un push nou
# (evita execucions redundants mentre edites un PR)
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # --- Job 1: Tests i build Java ---
  java:
    name: Java Build & Test
    runs-on: ubuntu-latest
    
    # Serveis Docker necessaris per als tests d'integració
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: esportspulse_test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
        # Healthcheck: espera que PostgreSQL estigui llest
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      # Pas 1: Descarregar el codi
      - uses: actions/checkout@v4

      # Pas 2: Configurar Java 21
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      # Pas 3: Cache de Maven — reutilitza dependències entre execucions
      - name: Cache Maven dependencies
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-

      # Pas 4: Compilar i testejar
      - name: Build and test
        run: mvn verify --batch-mode --no-transfer-progress
        env:
          # Configuració de la BD de test (usa el servei PostgreSQL de dalt)
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/esportspulse_test
          SPRING_DATASOURCE_USERNAME: test
          SPRING_DATASOURCE_PASSWORD: test

      # Pas 5: Publicar resultats de tests (visible al PR)
      - name: Publish test results
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Java Tests
          path: '**/target/surefire-reports/*.xml'
          reporter: java-junit

  # --- Job 2: Tests i lint Python ---
  python:
    name: Python Lint & Test
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      # Configurar Python
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      # Cache de pip
      - name: Cache pip dependencies
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
          restore-keys: |
            ${{ runner.os }}-pip-

      # Instal·lar dependències
      - name: Install dependencies
        run: |
          cd ai-python
          pip install -r requirements.txt

      # Linting amb ruff (substitut modern de flake8 + isort)
      - name: Lint with ruff
        run: |
          pip install ruff
          cd ai-python
          ruff check src/

      # Tests unitaris (excloure integració que necessita infraestructura)
      - name: Run unit tests
        run: |
          cd ai-python
          pytest tests/ --ignore=tests/integration -v

  # --- Job 3: Build Docker (només si tests passen) ---
  docker:
    name: Docker Build
    runs-on: ubuntu-latest
    needs: [java, python]   # Espera que Java i Python passin
    if: github.event_name == 'pull_request'  # Només a PRs (no a push a main)

    steps:
      - uses: actions/checkout@v4

      # Build de totes les imatges amb docker-compose
      - name: Build Docker images
        run: docker compose build

      # Verificar que les imatges s'han creat
      - name: Verify images
        run: docker images | grep esportspulse
```

### 3. Eliminar workflows antics (10 min)

```bash
# Esborrar els workflows individuals que ja estan consolidats
rm .github/workflows/build.yml
rm .github/workflows/test-java.yml
rm .github/workflows/test-python.yml
rm .github/workflows/lint.yml
# Mantenir docker-build.yml si fa coses específiques no cobertes per ci.yml
```

### 4. Mesurar l'impacte (10 min)

Compara els temps abans i després:

```bash
# Veure les últimes execucions del CI a GitHub
gh run list --limit 10

# Veure el detall d'una execució concreta
gh run view <RUN_ID>
```

Anota:
- Temps total del CI abans: ___
- Temps total del CI després: ___
- Workflows eliminats: ___
- Cache hits esperats: Maven (~200MB), pip (~100MB)

### 5. Commit (5 min)

```bash
git add .
git commit -m "ci: consolidate workflows into unified CI with caching and parallelism"
```

---

## Checklist de Lliurament

- [ ] Inventari de workflows actuals completat
- [ ] Workflow `ci.yml` unificat amb jobs Java, Python i Docker
- [ ] Cache de Maven i pip configurat
- [ ] `concurrency` configurat per cancel·lar execucions redundants
- [ ] Serveis Docker (PostgreSQL) configurats per tests d'integració
- [ ] Workflows antics eliminats o arxivats
- [ ] El CI passa en verd
- [ ] Commit fet
