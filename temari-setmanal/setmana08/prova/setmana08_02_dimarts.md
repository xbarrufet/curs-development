# Setmana 8 — Dimarts: Dockerfile: Empaquetar la Teva Aplicació

## Objectiu del Dia

Escriure Dockerfiles per al backend Java i el servei Python del projecte, construir les imatges i executar-les com a contenidors independents. Al final del dia has de poder fer `docker build` i `docker run` per a cadascun dels dos serveis i veure que responen correctament.

---

## Teoria

### Què és un Dockerfile?

Un Dockerfile és una **recepta** que descriu, pas a pas, com construir una imatge Docker. Una imatge és una plantilla immutable que conté tot el que necessita la teva aplicació per funcionar: sistema operatiu base, dependències, codi compilat i instruccions d'arrencada.

Pensa-hi com una recepta de cuina:
- **FROM** — el plat base (la imatge sobre la qual construïm)
- **COPY** — afegir ingredients (copiar fitxers al contenidor)
- **RUN** — cuinar (executar comandes durant la construcció)
- **CMD** — servir el plat (la comanda que s'executa quan arrenca el contenidor)
- **EXPOSE** — documentar quin port utilitza l'aplicació

```dockerfile
# Cada instrucció crea una "capa" (layer) de la imatge.
# Docker guarda en cache cada capa. Si no canvia, no la reconstrueix.
# Per això l'ordre importa: posar primer el que canvia menys.

FROM eclipse-temurin:21-jre        # Imatge base amb Java 21 (només JRE, no JDK)
COPY target/app.jar /app/app.jar   # Copiar el JAR compilat dins la imatge
EXPOSE 8080                        # Documentar que l'app escolta al port 8080
CMD ["java", "-jar", "/app/app.jar"]  # Comanda per arrencar el contenidor
```

> **Imatge vs Contenidor:** Una imatge és la recepta congelada; un contenidor és un plat servit. Pots crear molts contenidors a partir de la mateixa imatge.

### Les Instruccions Principals

| Instrucció | Què fa | Exemple |
|------------|--------|---------|
| `FROM` | Defineix la imatge base | `FROM python:3.12-slim` |
| `WORKDIR` | Estableix el directori de treball dins el contenidor | `WORKDIR /app` |
| `COPY` | Copia fitxers del host al contenidor | `COPY src/ /app/src/` |
| `RUN` | Executa una comanda durant el build | `RUN pip install -r requirements.txt` |
| `CMD` | Comanda per defecte quan arrenca el contenidor | `CMD ["python", "main.py"]` |
| `EXPOSE` | Documenta el port que utilitza l'aplicació | `EXPOSE 5000` |
| `ENV` | Defineix variables d'entorn | `ENV JAVA_OPTS="-Xmx512m"` |

### Multi-stage Builds: Imatges Petites i Segures

Quan compiles un projecte Java, necessites el JDK (Java Development Kit) i Maven. Però en producció, només necessites el JRE (Java Runtime Environment) i el JAR compilat. Incloure eines de compilació a la imatge final és:
- **Malbaratament d'espai** — el JDK ocupa centenars de MB innecessaris
- **Risc de seguretat** — menys eines = menys superfície d'atac

La solució és el **multi-stage build**: una primera etapa compila, una segona etapa només copia el resultat.

```dockerfile
# === ETAPA 1: Compilació ===
# Usem una imatge amb JDK + Maven per compilar el projecte.
# Aquesta etapa es descarta al final; no forma part de la imatge final.
FROM eclipse-temurin:21-jdk AS builder

# Establim el directori de treball dins el contenidor
WORKDIR /build

# Primer copiem NOMÉS el pom.xml per aprofitar la cache de Docker.
# Si les dependències no canvien, Docker reutilitza la capa anterior.
COPY pom.xml .

# Descarreguem les dependències (sense compilar el codi).
# -B = mode batch (sense output interactiu)
# go-offline = descarrega tot el que necessita Maven
RUN mvn dependency:go-offline -B

# Ara copiem el codi font. Si només canvia el codi,
# Docker reutilitza la capa de dependències (molt més ràpid).
COPY src/ src/

# Compilem el projecte i creem el JAR.
# -DskipTests perquè els tests es passen en CI, no durant el build de la imatge.
RUN mvn package -DskipTests -B

# === ETAPA 2: Imatge Final ===
# Usem una imatge lleugera amb NOMÉS el JRE (no JDK, no Maven).
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

# Copiem NOMÉS el JAR compilat des de l'etapa builder.
# Tot el JDK, Maven i codi font queden fora de la imatge final.
COPY --from=builder /build/target/*.jar app.jar

# Documentem el port que utilitza Spring Boot per defecte
EXPOSE 8080

# Comanda per arrencar l'aplicació
CMD ["java", "-jar", "app.jar"]
```

Comparem mides:

| Imatge | Mida aproximada |
|--------|-----------------|
| `eclipse-temurin:21-jdk` (tot inclòs) | ~450 MB |
| `eclipse-temurin:21-jre-alpine` (només JRE) | ~190 MB |
| La nostra imatge final (JRE + JAR) | ~200 MB |

> **Per què Alpine?** Alpine Linux és una distribució minimalista (~5 MB). Combinada amb JRE, obtenim una imatge molt més lleugera.

### .dockerignore: No Copiar Escombraries

Igual que `.gitignore` evita pujar fitxers innecessaris a Git, `.dockerignore` evita copiar-los dins la imatge Docker:

```dockerignore
# No copiar res que no sigui necessari per al build
target/
.git/
.gitignore
*.md
.idea/
*.iml
__pycache__/
.venv/
.env
```

> **Per què importa?** Sense `.dockerignore`, un `COPY . .` copiaria el directori `.git` (potencialment centenars de MB), fitxers temporals i secrets com `.env`.

---

## Activitat

### Part 1: Dockerfile per al Backend Java

**1.1. Crea el fitxer `backend-java/Dockerfile`:**

```dockerfile
# ===================================================================
# Dockerfile per al backend Java (Spring Boot + Maven)
# Multi-stage build: compila amb JDK, executa amb JRE
# ===================================================================

# --- Etapa 1: Compilar el projecte ---
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /build

# Copiar primer el pom.xml per cachear dependències
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copiar el codi font i compilar
COPY src/ src/
RUN mvn package -DskipTests -B

# --- Etapa 2: Imatge d'execució ---
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# Copiar només el JAR des de l'etapa de compilació
COPY --from=builder /build/target/*.jar app.jar

# El backend escolta al port 8080
EXPOSE 8080

# Arrencar l'aplicació
# -Djava.security.egd: millora la velocitat d'arrencada en contenidors
CMD ["java", "-Djava.security.egd=file:/dev/./urandom", "-jar", "app.jar"]
```

**1.2. Crea el fitxer `backend-java/.dockerignore`:**

```dockerignore
target/
.git/
.gitignore
*.md
.idea/
*.iml
```

**1.3. Construeix i executa:**

```bash
# Situats a la carpeta backend-java/
cd backend-java

# Construir la imatge. -t = tag (nom de la imatge)
# El "." final indica que el context de build és el directori actual
docker build -t esportspulse-backend:latest .

# Comprovar que la imatge s'ha creat
docker images | grep esportspulse

# Executar el contenidor
# -d = detached (en segon pla)
# -p 8080:8080 = mapejar el port 8080 del host al 8080 del contenidor
# --name = donar un nom al contenidor (per identificar-lo fàcilment)
docker run -d -p 8080:8080 --name backend esportspulse-backend:latest

# Verificar que funciona
curl http://localhost:8080/actuator/health
# Hauria de retornar: {"status":"UP"}

# Veure els logs del contenidor
docker logs backend
```

### Part 2: Dockerfile per al Servei Python

**2.1. Crea el fitxer `ai-python/Dockerfile`:**

```dockerfile
# ===================================================================
# Dockerfile per al servei Python (FastAPI o Flask)
# Imatge slim: sense eines de compilació innecessàries
# ===================================================================

# Usem python:3.12-slim en lloc de python:3.12
# slim = sense compiladors C, man pages, etc. (~150 MB vs ~1 GB)
FROM python:3.12-slim

WORKDIR /app

# Copiar PRIMER les dependències per aprofitar la cache de Docker.
# Si requirements.txt no canvia, Docker no reinstal·la els paquets.
COPY requirements.txt .

# Instal·lar dependències Python
# --no-cache-dir: no guardar cache de pip (redueix mida de la imatge)
# --no-compile: no generar fitxers .pyc durant la instal·lació
RUN pip install --no-cache-dir -r requirements.txt

# Ara copiar el codi font
# Si només canvia el codi, les dependències ja estan en cache
COPY src/ src/

# El servei Python escolta al port 5000
EXPOSE 5000

# Arrencar l'aplicació amb uvicorn (servidor ASGI per a FastAPI)
# --host 0.0.0.0: acceptar connexions de fora del contenidor
# (per defecte escoltaria només a 127.0.0.1, invisible des del host)
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "5000"]
```

**2.2. Crea el fitxer `ai-python/.dockerignore`:**

```dockerignore
__pycache__/
.venv/
.git/
.gitignore
*.md
.env
*.pyc
.pytest_cache/
```

**2.3. Construeix i executa:**

```bash
# Situats a la carpeta ai-python/
cd ai-python

# Construir la imatge
docker build -t esportspulse-ai:latest .

# Comprovar la mida de la imatge
docker images | grep esportspulse

# Executar el contenidor
# -p 5000:5000 = mapejar port 5000
docker run -d -p 5000:5000 --name ai-service esportspulse-ai:latest

# Verificar que funciona
curl http://localhost:5000/health
# Hauria de retornar: {"status":"ok"}

# Veure els logs
docker logs ai-service
```

### Part 3: Verificació i Neteja

```bash
# Llistar tots els contenidors en execució
docker ps

# Hauries de veure:
# CONTAINER ID   IMAGE                       PORTS                    NAMES
# abc123         esportspulse-backend:latest  0.0.0.0:8080->8080/tcp  backend
# def456         esportspulse-ai:latest       0.0.0.0:5000->5000/tcp  ai-service

# Aturar els contenidors
docker stop backend ai-service

# Eliminar els contenidors aturats
docker rm backend ai-service

# (Opcional) Eliminar les imatges si vols alliberar espai
# docker rmi esportspulse-backend:latest esportspulse-ai:latest
```

> **Nota:** Cada cop que canviïs el codi i vulguis actualitzar el contenidor, has de reconstruir la imatge (`docker build`) i recrear el contenidor (`docker run`). Demà veurem Docker Compose, que simplifica aquest procés.

---

## Checklist de Lliurament

- [ ] `backend-java/Dockerfile` utilitza multi-stage build amb `eclipse-temurin:21`
- [ ] `ai-python/Dockerfile` utilitza `python:3.12-slim` i instal·la dependències abans de copiar el codi
- [ ] Ambdós Dockerfiles tenen `.dockerignore` corresponent
- [ ] `docker build` funciona sense errors per a ambdós serveis
- [ ] `docker run` arrenca els contenidors i responen als seus ports respectius
- [ ] `docker images | grep esportspulse` mostra les dues imatges amb mides raonables (backend <250 MB, Python <300 MB)
- [ ] Tots els fitxers estan commitejats amb un missatge descriptiu (p.ex. `feat(docker): add Dockerfiles for Java backend and Python service`)
