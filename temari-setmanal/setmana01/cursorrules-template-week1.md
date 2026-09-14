# .cursorrules - EsportsPulse Initial Setup (Setmana 1)

```
# EsportsPulse - Java Backend + Python IA Rules

## Project Structure
- `/backend-java`: Spring Boot 3 microservices with Java 21
- `/ai-python`: Python agents, RAG pipelines, LLMs
- Keep concerns separated: never import Java code into Python; APIs via REST

## Naming Conventions
- **Java packages:** `com.esportspulse.{domain}.{layer}` (e.g., `com.esportspulse.game.service`)
- **Java classes:** PascalCase (e.g., `GameSearchService`, `GameRecord`)
- **Java variables/methods:** camelCase (e.g., `appId`, `searchByHash()`)
- **Python modules:** snake_case (e.g., `game_search_service.py`)
- **Python classes:** PascalCase (e.g., `GameSearchService`)

## Code Quality
- **Java:** Use records for immutable DTOs; no setters
- **Python:** Use Pydantic models for data validation
- **Git commits:** Conventional Commits format (`feat(java): X`, `fix(python): Y`)
- **Tests:** One test per behavior; use descriptive names

## IA Assistance Rules
- Use Cursor for boilerplate generation (e.g., builders, tests)
- Always review generated code for correctness
- Architecture decisions are human-only; don't let IA auto-generate package structures

## Setmana 1 Focus
- Benchmarking O(n) vs O(1) in Java
- No AI optimizations yet; focus on clarity
- All code reviewed manually before commit
```

---

**Com usar-ho:**
1. Copia aquest contingut al fitxer `.cursorrules` del projecte
2. Ajusta les regles que no quadrin amb la teva arquitectura
3. Setmana a setmana, afegeix noves regles (ex: RAG patterns a setmana 11, agents a setmana 15)
