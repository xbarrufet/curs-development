# Setmana 17 — Dijous: Skills i Hooks: Automatitzar el Workflow

## Objectiu del Dia

Crear una skill personalitzada de Claude Code que encapsuli el workflow de code review del projecte, i un git hook `pre-push` que executi evals automàticament. Al final del dia has de tenir un workflow automatitzat: escrius codi, fas push, i el sistema verifica automàticament que els evals passen.

---

## Teoria

### Què és una Skill de Claude Code?

Una skill és un conjunt d'instruccions empaquetades que Claude Code executa quan li ho demanes amb una comanda `/nom-skill`. Pensa-hi com un "macro" intel·ligent: en lloc de repetir les mateixes instruccions cada cop, les encapsules en una skill reutilitzable.

```
Sense skill (cada cop):                  Amb skill (una comanda):
─────────────────────────                ─────────────────────────
"Revisa el codi seguint                  /review-esportspulse
 el checklist de seguretat,
 verifica null handling,
 comprova que els tests
 cobreixen els edge cases,
 mira que hi hagi logging..."
```

### Estructura d'una Skill

Les skills es guarden a `.claude/skills/` dins del projecte. Cada skill és un fitxer Markdown amb un format específic:

```
esportspulse-engine/
├── .claude/
│   ├── CLAUDE.md              ← Instruccions globals del projecte
│   └── skills/
│       └── review-esportspulse/
│           └── SKILL.md       ← La teva skill personalitzada
```

El fitxer `SKILL.md` té aquesta estructura:

```markdown
# review-esportspulse

Revisa el codi del projecte EsportsPulse seguint el checklist de qualitat.

## Instruccions

Quan l'usuari executi `/review-esportspulse`, segueix aquests passos:

### 1. Identificar Canvis
- Executa `git diff --staged` per veure els canvis preparats per al commit.
- Si no hi ha canvis staged, executa `git diff` per veure canvis no staged.
- Mostra un resum dels fitxers modificats.

### 2. Checklist de Seguretat
Per cada fitxer modificat, verifica:
- [ ] No hi ha secrets hardcodejats (claus API, contrasenyes, tokens)
- [ ] Els inputs de l'usuari estan validats amb el regex definit a la spec
- [ ] Les respostes d'error no exposen stack traces ni paths interns
- [ ] No hi ha SQL injection (si aplica)

### 3. Checklist de Correcció
- [ ] Null/undefined gestionats correctament (no assumeix existència)
- [ ] Els tipus coincideixen amb la spec (String, int, double)
- [ ] Tots els edge cases de la spec estan implementats
- [ ] Els codis HTTP coincideixen amb la spec

### 4. Checklist de Tests
- [ ] Tests per al happy path
- [ ] Tests per a cada error case (400, 404, 500)
- [ ] Tests per als edge cases documentats
- [ ] Assertions verifiquen TOTS els camps de la resposta
- [ ] Tests independents (no depenen d'ordre)

### 5. Checklist d'Observabilitat
- [ ] Logging als punts d'entrada i sortida
- [ ] Traces de LangFuse configurades
- [ ] Mètriques registrades (temps, comptadors)
- [ ] Logs en format estructurat (JSON)

### 6. Convencions del Projecte
- [ ] Java: camelCase per a variables, PascalCase per a classes
- [ ] Python: snake_case per a variables i funcions
- [ ] Conventional Commits per als missatges
- [ ] Comentaris en anglès al codi, documentació en català

### 7. Generar Report
Genera un resum amb:
- Total de checks passats / fallats
- Llista de problemes trobats amb severitat (alta/mitjana/baixa)
- Suggeriments de millora
```

### Com Registrar la Skill

Perquè Claude Code reconegui la skill, cal que existeixi al directori `.claude/skills/`. No cal cap configuració addicional — Claude Code detecta automàticament els fitxers `SKILL.md` dins de subdirectoris de `.claude/skills/`.

Un cop creada, la pots invocar:

```bash
# Des de la terminal amb Claude Code
claude "/review-esportspulse"

# O dins d'una sessió interactiva
> /review-esportspulse
```

### Git Hooks: Automatitzar Verificacions

A S7 vas aprendre sobre `pre-commit` hooks (verificar format, linting). Ara apliquem el mateix patró a una escala més gran: **`pre-push` hooks que executen evals**.

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  git commit  │────▶│  pre-commit │────▶│  git push   │────▶│  pre-push   │
│             │     │  (format,   │     │             │     │  (evals,    │
│             │     │   lint)     │     │             │     │   tests)    │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
                         Ràpid                                  Més lent
                         (< 5s)                                 (30s-2min)
```

**Per què `pre-push` i no `pre-commit`?**
- Els evals poden trigar 30 segons o més (criden a APIs d'LLM).
- Executar-los a cada commit seria frustrant.
- `pre-push` s'executa només quan puges codi — un moment natural per verificar qualitat.

### Anatomia d'un Hook pre-push

Un hook és un script executable a `.git/hooks/`. El `pre-push` rep informació sobre què s'està pujant:

```bash
#!/bin/bash
# .git/hooks/pre-push
# S'executa automàticament abans de cada "git push".
# Si retorna codi != 0, el push es cancel·la.

# Colors per a la sortida (millora la llegibilitat)
RED='\033[0;31m'    # Vermell per errors
GREEN='\033[0;32m'  # Verd per èxits
YELLOW='\033[1;33m' # Groc per avisos
NC='\033[0m'        # Reset color

echo "================================================"
echo "  Pre-push Hook: Verificant qualitat del codi"
echo "================================================"

# Pas 1: Executar tests unitaris
# Això és ràpid i detecta errors bàsics
echo -e "${YELLOW}[1/3] Executant tests unitaris...${NC}"
mvn test -pl backend-java --quiet 2>&1
# $? conté el codi de sortida de l'última comanda (0 = èxit)
if [ $? -ne 0 ]; then
    echo -e "${RED}ERROR: Tests unitaris han fallat. Push cancel·lat.${NC}"
    echo "Executa 'mvn test' per veure els detalls."
    exit 1  # Codi != 0 cancel·la el push
fi
echo -e "${GREEN}Tests unitaris: OK${NC}"

# Pas 2: Executar evals (de S14-S16)
# Les evals verifiquen que l'agent genera respostes correctes
echo -e "${YELLOW}[2/3] Executant evals...${NC}"

# Comprova que el fitxer d'evals existeix
if [ -f "evals/run_evals.py" ]; then
    # Executa les evals amb un timeout de 120 segons
    # timeout evita que el hook es quedi penjat si l'API no respon
    timeout 120 python3 evals/run_evals.py 2>&1
    if [ $? -ne 0 ]; then
        echo -e "${RED}ERROR: Evals han fallat. Push cancel·lat.${NC}"
        echo "Executa 'python3 evals/run_evals.py' per veure els detalls."
        exit 1
    fi
    echo -e "${GREEN}Evals: OK${NC}"
else
    # Si no hi ha evals, avisa però no bloqueja
    echo -e "${YELLOW}AVÍS: No s'han trobat evals (evals/run_evals.py).${NC}"
    echo "Considera afegir evals per a les features noves."
fi

# Pas 3: Verificar que no hi ha secrets al codi
# Busca patrons comuns de secrets als fitxers staged
echo -e "${YELLOW}[3/3] Verificant que no hi ha secrets...${NC}"
# grep -r cerca recursivament en tots els fitxers
# -l mostra només el nom del fitxer (no el contingut)
# El pattern busca claus API, tokens, i contrasenyes hardcodejades
SECRETS=$(grep -rl \
    -e "AKIA[0-9A-Z]" \
    -e "sk-[a-zA-Z0-9]" \
    -e "password\s*=\s*[\"'][^\"']*[\"']" \
    --include="*.java" --include="*.py" --include="*.yml" \
    src/ ai-python/ 2>/dev/null)

if [ -n "$SECRETS" ]; then
    echo -e "${RED}ERROR: Possibles secrets detectats a:${NC}"
    echo "$SECRETS"
    echo "Revisa aquests fitxers i mou els secrets a variables d'entorn."
    exit 1
fi
echo -e "${GREEN}Secrets check: OK${NC}"

echo ""
echo "================================================"
echo -e "${GREEN}  Totes les verificacions han passat!${NC}"
echo "  Push autoritzat."
echo "================================================"

# Codi 0 = tot bé, el push continua
exit 0
```

### Connexió entre Skill i Hook

La skill i el hook treballen junts però en moments diferents:

```
Desenvolupament:                          Publicació:
──────────────                            ───────────
1. Escrius codi                           5. git push
2. /review-esportspulse (manual)          6. pre-push hook (automàtic)
3. Correccions                               - tests
4. git commit                                - evals
                                             - secrets check
                                          7. Push OK (o rebutjat)
```

La skill és la teva xarxa de seguretat **manual** (la fas servir quan vols). El hook és la xarxa **automàtica** (s'executa sempre).

---

## Activitat

### 1. Crear la Skill `/review-esportspulse` (25 min)

Crea l'estructura de directoris i el fitxer de la skill:

```bash
# Crea el directori per a la skill dins de .claude/skills/
mkdir -p .claude/skills/review-esportspulse
```

Crea el fitxer `.claude/skills/review-esportspulse/SKILL.md` amb el contingut de la secció de teoria. Personalitza'l:

- Afegeix checks específics del teu projecte (noms de paquets, estructura de directoris).
- Afegeix una secció per verificar que la spec existeix i està actualitzada.
- Afegeix una secció per verificar que els tests cobreixen els edge cases de la spec.

Verifica que funciona:

```bash
# Obre Claude Code i invoca la skill
claude "/review-esportspulse"
```

### 2. Crear el Hook pre-push (25 min)

Crea el fitxer de hook:

```bash
# Crea el hook pre-push (ha d'estar a .git/hooks/)
touch .git/hooks/pre-push
# Fes-lo executable (sense això, Git l'ignora)
chmod +x .git/hooks/pre-push
```

Escriu el contingut del hook basant-te en l'exemple de la teoria. Adapta'l al teu projecte:

- Si fas servir `pytest` en lloc de `mvn test`, canvia la comanda.
- Ajusta el path dels evals al teu projecte (`evals/run_evals.py` o on els tinguis).
- Afegeix o treu checks segons el que necessitis.

### 3. Testar el Workflow Complet (20 min)

Fes una prova completa del cicle:

```bash
# 1. Fes un canvi petit al codi (afegeix un comentari, per exemple)
echo "// Test del workflow spec-driven" >> src/main/java/com/esportspulse/engine/App.java

# 2. Stage i commit
git add .
git commit -m "test: verify pre-push hook workflow"

# 3. Push — hauria d'activar el hook
git push origin feature/week17-spec-driven
```

Documenta què passa:
- El hook s'ha executat? Quins passos han passat?
- Si ha fallat, per què? Corregeix el problema i torna a provar.
- Mesura quant triga el hook (hauria de ser < 2 minuts).

### 4. Versionar el Hook (10 min)

Els hooks de `.git/hooks/` no es versionen amb Git (el directori `.git/` no es puja). Per compartir el hook amb l'equip, copia'l a una carpeta versionada:

```bash
# Crea un directori per guardar hooks versionats
mkdir -p scripts/hooks

# Copia el hook al directori versionat
cp .git/hooks/pre-push scripts/hooks/pre-push

# Crea un script d'instal·lació perquè altres devs puguin activar-lo
cat > scripts/install-hooks.sh << 'SCRIPT'
#!/bin/bash
# Script per instal·lar els hooks del projecte.
# Copia els hooks de scripts/hooks/ a .git/hooks/ i els fa executables.

echo "Instal·lant hooks del projecte..."

# Copia cada hook del directori versionat al directori de Git
cp scripts/hooks/* .git/hooks/
# Fa tots els hooks executables
chmod +x .git/hooks/*

echo "Hooks instal·lats correctament."
echo "Hooks actius:"
# Llista els hooks instal·lats
ls -la .git/hooks/ | grep -v ".sample"
SCRIPT

# Fes el script d'instal·lació executable
chmod +x scripts/install-hooks.sh

# Commit tot
git add .claude/skills/ scripts/
git commit -m "feat(tooling): add review skill and pre-push hook with evals

- Custom /review-esportspulse skill for project-specific code review
- Pre-push hook runs tests, evals, and secrets check
- Install script for team hook setup"
```

---

## Checklist de Lliurament

- [ ] Skill `.claude/skills/review-esportspulse/SKILL.md` creada i funcional
- [ ] Hook `.git/hooks/pre-push` creat i executable
- [ ] Hook copia versionada a `scripts/hooks/pre-push`
- [ ] Script `scripts/install-hooks.sh` per instal·lar hooks
- [ ] Workflow testat: commit + push activa el hook automàticament
- [ ] Documentat el resultat del test del workflow (passes/fails)
- [ ] Commit a la branca `feature/week17-spec-driven`
