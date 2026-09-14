# Setmana 10 — Dilluns: Pydantic Models — Validacio, Serialitzacio, Type Safety

## Objectiu del Dia

Dominar Pydantic com a eina de validacio i serialitzacio de dades en Python. Al final del dia tindras models `ChampionAnalysis` i `ChampionTrend` completament validats, amb esquema JSON generat automaticament, i entendras per que l'output estructurat dels LLMs es clau per a EsportsPulse.

---

## Teoria

### Per que Pydantic?

En Java, per validar dades fas servir Bean Validation (`@NotNull`, `@Size`, `@Pattern`) sobre records o classes. En Python, sense eines addicionals, no tens cap garantia de tipus en temps d'execucio — els type hints son decoratius. Pydantic resol exactament aixo:

1. **Validacio automatica**: Si un camp espera un `int` i rep un `str`, Pydantic llanca un error clar.
2. **Serialitzacio**: Converteix objectes Python a JSON (i viceversa) amb una sola linia.
3. **Type Safety real**: A diferencia dels type hints natius, Pydantic els fa complir en temps d'execucio.

```python
# Sense Pydantic: Python no valida res en temps d'execucio
def create_champion(name: str, win_rate: float) -> dict:
    # Ningu impedeix passar name=123 o win_rate="hola"
    return {"name": name, "win_rate": win_rate}

# Amb Pydantic: validacio automatica en instanciar l'objecte
from pydantic import BaseModel

class Champion(BaseModel):
    name: str          # Ha de ser un string — si no, error
    win_rate: float    # Ha de ser un float — si no, error
```

### Per que ens importa l'output estructurat dels LLMs?

Aquesta setmana connectarem EsportsPulse amb un LLM (Claude o OpenAI) per generar analisis de campions. El problema: els LLMs retornen text lliure per defecte.

```
# Resposta tipica d'un LLM sense estructura:
"Jinx is a strong ADC with 52.3% win rate. She's good against
short-range champions but struggles against assassins like Zed..."

# Com parsejem aixo? Regex? Split? Es fragil i imprevisible.
```

La solucio: forcar el LLM a retornar JSON que compleixi un esquema Pydantic. Aixi obtenim dades estructurades, validades i directament usables pel nostre codi.

```json
{
  "champion_id": 222,
  "name": "Jinx",
  "role": "ADC",
  "strengths": ["Late game scaling", "AOE damage"],
  "counters": ["Zed", "Fizz"],
  "patch_tier": "S",
  "summary": "Jinx domina el late game amb el seu DPS excepcional."
}
```

### BaseModel: La Base de Tot

Tots els models Pydantic hereten de `BaseModel`. Definim els camps amb type hints i Pydantic fa la resta.

```python
from pydantic import BaseModel, Field
from typing import Literal

# Model aniuat: representa la tendencia d'un campio al llarg del temps
class ChampionTrend(BaseModel):
    """Tendencia del campio entre dos patches consecutius."""

    # Win rate actual — ha de ser entre 0 i 100
    current_win_rate: float = Field(
        ...,  # ... significa que el camp es obligatori
        ge=0.0,  # ge = greater or equal (minim 0%)
        le=100.0,  # le = less or equal (maxim 100%)
        description="Percentatge de victories actual del campio",
        examples=[52.3, 48.7]
    )

    # Win rate del patch anterior — mateixos limits
    previous_win_rate: float = Field(
        ...,
        ge=0.0,
        le=100.0,
        description="Percentatge de victories del patch anterior",
        examples=[50.1, 49.9]
    )

    # Direccio de la tendencia — nomes pot ser un d'aquests tres valors
    trend_direction: Literal["up", "down", "stable"] = Field(
        ...,
        description="Direccio de la tendencia: puja, baixa o estable",
        examples=["up"]
    )

    # Nombre de partides analitzades — minim 100 per ser significatiu
    games_analyzed: int = Field(
        ...,
        gt=0,  # gt = greater than (minim 1 partida)
        description="Nombre total de partides analitzades per calcular la tendencia",
        examples=[15234]
    )
```

### Field(): Control Fi sobre Cada Camp

`Field()` es l'equivalent de les anotacions de Bean Validation en Java. Permet definir restriccions, descripcions i exemples per a cada camp.

| Parametre Pydantic | Equivalent Java Bean Validation | Que fa                     |
|--------------------|---------------------------------|----------------------------|
| `ge=0`             | `@Min(0)`                       | Valor minim (inclusiu)     |
| `le=100`           | `@Max(100)`                     | Valor maxim (inclusiu)     |
| `min_length=1`     | `@Size(min=1)`                  | Longitud minima de string  |
| `max_length=50`    | `@Size(max=50)`                 | Longitud maxima de string  |
| `pattern=r"..."`   | `@Pattern(regexp="...")`        | Regex que ha de complir    |
| `description="…"`  | (Swagger/OpenAPI)               | Documentacio del camp      |

### El Model Principal: ChampionAnalysis

```python
from pydantic import BaseModel, Field, field_validator
from typing import Literal

class ChampionAnalysis(BaseModel):
    """Analisi completa d'un campio de League of Legends generada per IA."""

    # ID unic del campio a l'API de Riot Games
    champion_id: int = Field(
        ...,
        gt=0,
        description="Identificador unic del campio a l'API de Riot",
        examples=[222, 103]
    )

    # Nom del campio — entre 2 i 30 caracters
    name: str = Field(
        ...,
        min_length=2,
        max_length=30,
        description="Nom oficial del campio",
        examples=["Jinx", "Ahri"]
    )

    # Rol principal — nomes valors predefinits (com un enum)
    role: Literal["TOP", "JUNGLE", "MID", "ADC", "SUPPORT"] = Field(
        ...,
        description="Rol principal del campio a la Rift",
        examples=["ADC"]
    )

    # Punts forts — llista de strings, minim 1 element
    strengths: list[str] = Field(
        ...,
        min_length=1,
        max_length=5,
        description="Llista dels punts forts principals del campio",
        examples=[["Late game scaling", "AOE damage", "Tower pushing"]]
    )

    # Contrapartides — campions que el contraresten
    counters: list[str] = Field(
        ...,
        min_length=1,
        max_length=5,
        description="Campions que contraresten aquest campio",
        examples=[["Zed", "Fizz", "Katarina"]]
    )

    # Tier al patch actual — classificacio de forca
    patch_tier: Literal["S", "A", "B", "C", "D"] = Field(
        ...,
        description="Classificacio de forca del campio al patch actual",
        examples=["S"]
    )

    # Tendencia — model aniuat amb les dades de win rate
    champion_trend: ChampionTrend = Field(
        ...,
        description="Tendencia del campio entre patches"
    )

    # Resum generat per la IA — entre 20 i 500 caracters
    summary: str = Field(
        ...,
        min_length=20,
        max_length=500,
        description="Resum analitic generat per IA sobre l'estat actual del campio",
        examples=["Jinx domina el late game amb DPS excepcional i AOE devastador."]
    )

    # Validador personalitzat: el nom del campio ha de comencar amb majuscula
    @field_validator("name")
    @classmethod
    def name_must_be_capitalized(cls, v: str) -> str:
        """Valida que el nom del campio comenci amb lletra majuscula."""
        if not v[0].isupper():
            raise ValueError(f"El nom del campio ha de comencar amb majuscula, rebut: '{v}'")
        return v
```

### Validacio en Accio

```python
# --- CAS 1: Dades valides — tot funciona correctament ---
valid_data = {
    "champion_id": 222,
    "name": "Jinx",
    "role": "ADC",
    "strengths": ["Late game scaling", "AOE damage"],
    "counters": ["Zed", "Fizz"],
    "patch_tier": "S",
    "champion_trend": {
        "current_win_rate": 52.3,
        "previous_win_rate": 50.1,
        "trend_direction": "up",
        "games_analyzed": 15234
    },
    "summary": "Jinx domina el late game amb el seu DPS excepcional i capacitat de neteja."
}

# Pydantic valida automaticament al crear la instancia
analysis = ChampionAnalysis(**valid_data)
print(analysis.name)          # "Jinx"
print(analysis.patch_tier)    # "S"

# --- CAS 2: Dades invalides — Pydantic llanca ValidationError ---
try:
    invalid = ChampionAnalysis(
        champion_id=-1,         # ERROR: gt=0 viola la restriccio
        name="jinx",            # ERROR: no comenca amb majuscula
        role="ASSASSIN",        # ERROR: no es un Literal valid
        strengths=[],           # ERROR: min_length=1 viola la restriccio
        counters=["Zed"],
        patch_tier="S",
        champion_trend={
            "current_win_rate": 150.0,  # ERROR: le=100 viola la restriccio
            "previous_win_rate": 50.0,
            "trend_direction": "up",
            "games_analyzed": 0         # ERROR: gt=0 viola la restriccio
        },
        summary="Curt"          # ERROR: min_length=20 viola la restriccio
    )
except Exception as e:
    # Pydantic mostra TOTS els errors de cop, no nomes el primer
    print(f"Errors de validacio:\n{e}")
```

### Generacio d'Esquema JSON

Pydantic pot generar automaticament l'esquema JSON Schema dels teus models. Aixo es fonamental per a dimarts, quan enviarem aquest esquema al LLM per forcar output estructurat.

```python
import json

# Genera l'esquema JSON Schema del model ChampionAnalysis
schema = ChampionAnalysis.model_json_schema()

# Imprimeix l'esquema formatejat — sera un dict amb tots els camps, tipus i restriccions
print(json.dumps(schema, indent=2))

# L'esquema generat inclou:
# - Tipus de cada camp (string, integer, number, array...)
# - Restriccions (minimum, maximum, minLength, maxLength...)
# - Descripcions (el text que hem posat a Field(description=...))
# - Exemples (el que hem posat a Field(examples=[...]))
# - Definicions dels models aniuats (ChampionTrend)
```

### Comparativa Java vs Python+Pydantic

| Concepte                | Java (Spring Boot)                        | Python (Pydantic)                        |
|------------------------|-------------------------------------------|-------------------------------------------|
| Definicio de dades     | `record ChampionDTO(...)`                 | `class Champion(BaseModel)`               |
| Validacio              | `@NotNull`, `@Min`, `@Pattern`            | `Field(ge=0)`, `field_validator`          |
| Serialitzacio JSON     | Jackson (automatic amb Spring)            | `.model_dump_json()` / `.model_dump()`    |
| Esquema                | SpringDoc/OpenAPI (automatic)             | `.model_json_schema()`                    |
| Errors de validacio    | `MethodArgumentNotValidException`         | `ValidationError`                         |
| Models aniuats         | Records dins de records                   | BaseModel dins de BaseModel               |

---

## Activitat

### Pas 1: Configuracio del Projecte Python (10 min)

```bash
# Navega al directori del projecte Python d'EsportsPulse
cd esportspulse-engine/ai-python

# Crea un entorn virtual per aillar les dependencies
python -m venv .venv

# Activa l'entorn virtual (macOS/Linux)
source .venv/bin/activate

# Instal·la Pydantic
pip install pydantic

# Verifica la instal·lacio
python -c "import pydantic; print(pydantic.__version__)"
```

### Pas 2: Crea els Models (30 min)

Crea el fitxer `ai-python/src/models/champion_analysis.py` amb els models `ChampionTrend` i `ChampionAnalysis` tal com s'han descrit a la teoria. Assegura't d'incloure:
- Tots els camps amb `Field()` i descripcions
- El `field_validator` per al nom
- Tipus `Literal` per a rols, tiers i tendencies

### Pas 3: Testa la Validacio (20 min)

Crea el fitxer `ai-python/src/models/test_models.py`:

```python
# test_models.py — Tests manuals de validacio dels models Pydantic
from champion_analysis import ChampionAnalysis, ChampionTrend
from pydantic import ValidationError

def test_valid_champion():
    """Comprova que dades correctes creen una instancia valida."""
    data = {
        "champion_id": 222,
        "name": "Jinx",
        "role": "ADC",
        "strengths": ["Late game scaling", "AOE damage"],
        "counters": ["Zed", "Fizz"],
        "patch_tier": "S",
        "champion_trend": {
            "current_win_rate": 52.3,
            "previous_win_rate": 50.1,
            "trend_direction": "up",
            "games_analyzed": 15234
        },
        "summary": "Jinx es una ADC devastadora al late game amb gran potencial."
    }
    analysis = ChampionAnalysis(**data)
    assert analysis.name == "Jinx"
    assert analysis.champion_trend.trend_direction == "up"
    print("Test valid_champion: PASSED")

def test_invalid_role():
    """Comprova que un rol invalid llanca ValidationError."""
    try:
        ChampionAnalysis(
            champion_id=1, name="Jinx", role="ASSASSIN",
            strengths=["DPS"], counters=["Zed"], patch_tier="S",
            champion_trend={"current_win_rate": 50, "previous_win_rate": 50,
                           "trend_direction": "stable", "games_analyzed": 100},
            summary="Resum prou llarg per passar la validacio de longitud minima."
        )
        print("Test invalid_role: FAILED (no error)")
    except ValidationError as e:
        print(f"Test invalid_role: PASSED — {e.error_count()} errors detectats")

def test_schema_generation():
    """Comprova que l'esquema JSON es genera correctament."""
    schema = ChampionAnalysis.model_json_schema()
    assert "properties" in schema
    assert "champion_id" in schema["properties"]
    print("Test schema_generation: PASSED")

# Executa tots els tests
if __name__ == "__main__":
    test_valid_champion()
    test_invalid_role()
    test_schema_generation()
    print("\nTots els tests han passat!")
```

### Pas 4: Genera i Inspecciona l'Esquema (10 min)

```python
# Genera l'esquema i guarda'l a un fitxer JSON
import json
from champion_analysis import ChampionAnalysis

schema = ChampionAnalysis.model_json_schema()

# Guarda l'esquema a disc — el necessitarem dema per enviar-lo al LLM
with open("champion_analysis_schema.json", "w") as f:
    json.dump(schema, f, indent=2)

print("Esquema generat i guardat a champion_analysis_schema.json")
```

---

## Checklist de Lliurament

- [ ] Fitxer `champion_analysis.py` creat amb `ChampionTrend` i `ChampionAnalysis`
- [ ] Tots els camps tenen `Field()` amb `description` i `examples`
- [ ] `field_validator` per al camp `name` funciona correctament
- [ ] Dades valides creen instancies sense errors
- [ ] Dades invalides llancen `ValidationError` amb missatges clars
- [ ] Esquema JSON generat amb `model_json_schema()` i guardat a fitxer
- [ ] Entorn virtual creat i Pydantic instal·lat
