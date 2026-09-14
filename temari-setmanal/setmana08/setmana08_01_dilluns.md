# Setmana 8 — Dilluns: Què és Docker i Per Què Ho Necessites

## Objectiu del Dia

Entendre què és Docker, per què existeix i com funciona per dins. Al final del dia has de poder executar contenidors de PostgreSQL i Redis, connectar-t'hi des de la terminal, i inspeccionar-los com a processos del sistema.

---

## Teoria

### El Problema que Docker Resol

Fins ara el teu projecte EsportsPulse utilitza H2 com a base de dades. H2 és còmode perquè s'executa dins del procés Java (in-process) — no cal instal·lar res. Però el món real no funciona així:

- PostgreSQL necessita una instal·lació, un usuari del sistema, configuració de ports...
- Qdrant (la base de dades vectorial que faràs servir per al servei Python d'IA) necessita un binari compilat en Rust.
- Cada company del teu equip pot tenir macOS, Linux o Windows — i cada SO instal·la les coses de manera diferent.

Això genera el problema clàssic: **"A la meva màquina funciona."** El codi compila, els tests passen, però quan un altre dev es clona el repo, la base de dades no arrenca, li falta una versió, o el port està ocupat.

**Docker elimina aquest problema.** Empaqueta el programari i totes les seves dependències dins d'un contenidor aïllat que funciona igual a qualsevol màquina.

> **Analogia:** Pensa en un contenidor de transport marítim. No importa si el vaixell va de Barcelona a Xangai — el contenidor és estàndard, el contingut viatja intacte. Docker fa el mateix amb el programari.

### Contenidors vs Màquines Virtuals

A la Setmana 3 vas aprendre què és el kernel del sistema operatiu — el nucli que gestiona processos, memòria i hardware. Ara necessites entendre una diferència clau:

**Màquina Virtual (VM):**
- Emula un ordinador complet (CPU, RAM, disc, kernel propi).
- Necessita un hipervisor (VirtualBox, VMware) que simula el hardware.
- Cada VM té el seu propi sistema operatiu complet (pot ser 2-10 GB).
- Arrenca en minuts.

**Contenidor Docker:**
- Comparteix el kernel del sistema amfitrió (el teu macOS o Linux).
- No emula hardware — utilitza funcionalitats del kernel de Linux (namespaces i cgroups) per aïllar processos.
- Només conté l'aplicació i les seves dependències (pot ser 50-500 MB).
- Arrenca en segons.

```
┌─────────────────────────────────────┐   ┌─────────────────────────────────────┐
│         MÀQUINA VIRTUAL             │   │           CONTENIDOR                │
├─────────────────────────────────────┤   ├─────────────────────────────────────┤
│  App A     App B     App C          │   │  App A     App B     App C          │
│  Libs A    Libs B    Libs C         │   │  Libs A    Libs B    Libs C         │
│  Guest OS  Guest OS  Guest OS       │   │  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│  ───────────────────────────────    │   │  Docker Engine                      │
│  Hipervisor                         │   │  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│  ───────────────────────────────    │   │  Kernel del Host (compartit)        │
│  Hardware                           │   │  Hardware                           │
└─────────────────────────────────────┘   └─────────────────────────────────────┘
```

**Per què importa?** Un contenidor és essencialment un procés aïllat del teu sistema. Recordes `ps aux` de la Setmana 3? Els contenidors apareixen com a processos normals. Aquesta és la raó per la qual són tan lleugers.

### Arquitectura de Docker

Docker té quatre components principals:

**1. Docker Daemon (`dockerd`):**
El servei que corre en segon pla al teu sistema. Gestiona la creació, execució i destrucció de contenidors. Quan escrius `docker run`, l'ordre va al daemon.

**2. Imatge (Image):**
Una plantilla immutable amb tot el necessari per executar una aplicació: sistema de fitxers, binaris, configuració. Una imatge és com una **classe** en POO (Setmana 2) — defineix l'estructura però no fa res per si sola.

**3. Contenidor (Container):**
Una instància en execució d'una imatge. Si la imatge és la classe, el contenidor és l'**objecte**. Pots crear múltiples contenidors a partir de la mateixa imatge, cadascun amb el seu estat.

```java
// Analogia amb POO (Setmana 2):
// La imatge és la classe
class PostgreSQL { ... }

// El contenidor és l'objecte (instància)
PostgreSQL dbProducció = new PostgreSQL();   // contenidor 1
PostgreSQL dbTesting = new PostgreSQL();     // contenidor 2
```

**4. Registre (Registry):**
Un magatzem d'imatges. **Docker Hub** és el registre públic per defecte (com GitHub però per a imatges Docker). Quan fas `docker pull postgres`, descarregues la imatge oficial de PostgreSQL des de Docker Hub.

### Comandes Essencials

```bash
# Descarregar una imatge des de Docker Hub
docker pull <imatge>

# Crear i executar un contenidor a partir d'una imatge
# -d = detached (en segon pla, com el & de bash que vas veure a S3)
# --name = assignar un nom al contenidor
docker run -d --name <nom> <imatge>

# Llistar contenidors en execució (com ps aux per a processos normals)
docker ps

# Llistar TOTS els contenidors (inclosos els aturats)
docker ps -a

# Aturar un contenidor (envia SIGTERM, com kill -15 de S3)
docker stop <nom_o_id>

# Eliminar un contenidor aturat
docker rm <nom_o_id>

# Llistar imatges descarregades al teu sistema
docker images

# Veure els logs d'un contenidor (com tail -f d'un fitxer de log)
docker logs <nom_o_id>

# Veure els processos dins d'un contenidor (connexió directa amb ps de S3)
docker top <nom_o_id>
```

**Ports:** Un contenidor és aïllat — per defecte no pots accedir-hi des de fora. Necessites **mapejar ports** amb `-p`:

```bash
# -p host:contenidor — mapeja el port 5432 del teu sistema al 5432 del contenidor
# Això és com un túnel: quan accedeixis a localhost:5432, arribes al PostgreSQL del contenidor
docker run -d -p 5432:5432 --name pg postgres
```

**Variables d'entorn:** Moltes imatges es configuren amb variables d'entorn (`-e`):

```bash
# -e defineix variables dins del contenidor
# POSTGRES_PASSWORD és obligatòria per a la imatge oficial de PostgreSQL
docker run -d -e POSTGRES_PASSWORD=secret --name pg postgres
```

---

## Activitat

### 1. Instal·lar Docker (10 min)

Verifica que Docker està instal·lat:

```bash
# Comprova la versió de Docker (ha de ser 24.x o superior)
docker --version

# Verifica que el daemon està corrent (ha de mostrar info del sistema)
docker info
```

Si no el tens, instal·la [Docker Desktop](https://www.docker.com/products/docker-desktop/). A macOS:

```bash
# Instal·la Docker Desktop amb Homebrew
brew install --cask docker
```

Obre Docker Desktop un cop per acceptar les condicions. Després pots tancar la finestra — el daemon continuarà corrent en segon pla.

### 2. Executar PostgreSQL amb Docker (20 min)

Descarrega i executa PostgreSQL:

```bash
# Descarrega la imatge oficial de PostgreSQL 16 des de Docker Hub
# :16 és el "tag" — indica la versió concreta (com un git tag)
docker pull postgres:16

# Executa un contenidor de PostgreSQL amb:
# -d            → en segon pla (detached)
# --name pg16   → li posem nom "pg16" per referir-nos-hi fàcilment
# -p 5432:5432  → mapegem el port perquè sigui accessible des del host
# -e            → configurem la contrasenya obligatòria
docker run -d \
  --name pg16 \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=esportspulse \
  -e POSTGRES_DB=esportspulse_db \
  postgres:16
```

Verifica que funciona:

```bash
# Llista els contenidors en execució — hauries de veure pg16
docker ps

# Mira els logs del contenidor — hauries de veure "database system is ready to accept connections"
docker logs pg16
```

Connecta't a PostgreSQL des de la terminal:

```bash
# Executa psql dins del contenidor (com fer ssh però per a contenidors)
# exec = executa una comanda dins d'un contenidor existent
# -it  = interactiu + pseudo-terminal (com el -t de ssh)
docker exec -it pg16 psql -U postgres -d esportspulse_db
```

Un cop dins de `psql`, prova algunes comandes:

```sql
-- Crea una taula de prova per verificar que la DB funciona
CREATE TABLE equips (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    joc VARCHAR(50) NOT NULL
);

-- Insereix una fila de prova
INSERT INTO equips (nom, joc) VALUES ('T1', 'League of Legends');

-- Consulta les dades — hauries de veure la fila inserida
SELECT * FROM equips;

-- Surt de psql
\q
```

### 3. Executar Redis amb Docker (15 min)

Redis és una base de dades en memòria que s'utilitza com a cache. L'executaràs al costat de PostgreSQL:

```bash
# Descarrega i executa Redis 7 en un sol pas
# Si la imatge no existeix localment, docker run fa pull automàticament
docker run -d \
  --name redis7 \
  -p 6379:6379 \
  redis:7

# Verifica que els dos contenidors estan corrent
docker ps
```

Connecta't a Redis:

```bash
# Executa el client de Redis dins del contenidor
docker exec -it redis7 redis-cli
```

Prova comandes bàsiques de Redis:

```bash
# SET guarda un valor associat a una clau (com un HashMap de Java)
SET jugador:1 "Faker"

# GET recupera el valor d'una clau
GET jugador:1

# KEYS mostra totes les claus que coincideixen amb el patró
KEYS *

# EXIT per sortir
EXIT
```

### 4. Contenidors com a Processos (15 min)

Aquesta part connecta Docker amb el que vas aprendre a la Setmana 3 sobre processos:

```bash
# docker top mostra els processos dins del contenidor
# Fixa't que els PIDs són visibles des del host — és un procés normal!
docker top pg16

# Compara-ho amb ps aux del host — trobaràs el procés de postgres
# grep filtra la sortida (recordes pipes de S3?)
ps aux | grep postgres

# Mira els recursos que consumeix cada contenidor (com htop)
# Ctrl+C per sortir
docker stats
```

**Experiment:** atura i elimina un contenidor per veure què passa amb les dades:

```bash
# Atura PostgreSQL (les dades es perden perquè no hem configurat volums!)
docker stop pg16

# Verifica que ja no apareix a docker ps
docker ps

# Però sí que apareix com a aturat a docker ps -a
docker ps -a

# Elimina el contenidor
docker rm pg16

# Ara ja no apareix enlloc
docker ps -a
```

> **Lliçó important:** Les dades dins d'un contenidor són **efímeres** — quan elimines el contenidor, les dades desapareixen. Demà aprendràs a usar **volums** per persistir dades.

### 5. Recrea PostgreSQL i deixa'l llest (5 min)

Torna a crear el contenidor per tenir-lo disponible la resta de la setmana:

```bash
# Recrea PostgreSQL amb la mateixa configuració
docker run -d \
  --name pg16 \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=esportspulse \
  -e POSTGRES_DB=esportspulse_db \
  postgres:16

# Verifica que arrenca correctament
docker logs -f pg16
# Ctrl+C quan vegis "database system is ready to accept connections"
```

Llista totes les imatges que tens descarregades:

```bash
# Mostra les imatges locals — hauries de veure postgres:16 i redis:7
docker images
```

---

## Checklist de Lliurament

- [ ] Docker instal·lat i funcionant (`docker info` sense errors)
- [ ] Contenidor `pg16` executant-se amb PostgreSQL 16 al port 5432
- [ ] Contenidor `redis7` executant-se amb Redis 7 al port 6379
- [ ] Has connectat a PostgreSQL amb `psql` i has creat una taula
- [ ] Has connectat a Redis amb `redis-cli` i has fet SET/GET
- [ ] Has executat `docker top` i has vist els processos del contenidor
- [ ] Has vist els logs del contenidor amb `docker logs`
- [ ] Has experimentat amb aturar i eliminar un contenidor (dades efímeres)
