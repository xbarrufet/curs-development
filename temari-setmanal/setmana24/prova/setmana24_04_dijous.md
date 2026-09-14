# Setmana 24 — Dijous: Demo: Preparar la Presentacio de 5 Minuts

## Objectiu del Dia

Preparar una presentacio de 5 minuts del projecte EsportsPulse que puguis fer davant d'un tribunal, un reclutador, o un equip tecnic. Al final del dia, has de tenir el guio escrit, la demo practicada, i la capacitat d'explicar les decisions tecniques amb confianca.

---

## Teoria

### Estructura d'una Demo Tecnica de 5 Minuts

Cinc minuts semblen poc, pero es el temps habitual en entrevistes tecniques, demo days, i presentacions de projectes. Cada segon compta.

```
# Estructura recomanada (5 minuts):
#
# [0:00 - 0:30] HOOK — Captura l'atencio
#   "Imagina que ets analista d'esports i necessites..."
#
# [0:30 - 1:30] PROBLEMA I SOLUCIO — Que fa i per que
#   El problema, la solucio, i per que es interessant
#
# [1:30 - 3:30] DEMO EN VIU — Mostra que funciona
#   Login, cerca, consulta a l'agent, resposta
#   Mostra el dashboard de monitoritzacio (30s)
#
# [3:30 - 4:30] ARQUITECTURA — Com esta fet
#   Diagrama C4 Nivell 2 (containers)
#   2-3 decisions tecniques clau
#
# [4:30 - 5:00] TANCAMENT — Que he apres
#   Llicons principals i seguents passos
```

### El Hook: Els Primers 30 Segons

Els primers segons determinen si l'audiencia prestara atencio o mirara el mobil:

```
# MAL hook:
# "Hola, em dic X i us presentare el meu projecte de classe
#  que es una API amb Java i Python..."
# (generic, avorrit, no genera interes)

# BON hook:
# "L'any passat, els esports electronics van generar 1.8 bilions
#  de dolars. Pero els analistes encara busquen dades manualment
#  en 15 fonts diferents. EsportsPulse resol aixo amb agents d'IA
#  que analitzen dades i documents automaticament."
# (dada impactant, problema clar, solucio en una frase)

# ALTERNATIVA (demo-first):
# [Obre el dashboard, fa una consulta]
# "Acabo de preguntar 'Quina es la taxa de victoria de T1 
#  aquesta temporada?' i en 3 segons tinc una analisi completa
#  amb dades reals. Aixo es EsportsPulse."
# (mostra el resultat primer, despres explica com)
```

### La Demo en Viu: Preparacio

La demo en viu es la part mes arriscada i mes impactant. Prepara-la be:

```
# Regles d'una bona demo:
#
# 1. PREPARA L'ENTORN ABANS
#    - Tot ha d'estar arrencat (docker-compose up)
#    - Tingues el token JWT ja obtingut
#    - El navegador obert a la pagina correcta
#    - Tanca notificacions, email, Slack
#
# 2. TINGUES UN PLA B
#    - Si la demo falla, tingues screenshots preparats
#    - Si Internet cau, tingues tot en local
#    - Si l'agent triga massa, tingues una resposta cached
#
# 3. NARRA EL QUE PASSES
#    - "Ara faig login com a analista..."
#    - "Busco l'equip T1..."
#    - "Pregunto a l'agent sobre la seva taxa de victoria..."
#    - "L'agent busca les dades, les analitza, i em dona..."
#
# 4. MOSTRA EL QUE IMPORTA
#    - NO mostris el codi (no hi ha temps)
#    - SI mostra el resultat (la resposta de l'agent)
#    - SI mostra l'arquitectura (diagrama simple)
#    - SI mostra les metriques (Grafana, rapidament)
```

### Explicar Decisions Tecniques

Quan et preguntin "per que has triat X?", la resposta ha de seguir aquest patro:

```
# Estructura: PROBLEMA -> OPCIONS -> DECISIO -> RESULTAT

# Exemple 1: "Per que Java i Python en comptes d'un sol llenguatge?"
# PROBLEMA: Necessitava un backend robust per a dades i un servei
#           d'IA amb acces a l'ecosistema de ML.
# OPCIONS: Tot en Python (simple pero Spring Boot es mes madur
#          per a APIs empresarials), tot en Java (poc suport per
#          a LangChain i agents), o poligueta (mes complexitat
#          pero el millor de cada mon).
# DECISIO: Poligueta. Java per al backend de dades, Python per a IA.
# RESULTAT: Puc usar JPA i Spring Security per al backend, i
#           LangChain amb FastAPI per als agents.

# Exemple 2: "Per que Redis a mes de PostgreSQL?"
# PROBLEMA: Les crides a l'API de Claude triguen 3-10 segons
#           i costen diners.
# OPCIONS: No cachejar (car i lent), cachejar a PostgreSQL
#          (funciona pero no es optim per a cache), Redis
#          (dissenyat per a cache, molt rapid).
# DECISIO: Redis per a cache de respostes d'agents.
# RESULTAT: La mateixa pregunta repetida respon en <100ms
#           en comptes de 5 segons, i no costa res addicional.

# Exemple 3: "Per que agents en comptes de prompts directes?"
# PROBLEMA: Diferents preguntes requereixen accedir a diferents
#           dades (BD, documents, APIs externes).
# OPCIONS: Prompts fixos (limitats), cadenes de LangChain
#          (mes flexibles), agents amb eines (maxima flexibilitat).
# DECISIO: Agents amb eines que poden decidir quines dades buscar.
# RESULTAT: L'agent pot respondre preguntes que no havia previst
#           perque selecciona les eines adequades automaticament.
```

---

## Activitat

### 1. Escriure el Guio de la Presentacio (30 min)

Crea `docs/demo/presentation-script.md`:

```markdown
# Guio de la Presentacio — EsportsPulse Engine

## [0:00 - 0:30] Hook
<!-- Escriu les paraules exactes que diras.
     Practica-les fins que et surtin naturalment. -->

## [0:30 - 1:30] Problema i Solucio
<!-- Explica el problema que resols.
     Explica la solucio en 2-3 frases.
     NO entris en detalls tecnics encara. -->

## [1:30 - 3:30] Demo en Viu
### Passos de la demo:
1. <!-- Que obriré primer? -->
2. <!-- Quina accio faré? -->
3. <!-- Que espero que passi? -->
4. <!-- Que mostraré a continuacio? -->

### Narracíó de cada pas:
<!-- "Ara faig login com a analista..." -->

### Pla B (si la demo falla):
<!-- Quins screenshots tinc preparats? -->

## [3:30 - 4:30] Arquitectura i Decisions
### Diagrama a mostrar:
<!-- C4 Nivell 2 (containers) -->

### 3 decisions tecniques que explicare:
1. <!-- Java + Python: per que poligueta? -->
2. <!-- Agents amb eines: per que no prompts fixos? -->
3. <!-- Docker + CI/CD: com garanteixo qualitat -->

## [4:30 - 5:00] Tancament
### Que he apres:
<!-- 2-3 punts clau -->

### Seguents passos:
<!-- Que faria si tingues mes temps? -->
```

### 2. Preparar l'Entorn de Demo (15 min)

```bash
# Arrenca tot el sistema
docker-compose up -d

# Verifica que tot funciona
./smoke-test.sh

# Obte un token JWT (per si el necessites)
TOKEN=$(curl -s -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"test_user","password":"test_password"}' | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

# Prepara les pestanyes del navegador:
# 1. Dashboard Streamlit (http://localhost:8501)
# 2. Diagrama d'arquitectura (GitHub o local)
# 3. Grafana amb un dashboard (http://localhost:3000)
```

### 3. Practicar la Demo (30 min)

Practica 3 vegades seguides cronometrant-te:

```
# Primera practica:
# - Llegeix el guio en veu alta
# - Cronometra't (objectiu: 5 minuts)
# - Apunta on t'encalles o passes de temps

# Segona practica:
# - Ajusta el guio (elimina el que sobra)
# - Practica les transicions entre seccions
# - Verifica que la demo funciona sense problemes

# Tercera practica:
# - Sense mirar el guio (o mirant-lo poc)
# - Cronometra't (has d'estar entre 4:30 i 5:30)
# - Grava't amb el mobil per revisar-te
```

### 4. Preparar Respostes a Preguntes Habituals (20 min)

Escriu respostes per a les preguntes mes probables:

```markdown
# Preguntes i Respostes Preparades

## "Per que no has usat X en comptes de Y?"
<!-- Respon amb el patro PROBLEMA -> OPCIONS -> DECISIO -> RESULTAT -->

## "Quins son els principals reptes que has tingut?"
<!-- Sigues honest. Exemples:
     - Configurar la comunicacio entre Java i Python
     - Gestionar la latencia dels agents
     - Aprendre Docker Compose des de zero -->

## "Com escalaries aquest sistema?"
<!-- Pensa en:
     - Mes repliques del backend (load balancer)
     - Cache mes agressiu (Redis)
     - Cues per a peticions d'agents (RabbitMQ ja ho fa)
     - Kubernetes si cal mes control -->

## "Quin es el deute tecnic que tens?"
<!-- Sigues honest. Exemples:
     - Cobertura de tests podria ser mes alta
     - L'agent podria tenir mes eines
     - Monitoring podria tenir alertes automatiques -->

## "Que faries diferent si tornessis a comencar?"
<!-- Reflexiona de veritat. Exemples:
     - Hauria comencat amb tests des del dia 1
     - Hauria usat TypeScript per al frontend
     - Hauria documentat les decisions abans de codificar -->
```

### 5. Preparar el Pla B (10 min)

```bash
# Fes screenshots de la demo per si falla en directe
# Guarda'ls a docs/demo/screenshots/

mkdir -p docs/demo/screenshots

# Screenshot 1: Dashboard amb dades
# Screenshot 2: Resposta de l'agent
# Screenshot 3: Grafana amb metriques
# Screenshot 4: Swagger UI amb endpoints
```

### 6. Commit (10 min)

```bash
git add docs/demo/
git commit -m "docs: add presentation script and demo preparation materials"
```

---

## Checklist de Lliurament

- [ ] Guio escrit amb timing per a cada seccio
- [ ] Hook que captura l'atencio (no generic)
- [ ] Demo practicada minim 3 vegades
- [ ] Cronometrat entre 4:30 i 5:30
- [ ] 3 decisions tecniques preparades amb patro PROBLEMA -> DECISIO
- [ ] Respostes a 5+ preguntes habituals preparades
- [ ] Entorn de demo preparat i verificat
- [ ] Pla B amb screenshots per si la demo falla
- [ ] Transicions entre seccions fluides
- [ ] Commit amb el material de presentacio
