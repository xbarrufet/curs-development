# Setmana 13 — Dijous: Testing End-to-End amb el Full Stack

## Objectiu del Dia

Extreure la lògica de negoci del dashboard a funcions pures testejables, escriure tests amb pytest i ampliar el CI per validar Java i Python. Al final del dia, tindràs 4+ tests passant i un CI que valida tot el projecte.

---

## Teoria

### La Piràmide de Testing

```
         /\             E2E (pocs, cars, lents)
        /  \            → Sistema complet: UI + API + BD
       /----\
      /      \          Integració (alguns)
     /--------\         → Interacció entre components
    /          \
   / Unit Tests \       Unit (molts, barats, ràpids)
  /--------------\      → Funcions individuals aïllades
```

Avui: **unit tests** per la lògica de negoci + **E2E manual** pel flux complet.

### Per Què Extreure Lògica de Streamlit?

Streamlit barreja UI i lògica. `st.text_input()` necessita un navegador, `st.dataframe()` renderitza HTML. **No es poden testejar amb pytest.** Solució: extreure la lògica a funcions pures:

```python
# MAL: barreja UI i lògica (no testejable)
def mostrar():
    cerca = st.text_input("Cerca")    # Depèn de Streamlit!
    campions = [c for c in data if cerca.lower() in c["name"].lower()]
    st.dataframe(campions)             # Depèn de Streamlit!

# BÉ: lògica pura separada (testejable amb pytest)
def filter_champions(champions: list[dict], search: str) -> list[dict]:
    if not search:
        return champions
    return [c for c in champions if search.lower() in c["name"].lower()]
```

---

## Activitat

### Pas 1: Mòdul de Lògica de Negoci

```python
# dashboard/logic.py — Funcions pures sense dependències de Streamlit

def filter_champions(champions: list[dict], search: str = "",
                     min_wr: float = 0.0, max_wr: float = 100.0) -> list[dict]:
    """Filtra campions per nom (parcial, case-insensitive) i rang de WinRate."""
    resultat = champions
    if search:
        resultat = [c for c in resultat if search.lower() in c.get("name", "").lower()]
    resultat = [c for c in resultat if c.get("winRate", 0) >= min_wr]
    resultat = [c for c in resultat if c.get("winRate", 0) <= max_wr]
    return resultat

def calculate_kpis(champions: list[dict]) -> dict:
    """Calcula KPIs: total, avg winRate, campió més popular."""
    if not champions:
        return {"total": 0, "avg_win_rate": 0.0, "most_popular": "N/A",
                "most_popular_games": 0}
    total = len(champions)
    avg_wr = round(sum(c.get("winRate", 0) for c in champions) / total, 1)
    popular = max(champions, key=lambda c: c.get("gamesPlayed", 0))
    return {"total": total, "avg_win_rate": avg_wr,
            "most_popular": popular.get("name", "Desconegut"),
            "most_popular_games": popular.get("gamesPlayed", 0)}

def format_win_rate(win_rate: float) -> str:
    """Formata winRate com a string amb 1 decimal i %."""
    return f"{win_rate:.1f}%"

def get_top_champions(champions: list[dict], n: int = 10,
                      sort_by: str = "gamesPlayed") -> list[dict]:
    """Retorna els N campions amb valor més alt del camp indicat."""
    return sorted(champions, key=lambda c: c.get(sort_by, 0), reverse=True)[:n]

def validate_champion_form(name: str, role: str, win_rate: float,
                           games_played: int) -> list[str]:
    """Valida dades del formulari. Retorna llista d'errors (buida si OK)."""
    errors = []
    if not name or not name.strip():
        errors.append("El nom del campió és obligatori.")
    if role not in ["Top", "Jungle", "Mid", "ADC", "Support"]:
        errors.append(f"El rol ha de ser un de: Top, Jungle, Mid, ADC, Support")
    if win_rate < 0 or win_rate > 100:
        errors.append("El WinRate ha d'estar entre 0 i 100.")
    if games_played < 0:
        errors.append("El nombre de partides no pot ser negatiu.")
    return errors
```

### Pas 2: Tests amb Pytest

```python
# dashboard/test_logic.py — Tests unitaris per a la lògica de negoci
import pytest
from logic import filter_champions, calculate_kpis, format_win_rate, validate_champion_form

@pytest.fixture
def campions():
    """Dades d'exemple reutilitzables per als tests."""
    return [
        {"name": "Ahri", "role": "Mid", "winRate": 52.3, "gamesPlayed": 150000},
        {"name": "Jinx", "role": "ADC", "winRate": 51.1, "gamesPlayed": 120000},
        {"name": "Lux", "role": "Support", "winRate": 50.8, "gamesPlayed": 110000},
        {"name": "Yasuo", "role": "Mid", "winRate": 49.2, "gamesPlayed": 105000},
        {"name": "Thresh", "role": "Support", "winRate": 50.5, "gamesPlayed": 100000},
    ]

class TestFilterChampions:
    def test_filtra_per_nom_parcial(self, campions):
        # "ah" hauria de trobar "Ahri" (cerca parcial)
        resultat = filter_champions(campions, search="ah")
        assert len(resultat) == 1
        assert resultat[0]["name"] == "Ahri"

    def test_filtra_case_insensitive(self, campions):
        # "JINX" hauria de trobar "Jinx"
        resultat = filter_champions(campions, search="JINX")
        assert len(resultat) == 1
        assert resultat[0]["name"] == "Jinx"

    def test_filtra_per_winrate_minim(self, campions):
        # Campions amb WR >= 51.0: Ahri (52.3) i Jinx (51.1)
        resultat = filter_champions(campions, min_wr=51.0)
        assert len(resultat) == 2

    def test_cerca_buida_retorna_tots(self, campions):
        resultat = filter_champions(campions, search="")
        assert len(resultat) == 5

    def test_cerca_sense_resultats(self, campions):
        resultat = filter_champions(campions, search="Teemo")
        assert len(resultat) == 0

    def test_filtres_combinats(self, campions):
        # "u" trobaria Lux i Yasuo, però amb min_wr=50 exclou Yasuo (49.2)
        resultat = filter_champions(campions, search="u", min_wr=50.0)
        assert len(resultat) == 1
        assert resultat[0]["name"] == "Lux"

class TestCalculateKpis:
    def test_kpis_amb_dades(self, campions):
        kpis = calculate_kpis(campions)
        assert kpis["total"] == 5
        assert kpis["avg_win_rate"] == 50.8  # (52.3+51.1+50.8+49.2+50.5)/5
        assert kpis["most_popular"] == "Ahri"  # Més partides: 150K

    def test_kpis_llista_buida(self):
        # Cas límit: si l'API no retorna campions, no ha de petar
        kpis = calculate_kpis([])
        assert kpis["total"] == 0
        assert kpis["avg_win_rate"] == 0.0
        assert kpis["most_popular"] == "N/A"

class TestFormatWinRate:
    def test_format_normal(self):
        assert format_win_rate(52.345) == "52.3%"

    def test_format_zero(self):
        assert format_win_rate(0) == "0.0%"

class TestValidateChampionForm:
    def test_formulari_valid(self):
        errors = validate_champion_form("Morgana", "Support", 51.5, 80000)
        assert errors == []

    def test_nom_buit(self):
        errors = validate_champion_form("", "Mid", 50.0, 1000)
        assert len(errors) == 1
        assert "obligatori" in errors[0].lower()
```

```bash
# Executar tests i linter
cd esportspulse-engine/dashboard
pytest test_logic.py -v    # Hauria de mostrar tots els tests PASSED
ruff check dashboard/       # Verifica estil del codi Python
```

### Pas 3: Ampliar el CI per Python

```yaml
# .github/workflows/ci.yml — CI Full Stack (Java + Python)
name: CI Full Stack
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  java-tests:  # Job existent des de setmanes anteriors
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '21', distribution: 'temurin' }
      - run: cd backend-java && mvn verify  # Tests + checkstyle

  python-tests:  # NOU — afegit avui
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - name: Install dependencies
        run: pip install -r dashboard/requirements.txt pytest ruff
      - name: Ruff (linter)
        run: ruff check dashboard/
      - name: Pytest (unit tests)
        run: cd dashboard && pytest test_logic.py -v
```

### Pas 4: E2E Manual

Engega els 3 serveis i verifica el flux complet:

```bash
# Terminal 1: Java backend
cd esportspulse-engine/backend-java && mvn spring-boot:run
# Terminal 2: Python service
cd esportspulse-engine/backend-python && uvicorn main:app --reload --port 8000
# Terminal 3: Streamlit
cd esportspulse-engine && streamlit run dashboard/app.py
```

```markdown
# Script E2E Manual — marca cada pas:
1. [ ] Obre http://localhost:8501 → Veus Login
2. [ ] Credencials incorrectes → Missatge d'error
3. [ ] Credencials correctes → Dashboard amb dades
4. [ ] KPIs mostren valors coherents
5. [ ] Cerca "Ahri" → Taula filtrada
6. [ ] Crea campió nou → Confirmació verda
7. [ ] Nou campió apareix a la taula
8. [ ] Tancar sessió → Torna al Login
9. [ ] Para backend (Ctrl+C) → Missatge d'error de connexió
```

---

## Checklist de Lliurament

- [ ] `dashboard/logic.py` creat amb funcions pures: `filter_champions()`, `calculate_kpis()`, `format_win_rate()`, `get_top_champions()`, `validate_champion_form()`
- [ ] `dashboard/test_logic.py` creat amb 4+ tests unitaris
- [ ] Tots els tests passen amb `pytest test_logic.py -v`
- [ ] `ruff check dashboard/` passa sense errors
- [ ] `.github/workflows/ci.yml` actualitzat amb job `python-tests` (pytest + ruff)
- [ ] CI valida Java (mvn verify) i Python (pytest + ruff)
- [ ] Test E2E manual executat i flux complet verificat
