# Setmana 21 — Dimecres: Dashboard Avancat: Visualitzacio de l'Equip d'Agents

## Objectiu del Dia

Construir pagines avancades al dashboard Streamlit que mostrin l'estat dels agents, l'historial de consultes, metriques de qualitat i desglossament de costos. Al final del dia, tindras un panell de control complet per monitoritzar el sistema d'agents.

---

## Teoria

### Per Que un Dashboard d'Agents?

Quan tens agents d'IA en produccio, necessites visibilitat sobre:
- **Estat**: Estan funcionant? Quant triguen a respondre?
- **Qualitat**: Les respostes son bones? Els usuaris estan satisfets?
- **Cost**: Quant gastem en crides a l'API de Claude?
- **Historial**: Quines consultes rep el sistema? Quins patrons hi ha?

Sense aquesta visibilitat, els agents son una caixa negra. I les caixes negres en produccio son perilloses.

### LangFuse: Observabilitat per a LLMs

LangFuse es una plataforma d'observabilitat especifica per a aplicacions amb LLMs. Captura:

```python
# Exemple de tracing amb LangFuse
# LangFuse intercepta les crides als LLMs i registra:
# - Input (prompt enviat)
# - Output (resposta rebuda)
# - Latencia (temps de resposta)
# - Tokens consumits (input + output)
# - Cost estimat (basat en preus del model)

from langfuse import Langfuse

# Inicialitzacio del client de LangFuse
# Les credencials venen de variables d'entorn
langfuse = Langfuse(
    public_key=os.environ["LANGFUSE_PUBLIC_KEY"],
    secret_key=os.environ["LANGFUSE_SECRET_KEY"],
    host=os.environ.get("LANGFUSE_HOST", "https://cloud.langfuse.com")
)

# Crear una traca per a cada peticio d'usuari
# La traca agrupa totes les operacions d'una peticio
trace = langfuse.trace(
    name="agent-quantitatiu",      # Nom de l'agent
    user_id=user_id,                # Identificador de l'usuari
    metadata={"query": user_query}  # Dades adicionals
)
```

### Metriques Clau per a Agents

**1. Latencia (temps de resposta):**

```python
# Mesurar el temps que triga l'agent a respondre
import time

# Registrem el temps inicial
start_time = time.time()

# Executem l'agent (pot fer multiples crides a l'LLM)
response = agent.invoke({"query": user_query})

# Calculem la latencia total en segons
latency = time.time() - start_time

# Enviem la metrica a Prometheus
AGENT_LATENCY.labels(agent="quantitatiu").observe(latency)
```

**2. Cost per consulta:**

```python
# Calcul de cost basat en tokens consumits
# Preus aproximats per a Claude Sonnet (pot variar)
PRICE_PER_INPUT_TOKEN = 0.003 / 1000    # $0.003 per 1K tokens input
PRICE_PER_OUTPUT_TOKEN = 0.015 / 1000   # $0.015 per 1K tokens output

def calculate_cost(input_tokens: int, output_tokens: int) -> float:
    """Calcula el cost en dolars d'una crida a l'LLM.
    
    Args:
        input_tokens: Nombre de tokens al prompt (entrada)
        output_tokens: Nombre de tokens a la resposta (sortida)
    
    Returns:
        Cost total en dolars USD
    """
    input_cost = input_tokens * PRICE_PER_INPUT_TOKEN
    output_cost = output_tokens * PRICE_PER_OUTPUT_TOKEN
    return input_cost + output_cost
```

**3. Qualitat de resposta:**

```python
# Metriques de qualitat que podem mesurar automaticament
# Feedback de l'usuari: puntuacio 1-5 o thumbs up/down
# Completesa: l'agent ha respost la pregunta?
# Fonts citades: l'agent ha proporcionat fonts?

class QualityMetrics:
    """Recopila metriques de qualitat per a cada resposta d'agent."""
    
    def __init__(self):
        # Comptador de respostes per puntuacio
        self.ratings = {1: 0, 2: 0, 3: 0, 4: 0, 5: 0}
        # Historial de latenties per calcular percentils
        self.latencies = []
    
    def add_rating(self, score: int):
        """Registra una puntuacio d'usuari (1-5)."""
        self.ratings[score] += 1
    
    def average_rating(self) -> float:
        """Retorna la puntuacio mitjana de totes les respostes."""
        total = sum(k * v for k, v in self.ratings.items())
        count = sum(self.ratings.values())
        return total / count if count > 0 else 0.0
```

### Estructura del Dashboard Avancat

El dashboard tindra quatre pagines noves:

```
pages/
  01_Resum_General.py      # (ja existent) Resum basic
  02_Estat_Agents.py        # NOU: estat en temps real
  03_Historial_Consultes.py # NOU: log de totes les consultes
  04_Metriques_Qualitat.py  # NOU: grafics de qualitat
  05_Costos.py              # NOU: desglossament de costos
```

---

## Activitat

### 1. Pagina d'Estat dels Agents (30 min)

Crea `streamlit/pages/02_Estat_Agents.py`:

```python
"""
Pagina de dashboard: Estat dels Agents
Mostra l'estat actual de cada agent del sistema.
"""
import streamlit as st
import requests
from datetime import datetime

# Titol de la pagina amb icona d'estat
st.title("Estat dels Agents")

# Configuracio dels agents a monitoritzar
# Cada agent te un nom, URL de health check i descripcio
AGENTS = [
    {
        "name": "Agent Quantitatiu",
        "health_url": "http://ai-python:8000/health/quantitatiu",
        "description": "Analisi estadistic d'equips i jugadors"
    },
    {
        "name": "Agent Knowledge",
        "health_url": "http://ai-python:8000/health/knowledge",
        "description": "Recuperacio de coneixement i resums"
    }
]

def check_agent_health(url: str) -> dict:
    """Comprova l'estat d'un agent fent una peticio al seu health endpoint.
    
    Args:
        url: URL del health check de l'agent
    
    Returns:
        Dict amb status ('healthy'/'unhealthy') i detalls
    """
    try:
        # Timeout de 5 segons per evitar bloquejos
        response = requests.get(url, timeout=5)
        if response.status_code == 200:
            return {"status": "healthy", "details": response.json()}
        else:
            return {"status": "unhealthy", "details": f"HTTP {response.status_code}"}
    except requests.exceptions.ConnectionError:
        return {"status": "unhealthy", "details": "No es pot connectar"}
    except requests.exceptions.Timeout:
        return {"status": "unhealthy", "details": "Timeout (>5s)"}

# Mostrar l'estat de cada agent en columnes
# Cada columna es un "card" visual amb informacio de l'agent
cols = st.columns(len(AGENTS))
for col, agent in zip(cols, AGENTS):
    with col:
        health = check_agent_health(agent["health_url"])
        
        # Indicador visual: verd si healthy, vermell si no
        if health["status"] == "healthy":
            st.success(f"{agent['name']}: Actiu")
        else:
            st.error(f"{agent['name']}: Inactiu")
        
        st.caption(agent["description"])
        
        # Mostrar detalls expandibles
        with st.expander("Detalls"):
            st.json(health["details"])

# Boto per refrescar manualment l'estat
if st.button("Refrescar Estat"):
    st.rerun()
```

### 2. Pagina d'Historial de Consultes (30 min)

Crea `streamlit/pages/03_Historial_Consultes.py`:

```python
"""
Pagina de dashboard: Historial de Consultes
Mostra totes les consultes fetes als agents amb filtres i cerca.
"""
import streamlit as st
import pandas as pd
from datetime import datetime, timedelta

st.title("Historial de Consultes")

# Filtres a la barra lateral
st.sidebar.subheader("Filtres")

# Filtre per agent: permet seleccionar un o tots els agents
agent_filter = st.sidebar.selectbox(
    "Agent",
    ["Tots", "Quantitatiu", "Knowledge"]
)

# Filtre per data: per defecte mostra l'ultima setmana
date_range = st.sidebar.date_input(
    "Rang de dates",
    value=(datetime.now() - timedelta(days=7), datetime.now())
)

# Obtenir dades de LangFuse o de la BD local
# Aquesta funcio faria la consulta real al teu sistema
@st.cache_data(ttl=60)  # Cache de 60 segons per no sobrecarregar
def get_queries(agent: str, start_date, end_date) -> pd.DataFrame:
    """Obte l'historial de consultes des de LangFuse.
    
    Args:
        agent: Nom de l'agent o 'Tots'
        start_date: Data d'inici del filtre
        end_date: Data de fi del filtre
    
    Returns:
        DataFrame amb les consultes filtrades
    """
    # TODO: Connectar amb LangFuse API real
    # Per ara, dades d'exemple per al desenvolupament
    pass

# Taula interactiva amb les consultes
# st.dataframe permet ordenar per columna i buscar
st.dataframe(
    get_queries(agent_filter, date_range[0], date_range[1]),
    use_container_width=True
)
```

### 3. Pagina de Metriques de Qualitat (30 min)

Crea `streamlit/pages/04_Metriques_Qualitat.py` amb grafics de:
- Puntuacio mitjana al llarg del temps (line chart)
- Distribucio de latenties (histogram)
- Percentatge de respostes amb fonts citades (bar chart)

Utilitza `st.metric()` per a les xifres principals i `st.line_chart()` o `plotly` per als grafics.

### 4. Pagina de Costos (20 min)

Crea `streamlit/pages/05_Costos.py` amb:
- Cost total del mes actual
- Cost per agent (pie chart)
- Cost per dia (line chart)
- Tokens consumits (input vs output)

### 5. Connectar amb LangFuse (30 min)

Substitueix les dades d'exemple per dades reals de LangFuse:

```python
# Afegir al docker-compose.yml el servei LangFuse
# o connectar amb LangFuse Cloud (te free tier)
# langfuse:
#   image: langfuse/langfuse:latest
#   ports:
#     - "3001:3000"
#   environment:
#     - DATABASE_URL=postgresql://...
```

### 6. Commit (10 min)

```bash
git add streamlit/pages/
git commit -m "feat(streamlit): add advanced agent monitoring dashboard pages"
```

---

## Checklist de Lliurament

- [ ] Pagina d'estat dels agents funcionant amb health checks
- [ ] Pagina d'historial de consultes amb filtres
- [ ] Pagina de metriques de qualitat amb grafics
- [ ] Pagina de costos amb desglossament per agent
- [ ] Connexio amb LangFuse (cloud o local)
- [ ] Dashboard accessible des de `http://localhost:8501`
- [ ] Commit amb totes les pagines noves
