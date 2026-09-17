# Setmana 07 — Dilluns: Anti-patrons en Codi Generat per IA

## Objectiu del Dia

Aprendre a llegir i criticar codi de forma professional. Identificar els 5 anti-patrons més perillosos que generen les IA (i també els humans) i saber corregir-los. Al final del dia sabràs fer una revisió de codi bàsica i detectar vulnerabilitats comunes.

---

## Teoria

### El 80% del Teu Temps és Llegir Codi

Un developer junior espera escriure codi tot el dia. La realitat professional:

```
Distribució real del temps d'un developer:
┌──────────────────────────────────────────────────┐
│ ████████████████████████████████████████ 80% Llegir  │
│   → Entendre codi existent, reviews, debugging       │
│ ████████ 20% Escriure                                │
│   → Codi nou, refactoring                            │
└──────────────────────────────────────────────────┘
```

**Escenari real:** El teu primer dia a una empresa:
1. Et donen accés a un repositori de 200.000 linies.
2. Et donen un ticket Jira: "Bug: el winRate es mostra amb decimals incorrectes".
3. Has de trobar on es calcula el `winRate`, entendre la logica, i corregir-ho.

Ningu t'explicara el codi linia per linia. Has de saber llegir-lo sol.

---

### Principis de Clean Code

Abans d'entrar als anti-patrons, tres principis fonamentals:

#### 1. Noms Significatius

```java
// ❌ Noms críptics — què fa això?
public List<ChampionRecord> get(String s, int n) {
    return repo.findAll().stream()
        .filter(g -> g.name().contains(s))
        .limit(n)
        .toList();
}

// ✅ Noms descriptius — s'entén sense llegir el cos del mètode
public List<ChampionRecord> searchByName(String keyword, int maxResults) {
    // Filtra campions que continguin la paraula clau al nom
    // i limita els resultats per evitar respostes massa grans
    return championRepository.findAll().stream()
        .filter(champion -> champion.name().contains(keyword))
        .limit(maxResults)
        .toList();
}
```

```python
# ❌ Críptic
def get(s, n):
    return [c for c in repo.find_all() if s in c.name][:n]

# ✅ Clar
def search_champions_by_name(keyword: str, max_results: int) -> list[ChampionRecord]:
    """Cerca campions que continguin la paraula clau al nom."""
    # Retorna només els primers max_results per rendiment
    return [c for c in champion_repository.find_all()
            if keyword in c.name][:max_results]
```

#### 2. Funcions Petites (Una Sola Responsabilitat)

```java
// ❌ Funció que fa massa coses: valida, cerca, transforma i retorna
public String processChampion(String name, String role) {
    if (name == null || name.isEmpty()) return "error";
    var champ = repository.findByName(name);
    if (champ == null) return "not found";
    if (!champ.role().equals(role)) return "wrong role";
    return champ.name() + " - " + champ.winRate() + "%";
}

// ✅ Cada funció fa una sola cosa
public ChampionRecord findChampionOrThrow(String name) {
    // Busca el campió o llança una excepció descriptiva
    return repository.findByName(name)
        .orElseThrow(() -> new ChampionNotFoundException(name));
}

public String formatChampionSummary(ChampionRecord champion) {
    // Formata la informació del campió per mostrar a l'usuari
    return "%s - %.1f%%".formatted(champion.name(), champion.winRate());
}
```

#### 3. Early Return (Evitar Niuament Excessiu)

```java
// ❌ Piràmide de la mort (nesting profund)
public void updateChampion(Long id, ChampionUpdateRequest request) {
    if (id != null) {
        var champion = repository.findById(id);
        if (champion.isPresent()) {
            if (request.isValid()) {
                // finalment el codi útil, a 4 nivells de profunditat
                champion.get().update(request);
                repository.save(champion.get());
            }
        }
    }
}

// ✅ Early return — el cas feliç queda al final, sense nesting
public void updateChampion(Long id, ChampionUpdateRequest request) {
    // Validacions ràpides al principi — fallen aviat si hi ha problemes
    if (id == null) throw new IllegalArgumentException("L'ID no pot ser null");

    var champion = repository.findById(id)
        .orElseThrow(() -> new ChampionNotFoundException(id));

    if (!request.isValid()) throw new InvalidRequestException(request);

    // Cas feliç: tot és correcte, actualitzem
    champion.update(request);
    repository.save(champion);
}
```

---

### Els 5 Anti-patrons Més Perillosos del Codi IA

#### Anti-patró 1: SQL Injection

L'atac més antic i encara el més comú. La IA sovint concatena strings en queries.

```java
// ❌ VULNERABLE — Concatenació directa de l'input de l'usuari
// Un atacant pot enviar: name = "' OR '1'='1" i obtenir TOTS els campions
var query = "SELECT * FROM champions WHERE name = '" + name + "'";
var result = jdbcTemplate.queryForObject(query, Champion.class);

// ✅ SEGUR — Parameterized query amb placeholder (?)
// El driver JDBC escapa el valor, no es pot trencar la sintaxi SQL
var query = "SELECT * FROM champions WHERE name = ?";
var result = jdbcTemplate.queryForObject(query, Champion.class, name);
```

```python
# ❌ VULNERABLE — f-string dins de SQL
# Un atacant pot injectar: name = "'; DROP TABLE champions; --"
def find_by_name(name: str) -> list[dict]:
    cursor.execute(f"SELECT * FROM champions WHERE name = '{name}'")
    return cursor.fetchall()

# ✅ SEGUR — Parameterized query amb placeholders
# El driver de la base de dades s'encarrega d'escapar l'input
def find_by_name(name: str) -> list[dict]:
    cursor.execute("SELECT * FROM champions WHERE name = %s", (name,))
    return cursor.fetchall()
```

**Per què passa?** La IA aprèn de milions d'exemples, incloent codi antic i insegur. Concatenar strings és "més simple" i apareix en tutorials vells.

---

#### Anti-patró 2: Secrets Hardcoded

La IA no entén que el codi anirà a un repositori públic.

```java
// ❌ PERILLOS — La clau API queda al repositori per sempre
// Fins i tot si l'esborres, git recorda tots els commits anteriors
private static final String API_KEY = "sk-abc123secretkey456";
private static final String DB_PASSWORD = "admin123";

// ✅ SEGUR — Variables d'entorn, mai al codi font
// El fitxer .env NO es puja a git (està al .gitignore)
private final String apiKey = System.getenv("RIOT_API_KEY");
private final String dbPassword = System.getenv("DB_PASSWORD");
```

```python
# ❌ PERILLOS — Secret visible a GitHub
API_KEY = "sk-abc123secretkey456"
DATABASE_URL = "postgresql://admin:admin123@localhost/esportspulse"

# ✅ SEGUR — Ús de variables d'entorn amb python-dotenv
import os
from dotenv import load_dotenv

load_dotenv()  # Carrega variables del fitxer .env (que està al .gitignore)

API_KEY = os.getenv("RIOT_API_KEY")
DATABASE_URL = os.getenv("DATABASE_URL")
```

**Fitxer `.env` (MAI a git):**
```env
RIOT_API_KEY=sk-abc123secretkey456
DATABASE_URL=postgresql://admin:admin123@localhost/esportspulse
```

**Fitxer `.gitignore` (SEMPRE a git):**
```gitignore
# Secrets — mai pujar al repositori
.env
*.key
credentials/
```

---

#### Anti-patró 3: NullPointerException Amagat

La IA sovint ignora que `findById` retorna `Optional` (recorda el patró Repository de la S2) i el "desempaqueta" sense comprovar si hi ha valor.

```java
// ❌ PERILLÓS — Si el campió no existeix, NullPointerException en producció
// Això pot fer caure tota l'aplicació sense cap missatge útil
public ChampionRecord getChampion(Long id) {
    return repository.findById(id).get(); // Boom! NoSuchElementException
}

// ✅ SEGUR — Gestió explícita del cas "no trobat"
// L'excepció és descriptiva: l'usuari sap què ha passat
public ChampionRecord getChampion(Long id) {
    return repository.findById(id)
        .orElseThrow(() -> new ChampionNotFoundException(
            "Campió amb ID %d no trobat".formatted(id)));
}
```

```python
# ❌ PERILLÓS — AttributeError si el campió no existeix
# champion serà None, i None no té .name
def get_champion(champion_id: int) -> dict:
    champion = champion_repository.find_by_id(champion_id)
    return {"name": champion.name}  # AttributeError: 'NoneType' has no attribute 'name'

# ✅ SEGUR — Comprovació explícita amb error descriptiu
def get_champion(champion_id: int) -> dict:
    champion = champion_repository.find_by_id(champion_id)
    if champion is None:
        raise ChampionNotFoundError(f"Campió amb ID {champion_id} no trobat")
    return {"name": champion.name}
```

---

#### Anti-patró 4: Test que No Testa Res

El pitjor anti-patró perquè et dona falsa seguretat. El test passa, però no valida res.

**Nota:** Els exemples fan servir Mockito (`when`, `verify`, `@Mock`) i `pytest-mock`, que encara no hem vist (arribaran a S8). No cal que entenguis cada línia de sintaxi — fixa't només en el concepte: aquest primer test verifica el que li hem dit al mock que faci, no el comportament real del servei. És exactament l'exercici de llegir codi que no domines del tot, com dèiem al principi del dia.

```java
// ❌ FALS TEST — Testa el mock, no el servei real
// Li dius al mock "retorna X" i després comproves que retorna X. Obvio!
@Test
void testFindChampion() {
    // Preparem el mock perquè retorni Jinx
    var jinx = new ChampionRecord(1L, "Jinx", "ADC", 51.2);
    when(repository.findById(1L)).thenReturn(Optional.of(jinx));

    // Cridem el servei (que internament crida al mock)
    var result = service.findById(1L);

    // Comprovem que el resultat és... el que li hem dit al mock que retorni
    // Això NO testa la lògica del servei, testa que Mockito funciona
    assertEquals("Jinx", result.name());
}

// ✅ TEST REAL — Testa la lògica del servei, no el mock
@Test
void findById_whenChampionExists_returnsChampionRecord() {
    // Arrange — preparem el mock
    var jinx = new ChampionRecord(1L, "Jinx", "ADC", 51.2);
    when(repository.findById(1L)).thenReturn(Optional.of(jinx));

    // Act — cridem el servei
    var result = service.findById(1L);

    // Assert — comprovem comportament REAL del servei
    assertNotNull(result);
    assertEquals("Jinx", result.name());
    verify(repository).findById(1L); // Verifiquem que ha cridat al repo
}

@Test
void findById_whenChampionNotFound_throwsException() {
    // AQUEST és el test important: què passa quan NO existeix?
    when(repository.findById(999L)).thenReturn(Optional.empty());

    // Verifiquem que el servei llança l'excepció correcta
    assertThrows(ChampionNotFoundException.class,
        () -> service.findById(999L));
}
```

```python
# ❌ FALS TEST — No comprova res útil
def test_find_champion(mocker):
    mock_repo = mocker.patch("service.repository")
    mock_repo.find_by_id.return_value = Champion(1, "Jinx", "ADC", 51.2)

    result = service.find_by_id(1)

    # Només comprova que el mock retorna el que li hem dit...
    assert result.name == "Jinx"

# ✅ TEST REAL — Testa els casos importants (happy path + errors)
def test_find_champion_returns_record(mocker):
    """Verifica que el servei retorna correctament un campió existent."""
    mock_repo = mocker.patch("service.repository")
    mock_repo.find_by_id.return_value = Champion(1, "Jinx", "ADC", 51.2)

    result = service.find_by_id(1)

    assert result.name == "Jinx"
    mock_repo.find_by_id.assert_called_once_with(1)  # Verifica la crida

def test_find_champion_not_found_raises(mocker):
    """Verifica que el servei llança error quan el campió no existeix."""
    mock_repo = mocker.patch("service.repository")
    mock_repo.find_by_id.return_value = None

    with pytest.raises(ChampionNotFoundError):
        service.find_by_id(999)
```

**Regla d'or:** Si pots eliminar la linia que crida al servei i el test segueix passant, el test no testa res.

---

#### Anti-patró 5: Excepció Silenciada

La IA sovint genera blocs `catch` buits perquè "el codi compila".

```java
// ❌ PERILLÓS — L'error desapareix, el sistema falla en silenci
// Hores de debugging perquè no hi ha cap rastre de l'error
public List<ChampionRecord> importChampions(String filePath) {
    try {
        return fileReader.readChampions(filePath);
    } catch (Exception e) {
        // TODO: handle exception ← La IA deixa això i tu t'oblides
        return List.of(); // Retorna llista buida com si tot anés bé
    }
}

// ✅ CORRECTE — Loguejar i relançar (o gestionar de veritat)
public List<ChampionRecord> importChampions(String filePath) {
    try {
        return fileReader.readChampions(filePath);
    } catch (IOException e) {
        // Loguegem l'error amb context per poder depurar
        log.error("Error important fitxer de campions: {}", filePath, e);
        // Rellancem amb una excepció del nostre domini
        throw new ChampionImportException("No s'ha pogut importar: " + filePath, e);
    }
}
```

```python
# ❌ PERILLÓS — pass silencia l'error completament
def import_champions(file_path: str) -> list[Champion]:
    try:
        return file_reader.read_champions(file_path)
    except Exception:
        pass  # L'error desapareix, impossible depurar
        return []

# ✅ CORRECTE — Loguegem i rellancem
import logging

logger = logging.getLogger(__name__)

def import_champions(file_path: str) -> list[Champion]:
    try:
        return file_reader.read_champions(file_path)
    except IOError as e:
        # Loguegem amb el context necessari per depurar
        logger.error("Error important fitxer de campions: %s", file_path, exc_info=True)
        raise ChampionImportError(f"No s'ha pogut importar: {file_path}") from e
```

---

## Activitat

### Exercici 1: Detecta Anti-patrons (30 min)

Revisa el seguent codi i identifica **tots** els anti-patrons. Escriu la correcció per a cadascun.

```java
// ChampionService.java — Quants anti-patrons hi trobes?
public class ChampionService {
    private static final String API_KEY = "rgapi-1234-5678-abcd";

    public Champion findChampion(String name) {
        try {
            var query = "SELECT * FROM champions WHERE name = '" + name + "'";
            var result = jdbcTemplate.queryForObject(query, Champion.class);
            return result;
        } catch (Exception e) {
            return null;
        }
    }
}
```

**Resposta esperada:** Has de trobar almenys 3 anti-patrons (SQL Injection, secret hardcoded, excepció silenciada + retorn null).

### Exercici 2: Prompt Engineering amb IA (30 min)

1. Dona el codi de l'Exercici 1 a una IA (ChatGPT, Claude, Copilot).
2. Demana-li: "Revisa aquest codi i identifica problemes de seguretat i qualitat."
3. Compara la resposta de la IA amb la teva analisi manual.
4. Escriu un document comparant:
   - Que has trobat tu que la IA no ha trobat?
   - Que ha trobat la IA que tu no havies vist?
   - La IA ha generat algun fals positiu?

### Exercici 3: Refactoritza el Servei (45 min)

Refactoritza el `ChampionManagementService` del teu projecte aplicant:
- Noms significatius (renombra si cal)
- Early return
- Gestió correcta de `Optional`
- Excepcions descriptives (no nulls)
- Comentaris que expliquin el **perquè**, no el **què**

---

## Checklist de Lliurament

- [ ] He identificat els 5 anti-patrons als exemples de la teoria
- [ ] He completat l'Exercici 1 amb totes les correccions
- [ ] He fet l'Exercici 2 comparant la meva analisi amb la de la IA
- [ ] He refactoritzat el servei del meu projecte aplicant Clean Code
- [ ] Tot el codi compila (`mvn compile`) i els tests passen (`mvn test`)
- [ ] Commit amb missatge: `refactor: apply clean code principles to service layer`
