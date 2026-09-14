# Setmana 12 — Dilluns: Per Que Necessites Autenticacio i Com Funciona

## Objectiu del Dia

Entendre per que cal protegir una API, la diferencia entre autenticacio i autoritzacio, i com funciona JWT com a mecanisme d'autenticacio stateless. Al final del dia has de poder crear, decodificar i validar un JWT manualment, i entendre el flux complet login -> token -> accedir a recurs protegit.

---

## Teoria

### El Problema: La Teva API Esta Oberta al Mon

Ara mateix, qualsevol persona que conegui la URL del teu backend pot fer peticions sense cap restriccio:

```bash
# Qualsevol pot fer aixo — no hi ha cap barrera
curl http://localhost:8080/api/champions
curl -X POST http://localhost:8080/api/champions -d '{"name":"Faker","role":"mid"}'
curl -X DELETE http://localhost:8080/api/champions/1
```

**Consequencies reals d'una API desprotegida:**
- **Fuga de dades:** qualsevol pot llegir informacio sensible (dades d'usuaris, estadistiques privades)
- **Manipulacio:** qualsevol pot crear, modificar o eliminar registres
- **Abussos:** bots poden saturar el servei amb peticions massives
- **Responsabilitat legal:** el GDPR (Europa) obliga a protegir dades personals amb sancions de fins al 4% de la facturacio

> **Cas real:** El 2019, Facebook va exposar 540 milions de registres d'usuaris per una API mal protegida. No era un atac sofisticat — simplement no hi havia autenticacio als endpoints.

### Autenticacio vs Autoritzacio

Dos conceptes que es confonen constantment pero son radicalment diferents:

| Concepte | Pregunta | Exemple |
|---|---|---|
| **Autenticacio** (AuthN) | Qui ets? | Login amb usuari i contrasenya |
| **Autoritzacio** (AuthZ) | Que pots fer? | Un USER pot llegir, un ADMIN pot esborrar |

```
Flux complet:
1. L'usuari fa login (autenticacio)          → "Soc en Pere"
2. El servidor verifica la identitat          → "Si, ets en Pere"
3. El servidor comprova permisos (autoritz.)  → "En Pere te rol USER, pot llegir pero no esborrar"
4. El servidor respon segons els permisos     → 200 OK o 403 Forbidden
```

**Analogia:** Autenticacio es el DNI que portes. Autoritzacio es el que pots fer amb ell (votar, conduir, comprar alcohol). Tenir DNI no et dona tots els drets — depenen de la teva edat, llicencia, etc.

### HTTP es Stateless — I Aixo es un Problema

HTTP no te memoria. Cada peticio es independent: el servidor no recorda qui ets d'una peticio a la seguent.

```
Peticio 1: GET /api/champions    → El servidor no sap qui ets
Peticio 2: GET /api/champions    → El servidor TAMPOC sap qui ets
                                   (no recorda la Peticio 1)
```

**Dues solucions historiques:**

**1. Sessions (server-side state)**
```
Login → Servidor crea una sessio (ID unic) → Guarda a memoria del servidor
      → Envia cookie amb session_id al client
Peticions posteriors → Client envia cookie → Servidor busca la sessio a memoria
```
- Problema: el servidor ha de guardar totes les sessions. Amb 10.000 usuaris, son 10.000 objectes en memoria.
- Problema: si tens 3 servidors (load balancer), la sessio nomes existeix a UN d'ells.

**2. Tokens (client-side state) — JWT**
```
Login → Servidor crea un token signat → L'envia al client
Peticions posteriors → Client envia el token a cada peticio → Servidor el VALIDA (no el busca en memoria)
```
- Avantatge: el servidor no guarda res. Escala infinitament.
- Avantatge: qualsevol servidor amb la clau secreta pot validar el token.

### JWT (JSON Web Token) Explicat

Un JWT te tres parts separades per punts:

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJwZXJlIiwicm9sZSI6IlVTRVIiLCJpYXQiOjE3MjYwMDAwMDAsImV4cCI6MTcyNjAwMzYwMH0.abc123signature
|_______ HEADER _______|________________________ PAYLOAD _________________________|___ SIGNATURE ___|
```

**1. Header** — Algorisme de signatura:
```json
{
  "alg": "HS256",   // Algorisme: HMAC amb SHA-256
  "typ": "JWT"      // Tipus de token
}
```

**2. Payload** — Dades (claims):
```json
{
  "sub": "pere",           // Subject: identificador de l'usuari
  "role": "USER",          // Claim personalitzat: rol de l'usuari
  "iat": 1726000000,       // Issued At: quan es va crear (Unix timestamp)
  "exp": 1726003600        // Expiration: quan caduca (1 hora despres)
}
```

**3. Signature** — Garantia d'integritat:
```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  SECRET_KEY    // Clau que nomes el servidor coneix
)
```

> **Per que es signa?** Si algu modifica el payload (per exemple, canvia `"role":"USER"` per `"role":"ADMIN"`), la signatura ja no coincidira i el servidor rebutjara el token. El JWT es **tamper-proof**: pots llegir-lo, pero no pots modificar-lo sense la clau secreta.

> **Lectura recomanada (no bloquejant):**
> - [JWT.io Introduction](https://jwt.io/introduction) — Explicacio visual interactiva
> - [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)

### El Flux Complet: Login -> Token -> Acces

```
Client                           Servidor
  |                                 |
  |-- POST /auth/login ----------->|  1. Client envia credencials
  |   { "username":"pere",          |
  |     "password":"s3cr3t" }       |
  |                                 |  2. Servidor verifica contra la BD
  |<-- 200 OK ---------------------|  3. Si OK, genera JWT i el retorna
  |   { "token":"eyJhbG..." }      |
  |                                 |
  |-- GET /api/champions ---------->|  4. Client envia token a la capcalera
  |   Authorization: Bearer eyJhbG..|
  |                                 |  5. Servidor valida el JWT (signatura + expiracio)
  |<-- 200 OK ---------------------|  6. Si valid, retorna les dades
  |   [{ "name":"Faker", ... }]    |
  |                                 |
  |-- GET /api/champions ---------->|  7. Sense token o token invalid
  |   (sense Authorization header)  |
  |<-- 401 Unauthorized ------------|  8. Acces denegat
```

La capcalera **Authorization** segueix el format `Bearer <token>`. "Bearer" significa "portador" — qui porti aquest token te acces.

---

## Activitat

### 1. Explorar JWT a jwt.io (15 min)

Ves a [jwt.io](https://jwt.io) i experimenta:

1. Observa les tres parts codificades en colors (vermell, lila, blau)
2. Al payload, canvia el contingut:
```json
{
  "sub": "EL_TEU_NOM",
  "role": "ADMIN",
  "game": "League of Legends",
  "iat": 1726000000,
  "exp": 1726086400
}
```
3. Observa com el token canvia en temps real
4. Modifica un caracter del token codificat (a la part esquerra) — veuràs "Invalid Signature"

> **Lliçó clau:** El JWT no es xifrat — qualsevol pot decodificar el payload amb base64. La seguretat esta en la **signatura**, no en el secret del contingut. Mai posis contrasenyes o dades sensibles al payload.

### 2. Crear un JWT amb Python (20 min)

Crea un script per entendre la mecanica interna:

```python
# fitxer: ai-python/src/jwt_playground.py
# Objectiu: entendre com es crea i valida un JWT sense llibreries magiques

import json
import base64
import hmac
import hashlib
import time

# --- Clau secreta (en produccio, vindria d'una variable d'entorn) ---
SECRET_KEY = "la-meva-clau-secreta-super-segura-12345"

def base64url_encode(data: bytes) -> str:
    """Codifica bytes a base64url (variant de base64 segura per URLs).
    Elimina el padding '=' perque JWT no el necessita."""
    return base64.urlsafe_b64encode(data).rstrip(b'=').decode('utf-8')

def base64url_decode(data: str) -> bytes:
    """Decodifica base64url afegint el padding necessari."""
    padding = 4 - len(data) % 4
    data += '=' * padding
    return base64.urlsafe_b64decode(data)

def create_jwt(username: str, role: str, expiration_hours: int = 1) -> str:
    """Crea un JWT manualment per entendre cada pas."""

    # 1. Header: especifica l'algorisme de signatura
    header = {"alg": "HS256", "typ": "JWT"}
    header_b64 = base64url_encode(json.dumps(header).encode())

    # 2. Payload: les dades que volem transportar
    now = int(time.time())
    payload = {
        "sub": username,               # Qui es l'usuari
        "role": role,                   # Quin rol te
        "iat": now,                     # Quan s'ha creat el token
        "exp": now + (expiration_hours * 3600)  # Quan caduca
    }
    payload_b64 = base64url_encode(json.dumps(payload).encode())

    # 3. Signatura: HMAC-SHA256 del header.payload amb la clau secreta
    #    Aixo garanteix que ningu pot modificar el token sense la clau
    message = f"{header_b64}.{payload_b64}"
    signature = hmac.new(
        SECRET_KEY.encode(),
        message.encode(),
        hashlib.sha256
    ).digest()
    signature_b64 = base64url_encode(signature)

    # 4. Combinar les tres parts amb punts
    return f"{header_b64}.{payload_b64}.{signature_b64}"

def decode_jwt(token: str) -> dict:
    """Decodifica el payload d'un JWT (sense verificar signatura)."""
    parts = token.split('.')
    if len(parts) != 3:
        raise ValueError("El token no te 3 parts")

    payload_json = base64url_decode(parts[1])
    return json.loads(payload_json)

def verify_jwt(token: str) -> dict:
    """Verifica la signatura i retorna el payload si es valid."""
    parts = token.split('.')
    if len(parts) != 3:
        raise ValueError("Token amb format invalid")

    # Recalcula la signatura amb la clau secreta
    message = f"{parts[0]}.{parts[1]}"
    expected_sig = base64url_encode(
        hmac.new(SECRET_KEY.encode(), message.encode(), hashlib.sha256).digest()
    )

    # Compara la signatura del token amb l'esperada
    if not hmac.compare_digest(parts[2], expected_sig):
        raise ValueError("Signatura invalida — el token ha estat manipulat!")

    # Comprova expiracio
    payload = decode_jwt(token)
    if payload.get("exp", 0) < time.time():
        raise ValueError("Token caducat!")

    return payload


# --- Proves ---
if __name__ == "__main__":
    # Crear un token
    token = create_jwt("pere", "ADMIN", expiration_hours=1)
    print(f"Token creat:\n{token}\n")

    # Decodificar (qualsevol pot fer-ho — no es secret)
    payload = decode_jwt(token)
    print(f"Payload decodificat: {json.dumps(payload, indent=2)}\n")

    # Verificar (nomes qui te la clau secreta pot fer-ho)
    verified = verify_jwt(token)
    print(f"Token verificat correctament: {verified['sub']} ({verified['role']})\n")

    # Intentar manipular el token (canviar role a SUPERADMIN)
    print("--- Intent de manipulacio ---")
    tampered_payload = payload.copy()
    tampered_payload["role"] = "SUPERADMIN"
    tampered_payload_b64 = base64url_encode(json.dumps(tampered_payload).encode())
    parts = token.split('.')
    tampered_token = f"{parts[0]}.{tampered_payload_b64}.{parts[2]}"

    try:
        verify_jwt(tampered_token)
        print("ERROR: el token manipulat ha passat la verificacio!")
    except ValueError as e:
        print(f"Correcte! El servidor detecta la manipulacio: {e}")
```

Executa l'script:
```bash
cd esportspulse-engine
python3 ai-python/src/jwt_playground.py
```

### 3. Simular el Flux Login -> Token -> Acces amb curl (15 min)

Encara no tenim auth al backend, pero pots simular el flux complet:

```bash
# 1. Simula un login (imagina que el servidor retorna aquest token)
TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJwZXJlIiwicm9sZSI6IkFETUlOIiwiaWF0IjoxNzI2MDAwMDAwLCJleHAiOjE3MjYwODY0MDB9.placeholder"

# 2. Peticio AMB token (capcalera Authorization)
curl -H "Authorization: Bearer $TOKEN" http://localhost:8080/api/champions
# Ara mateix retorna 200 perque l'API no valida res (ho canviarem dimarts)

# 3. Peticio SENSE token
curl http://localhost:8080/api/champions
# Tambe retorna 200 — AQUEST es el problema que resoldrem
```

### 4. Documentar el Flux d'Autenticacio (10 min)

Crea un fitxer `docs/auth-flow.md` al projecte amb:
- Diagrama del flux (pots usar ASCII art o descripcio textual)
- Quins endpoints seran publics (register, login)
- Quins endpoints necessitaran autenticacio (CRUD de champions)
- Quins endpoints necessitaran autoritzacio especifica (crear/esborrar nomes ADMIN)

---

## Checklist de Lliurament

- [ ] Pots explicar la diferencia entre autenticacio i autoritzacio amb un exemple
- [ ] Has decodificat un JWT a jwt.io i has vist que passa si el manipules
- [ ] L'script `jwt_playground.py` funciona: crea, decodifica i verifica JWTs
- [ ] L'script detecta correctament un token manipulat (signatura invalida)
- [ ] Tens documentat el flux d'autenticacio previst per EsportsPulse
- [ ] Commit: `feat(auth): add JWT playground script and auth flow documentation`
