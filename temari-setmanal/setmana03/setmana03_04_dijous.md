# Setmana 3 — Dijous: Bash Scripting: Automatitzar Tasques

## Objectiu del Dia

Escriure scripts Bash que automatitzin tasques repetitives del projecte. Al final del dia has de tenir un script `build-and-test.sh` que compili Java, executi tests JUnit i pytest, i imprimeixi un resum amb colors. Has de poder fer `./build-and-test.sh` des de l'arrel del projecte i veure d'un cop d'ull si tot funciona.

---

## Teoria

### Bash Scripts: Fonaments

Un script Bash es un fitxer de text amb comandes que el sistema operatiu executa en sequeencia. En lloc d'escriure 10 comandes a ma cada cop, les poses en un fitxer i l'executes una sola vegada.

**Shebang: la primera linia**

```bash
#!/bin/bash
# El shebang (#!/bin/bash) indica al sistema operatiu QUIN interpret ha d'usar.
# Sense aquesta linia, el sistema no sap si es Python, Bash, Perl o un altre.
# Sempre ha de ser la PRIMERA linia del fitxer, sense espais abans.
```

Per que importa? Si no hi ha shebang i fas `./script.sh`, el sistema pot intentar executar-lo amb `/bin/sh` (que no es exactament Bash) i fallar en funcionalitats com arrays o `[[ ]]`.

**Variables**

```bash
# Assignacio de variables — SENSE espais al voltant del '='
# Si poses espais (NAME = "valor"), Bash interpreta NAME com una comanda i falla.
NOM_PROJECTE="EsportsPulse"
VERSIO="0.1.0"

# Per llegir el valor d'una variable, usa el prefix $
echo "Projecte: $NOM_PROJECTE versio $VERSIO"

# Variables d'entorn del sistema — ja existeixen quan obres el terminal
echo "Usuari: $USER"       # El teu nom d'usuari
echo "Directori: $PWD"     # El directori actual
echo "Home: $HOME"         # El directori home
```

**Cometes simples vs dobles**

```bash
NOM="Bash"

# Cometes dobles: expandeixen variables (substitueixen $NOM pel seu valor)
echo "Hola $NOM"       # Imprimeix: Hola Bash

# Cometes simples: text literal (no substitueixen res)
echo 'Hola $NOM'       # Imprimeix: Hola $NOM

# Regla practica: usa cometes dobles quan vulguis variables dins del text,
# i cometes simples quan vulguis el text exacte sense cap substitucio.
```

**Substitucio de comandes**

```bash
# $(...) executa una comanda i retorna el seu output com a text.
# Es com dir: "executa aixo i guarda el resultat en una variable".
DATA_ACTUAL=$(date +"%Y-%m-%d")
echo "Avui es: $DATA_ACTUAL"

# Exemple practic: capturar la versio de Java
JAVA_VERSION=$(java --version 2>&1 | head -n 1)
echo "Java: $JAVA_VERSION"

# Nota: 2>&1 redirigeix stderr a stdout perque algunes eines
# imprimeixen la versio per stderr en lloc de stdout.
```

### Codis de Sortida (Exit Codes)

Cada comanda que executes retorna un nombre entre 0 i 255 quan acaba:
- **0** = tot ha anat be (exit)
- **Qualsevol altre nombre** = error (1 es generic, 2 es us incorrecte, etc.)

```bash
# La variable especial $? conte el codi de sortida de l'ULTIMA comanda
ls /tmp                  # Directori que existeix
echo "Codi: $?"          # Imprimeix: Codi: 0 (exit)

ls /directori_inexistent # Directori que no existeix
echo "Codi: $?"          # Imprimeix: Codi: 2 (error)

# Aixo es la base de TOTA l'automatitzacio:
# executar comanda → comprovar si ha funcionat → decidir que fer
```

### Condicionals

```bash
# Estructura basica: if [ condicio ]; then ... fi
# IMPORTANT: els espais dins de [ ] son obligatoris.
# [ -f fitxer ] funciona; [-f fitxer] NO funciona.

# Comprovar si un fitxer existeix
if [ -f "pom.xml" ]; then
    # -f comprova si el fitxer existeix i es un fitxer regular (no directori)
    echo "pom.xml trobat — es un projecte Maven"
else
    echo "pom.xml no trobat — no es un projecte Maven"
fi

# Comprovar si un directori existeix
if [ -d "backend-java" ]; then
    # -d comprova si existeix i es un directori
    echo "Directori backend-java trobat"
fi

# Comprovar el codi de sortida d'una comanda
mvn compile
if [ $? -eq 0 ]; then
    # -eq compara numeros (equal). $? es el codi de l'ultima comanda.
    echo "Compilacio correcta"
else
    echo "Error de compilacio"
fi

# Comprovar si una variable esta buida
RESULTAT=""
if [ -z "$RESULTAT" ]; then
    # -z retorna true si la cadena es buida (zero length)
    echo "No hi ha resultat"
fi
```

**Operadors de comparacio numerica:**

| Operador | Significat | Exemple |
|----------|-----------|---------|
| `-eq` | Igual (equal) | `[ $a -eq 0 ]` |
| `-ne` | Diferent (not equal) | `[ $a -ne 0 ]` |
| `-gt` | Mes gran (greater than) | `[ $a -gt 5 ]` |
| `-lt` | Mes petit (less than) | `[ $a -lt 10 ]` |
| `-ge` | Mes gran o igual | `[ $a -ge 1 ]` |
| `-le` | Mes petit o igual | `[ $a -le 100 ]` |

### Bucles

```bash
# FOR: iterar sobre una llista d'elements
# Aqui iterem sobre tots els fitxers .java del directori actual
for fitxer in *.java; do
    # $fitxer pren el valor de cada fitxer trobat, un per un
    echo "Fitxer Java trobat: $fitxer"
done

# FOR amb llista explicita
for eina in "java" "python3" "mvn" "git"; do
    # Comprovem si cada eina esta instal·lada
    if command -v "$eina" > /dev/null 2>&1; then
        # command -v retorna la ruta de l'eina si existeix
        # > /dev/null descarta l'output (nomes volem el codi de sortida)
        echo "$eina: instal·lada"
    else
        echo "$eina: NO trobada"
    fi
done

# WHILE: llegir un fitxer linia per linia
while read -r linia; do
    # read -r llegeix una linia sense interpretar backslashes
    # -r es important per evitar que \n, \t, etc. es transformin
    echo "Linia: $linia"
done < fitxer.txt
# El < redirigeix el contingut del fitxer com a input del while
```

### Funcions

```bash
# Definicio d'una funcio — agrupa comandes reutilitzables
function comprovar_eina() {
    # $1 es el primer argument que rep la funcio
    # Les funcions Bash reben arguments per posicio: $1, $2, $3...
    local nom_eina="$1"  # 'local' limita la variable a dins la funcio

    if command -v "$nom_eina" > /dev/null 2>&1; then
        echo "$nom_eina esta instal·lada"
        return 0  # return 0 = exit, equivalent a "true"
    else
        echo "$nom_eina NO esta instal·lada"
        return 1  # return 1 = error, equivalent a "false"
    fi
}

# Cridar la funcio
comprovar_eina "java"

# Capturar l'output d'una funcio
function obtenir_versio_java() {
    # Imprimim NOMES el que volem capturar (res mes!)
    java --version 2>&1 | head -n 1
}

# $(...) captura tot el que la funcio imprimeix amb echo/printf
VERSIO=$(obtenir_versio_java)
echo "Java version: $VERSIO"
```

---

## Activitat

### Script Complet: `build-and-test.sh`

Crea el fitxer `build-and-test.sh` a l'arrel del projecte `esportspulse-engine`. L'has d'escriure pas per pas, comprovant que cada seccio funciona abans de continuar.

### Pas 1: Estructura basica i colors (15 min)

Crea el fitxer amb l'esquelet inicial:

```bash
#!/bin/bash
# =============================================================================
# build-and-test.sh — Script d'automatitzacio per al projecte EsportsPulse
# Comprova eines, compila Java, executa tests JUnit i pytest, i mostra un resum.
# Us: ./build-and-test.sh des de l'arrel del projecte
# =============================================================================

# --- Codis de color ANSI per a output formatat ---
# Aquests codis especials fan que el terminal mostri text en colors.
# \033[ inicia una seqeencia de color, i el nombre indica el color.
# 0m al final (RESET) torna al color normal.
VERD='\033[0;32m'    # Per a missatges d'exit (compilacio OK, tests OK)
VERMELL='\033[0;31m' # Per a missatges d'error (compilacio FAIL, eina no trobada)
GROC='\033[1;33m'    # Per a avisos (warnings, informacio important)
BLAU='\033[0;34m'    # Per a encapcalaments de seccio
RESET='\033[0m'      # Torna al color per defecte del terminal

# --- Variables globals per rastrejar l'estat de cada pas ---
# Inicialitzem tot a "FAIL". Si un pas va be, canviem a "OK".
# Aixi, si un pas es salta per error, queda marcat com a FAIL automaticament.
ESTAT_JAVA="FAIL"
ESTAT_PYTHON="FAIL"
ESTAT_COMPILACIO="FAIL"
ESTAT_TESTS_JAVA="FAIL"
ESTAT_TESTS_PYTHON="FAIL"

# --- Funcions auxiliars per imprimir missatges amb format ---
function info() {
    # Imprimeix un missatge informatiu en blau
    # -e activa la interpretacio de seqeencies d'escapament (\033)
    echo -e "${BLAU}[INFO]${RESET} $1"
}

function ok() {
    # Imprimeix un missatge d'exit en verd
    echo -e "${VERD}[OK]${RESET} $1"
}

function error() {
    # Imprimeix un missatge d'error en vermell
    echo -e "${VERMELL}[ERROR]${RESET} $1"
}

function warning() {
    # Imprimeix un avis en groc
    echo -e "${GROC}[AVIS]${RESET} $1"
}

function seccio() {
    # Imprimeix un separador visual per a cada fase del build
    # Fa mes facil trobar cada seccio a l'output del terminal
    echo ""
    echo -e "${BLAU}============================================${RESET}"
    echo -e "${BLAU}  $1${RESET}"
    echo -e "${BLAU}============================================${RESET}"
}
```

Prova que funciona:

```bash
# Dona permisos d'execucio al script
chmod +x build-and-test.sh

# Executa'l — hauria de mostrar res (encara no cridem cap funcio)
./build-and-test.sh
```

### Pas 2: Comprovacio d'eines (15 min)

Afegeix aquestes funcions al script, despres de les funcions auxiliars:

```bash
# =============================================================================
# FASE 1: Comprovacio d'eines necessaries
# Abans de compilar res, verifiquem que Java, Maven, Python i pytest existeixen.
# Si falta una eina critica, no te sentit continuar.
# =============================================================================

function comprovar_java() {
    seccio "Comprovant Java"

    # Comprovem si la comanda 'java' existeix al sistema
    if ! command -v java > /dev/null 2>&1; then
        # 'command -v' retorna la ruta de l'executable si existeix
        # '!' nega la condicio: si NO existeix, entrem aqui
        # '> /dev/null 2>&1' descarta tot l'output (nomes volem el codi de sortida)
        error "Java no esta instal·lat. Instal·la JDK 21: brew install openjdk@21 (macOS) o descarrega d'adoptium.net (Windows)"
        return 1  # Retornem 1 (error) per indicar que ha fallat
    fi

    # Java existeix — ara comprovem la versio
    # Capturem l'output de 'java --version' (que va per stderr, per aixo 2>&1)
    local versio_completa
    versio_completa=$(java --version 2>&1 | head -n 1)

    # Extraiem nomes el numero de versio major (21, 17, 11, etc.)
    # grep -oE busca el patro i retorna NOMES el que coincideix
    # [0-9]+ captura una seqeencia de digits
    # head -n 1 agafa nomes el primer numero trobat
    local versio_major
    versio_major=$(echo "$versio_completa" | grep -oE '[0-9]+' | head -n 1)

    # Comprovem que la versio es 21 o superior
    if [ "$versio_major" -ge 21 ]; then
        # -ge = greater or equal (mes gran o igual)
        ok "Java $versio_major detectat ($versio_completa)"
        ESTAT_JAVA="OK"  # Actualitzem l'estat global
        return 0
    else
        error "Java $versio_major detectat, pero necessitem 21+. Actualitza: brew install openjdk@21 (macOS) o descarrega d'adoptium.net (Windows)"
        return 1
    fi
}

function comprovar_python() {
    seccio "Comprovant Python"

    # Comprovem si Python 3 existeix
    if ! command -v python3 > /dev/null 2>&1; then
        error "Python 3 no esta instal·lat. Instal·la: brew install python@3.12 (macOS) o descarrega de python.org (Windows)"
        return 1
    fi

    # Capturem la versio de Python
    local versio_completa
    versio_completa=$(python3 --version 2>&1)
    # L'output es tipus "Python 3.12.1" — extraiem el numero menor (12)
    local versio_menor
    versio_menor=$(echo "$versio_completa" | grep -oE '[0-9]+\.[0-9]+' | head -n 1 | cut -d'.' -f2)
    # cut -d'.' -f2 agafa el segon camp separat per '.' (el numero menor)

    if [ "$versio_menor" -ge 11 ]; then
        ok "Python 3.$versio_menor detectat ($versio_completa)"
        ESTAT_PYTHON="OK"
        return 0
    else
        warning "Python 3.$versio_menor detectat. Recomanem 3.11+."
        ESTAT_PYTHON="OK"  # No es bloquejant, nomes un avís
        return 0
    fi
}

function comprovar_maven() {
    # Maven es imprescindible per compilar Java — si no existeix, aturem
    if ! command -v mvn > /dev/null 2>&1; then
        error "Maven no esta instal·lat. Instal·la: brew install maven (macOS) o descarrega de maven.apache.org (Windows)"
        return 1
    fi
    ok "Maven detectat: $(mvn --version 2>&1 | head -n 1)"
    return 0
}

function comprovar_pytest() {
    # pytest es necessari per als tests Python
    if ! command -v pytest > /dev/null 2>&1; then
        # Si pytest no existeix com a comanda global, provem amb python3 -m pytest
        if python3 -m pytest --version > /dev/null 2>&1; then
            ok "pytest detectat (via python3 -m pytest)"
            return 0
        fi
        warning "pytest no trobat. Instal·la: pip install pytest"
        return 1
    fi
    ok "pytest detectat: $(pytest --version 2>&1)"
    return 0
}
```

### Pas 3: Compilacio i tests (15 min)

Afegeix les funcions de build:

```bash
# =============================================================================
# FASE 2: Compilacio del projecte Java
# Executem 'mvn compile' i comprovem si ha anat be.
# =============================================================================

function compilar_java() {
    seccio "Compilant Java (mvn compile)"

    # Comprovem que pom.xml existeix (estem al directori correcte?)
    if [ ! -f "pom.xml" ]; then
        # ! nega la condicio: si NO existeix el fitxer
        error "pom.xml no trobat. Executa l'script des de l'arrel del projecte."
        return 1
    fi

    info "Executant mvn compile..."
    # Executem Maven i guardem l'output en un fitxer temporal
    # Aixi podem mostrar nomes les linies rellevants si falla
    local log_compilacio="/tmp/esportspulse-compile.log"
    mvn compile > "$log_compilacio" 2>&1
    local codi_sortida=$?
    # Guardem $? immediatament perque qualsevol comanda posterior el sobreescriuria

    if [ $codi_sortida -eq 0 ]; then
        ok "Compilacio Java correcta"
        ESTAT_COMPILACIO="OK"
        return 0
    else
        error "Compilacio Java ha fallat (codi de sortida: $codi_sortida)"
        # Mostrem les ultimes 15 linies del log per ajudar a debugar
        echo "--- Ultimes linies del log ---"
        tail -n 15 "$log_compilacio"
        echo "--- Fi del log ---"
        return 1
    fi
}

# =============================================================================
# FASE 3: Execucio de tests
# Primer els tests Java (JUnit via Maven), despres els tests Python (pytest).
# =============================================================================

function executar_tests_java() {
    seccio "Executant Tests Java (mvn test)"

    info "Executant mvn test..."
    local log_tests="/tmp/esportspulse-test-java.log"
    mvn test > "$log_tests" 2>&1
    local codi_sortida=$?

    if [ $codi_sortida -eq 0 ]; then
        # Extraiem el resum de tests del log de Maven
        # La linia que ens interessa te el format: "Tests run: X, Failures: Y, Errors: Z"
        local resum
        resum=$(grep -E "Tests run:" "$log_tests" | tail -n 1)

        if [ -n "$resum" ]; then
            # -n es l'oposat de -z: retorna true si la cadena NO esta buida
            ok "Tests Java correctes — $resum"
        else
            ok "Tests Java correctes (cap test trobat o format no reconegut)"
        fi
        ESTAT_TESTS_JAVA="OK"
        return 0
    else
        error "Tests Java han fallat (codi de sortida: $codi_sortida)"
        echo "--- Ultimes linies del log ---"
        tail -n 20 "$log_tests"
        echo "--- Fi del log ---"
        return 1
    fi
}

function executar_tests_python() {
    seccio "Executant Tests Python (pytest)"

    # Comprovem si hi ha directori Python al projecte
    if [ ! -d "ai-python" ]; then
        warning "Directori ai-python/ no trobat — saltem tests Python"
        ESTAT_TESTS_PYTHON="SKIP"
        return 0  # No es un error, simplement no hi ha codi Python
    fi

    # Comprovem si hi ha fitxers de test Python
    # find retorna els fitxers trobats; wc -l compta les linies
    local num_tests
    num_tests=$(find ai-python -name "test_*.py" -o -name "*_test.py" 2>/dev/null | wc -l | tr -d ' ')
    # tr -d ' ' elimina espais en blanc que macOS afegeix al output de wc

    if [ "$num_tests" -eq 0 ]; then
        warning "Cap fitxer de test Python trobat a ai-python/"
        ESTAT_TESTS_PYTHON="SKIP"
        return 0
    fi

    info "Trobats $num_tests fitxer(s) de test Python"
    local log_tests="/tmp/esportspulse-test-python.log"

    # Provem amb pytest directe o via python3 -m
    if command -v pytest > /dev/null 2>&1; then
        pytest ai-python/ -v > "$log_tests" 2>&1
    else
        python3 -m pytest ai-python/ -v > "$log_tests" 2>&1
    fi
    local codi_sortida=$?

    if [ $codi_sortida -eq 0 ]; then
        ok "Tests Python correctes"
        ESTAT_TESTS_PYTHON="OK"
        return 0
    else
        error "Tests Python han fallat (codi de sortida: $codi_sortida)"
        tail -n 15 "$log_tests"
        return 1
    fi
}
```

### Pas 4: Resum final i execucio (15 min)

Afegeix la funcio de resum i el bloc principal que ho crida tot:

```bash
# =============================================================================
# FASE 4: Resum final
# Mostra un resum clar de tots els passos amb colors.
# =============================================================================

function mostrar_resum() {
    seccio "RESUM FINAL"

    # Funcio local per formatar cada linia del resum
    # Rep dos arguments: $1 = nom del pas, $2 = estat (OK, FAIL, SKIP)
    function linia_resum() {
        local nom="$1"
        local estat="$2"

        # Triem el color i el simbol segons l'estat
        if [ "$estat" = "OK" ]; then
            echo -e "  ${VERD}PASS${RESET}  $nom"
        elif [ "$estat" = "SKIP" ]; then
            echo -e "  ${GROC}SKIP${RESET}  $nom"
        else
            echo -e "  ${VERMELL}FAIL${RESET}  $nom"
        fi
    }

    echo ""
    linia_resum "Java instal·lat" "$ESTAT_JAVA"
    linia_resum "Python instal·lat" "$ESTAT_PYTHON"
    linia_resum "Compilacio Java" "$ESTAT_COMPILACIO"
    linia_resum "Tests Java (JUnit)" "$ESTAT_TESTS_JAVA"
    linia_resum "Tests Python (pytest)" "$ESTAT_TESTS_PYTHON"
    echo ""

    # Determinem l'estat global: si qualsevol pas es FAIL, el build es FAIL
    if [ "$ESTAT_JAVA" = "FAIL" ] || [ "$ESTAT_COMPILACIO" = "FAIL" ] || \
       [ "$ESTAT_TESTS_JAVA" = "FAIL" ] || [ "$ESTAT_TESTS_PYTHON" = "FAIL" ]; then
        # || es l'operador OR: si QUALSEVOL condicio es true, entrem aqui
        # \ al final de linia indica que la linia continua (per llegibilitat)
        echo -e "${VERMELL}Resultat: BUILD FAILED${RESET}"
        return 1
    else
        echo -e "${VERD}Resultat: BUILD SUCCESSFUL${RESET}"
        return 0
    fi
}

# =============================================================================
# BLOC PRINCIPAL — Aqui comenca l'execucio real de l'script
# Cridem cada funcio en ordre. Si una fase critica falla, mostrem el resum
# i sortim amb codi d'error.
# =============================================================================

# Registrem el temps d'inici per mostrar la durada total
INICI=$(date +%s)  # Segons des de 1970-01-01 (epoch time)

echo -e "${BLAU}================================================${RESET}"
echo -e "${BLAU}  EsportsPulse — Build & Test Automatitzat${RESET}"
echo -e "${BLAU}  $(date '+%Y-%m-%d %H:%M:%S')${RESET}"
echo -e "${BLAU}================================================${RESET}"

# Fase 1: Comprovar eines
comprovar_java
# Si Java falla, no te sentit continuar (no podem compilar)
if [ "$ESTAT_JAVA" = "FAIL" ]; then
    error "Java es obligatori. Aturant."
    mostrar_resum
    exit 1  # exit 1 acaba l'script amb codi d'error
fi

comprovar_python
comprovar_maven
# Si Maven falla, no podem compilar
if ! comprovar_maven > /dev/null 2>&1; then
    error "Maven es obligatori. Aturant."
    mostrar_resum
    exit 1
fi

comprovar_pytest

# Fase 2: Compilar
compilar_java
# Si la compilacio falla, no te sentit executar tests
if [ "$ESTAT_COMPILACIO" = "FAIL" ]; then
    warning "Compilacio fallida — saltem els tests."
    mostrar_resum
    exit 1
fi

# Fase 3: Tests
executar_tests_java
executar_tests_python

# Fase 4: Resum
# Calculem el temps total d'execucio
FI=$(date +%s)
DURADA=$((FI - INICI))  # Aritmetica Bash: $((...)) calcula expressions numeriques
info "Temps total: ${DURADA} segons"

mostrar_resum
# L'script surt amb el codi de retorn de mostrar_resum (0=ok, 1=fail)
exit $?
```

### Pas 5: Provar l'script (15 min)

```bash
# 1. Assegura't que l'script es executable
chmod +x build-and-test.sh

# 2. Executa'l des de l'arrel del projecte
./build-and-test.sh

# 3. Comprova el codi de sortida (hauria de ser 0 si tot va be)
echo "Codi de sortida: $?"

# 4. Prova que detecta errors: canvia el pom.xml per forcar un error
# (afegeix un caracter invàlid i torna a executar — hauria de mostrar FAIL)

# 5. Fes commit de l'script
git add build-and-test.sh
git commit -m "feat(scripts): add automated build and test script for Java + Python"
```

### Referencia rapida: operadors de test

Per consultar en qualsevol moment mentre escrius scripts:

| Operador | Tipus | Significat |
|----------|-------|-----------|
| `-f fitxer` | Fitxer | Existeix i es fitxer regular |
| `-d directori` | Fitxer | Existeix i es directori |
| `-x fitxer` | Fitxer | Existeix i es executable |
| `-z "$var"` | String | La cadena es buida |
| `-n "$var"` | String | La cadena NO es buida |
| `"$a" = "$b"` | String | Les cadenes son iguals |
| `$a -eq $b` | Nombre | Iguals |
| `$a -ne $b` | Nombre | Diferents |
| `$a -gt $b` | Nombre | a mes gran que b |
| `$a -lt $b` | Nombre | a mes petit que b |

---

## Checklist de Lliurament

- [ ] `build-and-test.sh` existeix a l'arrel del projecte
- [ ] L'script te shebang (`#!/bin/bash`) a la primera linia
- [ ] L'script es executable (`chmod +x`) — pots verificar amb `ls -l build-and-test.sh`
- [ ] Comprova que Java i Python estan instal·lats (amb versio minima)
- [ ] Executa `mvn compile` i comprova el codi de sortida
- [ ] Executa `mvn test` i mostra el resum de tests
- [ ] Executa `pytest` sobre `ai-python/` (o mostra SKIP si no hi ha tests)
- [ ] Mostra un resum final amb colors (verd/vermell)
- [ ] Cada linia de l'script te un comentari explicant que fa i per que
- [ ] L'script esta comitejat: `feat(scripts): add automated build and test script`
