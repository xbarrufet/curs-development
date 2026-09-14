# Setmana 3 — Divendres: Xarxes Basiques, SSH i curl — Exercici Integrador

## Objectiu del Dia

Entendre com es comuniquen les maquines a la xarxa: adreces IP, ports, DNS. Connectar-se a una maquina remota amb SSH. Fer peticions HTTP des del terminal amb `curl`. Tancar la setmana amb un script bash integrador que descarrega dades d'una API publica, les processa amb `jq` i genera un informe. Al final del dia tens coneixements de xarxa suficients per entendre Docker networking (S8), REST APIs (S9) i desplegament al nuvol (S22), i un script funcional que demostra tot el que has apres aquesta setmana.

---

## Teoria

### 1. Xarxes Basiques: IP, Ports i DNS

#### Adreces IP — La "Direccio Postal" de Cada Maquina

Cada maquina connectada a una xarxa te una adreca IP. Es com una direccio postal: identifica de forma unica on es la maquina.

```
# Direccions IP que veuràs constantment:
127.0.0.1       → localhost, la teva pròpia màquina
192.168.1.x     → xarxa local (casa/oficina)
10.0.0.x        → xarxa privada (Docker, VPN)
142.250.185.14  → IP pública (per exemple, Google)
```

**Per que importa:** Quan arrenques Spring Boot i veus `Tomcat started on port 8080`, el servidor escolta a `127.0.0.1:8080`. Quan a S8 connectis contenidors Docker, cada un tindra la seva propia IP dins d'una xarxa virtual.

```bash
# Comprovar la connexió amb una màquina remota
# ping envia paquets i mesura el temps de resposta (latència)
ping -c 4 google.com
# -c 4 → envia només 4 paquets (sense -c, no para mai)

# Sortida esperada:
# PING google.com (142.250.185.14): 56 data bytes
# 64 bytes from 142.250.185.14: icmp_seq=0 ttl=117 time=12.3 ms
# --- google.com ping statistics ---
# 4 packets transmitted, 4 packets received, 0% packet loss
```

#### Ports — Les "Portes" de Cada Maquina

Una IP identifica la maquina, pero una maquina pot tenir molts serveis. El **port** identifica quin servei concret vols contactar. Es com el numero de pis d'un edifici: l'IP es l'adreça, el port es la porta concreta.

```
Port    Servei              Quan el veuràs
────    ──────              ──────────────
22      SSH                 Connectar-te a servidors remots (avui)
80      HTTP                Web no encriptada
443     HTTPS               Web encriptada (el navegador usa això)
5432    PostgreSQL          Base de dades (S5-S6)
8080    Spring Boot         El teu backend Java (S9+)
8000    FastAPI/Django      El teu servei Python
3000    React/Next.js       Frontend (si n'uses un)
6333    Qdrant              Base de dades vectorial (S14)
```

**Per que importa:** Quan a Docker (S8) defineixis `ports: "8080:8080"`, estaras mapejant el port 8080 del contenidor al port 8080 de la teva maquina. Si dos serveis intenten usar el mateix port, un dels dos fallara amb `Address already in use`.

```bash
# Veure quins ports estan en ús a la teva màquina
# lsof = List Open Files (a Unix, tot és un fitxer, inclosos els sockets de xarxa)
lsof -i -P -n | grep LISTEN
# -i → mostra connexions de xarxa
# -P → mostra números de port (no noms)
# -n → no resol IPs a noms (més ràpid)

# Sortida típica:
# java    1234 user   50u  IPv6 0x...  TCP *:8080 (LISTEN)
# postgres 567 user   10u  IPv6 0x...  TCP *:5432 (LISTEN)
```

#### DNS — El "Llistí Telefonic" d'Internet

Quan escrius `github.com` al navegador, el teu ordinador no sap on es `github.com`. Necessita traduir el nom a una IP. Aixo ho fa el **DNS** (Domain Name System): un sistema distribuit que converteix noms llegibles en adreces IP.

```
Tu escrius:       github.com
El DNS resol:     github.com → 140.82.121.4
El navegador va:  140.82.121.4:443 (HTTPS)
```

```bash
# nslookup — consultar el DNS per un nom de domini
# Pregunta: "Quina IP té github.com?"
nslookup github.com
# Sortida:
# Name:    github.com
# Address: 140.82.121.4

# dig — eina més detallada (útil per debugging DNS)
# Mostra el registre complet de DNS amb TTL (temps de cache)
dig github.com
# Busca la línia "ANSWER SECTION" — allà hi ha la IP

# Cas pràctic: si docker-compose no resol un nom de servei,
# saber com funciona DNS t'ajudarà a diagnosticar-ho (S8)
```

**Resum visual de la connexio:**

```
El teu navegador:  https://github.com/esportspulse
                        │
         ┌──────────────┘
         ▼
   DNS: github.com → 140.82.121.4
         │
         ▼
   Connexió TCP a 140.82.121.4:443
         │
         ▼
   El servidor GitHub respon amb la pàgina
```

---

### 2. SSH — Connectar-se a Maquines Remotes

#### Que es SSH?

SSH (Secure Shell) es un protocol per connectar-te a una maquina remota de forma segura i encriptada. Tot el que escriguis viatge xifrat — ningu pot interceptar les teves comandes ni contrasenyes.

**Per que importa:** A S22 desplegaras el teu codi a un servidor al nuvol. Hi accedeixes amb SSH. Tambe es com GitHub verifica la teva identitat quan fas `git push` amb clau SSH.

```bash
# Connexió bàsica: ssh usuari@adreça-de-la-màquina
# Exemple: connectar-se a un servidor remot
ssh alumne@192.168.1.100
# Demana la contrasenya de l'usuari "alumne" al servidor

# Un cop connectat, estàs al terminal del servidor remot
# Totes les comandes que escriguis s'executen ALLÀ, no a casa teva
# Per sortir: escriu "exit" o prem Ctrl+D
```

#### Claus SSH — Connexio Sense Contrasenya

Les claus SSH funcionen amb criptografia asimetrica: una **clau privada** (secreta, mai la comparteixis) i una **clau publica** (la dones als servidors on vols accedir).

```bash
# Generar un parell de claus SSH
ssh-keygen -t ed25519 -C "el-teu-email@exemple.com"
# -t ed25519 → algorisme modern i segur (millor que RSA per a claus noves)
# -C "..." → comentari per identificar la clau (convenció: el teu email)

# Et demana:
# 1. On guardar la clau → prem Enter per acceptar ~/.ssh/id_ed25519
# 2. Passphrase → contrasenya extra per protegir la clau (recomanat)

# Resultat: dos fitxers
# ~/.ssh/id_ed25519       → CLAU PRIVADA — MAI la comparteixis, mai la copiïs
# ~/.ssh/id_ed25519.pub   → CLAU PÚBLICA — aquesta sí la pots donar a GitHub, servidors, etc.
```

```bash
# Copiar la clau pública a un servidor remot
# Això afegeix la teva clau a ~/.ssh/authorized_keys del servidor
ssh-copy-id alumne@192.168.1.100
# Ara pots connectar-te sense contrasenya:
ssh alumne@192.168.1.100
# Entra directament!

# Per afegir la clau a GitHub:
# 1. Copia el contingut de la clau pública
cat ~/.ssh/id_ed25519.pub
# 2. Ves a GitHub → Settings → SSH and GPG keys → New SSH key
# 3. Enganxa el contingut → Add SSH key
```

#### Fitxer de Configuracio SSH — Alies per a Connexions

Si et connectes sovint a les mateixes maquines, pots crear alies al fitxer `~/.ssh/config`:

```bash
# Fitxer: ~/.ssh/config
# Cada bloc "Host" defineix un àlies per a una connexió

Host servidor-proves
    HostName 192.168.1.100      # IP o domini del servidor
    User alumne                  # Usuari amb què connectar
    Port 22                      # Port SSH (22 és el per defecte)
    IdentityFile ~/.ssh/id_ed25519  # Quina clau privada usar

Host github
    HostName github.com
    User git                     # GitHub sempre usa l'usuari "git"
    IdentityFile ~/.ssh/id_ed25519

# Ara en comptes de:
#   ssh alumne@192.168.1.100
# Pots escriure:
#   ssh servidor-proves
```

---

### 3. curl — Peticions HTTP des del Terminal

#### Per Que curl?

`curl` et permet fer peticions HTTP (les mateixes que fa el navegador) des del terminal. Aixo es essencial per:
- **Testejar APIs** sense obrir un navegador ni Postman (S9+)
- **Automatitzar** peticions dins de scripts bash
- **Depurar** problemes de xarxa veient exactament que s'envia i que es rep

#### Peticions GET — Demanar Dades

```bash
# GET és el mètode per defecte de curl — demana dades al servidor
# Això és el que fa el navegador quan escrius una URL
curl https://api.github.com
# Retorna un JSON amb informació de la API de GitHub

# Afegir capçaleres per identificar-te o especificar el format
# -H afegeix una capçalera HTTP a la petició
curl -H "Accept: application/json" https://api.github.com/users/octocat
# Accept: application/json → diu al servidor que vols la resposta en JSON

# Veure les capçaleres de resposta (útil per debugging)
# -I fa una petició HEAD — retorna NOMÉS les capçaleres, sense el cos
curl -I https://api.github.com
# Veuràs coses com:
# HTTP/2 200               → Codi de resposta (200 = OK)
# content-type: application/json  → Format de la resposta
# x-ratelimit-remaining: 58       → Quantes peticions et queden

# Descarregar un fitxer i guardar-lo amb el nom original
# -O guarda el fitxer amb el nom que té al servidor
curl -O https://exemple.com/dades.json
# Crea el fitxer "dades.json" al directori actual
```

#### Peticions POST — Enviar Dades

```bash
# POST envia dades al servidor (crear un recurs, enviar un formulari, etc.)
# -X POST → especifica el mètode HTTP
# -H "Content-Type: ..." → diu al servidor quin format tenen les dades
# -d '...' → les dades a enviar (el "body" de la petició)
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"name": "Ahri", "role": "Mage", "difficulty": 5}' \
  http://localhost:8080/api/champions
# Això crea un nou campió al teu backend (quan el tinguis a S9)

# POST amb dades des d'un fitxer (útil per JSONs grans)
# @fitxer.json → curl llegeix el contingut del fitxer com a body
curl -X POST \
  -H "Content-Type: application/json" \
  -d @champion_data.json \
  http://localhost:8080/api/champions/batch
```

#### Codis de Resposta HTTP — Que Significa Cada Numero

Cada resposta HTTP porta un codi de 3 digits. Saber-los et fa molt mes eficient depurant:

```
Codi    Significat           Quan el veuràs
────    ─────────           ──────────────
200     OK                  Tot ha anat bé
201     Created             Recurs creat (POST exitós)
204     No Content          Operació OK, sense cos de resposta (DELETE)
301     Moved Permanently   Redirecció permanent (URL ha canviat)
400     Bad Request         Has enviat dades mal formades (JSON invàlid, etc.)
401     Unauthorized        No estàs autenticat (falta token/clau)
403     Forbidden           Estàs autenticat però no tens permís
404     Not Found           La URL no existeix (la més famosa)
500     Internal Server Error  Bug al servidor (veuràs això MOLT)
503     Service Unavailable   Servidor sobrecarregat o en manteniment
```

```bash
# Veure el codi de resposta juntament amb el cos
# -w '\n%{http_code}\n' → afegeix el codi HTTP al final de la sortida
curl -w '\n%{http_code}\n' https://api.github.com/users/octocat
# L'última línia serà "200" si tot ha anat bé

# Redirigir la sortida i quedar-te només amb el codi
# -s (silent) → no mostra la barra de progrés
# -o /dev/null → descarta el cos de la resposta
curl -s -o /dev/null -w '%{http_code}' https://api.github.com
# Retorna només "200" — útil dins scripts per verificar si un servei respon
```

---

## Activitat

### Exercici Integrador: `fetch-gamedata.sh`

Escriuras un script bash que combina tot el que has apres aquesta setmana: variables, condicionals, funcions, pipes, `curl` i `jq`. El script descarrega dades de campions de League of Legends des d'una API publica (no cal autenticacio), les processa i genera un informe.

**API que usarem:** Riot Data Dragon — proporciona dades estatiques de League of Legends. Es publica, gratuita, i no requereix clau API.

### 1. Crear l'script (60 min)

Crea el fitxer a l'arrel del teu projecte:
```
scripts/fetch-gamedata.sh
```

Escriu el contingut seguent. **Llegeix cada comentari** — expliquen el per que de cada decisio:

```bash
#!/usr/bin/env bash
# fetch-gamedata.sh — Descarrega i processa dades de campions de League of Legends
# Usa l'API pública de Riot Data Dragon (no requereix autenticació)
# Combina: curl (xarxa), jq (processament JSON), pipes, funcions, variables
#
# Ús: ./fetch-gamedata.sh [--output fitxer.csv] [--top N]

# === CONFIGURACIÓ ===

# "set -euo pipefail" — les tres proteccions essencials de qualsevol script bash:
#   -e → surt immediatament si qualsevol comanda falla (no continua en silenci)
#   -u → tracta les variables no definides com a error (evita bugs per typos)
#   -o pipefail → si qualsevol comanda d'un pipe falla, tot el pipe falla
set -euo pipefail

# URL de l'API de Riot Data Dragon
# Aquesta API retorna un JSON amb TOTS els campions de League of Legends
# És pública i no requereix cap clau — perfecta per practicar
API_URL="https://ddragon.leagueoflegends.com/cdn/14.10.1/data/en_US/champion.json"

# Valors per defecte — es poden sobreescriure amb arguments
OUTPUT_FILE="champions.csv"   # Fitxer de sortida per defecte
TOP_N=0                       # 0 = mostra tots; >0 = mostra els N primers

# Directori temporal per guardar la resposta crua de l'API
# mktemp crea un fitxer temporal únic (evita col·lisions si dos scripts corren alhora)
TEMP_DIR=$(mktemp -d)
RAW_JSON="${TEMP_DIR}/champions_raw.json"

# === FUNCIONS ===

# Funció de neteja: s'executa quan l'script acaba (bé o malament)
# trap ... EXIT → registra una funció que s'executarà quan l'script surti
# Això garanteix que els fitxers temporals es netegen SEMPRE
cleanup() {
    # rm -rf esborra el directori temporal i tot el seu contingut
    rm -rf "${TEMP_DIR}"
}
trap cleanup EXIT

# Funció per imprimir missatges d'error a stderr
# stderr (>&2) és el canal d'error — separat de stdout
# Això permet que els errors es vegin fins i tot si redirigeixes stdout a un fitxer
error() {
    echo "[ERROR] $1" >&2
}

# Funció per imprimir missatges informatius
info() {
    echo "[INFO] $1"
}

# Funció per mostrar l'ús de l'script
# Es crida si l'usuari passa arguments invàlids
usage() {
    cat <<EOF
Ús: $(basename "$0") [opcions]

Descarrega dades de campions de LoL i genera un CSV.

Opcions:
  --output FITXER   Fitxer CSV de sortida (defecte: champions.csv)
  --top N           Mostra només els N campions amb més atac
  --help            Mostra aquest missatge

Exemples:
  $(basename "$0")                      # Tots els campions a champions.csv
  $(basename "$0") --output top.csv --top 10   # Top 10 per atac a top.csv
EOF
}

# === PARSEIG D'ARGUMENTS ===

# Bucle while que processa els arguments de la línia de comandes
# "$#" és el nombre d'arguments restants
# "shift" elimina el primer argument i mou els altres una posició
while [[ $# -gt 0 ]]; do
    case "$1" in
        --output)
            # El valor ve just després del flag
            OUTPUT_FILE="$2"
            shift 2   # Salta el flag i el valor (2 posicions)
            ;;
        --top)
            TOP_N="$2"
            shift 2
            ;;
        --help)
            usage
            exit 0
            ;;
        *)
            error "Argument desconegut: $1"
            usage
            exit 1
            ;;
    esac
done

# === VERIFICACIÓ DE DEPENDÈNCIES ===

# Abans de fer res, comprovem que tenim les eines necessàries
# "command -v" retorna el path de l'executable si existeix, o falla si no
# Això evita errors críptics a la meitat de l'execució
check_dependencies() {
    local missing=0

    if ! command -v curl &>/dev/null; then
        error "curl no està instal·lat. Instal·la'l amb: brew install curl (macOS), apt install curl (Linux), o ve inclòs a Windows 10+"
        missing=1
    fi

    if ! command -v jq &>/dev/null; then
        error "jq no està instal·lat. Instal·la'l amb: brew install jq (macOS), apt install jq (Linux), o choco install jq (Windows)"
        error "jq és un processador de JSON per la línia de comandes — essencial per treballar amb APIs"
        missing=1
    fi

    # Si falta alguna dependència, sortim amb codi 1 (error)
    if [[ $missing -eq 1 ]]; then
        exit 1
    fi
}

# === DESCARREGA DE DADES ===

# Funció que descarrega el JSON de l'API amb curl
fetch_data() {
    info "Descarregant dades de campions des de Data Dragon..."

    # curl amb opcions de robustesa:
    #   -s → silent, no mostra barra de progrés (volem output net)
    #   -f → fail, retorna error si el servidor respon amb 4xx/5xx
    #   --max-time 30 → timeout de 30 segons (no es queda penjat per sempre)
    #   --retry 3 → reintenta fins a 3 vegades si falla (xarxa inestable)
    #   -o → escriu la resposta al fitxer indicat
    if ! curl -sf --max-time 30 --retry 3 -o "${RAW_JSON}" "${API_URL}"; then
        error "No s'han pogut descarregar les dades. Comprova la connexió a internet."
        error "URL: ${API_URL}"
        exit 1
    fi

    # Verifiquem que el fitxer no està buit i és JSON vàlid
    # jq empty intenta parsejar el JSON — si falla, el fitxer no és JSON vàlid
    if [[ ! -s "${RAW_JSON}" ]]; then
        error "El fitxer descarregat està buit."
        exit 1
    fi

    if ! jq empty "${RAW_JSON}" 2>/dev/null; then
        error "La resposta no és un JSON vàlid."
        exit 1
    fi

    # Mostrem quantes dades hem descarregat (mida del fitxer)
    local size
    # wc -c compta els bytes del fitxer
    # awk divideix per 1024 per mostrar KB (més llegible)
    size=$(wc -c < "${RAW_JSON}" | awk '{printf "%.1f", $1/1024}')
    info "Descarregats ${size} KB de dades."
}

# === PROCESSAMENT AMB JQ ===

# Funció que transforma el JSON cru en CSV usant jq
# jq és un processador de JSON per la línia de comandes — com sed/awk però per JSON
process_data() {
    info "Processant dades amb jq..."

    # Estructura del JSON de Data Dragon:
    # {
    #   "data": {
    #     "Aatrox": { "name": "Aatrox", "tags": ["Fighter","Tank"], "info": {"attack":8,...} },
    #     "Ahri":   { "name": "Ahri",   "tags": ["Mage","Assassin"], "info": {"attack":3,...} },
    #     ...
    #   }
    # }

    # Escrivim la capçalera del CSV
    echo "name,title,tags,attack,defense,magic,difficulty" > "${OUTPUT_FILE}"

    # Explicació del filtre jq pas a pas:
    #   .data                → accedeix a l'objecte "data"
    #   | to_entries[]       → converteix l'objecte en array de {key, value} i itera
    #   | .value             → agafa el valor de cada entrada (les dades del campió)
    #   | [.name, ...]       → construeix un array amb els camps que volem
    #   | @csv               → formata l'array com a línia CSV (amb cometes si cal)
    # El resultat és una línia CSV per campió, que afegim al fitxer amb >>
    jq -r '
        .data
        | to_entries[]
        | .value
        | [
            .name,
            .title,
            (.tags | join("/")),
            .info.attack,
            .info.defense,
            .info.magic,
            .info.difficulty
          ]
        | @csv
    ' "${RAW_JSON}" >> "${OUTPUT_FILE}"

    # Comptem els campions processats
    # wc -l compta línies; restem 1 per la capçalera
    local total
    total=$(( $(wc -l < "${OUTPUT_FILE}") - 1 ))
    info "Processats ${total} campions."
}

# === GENERACIÓ DE L'INFORME ===

# Funció que imprimeix un resum formatat per la terminal
generate_report() {
    local total
    total=$(( $(wc -l < "${OUTPUT_FILE}") - 1 ))

    echo ""
    echo "=========================================="
    echo "  INFORME DE CAMPIONS — League of Legends"
    echo "=========================================="
    echo ""
    echo "  Total de campions: ${total}"
    echo ""

    # Estadístiques per rol (tags)
    # Expliquem el pipeline pas a pas:
    #   tail -n +2          → salta la capçalera (comença a la línia 2)
    #   cut -d',' -f3       → agafa el camp 3 (tags), delimiter = coma
    #   tr '/' '\n'         → cada tag en una línia separada (els tags estan separats per /)
    #   sed 's/"//g'        → elimina les cometes del CSV
    #   sort                → ordena alfabèticament (necessari per uniq)
    #   uniq -c             → compta ocurrències consecutives (per això cal sort)
    #   sort -rn            → ordena per nombre (descendent)
    echo "  Campions per rol:"
    echo "  ─────────────────"
    tail -n +2 "${OUTPUT_FILE}" \
        | cut -d',' -f3 \
        | tr '/' '\n' \
        | sed 's/"//g' \
        | sort \
        | uniq -c \
        | sort -rn \
        | while read -r count role; do
            # printf formata la sortida amb amplada fixa per alinear els números
            printf "    %-15s %3d\n" "${role}" "${count}"
          done

    echo ""

    # Top campions per atac (si s'ha demanat amb --top, o els 5 primers per defecte)
    local show_n
    if [[ ${TOP_N} -gt 0 ]]; then
        show_n=${TOP_N}
    else
        show_n=5
    fi

    echo "  Top ${show_n} campions per atac:"
    echo "  ──────────────────────────────"
    # Pipeline:
    #   tail -n +2          → salta la capçalera
    #   sort -t',' -k4 -rn  → ordena pel camp 4 (attack), numèricament, descendent
    #   head -n N           → agafa els N primers
    #   awk amb FS=","      → separa per comes i formata la sortida
    tail -n +2 "${OUTPUT_FILE}" \
        | sort -t',' -k4 -rn \
        | head -n "${show_n}" \
        | awk -F',' '{
            # Eliminem les cometes dels camps de text
            gsub(/"/, "", $1);
            gsub(/"/, "", $3);
            printf "    %-20s ATK:%-3s DEF:%-3s MAG:%-3s  [%s]\n", $1, $4, $5, $6, $3
          }'

    echo ""

    # Top campions per dificultat
    echo "  Top ${show_n} campions mes dificils:"
    echo "  ────────────────────────────────────"
    tail -n +2 "${OUTPUT_FILE}" \
        | sort -t',' -k7 -rn \
        | head -n "${show_n}" \
        | awk -F',' '{
            gsub(/"/, "", $1);
            gsub(/"/, "", $3);
            printf "    %-20s DIF:%-3s  [%s]\n", $1, $7, $3
          }'

    echo ""
    echo "  Fitxer CSV generat: ${OUTPUT_FILE}"
    echo "=========================================="
}

# === EXECUCIÓ PRINCIPAL ===

# Funció main — punt d'entrada de l'script
# Agrupar tot en una funció main és bona pràctica:
#   1. Fa explícit l'ordre d'execució
#   2. Permet declarar variables locals
#   3. Segueix el patró de qualsevol programa (C, Java, Python tenen un main)
main() {
    info "=== fetch-gamedata.sh ==="
    info "Script integrador — Setmana 3: Linux i Terminal"
    echo ""

    # Pas 1: Verificar que tenim curl i jq
    check_dependencies

    # Pas 2: Descarregar les dades de l'API
    fetch_data

    # Pas 3: Processar el JSON i generar el CSV
    process_data

    # Pas 4: Mostrar l'informe formatat
    generate_report

    info "Fet! Pots obrir ${OUTPUT_FILE} amb qualsevol editor o full de càlcul."
}

# Executem main
# Això és una convenció: definir funcions a dalt i cridar main a baix
# Garanteix que totes les funcions estan definides abans d'usar-les
main
```

### 2. Fer l'script executable i provar-lo (15 min)

```bash
# Dona permisos d'execució a l'script (recorda: dimecres vam veure els permisos)
chmod +x scripts/fetch-gamedata.sh

# Executa'l sense arguments — genera champions.csv amb tots els campions
./scripts/fetch-gamedata.sh

# Prova amb arguments — genera un fitxer diferent amb el top 10
./scripts/fetch-gamedata.sh --output top_attackers.csv --top 10

# Verifica que el CSV s'ha generat correctament
# head mostra les primeres línies — comprova que la capçalera i les dades són correctes
head -5 champions.csv

# Compta quantes línies té (campions + capçalera)
wc -l champions.csv
```

**Sortida esperada:**

```
[INFO] === fetch-gamedata.sh ===
[INFO] Script integrador — Setmana 3: Linux i Terminal

[INFO] Descarregant dades de campions des de Data Dragon...
[INFO] Descarregats 432.1 KB de dades.
[INFO] Processant dades amb jq...
[INFO] Processats 168 campions.

==========================================
  INFORME DE CAMPIONS — League of Legends
==========================================

  Total de campions: 168

  Campions per rol:
  ─────────────────
    Fighter          65
    Mage             57
    Assassin         42
    Tank             38
    Marksman         27
    Support          23

  Top 5 campions per atac:
  ──────────────────────────────
    Renekton             ATK:8   DEF:5   MAG:2    [Fighter/Tank]
    Draven               ATK:9   DEF:3   MAG:1    [Marksman]
    ...

  Fitxer CSV generat: champions.csv
==========================================
[INFO] Fet! Pots obrir champions.csv amb qualsevol editor o full de càlcul.
```

### 3. Explorar les dades amb comandes del terminal (10 min)

Usa les comandes que has apres aquesta setmana per explorar el CSV generat:

```bash
# Quants campions tenen rol "Mage"?
# grep busca línies que continguin "Mage"
# wc -l compta quantes línies coincideixen
grep "Mage" champions.csv | wc -l

# Quins campions tenen atac >= 9?
# awk pot filtrar per valor numèric d'un camp
awk -F',' '$4 >= 9 {gsub(/"/, "", $1); print $1, "ATK:"$4}' champions.csv

# Exporta només els Assassins a un fitxer separat
# head -1 → capçalera; grep → línies amb "Assassin"
head -1 champions.csv > assassins.csv
grep "Assassin" champions.csv >> assassins.csv
echo "Assassins exportats: $(( $(wc -l < assassins.csv) - 1 ))"

# Ordena per dificultat (camp 7) i mostra els 3 més fàcils
tail -n +2 champions.csv | sort -t',' -k7 -n | head -3 | cut -d',' -f1,7
```

### 4. Commit de tots els scripts de la setmana i Pull Request (15 min)

Aquesta es l'ultima tasca de la Setmana 3. Fem commit de tot el treball de la setmana i creem una PR.

```bash
# Primer, assegura't que ets a la branca de la setmana
git checkout feature/week3-linux
# Si no existeix, crea-la:
# git checkout -b feature/week3-linux

# Revisa l'estat — hauries de veure tots els scripts de la setmana
git status

# Afegeix els scripts al staging area
# Llista cada fitxer explícitament — és més segur que "git add ."
git add scripts/fetch-gamedata.sh
# Afegeix també qualsevol altre script que hagis creat dilluns-dijous:
# git add scripts/explora_sistema.sh
# git add scripts/processa_logs.sh
# git add scripts/utils.sh

# Commit amb format Conventional Commits
# feat → funcionalitat nova; (bash) → àmbit del canvi
git commit -m "feat(bash): add week 3 Linux scripts and game data fetcher"

# Puja la branca a GitHub
git push -u origin feature/week3-linux
```

Crea la Pull Request a GitHub:
- **Titol:** `feat: Week 3 — Linux terminal, bash scripting, networking`
- **Descripcio:** Explica breument que has apres (terminal, pipes, bash, SSH, curl) i que l'script integrador demostra totes les habilitats. Menciona que els scripts segueixen les bones practiques (`set -euo pipefail`, funcions, error handling).

Fusiona la PR quan estigui llesta.

---

## Checklist de Lliurament

- [ ] `fetch-gamedata.sh` s'executa sense errors i genera un CSV valid
- [ ] L'script te error handling: comprova que `curl` i `jq` existeixen, i que la descarrega ha funcionat
- [ ] L'informe mostra: total de campions, distribucio per rol, top campions per atac i dificultat
- [ ] Pots explicar que fan `ping`, `nslookup`, `ssh-keygen` i `curl -X POST`
- [ ] Saps que es un port i per que Spring Boot usa 8080
- [ ] Commit amb format Conventional Commits i PR creada/fusionada a `main`
- [ ] Branca `feature/week3-linux` eliminada despres de fusionar
