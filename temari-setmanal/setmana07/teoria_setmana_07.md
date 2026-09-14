# Setmana 7 - Teoria: REST APIs, DTOs, Persistència i Virtual Threads

## 1. Per què una API REST és la capa de comunicació de la majoria de projectes

Fins a la setmana 6, el desenvolupament ha estat majoritàriament local i orientat al domini. Ara el sistema deixa de ser una cosa aïllada i es converteix en una plataforma amb serveis interconnectats.

Una API REST és la frontera entre:
- la lògica de negoci,
- la persistència,
- i els clients externs (frontend, CLI, altres serveis, IA).

El patró bàsic és:

```text
Client -> Controller -> Service -> Repository -> Database
```

La capa d’API no ha de conèixer com s’emmagatzema la informació; només ha de definir un contracte clar: què rep, què retorna i quan falla.

---

## 2. DTOs: separar el domini de l’API

El model de persistència (`Entity`) no és el mateix que el model de contracte de la API (`DTO`).

Per què és important?
- el model de dades pot canviar per raons internes,
- però l’API ha de mantenir estabilitat,
- i evitar exposar camps interns o sensibles.

Exemple conceptual:

```java
public record GameDto(
    String appId,
    String title,
    BigDecimal price,
    Long activePlayerCount
) {}
```

La capa REST rep i retorna DTOs, mentre la persistència continua treballant amb entitats JPA.

### Regla professional

- `Entity`: parla amb la base de dades.
- `DTO`: parla amb el client HTTP.
- `Mapper`: converteix un a l’altre.

---

## 3. Contractes REST i validació

La part més important d’una API és el contracte.

Abans de programar, cal definir:
- ruta,
- mètode HTTP,
- request body,
- response body,
- codis d’error.

Exemple:

```http
GET /games/{appId}
```

Resposta 200:

```json
{
  "appId": "APP-42",
  "title": "League of Legends",
  "price": 0.0,
  "activePlayerCount": 5000000
}
```

Resposta 404:

```json
{
  "error": "Game APP-999 not found",
  "status": 404
}
```

La validació és part del contracte. No serveix un endpoint que accepti qualsevol cosa. Amb Bean Validation es validen camps com `@NotBlank`, `@Min`, `@NotNull`.

---

## 4. Error handling: una API no és només “ok o ko”

Una API resta professional quan els errors tenen sentit per al client.

Patrons útils:
- `404` si no existeix el recurs,
- `400` si la request és invàlida,
- `500` només per errors inesperats,
- `@ControllerAdvice` per centralitzar la gestió d’excepcions.

Això evita:
- stack traces al client,
- errors no estructurats,
- dificultat per producció i debugging.

---

## 5. Virtual Threads: quan la concurrència real es manifesta

La setmana 3 ja va introduir conceptes de concurrència. Ara, amb REST, la necessitat es fa evident.

### Problema real

Si un endpoint fa crides externes (Steam, RAWG, altres APIs) i cada petició consumeix un thread de l’ecosistema Java, llavors amb moltes peticions simultànies el pool es pot saturar.

### Solució

Java 21 introdueix Virtual Threads:
- més lleugers que threads del sistema,
- adequades per treballar amb E/S bound tasks,
- ideades per a aplicacions amb moltes peticions async i I/O.

Configuració típica:

```properties
spring.threads.virtual.enabled=true
```

Aquest canvi no és una bala màgica, però sí una solució molt útil per a frameworks web amb moltes peticions de lectura i crides externes.

---

## 6. OpenAPI / Swagger: documentar el contracte

La documentació no és extra; és part del producte.

Amb SpringDoc/OpenAPI, l’API es documenta automàticament i es pot provar des d’UI:

```text
http://localhost:8080/swagger-ui.html
```

Això facilita:
- frontend,
- tests,
- integració,
- i manteniment del servei.

---

## 7. Connexió amb Python i el flux del curs

La setmana 7 posa la base de l’arquitectura de software moderna:

- Java ofereix la capa de backend robusta,
- Python pot actuar com a client de la API o com a servei addicional,
- i l’API és el punt de contracte entre ambdós.

Aquest patró és el que després s’amplia amb:
- FastAPI,
- MCP,
- agent tools,
- dashboards i integració de sistemes.

L’important és entendre que l’API no és només codi: és un contracte entre equips i serveis.

---

## 8. Objectiu d’aprenentatge de la setmana

Al final d’aquesta setmana, l’estudiant ha de ser capaç de:
- crear DTOs i controllers REST,
- validar input i output,
- gestionar errors de forma professional,
- documentar la API amb Swagger,
- i comprendre la necessitat de Virtual Threads en un sistema real.
