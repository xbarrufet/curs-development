# Setmana 13 — Dimarts: Components del Dashboard — Taules, Gràfics, Formularis i Login

## Objectiu del Dia

Ampliar el dashboard amb mètriques KPI, gràfic de barres, formulari per crear campions i login amb JWT. Al final del dia, el dashboard serà una aplicació completa amb autenticació.

---

## Teoria

### Estat de Sessió amb `st.session_state`

Streamlit re-executa tot l'script cada cop. Per mantenir dades entre execucions (com el token JWT), fem servir `st.session_state`:

```python
import streamlit as st
# session_state és un diccionari persistent mentre la pestanya està oberta
if "token" not in st.session_state:
    st.session_state.token = None  # Inicialitzem només la primera vegada
st.session_state.token = "eyJhbGci..."  # Persisteix entre re-execucions
```

### Login amb JWT

A la Setmana 12 vam crear `POST /auth/login`. Ara l'usarem des de Streamlit:

```python
import requests
# Cridem l'endpoint de login amb les credencials
resposta = requests.post("http://localhost:8080/auth/login", json={
    "username": "admin", "password": "password123"
})
token = resposta.json()["token"]
# Per crides autenticades, afegim el token a la capçalera
headers = {"Authorization": f"Bearer {token}"}
campions = requests.get("http://localhost:8080/api/champions", headers=headers)
```

### KPIs, Gràfics i Formularis

```python
import streamlit as st
import pandas as pd

# --- KPIs amb st.metric (valor gran amb etiqueta i delta) ---
col1, col2, col3 = st.columns(3)
with col1:
    st.metric(label="Total Campions", value=165, delta="+3 nous")
with col2:
    st.metric(label="WinRate Mitjà", value="50.2%", delta="-0.3%")
with col3:
    st.metric(label="Més Popular", value="Jinx", delta="120K partides")

# --- Gràfic de barres des d'un DataFrame ---
df_top = pd.DataFrame({"Champion": ["Jinx", "Ahri"], "Games": [120000, 115000]})
st.bar_chart(df_top.set_index("Champion"))  # L'índex és l'eix X

# --- Formulari (només re-executa quan es clica Submit) ---
with st.form("crear_campio"):
    nom = st.text_input("Nom")
    rol = st.selectbox("Rol", ["Top", "Jungle", "Mid", "ADC", "Support"])
    if st.form_submit_button("Crear"):
        st.success(f"Campió {nom} creat!")
```

---

## Activitat

Modifica `dashboard/app.py` per tenir login, KPIs, gràfic, taula i formulari:

```python
# dashboard/app.py — Dashboard complet amb login JWT
import streamlit as st
import requests
import pandas as pd

st.set_page_config(page_title="EsportsPulse", page_icon="🎮", layout="wide")
API_BASE_URL = "http://localhost:8080"

# --- Estat de sessió (persistent entre re-execucions) ---
if "token" not in st.session_state:
    st.session_state.token = None
if "username" not in st.session_state:
    st.session_state.username = None

def fer_login(username: str, password: str) -> bool:
    """Envia credencials a POST /auth/login, guarda el token JWT."""
    try:
        resp = requests.post(f"{API_BASE_URL}/auth/login",
                             json={"username": username, "password": password}, timeout=5)
        if resp.status_code == 200:
            st.session_state.token = resp.json()["token"]
            st.session_state.username = username
            return True
        st.error("Credencials incorrectes.")
        return False
    except requests.exceptions.ConnectionError:
        st.error("No s'ha pogut connectar amb l'API.")
        return False

def fer_logout():
    """Esborra token i usuari de la sessió."""
    st.session_state.token = None
    st.session_state.username = None

def crida_api(endpoint: str, metode: str = "GET", dades: dict = None):
    """Crida autenticada a l'API. Afegeix Authorization: Bearer <token>."""
    headers = {"Authorization": f"Bearer {st.session_state.token}"}
    try:
        if metode == "GET":
            resp = requests.get(f"{API_BASE_URL}{endpoint}", headers=headers, timeout=5)
        elif metode == "POST":
            resp = requests.post(f"{API_BASE_URL}{endpoint}", json=dades,
                                 headers=headers, timeout=5)
        else:
            return None
        # Token expirat → tornem al login
        if resp.status_code == 401:
            st.warning("Sessió expirada. Torna a fer login.")
            fer_logout()
            st.rerun()
        resp.raise_for_status()
        return resp.json()
    except requests.exceptions.ConnectionError:
        st.error("No s'ha pogut connectar amb l'API.")
        return None
    except requests.exceptions.Timeout:
        st.warning("L'API no respon. Torna-ho a intentar.")
        return None
    except requests.exceptions.HTTPError as e:
        st.error(f"Error API: {e.response.status_code}")
        return None

# === PÀGINA DE LOGIN (si no autenticat) ===
if st.session_state.token is None:
    st.title("EsportsPulse — Login")
    _, col_login, _ = st.columns([1, 2, 1])
    with col_login:
        with st.form("login_form"):
            username = st.text_input("Usuari", placeholder="admin")
            password = st.text_input("Contrasenya", type="password")
            if st.form_submit_button("Entrar", type="primary"):
                if username and password:
                    if fer_login(username, password):
                        st.rerun()  # Mostra el dashboard
                else:
                    st.warning("Omple tots els camps.")
    st.stop()  # IMPORTANT: no executa el dashboard si no hi ha token

# === DASHBOARD (autenticat) ===
col_titol, col_user = st.columns([3, 1])
with col_titol:
    st.title("EsportsPulse Dashboard")
with col_user:
    st.write(f"Usuari: **{st.session_state.username}**")
    if st.button("Tancar sessió"):
        fer_logout()
        st.rerun()

campions = crida_api("/api/champions")
if not campions:
    st.warning("No s'han pogut obtenir dades.")
    st.stop()

df = pd.DataFrame(campions)

# --- KPIs en 3 columnes ---
kpi1, kpi2, kpi3 = st.columns(3)
with kpi1:
    st.metric("Total Campions", len(df))
with kpi2:
    st.metric("WinRate Mitjà", f"{round(df['winRate'].mean(), 1)}%")
with kpi3:
    idx = df["gamesPlayed"].idxmax()
    st.metric("Més Popular", df.loc[idx, "name"], delta=f"{df.loc[idx, 'gamesPlayed']:,} partides")

st.divider()

# --- Gràfic Top 10 per partides jugades ---
st.subheader("Top 10 Campions per Partides Jugades")
df_top10 = df.nlargest(10, "gamesPlayed").set_index("name")[["gamesPlayed"]]
st.bar_chart(df_top10, use_container_width=True)

st.divider()

# --- Taula amb cerca ---
cerca = st.text_input("Cerca per nom", key="cerca_campions")
df_filtrat = df[df["name"].str.contains(cerca, case=False, na=False)] if cerca else df
st.dataframe(df_filtrat, use_container_width=True, hide_index=True, column_config={
    "name": st.column_config.TextColumn("Campió"),
    "winRate": st.column_config.NumberColumn("Win Rate (%)", format="%.1f%%"),
    "gamesPlayed": st.column_config.NumberColumn("Partides", format="%d"),
    "role": st.column_config.TextColumn("Rol"),
})

st.divider()

# --- Formulari per crear campió (POST /api/champions) ---
st.subheader("Crear Nou Campió")
with st.form("form_crear", clear_on_submit=True):
    c1, c2 = st.columns(2)
    with c1:
        nou_nom = st.text_input("Nom", placeholder="Ex: Morgana")
    with c2:
        nou_rol = st.selectbox("Rol", ["Top", "Jungle", "Mid", "ADC", "Support"])
    c3, c4 = st.columns(2)
    with c3:
        nou_wr = st.number_input("Win Rate (%)", 0.0, 100.0, 50.0, step=0.1)
    with c4:
        nou_games = st.number_input("Partides", min_value=0, value=1000, step=100)
    if st.form_submit_button("Crear Campió", type="primary"):
        if not nou_nom:
            st.error("El nom és obligatori.")
        else:
            res = crida_api("/api/champions", "POST",
                            {"name": nou_nom, "role": nou_rol, "winRate": nou_wr,
                             "gamesPlayed": nou_games})
            if res:
                st.success(f"Campió '{nou_nom}' creat!")
                st.rerun()
```

### Resum de Gestió d'Errors

```python
# Patrons d'error a Streamlit — cada un amb un color i propòsit diferent:
st.error("No s'ha pogut connectar.")    # Vermell — error crític
st.warning("Sessió expirada.")          # Groc — avís no bloquejant
st.info("Cap resultat per la cerca.")   # Blau — informatiu
st.success("Campió creat!")             # Verd — acció completada
```

---

## Checklist de Lliurament

- [ ] Login funciona: credencials a `POST /auth/login`, token JWT a `st.session_state`
- [ ] Pàgines protegides: dashboard només visible amb token vàlid
- [ ] Capçalera `Authorization: Bearer <token>` a totes les crides API
- [ ] KPIs: total campions, WinRate mitjà, campió més popular
- [ ] Gràfic de barres: top 10 campions per partides jugades
- [ ] Formulari crea campió via `POST /api/champions`
- [ ] Errors gestionats amb `st.error()` i `st.warning()`
- [ ] Botó "Tancar sessió" esborra token i torna al login
