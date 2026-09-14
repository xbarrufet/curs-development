# Setmana 21 — Dimarts: .cursorrules Finals i Specs d'Agents

## Objectiu del Dia

Consolidar totes les regles de Cursor en un `.cursorrules` definitiu i finalitzar les especificacions OpenSpec dels dos agents (Quantitatiu i Knowledge). Al final del dia, qualsevol desenvolupador (huma o IA) ha de tenir una guia completa per treballar amb el projecte.

---

## Teoria

### .cursorrules: Regles per a l'IA de Cursor

El fitxer `.cursorrules` es el que Cursor llegeix automaticament per entendre com ha de generar codi al teu projecte. Durant 20 setmanes has anat afegint regles incrementalment. Ara es el moment de consolidar-les.

**Estructura d'un .cursorrules madur:**

```yaml
# .cursorrules — EsportsPulse Engine
# Aquest fitxer defineix les regles que Cursor segueix
# quan genera o modifica codi al projecte.

## Llenguatge i Versions
# - Java 21 amb Spring Boot 3.2
# - Python 3.12 amb FastAPI
# - No usar funcionalitats deprecated

## Estil de Codi
# Java:
#   - camelCase per a variables i metodes
#   - PascalCase per a classes i interfaces
#   - Constants en UPPER_SNAKE_CASE
#   - Packages: com.esportspulse.engine.[modul]
# Python:
#   - snake_case per a funcions i variables
#   - PascalCase per a classes
#   - Type hints obligatoris a totes les funcions

## Arquitectura
# - Backend Java: controladors -> serveis -> repositoris
#   (mai saltar capes)
# - AI Python: routers -> services -> agents
# - Comunicacio entre serveis: REST + RabbitMQ
# - Cache: Redis per a resultats d'agents i sessions

## Tests
# - Cada classe publica ha de tenir test unitari
# - Endpoints: test d'integracio amb MockMvc (Java) o TestClient (Python)
# - Noms de test: should_[resultat]_when_[condicio]

## Seguretat
# - Mai hardcodejar secrets, API keys o passwords
# - Usar @Value("${...}") en Java, os.environ en Python
# - Tots els endpoints (excepte /health i /auth) requereixen JWT

## Docker
# - Cada servei te el seu Dockerfile
# - docker-compose.yml a l'arrel per arrencar tot
# - Variables d'entorn via .env (no commitejat)

## Commits
# - Format: Conventional Commits
# - Tipus: feat, fix, docs, test, refactor, chore
# - Scope: java, python, streamlit, docker, ci
```

### Especificacions d'Agents amb OpenSpec

OpenSpec es un format per documentar com funciona un agent d'IA. Inclou:
- **Proposit**: Que fa l'agent i per a qui.
- **Inputs**: Que rep (format, tipus, restriccions).
- **Outputs**: Que retorna (format, estructura).
- **Eines**: Quines eines (tools) te disponibles.
- **Fluxos**: Pas a pas de com processa una peticio.
- **Errors**: Com gestiona situacions inesperades.

**Per que documentar els agents?**

1. **Reproduibilitat**: Qualsevol pot reconstruir l'agent seguint l'spec.
2. **Depuracio**: Si l'agent falla, l'spec indica on mirar.
3. **Evolucio**: Per afegir funcionalitat, primer actualitza l'spec.

```markdown
# Exemple d'OpenSpec per a l'Agent Quantitatiu

## Proposit
# Analitzar dades estadistiques d'equips i jugadors d'esports electronics.
# Rep preguntes en llenguatge natural i retorna analisis amb dades.

## Inputs
# - query: string (pregunta de l'usuari en catala o castella)
# - context: opcional, dades addicionals (equip_id, temporada)

## Eines Disponibles
# - search_teams: cerca equips per nom o regio
# - get_team_stats: obte estadistiques d'un equip
# - get_player_stats: obte estadistiques d'un jugador
# - calculate_metrics: calcula metriques derivades (winrate, KDA, etc.)

## Flux de Processament
# 1. Rep la query de l'usuari
# 2. Identifica les entitats (equips, jugadors, metriques)
# 3. Selecciona les eines necessaries
# 4. Executa les eines en ordre
# 5. Combina resultats
# 6. Genera resposta en llenguatge natural amb dades

## Gestio d'Errors
# - Equip no trobat: suggereix noms similars
# - Dades insuficients: informa de les limitacions
# - Error de connexio a BD: retorna missatge amable + log
```

### Diferencies entre Agent Quantitatiu i Knowledge

| Aspecte | Quantitatiu | Knowledge |
|---------|-------------|-----------|
| Dades | Estructurades (BD) | No estructurades (docs, webs) |
| Eines | SQL, calculs | RAG, embeddings, cerca |
| Output | Numeros, grafics | Text, resums, cites |
| Validacio | Resultats verificables | Qualitat subjectiva |

---

## Activitat

### 1. Consolidar .cursorrules (30 min)

Revisa tots els `.cursorrules` que has creat durant el curs:

```bash
# Busca tots els fitxers .cursorrules existents
find . -name ".cursorrules*" -type f

# Revisa el contingut de cadascun
cat .cursorrules
```

Crea el `.cursorrules` definitiu unificant totes les regles. Organitza'l en seccions clares:

1. **Llenguatge i Versions** — Que i quina versio.
2. **Estil de Codi** — Naming, format, imports.
3. **Arquitectura** — Capes, patrons, comunicacio.
4. **Tests** — Convencions, cobertura, noms.
5. **Seguretat** — Secrets, autenticacio, validacio.
6. **Docker** — Imatges, compose, xarxes.
7. **Commits i Branques** — Format, flux de treball.

### 2. Escriure OpenSpec de l'Agent Quantitatiu (30 min)

Crea `docs/specs/agent-quantitatiu.md`:

```markdown
# OpenSpec: Agent Quantitatiu d'EsportsPulse

## Metadades
<!-- Versio de l'spec, autor, data de creacio. -->

## Proposit
<!-- Que fa l'agent, per a qui, en quin context. -->

## Interficie
### Input
<!-- Format exacte de la peticio (JSON schema si es possible). -->

### Output
<!-- Format exacte de la resposta. Inclou exemples. -->

## Eines (Tools)
<!-- Llista completa d'eines amb descripcio i parametres. -->

## Flux de Processament
<!-- Diagrama o llista pas a pas. -->

## Prompts del Sistema
<!-- System prompt real que usa l'agent. -->

## Exemples
<!-- 3-5 exemples de preguntes i respostes esperades. -->

## Metriques de Qualitat
<!-- Com mesurem si l'agent funciona be? -->

## Limitacions Conegudes
<!-- Que NO pot fer l'agent? -->
```

### 3. Escriure OpenSpec de l'Agent Knowledge (30 min)

Crea `docs/specs/agent-knowledge.md` amb la mateixa estructura, pero adaptat al Knowledge agent:

- Documenta el flux de RAG (Retrieval-Augmented Generation).
- Especifica les fonts de coneixement (documents, URLs).
- Inclou com es gestionen les cites i les fonts.

### 4. Validar Coherencia (15 min)

Verifica que tot es coherent:

```bash
# El CLAUDE.md menciona els mateixos agents que les specs?
grep -i "agent" CLAUDE.md

# Les convencions del .cursorrules coincideixen amb les del CLAUDE.md?
diff <(grep -i "convention\|naming\|style" CLAUDE.md) \
     <(grep -i "convention\|naming\|style" .cursorrules)

# Les eines dels agents estan implementades al codi?
grep -rn "def search_teams\|def get_team_stats" ai-python/
```

### 5. Commit (10 min)

```bash
git add .cursorrules docs/specs/
git commit -m "docs: consolidate cursorrules and add agent OpenSpecs"
```

---

## Checklist de Lliurament

- [ ] `.cursorrules` definitiu amb totes les seccions
- [ ] OpenSpec de l'Agent Quantitatiu complet amb exemples
- [ ] OpenSpec de l'Agent Knowledge complet amb flux RAG
- [ ] Coherencia entre CLAUDE.md, .cursorrules i specs
- [ ] Eines dels agents documentades amb parametres
- [ ] Commit amb tots els fitxers de documentacio
