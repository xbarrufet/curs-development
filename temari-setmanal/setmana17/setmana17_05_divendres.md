# Setmana 17 — Divendres: Consolidació: Mesurar el Procés Spec-Driven

## Objectiu del Dia

Fer una retrospectiva completa del procés spec-driven de la setmana. Al final del dia has de tenir mètriques concretes de quantes iteracions vas necessitar, per què, i com millorar. Has de crear un PR final que integri tota la feature amb les mètriques al description.

---

## Teoria

### Per Què Mesurar el Procés

Sense mètriques, no pots millorar. "Crec que va anar bé" no és una avaluació professional — "vaig necessitar 3 iteracions, la primera va fallar per un edge case no documentat a la spec" sí que ho és.

Les mètriques del procés spec-driven responen tres preguntes:

```
┌───────────────────────────────────────────────────────────┐
│  1. Quantes iteracions va caldre?                         │
│     → Mesura la qualitat de la spec                       │
│                                                           │
│  2. Per què va caldre cada iteració?                      │
│     → Identifica patrons d'error repetitius               │
│                                                           │
│  3. Quant temps va trigar?                                │
│     → Mesura l'eficiència del workflow                    │
└───────────────────────────────────────────────────────────┘
```

### Mètriques Clau del Procés

**1. Iteration Count (nombre d'iteracions)**

Compta cada vegada que vas haver de donar feedback a l'agent i esperar una nova generació. La primera generació compta com a iteració 1.

```markdown
| Iteració | Què va fallar                        | Causa arrel          |
|----------|--------------------------------------|----------------------|
| 1        | Primera generació — tests no passaven| Edge case no a spec  |
| 2        | Corregit edge case — logging absent  | Spec no detallava    |
| 3        | Tot correcte                         | —                    |
```

**Objectiu:** 1-2 iteracions per feature. Si necessites 4+, la spec era insuficient.

**2. Time per Iteration (temps per iteració)**

Mesura el temps des que dones la instrucció fins que tens el resultat revisat:

```markdown
| Iteració | Temps generació | Temps review | Temps correcció | Total |
|----------|-----------------|--------------|-----------------|-------|
| 1        | 2 min           | 15 min       | —               | 17 min|
| 2        | 1 min           | 5 min        | 10 min          | 16 min|
| 3        | 1 min           | 3 min        | —               | 4 min |
```

**3. Test Pass Rate per Iteration (taxa de tests per iteració)**

Quants tests passen a cada iteració:

```markdown
| Iteració | Tests totals | Tests OK | Tests KO | Pass Rate |
|----------|-------------|----------|----------|-----------|
| 1        | 8           | 5        | 3        | 62.5%     |
| 2        | 8           | 7        | 1        | 87.5%     |
| 3        | 8           | 8        | 0        | 100%      |
```

### El Camp "Operability Criterion"

Hi ha un camp que falta a moltes specs i que hauries d'afegir com a obligatori a partir d'ara:

```markdown
## Operability Criterion
Com sabré que aquesta feature funciona correctament en producció?

- [ ] Dashboard amb comptador de comparacions per minut
- [ ] Alerta si el temps de resposta > 500ms durant 5 minuts consecutius
- [ ] Log entry per cada comparació amb champion1, champion2, durada
- [ ] Mètrica de parelles més comparades (top 10 diari)
- [ ] Health check endpoint retorna verd si la feature està operativa
```

**Per què és crític?**

Sense operability criterion, una feature pot estar desplegada i trencada durant dies sense que ningú ho noti. La pregunta "com sabré que funciona?" obliga a pensar en:

1. **Monitorització activa** — Dashboards, mètriques en temps real
2. **Alertes** — Notificacions automàtiques quan alguna cosa va malament
3. **Diagnòstic** — Logs suficients per entendre un problema sense reproduir-lo
4. **Verificació** — Una manera de confirmar que la feature funciona just després del deploy

Connecta directament amb el que vas aprendre a S14-S16 sobre LangFuse i observabilitat. L'operability criterion és el pont entre "el codi funciona als tests" i "el codi funciona a producció".

### Anatomia d'una Bona Retrospectiva

Una retrospectiva del procés spec-driven hauria de cobrir:

```markdown
# Retrospectiva Spec-Driven — Setmana 17

## 1. Resum de la Feature
- Què s'ha implementat
- Quins fitxers s'han creat/modificat

## 2. Qualitat de la Spec
- La spec original era completa? (sí/no, per què)
- Quins canvis es van fer a la spec durant el procés?
- Quins camps van resultar ser els més importants?

## 3. Mètriques del Procés
- Nombre total d'iteracions: X
- Temps total: X minuts
- Test pass rate a la primera iteració: X%
- Test pass rate final: 100%

## 4. Anàlisi de Causes
Per cada iteració extra (a partir de la 2a):
- Què va fallar?
- La causa era la spec, l'agent, o el projecte?
- Com es podria haver evitat?

## 5. Lliçons Apreses
- Què faria diferent la pròxima vegada?
- Quines regles noves afegeixo al meu procés?

## 6. Operability
- S'han definit criteris d'operabilitat?
- Es poden verificar automàticament?
```

### El Cicle Complet: De Spec a PR

Aquesta setmana has completat un cicle sencer de desenvolupament professional:

```
Dilluns:    Entendre què és una bona spec
                     ↓
Dimarts:    Escriure la spec completa
                     ↓
Dimecres:   Generar codi amb agent + review
                     ↓
Dijous:     Automatitzar amb skills i hooks
                     ↓
Divendres:  Mesurar, retrospectiva, PR final
```

Això no és un exercici acadèmic — és exactament com treballen els equips d'enginyeria que usen IA de manera efectiva. La diferència entre un enginyer junior i un senior assistit per IA és la qualitat de les specs i la disciplina del procés.

---

## Activitat

### 1. Recollir Mètriques (20 min)

Revisa els commits, les reviews, i els fitxers de la setmana. Crea el fitxer `metriques-setmana17.md`:

```markdown
# Mètriques del Procés Spec-Driven — Setmana 17

## Feature: Comparació de Campions

### Iteration Count
| Iteració | Què va fallar                        | Causa arrel           |
|----------|--------------------------------------|-----------------------|
| 1        |                                      |                       |
| 2        |                                      |                       |
| ...      |                                      |                       |

### Time per Iteration
| Iteració | Generació | Review | Correcció | Total  |
|----------|-----------|--------|-----------|--------|
| 1        |           |        |           |        |
| ...      |           |        |           |        |
| TOTAL    |           |        |           |        |

### Test Pass Rate
| Iteració | Tests totals | OK | KO | Pass Rate |
|----------|-------------|----|----|-----------|
| 1        |             |    |    |           |
| ...      |             |    |    |           |

### Resum
- Iteracions totals:
- Temps total:
- Causa principal d'iteracions extra:
```

### 2. Escriure la Retrospectiva (25 min)

Crea el fitxer `retrospectiva-setmana17.md` seguint l'anatomia de la secció de teoria. Sigues honest amb l'anàlisi — l'objectiu no és demostrar que tot va anar perfecte, sinó aprendre del procés.

Preguntes guia:
- La spec era completa quan la vas donar a l'agent?
- Quins camps de la spec van ser els més valuosos per l'agent?
- Hi havia algo que la spec no cobria i que l'agent va haver d'inventar?
- El code review va trobar problemes que la spec hauria d'haver previngut?
- L'operability criterion estava definit? Si no, afegeix-lo ara a la spec.

### 3. Afegir Operability Criterion a la Spec (10 min)

Obre `spec-compare-champions.md` i afegeix la secció d'Operability Criterion si no la tenia, o millora-la basant-te en el que has après:

```markdown
## Operability Criterion
Com sabré que la comparació de campions funciona en producció?

- [ ] Log de cada comparació: champion1, champion2, temps de resposta, resultat
- [ ] Mètrica: comparacions per minut (normal: X, alerta si < Y o > Z)
- [ ] Mètrica: temps de resposta p50, p95, p99
- [ ] Alerta: temps de resposta p95 > 500ms durant 5 min consecutius
- [ ] Dashboard: top 10 parelles més comparades (actualitzat diàriament)
- [ ] Health check: GET /api/v1/health inclou status de la feature compare
```

### 4. Crear el PR Final (20 min)

Prepara tots els fitxers i crea un Pull Request que integri tota la feina de la setmana:

```bash
# Assegura't que tots els fitxers estan comitejats
git add metriques-setmana17.md retrospectiva-setmana17.md
git add spec-compare-champions.md  # amb l'operability criterion afegit

git commit -m "docs(retro): add metrics and retrospective for spec-driven week

- Iteration count: X iterations needed
- Main cause: [causa principal]
- Added operability criterion to spec"
```

Crea el PR amb les mètriques al description:

```bash
# Crea el Pull Request amb un resum complet
gh pr create \
  --title "feat: champion comparison endpoint (spec-driven)" \
  --body "## Resum
Feature de comparació de campions implementada seguint el workflow spec-driven.

## Mètriques del Procés
- **Iteracions:** X
- **Temps total:** X minuts
- **Test pass rate 1a iteració:** X%
- **Test pass rate final:** 100%
- **Causa principal d'iteracions:** [descripció]

## Fitxers Clau
- \`spec-compare-champions.md\` — Especificació completa
- \`review-iteration-1.md\` — Code review documentat
- \`metriques-setmana17.md\` — Mètriques del procés
- \`retrospectiva-setmana17.md\` — Retrospectiva i lliçons

## Artefactes de Tooling
- \`.claude/skills/review-esportspulse/SKILL.md\` — Skill de code review
- \`scripts/hooks/pre-push\` — Hook que executa tests i evals

## Checklist
- [ ] Spec completa amb operability criterion
- [ ] Codi implementat i revisat
- [ ] Tots els tests passen
- [ ] Skill de review funcional
- [ ] Hook pre-push funcional
- [ ] Mètriques i retrospectiva documentades"
```

### 5. Verificació Final (5 min)

Executa el push per activar el hook `pre-push` i confirma que tot passa:

```bash
# Push que activa el hook automàticament
git push origin feature/week17-spec-driven
```

Si el hook falla, corregeix i torna a intentar. Documenta-ho a la retrospectiva.

---

## Checklist de Lliurament

- [ ] Fitxer `metriques-setmana17.md` amb totes les mètriques omplertes
- [ ] Fitxer `retrospectiva-setmana17.md` amb anàlisi honest del procés
- [ ] Operability criterion afegit a `spec-compare-champions.md`
- [ ] PR creat amb mètriques al description
- [ ] Push executat i hook `pre-push` passat correctament
- [ ] Tots els fitxers de la setmana comitejats i pujats
- [ ] Pots explicar quantes iteracions vas necessitar i per què
- [ ] Pots explicar què és l'operability criterion i per què és obligatori
