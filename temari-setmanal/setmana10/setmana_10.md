**Setmana 8: Python com a Co-Protagonista, Pydantic v2, LLMs i MCP Basics**

---

### **Dilluns: Pydantic v2 — Models amb Superpoderes**

* **Cursos i Material de Lectura:**
* **Documentació:** Pydantic — [*Models*](https://docs.pydantic.dev/latest/concepts/models/).
* **Article:** Samuel Colvin — [*Why Pydantic v2?*](https://docs.pydantic.dev/latest/why/).
* **Documentació:** Pydantic — [*Field Types & Validators*](https://docs.pydantic.dev/latest/concepts/validators/).


* **Activitat i Què s'espera programar:**
* **Crear els models Pydantic per EsportsPulse:**
  * `ChampionAnalysis(champion_id, name, role, strengths, counters, patch_tier, champion_trend, summary)` — el resultat d'una anàlisi de champion.
  * `ChampionTrend(current_win_rate, previous_win_rate, trend_direction, games_analyzed)` — tendència del champion (up/down/stable).
  * `PatchSummary(version, date, champion_changes, meta_impact)` — resum d'un patch de LoL.
* **Validadors:**
  * `champion_id` ha de ser un nom de champion vàlid amb almenys 2 caràcters (usa `@field_validator`).
  * `patch_tier` ha de ser S/A/B/C/D (usa `Literal`).
  * `current_win_rate` / `previous_win_rate` entre 0 i 100 (usa `Field(ge=0, le=100)`).
  * `trend_direction` limitat a `"up"`, `"down"`, `"stable"` (usa `Literal`).
* **Field() amb metadades:**
  * Afegeix `description` i `examples` a cada camp.
  * Genera el JSON schema amb `model_json_schema()` i observa el resultat.
* **Comparativa amb Java:**
  * Compara amb els DTOs de S7 (`ChampionDTO`, `CreateChampionRequest`) i els records de S2.
  * Java record amb compact constructor validation vs Pydantic `@field_validator`.
  * **Clau:** "Pydantic fa en Python el que DTOs + Bean Validation fan en Java, però en 3 línies."

---

### **Dimarts: LLM Output Estructurat — La IA com a Funció**

* **Cursos i Material de Lectura:**
* **Documentació:** Anthropic — [*Claude Messages API*](https://docs.anthropic.com/en/api/messages).
* **Documentació:** OpenAI — [*Function Calling*](https://platform.openai.com/docs/guides/function-calling).
* **Article:** Anthropic — [*Tool Use (Function Calling)*](https://docs.anthropic.com/en/docs/build-with-claude/tool-use).


* **Activitat i Què s'espera programar:**
* **Primera crida: text lliure (sense estructura).**
  * Instal·la el SDK: `pip install anthropic` (o `openai`).
  * Fes una crida simple: "Analitza el champion Jinx amb 52.3% win rate i 150K partides."
  * Observa: la resposta és text natural, impossible de parsejar automàticament.
* **Segona crida: output estructurat amb tool_use.**
  * Defineix un tool amb l'schema generat per `ChampionAnalysis.model_json_schema()`.
  * Crida l'API amb `tools=[...]` perquè el model retorni JSON estructurat.
  * Parseja la resposta amb `ChampionAnalysis.model_validate(tool_input)`.
* **Few-shot prompting:**
  * Inclou 2 exemples al prompt per millorar la qualitat:
    ```python
    examples = """
    Exemple 1: Jinx, 52.3% win rate, 150K partides → patch_tier: S, trend: up
    Exemple 2: Ryze, 44.8% win rate, 80K partides → patch_tier: D, trend: down
    """
    ```
  * Compara la qualitat amb i sense exemples.
* **Exercici concret:**
  * Script que rep `champion_id`, `win_rate`, `games_played` → crida LLM → retorna `ChampionAnalysis` validat.
  * "Analitza el champion {name} amb {win_rate}% win rate i {games_played} partides. Retorna un ChampionAnalysis."
* **Clau:** "Un LLM sense schema és un generador de text. Amb schema, és una funció amb signatura."

---

### **Dimecres: FastAPI Exprés — El Primer Endpoint Python**

* **Cursos i Material de Lectura:**
* **Documentació:** FastAPI — [*First Steps*](https://fastapi.tiangolo.com/tutorial/first-steps/).
* **Documentació:** FastAPI — [*Request Body (Pydantic)*](https://fastapi.tiangolo.com/tutorial/body/).
* **Article:** FastAPI — [*Path Parameters*](https://fastapi.tiangolo.com/tutorial/path-params/).


* **Activitat i Què s'espera programar:**
* **Crear l'app FastAPI mínima:**
  ```python
  from fastapi import FastAPI
  app = FastAPI(title="EsportsPulse Analysis Service")

  @app.get("/health")
  def health():
      return {"status": "ok", "service": "esportspulse-analysis"}
  ```
* **Endpoint principal: `GET /analyze/{champion_id}`**
  * Rep un `champion_id` (path parameter).
  * Consulta l'API Java (S7) amb `httpx` per obtenir dades del champion.
  * Crida el LLM amb tool_use (dimarts).
  * Retorna `ChampionAnalysis` (Pydantic model com a response).
* **Endpoint batch: `POST /batch-analyze`**
  * Rep una llista de `champion_ids` al body.
  * Retorna `list[ChampionAnalysis]`.
* **Comparativa amb Spring Boot (S7):**

  | Concepte         | Spring Boot              | FastAPI                |
  |:-----------------|:-------------------------|:-----------------------|
  | Routing          | `@GetMapping("/path")`   | `@app.get("/path")`   |
  | Validació        | `@Valid` + Bean Val.     | Pydantic built-in     |
  | Docs auto        | SpringDoc/Swagger        | `/docs` automàtic     |
  | Server           | Tomcat embedded          | Uvicorn (ASGI)        |

* **Executar i provar:**
  * `uvicorn main:app --reload`
  * Obre `http://localhost:8000/docs` — documentació auto-generada.
  * Prova amb curl: `curl http://localhost:8000/analyze/Jinx`.

---

### **Dijous: MCP Basics — Connectar la IA al Teu Codi**

* **Cursos i Material de Lectura:**
* **Documentació:** [*Model Context Protocol Specification*](https://modelcontextprotocol.io/).
* **Article:** Anthropic — [*What is MCP?*](https://www.anthropic.com/news/model-context-protocol).
* **Documentació:** [*Cursor MCP Configuration*](https://docs.cursor.com/context/model-context-protocol).


* **Activitat i Què s'espera programar:**
* **Entendre l'arquitectura MCP:**
  ```
  ┌────────────────────┐         stdio/SSE         ┌───────────────────┐
  │   MCP Client       │◄──────────────────────────►│   MCP Server      │
  │  (Cursor / Claude) │      JSON-RPC 2.0          │  (el teu codi)    │
  └────────────────────┘                             └───────────────────┘
        Pregunta:                                     Exposa:
        "Quants champions                             - tools (accions)
         tenen winrate                                - resources (dades)
         > 52%?"                                      - prompts (plantilles)
  ```
* **Els tres primitius MCP:**
  * **Tools:** funcions que la IA pot cridar (ex: `get_champion(champion_id)`).
  * **Resources:** dades que la IA pot llegir (ex: `esportspulse://champions/top10`).
  * **Prompts:** plantilles reutilitzables (ex: `analyze_champion` amb paràmetres).
* **Exercici 1: Connectar un MCP server existent.**
  * Instal·la el MCP server de filesystem o SQLite.
  * Configura a `.cursor/mcp.json`:
    ```json
    {
      "mcpServers": {
        "filesystem": {
          "command": "npx",
          "args": ["-y", "@modelcontextprotocol/server-filesystem", "./data"]
        }
      }
    }
    ```
  * Verifica que Cursor detecta el server i pot fer queries.
* **Exercici 2: Usa l'MCP server des del chat de Cursor.**
  * Demana a Cursor: "Llista els fitxers del directori data."
  * Observa com Cursor crida el tool automàticament.
* **Clau:** "MCP és l'USB de la IA — un protocol estàndard perquè qualsevol eina es connecti a qualsevol model."

---

### **Divendres: Integració — Agent Analista de Champions**

* **Cursos i Material de Lectura:**
* Repassa tot el material de la setmana.
* **Article:** Martin Fowler — [*Microservices*](https://martinfowler.com/articles/microservices.html) (només la secció introductòria).


* **Activitat i Què s'espera programar:**
* **Construir el primer "agent" (senzill però complet):**
  1. Script Python que rep una pregunta sobre un champion.
  2. Fa fetch de dades de l'API Java REST (S7) amb `httpx`.
  3. Crida l'LLM amb tool_use i output estructurat (dimarts).
  4. Valida la resposta amb Pydantic (dilluns).
  5. Retorna `ChampionAnalysis` formatat.
* **Flux complet:**
  ```
  Pregunta         API Java (S7)         LLM (Claude/OpenAI)      Pydantic
  ────────► fetch ────────────► prompt ──────────────────► parse ──────────► ChampionAnalysis
            httpx   GET /champions/   tool_use + schema    model_validate()
  ```
* **Test end-to-end:**
  * Java backend running (`mvn spring-boot:run`).
  * Python service running (`uvicorn main:app`).
  * `curl http://localhost:8000/analyze/Jinx` retorna un `ChampionAnalysis` vàlid.
* **Tests amb pytest (3 mínim):**
  * Mock del LLM amb `monkeypatch` per no gastar tokens als tests.
  * Test 1: resposta LLM vàlida → `ChampionAnalysis` correcte.
  * Test 2: resposta LLM invàlida → error de validació Pydantic.
  * Test 3: champion no trobat a l'API Java → error 404 propagat.
* **Commit i PR:**
  * Commit message: `feat(python): analysis service with Pydantic, LLM structured output, FastAPI endpoint`
  * Branca: `feature/week8-python-analysis`
  * Verifica que CI passa (tests Python + tests Java).

---

## Vídeos Recomanats

- **Pydantic v2:** Cerca "ArjanCodes Pydantic v2" (anglès, el millor tutorial de Pydantic a YouTube). En castellà: "Pydantic tutorial español FastAPI".
- **FastAPI:** Cerca "MoureDev FastAPI" (castellà) o "TechWithTim FastAPI tutorial" (anglès). FastAPI + Pydantic van de la mà.
- **LLMs i structured output:** Cerca "Sam Witteveen structured output LLM" o "AI Jason function calling" (anglès). Temes nous — la majoria de contingut bo és en anglès.
- **MCP (Model Context Protocol):** Cerca "Anthropic MCP tutorial" o "MCP server tutorial Claude" — contingut recent, prioritza vídeos de 2024-2025.

---

## Nota sobre Python i Empliabilitat

- **S1-S6:** Python com a mirall — exercicis paral·lels per aprendre la sintaxi.
- **S7:** Python com a consumidor — CLI que crida l'API Java.
- **S8:** Python com a co-protagonista — servei real amb Pydantic, FastAPI, LLMs.

Ara tens dos llenguatges de producció: Java per al backend core i Python per a IA/anàlisi. Aquesta combinació és exactament el que demanen els equips que treballen amb LLMs: el backend no desapareix, però necessites Python per connectar-hi models. A partir d'aquí, els dos llenguatges creixen en paral·lel.
