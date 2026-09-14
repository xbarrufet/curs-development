# Setmana 22 — Divendres: Verificacio en Produccio

## Objectiu del Dia

Fer smoke testing de l'aplicacio desplegada, revisar logs al nuvol, i documentar tot el proces de desplegament. Al final del dia, has de tenir confianca que el sistema funciona correctament en produccio i un document que permeti reproduir el desplegament des de zero.

---

## Teoria

### Que es Smoke Testing?

El smoke testing es una verificacio rapida dels fluxos principals. No es un test exhaustiu, sino una comprovacio que "no surt fum" (d'aqui el nom, originari de l'electronica).

```
# Smoke tests per a EsportsPulse:
# 1. L'aplicacio arrenca? (health check OK)
# 2. Es pot fer login? (autenticacio funciona)
# 3. Es pot cercar un equip? (backend + BD funcionen)
# 4. Es pot fer una pregunta a un agent? (IA funciona)
# 5. El dashboard mostra dades? (frontend + backend integrats)
#
# Si qualsevol d'aquests falla, hi ha un problema critic.
# Si tots passen, el sistema esta "funcionalment operatiu".
```

### Diferencies entre Testing Local i en Produccio

```
# LOCAL:
# - Xarxa interna de Docker (rapida, fiable)
# - Tots els serveis a la mateixa maquina
# - Logs a la terminal (docker-compose logs)
# - Secrets al fitxer .env local

# PRODUCCIO:
# - Xarxa publica (latencia, possibles fallades)
# - Serveis en maquines/contenidors separats
# - Logs a la plataforma (Render dashboard, flyctl logs)
# - Secrets a la configuracio de la plataforma
# - HTTPS obligatori
# - CORS configurat
# - Cold starts (el servei pot trigar a arrencar)
```

### Cold Starts

A les plataformes amb free tier, els serveis s'adormen despres d'un temps d'inactivitat:

```
# Render free tier:
# - El servei s'adorm despres de 15 minuts sense peticions
# - La primera peticio despres d'adormir-se triga 30-60 segons
#   (temps de "cold start" mentre arrenca el contenidor)
# - Despres d'arrencar, les peticions son normals
#
# Fly.io free tier:
# - Comportament similar, configurable amb min_machines_running
#
# Solucions:
# 1. Acceptar el cold start (gratuit, acceptable per a portfolios)
# 2. Configurar un cron job que faci ping cada 10 min (manté despert)
# 3. Pagar pel tier professional (el servei esta sempre actiu)
```

### Monitoritzar Logs al Nuvol

```bash
# RENDER:
# 1. Dashboard web: render.com -> Service -> Logs
# 2. API: curl -H "Authorization: Bearer $RENDER_API_KEY" \
#    https://api.render.com/v1/services/$SERVICE_ID/logs

# FLY.IO:
# flyctl logs --app esportspulse-backend
# flyctl logs --app esportspulse-ai

# Que buscar als logs:
# - Errors d'inici (connexio a BD fallida, variable d'entorn absent)
# - Errors 500 (excepcions no gestionades)
# - Temps de resposta anormals (>5s per a peticions simples)
# - Errors de connexio entre serveis
```

### Runbook de Desplegament

Un runbook es un document que descriu pas a pas com desplegar (o re-desplegar) el sistema. Es essential per:
- Reproduir el desplegament si cal
- Que un company (o tu mateix en 6 mesos) pugui fer-ho
- Respondre a incidents rapidament

```markdown
# Estructura d'un bon runbook:
# 1. Prerequisits (comptes, eines, permisos)
# 2. Pas a pas del desplegament (amb comandes exactes)
# 3. Verificacio (com saber que funciona)
# 4. Rollback (com tornar enrere si falla)
# 5. Troubleshooting (problemes comuns i solucions)
```

---

## Activitat

### 1. Smoke Testing Automatitzat (30 min)

Crea un script de smoke testing que verifici tots els fluxos:

```bash
#!/bin/bash
# smoke-test.sh — Smoke testing de l'aplicacio en produccio
# Executa: bash smoke-test.sh https://api.esportspulse.dev

# URL base del backend (passada com a argument)
BASE_URL="${1:-https://esportspulse-backend.onrender.com}"
AI_URL="${2:-https://esportspulse-ai.onrender.com}"
DASHBOARD_URL="${3:-https://esportspulse-dashboard.onrender.com}"

# Colors per a la sortida
GREEN='\033[0;32m'
RED='\033[0;31m'
NC='\033[0m'  # Sense color

# Comptadors de resultats
PASSED=0
FAILED=0

# Funcio helper per executar un test
run_test() {
    local test_name="$1"    # Nom descriptiu del test
    local url="$2"          # URL a provar
    local expected="$3"     # Codi HTTP esperat (200, 301, etc.)
    
    # Fa la peticio i captura el codi HTTP
    local status_code
    status_code=$(curl -s -o /dev/null -w "%{http_code}" "$url" --max-time 30)
    
    if [ "$status_code" = "$expected" ]; then
        echo -e "${GREEN}PASS${NC} $test_name (HTTP $status_code)"
        ((PASSED++))
    else
        echo -e "${RED}FAIL${NC} $test_name (esperava $expected, rebut $status_code)"
        ((FAILED++))
    fi
}

echo "=== Smoke Testing EsportsPulse ==="
echo "Backend: $BASE_URL"
echo "AI: $AI_URL"
echo "Dashboard: $DASHBOARD_URL"
echo ""

# Test 1: Health check del backend
run_test "Backend health check" "$BASE_URL/health" "200"

# Test 2: Health check del servei AI
run_test "AI service health check" "$AI_URL/health" "200"

# Test 3: Dashboard accessible
run_test "Dashboard accessible" "$DASHBOARD_URL" "200"

# Test 4: Login (POST amb credencials)
echo ""
echo "--- Tests amb autenticacio ---"
LOGIN_RESPONSE=$(curl -s -X POST "$BASE_URL/api/v1/auth/login" \
    -H "Content-Type: application/json" \
    -d '{"username":"test","password":"test123"}' \
    --max-time 30)

# Extrau el token de la resposta JSON
TOKEN=$(echo "$LOGIN_RESPONSE" | python3 -c "
import sys, json
try:
    data = json.load(sys.stdin)
    print(data.get('token', ''))
except:
    print('')
" 2>/dev/null)

if [ -n "$TOKEN" ]; then
    echo -e "${GREEN}PASS${NC} Login funciona (token rebut)"
    ((PASSED++))
    
    # Test 5: Endpoint autenticat
    TEAMS_STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
        -H "Authorization: Bearer $TOKEN" \
        "$BASE_URL/api/v1/teams" --max-time 30)
    
    if [ "$TEAMS_STATUS" = "200" ]; then
        echo -e "${GREEN}PASS${NC} Endpoint autenticat /teams (HTTP $TEAMS_STATUS)"
        ((PASSED++))
    else
        echo -e "${RED}FAIL${NC} Endpoint autenticat /teams (HTTP $TEAMS_STATUS)"
        ((FAILED++))
    fi
else
    echo -e "${RED}FAIL${NC} Login no funciona (no s'ha rebut token)"
    ((FAILED++))
fi

# Resum
echo ""
echo "=== Resultats ==="
echo -e "${GREEN}Passats: $PASSED${NC}"
echo -e "${RED}Fallats: $FAILED${NC}"

# Exit code: 0 si tot passa, 1 si algun falla
[ "$FAILED" -eq 0 ]
```

Executa'l:

```bash
chmod +x smoke-test.sh
./smoke-test.sh
```

### 2. Revisar Logs al Nuvol (20 min)

Ves al dashboard de Render (o usa `flyctl logs`) i revisa:

```bash
# Busca errors al backend Java
# Filtra per "ERROR" o "Exception" als logs

# Busca errors al servei Python
# Filtra per "ERROR" o "Traceback"

# Coses comunes a trobar:
# - "Connection refused" -> un servei no esta llest
# - "401 Unauthorized" -> JWT no es valid entre serveis
# - "CORS error" -> falta el domini als allowed origins
# - "Connection timeout" -> cold start o servei caigut
```

Documenta qualsevol error que trobis i com l'has resolt.

### 3. Escriure el Runbook de Desplegament (30 min)

Crea `docs/runbook-deployment.md`:

```markdown
# Runbook: Desplegament d'EsportsPulse

## Prerequisits
<!-- Comptes necessaris, eines, permisos -->
- Compte a Render/Fly.io
- Repositori GitHub connectat
- Domini configurat (opcional)

## Serveis a Desplegar
<!-- Llista ordenada amb dependencias -->

### 1. PostgreSQL
<!-- Com crear la BD, URL de connexio -->

### 2. Redis
<!-- Opcions: Upstash, Render Redis, etc. -->

### 3. Backend Java
<!-- Passos exactos de desplegament -->

### 4. Servei Python (FastAPI)
<!-- Passos exactos de desplegament -->

### 5. Dashboard (Streamlit)
<!-- Passos exactos de desplegament -->

## Variables d'Entorn per Servei
<!-- Taula completa de variables necessaries per servei -->

## Verificacio Post-Desplegament
<!-- Comandes per verificar que tot funciona -->

## Rollback
<!-- Com tornar a la versio anterior si algo falla -->

## Troubleshooting
### El servei no arrenca
<!-- Possibles causes i solucions -->

### Error de connexio a la BD
<!-- Possibles causes i solucions -->

### Cold start massa lent
<!-- Possibles solucions -->
```

### 4. Actualitzar el README (15 min)

Afegeix una seccio de "Desplegament" al README amb:
- URL de l'aplicacio desplegada
- Instruccions basiques per re-desplegar

### 5. Crear PR de la Setmana (20 min)

```bash
# Assegura que tot esta comitejat
git add smoke-test.sh docs/runbook-deployment.md
git commit -m "docs: add smoke testing script and deployment runbook"

# Crea el PR
gh pr create \
  --title "feat(deploy): cloud deployment with Render/Fly.io" \
  --body "## Resum

Desplegament complet d'EsportsPulse al nuvol.

## Canvis
- Auditoria de seguretat pre-desplegament
- Dockerfiles de produccio (multi-stage build)
- Desplegament de backend Java, servei Python i Streamlit
- Domini personalitzat amb HTTPS
- Smoke testing automatitzat
- Runbook de desplegament

## URLs
- Dashboard: https://app.esportspulse.dev
- API: https://api.esportspulse.dev
- AI: https://ai.esportspulse.dev

## Verificacio
- [ ] Smoke tests passen
- [ ] HTTPS funciona
- [ ] Login + endpoints autenticats funcionen
- [ ] Agents responen correctament"
```

---

## Checklist de Lliurament

- [ ] Script de smoke testing creat i executat
- [ ] Tots els smoke tests passen
- [ ] Logs revisats al nuvol (sense errors critics)
- [ ] Runbook de desplegament escrit amb passos reproduibles
- [ ] README actualitzat amb URLs i instruccions
- [ ] Cold start documentat (temps aproximat)
- [ ] PR creat amb tots els canvis de la setmana
- [ ] Aplicacio accessible publicamente
