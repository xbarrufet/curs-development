# Setmana 3 — Dimarts: Permisos, Variables d'Entorn i PATH

## Objectiu del Dia

Entendre el model de permisos de fitxers Unix, dominar les variables d'entorn i comprendre com el sistema operatiu localitza els executables via PATH. Al final del dia has de poder crear scripts executables, configurar JAVA_HOME correctament i entendre per que `python3` pot apuntar a versions diferents segons el PATH.

---

## Teoria

### Permisos de Fitxers: El Model rwx

Cada fitxer i directori a Unix te tres nivells de permisos per a tres grups d'usuaris:

| Grup | Significat |
|------|-----------|
| **owner** (u) | L'usuari que ha creat el fitxer |
| **group** (g) | El grup al qual pertany el fitxer (cada usuari pertany a un o mes grups) |
| **others** (o) | Tothom qui no sigui l'owner ni estigui al grup |

Per a cada grup hi ha tres permisos:

| Permis | Lletra | Valor octal | En fitxers | En directoris |
|--------|--------|-------------|------------|---------------|
| Lectura | r | 4 | Pot llegir el contingut | Pot llistar els fitxers del directori |
| Escriptura | w | 2 | Pot modificar el contingut | Pot crear/eliminar fitxers dins |
| Execucio | x | 1 | Pot executar-lo com a programa | Pot entrar-hi amb `cd` |

```bash
# Mostra els permisos d'un fitxer amb format llarg
ls -l script.sh
# Sortida exemple:
# -rwxr-xr-- 1 alumne staff 128 Sep 14 10:00 script.sh
#  ^^^         -> owner: lectura + escriptura + execucio (7)
#     ^^^      -> group: lectura + execucio (5)
#        ^^^   -> others: lectura (4)
# El resultat en octal seria 754
```

**Notacio octal:** Cada permis te un valor numeric (r=4, w=2, x=1). Se sumen per obtenir un digit per grup:

```bash
# Exemples comuns d'octals:
# 755 = rwxr-xr-x -> owner tot, grup i others lectura+execucio
#   Cas d'us: scripts executables, directoris
# 644 = rw-r--r-- -> owner lectura+escriptura, grup i others nomes lectura
#   Cas d'us: fitxers de configuracio, codi font
# 700 = rwx------ -> nomes l'owner te acces total
#   Cas d'us: claus SSH, fitxers privats
# 600 = rw------- -> owner lectura+escriptura, ningu mes
#   Cas d'us: fitxers amb secrets (.env, credencials)
```

### chmod i chown: Canviar Permisos i Propietari

```bash
# chmod canvia els permisos d'un fitxer
# Format: chmod [octal] [fitxer]
chmod 755 script.sh        # Dona permisos d'execucio a owner, group i others

# Tambe pots usar format simbolic (mes llegible per canvis puntuals):
chmod +x script.sh         # Afegeix permis d'execucio a tothom
chmod u+x script.sh        # Afegeix permis d'execucio nomes a l'owner
chmod go-w fitxer.txt      # Treu permis d'escriptura a group i others

# chown canvia el propietari d'un fitxer
# Format: chown [usuari]:[grup] [fitxer]
# Nota: normalment requereix sudo (permisos d'administrador)
sudo chown alumne:staff projecte/   # L'usuari "alumne" del grup "staff" sera l'owner
sudo chown -R alumne:staff projecte/ # -R aplica recursivament a tots els fitxers dins
```

**Per que `./script.sh` falla amb "Permission denied"?** Quan crees un fitxer, per defecte no te permis d'execucio (x). El sistema operatiu es nega a executar-lo com a programa encara que el contingut sigui un script valid. Amb `chmod +x script.sh` li dius al sistema: "aquest fitxer es pot executar".

### Variables d'Entorn: Parells Clau-Valor del Sistema

Les variables d'entorn son parells clau-valor que el sistema operatiu passa a cada proces. Cada programa que executes hereta les variables del proces pare (la teva terminal).

```bash
# Mostra el valor d'una variable d'entorn existent
echo $HOME          # /Users/alumne — directori personal de l'usuari
echo $USER          # alumne — nom d'usuari actual
echo $SHELL         # /bin/zsh — shell per defecte
echo $LANG          # ca_ES.UTF-8 — idioma del sistema

# Llista TOTES les variables d'entorn del sistema
env                 # Mostra centenars de variables — massa per llegir d'un cop
env | grep JAVA     # Filtra nomes les que contenen "JAVA"

# Crear una variable d'entorn (nomes per a la sessio actual)
MY_VAR="hola"       # Crea la variable — SENSE espais al voltant del =
echo $MY_VAR        # hola

# IMPORTANT: sense "export", la variable nomes existeix a la shell actual.
# Si obres un subproces (per exemple executant un script), NO la veura.
export MY_VAR="hola"  # Ara qualsevol subproces heretara MY_VAR
```

### JAVA_HOME i PATH: Per que `java` Funciona des de Qualsevol Lloc

```bash
# PATH es la variable mes important per a desenvolupadors.
# Conte una llista de directoris separats per ":" on la shell busca executables.
echo $PATH
# Sortida exemple:
# /usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin

# Quan escrius "java" a la terminal, la shell:
# 1. Mira a /usr/local/bin/ — hi ha un "java" aqui? No.
# 2. Mira a /usr/bin/ — hi ha un "java" aqui? Si! L'executa.
# Si no el troba a cap directori del PATH -> "command not found"
```

```bash
# JAVA_HOME indica on esta instal·lat el JDK.
# Moltes eines (Maven, Gradle, IntelliJ) consulten aquesta variable
# per saber quina versio de Java usar.
echo $JAVA_HOME
# Sortida exemple a macOS: /Library/Java/JavaVirtualMachines/openjdk-21.jdk/Contents/Home

# "which" mostra la ruta completa d'un executable — util per saber
# QUINA versio s'esta executant realment
which java             # /usr/bin/java
which mvn              # /usr/local/bin/mvn
```

### PATH en Profunditat: L'Ordre Importa

La shell recorre els directoris del PATH **d'esquerra a dreta**. El primer executable que coincideix amb el nom, guanya. Aixo explica per que podem tenir multiples versions d'un programa i que nomes una s'executi:

```bash
# Imagina que tens dues versions de Python instal·lades:
which -a python3       # Mostra TOTES les ubicacions, no nomes la primera
# /usr/local/bin/python3   <- Python 3.12 (instal·lat amb Homebrew)
# /usr/bin/python3         <- Python 3.9 (preinstal·lat amb macOS)

# Si PATH es: /usr/local/bin:/usr/bin
# -> "python3" executa 3.12 (primer al PATH)
# Si PATH es: /usr/bin:/usr/local/bin
# -> "python3" executa 3.9 (ara /usr/bin va primer)
```

### Com Funciona `venv` Per Dins: Manipulacio del PATH

Quan crees un entorn virtual de Python (`venv`), el que fa es senzill pero potent:

```bash
# Crear un venv genera una copia local de l'interpret Python
python3 -m venv .venv
# Dins .venv/ hi ha:
# .venv/bin/python3      <- Copia/link de l'interpret
# .venv/bin/pip          <- Pip local a aquest entorn
# .venv/bin/activate     <- Script que modifica el PATH

# ABANS d'activar:
echo $PATH
# /usr/local/bin:/usr/bin:/bin

# Activar el venv:
source .venv/bin/activate

# DESPRES d'activar:
echo $PATH
# /Users/alumne/projecte/.venv/bin:/usr/local/bin:/usr/bin:/bin
# ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
# El directori del venv s'afegeix al PRINCIPI del PATH!
# Ara "python3" i "pip" apunten als del venv, no als del sistema.

which python3          # /Users/alumne/projecte/.venv/bin/python3
# Qualsevol paquet instal·lat amb pip s'instal·la DINS el venv,
# sense afectar el Python del sistema.

# Desactivar torna el PATH a l'estat original:
deactivate
which python3          # /usr/local/bin/python3
```

### `.bashrc` i `.zshrc`: Configuracio Permanent

Les variables definides a la terminal es perden en tancar-la. Per fer-les permanents, s'afegeixen al fitxer d'inici de la shell:

```bash
# .zshrc (macOS per defecte) o .bashrc (Linux per defecte)
# S'executa automaticament cada cop que obres una terminal nova.

# Per editar-lo:
nano ~/.zshrc          # Obre l'editor de text al terminal

# Exemple de linies que hi podries afegir:
export JAVA_HOME=$(/usr/libexec/java_home -v 21)  # macOS: detecta Java 21 automaticament
export PATH="$JAVA_HOME/bin:$PATH"                 # Afegeix Java 21 al PATH

# Despres de guardar, aplica els canvis sense tancar la terminal:
source ~/.zshrc        # Re-executa el fitxer de configuracio
```

> **Lectura recomanada (opcional, no bloquejant):**
> - [Linux File Permissions Explained](https://www.redhat.com/sysadmin/linux-file-permissions-explained) — Red Hat
> - [Environment Variables](https://wiki.archlinux.org/title/Environment_variables) — Arch Wiki

---

## Activitat

### 1. Crear un Script Bash i Entendre Permisos (20 min)

Crea un petit script dins el teu projecte:

```bash
# Crea un directori per a scripts d'utilitat
mkdir -p scripts

# Crea el fitxer amb un editor o directament des de la terminal:
cat > scripts/info.sh << 'EOF'
#!/bin/bash
# Shebang (primera linia): indica al sistema quin interpret usar per executar-lo.
# Sense aquesta linia, el sistema no sap si es bash, python, perl, etc.

# Mostra informacio de l'entorn de desenvolupament
echo "=== Informacio de l'Entorn ==="

echo "Usuari: $USER"                 # Variable d'entorn amb el nom d'usuari
echo "Directori actual: $(pwd)"      # $(cmd) executa el comando i insereix el resultat
echo "Java: $(java --version 2>&1 | head -1)"   # 2>&1 redirigeix errors a stdout
echo "Python: $(python3 --version)"
echo "JAVA_HOME: $JAVA_HOME"         # Pot estar buit si no l'has configurat
echo "PATH: $PATH"                   # Mostra tots els directoris on busca executables
EOF
```

Ara intenta executar-lo:

```bash
# Intent 1: executar directament — FALLARA
./scripts/info.sh
# bash: ./scripts/info.sh: Permission denied
# Per que? El fitxer no te permis d'execucio (x)

# Comprova els permisos actuals:
ls -l scripts/info.sh
# -rw-r--r--  -> 644: lectura i escriptura per owner, lectura per la resta. Cap "x".

# Dona permis d'execucio:
chmod 755 scripts/info.sh
# 755 = rwxr-xr-x -> ara tothom pot executar-lo

# Verifica el canvi:
ls -l scripts/info.sh
# -rwxr-xr-x  -> Ara te la "x"!

# Intent 2: executar — ARA FUNCIONA
./scripts/info.sh
```

**Per que `./scripts/info.sh` funciona pero `info.sh` sol no?**

```bash
# Prova executar sense el "./" :
info.sh
# zsh: command not found: info.sh

# Motiu: la shell NOMES busca executables als directoris del PATH.
# El directori actual (.) NO esta al PATH per defecte (per seguretat).
# "./" vol dir explicitament "busca al directori actual".

# Podries afegir "." al PATH, pero es PERIGOS:
# Si algu posa un fitxer malicious anomenat "ls" al directori, l'executaries sense voler.
```

### 2. Configurar JAVA_HOME (15 min)

```bash
# Comprova si JAVA_HOME ja esta configurat:
echo $JAVA_HOME
# Si esta buit o apunta a una versio incorrecta, configura'l:

# macOS: java_home es una utilitat que detecta les versions instal·lades
/usr/libexec/java_home -V          # Llista totes les versions de Java instal·lades
/usr/libexec/java_home -v 21       # Mostra la ruta de Java 21 especificament

# Configura JAVA_HOME per a la sessio actual:
export JAVA_HOME=$(/usr/libexec/java_home -v 21)

# Verifica que funciona:
echo $JAVA_HOME                    # Ha de mostrar la ruta al JDK 21
$JAVA_HOME/bin/java --version      # Executa Java directament des del JDK configurat

# Per fer-ho permanent, afegeix al teu .zshrc:
echo '' >> ~/.zshrc
echo '# Java 21 — configurat a la Setmana 3 del curs' >> ~/.zshrc
echo 'export JAVA_HOME=$(/usr/libexec/java_home -v 21)' >> ~/.zshrc
echo 'export PATH="$JAVA_HOME/bin:$PATH"' >> ~/.zshrc

# Aplica els canvis:
source ~/.zshrc

# Verifica que Maven tambe detecta Java 21:
mvn --version
# Ha de mostrar "Java version: 21.x.x"
```

### 3. Explorar el PATH amb un venv de Python (20 min)

```bash
# Situa't al directori del projecte
cd ai-python

# ABANS de crear el venv: observa el PATH i quin Python s'usa
echo "PATH actual:"
echo $PATH
echo ""
echo "Python actual:"
which python3                      # Mostra quina versio s'executa
python3 --version                  # Confirma la versio

# Crea un entorn virtual
python3 -m venv .venv              # Genera el directori .venv/ amb un Python aillat

# Observa que s'ha creat dins .venv/:
ls -la .venv/bin/                  # Veureu python3, pip, activate, etc.

# ACTIVA el venv:
source .venv/bin/activate

# DESPRES d'activar: compara el PATH
echo "PATH despres d'activar venv:"
echo $PATH
# Observa: el directori .venv/bin/ apareix al PRINCIPI del PATH

echo ""
echo "Python dins del venv:"
which python3                      # Ara apunta a .venv/bin/python3
python3 --version                  # Mateixa versio, pero des del venv

# Instal·la un paquet de prova per veure l'aillament:
pip install cowsay                 # S'instal·la DINS el venv, no al sistema
which cowsay                       # .venv/bin/cowsay
cowsay "El PATH mana"

# Desactiva el venv:
deactivate

# Verifica que tot torna a la normalitat:
which python3                      # Torna al Python del sistema
which cowsay                       # command not found — el paquet era del venv!
```

### 4. Commit (5 min)

```bash
# Afegeix l'script creat a Git
git add scripts/info.sh

# Commit amb missatge descriptiu
git commit -m "feat(scripts): add environment info script with executable permissions"
```

---

## Checklist de Lliurament

- [ ] `scripts/info.sh` es executable i mostra informacio de l'entorn
- [ ] Entens per que `chmod +x` es necessari i que significa 755
- [ ] JAVA_HOME esta configurat al `.zshrc` i `mvn --version` mostra Java 21
- [ ] Has creat un venv de Python i has observat com canvia el PATH en activar-lo
- [ ] Entens per que `./script.sh` funciona pero `script.sh` sol no
- [ ] Commit fet a la branca de treball
