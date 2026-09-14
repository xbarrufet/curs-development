# Setmana 13 — Dilluns: Introducció a Streamlit

## Objectiu del Dia

Crear la primera versió del dashboard d'EsportsPulse amb Streamlit, connectant-lo a l'API REST Java per mostrar la llista de campions en una taula interactiva amb cerca i botó de recàrrega. Al final del dia has de poder executar `streamlit run dashboard/app.py` i veure dades reals des de l'API.

---

## Teoria

### Per Què Streamlit?

Streamlit és un framework Python per crear aplicacions web interactives sense necessitat de conèixer HTML, CSS ni JavaScript. Els seus avantatges principals són:

- **Prototipatge ràpid** — En menys de 50 línies tens un dashboard funcional
- **Python-natiu** — Tot el codi és Python pur, sense capes de frontend
- **Sense coneixement frontend** — No cal saber React, Angular ni Vue
- **Ideal per a dades** — Integra taules, gràfics i mètriques de forma nativa

### Model d'Execució de Streamlit

Streamlit funciona de manera molt diferent a frameworks com React o Angular:

> **Cada interacció de l'usuari re-executa tot l'script de dalt a baix.**

Això vol dir:
- **No hi ha event handlers** — No escrius `onClick` ni `onChange`
- **No hi ha estat implícit** — Les variables locals es reinicien cada cop
- **L'estat persistent** es guarda a `st.session_state` (ho veurem dimarts)
- **El flux és seqüencial** — El codi s'executa línia per línia, de dalt a baix

```python
# Exemple: model d'execució de Streamlit
# IMPORTANT: aquest codi s'executa SENCER cada cop que l'usuari interactua
import streamlit as st

st.title("Hola!")          # Es renderitza cada cop
nom = st.text_input("Nom") # Mostra l'input, retorna el valor actual
if nom:                     # Si l'usuari ha escrit alguna cosa...
    st.write(f"Hola, {nom}!")  # ...mostra el salut
# Quan l'usuari escriu una lletra, TOT aquest script torna a executar-se
```

### Instal·lació

```bash
# Instal·lem Streamlit al projecte Python
pip install streamlit requests

# Estructura de fitxers que crearem avui
# esportspulse-engine/
# ├── dashboard/
# │   ├── app.py              ← Aplicació Streamlit principal
# │   └── requirements.txt    ← Dependències del dashboard
```

### Components Bàsics de Streamlit

```python
import streamlit as st

# --- Títols i text ---
st.title("Títol principal")       # Capçalera gran (h1)
st.header("Secció")               # Capçalera mitjana (h2)
st.subheader("Subsecció")         # Capçalera petita (h3)
st.text("Text pla")               # Text sense format
st.markdown("**Negreta** i *cursiva*")  # Suporta Markdown complet

# --- Inputs ---
# text_input retorna el text que l'usuari ha escrit (string buit si no ha escrit res)
nom = st.text_input("Nom del campió", placeholder="Ex: Ahri")

# number_input retorna el nombre seleccionat
min_wr = st.number_input("WinRate mínim", min_value=0.0, max_value=100.0, value=50.0)

# button retorna True NOMÉS en el moment que es clica (False la resta del temps)
if st.button("Cerca"):
    st.write("Cercant...")

# --- Taules ---
import pandas as pd

# st.dataframe mostra una taula interactiva (ordenable, cercable)
df = pd.DataFrame({"Champion": ["Ahri", "Jinx"], "WinRate": [52.3, 51.1]})
st.dataframe(df, use_container_width=True)  # use_container_width per ocupar tot l'ample
```

---

## Activitat

### Pas 1: Configurar l'Estructura

Crea el directori `dashboard/` dins del projecte i el fitxer de dependències:

```txt
# dashboard/requirements.txt
# Dependències del dashboard Streamlit
streamlit>=1.30.0
requests>=2.31.0
pandas>=2.0.0
```

```bash
# Instal·la les dependències
cd esportspulse-engine
pip install -r dashboard/requirements.txt
```

### Pas 2: Connectar amb l'API REST Java

L'API REST que vam crear a la Setmana 9 exposa `GET /api/champions`. Farem servir `requests` per cridar-la:

```python
# dashboard/app.py — Primera versió del dashboard EsportsPulse
import streamlit as st
import requests
import pandas as pd

# --- Configuració de la pàgina ---
# page_config ha de ser la PRIMERA crida Streamlit del fitxer
st.set_page_config(
    page_title="EsportsPulse Dashboard",  # Títol de la pestanya del navegador
    page_icon="🎮",                        # Icona de la pestanya
    layout="wide"                          # Layout ample per aprofitar la pantalla
)

# --- Constants ---
# URL base de l'API Java (Spring Boot corre al port 8080 per defecte)
API_BASE_URL = "http://localhost:8080/api"


def obtenir_campions():
    """
    Crida GET /api/champions a l'API Java.
    Retorna una llista de diccionaris amb les dades dels campions.
    Si l'API no respon, retorna una llista buida i mostra un error.
    """
    try:
        # Fem la petició GET amb un timeout de 5 segons
        # (evitem que el dashboard es quedi penjat si l'API no respon)
        resposta = requests.get(f"{API_BASE_URL}/champions", timeout=5)

        # raise_for_status llança una excepció si el codi HTTP és 4xx o 5xx
        resposta.raise_for_status()

        # Retornem el JSON parsejat (llista de diccionaris)
        return resposta.json()
    except requests.exceptions.ConnectionError:
        # L'API no està engegada o no és accessible
        st.error("No s'ha pogut connectar amb l'API. Assegura't que el backend Java està engegat.")
        return []
    except requests.exceptions.Timeout:
        # L'API triga massa a respondre
        st.warning("L'API triga massa a respondre. Torna-ho a intentar.")
        return []
    except requests.exceptions.HTTPError as e:
        # L'API ha retornat un error (4xx, 5xx)
        st.error(f"Error de l'API: {e.response.status_code} — {e.response.text}")
        return []


# --- Capçalera del Dashboard ---
st.title("EsportsPulse Dashboard")
st.markdown("Plataforma d'analítica de League of Legends — Dades en temps real des de l'API")

# --- Barra de cerca i botó de recàrrega ---
# Utilitzem columnes per posar el camp de cerca i el botó al costat
col_cerca, col_boto = st.columns([3, 1])

with col_cerca:
    # text_input retorna el text actual cada cop que l'script es re-executa
    # La key és important per mantenir l'estat entre re-execucions
    cerca = st.text_input(
        "Cerca per nom de campió",
        placeholder="Escriu un nom... (ex: Ahri, Jinx)",
        key="cerca_campions"
    )

with col_boto:
    # Afegim un espai vertical per alinear el botó amb l'input
    st.write("")  # Espai en blanc per alineació
    st.write("")
    # El botó retorna True quan es clica, False la resta del temps
    # Com que Streamlit re-executa tot l'script, clicar el botó
    # provoca una nova càrrega de dades automàticament
    recarregar = st.button("Actualitza dades", type="primary")

# --- Obtenir i mostrar dades ---
# Cada cop que l'script es re-executa (cerca canvia, botó clicat, etc.)
# tornem a cridar l'API per obtenir dades fresques
campions = obtenir_campions()

if campions:
    # Convertim la llista de diccionaris a DataFrame de Pandas
    # Pandas ens permet filtrar i manipular dades fàcilment
    df = pd.DataFrame(campions)

    # --- Filtre de cerca ---
    if cerca:
        # Filtrem pel nom del campió (case-insensitive)
        # str.contains amb case=False fa cerca parcial sense importar majúscules
        df = df[df["name"].str.contains(cerca, case=False, na=False)]
        st.info(f"Mostrant {len(df)} campions que coincideixen amb '{cerca}'")

    # --- Taula de resultats ---
    st.subheader(f"Campions ({len(df)} resultats)")

    # st.dataframe renderitza una taula interactiva
    # L'usuari pot ordenar per qualsevol columna clicant la capçalera
    st.dataframe(
        df,
        use_container_width=True,   # Ocupa tot l'ample disponible
        hide_index=True,            # Amaga l'índex numèric de Pandas
        column_config={             # Configurem la visualització de columnes
            "name": st.column_config.TextColumn("Campió", width="medium"),
            "winRate": st.column_config.NumberColumn("Win Rate (%)", format="%.1f%%"),
            "gamesPlayed": st.column_config.NumberColumn("Partides", format="%d"),
            "role": st.column_config.TextColumn("Rol", width="small"),
        }
    )
else:
    # Si no hi ha campions (API offline o error), mostrem un avís
    st.info("No hi ha dades disponibles. Comprova que l'API està funcionant.")
```

### Pas 3: Executar el Dashboard

```bash
# Engega primer el backend Java (en un terminal separat)
cd esportspulse-engine/backend-java
mvn spring-boot:run

# En un altre terminal, engega el dashboard Streamlit
cd esportspulse-engine
streamlit run dashboard/app.py
# S'obrirà automàticament al navegador a http://localhost:8501
```

---

## Checklist de Lliurament

- [ ] `dashboard/app.py` creat i funcional
- [ ] El dashboard mostra la llista de campions obtinguda des de l'API REST Java (`GET /api/champions`)
- [ ] El camp de cerca filtra campions per nom (case-insensitive, cerca parcial)
- [ ] El botó "Actualitza dades" recarrega les dades des de l'API
- [ ] Si l'API no està disponible, es mostra un missatge d'error clar (no una excepció)
- [ ] `dashboard/requirements.txt` creat amb les dependències
- [ ] El dashboard s'executa correctament amb `streamlit run dashboard/app.py`
