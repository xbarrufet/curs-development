# Setmana 3 — Dilluns: El Sistema Operatiu i el Terminal

## Objectiu del Dia

Entendre com el teu codi realment s'executa: què fa el sistema operatiu quan escrius `java MyApp`, com la JVM es converteix en un procés que ocupa RAM i cicles de CPU, i com el terminal és la teva interfície directa amb el kernel. Al final del dia sabràs navegar el sistema de fitxers amb consciència del que passa per sota, localitzar on viuen les eines que fas servir (Java, Python, Maven) i observar en temps real com la teva màquina gestiona els processos que llances.

---

## Prerequisit: Instal·lar Git Bash (Windows)

Aquesta setmana treballaràs intensivament amb el terminal. Les comandes que aprendràs (`grep`, `find`, `sort`, `ps`, pipes, bash scripting) són l'estàndard de la indústria — els servidors de producció, els pipelines de CI/CD (GitHub Actions), i els contenidors Docker que veuràs a la setmana 9 usen Linux. Per tant, és important que les aprenguis en el seu entorn natural.

Si fas servir **Windows**, obre **Git Bash** (ve instal·lat amb Git for Windows) per a totes les activitats d'aquesta setmana. Git Bash et dona un terminal bash real a Windows amb totes les comandes Unix que necessites: `ls`, `grep`, `find`, `sort`, `awk`, `sed`, `curl`, `ssh`, i molt més.

**Com obrir Git Bash:**
- Clic dret a qualsevol carpeta → "Open Git Bash here"
- O busca "Git Bash" al menú d'inici

**Per què no PowerShell?** PowerShell és potent, però usa una sintaxi completament diferent. A la indústria, els scripts de build, CI/CD, i servidors usen bash. Aprendre bash ara et prepara per a Docker (S9), GitHub Actions (S7), i qualsevol feina amb servidors Linux.

> **Nota sobre permisos Unix (dimarts):** A Windows, el model de permisos de fitxers (`rwx`, `chmod`) no existeix — Windows usa un sistema diferent (ACLs). Dimarts l'aprendràs com a teoria perquè és essencial per entendre servidors Linux i Docker, tot i que no el faràs servir directament al teu Windows del dia a dia.

---

## Teoria

### Què Passa Quan Executes `java MyApp`?

Les dues setmanes anteriors has escrit Java i Python, has compilat amb Maven, has fet benchmarks. Però, què fa realment la màquina quan executes el teu codi?

**El sistema operatiu (OS) és un gestor de recursos.** La seva feina principal és repartir tres coses entre tots els programes que corren alhora:

1. **CPU** — el processador que executa instruccions (el cervell)
2. **RAM** — la memòria ràpida on viuen les dades mentre el programa corre (l'escriptori de treball)
3. **Disc** — l'emmagatzematge permanent on viuen els fitxers (l'arxivador)

Quan escrius `java MyApp`, passa això:

```bash
# 1. El shell (bash/zsh) rep la comanda
java MyApp

# 2. El shell busca l'executable "java" al $PATH (una llista de carpetes on buscar programes)
# 3. El kernel del SO crea un PROCÉS nou — una instància del programa amb el seu propi espai de memòria
# 4. Dins d'aquest procés, la JVM (Java Virtual Machine) arrenca
# 5. La JVM carrega el fitxer MyApp.class a RAM
# 6. La JVM interpreta/compila el bytecode i el CPU l'executa
# 7. El SO assigna temps de CPU al procés (scheduling) — comparteix amb altres processos
# 8. Quan el programa acaba, el SO allibera la RAM i destrueix el procés
```

**Concepte clau:** La JVM NO és màgia — és un programa més que corre dins d'un procés del SO. Ocupa RAM (normalment entre 256 MB i diversos GB), usa temps de CPU, i el SO pot matar-la si es comporta malament.

### El Terminal: Parlar Directament amb el SO

El terminal que has fet servir per `mvn compile` o `git push` no és una eina de desenvolupament — és la interfície original del sistema operatiu. Quan escrius una comanda, això és el que passa:

```
Tu (teclat) → Terminal (emulador) → Shell (bash/zsh) → Kernel del SO → Hardware
```

- **Terminal:** la finestra on escrius (Git Bash a Windows, Terminal.app a macOS, el terminal de Cursor)
- **Shell:** el programa que interpreta les comandes (bash, zsh). Avui tens zsh per defecte a macOS
- **Kernel:** el nucli del SO que realment parla amb el hardware

### Comandes Bàsiques: Ara amb Consciència

Algunes d'aquestes ja les has fet servir. La diferència és que ara entens QUÈ fan per sota.

**Navegació — On sóc? Què hi ha aquí?**

```bash
# Mostra el directori actual (Print Working Directory)
# El shell manté sempre una "posició" dins l'arbre de fitxers
pwd

# Llista els fitxers del directori actual
# Sense arguments, mostra el directori on ets
ls

# Llista amb detalls: permisos, propietari, mida, data de modificació
# -l = format llarg (long), -a = mostra fitxers ocults (que comencen per .)
ls -la

# Canvia de directori (Change Directory)
# El kernel actualitza la posició del teu shell dins l'arbre de fitxers
cd /home/alumne/projectes

# Torna al directori anterior — útil per anar i tornar
cd -

# Torna al directori HOME (la teva carpeta personal, ~ és un àlies)
cd ~
```

**Manipulació de fitxers i directoris:**

```bash
# Crea un directori nou al sistema de fitxers
# -p = crea els directoris pare si no existeixen (no dóna error si ja existeix)
mkdir -p projecte/src/main

# Crea un fitxer buit (o actualitza la data de modificació si ja existeix)
touch notes.txt

# Copia un fitxer — el SO duplica les dades al disc
cp notes.txt notes_backup.txt

# Copia un directori sencer — -r = recursiu (inclou subdirectoris i contingut)
cp -r projecte/ projecte_backup/

# Mou o reanomena un fitxer (internament, el SO actualitza la referència al disc)
mv notes.txt apunts.txt

# Elimina un fitxer — ATENCIÓ: no hi ha paperera al terminal, és permanent
rm apunts.txt

# Elimina un directori i tot el que conté — -r = recursiu, -f = força (no demana confirmació)
# PERILL: rm -rf és irreversible. Sempre verifica què estàs esborrant
rm -rf projecte_backup/
```

**Lectura de fitxers — Veure contingut sense obrir un editor:**

```bash
# Mostra TOT el contingut d'un fitxer a la terminal (bo per fitxers petits)
cat pom.xml

# Mostra el fitxer pàgina a pàgina (bo per fitxers llargs)
# Navega amb espai (avançar), b (enrere), q (sortir)
less pom.xml

# Mostra les primeres 10 línies d'un fitxer (útil per veure capçaleres)
head pom.xml

# Mostra les primeres 20 línies (amb -n canvies el nombre)
head -n 20 pom.xml

# Mostra les últimes 10 línies (útil per veure logs recents)
tail application.log

# Mostra les últimes línies en temps real — el terminal es queda escoltant
# Molt útil per monitorar logs d'un servidor. Ctrl+C per sortir
tail -f application.log
```

### L'Arbre de Fitxers: On Viu Tot

El sistema de fitxers de Linux/macOS és un arbre invertit que comença a `/` (arrel):

```
/                        ← Arrel del sistema. Tot penja d'aquí
├── home/ (o Users/)     ← Carpetes personals dels usuaris
│   └── alumne/          ← La teva carpeta (~ és un àlies d'aquí)
├── etc/                 ← Fitxers de configuració del sistema
│   └── hosts            ← Mapeig de noms de domini a IPs (DNS local)
├── tmp/                 ← Fitxers temporals (el SO els pot esborrar)
├── usr/                 ← Programes instal·lats per l'usuari
│   ├── bin/             ← Executables dels programes (java, python3, git, mvn)
│   └── lib/             ← Llibreries compartides
├── var/                 ← Dades variables (logs, bases de dades)
│   └── log/             ← Logs del sistema
└── bin/                 ← Executables essencials del sistema (ls, cp, mv)
```

### $PATH: Com el Shell Troba els Programes

Quan escrius `java`, el shell no busca per tot el disc. Consulta la variable `$PATH`, que conté una llista de directoris separats per `:`.

```bash
# Mostra el contingut de $PATH — és una llista de carpetes on buscar executables
echo $PATH
# Exemple de sortida: /usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin

# "which" et diu EXACTAMENT on viu un executable
# El shell recorre $PATH d'esquerra a dreta fins trobar-lo
which java       # Exemple: /usr/bin/java
which python3    # Exemple: /usr/local/bin/python3
which mvn        # Exemple: /usr/local/bin/mvn

# "whereis" busca l'executable, les pàgines de manual i el codi font
whereis java
```

Per això quan instal·les un programa i el terminal diu "command not found", normalment vol dir que el directori on s'ha instal·lat no està al `$PATH`.

### Processos: Veure Què Corre a la Teva Màquina

Cada programa en execució és un **procés** amb un identificador únic (PID):

```bash
# Mostra els processos que corren a la teva sessió de terminal
ps

# Mostra TOTS els processos del sistema amb detalls
# -e = every (tots), -f = full (format complet amb PID, usuari, comanda, hora)
ps -ef

# Filtra processos per nom — el pipe (|) envia la sortida de ps a grep
# grep busca línies que continguin el text "java"
ps -ef | grep java

# "top" mostra els processos en temps real, ordenats per ús de CPU
# És com el Monitor d'Activitat però al terminal. q per sortir
top

# "htop" és una versió millorada de top amb colors i navegació
# macOS: brew install htop / Windows Git Bash: top funciona, htop no (usa el Gestor de Tasques)
htop
```

---

## Activitat

### 1. Explorar el Sistema de Fitxers (15 min)

Obre el terminal i navega per l'arbre de fitxers per entendre on ets i què hi ha:

```bash
# Comprova on ets ara — hauria de ser el teu HOME
pwd

# Mira què hi ha al directori arrel del sistema
ls -la /

# Explora les carpetes del sistema
ls /usr/bin | head -20    # Mostra els primers 20 executables instal·lats
ls /etc | head -20        # Mostra els primers 20 fitxers de configuració
ls /tmp                   # Mostra els fitxers temporals actuals

# Ves al teu projecte esportspulse-engine
cd ~/esportspulse-engine

# Mostra l'estructura amb un ls recursiu
# Si tens "tree" instal·lat (brew install tree a macOS; a Windows ja ve inclòs), és més visual:
ls -R
# o bé:
tree -L 2                 # -L 2 = només 2 nivells de profunditat
```

### 2. Localitzar les Teves Eines (10 min)

Ara descobreix on viuen els programes que fas servir cada dia:

```bash
# On viu Java? Segueix el camí complet fins l'executable real
which java                # Mostra el camí de l'executable
ls -la $(which java)      # Sovint és un symlink (enllaç simbòlic) — mira on apunta

# On viu Python?
which python3
python3 --version         # Verifica la versió

# On viu Maven?
which mvn
cat $(which mvn) | head -5  # mvn és un script bash — mira les primeres línies

# Mostra el teu $PATH formatat (un directori per línia per llegir-ho millor)
# tr reemplaça els ":" per salts de línia "\n"
echo $PATH | tr ':' '\n'

# Comprova la versió de Java i on viu la JVM completa
java -version 2>&1        # 2>&1 redirigeix l'error estàndard a la sortida (java escriu la versió per stderr)
echo $JAVA_HOME           # Variable que indica on està instal·lat el JDK complet
```

### 3. Observar un Procés Java en Viu (20 min)

Ara ve la part interessant: veuràs la JVM com un procés real que ocupa recursos.

**Primer, crea un programa Java que no acabi immediatament:**

Crea el fitxer `ProcessDemo.java` al teu directori de treball:

```java
// ProcessDemo.java
// Programa senzill que manté la JVM viva durant 60 segons
// per poder observar-la com a procés del sistema operatiu
public class ProcessDemo {
    public static void main(String[] args) throws InterruptedException {
        // Mostra el PID del procés — ProcessHandle.current() accedeix al procés actual
        System.out.println("JVM arrencada! PID: " + ProcessHandle.current().pid());

        // Mostra la memòria total assignada a la JVM
        // Runtime.getRuntime() dóna accés a les mètriques de la JVM
        long memoriaMB = Runtime.getRuntime().totalMemory() / (1024 * 1024);
        System.out.println("Memòria total de la JVM: " + memoriaMB + " MB");

        System.out.println("El procés estarà viu 60 segons. Obre un altre terminal i observa'l!");

        // Thread.sleep pausa l'execució — el procés segueix viu però no fa res
        // El SO manté el procés actiu assignant-li mínima CPU
        Thread.sleep(60000);  // 60.000 ms = 60 segons

        System.out.println("Procés finalitzat. El SO alliberarà els recursos.");
    }
}
```

**Ara compila i executa:**

```bash
# Compila el fitxer .java a bytecode .class
javac ProcessDemo.java

# Executa el programa — la JVM arrenca com un procés nou
java ProcessDemo
```

**Sense tancar aquest terminal**, obre un segon terminal i observa el procés:

```bash
# Busca el procés Java que acabes de llançar
# ps mostra tots els processos, grep filtra els que contenen "java"
ps -ef | grep java

# Observa el consum de CPU i RAM amb top, filtrat per processos java
# -l 1 = una sola mostra (no interactiu)
top -l 1 | grep java

# Alternativament, si tens htop, obre'l i busca "java" amb F4 (filtre)
htop
```

Apunta el PID, el consum de RAM i de CPU del teu procés.

### 4. Crear un Directori de Treball i Documentar (15 min)

Crea un fitxer amb les troballes del dia:

```bash
# Crea un directori per als exercicis de la setmana 3
mkdir -p ~/esportspulse-engine/docs/setmana03

# Crea el fitxer de notes amb les teves troballes
cat > ~/esportspulse-engine/docs/setmana03/dilluns-notes.md << 'EOF'
# Notes Dilluns S3 — El SO i el Terminal

## On viuen les meves eines
- Java: [posa el camí de `which java`]
- Python: [posa el camí de `which python3`]
- Maven: [posa el camí de `which mvn`]
- JAVA_HOME: [posa el valor de `echo $JAVA_HOME`]

## Procés Java observat
- PID: [posa el PID que has vist]
- RAM usada per la JVM: [posa el valor en MB]
- CPU aproximada: [posa el percentatge]

## Reflexió
- El SO gestiona els recursos perquè...
- $PATH serveix per...
- La JVM és un procés com qualsevol altre perquè...
EOF
```

Edita el fitxer amb el teu editor i completa les respostes.

```bash
# Fes commit del treball del dia
cd ~/esportspulse-engine
git add docs/setmana03/dilluns-notes.md
git commit -m "docs(s3): notes dilluns - SO, terminal i processos"
```

> **Lectura recomanada (opcional, no bloquejant):**
> - [The Linux Command Line](https://linuxcommand.org/tlcl.php) — Capítols 1-4 (gratuït)
> - `man bash` — La documentació oficial del shell (escriu-ho al terminal)

---

## Checklist de Lliurament

- [ ] Saps navegar pel sistema de fitxers amb `cd`, `ls`, `pwd`
- [ ] Has localitzat on viuen `java`, `python3` i `mvn` amb `which`
- [ ] Has executat `ProcessDemo.java` i has observat el procés amb `ps` o `top`
- [ ] Has anotat el PID, la RAM i la CPU del procés Java
- [ ] Has creat `docs/setmana03/dilluns-notes.md` amb les troballes
- [ ] Has fet commit amb les notes del dia
