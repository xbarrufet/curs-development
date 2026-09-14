# Setmana 8 — Exercicis de Consolidació

---

## Bàsics (has de saber fer-ho)

### 1. Pydantic models per EsportsPulse
Crea els tres models Pydantic amb validació completa: `ChampionAnalysis`, `ChampionTrend`, `PatchSummary`. Validacions: `champion_id` ha de ser un nom vàlid amb almenys 2 caràcters, `patch_tier` limitat a S/A/B/C/D, `current_win_rate` i `previous_win_rate` entre 0 i 100, `trend_direction` limitat a `up`/`down`/`stable`, `date` ha de ser una data vàlida. Afegeix `Field()` amb `description` i `examples` a cada camp. Genera el JSON schema amb `model_json_schema()` i guarda'l a un fitxer `schemas/champion_analysis.json`.

**Connexió S2 (immutabilitat, records), S7 (DTOs):** Compara mentalment amb els Java records i els DTOs de Spring — el concepte és el mateix, la sintaxi canvia.

**Fet quan:** Models creats, validació funciona (dades vàlides acceptades, dades invàlides rebutjades amb errors clars), schema JSON generat.

### 2. Script d'anàlisi amb LLM
Script Python que rep 3 `champion_ids` per línia de comandes, crida l'API Java (S7) per obtenir les dades de cada champion, construeix un prompt amb few-shot (2 exemples), crida un LLM amb `tool_use`/`function_calling` per obtenir output estructurat, i valida cada resposta amb Pydantic. Imprimeix els 3 `ChampionAnalysis` resultants en JSON.

**Connexió S7 (REST API consumida des de Python).**

**Fet quan:** Script funciona end-to-end amb 3 champions (Java backend running), output és JSON vàlid i validat per Pydantic.

### 3. Endpoint FastAPI /analyze/{champion_id}
Crea un endpoint FastAPI que rep un `champion_id`, consulta l'API Java, crida el LLM amb tool_use, i retorna un `ChampionAnalysis`. Afegeix un `GET /health` i verifica que `/docs` mostra la documentació auto-generada. Escriu 3 tests amb pytest: resposta LLM vàlida, resposta LLM invàlida (Pydantic error), champion no trobat (404). Usa `monkeypatch` per mockejar tant l'API Java com el LLM.

**Connexió S6 (testing, mocks), S7 (REST API).**

**Fet quan:** Endpoint funciona, 3 tests passen, documentació visible a `/docs`.

---

## Avançats (si vas sobrat)

### 4. MCP server bàsic per EsportsPulse
Crea un MCP server en Python (amb `mcp` SDK) que exposa 3 tools read-only: `get_champion(champion_id)` retorna les dades d'un champion, `search_champions(query)` cerca per nom, `get_top_champions(n)` retorna els N champions amb més winrate. El server consulta l'API Java internament. Configura'l a `.cursor/mcp.json` i verifica que Cursor pot fer queries a EsportsPulse des del chat.

**Fet quan:** MCP server arrenca, Cursor el detecta, i pots preguntar "Quants champions tenen winrate > 52%?" des del chat i obtenir resposta real.

### 5. Comparativa Java DTOs vs Pydantic
Implementa els mateixos 3 models (`ChampionAnalysis`, `ChampionTrend`, `PatchSummary`) com a Java records amb Bean Validation (`@NotBlank`, `@Min`, `@Max`, compact constructor) i com a Pydantic models. Compara: línies de codi per model, facilitat de validació, serialització JSON (Jackson vs `model_dump_json`), generació de schema. Escriu un document breu amb les conclusions: quan prefereixes cada opció i per què.

**Connexió S2 (records, immutabilitat), S7 (DTOs, Bean Validation).**

**Fet quan:** Models implementats en ambdós llenguatges, comparativa escrita amb exemples concrets i mètriques (línies, features).
