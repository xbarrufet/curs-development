# Setmana 13 — Divendres: Poliment Final, Dashboard Complet i PR

## Objectiu del Dia

Polir el dashboard, executar el test E2E complet, actualitzar l'especificació, practicar la demo i crear el Pull Request. Al final del dia, el PR ha d'estar creat i el CI verd.

---

## Teoria

### Checklist de Qualitat del Dashboard

Abans de crear el PR, verifica que el dashboard compleix tots els requisits:

```python
# Cada punt correspon a una funcionalitat implementada durant la setmana
CHECKLIST = {
    # Autenticació (Dimarts)
    "login": "Login amb POST /auth/login, token JWT a session_state",
    "protected": "Dashboard només visible amb token vàlid",
    "bearer": "Authorization: Bearer <token> a totes les crides",
    "logout": "Botó que esborra token i torna al login",
    # Visualització (Dilluns + Dimarts)
    "kpis": "Total campions, WinRate mitjà, campió més popular",
    "table": "Taula interactiva amb cerca i filtres",
    "chart": "Gràfic de barres top 10 per partides",
    "form": "Formulari per crear campió via POST /api/champions",
    # Qualitat (Dimecres + Dijous)
    "spec": "dashboard-spec.md coincideix amb el dashboard final",
    "tests": "4+ tests pytest passant per logic.py",
    "ci": "CI valida Java + Python",
    "errors": "st.error/warning per errors d'API i connexió",
}
```

### Preparar la Demo (3 Minuts)

```
[0:00 - 0:30] Arquitectura
  "EsportsPulse té 3 components:
   [Streamlit :8501] --JWT--> [Spring Boot :8080] --> [BD]
                               [FastAPI :8000] --> [LLM]"

[0:30 - 1:30] Demo en Viu
  Login → KPIs → Cerca → Crear campió → Verificar a la taula

[1:30 - 2:30] Codi Rellevant
  - logic.py: lògica separada per testejar
  - test_logic.py: tests unitaris
  - ci.yml: pipeline Java + Python

[2:30 - 3:00] Tancament
  "Java per la lògica de negoci, Python per la IA, Streamlit per la UI."
```

---

## Activitat

### Pas 1: Versió Final del Dashboard

El canvi clau respecte a dimarts: importem les funcions de `logic.py` en lloc de calcular tot inline:

```python
# dashboard/app.py — Canvi principal: importar lògica pura
from logic import (filter_champions, calculate_kpis, format_win_rate,
                   get_top_champions, validate_champion_form)

# La resta de l'app.py és idèntica a dimarts, però substituïm:

# ABANS (inline):
# avg_wr = round(df["winRate"].mean(), 1)
# DESPRÉS (funció pura testejable):
kpis = calculate_kpis(campions)
st.metric("WinRate Mitjà", format_win_rate(kpis["avg_win_rate"]))

# ABANS (inline):
# df = df[df["name"].str.contains(cerca, case=False)]
# DESPRÉS (funció pura testejable):
filtrats = filter_champions(campions, search=cerca, min_wr=min_wr, max_wr=max_wr)

# ABANS (inline):
# df_top10 = df.nlargest(10, "gamesPlayed")
# DESPRÉS (funció pura testejable):
top10 = get_top_champions(campions, n=10)

# ABANS (inline):
# if not nou_nom: st.error("Obligatori")
# DESPRÉS (funció pura testejable):
errs = validate_champion_form(nom, rol, wr, games)
if errs:
    for e in errs:
        st.error(e)
```

Aplica aquests canvis al teu `app.py` de dimarts. La versió final ha de:
- Importar totes les funcions de `logic.py`
- Substituir la lògica inline pels crides a funcions pures
- Mantenir la mateixa UI i gestió d'errors

### Pas 2: Test E2E Manual Complet

Engega els 3 serveis en terminals separats:

```bash
# Terminal 1: Backend Java (Spring Boot al port 8080)
cd esportspulse-engine/backend-java
mvn spring-boot:run

# Terminal 2: Servei Python (FastAPI al port 8000)
cd esportspulse-engine/backend-python
uvicorn main:app --reload --port 8000

# Terminal 3: Dashboard Streamlit (al port 8501)
cd esportspulse-engine
streamlit run dashboard/app.py
```

Segueix el script i marca cada pas:

```markdown
## Flux Principal
1. [ ] http://localhost:8501 → Pàgina de Login
2. [ ] Credencials incorrectes → Error vermell
3. [ ] Credencials correctes → Dashboard amb KPIs
4. [ ] KPIs: total, WR mitjà, campió popular

## Filtres i Taula
5. [ ] Cerca "Ahri" → Només Ahri a la taula
6. [ ] WR mínim 51 → Exclou campions amb WR < 51
7. [ ] Esborra filtres → Tots els campions

## Gràfic i Formulari
8. [ ] Gràfic Top 10 visible
9. [ ] Crea "TestChampion" → Missatge verd
10. [ ] TestChampion apareix a la taula

## Sessió i Errors
11. [ ] Tancar sessió → Login
12. [ ] Para backend → Missatge d'error de connexió
```

### Pas 3: Revisar `dashboard-spec.md`

Compara l'spec de dimecres amb el dashboard final. Revisa punt per punt:

```markdown
# Checklist de revisió de l'spec vs dashboard final:

## Layout
- [ ] Login apareix si no hi ha token (coincideix amb wireframe?)
- [ ] Dashboard té layout "wide" com diu l'spec
- [ ] Filtres a l'esquerra, taula a la dreta

## Components nous no documentats
- [ ] validate_champion_form() — afegit dijous, no era a l'spec
      → Afegir a la secció "Formulari" de l'spec
- [ ] Funcions de logic.py importades a app.py
      → Afegir secció "Arquitectura" a l'spec

## Interaccions
- [ ] Login crida POST /auth/login (OK?)
- [ ] Filtres WR mínim/màxim funcionen (OK?)
- [ ] Formulari valida ABANS d'enviar (canvi respecte spec original)

## Restriccions tècniques
- [ ] API_BASE_URL = http://localhost:8080 (OK?)
- [ ] Timeout de 5 segons a totes les crides (OK?)
- [ ] Token a st.session_state (OK?)
```

Si hi ha desviacions justificades (validació afegida, lògica extreta), actualitza `dashboard-spec.md` perquè reflecteixi l'estat real del dashboard.

### Pas 4: Practicar la Demo

Practica l'explicació en veu alta seguint el guió de la secció de Teoria. Consells:

```python
# Punts clau per a una bona demo:
#
# 1. NO llegeixis codi línia per línia — explica EL CONCEPTE
#    MAL:  "Aquí importo streamlit, requests i pandas..."
#    BÉ:   "El dashboard usa Streamlit per la UI i requests per parlar amb l'API"
#
# 2. MOSTRA el flux complet, no funcionalitats aïllades
#    MAL:  "Aquí tenim els KPIs, aquí la taula, aquí el form..."
#    BÉ:   "Faig login, veig els KPIs, cerco Ahri, creo un campió nou..."
#
# 3. EXPLICA per què les decisions tècniques
#    "Hem extret la lògica a logic.py per poder-la testejar amb pytest
#     sense necessitat d'arrencar Streamlit"
#
# 4. MOSTRA el CI
#    "El pipeline valida Java amb mvn verify i Python amb pytest + ruff,
#     assegurant que cap commit trenca res"
```

### Pas 5: Commit, Push i Pull Request

```bash
# Verificar estat
cd esportspulse-engine
git status

# Afegir fitxers del dashboard
git add dashboard/app.py dashboard/logic.py dashboard/test_logic.py
git add dashboard/requirements.txt dashboard/dashboard-spec.md
git add .github/workflows/ci.yml

# Commit
git commit -m "feat(dashboard): add Streamlit dashboard with JWT auth, KPIs and tests

- Login page with JWT authentication (POST /auth/login)
- KPIs, interactive table with filters, bar chart top 10
- Form to create champions via POST /api/champions
- Business logic extracted to logic.py (pure functions)
- 12+ pytest tests for all business logic
- CI updated: Java (mvn verify) + Python (pytest + ruff)"

# Push i crear PR
git push origin feature/week13-streamlit-dashboard

gh pr create --title "feat(dashboard): Streamlit dashboard amb JWT" --body "$(cat <<'EOF'
## Resum
- Dashboard Streamlit amb login JWT, KPIs, taula, gràfic i formulari
- Lògica extreta a funcions pures testejables (logic.py)
- 12+ tests pytest + CI ampliat per Python

## Test Plan
- [ ] pytest test_logic.py -v passa
- [ ] ruff check dashboard/ passa
- [ ] E2E manual: login → dashboard → cerca → crea campió → logout
- [ ] CI verd (Java + Python)
EOF
)"
```

---

## Checklist de Lliurament

- [ ] Dashboard complet: login JWT, KPIs, taula amb cerca, gràfic, formulari, gestió d'errors
- [ ] Test E2E manual completat (12 passos verificats)
- [ ] `dashboard-spec.md` actualitzat per reflectir el dashboard final
- [ ] Demo practicada (3 minuts): arquitectura, demo en viu, codi rellevant
- [ ] Commit amb missatge descriptiu (conventional commits)
- [ ] Pull Request creat amb descripció i test plan
- [ ] CI verd: Java (mvn verify) + Python (pytest + ruff)
