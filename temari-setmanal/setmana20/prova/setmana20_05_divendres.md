# Setmana 20 — Divendres: Consolidació Final del Bloc 4

## Objectiu del Dia

Verificar que tot el sistema funciona de punta a punta amb `docker-compose up`. CI net. Health checks passen. Mètriques visibles. Documentar el runbook operacional. Això no és aprendre una tecnologia nova: és unificar tot el que has construït des de la Setmana 1.

---

## Teoria

### Què Hem Construït en 20 Setmanes

Pren un moment per mirar enrere:

```
S1-S3:   Java bàsic, Git, terminal, algorismes
S4-S5:   Spring Boot REST API, CI amb GitHub Actions
S6-S7:   Python, FastAPI, tests
S8:      Docker, contenidors
S9-S10:  Autenticació JWT, Streamlit frontend
S11-S12: LLM integration, knowledge retrieval amb Qdrant
S13-S14: Agents, tool use, spec-driven development
S15-S17: Integració completa
S18:     PostgreSQL, Flyway, SQL avançat, Redis cache
S19:     RabbitMQ, missatgeria asíncrona, resiliència
S20:     CI unificat, Docker complet, health checks, monitoring
```

**Resultat:** Una plataforma amb 8 serveis interconnectats, CI automatitzat, processament asíncron, cache, monitoring i una pipeline LLM. Això és una arquitectura de microserveis real.

### Què És un Runbook?

Un **runbook** és un document operacional que explica com gestionar la plataforma en situacions reals:

- Com arrencar el sistema
- Com verificar que tot funciona
- Com llegir logs quan alguna cosa falla
- Com investigar errors comuns
- Com aturar el sistema de forma segura

No és un manual acadèmic: és una guia pràctica per a qui hagi d'operar el sistema (inclòs tu en 6 mesos quan no recordis com funciona).

### La Prova de Foc: El Cold Start

La prova definitiva del sistema: clonar el repositori en una màquina neta i fer-lo funcionar.

```bash
# Simulació de cold start
git clone https://github.com/user/esportspulse-engine.git
cd esportspulse-engine
cp .env.example .env
# Editar .env amb els valors reals
docker-compose up --build
```

Si això funciona sense cap intervenció manual, el projecte és operable.

---

## Activitat

### 1. Full Platform Test — Cold Start (30 min)

Simula un cold start complet:

```bash
# 1. Aturar tot i esborrar volums (reset total)
docker-compose down -v

# 2. Netejar imatges locals per forçar un rebuild
docker-compose build --no-cache

# 3. Arrencar des de zero
docker-compose up -d

# 4. Esperar que tots els serveis estiguin healthy
echo "Esperant que els serveis estiguin sans..."
for i in $(seq 1 30); do
    HEALTHY=$(docker-compose ps | grep -c "healthy")
    TOTAL=$(docker-compose ps | grep -c "esportspulse")
    echo "  [$i/30] $HEALTHY/$TOTAL serveis healthy"
    if [ "$HEALTHY" -eq "$TOTAL" ]; then
        echo "Tots els serveis estan sans!"
        break
    fi
    sleep 10
done

# 5. Verificar cada servei
echo "--- Backend Java ---"
curl -s http://localhost:8080/actuator/health | jq '.status'

echo "--- Python Service ---"
curl -s http://localhost:8000/health | jq '.status'

echo "--- Streamlit ---"
curl -s -o /dev/null -w "%{http_code}" http://localhost:8501

echo "--- RabbitMQ Management ---"
curl -s -o /dev/null -w "%{http_code}" http://localhost:15672

echo "--- PostgreSQL ---"
docker exec esportspulse-postgres pg_isready -U esports && echo "OK"

echo "--- Redis ---"
docker exec esportspulse-redis redis-cli ping
```

### 2. Test funcional end-to-end (20 min)

Verifica que tots els fluxos funcionals operen:

```bash
# --- Test 1: Crear un campió (síncron) ---
echo "Test 1: Crear campió..."
RESPONSE=$(curl -s -w "\n%{http_code}" -X POST http://localhost:8080/api/champions \
  -H "Content-Type: application/json" \
  -d '{"name":"Vi","role":"JUNGLE","description":"Boxejadora amb guants gegants"}')
HTTP_CODE=$(echo "$RESPONSE" | tail -1)
echo "  HTTP: $HTTP_CODE (esperat: 201)"

# --- Test 2: Consultar campions (síncron amb cache) ---
echo "Test 2: Consultar campions..."
curl -s http://localhost:8080/api/champions | jq '.[0].name'
# Segona crida (cache hit)
curl -s http://localhost:8080/api/champions | jq '.[0].name'

# --- Test 3: Verificar event asíncron ---
echo "Test 3: Esperant processament asíncron (LLM via RabbitMQ)..."
sleep 15  # Donar temps al consumer
# Comprovar si el resum existeix a Redis
docker exec esportspulse-redis redis-cli KEYS "champion:summary:*"

# --- Test 4: Verificar mètriques ---
echo "Test 4: Mètriques..."
curl -s http://localhost:8080/actuator/metrics/http.server.requests | jq '.measurements'

# --- Test 5: Health checks ---
echo "Test 5: Health checks..."
curl -s http://localhost:8080/actuator/health | jq
curl -s http://localhost:8000/health | jq
```

### 3. Revisar el CI (15 min)

```bash
# Verificar que el CI està en verd
gh run list --limit 5

# Si hi ha alguna execució fallida, investigar
gh run view <RUN_ID> --log-failed
```

Comprova que el workflow `ci.yml` de dilluns funciona:
- Els tests Java passen
- Els tests Python passen
- El build Docker funciona
- El cache de dependències s'utilitza

### 4. Escriure el Runbook Operacional (30 min)

Crea `docs/runbook.md`:

```markdown
# Runbook Operacional — EsportsPulse

## Arrencar la Plataforma

### Prerequisits
- Docker i Docker Compose instal·lats
- Fitxer `.env` creat a partir de `.env.example`

### Arrencada
```bash
# Arrencar tots els serveis
docker-compose up -d

# Verificar que tot està sa (esperar 1-2 minuts)
docker-compose ps
# Tots els serveis han de mostrar "(healthy)"
```

## Verificar l'Estat

### Health Checks
```bash
# Backend Java
curl -s http://localhost:8080/actuator/health | jq

# Servei Python
curl -s http://localhost:8000/health | jq

# Estat general de tots els contenidors
docker-compose ps
```

### Dashboard
- Monitoring: http://localhost:8501 (pàgina de monitoring)
- RabbitMQ: http://localhost:15672 (user/pass a .env)

## Llegir Logs

### Logs d'un servei específic
```bash
# Últimes 100 línies del backend Java
docker-compose logs --tail 100 backend-java

# Seguir els logs en temps real (Ctrl+C per aturar)
docker-compose logs -f backend-java

# Logs de tots els serveis
docker-compose logs -f

# Filtrar per error
docker-compose logs backend-java 2>&1 | grep -i error
```

### Logs del consumer
```bash
# El consumer mostra: events rebuts, temps de processament, errors
docker-compose logs -f consumer-python
```

## Investigar Errors Comuns

### El backend Java no arrenca
1. Comprovar si PostgreSQL està sa: `docker-compose ps postgres`
2. Comprovar logs: `docker-compose logs backend-java | tail -50`
3. Errors habituals:
   - "Connection refused": PostgreSQL no està llest → esperar o reiniciar
   - "Flyway migration failed": error a un fitxer SQL → revisar la migració

### El consumer no processa events
1. Comprovar la cua a RabbitMQ: http://localhost:15672 → Queues
2. Si la cua `llm-processing` té missatges acumulats:
   - Comprovar logs del consumer: `docker-compose logs consumer-python`
   - Si el consumer ha caigut: `docker-compose restart consumer-python`
3. Si els missatges van al DLQ (`llm-processing.dlq`):
   - Inspeccionar el missatge a la UI de RabbitMQ
   - Probablement un error de format o el servei LLM caigut

### Cache no funciona
1. Comprovar Redis: `docker exec esportspulse-redis redis-cli ping`
2. Comprovar si hi ha claus: `docker exec esportspulse-redis redis-cli KEYS "*"`
3. Cache hit ratio al dashboard < 50%: revisar TTL o patrons d'accés

## Aturar la Plataforma

```bash
# Aturar tots els serveis (manté les dades)
docker-compose down

# Aturar i ESBORRAR totes les dades (reset total)
docker-compose down -v
```

## URLs dels Serveis

| Servei | URL | Notes |
|--------|-----|-------|
| Backend Java | http://localhost:8080 | API REST |
| Actuator | http://localhost:8080/actuator/health | Health check |
| Python Service | http://localhost:8000 | FastAPI |
| Streamlit | http://localhost:8501 | Frontend |
| RabbitMQ UI | http://localhost:15672 | Management |
| PostgreSQL | localhost:5432 | psql -h localhost -U esports |
| Redis | localhost:6379 | redis-cli |
| Qdrant | http://localhost:6333 | REST API |
```

### 5. Commit final i PR del Bloc 4 (15 min)

```bash
git add .
git commit -m "docs(runbook): add operational runbook and complete platform validation"

# Push i crear el PR final del Bloc 4
git push -u origin feature/week20-consolidation
```

Crea el PR amb un resum del Bloc 4 complet:

```
## Bloc 4: Infraestructura Avançada i Integració (S18-S20)

### Setmana 18 — SQL Avançat i PostgreSQL
- Migració d'H2 a PostgreSQL amb Flyway
- Esquema normalitzat amb FK i constraints
- Queries avançades: JOINs, window functions
- Índexs i EXPLAIN ANALYZE
- Cache-aside amb Redis

### Setmana 19 — Message Queues
- Disseny d'events asíncrons
- Productor Java amb Spring AMQP
- Consumer Python amb pika + LLM
- Retry, idempotència i circuit breaker
- Tests d'integració end-to-end

### Setmana 20 — Consolidació
- CI unificat amb caching
- Docker Compose complet (8 serveis)
- Health endpoints (Actuator + FastAPI)
- Dashboard de mètriques operacionals
- Runbook operacional
```

---

## Checklist de Lliurament

- [ ] Cold start funciona: `docker-compose down -v` + `docker-compose up` = tot sa
- [ ] Test funcional end-to-end: crear campió → event → LLM → Redis
- [ ] CI en verd (workflow unificat `ci.yml`)
- [ ] Tots els health checks passen
- [ ] Dashboard de mètriques mostra dades reals
- [ ] Runbook escrit a `docs/runbook.md`
- [ ] Tots els commits de S18-S20 fets
- [ ] PR creat amb resum del Bloc 4
- [ ] `docker-compose ps` mostra 8 serveis healthy
