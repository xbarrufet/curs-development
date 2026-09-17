# Setmana 17 — Dimecres: Generació amb Agent i Code Review

## Objectiu del Dia

Donar la spec d'ahir a un agent (Claude Code o Cursor) i obtenir una implementació funcional. Al final del dia has de tenir codi generat, revisat sistemàticament, i iterat almenys una vegada. Has de documentar què va bé i què cal corregir — i si l'agent s'equivoca, la culpa és de la spec.

---

## Teoria

### El Workflow Spec-Driven amb Agent

El procés que seguiràs avui és el nucli del desenvolupament assistit per IA:

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  1. Spec  │────▶│ 2. Agent │────▶│ 3. Review│────▶│ 4. Iterate│
│  (ahir)   │     │  genera  │     │  humà    │     │  spec o   │
│           │     │  codi    │     │  revisa  │     │  codi     │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
                                       │                  │
                                       │    ┌─────────┐   │
                                       └───▶│ 5. Done │◀──┘
                                            │ (tests  │
                                            │  passen)│
                                            └─────────┘
```

**Regla fonamental:** Quan l'agent genera codi incorrecte, pregunta't primer: "La meva spec era prou clara?" Sovint el problema no és l'agent — és l'ambigüitat de la spec.

### Com Donar la Spec a l'Agent

No copïis la spec i la enganxis al chat. Això perd context i format. En lloc d'això:

**Amb Claude Code (terminal):**

```bash
# Opció 1: Referència directa al fitxer
# Claude Code pot llegir fitxers del projecte automàticament
claude "Implementa la feature descrita a spec-compare-champions.md.
Segueix l'estructura existent del projecte.
Genera els tests que la spec defineix."

# Opció 2: Amb context explícit del projecte
claude "Llegeix spec-compare-champions.md i implementa-la.
Revisa primer l'estructura del projecte (src/, tests/) per
seguir les mateixes convencions. Genera codi + tests."
```

**Amb Cursor (IDE):**

```
@spec-compare-champions.md Implementa aquesta spec.
Segueix les convencions del projecte existent.
Genera tots els tests que la spec defineix.
```

### Què Esperar de la Primera Generació

Amb una spec completa, l'agent hauria de generar:

1. **Fitxers de codi** — Controller/route, service, model (si cal)
2. **Tests** — Almenys els que la spec demana
3. **Validacions** — Input validation segons les restriccions de la spec
4. **Errors** — Gestió dels codis HTTP definits

Però **sempre** hi haurà coses a corregir. L'objectiu no és que surti perfecte a la primera — és que surti prou bé per iterar ràpidament.

### Code Review Checklist

Recorda el checklist de S7 (Git i Code Review). Ara l'apliques al codi generat per un agent. La llista es divideix en categories:

**Seguretat:**
```markdown
- [ ] No hi ha secrets hardcodejats (claus API, contrasenyes)
- [ ] Els inputs de l'usuari estan validats i sanititzats
- [ ] No hi ha SQL injection possible (si aplica)
- [ ] Les respostes d'error no exposen informació interna (stack traces, paths)
```

**Correcció:**
```markdown
- [ ] Gestiona null/undefined correctament (no assumeix que tot existeix)
- [ ] Els tipus són correctes (no barreja String amb int)
- [ ] Els edge cases de la spec estan implementats
- [ ] Els codis HTTP coincideixen amb la spec
```

**Tests:**
```markdown
- [ ] Hi ha tests per al happy path (cas normal)
- [ ] Hi ha tests per a cada error case (400, 404)
- [ ] Hi ha tests per als edge cases de la spec
- [ ] Els tests són independents (no depenen de l'ordre d'execució)
- [ ] Els assertions verifiquen TOTS els camps, no només l'status code
```

**Convencions:**
```markdown
- [ ] Noms de variables/funcions segueixen la convenció del projecte
- [ ] Estructura de fitxers coherent amb el projecte existent
- [ ] Comentaris on el codi no és obvi (lògica de negoci, decisions)
- [ ] Logging present als punts clau (entrada, error, sortida)
```

**Observabilitat (de S14-S16):**
```markdown
- [ ] Traces de LangFuse als punts definits a la spec
- [ ] Mètriques registrades (temps de resposta, comptadors)
- [ ] Logs estructurats (JSON, no text pla)
```

### Iterar la Spec, No Només el Codi

Quan trobis un problema al codi generat, classifica'l:

| Tipus de Problema             | Acció                                    |
|-------------------------------|------------------------------------------|
| L'agent va ignorar la spec    | Repetir la instrucció, ser més explícit   |
| La spec era ambigua           | **Actualitzar la spec** i regenerar       |
| La spec no cobria aquest cas  | **Afegir el cas a la spec** i regenerar   |
| Bug d'implementació           | Reportar a l'agent amb detall específic   |
| Problema de convencions       | Afegir a les regles del projecte          |

> **Lliçó clau:** Si has d'explicar a l'agent el mateix 3 vegades, el problema és la spec. Actualitza-la perquè el proper cop (o un altre agent) no tingui el mateix dubte.

---

## Activitat

### 1. Donar la Spec a l'Agent (15 min)

Obre Claude Code o Cursor. Dona-li la spec `spec-compare-champions.md` amb una instrucció clara:

```
Implementa la feature descrita a spec-compare-champions.md.
- Llegeix primer l'estructura del projecte per seguir les convencions existents.
- Genera el codi de producció (controller, service, repository si cal).
- Genera TOTS els tests definits a la secció "Tests Esperats".
- Afegeix logging segons la secció "Observabilitat".
- Comenta el codi on la lògica de negoci no sigui òbvia.
```

Deixa que l'agent generi. No l'interrompis. Quan acabi, guarda tot el que ha generat.

### 2. Code Review Sistemàtic (30 min)

Crea un fitxer `review-iteration-1.md` i revisa el codi punt per punt:

```markdown
# Code Review — Iteració 1

## Data: [avui]
## Agent: [Claude Code / Cursor]
## Spec: spec-compare-champions.md

### Seguretat
| Check                                 | OK? | Notes           |
|---------------------------------------|-----|-----------------|
| No secrets hardcodejats               |     |                 |
| Inputs validats                       |     |                 |
| Respostes d'error segures             |     |                 |

### Correcció
| Check                                 | OK? | Notes           |
|---------------------------------------|-----|-----------------|
| Null handling correcte                 |     |                 |
| Tipus correctes                       |     |                 |
| Edge cases implementats               |     |                 |
| Codis HTTP coincideixen amb spec      |     |                 |

### Tests
| Check                                 | OK? | Notes           |
|---------------------------------------|-----|-----------------|
| Happy path                            |     |                 |
| Error cases (400, 404)                |     |                 |
| Edge cases                            |     |                 |
| Assertions completes                  |     |                 |

### Convencions
| Check                                 | OK? | Notes           |
|---------------------------------------|-----|-----------------|
| Noms segueixen convencions            |     |                 |
| Estructura de fitxers coherent        |     |                 |
| Comentaris presents                   |     |                 |
| Logging als punts clau                |     |                 |

### Observabilitat
| Check                                 | OK? | Notes           |
|---------------------------------------|-----|-----------------|
| LangFuse traces                       |     |                 |
| Mètriques registrades                 |     |                 |
| Logs estructurats                     |     |                 |

## Resum
- Total checks: XX
- Checks OK: XX
- Checks KO: XX

## Problemes Trobats
1. [Descripció del problema] → [Causa: spec ambigua / agent va ignorar / bug]
2. ...

## Canvis a la Spec Necessaris
1. [Què cal afegir o clarificar a la spec]
2. ...
```

### 3. Iterar (30 min)

Amb els problemes identificats:

1. **Actualitza la spec** si el problema era d'ambigüitat o falta d'informació.
2. **Dona feedback específic a l'agent** per als bugs d'implementació:

```
# Exemple de feedback específic (bo)
"Al CompareController, línia 45: retornes 500 quan champion1 no existeix,
però la spec diu 404. Corregeix per retornar 404 amb el JSON d'error
definit a la secció 'Output > 404 Not Found' de la spec."

# Exemple de feedback vague (dolent)
"Els errors no funcionen bé, arregla-ho."
```

3. **Executa els tests** per verificar que la iteració millora el resultat:

```bash
# Executa els tests de la feature nova
mvn test -pl backend-java -Dtest="CompareChampionsTest"
# O si és Python:
pytest tests/test_compare_champions.py -v
```

4. Documenta la iteració al fitxer `review-iteration-1.md` afegint una secció:

```markdown
## Resultat de la Iteració
- Tests que passen ara: X/Y
- Problemes resolts: [llista]
- Problemes pendents: [llista]
- Canvis fets a la spec: [llista]
```

### 4. Commit del Codi i la Review (5 min)

```bash
# Afegeix el codi generat i revisat
git add src/ tests/ review-iteration-1.md spec-compare-champions.md
# Commit amb referència a la spec
git commit -m "feat(compare): implement champion comparison from spec

- Generated from spec-compare-champions.md
- Reviewed and iterated once
- See review-iteration-1.md for review details"
```

---

## Checklist de Lliurament

- [ ] Codi generat per l'agent per a l'endpoint de comparació
- [ ] Fitxer `review-iteration-1.md` complet amb tots els checks
- [ ] Almenys una iteració feta (feedback a l'agent + nova generació)
- [ ] Spec actualitzada si s'han trobat ambigüitats
- [ ] Tests executats i documentat quants passen
- [ ] Commit a la branca `feature/week17-spec-driven`
- [ ] Pots explicar per què un problema era de la spec i no de l'agent
