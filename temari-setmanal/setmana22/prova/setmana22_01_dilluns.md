# Setmana 22 — Dilluns: Auditoria de Seguretat Pre-Desplegament

## Objectiu del Dia

Fer una auditoria completa de seguretat del projecte abans de desplegar-lo al nuvol. Verificar que no hi ha secrets exposats al codi, a la historia de Git, ni a les imatges Docker. Al final del dia, el projecte ha d'estar net de qualsevol informacio sensible.

---

## Teoria

### Per Que una Auditoria de Seguretat Abans de Desplegar?

Quan el teu projecte nomes corre a `localhost`, un secret hardcodejat es un mal habit. Quan el poses al nuvol amb una URL publica, es una vulnerabilitat critica.

**Que pot anar malament?**

```
# Escenari real (passa constantment):
# 1. Un dev posa una API key al codi per "provar rapidament"
# 2. Fa commit i push
# 3. L'elimina al commit seguent
# 4. Pensa que ja esta resolt
# 5. FALS: la clau segueix a la historia de Git
# 6. Bots escanegen GitHub cada minut buscant claus exposades
# 7. En menys de 24h, algú usa la clau per generar costos
```

### Tipus de Secrets a Buscar

```bash
# Secrets habituals en un projecte com EsportsPulse:
# 1. API Keys (Claude/Anthropic, LangFuse, etc.)
# 2. Contrasenyes de bases de dades (PostgreSQL)
# 3. JWT Secrets (per signar tokens)
# 4. Credencials de Redis
# 5. Credencials de RabbitMQ (guest/guest es el defecte!)
# 6. Tokens de GitHub (per CI/CD)
# 7. Contrasenyes de Grafana/Prometheus
```

### On Buscar Secrets

**1. Al codi font:**

```bash
# Busca patrons de secrets al codi actual
# grep amb expressions regulars per trobar patrons sospitosos
grep -rn "password\|secret\|api_key\|api-key\|apikey\|token" \
  --include="*.java" --include="*.py" --include="*.yml" --include="*.yaml" \
  --include="*.properties" --include="*.json" \
  --exclude-dir=node_modules --exclude-dir=.git .

# Busca strings que semblin claus (llargs, alfanumerics)
# Patro: qualsevol string de mes de 20 caracters entre cometes
grep -rn "'[A-Za-z0-9/+=]\{20,\}'" --include="*.py" .
grep -rn '"[A-Za-z0-9/+=]\{20,\}"' --include="*.java" .
```

**2. A la historia de Git:**

```bash
# CRUCIAL: busca secrets a TOTS els commits, no nomes l'actual
# -p mostra el contingut de cada commit (el diff)
# --all busca a totes les branques
# -S busca una string especifica dins els canvis
git log -p --all -S 'password'
git log -p --all -S 'secret'
git log -p --all -S 'api_key'
git log -p --all -S 'ANTHROPIC_API_KEY'

# Si trobes un secret a la historia, eliminar-lo del commit actual
# NO es suficient. Cal reescriure la historia amb git-filter-repo
# o considerar el secret compromès i rotar-lo.
```

**3. Al docker-compose.yml:**

```yaml
# MAL: secrets hardcodejats al docker-compose
# services:
#   postgres:
#     environment:
#       POSTGRES_PASSWORD: supersecret123  # PERILL!

# BE: usar variables d'entorn des de .env
# services:
#   postgres:
#     environment:
#       POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}  # Llegeix de .env
```

**4. A les imatges Docker:**

```bash
# Les variables d'entorn dins un Dockerfile queden a la imatge
# Qualsevol amb acces a la imatge pot veure-les
docker history <imatge> --no-trunc

# MAI fer aixo:
# ENV API_KEY=sk-abc123...
# En comptes d'aixo, passa les variables en temps d'execucio:
# docker run -e API_KEY=$API_KEY ...
```

### Variables d'Entorn: El Patro Correcte

```bash
# Estructura recomanada:

# 1. .env.example (comitejat) — plantilla sense valors reals
# POSTGRES_PASSWORD=canvia_aquest_valor
# JWT_SECRET=genera_un_secret_aleatori
# ANTHROPIC_API_KEY=la_teva_api_key_aqui

# 2. .env (NO comitejat, al .gitignore) — valors reals
# POSTGRES_PASSWORD=xK9#mP2$vL5nQ8
# JWT_SECRET=a1b2c3d4e5f6g7h8i9j0...
# ANTHROPIC_API_KEY=sk-ant-...

# 3. docker-compose.yml — referencia variables de .env
# environment:
#   POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

### Eines d'Auditoria Automatica

```bash
# gitleaks: eina especialitzada en trobar secrets a repos Git
# Instal·lacio amb Homebrew
brew install gitleaks

# Escaneja el repositori complet (inclosa la historia)
gitleaks detect --source . --verbose

# Escaneja nomes els fitxers actuals (sense historia)
gitleaks detect --source . --no-git --verbose

# trufflehog: alternativa que busca secrets amb alta entropia
# (strings molt aleatoris que probablement son claus)
brew install trufflehog
trufflehog git file://. --only-verified
```

---

## Activitat

### 1. Escaneig Manual del Codi (20 min)

Executa les cerques manuals:

```bash
# Busca paraules clau relacionades amb secrets
grep -rn "password\|secret\|api_key\|token\|credential" \
  --include="*.java" --include="*.py" --include="*.yml" \
  --include="*.properties" --include="*.toml" .

# Busca URLs amb credencials incrustades
# Patro: protocol://user:password@host
grep -rn "://.*:.*@" --include="*.java" --include="*.py" \
  --include="*.yml" --include="*.properties" .

# Busca fitxers .env que NO haurien d'estar comitejats
git ls-files | grep -i "\.env$\|\.env\."
```

Per a cada resultat, classifica:
- **Fals positiu**: no es un secret real (ex: `password` com a nom de camp).
- **Referencia a variable**: correcte (ex: `os.environ["PASSWORD"]`).
- **Secret hardcodejat**: cal corregir immediatament.

### 2. Escaneig de la Historia de Git (20 min)

```bash
# Busca secrets a tota la historia del repositori
git log -p --all -S 'password' -- '*.java' '*.py' '*.yml' '*.properties'
git log -p --all -S 'secret'
git log -p --all -S 'sk-ant'
git log -p --all -S 'sk-'

# Si trobes un secret:
# 1. Considera'l COMPROMÈS (algú podria haver-lo vist)
# 2. Rota'l immediatament (genera una nova clau)
# 3. Opcional: reescriu la historia amb git-filter-repo
#    (complicat, no sempre necessari si la clau ja esta rotada)
```

### 3. Verificar docker-compose.yml (15 min)

Revisa el `docker-compose.yml` linia per linia:

```bash
# Mostra totes les variables d'entorn definides al compose
grep -n "environment" -A 20 docker-compose.yml
```

Assegura't que:
- Totes les contrasenyes venen de `${VARIABLE}`, no estan hardcodejades.
- El fitxer `.env` esta al `.gitignore`.
- Existeix un `.env.example` amb valors de plantilla.

### 4. Crear .env.example (15 min)

Si no existeix, crea `/.env.example`:

```bash
# .env.example — Plantilla de variables d'entorn
# Copia aquest fitxer com a .env i omple els valors reals
# IMPORTANT: mai comitejar el fitxer .env amb valors reals

# Base de dades PostgreSQL
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=esportspulse
POSTGRES_USER=esportspulse
POSTGRES_PASSWORD=canvia_aquest_valor

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=canvia_aquest_valor

# RabbitMQ
RABBITMQ_HOST=localhost
RABBITMQ_PORT=5672
RABBITMQ_USER=esportspulse
RABBITMQ_PASSWORD=canvia_aquest_valor

# JWT
JWT_SECRET=genera_un_secret_aleatori_de_minim_32_caracters

# Anthropic (Claude API)
ANTHROPIC_API_KEY=la_teva_api_key_aqui

# LangFuse (observabilitat)
LANGFUSE_PUBLIC_KEY=la_teva_public_key
LANGFUSE_SECRET_KEY=la_teva_secret_key
LANGFUSE_HOST=https://cloud.langfuse.com
```

### 5. Verificar .gitignore (10 min)

```bash
# Comprova que .env esta al gitignore
grep "\.env" .gitignore

# Si no hi es, afegeix-lo
echo ".env" >> .gitignore
echo ".env.local" >> .gitignore
echo ".env.production" >> .gitignore
```

### 6. Escaneig Automatic amb gitleaks (15 min)

```bash
# Instal·la gitleaks si no el tens
brew install gitleaks

# Escaneja el repositori complet
gitleaks detect --source . --verbose --report-path gitleaks-report.json

# Revisa el report
cat gitleaks-report.json | python3 -m json.tool
```

### 7. Moure Secrets Restants a Variables d'Entorn (20 min)

Per a cada secret trobat, mou-lo a variable d'entorn:

```java
// ABANS (MAL): secret hardcodejat al codi Java
// private static final String JWT_SECRET = "elMeuSecretSuperSegur";

// DESPRES (BE): llegir de variable d'entorn
// La anotacio @Value injecta el valor de la variable d'entorn
// El valor despres dels dos punts es el valor per defecte (nomes per dev)
@Value("${JWT_SECRET:default-dev-secret}")
private String jwtSecret;
```

```python
# ABANS (MAL): secret hardcodejat al codi Python
# ANTHROPIC_API_KEY = "sk-ant-abc123..."

# DESPRES (BE): llegir de variable d'entorn
# os.environ.get() retorna None si la variable no existeix
# Millor usar os.environ[] que llenca error si falta (fail fast)
import os
ANTHROPIC_API_KEY = os.environ["ANTHROPIC_API_KEY"]
```

### 8. Commit (10 min)

```bash
git add .env.example .gitignore
git add -u  # Afegeix els fitxers modificats (secrets moguts a env vars)
git commit -m "security: complete pre-deployment secrets audit and cleanup"
```

---

## Checklist de Lliurament

- [ ] Escaneig manual completat (grep de paraules clau)
- [ ] Historia de Git revisada (git log -S)
- [ ] docker-compose.yml sense secrets hardcodejats
- [ ] `.env.example` creat amb tots els valors necessaris
- [ ] `.env` al `.gitignore`
- [ ] gitleaks executat sense troballes critiques
- [ ] Tots els secrets moguts a variables d'entorn
- [ ] Credencials per defecte canviades (RabbitMQ guest/guest!)
- [ ] Commit amb els canvis de seguretat
