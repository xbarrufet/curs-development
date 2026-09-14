# Setmana 8 — Dijous: Volums, Health Checks i Variables d'Entorn

## Objectiu del Dia

Fer que l'stack de Docker Compose sigui robust: les dades de PostgreSQL sobreviuen a reinicis, els serveis no arranquen fins que les seves dependències estiguin realment llestes, i tota la configuració sensible està externalitzada en variables d'entorn. Al final del dia, `docker-compose up` ha d'arrencar l'stack de forma fiable i ordenada.

---

## Teoria

### Volums: Per Què les Dades Desapareixen

Un contenidor Docker és **efímer**: quan el destrueixes (`docker rm`), tot el que hi havia dins desapareix. Això inclou les dades de PostgreSQL. Si fas `docker-compose down` i després `docker-compose up`, la base de dades tornarà a estar buida.

```
Sense volum:
┌──────────────────┐
│   Contenidor     │
│   PostgreSQL     │  ← docker rm → 💀 Dades perdudes!
│   /var/lib/      │
│   postgresql/data│
└──────────────────┘

Amb volum:
┌──────────────────┐       ┌───────────────┐
│   Contenidor     │       │  Volum Docker  │
│   PostgreSQL     │──────▶│   "pgdata"     │  ← Dades segures!
│   (efímer)       │       │  (persistent)  │
└──────────────────┘       └───────────────┘
```

### Tipus de Volums

Hi ha dues maneres principals de persistir dades:

**1. Volums amb nom (Named Volumes):**
Docker gestiona on s'emmagatzemen al host. Ideals per a bases de dades.

```yaml
services:
  postgres:
    image: postgres:16-alpine
    volumes:
      # Sintaxi: nom_volum:ruta_dins_contenidor
      # Docker decideix on guarda "pgdata" al sistema host
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:    # Declarar el volum a nivell superior
```

```bash
# Veure els volums creats per Docker
docker volume ls

# Inspeccionar un volum (veure on s'emmagatzema al host)
docker volume inspect esportspulse-engine_pgdata
```

**2. Bind Mounts:**
Muntes un directori concret del host dins del contenidor. Útils per al desenvolupament (veure canvis en temps real).

```yaml
services:
  ai-service:
    build: ./ai-python
    volumes:
      # Sintaxi: ./ruta_host:ruta_contenidor
      # El codi del host es munta dins del contenidor
      # Qualsevol canvi al host es reflecteix instantàniament
      - ./ai-python/src:/app/src
```

> **Quan usar cada un?**
> - **Volums amb nom** per a dades de bases de dades (PostgreSQL, Qdrant) i dades que no necessites editar directament.
> - **Bind mounts** per al codi durant el desenvolupament (hot-reload sense reconstruir la imatge).

### Variables d'Entorn: Configuració Flexible

Les variables d'entorn permeten configurar l'aplicació **sense modificar el codi ni la imatge**. La mateixa imatge pot executar-se en desenvolupament, staging o producció canviant només les variables.

Recordes les variables d'entorn del terminal (S3)? En Docker funcionen exactament igual, però les passem de tres maneres:

**1. Directament al `docker-compose.yml`:**

```yaml
services:
  backend:
    environment:
      # Llista de variables (format amb guió)
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/esportspulse_db
      - SPRING_DATASOURCE_PASSWORD=secret
```

**2. Amb un fitxer `.env`:**

Crea un fitxer `.env` a l'arrel del projecte (al costat de `docker-compose.yml`):

```env
# .env — Variables d'entorn per a Docker Compose
# ATENCIÓ: NO PUGIS AQUEST FITXER A GIT (afegir-lo a .gitignore)

# PostgreSQL
POSTGRES_USER=esportspulse
POSTGRES_PASSWORD=super_secret_dev_password
POSTGRES_DB=esportspulse_db

# Backend Java
SPRING_PROFILES_ACTIVE=dev

# Servei Python
LOG_LEVEL=DEBUG
QDRANT_HOST=qdrant
```

I referencia-les al `docker-compose.yml`:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      # ${VARIABLE} agafa el valor del fitxer .env
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
```

**3. Amb `env_file`:**

```yaml
services:
  backend:
    # Carrega TOTES les variables del fitxer especificat
    env_file:
      - .env
```

> **Connexió amb S5 (CI/CD):** En producció, les variables sensibles (passwords, API keys) es guarden en un gestor de secrets (GitHub Secrets, AWS Secrets Manager, etc.) i s'injecten com a variables d'entorn. El patró que aprenem avui amb `.env` és el mateix: l'aplicació llegeix la configuració de l'entorn, mai del codi.

**Crea un fitxer `.env.example`** amb valors d'exemple (SENSE secrets reals) i puja'l a Git. Serveix de documentació per a qui cloni el projecte:

```env
# .env.example — Copia aquest fitxer a .env i omple els valors reals
POSTGRES_USER=esportspulse
POSTGRES_PASSWORD=change_me
POSTGRES_DB=esportspulse_db
SPRING_PROFILES_ACTIVE=dev
LOG_LEVEL=INFO
QDRANT_HOST=qdrant
```

### Health Checks: Saber si un Servei Està Realment Llest

Dimecres vam veure que `depends_on` només espera que el **contenidor** arrenqui, no que l'**aplicació** estigui llesta. Això causa errors:

```
backend    | Connection refused: postgres:5432
backend    | Retrying in 5 seconds...
```

PostgreSQL pot trigar 5-10 segons a estar llest. El backend intenta connectar-se immediatament i falla.

**Solució: Health checks.** Definim una comanda que Docker executa periòdicament per verificar si el servei funciona. I amb `depends_on: condition: service_healthy`, el servei dependent espera fins que la dependència estigui sana.

**Health check al Dockerfile:**

```dockerfile
# Dins del Dockerfile de PostgreSQL (o com a override al compose)
HEALTHCHECK --interval=10s --timeout=5s --retries=3 \
  CMD pg_isready -U esportspulse -d esportspulse_db || exit 1
# --interval: cada quant comprova (10 segons)
# --timeout: temps màxim per a la comprovació (5 segons)
# --retries: quantes vegades ha de fallar abans de marcar-lo "unhealthy" (3)
# pg_isready: comanda pròpia de PostgreSQL per comprovar si accepta connexions
```

**Health check al `docker-compose.yml`:**

```yaml
services:
  postgres:
    image: postgres:16-alpine
    healthcheck:
      # pg_isready: comanda pròpia de PostgreSQL
      test: ["CMD-SHELL", "pg_isready -U esportspulse -d esportspulse_db"]
      interval: 10s       # Comprova cada 10 segons
      timeout: 5s         # Si no respon en 5 segons, falla
      retries: 3          # Després de 3 fallades, estat "unhealthy"
      start_period: 30s   # Dona 30 segons d'arrencada abans de començar a comprovar

  backend:
    depends_on:
      postgres:
        condition: service_healthy   # Espera fins que PostgreSQL estigui "healthy"
```

**Estats d'un contenidor amb health check:**

```
starting → healthy → (si falla 3 cops) → unhealthy
                ↑                             │
                └─────────────────────────────┘
                     (si torna a funcionar)
```

> **Connexió amb producció:** En un entorn real, els orquestradors (Kubernetes, ECS) utilitzen health checks per reiniciar automàticament serveis que fallen. El que aprenem aquí és el mateix patró, però a escala local.

---

## Activitat

### Part 1: Externalitzar Configuració amb .env

**1.1. Crea el fitxer `.env` a l'arrel del projecte:**

```env
# ===================================================================
# Variables d'entorn per al desenvolupament local
# NO PUGIS AQUEST FITXER A GIT — conté secrets
# ===================================================================

# PostgreSQL
POSTGRES_USER=esportspulse
POSTGRES_PASSWORD=dev_secret_2024
POSTGRES_DB=esportspulse_db

# Backend Java
SPRING_PROFILES_ACTIVE=dev
JAVA_OPTS=-Xmx512m

# Servei Python
LOG_LEVEL=DEBUG
QDRANT_HOST=qdrant
QDRANT_PORT=6333
```

**1.2. Crea el fitxer `.env.example`:**

```env
# Copia aquest fitxer a .env i omple els valors
POSTGRES_USER=esportspulse
POSTGRES_PASSWORD=change_me
POSTGRES_DB=esportspulse_db
SPRING_PROFILES_ACTIVE=dev
JAVA_OPTS=-Xmx512m
LOG_LEVEL=INFO
QDRANT_HOST=qdrant
QDRANT_PORT=6333
```

**1.3. Afegeix `.env` al `.gitignore`:**

```gitignore
# Secrets locals — MAI pujar a Git
.env
```

### Part 2: Afegir Health Checks a Tots els Serveis

**2.1. Actualitza el `docker-compose.yml` complet:**

```yaml
# ===================================================================
# Docker Compose amb volums, health checks i variables d'entorn
# ===================================================================

services:
  # --- Backend Java ---
  backend:
    build:
      context: ./backend-java
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/${POSTGRES_DB}
      - SPRING_DATASOURCE_USERNAME=${POSTGRES_USER}
      - SPRING_DATASOURCE_PASSWORD=${POSTGRES_PASSWORD}
      - SPRING_PROFILES_ACTIVE=${SPRING_PROFILES_ACTIVE}
      - JAVA_OPTS=${JAVA_OPTS}
    depends_on:
      postgres:
        # El backend NO arrenca fins que PostgreSQL estigui "healthy"
        condition: service_healthy
    healthcheck:
      # Spring Boot Actuator exposa /actuator/health
      test: ["CMD-SHELL", "curl -f http://localhost:8080/actuator/health || exit 1"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 60s    # Spring Boot pot trigar a arrencar
    restart: unless-stopped

  # --- Servei Python ---
  ai-service:
    build:
      context: ./ai-python
      dockerfile: Dockerfile
    ports:
      - "5000:5000"
    environment:
      - QDRANT_HOST=${QDRANT_HOST}
      - QDRANT_PORT=${QDRANT_PORT}
      - LOG_LEVEL=${LOG_LEVEL}
    depends_on:
      qdrant:
        condition: service_healthy
    healthcheck:
      # El servei Python exposa /health
      test: ["CMD-SHELL", "curl -f http://localhost:5000/health || exit 1"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 30s
    restart: unless-stopped

  # --- PostgreSQL ---
  postgres:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      # Volum amb nom: les dades sobreviuen a docker-compose down
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      # pg_isready: comanda nativa de PostgreSQL per comprovar l'estat
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s
    restart: unless-stopped

  # --- Qdrant ---
  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"
      - "6334:6334"
    volumes:
      # Volum amb nom per persistir els vectors
      - qdrant_data:/qdrant/storage
    healthcheck:
      # Qdrant exposa /healthz per a comprovacions d'estat
      test: ["CMD-SHELL", "curl -f http://localhost:6333/healthz || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 20s
    restart: unless-stopped

volumes:
  pgdata:
  qdrant_data:
```

### Part 3: Verificar que Tot Funciona

**3.1. Arrenca l'stack i observa l'ordre d'arrencada:**

```bash
# Arrencar en primer pla per veure l'ordre
docker-compose up --build

# Hauries de veure:
# 1. postgres i qdrant arranquen primer
# 2. Docker espera que passin els health checks
# 3. backend i ai-service arranquen quan les dependències estan healthy
```

**3.2. Comprova els health checks:**

```bash
# En un altre terminal, veure l'estat amb health checks
docker-compose ps

# Hauries de veure:
# NAME         STATUS                  PORTS
# postgres     Up (healthy)            0.0.0.0:5432->5432/tcp
# qdrant       Up (healthy)            0.0.0.0:6333->6333/tcp
# backend      Up (healthy)            0.0.0.0:8080->8080/tcp
# ai-service   Up (healthy)            0.0.0.0:5000->5000/tcp

# Inspeccionar el health check d'un servei concret
docker inspect --format='{{json .State.Health}}' esportspulse-engine-postgres-1 | python3 -m json.tool
```

**3.3. Verificar la persistència de dades:**

```bash
# Crear una taula de prova a PostgreSQL
docker-compose exec postgres psql -U esportspulse -d esportspulse_db -c "
CREATE TABLE IF NOT EXISTS test_persistencia (
    id SERIAL PRIMARY KEY,
    missatge TEXT NOT NULL,
    creat_a TIMESTAMP DEFAULT NOW()
);
INSERT INTO test_persistencia (missatge) VALUES ('Dades que sobreviuen!');
SELECT * FROM test_persistencia;
"

# Aturar i eliminar tots els contenidors (però NO els volums)
docker-compose down

# Tornar a arrencar
docker-compose up -d

# Verificar que les dades segueixen allà
docker-compose exec postgres psql -U esportspulse -d esportspulse_db -c "
SELECT * FROM test_persistencia;
"
# Hauries de veure la fila "Dades que sobreviuen!"

# ARA prova amb -v (elimina volums) — les dades es perden!
docker-compose down -v
docker-compose up -d
docker-compose exec postgres psql -U esportspulse -d esportspulse_db -c "
SELECT * FROM test_persistencia;
"
# ERROR: relation "test_persistencia" does not exist
# Les dades han desaparegut perquè hem eliminat el volum!
```

> **Lliçó important:** `docker-compose down` conserva els volums. `docker-compose down -v` els elimina. En desenvolupament, `-v` és útil per començar de zero. En producció, MAI facis `-v` sense una còpia de seguretat.

---

## Checklist de Lliurament

- [ ] El fitxer `.env` existeix amb totes les variables i **NO** està a Git
- [ ] El fitxer `.env.example` existeix i **SÍ** està a Git (sense secrets reals)
- [ ] `.gitignore` inclou `.env`
- [ ] Els 4 serveis del `docker-compose.yml` tenen `healthcheck` definit
- [ ] `depends_on` amb `condition: service_healthy` per a backend i ai-service
- [ ] PostgreSQL i Qdrant tenen volums amb nom declarats
- [ ] `docker-compose up` arrenca els serveis en l'ordre correcte (DB primer, apps després)
- [ ] `docker-compose ps` mostra tots els serveis com a "healthy"
- [ ] Les dades de PostgreSQL sobreviuen a `docker-compose down` (sense `-v`)
- [ ] Tots els canvis estan commitejats: `feat(docker): add health checks, volumes and env configuration`
