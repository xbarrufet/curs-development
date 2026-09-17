# Setmana 7 — Exercicis de Consolidació

Aquests exercicis repassen els conceptes clau de la setmana. No cal lliurar-los — són per verificar que has entès la teoria i la pràctica abans de passar a la setmana 8. Intenta resoldre'ls sense mirar els apunts; si et quedes encallat, revisa el dia corresponent.

---

## Bloc 1: Anti-patrons de Codi IA (Dilluns)

**Exercici 1.1 — Identifica l'Anti-patró**

Per a cada tros de codi, indica quin dels 5 anti-patrons conté (SQL Injection, Secret Hardcoded, NullPointerException Amagat, Test que No Testa Res, Excepció Silenciada) i escriu la correcció:

```java
// A
var query = "SELECT * FROM champions WHERE role = '" + role + "'";
var result = jdbcTemplate.query(query, Champion.class);
```

```java
// B
public Champion getChampion(Long id) {
    return repository.findById(id).get();
}
```

```python
# C
def import_data(path):
    try:
        return reader.read(path)
    except Exception:
        pass
        return []
```

**Exercici 1.2 — Detecta el Test Fals**

Aquest test passa sempre. Explica per què no verifica res útil i reescriu-lo perquè sí ho faci:

```java
@Test
void testGetChampion() {
    when(repository.findById(1L)).thenReturn(Optional.of(jinx));
    var result = service.getChampion(1L);
    assertEquals(jinx, result);
}
```

---

## Bloc 2: Git Rebase i Workflow (Dimarts)

**Exercici 2.1 — Rebase vs Merge**

1. Per què `rebase` deixa un historial més net que `merge`?
2. Quina regla no s'ha de trencar mai amb `rebase`? Per què?
3. Quan faries servir `merge` en comptes de `rebase`?

**Exercici 2.2 — Resol el Conflicte**

Se't dona aquest conflicte. Escriu la resolució correcta (no "accepta la teva" ni "accepta l'entrant" cegament — combina el que té sentit):

```java
<<<<<<< HEAD
public List<Champion> search(String name) {
    return repo.findByName(name);
}
=======
public List<Champion> search(String query) {
    return repo.findByNameContainingIgnoreCase(query);
}
>>>>>>> main
```

**Exercici 2.3 — Git Bisect**

Tens un repositori amb 200 commits i un bug introduït en algun d'ells. Aproximadament quants passos necessitarà `git bisect` per trobar-lo? Justifica el càlcul.

---

## Bloc 3: GitHub Actions i CI (Dimecres)

**Exercici 3.1 — Interpreta el Workflow**

```yaml
name: Nightly Audit
on:
  schedule:
    - cron: '30 1 * * *'
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: mvn dependency-check:check
```

1. Quan s'executa aquest workflow?
2. Es pot disparar manualment amb la configuració actual? Com ho canviaries perquè sí?
3. Diferència entre `uses` i `run` en aquest exemple.

**Exercici 3.2 — Escriu un Step**

Afegeix al workflow anterior un step nou que executi els tests de Python (`pytest`) al directori `python/`, després del checkout.

---

## Bloc 4: Code Review i Specs (Dijous)

**Exercici 4.1 — Millora el Comentari**

Reescriu aquest comentari de review seguint la fórmula Observació + Impacte + Suggeriment:

> "Aquest catch està malament."

(codi al qual es refereix: `catch (Exception e) { return null; }`)

**Exercici 4.2 — Detecta la Spec Vaga**

Aquesta spec té un problema greu. Quin és, i com la reescriuries?

> "Millora el ChampionService perquè sigui més ràpid i net."

**Exercici 4.3 — Pre-commit Hooks**

Vertader o fals (justifica la resposta):
1. Un pre-commit hook s'executa al servidor de CI, no a l'ordinador del developer.
2. Si el pre-commit hook falla, el commit no s'arriba a crear.
3. Checkstyle i ruff es poden configurar com a pre-commit hooks.

---

## Bloc 5: SemVer, Commits i Release (Divendres)

**Exercici 5.1 — Classifica el Canvi**

Per a cada canvi, indica si incrementaria MAJOR, MINOR o PATCH:

1. Afegir un nou endpoint `GET /api/champions/{id}/stats`.
2. Corregir que `winRate` es calculava amb un decimal incorrecte.
3. Canviar el format de resposta de tots els endpoints REST (trenca clients existents).

**Exercici 5.2 — Escriu el Commit i el Tag**

Acabes d'arreglar un bug on `findById` llançava `NullPointerException` en comptes d'una excepció descriptiva. Escriu:
1. El missatge de commit (Conventional Commits).
2. Si aquest fix, per si sol, justificaria pujar de v0.1.0 a v0.2.0 o a v0.1.1. Per què?

**Exercici 5.3 — Checklist Pre-release**

Abans de crear el tag, quines 3 comprovacions farà sí o sí (a part de "els tests passen")?

---

## Exercici Final: Integració

Se't dona aquest Pull Request per revisar abans de crear el tag `v0.2.0`:

```java
@RestController
@RequestMapping("/api/champions")
public class ChampionController {
    @Autowired
    private ChampionManagementService service;

    @GetMapping("/{name}")
    public Champion getByName(@PathVariable String name) {
        return service.findByName(name).get();
    }
}
```

```yaml
# .github/workflows/ci.yml — afegit en aquest PR
on: push
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: mvn test
```

```
Commit d'aquest PR: "canvis"
```

1. Troba almenys 3 problemes (codi, CI, o procés de commit).
2. Escriu un comentari de review per a cadascun (Observació + Impacte + Suggeriment).
3. Aquest PR es pot fusionar tal com està? Per què?
4. Un cop corregit, quin número de versió li tocaria (v0.2.0 és correcte, o hauria de ser un altre)?

---
---

# Solucions

> **Atenció:** Intenta resoldre els exercicis abans de mirar les solucions. Aprens molt més si primer t'equivoques i després compares.

---

## Bloc 1: Anti-patrons de Codi IA

**Solució 1.1 — Identifica l'Anti-patró**

- **A: SQL Injection.** El `role` es concatena directament a la query. Correcció: `"SELECT * FROM champions WHERE role = ?"` amb `jdbcTemplate.query(query, new Object[]{role}, Champion.class)`.
- **B: NullPointerException Amagat.** `.get()` sobre un `Optional` sense comprovar. Correcció: `repository.findById(id).orElseThrow(() -> new ChampionNotFoundException(id))`.
- **C: Excepció Silenciada.** `except Exception: pass` s'empassa l'error i retorna `[]` com si tot anés bé. Correcció: capturar l'excepció específica, loguejar-la amb `logger.error(..., exc_info=True)`, i rellançar-la com a excepció de domini.

**Solució 1.2 — Detecta el Test Fals**

El test només comprova que el mock retorna el que li hem dit que retorni — no verifica cap lògica del servei. Reescriptura:

```java
@Test
void getChampion_whenExists_returnsChampion() {
    when(repository.findById(1L)).thenReturn(Optional.of(jinx));
    var result = service.getChampion(1L);
    assertEquals(jinx, result);
    verify(repository).findById(1L);
}

@Test
void getChampion_whenNotFound_throwsException() {
    when(repository.findById(999L)).thenReturn(Optional.empty());
    assertThrows(ChampionNotFoundException.class, () -> service.getChampion(999L));
}
```

El segon test (el cas "no trobat") és el que realment aporta valor.

---

## Bloc 2: Git Rebase i Workflow

**Solució 2.1 — Rebase vs Merge**

1. `rebase` mou els teus commits al final de la branca objectiu en lloc de crear un "merge commit" extra — l'historial queda com una línia recta en comptes d'un graf amb bifurcacions.
2. Mai fer `rebase` d'una branca compartida (main, develop). Rebase reescriu els commits (nous hashos); si algú altre ja té la versió antiga, els historials divergeixen i es trenca per a tothom.
3. `merge` (via PR) quan integres una feature a `main` — vols preservar que aquell conjunt de commits va entrar junt, i `main` és compartida.

**Solució 2.2 — Resol el Conflicte**

```java
public List<Champion> search(String query) {
    return repo.findByNameContainingIgnoreCase(query);
}
```

Es manté el nom de paràmetre `query` (més genèric, ja consolidat a `main`) i el comportament `IgnoreCase` (una millora respecte a la versió local).

**Solució 2.3 — Git Bisect**

Aproximadament **log2(200) ≈ 8 passos**. Bisect divideix per la meitat l'espai de cerca a cada iteració (200 → 100 → 50 → 25 → 13 → 7 → 4 → 2 → 1).

---

## Bloc 3: GitHub Actions i CI

**Solució 3.1 — Interpreta el Workflow**

1. Cada dia a la 1:30 del matí (`cron: '30 1 * * *'`).
2. No, tal com està no té `workflow_dispatch`. Caldria afegir-lo a `on:` per tenir el botó "Run workflow" a GitHub.
3. `uses` executa una Action pre-feta del Marketplace (`actions/checkout@v4` descarrega el codi); `run` executa una comanda de terminal directament (`mvn dependency-check:check`).

**Solució 3.2 — Escriu un Step**

```yaml
      - uses: actions/checkout@v4
      - name: Executar tests Python
        run: pytest --verbose
        working-directory: ./python
      - run: mvn dependency-check:check
```

---

## Bloc 4: Code Review i Specs

**Solució 4.1 — Millora el Comentari**

> "Observació: aquest `catch` captura `Exception` genèric i retorna `null` sense loguejar res.
> Impacte: si falla, per exemple, la connexió a la BD, l'error desapareix i qui crida aquest mètode rep `null` sense saber que hi ha hagut un problema — impossible de depurar en producció.
> Suggeriment: captura l'excepció específica, loguega-la amb context (`log.error(...)`), i llança una excepció de domini en comptes de retornar `null`."

**Solució 4.2 — Detecta la Spec Vaga**

El problema és que "més ràpid" i "més net" no són verificables — no diuen com mesurar-ho ni què ha de complir el resultat. Reescriptura:

> "Objectiu: reduir el temps de resposta de `findChampionsByRole` per sota de 50ms amb 100.000 registres.
> Regles: afegir un índex a la columna `role`; no canviar la signatura pública del mètode.
> Criteris d'acceptació: benchmark abans/després documentat; tots els tests existents continuen passant."

**Solució 4.3 — Pre-commit Hooks**

1. **Fals.** S'executa a l'ordinador del developer, abans que el commit es creï — no al CI.
2. **Vertader.** Si el hook falla (exit code != 0), Git rebutja el commit.
3. **Vertader.** Ambdós es poden configurar com a hooks dins `.pre-commit-config.yaml` (ruff) o com a script personalitzat (checkstyle).

---

## Bloc 5: SemVer, Commits i Release

**Solució 5.1 — Classifica el Canvi**

1. **MINOR** — nova funcionalitat, backward compatible.
2. **PATCH** — correcció de bug, backward compatible.
3. **MAJOR** — trenca compatibilitat amb clients existents.

**Solució 5.2 — Escriu el Commit i el Tag**

1. `fix(service): throw ChampionNotFoundException instead of NullPointerException in findById`
2. **v0.1.1** (PATCH) — és una correcció de bug, no afegeix funcionalitat ni trenca res.

**Solució 5.3 — Checklist Pre-release**

Tres d'aquestes (n'hi ha més a la teoria, però n'hi ha prou amb 3 ben justificades):
- Cap secret al codi (`grep -r "API_KEY\|SECRET\|PASSWORD"`).
- Checkstyle/ruff passen sense errors.
- CI pipeline en verd a GitHub Actions.

---

## Exercici Final: Integració

**Solució**

Problemes trobats (n'hi ha almenys 4):

1. **NullPointerException Amagat** — `service.findByName(name).get()` sense comprovar si l'`Optional` és buit.
   > "Observació: es crida `.get()` directament sobre l'`Optional` que retorna `findByName`. Impacte: si el campió no existeix, es llança `NoSuchElementException` sense context, i el client rep un error 500 genèric. Suggeriment: usa `.orElseThrow(() -> new ChampionNotFoundException(name))` i mapeja-ho a un 404."

2. **CI massa permissiu** — `on: push` s'executa a qualsevol branca, sense `pull_request`, i sense checkstyle/ruff.
   > "Observació: el workflow només s'activa amb `push` genèric i només executa `mvn test`. Impacte: no hi ha verificació abans de fer merge (no hi ha trigger de `pull_request`), i l'estil de codi no es valida mai. Suggeriment: afegir `pull_request: branches: [main]` com a trigger i un step de checkstyle."

3. **Commit sense sentit** — `"canvis"` no segueix Conventional Commits i no diu res del contingut.
   > "Observació: el missatge de commit és 'canvis'. Impacte: fa impossible generar un changelog automàtic o entendre l'historial sense obrir cada commit. Suggeriment: `feat(api): add champion lookup by name endpoint`."

4. (Opcional, si es detecta) **Falta de validació de `name`** al controller — no es valida que no sigui buit abans de cridar el servei.

3. **No es pot fusionar.** El bug de `NullPointerException` és un problema de correcció real (pot tombar l'endpoint en producció), i el CI no verifica prou coses per confiar-hi.

4. Un cop corregit el bug de `NullPointerException` (és una correcció, no una funcionalitat trencada) i mantenint compatibilitat, **v0.2.0 és correcte** només si el PR original també aporta la nova funcionalitat que justificava el MINOR (l'endpoint `getByName`); si l'únic canvi acaba sent el fix, hauria de ser **v0.1.1**.
