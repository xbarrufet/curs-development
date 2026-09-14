# Setmana 23 — Dilluns: Tests d'Integracio del Flux Complet

## Objectiu del Dia

Escriure una suite de tests end-to-end (E2E) que exerciti el flux complet de l'aplicacio: login, cerca, knowledge retrieval i resposta d'agent. Al final del dia, has de tenir tests automatitzats que verifiquin que tots els serveis funcionen junts correctament.

---

## Teoria

### Piramide de Tests

Fins ara has escrit tests unitaris (una funcio) i tests d'integracio (un servei). Ara toca el nivell superior: tests E2E.

```
#          /\
#         /E2E\         <- Pocs, lents, cobreixen fluxos complets
#        /------\
#       /Integracio\    <- Moderats, verifiquen connexions entre parts
#      /------------\
#     / Unitaris     \  <- Molts, rapids, cobreixen funcions individuals
#    /________________\
#
# E2E: Simula el que fa un usuari real
# - Fa login
# - Navega per l'aplicacio
# - Fa accions (cerques, consultes)
# - Verifica els resultats
#
# Son lents (segons, no mil·lisegons) pero imprescindibles
# per assegurar que tot funciona junt.
```

### Que es un Test E2E?

Un test E2E simula un flux complet d'un usuari:

```python
# Exemple conceptual d'un test E2E
# No testa una funcio, sino tot un escenari d'us

def test_user_can_search_and_get_agent_analysis():
    """
    Escenari: Un usuari vol analitzar les estadistiques d'un equip.
    
    Flux:
    1. L'usuari fa login
    2. Busca un equip per nom
    3. Selecciona l'equip
    4. Demana a l'agent quantitatiu una analisi
    5. Rep una resposta amb dades estadistiques
    """
    # 1. Login — obte un token JWT valid
    token = login("test_user", "password123")
    assert token is not None
    
    # 2. Cerca — busca un equip
    teams = search_teams(token, query="T1")
    assert len(teams) > 0
    assert teams[0]["name"] == "T1"
    
    # 3. Consulta a l'agent — demana analisi
    response = query_agent(
        token,
        agent="quantitatiu",
        query="Quina es la taxa de victoria de T1 aquesta temporada?"
    )
    
    # 4. Verificacio — la resposta te sentit
    assert response["status"] == "success"
    assert "taxa" in response["answer"].lower() or "winrate" in response["answer"].lower()
    assert response["sources"] is not None  # L'agent cita les fonts
```

### Eines per a Tests E2E

**Per a APIs (el nostre cas):**

```python
# Usarem requests + pytest per testar les APIs
# No necessitem Selenium ni eines de navegador
# perque el nostre frontend es Streamlit (testa via API)

import requests
import pytest

class TestAPI:
    """Base class per a tests E2E de l'API."""
    
    # URL base — configurable per entorn
    BASE_URL = os.environ.get("TEST_BASE_URL", "http://localhost:8080")
    AI_URL = os.environ.get("TEST_AI_URL", "http://localhost:8000")
    
    def get_token(self, username: str, password: str) -> str:
        """Obte un token JWT fent login.
        
        Args:
            username: Nom d'usuari
            password: Contrasenya
        
        Returns:
            Token JWT com a string
        
        Raises:
            AssertionError: Si el login falla
        """
        response = requests.post(
            f"{self.BASE_URL}/api/v1/auth/login",
            json={"username": username, "password": password}
        )
        assert response.status_code == 200, f"Login fallit: {response.text}"
        return response.json()["token"]
    
    def auth_headers(self, token: str) -> dict:
        """Genera les capcaleres d'autenticacio amb el token JWT.
        
        Args:
            token: Token JWT valid
        
        Returns:
            Dict amb la capcalera Authorization
        """
        return {"Authorization": f"Bearer {token}"}
```

### Fixtures de Pytest per a E2E

```python
# conftest.py — Fixtures compartides per a tots els tests E2E

import pytest
import requests
import os

@pytest.fixture(scope="session")
def base_url():
    """URL base del backend. Configurable per variable d'entorn."""
    return os.environ.get("TEST_BASE_URL", "http://localhost:8080")

@pytest.fixture(scope="session")
def ai_url():
    """URL base del servei d'IA."""
    return os.environ.get("TEST_AI_URL", "http://localhost:8000")

@pytest.fixture(scope="session")
def auth_token(base_url):
    """Obte un token JWT valid per a tota la sessio de tests.
    
    scope='session' vol dir que el token es reutilitza per a tots
    els tests, en comptes de fer login per a cadascun.
    """
    response = requests.post(
        f"{base_url}/api/v1/auth/login",
        json={"username": "test_user", "password": "test_password"}
    )
    assert response.status_code == 200, "No s'ha pogut fer login per als tests"
    return response.json()["token"]

@pytest.fixture(scope="session")
def auth_headers(auth_token):
    """Capcaleres HTTP amb el token JWT."""
    return {"Authorization": f"Bearer {auth_token}"}
```

### Estrategia: Que Testar i Que No

```
# TESTAR (fluxos critics):
# - Login complet (credencials valides i invalides)
# - CRUD d'equips (crear, llegir, actualitzar, eliminar)
# - Cerca d'equips i jugadors
# - Consulta a l'agent quantitatiu
# - Consulta a l'agent knowledge
# - Paginacio de resultats
# - Gestio d'errors (404, 401, 400)

# NO TESTAR a E2E (ja cobert per unitaris/integracio):
# - Logica interna de cada funcio
# - Validacio de camps individuals
# - Formatejat de dades
# - Caching intern
```

---

## Activitat

### 1. Configurar l'Entorn de Tests E2E (15 min)

```bash
# Crea el directori per als tests E2E
mkdir -p tests/e2e

# Crea el fitxer de configuracio de pytest
# pytest.ini o pyproject.toml ja existents, afegeix:
```

Crea `tests/e2e/conftest.py` amb les fixtures de la teoria.

### 2. Test E2E: Flux d'Autenticacio (20 min)

Crea `tests/e2e/test_auth_flow.py`:

```python
"""
Tests E2E per al flux d'autenticacio.
Verifica login, acces autenticat, i rebuig sense token.
"""
import requests
import pytest

class TestAuthFlow:
    """Tests del flux complet d'autenticacio."""
    
    def test_login_with_valid_credentials(self, base_url):
        """Verifica que un usuari pot fer login i rebre un token."""
        response = requests.post(
            f"{base_url}/api/v1/auth/login",
            json={"username": "test_user", "password": "test_password"}
        )
        
        assert response.status_code == 200
        data = response.json()
        assert "token" in data
        assert len(data["token"]) > 0
    
    def test_login_with_invalid_credentials(self, base_url):
        """Verifica que credencials invalides retornen 401."""
        response = requests.post(
            f"{base_url}/api/v1/auth/login",
            json={"username": "fake_user", "password": "wrong_password"}
        )
        
        assert response.status_code == 401
    
    def test_protected_endpoint_without_token(self, base_url):
        """Verifica que un endpoint protegit rebutja peticions sense token."""
        response = requests.get(f"{base_url}/api/v1/teams")
        
        assert response.status_code == 401 or response.status_code == 403
    
    def test_protected_endpoint_with_token(self, base_url, auth_headers):
        """Verifica que un endpoint protegit accepta peticions amb token valid."""
        response = requests.get(
            f"{base_url}/api/v1/teams",
            headers=auth_headers
        )
        
        assert response.status_code == 200
```

### 3. Test E2E: Flux de Cerca i Dades (25 min)

Crea `tests/e2e/test_search_flow.py`:

```python
"""
Tests E2E per al flux de cerca d'equips i jugadors.
Exercita el backend Java i la base de dades.
"""
import requests
import pytest

class TestSearchFlow:
    """Tests del flux complet de cerca."""
    
    def test_search_teams_returns_results(self, base_url, auth_headers):
        """Verifica que la cerca d'equips retorna resultats."""
        response = requests.get(
            f"{base_url}/api/v1/teams",
            headers=auth_headers,
            params={"search": "T1"}
        )
        
        assert response.status_code == 200
        data = response.json()
        # Ha de retornar almenys un resultat
        assert len(data) > 0 or "content" in data
    
    def test_get_team_details(self, base_url, auth_headers):
        """Verifica que es poden obtenir els detalls d'un equip."""
        # Primer, obtenir la llista d'equips
        teams_response = requests.get(
            f"{base_url}/api/v1/teams",
            headers=auth_headers
        )
        assert teams_response.status_code == 200
        
        teams = teams_response.json()
        if isinstance(teams, dict) and "content" in teams:
            teams = teams["content"]  # Paginacio Spring Boot
        
        if len(teams) > 0:
            # Obtenir detalls del primer equip
            team_id = teams[0]["id"]
            detail_response = requests.get(
                f"{base_url}/api/v1/teams/{team_id}",
                headers=auth_headers
            )
            
            assert detail_response.status_code == 200
            detail = detail_response.json()
            assert "name" in detail
    
    def test_pagination_works(self, base_url, auth_headers):
        """Verifica que la paginacio retorna resultats correctes."""
        # Pagina 0 amb 5 elements
        response = requests.get(
            f"{base_url}/api/v1/teams",
            headers=auth_headers,
            params={"page": 0, "size": 5}
        )
        
        assert response.status_code == 200
```

### 4. Test E2E: Flux d'Agents (30 min)

Crea `tests/e2e/test_agent_flow.py`:

```python
"""
Tests E2E per al flux de consultes als agents.
Exercita el servei Python, els agents LangChain, i la comunicacio amb el backend.
"""
import requests
import pytest
import time

class TestAgentFlow:
    """Tests del flux complet de consultes als agents."""
    
    def test_quantitative_agent_responds(self, ai_url, auth_headers):
        """Verifica que l'agent quantitatiu respon a una consulta."""
        response = requests.post(
            f"{ai_url}/api/v1/agents/quantitatiu/query",
            headers=auth_headers,
            json={"query": "Quina es la taxa de victoria de T1?"},
            timeout=60  # Els agents poden trigar mes
        )
        
        assert response.status_code == 200
        data = response.json()
        assert "answer" in data or "response" in data
    
    def test_knowledge_agent_responds(self, ai_url, auth_headers):
        """Verifica que l'agent knowledge respon a una consulta."""
        response = requests.post(
            f"{ai_url}/api/v1/agents/knowledge/query",
            headers=auth_headers,
            json={"query": "Explica l'estrategia de T1 al Worlds 2024"},
            timeout=60
        )
        
        assert response.status_code == 200
        data = response.json()
        assert "answer" in data or "response" in data
    
    def test_full_user_journey(self, base_url, ai_url, auth_headers):
        """Test del recorregut complet d'un usuari.
        
        Simula: login -> cerca equip -> consulta agent -> verifica resposta.
        Aquest es el test mes important: exercita TOT el sistema.
        """
        # Pas 1: Cercar un equip
        teams = requests.get(
            f"{base_url}/api/v1/teams",
            headers=auth_headers,
            params={"search": "T1"}
        ).json()
        
        # Pas 2: Consultar l'agent sobre l'equip trobat
        agent_response = requests.post(
            f"{ai_url}/api/v1/agents/quantitatiu/query",
            headers=auth_headers,
            json={
                "query": "Dona'm un resum estadistic d'aquest equip",
                "context": {"team": "T1"}
            },
            timeout=60
        )
        
        assert agent_response.status_code == 200
        # La resposta ha de contenir dades, no nomes text buit
        data = agent_response.json()
        answer = data.get("answer", data.get("response", ""))
        assert len(answer) > 50  # Resposta substancial
```

### 5. Executar els Tests (15 min)

```bash
# Assegura que els serveis estan arrenquen
docker-compose up -d

# Espera que els serveis estiguin llestos
sleep 10

# Executa els tests E2E
# -v: verbose (mostra cada test)
# -s: mostra els prints (util per depurar)
# --tb=short: tracebacks curts
cd tests/e2e
pytest -v -s --tb=short

# Per executar contra produccio:
TEST_BASE_URL=https://api.esportspulse.dev \
TEST_AI_URL=https://ai.esportspulse.dev \
pytest -v -s --tb=short
```

### 6. Commit (10 min)

```bash
git add tests/e2e/
git commit -m "test: add E2E test suite for complete user journey"
```

---

## Checklist de Lliurament

- [ ] Directori `tests/e2e/` creat amb `conftest.py`
- [ ] Tests d'autenticacio (login valid, invalid, sense token)
- [ ] Tests de cerca (equips, detalls, paginacio)
- [ ] Tests d'agents (quantitatiu, knowledge)
- [ ] Test del recorregut complet (full user journey)
- [ ] Tots els tests passen en local
- [ ] Tests executables contra produccio (URLs configurables)
- [ ] Commit amb la suite de tests E2E
