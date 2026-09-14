# Setmana 7 — Exercicis de Consolidació

## Bàsics (has de saber fer-ho)

### 1. Construir una API CRUD de champions
Crea una API REST per `Champion` amb els endpoints:
- `GET /champions/{championId}`
- `GET /champions`
- `POST /champions`
- `PUT /champions/{championId}`
- `DELETE /champions/{championId}`

Requisits:
- DTOs separats del model de persistència
- validació d’input (`name` no buit, `winRate` entre 0 i 100)
- `404` si el champion no existeix
- `201` en creació
- `204` en eliminació

**Fet quan:** els endpoints funcionen amb curl/Postman i retornin errors consistents.

### 2. Spec d’API en markdown i validació contra codi
Escriu una `api-spec.md` amb:
- ruta,
- mètode,
- request body,
- response body,
- codis d’error

Després genera el controller i els DTOs a partir d’aquesta spec i valida:
- un GET existent retorna JSON correcte
- un GET no existent retorna 404
- un POST invàlid retorna 400

**Fet quan:** el codi generat coincideix amb la spec i els exemples de resposta són correctes.

### 3. Global exception handler
Configura un `@ControllerAdvice` que converteix excepcions com:
- `EntityNotFoundException`
- `ValidationException`
- errors de serialització

en una resposta JSON estructurada amb `status` i `error`.

**Fet quan:** totes les respostes d’error tenen format coherent i el client no rep stack traces cruels.

---

## Avançats (si vas sobrat)

### 4. Virtual Threads i benchmark simple
Crea una versió de prova de `ChampionDataExtractor` que executi 20 crides concurrentes a la Riot API o un endpoint local simulant espera.

Compara:
- `Executors.newFixedThreadPool(10)`
- `Executors.newVirtualThreadPerTaskExecutor()`

Mesura temps i anota la diferència.

**Fet quan:** pots explicar en què millora Virtual Threads i quan no és la solució adequada.

### 5. CLI Python que consumeix l’API REST
Crea una mini CLI en Python amb `requests` que permeti:
- llistar champions,
- consultar un champion per ID,
- crear un champion nou,
- i mostrar resultats en format legible.

**Fet quan:** la CLI pot interactuar amb l’API Java i la sortida és clara per a un usuari de terminal.

---

## Connexió amb les setmanes anteriors

- **S5**: persistència JPA i repositories
- **S6**: tests de controller i validació
- **S3**: concurrència i async

La setmanes 7 és on la teoria de domini i arquitectura es transforma en un sistema de producció amb contractes reals.
