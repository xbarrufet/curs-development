# Setmana 05 — Dijous: Especificacions de Refactoring, Code Review i Pre-commit Hooks

## Objectiu del Dia

Aprendre a escriure especificacions de refactoring que un agent IA (o un company) pugui seguir. Dominar el code review com a eina professional. Configurar pre-commit hooks per automatitzar la validacio d'estil. Al final del dia sabras escriure specs precises, fer reviews constructives i tenir guardianes automàtics al teu repositori.

---

## Teoria

### Code Review com a Habilitat Professional

El code review no es buscar errors (per això tenim tests i CI). Es una conversa professional sobre la qualitat del codi.

#### Què Buscar en un Code Review

```
Ordre de prioritat (de més a menys important):

1. 🔒 Seguretat     → SQL Injection, secrets exposats, input no validat
2. ✅ Correcció     → El codi fa el que diu que fa?
3. 🧪 Tests         → Hi ha tests? Cobreixen els casos importants?
4. 🏗️ Mantenibilitat → Es pot entendre en 6 mesos? Noms clars?
5. ⚡ Rendiment     → Hi ha bucles innecessaris o queries N+1?
6. 📝 Estil         → L'automatitza Checkstyle/ruff, no ho revisis tu
```

**Regla:** Mai comentes sobre estil manualment. Per a això tenim eines automàtiques (Checkstyle, ruff). El teu temps de reviewer es massa valuós per discutir espais.

---

#### Com Escriure Comentaris de Review

La formula: **Observació + Impacte + Suggeriment**

```
❌ MAL comentari:
"Això està malament."
→ No explica QUÈ ni PER QUÈ. Desmotiva.

❌ MAL comentari:
"Hauries d'usar Optional aquí."
→ No explica per què. Sembla una ordre.

✅ BON comentari:
"Observació: `findById()` retorna Optional, però aquí cridem `.get()` directament.
Impacte: Si l'ID no existeix, llançarà NoSuchElementException sense context.
Suggeriment: Considera `.orElseThrow(() -> new ChampionNotFoundException(id))`
per donar un missatge d'error descriptiu."
→ Explica el problema, l'impacte, i proposa solució.
```

**En Python:**

```
✅ BON comentari:
"Observació: Aquí capturem Exception genèric amb `except Exception: pass`.
Impacte: Si falla la connexió a la BD, l'error es perd i el servei retorna
una llista buida com si tot anés bé. L'usuari veu "0 campions" sense saber per què.
Suggeriment: Captura l'excepció específica (IOError) i logeja-la amb `logger.error()`."
```

---

### Escriure Especificacions de Refactoring

Una "spec" és un document que descriu **exactament** què vols que faci un agent (humà o IA). La qualitat de la spec determina la qualitat del resultat.

#### Estructura d'una Spec

```markdown
# Spec: Refactoritzar ChampionManagementService

## Objectiu
Separar la lògica de cerca de la lògica de persistència al servei de campions.

## Context
Actualment, `ChampionManagementService` té 15 mètodes que mesclen:
- Cerca (findByName, searchByRole, filterByWinRate)
- CRUD (create, update, delete)
- Validació (validateChampion, checkDuplicate)

## Regles
1. Crear `ChampionSearchService` amb els mètodes de cerca
2. Mantenir `ChampionManagementService` amb CRUD i validació
3. Tots els mètodes existents han de seguir funcionant (backward compatible)
4. No canviar les signatures públiques dels mètodes
5. Afegir @Transactional als mètodes que modifiquen dades

## Tests Esperats
- Tots els tests existents han de continuar passant SENSE modificacions
- Afegir test: `searchByRole_whenNoResults_returnsEmptyList`
- Afegir test: `create_whenDuplicate_throwsDuplicateException`

## Criteris d'Acceptació
- [ ] mvn test passa amb 0 errors
- [ ] mvn checkstyle:check passa
- [ ] Cap mètode té més de 20 línies
- [ ] Cap classe té més de 200 línies
```

#### Per Què les Specs Importen

```
Spec vaga:
"Refactoritza el servei de campions perquè sigui més net."
→ La IA/company no sap què vol dir "més net"
→ Necessites 5 iteracions per arribar al resultat

Spec precisa:
(La de l'exemple de dalt)
→ La IA/company sap exactament què fer
→ Resultat correcte al primer intent (o molt proper)
```

**Regla:** Si no pots escriure la spec, no entens prou bé el problema. Escriure la spec ES l'acte de pensar.

---

### Pre-commit Hooks — Guardianes Automàtics

Un pre-commit hook es un script que s'executa **automàticament** abans de cada commit. Si falla, el commit no es crea.

```
Flux amb pre-commit hook:

git commit -m "feat: add search"
       │
       ▼
  Pre-commit hook s'executa:
  1. Checkstyle (Java)     → ✅ Passa
  2. Ruff (Python)         → ❌ Falla!
     Error: unused import 'os'
       │
       ▼
  COMMIT REBUTJAT ❌
  "Fix the issues and try again"
       │
       ▼
  Developer arregla el problema
  git commit -m "feat: add search"  → ✅ Commit creat
```

---

#### Instal·lar Pre-commit (eina multiplataforma)

```bash
# Instal·la l'eina pre-commit (funciona amb Python, Java, JS, etc.)
pip install pre-commit
```

#### Configurar `.pre-commit-config.yaml`

```yaml
# .pre-commit-config.yaml
# Defineix quins hooks s'executen abans de cada commit

repos:
  # Hook per Python: ruff (linter ultra-ràpid)
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.5.0                    # Versió del hook
    hooks:
      - id: ruff                    # Lint: detecta errors d'estil i bugs
        args: [ --fix ]             # Corregeix automàticament si pot
      - id: ruff-format             # Format: reformata el codi automàticament

  # Hook per fitxers generals
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace     # Elimina espais al final de les línies
      - id: end-of-file-fixer       # Assegura newline al final del fitxer
      - id: check-yaml              # Valida que els YAML són correctes
      - id: check-added-large-files # Evita pujar fitxers grans per accident
        args: [ '--maxkb=500' ]     # Màxim 500KB per fitxer
      - id: detect-private-key      # Detecta claus privades al codi
```

```bash
# Instal·la els hooks al repositori (crea .git/hooks/pre-commit)
pre-commit install

# Executa manualment sobre tots els fitxers (útil la primera vegada)
pre-commit run --all-files
```

---

#### Checkstyle com a Hook per Java

Per Java, podem usar un script personalitzat com a pre-commit hook:

```bash
#!/bin/bash
# .git/hooks/pre-commit (fer executable amb chmod +x)
# Executa Checkstyle abans de cada commit

echo "Executant Checkstyle..."

# Executa checkstyle al directori Java del projecte
cd java/ && mvn checkstyle:check --batch-mode -q

# Si checkstyle falla (exit code != 0), el commit es rebutja
if [ $? -ne 0 ]; then
    echo ""
    echo "❌ Checkstyle ha fallat. Corregeix els errors abans de fer commit."
    echo "   Executa: mvn checkstyle:check per veure els detalls."
    exit 1    # Exit code 1 = el commit es cancel·la
fi

echo "✅ Checkstyle ha passat."
exit 0        # Exit code 0 = el commit continua
```

```bash
# Fer el script executable (necessari a Linux/Mac)
chmod +x .git/hooks/pre-commit
```

---

#### Ruff per Python — Configuració

```toml
# pyproject.toml o ruff.toml — Configuració de ruff per al projecte
[tool.ruff]
# Longitud màxima de línia
line-length = 100

# Regles activades (cada lletra és una categoria)
select = [
    "E",    # pycodestyle errors (estil bàsic)
    "F",    # pyflakes (variables no usades, imports duplicats)
    "I",    # isort (ordre dels imports)
    "N",    # pep8-naming (noms de variables i funcions)
    "UP",   # pyupgrade (modernitzar codi antic)
    "B",    # flake8-bugbear (bugs comuns)
    "S",    # flake8-bandit (seguretat)
]

# Regles ignorades
ignore = [
    "S101",  # Permetre 'assert' als tests
]
```

```bash
# Executar ruff manualment
ruff check .                    # Només reportar errors
ruff check . --fix              # Corregir automàticament el que pugui
ruff format .                   # Reformatar tot el codi
```

---

## Activitat

### Exercici 1: Escriure una Spec de Refactoring (30 min)

Escriu una spec per refactoritzar el `ChampionManagementService` del teu projecte EsportsPulse. La spec ha de seguir l'estructura:

1. **Objectiu** — Què vols aconseguir (1-2 frases)
2. **Context** — Estat actual del codi
3. **Regles** — Restriccions que s'han de complir
4. **Tests Esperats** — Quins tests han de passar
5. **Criteris d'Acceptació** — Checklist verificable

### Exercici 2: Dona la Spec a un Agent IA (30 min)

1. Copia la spec de l'Exercici 1 i dona-la a una IA (Claude, ChatGPT, Cursor).
2. Avalua el resultat:
   - Ha seguit totes les regles?
   - Els tests que ha generat cobreixen els casos importants?
   - Ha mantingut backward compatibility?
3. Compara amb el que hauries fet manualment.
4. **Reflexió:** Quant de temps has estalviat? La qualitat es comparable?

### Exercici 3: Configurar Pre-commit Hooks (30 min)

1. Instal·la `pre-commit`:

```bash
pip install pre-commit
```

2. Crea el fitxer `.pre-commit-config.yaml` al directori arrel del projecte (veure secció de teoria).

3. Instal·la els hooks:

```bash
pre-commit install
```

4. Prova que funciona:

```bash
# Introdueix un error d'estil intencionat (per exemple, import no usat)
# Intenta fer commit — ha de fallar
git add .
git commit -m "test: pre-commit hook"

# Corregeix l'error i torna a intentar
```

### Exercici 4: Code Review entre Companys (30 min)

Intercanvia el teu codi amb un company de classe. Fes una revisió seguint:

1. Revisa **seguretat** primer: hi ha secrets? SQL injection? Input no validat?
2. Revisa **correcció**: El codi fa el que diu?
3. Revisa **tests**: Quins casos importants falten?
4. Escriu 3 comentaris seguint la formula Observacio + Impacte + Suggeriment.

Si no tens company, revisa el teu propi codi de la setmana 4 amb ulls frescos.

---

## Checklist de Lliurament

- [ ] He escrit una spec de refactoring completa amb tots els apartats
- [ ] He donat la spec a una IA i he avaluat el resultat
- [ ] He configurat `.pre-commit-config.yaml` al meu projecte
- [ ] Els pre-commit hooks funcionen (he provat que rebutgen codi amb errors)
- [ ] He fet (o simulat) un code review amb 3 comentaris constructius
- [ ] Entenc la formula Observacio + Impacte + Suggeriment
- [ ] Commit amb missatge: `chore: add pre-commit hooks with ruff and checkstyle`
