# Setmana 9 — Exercicis de Consolidació

---

## Bàsics (has de saber fer-ho)

### 1. Error handling al bridge Java-Python
Implementar timeout (5s) + retry (3 intents, backoff exponencial) al servei Python quan crida l'API Java. Usar la llibreria `tenacity` per al retry. Quan tots els intents fallen, retornar un error HTTP 502 amb format RFC 7807 Problem Details (`type`, `title`, `status`, `detail`, `instance`). Testejar: atura el servei Java, crida Python, verifica que el retry funciona (veuràs 3 intents als logs) i finalment retorna l'error 502 correctament formatat.

**Connexió S7, S8:** Reutilitzes el bridge HTTP de S8 i el `@ControllerAdvice` de S7, ara amb resiliència.

**Fet quan:** Retry funciona amb 3 intents i backoff visible als logs. Error response en format Problem Details (Content-Type: `application/problem+json`). Test amb Java aturat que verifica el 502 i el format de l'error.

### 2. Logging estructurat amb correlation_id
Afegir `structlog` a Python i `logstash-logback-encoder` a Java per emetre logs en JSON. Implementar middleware (FastAPI) i filter (Spring) que generen un `X-Correlation-ID` (UUID) si no ve al header, o el propaguen si ja existeix. Cada log entry ha de tenir com a mínim: `timestamp`, `level`, `service_name`, `correlation_id`, `message`. Fer una request completa (curl -> Python -> Java -> H2) i verificar que el mateix `correlation_id` apareix als logs dels dos serveis.

**Connexió S4 (CI), S7, S8:** El logging s'integra amb el pipeline CI de S4 i els serveis de S7-S8.

**Fet quan:** Logs JSON als dos serveis amb camps consistents. El `correlation_id` es propaga correctament via header. Es pot traçar una request completa de principi a fi amb una sola cerca per `correlation_id`.

### 3. MCP server amb 3+ tools
Crear un MCP server Python amb `FastMCP` que exposa: `get_champion(champion_id)`, `search_champions(query)`, `get_champion_stats()`. El server consulta l'API REST d'EsportsPulse (S7) internament. Connectar-lo a Cursor (`.cursor/mcp.json`) i verificar que les tools apareixen i funcionen: fer preguntes en llenguatge natural i confirmar que les respostes venen de les dades reals de la BD, no de l'entrenament del model.

**Connexió S8:** A S8 vas connectar MCP servers d'altri. Ara en crees un de propi amb el mateix protocol.

**Fet quan:** MCP server funciona amb `mcp.run()`. Cursor mostra les 3+ tools al panell MCP. Es poden fer preguntes sobre champions d'EsportsPulse via chat i les respostes són dades reals de la BD.

---

## Avançats (si vas sobrat)

### 4. Circuit breaker manual
Implementar un circuit breaker simple en Python: si l'API Java falla 5 cops seguits, el circuit s'obre i retorna dades cached (últim resultat exitós guardat en memòria) durant 60 segons, sense intentar la crida HTTP. Passat el temps, prova una sola request (HALF-OPEN): si funciona, torna a CLOSED; si falla, torna a OPEN. Escriu un test que simula la seqüència CLOSED -> OPEN -> HALF-OPEN -> CLOSED amb mocks de fallades i èxits.

**Connexió S3:** Apliques conceptes de concurrència (accés thread-safe al comptador de fallades i a la cache).

**Fet quan:** Circuit breaker funciona amb els 3 estats. Test que simula 5 fallades consecutives, verifica que les crides es responen des de cache, espera el timeout, i verifica la transició HALF-OPEN -> CLOSED.

### 5. MCP server amb resource i prompt
Ampliar el MCP server amb: un resource `esportspulse://dashboard` que retorna un resum en text de tots els champions (total, top 5, estadístiques) i un prompt template `analyze-champion` que guia el model pas a pas per analitzar un champion concret. Connectar a Cursor i comparar l'experiència d'usar el MCP server amb i sense el prompt template — documenta en quin cas el model fa millors anàlisis i per què.

**Connexió S8:** Apliques els conceptes de tool_use i prompt engineering de S8 al disseny del MCP server.

**Fet quan:** Resource i prompt funcionen a Cursor. Document breu (5-10 línies) amb la comparativa: quan el prompt template millora la resposta i quan no fa diferència.
