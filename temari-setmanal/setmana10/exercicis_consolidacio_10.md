# Setmana 10 — Exercicis de Consolidació

## Bàsics (has de saber fer-ho)

### 1. Dashboard minimal amb Streamlit
Crea un dashboard que:
- llegeixi jocs via `GET /games`,
- mostri una taula,
- permeti cercar per títol,
- i permeti refrescar dades.

**Fet quan:** l’usuari pot veure jocs i filtrar-los sense tocar la base de dades directament.

### 2. Spec visual del dashboard
Escriu un `dashboard-spec.md` amb:
- wireframe ASCII,
- components principals,
- flux d’interacció,
- i restriccions tècniques.

Després genera el dashboard a partir d’aquesta spec i compara amb el wireframe original.

**Fet quan:** el dashboard implementat segueix la spec sense ambigüitats importants.

### 3. Tests de lògica del dashboard
Crea funcions pures com:
- `filter_games(...)`
- `calculate_kpis(...)`
- `format_price(...)`

Escriu tests amb `pytest` que validin:
- filtrat per text,
- filtrat de gratuïts,
- Kpis amb llista buida,
- format de preus.

**Fet quan:** la lògica es validada sense necessitat de mockejar la UI.

---

## Avançats (si vas sobrat)

### 4. Formulari de creació de joc
Afegeix un `st.form` amb camps de títol i preu i connecta-ho amb `POST /games`.

**Fet quan:** el dashboard permet crear un joc des de la UI i el nou registre apareix a la taula.

### 5. Flux E2E manual del sistema complets
Arrenca:
- backend Java,
- dashboard Streamlit,
- i dades reals o de prova.

Valida el flux:
1. el dashboard mostra els jocs,
2. filtra per cerca,
3. crea un joc,
4. el nou joc apareix,
5. l’error del backend es mostra de forma clara.

**Fet quan:** el sistema es pot explotar visualment i no només provar unitats aïllades.

---

## Connexió amb les setmanes anteriors

- **S7**: API REST i DTOs
- **S6**: tests i CI
- **S8/S9**: Python, validació, errors i integració

La setmana 10 converteix el codi de backend en una eina visible i operativa, i prepara la transició cap a infraestructura, IA i producció.
