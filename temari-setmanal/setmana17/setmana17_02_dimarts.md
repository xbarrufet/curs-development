# Setmana 17 — Dimarts: Escriure la Spec: Endpoint de Comparació de Campions

## Objectiu del Dia

Escriure una especificació completa per a un nou endpoint de negoci: comparar dos campions costat a costat. Al final del dia has de tenir un document de spec que compleixi totes les "regles d'or" que vas definir ahir, llest per donar-lo a un agent demà.

---

## Teoria

### Del Requisit de Negoci a la Spec Tècnica

El procés professional de desenvolupament comença amb un requisit de negoci (el que vol el client o el producte) i el transforma en una spec tècnica (el que necessita l'enginyer — o l'agent — per implementar).

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Requisit de    │     │  Spec           │     │  Implementació  │
│  Negoci         │────▶│  Tècnica        │────▶│  (Agent o Dev)  │
│                 │     │                 │     │                 │
│  "Vull comparar │     │  GET /compare   │     │  Codi + Tests   │
│   campions"     │     │  Input/Output   │     │  + Logging      │
│                 │     │  Edge Cases     │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

El requisit de negoci d'avui:

> **"Com a usuari d'EsportsPulse, vull poder comparar dos campions costat a costat per veure les seves estadístiques i decidir quin triar en la meva partida."**

### Template de Spec (El Teu Format Estàndard)

A partir d'avui, cada feature nova que facis ha de seguir aquest template. Guarda'l com a referència permanent:

```markdown
# Feature: [Nom descriptiu en 3-5 paraules]

## Context
[1-3 frases. Per què existeix? Quin problema de l'usuari resol?]

## Endpoint
[Mètode] [URL]
- Versió: v1
- Autenticació: [sí/no, quin tipus]

## Input

### Path Parameters
| Param | Tipus | Obligatori | Restriccions | Exemple |
|-------|-------|------------|--------------|---------|

### Query Parameters
| Param | Tipus | Obligatori | Restriccions | Exemple |
|-------|-------|------------|--------------|---------|

### Request Body (si aplica)
[JSON d'exemple amb comentaris]

## Output

### [Codi HTTP] [Descripció]
[JSON d'exemple]

### [Codi HTTP] [Descripció]
[JSON d'exemple]

## Lògica de Negoci
1. [Regla 1]
2. [Regla 2]

## Casos Límit
- [Cas 1] → [Resposta esperada]
- [Cas 2] → [Resposta esperada]

## Tests Esperats
1. Donat [precondició], quan [acció], llavors [resultat]
2. ...

## Criteris d'Acceptació
- [ ] [Criteri 1]
- [ ] [Criteri 2]

## Observabilitat
- Logs: [què es registra]
- Mètriques: [què es mesura]
- Alerta: [quan s'hauria d'alertar]
```

### Decisions de Disseny que Cal Prendre

Abans d'escriure la spec, has de prendre decisions de disseny. Cada decisió ha de quedar documentada a la spec:

**1. Com s'identifiquen els campions?**
- Per `id` (slug)? Per `name`? Per ambdós?
- Decisió recomanada: per `championId` (slug), coherent amb l'endpoint GET existent.

**2. Quin format d'URL?**
- Opció A: `GET /api/v1/champions/compare?champion1=jinx&champion2=lee-sin`
- Opció B: `GET /api/v1/compare/champions/{id1}/{id2}`
- Opció C: `POST /api/v1/champions/compare` amb body `{ "champions": ["jinx", "lee-sin"] }`

Cada opció té avantatges:
```
Opció A (query params):
  + Fàcil de compartir com a URL (bookmarkable)
  + Segueix REST — és una consulta, no una creació
  - Els noms dels params són arbitraris

Opció B (path params):
  + URL neta i llegible
  - Implica un ordre (id1 vs id2) que potser no importa
  - Difícil d'estendre a 3+ campions

Opció C (POST amb body):
  + Extensible a N campions
  - POST per a una lectura trenca la semàntica REST
  - No es pot compartir com a URL
```

**3. Què retorna la comparació?**
- Només les dades en paral·lel? O també un "veredicte" (qui guanya en cada categoria)?
- Decisió recomanada: dades en paral·lel + diferències calculades. Sense veredicte — el client decideix.

### Documentar el "Per Què" de Cada Decisió

Un error comú en specs és documentar QUÈ s'ha decidit però no PER QUÈ. Això és problemàtic perquè:

- Un futur dev (o tu mateix en 3 mesos) no sabrà si la decisió era important o arbitrària.
- L'agent no pot qüestionar una decisió si no n'entén la raó.

```markdown
## Decisions de Disseny

### Format d'URL: Query Parameters (Opció A)
**Decisió:** GET /api/v1/champions/compare?champion1=X&champion2=Y
**Raó:** Volem que la URL sigui compartible (bookmarkable). Com que és una
operació de lectura, GET és semànticament correcte. Els query params permeten
afegir un tercer campió en el futur sense trencar l'API.
**Alternatives descartades:** POST amb body (trenca semàntica REST per lectura),
path params (implica ordre i limita a 2 campions).
```

---

## Activitat

### 1. Prendre les Decisions de Disseny (20 min)

Abans d'escriure la spec, documenta les teves decisions. Crea un fitxer `decisions-compare.md`:

```markdown
# Decisions de Disseny: Compare Champions

## 1. Identificació de campions
- Decisió:
- Raó:

## 2. Format d'URL
- Decisió:
- Raó:
- Alternatives descartades:

## 3. Estructura de la resposta
- Decisió:
- Raó:

## 4. Què passa si els dos campions són iguals?
- Decisió:
- Raó:
```

### 2. Escriure la Spec Completa (40 min)

Crea el fitxer `spec-compare-champions.md` seguint el template. Aquí tens una guia parcial per orientar-te — completa els camps que falten:

```markdown
# Feature: Comparar Dos Campions

## Context
Els usuaris d'EsportsPulse volen comparar estadístiques de dos campions
per prendre decisions informades sobre quin triar. Actualment han d'obrir
dues pestanyes i comparar manualment — aquest endpoint ho automatitza.

## Endpoint
GET /api/v1/champions/compare?champion1={id1}&champion2={id2}
- Versió: v1
- Autenticació: No (endpoint públic)

## Input

### Query Parameters
| Param      | Tipus  | Obligatori | Restriccions                    | Exemple   |
|------------|--------|------------|---------------------------------|-----------|
| champion1  | string | Sí         | slug, ^[a-z][a-z\-]{1,29}$     | jinx      |
| champion2  | string | Sí         | slug, ^[a-z][a-z\-]{1,29}$     | lee-sin   |

## Output

### 200 OK
<!-- COMPLETA: escriu el JSON de resposta amb tots els camps.
     Inclou les dades de cada campió i les diferències calculades. -->

### 400 Bad Request — paràmetres invàlids
<!-- COMPLETA: exemples de 400 per cada cas -->

### 404 Not Found — un o ambdós campions no existeixen
<!-- COMPLETA: indica QUIN campió no s'ha trobat -->

## Lògica de Negoci
1. Validar format dels dos championId (regex)
2. Buscar ambdós campions a la base de dades
3. Si un no existeix, retornar 404 indicant quin
4. Calcular diferències: per cada camp numèric, diff = champion1 - champion2
5. <!-- COMPLETA: hi ha més regles? -->

## Casos Límit
- champion1 == champion2 (mateix campió) → ???
- champion1 existeix, champion2 no → ???
- cap dels dos existeix → ???
- champion1 falta al query string → ???
- championId amb format invàlid → ???

## Tests Esperats
<!-- COMPLETA: mínim 8 tests seguint el format
     "Donat X, quan Y, llavors Z" -->

## Criteris d'Acceptació
- [ ] Tots els tests passen
- [ ] Logging amb LangFuse
- [ ] Respon en < 150ms
- [ ] <!-- COMPLETA: afegeix-ne més -->

## Observabilitat
- Logs: cada comparació registra champion1, champion2, temps de resposta
- Mètriques: comptador de comparacions, parelles més comparades
- Alerta: si el temps de resposta supera 500ms durant 5 minuts
```

### 3. Auto-revisió amb les Regles d'Or (15 min)

Agafa les "regles d'or" que vas escriure ahir i revisa la teva pròpia spec punt per punt. Per cada regla, marca si la teva spec la compleix o no:

```markdown
# Auto-revisió de la Spec

| # | Regla                              | Compleix? | Notes              |
|---|------------------------------------|-----------|--------------------|
| 1 | Exemples reals de request/response | Sí/No     |                    |
| 2 | Tots els codis HTTP definits       | Sí/No     |                    |
| 3 | ...                                | ...       |                    |
```

Si trobes regles que no compleixes, actualitza la spec abans de fer commit.

### 4. Commit de la Spec (5 min)

```bash
# Afegeix la spec i les decisions de disseny
git add spec-compare-champions.md decisions-compare.md
# Commit amb format Conventional Commits
git commit -m "feat(spec): write complete spec for champion comparison endpoint"
```

---

## Checklist de Lliurament

- [ ] Fitxer `decisions-compare.md` amb totes les decisions documentades amb raons
- [ ] Fitxer `spec-compare-champions.md` complet seguint el template
- [ ] Tots els camps del template omplerts (cap `<!-- COMPLETA -->` pendent)
- [ ] Mínim 8 tests esperats definits
- [ ] Tots els codis HTTP (200, 400, 404) amb exemples JSON
- [ ] Casos límit identificats i resolts
- [ ] Auto-revisió feta amb les regles d'or de dilluns
- [ ] Commit a la branca `feature/week17-spec-driven`
