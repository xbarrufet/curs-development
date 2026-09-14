**Setmana 8: Python com a Co-Protagonista, Pydantic v2, LLMs i MCP Basics**

---

### **Dilluns: Pydantic v2 — Models amb Superpoderes**

* **Cursos i Material de Lectura:**
* **Documentació:** Pydantic — [*Models*](https://docs.pydantic.dev/latest/concepts/models/).
* **Article:** Samuel Colvin — [*Why Pydantic v2?*](https://docs.pydantic.dev/latest/why/).
* **Documentació:** Pydantic — [*Field Types & Validators*](https://docs.pydantic.dev/latest/concepts/validators/).


* **Activitat i Què s'espera programar:**
* **Crear els models Pydantic per GamePulse:**
  * `GameAnalysis(game_id, title, sentiment, player_trend, summary)` — el resultat d'una anàlisi de joc.
  * `PlayerTrend(current_players, peak_players, trend_direction)` — tendència de jugadors (up/down/stable).
  * `PatchSummary(version, date, changes, impact_score)` — resum d'un patch de joc.
* **Validadors:**
  * `game_id` ha de començar per `"APP-"` (usa `@field_validator`).
  * `impact_score` ha d'estar entre 0 i 10 (usa `Field(ge=0, le=10)`).
  * `trend_direction` limitat a `"up"`, `"down"`, `"stable"` (usa `Literal`).
* **Field() amb metadades:**
  * Afegeix `description` i `examples` a cada camp.
  * Genera el JSON schema amb `model_json_schema()` i observa el resultat.
* **Comparativa amb Java:**
  * Compara amb els DTOs de S7 (`GameDTO`, `CreateGameRequest`) i els records de S2.
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
  * Fes una crida simple: "Analitza el joc League of Legends amb 5M jugadors i preu 0€."
  * Observa: la resposta és text natural, impossible de parsejar automàticament.
* **Segona crida: output estructurat amb tool_use.**
  * Defineix un tool amb l'schema generat per `GameAnalysis.model_json_schema()`.
  * Crida l'API amb `tools=[...]` perquè el model retorni JSON estructurat.
  * Parseja la resposta amb `GameAnalysis.model_validate(tool_input)`.
* **Few-shot prompting:**
  * Inclou 2 exemples al prompt per millorar la qualitat:
    ```python
    examples = """
    Exemple 1: League of Legends, 5M jugadors, free → sentiment: positive, trend: up
    Exemple 2: Cyberpunk 2077, 200K jugadors, 59.99€ → sentiment: mixed, trend: stable
    """
    ```
  * Compara la qualitat amb i sense exemples.
* **Exercici concret:**
  * Script que rep `title`, `player_count`, `price` → crida LLM → retorna `GameAnalysis` validat.
  * "Analitza el joc {title} amb {player_count} jugadors i preu {price}. Retorna un GameAnalysis."
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
  app = FastAPI(title="GamePulse Analysis Service")

  @app.get("/health")
  def health():
      return {"status": "ok", "service": "gamepulse-analysis"}
  ```
* **Endpoint principal: `GET /analyze/{game_id}`**
  * Rep un `game_id` (path parameter).
  * Consulta l'API Java (S7) amb `httpx` per obtenir dades del joc.
  * Crida el LLM amb tool_use (dimarts).
  * Retorna `GameAnalysis` (Pydantic model com a response).
* **Endpoint batch: `POST /batch-analyze`**
  * Rep una llista de `game_ids` al body.
  * Retorna `list[GameAnalysis]`.
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
  * Prova amb curl: `curl http://localhost:8000/analyze/APP-1`.

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
        "Quants jugadors                              - tools (accions)
         té el LoL?"                                  - resources (dades)
                                                      - prompts (plantilles)
  ```
* **Els tres primitius MCP:**
  * **Tools:** funcions que la IA pot cridar (ex: `get_game(game_id)`).
  * **Resources:** dades que la IA pot llegir (ex: `gamepulse://games/top10`).
  * **Prompts:** plantilles reutilitzables (ex: `analyze_game` amb paràmetres).
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

### **Divendres: Integració — Agent Analista de Jocs**

* **Cursos i Material de Lectura:**
* Repassa tot el material de la setmana.
* **Article:** Martin Fowler — [*Microservices*](https://martinfowler.com/articles/microservices.html) (només la secció introductòria).


* **Activitat i Què s'espera programar:**
* **Construir el primer "agent" (senzill però complet):**
  1. Script Python que rep una pregunta sobre un joc.
  2. Fa fetch de dades de l'API Java REST (S7) amb `httpx`.
  3. Crida l'LLM amb tool_use i output estructurat (dimarts).
  4. Valida la resposta amb Pydantic (dilluns).
  5. Retorna `GameAnalysis` formatat.
* **Flux complet:**
  ```
  Pregunta         API Java (S7)         LLM (Claude/OpenAI)      Pydantic
  ────────► fetch ────────────► prompt ──────────────────► parse ──────────► GameAnalysis
            httpx   GET /games/   tool_use + schema        model_validate()
  ```
* **Test end-to-end:**
  * Java backend running (`mvn spring-boot:run`).
  * Python service running (`uvicorn main:app`).
  * `curl http://localhost:8000/analyze/APP-1` retorna un `GameAnalysis` vàlid.
* **Tests amb pytest (3 mínim):**
  * Mock del LLM amb `monkeypatch` per no gastar tokens als tests.
  * Test 1: resposta LLM vàlida → `GameAnalysis` correcte.
  * Test 2: resposta LLM invàlida → error de validació Pydantic.
  * Test 3: joc no trobat a l'API Java → error 404 propagat.
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
