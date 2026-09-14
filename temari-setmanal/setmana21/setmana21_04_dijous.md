# Setmana 21 — Dijous: Exercici Avaluat: Pot un Agent Nou Contribuir?

## Objectiu del Dia

Posar a prova la qualitat de la documentacio (CLAUDE.md, .cursorrules, OpenSpecs) donant-la a una sessio nova de Claude Code i verificant si pot contribuir al projecte sense explicacions addicionals. Al final del dia, hauran quedat documentats els punts febles i corregides les mancances.

---

## Teoria

### El Test del "Nou Contribuidor"

En el mon del software professional, la qualitat de la documentacio es mesura amb una prova simple: **pot un nou desenvolupador contribuir al projecte sense preguntar res?**

Amb eines d'IA, aquesta prova es encara mes exigent:
- Un huma pot inferir coses pel context visual (estructura de carpetes, noms de fitxers).
- Una IA dependra exclusivament del que esta escrit explicitament.

**Per que es important?**

```
# Escenari 1: Documentacio insuficient
# Demanes a Claude: "Afegeix un endpoint per llistar tornejos"
# Claude genera un endpoint... pero:
# - Posa el fitxer en una carpeta que no existeix
# - Usa Spring Boot 2.x en comptes de 3.x
# - No afegeix autenticacio JWT
# - No segueix el patro controlador -> servei -> repositori
# Resultat: has de corregir 5 coses. No has estalviat temps.

# Escenari 2: Documentacio completa
# Demanes a Claude: "Afegeix un endpoint per llistar tornejos"
# Claude genera un endpoint que:
# - Posa el fitxer a backend-java/src/main/java/.../controller/
# - Usa Spring Boot 3.2 amb els imports correctes
# - Inclou @PreAuthorize per a JWT
# - Segueix el patro de 3 capes amb tests
# Resultat: fas review, tot correcte, fas merge.
```

### Metodologia del Test

El test segueix un protocol sistematic:

**1. Preparacio:**
- Tanca totes les sessions de Claude Code / Cursor.
- No tinguis cap fitxer obert que doni context addicional.
- Obre una sessio completament nova.

**2. Tasques de prova (de menor a major complexitat):**

```markdown
# Nivell 1 — Comprensio basica
# Pregunta: "Explica'm l'arquitectura del projecte"
# Esperat: Que descrigui els serveis, ports i connexions correctament

# Nivell 2 — Seguir convencions
# Tasca: "Crea un test unitari per al servei TeamService"
# Esperat: Que segueixi les convencions de naming i estructura

# Nivell 3 — Tasca real de desenvolupament
# Tasca: "Afegeix un endpoint GET /api/v1/tournaments que retorni
#         una llista de tornejos amb paginacio"
# Esperat: Que generi controlador + servei + repositori + test,
#          amb JWT i al directori correcte

# Nivell 4 — Integracio entre serveis
# Tasca: "Fes que l'agent quantitatiu pugui respondre preguntes
#         sobre tornejos"
# Esperat: Que sàpiga que ha de tocar tant Java com Python,
#          i que conegui el flux de comunicacio
```

**3. Avaluacio:**

Per a cada tasca, avalua:
- **Correctesa**: El codi generat funciona?
- **Convencions**: Segueix les regles del projecte?
- **Ubicacio**: Els fitxers estan al lloc correcte?
- **Completesa**: Inclou tests? Gestio d'errors? Logging?

### Rubrica d'Avaluacio

```markdown
# Rubrica per avaluar la qualitat de la documentacio
# Puntua cada criteri de 0 a 3:

# 0 — L'agent no ha pogut fer la tasca
# 1 — L'agent ha fet la tasca pero amb errors greus
# 2 — L'agent ha fet la tasca amb errors menors
# 3 — L'agent ha fet la tasca correctament

# Criteris:
# A. Comprensio de l'arquitectura        [ /3 ]
# B. Convencions de codi                  [ /3 ]
# C. Ubicacio dels fitxers                [ /3 ]
# D. Qualitat del codi generat            [ /3 ]
# E. Tests inclosos                       [ /3 ]
# F. Seguretat (JWT, secrets)             [ /3 ]
# G. Docker / infraestructura             [ /3 ]
# H. Comunicacio entre serveis            [ /3 ]
#                              Total:     [ /24 ]

# Interpretacio:
# 20-24: Documentacio excel·lent
# 15-19: Documentacio bona, petites millores
# 10-14: Documentacio insuficient, cal reescriure seccions
# 0-9:   Documentacio critica, l'agent no pot treballar
```

---

## Activitat

### 1. Preparar l'Entorn de Test (10 min)

```bash
# Assegura't que tot el projecte esta comitejat
git status
# Si hi ha canvis pendents, fes commit
git add -A && git commit -m "chore: prepare for documentation test"

# Verifica que el projecte arrenca correctament
docker-compose up -d
# Espera que tot estigui llest
docker-compose ps
```

### 2. Executar el Test — Nivell 1: Comprensio (15 min)

Obre una sessio **nova** de Claude Code:

```bash
# Obre una sessio nova al directori del projecte
# Claude Code llegira automaticament CLAUDE.md
claude
```

Primera pregunta:
> "Descriu l'arquitectura completa del projecte. Quins serveis hi ha, com es comuniquen, i quins ports utilitzen?"

**Avalua la resposta:**
- Menciona tots els serveis del docker-compose?
- Els ports son correctes?
- La comunicacio (REST, RabbitMQ) es correcta?

Apunta la puntuacio (0-3) i els errors al fitxer `docs/test-results/doc-test-s21.md`.

### 3. Executar el Test — Nivell 2: Convencions (15 min)

A la mateixa sessio, demana:
> "Crea un test unitari per al servei que gestiona equips (TeamService). Segueix les convencions del projecte."

**Avalua:**
- El nom del test segueix `should_X_when_Y`?
- Usa JUnit 5 amb les anotacions correctes?
- Esta al directori correcte dins `src/test/java/`?
- Importa les classes del package correcte?

### 4. Executar el Test — Nivell 3: Desenvolupament (20 min)

Demana una tasca mes complexa:
> "Afegeix un endpoint GET /api/v1/tournaments que retorni una llista paginada de tornejos. Inclou model, repositori, servei, controlador i test d'integracio."

**Avalua:**
- Crea tots els fitxers necessaris (model, repo, service, controller, test)?
- Usa Spring Boot 3.2 correctament?
- Inclou paginacio (Pageable)?
- L'endpoint requereix autenticacio JWT?
- El test usa MockMvc o similar?

### 5. Executar el Test — Nivell 4: Integracio (20 min)

La tasca mes exigent:
> "Fes que l'agent quantitatiu pugui respondre preguntes sobre estadistiques de tornejos. Necessito tant els canvis al backend Java com al servei Python."

**Avalua:**
- Sap que ha de tocar dos serveis?
- La comunicacio entre serveis es correcta?
- L'agent te la nova eina documentada?
- Els tests cobreixen la integracio?

### 6. Documentar Resultats i Corregir (30 min)

Crea `docs/test-results/doc-test-s21.md`:

```markdown
# Test de Documentacio — Setmana 21

## Resultats

| Criteri | Puntuacio | Notes |
|---------|-----------|-------|
| A. Comprensio arquitectura | X/3 | ... |
| B. Convencions de codi | X/3 | ... |
| C. Ubicacio fitxers | X/3 | ... |
| D. Qualitat codi | X/3 | ... |
| E. Tests inclosos | X/3 | ... |
| F. Seguretat | X/3 | ... |
| G. Docker | X/3 | ... |
| H. Comunicacio serveis | X/3 | ... |
| **Total** | **X/24** | |

## Problemes Detectats
<!-- Llista de cada problema amb la solucio aplicada -->

## Canvis Fets al CLAUDE.md
<!-- Que has afegit o corregit -->

## Canvis Fets al .cursorrules
<!-- Que has afegit o corregit -->
```

**Per a cada problema detectat:**
1. Identifica quina seccio de la documentacio era insuficient.
2. Corregeix-la amb informacio mes especifica.
3. Torna a provar la mateixa tasca per verificar la millora.

### 7. Commit (10 min)

```bash
git add docs/test-results/ CLAUDE.md .cursorrules
git commit -m "docs: add documentation quality test results and fix gaps"
```

---

## Checklist de Lliurament

- [ ] Test de Nivell 1 executat i documentat
- [ ] Test de Nivell 2 executat i documentat
- [ ] Test de Nivell 3 executat i documentat
- [ ] Test de Nivell 4 executat i documentat
- [ ] Resultats documentats a `docs/test-results/doc-test-s21.md`
- [ ] CLAUDE.md corregit amb les mancances detectades
- [ ] .cursorrules actualitzat si calia
- [ ] Puntuacio total calculada (objectiu: minim 18/24)
- [ ] Commit amb tots els canvis
