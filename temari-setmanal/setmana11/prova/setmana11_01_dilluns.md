# Setmana 11 — Dilluns: Exception Handling Design

## Objectiu del Dia

Implementar un sistema centralitzat de gestió d'errors tant al servei Java (Spring Boot) com al servei Python (FastAPI), de manera que tots els errors retornin respostes consistents en format **Problem Details (RFC 7807)** i cap stack trace es filtri mai al client.

---

## Teoria

### Per Què try-catch No és Suficient en Sistemes Distribuïts

Quan EsportsPulse era un sol servei Java, un `try-catch` genèric podia capturar qualsevol error. Ara tenim dos serveis (Java i Python) que es comuniquen per HTTP. Això canvia completament la gestió d'errors:

```
[Client] → [Python FastAPI] → [Java Spring Boot] → [H2 Database]
                  ↓                    ↓
              [LLM API]          [Riot API Mock]
```

Un `try-catch` local **no pot gestionar** errors que vénen de la xarxa. El servei Python no sap si Java ha fallat perquè:
- La base de dades estava plena
- El servei Java ni tan sols estava arrencat
- La xarxa ha tallat a mig camí
- Java ha respost però amb un format inesperat

### Tipus d'Errors Remots

| Error | Què Passa | Codi HTTP |
|-------|-----------|-----------|
| **ConnectionError** | El servei destí no respon (apagat, port incorrecte) | 502 Bad Gateway |
| **Timeout** | El servei destí triga massa (consulta pesada, deadlock) | 504 Gateway Timeout |
| **Partial Failure** | El servei respon però amb dades incompletes | 502 o 206 |
| **Cascading Failure** | Un error a Java causa errors en cadena a Python i al client | 503 Service Unavailable |

### Cascading Failure — L'Error en Cascada

```
1. Java triga 30s a respondre (query lenta a H2)
2. Python espera 30s → els seus threads s'esgoten
3. Més peticions arriben a Python → totes queden bloquejades
4. El client veu timeouts massius
5. Tot el sistema cau
```

**Regla d'or:** Mai confiar que un servei extern respondrà ràpid. Sempre timeout + retry + fallback.

### Excepcions Personalitzades a Java

En lloc de llençar `RuntimeException` genèriques, creem excepcions que descriuen exactament què ha fallat:

```java
// Excepció quan un campió no existeix a la base de dades.
// Extenem RuntimeException perquè és unchecked (no cal declarar-la amb throws).
public class ChampionNotFoundException extends RuntimeException {

    // Guardem l'ID que s'ha buscat per poder informar l'usuari.
    private final Long championId;

    public ChampionNotFoundException(Long championId) {
        // Missatge descriptiu per als logs interns.
        super("Champion amb ID " + championId + " no trobat");
        this.championId = championId;
    }

    public Long getChampionId() {
        return championId;
    }
}
```

```java
// Excepció quan un servei extern (Python, LLM, Riot API) no respon.
// Distingim de ChampionNotFound perquè la causa i el codi HTTP són diferents.
public class ExternalServiceUnavailableException extends RuntimeException {

    // Nom del servei que ha fallat, per logs i diagnòstic.
    private final String serviceName;

    public ExternalServiceUnavailableException(String serviceName, Throwable cause) {
        super("Servei extern no disponible: " + serviceName, cause);
        this.serviceName = serviceName;
    }

    public String getServiceName() {
        return serviceName;
    }
}
```

### @ControllerAdvice — Gestió Centralitzada d'Errors a Spring Boot

En lloc de posar try-catch a cada controller, creem **un sol punt** que intercepta totes les excepcions:

```java
// @ControllerAdvice intercepta TOTES les excepcions llençades des de qualsevol controller.
// Això centralitza la gestió d'errors: cap controller necessita try-catch.
@ControllerAdvice
public class GlobalExceptionHandler {

    // Captura ChampionNotFoundException i retorna 404 amb Problem Details.
    @ExceptionHandler(ChampionNotFoundException.class)
    public ProblemDetail handleChampionNotFound(ChampionNotFoundException ex) {
        // ProblemDetail és la classe nativa de Spring Boot 3 per RFC 7807.
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND,
            ex.getMessage()
        );
        // 'type' és una URI que identifica el tipus d'error (documentació).
        problem.setType(URI.create("https://esportspulse.dev/errors/champion-not-found"));
        // 'title' és un resum curt per a humans.
        problem.setTitle("Champion No Trobat");
        // Camps addicionals personalitzats per facilitar debugging.
        problem.setProperty("championId", ex.getChampionId());
        return problem;
    }

    // Captura errors de serveis externs i retorna 502 (Bad Gateway).
    @ExceptionHandler(ExternalServiceUnavailableException.class)
    public ProblemDetail handleExternalServiceUnavailable(ExternalServiceUnavailableException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.BAD_GATEWAY,
            ex.getMessage()
        );
        problem.setType(URI.create("https://esportspulse.dev/errors/external-service-unavailable"));
        problem.setTitle("Servei Extern No Disponible");
        problem.setProperty("serviceName", ex.getServiceName());
        return problem;
    }

    // Captura qualsevol altra excepció no prevista.
    // MAI retornem el stack trace — seria una vulnerabilitat de seguretat.
    @ExceptionHandler(Exception.class)
    public ProblemDetail handleGenericError(Exception ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR,
            // Missatge genèric per al client. El detall va als logs.
            "Error intern del servidor. Contacta l'administrador."
        );
        problem.setType(URI.create("https://esportspulse.dev/errors/internal"));
        problem.setTitle("Error Intern");
        return problem;
    }
}
```

### RFC 7807 — Problem Details for HTTP APIs

És l'estàndard que defineix com retornar errors en APIs REST. Spring Boot 3 el suporta de forma nativa.

**Format de resposta:**

```json
{
    "type": "https://esportspulse.dev/errors/champion-not-found",
    "title": "Champion No Trobat",
    "status": 404,
    "detail": "Champion amb ID 999 no trobat",
    "instance": "/api/champions/999",
    "championId": 999
}
```

| Camp | Descripció | Obligatori |
|------|------------|------------|
| `type` | URI que identifica el tipus d'error | Sí |
| `title` | Resum curt per a humans | Sí |
| `status` | Codi HTTP numèric | Sí |
| `detail` | Explicació detallada d'aquest cas concret | Recomanat |
| `instance` | URI de la petició que ha causat l'error | Opcional |

### Exception Handlers a FastAPI (Python)

L'equivalent de `@ControllerAdvice` a FastAPI:

```python
# Excepció personalitzada per quan el servei Java no respon.
# Definim una classe pròpia per distingir-la d'errors genèrics.
class JavaServiceUnavailableError(Exception):
    def __init__(self, detail: str = "Servei Java no disponible"):
        self.detail = detail

# Excepció quan un campió no existeix.
class ChampionNotFoundError(Exception):
    def __init__(self, champion_id: int):
        self.champion_id = champion_id
        self.detail = f"Champion amb ID {champion_id} no trobat"
```

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()

# Handler global per a ChampionNotFoundError.
# FastAPI el cridarà automàticament quan es llenci aquesta excepció.
@app.exception_handler(ChampionNotFoundError)
async def champion_not_found_handler(request: Request, exc: ChampionNotFoundError):
    # Retornem Problem Details format, igual que Java.
    # Consistència entre serveis = menys confusió per al client.
    return JSONResponse(
        status_code=404,
        content={
            "type": "https://esportspulse.dev/errors/champion-not-found",
            "title": "Champion No Trobat",
            "status": 404,
            "detail": exc.detail,
            "instance": str(request.url),
            "champion_id": exc.champion_id,
        },
    )

# Handler per quan Java no respon.
@app.exception_handler(JavaServiceUnavailableError)
async def java_unavailable_handler(request: Request, exc: JavaServiceUnavailableError):
    return JSONResponse(
        status_code=502,
        content={
            "type": "https://esportspulse.dev/errors/external-service-unavailable",
            "title": "Servei Extern No Disponible",
            "status": 502,
            "detail": exc.detail,
            "instance": str(request.url),
        },
    )

# Handler genèric per a qualsevol excepció no controlada.
# Protecció: mai retornem stack traces al client.
@app.exception_handler(Exception)
async def generic_error_handler(request: Request, exc: Exception):
    return JSONResponse(
        status_code=500,
        content={
            "type": "https://esportspulse.dev/errors/internal",
            "title": "Error Intern",
            "status": 500,
            "detail": "Error intern del servidor.",
            "instance": str(request.url),
        },
    )
```

### Codis d'Estat HTTP — Quan Usar Cada Un

| Codi | Significat | Quan Usar-lo a EsportsPulse |
|------|-----------|---------------------------|
| **400** Bad Request | Dades invàlides enviades pel client | JSON mal format, camp obligatori absent |
| **404** Not Found | El recurs no existeix | `GET /champions/999` quan l'ID 999 no existeix |
| **422** Unprocessable Entity | Format correcte però dades invàlides semànticament | `winRate: -5` (negatiu no té sentit) |
| **500** Internal Server Error | Error intern no previst | NullPointerException, error de lògica |
| **502** Bad Gateway | El servei intermediari ha rebut resposta invàlida | Python rep error de Java |
| **504** Gateway Timeout | El servei intermediari no ha rebut resposta a temps | Java no respon en 5s |

**Regla pràctica:**
- **4xx** → El client ha fet alguna cosa malament (pot corregir-ho).
- **5xx** → El servidor ha fallat (el client no pot fer-hi res, ha de reintentar o esperar).

---

## Activitat

### Pas 1: Crear les Excepcions Personalitzades a Java

Crea les classes `ChampionNotFoundException` i `ExternalServiceUnavailableException` al paquet `com.esportspulse.engine.exception`.

### Pas 2: Implementar @ControllerAdvice

Crea `GlobalExceptionHandler` com es mostra a la teoria. Habilita Problem Details a `application.properties`:

```properties
# Activa el suport natiu de Problem Details a Spring Boot 3.
# Sense això, Spring retorna el format d'error antic (timestamp, path, error).
spring.mvc.problemdetails.enabled=true
```

### Pas 3: Actualitzar el Controller Existent

```java
// Al ChampionController, llencem l'excepció personalitzada.
// El @ControllerAdvice la capturarà i la convertirà en Problem Details.
@GetMapping("/{id}")
public ChampionDTO getChampion(@PathVariable Long id) {
    return championService.findById(id)
        .orElseThrow(() -> new ChampionNotFoundException(id));
}
```

### Pas 4: Implementar Handlers a FastAPI

Afegeix les excepcions i handlers a l'aplicació Python com es mostra a la teoria. Modifica les crides a Java per llençar les excepcions correctes:

```python
import httpx

# Funció que crida al servei Java per obtenir un campió.
# Gestiona els errors de xarxa i els converteix en excepcions nostres.
async def get_champion_from_java(champion_id: int) -> dict:
    try:
        async with httpx.AsyncClient(timeout=5.0) as client:
            response = await client.get(
                f"http://localhost:8080/api/champions/{champion_id}"
            )
        # Si Java retorna 404, llencem la nostra excepció.
        if response.status_code == 404:
            raise ChampionNotFoundError(champion_id)
        # Qualsevol altre error de Java.
        response.raise_for_status()
        return response.json()
    except httpx.ConnectError:
        # Java no està arrencat o el port és incorrecte.
        raise JavaServiceUnavailableError("No es pot connectar al servei Java")
    except httpx.TimeoutException:
        # Java triga massa a respondre.
        raise JavaServiceUnavailableError("Timeout connectant al servei Java")
```

### Pas 5: Verificar

```bash
# Arrencar Java i Python, i provar:
# 1. Campió que existeix → 200 OK
curl http://localhost:8000/api/champions/1

# 2. Campió que NO existeix → 404 Problem Details
curl http://localhost:8000/api/champions/999

# 3. Apagar Java i cridar Python → 502 Problem Details
curl http://localhost:8000/api/champions/1
```

---

## Checklist de Lliurament

- [ ] `ChampionNotFoundException` i `ExternalServiceUnavailableException` creades a Java
- [ ] `@ControllerAdvice` implementat amb `ProblemDetail` per a tots els casos
- [ ] Exception handlers equivalents a FastAPI
- [ ] Totes les respostes d'error segueixen el format Problem Details (RFC 7807)
- [ ] Cap stack trace es filtra al client (verificat amb curl)
- [ ] Codis HTTP correctes: 404 per recursos inexistents, 502 per servei extern caigut
- [ ] Commit amb missatge: `feat(error-handling): implement centralized exception handling with Problem Details`
