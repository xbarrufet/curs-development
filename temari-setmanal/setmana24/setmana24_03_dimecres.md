# Setmana 24 — Dimecres: Preparar el Repositori com a Portfolio

## Objectiu del Dia

Polir el repositori d'EsportsPulse perque tingui el nivell de presentacio d'un projecte professional de codi obert. Al final del dia, el repositori ha de tenir descripcio, topics, social preview, README complet, LICENSE, CONTRIBUTING.md, badges, i diagrama d'arquitectura.

---

## Teoria

### El Repositori com a Carta de Presentacio

Quan un reclutador o tech lead visita el teu repositori GitHub, es fixa en:

```
# En 10 segons (primera impressio):
# 1. Nom del repositori (descriptiu?)
# 2. Descripcio (1 linia que expliqui que fa)
# 3. Topics/tags (Java, Python, Docker, AI...)
# 4. README (te estructura? te imatges?)
# 5. Activitat recent (hi ha commits recents?)

# En 1 minut (si la primera impressio es bona):
# 6. Estructura del codi (organitzada?)
# 7. Tests (n'hi ha? passen?)
# 8. CI/CD (te GitHub Actions?)
# 9. Docker (es facil d'executar?)
# 10. Documentacio (es pot entendre sense ajuda?)

# En 5 minuts (si estan realment interessats):
# 11. Qualitat del codi (noms, patrons, comentaris)
# 12. Historial de commits (missatges descriptius?)
# 13. PRs i issues (flux de treball professional?)
```

### Pinned Repositories

GitHub permet "fixar" fins a 6 repositoris al teu perfil. Son els primers que es veuen.

```
# Per fixar un repositori:
# 1. Ves al teu perfil (github.com/username)
# 2. Clica "Customize your pins"
# 3. Selecciona EsportsPulse i altres projectes importants
# 4. Ordena'ls per importancia
```

### Social Preview Image

La social preview es la imatge que es mostra quan comparteixes el link del repositori a xarxes socials o Slack:

```
# Recomanacions per a la social preview:
# - Tamany: 1280x640px
# - Format: PNG o JPG
# - Contingut: nom del projecte + breu descripcio + logo/icona
# - Colors: coherents amb el projecte
# - Text: llegible (font gran)
#
# Eines per crear-la:
# - Canva (gratuit, facil)
# - Figma (gratuit, mes control)
# - OG Image Generator de Vercel (automatic)
```

### Fitxers Estandard d'un Repositori Professional

```
# Fitxers que tot repositori professional ha de tenir:

# README.md — Documentacio principal
# Ja creat a S23. Avui el polim.

# LICENSE — Llicencia del projecte
# Sense llicencia, legalment ningu pot usar el teu codi.
# Opcions comunes:
# - MIT: permissiva, qualsevol pot fer el que vulgui
# - Apache 2.0: com MIT pero amb proteccio de patents
# - GPL 3.0: copyleft, derivats han de ser GPL tambe

# CONTRIBUTING.md — Guia per a contribuidors
# Ja creat a S23. Avui el revisem.

# CODE_OF_CONDUCT.md — Codi de conducta
# Opcional pero recomanat per a projectes publics.

# .github/
#   ISSUE_TEMPLATE/ — Plantilles per a issues
#     bug_report.md
#     feature_request.md
#   PULL_REQUEST_TEMPLATE.md — Plantilla per a PRs
```

### Badges: Senyals de Qualitat

Les badges son un llenguatge visual reconegut a tot GitHub:

```markdown
<!-- Badges comuns amb shields.io -->

<!-- CI Status — mostra si el build passa -->
[![CI](https://github.com/USER/REPO/actions/workflows/ci.yml/badge.svg)](https://github.com/USER/REPO/actions)

<!-- Llicencia -->
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

<!-- Java version -->
[![Java](https://img.shields.io/badge/Java-21-ED8B00?logo=openjdk&logoColor=white)](https://openjdk.org/)

<!-- Python version -->
[![Python](https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white)](https://python.org/)

<!-- Docker -->
[![Docker](https://img.shields.io/badge/Docker-Ready-2496ED?logo=docker&logoColor=white)](https://docker.com/)

<!-- Spring Boot -->
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.2-6DB33F?logo=spring&logoColor=white)](https://spring.io/)

<!-- FastAPI -->
[![FastAPI](https://img.shields.io/badge/FastAPI-0.109-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
```

---

## Activitat

### 1. Configurar Metadades del Repositori (15 min)

A GitHub, ves a Settings del repositori EsportsPulse:

**Descripcio:**
```
Esports analytics platform with AI agents for statistical analysis and 
knowledge retrieval. Java/Spring Boot + Python/FastAPI + Docker.
```

**Topics (tags):**
```
java, python, spring-boot, fastapi, docker, ai-agents, langchain, 
postgresql, redis, rabbitmq, prometheus, grafana, streamlit
```

**Website:**
```
https://app.esportspulse.dev (o la URL del teu portfolio)
```

### 2. Crear Social Preview (15 min)

Crea una imatge de 1280x640px amb:
- Nom del projecte: "EsportsPulse Engine"
- Subtitol: "AI-Powered Esports Analytics"
- Icones o logos de les tecnologies principals
- Colors professionals

```bash
# Opcio rapida amb Canva:
# 1. Ves a canva.com
# 2. Crea un disseny personalitzat (1280x640)
# 3. Afegeix text i icones
# 4. Descarrega com a PNG

# Puja la imatge a GitHub:
# Settings -> Social preview -> Upload
```

### 3. Polir el README (20 min)

Revisa el README creat a S23 i afegeix:

```markdown
<!-- Al principi del README, despres del titol -->

<!-- Badges en una sola linia -->
[![CI](badge_url)](link)
[![Java 21](badge_url)](link)
[![Python 3.12](badge_url)](link)
[![Docker](badge_url)](link)
[![License: MIT](badge_url)](link)

<!-- Despres dels badges, afegeix un GIF o screenshot -->
<!-- Un visual al principi captura l'atencio immediatament -->

![EsportsPulse Dashboard](docs/images/dashboard-preview.png)
```

Verifica que:
- Les comandes del Quickstart funcionen copy-paste
- No hi ha seccions buides o amb placeholder text
- Els links funcionen (repositori, demo, documentacio)

### 4. Crear Plantilles d'Issues i PR (20 min)

Crea `.github/ISSUE_TEMPLATE/bug_report.md`:

```markdown
---
name: Bug Report
about: Report a bug in EsportsPulse
title: '[BUG] '
labels: bug
---

## Descripcio del Bug
<!-- Que passa? -->

## Passos per Reproduir
1. ...
2. ...
3. ...

## Comportament Esperat
<!-- Que hauria de passar? -->

## Comportament Actual
<!-- Que passa realment? -->

## Entorn
- OS: [ex: macOS 14.0]
- Docker version: [ex: 24.0.7]
- Java version: [ex: 21]

## Screenshots
<!-- Si aplica, afegeix captures de pantalla -->

## Logs
<!-- Si aplica, enganxa els logs rellevants -->
```

Crea `.github/ISSUE_TEMPLATE/feature_request.md`:

```markdown
---
name: Feature Request
about: Suggest a new feature for EsportsPulse
title: '[FEATURE] '
labels: enhancement
---

## Descripcio de la Funcionalitat
<!-- Que vols que faci? -->

## Motivacio
<!-- Per que es necessaria? Quin problema resol? -->

## Proposta d'Implementacio
<!-- Com creus que s'hauria d'implementar? -->

## Alternatives Considerades
<!-- Has considerat altres opcions? -->
```

Crea `.github/PULL_REQUEST_TEMPLATE.md`:

```markdown
## Resum
<!-- Que fa aquest PR? -->

## Tipus de Canvi
- [ ] Bug fix
- [ ] Nova funcionalitat
- [ ] Refactoring
- [ ] Documentacio
- [ ] Altres

## Canvis Principals
<!-- Llista dels canvis mes importants -->

## Com Provar
<!-- Instruccions per verificar els canvis -->

## Checklist
- [ ] He afegit tests per als meus canvis
- [ ] Tots els tests passen (`mvn test && pytest`)
- [ ] He actualitzat la documentacio si calia
- [ ] El meu codi segueix les convencions del projecte
```

### 5. Verificar Badges i Links (10 min)

```bash
# Genera les URLs de badges correctes per al teu repositori
echo "Badges per al README:"
echo ""
echo "CI: https://github.com/USERNAME/esportspulse-engine/actions/workflows/ci.yml/badge.svg"
echo "License: https://img.shields.io/badge/License-MIT-yellow.svg"
echo "Java: https://img.shields.io/badge/Java-21-ED8B00?logo=openjdk&logoColor=white"
echo "Python: https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white"
```

Verifica que cada badge mostra la informacio correcta renderitzant el README a GitHub.

### 6. Fixar el Repositori al Perfil (5 min)

1. Ves a `github.com/username`.
2. "Customize your pins".
3. Selecciona `esportspulse-engine` com a primer repositori fixat.

### 7. Commit Final (10 min)

```bash
cd ~/esportspulse-engine
git add .github/ README.md LICENSE CONTRIBUTING.md
git commit -m "chore: polish repository with badges, templates, and metadata"
```

---

## Checklist de Lliurament

- [ ] Descripcio del repositori configurada a GitHub
- [ ] Topics/tags afegits (minim 5)
- [ ] Social preview image pujada
- [ ] README amb badges funcionals
- [ ] LICENSE creat (MIT o equivalent)
- [ ] CONTRIBUTING.md revisat
- [ ] Plantilla de Bug Report creada
- [ ] Plantilla de Feature Request creada
- [ ] Plantilla de Pull Request creada
- [ ] Repositori fixat al perfil de GitHub
- [ ] Tots els links del README funcionen
- [ ] Commit amb tots els canvis
