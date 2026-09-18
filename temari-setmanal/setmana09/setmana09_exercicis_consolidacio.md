# Setmana 9 — Exercicis de Consolidació

Aquests exercicis repassen els conceptes clau de la setmana. No cal lliurar-los — són per verificar que has entès la teoria i la pràctica abans de passar a la setmana 10. Intenta resoldre'ls sense mirar els apunts; si et quedes encallat, revisa el dia corresponent.

---

## Bloc 1: Conceptes de Docker (Dilluns)

**Exercici 1.1 — Conceptes Fonamentals**

Respon sense consultar apunts:

1. Quina diferència hi ha entre una **imatge** i un **contenidor**? Dona l'analogia amb POO.
2. Quina diferència hi ha entre un **contenidor** i una **màquina virtual**? Quins recursos comparteix un contenidor amb el host?
3. Què fa el **Docker Daemon**? Què passa si no està corrent?
4. Què és **Docker Hub**? Amb quina eina del teu dia a dia ho compararies?

**Exercici 1.2 — Comandes Bàsiques**

Escriu la comanda Docker per a cada acció (sense executar-les):

1. Descarregar la imatge de MongoDB versió 7.
2. Crear i executar un contenidor de Redis en segon pla amb nom `cache`.
3. Veure els logs en temps real del contenidor `cache`.
4. Llistar tots els contenidors (inclosos els aturats).
5. Aturar i eliminar el contenidor `cache` en dues comandes.
6. Executar una shell interactiva dins d'un contenidor `pg16`.

**Exercici 1.3 — Ports i Variables d'Entorn**

Donat:

```bash
docker run -d \
  --name mydb \
  -p 3307:3306 \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=esportspulse \
  mysql:8
```

1. Quin port has d'usar des del teu ordinador per connectar-te a MySQL? I des de dins del contenidor?
2. Per què el port del host (3307) és diferent del port del contenidor (3306)?
3. Què passaria si intentes executar un segon contenidor MySQL amb `-p 3307:3306`?
4. Les variables `MYSQL_ROOT_PASSWORD` i `MYSQL_DATABASE` existeixen al teu sistema? On existeixen?

**Exercici 1.4 — Docker a Windows**

1. Per què Docker a Windows necessita WSL 2 o Hyper-V?
2. Quin kernel fan servir els contenidors a Windows: el de Windows o un de Linux?
3. Un company et diu "docker: command not found" a Windows. Quines 3 coses comprovaries?

---

## Bloc 2: Dockerfile (Dimarts)

**Exercici 2.1 — Llegeix el Dockerfile**

```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY target/esportspulse-0.1.0.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

1. Quina imatge base fa servir? Per què `-jre-alpine` i no `-jdk`?
2. Què fa `WORKDIR /app`?
3. `EXPOSE 8080` obre el port al host? Per què sí o per què no?
4. Quina diferència hi ha entre `ENTRYPOINT` i `CMD`?
5. Si fas `docker build .` sense `-t`, com s'anomena la imatge resultant?

**Exercici 2.2 — Detecta els Errors**

Troba tots els problemes d'aquest Dockerfile:

```dockerfile
FROM eclipse-temurin:21-jdk
COPY . /app
WORKDIR /app
RUN mvn clean package
COPY .env /app/.env
EXPOSE 8080
CMD java -jar target/app.jar
```

Pista: hi ha almenys 4 problemes (seguretat, mida, secrets, bones pràctiques).

**Exercici 2.3 — Multi-stage Build**

Escriu un Dockerfile multi-stage per a una aplicació Java que:

1. **Stage 1 (build):** Usa `eclipse-temurin:21-jdk` per compilar amb Maven.
2. **Stage 2 (runtime):** Usa `eclipse-temurin:21-jre-alpine` i només copia el `.jar` resultant.

Per què la imatge final és molt més petita? Aproximadament, quina mida tindria cada stage?

**Exercici 2.4 — .dockerignore**

Escriu un `.dockerignore` per a un projecte que té:
- Codi Java (`src/`, `pom.xml`)
- Codi Python (`ai-python/`)
- Directori `target/` amb compilats
- `.git/` amb historial
- `node_modules/` d'una eina de frontend
- `.env` amb secrets
- `*.md` de documentació

Quins fitxers/directoris has d'excloure i per què?

---

## Bloc 3: Docker Compose (Dimecres)

**Exercici 3.1 — Llegeix el docker-compose.yml**

```yaml
services:
  backend:
    build: ./backend-java
    ports:
      - "8080:8080"
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/esportspulse
      - SPRING_DATASOURCE_PASSWORD=secret
    depends_on:
      - db

  db:
    image: postgres:16
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB=esportspulse
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

1. Quants serveis hi ha? Quins són?
2. Per què `backend` usa `build:` i `db` usa `image:`?
3. A la URL JDBC, per què diu `db:5432` i no `localhost:5432`?
4. Què passaria si eliminessis la secció `volumes:`? Què passaria amb les dades quan facis `docker-compose down`?
5. Què garanteix `depends_on: - db`? Garanteix que PostgreSQL està llest per rebre connexions?

**Exercici 3.2 — Completa el Compose**

Al docker-compose.yml anterior, afegeix:

1. Un servei `redis` (imatge `redis:7`, port 6379).
2. Un servei `ai-python` que es construeixi des de `./ai-python`, amb port 8000 i variable `REDIS_URL=redis://redis:6379`.
3. `ai-python` ha de dependre de `redis` i `backend`.

**Exercici 3.3 — Comandes de Compose**

Escriu la comanda per a cada acció:

1. Arrencar tots els serveis en segon pla.
2. Veure els logs de tots els serveis en temps real.
3. Veure només els logs del servei `backend`.
4. Aturar tot sense eliminar els volums.
5. Aturar tot i eliminar volums (reset complet).
6. Reconstruir les imatges i arrencar (després de canviar codi).
7. Veure l'estat de tots els serveis.

**Exercici 3.4 — Xarxa Interna**

Donat el compose de l'exercici 3.1:

1. Docker Compose crea una xarxa automàticament. Com s'anomena?
2. Des del contenidor `backend`, pots fer `ping db`? Per què?
3. Des del teu ordinador (host), pots fer `ping db`? Per què?
4. Si el servei `db` no tingués `ports: - "5432:5432"`, podria el `backend` connectar-s'hi? I tu des del host?

---

## Bloc 4: Volums, Health Checks i Variables d'Entorn (Dijous)

**Exercici 4.1 — Volums**

1. Quina diferència hi ha entre un **named volume** i un **bind mount**?
2. Quan usaries un named volume? Quan un bind mount?
3. Escriu la línia de `docker-compose.yml` per:
   - Muntar un named volume `pgdata` a `/var/lib/postgresql/data`.
   - Muntar el directori local `./scripts` a `/docker-entrypoint-initdb.d` (bind mount).

**Exercici 4.2 — Dades Efímeres vs Persistents**

Descriu què passa amb les dades en cada escenari:

1. `docker-compose down` (sense `-v`) amb un named volume configurat.
2. `docker-compose down -v` amb un named volume configurat.
3. `docker-compose down` sense cap volum configurat.
4. `docker stop pg16` seguit de `docker start pg16`.
5. `docker stop pg16` seguit de `docker rm pg16`.

**Exercici 4.3 — Variables d'Entorn i .env**

Donat un fitxer `.env`:

```env
POSTGRES_PASSWORD=supersecret
POSTGRES_DB=esportspulse_db
JAVA_OPTS=-Xmx512m
```

I un `docker-compose.yml`:

```yaml
services:
  db:
    image: postgres:16
    environment:
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
  backend:
    build: ./backend-java
    environment:
      - JAVA_OPTS=${JAVA_OPTS}
      - SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/${POSTGRES_DB}
```

1. D'on llegeix Compose els valors de `${POSTGRES_PASSWORD}`?
2. Per què el `.env` NO s'ha de pujar a Git? Què fas en lloc d'això?
3. Si un company es clona el repo i no té `.env`, què passarà quan faci `docker-compose up`?

**Exercici 4.4 — Health Checks**

Escriu health checks per a cada servei:

1. **PostgreSQL:** Comprova que accepta connexions amb `pg_isready`.
2. **Redis:** Comprova que respon amb `redis-cli ping`.
3. **Backend Java:** Comprova que l'endpoint `/actuator/health` retorna 200.
4. **Servei Python:** Comprova que l'endpoint `/health` retorna 200.

Per a cada un, indica: comanda, interval, timeout, retries i start_period.

**Exercici 4.5 — depends_on amb Health Checks**

Explica la diferència entre:

```yaml
depends_on:
  - db
```

i:

```yaml
depends_on:
  db:
    condition: service_healthy
```

Per què la segona forma és molt més fiable en producció?

---

## Bloc 5: Observabilitat de Contenidors (Divendres)

**Exercici 5.1 — Diagnòstic amb Logs**

Donat aquest output de `docker-compose logs`:

```
backend-1  | 2024-09-15 10:00:01 INFO  Starting application...
backend-1  | 2024-09-15 10:00:03 INFO  Connected to database at db:5432
backend-1  | 2024-09-15 10:00:05 INFO  Application started on port 8080
db-1       | 2024-09-15 09:59:58 LOG  database system is ready to accept connections
redis-1    | 2024-09-15 09:59:57 Ready to accept connections tcp
ai-python-1| 2024-09-15 10:00:02 ERROR ConnectionError: Cannot connect to redis:6379
ai-python-1| 2024-09-15 10:00:07 INFO  Connected to Redis
ai-python-1| 2024-09-15 10:00:08 INFO  Server running on port 8000
```

1. Quin servei ha arrencat primer? Quin últim?
2. El servei `ai-python` ha tingut un error. Ha estat un problema greu? Per què?
3. Com podries evitar l'error de connexió de `ai-python`?
4. Si `backend` mostrés "Connection refused" a `db:5432`, quines 3 coses comprovaries?

**Exercici 5.2 — docker stats**

Donat aquest output de `docker stats`:

```
CONTAINER    CPU %   MEM USAGE / LIMIT     MEM %   NET I/O         BLOCK I/O
backend-1    45.2%   489MiB / 512MiB       95.5%   1.2MB / 800KB   5MB / 0B
db-1         2.3%    85MiB / 256MiB        33.2%   800KB / 1.2MB   15MB / 8MB
redis-1      0.1%    8MiB / 128MiB         6.3%    200KB / 150KB   0B / 0B
ai-python-1  12.5%   120MiB / 256MiB       46.9%   500KB / 300KB   1MB / 0B
```

1. Quin contenidor té un problema imminent? Per què?
2. Quin contenidor és el més eficient en ús de memòria?
3. Com solucionaries el problema del `backend`? Dona dues opcions.
4. Per què `db` té més BLOCK I/O que la resta?

**Exercici 5.3 — docker exec**

Escriu la comanda `docker exec` per a cada tasca:

1. Obrir una shell bash dins del contenidor `backend-1`.
2. Comprovar que el backend pot arribar a la base de dades (`ping db`).
3. Llistar els fitxers del directori `/app` dins del contenidor.
4. Veure les variables d'entorn dins del contenidor `ai-python-1`.
5. Executar una query SQL dins del contenidor `db-1` sense entrar a mode interactiu.

**Exercici 5.4 — docker inspect**

Sense executar-lo, què esperaries trobar a la sortida de:

```bash
docker inspect pg16 --format '{{.NetworkSettings.Networks}}'
```

1. Quina informació retorna?
2. Per què és útil quan un contenidor no pot connectar-se a un altre?
3. Quina altra comanda podries usar per veure la xarxa de Docker Compose?

**Exercici 5.5 — Troubleshooting**

Per a cada problema, descriu els passos que seguiries per diagnosticar-lo i quines comandes Docker usaries:

1. El `backend` no arrenca i no hi ha logs.
2. El `backend` arrenca però no pot connectar-se a `db`.
3. Tot funciona però l'aplicació va molt lenta.
4. Després de fer `docker-compose down` i `docker-compose up`, les dades de PostgreSQL han desaparegut.
5. El port 8080 està "already in use" quan intentes arrencar el compose.

---

## Autoavaluació

Abans de passar a la setmana 10, hauries de poder respondre "sí" a tot:

- [ ] Sé la diferència entre imatge, contenidor, daemon i registre
- [ ] Sé executar, aturar, eliminar i inspeccionar contenidors
- [ ] Sé escriure un Dockerfile amb multi-stage build
- [ ] Sé escriure un .dockerignore adequat
- [ ] Sé crear un docker-compose.yml amb múltiples serveis
- [ ] Entenc com funciona la xarxa interna de Docker Compose
- [ ] Sé la diferència entre named volumes i bind mounts
- [ ] Sé configurar health checks per a cada servei
- [ ] Sé usar `depends_on` amb `condition: service_healthy`
- [ ] Sé externalitzar configuració amb `.env`
- [ ] Sé diagnosticar problemes amb `docker logs`, `docker stats`, `docker exec` i `docker inspect`
- [ ] Puc aixecar tota la plataforma amb `docker-compose up` i verificar que tot funciona
