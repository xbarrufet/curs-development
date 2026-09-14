# Setmana 21 — Divendres: Consolidacio de Specs i PR

## Objectiu del Dia

Fer la revisio final de tota la documentacio del projecte (CLAUDE.md, .cursorrules, OpenSpecs, dashboard), assegurar coherencia entre tots els fitxers, i crear un Pull Request que consolidi tot el treball de la setmana.

---

## Teoria

### La Revisio Final com a Disciplina Professional

En un equip de desenvolupament real, abans de fer merge d'una branca, es fa una revisio exhaustiva. No es busca nomes que "funcioni", sino que:

1. **Coherencia**: Tot el que diu un document coincideix amb el que diuen els altres.
2. **Completesa**: No hi ha seccions buides, TODO pendents, o placeholder text.
3. **Actualitzacio**: Tot reflecteix l'estat actual del codi, no una versio anterior.
4. **Executabilitat**: Totes les comandes i instruccions funcionen copy-paste.

```bash
# Checklist de coherencia entre documents
# 1. CLAUDE.md llista els mateixos serveis que docker-compose.yml?
# 2. Les convencions del .cursorrules coincideixen amb el codi real?
# 3. Les eines dels agents (OpenSpecs) estan implementades?
# 4. El dashboard mostra les metriques documentades?
# 5. Les variables d'entorn documentades existeixen al .env.example?
```

### Anatomia d'un Bon Pull Request

Un PR professional conte:

```markdown
# Titol: descriptiu i concis
# feat(docs): add definitive project specifications and agent dashboard

# Cos del PR:
## Resum
# Que s'ha fet i per que.

## Canvis Principals
# Llista dels canvis mes importants amb context.

## Com Provar
# Instruccions pas a pas per verificar els canvis.

## Screenshots
# Si hi ha canvis visuals (dashboard), inclou captures.

## Checklist
# - [ ] Tests passen
# - [ ] Documentacio actualitzada
# - [ ] Sense secrets al codi
```

### Branques i Merging

```bash
# Flux de treball per a un PR net:
# 1. Assegura't que la branca esta actualitzada amb main
git fetch origin
git rebase origin/main

# 2. Verifica que no hi ha conflictes
git status

# 3. Executa tots els tests
mvn test
cd ai-python && pytest

# 4. Crea el PR
gh pr create --title "feat(docs): project specs and agent dashboard" \
  --body "## Resum\n..."
```

---

## Activitat

### 1. Auditoria de Coherencia (30 min)

Revisa sistematicament que tot es coherent:

```bash
# 1. Compara serveis documentats amb serveis reals
echo "=== Serveis al docker-compose ==="
docker-compose config --services

echo "=== Serveis documentats al CLAUDE.md ==="
grep -A 20 "## Arquitectura" CLAUDE.md

# 2. Compara convencions amb codi real
echo "=== Naming al codi Java ==="
# Busca noms de classes per verificar PascalCase
grep -rn "public class\|public interface" backend-java/ | head -10

echo "=== Naming al codi Python ==="
# Busca noms de funcions per verificar snake_case
grep -rn "def " ai-python/src/ | head -10

# 3. Verifica que les eines dels agents existeixen
echo "=== Eines documentades a l'OpenSpec ==="
grep -A 1 "##.*Eines\|##.*Tools" docs/specs/*.md

echo "=== Eines implementades al codi ==="
grep -rn "def search_teams\|def get_team_stats\|def get_player_stats" ai-python/
```

Corregeix qualsevol incoherencia que trobis.

### 2. Eliminar Placeholders i TODOs (15 min)

```bash
# Busca todos, fixme, placeholder, i exemples no completats
grep -rni "TODO\|FIXME\|placeholder\|TBD\|CANVIA AIXO" \
  CLAUDE.md .cursorrules docs/

# Si trobes algun, completa'l o elimina'l.
# Un document amb TODOs no es un document finalitzat.
```

### 3. Verificar Executabilitat (20 min)

Prova cada comanda documentada al CLAUDE.md:

```bash
# Copia i enganxa literalment cada comanda del CLAUDE.md
# i verifica que funciona sense modificacions.

# Exemple: si el CLAUDE.md diu "docker-compose up -d"
docker-compose up -d

# Si diu "mvn test -pl backend-java"
mvn test -pl backend-java

# Si diu "cd ai-python && pytest"
cd ai-python && pytest
```

Si alguna comanda falla, actualitza el CLAUDE.md amb la comanda correcta.

### 4. Verificar el Dashboard (15 min)

```bash
# Arrenca tot el sistema
docker-compose up -d

# Obre el dashboard i verifica cada pagina
# http://localhost:8501
```

Comprova:
- La pagina d'estat dels agents mostra informacio real?
- L'historial de consultes te dades?
- Les metriques de qualitat es visualitzen correctament?
- El desglossament de costos es precis?

### 5. Preparar el PR (20 min)

```bash
# Assegura't que tot esta comitejat
git status

# Si hi ha canvis pendents, fes commit
git add -A
git commit -m "docs: final review and coherence fixes"

# Rebase amb main per estar al dia
git fetch origin
git rebase origin/main

# Executa els tests una ultima vegada
mvn test
cd ai-python && pytest

# Crea el Pull Request
gh pr create \
  --title "feat(docs): definitive project specs and agent monitoring dashboard" \
  --body "## Resum

Consolidacio completa de la documentacio del projecte i creacio del dashboard
avancat de monitoritzacio d'agents.

## Canvis Principals

- **CLAUDE.md definitiu**: Descripcio completa del projecte, arquitectura,
  convencions, comandes d'execucio, i guia de contribucio.
- **.cursorrules consolidat**: Totes les regles de Cursor unificades.
- **OpenSpecs d'agents**: Especificacions completes dels agents Quantitatiu
  i Knowledge (proposit, eines, fluxos, exemples).
- **Dashboard avancat**: 4 pagines noves a Streamlit (estat agents,
  historial, metriques qualitat, costos).
- **Test de documentacio**: Resultats del test amb sessio nova de Claude Code.

## Com Provar

1. \`docker-compose up -d\`
2. Obrir http://localhost:8501 i navegar per les pagines del dashboard
3. Obrir una sessio nova de Claude Code al directori del projecte
4. Verificar que Claude enten l'arquitectura i genera codi correcte

## Checklist
- [ ] CLAUDE.md complet i sense TODOs
- [ ] .cursorrules coherent amb el codi
- [ ] OpenSpecs dels dos agents
- [ ] Dashboard amb 4 pagines noves
- [ ] Tests passen
- [ ] Sense secrets al codi"
```

### 6. Auto-Revisio del PR (10 min)

Revisa el teu propi PR com si fossis un reviewer extern:

```bash
# Mira el diff complet del PR
gh pr diff

# Verifica que no hi ha fitxers que no haurien d'estar
# (secrets, .env, fitxers temporals)
gh pr diff --name-only
```

---

## Checklist de Lliurament

- [ ] Auditoria de coherencia completada (tots els documents coincideixen)
- [ ] Cap TODO, FIXME o placeholder als documents finals
- [ ] Totes les comandes del CLAUDE.md provades i funcionals
- [ ] Dashboard verificat amb dades reals o simulades
- [ ] Tests Java i Python passen
- [ ] PR creat amb descripcio completa
- [ ] PR auto-revisat (cap fitxer incorrecte al diff)
- [ ] Branca rebased amb main
