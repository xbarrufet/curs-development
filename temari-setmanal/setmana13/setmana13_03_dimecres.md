# Setmana 13 — Dimecres: Desenvolupament Guiat per Wireframe

## Objectiu del Dia

Aprendre a escriure una especificació de dashboard (wireframe + components + interaccions) ABANS de codificar, donar-la a un agent d'IA (Cursor, Claude Code) i comparar el codi generat amb l'especificació. Al final del dia, tindràs un `dashboard-spec.md` i un dashboard generat que hi coincideix.

---

## Teoria

### El Concepte: Escriu l'Spec Primer, Codifica Després

En el desenvolupament professional, sovint es dissenya la interfície abans de programar-la. Amb eines d'IA generativa, aquest procés es torna encara més potent:

1. **Escrius una especificació** (wireframe + components + regles)
2. **Dones l'spec a un agent IA** (Cursor, Claude Code, Copilot)
3. **Compares el resultat** amb l'especificació original
4. **Iteres l'spec** si el resultat no coincideix (ajustes l'spec, no el codi)

> **Principi clau:** L'especificació és la font de veritat. Si el codi no coincideix amb l'spec, el problema és l'spec (no era prou clara) o l'agent (no l'ha interpretat bé). En ambdós casos, millores l'spec i tornes a generar.

### Per Què Wireframes ASCII?

Els wireframes ASCII són ideals per donar a agents d'IA perquè:
- **Són text pla** — L'agent els pot llegir perfectament (no cal imatges)
- **Són precisos** — Defineixen posicions relatives dels elements
- **Són versionables** — Es poden guardar a Git com qualsevol fitxer
- **Són ràpids** — Es creen amb el teclat, sense eines de disseny

### Format d'un Wireframe ASCII

```
+--------------------------------------------------+
|  [Títol de la secció]                             |   ← Capçalera
+--------------------------------------------------+
| [Element]  | [Element]  | [Element]               |   ← Files
| [_input__] | [Botó]     | Valor                   |   ← Inputs i botons
+--------------------------------------------------+
```

Convencions:
- `[text]` = Element interactiu (botó, link, etc.)
- `[_____]` = Camp d'entrada (input)
- `|` i `+` = Separadors de columnes i seccions
- Text pla = Contingut estàtic

---

## Activitat

### Pas 1: Escriure l'Especificació (`dashboard-spec.md`)

Crea el fitxer `dashboard/dashboard-spec.md` amb el següent contingut. Aquesta especificació defineix exactament com ha de ser el dashboard:

```markdown
# EsportsPulse Dashboard — Especificació

## 1. Wireframe General

### Pantalla de Login (si no autenticat)
```
+--------------------------------------------------+
|           EsportsPulse — Login                    |
+--------------------------------------------------+
|                                                    |
|            Usuari:    [_________________]          |
|            Password:  [_________________]          |
|                                                    |
|                     [  Entrar  ]                   |
|                                                    |
+--------------------------------------------------+
```

### Dashboard Principal (autenticat)
```
+--------------------------------------------------+
| EsportsPulse Dashboard              [Refresh]    |
| Usuari: admin                  [Tancar sessió]   |
+--------------------------------------------------+
| KPI 1            | KPI 2            | KPI 3       |
| Total Campions   | WinRate Mitjà    | Més Popular  |
| 165              | 50.2%            | Jinx         |
+--------------------------------------------------+
| Filtres:         | Taula Champions:                |
| [Cerca nom____]  | Champion | WinRate | Games      |
| WinRate min [__] | Ahri     | 52.3%   | 150K       |
| WinRate max [__] | Jinx     | 51.1%   | 120K       |
|                  | Lux      | 50.8%   | 110K       |
+------------------+--------------------------------+
| Gràfic: Top 10 Champions per Games Played         |
| ████████████████████████████ Jinx    120K          |
| ██████████████████████████  Ahri    115K           |
| ████████████████████████    Lux     110K           |
| ██████████████████████      Yasuo   105K           |
+--------------------------------------------------+
| Crear Nou Campió:                                  |
| Nom [________] Rol [Dropdown▼] WR [__] Games [__] |
|                              [  Crear Campió  ]   |
+--------------------------------------------------+
```

## 2. Components i Comportament

### Login
- **Formulari:** Usuari (text) + Password (password) + Botó "Entrar"
- **Acció:** POST /auth/login amb JSON {username, password}
- **Resultat OK:** Guarda token a session_state, mostra dashboard
- **Resultat KO:** Mostra error "Credencials incorrectes"

### Capçalera
- **Títol:** "EsportsPulse Dashboard" (alineat a l'esquerra)
- **Botó Refresh:** Recarrega dades de l'API
- **Info usuari:** Mostra nom d'usuari + botó "Tancar sessió"

### KPIs (3 columnes)
- **Total Campions:** Nombre total de campions (len(data))
- **WinRate Mitjà:** Mitjana de winRate amb 1 decimal
- **Més Popular:** Nom del campió amb més gamesPlayed

### Filtres (columna esquerra)
- **Cerca nom:** text_input, filtra per name (case-insensitive, parcial)
- **WinRate mínim:** number_input, filtra campions amb winRate >= valor
- **WinRate màxim:** number_input, filtra campions amb winRate <= valor
- Tots els filtres s'apliquen simultàniament (AND lògic)

### Taula (columna dreta)
- **Columnes:** Champion, WinRate (%), Games
- **Ordenació:** Per qualsevol columna (interactiva)
- **Dades:** Filtrades segons els filtres actius

### Gràfic de Barres
- **Dades:** Top 10 campions per gamesPlayed (descendent)
- **Eix X:** Nom del campió
- **Eix Y:** Nombre de partides

### Formulari Crear Campió
- **Camps:** Nom (text), Rol (dropdown), WinRate (number), Games (number)
- **Acció:** POST /api/champions amb JSON
- **Resultat OK:** Missatge verd + recarrega taula
- **Resultat KO:** Missatge d'error vermell

## 3. Interaccions i Flux

1. Usuari obre l'app → veu Login
2. Fa login → POST /auth/login → guarda token → veu Dashboard
3. Dashboard carrega dades → GET /api/champions amb Bearer token
4. Escriu al camp de cerca → taula es filtra en temps real
5. Canvia filtres WinRate → taula es filtra
6. Clica Refresh → torna a cridar GET /api/champions
7. Omple formulari + clica Crear → POST /api/champions → taula s'actualitza
8. Clica Tancar sessió → esborra token → torna a Login

## 4. Restriccions Tècniques

- Framework: Streamlit (Python)
- API Base URL: http://localhost:8080
- Autenticació: JWT Bearer token a totes les crides
- Timeout de crides: 5 segons
- Gestió d'errors: st.error() per errors, st.warning() per avisos
- Layout: wide (layout="wide" a set_page_config)
```

### Pas 2: Donar l'Spec a l'Agent IA

Ara dóna l'especificació a un agent d'IA per generar el codi. Pots fer-ho de diverses maneres:

```bash
# Opció 1: Amb Claude Code (al terminal)
# Obre Claude Code al directori del projecte i dóna-li l'spec
claude "Llegeix dashboard/dashboard-spec.md i genera dashboard/app.py 
        que implementi exactament l'especificació. Fes servir Streamlit.
        L'API base és http://localhost:8080. 
        Inclou autenticació JWT amb session_state."

# Opció 2: Amb Cursor (a l'editor)
# Obre dashboard-spec.md, selecciona tot el contingut,
# obre el chat de Cursor (Ctrl+L) i escriu:
# "Genera dashboard/app.py que implementi exactament aquest wireframe"
```

### Pas 3: Comparar Resultat amb l'Spec

Després de generar el codi, compara-ho element per element:

```markdown
# Checklist de Comparació (copia i omple)

## Layout
- [ ] Login apareix si no hi ha token
- [ ] Dashboard apareix si hi ha token
- [ ] Layout és "wide"

## Components
- [ ] KPIs en 3 columnes amb els valors correctes
- [ ] Camp de cerca per nom
- [ ] Filtre WinRate mínim
- [ ] Filtre WinRate màxim
- [ ] Taula amb columnes: Champion, WinRate, Games
- [ ] Gràfic de barres amb Top 10
- [ ] Formulari de creació amb 4 camps

## Interaccions
- [ ] Login crida POST /auth/login
- [ ] Token es guarda a session_state
- [ ] Cerca filtra en temps real
- [ ] Filtres WinRate funcionen
- [ ] Formulari crida POST /api/champions
- [ ] Refresh recarrega dades
- [ ] Tancar sessió esborra token

## Errors
- [ ] API offline → missatge d'error
- [ ] Login incorrecte → missatge d'error
- [ ] Token expirat → redirigeix a login
```

### Pas 4: Iterar l'Spec (si cal)

Si el resultat no coincideix amb l'wireframe, **NO modifiques el codi generat**. En comptes d'això, millora l'spec:

```markdown
# Exemples de millores a l'spec quan el resultat no coincideix:

# PROBLEMA: L'agent ha posat els filtres a dalt en lloc d'a l'esquerra
# SOLUCIÓ: Ser més explícit al wireframe
# ABANS: "Filtres: cerca, winRate min, winRate max"
# DESPRÉS: "Filtres a la columna esquerra (30% ample), Taula a la dreta (70%)"

# PROBLEMA: L'agent no ha posat el gràfic de barres
# SOLUCIÓ: Afegir una secció explícita amb dades d'exemple
# AFEGIR: "Secció 'Gràfic': st.bar_chart amb top 10 per gamesPlayed"

# PROBLEMA: El formulari no envia les dades correctes
# SOLUCIÓ: Especificar el JSON exacte que s'ha d'enviar
# AFEGIR: "JSON: {name: string, role: string, winRate: float, gamesPlayed: int}"
```

Repeteix el procés (genera, compara, itera) fins que el resultat coincideixi. Idealment, en 3 iteracions o menys.

### Reflexió: Per Què Funciona?

```python
# La clau del desenvolupament guiat per wireframe és que SEPARA:
#
# 1. DISSENY (humà) — Decidir QUÈ ha de fer l'aplicació
#    → L'humà escriu l'spec amb wireframe, components, interaccions
#    → Requereix coneixement del domini i criteri de disseny
#
# 2. IMPLEMENTACIÓ (agent IA) — Decidir COM fer-ho
#    → L'agent genera el codi Streamlit a partir de l'spec
#    → Requereix coneixement tècnic del framework
#
# 3. VALIDACIÓ (humà) — Comparar resultat amb spec
#    → L'humà comprova que el codi coincideix amb l'spec
#    → Requereix atenció al detall i capacitat crítica
#
# Aquesta separació fa que l'IA sigui molt més útil:
# - L'IA no ha de prendre decisions de disseny (que sovint fa malament)
# - L'humà no ha d'escriure codi boilerplate (que és tediós)
# - El resultat és verificable objectivament (coincideix amb l'spec o no)
```

---

## Checklist de Lliurament

- [ ] `dashboard/dashboard-spec.md` creat amb wireframe ASCII, components, interaccions i restriccions
- [ ] L'spec s'ha donat a un agent IA (Cursor, Claude Code o similar) per generar `dashboard/app.py`
- [ ] S'ha comparat el resultat amb l'spec usant el checklist de comparació
- [ ] S'han fet com a màxim 3 iteracions per ajustar l'spec fins que el resultat coincideixi
- [ ] El dashboard generat implementa tots els components de l'wireframe
- [ ] L'spec final reflecteix exactament el que fa el dashboard
