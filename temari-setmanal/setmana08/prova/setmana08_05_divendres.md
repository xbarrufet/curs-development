# Setmana 8 — Divendres: Observabilitat de Contenidors i Consolidació

## Objectiu del Dia

Dominar les eines d'observabilitat de Docker per monitoritzar, depurar i diagnosticar problemes als contenidors. Al final del dia, has de poder arrencar l'stack complet, verificar que tot funciona, depurar un problema dins d'un contenidor i crear un PR amb tota la configuració Docker de la setmana.

---

## Teoria

### Per Què l'Observabilitat Importa

Quan tens 4 contenidors executant-se, les coses es compliquen. Un error pot estar en qualsevol d'ells, i la sortida de tots es barreja. Necessites eines per respondre preguntes com:
- Quin servei ha fallat?
- Quanta memòria consumeix cada contenidor?
- Què passa dins d'un contenidor que no respon?

### Logs: La Primera Línia de Diagnòstic

```bash
# Logs de tots els serveis (últimes línies)
docker-compose logs

# Logs en temps real de tots els serveis
# -f = follow (com tail -f de Linux, que vam veure a S3)
docker-compose logs -f

# Logs d'un servei concret, en temps real
docker-compose logs -f backend

# Últimes 100 línies d'un servei
docker-compose logs --tail=100 ai-service

# Logs amb marca de temps (útil per correlacionar entre serveis)
docker-compose logs -f --timestamps
```

**Quan tens 4 contenidors emetent logs alhora, la sortida és caòtica:**

```
backend      | 2024-03-15 10:23:45 INFO  Starting EsportsPulseApplication
postgres     | 2024-03-15 10:23:44 LOG  database system is ready
qdrant       | 2024-03-15 10:23:43 INFO  Qdrant gRPC listening on 6334
ai-service   | 2024-03-15 10:23:46 INFO  Uvicorn running on 0.0.0.0:5000
backend      | 2024-03-15 10:23:47 INFO  Connected to database
ai-service   | 2024-03-15 10:23:47 INFO  Connected to Qdrant
```

**Per què els logs en format JSON (S4, correlation IDs) són importants aquí:** Si cada servei emet logs en JSON amb un `correlation_id`, pots filtrar per una petició concreta que travessa múltiples serveis. Sense JSON estructurat, només tens text pla barrejat.

```json
{"timestamp": "2024-03-15T10:23:47Z", "level": "INFO", "service": "backend", "correlation_id": "abc-123", "message": "Request received"}
{"timestamp": "2024-03-15T10:23:47Z", "level": "INFO", "service": "ai-service", "correlation_id": "abc-123", "message": "Processing embedding"}
```

Amb un `grep` pots extreure tota la traça:

```bash
# Filtrar per correlation_id (funciona perquè són JSON, no text lliure)
docker-compose logs | grep "abc-123"
```

### Estadístiques de Recursos: CPU i Memòria

```bash
# Veure CPU, memòria, xarxa i I/O de tots els contenidors en temps real
docker stats

# Exemple de sortida:
# CONTAINER    CPU %    MEM USAGE / LIMIT     MEM %    NET I/O
# backend      0.50%    256MiB / 8GiB         3.20%    1.2kB / 0B
# ai-service   0.10%    128MiB / 8GiB         1.60%    500B / 0B
# postgres     0.05%    64MiB / 8GiB          0.80%    300B / 0B
# qdrant       0.03%    96MiB / 8GiB          1.20%    200B / 0B

# Estadístiques d'un contenidor concret (útil en scripts)
docker stats --no-stream backend
# --no-stream: mostra una captura i surt (no actualitza en temps real)
```

> **Pregunta per reflexionar:** Si el backend consumeix 2 GB de RAM i creix constantment, probablement tens un **memory leak**. `docker stats` t'alerta d'això abans que el contenidor caigui.

### Processos dins d'un Contenidor

```bash
# Veure els processos que s'executen dins d'un contenidor
docker top backend

# Exemple de sortida:
# PID     USER    COMMAND
# 1       root    java -jar app.jar

# Un contenidor ben fet hauria de tenir UN sol procés principal.
# Si veus molts processos, potser el Dockerfile està mal dissenyat.
```

### docker exec: Entrar dins d'un Contenidor

A la setmana 3 vam treballar amb el terminal i comandes de Linux. Ara podem aplicar exactament les mateixes habilitats **dins d'un contenidor en execució**:

```bash
# Obrir un shell interactiu dins d'un contenidor
# -i = interactiu (mantenir STDIN obert)
# -t = pseudo-TTY (terminal)
docker exec -it backend sh

# Un cop dins, ets "dins la màquina" del contenidor.
# Pots executar qualsevol comanda que tingui instal·lada:
ls -la /app/            # Veure els fitxers de l'aplicació
env                     # Veure les variables d'entorn
cat /etc/os-release     # Saber quina distribució Linux és
ps aux                  # Processos en execució

# Per sortir del contenidor
exit
```

**Per a PostgreSQL (shell interactiu de psql):**

```bash
# Connectar directament a psql dins del contenidor
docker-compose exec postgres psql -U esportspulse -d esportspulse_db

# Un cop dins de psql:
\dt                     -- Llistar taules
\d+ nom_taula           -- Veure estructura d'una taula
SELECT * FROM test;     -- Executar queries
\q                      -- Sortir de psql
```

**Per al servei Python:**

```bash
# Entrar al contenidor Python
docker-compose exec ai-service sh

# Un cop dins:
python3 --version       # Verificar versió de Python
pip list                # Veure paquets instal·lats
python3 -c "import fastapi; print(fastapi.__version__)"  # Verificar dependència

exit
```

> **Quan usar `docker exec`?**
> - Depurar un servei que no respon
> - Verificar que les variables d'entorn s'han carregat correctament
> - Executar comandes de diagnòstic (ping, curl, nslookup)
> - Inspeccionar fitxers dins del contenidor
> - Executar migracions de base de dades manualment

### Inspecció Avançada de Contenidors

```bash
# Informació completa d'un contenidor (JSON)
docker inspect backend

# Filtrar informació específica amb --format (sintaxi Go templates)
# Veure la IP del contenidor dins la xarxa Docker
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' backend

# Veure les variables d'entorn d'un contenidor
docker inspect --format='{{range .Config.Env}}{{println .}}{{end}}' backend

# Veure els volums muntats
docker inspect --format='{{json .Mounts}}' postgres | python3 -m json.tool
```

---

## Activitat

### Part 1: Arrencar l'Stack i Monitoritzar

**1.1. Arrenca tot l'stack en segon pla:**

```bash
cd esportspulse-engine

# Arrencar amb build (per si hi ha canvis des de dijous)
docker-compose up -d --build

# Esperar uns segons i verificar que tot és healthy
docker-compose ps
```

**1.2. Observa els logs d'arrencada:**

```bash
# Veure els logs de tots els serveis amb timestamps
docker-compose logs --timestamps

# Observa l'ordre: postgres i qdrant arranquen primer,
# després backend i ai-service quan les dependències són healthy.
```

**1.3. Monitoritza els recursos:**

```bash
# Deixa docker stats corrent en un terminal
docker stats

# En un altre terminal, fes peticions al backend per veure
# com canvia el consum de CPU i memòria
curl http://localhost:8080/actuator/health
```

### Part 2: Depurar dins dels Contenidors

**2.1. Verifica les variables d'entorn del backend:**

```bash
# Entrar al contenidor del backend
docker-compose exec backend sh

# Verificar que les variables s'han carregat correctament
env | grep SPRING
# Hauries de veure:
# SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/esportspulse_db
# SPRING_DATASOURCE_USERNAME=esportspulse
# SPRING_DATASOURCE_PASSWORD=dev_secret_2024
# SPRING_PROFILES_ACTIVE=dev

# Verificar connectivitat amb PostgreSQL des de dins del contenidor
# (ping per comprovar que el DNS resol correctament)
ping -c 2 postgres

exit
```

**2.2. Verifica la base de dades:**

```bash
# Entrar a psql dins del contenidor de PostgreSQL
docker-compose exec postgres psql -U esportspulse -d esportspulse_db

# Dins de psql:
# Llistar totes les taules (si el backend ha creat les taules amb JPA)
\dt

# Veure la versió de PostgreSQL
SELECT version();

# Sortir
\q
```

**2.3. Verifica el servei Python:**

```bash
# Entrar al contenidor Python
docker-compose exec ai-service sh

# Verificar que Qdrant és accessible pel nom de servei
# (necessitaràs curl o wget; si no estan instal·lats, veure nota)
curl http://qdrant:6333/healthz

# Verificar les variables d'entorn
env | grep QDRANT
# QDRANT_HOST=qdrant
# QDRANT_PORT=6333

exit
```

> **Nota:** Les imatges `slim` i `alpine` no inclouen `curl` ni `ping` per defecte. Si els necessites per depurar, pots instal·lar-los temporalment: `apk add curl` (Alpine) o `apt-get update && apt-get install -y curl` (Debian/slim). Recorda que els canvis dins d'un contenidor es perden quan es reinicia.

### Part 3: Exercici Integrador

Ara posarem tot junt. Segueix els passos i verifica cada punt:

**3.1. Comprova que l'stack és complet i robust:**

```bash
# 1. Arrencar l'stack
docker-compose up -d --build

# 2. Verificar que tots els serveis són healthy
docker-compose ps
# Tots han de mostrar "Up (healthy)"

# 3. Verificar health checks individualment
curl http://localhost:8080/actuator/health    # Backend Java
curl http://localhost:5000/health              # Servei Python
curl http://localhost:6333/healthz             # Qdrant

# Verificar PostgreSQL
docker-compose exec postgres pg_isready -U esportspulse -d esportspulse_db

# 4. Verificar logs (no hi ha errors)
docker-compose logs | grep -i error
# Idealment no hauria de mostrar res (o només errors esperats d'arrencada)

# 5. Verificar recursos
docker stats --no-stream

# 6. Verificar persistència
docker-compose exec postgres psql -U esportspulse -d esportspulse_db -c "
INSERT INTO test_persistencia (missatge) VALUES ('Test divendres');
SELECT * FROM test_persistencia;
"

# 7. Reiniciar i verificar persistència
docker-compose down
docker-compose up -d
docker-compose exec postgres psql -U esportspulse -d esportspulse_db -c "
SELECT * FROM test_persistencia;
"
# Les dades han de seguir allà
```

### Part 4: Commit i Pull Request

**4.1. Assegura't que tot està en ordre:**

```bash
# Verificar l'estructura de fitxers Docker
find . -name "Dockerfile" -o -name "docker-compose.yml" -o -name ".dockerignore" -o -name ".env.example" | sort

# Hauries de veure:
# ./ai-python/.dockerignore
# ./ai-python/Dockerfile
# ./backend-java/.dockerignore
# ./backend-java/Dockerfile
# ./docker-compose.yml
# ./.env.example

# Verificar que .env NO està a Git
git status
# .env NO hauria d'aparèixer com a fitxer "untracked"
# (si apareix, revisa el .gitignore)
```

**4.2. Crea el commit i el PR:**

```bash
# Afegir tots els fitxers Docker
git add backend-java/Dockerfile backend-java/.dockerignore
git add ai-python/Dockerfile ai-python/.dockerignore
git add docker-compose.yml .env.example .gitignore

# Commit amb missatge descriptiu
git commit -m "feat(docker): full Docker setup with Compose, health checks and volumes

- Multi-stage Dockerfiles for Java backend and Python service
- docker-compose.yml with PostgreSQL, Qdrant, backend and ai-service
- Health checks for all services with proper dependency ordering
- Named volumes for data persistence
- Environment variables externalized to .env
- .env.example with documentation for team members"

# Crear la branca i el PR
git checkout -b feature/week8-docker
git push -u origin feature/week8-docker
```

### Retrospectiva: I Si Això Fos Producció?

Pren-te 10 minuts per reflexionar sobre aquestes preguntes:

1. **Escalabilitat:** Ara tens una instància de cada servei. Què passes si el backend necessita gestionar 10x més peticions? (Pista: `docker-compose up --scale backend=3` existeix, però... com reparteixes les peticions?)

2. **Actualitzacions:** Com actualitzes el backend sense aturar el servei? (Pista: **zero-downtime deployment** -- ho veurem a la S20 amb CI/CD.)

3. **Seguretat:** El fitxer `.env` és per a desenvolupament. En producció, qui gestiona els secrets? (Pista: GitHub Secrets, AWS Secrets Manager, HashiCorp Vault.)

4. **Monitorització:** `docker stats` és manual. En producció necessites alertes automàtiques. (Pista: Prometheus + Grafana, que podríem afegir al compose.)

5. **Backups:** Si el volum de PostgreSQL es corromp, has perdut tot. (Pista: backups automatitzats, replicació, serveis gestionats com AWS RDS.)

> **Connexió amb la S20 (CI/CD):** Tot el que hem fet aquesta setmana -- Dockerfiles, Compose, health checks -- és el fonament del desplegament automatitzat. A la S20 integrarem Docker amb un pipeline de CI/CD que construeix les imatges, passa els tests i desplega automàticament.

---

## Checklist de Lliurament

- [ ] L'stack complet arrenca amb `docker-compose up -d --build` sense errors
- [ ] `docker-compose ps` mostra tots els serveis com a "healthy"
- [ ] `docker-compose logs | grep -i error` no mostra errors inesperats
- [ ] `docker stats --no-stream` mostra recursos raonables per a cada contenidor
- [ ] Has pogut entrar dins d'un contenidor amb `docker exec` i verificar variables d'entorn
- [ ] Les dades de PostgreSQL sobreviuen a `docker-compose down` i `docker-compose up`
- [ ] Tots els fitxers Docker estan commitejats i pujats al repositori
- [ ] El PR inclou: Dockerfiles, .dockerignore, docker-compose.yml, .env.example, .gitignore actualitzat
- [ ] Has reflexionat sobre les preguntes de la retrospectiva (no cal lliurar-ho, és per a tu)
