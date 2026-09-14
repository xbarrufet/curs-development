# Setmana 19 — Dilluns: Síncrono vs Asíncron: Quan Usar Cues

## Objectiu del Dia

Entendre la diferència fonamental entre comunicació síncrona (REST) i asíncrona (message queues). Identificar quins fluxos d'EsportsPulse es beneficien de processament asíncron. Al final del dia, tens un disseny clar dels events que implementaràs durant la setmana.

---

## Teoria

### Comunicació Síncrona: REST

Fins ara, tots els fluxos d'EsportsPulse són síncrons:

```
Client → POST /api/champions → Backend Java → Guarda a PostgreSQL → 201 Created
         ←───────────────────────────────────────────────────────────┘
         El client ESPERA fins que tot acabi (~50ms)
```

**Avantatges:**
- Simple: fas la crida i obtens la resposta immediatament
- Fàcil de depurar: si falla, saps on i per què
- El client sap si l'operació ha tingut èxit

**Problemes:**
- **Bloqueig:** Si el backend triga 30 segons (ex: cridar a un LLM), el client espera 30 segons
- **Acoblament:** Si el servei Python cau, el servei Java no pot completar l'operació
- **Escalabilitat:** Cada petició ocupa un thread. Amb 1000 peticions simultànies a un LLM, el servidor es queda sense threads

### Comunicació Asíncrona: Message Queues

```
Client → POST /api/champions → Backend Java → Guarda a PostgreSQL → 202 Accepted
         ←───────────────────────────────────┘
         El client rep resposta IMMEDIATAMENT (~5ms)

         (En segon pla, al seu ritme:)
         Backend Java → Publica event → [RabbitMQ] → Consumer Python → LLM → Redis
```

**Avantatges:**
- **No bloqueig:** El client no espera processos llargs
- **Desacoblament:** Si el servei Python cau, els missatges s'acumulen a la cua i es processen quan torni
- **Escalabilitat:** Pots tenir 10 consumers processant en paral·lel

**Desavantatges:**
- Més complex: el client no sap immediatament si el processament ha tingut èxit
- Debugging més difícil: el missatge pot estar a la cua, al consumer, o perdut
- Eventual consistency: les dades no estan actualitzades instantàniament

### Quan Usar Cada Model

| Situació | Model | Per què |
|----------|-------|---------|
| Usuari crea un campió | Síncrono | L'usuari necessita saber si s'ha creat correctament |
| Generar resum LLM del campió | Asíncron | Triga 5-30s, l'usuari no vol esperar |
| Consultar llista de campions | Síncrono | Resposta ràpida des de cache/DB |
| Indexar campió a Qdrant | Asíncron | Procés en segon pla, no bloqueja l'usuari |
| Login / autenticació | Síncrono | L'usuari necessita el token immediatament |
| Enviar notificació | Asíncron | L'usuari no espera que arribi l'email |

### Anatomia d'un Event

Un event és un missatge que descriu **quelcom que ha passat** (no una ordre de fer alguna cosa):

```json
{
  "eventId": "evt-abc123",
  "eventType": "champion.created",
  "timestamp": "2024-03-15T10:30:00Z",
  "payload": {
    "championId": 42,
    "name": "Ahri",
    "role": "MID",
    "description": "Maga amb mobilitat..."
  }
}
```

**Bones pràctiques:**
- **eventId únic:** Permet detectar duplicats (idempotència, ho veurem dijous)
- **eventType:** Identifica què ha passat. Conveni: `entitat.acció` (champion.created, champion.updated)
- **timestamp:** Quan va passar. Imprescindible per ordering i debugging
- **payload:** Les dades necessàries perquè el consumer pugui treballar sense cridar de tornada al productor

> **Lectura recomanada (opcional, no bloquejant):**
> - [RabbitMQ Tutorials](https://www.rabbitmq.com/tutorials) — Tutorials oficials
> - [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/) — Patrons de missatgeria

---

## Activitat

### 1. Mapejar els fluxos d'EsportsPulse (30 min)

Revisa l'aplicació i classifica cada flux com a síncron o asíncron. Crea un document `docs/async-design.md`:

```markdown
# Disseny de Fluxos Asíncrons — EsportsPulse

## Fluxos que es mantenen síncrons
| Flux | Motiu |
|------|-------|
| GET /api/champions | Resposta ràpida des de cache/DB |
| POST /api/champions (crear) | L'usuari necessita confirmació immediata |
| POST /api/auth/login | El token JWT s'ha de retornar immediatament |
| GET /api/champions/search | Query directa a PostgreSQL/Qdrant |

## Fluxos que passen a asíncrons
| Flux | Event | Consumer | Motiu |
|------|-------|----------|-------|
| Crear campió → generar resum LLM | champion.created | Python LLM | Triga 5-30s |
| Crear campió → indexar a Qdrant | champion.created | Python Qdrant | Procés secundari |
| Actualitzar campió → re-generar resum | champion.updated | Python LLM | No bloquejar PUT |
```

### 2. Dissenyar els schemas dels events (30 min)

Defineix l'estructura JSON de cada event. Crea `docs/event-schemas.json`:

```json
{
  "champion.created": {
    "description": "S'ha creat un nou campió a la base de dades",
    "schema": {
      "eventId": "string (UUID)",
      "eventType": "champion.created",
      "timestamp": "string (ISO 8601)",
      "payload": {
        "championId": "number",
        "name": "string",
        "role": "string",
        "description": "string"
      }
    },
    "producer": "backend-java",
    "consumers": ["python-llm-summary", "python-qdrant-indexer"],
    "example": {
      "eventId": "evt-550e8400-e29b-41d4-a716-446655440000",
      "eventType": "champion.created",
      "timestamp": "2024-03-15T10:30:00Z",
      "payload": {
        "championId": 42,
        "name": "Ahri",
        "role": "MID",
        "description": "Maga amb mobilitat i ràfegues de dany"
      }
    }
  },
  "champion.updated": {
    "description": "S'han modificat les dades d'un campió existent",
    "schema": {
      "eventId": "string (UUID)",
      "eventType": "champion.updated",
      "timestamp": "string (ISO 8601)",
      "payload": {
        "championId": "number",
        "name": "string",
        "role": "string",
        "description": "string",
        "changedFields": ["string"]
      }
    },
    "producer": "backend-java",
    "consumers": ["python-llm-summary"]
  }
}
```

### 3. Dibuixar el diagrama d'arquitectura (20 min)

Dibuixa (paper, Miro, o ASCII) com queda el sistema amb les cues:

```
                                    ┌─────────────┐
                                    │   Streamlit  │
                                    │  (Frontend)  │
                                    └──────┬───────┘
                                           │ HTTP
                                    ┌──────┴───────┐
        ┌───────────────────────────│  Java Backend │
        │                           │  (Spring Boot)│
        │                           └──────┬───────┘
        │ Events                           │ SQL
   ┌────┴────┐                      ┌──────┴───────┐
   │ RabbitMQ │                      │  PostgreSQL   │
   │  (Cues)  │                      └──────────────┘
   └────┬────┘
        │ Consume
   ┌────┴────────────┐
   │ Python Consumer  │
   │ (LLM + Qdrant)  │
   └──┬──────────┬───┘
      │          │
 ┌────┴───┐ ┌───┴────┐
 │ Redis  │ │ Qdrant │
 │(Cache) │ │(Vector)│
 └────────┘ └────────┘
```

### 4. Crear la classe Event a Java (20 min)

Prepara el codi base per als events:

```java
// ChampionEvent.java — Representa un event de campió
// Aquesta classe es serialitzarà a JSON i s'enviarà a RabbitMQ

public class ChampionEvent {
    // Identificador únic de l'event (per detectar duplicats)
    private String eventId;
    // Tipus d'event: "champion.created", "champion.updated"
    private String eventType;
    // Moment exacte en què s'ha produït l'event
    private Instant timestamp;
    // Dades del campió afectat
    private ChampionPayload payload;

    // Constructor que genera automàticament l'eventId i el timestamp
    public static ChampionEvent created(Champion champion) {
        ChampionEvent event = new ChampionEvent();
        event.setEventId(UUID.randomUUID().toString());
        event.setEventType("champion.created");
        event.setTimestamp(Instant.now());
        event.setPayload(ChampionPayload.from(champion));
        return event;
    }

    // Getters i setters...
}

// ChampionPayload.java — Dades del campió dins l'event
public class ChampionPayload {
    private Long championId;
    private String name;
    private String role;
    private String description;

    // Factory method per crear el payload des d'una entitat Champion
    public static ChampionPayload from(Champion champion) {
        ChampionPayload payload = new ChampionPayload();
        payload.setChampionId(champion.getId());
        payload.setName(champion.getName());
        payload.setRole(champion.getRole());
        payload.setDescription(champion.getDescription());
        return payload;
    }

    // Getters i setters...
}
```

### 5. Commit (5 min)

```bash
git add .
git commit -m "docs(async): design async event flows and champion event schema"
```

---

## Checklist de Lliurament

- [ ] Document `async-design.md` amb fluxos classificats (síncron vs asíncron)
- [ ] Event schemas definits per `champion.created` i `champion.updated`
- [ ] Diagrama d'arquitectura amb RabbitMQ dibuixat
- [ ] Classe `ChampionEvent` creada amb factory method
- [ ] Classe `ChampionPayload` creada
- [ ] Justificació documentada de per què cada flux és síncron o asíncron
- [ ] Commit fet
