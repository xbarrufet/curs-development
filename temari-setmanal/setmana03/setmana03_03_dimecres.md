# Setmana 3 — Dimecres: Pipes, Redirecció i Processament de Text

## Objectiu del Dia

Dominar les eines de processament de text de la terminal: pipes, redirecció i comandes com `grep`, `sort`, `find` i `jq`. Al final del dia has de poder encadenar comandes per analitzar codi, filtrar logs i processar JSON directament des de la terminal, sense obrir cap editor.

---

## Teoria

### La Filosofia UNIX: Eines Petites Connectades

La terminal de Linux/macOS segueix un principi de disseny molt potent: **cada comanda fa una sola cosa, i la fa bé**. La gràcia ve quan les connectes entre elles. Això s'anomena la filosofia UNIX.

Tres conceptes clau:
- **stdin** (entrada estàndard): d'on llegeix dades una comanda (per defecte, el teclat)
- **stdout** (sortida estàndard): on escriu el resultat (per defecte, la pantalla)
- **stderr** (sortida d'errors): on escriu els errors (per defecte, també la pantalla)

### Pipes (`|`): Connectar Comandes

El pipe (`|`) agafa la sortida (stdout) d'una comanda i la passa com a entrada (stdin) a la següent. Pensa-ho com una canonada que connecta eines.

```bash
# Llegeix un fitxer, filtra les línies amb "ERROR" i compta quantes n'hi ha
# cat: mostra el contingut → grep: filtra per patró → wc -l: compta línies
cat server.log | grep "ERROR" | wc -l

# Llista fitxers, ordena per mida (columna 5) i mostra els 5 més grans
# ls -l: llista detallada → sort: ordena numèricament per columna 5 → tail: últimes 5 línies
ls -l | sort -k5 -n | tail -5

# Mostra les 10 extensions de fitxer més comunes al projecte
# find: busca fitxers → sed: extreu l'extensió → sort + uniq -c: compta → sort -rn: ordena
find . -type f | sed 's/.*\.//' | sort | uniq -c | sort -rn | head -10
```

**Per què importa?** Amb 3-4 comandes encadenades pots fer anàlisis que amb un llenguatge de programació necessitarien 20 línies. Quan depures un bug en producció, això et salva minuts.

### Redirecció: Controlar On Van les Dades

La redirecció et permet enviar la sortida a fitxers en comptes de la pantalla, o llegir l'entrada des de fitxers.

```bash
# > sobreescriu el fitxer (COMPTE: esborra el contingut anterior!)
# Guarda la llista de fitxers Java en un fitxer de text
find . -name "*.java" > fitxers_java.txt

# >> afegeix al final del fitxer (no esborra res)
# Afegeix la data actual al final d'un log
echo "Compilació: $(date)" >> build.log

# 2> redirigeix NOMÉS els errors a un fitxer separat
# Compila i guarda els errors en un fitxer apart per revisar-los després
mvn compile 2> errors_compilacio.txt

# 2>&1 fusiona els errors amb la sortida normal
# Útil per guardar TOT el que passa (output + errors) en un sol fitxer
mvn test > resultats_tests.txt 2>&1

# < llegeix l'entrada des d'un fitxer (en comptes del teclat)
# Passa una llista de noms com a entrada a un script
python3 processa_noms.py < llista_noms.txt
```

**Cas real:** Imagina que tens un build que falla de nit. Si redirigeixes la sortida a un fitxer (`mvn test > build.log 2>&1`), l'endemà pots revisar exactament què ha passat.

### Comandes de Processament de Text

#### `grep` — Buscar Text

```bash
# -r: busca recursivament en tots els fitxers del directori
# -n: mostra el número de línia (imprescindible per trobar on és el codi)
# Busca totes les línies que contenen "TODO" al projecte Java
grep -rn "TODO" backend-java/src/

# -i: ignora majúscules/minúscules (útil quan no recordes l'estil)
grep -rni "exception" backend-java/src/

# -l: mostra NOMÉS els noms dels fitxers (no el contingut)
# Útil per saber QUINS fitxers tenen imports de Spring
grep -rl "import org.springframework" backend-java/src/

# -c: compta coincidències per fitxer
# Quants System.out.println té cada fitxer? (haurien de ser 0 en producció)
grep -rc "System.out.println" backend-java/src/main/
```

#### `sort`, `uniq`, `wc` — Ordenar, Deduplicar, Comptar

```bash
# sort: ordena línies alfabèticament (per defecte)
# sort -n: ordena numèricament (1, 2, 10 en comptes de 1, 10, 2)
# sort -r: ordre invers (descendent)

# uniq: elimina línies duplicades CONSECUTIVES
# Per això sempre va precedit de sort (per agrupar els duplicats)
# uniq -c: compta quantes vegades apareix cada línia

# Exemple: quines classes s'importen més al projecte?
# Extreu les línies d'import, ordena-les, compta duplicats, mostra el top 10
grep -rh "^import " backend-java/src/main/ | sort | uniq -c | sort -rn | head -10

# wc: compta línies (-l), paraules (-w) o caràcters (-c)
# Quantes línies de codi Java tens en total?
find . -name "*.java" | xargs wc -l
```

#### `cut` i `tr` — Retallar i Transformar

```bash
# cut: extreu columnes d'un text. -d',': delimitador coma, -f2: segon camp
cut -d',' -f2 jugadors.csv

# tr: tradueix o elimina caràcters
echo "HOLA MÓN" | tr 'A-Z' 'a-z'         # Converteix a minúscules
echo "col1    col2    col3" | tr -s ' '    # Comprimeix espais múltiples en un
```

### `find` — Buscar Fitxers

```bash
# Busca fitxers Java per nom i tipus (-type f = només fitxers, no directoris)
find . -name "*.java" -type f

# Busca fitxers Python modificats en les últimes 24 hores (-mtime -1)
find . -name "*.py" -mtime -1

# Detecta binaris accidentals al repo (fitxers de més d'1MB)
find . -type f -size +1M

# Combina find amb grep: busca "deprecated" dins de tots els .java
# -exec: executa grep per cada fitxer trobat
find . -name "*.java" -exec grep -l "deprecated" {} \;

# Alternativa amb xargs (més eficient: agrupa fitxers en una sola crida)
find . -name "*.java" | xargs grep -l "deprecated"
```

**Nota:** Busca sempre des de `.` o un directori concret, mai des de `/` (arrel del sistema).

### `jq` — Processar JSON des de la Terminal

Com a developer treballaràs amb APIs REST que retornen JSON constantment. `jq` et permet filtrar i transformar JSON sense obrir Python ni cap editor.

```bash
# Instal·la jq si no el tens (macOS)
brew install jq

# Descarrega dades d'una API pública i formata el JSON
# curl -s: petició HTTP en mode silenciós / jq '.': formata amb colors
curl -s "https://jsonplaceholder.typicode.com/users/1" | jq '.'

# Extreu un camp concret: .name accedeix al camp "name" de l'objecte
curl -s "https://jsonplaceholder.typicode.com/users/1" | jq '.name'

# .[] recorre l'array; {} crea objectes nous amb els camps que volem
curl -s "https://jsonplaceholder.typicode.com/users" | jq '.[] | {nom: .name, correu: .email}'

# select() filtra per condició
curl -s "https://jsonplaceholder.typicode.com/users" | jq '.[] | select(.name | contains("Cle"))'

# length compta els elements de l'array
curl -s "https://jsonplaceholder.typicode.com/users" | jq 'length'
```

---

## Activitat

Tots els exercicis es fan dins el directori del teu projecte `esportspulse-engine`. Obre la terminal i situa't a l'arrel del projecte.

### 1. Trobar els 5 Fitxers Java Més Llargs (10 min)

```bash
# Encadena 4 comandes per trobar els fitxers amb més línies de codi:
# 1. find: localitza tots els fitxers .java
# 2. xargs wc -l: compta les línies de cada fitxer
# 3. sort -n: ordena per número de línies (ascendent)
# 4. tail -6: agafa els 5 últims + la línia de total
find . -name "*.java" -type f | xargs wc -l | sort -n | tail -6

# Guarda el resultat en un fitxer per referència futura
find . -name "*.java" -type f | xargs wc -l | sort -n | tail -6 > analisi_mida_fitxers.txt
```

**Pregunta per reflexionar:** Hi ha algun fitxer amb més de 200 línies? Si és així, potser caldria dividir-lo en classes més petites (principi de responsabilitat única).

### 2. Buscar Tots els TODO del Codi (10 min)

```bash
# Busca TODO i FIXME al codi Java i Python (amb número de línia)
grep -rn "TODO\|FIXME" backend-java/src/
grep -rn "TODO\|FIXME" ai-python/src/

# Genera un informe i compta fitxers afectats
grep -rn "TODO\|FIXME" backend-java/src/ > todos_pendents.txt
echo "Fitxers amb TODOs: $(grep -rc 'TODO\|FIXME' backend-java/src/ | grep -v ':0$' | wc -l)"
```

### 3. Processar JSON d'una API Pública (15 min)

```bash
# Descarrega la llista d'usuaris (-s: silenciós, -o: guarda a fitxer)
curl -s "https://jsonplaceholder.typicode.com/users" -o usuaris.json

# Formata el JSON per inspeccionar-lo
jq '.' usuaris.json

# Extreu noms i ciutats (navega objectes niuats amb .address.city)
jq '.[] | {nom: .name, ciutat: .address.city}' usuaris.json

# Filtra per condició: webs que acaben en .org
jq '.[] | select(.website | endswith(".org")) | .name' usuaris.json

# Compta posts per usuari (group_by agrupa, length compta)
curl -s "https://jsonplaceholder.typicode.com/posts" | jq 'group_by(.userId) | .[] | {userId: .[0].userId, total_posts: length}'
```

### 4. Redirecció: Separar Output i Errors (10 min)

```bash
# Compila i separa stdout dels errors (stderr)
mvn compile > build_output.txt 2> build_errors.txt

# Guarda-ho tot junt en un sol fitxer (útil per enviar a un company)
mvn test > build_complet.txt 2>&1

# Afegeix un timestamp al final del log (>> no sobreescriu)
echo "--- Executat: $(date) ---" >> build_complet.txt
```

### 5. Pipeline Combinat: Anàlisi del Projecte (15 min)

Combina tot el que has après en un mini-script d'anàlisi:

```bash
# Crea un informe complet del projecte redirigint tot a un fitxer
echo "=== INFORME DEL PROJECTE ===" > informe_projecte.txt
echo "Data: $(date)" >> informe_projecte.txt

# Nombre total de fitxers per llenguatge
echo "--- Fitxers per tipus ---" >> informe_projecte.txt
echo "Java: $(find . -type f -name '*.java' | wc -l) fitxers" >> informe_projecte.txt
echo "Python: $(find . -type f -name '*.py' | wc -l) fitxers" >> informe_projecte.txt

# Línies totals de codi Java
echo "--- Línies de codi ---" >> informe_projecte.txt
find . -name "*.java" -type f | xargs wc -l | tail -1 >> informe_projecte.txt

# Els 3 imports Java més usats
echo "--- Top 3 imports Java ---" >> informe_projecte.txt
grep -rh "^import " backend-java/src/main/ 2>/dev/null | sort | uniq -c | sort -rn | head -3 >> informe_projecte.txt

# Mostra l'informe final
cat informe_projecte.txt
```

---

## Checklist de Lliurament

- [ ] Saps encadenar 3+ comandes amb pipes per analitzar fitxers de codi
- [ ] Saps redirigir stdout i stderr a fitxers separats
- [ ] Has usat `grep -rn` per buscar patrons al codi del projecte
- [ ] Has usat `find` combinat amb `xargs` o `-exec`
- [ ] Has descarregat JSON d'una API amb `curl` i l'has processat amb `jq`
- [ ] Has generat un fitxer `informe_projecte.txt` amb l'anàlisi del teu projecte
- [ ] Has fet commit: `feat(cli): add project analysis scripts with pipes and jq`
