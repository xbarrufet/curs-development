# Setmana 8 — Dimecres: Docker Compose: Orquestrar Múltiples Serveis

## Objectiu del Dia

Definir tota l'arquitectura del projecte en un sol fitxer `docker-compose.yml` i poder arrencar els quatre serveis (Java backend, Python service, PostgreSQL, Qdrant) amb una única comanda. Al final del dia, `docker-compose up` ha d'aixecar tot l'stack i els serveis han de comunicar-se entre ells per nom.

---

## Teoria

### El Problema: Massa Comandes Manuals

Ahir vam executar dos contenidors amb `docker run`. Cadascun necessitava flags:

```bash
# Backend Java
docker run -d -p 8080:8080 --name backend esportspulse-backend:latest

# Servei Python
docker run -d -p 5000:5000 --name ai-service esportspulse-ai:latest

# PostgreSQL (amb variables d'entorn i volum)
docker run -d -p 5432:5432 \
  --name postgres \
  -e POSTGRES_USER=esportspulse \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=esportspulse_db \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16-alpine

# Qdrant (base de dades vectorial per al servei d'IA)
docker run -d -p 6333:6333 \
  --name qdrant \
  -v qdrant_data:/qdrant/storage \
  qdrant/qdrant:latest
```

Quatre comandes, cadascuna amb diversos flags. I encara no hem configurat la xarxa perquè es comuniquin entre ells. Imagina haver de recordar tot això cada cop que vulguis arrencar el projecte. **Insostenible.**

### Docker Compose: Un Sol Fitxer, Tot Definit

Docker Compose és una eina que permet definir i executar aplicacions multi-contenidor. Tot es descriu en un fitxer YAML (`docker-compose.yml`), i amb una sola comanda aixeques o atures tot.

```yaml
# docker-compose.yml és la "planta" de l'edifici.
# Cada servei és una "habitació" amb la seva funció.
# Docker Compose construeix l'edifici sencer amb una comanda.
```

### Anatomia d'un docker-compose.yml

```yaml
# Cada bloc de primer nivell sota "services" defineix un contenidor.
services:

  # Nom del servei. Els altres contenidors el troben amb aquest nom.
  backend:
    # "build" diu a Compose que construeixi la imatge des d'un Dockerfile
    build:
      context: ./backend-java      # Carpeta on hi ha el Dockerfile
      dockerfile: Dockerfile        # Nom del Dockerfile (per defecte ja és "Dockerfile")

    # Mapeig de ports: HOST:CONTENIDOR
    # El port de l'esquerra és el del teu ordinador
    # El port de la dreta és el de dins del contenidor
    ports:
      - "8080:8080"

    # Variables d'entorn que rep el contenidor
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/esportspulse_db
      - SPRING_DATASOURCE_USERNAME=esportspulse
      - SPRING_DATASOURCE_PASSWORD=secret

    # depends_on: aquest servei s'arrencarà DESPRÉS dels serveis llistats.
    # Atenció: "després" vol dir que el contenidor ha arrencat,
    # NO que l'aplicació dins estigui llesta. Ho millorarem dijous.
    depends_on:
      - postgres

  ai-service:
    build:
      context: ./ai-python
    ports:
      - "5000:5000"
    environment:
      - QDRANT_HOST=qdrant          # El servei Python es connecta a Qdrant pel nom
      - QDRANT_PORT=6333
    depends_on:
      - qdrant

  # Serveis de tercers: no cal "build", usem "image" directament
  postgres:
    image: postgres:16-alpine       # Imatge oficial de PostgreSQL (versió Alpine, lleugera)
    ports:
      - "5432:5432"                 # Exposar port per si volem connectar-nos des del host
    environment:
      - POSTGRES_USER=esportspulse
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB=esportspulse_db
    volumes:
      - pgdata:/var/lib/postgresql/data   # Volum per persistir dades (ho veurem dijous)

  qdrant:
    image: qdrant/qdrant:latest     # Base de dades vectorial per a embeddings
    ports:
      - "6333:6333"                 # API REST de Qdrant
      - "6334:6334"                 # API gRPC de Qdrant
    volumes:
      - qdrant_data:/qdrant/storage

# Declaració de volums amb nom.
# Docker els gestiona automàticament. Les dades sobreviuen a docker-compose down.
volumes:
  pgdata:
  qdrant_data:
```

### Networking: Com es Comuniquen els Contenidors

Quan fas `docker-compose up`, Docker Compose crea automàticament una **xarxa virtual** (bridge network) per al teu projecte. Dins d'aquesta xarxa:

- Cada servei és accessible pel **nom del servei** com a hostname
- `localhost` dins d'un contenidor es refereix **al propi contenidor**, no al teu ordinador
- Els contenidors es resolen entre ells per DNS intern de Docker

```
┌─────────────────────────────────────────────────────┐
│              Xarxa Docker (bridge)                  │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────┐ │
│  │ backend  │  │ai-service│  │ postgres │  │qdr.│ │
│  │ :8080    │  │ :5000    │  │ :5432    │  │:6333│ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──┬─┘ │
│       │              │              │            │   │
│       └──────────────┴──────┬───────┴────────────┘   │
│                             │                        │
└─────────────────────────────┼────────────────────────┘
                              │
                    Ports exposats al host:
                    localhost:8080 → backend
                    localhost:5000 → ai-service
                    localhost:5432 → postgres
```

**Exemple pràctic:** El backend Java es connecta a PostgreSQL amb:
```
jdbc:postgresql://postgres:5432/esportspulse_db
                  ^^^^^^^^
                  Nom del servei, NO localhost!
```

Des del teu ordinador (fora de Docker) sí que uses `localhost:5432` perquè el port està mapejat.

> **Error habitual:** Configurar la connexió a la base de dades com `localhost:5432` dins del contenidor. Dins de Docker, `localhost` és el propi contenidor, que no té PostgreSQL. Has d'usar el nom del servei: `postgres`.

### Comandes Essencials de Docker Compose

```bash
# Arrencar tots els serveis (en segon pla)
# --build = reconstruir imatges si el Dockerfile o el codi han canviat
docker-compose up -d --build

# Arrencar tots els serveis (en primer pla, veient logs en directe)
docker-compose up --build

# Aturar i eliminar tots els contenidors (les dades dels volums es mantenen)
docker-compose down

# Aturar i eliminar tot, INCLOENT els volums (pèrdua de dades!)
docker-compose down -v

# Veure els logs de tots els serveis
docker-compose logs

# Veure els logs d'un servei concret, en temps real
docker-compose logs -f backend

# Veure l'estat dels serveis
docker-compose ps

# Reconstruir una imatge concreta sense cache
docker-compose build --no-cache backend

# Arrencar només un servei (i les seves dependències)
docker-compose up -d postgres
```

---

## Activitat

### Part 1: Crear el fitxer docker-compose.yml

**1.1.** A l'arrel del projecte `esportspulse-engine/`, crea el fitxer `docker-compose.yml`:

```yaml
# ===================================================================
# Docker Compose per al projecte EsportsPulse
# Defineix tots els serveis necessaris per al desenvolupament local
# ===================================================================

services:
  # --- Backend Java (Spring Boot) ---
  backend:
    build:
      context: ./backend-java
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      # Connexió a PostgreSQL: "postgres" és el nom del servei, no localhost
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/esportspulse_db
      - SPRING_DATASOURCE_USERNAME=esportspulse
      - SPRING_DATASOURCE_PASSWORD=secret
      # Perfil de Spring per a desenvolupament
      - SPRING_PROFILES_ACTIVE=dev
    depends_on:
      - postgres
    # Reiniciar automàticament si el contenidor cau
    restart: unless-stopped

  # --- Servei Python (IA / Embeddings) ---
  ai-service:
    build:
      context: ./ai-python
      dockerfile: Dockerfile
    ports:
      - "5000:5000"
    environment:
      # Connexió a Qdrant: "qdrant" és el nom del servei
      - QDRANT_HOST=qdrant
      - QDRANT_PORT=6333
      - LOG_LEVEL=INFO
    depends_on:
      - qdrant
    restart: unless-stopped

  # --- PostgreSQL (base de dades relacional) ---
  postgres:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_USER=esportspulse
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB=esportspulse_db
    volumes:
      # Volum amb nom per persistir les dades de PostgreSQL
      - pgdata:/var/lib/postgresql/data
    restart: unless-stopped

  # --- Qdrant (base de dades vectorial) ---
  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"    # API REST
      - "6334:6334"    # API gRPC
    volumes:
      # Volum amb nom per persistir els vectors
      - qdrant_data:/qdrant/storage
    restart: unless-stopped

# Declaració de volums
# Docker gestiona on s'emmagatzemen físicament les dades
volumes:
  pgdata:
  qdrant_data:
```

### Part 2: Arrencar i Verificar

**2.1. Arrenca tot l'stack:**

```bash
# Des de l'arrel del projecte (on hi ha docker-compose.yml)
cd esportspulse-engine

# Arrencar tots els serveis, reconstruint les imatges
docker-compose up -d --build

# Seguir l'arrencada en temps real
docker-compose logs -f
# Ctrl+C per sortir dels logs (els contenidors segueixen corrent)
```

**2.2. Comprova que tot funciona:**

```bash
# Veure l'estat de tots els serveis
docker-compose ps

# Hauries de veure els 4 serveis amb estat "Up":
# NAME              STATUS    PORTS
# backend           Up        0.0.0.0:8080->8080/tcp
# ai-service        Up        0.0.0.0:5000->5000/tcp
# postgres          Up        0.0.0.0:5432->5432/tcp
# qdrant            Up        0.0.0.0:6333->6333/tcp, 0.0.0.0:6334->6334/tcp

# Verificar el backend Java
curl http://localhost:8080/actuator/health

# Verificar el servei Python
curl http://localhost:5000/health

# Verificar PostgreSQL (des del host, perquè hem exposat el port)
# Necessites psql instal·lat, o pots fer-ho amb docker exec (ho veurem divendres)
docker-compose exec postgres psql -U esportspulse -d esportspulse_db -c "SELECT 1;"

# Verificar Qdrant (API REST)
curl http://localhost:6333/healthz
```

### Part 3: Experimentar amb el Cicle de Vida

**3.1. Aturar i reprendre:**

```bash
# Aturar tot (les dades dels volums es mantenen)
docker-compose down

# Verificar que no hi ha contenidors
docker-compose ps

# Tornar a arrencar (no cal --build si no has canviat codi)
docker-compose up -d

# Les dades de PostgreSQL segueixen allà gràcies als volums
docker-compose exec postgres psql -U esportspulse -d esportspulse_db -c "SELECT 1;"
```

**3.2. Veure logs d'un servei concret:**

```bash
# Només els logs del backend, en temps real
docker-compose logs -f backend

# Els últims 50 logs del servei Python
docker-compose logs --tail=50 ai-service
```

**3.3. Reconstruir un servei després de canviar codi:**

```bash
# Si canvies codi al backend Java, reconstrueix només aquell servei
docker-compose up -d --build backend

# Docker Compose detecta que la resta de serveis no han canviat
# i no els reinicia (intel·ligent!)
```

### Part 4: Entendre la Xarxa

**4.1. Comprova la resolució de noms:**

```bash
# Entra dins del contenidor del backend
docker-compose exec backend sh

# Des de dins del contenidor, resol el nom "postgres"
# (pot ser que necessitis instal·lar eines de xarxa)
nslookup postgres
# Hauria de mostrar una IP interna de Docker (ex: 172.18.0.3)

# Prova la connexió a PostgreSQL des de dins del backend
# (si tens les eines instal·lades)
ping -c 2 postgres

# Surt del contenidor
exit
```

> **Nota important sobre `depends_on`:** Per defecte, `depends_on` només espera que el **contenidor** s'hagi iniciat, no que l'**aplicació** dins estigui llesta. PostgreSQL pot trigar uns segons a arrencar, i el backend podria intentar connectar-s'hi abans que estigui llest. Dijous veurem com solucionar-ho amb **health checks**.

---

## Checklist de Lliurament

- [ ] El fitxer `docker-compose.yml` existeix a l'arrel del projecte amb els 4 serveis definits
- [ ] `docker-compose up -d --build` arrenca tots els serveis sense errors
- [ ] `docker-compose ps` mostra els 4 serveis amb estat "Up"
- [ ] El backend Java respon a `http://localhost:8080/actuator/health`
- [ ] El servei Python respon a `http://localhost:5000/health`
- [ ] PostgreSQL accepta connexions a `localhost:5432`
- [ ] Qdrant respon a `http://localhost:6333/healthz`
- [ ] `docker-compose down` i `docker-compose up -d` funcionen correctament (les dades de PostgreSQL es mantenen)
- [ ] Tots els fitxers estan commitejats: `feat(docker): add docker-compose with all services`
