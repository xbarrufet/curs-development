# Setmana 3 — Exercicis de Consolidació

Aquests exercicis repassen els conceptes clau de la setmana. No cal lliurar-los — són per verificar que has entès la teoria i la pràctica abans de passar a la setmana 4. Intenta resoldre'ls sense mirar els apunts; si et quedes encallat, revisa el dia corresponent.

> **Recordatori Windows:** Fes servir Git Bash per a tots els exercicis.

---

## Bloc 1: Sistema Operatiu i Terminal (Dilluns)

**Exercici 1.1 — Processos**

Sense executar res, respon:

1. Què és un **procés**? Quin procés es crea quan executes `java -jar app.jar`?
2. Pots tenir múltiples processos Java executant-se alhora al teu sistema? Com ho comprovaries?
3. Quina comanda fas servir per veure tots els processos Java que s'estan executant?

**Exercici 1.2 — Navegar el Sistema de Fitxers**

Escriu la comanda per a cada acció (sense executar-les):

1. Llistar tots els fitxers (inclosos els ocults) del directori actual.
2. Mostrar el directori actual complet (path absolut).
3. Trobar on està instal·lat Python al teu sistema.
4. Veure quanta memòria RAM ocupa un procés Java.

**Exercici 1.3 — $PATH**

1. Què és el `$PATH`?
2. Quan escrius `java --version` al terminal, com sap el shell on trobar l'executable `java`?
3. Si tens dues versions de Python instal·lades, quina s'executa quan escrius `python`? Com ho comproves?

---

## Bloc 2: Permisos, Variables d'Entorn i PATH (Dimarts)

**Exercici 2.1 — Permisos Unix (Teoria)**

> Nota: Això no s'aplica directament a Windows, però és essencial per a servidors Linux i Docker.

Donat el següent output de `ls -l`:

```
-rwxr-x--- 1 dev team 4096 Sep 10 09:00 build.sh
-rw-r--r-- 1 dev team 2048 Sep 10 09:00 README.md
drwxr-xr-x 3 dev team 4096 Sep 10 09:00 src/
```

1. Qui pot executar `build.sh`? (propietari, grup, altres)
2. Qualsevol persona pot llegir `README.md`?
3. Què significa la `d` al principi de `src/`?
4. Quina comanda faries servir per fer `build.sh` executable per a tothom?

**Exercici 2.2 — Variables d'Entorn**

1. Quina comanda mostra el valor de `JAVA_HOME`?
2. Quina diferència hi ha entre `export VAR=valor` i `VAR=valor` (sense export)?
3. Si configures una variable amb `export` al terminal i tanques la terminal, la variable sobreviu? Per què?
4. En quin fitxer guardes les variables perquè siguin permanents?

**Exercici 2.3 — venv i PATH**

Explica pas a pas què passa internament quan executes:

```bash
source .venv/bin/activate
```

Concretament: què canvia al `$PATH`? Per què `python` ara apunta a una versió diferent?

---

## Bloc 3: Pipes, Redirecció i Processament de Text (Dimecres)

**Exercici 3.1 — Llegeix la Pipe**

Explica què fa cada pas d'aquesta comanda:

```bash
find . -name "*.java" | xargs wc -l | sort -n | tail -5
```

**Exercici 3.2 — Escriu la Comanda**

Escriu la comanda (una sola línia amb pipes) per a cada tasca:

1. Comptar quantes vegades apareix la paraula "TODO" als fitxers `.java` del projecte.
2. Trobar els 3 fitxers `.py` més grans del projecte (per mida de fitxer).
3. Extreure tots els noms de campions d'un JSON de l'API de Riot Data Dragon i ordenar-los alfabèticament.
4. Guardar tots els errors (`stderr`) d'una compilació Maven a un fitxer `errors.log` sense que apareguin per pantalla.

**Exercici 3.3 — Redirecció**

Indica la diferència entre:

1. `mvn compile > output.txt`
2. `mvn compile >> output.txt`
3. `mvn compile 2> errors.txt`
4. `mvn compile > output.txt 2>&1`

---

## Bloc 4: Bash Scripting (Dijous)

**Exercici 4.1 — Llegeix l'Script**

Analitza aquest script i respon les preguntes:

```bash
#!/bin/bash
set -e

check_tool() {
    if ! command -v "$1" > /dev/null 2>&1; then
        echo "ERROR: $1 no trobat"
        return 1
    fi
    echo "OK: $1 trobat a $(which $1)"
}

check_tool java
check_tool python3
check_tool mvn

echo "Totes les eines instal·lades!"
```

1. Què fa `set -e`?
2. Què fa `command -v "$1"`?
3. Què significa `> /dev/null 2>&1`?
4. Si `python3` no està instal·lat, arriba a executar-se la línia `check_tool mvn`? Per què?

**Exercici 4.2 — Escriu un Script**

Escriu un script bash `count-code.sh` que:

1. Compti el nombre de fitxers `.java` al projecte
2. Compti el nombre de fitxers `.py` al projecte
3. Compti el total de línies de codi (Java + Python)
4. Imprimeixi un resum amb format:

```
=== Resum del Projecte ===
Fitxers Java:  12
Fitxers Python: 5
Línies Java:   1.234
Línies Python:   456
Total línies:  1.690
```

**Exercici 4.3 — Exit Codes**

1. Què retorna un programa quan acaba correctament? I quan falla?
2. Com comproves l'exit code de l'última comanda executada?
3. Per què és important que `build-and-test.sh` retorni el codi de sortida correcte?

---

## Bloc 5: Xarxes, SSH i curl (Divendres)

**Exercici 5.1 — Conceptes de Xarxa**

Respon sense mirar els apunts:

1. Què és una adreça IP? Quina IP té la teva pròpia màquina (localhost)?
2. Què és un port? Per què un servidor web típicament usa el port 8080?
3. Què fa un DNS? Per què pots escriure `google.com` en comptes d'una IP?
4. Quina diferència hi ha entre HTTP i HTTPS?

**Exercici 5.2 — curl**

Escriu la comanda `curl` per a cada cas:

1. Fer un GET a `https://jsonplaceholder.typicode.com/users/1` i veure la resposta formatada amb `jq`.
2. Veure només les capçaleres de la resposta (sense el body).
3. Descarregar el JSON de tots els campions de Data Dragon i guardar-lo a un fitxer `champions.json`.

**Exercici 5.3 — SSH**

1. Quina comanda genera un parell de claus SSH?
2. On es guarden les claus generades? Quina és la pública i quina la privada?
3. Per què **mai** comparteixes la clau privada?

---

## Exercici Final: Integració

Escriu un script bash `project-health.sh` que faci el següent:

1. **Comprova eines:** Verifica que `java`, `python` (o `python3`), `mvn` i `git` estan instal·lats. Si falta alguna, imprimeix un error i surt.
2. **Comprova el projecte:** Verifica que existeix `pom.xml` al directori actual. Si no, imprimeix "No ets a l'arrel del projecte" i surt.
3. **Resum del codi:** Compta fitxers `.java` i `.py`, i total de línies.
4. **Estat de Git:** Mostra la branca actual i si hi ha canvis sense commit.
5. **Test ràpid:** Executa `mvn compile` i informa si ha passat o no.
6. **Imprimeix tot** amb colors (verd per OK, vermell per errors).

> **Pista:** Reutilitza patrons del `build-and-test.sh` de dijous. L'script ha de retornar exit code 0 si tot va bé, 1 si alguna cosa falla.

---
---

# Solucions

> **Atenció:** Intenta resoldre els exercicis abans de mirar les solucions.

---

## Bloc 1: Sistema Operatiu i Terminal

**Solució 1.1 — Processos**

1. Un **procés** és una instància d'un programa en execució amb la seva pròpia memòria. Quan executes `java -jar app.jar`, es crea un procés de la JVM (Java Virtual Machine) que carrega `app.jar` i l'executa.
2. Sí — cada cop que executes `java`, es crea un procés nou i independent. Pots tenir 5 aplicacions Java executant-se alhora, cadascuna en el seu propi procés amb la seva pròpia memòria.
3. `ps -ef | grep java` (o a Git Bash Windows: `ps | grep java`)

**Solució 1.2 — Navegar el Sistema de Fitxers**

1. `ls -la`
2. `pwd`
3. `which python` o `which python3`
4. `ps aux | grep java` (mostra la columna RSS/VSZ amb la memòria)

**Solució 1.3 — $PATH**

1. `$PATH` és una variable d'entorn que conté una llista de directoris (separats per `:`) on el shell busca els executables.
2. El shell recorre els directoris del `$PATH` d'esquerra a dreta fins trobar un fitxer anomenat `java`. L'executa.
3. S'executa la primera que trobi al `$PATH`. Ho comproves amb `which python` (mostra el path de l'executable que s'executaria).

---

## Bloc 2: Permisos, Variables d'Entorn i PATH

**Solució 2.1 — Permisos Unix**

1. El **propietari** (dev) i el **grup** (team) poden executar `build.sh`. Altres no (`---`).
2. Sí — `r--` al final significa que "altres" tenen permís de lectura.
3. `d` significa que és un **directori**, no un fitxer.
4. `chmod +x build.sh` (afegeix permís d'execució) o `chmod 755 build.sh` (rwxr-xr-x).

**Solució 2.2 — Variables d'Entorn**

1. `echo $JAVA_HOME`
2. `export` fa que la variable sigui visible als processos fills (subshells, programes que llances des del terminal). Sense `export`, la variable només existeix al shell actual.
3. No sobreviu — les variables `export` viuen en memòria del procés del shell. Quan tanques el terminal, el procés mor i les variables desapareixen.
4. `~/.bashrc` (Git Bash / Linux) o `~/.zshrc` (macOS). S'executen automàticament quan obres un terminal nou.

**Solució 2.3 — venv i PATH**

Quan executes `source .venv/bin/activate`:
1. El script modifica `$PATH` afegint `.venv/bin/` al principi.
2. Ara quan escrius `python`, el shell busca primer a `.venv/bin/` i hi troba el Python del venv — no el del sistema.
3. També configura la variable `VIRTUAL_ENV` amb el path del venv.
4. El prompt canvia per mostrar `(.venv)` com a recordatori visual.

---

## Bloc 3: Pipes, Redirecció i Processament de Text

**Solució 3.1 — Llegeix la Pipe**

1. `find . -name "*.java"` — Busca tots els fitxers `.java` recursivament
2. `xargs wc -l` — Compta les línies de cada fitxer trobat
3. `sort -n` — Ordena numèricament (de menys a més línies)
4. `tail -5` — Mostra només els 5 últims (els més llargs)

Resultat: els 5 fitxers Java amb més línies de codi.

**Solució 3.2 — Escriu la Comanda**

1. `grep -r "TODO" --include="*.java" . | wc -l`
2. `find . -name "*.py" -type f -exec ls -la {} \; | sort -k5 -n | tail -3`
3. `curl -s "https://ddragon.leagueoflegends.com/cdn/15.1.1/data/en_US/champion.json" | jq '.data | keys[]' | sort`
4. `mvn compile 2> errors.log`

**Solució 3.3 — Redirecció**

1. `>` redirigeix stdout a un fitxer, **sobreescrivint-lo** si existeix.
2. `>>` redirigeix stdout a un fitxer, **afegint** al final (no sobreescriu).
3. `2>` redirigeix **stderr** (errors) a un fitxer. Stdout segueix apareixent per pantalla.
4. `2>&1` redirigeix stderr al mateix lloc que stdout. Combinat amb `>`, tot (output + errors) va al fitxer.

---

## Bloc 4: Bash Scripting

**Solució 4.1 — Llegeix l'Script**

1. `set -e` fa que l'script s'aturi immediatament si qualsevol comanda retorna un error (exit code ≠ 0).
2. `command -v "$1"` comprova si l'executable `$1` existeix al `$PATH`. Retorna 0 si el troba, 1 si no.
3. `> /dev/null` descarta l'stdout. `2>&1` redirigeix stderr al mateix lloc (també descartat). Resultat: la comanda s'executa en silenci, només ens interessa el codi de sortida.
4. **No.** `set -e` fa que l'script s'aturi quan `check_tool python3` retorna 1 (error). L'script acaba sense arribar a `check_tool mvn`.

**Solució 4.2 — Escriu un Script**

```bash
#!/bin/bash

java_files=$(find . -name "*.java" -type f | wc -l)
py_files=$(find . -name "*.py" -type f | wc -l)

java_lines=$(find . -name "*.java" -type f -exec cat {} + | wc -l)
py_lines=$(find . -name "*.py" -type f -exec cat {} + | wc -l)

total=$((java_lines + py_lines))

echo "=== Resum del Projecte ==="
printf "Fitxers Java:  %d\n" "$java_files"
printf "Fitxers Python: %d\n" "$py_files"
printf "Línies Java:   %d\n" "$java_lines"
printf "Línies Python:   %d\n" "$py_lines"
printf "Total línies:  %d\n" "$total"
```

**Solució 4.3 — Exit Codes**

1. **0** quan acaba correctament, **qualsevol valor ≠ 0** (típicament 1) quan falla.
2. `echo $?` — mostra l'exit code de l'última comanda.
3. Perquè si l'script es crida des d'un pipeline CI/CD (GitHub Actions), el sistema necessita saber si ha passat o ha fallat. Exit code 0 = el build és verd; ≠ 0 = vermell.

---

## Bloc 5: Xarxes, SSH i curl

**Solució 5.1 — Conceptes de Xarxa**

1. Una **adreça IP** identifica una màquina a la xarxa. Localhost = `127.0.0.1` — és la teva pròpia màquina.
2. Un **port** és un número que identifica un servei dins d'una màquina. 8080 és un port convencional per a servidors web de desenvolupament (el 80 és per HTTP de producció, el 443 per HTTPS).
3. Un **DNS** (Domain Name System) tradueix noms de domini (`google.com`) a adreces IP (`142.250.x.x`). Sense DNS, hauries de recordar IPs.
4. **HTTPS** = HTTP + encriptació TLS. Les dades viatgen xifrades — ningú entre tu i el servidor pot llegir-les.

**Solució 5.2 — curl**

1. `curl -s "https://jsonplaceholder.typicode.com/users/1" | jq '.'`
2. `curl -I "https://jsonplaceholder.typicode.com/users/1"`
3. `curl -s "https://ddragon.leagueoflegends.com/cdn/15.1.1/data/en_US/champion.json" -o champions.json`

**Solució 5.3 — SSH**

1. `ssh-keygen -t ed25519` (o `ssh-keygen -t rsa -b 4096`)
2. A `~/.ssh/`. La pública és `id_ed25519.pub`, la privada `id_ed25519` (sense extensió).
3. La clau privada és com la contrasenya de casa teva. Si algú la té, pot entrar a qualsevol servidor on la teva clau pública estigui autoritzada. La pública es pot compartir lliurement — serveix per verificar que tu ets tu, però no permet entrar sense la privada.

---

## Exercici Final: Integració

```bash
#!/bin/bash

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
NC='\033[0m'

ok()    { echo -e "${GREEN}✓ $1${NC}"; }
error() { echo -e "${RED}✗ $1${NC}"; }
info()  { echo -e "${YELLOW}→ $1${NC}"; }

errors=0

# 1. Comprovar eines
echo "=== Comprovació d'eines ==="
for tool in java python mvn git; do
    if command -v "$tool" > /dev/null 2>&1; then
        ok "$tool trobat a $(which $tool)"
    else
        error "$tool no trobat"
        errors=$((errors + 1))
    fi
done

# 2. Comprovar projecte
echo ""
echo "=== Comprovació del projecte ==="
if [ -f "pom.xml" ]; then
    ok "pom.xml trobat"
else
    error "No ets a l'arrel del projecte (pom.xml no trobat)"
    exit 1
fi

# 3. Resum del codi
echo ""
echo "=== Resum del codi ==="
java_files=$(find . -name "*.java" -type f | wc -l)
py_files=$(find . -name "*.py" -type f | wc -l)
java_lines=$(find . -name "*.java" -type f -exec cat {} + 2>/dev/null | wc -l)
py_lines=$(find . -name "*.py" -type f -exec cat {} + 2>/dev/null | wc -l)
total=$((java_lines + py_lines))
info "Fitxers Java: $java_files ($java_lines línies)"
info "Fitxers Python: $py_files ($py_lines línies)"
info "Total: $total línies"

# 4. Estat de Git
echo ""
echo "=== Estat de Git ==="
branch=$(git branch --show-current 2>/dev/null)
if [ -n "$branch" ]; then
    ok "Branca: $branch"
    changes=$(git status --porcelain | wc -l)
    if [ "$changes" -eq 0 ]; then
        ok "Directori net (sense canvis pendents)"
    else
        info "$changes fitxers amb canvis sense commit"
    fi
else
    error "No és un repositori Git"
    errors=$((errors + 1))
fi

# 5. Test ràpid
echo ""
echo "=== Compilació ==="
if mvn compile -q 2>/dev/null; then
    ok "mvn compile OK"
else
    error "mvn compile ha fallat"
    errors=$((errors + 1))
fi

# Resum final
echo ""
if [ "$errors" -eq 0 ]; then
    ok "Tot correcte!"
    exit 0
else
    error "$errors problemes detectats"
    exit 1
fi
```
