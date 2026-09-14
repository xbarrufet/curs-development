# Setmana 24 — Dilluns: GitHub Pages: Crear la Teva Pagina Personal

## Objectiu del Dia

Crear una pagina personal amb GitHub Pages que serveixi com a portfolio professional. Al final del dia, has de tenir una web accessible a `el-teu-username.github.io` amb seccions d'About Me, Projectes, Habilitats Tecniques i Contacte.

---

## Teoria

### Que es GitHub Pages?

GitHub Pages es un servei gratuit de GitHub que converteix un repositori en una pagina web estatica. Es ideal per a portfolios, documentacio de projectes, i blogs tecnics.

```
# Com funciona:
# 1. Crees un repositori amb nom especial: username.github.io
# 2. Hi poses fitxers HTML/CSS/JS (o Markdown amb Jekyll)
# 3. GitHub compila i publica automaticament
# 4. La web es accessible a https://username.github.io
#
# Caracteristiques:
# - Gratuit (sense limits raonables)
# - HTTPS automatic
# - Domini personalitzat (opcional)
# - Suport per a Jekyll (generador de webs estatics)
# - Desplegament automatic amb cada push
```

### Jekyll: Generador de Webs Estatics

Jekyll es un generador de webs estatics integrat amb GitHub Pages. Converteix Markdown en HTML automaticament.

```
# Per que Jekyll?
# - No cal compilar res localment (GitHub ho fa)
# - Escrius en Markdown (ja ho saps fer)
# - Temes professionals disponibles
# - Suport per a blogs, portfolios, i documentacio
# - Gratuit amb GitHub Pages

# Alternatives (mes complexes):
# - Hugo (mes rapid, mes configurable)
# - Astro (modern, basat en components)
# - HTML/CSS pur (maxim control, mes feina)
```

### Estructura d'un Repositori GitHub Pages amb Jekyll

```
username.github.io/
  _config.yml        # Configuracio de Jekyll (tema, titol, etc.)
  index.md           # Pagina principal
  about.md           # Pagina "Sobre Mi"
  _posts/            # Blog (opcional)
    2024-01-15-primer-post.md
  assets/
    css/
      style.css      # Estils personalitzats
    images/
      profile.jpg    # Foto de perfil
      project-screenshot.png
  _data/
    projects.yml     # Dades dels projectes (opcional, per a templates)
```

### Temes de Jekyll

GitHub Pages suporta directament diversos temes sense instal·lar res:

```yaml
# _config.yml — Configuracio basica de Jekyll

# Tema: tria un dels temes suportats per GitHub Pages
# Opcions populars:
# - minima (net, minimal)
# - cayman (modern, verd)
# - architect (classic, gris)
# - minimal (ultrasimple)
remote_theme: pages-themes/cayman@v0.2.0

# O usa un tema de tercers (mes opcions):
# remote_theme: jekyll/minima

# Metadades del lloc
title: "El Teu Nom — Software Engineer"
description: "Portfolio de desenvolupament de software"
url: "https://username.github.io"

# Xarxes socials
github_username: el-teu-username
linkedin_username: el-teu-linkedin

# Configuracio de Markdown
markdown: kramdown
```

### Per Que un Portfolio?

```
# En una entrevista tecnica:
#
# Candidat A: "Se Java i Python" (sense proves)
# Candidat B: "He construït una plataforma d'analisi d'esports amb 
#              Java/Spring Boot + Python/FastAPI + Docker + agents IA.
#              Aqui tens el repositori i la demo desplegada."
#              (amb link al portfolio)
#
# Qui te mes probabilitats d'aconseguir la feina?
#
# El portfolio NO substitueix la competencia tecnica,
# pero DEMOSTRA que la tens d'una manera tangible.
```

---

## Activitat

### 1. Crear el Repositori (10 min)

```bash
# Crea un directori nou per al portfolio
# (no dins del projecte EsportsPulse)
mkdir ~/username.github.io
cd ~/username.github.io
git init

# O crea'l directament a GitHub:
# 1. GitHub.com -> New Repository
# 2. Nom: username.github.io (substitueix "username" pel teu)
# 3. Public (obligatori per a GitHub Pages gratuit)
# 4. Marca "Add a README file"
# 5. Clone: git clone https://github.com/username/username.github.io.git
```

### 2. Configurar Jekyll (15 min)

Crea `_config.yml`:

```yaml
# _config.yml — Configuracio del portfolio

# Tema visual — cayman es net i professional
remote_theme: pages-themes/cayman@v0.2.0

# Informacio del lloc
title: "El Teu Nom"
description: "Software Engineer | Java | Python | AI Agents | Cloud"

# Plugins necessaris per a GitHub Pages
plugins:
  - jekyll-remote-theme
  - jekyll-seo-tag

# Metadades per a SEO (apareix a Google)
author: "El Teu Nom"
lang: "ca"

# Xarxes socials
social:
  github: el-teu-username
  linkedin: el-teu-linkedin
  email: el-teu-email@example.com
```

### 3. Escriure la Pagina Principal (30 min)

Crea `index.md`:

```markdown
---
layout: default
title: Home
---

# Hola, soc [El Teu Nom]

Software Engineer amb experiencia en Java, Python, i Intel·ligencia Artificial.

## Sobre Mi

<!-- 2-3 paragrafs sobre tu:
     - Que fas (estudiant, professional, autodidacta)
     - Que t'interessa (backend, IA, cloud, etc.)
     - Que busques (feina, projectes, col·laboracions) -->

Soc un desenvolupador de software amb passio per construir sistemes
escalables i aplicacions intel·ligents. He treballat amb tecnologies
com Java/Spring Boot, Python/FastAPI, Docker, i agents d'IA.

## Projectes Destacats

### EsportsPulse Engine
Plataforma d'analisi d'esports electronics amb agents d'IA.

**Tecnologies:** Java 21, Spring Boot, Python, FastAPI, Docker, PostgreSQL,
Redis, RabbitMQ, Prometheus, Grafana, LangChain, Claude API

**Funcionalitats:**
- API REST completa amb autenticacio JWT
- Agents d'IA per a analisi quantitatiu i knowledge retrieval
- Dashboard interactiu amb Streamlit
- Monitoritzacio amb Prometheus i Grafana
- Desplegament al nuvol amb CI/CD

[Veure Repositori](https://github.com/username/esportspulse-engine) |
[Veure Demo](https://app.esportspulse.dev)

<!-- Afegeix mes projectes si en tens -->

## Habilitats Tecniques

### Llenguatges
- Java (21) — Spring Boot, JPA, JUnit
- Python (3.12) — FastAPI, LangChain, pytest

### Infraestructura
- Docker i Docker Compose
- CI/CD amb GitHub Actions
- Desplegament al nuvol (Render/Fly.io)

### Bases de Dades
- PostgreSQL (relacional)
- Redis (cache)
- RabbitMQ (missatgeria)

### Eines d'IA
- LangChain (agents)
- RAG (Retrieval-Augmented Generation)
- Claude API (Anthropic)

### Observabilitat
- Prometheus + Grafana
- LangFuse (tracing de LLMs)
- Logging estructurat

## Contacte

- GitHub: [username](https://github.com/username)
- LinkedIn: [El Teu Nom](https://linkedin.com/in/username)
- Email: el-teu-email@example.com
```

### 4. Afegir Estils Personalitzats (20 min)

Crea `assets/css/style.scss`:

```scss
---
---

// Importa els estils base del tema
@import "{{ site.theme }}";

// Personalitzacions

// Estil per a les seccions de projectes
.project-card {
  border: 1px solid #e1e4e8;
  border-radius: 6px;
  padding: 16px;
  margin-bottom: 16px;
  // Ombra subtil per donar profunditat
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.12);
}

// Estil per a les habilitats (tags)
.skill-tag {
  display: inline-block;
  background-color: #0366d6;
  color: white;
  padding: 2px 8px;
  border-radius: 12px;
  margin: 2px;
  font-size: 0.85em;
}

// Seccions amb mes espai
section {
  margin-bottom: 2em;
}
```

### 5. Publicar i Verificar (15 min)

```bash
# Afegeix tots els fitxers i puja a GitHub
git add -A
git commit -m "feat: initial portfolio with Jekyll and GitHub Pages"
git remote add origin https://github.com/username/username.github.io.git
git push -u origin main
```

Configura GitHub Pages:
1. Ves a Settings -> Pages al repositori.
2. Source: "Deploy from a branch"
3. Branch: main, carpeta: / (root)
4. Espera 1-2 minuts.
5. Obre `https://username.github.io`.

```bash
# Verifica que la pagina es accessible
curl -I https://username.github.io
# Hauria de retornar HTTP 200
```

### 6. Commit al Projecte Principal (10 min)

Torna al projecte EsportsPulse i documenta la URL del portfolio:

```bash
cd ~/esportspulse-engine
# Actualitza el README amb el link al portfolio
git add README.md
git commit -m "docs: add portfolio link to README"
```

---

## Checklist de Lliurament

- [ ] Repositori `username.github.io` creat a GitHub
- [ ] `_config.yml` amb tema i metadades
- [ ] Pagina principal amb About Me, Projectes, Habilitats, Contacte
- [ ] EsportsPulse destacat com a projecte principal
- [ ] Estils personalitzats aplicats
- [ ] Pagina accessible a `https://username.github.io`
- [ ] HTTPS funciona correctament
- [ ] El contingut es professional i sense faltes
- [ ] Commit al repositori del portfolio
