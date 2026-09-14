**Setmana 4: Clean Code, Code Review, Git Workflow i CI Bàsic**

---

### **Dilluns: Llegir i Entendre Codi d'Altri**

* **Cursos i Material de Lectura:**
* **Llibre:** [*Clean Code*](https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882) (Robert C. Martin) — Capítols 1-3 (Clean Code, Meaningful Names, Functions).
* **Article:** [*How to Read Other People's Code*](https://blog.codinghorror.com/learn-to-read-the-source-luke/) — Coding Horror.
* **Vídeo:** [*Code Review Best Practices*](https://www.youtube.com/results?search_query=code+review+best+practices+developers) — qualsevol vídeo pràctic.


* **Activitat i Què s'espera programar:**
* **Teoria: El 80% del temps d'un developer és llegir codi, no escriure'l.**
  * A la feina, el primer que faràs és obrir un repositori de 50.000 línies que no has escrit tu. Has de poder navegar-lo, entendre la intenció, i trobar on fer un canvi sense trencar res.
  * Skills de lectura: seguir el flux d'una request (Controller → Service → Repository → BD), identificar responsabilitats, detectar code smells.
* **Exercici pràctic: "Investiga el bug".**
  * Es proporciona un projecte Spring Boot petit (preparat pel formador) amb un bug real: un endpoint `GET /games?title=X` retorna resultats duplicats en certes condicions.
  * L'estudiant ha de:
    1. Llegir el codi (sense executar-lo encara) i intentar identificar el bug.
    2. Executar-lo i reproduir el bug amb curl/Postman.
    3. Escriure un test que falla per demostrar el bug.
    4. Corregir el bug.
    5. Verificar que el test ara passa.
  * **Lliçó:** Això és el dia a dia. Et donen un ticket Jira que diu "resultats duplicats al cercar jocs" i tu has de trobar-ho en codi que no has escrit.
* **Exercici de Prompt Engineering:** Dona el codi bugat a Cursor i demana: *"Troba el bug en aquest codi."* Compara la resposta de l'IA amb la teva anàlisi. L'IA l'ha trobat? Ha donat la causa correcta o només un símptoma?


---

### **Dimarts: Refactoritzar Codi Generat per IA — Anti-Patrons Pràctics**

* **Cursos i Material de Lectura:**
* **Article:** [*OWASP Top 10*](https://owasp.org/www-project-top-ten/) — Llegir els 3 primers (Injection, Broken Auth, Sensitive Data Exposure).
* **Article:** Baeldung — [*Common Java Mistakes*](https://www.baeldung.com/java-common-mistakes).
* **Article:** Baeldung — [*SQL Injection Prevention*](https://www.baeldung.com/sql-injection).


* **Activitat i Què s'espera programar:**
* **Exercici: Auditoria de 5 snippets generats per IA.**
  * Es proporcionen 5 blocs de codi Java "generats per un assistent IA" (preparats pel formador). Cada un conté almenys un problema greu. L'estudiant ha d'identificar-lo i corregir-lo:

  * **Snippet 1 — SQL Injection:**
    ```java
    @Query("SELECT g FROM Game g WHERE g.title = '" + title + "'")
    ```
    Problema: concatenació de strings en query → SQL injection. Solució: paràmetres vinculats (`:title`).

  * **Snippet 2 — Secret hardcodejat:**
    ```java
    private static final String API_KEY = "sk-abc123def456";
    ```
    Problema: secret en codi font → acaba a Git. Solució: variable d'entorn o `application.properties` exclòs de Git.

  * **Snippet 3 — NullPointerException amagat:**
    ```java
    GameRecord game = repository.findById(appId);
    return game.title();  // Si no existeix → NPE en producció
    ```
    Problema: `findById` retorna `null` si no existeix. Solució: `Optional` + `orElseThrow()` amb missatge clar.

  * **Snippet 4 — Test que no testeja res:**
    ```java
    @Test void testGetGame() {
        when(mockService.findById("APP-1")).thenReturn(testGame);
        GameRecord result = mockService.findById("APP-1");
        assertNotNull(result);  // Verifica que el mock retorna el que li hem dit que retorni!
    }
    ```
    Problema: el test verifica el mock, no el codi real. Solució: testejar el controller o service, no el mock directament.

  * **Snippet 5 — Excepció silenciada:**
    ```java
    try {
        apiClient.fetchData(appId);
    } catch (Exception e) {
        // TODO: handle later
    }
    ```
    Problema: l'error desapareix silenciosament → bugs impossibles de diagnosticar en producció. Solució: com a mínim `log.error()`, o re-throw amb context.

* **Exercici de Prompt Engineering:** Per cada snippet corregit, demana a Cursor: *"Revisa aquest codi per problemes de seguretat i robustesa."* La IA ha trobat el que tu has trobat? Ha trobat coses que tu no has vist?
* **Lliçó:** La IA genera codi que **compila i funciona en el happy path**. El teu valor com a developer és detectar el que falla en producció: seguretat, nulls, errors silenciats, tests buits.


---

### **Dimecres: Git Workflow Professional — Branching, Rebase i Conflictes**

* **Cursos i Material de Lectura:**
* **Llibre:** [*Pro Git*](https://git-scm.com/book/en/v2) — Capítol 3 (Branching) i Capítol 6.4 (Rewriting History).
* **Article:** Atlassian — [*Merging vs Rebasing*](https://www.atlassian.com/git/tutorials/merging-vs-rebasing).
* **Article:** Atlassian — [*Resolving Merge Conflicts*](https://www.atlassian.com/git/tutorials/using-branches/merge-conflicts).


* **Activitat i Què s'espera programar:**
* **Teoria: Git Flow simplificat per equips.**
  * `main` → codi estable, sempre desplegable.
  * `feature/xxx` → branca per cada tasca/ticket.
  * Workflow: crea branca → treballa → rebase sobre main → PR → code review → merge.
  * Per què rebase i no merge? Historial net, més fàcil de llegir `git log`.
* **Exercici pràctic: Resolució de conflictes.**
  * Simula un conflicte real:
    1. Crea branca `feature/price-format` que canvia com es mostra el preu a `GameDTO`.
    2. A `main`, un "company" (el formador o el propi estudiant en una altra branca) ha canviat el mateix fitxer per afegir un camp `currency`.
    3. Intenta `git rebase main` → conflicte.
    4. Resol el conflicte manualment (no amb "accept theirs/ours" cegament).
    5. Verifica que els tests passen després del rebase.
  * Repeteix amb un conflicte al `pom.xml` (dependències) — això passa sovint en equip.
* **Exercici: `git bisect` per trobar un bug.**
  * Prepara un historial de 10 commits on un d'ells introdueix un bug (un test que falla).
  * Usa `git bisect` per trobar automàticament el commit culpable.
  * **Lliçó:** `git bisect` és una eina que pocs juniors coneixen i que impressiona en entrevistes.
* **Comandes que has de dominar:**
  * `git rebase main` / `git rebase -i HEAD~3` (squash commits)
  * `git stash` / `git stash pop` (guardar treball temporal)
  * `git log --oneline --graph` (visualitzar historial)
  * `git cherry-pick <commit>` (portar un commit específic)
  * `git reflog` (recuperar treball "perdut")


---

### **Dijous: Code Review i GitHub Actions Bàsic**

* **Cursos i Material de Lectura:**
* **Article:** Google — [*How to Do a Code Review*](https://google.github.io/eng-practices/review/reviewer/).
* **Article:** GitHub — [*About Pull Request Reviews*](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/reviewing-changes-in-pull-requests/about-pull-request-reviews).
* **Documentació:** GitHub — [*GitHub Actions Quickstart*](https://docs.github.com/en/actions/quickstart).


* **Activitat i Què s'espera programar:**
* **Teoria: Code Review com a skill professional.**
  * A una empresa, no es merja res sense review. El reviewer busca:
    * Correcció: fa el que hauria de fer?
    * Seguretat: hi ha vulnerabilitats?
    * Mantenibilitat: un altre developer ho entendrà dins 6 mesos?
    * Tests: els canvis estan testejats?
  * Feedback constructiu: "Això podria fallar si X perquè Y. Suggeriria Z." — mai "Això està malament."
* **Exercici pràctic: Fer una code review real.**
  * Es proporciona una PR preparada (pel formador) amb 8-10 fitxers canviats. Conté:
    * 2 problemes de seguretat (els de dimarts en context real)
    * 1 test insuficient
    * 1 canvi de rendiment dubtós (bucle innecessari)
    * 3-4 fitxers correctes
  * L'estudiant ha d'escriure comentaris de review a cada fitxer, com si fos un reviewer real a GitHub.
  * **Lliçó:** No tot és un problema. Saber dir "LGTM" als fitxers correctes és tan important com trobar bugs.
* **Exercici d'Escriptura de Specs: Spec de Refactorització.**
  * Agafa un dels mòduls refactoritzats dimarts (ex: el snippet amb SQL injection corregit dins del seu context complet).
  * Escriu una spec en markdown que descrigui la refactorització desitjada:
    ```markdown
    ## Spec: Refactoritzar GameSearchService
    
    ### Objectiu
    Eliminar SQL injection i millorar error handling.
    
    ### Regles
    - Totes les queries SQL han d'usar paràmetres vinculats (:param), mai concatenació.
    - Tot accés a repository que retorna Optional ha d'usar orElseThrow() amb missatge descriptiu.
    - Les excepcions mai es poden silenciar (catch buit). Mínim: log.error() + re-throw.
    
    ### Tests esperats
    - Test que verifica que una query amb caràcters especials (', ", ;) no causa error.
    - Test que verifica que buscar un ID inexistent llança EntityNotFoundException.
    - Test que verifica que un error d'API externa es propaga amb missatge clar.
    ```
  * Dona la spec a Cursor (en una conversa nova, sense context previ) i demana: *"Refactoritza GameSearchService seguint aquesta spec."*
  * Compara el resultat amb la refactorització manual de dimarts.
  * Si el resultat és dolent, **itera la spec** (no el prompt): afegeix exemples, restriccions, o context que faltava.
  * **Lliçó:** Una bona spec produeix bon codi al primer intent. Una spec vaga requereix 5 iteracions de "no, això no és el que volia". El teu valor és saber escriure la spec, no picar el codi.
* **GitHub Actions — El mínim funcional:**
  * Crear `.github/workflows/ci.yml`:
    ```yaml
    name: CI
    on: [push, pull_request]
    jobs:
      build-and-test:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v4
          - uses: actions/setup-java@v4
            with:
              java-version: '21'
              distribution: 'temurin'
          - run: mvn test
          - run: mvn checkstyle:check
    ```
  * Entendre cada línia: trigger, runner, steps.
  * **No anar més enllà.** S6 afegirà coverage, S10 afegirà Python, S22 consolidarà tot.
  * Configurar Checkstyle mínim al `pom.xml`: format d'importacions, llargada de línia, noms de variables.


---

### **Divendres: Consolidació, Tag v0.1 i Pull Request**

* **Cursos i Material de Lectura:**
* **Especificació:** [*Semantic Versioning*](https://semver.org/).
* **Especificació:** [*Conventional Commits*](https://www.conventionalcommits.org/).


* **Activitat i Què s'espera programar:**
* **Revisió global del repositori:**
  * Verificar que l'estructura de carpetes és neta i consistent.
  * Verificar que `.gitignore` exclou tot el que ha d'excloure (classes compilades, IDE config, `.env`).
  * Verificar que no hi ha secrets al repositori (`git log --all -p | grep -i "api_key\|password\|secret"` — eina bàsica).
  * Verificar que tots els tests passen: `mvn test`.
  * Verificar que Checkstyle passa: `mvn checkstyle:check`.
* **README professional:**
  * Secció "Què és GamePulse" (2 frases).
  * Secció "Com executar" amb comandes exactes.
  * Badge de CI: `![CI](https://github.com/USER/gamepulse-engine/actions/workflows/ci.yml/badge.svg)`.
* **Tag de versió:**
  * `git tag -a v0.1 -m "Bloc 1 complete: domain model, repository patterns, concurrency, CI"`.
  * `git push origin v0.1`.
* **Reflexió de Bloc 1:**
  * Escriu en 5 línies: *"Què he après en les primeres 4 setmanes que podria explicar en una entrevista?"* Exemples:
    * "Sé per què HashMap és O(1) i quan importa per rendiment."
    * "Sé implementar el patró Repository per desacoblar la persistència."
    * "Sé detectar race conditions i entenc @Transactional."
    * "Sé fer code review identificant problemes de seguretat i tests buits."
    * "He configurat CI amb GitHub Actions des de zero."

* **Finalització del cicle Git:**
* Commit final: `chore: tag v0.1 with CI badge, README, and repository cleanup`
* Puja branca `feature/week4-clean-code-ci`, crea PR.
* Practica: fes la teva pròpia code review de la PR abans de merge.
* Merge a `main`.

---

## Vídeos Recomanats

- **Clean Code:** Cerca "CodelyTV Clean Code" (castellà, equip de referència en bones pràctiques). En anglès: "Uncle Bob Clean Code" (la conferència original).
- **Git workflow:** Cerca "MoureDev Git y GitHub" (castellà, molt complet) o "Midudev Git tutorial". En anglès: "Fireship Git explained in 100 seconds" (molt visual).
- **Code review:** Cerca "Google Engineering code review best practices" o "CodelyTV code review".
- **CI/CD bàsic:** Cerca "GitHub Actions tutorial español" o "Fireship GitHub Actions" (anglès, curt i directe).

---

## Nota sobre Empliabilitat

Setmana 4 tanca el Bloc 1 amb les skills que un junior utilitza des del **dia 1 a qualsevol empresa**:

| Skill | On es practica | Per què importa |
|-------|---------------|-----------------|
| Llegir codi d'altri | Dilluns | El 80% del temps d'un dev és llegir, no escriure |
| Detectar problemes en codi IA | Dimarts | La IA és el teu copilot, però tu ets el responsable |
| Git workflow amb rebase | Dimecres | Cap empresa fa merge sense PR + historial net |
| Code review | Dijous | El filtre de qualitat de qualsevol equip |
| CI bàsic | Dijous | Saber per què el build falla és survival skill |

Cap d'aquestes skills requereix algorítmica avançada. Totes requereixen **criteri professional** — que és el que diferencia un junior contractable d'un que només sap fer tutorials.
