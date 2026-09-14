# Setmana 22 — Dijous: Domini, HTTPS i DNS

## Objectiu del Dia

Configurar un domini personalitzat per al projecte amb HTTPS automatic. Al final del dia, el teu dashboard ha de ser accessible per una URL com `esportspulse.el-teu-domini.dev` en comptes de la URL generica de la plataforma.

---

## Teoria

### Com Funciona un Domini a Internet

Quan escrius `esportspulse.example.com` al navegador, passa el seguent:

```
# 1. El navegador pregunta al DNS: "Quina IP te esportspulse.example.com?"
#
# 2. El DNS respon: "Apunta a 151.101.1.195" (IP del servidor de Render)
#    O be: "Es un CNAME que apunta a esportspulse.onrender.com"
#
# 3. El navegador connecta a la IP amb HTTPS (port 443)
#
# 4. El servidor (Render/Fly.io) rep la peticio,
#    mira el header "Host: esportspulse.example.com",
#    i la redirigeix al teu servei
#
# 5. El teu servei respon i el navegador mostra la pagina
```

### DNS: El Sistema de Noms de Domini

DNS es com una agenda telefonica d'Internet. Tradueix noms llegibles (dominis) a adreces IP.

**Tipus de registres DNS que necessitaras:**

```
# Registre A — Apunta un domini a una IP directament
# Exemple: example.com -> 151.101.1.195
# Usat per: dominis arrel (sense subdomini)

# Registre CNAME — Apunta un domini a un altre domini
# Exemple: esportspulse.example.com -> esportspulse.onrender.com
# Usat per: subdominis que apunten a plataformes al nuvol
# IMPORTANT: un CNAME NO pot ser al domini arrel (example.com)

# Registre AAAA — Com l'A pero per a IPv6
# Exemple: example.com -> 2606:4700:3030::6815:1234

# Registre TXT — Text arbitrari, usat per verificacio
# Exemple: _verify.example.com -> "render-verification=abc123"
# Usat per: demostrar que ets el propietari del domini
```

**Propagacio DNS:**

```
# Quan canvies un registre DNS, el canvi NO es instantani.
# Els servidors DNS de tot el mon guarden una copia (cache)
# amb un TTL (Time To Live) que indica quant temps es valida.
#
# TTL baix (300s = 5 min): canvis rapids, mes consultes DNS
# TTL alt (86400s = 24h): menys consultes, canvis lents
#
# Recomanacio: posa TTL baix (300) durant la configuracio.
# Quan tot funcioni, puja'l a 3600 (1h) o mes.
```

### HTTPS i Certificats SSL/TLS

HTTPS xifra la comunicacio entre el navegador i el servidor. Necessites un certificat SSL/TLS.

```
# Let's Encrypt: certificats SSL GRATIS i automatics
# La majoria de plataformes (Render, Fly.io) gestionen
# Let's Encrypt automaticament quan configures un domini.

# El proces es:
# 1. Configures el domini a la plataforma
# 2. Configures el DNS perque apunti a la plataforma
# 3. La plataforma demana un certificat a Let's Encrypt
# 4. Let's Encrypt verifica que el domini apunta a la plataforma
# 5. El certificat s'instal·la automaticament
# 6. HTTPS funciona sense que hagis de fer res mes

# El certificat es renova automaticament cada 90 dies
```

### On Comprar un Domini?

```
# Opcions populars (preus aproximats per a .dev o .com):
#
# Namecheap: ~10-15 EUR/any — Interficie clara, bon preu
# Cloudflare Registrar: ~10 EUR/any — Preu de cost, sense marges
# Google Domains (ara Squarespace): ~12 EUR/any
# Porkbun: ~8-12 EUR/any — Molt bons preus
#
# Dominis gratuïts per a estudiants:
# GitHub Student Pack inclou un domini .me gratuit (Namecheap)
# Freenom (*.tk, *.ml) — NO recomanat, poc fiable
#
# Si no vols comprar un domini, pots usar el subdomini
# gratuit de la plataforma (*.onrender.com, *.fly.dev)
```

### Subdominis vs Dominis Separats

```
# Opcio 1: Subdominis (recomanat)
# api.esportspulse.dev      -> Backend Java
# ai.esportspulse.dev       -> Servei Python
# app.esportspulse.dev      -> Dashboard Streamlit
# monitor.esportspulse.dev  -> Grafana
#
# Avantatge: un sol domini, organitzat per funció
# CNAME per a cada subdomini

# Opcio 2: Paths (mes senzill pero menys flexible)
# esportspulse.dev/api      -> Backend Java
# esportspulse.dev/ai       -> Servei Python
# esportspulse.dev/          -> Dashboard Streamlit
#
# Requereix un reverse proxy (Nginx/Caddy) davant de tot
# Mes complicat de configurar
```

---

## Activitat

### 1. Obtenir un Domini (20 min)

**Opcio A: Domini gratuit (GitHub Student Pack)**

Si tens GitHub Student Pack:
1. Ves a [education.github.com/pack](https://education.github.com/pack).
2. Busca Namecheap (.me gratuit) o .tech domains.
3. Registra el domini.

**Opcio B: Comprar un domini barat**

```bash
# Dominis .dev son ideals per a projectes de software
# Preu: ~10-15 EUR/any
# Alternatives barates: .xyz (~2 EUR/any), .site (~3 EUR/any)
```

**Opcio C: Usar el subdomini de la plataforma**

Si no vols gastar diners, pots usar directament:
- `esportspulse-backend.onrender.com`
- `esportspulse-ai.onrender.com`
- `esportspulse-dashboard.onrender.com`

Pero la practica de configurar DNS es molt valuosa.

### 2. Configurar DNS (30 min)

Al panell de control del teu proveidor de dominis:

```
# Registres DNS a crear:

# Subdomini per al dashboard (el principal)
# Tipus: CNAME
# Nom: app (o @ per al domini arrel)
# Valor: esportspulse-dashboard.onrender.com
# TTL: 300

# Subdomini per a l'API
# Tipus: CNAME
# Nom: api
# Valor: esportspulse-backend.onrender.com
# TTL: 300

# Subdomini per al servei d'IA
# Tipus: CNAME
# Nom: ai
# Valor: esportspulse-ai.onrender.com
# TTL: 300
```

### 3. Configurar el Domini a la Plataforma (20 min)

**A Render:**

1. Ves al servei (ex: esportspulse-dashboard).
2. Settings -> Custom Domain.
3. Afegeix `app.esportspulse.dev` (el teu domini).
4. Render et demanara verificar el DNS.
5. Render configurara HTTPS automaticament.

```bash
# Verifica que el DNS s'ha propagat
# dig mostra la resolucio DNS actual
dig app.esportspulse.dev CNAME

# nslookup es una alternativa
nslookup app.esportspulse.dev

# O usa un servei online
# https://dnschecker.org
```

**A Fly.io:**

```bash
# Afegeix el domini personalitzat
flyctl certs create app.esportspulse.dev

# Verifica l'estat del certificat
flyctl certs show app.esportspulse.dev
```

### 4. Verificar HTTPS (15 min)

```bash
# Comprova que HTTPS funciona correctament
curl -I https://app.esportspulse.dev

# Hauries de veure:
# HTTP/2 200
# ...
# strict-transport-security: max-age=31536000

# Comprova que HTTP redirigeix a HTTPS
curl -I http://app.esportspulse.dev
# Hauries de veure:
# HTTP/1.1 301 Moved Permanently
# location: https://app.esportspulse.dev/
```

### 5. Verificar Certificat SSL (10 min)

```bash
# Mostra els detalls del certificat SSL
# Ha de dir "Let's Encrypt" com a emissor
openssl s_client -connect app.esportspulse.dev:443 -servername app.esportspulse.dev < /dev/null 2>/dev/null | openssl x509 -noout -text | grep -E "Issuer|Not After|Subject:"

# O usa un servei online
# https://www.ssllabs.com/ssltest/
```

### 6. Actualitzar CORS al Backend (15 min)

Amb dominis personalitzats, cal actualitzar els CORS:

```java
// WebConfig.java — Actualitzar els origens permesos
// CORS (Cross-Origin Resource Sharing) controla quins dominis
// poden fer peticions a la nostra API

@Configuration
public class WebConfig implements WebMvcConfigurer {

    // Llegim els origens permesos de variables d'entorn
    // Permet configurar-los diferent en local i produccio
    @Value("${cors.allowed-origins:http://localhost:8501}")
    private String[] allowedOrigins;

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins(allowedOrigins)  // Dominis permesos
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("*")
            .allowCredentials(true);
    }
}
```

```bash
# Afegeix la variable d'entorn a Render/Fly.io:
# CORS_ALLOWED_ORIGINS=https://app.esportspulse.dev,https://esportspulse-dashboard.onrender.com
```

### 7. Commit i Documentar (10 min)

```bash
# Documenta les URLs al README o CLAUDE.md
git add -u
git commit -m "feat(deploy): configure custom domain and HTTPS"
```

---

## Checklist de Lliurament

- [ ] Domini obtingut (propi o subdomini de la plataforma)
- [ ] Registres DNS configurats (CNAME per a cada subdomini)
- [ ] Domini verificat a la plataforma (Render/Fly.io)
- [ ] HTTPS funciona correctament (certificat valid)
- [ ] HTTP redirigeix a HTTPS
- [ ] CORS actualitzat amb els nous dominis
- [ ] Dashboard accessible per URL personalitzada
- [ ] API accessible per URL personalitzada
- [ ] URLs documentades al CLAUDE.md/README
- [ ] Commit amb els canvis
