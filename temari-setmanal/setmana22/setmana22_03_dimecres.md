# Setmana 22 — Dimecres: Desplegament: Python, Streamlit i Serveis

## Objectiu del Dia

Desplegar la resta de serveis al nuvol: el servei Python (FastAPI), el dashboard Streamlit, i connectar-los amb el backend Java ja desplegat. Al final del dia, tot el stack ha de funcionar al nuvol com funciona en local amb `docker-compose up`.

---

## Teoria

### Arquitectura al Nuvol vs Local

En local, tots els serveis es comuniquen per la xarxa interna de Docker:

```
# LOCAL (docker-compose)
# backend-java:8080  <-->  ai-python:8000
# Comunicacio per nom de servei (DNS intern de Docker)
# URL: http://ai-python:8000/api/v1/agents/query

# NUVOL (serveis independents)
# backend-java.onrender.com  <-->  ai-python.onrender.com
# Comunicacio per URL publica (HTTPS)
# URL: https://ai-python.onrender.com/api/v1/agents/query
```

Aixo implica canvis a la configuracio:

```python
# ABANS (local): URL fixa del servei dins Docker
# BACKEND_URL = "http://backend-java:8080"

# DESPRES (nuvol): URL configurable per variable d'entorn
# Cada servei necessita saber la URL dels altres
import os

# URL del backend Java — diferent en local i en produccio
BACKEND_URL = os.environ.get(
    "BACKEND_URL",           # Variable d'entorn per produccio
    "http://backend-java:8080"  # Valor per defecte per a local
)
```

### Desplegar Multiples Serveis

Quan despleguem mes d'un servei, cal pensar en:

**1. Ordre de desplegament:**

```
# L'ordre importa per les dependencies:
# 1. Base de dades (PostgreSQL) — ja desplegada ahir
# 2. Backend Java — ja desplegat ahir
# 3. Servei Python (FastAPI) — depèn del backend per a dades
# 4. Streamlit — depèn de Python i Java per funcionar
```

**2. Descobriment de serveis (Service Discovery):**

```yaml
# En local, Docker Compose fa el descobriment automaticament.
# Al nuvol, hem de configurar les URLs manualment.

# Variables d'entorn per a cada servei:

# Servei Python necessita saber:
# BACKEND_JAVA_URL=https://esportspulse-backend.onrender.com
# REDIS_URL=redis://...
# RABBITMQ_URL=amqp://...
# ANTHROPIC_API_KEY=sk-ant-...

# Streamlit necessita saber:
# BACKEND_JAVA_URL=https://esportspulse-backend.onrender.com
# AI_PYTHON_URL=https://esportspulse-ai.onrender.com
```

**3. Serveis externs (Redis, RabbitMQ):**

```
# Opcions per a Redis al nuvol:
# - Render Redis: integrat, facil, free tier limitat
# - Upstash: Redis serverless, generous free tier
# - Redis Cloud: free tier de 30MB

# Opcions per a RabbitMQ al nuvol:
# - CloudAMQP: free tier (Little Lemur)
# - Si no es crític, podem desactivar-lo temporalment
#   i usar comunicacio sincrona (REST directe)
```

### Dockerfile per a Python (FastAPI)

```dockerfile
# Dockerfile per a produccio del servei Python/FastAPI

# Imatge base lleugera de Python
FROM python:3.12-slim

# Directori de treball
WORKDIR /app

# Copia els requisits primer (cache de dependencies)
# Si requirements.txt no canvia, Docker reutilitza aquesta capa
COPY requirements.txt .

# Instal·la les dependencies sense cache (redueix mida)
# --no-cache-dir evita guardar els fitxers descarregats
RUN pip install --no-cache-dir -r requirements.txt

# Copia el codi font
COPY src/ ./src/

# Port que exposa FastAPI
EXPOSE 8000

# Executar amb uvicorn (servidor ASGI per a produccio)
# --host 0.0.0.0: accepta connexions de qualsevol IP
# --port: llegeix del PORT de la variable d'entorn o 8000
# --workers 2: dos processos per gestionar peticions en paral·lel
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "2"]
```

### Dockerfile per a Streamlit

```dockerfile
# Dockerfile per a produccio del dashboard Streamlit

FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copia tot el codi del dashboard (incloent pages/)
COPY . .

# Streamlit usa el port 8501 per defecte
EXPOSE 8501

# Configuracio de Streamlit per a produccio:
# --server.address=0.0.0.0: accepta connexions externes
# --server.port: port del servei
# --server.headless=true: no intenta obrir el navegador
# --browser.gatherUsageStats=false: no envia telemetria
CMD ["streamlit", "run", "app.py", \
     "--server.address=0.0.0.0", \
     "--server.port=8501", \
     "--server.headless=true", \
     "--browser.gatherUsageStats=false"]
```

---

## Activitat

### 1. Preparar el Servei Python per al Nuvol (20 min)

Actualitza la configuracio perque les URLs siguin configurables:

```python
# config.py — Configuracio centralitzada del servei Python
import os

class Settings:
    """Configuracio del servei AI Python.
    
    Totes les URLs i credencials es llegeixen de variables d'entorn.
    Els valors per defecte son per a desenvolupament local.
    """
    
    # URL del backend Java
    BACKEND_URL: str = os.environ.get(
        "BACKEND_URL", "http://backend-java:8080"
    )
    
    # Connexio a Redis
    REDIS_URL: str = os.environ.get(
        "REDIS_URL", "redis://redis:6379"
    )
    
    # API Key d'Anthropic (obligatoria, sense valor per defecte)
    ANTHROPIC_API_KEY: str = os.environ["ANTHROPIC_API_KEY"]
    
    # Entorn (dev/prod) — per ajustar nivells de logging
    ENVIRONMENT: str = os.environ.get("ENVIRONMENT", "development")

# Instancia global de configuracio
settings = Settings()
```

### 2. Desplegar el Servei Python (30 min)

**A Render:**

1. Crea un "New Web Service" a Render.
2. Configura:

```
Name: esportspulse-ai
Environment: Docker
Region: Frankfurt (mateixa regio que el backend)
Root Directory: ai-python

# Variables d'entorn:
BACKEND_URL=https://esportspulse-backend.onrender.com
ANTHROPIC_API_KEY=(la teva clau)
REDIS_URL=(URL del Redis al nuvol, si en tens)
ENVIRONMENT=production
```

3. Verifica:

```bash
AI_URL="https://esportspulse-ai.onrender.com"

# Health check
curl -s $AI_URL/health | python3 -m json.tool

# Test d'un endpoint basic
curl -s $AI_URL/api/v1/agents/status | python3 -m json.tool
```

### 3. Desplegar Streamlit (30 min)

1. Crea un altre "New Web Service" a Render per a Streamlit.
2. Configura:

```
Name: esportspulse-dashboard
Environment: Docker
Root Directory: streamlit

# Variables d'entorn:
BACKEND_URL=https://esportspulse-backend.onrender.com
AI_PYTHON_URL=https://esportspulse-ai.onrender.com
```

3. Verifica obrint la URL al navegador.

### 4. Configurar Redis al Nuvol (20 min)

Si els agents necessiten Redis (cache, sessions):

```bash
# Opcio 1: Upstash (recomanat per free tier generós)
# 1. Crea compte a upstash.com
# 2. Crea una base de dades Redis
# 3. Copia la URL de connexio
# Format: rediss://default:password@host:port

# Opcio 2: Render Redis (si ja estàs a Render)
# 1. Crea un "New Redis" al dashboard de Render
# 2. Copia la Internal URL (si es a la mateixa regio)
#    o la External URL

# Actualitza la variable d'entorn del servei Python:
# REDIS_URL=rediss://default:abc123@eu1-curious-mouse-12345.upstash.io:6379
```

### 5. Verificar la Connexio Completa (20 min)

Prova el flux complet des del navegador:

```bash
# 1. Obre el dashboard
open "https://esportspulse-dashboard.onrender.com"

# 2. Inicia sessio (si te login)
# 3. Fes una consulta a l'agent quantitatiu
# 4. Fes una consulta a l'agent knowledge
# 5. Verifica que les dades es mostren correctament

# Des de la terminal, verifica els logs:
# A Render: Dashboard -> Service -> Logs
# A Fly.io: flyctl logs --app esportspulse-ai
```

### 6. Gestionar la Comunicacio Sincrona/Asincrona (15 min)

Si RabbitMQ no esta disponible al nuvol, implementa un fallback:

```python
# message_service.py — Servei de missatgeria amb fallback

import os
import logging

logger = logging.getLogger(__name__)

class MessageService:
    """Gestiona la comunicacio entre serveis.
    
    En produccio, si RabbitMQ no esta disponible,
    fa fallback a comunicacio sincrona via HTTP.
    """
    
    def __init__(self):
        self.rabbitmq_url = os.environ.get("RABBITMQ_URL")
        # Si tenim RabbitMQ, usem-lo; si no, usem HTTP directe
        self.use_async = self.rabbitmq_url is not None
        
        if self.use_async:
            logger.info("Usant RabbitMQ per a comunicacio asincrona")
        else:
            logger.info("RabbitMQ no disponible, usant HTTP sincron")
    
    async def send_to_backend(self, endpoint: str, data: dict):
        """Envia dades al backend Java."""
        if self.use_async:
            # Publicar missatge a la cua de RabbitMQ
            await self._publish_to_queue(endpoint, data)
        else:
            # Fer peticio HTTP directa al backend
            await self._http_request(endpoint, data)
```

### 7. Commit (10 min)

```bash
git add ai-python/Dockerfile ai-python/src/config.py
git add streamlit/Dockerfile
git commit -m "feat(deploy): deploy Python service and Streamlit dashboard to cloud"
```

---

## Checklist de Lliurament

- [ ] Servei Python desplegat i responent al health check
- [ ] Dashboard Streamlit accessible per URL publica
- [ ] Connexio entre Streamlit i Python verificada
- [ ] Connexio entre Python i Backend Java verificada
- [ ] Redis al nuvol configurat (o fallback implementat)
- [ ] Flux complet funciona: login -> consulta -> resposta
- [ ] Logs accessibles a la plataforma
- [ ] Variables d'entorn configurades a cada servei
- [ ] Commit amb els Dockerfiles i configuracio
