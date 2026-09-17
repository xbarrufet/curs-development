# Setmana 12 — Divendres: Headers de Seguretat, CSRF i Consolidacio

## Objectiu del Dia

Completar la seguretat de l'API amb headers de proteccio, entendre CSRF i quan desactivar-lo, i fer una auditoria de seguretat completa de tot el projecte. Al final del dia, l'API ha de complir les recomanacions basiques d'OWASP i tots els canvis de seguretat han d'estar comitejats i integrats en un PR.

---

## Teoria

### Security Headers: La Primera Linia de Defensa

Els headers HTTP de seguretat instrueixen el navegador sobre com tractar el contingut. Son facils d'implementar i prevenen categories senceres d'atacs:

**1. X-Content-Type-Options: nosniff**
```
Sense header: El navegador "endevina" el tipus de contingut (MIME sniffing).
             Un fitxer .txt amb codi JavaScript pot ser executat com a script.
Amb header:  El navegador respecta el Content-Type declarat pel servidor.
             Un .txt mai s'executara com a JavaScript.
```

**2. X-Frame-Options: DENY**
```
Sense header: La teva pagina pot ser carregada dins un <iframe> d'un altre web.
             Atac clickjacking: l'atacant posa la teva web dins un iframe invisible
             i l'usuari clica botons sense saber que esta interactuant amb el teu site.
Amb header:  El navegador refusa carregar la pagina dins un iframe.
```

**3. Strict-Transport-Security (HSTS)**
```
Sense header: L'usuari pot accedir per HTTP (sense xifrar). Un atacant a la
             mateixa xarxa WiFi pot interceptar les dades (man-in-the-middle).
Amb header:  El navegador forca HTTPS durant el temps especificat.
             Fins i tot si l'usuari escriu http://, el navegador el redirigeix a https://.
```

```
Strict-Transport-Security: max-age=31536000; includeSubDomains
                           |                 |
                           1 any en segons   Aplica tambe a subdominis
```

**4. Content-Security-Policy (CSP)**
```
Sense header: El navegador carrega scripts, estils i imatges de qualsevol origen.
             Un atac XSS pot injectar un <script src="https://evil.com/steal.js">
Amb header:  Nomes es carreguen recursos dels origens permesos.
```

### CSRF (Cross-Site Request Forgery)

CSRF es un atac on un site malicious fa peticions al teu backend **usant les cookies de sessio de l'usuari**:

```
1. L'usuari fa login a esportspulse.com → El navegador guarda una cookie de sessio
2. L'usuari visita evil.com (sense fer logout)
3. evil.com te un formulari ocult que fa POST a esportspulse.com/api/champions
4. El navegador envia la cookie de sessio automaticament (es del mateix domini!)
5. El backend rep la peticio amb cookie valida → Executa l'accio sense saber que ve d'evil.com
```

**Per que les APIs REST stateless amb JWT NO necessiten proteccio CSRF?**

```
Amb cookies (vulnerable a CSRF):
- El navegador envia la cookie AUTOMATICAMENT a cada peticio al domini
- evil.com pot fer que el navegador enviï la cookie sense consentiment de l'usuari

Amb JWT a la capcalera Authorization (NO vulnerable):
- El token s'envia MANUALMENT a la capcalera (JavaScript ho ha de fer explicitament)
- evil.com NO pot accedir al token guardat a localStorage d'un altre domini (Same-Origin Policy)
- Per tant, evil.com no pot enviar el JWT → La peticio falla amb 401
```

> **Regla simple:** Si uses cookies per autenticacio → activa CSRF. Si uses JWT a headers → pots desactivar CSRF amb seguretat. Per aixo hem posat `.csrf(csrf -> csrf.disable())` a Spring Security.

### OWASP Top 10: Que Ja Has Resolt

L'OWASP (Open Worldwide Application Security Project) publica les 10 vulnerabilitats web mes critiques. Revisem quines ja has abordat:

| # | Vulnerabilitat | Estat | On |
|---|---|---|---|
| A01 | Broken Access Control | Resolt | @PreAuthorize, Depends(require_role) |
| A02 | Cryptographic Failures | Resolt | BCrypt per contrasenyes, JWT signat amb HS256 |
| A03 | Injection | Parcialment | JPA parameterized queries (S4), pero cal revisar |
| A04 | Insecure Design | Resolt | Principi deny-by-default a Spring Security |
| A05 | Security Misconfiguration | Avui | Headers de seguretat, CORS configurat |
| A06 | Vulnerable Components | Pendent | Cal actualitzar dependencies regularment |
| A07 | Auth Failures | Resolt | JWT, BCrypt, no revelar info en errors d'auth |
| A08 | Software/Data Integrity | Parcial | JWT signat, pero cal validar inputs |
| A09 | Logging Failures | Resolt | Logging estructurat (S11) |
| A10 | SSRF | N/A | No fem peticions a URLs externes des del backend |

> **Lectura recomanada (no bloquejant):**
> - [OWASP Top 10 (2021)](https://owasp.org/www-project-top-ten/)
> - [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/)
> - [MDN: Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP)

---

## Activitat

### 1. Afegir Security Headers a Spring Boot (15 min)

```java
// fitxer: src/main/java/com/esportspulse/engine/security/SecurityConfig.java
// Actualitzar el SecurityFilterChain per incloure headers de seguretat.
// Afegir despres de .csrf() i abans de .authorizeHttpRequests()

@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .cors(cors -> cors.configurationSource(corsConfigurationSource()))
        .csrf(csrf -> csrf.disable())

        // Headers de seguretat — proteccio a nivell de navegador
        .headers(headers -> headers
            // X-Content-Type-Options: nosniff
            // Preveu que el navegador "endevini" el tipus MIME
            .contentTypeOptions(contentType -> {})  // Activat per defecte amb nosniff

            // X-Frame-Options: DENY
            // Preveu que la pagina es carregui dins un iframe (anti-clickjacking)
            .frameOptions(frame -> frame.deny())

            // Strict-Transport-Security: max-age=31536000; includeSubDomains
            // Forca HTTPS durant 1 any (nomes efectiu si el servidor usa HTTPS)
            .httpStrictTransportSecurity(hsts -> hsts
                .includeSubDomains(true)
                .maxAgeInSeconds(31536000)  // 1 any
            )

            // Content-Security-Policy: restringeix d'on es poden carregar recursos
            // Aquesta politica es conservadora — nomes permet recursos del propi origen
            .contentSecurityPolicy(csp ->
                csp.policyDirectives("default-src 'self'; frame-ancestors 'none'"))
        )

        .sessionManagement(session ->
            session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/auth/**").permitAll()
            .requestMatchers("/actuator/health").permitAll()
            .anyRequest().authenticated()
        )
        .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

    return http.build();
}
```

### 2. Afegir Security Headers a FastAPI (10 min)

```python
# fitxer: ai-python/src/middleware/security_headers.py
# Middleware que afegeix headers de seguretat a TOTES les respostes.
# Equivalent de la configuracio .headers() de Spring Security.

from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response


class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    """Afegeix headers de seguretat a cada resposta HTTP.

    Aquests headers instrueixen el navegador sobre com tractar el contingut
    i prevenen categories senceres d'atacs (XSS, clickjacking, MIME sniffing).
    """

    async def dispatch(self, request: Request, call_next) -> Response:
        response = await call_next(request)

        # Preveu MIME sniffing — el navegador respecta el Content-Type declarat
        response.headers["X-Content-Type-Options"] = "nosniff"

        # Preveu clickjacking — la pagina no es pot carregar en un iframe
        response.headers["X-Frame-Options"] = "DENY"

        # Forca HTTPS (efectiu nomes amb HTTPS configurat)
        response.headers["Strict-Transport-Security"] = (
            "max-age=31536000; includeSubDomains"
        )

        # Restringeix d'on es poden carregar scripts i recursos
        response.headers["Content-Security-Policy"] = (
            "default-src 'self'; frame-ancestors 'none'"
        )

        # Desactiva la cache per respostes amb dades sensibles
        response.headers["Cache-Control"] = "no-store"

        # Controla quina informacio de referrer s'envia
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"

        return response
```

Registra el middleware a `main.py`:

```python
# fitxer: ai-python/src/main.py
# Afegir al principi, despres de crear l'app

from middleware.security_headers import SecurityHeadersMiddleware
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI(title="EsportsPulse AI Service", version="0.12.0")

# Middleware de seguretat (s'executa a CADA resposta)
app.add_middleware(SecurityHeadersMiddleware)

# CORS (equivalent de corsConfigurationSource() a Spring)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:8501", "http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)
```

### 3. Verificar Headers amb curl (10 min)

```bash
# Comprova els headers del backend Java
curl -s -D - -o /dev/null http://localhost:8080/auth/login \
  -X POST -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin-password-123"}' | grep -iE "x-content|x-frame|strict-transport|content-security"

# Esperat:
# X-Content-Type-Options: nosniff
# X-Frame-Options: DENY
# Strict-Transport-Security: max-age=31536000; includeSubDomains
# Content-Security-Policy: default-src 'self'; frame-ancestors 'none'

# Comprova els headers del servei Python
curl -s -D - -o /dev/null http://localhost:8000/api/ai/health | grep -iE "x-content|x-frame|strict-transport|content-security|cache-control"
```

### 4. Auditoria de Seguretat Completa (40 min)

Crea un fitxer `docs/security-audit.md` amb els resultats de l'auditoria:

```markdown
# Auditoria de Seguretat — EsportsPulse (Setmana 12)

## 1. Endpoints Protegits

### Backend Java (Spring Boot)
| Endpoint | Metode | Auth Required | Rol Minim | Verificat |
|---|---|---|---|---|
| /auth/register | POST | No | - | [ ] |
| /auth/login | POST | No | - | [ ] |
| /api/champions | GET | Si | USER | [ ] |
| /api/champions/{id} | GET | Si | USER | [ ] |
| /api/champions | POST | Si | ADMIN | [ ] |
| /api/champions/{id} | PUT | Si | ADMIN | [ ] |
| /api/champions/{id} | DELETE | Si | ADMIN | [ ] |
| /actuator/health | GET | No | - | [ ] |

### Servei Python (FastAPI)
| Endpoint | Metode | Auth Required | Rol Minim | Verificat |
|---|---|---|---|---|
| /auth/register | POST | No | - | [ ] |
| /auth/login | POST | No | - | [ ] |
| /auth/logout | POST | Si (Bearer) | USER | [ ] |
| /api/ai/health | GET | No | - | [ ] |
| /api/ai/analysis | GET | Si | USER | [ ] |
| /api/ai/models/train | POST | Si | ADMIN | [ ] |

## 2. Checklist de Seguretat

### Autenticacio
- [ ] Contrasenyes hashejades amb BCrypt (verificat a la BD)
- [ ] JWT signat amb HS256 i clau de minim 256 bits
- [ ] Clau JWT en variable d'entorn (no hardcoded al codi)
- [ ] Tokens amb expiracio (1 hora)
- [ ] Login no revela si l'error es d'username o password

### Autoritzacio
- [ ] Endpoints de lectura: USER o superior
- [ ] Endpoints de modificacio: ADMIN
- [ ] Deny-by-default: endpoints nous requereixen auth per defecte
- [ ] Token blacklist funcional (logout invalida el token)

### Headers de Seguretat
- [ ] X-Content-Type-Options: nosniff
- [ ] X-Frame-Options: DENY
- [ ] Strict-Transport-Security configurat
- [ ] Content-Security-Policy configurat
- [ ] CSRF desactivat (justificacio: API stateless amb JWT)

### CORS
- [ ] Origens permesos explicits (no wildcard *)
- [ ] Metodes permesos explicits
- [ ] Capcalera Authorization permesa

### Infraestructura
- [ ] Redis funcionant per token blacklist
- [ ] Secrets al .env (exclòs de Git per .gitignore)
- [ ] .env NO esta al repositori (verificar amb git log)
```

### 5. Executar l'Auditoria (30 min)

Completa cada punt de la checklist verificant manualment:

```bash
# === AUTENTICACIO ===

# Verificar que les contrasenyes estan hashejades
docker exec -it esportspulse-db psql -U postgres -d esportspulse \
  -c "SELECT username, LEFT(password, 10) as password_prefix FROM app_users;"
# Ha de mostrar "$2a$10$..." (prefix BCrypt), MAI text pla

# Verificar que la clau JWT NO esta hardcoded al codi comitejat
grep -r "jwt.secret" --include="*.properties" --include="*.yml" .
# Ha de referenciar ${JWT_SECRET:...}, no un valor literal

# Verificar que .env no esta al repositori
git log --all --diff-filter=A -- "*.env" ".env"
# No ha de retornar res

# === AUTORITZACIO ===

# Token USER intenta crear champion (ha de ser 403)
USER_TOKEN=$(curl -s -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"viewer","password":"viewer-pass-123"}' | jq -r .token)

curl -s -w "\n%{http_code}\n" -X POST http://localhost:8080/api/champions \
  -H "Authorization: Bearer $USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"Test"}'
# Esperat: 403

# === HEADERS ===

# Verificar TOTS els headers de seguretat (Java)
echo "=== Java Backend Headers ==="
curl -s -D - -o /dev/null http://localhost:8080/actuator/health

# Verificar TOTS els headers de seguretat (Python)
echo "=== Python Service Headers ==="
curl -s -D - -o /dev/null http://localhost:8000/api/ai/health

# === CORS ===

# Verificar que wildcard NO esta configurat
curl -s -X OPTIONS http://localhost:8080/api/champions \
  -H "Origin: http://evil.com" \
  -H "Access-Control-Request-Method: GET" \
  -D - -o /dev/null | grep "Access-Control-Allow-Origin"
# NO ha de mostrar "http://evil.com" (nomes localhost:8501 i localhost:3000 son permesos)

# === REDIS ===

# Verificar que Redis esta operatiu
docker exec esportspulse-redis redis-cli ping
# PONG

# Verificar que els tokens expirats s'eliminen sols (TTL)
docker exec esportspulse-redis redis-cli dbsize
# Ha de mostrar el nombre de tokens a la blacklist
```

### 6. Corregir Problemes Trobats (15 min)

Si l'auditoria detecta problemes, corregeix-los. Problemes comuns:

```bash
# Problema: clau JWT massa curta (menys de 32 caracters)
# Solucio: generar una clau segura
python3 -c "import secrets; print(secrets.token_hex(32))"
# Copia el resultat al .env com a JWT_SECRET

# Problema: endpoint sense proteccio que hauria de tenir-la
# Solucio: afegir @PreAuthorize o Depends(get_current_user)

# Problema: CORS amb wildcard (allow_origins=["*"])
# Solucio: especificar origens explicits
```

### 7. Commit i Pull Request (15 min)

```bash
# Afegir tots els canvis de seguretat
git add -A
git status  # Revisar que no hi ha fitxers sensibles

# Commit amb tots els canvis de la setmana
git commit -m "feat(security): complete auth system with JWT, roles, headers and Redis blacklist

- Spring Security with BCrypt password hashing and JWT authentication
- Role-based access control (USER/ADMIN) with @PreAuthorize
- JWT authentication filter for stateless session management
- FastAPI JWT auth with dependency injection pattern
- Redis token blacklist for session invalidation
- Security headers (X-Content-Type-Options, X-Frame-Options, HSTS, CSP)
- CORS configuration for frontend integration
- Security audit documentation"

# Crear PR
git push -u origin feature/week12-security
# Ves a GitHub i crea un Pull Request amb:
# - Titol: "feat(security): Authentication, authorization and security headers"
# - Descripcio: resum dels canvis i resultats de l'auditoria
```

---

## Checklist de Lliurament

- [ ] Security headers presents a les respostes del backend Java (verificat amb curl)
- [ ] Security headers presents a les respostes del servei Python (verificat amb curl)
- [ ] CORS no accepta origens arbitraris (verificat amb Origin: http://evil.com)
- [ ] Auditoria de seguretat completada amb tots els punts verificats
- [ ] Cap contrasenya en text pla a la base de dades
- [ ] Cap secret hardcoded al codi font (clau JWT ve de variable d'entorn)
- [ ] `.env` NO esta al repositori Git
- [ ] Redis operatiu i eliminant tokens expirats automaticament
- [ ] Tots els canvis comitejats i PR creat a GitHub
- [ ] Commit final: `feat(security): complete auth system with JWT, roles, headers and Redis blacklist`
