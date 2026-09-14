**Setmana 10: Dashboard Streamlit, Spec-Driven UI i Testing End-to-End**

---

### **Dilluns: Introducció a Streamlit i Primer Dashboard**

* **Cursos i Material de Lectura:**
* **Documentació:** [*Streamlit Get Started*](https://docs.streamlit.io/get_started).
* **Article:** [*Build a Dashboard in 10 Minutes*](https://docs.streamlit.io/develop/tutorials).
* **Documentació:** [*Streamlit API Reference*](https://docs.streamlit.io/develop/api-reference) — `st.title`, `st.dataframe`, `st.text_input`, `st.button`, `st.columns`.


* **Activitat i Què s'espera programar:**
* **Instal·lació i primer contacte:**
  * `pip install streamlit` al venv del projecte.
  * Crear `dashboard/app.py` amb un "Hello EsportsPulse" i executar `streamlit run dashboard/app.py`.
  * Entendre el model de Streamlit: cada interacció de l'usuari re-executa l'script sencer. No hi ha "event handlers" com a React — el flux és top-to-bottom.
* **Dashboard mínim connectat a l'API REST:**
  * Importar `requests` per consumir l'API Java (S7).
  * Mostrar una llista de champions en una taula (`st.dataframe`) cridant `GET /champions`.
  * Afegir un camp de cerca (`st.text_input`) que filtra per nom (`GET /champions?name=X`).
  * Afegir un botó "Refresh" que torna a cridar l'API.
* **Prova manual:** Arrenca el backend Java (`mvn spring-boot:run`) i el dashboard (`streamlit run`) en dues terminals. Verifica que les dades es mostren.
* **Lliçó:** Streamlit és ràpid per prototipar — en 30 línies tens un dashboard funcional. Però no és React: no hi ha components reutilitzables, no hi ha routing, no escala per a producció. El seu valor és **velocitat de prototipatge i feedback visual per iterar specs**.


---

### **Dimarts: Spec del Dashboard — Wireframe ASCII + Agent**

* **Cursos i Material de Lectura:**
* **Article:** [*How to Write UI Specs*](https://www.nngroup.com/articles/wireframing/) — Nielsen Norman Group (concepte general).
* **Documentació:** Streamlit — [*Layouts and Containers*](https://docs.streamlit.io/develop/api-reference/layout).


* **Activitat i Què s'espera programar:**
* **Exercici d'Escriptura de Specs: Dashboard Spec en Markdown.**
  * Escriu un fitxer `dashboard-spec.md` que descrigui el dashboard complet:
    ```markdown
    ## EsportsPulse Dashboard — Spec

    ### Layout
    ┌──────────────────────────────────────────────────┐
    │  EsportsPulse Dashboard              [🔄 Refresh] │
    ├──────────────────┬───────────────────────────────┤
    │                  │                               │
    │  Filtres:        │  Taula de Champions:           │
    │  [Cerca nom____] │  | Champion | WinRate | GamesPlayed│
    │  [WinRate min: _] │  | Ahri     | 52.3%  | 150K   │
    │  [WinRate max: _] │  | Jinx     | 51.1%  | 120K   │
    │  [Només role: ☐] │  | Thresh   | 50.8%  | 200K   │
    │                  │                               │
    ├──────────────────┴───────────────────────────────┤
    │  Detall del champion seleccionat:                │
    │  Nom: Ahri                                       │
    │  Games played: 150.000                           │
    │  Gràfic: [barra de winRate vs mitjana del role]  │
    └──────────────────────────────────────────────────┘

    ### Components
    1. **Header:** Títol + botó Refresh que recarrega dades de l'API.
    2. **Sidebar (columna esquerra):** Filtres de cerca:
       - Text input per nom de champion (cerca parcial).
       - Sliders per rang de winRate (min/max).
       - Checkbox per filtrar per role.
    3. **Taula principal:** Llista de champions filtrats. Columnes: champion, winRate, gamesPlayed.
       - Clicable: seleccionar un champion mostra el detall a sota.
    4. **Detall:** Informació ampliada del champion seleccionat + gràfic de barres (opcional).

    ### Flux d'interacció
    1. L'usuari obre el dashboard → es carreguen tots els champions via GET /champions.
    2. L'usuari escriu "Ahri" al camp de cerca → la taula es filtra en temps real.
    3. L'usuari selecciona un champion → el panell de detall mostra info ampliada.
    4. L'usuari clica Refresh → es tornen a cridar les APIs.

    ### Restriccions
    - Streamlit, no React/HTML.
    - Consumeix l'API REST de Java (localhost:8080), mai accedeix a la BD directament.
    - Errors d'API (timeout, 500) es mostren com st.error(), no es silencien.
    ```
  * **Dona la spec a l'agent** (Cursor o Claude Code, conversa nova) amb el prompt: *"Implementa aquest dashboard Streamlit seguint exactament aquesta spec."*
  * **Compara el resultat** amb el wireframe ASCII. L'agent ha respectat el layout? Els filtres funcionen? La taula és clicable?
  * **Itera la spec:** Si el layout no és correcte, no arreglis el codi — arregla la spec i regenera. Afegeix detalls que faltaven (mida de columnes, ordre per defecte, format de preus).
  * **Anota:** Quantes iteracions han sigut necessàries? Què faltava a la spec original?
* **Lliçó:** El wireframe ASCII és una spec visual que l'agent interpreta. Com més precís sigui, menys iteracions necessites. Això és exactament el que faries a una empresa: escriure un ticket amb wireframe → algú (humà o IA) l'implementa → tu revises.


---

### **Dimecres: Visualitzacions i Components Avançats**

* **Cursos i Material de Lectura:**
* **Documentació:** Streamlit — [*Charts and Maps*](https://docs.streamlit.io/develop/api-reference/charts).
* **Documentació:** Streamlit — [*st.metric, st.columns*](https://docs.streamlit.io/develop/api-reference/data/st.metric).


* **Activitat i Què s'espera programar:**
* **KPIs amb `st.metric`:**
  * Afegir a la part superior del dashboard 3 mètriques:
    * Total de champions a la BD.
    * WinRate mitjà de tots els champions.
    * Champion amb més games played.
  * Usar `st.columns(3)` per mostrar-los en fila.
* **Gràfic de barres:**
  * Top 10 champions per games played, visualitzat amb `st.bar_chart`.
  * Alternativa: usar `plotly` per a gràfics interactius (hover amb detalls).
* **Formulari de registre de champion:**
  * `st.form` amb camps: nom, role, winRate, gamesPlayed.
  * En submit: `POST /champions` a l'API.
  * Mostrar `st.success("Champion registrat!")` o `st.error("Error: ...")`.
* **Connexió S9 (Error Handling):**
  * Si l'API Java no respon (timeout): mostrar `st.warning("Backend no disponible. Revisa que el servidor Java estigui actiu.")`.
  * Si una crida retorna 400/404: mostrar l'error de l'API de forma llegible, no el stacktrace.


---

### **Dijous: Testing i CI per Python**

* **Cursos i Material de Lectura:**
* **Documentació:** [*Testing Streamlit Apps*](https://docs.streamlit.io/develop/concepts/app-testing).
* **Article:** Baeldung equivalent — [*pytest-mock*](https://pytest-mock.readthedocs.io/).


* **Activitat i Què s'espera programar:**
* **Tests del dashboard (lògica, no UI):**
  * Extreure la lògica de negoci del dashboard a funcions pures testejables:
    * `filter_champions(champions: list, name: str, min_winrate: float, max_winrate: float) -> list`
    * `calculate_kpis(champions: list) -> dict`
    * `format_win_rate(win_rate: float) -> str`
  * Tests amb `pytest`:
    * `test_filter_by_name()`: filtra correctament per substring.
    * `test_filter_by_role()`: filtra champions per role (ADC, Support, etc.).
    * `test_kpis_empty_list()`: no falla amb llista buida.
    * `test_format_win_rate()`: 52.3 → "52,3%" (o el format escollit).
  * **Lliçó:** Streamlit és difícil de testejar com a UI. La solució és la mateixa que a Spring: separar lògica de presentació. Les funcions pures es testegen fàcilment; la capa de Streamlit és "només" la visualització.
* **Ampliar CI per a Python:**
  * Afegir al workflow `.github/workflows/ci.yml`:
    ```yaml
    python-tests:
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v4
        - uses: actions/setup-python@v5
          with:
            python-version: '3.12'
        - run: pip install -r requirements.txt
        - run: python -m pytest ai-python/tests/ -v
        - run: ruff check ai-python/
    ```
  * Ara el CI valida Java (mvn test + checkstyle) i Python (pytest + ruff) a cada push.
* **Connexió S6:** El patró és el mateix que a Java — separar lògica testejable de la capa de presentació. JUnit testeja el Service, no el Controller. pytest testeja les funcions, no Streamlit.


---

### **Divendres: Integració End-to-End, Demo i PR**

* **Cursos i Material de Lectura:**
* **Article:** [*End-to-End Testing Best Practices*](https://martinfowler.com/articles/practical-test-pyramid.html) — Martin Fowler.


* **Activitat i Què s'espera programar:**
* **Test d'integració end-to-end (manual):**
  1. Arrenca PostgreSQL/H2 + backend Java (`mvn spring-boot:run`).
  2. Arrenca el servei Python (FastAPI) si existeix (S8-S9).
  3. Arrenca el dashboard (`streamlit run dashboard/app.py`).
  4. Verifica el flux complet:
     * El dashboard mostra champions de la BD.
     * La cerca filtra correctament.
     * Registrar un champion via formulari → apareix a la taula.
     * KPIs es recalculen.
     * Errors d'API es gestionen (para el backend, refresca el dashboard → error visible).
* **Revisió de la spec:**
  * Obre `dashboard-spec.md` de dimarts. El dashboard final compleix la spec? Quines desviacions hi ha?
  * Si hi ha desviacions justificades (millores que han sorgit), actualitza la spec per reflectir l'estat real.
  * **Lliçó:** La spec és un document viu — evoluciona amb el producte. No és un contracte rígid; és un acord que es revisa.
* **Demo de 3 minuts:**
  * Practica explicar el dashboard a algú que no l'ha vist: "Això és EsportsPulse, consumeix una API REST que..."
  * **Connexió S24:** Aquesta és la primera demo. A S24 serà la demo final del portfolio.

* **Finalització del cicle Git:**
* Commit: `feat(python): Streamlit dashboard with API integration, specs, and E2E testing`
* Puja branca `feature/week10-dashboard` a GitHub.
* Verifica que el CI passa (Java + Python).
* Merge a `main`.

---

## Vídeos Recomanats

- **Streamlit:** Cerca "Streamlit tutorial español" o "Data Professor Streamlit" (anglès, canal de referència per Streamlit amb molts exemples pràctics).
- **Dashboards amb Streamlit:** Cerca "Coding Is Fun Streamlit dashboard" (anglès, projectes complets pas a pas amb gràfics i KPIs).
- **Plotly:** Cerca "Plotly Python tutorial" o "Charming Data Plotly Dash" (anglès, molt visual).
- **Spec-driven development:** Cerca "spec driven development AI" — tema emergent, pocs vídeos dedicats. La pràctica de dimarts (wireframe ASCII → agent → codi) és el millor recurs.

---

## Nota sobre Spec-Driven UI

Setmana 10 és la primera vegada que l'estudiant escriu una **spec visual** (wireframe ASCII) i la dona a un agent per generar UI. La progressió de specs fins ara:

| Setmana | Tipus de Spec | Què genera l'agent |
|---------|--------------|-------------------|
| S2 | `.cursorrules` (convencions de codi) | Codi que segueix les regles |
| S4 | Spec de refactorització (regles + tests) | Codi refactoritzat |
| S7 | API spec (endpoints + JSON exemples) | Controllers REST |
| **S10** | **Dashboard spec (wireframe + flux + restriccions)** | **Interfície Streamlit** |
| S15 | Spec d'agent (role + tools + guardrails) | Agent funcional |
| S18 | Spec completa (feature de negoci → tests) | Feature end-to-end |

A S10 l'estudiant aprèn que les specs no són només per codi backend — serveixen per a qualsevol artefacte que un agent pugui generar. El wireframe ASCII és un format sorprenentment efectiu perquè és precís (posicions, mides relatives) sense requerir eines gràfiques.

## Nota sobre Frontend

Streamlit **no és frontend web**. És una eina de prototipatge ràpid per a developers Python. Al curs s'utilitza per:
1. **Validar visualmente** que l'API funciona (complementa tests automatitzats).
2. **Practicar spec-driven development** amb feedback visual immediat.
3. **Preparar demos** del sistema complet (S24).

Si l'estudiant vol afegir frontend web (React/Vue), l'API REST de S7 amb Swagger és el punt de partida — però això és una formació separada.
