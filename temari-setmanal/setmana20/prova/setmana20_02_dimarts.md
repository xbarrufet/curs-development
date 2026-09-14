# Setmana 20 — Dimarts: Dockeritzar Tot: docker-compose up = Tota la Plataforma

## Objectiu del Dia

Aconseguir que `docker-compose up` arrenqui tota la plataforma EsportsPulse amb una sola comanda: backend Java, servei Python, Streamlit, PostgreSQL, Redis, Qdrant i RabbitMQ. Al final del dia, qualsevol persona pot clonar el repo, fer `docker-compose up` i tenir l'aplicació completa funcionant.

---

## Teoria

### L'Objectiu Final de Docker Compose

```bash
git clone https://github.com/user/esportspulse-engine.git
cd esportspulse-engine
docker-compose up
# → Tot funciona. Zero configuració manual.
```

Això és el que busquem. Fins ara hem anat afegint serveis a Docker Compose incrementalment. Avui unificar-ho tot.

### Dockerfile per al Backend Java

Si encara no tens un Dockerfile per al backend Java, o és bàsic:

```dockerfile
# Dockerfile multi-stage: primera etapa compila, segona executa
# Això redueix la mida de la imatge final (no inclou el JDK de compilació)

# --- Etapa 1: Build ---
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
# Copiar primer el pom.xml per aprofitar el cache de Docker
# (si el pom.xml no canvia, Maven no re-descarrega dependències)
COPY pom.xml .
RUN mvn dependency:go-offline -B
# Ara copiar el codi font i compilar
COPY src ./src
RUN mvn package -DskipTests -B

# --- Etapa 2: Runtime ---
FROM eclipse-temurin:21-jre
WORKDIR /app
# Copiar NOMÉS el JAR compilat de l'etapa anterior
COPY --from=build /app/target/*.jar app.jar
# Port que exposa l'aplicació
EXPOSE 8080
# Comanda d'execució amb configuració de memòria
ENTRYPOINT ["java", "-Xmx512m", "-jar", "app.jar"]
```

### Dockerfile per al Servei Python

```dockerfile
FROM python:3.12-slim
WORKDIR /app
# Copiar requirements primer (cache de Docker)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
# Copiar el codi font
COPY src/ ./src/
# Port de FastAPI
EXPOSE 8000
# Comanda d'execució
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Gestió de Secrets: .env i Docker Secrets

**MAI hardcodejar secrets** (contrasenyes, API keys) al codi o al `docker-compose.yml`.

```
# .env — Fitxer de variables d'entorn (NO commitar a Git!)
POSTGRES_PASSWORD=esports_pwd_segur
RABBITMQ_PASSWORD=rabbit_pwd_segur
REDIS_PASSWORD=redis_pwd_segur
LLM_API_KEY=sk-...
```

```yaml
# docker-compose.yml — Referència a les variables de .env
services:
  postgres:
    environment:
      # ${VARIABLE} llegeix del .env automàticament
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

**Regles:**
1. `.env` ha d'estar al `.gitignore` (ja ho està des de S1)
2. Crea un `.env.example` amb valors de placeholder que SÍ es commita
3. Al README, documenta que cal copiar `.env.example` a `.env`

### Ordre d'Arrencada i Healthchecks

Els serveis tenen dependències: el backend Java necessita PostgreSQL i RabbitMQ. Docker Compose pot gestionar l'ordre:

```yaml
services:
  backend-java:
    depends_on:
      postgres:
        condition: service_healthy    # Espera que PostgreSQL estigui sa
      rabbitmq:
        condition: service_healthy    # Espera que RabbitMQ estigui sa
```

> **Lectura recomanada (opcional, no bloquejant):**
> - [Docker Compose Best Practices](https://docs.docker.com/compose/production/)
> - [Docker Multi-stage Builds](https://docs.docker.com/build/building/multi-stage/)

---

## Activitat

### 1. Inventariar tots els serveis necessaris (10 min)

Llista tots els components d'EsportsPulse:

| Servei | Imatge/Build | Port | Depèn de |
|--------|-------------|------|----------|
| PostgreSQL | postgres:16 | 5432 | - |
| Redis | redis:7-alpine | 6379 | - |
| RabbitMQ | rabbitmq:3-management | 5672, 15672 | - |
| Qdrant | qdrant/qdrant:latest | 6333, 6334 | - |
| Backend Java | Build local | 8080 | PostgreSQL, RabbitMQ, Redis |
| Servei Python (FastAPI) | Build local | 8000 | Redis, Qdrant |
| Consumer Python | Build local | - | RabbitMQ, Redis |
| Streamlit | Build local | 8501 | Backend Java, Servei Python |

### 2. Crear/actualitzar els Dockerfiles (30 min)

Crea o actualitza el Dockerfile del backend Java (`backend-java/Dockerfile`):

```dockerfile
# Multi-stage build per reduir la mida de la imatge final
# Etapa 1: compilar el projecte amb Maven
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app

# Copiar pom.xml primer per aprofitar la cache de Docker
# (les dependències només es re-descarreguen si canvia el pom.xml)
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copiar el codi font i compilar (sense tests, es fan al CI)
COPY src ./src
COPY src/main/resources ./src/main/resources
RUN mvn package -DskipTests -B

# Etapa 2: imatge lleugera amb només el JRE (no el JDK complet)
FROM eclipse-temurin:21-jre
WORKDIR /app
# Copiar el JAR compilat de l'etapa de build
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
# Configurar memòria i perfil Spring
ENTRYPOINT ["java", "-Xmx512m", "-jar", "app.jar"]
```

Crea el Dockerfile del servei Python (`ai-python/Dockerfile`):

```dockerfile
# Imatge base lleugera de Python
FROM python:3.12-slim
WORKDIR /app

# Instal·lar dependències primer (cache de Docker)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copiar el codi font
COPY src/ ./src/

# Port per defecte de FastAPI
EXPOSE 8000

# Arrencar amb uvicorn (servidor ASGI per FastAPI)
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Crea el Dockerfile del consumer (`ai-python/Dockerfile.consumer`):

```dockerfile
# Imatge per al consumer de RabbitMQ (sense servidor web)
FROM python:3.12-slim
WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ ./src/

# El consumer no exposa cap port — és un procés en segon pla
CMD ["python", "-m", "src.consumers.champion_consumer"]
```

Crea el Dockerfile de Streamlit (`ai-python/Dockerfile.streamlit`):

```dockerfile
FROM python:3.12-slim
WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ ./src/

EXPOSE 8501

# Arrencar Streamlit sense obrir el navegador automàticament
CMD ["streamlit", "run", "src/streamlit_app.py", "--server.port=8501", "--server.headless=true"]
```

### 3. Escriure el docker-compose.yml complet (40 min)

```yaml
# docker-compose.yml — Plataforma EsportsPulse completa
# Ús: docker-compose up (arrencar tot) / docker-compose down (aturar tot)

version: '3.8'

services:

  # ============================================================
  # INFRAESTRUCTURA (serveis externs)
  # ============================================================

  # Base de dades relacional principal
  postgres:
    image: postgres:16
    container_name: esportspulse-postgres
    environment:
      POSTGRES_DB: esportspulse
      POSTGRES_USER: ${POSTGRES_USER:-esports}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-esports_pwd}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-esports}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Cache en memòria
  redis:
    image: redis:7-alpine
    container_name: esportspulse-redis
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

  # Message broker per comunicació asíncrona
  rabbitmq:
    image: rabbitmq:3-management
    container_name: esportspulse-rabbitmq
    environment:
      RABBITMQ_DEFAULT_USER: ${RABBITMQ_USER:-esports}
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASSWORD:-esports_pwd}
    ports:
      - "5672:5672"      # AMQP
      - "15672:15672"    # Management UI
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_port_connectivity"]
      interval: 15s
      timeout: 10s
      retries: 5

  # Base de dades vectorial per a knowledge retrieval
  qdrant:
    image: qdrant/qdrant:latest
    container_name: esportspulse-qdrant
    ports:
      - "6333:6333"      # REST API
      - "6334:6334"      # gRPC
    volumes:
      - qdrant_data:/qdrant/storage

  # ============================================================
  # APLICACIÓ (serveis propis)
  # ============================================================

  # Backend Java (Spring Boot REST API)
  backend-java:
    build:
      context: ./backend-java
      dockerfile: Dockerfile
    container_name: esportspulse-backend
    environment:
      # Connexió a PostgreSQL (usa el nom del servei Docker com a host)
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/esportspulse
      SPRING_DATASOURCE_USERNAME: ${POSTGRES_USER:-esports}
      SPRING_DATASOURCE_PASSWORD: ${POSTGRES_PASSWORD:-esports_pwd}
      # Connexió a Redis
      SPRING_DATA_REDIS_HOST: redis
      # Connexió a RabbitMQ
      SPRING_RABBITMQ_HOST: rabbitmq
      SPRING_RABBITMQ_USERNAME: ${RABBITMQ_USER:-esports}
      SPRING_RABBITMQ_PASSWORD: ${RABBITMQ_PASSWORD:-esports_pwd}
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy

  # Servei Python (FastAPI — LLM, knowledge retrieval)
  service-python:
    build:
      context: ./ai-python
      dockerfile: Dockerfile
    container_name: esportspulse-python
    environment:
      REDIS_HOST: redis
      QDRANT_HOST: qdrant
      LLM_API_KEY: ${LLM_API_KEY:-}
    ports:
      - "8000:8000"
    depends_on:
      - redis
      - qdrant

  # Consumer Python (processa events de RabbitMQ)
  consumer-python:
    build:
      context: ./ai-python
      dockerfile: Dockerfile.consumer
    container_name: esportspulse-consumer
    environment:
      RABBITMQ_HOST: rabbitmq
      RABBITMQ_PORT: 5672
      RABBITMQ_USER: ${RABBITMQ_USER:-esports}
      RABBITMQ_PASS: ${RABBITMQ_PASSWORD:-esports_pwd}
      REDIS_HOST: redis
      LLM_SERVICE_URL: http://service-python:8000
    depends_on:
      rabbitmq:
        condition: service_healthy
      redis:
        condition: service_healthy
      service-python:
        condition: service_started
    # Reiniciar si falla (ex: RabbitMQ encara no llest)
    restart: on-failure

  # Dashboard Streamlit (frontend)
  streamlit:
    build:
      context: ./ai-python
      dockerfile: Dockerfile.streamlit
    container_name: esportspulse-streamlit
    environment:
      BACKEND_URL: http://backend-java:8080
      PYTHON_SERVICE_URL: http://service-python:8000
    ports:
      - "8501:8501"
    depends_on:
      - backend-java
      - service-python

# Volums persistents (les dades sobreviuen a docker-compose down)
volumes:
  postgres_data:
  qdrant_data:
```

### 4. Crear el fitxer .env.example (10 min)

```bash
# .env.example — Copiar a .env i omplir amb valors reals
# cp .env.example .env

# PostgreSQL
POSTGRES_USER=esports
POSTGRES_PASSWORD=canvia_aquesta_contrasenya

# RabbitMQ
RABBITMQ_USER=esports
RABBITMQ_PASSWORD=canvia_aquesta_contrasenya

# LLM API Key (Anthropic, OpenAI, etc.)
LLM_API_KEY=sk-...

# Entorn (development/production)
ENVIRONMENT=development
```

Verifica que `.env` està al `.gitignore`:
```bash
grep ".env" .gitignore
```

### 5. Testejar l'arrencada completa (15 min)

```bash
# Aturar tot el que estigui corrent
docker-compose down -v

# Arrencar tota la plataforma des de zero
docker-compose up --build

# En una altra terminal, verificar que tot funciona:
# Backend Java
curl -s http://localhost:8080/api/champions | jq

# Servei Python
curl -s http://localhost:8000/health | jq

# Streamlit (obre al navegador)
open http://localhost:8501

# RabbitMQ Management
open http://localhost:15672

# Verificar l'estat de tots els contenidors
docker-compose ps
```

### 6. Commit (5 min)

```bash
git add .
git commit -m "feat(docker): complete docker-compose with all services and healthchecks"
```

---

## Checklist de Lliurament

- [ ] Dockerfiles creats per: backend Java, servei Python, consumer, Streamlit
- [ ] `docker-compose.yml` amb tots 8 serveis
- [ ] Healthchecks configurats per PostgreSQL, Redis i RabbitMQ
- [ ] `depends_on` amb `condition: service_healthy` per a l'ordre d'arrencada
- [ ] `.env.example` creat (sense secrets reals)
- [ ] `.env` al `.gitignore`
- [ ] `docker-compose up` arrencar tota la plataforma sense errors
- [ ] Tots els serveis responen correctament
- [ ] Commit fet
