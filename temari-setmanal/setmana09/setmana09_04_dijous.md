# Setmana 09 — Dijous: Middleware de Logging i Especificació de l'API

## Objectiu del Dia

Implementar un filtre de logging que registri cada petició HTTP amb tota la informació necessària per a depuració i monitoratge. Escriure l'especificació formal de l'API de Champions. Al final del dia, cada petició quedarà registrada amb mètode, path, codi d'estat, durada i X-Request-Id.

---

## Teoria

### Per Què Registrar Cada Petició?

En producció, quan alguna cosa falla, el log és l'única eina que tens per entendre què ha passat. Sense logs adequats, depurar un error és com buscar una agulla en un paller a les fosques.

**Tres raons per loguejar peticions:**

1. **Depuració**: "L'endpoint /api/champions va retornar 500 fa 5 minuts. Què va passar?"
2. **Monitoratge**: "Quants requests per segon estem rebent? Quins endpoints són més lents?"
3. **Auditoria**: "Qui va esborrar el campió amb ID 42? A quina hora?"

**Informació que necessitem per cada petició:**

```
[2024-03-15 14:32:01] INFO  --- REQUEST ---
  Method: DELETE
  Path: /api/champions/42
  Status: 204
  Duration: 23ms
  Request-Id: 550e8400-e29b-41d4-a716-446655440000
```

### X-Request-Id: Traçabilitat entre Serveis

Quan una petició travessa múltiples serveis (API Java -> Servei Python -> BD), necessitem un identificador únic que permeti seguir-la per tots els logs:

```
Client
  ↓ X-Request-Id: abc-123
API Java (log: abc-123 → GET /api/champions)
  ↓ X-Request-Id: abc-123
Servei Python (log: abc-123 → analyze champion stats)
  ↓ X-Request-Id: abc-123
Base de Dades (log: abc-123 → SELECT * FROM champions)
```

**Regla**: Si el client envia `X-Request-Id`, l'usem. Si no l'envia, en generem un de nou (UUID). Sempre el retornem a la resposta.

### OncePerRequestFilter: El Filtre de Spring Boot

Spring Boot proporciona `OncePerRequestFilter`, que garanteix que el filtre s'executa exactament un cop per petició (important amb forwards i redirects interns):

```java
// === Filtre de logging per a totes les peticions HTTP ===
// OncePerRequestFilter garanteix una sola execució per request
// @Component fa que Spring el registri automàticament
@Component
public class RequestLoggingFilter extends OncePerRequestFilter {

    // Logger estàndard de SLF4J — el framework de logging de Spring Boot
    private static final Logger log = LoggerFactory.getLogger(RequestLoggingFilter.class);

    // Nom de la capçalera que conté l'identificador únic de la petició
    private static final String REQUEST_ID_HEADER = "X-Request-Id";

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        // 1. Obtenim o generem el X-Request-Id
        // Si el client l'envia, el respectem; si no, en generem un de nou
        String requestId = request.getHeader(REQUEST_ID_HEADER);
        if (requestId == null || requestId.isBlank()) {
            requestId = UUID.randomUUID().toString();
        }

        // 2. Afegim el X-Request-Id a la resposta perquè el client el pugui veure
        response.setHeader(REQUEST_ID_HEADER, requestId);

        // 3. Registrem el moment d'inici per calcular la durada
        long startTime = System.currentTimeMillis();

        // 4. Guardem el requestId al MDC (Mapped Diagnostic Context)
        // MDC és un magatzem thread-local que permet incloure dades a TOTS els logs
        // del mateix thread sense passar-les explícitament
        MDC.put("requestId", requestId);

        try {
            // 5. Deixem que la petició continuï cap al controller
            // filterChain.doFilter() passa la petició al següent filtre o al controller
            filterChain.doFilter(request, response);
        } finally {
            // 6. Calculem la durada total de la petició
            long duration = System.currentTimeMillis() - startTime;

            // 7. Registrem tota la informació al log
            log.info("HTTP {} {} — Status: {} — Duration: {}ms — RequestId: {}",
                request.getMethod(),              // GET, POST, PUT, DELETE
                request.getRequestURI(),           // /api/champions/42
                response.getStatus(),              // 200, 404, 500...
                duration,                          // Temps en mil·lisegons
                requestId                          // Identificador únic
            );

            // 8. Netegem el MDC per evitar fuites de memòria
            // Crític amb virtual threads: el MDC és thread-local
            MDC.clear();
        }
    }
}
```

### Configuració del Format de Log

Per aprofitar el MDC, configurem el format de log:

```properties
# application.properties — Format de log personalitzat
# Incloem el requestId del MDC directament al format del log
# %X{requestId} extreu el valor del MDC amb clau "requestId"
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} [%X{requestId}] %-5level %logger{36} - %msg%n
```

Ara tots els logs dins de la mateixa petició (no només el del filtre) inclouran el requestId:

```
2024-03-15 14:32:01 [550e8400] INFO  RequestLoggingFilter - HTTP GET /api/champions — Status: 200 — Duration: 45ms — RequestId: 550e8400
2024-03-15 14:32:01 [550e8400] DEBUG ChampionService - Finding all champions with filters
2024-03-15 14:32:01 [550e8400] DEBUG ChampionRepository - SELECT * FROM champions
```

### Filtrar Paths que No Volem Loguejar

No ens interessa loguejar peticions a recursos estàtics o endpoints interns:

```java
// === Dins de RequestLoggingFilter ===
// shouldNotFilter determina quins paths NO passaran pel filtre
@Override
protected boolean shouldNotFilter(HttpServletRequest request) {
    String path = request.getRequestURI();
    // No loguegem la consola H2 ni endpoints d'actuator
    // Aquests generen molt tràfic intern que embruta els logs
    return path.startsWith("/h2-console")
        || path.startsWith("/actuator")
        || path.startsWith("/favicon.ico");
}
```

### Testejar el Filtre

```java
// === Test unitari per al filtre de logging ===
@WebMvcTest(ChampionController.class)
class RequestLoggingFilterTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private ChampionService service;

    @Test
    void shouldAddRequestIdToResponse() throws Exception {
        // Fem una petició GET sense enviar X-Request-Id
        mockMvc.perform(get("/api/champions"))
            // Verifiquem que la resposta inclou la capçalera X-Request-Id
            .andExpect(header().exists("X-Request-Id"))
            // I que és un UUID vàlid (36 caràcters amb guions)
            .andExpect(header().string("X-Request-Id",
                org.hamcrest.Matchers.matchesPattern(
                    "[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}"
                )));
    }

    @Test
    void shouldUseProvidedRequestId() throws Exception {
        String customId = "el-meu-request-id-personalitzat";
        // Enviem un X-Request-Id propi
        mockMvc.perform(get("/api/champions")
                .header("X-Request-Id", customId))
            // Verifiquem que el servidor respecta el nostre ID
            .andExpect(header().string("X-Request-Id", customId));
    }
}
```

### Especificació Formal de l'API

Una bona especificació documenta tot el que un client necessita per consumir l'API:

```markdown
# EsportsPulse — Champions API Specification

## Base URL
`http://localhost:8080/api`

## Headers Comuns
| Header          | Descripció                              | Obligatori |
|-----------------|-----------------------------------------|------------|
| Content-Type    | `application/json` per POST i PUT       | Sí (*)     |
| X-Request-Id    | UUID per traçabilitat. Generat si absent | No         |

## Endpoints

### 1. Llistar Campions
- **URL**: `GET /champions`
- **Query Params**: `name` (String), `role` (String), `minGames` (int)
- **Resposta 200**:
  ```json
  [
    { "id": 1, "name": "Ahri", "role": "Mage", "winRate": 52.3, "totalGames": 1200 }
  ]
  ```

### 2. Obtenir Campió per ID
- **URL**: `GET /champions/{id}`
- **Resposta 200**: Un objecte ChampionDTO
- **Resposta 404**: `{ "error": "Champion amb id 99 no trobat" }`

### 3. Crear Campió
- **URL**: `POST /champions`
- **Cos**:
  ```json
  { "name": "Jinx", "role": "Marksman", "winRate": 51.8 }
  ```
- **Resposta 201**: ChampionDTO creat (amb id generat)
- **Resposta 400**: `{ "name": "El nom del campió és obligatori" }`

### 4. Actualitzar Campió
- **URL**: `PUT /champions/{id}`
- **Cos**:
  ```json
  { "name": "Ahri", "role": "Mage", "winRate": 53.1, "totalGames": 1500 }
  ```
- **Resposta 200**: ChampionDTO actualitzat
- **Resposta 404**: Si l'ID no existeix

### 5. Esborrar Campió
- **URL**: `DELETE /champions/{id}`
- **Resposta 204**: Sense cos
- **Resposta 404**: Si l'ID no existeix
```

---

## Activitat

### Part 1: Implementa el RequestLoggingFilter

1. Crea `RequestLoggingFilter.java` al paquet `com.esportspulse.engine.filter`
2. Implementa tot el codi del filtre: mètode, path, status, durada, X-Request-Id
3. Configura el format de log a `application.properties`
4. Afegeix `shouldNotFilter` per excloure paths innecessaris

### Part 2: Verifica el Funcionament

```bash
# Sense X-Request-Id (el servidor en genera un)
curl -v http://localhost:8080/api/champions
# Comprova que la resposta inclou la capçalera X-Request-Id

# Amb X-Request-Id propi
curl -v -H "X-Request-Id: test-123" http://localhost:8080/api/champions
# Comprova que la resposta retorna X-Request-Id: test-123

# Verifica el log del servidor — ha de mostrar:
# HTTP GET /api/champions — Status: 200 — Duration: 23ms — RequestId: test-123
```

### Part 3: Escriu l'Especificació Completa de l'API

1. Crea `docs/api-spec.md` amb tots els endpoints, request/response bodies i codis d'error
2. Revisa que cada endpoint del controller apareix a l'especificació
3. Comprova que els exemples JSON coincideixen amb els DTOs reals

### Part 4: Tests del Filtre

1. Crea `RequestLoggingFilterTest.java`
2. Verifica que el X-Request-Id s'afegeix automàticament
3. Verifica que un X-Request-Id enviat pel client es respecta

---

## Checklist de Lliurament

- [ ] `RequestLoggingFilter` implementat amb mètode, path, status, durada i X-Request-Id
- [ ] Cada petició genera un log amb tota la informació
- [ ] X-Request-Id present a totes les respostes HTTP
- [ ] Si el client no envia X-Request-Id, el servidor en genera un (UUID)
- [ ] Si el client envia X-Request-Id, el servidor el respecta
- [ ] `docs/api-spec.md` complet amb tots els endpoints documentats
- [ ] Tests del filtre passen correctament
- [ ] Commit: `feat(logging): add request logging filter with X-Request-Id`
