# Setmana 23 — Divendres: Documentacio Tecnica i PR

## Objectiu del Dia

Escriure la documentacio tecnica professional del projecte: README complet, documentacio d'API (Swagger/OpenAPI), i guia d'arquitectura. Al final del dia, el repositori ha de tenir el nivell de documentacio que s'espera d'un projecte professional de codi obert.

---

## Teoria

### Que Fa un README Professional?

El README es la primera cosa que veu algú quan arriba al teu repositori. En 30 segons ha de respondre:

```markdown
# Un bon README respon en ordre:
# 1. QUE es? (1-2 frases)
# 2. PER QUE existeix? (quin problema resol)
# 3. COM s'executa? (comandes exactes)
# 4. DEMO (screenshot o link)
# 5. ARQUITECTURA (diagrama simplificat)
# 6. STACK TECNOLOGIC (llista amb versions)
# 7. COM CONTRIBUIR (guia basica)
# 8. LLICENCIA
```

**Exemples de projectes amb READMEs excel·lents:**
- [FastAPI](https://github.com/tiangolo/fastapi) — Clar, amb exemples i badges
- [Spring Boot](https://github.com/spring-projects/spring-boot) — Professional, complet
- [LangChain](https://github.com/langchain-ai/langchain) — Bon eix de navegacio

### Badges: Indicadors Visuals

Les badges son indicadors visuals que mostren l'estat del projecte:

```markdown
<!-- Exemples de badges amb shields.io -->

<!-- Badge de CI: mostra si els tests passen -->
![CI](https://github.com/USER/REPO/actions/workflows/ci.yml/badge.svg)

<!-- Badge de cobertura de tests -->
![Coverage](https://img.shields.io/badge/coverage-85%25-green)

<!-- Badge de llicencia -->
![License](https://img.shields.io/badge/license-MIT-blue)

<!-- Badge de versio de Java -->
![Java](https://img.shields.io/badge/Java-21-orange)

<!-- Badge de versio de Python -->
![Python](https://img.shields.io/badge/Python-3.12-blue)

<!-- Badge de Docker -->
![Docker](https://img.shields.io/badge/Docker-Ready-blue)
```

### Swagger/OpenAPI: Documentacio Interactiva d'API

OpenAPI (anteriorment Swagger) es l'estandard per documentar APIs REST. Genera una interficie interactiva on pots provar els endpoints.

**En Spring Boot:**

```java
// Afegir la dependencia a pom.xml
// <dependency>
//     <groupId>org.springdoc</groupId>
//     <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
//     <version>2.3.0</version>
// </dependency>

// Amb aquesta dependencia, Spring Boot genera automaticament
// la documentacio a /swagger-ui/index.html

// Per personalitzar la documentacio d'un endpoint:
@Operation(
    summary = "Obtenir equips",
    description = "Retorna una llista paginada d'equips d'esports electronics"
)
@ApiResponse(responseCode = "200", description = "Llista d'equips retornada")
@ApiResponse(responseCode = "401", description = "No autenticat")
@GetMapping("/api/v1/teams")
public Page<TeamDTO> getTeams(
    @Parameter(description = "Terme de cerca") @RequestParam(required = false) String search,
    @Parameter(description = "Pagina (0-indexed)") Pageable pageable
) {
    return teamService.findTeams(search, pageable);
}
```

**En FastAPI (automatic):**

```python
# FastAPI genera la documentacio automaticament a /docs
# Nomes cal afegir docstrings i type hints

@router.post(
    "/api/v1/agents/quantitatiu/query",
    summary="Consultar l'agent quantitatiu",
    description="Envia una pregunta en llenguatge natural i rep una analisi estadistica.",
    response_model=AgentResponse
)
async def query_quantitative_agent(
    request: AgentQueryRequest,  # FastAPI genera l'schema del body
    current_user: User = Depends(get_current_user)  # Auth automatica
) -> AgentResponse:
    """Processa una consulta amb l'agent quantitatiu.
    
    L'agent analitza la pregunta, busca les dades rellevants,
    i retorna una resposta amb estadistiques i fonts.
    """
    return await agent_service.query(request.query, current_user)
```

### Estructura de la Documentacio Tecnica

```
docs/
  README.md                       # README principal
  architecture/
    c4-context.md                  # Diagrama C4 Nivell 1
    c4-containers.md               # Diagrama C4 Nivell 2
    c4-components-java.md          # Diagrama C4 Nivell 3 (Java)
    c4-components-python.md        # Diagrama C4 Nivell 3 (Python)
    observability-flow.md          # Flux d'observabilitat
  api/
    openapi.yml                    # Especificacio OpenAPI (generada)
  deployment/
    runbook-deployment.md          # Guia de desplegament
  security/
    owasp-checklist.md             # Checklist de seguretat
  specs/
    agent-quantitatiu.md           # OpenSpec de l'agent
    agent-knowledge.md             # OpenSpec de l'agent
```

---

## Activitat

### 1. Escriure el README Professional (40 min)

Reescriu el README des de zero seguint aquesta estructura:

```markdown
# EsportsPulse Engine

<!-- Badges -->
![CI](badge_url) ![Java](badge) ![Python](badge) ![Docker](badge)

> Plataforma d'analisi d'esports electronics amb agents d'IA
> per a analisi estadistic i knowledge retrieval.

## Demo

<!-- Screenshot del dashboard o GIF animat -->
<!-- Si tens l'app desplegada, posa el link -->

## Que Fa?

<!-- 3-5 bullets explicant les funcionalitats principals -->

## Arquitectura

<!-- Diagrama Mermaid simplificat (del C4 Nivell 2) -->

## Stack Tecnologic

| Component | Tecnologia | Versio |
|-----------|-----------|--------|
| Backend | Java + Spring Boot | 21 / 3.2 |
| AI Service | Python + FastAPI | 3.12 / 0.109 |
| Dashboard | Streamlit | 1.31 |
| Database | PostgreSQL | 16 |
| Cache | Redis | 7 |
| Message Broker | RabbitMQ | 3.13 |
| Monitoring | Prometheus + Grafana | - |
| LLM | Claude (Anthropic) | Sonnet |

## Quickstart

### Prerequisits
<!-- Que cal tenir instal·lat -->

### Execucio
<!-- docker-compose up -d i ja -->

### Verificacio
<!-- Com saber que funciona -->

## Documentacio d'API

<!-- Link a Swagger UI -->
- Backend Java: http://localhost:8080/swagger-ui/index.html
- Servei Python: http://localhost:8000/docs

## Estructura del Projecte

<!-- Arbre de directoris simplificat -->

## Desplegament

<!-- Link al runbook o instruccions basiques -->

## Contribuir

<!-- Link a CONTRIBUTING.md o instruccions basiques -->

## Llicencia

MIT License — veure [LICENSE](LICENSE)
```

### 2. Afegir Swagger al Backend Java (20 min)

```xml
<!-- Afegeix a pom.xml -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>
```

Afegeix anotacions als endpoints principals:

```java
// A cada controlador, afegeix @Operation i @ApiResponse
// per documentar que fa cada endpoint
```

Verifica que funciona:

```bash
# Arrenca el backend
docker-compose up backend-java -d

# Obre Swagger UI al navegador
open http://localhost:8080/swagger-ui/index.html
```

### 3. Verificar Documentacio de FastAPI (10 min)

FastAPI genera la documentacio automaticament:

```bash
# Arrenca el servei Python
docker-compose up ai-python -d

# Obre la documentacio interactiva
open http://localhost:8000/docs

# Verifica que tots els endpoints tenen:
# - Descripcio
# - Parametres documentats
# - Respostes d'exemple
```

### 4. Crear CONTRIBUTING.md (15 min)

```markdown
# Com Contribuir a EsportsPulse

## Flux de Treball

1. Fes fork del repositori
2. Crea una branca: `git checkout -b feature/la-teva-funcionalitat`
3. Escriu tests per als teus canvis
4. Verifica que tots els tests passen: `mvn test && pytest`
5. Fes commit amb Conventional Commits
6. Crea un Pull Request

## Convencions de Codi

### Java
- camelCase per a variables i metodes
- PascalCase per a classes
- Cada classe publica ha de tenir test unitari

### Python
- snake_case per a funcions i variables
- Type hints obligatoris
- Docstrings a totes les funcions publiques

## Executar Tests

### Backend Java
<!-- comandes -->

### Servei Python
<!-- comandes -->

### Tests E2E
<!-- comandes -->
```

### 5. Crear Fitxer LICENSE (5 min)

```bash
# Crea un fitxer LICENSE amb la llicencia MIT
# (o la que prefereixis)
```

### 6. Crear PR de la Setmana (20 min)

```bash
# Afegeix tots els fitxers de documentacio
git add README.md CONTRIBUTING.md LICENSE docs/
git add backend-java/pom.xml  # Si has afegit springdoc
git commit -m "docs: add professional README, API docs, and contributing guide"

# Crea el PR
gh pr create \
  --title "feat(s23): E2E tests, security review, and technical documentation" \
  --body "## Resum

Testing complet, revisio de seguretat, diagrames d'arquitectura,
i documentacio tecnica professional.

## Canvis
- Tests E2E per al flux complet (auth, cerca, agents)
- Load testing amb wrk (resultats documentats)
- Checklist OWASP Top 10
- Diagrames C4 (context, containers, components)
- Exercici de troubleshooting documentat
- README professional amb badges
- Swagger/OpenAPI al backend Java
- CONTRIBUTING.md i LICENSE

## Com Provar
1. docker-compose up -d
2. pytest tests/e2e/ -v
3. Obrir http://localhost:8080/swagger-ui/index.html
4. Obrir http://localhost:8000/docs

## Checklist
- [ ] Tests E2E passen
- [ ] Swagger UI funciona
- [ ] README es informatiu i complet
- [ ] Diagrames C4 renderitzen correctament"
```

---

## Checklist de Lliurament

- [ ] README professional amb totes les seccions
- [ ] Badges de CI, Java, Python, Docker al README
- [ ] Swagger UI funcionant al backend Java
- [ ] Documentacio de FastAPI completa a `/docs`
- [ ] CONTRIBUTING.md amb guia de contribucio
- [ ] LICENSE creat
- [ ] Estructura de `docs/` organitzada
- [ ] PR creat amb tots els canvis de la setmana
