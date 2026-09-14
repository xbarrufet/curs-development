# Setmana 1 — Dilluns: Entorn, Git i Estructura del Projecte

## Objectiu del Dia

Tenir el projecte `esportspulse-engine` creat, compilable i pujat a GitHub amb una estructura políglota (Java + Python). Al final del dia has de poder fer `mvn compile` sense errors i tenir el primer commit a la branca `main`.

---

## Teoria

### Git: El Mínim per Començar

Git és el sistema de control de versions que usa tota la indústria. Et permet:
- Guardar versions del codi (commits)
- Treballar en paral·lel amb branques
- Col·laborar amb altres devs sense trepitjar-se

**Comandes que faràs servir avui:**

```bash
git init                          # Crea un repositori nou
git add .                         # Prepara fitxers per al commit
git commit -m "missatge"          # Guarda una versió
git branch feature/week1-bench    # Crea una branca
git checkout feature/week1-bench  # Canvia a la branca
git push -u origin main           # Puja a GitHub
```

**Conventional Commits** — Els missatges de commit segueixen un format estàndard:
```
feat(java): initial project structure with Java 21 + Python
fix(search): correct null handling in linear search
docs: add README with setup instructions
```

Format: `type(scope): description`. Tipus habituals: `feat`, `fix`, `docs`, `test`, `refactor`.

> **Lectura recomanada (opcional, no bloquejant):**
> - [Pro Git](https://git-scm.com/book/en/v2) — Capítols 1-2
> - [Curso de Git y GitHub desde cero](https://www.youtube.com/watch?v=niPExbK8lSw) — Midudev (YouTube)

### Estructura d'un Projecte Java amb Maven

Maven és l'eina que compila, testeja i gestiona dependències en Java. El fitxer `pom.xml` és el seu fitxer de configuració (l'equivalent del `package.json` en Node.js o `requirements.txt` en Python).

```
esportspulse-engine/
├── pom.xml                          ← Configuració Maven (dependències, versió Java)
├── .gitignore
├── backend-java/
│   └── src/
│       ├── main/java/               ← Codi font Java
│       │   └── com/esportspulse/
│       │       └── engine/
│       └── test/java/               ← Tests JUnit
│           └── com/esportspulse/
│               └── engine/
└── ai-python/
    ├── requirements.txt             ← Dependències Python
    └── src/
```

### Cursor IDE + Claude: Primer Contacte

Cursor és un IDE basat en VS Code amb IA integrada. El faràs servir com a eina de productivitat, no com a substitut del teu criteri. Avui l'objectiu és mínim: instal·lar-lo, connectar Claude, i generar un fitxer de configuració per veure com funciona el cicle "demanar → revisar → ajustar".

---

## Activitat

### 1. Instal·lació de l'entorn (15 min)

Verifica que tens les eines instal·lades:

```bash
java --version    # Ha de ser 21+
mvn --version     # Ha de ser 3.9+
python3 --version # Ha de ser 3.11+
git --version     # Ha de ser 2.40+
```

Si falta alguna eina, instal·la-la:
```bash
# macOS amb Homebrew
brew install openjdk@21 maven python@3.12 git
```

Instal·la [Cursor IDE](https://cursor.sh). A Settings → Features, activa "Tab autocomplete" i "Agent mode". Configura Claude com a model per defecte.

### 2. Crear el projecte Maven (30 min)

Crea el projecte des de zero. Obre la terminal i executa:

```bash
mkdir esportspulse-engine && cd esportspulse-engine
```

Crea el fitxer `pom.xml` a l'arrel amb aquest contingut mínim:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.esportspulse</groupId>
    <artifactId>esportspulse-engine</artifactId>
    <version>0.1.0-SNAPSHOT</version>

    <properties>
        <java.version>21</java.version>
        <maven.compiler.source>21</maven.compiler.source>
        <maven.compiler.target>21</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>5.10.2</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

Crea l'estructura de directoris:

```bash
# Java
mkdir -p backend-java/src/main/java/com/esportspulse/engine
mkdir -p backend-java/src/test/java/com/esportspulse/engine

# Python
mkdir -p ai-python/src
touch ai-python/requirements.txt
```

Verifica que compila:

```bash
mvn compile
```

Si surt `BUILD SUCCESS`, l'estructura és correcta.

### 3. Configurar `.gitignore` (10 min)

Crea el fitxer `.gitignore` a l'arrel del projecte:

```gitignore
# Java
target/
*.class
*.jar

# IDE
.idea/
*.iml
.vscode/
.cursor/

# Python
__pycache__/
*.pyc
.venv/

# OS
.DS_Store

# Secrets
.env
```

### 4. Inicialitzar Git i pujar a GitHub (15 min)

```bash
git init
git add .
git commit -m "feat: initial project structure (Java 21 + Python polyglot)"
```

Crea un repositori a GitHub (`esportspulse-engine`, privat o públic). Connecta'l:

```bash
git remote add origin https://github.com/EL_TEU_USER/esportspulse-engine.git
git push -u origin main
```

Crea la branca de treball de la setmana:

```bash
git checkout -b feature/week1-benchmarking
```

### 5. Primer contacte amb Cursor + Claude (20 min)

Obre el projecte amb Cursor. A la finestra de chat, demana:

> "Escriu un `.cursorrules` per a un projecte Java 21 + Python amb Maven. Inclou regles per a noms de variable (camelCase Java, snake_case Python), format de commit (Conventional Commits), i estructura de carpetes."

- Deixa que generi el fitxer.
- **Revisa el contingut.** Ajusta el que no quadri amb l'estructura que acabes de crear.
- Fes commit: `feat: add .cursorrules for Java + Python conventions`

> **Lliçó clau:** L'IA genera boilerplate ràpidament, però l'arquitectura la dibuixes tu. Si el `.cursorrules` generat proposa una estructura diferent de la que has creat, el teu criteri mana.

---

## Checklist de Lliurament

- [ ] `mvn compile` passa sense errors
- [ ] Repositori a GitHub amb 2 commits mínims (estructura + .cursorrules)
- [ ] Branca `feature/week1-benchmarking` creada
- [ ] `.gitignore` exclou `target/`, `.idea/`, `.venv/`, `.env`
- [ ] Cursor instal·lat i funcionant amb Claude
