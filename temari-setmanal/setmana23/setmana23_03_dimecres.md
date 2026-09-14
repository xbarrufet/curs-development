# Setmana 23 — Dimecres: Diagrames C4: Arquitectura Visual

## Objectiu del Dia

Crear els tres nivells de diagrames C4 (Context, Container, Component) del projecte EsportsPulse utilitzant eines basades en text (Mermaid o PlantUML). Al final del dia, tindras una representacio visual completa de l'arquitectura que podras incloure al README i a la documentacio.

---

## Teoria

### Que es el Model C4?

El model C4 (creat per Simon Brown) es una forma estandaritzada de visualitzar l'arquitectura de software. Te 4 nivells, com fer zoom en un mapa:

```
# Nivell 1: Context
# "Qui usa el sistema i amb que es connecta?"
# Mostra: el sistema com una caixa, els usuaris, i els sistemes externs
# Public: tothom (stakeholders, managers, devs)

# Nivell 2: Container
# "Quins contenidors (serveis) te el sistema?"
# Mostra: aplicacions, bases de dades, cues de missatges
# Public: equip tecnic

# Nivell 3: Component
# "Com esta organitzat internament cada contenidor?"
# Mostra: controladors, serveis, repositoris dins d'un servei
# Public: desenvolupadors del servei

# Nivell 4: Code (opcional)
# "Com es el codi?"
# Mostra: diagrames UML de classes
# Normalment no es fa (el codi ja esta al repositori)
```

### Diagrama de Context (Nivell 1)

El diagrama de context mostra el sistema des de fora:

```mermaid
%% Diagrama C4 Nivell 1: Context
%% Mostra: qui usa EsportsPulse i amb que es connecta

graph TB
    %% Actors (persones que usen el sistema)
    User["Usuari<br/>(Analista d'esports)"]
    Admin["Administrador"]
    
    %% El nostre sistema (una sola caixa)
    System["EsportsPulse<br/>[Sistema de Software]<br/>Plataforma d'analisi<br/>d'esports electronics"]
    
    %% Sistemes externs
    Claude["Claude API<br/>[Sistema Extern]<br/>Model de llenguatge<br/>per als agents"]
    RiotAPI["Riot Games API<br/>[Sistema Extern]<br/>Dades d'equips<br/>i competicions"]
    
    %% Relacions
    User -->|"Consulta estadistiques<br/>i demana analisis"| System
    Admin -->|"Gestiona equips<br/>i configuracio"| System
    System -->|"Envia prompts<br/>i rep respostes"| Claude
    System -->|"Obte dades<br/>de competicions"| RiotAPI
```

### Diagrama de Containers (Nivell 2)

El diagrama de containers mostra els serveis interns:

```mermaid
%% Diagrama C4 Nivell 2: Containers
%% Mostra: els serveis que componen EsportsPulse

graph TB
    User["Usuari"]
    
    subgraph EsportsPulse["EsportsPulse System"]
        %% Frontend
        Streamlit["Dashboard<br/>[Container: Streamlit]<br/>Interficie d'usuari<br/>per a visualitzacio i consultes"]
        
        %% Backend
        JavaAPI["Backend API<br/>[Container: Java/Spring Boot]<br/>API REST per a<br/>dades i autenticacio"]
        
        %% AI Service
        PythonAI["Servei d'IA<br/>[Container: Python/FastAPI]<br/>Agents per a analisi<br/>i knowledge retrieval"]
        
        %% Data stores
        Postgres[("PostgreSQL<br/>[Container: Database]<br/>Dades d'equips,<br/>jugadors, usuaris")]
        Redis[("Redis<br/>[Container: Cache]<br/>Cache de resultats<br/>i sessions")]
        RabbitMQ["RabbitMQ<br/>[Container: Message Broker]<br/>Comunicacio asincrona<br/>entre serveis"]
        
        %% Monitoring
        Prometheus["Prometheus<br/>[Container: Monitoring]<br/>Recol·leccio de metriques"]
        Grafana["Grafana<br/>[Container: Dashboard]<br/>Visualitzacio de metriques"]
    end
    
    %% External
    Claude["Claude API"]
    
    %% Relacions
    User --> Streamlit
    Streamlit -->|"HTTP/REST"| JavaAPI
    Streamlit -->|"HTTP/REST"| PythonAI
    JavaAPI -->|"JDBC"| Postgres
    JavaAPI -->|"Cache reads/writes"| Redis
    JavaAPI -->|"AMQP"| RabbitMQ
    PythonAI -->|"HTTP"| JavaAPI
    PythonAI -->|"AMQP"| RabbitMQ
    PythonAI -->|"API calls"| Claude
    Prometheus -->|"Scrape /metrics"| JavaAPI
    Prometheus -->|"Scrape /metrics"| PythonAI
    Grafana -->|"PromQL"| Prometheus
```

### Diagrama de Components (Nivell 3)

Zoom dins d'un contenidor (per exemple, el Backend Java):

```mermaid
%% Diagrama C4 Nivell 3: Components del Backend Java
%% Mostra: l'organitzacio interna del servei Spring Boot

graph TB
    subgraph JavaAPI["Backend Java (Spring Boot)"]
        %% Capa de presentacio
        AuthController["AuthController<br/>[Component]<br/>Login, registre, JWT"]
        TeamController["TeamController<br/>[Component]<br/>CRUD d'equips"]
        PlayerController["PlayerController<br/>[Component]<br/>CRUD de jugadors"]
        HealthController["HealthController<br/>[Component]<br/>Health check"]
        
        %% Capa de seguretat
        JwtFilter["JwtAuthFilter<br/>[Component]<br/>Filtre d'autenticacio JWT"]
        SecurityConfig["SecurityConfig<br/>[Component]<br/>Configuracio de seguretat"]
        
        %% Capa de servei
        TeamService["TeamService<br/>[Component]<br/>Logica de negoci d'equips"]
        PlayerService["PlayerService<br/>[Component]<br/>Logica de negoci de jugadors"]
        AuthService["AuthService<br/>[Component]<br/>Autenticacio i tokens"]
        
        %% Capa de persistencia
        TeamRepo["TeamRepository<br/>[Component]<br/>Acces a dades d'equips"]
        PlayerRepo["PlayerRepository<br/>[Component]<br/>Acces a dades de jugadors"]
        UserRepo["UserRepository<br/>[Component]<br/>Acces a dades d'usuaris"]
    end
    
    %% Relacions entre capes
    AuthController --> AuthService
    TeamController --> TeamService
    PlayerController --> PlayerService
    
    AuthService --> UserRepo
    TeamService --> TeamRepo
    PlayerService --> PlayerRepo
    
    JwtFilter --> AuthService
```

### Flux d'Observabilitat

Un diagrama addicional que mostra com flueixen els logs, metriques i traces:

```mermaid
%% Diagrama de flux d'observabilitat
%% Mostra: on van els logs, metriques i traces

graph LR
    %% Serveis que generen dades
    Java["Backend Java"]
    Python["Servei Python"]
    Streamlit["Dashboard"]
    
    %% Observabilitat
    Prometheus["Prometheus<br/>Metriques"]
    Grafana["Grafana<br/>Dashboards"]
    LangFuse["LangFuse<br/>Traces LLM"]
    Logs["Logs<br/>stdout/stderr"]
    
    %% Fluxos
    Java -->|"/metrics endpoint"| Prometheus
    Python -->|"/metrics endpoint"| Prometheus
    Prometheus -->|"PromQL queries"| Grafana
    
    Python -->|"Traces de crides LLM"| LangFuse
    
    Java -->|"stdout"| Logs
    Python -->|"stdout"| Logs
    Streamlit -->|"stdout"| Logs
    
    %% Correlacio
    Java -.->|"correlation-id header"| Python
```

---

## Activitat

### 1. Triar l'Eina (10 min)

**Opcio A: Mermaid (recomanat)**

Mermaid es renderitza directament a GitHub, GitLab, i moltes eines.

```bash
# No cal instal·lar res — GitHub renderitza Mermaid nativament
# Escriu els diagrames dins blocs ```mermaid``` al Markdown

# Per previsualitzar localment:
# - VS Code: extensio "Markdown Preview Mermaid Support"
# - Online: mermaid.live
```

**Opcio B: PlantUML**

```bash
# Instal·lacio amb Homebrew
brew install plantuml

# Generar imatge des d'un fitxer .puml
plantuml docs/architecture/c4-context.puml
```

### 2. Crear Diagrama de Context (20 min)

Crea `docs/architecture/c4-context.md`:

Pensa en:
- Qui son els actors que usen el sistema?
- Amb quins sistemes externs es connecta?
- Quines accions fa cada actor?

Escriu el diagrama Mermaid seguint l'exemple de la teoria, pero adaptat al TEU projecte real.

### 3. Crear Diagrama de Containers (30 min)

Crea `docs/architecture/c4-containers.md`:

Pensa en:
- Quins serveis tens al docker-compose?
- Com es comuniquen (HTTP, AMQP, JDBC)?
- Quins ports utilitzen?
- On estan les dades persistents?

```bash
# Referencia rapida: llista els serveis reals
docker-compose config --services

# Llista els ports exposats
docker-compose config | grep -A 5 "ports:"
```

### 4. Crear Diagrama de Components (30 min)

Crea `docs/architecture/c4-components-java.md` i `docs/architecture/c4-components-python.md`:

Per al backend Java:

```bash
# Llista les classes del controlador
find backend-java -name "*Controller.java" -type f

# Llista les classes del servei
find backend-java -name "*Service.java" -type f

# Llista els repositoris
find backend-java -name "*Repository.java" -type f
```

Per al servei Python:

```bash
# Llista els routers
find ai-python -name "*router*" -o -name "*route*" | grep -v __pycache__

# Llista els serveis
find ai-python -name "*service*" | grep -v __pycache__

# Llista els agents
find ai-python -name "*agent*" | grep -v __pycache__
```

### 5. Crear Diagrama d'Observabilitat (15 min)

Crea `docs/architecture/observability-flow.md`:

Inclou:
- D'on surten les metriques (endpoints `/metrics`)
- On van els logs (stdout -> plataforma)
- Com es propaguen els correlation IDs
- On es consulten les metriques (Grafana, LangFuse)

### 6. Integrar els Diagrames (15 min)

Afegeix els diagrames al CLAUDE.md i/o README:

```markdown
## Arquitectura

### Diagrama de Context
<!-- Mermaid diagram aqui -->

### Diagrama de Containers
<!-- Mermaid diagram aqui -->

Per als diagrames de components, consulta:
- [Components del Backend Java](docs/architecture/c4-components-java.md)
- [Components del Servei Python](docs/architecture/c4-components-python.md)
```

### 7. Commit (10 min)

```bash
git add docs/architecture/
git commit -m "docs: add C4 architecture diagrams (context, containers, components)"
```

---

## Checklist de Lliurament

- [ ] Diagrama de Context (Nivell 1) amb actors i sistemes externs
- [ ] Diagrama de Containers (Nivell 2) amb tots els serveis
- [ ] Diagrama de Components (Nivell 3) per al backend Java
- [ ] Diagrama de Components (Nivell 3) per al servei Python
- [ ] Diagrama de flux d'observabilitat
- [ ] Diagrames en format Mermaid (renderitzables a GitHub)
- [ ] Diagrames integrats al CLAUDE.md o README
- [ ] Tots els serveis reals del projecte reflectits als diagrames
- [ ] Commit amb tots els diagrames
