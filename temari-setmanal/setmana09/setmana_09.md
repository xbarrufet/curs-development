**Setmana 9: Error Handling Inter-Serveis, Logging Estructurat i MCP Server Personalitzat**

---

### **Dilluns: Error Handling Inter-Serveis**

* **Cursos i Material de Lectura:**
* **Article:** Microsoft Azure Architecture — [*Retry Pattern*](https://learn.microsoft.com/en-us/azure/architecture/patterns/retry).
* **Especificació:** IETF — [*RFC 7807: Problem Details for HTTP APIs*](https://www.rfc-editor.org/rfc/rfc7807).
* **Article:** Baeldung — [*Exception Handling in Spring Boot*](https://www.baeldung.com/exception-handling-for-rest-with-spring).
* **Article:** Baeldung — [*Spring RestTemplate Error Handling*](https://www.baeldung.com/spring-rest-template-error-handling).


* **Activitat i Què s'espera programar:**
* **Experiment: Què passa quan Python crida Java i falla?**
  * Atura el servei Java (`Ctrl+C` al Spring Boot).
  * Des de Python, crida `requests.get("http://localhost:8080/games/APP-1")` — obtens `ConnectionError`.
  * Ara arrenca Java però afegeix un `Thread.sleep(30000)` al controller — obtens `ReadTimeout` (si tens timeout) o un thread bloquejat per sempre (si no en tens).
  * Lliço: sense timeout, un servei lent pot matar el teu servei.
* **Afegir timeout a totes les crides HTTP:**
  ```python
  # Python — SEMPRE timeout
  response = requests.get(
      f"{JAVA_API_URL}/games/{game_id}",
      timeout=5  # 5 segons màxim
  )
  ```
  ```java
  // Java — RestTemplate amb timeout
  @Bean
  public RestTemplate restTemplate() {
      var factory = new SimpleClientHttpRequestFactory();
      factory.setConnectTimeout(Duration.ofSeconds(3));
      factory.setReadTimeout(Duration.ofSeconds(5));
      return new RestTemplate(factory);
  }
  ```
* **Implementar retry amb backoff exponencial:**
  * Primer a mà (loop amb `time.sleep`), per entendre el patró.
  * Després amb la llibreria `tenacity`:
  ```python
  from tenacity import retry, stop_after_attempt, wait_exponential

  @retry(
      stop=stop_after_attempt(3),
      wait=wait_exponential(multiplier=1, min=1, max=8)
  )
  def call_java_api(game_id: str) -> dict:
      response = requests.get(
          f"{JAVA_API_URL}/games/{game_id}",
          timeout=5
      )
      response.raise_for_status()
      return response.json()
  ```
* **Definir format d'error estàndard (RFC 7807 Problem Details):**
  * Java: ampliar el `@ControllerAdvice` de S7 per retornar Problem Details.
  * Python: crear exception handlers a FastAPI.
  * Custom exceptions: `GameNotFoundException`, `ExternalServiceUnavailableException`.
* **HTTP status codes — quan usar cada un:**
  * 400: request malformada (JSON invàlid, camp obligatori absent).
  * 404: recurs no trobat (`/games/APP-999`).
  * 422: validació de negoci fallida (preu negatiu, nom duplicat).
  * 500: bug intern del teu servei.
  * 502: l'upstream (Java) ha retornat un error.
  * 503: el teu servei està sobrecarregat o en manteniment.
  * 504: l'upstream (Java) no ha respost a temps.
* **Clau del dia:** "El teu servei no pot crashar perque un altre servei ha caigut."


---

### **Dimarts: Logging Estructurat — JSON, No Text**

* **Cursos i Material de Lectura:**
* **Documentació:** structlog — [*Getting Started*](https://www.structlog.org/en/stable/getting-started.html).
* **Article:** Baeldung — [*SLF4J Best Practices*](https://www.baeldung.com/slf4j-with-log4j2-logback).
* **Article:** [*Structured Logging: Why and How*](https://www.honeycomb.io/blog/structured-logging-and-your-team).


* **Activitat i Què s'espera programar:**
* **Substituir tots els `print()` per structlog (Python):**
  ```python
  import structlog

  logger = structlog.get_logger()

  # Abans (S7-S8):
  print(f"Error fetching game {game_id}: {e}")

  # Ara:
  logger.error("game_fetch_failed", game_id=game_id, error=str(e))
  ```
* **Configurar SLF4J/Logback amb JSON output (Java):**
  * Afegir dependència `logstash-logback-encoder` al `pom.xml`.
  * Configurar `logback-spring.xml` per emetre JSON en lloc de text pla.
* **Camps obligatoris a cada log entry:**
  * `timestamp`, `level`, `service_name`, `correlation_id`, `message` + camps de context.
* **Implementar correlation_id:**
  * Generar UUID a l'entry point (middleware FastAPI / filter Spring).
  * Propagar via header `X-Correlation-ID` entre serveis.
  * Python: FastAPI middleware que genera o llegeix el header.
  * Java: `OncePerRequestFilter` que fa el mateix.
  ```python
  # FastAPI middleware
  @app.middleware("http")
  async def correlation_id_middleware(request: Request, call_next):
      correlation_id = request.headers.get(
          "X-Correlation-ID", str(uuid.uuid4())
      )
      structlog.contextvars.bind_contextvars(correlation_id=correlation_id)
      response = await call_next(request)
      response.headers["X-Correlation-ID"] = correlation_id
      return response
  ```
* **Comparació: text vs JSON logs:**

  | Aspecte            | Text pla                        | JSON estructurat                  |
  |--------------------|----------------------------------|-----------------------------------|
  | Llegibilitat humana | Fàcil per un log                | Menys intuïtiu                    |
  | Cerca automatitzada | grep amb regex fràgils          | `jq '.correlation_id == "abc"'`   |
  | Agregació          | Molt difícil                     | Trivial (ELK, Datadog, etc.)     |
  | Context addicional | Calen convencions manuals        | Camps tipats i estructurats       |

* **Exercici:** fer una request completa (curl -> Python -> Java -> H2) i buscar el correlation_id als logs dels dos serveis.
* **Clau del dia:** "print() és per debugging local. Logging estructurat és per diagnosticar bugs en producció a les 3 de la matinada."


---

### **Dimecres: MCP Server Personalitzat — GamePulse com a Tool**

* **Cursos i Material de Lectura:**
* **Documentació:** MCP Python SDK — [*GitHub: modelcontextprotocol/python-sdk*](https://github.com/modelcontextprotocol/python-sdk).
* **Especificació:** MCP — [*Tools*](https://spec.modelcontextprotocol.io/specification/server/tools/).
* **Tutorial:** Anthropic — [*Build MCP Servers*](https://docs.anthropic.com/en/docs/agents-and-tools/mcp).


* **Activitat i Què s'espera programar:**
* **Crear un MCP server que exposa GamePulse com a tools.**
  * A S8 vas connectar MCP servers existents. Ara en crees un de propi.
  ```python
  from mcp.server.fastmcp import FastMCP

  mcp = FastMCP("gamepulse")

  @mcp.tool()
  async def get_game(game_id: str) -> dict:
      """Obté les dades d'un joc per ID (appId).
      Retorna títol, preu i jugadors actius."""
      response = requests.get(
          f"{JAVA_API_URL}/games/{game_id}", timeout=5
      )
      response.raise_for_status()
      return response.json()

  @mcp.tool()
  async def search_games(query: str, limit: int = 10) -> list[dict]:
      """Cerca jocs per títol. Retorna fins a `limit` resultats."""
      response = requests.get(
          f"{JAVA_API_URL}/games",
          params={"title": query, "size": limit},
          timeout=5
      )
      response.raise_for_status()
      return response.json()

  @mcp.tool()
  async def get_game_analysis(game_id: str) -> dict:
      """Analitza el balanç d'un joc usant un LLM (crida el servei Python de S8)."""
      # Crida al endpoint d'anàlisi de FastAPI
      response = requests.post(
          f"{PYTHON_API_URL}/analyze/{game_id}", timeout=30
      )
      response.raise_for_status()
      return response.json()

  @mcp.tool()
  async def get_player_stats() -> dict:
      """Retorna estadístiques agregades de tots els jocs:
      total jocs, mitjana jugadors, joc més popular."""
      response = requests.get(
          f"{JAVA_API_URL}/games", timeout=5
      )
      games = response.json()
      total = len(games)
      avg_players = sum(g["activePlayerCount"] for g in games) / total if total else 0
      top_game = max(games, key=lambda g: g["activePlayerCount"], default=None)
      return {
          "totalGames": total,
          "averagePlayers": round(avg_players),
          "mostPopular": top_game["title"] if top_game else None,
      }
  ```
* **Testejar localment:**
  * Arrencar: `python gamepulse_mcp_server.py` (mode stdio).
  * Connectar a Cursor: afegir el server a `.cursor/mcp.json`.
  * Verificar que les 4 tools apareixen al panell MCP.
  * Fer una pregunta: "Quin preu té el joc APP-1?" — Cursor ha de cridar `get_game`.
* **Clau del dia:** "Ara Cursor pot parlar directament amb GamePulse sense que tu facis de pont."


---

### **Dijous: MCP al Workflow de Desenvolupament**

* **Cursos i Material de Lectura:**
* **Documentació:** Cursor — [*MCP Configuration*](https://docs.cursor.com/context/model-context-protocol).
* **Article:** Anthropic — [*Model Context Protocol*](https://www.anthropic.com/news/model-context-protocol).


* **Activitat i Què s'espera programar:**
* **Usar el MCP server creat ahir en un workflow real:**
  * Pregunta a Cursor (amb MCP connectat):
    * "Quin és el joc més popular?" — espera que cridi `get_player_stats`.
    * "Analitza el balanç de League of Legends" — espera que cridi `get_game_analysis`.
    * "Mostra'm tots els jocs de menys de 20 euros" — espera que cridi `search_games`.
  * Les respostes han de venir del MCP server (dades reals de la BD), no de les dades d'entrenament de Cursor.
* **Iterar l'spec del MCP server:**
  * Afegir un **resource** (dades que el model pot llegir sense cridar un tool):
  ```python
  @mcp.resource("gamepulse://games/top10")
  async def top10_games() -> str:
      """Retorna els 10 jocs més populars en format text."""
      response = requests.get(
          f"{JAVA_API_URL}/games?sort=activePlayerCount,desc&size=10",
          timeout=5
      )
      games = response.json()
      lines = [f"- {g['title']}: {g['activePlayerCount']} jugadors" for g in games]
      return "\n".join(lines)
  ```
  * Afegir un **prompt template** (prompt pre-definit que el model pot usar):
  ```python
  @mcp.prompt()
  async def analyze_game(game_id: str) -> str:
      """Prompt per analitzar el balanç d'un joc."""
      return f"""Analitza el balanç del joc amb ID {game_id}.
  Usa el tool get_game per obtenir les dades del joc.
  Després usa get_game_analysis per obtenir l'anàlisi del LLM.
  Presenta els resultats de forma clara amb punts forts i febles."""
  ```
* **Reflexió:** Què ha funcionat? Què no? Quan el model no crida el tool correcte, és culpa del prompt o de la descripció del tool?
* **Clau del dia:** "El MCP server és una spec executable — si les respostes no són bones, millora el server, no el prompt."


---

### **Divendres: Integració i Testing**

* **Cursos i Material de Lectura:**
* Repàs del material de la setmana.
* **Article:** Baeldung — [*Spring Boot Integration Testing*](https://www.baeldung.com/spring-boot-testing).


* **Activitat i Què s'espera programar:**
* **Tests per escenaris d'error:**
  * Python amb Java caigut: mock HTTP errors amb `responses` o `httpx_mock`.
  ```python
  @responses.activate
  def test_java_unavailable_returns_502():
      responses.add(
          responses.GET,
          f"{JAVA_API_URL}/games/APP-1",
          body=ConnectionError("Connection refused")
      )
      response = client.get("/games/APP-1")
      assert response.status_code == 502
      body = response.json()
      assert body["type"] == "about:blank"
      assert body["title"] == "External Service Unavailable"
  ```
  * Java amb requests malformades: verificar que retorna 400 amb Problem Details.
  * Timeout scenarios: mock una resposta lenta, verificar que el timeout actua.
* **Verificar correlation_ids als logs:**
  * Fer una request end-to-end.
  * Comprovar que el mateix `correlation_id` apareix als logs de Python i Java.
* **Demo completa:**
  ```
  curl → Python FastAPI → Java Spring Boot → H2 DB → LLM (S8) → Response
       ← correlation_id: "abc-123" apareix a TOTS els logs ←
  ```
* **CI update:**
  * Afegir tests Python a GitHub Actions (S4).
  * Verificar que `pytest` i `mvn test` passen al pipeline.
* **Merge i reflexió:**
  * Branca: `feature/week9-error-handling-mcp`.
  * Commit: `feat: error handling, structured logging, custom MCP server`
  * Reflexió sobre el progrés del Bloc 2:
    * S7: Java REST API + Python CLI.
    * S8: LLM integration, Pydantic, MCP basics.
    * S9: Comunicació robusta + MCP server propi.

---

## Nota sobre Robustesa i MCP

| Concepte               | Setmana | Estat                                     |
|------------------------|---------|-------------------------------------------|
| REST API               | S7      | Endpoints funcionals, @ControllerAdvice   |
| Bridge Java-Python     | S8      | Comunicació bàsica, sense error handling  |
| Error handling          | S9      | Timeout, retry, Problem Details (RFC 7807)|
| Logging                | S9      | Estructurat, JSON, correlation_id         |
| MCP (consumidor)       | S8      | Connectar servers existents               |
| MCP (creador)          | S9      | Server propi amb tools, resources, prompts|

La comunicació entre serveis ara és robusta: si Java cau, Python no crasha — retorna un error net, loggeja el problema amb context suficient, i el correlation_id et permet trobar exactament què ha passat.
