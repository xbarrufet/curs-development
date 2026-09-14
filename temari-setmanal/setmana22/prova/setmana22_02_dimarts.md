# Setmana 22 — Dimarts: Desplegament a Render/Fly.io: Backend Java

## Objectiu del Dia

Desplegar el backend Java (Spring Boot) a una plataforma al nuvol (Render o Fly.io). Al final del dia, l'API ha de ser accessible des de qualsevol lloc amb una URL publica, i has de poder verificar-ho amb `curl` des de la teva maquina local.

---

## Teoria

### Per Que Render o Fly.io?

Hi ha desenes de plataformes per desplegar aplicacions. Per a un projecte d'aprenentatge, busquem:

| Criteri | Render | Fly.io |
|---------|--------|--------|
| Free tier | Si (750h/mes) | Si (3 maquines compartides) |
| Docker support | Si | Si |
| PostgreSQL | Si (free tier 90 dies) | Si (amb Supabase o similar) |
| Setup | Molt facil (connecta GitHub) | Facil (CLI) |
| Escalabilitat | Automatica | Manual |
| Custom domains | Si | Si |

**Recomanacio**: Render es mes senzill per comenar. Fly.io dona mes control.

### Com Funciona el Desplegament amb Docker

Quan despleguem a Render o Fly.io, la plataforma:

```
# 1. Rep el teu codi (via GitHub o docker push)
# 2. Construeix la imatge Docker (docker build)
# 3. Executa el contenidor (docker run)
# 4. Assigna una URL publica (xxxxx.onrender.com)
# 5. Redirigeix el trafic HTTP/HTTPS al teu contenidor
# 6. Monitoritza la salut (health checks)
# 7. Reinicia si cau (restart policy)
```

### Preparar el Dockerfile per Produccio

El Dockerfile de desenvolupament i el de produccio son diferents:

```dockerfile
# Dockerfile per a PRODUCCIO del backend Java
# Usa multi-stage build per reduir la mida de la imatge final

# ETAPA 1: Compilacio
# Usa una imatge amb Maven i JDK per compilar el codi
FROM maven:3.9-eclipse-temurin-21 AS builder

# Directori de treball dins el contenidor
WORKDIR /app

# Copia primer el pom.xml per aprofitar la cache de Docker
# Si les dependencies no canvien, Docker reutilitza aquesta capa
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copia el codi font i compila
COPY src ./src
RUN mvn package -DskipTests -B

# ETAPA 2: Execucio
# Usa una imatge mes lleugera (nomes JRE, sense JDK ni Maven)
# eclipse-temurin es la distribucio oficial de Java d'Eclipse
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

# Copia NOMES el JAR compilat des de l'etapa anterior
# Aixo redueix la imatge de ~800MB a ~200MB
COPY --from=builder /app/target/*.jar app.jar

# Port que exposa l'aplicacio Spring Boot
EXPOSE 8080

# Configuracio de la JVM per a contenidors
# -XX:+UseContainerSupport: la JVM respecta els limits del contenidor
# -XX:MaxRAMPercentage=75: usa com a maxim el 75% de la RAM assignada
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-jar", "app.jar"]
```

**Per que multi-stage build?**

```
# Imatge amb Maven + JDK:    ~800 MB
# Imatge amb nomes JRE:      ~200 MB
# La diferencia importa per a:
# - Temps de desplegament (menys dades a transferir)
# - Cost (menys espai de disc al nuvol)
# - Seguretat (menys eines = menys superfici d'atac)
```

### Configuracio de Perfils (Dev vs Prod)

Spring Boot permet configuracions diferents per entorn:

```yaml
# application.yml — configuracio per defecte (desenvolupament)
spring:
  datasource:
    # En desenvolupament, connectem al postgres local
    url: jdbc:postgresql://localhost:5432/esportspulse
    username: esportspulse
    password: ${POSTGRES_PASSWORD:dev-password}

# application-prod.yml — configuracio de produccio
# S'activa amb SPRING_PROFILES_ACTIVE=prod
spring:
  datasource:
    # En produccio, la URL ve de la variable d'entorn
    # que proporciona la plataforma (Render, Fly.io)
    url: ${DATABASE_URL}
    username: ${POSTGRES_USER}
    password: ${POSTGRES_PASSWORD}
```

```bash
# Per activar el perfil de produccio, la plataforma defineix:
# SPRING_PROFILES_ACTIVE=prod
# Aixo fa que Spring Boot llegeixi application-prod.yml
```

### Health Checks

Les plataformes al nuvol necessiten saber si la teva aplicacio esta sana:

```java
// HealthController.java
// Endpoint que la plataforma crida periodicament per verificar
// que l'aplicacio funciona correctament

@RestController
@RequestMapping("/health")
public class HealthController {

    // Injectem el DataSource per verificar la connexio a BD
    private final DataSource dataSource;

    public HealthController(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    // GET /health — retorna l'estat de l'aplicacio
    // NO requereix autenticacio (la plataforma no te JWT)
    @GetMapping
    public ResponseEntity<Map<String, String>> health() {
        Map<String, String> status = new HashMap<>();
        status.put("status", "UP");
        status.put("service", "esportspulse-backend");
        
        // Verificar connexio a la base de dades
        try (Connection conn = dataSource.getConnection()) {
            status.put("database", "connected");
        } catch (Exception e) {
            status.put("database", "disconnected");
            status.put("status", "DEGRADED");
        }
        
        return ResponseEntity.ok(status);
    }
}
```

---

## Activitat

### 1. Preparar el Dockerfile de Produccio (20 min)

Crea o actualitza el Dockerfile del backend Java amb multi-stage build:

```bash
# Verifica que el Dockerfile actual funciona localment
cd backend-java
docker build -t esportspulse-backend:prod .

# Prova que l'imatge arrenca
docker run -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  -e DATABASE_URL=jdbc:postgresql://host.docker.internal:5432/esportspulse \
  -e POSTGRES_USER=esportspulse \
  -e POSTGRES_PASSWORD=test \
  esportspulse-backend:prod
```

### 2. Crear el Perfil de Produccio (15 min)

Crea `src/main/resources/application-prod.yml`:

```yaml
# Configuracio de produccio
# TOTES les dades sensibles venen de variables d'entorn
spring:
  datasource:
    url: ${DATABASE_URL}
    username: ${POSTGRES_USER}
    password: ${POSTGRES_PASSWORD}
  jpa:
    # En produccio, NO volem que Hibernate modifiqui l'esquema
    # automaticament. Usem migracions (Flyway o Liquibase)
    hibernate:
      ddl-auto: validate

# Desactivar informacio de depuracio en produccio
# Nomes mostrar errors greus
logging:
  level:
    root: WARN
    com.esportspulse: INFO

# Configuracio del servidor
server:
  port: ${PORT:8080}
```

### 3. Crear Compte i Desplegar a Render (40 min)

**Opcio A: Render (recomanat per simplicitat)**

1. Crea un compte a [render.com](https://render.com).
2. Connecta el teu repositori GitHub.
3. Crea un "New Web Service":

```
# Configuracio a Render:
Name: esportspulse-backend
Environment: Docker
Region: Frankfurt (EU)
Branch: main
Root Directory: backend-java (si es un subdirectori)

# Variables d'entorn:
SPRING_PROFILES_ACTIVE=prod
PORT=8080
POSTGRES_USER=esportspulse
POSTGRES_PASSWORD=(genera una contrasenya segura)
JWT_SECRET=(genera un secret aleatori)
DATABASE_URL=(Render la proporciona si crees una BD)
```

4. Crea una base de dades PostgreSQL a Render:

```
# Render ofereix PostgreSQL gratis durant 90 dies
# Crea una nova BD i copia la Internal Database URL
# Format: postgres://user:password@host:5432/dbname
```

**Opcio B: Fly.io**

```bash
# Instal·la la CLI de Fly.io
brew install flyctl

# Inicia sessio (o crea compte)
flyctl auth login

# Inicialitza l'aplicacio
cd backend-java
flyctl launch --name esportspulse-backend

# Configura les variables d'entorn (secrets)
flyctl secrets set SPRING_PROFILES_ACTIVE=prod
flyctl secrets set POSTGRES_PASSWORD=la-teva-contrasenya
flyctl secrets set JWT_SECRET=el-teu-secret
flyctl secrets set DATABASE_URL=postgres://...

# Desplega
flyctl deploy
```

### 4. Verificar el Desplegament (20 min)

```bash
# Substitueix la URL per la que t'assigni la plataforma
BACKEND_URL="https://esportspulse-backend.onrender.com"

# 1. Health check
curl -s $BACKEND_URL/health | python3 -m json.tool

# 2. Endpoint public (si tens algun sense auth)
curl -s $BACKEND_URL/api/v1/public/status

# 3. Login per obtenir token JWT
TOKEN=$(curl -s -X POST $BACKEND_URL/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"test123"}' | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

echo "Token: $TOKEN"

# 4. Endpoint autenticat
curl -s -H "Authorization: Bearer $TOKEN" \
  $BACKEND_URL/api/v1/teams | python3 -m json.tool
```

### 5. Configurar Health Check a la Plataforma (10 min)

A Render o Fly.io, configura:

```
# Health Check Path: /health
# Interval: 30 segons
# Timeout: 10 segons
# Retries: 3
# Aixo fa que la plataforma reiniciï el servei si no respon
```

### 6. Commit (10 min)

```bash
git add Dockerfile application-prod.yml
git commit -m "feat(deploy): add production Dockerfile and Spring profile for cloud deployment"
```

---

## Checklist de Lliurament

- [ ] Dockerfile de produccio amb multi-stage build
- [ ] Perfil `application-prod.yml` creat
- [ ] Servei desplegat a Render o Fly.io
- [ ] Health check respon correctament (`/health`)
- [ ] Login funciona des de fora (curl des de local)
- [ ] Endpoint autenticat funciona amb JWT
- [ ] Base de dades PostgreSQL al nuvol connectada
- [ ] Variables d'entorn configurades (cap secret hardcodejat)
- [ ] Commit amb els canvis de desplegament
