# Setmana 05 — Dijous: CLI Python i Tests d'Integració

## Objectiu del Dia

Crear un client CLI en Python que consumeixi l'API REST de Java, i escriure tests d'integració amb Spring Boot. Al final del dia tindràs un CLI funcional, tests que validen el cicle complet CRUD i un PR creat.

---

## Teoria

### Per Què un CLI en Python?

El projecte EsportsPulse és políglota: el backend és Java i els serveis d'intel·ligència artificial són Python. El CLI és el primer pont entre els dos mons:

```
CLI Python (requests)  ──HTTP──→  API Java (Spring Boot)  ──JPA──→  H2 DB
     ↑                                   ↑
  L'usuari interactua               Endpoints REST
  des de la terminal               que hem creat
```

### Client HTTP amb requests

Primer cal instal·lar la dependència: `pip install requests` (afegir `requests>=2.31.0` a `requirements.txt`).

```python
# === champions_client.py ===
# Client HTTP que encapsula les crides a l'API de Champions
# Separar el client del CLI facilita el testing i la reutilització

import requests

class ChampionsClient:
    """Client per a l'API REST de Champions d'EsportsPulse."""

    def __init__(self, base_url="http://localhost:8080/api"):
        # URL base de l'API — configurable per entorns diferents
        self.base_url = base_url
        # Timeout de 10 segons per evitar que el CLI es quedi penjat
        self.timeout = 10

    def list_champions(self, name=None, role=None, min_games=None):
        """Obté la llista de campions, opcionalment filtrada."""
        # Construïm els query params dinàmicament
        # Només incloem els paràmetres que tenen valor (no None)
        params = {}
        if name:
            params["name"] = name
        if role:
            params["role"] = role
        if min_games is not None:
            params["minGames"] = min_games

        # GET /api/champions?name=X&role=Y
        response = requests.get(
            f"{self.base_url}/champions",
            params=params,        # requests codifica els params automàticament
            timeout=self.timeout
        )
        # raise_for_status() llença una excepció si el status és 4xx o 5xx
        response.raise_for_status()
        # .json() parseja el cos de la resposta com a diccionari/llista Python
        return response.json()

    def get_champion(self, champion_id):
        """Obté un campió pel seu ID."""
        response = requests.get(
            f"{self.base_url}/champions/{champion_id}",
            timeout=self.timeout
        )
        response.raise_for_status()
        return response.json()

    def create_champion(self, name, role, win_rate):
        """Crea un campió nou a l'API."""
        # El cos de la petició POST és un diccionari que requests serialitza a JSON
        payload = {
            "name": name,
            "role": role,
            "winRate": win_rate
        }
        response = requests.post(
            f"{self.base_url}/champions",
            json=payload,          # json= serialitza i posa Content-Type automàticament
            timeout=self.timeout
        )
        response.raise_for_status()
        return response.json()

    def delete_champion(self, champion_id):
        """Esborra un campió pel seu ID."""
        response = requests.delete(
            f"{self.base_url}/champions/{champion_id}",
            timeout=self.timeout
        )
        response.raise_for_status()  # 204 No Content — no hi ha cos a parsejar
```

### CLI amb argparse

```python
# === cli.py ===
# Interfície de línia de comandes per interactuar amb l'API de Champions
# Utilitza argparse per definir subcomandes amb arguments tipats

import argparse
import sys
from champions_client import ChampionsClient

def main():
    parser = argparse.ArgumentParser(description="EsportsPulse CLI — Gestiona campions")
    parser.add_argument("--base-url", default="http://localhost:8080/api")
    subparsers = parser.add_subparsers(dest="command", help="Comanda a executar")

    # Subcomanda: list-champions (amb filtres opcionals)
    lp = subparsers.add_parser("list-champions", help="Llista campions")
    lp.add_argument("--name", help="Filtra per nom")
    lp.add_argument("--role", help="Filtra per rol")
    lp.add_argument("--min-games", type=int, help="Mínim de partides")

    # Subcomanda: get-champion
    gp = subparsers.add_parser("get-champion", help="Obté un campió per ID")
    gp.add_argument("id", type=int)

    # Subcomanda: create-champion
    cp = subparsers.add_parser("create-champion", help="Crea un campió nou")
    cp.add_argument("--name", required=True)
    cp.add_argument("--role", required=True)
    cp.add_argument("--win-rate", type=float, required=True)

    # Subcomanda: delete-champion
    dp = subparsers.add_parser("delete-champion", help="Esborra un campió")
    dp.add_argument("id", type=int)

    args = parser.parse_args()
    if not args.command:
        parser.print_help()
        sys.exit(1)

    client = ChampionsClient(base_url=args.base_url)
    try:
        execute_command(client, args)
    except Exception as e:
        handle_error(e)
        sys.exit(1)


def execute_command(client, args):
    """Executa la comanda especificada per l'usuari."""
    if args.command == "list-champions":
        champions = client.list_champions(args.name, args.role, args.min_games)
        if not champions:
            print("No s'han trobat campions.")
            return
        # Format tabular per a fàcil lectura
        print(f"{'ID':<5} {'Nom':<15} {'Rol':<12} {'Win Rate':<10} {'Partides'}")
        print("-" * 52)
        for c in champions:
            print(f"{c['id']:<5} {c['name']:<15} {c['role']:<12} "
                  f"{c['winRate']:<10.1f} {c['totalGames']}")
    elif args.command == "get-champion":
        c = client.get_champion(args.id)
        print(f"ID: {c['id']} | {c['name']} | {c['role']} | "
              f"WR: {c['winRate']}% | Partides: {c['totalGames']}")
    elif args.command == "create-champion":
        created = client.create_champion(args.name, args.role, args.win_rate)
        print(f"Campió creat amb ID: {created['id']} — {created['name']}")
    elif args.command == "delete-champion":
        client.delete_champion(args.id)
        print(f"Campió amb ID {args.id} esborrat correctament.")


def handle_error(error):
    """Gestiona errors HTTP i de connexió de forma amigable."""
    import requests as req
    if isinstance(error, req.exceptions.ConnectionError):
        print("ERROR: No s'ha pogut connectar amb l'API.")
    elif isinstance(error, req.exceptions.HTTPError):
        status = error.response.status_code
        if status == 404:
            print("ERROR: Recurs no trobat (404).")
        elif status == 400:
            # Mostrem els detalls de validació si n'hi ha
            print("ERROR: Dades no vàlides (400).")
            try:
                for field, msg in error.response.json().items():
                    print(f"  - {field}: {msg}")
            except ValueError:
                print(f"  {error.response.text}")
        else:
            print(f"ERROR HTTP {status}: {error.response.text}")
    else:
        print(f"ERROR inesperat: {error}")

if __name__ == "__main__":
    main()
```

### Tests d'Integració amb Spring Boot

Els tests d'integració verifiquen que totes les capes funcionen juntes (controller + service + repository + BD):

```java
// === Test d'integració: arrenca Spring Boot complet amb BD H2 ===
// @SpringBootTest arrenca tota l'aplicació com si fos producció
// webEnvironment = RANDOM_PORT evita conflictes de port amb altres tests
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ChampionApiIntegrationTest {

    // TestRestTemplate: client HTTP per fer peticions reals al servidor
    @Autowired
    private TestRestTemplate restTemplate;

    // Repositori per preparar dades de test
    @Autowired
    private ChampionRepository repository;

    // Netegem la BD abans de cada test per evitar dependències entre tests
    @BeforeEach
    void setUp() {
        repository.deleteAll();
    }

    @Test
    void shouldCreateAndRetrieveChampion() {
        // POST per crear, després GET per verificar que existeix
        var request = new CreateChampionRequest("Ahri", "Mage", 52.3);
        var createResp = restTemplate.postForEntity(
            "/api/champions", request, ChampionDTO.class);
        assertThat(createResp.getStatusCode()).isEqualTo(HttpStatus.CREATED);

        // GET del campió creat — les dades han de coincidir
        var getResp = restTemplate.getForEntity(
            "/api/champions/" + createResp.getBody().id(), ChampionDTO.class);
        assertThat(getResp.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(getResp.getBody().name()).isEqualTo("Ahri");
        assertThat(getResp.getBody().winRate()).isEqualTo(52.3);
    }

    @Test
    void shouldReturnNotFoundForNonExistentChampion() {
        // GET d'un ID que no existeix ha de retornar 404
        ResponseEntity<String> response = restTemplate.getForEntity(
            "/api/champions/9999",
            String.class
        );
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
    }

    @Test
    void shouldReturnBadRequestForInvalidData() {
        // Enviem dades invàlides: nom buit i win rate fora de rang
        CreateChampionRequest invalidRequest = new CreateChampionRequest("", "Mage", 150.0);
        ResponseEntity<String> response = restTemplate.postForEntity(
            "/api/champions", invalidRequest, String.class
        );
        // L'API ha de retornar 400 Bad Request amb els errors de validació
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.BAD_REQUEST);
    }

    @Test
    void shouldDeleteAndVerifyGone() {
        // Creem, esborrem, i verifiquem que GET retorna 404
        ResponseEntity<ChampionDTO> created = restTemplate.postForEntity(
            "/api/champions",
            new CreateChampionRequest("Jinx", "Marksman", 51.8),
            ChampionDTO.class
        );
        Long id = created.getBody().id();
        restTemplate.delete("/api/champions/" + id);

        ResponseEntity<String> getResponse = restTemplate.getForEntity(
            "/api/champions/" + id, String.class
        );
        assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
    }
}
```

---

## Activitat

### Part 1: Crea el CLI Python

1. Crea `ai-python/src/champions_client.py` amb la classe `ChampionsClient`
2. Crea `ai-python/src/cli.py` amb les subcomandes: `list-champions`, `get-champion`, `create-champion`, `delete-champion`
3. Afegeix `requests` a `requirements.txt`

### Part 2: Testa el CLI contra l'API Java

```bash
# Arrenca l'API i prova totes les comandes
cd backend-java && mvn spring-boot:run &
python ai-python/src/cli.py create-champion --name "Ahri" --role "Mage" --win-rate 52.3
python ai-python/src/cli.py list-champions
python ai-python/src/cli.py get-champion 1
python ai-python/src/cli.py get-champion 9999    # Ha de mostrar error 404
```

### Part 3: Escriu Tests d'Integració i Crea el PR

1. Crea `ChampionApiIntegrationTest.java` amb els tests de la teoria
2. Executa `mvn clean verify` per assegurar que tot passa
3. Crea la branca `feature/week5-rest-api` i obre un PR amb tots els canvis de la setmana

---

## Checklist de Lliurament

- [ ] `champions_client.py` funciona amb tots els mètodes HTTP (GET, POST, PUT, DELETE)
- [ ] CLI amb subcomandes: `list-champions`, `get-champion`, `create-champion`, `delete-champion`
- [ ] El CLI gestiona errors: connexió, 404, 400 amb missatges clars
- [ ] Tests d'integració amb `@SpringBootTest` i `TestRestTemplate`
- [ ] Tests cobreixen: create+get, delete+get404, validació 400, update, llista
- [ ] `mvn clean verify` passa sense errors
- [ ] PR creat amb descripció clara de tots els canvis de la setmana
- [ ] Commit: `feat(api): add Python CLI consumer and integration tests`
