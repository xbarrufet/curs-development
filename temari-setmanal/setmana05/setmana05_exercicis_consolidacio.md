# Setmana 5 — Exercicis de Consolidació

Aquests exercicis repassen els conceptes clau de la setmana. No cal lliurar-los — són per verificar que has entès la teoria i la pràctica abans de passar a la setmana 6. Intenta resoldre'ls sense mirar els apunts; si et quedes encallat, revisa el dia corresponent.

---

## Bloc 1: Principis de Disseny d'APIs REST (Dilluns)

**Exercici 1.1 — Mètodes HTTP**

Per a cadascuna d'aquestes operacions, indica quin mètode HTTP i quina URI seria correcta:

1. Obtenir la llista de tots els campions.
2. Obtenir les dades d'un campió concret pel seu ID.
3. Crear un campió nou.
4. Actualitzar completament les dades d'un campió existent.
5. Actualitzar només el `winRate` d'un campió.
6. Esborrar un campió.

**Exercici 1.2 — Codis d'Estat**

Indica quin codi HTTP retornaries en cada situació:

1. S'ha creat un campió correctament.
2. L'usuari demana un campió que no existeix.
3. L'usuari envia un JSON amb el camp `name` buit.
4. L'usuari fa GET a `/api/champions` i hi ha resultats.
5. L'usuari fa DELETE d'un campió que existeix.
6. Error intern del servidor al connectar amb la base de dades.

**Exercici 1.3 — Dissenya l'API**

Tens una nova entitat `Match` amb: `id`, `championId`, `opponentId`, `kills`, `deaths`, `assists`, `win`, `duration`.

Dissenya els endpoints REST per a:
1. CRUD bàsic de partides.
2. Obtenir totes les partides d'un campió concret.
3. Obtenir les estadístiques agregades d'un campió (total partides, win rate, KDA mitjà).

Per cada endpoint escriu: mètode HTTP, URI, body (si aplica), codi de resposta d'èxit i un exemple de resposta JSON.

**Exercici 1.4 — Detecta els Errors de Disseny**

Quins problemes de disseny REST tenen aquests endpoints? Proposa una alternativa correcta:

```
GET    /getChampion?id=jinx
POST   /champions/delete/jinx
GET    /champions/jinx/updateWinRate/52.5
POST   /api/v1/getAllChampionsList
PUT    /champion
```

---

## Bloc 2: Endpoints CRUD amb JPA (Dimarts)

**Exercici 2.1 — Arquitectura en Capes**

Dibuixa (amb text) el recorregut d'una petició `POST /api/champions` des que arriba al servidor fins que es guarda a la base de dades. Indica quines classes travessa i què fa cada una:

```
Petició HTTP → ? → ? → ? → Base de Dades
```

**Exercici 2.2 — Llegeix el Controller**

```java
@RestController
@RequestMapping("/api/champions")
public class ChampionController {

    private final ChampionManagementService service;

    public ChampionController(ChampionManagementService service) {
        this.service = service;
    }

    @GetMapping
    public List<ChampionRecord> findAll() {
        return service.findAll();
    }

    @GetMapping("/{id}")
    public ResponseEntity<ChampionRecord> findById(@PathVariable String id) {
        return service.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ChampionRecord create(@RequestBody ChampionRecord champion) {
        return service.register(champion);
    }
}
```

Respon:

1. Quina URL activa `findAll()`?
2. Quina URL activa `findById()` per al campió "jinx"?
3. Quin codi HTTP retorna `create()` quan tot va bé?
4. Quin codi HTTP retorna `findById()` quan el campió no existeix?
5. Per què el controller no accedeix directament al repository?

**Exercici 2.3 — Completa el CRUD**

Al controller anterior li falten els endpoints de `PUT` (actualitzar) i `DELETE` (esborrar). Escriu-los:

1. `PUT /api/champions/{id}` — actualitza un campió existent. Ha de retornar 200 si existeix o 404 si no.
2. `DELETE /api/champions/{id}` — esborra un campió. Ha de retornar 204 si existeix o 404 si no.

**Exercici 2.4 — Validació**

Què passa si l'usuari fa `POST /api/champions` amb aquest body?

```json
{
    "name": "",
    "role": "Mage",
    "winRate": -5.0
}
```

1. Quin codi HTTP hauria de retornar?
2. On hauria d'estar la validació: al controller, al servei o al model? Per què?
3. Escriu la resposta JSON d'error que retornaries.

---

## Bloc 3: Middleware de Logging i Especificació de l'API (Dimecres)

**Exercici 3.1 — Llegeix el Log**

Donat aquest log de peticions:

```
2024-09-15 10:23:01 INFO  [req-a1b2] GET /api/champions 200 45ms
2024-09-15 10:23:02 INFO  [req-c3d4] POST /api/champions 201 120ms
2024-09-15 10:23:02 INFO  [req-e5f6] GET /api/champions/zzz 404 8ms
2024-09-15 10:23:03 ERROR [req-g7h8] POST /api/champions 400 5ms
2024-09-15 10:23:05 ERROR [req-i9j0] GET /api/champions 500 2034ms
```

1. Quina petició ha creat un campió correctament?
2. Quina petició indica que l'usuari ha enviat dades invàlides?
3. Quina petició indica un problema al servidor? Per què ho saps?
4. Quina petició és anormalment lenta? Què podria estar passant?
5. Per què cada petició té un identificador únic (`[req-xxxx]`)?

**Exercici 3.2 — Escriu el Middleware**

Escriu el codi d'un `Filter` o `HandlerInterceptor` de Spring que:

1. Generi un request ID únic (UUID) per cada petició.
2. Registri al log: mètode HTTP, URI, codi de resposta i temps de resposta en mil·lisegons.
3. Afegeixi el request ID com a header `X-Request-Id` a la resposta.

**Exercici 3.3 — Especificació d'API**

Escriu l'especificació (en format lliure o pseudocodi OpenAPI) per a l'endpoint:

```
GET /api/champions?role={role}&minWinRate={minWinRate}
```

Inclou:
1. Descripció de l'endpoint.
2. Paràmetres (tipus, obligatori/opcional, valors per defecte).
3. Respostes possibles (200, 400, 500) amb exemple de body.
4. Un exemple de petició completa amb curl.

---

## Bloc 4: CLI Python i Tests d'Integració (Dijous)

**Exercici 4.1 — Client HTTP**

Tens l'API Java funcionant a `http://localhost:8080`. Escriu un client Python amb `requests` que:

1. Obtingui tots els campions (`GET /api/champions`).
2. Creï un campió nou (`POST /api/champions` amb JSON body).
3. Gestioni errors: si l'API retorna 404, imprimeixi un missatge clar en lloc del traceback.
4. Tingui un timeout de 5 segons per a cada petició.

**Exercici 4.2 — CLI amb argparse**

Dissenya un CLI amb `argparse` que suporti:

```bash
# Llistar tots els campions
python cli.py list

# Llistar campions filtrats per rol
python cli.py list --role Mage

# Mostrar un campió concret
python cli.py show jinx

# Crear un campió
python cli.py create --name jinx --role Marksman --win-rate 51.5

# Esborrar un campió
python cli.py delete jinx
```

Escriu l'estructura del codi (parser, subcommands) sense implementar tota la lògica.

**Exercici 4.3 — Tests d'Integració**

Explica la diferència entre testejar el CLI amb:

1. **Mocks** (`unittest.mock.patch` sobre `requests.get`).
2. **Servidor real** (aixecar l'API Java i fer peticions reals).
3. **Servidor fals** (`responses` library o `httpretty`).

Per a cadascun, indica: avantatges, inconvenients i quan l'usaries.

**Exercici 4.4 — Escriu el Test**

Escriu un test amb `pytest` i `unittest.mock` que:

1. Mockegi `requests.get` perquè retorni una llista de 2 campions.
2. Cridi la funció `list_champions()` del teu client.
3. Verifiqui que la URL cridada és correcta.
4. Verifiqui que el resultat conté 2 campions.

---

## Bloc 5: Demo End-to-End i Walking Skeleton (Divendres)

**Exercici 5.1 — Walking Skeleton**

Respon:

1. Què és un "Walking Skeleton"?
2. Per què és valuós tenir un Walking Skeleton al final de la setmana 5, encara que el projecte estigui molt lluny d'estar complet?
3. Quines capes travessa una petició en el teu Walking Skeleton actual (de punta a punta)?

**Exercici 5.2 — Checklist de Demo**

Si haguessis de fer una demo de 3 minuts del que has construït aquesta setmana, quins 5 passos mostraries? Ordena'ls de més a menys impactant:

1. ...
2. ...
3. ...
4. ...
5. ...

Pista: pensa en què impressionaria a un enginyer senior que mai ha vist el teu projecte.

**Exercici 5.3 — Flux Complet**

Dibuixa (amb text) el flux complet d'una demo:

```
Usuari executa CLI → ... → ... → ... → Resposta al terminal
```

Inclou: CLI Python, petició HTTP, Controller, Service, Repository, H2, i la tornada.

**Exercici 5.4 — Reflexió**

Respon breument:

1. Quina part de la setmana t'ha costat més? Per què?
2. Si poguessis tornar al dilluns, què faries diferent?
3. Quina habilitat nova de la setmana 5 creus que usaràs més sovint en un treball real?
4. Escriu una frase que podries dir en una entrevista sobre el que has après aquesta setmana.

---

## Autoavaluació

Abans de passar a la setmana 6, hauries de poder respondre "sí" a tot:

- [ ] Sé els 5 mètodes HTTP principals i quan usar cadascun
- [ ] Sé dissenyar URIs RESTful per a qualsevol recurs
- [ ] Sé quins codis HTTP retornar en cada situació (200, 201, 204, 400, 404, 500)
- [ ] Sé implementar un CRUD complet amb Spring Boot, JPA i H2
- [ ] Entenc l'arquitectura en capes: Controller → Service → Repository
- [ ] Sé escriure un middleware de logging que registri cada petició
- [ ] Sé crear un CLI en Python amb argparse que consumeixi l'API
- [ ] Sé escriure tests d'integració que validen el flux CLI → API
- [ ] Puc fer una demo end-to-end del Walking Skeleton
- [ ] Sé explicar què és un Walking Skeleton i per què és valuós
