# Setmana 17 — Dilluns: Què és una Bona Spec? Anatomia d'una Especificació

## Objectiu del Dia

Entendre què fa que una especificació sigui efectiva per guiar un agent de codi. Al final del dia has de poder distingir una spec bona d'una de dolenta, conèixer els camps obligatoris d'una spec completa, i analitzar exemples reals amb criteri propi.

---

## Teoria

### El Canvi de Mentalitat: De Codi a Especificació

Fins ara has après a escriure codi, tests, evals i observabilitat. Ara toca un salt qualitatiu: **la feina més important d'un enginyer modern és especificar**, no implementar.

Quan treballes amb un agent (Claude Code, Cursor, o qualsevol LLM amb eines), la qualitat del codi generat depèn directament de la qualitat de l'especificació que li dones. Això no és una opinió — és mesurable:

```
┌──────────────────────────────────────────────────────────┐
│  Qualitat de la Spec   →   Iteracions necessàries        │
├──────────────────────────────────────────────────────────┤
│  Spec vaga             →   5-10 iteracions (frustració)  │
│  Spec parcial          →   3-5 iteracions (acceptable)   │
│  Spec completa         →   1-2 iteracions (eficient)     │
└──────────────────────────────────────────────────────────┘
```

> **Lliçó clau:** Una spec és la forma més avançada de prompt engineering. No estàs "demanant" a l'agent que faci alguna cosa — estàs **definint amb precisió** què ha d'existir quan acabi.

### Què és una Spec (i Què NO és)

Una spec defineix el **QUÈ**, mai el **COM**.

| Spec defineix (QUÈ)                    | Spec NO defineix (COM)                  |
|-----------------------------------------|-----------------------------------------|
| Endpoint: `GET /api/v1/champions/{id}`  | "Usa un HashMap per guardar campions"   |
| Input: path param `id` (string, UUID)   | "Crea una classe ChampionService"       |
| Output: JSON amb camps X, Y, Z          | "Itera amb un for-each"                 |
| Error 404 si no existeix                | "Llança una RuntimeException"           |
| Test: retorna 200 amb dades vàlides     | "Fes servir Mockito per al mock"        |

**Per què?** Perquè l'agent pot trobar una implementació millor que la que tu imaginaves. Si li dius COM fer-ho, limites la seva capacitat. Si li dius QUÈ ha de fer, pot sorprendre't positivament.

### Anatomia d'una Spec Completa

Tota spec ben escrita conté aquests camps:

```markdown
# Feature: [Nom descriptiu]

## Context
Per què existeix aquesta feature? Quin problema resol?

## Endpoint
- Mètode HTTP + URL
- Versió de l'API

## Input
- Paràmetres (path, query, body)
- Tipus de cada paràmetre
- Restriccions (obligatori/opcional, longitud, format)
- Exemple de request vàlid

## Output
- Estructura del JSON de resposta
- Codis HTTP per cada cas (200, 400, 404, 500)
- Exemple de response per cada codi

## Lògica de Negoci
- Regles que s'apliquen (càlculs, transformacions, validacions)
- Ordre d'execució si importa

## Casos Límit (Edge Cases)
- Què passa si l'input és buit?
- Què passa si l'entitat no existeix?
- Què passa si hi ha dades duplicades?
- Què passa amb valors extrems?

## Tests Esperats
- Llista de tests que han de passar
- Cada test descrit com: "Donat X, quan Y, llavors Z"

## Criteris d'Acceptació
- Llista de condicions que han de ser certes perquè la feature es consideri completa
- Inclou rendiment si és rellevant ("respon en < 200ms")

## Observabilitat
- Quins logs ha de generar?
- Quines mètriques s'han de registrar?
- Com sabré que funciona en producció?
```

### Exemples: Bona Spec vs Mala Spec

**Exemple A — Spec dolenta:**

```markdown
# Endpoint de campions
Fes un endpoint que retorni informació d'un campió.
Ha de funcionar bé i tenir tests.
```

Problemes: No diu quin mètode HTTP, quina URL, quin format de resposta, quins errors gestionar, ni quins tests concretament.

**Exemple B — Spec parcial:**

```markdown
# GET /api/v1/champions/{id}
Retorna un campió per ID.
- 200 amb el campió si existeix
- 404 si no existeix
- Ha de tenir tests
```

Millor, però: No especifica l'estructura del JSON, no defineix edge cases, no diu quins camps té un campió, no inclou exemples.

**Exemple C — Spec completa:**

```markdown
# Feature: Obtenir Campió per ID

## Context
L'aplicació EsportsPulse necessita un endpoint per consultar la informació
detallada d'un campió de League of Legends. Els clients (frontend, mòbil)
consumiran aquest endpoint per mostrar la fitxa d'un campió.

## Endpoint
GET /api/v1/champions/{championId}

## Input
- `championId` (path param, string, obligatori)
  - Format: slug en minúscules (ex: "jinx", "lee-sin")
  - Restricció: només lletres minúscules i guions, 2-30 caràcters
  - Regex: ^[a-z][a-z\-]{1,29}$

## Output

### 200 OK
{
  "id": "jinx",
  "name": "Jinx",
  "role": "ADC",
  "difficulty": 6,
  "winRate": 51.3,
  "pickRate": 12.7,
  "banRate": 8.2
}

### 400 Bad Request (championId no vàlid)
{
  "error": "INVALID_CHAMPION_ID",
  "message": "Champion ID must match pattern: ^[a-z][a-z\\-]{1,29}$",
  "detail": "Received: '123-abc!!'"
}

### 404 Not Found
{
  "error": "CHAMPION_NOT_FOUND",
  "message": "Champion 'zzz-fake' not found"
}

## Casos Límit
- championId amb caràcters especials → 400
- championId buit → 400
- championId amb majúscules → 400 (no fem lowercase automàtic)
- championId de campió eliminat del joc → 404

## Tests Esperats
1. Donat un championId vàlid ("jinx"), retorna 200 amb totes les propietats
2. Donat un championId inexistent ("zzz"), retorna 404
3. Donat un championId amb format invàlid ("123"), retorna 400
4. Donat un championId buit, retorna 400
5. Verificar que el Content-Type és application/json

## Criteris d'Acceptació
- [ ] Tots els tests passen
- [ ] Logging amb LangFuse del temps de resposta
- [ ] Respon en < 100ms (sense xarxa externa)
```

### Per Què la Spec és Prompt Engineering Avançat

Quan dones una spec completa a un agent, estàs fent exactament el que un bon prompt fa:

1. **Context clar** — L'agent sap per què existeix la feature
2. **Exemples concrets** — Input/output reals, no descripcions abstractes
3. **Restriccions explícites** — L'agent no ha d'endevinar què és vàlid i què no
4. **Criteris de verificació** — L'agent pot auto-avaluar si ha fet bé la feina

Això connecta directament amb el que vas aprendre a S14-S16 sobre evals: els tests esperats de la spec es converteixen en evals automàtiques.

---

## Activitat

### 1. Analitzar les Tres Specs (30 min)

Rellegeix els tres exemples de la secció de teoria (Exemple A, B, C). Per a cadascun, respon aquestes preguntes en un fitxer `analisi-specs.md`:

```markdown
# Anàlisi de Specs — Setmana 17

## Exemple A
- Puntuació (1-10):
- Camps que falten:
- Quantes iteracions necessitaria un agent per implementar-ho correctament?
- Què podria malinterpretar l'agent?

## Exemple B
- Puntuació (1-10):
- Camps que falten:
- Quantes iteracions necessitaria un agent per implementar-ho correctament?
- Què podria malinterpretar l'agent?

## Exemple C
- Puntuació (1-10):
- Camps que falten (si n'hi ha):
- Quantes iteracions necessitaria un agent per implementar-ho correctament?
- Quins aspectes estan ben definits que els altres exemples no tenen?
```

### 2. Redactar Criteris d'una Bona Spec (20 min)

A partir de la teva anàlisi, escriu una llista de "regles d'or" per escriure specs. Mínim 7 regles. Exemple:

```markdown
# Regles d'Or per Escriure Specs

1. Sempre inclou exemples de request i response reals (no pseudocodi)
2. Defineix TOTS els codis HTTP possibles, no només el 200
3. ...
```

### 3. Trobar una Spec Dolenta al Teu Projecte (20 min)

Revisa el codi que has generat amb agent durant les setmanes anteriors. Identifica un cas on la instrucció que vas donar a l'agent era vaga o incompleta. Escriu:

- Què li vas demanar (el prompt original)
- Què va generar l'agent
- Què hauria estat millor si haguessis escrit una spec formal
- Com reescriuries la instrucció com a spec completa

### 4. Commit de l'Anàlisi (10 min)

```bash
# Afegeix els fitxers d'anàlisi creats durant l'activitat
git add analisi-specs.md regles-specs.md
# Fes commit amb format Conventional Commits
git commit -m "docs(spec): analyze spec quality examples and define golden rules"
```

---

## Checklist de Lliurament

- [ ] Fitxer `analisi-specs.md` amb puntuació i anàlisi dels 3 exemples
- [ ] Fitxer `regles-specs.md` amb mínim 7 regles d'or
- [ ] Identificat un cas real del projecte on una spec formal hauria millorat el resultat
- [ ] Commit a la branca `feature/week17-spec-driven`
- [ ] Pots explicar la diferència entre QUÈ (spec) i COM (implementació)
