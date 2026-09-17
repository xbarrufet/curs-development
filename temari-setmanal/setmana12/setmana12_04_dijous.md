# Setmana 12 — Dijous: Protegir FastAPI amb JWT i Redis Sessions

## Objectiu del Dia

Afegir autenticacio JWT al servei Python FastAPI d'EsportsPulse. Implementar un sistema d'invalidacio de tokens amb Redis (token blacklist). Al final del dia, els endpoints Python estaran protegits amb el mateix mecanisme que el backend Java, i podras invalidar tokens quan un usuari fa logout.

---

## Teoria

### FastAPI Security: Un Enfocament Diferent

A Spring Security, la configuracio es centralitzada (SecurityFilterChain, filtres). A FastAPI, la seguretat es basa en **dependency injection** — cada endpoint declara les seves dependecies, incloent l'autenticacio:

```python
# Spring Boot: anotacio a nivell de classe/metode
@PreAuthorize("hasRole('ADMIN')")
@PostMapping("/champions")
public Champion create(...) { ... }

# FastAPI: dependencia injectada com a parametre
@app.post("/champions")
async def create(champion: Champion, user: User = Depends(get_current_user)):
    if user.role != "ADMIN":
        raise HTTPException(status_code=403)
    ...
```

**Avantatges del patro FastAPI:**
- Cada endpoint declara explicitament que necessita (mes transparent)
- Les dependencies es poden composar (una depende de l'altra)
- Es facil testejar: nomes has de substituir la dependencia al test

**Avantatges del patro Spring:**
- Configuracio centralitzada (un sol fitxer per veure tota la seguretat)
- Menys codi repetitiu als controllers
- El framework s'encarrega de molts detalls automaticament

### Per Que JWT Sol No Es Suficient: El Problema del Logout

JWT te un defecte intrinsec: **un cop emès, no es pot revocar**. El token es valid fins que caduca, independentment del que passi:

```
1. Usuari fa login  → Rep token (valid 1 hora)
2. Usuari fa logout → El client esborra el token
3. Pero... si algu ha copiat el token, encara pot usar-lo durant 59 minuts!
```

**Solucio: Token Blacklist amb Redis**

Redis es una base de dades en memoria ultra-rapida (ja la vam veure conceptualment a S9 amb Docker). L'usarem per mantenir una llista de tokens invalidats:

```
Logout → Afegim el token a Redis amb TTL = temps restant del token
Peticio → Validem JWT + Comprovem que NO esta a la blacklist de Redis
```

**Per que Redis i no PostgreSQL?**
- Cada peticio ha de consultar la blacklist — necessitem latencia minima
- Redis opera en memoria (microsegons vs mil·lisegons de PostgreSQL)
- El TTL automatic de Redis elimina tokens caducats sense necessitat de jobs de neteja

> **Lectura recomanada (no bloquejant):**
> - [FastAPI Security](https://fastapi.tiangolo.com/tutorial/security/) — Tutorial oficial
> - [python-jose Documentation](https://python-jose.readthedocs.io/)

---

## Activitat

### 1. Afegir Redis al docker-compose (10 min)

Actualitza el `docker-compose.yml` del projecte (connexio amb S9 Docker):

```yaml
# fitxer: docker-compose.yml
# Afegir el servei Redis sota els serveis existents (postgres, qdrant)

services:
  # ... serveis existents (postgres, qdrant) ...

  redis:
    image: redis:7-alpine          # Imatge lleugera de Redis
    container_name: esportspulse-redis
    ports:
      - "6379:6379"                # Port per defecte de Redis
    volumes:
      - redis-data:/data           # Persistencia de dades
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
    restart: unless-stopped

volumes:
  # ... volums existents ...
  redis-data:                      # Volum per les dades de Redis
```

Arrenca Redis:
```bash
docker compose up -d redis
# Verifica que funciona
docker exec esportspulse-redis redis-cli ping
# Resposta esperada: PONG
```

### 2. Instal·lar Dependencies Python (5 min)

```bash
# Afegir les noves dependencies al requirements.txt
cd esportspulse-engine/ai-python

# python-jose: crear i validar JWTs (equivalent de JJWT a Java)
# passlib[bcrypt]: hasheja contrasenyes amb BCrypt (equivalent de Spring BCryptPasswordEncoder)
# redis: client Redis per Python
pip install "python-jose[cryptography]" "passlib[bcrypt]" redis

# Actualitza requirements.txt
pip freeze > requirements.txt
```

### 3. Configuracio de Seguretat Python (10 min)

```python
# fitxer: ai-python/src/config/security_config.py
# Centralitza tota la configuracio de seguretat en un sol lloc.
# Les claus venen de variables d'entorn (com a Java amb @Value).

import os

# La mateixa clau secreta que el backend Java — aixi els dos serveis
# poden validar tokens emesos per l'altre (Single Sign-On bàsic)
JWT_SECRET = os.getenv("JWT_SECRET", "clau-secreta-de-desenvolupament-que-te-32-chars-minim")
JWT_ALGORITHM = "HS256"
JWT_EXPIRATION_MINUTES = 60

# Configuracio de Redis
REDIS_HOST = os.getenv("REDIS_HOST", "localhost")
REDIS_PORT = int(os.getenv("REDIS_PORT", "6379"))
REDIS_DB = int(os.getenv("REDIS_DB", "0"))
```

### 4. Servei JWT per Python (15 min)

```python
# fitxer: ai-python/src/security/jwt_service.py
# Equivalent del JwtService.java — crea i valida tokens JWT.
# Usa python-jose en comptes de JJWT.

from datetime import datetime, timedelta, timezone
from jose import jwt, JWTError
from config.security_config import JWT_SECRET, JWT_ALGORITHM, JWT_EXPIRATION_MINUTES


def create_token(username: str, role: str) -> str:
    """Crea un JWT amb username i rol.
    Estructura identica a la del backend Java perque siguin intercanviables."""
    now = datetime.now(timezone.utc)
    payload = {
        "sub": username,                                    # Subject: qui es l'usuari
        "role": role,                                       # Claim personalitzat: rol
        "iat": now,                                         # Issued At: quan s'ha creat
        "exp": now + timedelta(minutes=JWT_EXPIRATION_MINUTES),  # Expiracio
    }
    # Signa amb la mateixa clau i algorisme que Java
    return jwt.encode(payload, JWT_SECRET, algorithm=JWT_ALGORITHM)


def decode_token(token: str) -> dict:
    """Decodifica i valida un JWT. Llanca JWTError si es invalid o caducat.
    python-jose verifica automaticament la signatura i l'expiracio."""
    try:
        payload = jwt.decode(token, JWT_SECRET, algorithms=[JWT_ALGORITHM])
        return payload
    except JWTError as e:
        raise ValueError(f"Token invalid: {e}")


def extract_username(token: str) -> str:
    """Extreu el username (subject) del token."""
    return decode_token(token)["sub"]


def extract_role(token: str) -> str:
    """Extreu el rol de l'usuari del token."""
    return decode_token(token)["role"]
```

### 5. Token Blacklist amb Redis (15 min)

```python
# fitxer: ai-python/src/security/token_blacklist.py
# Gestiona la llista de tokens invalidats (logout).
# Usa Redis amb TTL automatic — els tokens caducats s'esborren sols.

import redis
from jose import jwt
from datetime import datetime, timezone
from config.security_config import REDIS_HOST, REDIS_PORT, REDIS_DB, JWT_SECRET, JWT_ALGORITHM

# Connexio a Redis (connection pooling automatic)
redis_client = redis.Redis(
    host=REDIS_HOST,
    port=REDIS_PORT,
    db=REDIS_DB,
    decode_responses=True  # Retorna strings en comptes de bytes
)


def blacklist_token(token: str) -> None:
    """Afegeix un token a la blacklist de Redis.
    El TTL es calcula a partir de l'expiracio del token —
    no cal guardar-lo mes temps del que seria valid."""
    try:
        # Decodifica sense verificar expiracio (el token pot estar a punt de caducar)
        payload = jwt.decode(
            token, JWT_SECRET, algorithms=[JWT_ALGORITHM],
            options={"verify_exp": False}
        )
        exp = payload.get("exp", 0)
        now = int(datetime.now(timezone.utc).timestamp())
        ttl = exp - now  # Segons fins que el token caduca

        if ttl > 0:
            # Guarda el token a Redis amb TTL automatic
            # Clau: "blacklist:{token}" — Valor: "1" (qualsevol valor, nomes importa l'existencia)
            redis_client.setex(f"blacklist:{token}", ttl, "1")
    except Exception:
        # Si no podem parsejar el token, el posem amb TTL de 1 hora (seguretat)
        redis_client.setex(f"blacklist:{token}", 3600, "1")


def is_blacklisted(token: str) -> bool:
    """Comprova si un token esta a la blacklist.
    O(1) a Redis — no ralenteix les peticions."""
    return redis_client.exists(f"blacklist:{token}") > 0
```

### 6. Dependencia d'Autenticacio FastAPI (20 min)

```python
# fitxer: ai-python/src/security/auth_dependency.py
# Dependencia FastAPI per autenticar peticions.
# Equivalent del JwtAuthenticationFilter de Spring, pero amb Depends().

from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from pydantic import BaseModel
from security.jwt_service import decode_token
from security.token_blacklist import is_blacklisted

# Esquema de seguretat: extreu el token de la capcalera "Authorization: Bearer ..."
# FastAPI genera automaticament la documentacio Swagger amb el boto "Authorize"
bearer_scheme = HTTPBearer()


class CurrentUser(BaseModel):
    """Model que representa l'usuari autenticat actual."""
    username: str
    role: str


async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(bearer_scheme)
) -> CurrentUser:
    """Dependencia que extreu i valida l'usuari del JWT.

    Flux:
    1. HTTPBearer extreu el token de la capcalera Authorization
    2. Comprovem que el token no esta a la blacklist (logout)
    3. Decodifiquem el JWT i extraiem username i rol
    4. Retornem un objecte CurrentUser

    Si qualsevol pas falla, retorna 401 Unauthorized.
    """
    token = credentials.credentials

    # Comprova la blacklist (logout) ABANS de validar el token
    # Aixi evitem processar tokens que sabem que son invalids
    if is_blacklisted(token):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token invalidat (sessio tancada)",
            headers={"WWW-Authenticate": "Bearer"},
        )

    try:
        payload = decode_token(token)
        username = payload.get("sub")
        role = payload.get("role")

        if username is None or role is None:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Token amb format invalid",
            )

        return CurrentUser(username=username, role=role)

    except ValueError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token invalid o caducat",
            headers={"WWW-Authenticate": "Bearer"},
        )


def require_role(required_role: str):
    """Factory de dependencia per exigir un rol especific.

    Us:
        @app.post("/admin-only")
        async def admin_endpoint(user: CurrentUser = Depends(require_role("ADMIN"))):
            ...

    Compara amb Spring: @PreAuthorize("hasRole('ADMIN')")
    """
    async def role_checker(
        user: CurrentUser = Depends(get_current_user)
    ) -> CurrentUser:
        if user.role != required_role:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Es requereix rol {required_role}. El teu rol es {user.role}.",
            )
        return user
    return role_checker
```

### 7. Protegir els Endpoints FastAPI (20 min)

```python
# fitxer: ai-python/src/main.py
# Actualitzar l'aplicacio FastAPI per integrar autenticacio.
# Els endpoints existents (S10) ara requereixen JWT.

from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
from passlib.hash import bcrypt
from pydantic import BaseModel, Field

from security.jwt_service import create_token, extract_username
from security.auth_dependency import get_current_user, require_role, CurrentUser
from security.token_blacklist import blacklist_token

app = FastAPI(
    title="EsportsPulse AI Service",
    description="Servei Python amb autenticacio JWT",
    version="0.12.0",
)

# --- Models Pydantic per auth (connexio amb S10 Pydantic) ---

class UserRegister(BaseModel):
    """Request body per registrar un usuari."""
    username: str = Field(min_length=3, max_length=50)
    password: str = Field(min_length=8)

class UserLogin(BaseModel):
    """Request body per fer login."""
    username: str
    password: str

class TokenResponse(BaseModel):
    """Resposta amb el JWT."""
    token: str
    username: str
    role: str

# Simulacio de base de dades d'usuaris (en produccio, compartiries la BD amb Java)
# Per a aquest exercici, guardem en memoria
fake_users_db: dict[str, dict] = {}

bearer_scheme = HTTPBearer()


# --- Endpoints d'autenticacio (publics) ---

@app.post("/auth/register", response_model=TokenResponse, status_code=201)
async def register(request: UserRegister):
    """Registra un nou usuari amb contrasenya hashejada."""
    if request.username in fake_users_db:
        raise HTTPException(status_code=409, detail="Username ja existeix")

    # Hasheja la contrasenya amb BCrypt (mateixa funcio que Spring)
    hashed = bcrypt.hash(request.password)

    fake_users_db[request.username] = {
        "username": request.username,
        "password": hashed,
        "role": "USER",
    }

    token = create_token(request.username, "USER")
    return TokenResponse(token=token, username=request.username, role="USER")


@app.post("/auth/login", response_model=TokenResponse)
async def login(request: UserLogin):
    """Login amb username i contrasenya."""
    user = fake_users_db.get(request.username)

    # Mateixa practica que a Java: no revelem si l'error es al username o password
    if user is None or not bcrypt.verify(request.password, user["password"]):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Credencials invalides",
        )

    token = create_token(user["username"], user["role"])
    return TokenResponse(token=token, username=user["username"], role=user["role"])


@app.post("/auth/logout")
async def logout(credentials: HTTPAuthorizationCredentials = Depends(bearer_scheme)):
    """Logout: afegeix el token actual a la blacklist de Redis.
    El token queda invalidat fins que caduca naturalment."""
    token = credentials.credentials
    blacklist_token(token)
    return {"message": "Sessio tancada correctament"}


# --- Endpoints protegits ---

@app.get("/api/ai/health")
async def health():
    """Health check public (no requereix auth)."""
    return {"status": "healthy", "service": "ai-python"}


@app.get("/api/ai/analysis")
async def get_analysis(user: CurrentUser = Depends(get_current_user)):
    """Endpoint protegit — qualsevol usuari autenticat."""
    return {
        "message": f"Hola {user.username}, aqui tens l'analisi",
        "user_role": user.role,
    }


@app.post("/api/ai/models/train")
async def train_model(user: CurrentUser = Depends(require_role("ADMIN"))):
    """Entrenar un model — nomes ADMIN.
    Compara amb @PreAuthorize('hasRole(ADMIN)') de Spring."""
    return {
        "message": "Entrenant model... (nomes ADMINs poden fer aixo)",
        "triggered_by": user.username,
    }


@app.get("/api/ai/me")
async def who_am_i(user: CurrentUser = Depends(get_current_user)):
    """Retorna informacio de l'usuari autenticat (util per depurar)."""
    return {"username": user.username, "role": user.role}
```

### 8. Proves Completes del Flux (20 min)

```bash
# Arrenca el servei Python
cd esportspulse-engine/ai-python
uvicorn src.main:app --reload --port 8000

# === FLUX COMPLET ===

# 1. Registrar un usuari
curl -s -X POST http://localhost:8000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"anna","password":"anna-pass-123"}' | jq .

# 2. Login
PYTHON_TOKEN=$(curl -s -X POST http://localhost:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"anna","password":"anna-pass-123"}' | jq -r .token)
echo "Token: $PYTHON_TOKEN"

# 3. Accedir a endpoint protegit AMB token (200 OK)
curl -s -H "Authorization: Bearer $PYTHON_TOKEN" \
  http://localhost:8000/api/ai/analysis | jq .

# 4. Accedir SENSE token (401 Unauthorized)
curl -s -w "\nHTTP %{http_code}\n" http://localhost:8000/api/ai/analysis

# 5. Intentar accedir a endpoint ADMIN amb rol USER (403 Forbidden)
curl -s -w "\nHTTP %{http_code}\n" \
  -X POST http://localhost:8000/api/ai/models/train \
  -H "Authorization: Bearer $PYTHON_TOKEN"

# 6. Verificar identitat
curl -s -H "Authorization: Bearer $PYTHON_TOKEN" \
  http://localhost:8000/api/ai/me | jq .

# === PROVA DE LOGOUT (BLACKLIST) ===

# 7. Logout (afegeix token a Redis blacklist)
curl -s -X POST http://localhost:8000/auth/logout \
  -H "Authorization: Bearer $PYTHON_TOKEN" | jq .

# 8. Intentar usar el MATEIX token despres del logout (401 Unauthorized)
curl -s -w "\nHTTP %{http_code}\n" \
  -H "Authorization: Bearer $PYTHON_TOKEN" \
  http://localhost:8000/api/ai/analysis
# Esperat: 401 — el token esta a la blacklist!

# 9. Verificar a Redis que el token esta a la blacklist
docker exec esportspulse-redis redis-cli keys "blacklist:*"
# Ha de mostrar almenys una clau

# 10. Verificar el TTL del token a Redis
docker exec esportspulse-redis redis-cli ttl "blacklist:$PYTHON_TOKEN"
# Ha de mostrar els segons restants fins l'expiracio

# === SWAGGER UI ===
# Obre http://localhost:8000/docs al navegador
# Veuràs el boto "Authorize" per provar endpoints amb JWT directament
```

### 9. Comparacio Spring vs FastAPI (Reflexio)

| Aspecte | Spring Security | FastAPI |
|---|---|---|
| Configuracio | Centralitzada (SecurityConfig) | Distribuida (Depends a cada endpoint) |
| Filtres | FilterChain (Servlet level) | Middleware o Depends |
| Rols | @PreAuthorize("hasRole()") | require_role() dependency |
| Password hash | BCryptPasswordEncoder bean | passlib.hash.bcrypt |
| JWT | JJWT (io.jsonwebtoken) | python-jose |
| Token revocation | Necessita implementacio manual | Redis blacklist |

---

## Checklist de Lliurament

- [ ] Redis funciona al docker-compose (`redis-cli ping` retorna PONG)
- [ ] `POST /auth/register` crea un usuari i retorna JWT al servei Python
- [ ] `POST /auth/login` valida credencials i retorna JWT
- [ ] `GET /api/ai/analysis` requereix token valid (401 sense token)
- [ ] `POST /api/ai/models/train` requereix rol ADMIN (403 amb rol USER)
- [ ] `POST /auth/logout` invalida el token (verificat amb peticio posterior)
- [ ] El token invalidat apareix a Redis amb TTL correcte
- [ ] Swagger UI (`/docs`) mostra el boto "Authorize" per provar amb JWT
- [ ] Commit: `feat(auth): add JWT authentication and Redis token blacklist to FastAPI service`
