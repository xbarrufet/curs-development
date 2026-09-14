# Setmana 14 — Dimarts: Chunking i Metadades: Preparar Documents per a IA

## Objectiu del Dia

Aprendre a dividir documents en fragments (chunks) de mida adequada i enriquir-los amb metadades. Al final del dia tindràs un script Python que converteix patch notes en chunks estructurats preparats per a indexació.

---

## Teoria

### Què és el Chunking?

Chunking és el procés de dividir un document gran en trossos més petits i manejables. Per què cal?

```
┌─────────────────────────────────────────────────────────┐
│  Document Sencer (10.000 tokens)                        │
│                                                          │
│  Si el passes tot al LLM:                               │
│  - Pot superar el límit de context                       │
│  - El model es "perd" entre tanta informació             │
│  - No saps quina part és rellevant per la pregunta       │
│  - Costa més tokens (= més diners)                       │
│                                                          │
│  Si el divideixes en chunks (300-500 tokens):            │
│  - Pots buscar NOMÉS els chunks rellevants               │
│  - El context és precís i net                            │
│  - Menys tokens per consulta                             │
│  - Millors respostes                                     │
└─────────────────────────────────────────────────────────┘
```

### La Mida Importa

No tots els chunks són iguals. La mida afecta directament la qualitat:

```python
# MASSA GRAN — El chunk porta informació irrellevant (soroll)
chunk_gran = """
Patch 14.15 Notes: Jinx base AD reduced from 59 to 57.
Fishbones bonus range reduced. Meanwhile Ahri charm duration
increased to 1.5s at all ranks. Fox-Fire mana cost reduced.
Thresh Q cooldown slightly reduced. Flay slow increased.
Stormsurge proc damage nerfed. New skin line announced for
summer event. Bug fixes for Mordekaiser ultimate interaction
with terrain. Ranked season split 2 starts next week...
"""
# Si preguntes "Què va canviar de Jinx?" → el chunk porta
# informació de 4 champions, items, skins i bugs barrejats

# MASSA PETIT — El chunk no té prou context
chunk_petit = "Base AD: 59 → 57"
# Qui? Quin patch? Quin joc? No ho sabem.

# MIDA CORRECTA — Un chunk per concepte, amb context suficient
chunk_bo = """
## Jinx — Patch 14.15 (Nerf)
- Base AD: 59 → 57
- Fishbones (Q) bonus range: 100/125/150/175/200 → 75/100/125/150/175
Context: Mid-season adjustments focused on bot lane carry diversity.
"""
# Saps el champion, el patch, el tipus de canvi i els detalls
```

### Overlap: Per Què els Chunks es Superposen

Quan talles un document, pots perdre context a les fronteres. L'overlap resol això:

```
Document: [A B C D E F G H I J K L M N O]

Sense overlap (chunks de 5):
  Chunk 1: [A B C D E]
  Chunk 2: [F G H I J]        ← Si la idea important va de D a F,
  Chunk 3: [K L M N O]          es perd entre dos chunks!

Amb overlap de 2 (chunks de 5, overlap de 2):
  Chunk 1: [A B C D E]
  Chunk 2: [D E F G H]        ← D i E apareixen als dos chunks.
  Chunk 3: [G H I J K]          La idea D→F queda capturada.
  Chunk 4: [J K L M N O]

# L'overlap típic és del 10-20% de la mida del chunk
# Per chunks de 500 tokens → overlap de 50-100 tokens
```

### Metadades: El Context que Necessita Cada Chunk

Un chunk sense metadades és com un paràgraf arrencat d'un llibre — no saps d'on ve. Les metadades permeten:

1. **Filtrar** abans de buscar (només chunks d'un patch concret)
2. **Contextualitzar** la resposta ("segons el patch 14.15...")
3. **Ordenar** resultats per rellevància temporal
4. **Deduplicar** si un document s'indexa dues vegades

```python
from dataclasses import dataclass, field
from datetime import date

@dataclass
class ChunkMetadata:
    """
    Metadades que acompanyen cada chunk.
    Permeten filtrar i contextualitzar els resultats de cerca.
    """
    source: str          # D'on ve: "patch_notes", "wiki", "guide"
    game: str            # Joc: "league_of_legends", "valorant"
    patch_version: str   # Versió: "14.15"
    date: date           # Data de publicació
    section: str         # Secció: "champion_changes", "item_changes"
    entity: str          # Entitat principal: "Jinx", "Stormsurge"
    change_type: str     # Tipus: "buff", "nerf", "adjustment"

@dataclass
class Chunk:
    """
    Un fragment de document preparat per a indexació.
    Combina el text del chunk amb les seves metadades.
    """
    id: str                        # Identificador únic
    content: str                   # El text del chunk
    metadata: ChunkMetadata        # Context del chunk
    token_count: int = 0           # Mida en tokens (aproximada)
    embedding: list[float] = field(default_factory=list)  # Vector (es calcula després)
```

---

## Activitat

### 1. Model de Dades amb Dataclasses (15 min)

Crea el model de dades per als chunks. Usem `dataclasses` perquè ja les coneixes de Python bàsic i són netes:

```python
# ai-python/src/knowledge/chunk_model.py
"""
Models de dades per al sistema de chunking.
Definim l'estructura de cada chunk i les seves metadades.
"""

from dataclasses import dataclass, field
from datetime import date
from typing import Optional
import hashlib
import json


@dataclass
class ChunkMetadata:
    """
    Metadades associades a un chunk de text.
    Cada camp permet un tipus de filtratge o contextualització.
    """
    source: str              # Origen: "patch_notes", "wiki", "guide"
    game: str                # Joc associat
    patch_version: str       # Versió del patch (si aplica)
    date: date               # Data de publicació o ingestió
    section: str             # Secció dins del document
    entity: str              # Entitat principal (champion, item...)
    change_type: str = ""    # buff/nerf/adjustment/info (opcional)

    def to_dict(self) -> dict:
        """Converteix a diccionari per a serialització (Qdrant ho necessita)."""
        return {
            "source": self.source,
            "game": self.game,
            "patch_version": self.patch_version,
            "date": self.date.isoformat(),
            "section": self.section,
            "entity": self.entity,
            "change_type": self.change_type,
        }


@dataclass
class Chunk:
    """
    Un fragment de document preparat per a indexació vectorial.
    
    Conté el text, les metadades i opcionalment l'embedding.
    L'id es genera automàticament a partir del contingut
    per garantir idempotència (si indexes dues vegades, no es duplica).
    """
    content: str
    metadata: ChunkMetadata
    token_count: int = 0
    embedding: list[float] = field(default_factory=list)

    @property
    def id(self) -> str:
        """
        Genera un ID determinístic basat en el contingut.
        Això fa que la ingestió sigui idempotent:
        el mateix document sempre genera els mateixos IDs.
        """
        # Usem un hash del contingut + metadades clau
        key = f"{self.metadata.source}:{self.metadata.patch_version}:{self.metadata.entity}:{self.content}"
        return hashlib.sha256(key.encode()).hexdigest()[:16]

    def to_dict(self) -> dict:
        """Serialitza el chunk complet per a depuració o exportació."""
        return {
            "id": self.id,
            "content": self.content,
            "metadata": self.metadata.to_dict(),
            "token_count": self.token_count,
        }
```

### 2. Parser de Patch Notes (30 min)

Escriu un script que converteixi les dades d'exemple (de dilluns) en chunks:

```python
# ai-python/src/knowledge/chunker.py
"""
Chunker per a patch notes.
Converteix dades estructurades en chunks preparats per a indexació.

Estratègia de chunking:
- Un chunk per champion/item per patch
- Cada chunk inclou tot el context necessari per respondre preguntes
- Les metadades permeten filtrar per patch, champion, tipus de canvi
"""

from datetime import date
from chunk_model import Chunk, ChunkMetadata

# Importem les dades d'exemple del dilluns
import sys
sys.path.insert(0, ".")
from sample_data import SAMPLE_PATCHES


def estimate_tokens(text: str) -> int:
    """
    Estimació ràpida del nombre de tokens.
    Regla general: 1 token ≈ 4 caràcters en anglès.
    No és precís però és suficient per a chunking.
    """
    return len(text) // 4


def chunk_champion_change(
    champion: dict,
    patch: dict,
) -> Chunk:
    """
    Crea un chunk per a un canvi de champion específic.
    
    Cada chunk inclou:
    - Nom del champion i rol
    - Tipus de canvi (buff/nerf/adjustment)
    - Tots els canvis detallats
    - Context del patch (versió, data, highlights)
    
    Això permet respondre preguntes com:
    "Què va canviar de Jinx al 14.15?"
    """
    # Construïm el text del chunk amb context complet
    lines = [
        f"# {champion['name']} — Patch {patch['patch']} ({champion['change_type'].upper()})",
        f"Rol: {champion['role']}",
        f"Data: {patch['date']}",
        f"Context del patch: {patch['highlights']}",
        "",
        "Canvis:",
    ]
    for change in champion["changes"]:
        lines.append(f"- {change}")

    content = "\n".join(lines)

    metadata = ChunkMetadata(
        source="patch_notes",
        game="league_of_legends",
        patch_version=patch["patch"],
        date=date.fromisoformat(patch["date"]),
        section="champion_changes",
        entity=champion["name"],
        change_type=champion["change_type"],
    )

    return Chunk(
        content=content,
        metadata=metadata,
        token_count=estimate_tokens(content),
    )


def chunk_item_change(
    item: dict,
    patch: dict,
) -> Chunk:
    """Crea un chunk per a un canvi d'item. Mateixa lògica que champions."""
    lines = [
        f"# {item['name']} — Patch {patch['patch']} ({item['change_type'].upper()})",
        f"Data: {patch['date']}",
        f"Context del patch: {patch['highlights']}",
        "",
        "Canvis:",
    ]
    for change in item["changes"]:
        lines.append(f"- {change}")

    content = "\n".join(lines)

    metadata = ChunkMetadata(
        source="patch_notes",
        game="league_of_legends",
        patch_version=patch["patch"],
        date=date.fromisoformat(patch["date"]),
        section="item_changes",
        entity=item["name"],
        change_type=item["change_type"],
    )

    return Chunk(
        content=content,
        metadata=metadata,
        token_count=estimate_tokens(content),
    )


def chunk_all_patches(patches: list[dict]) -> list[Chunk]:
    """
    Processa totes les patch notes i genera chunks.
    Retorna una llista de chunks preparats per a indexació.
    """
    chunks = []

    for patch in patches:
        # Un chunk per cada canvi de champion
        for champion in patch["champion_changes"]:
            chunk = chunk_champion_change(champion, patch)
            chunks.append(chunk)

        # Un chunk per cada canvi d'item
        for item in patch.get("item_changes", []):
            chunk = chunk_item_change(item, patch)
            chunks.append(chunk)

    return chunks


if __name__ == "__main__":
    # Genera els chunks i mostra'ls per verificar
    chunks = chunk_all_patches(SAMPLE_PATCHES)

    print(f"Total chunks generats: {len(chunks)}\n")

    for chunk in chunks:
        print(f"--- Chunk {chunk.id} ---")
        print(f"Entity: {chunk.metadata.entity}")
        print(f"Patch: {chunk.metadata.patch_version}")
        print(f"Type: {chunk.metadata.change_type}")
        print(f"Tokens (aprox): {chunk.token_count}")
        print(f"Content preview: {chunk.content[:80]}...")
        print()

    # Exporta a JSON per verificació
    import json
    output = [c.to_dict() for c in chunks]
    with open("ai-python/data/raw/chunks_preview.json", "w") as f:
        json.dump(output, f, indent=2, ensure_ascii=False)
    print("Chunks exportats a ai-python/data/raw/chunks_preview.json")
```

### 3. Chunking de Text Lliure amb Overlap (20 min)

No sempre tindrem dades ja estructurades. Escriu un chunker genèric per a text pla:

```python
# ai-python/src/knowledge/text_chunker.py
"""
Chunker genèric per a text pla.
Quan les dades NO estan pre-estructurades (ex: un article wiki copiat),
necessitem dividir el text en chunks de mida controlada amb overlap.
"""

from dataclasses import dataclass


@dataclass
class TextChunk:
    """Un fragment de text amb la seva posició dins del document original."""
    content: str
    start_char: int      # Posició inicial en el document original
    end_char: int        # Posició final
    chunk_index: int     # Número de chunk (0, 1, 2...)
    token_count: int     # Tokens aproximats


def chunk_text(
    text: str,
    chunk_size: int = 500,
    overlap: int = 50,
) -> list[TextChunk]:
    """
    Divideix text pla en chunks amb overlap.
    
    Args:
        text: El text a dividir
        chunk_size: Mida objectiu en tokens (1 token ≈ 4 chars)
        overlap: Tokens de superposició entre chunks consecutius
    
    Returns:
        Llista de TextChunk amb el text dividit
    
    Estratègia:
    1. Convertim tokens a caràcters (aproximació)
    2. Busquem el punt de tall més proper a un final de frase
    3. Apliquem overlap per no perdre context a les fronteres
    """
    # Convertim tokens a caràcters (aproximació)
    char_size = chunk_size * 4
    char_overlap = overlap * 4
    
    chunks = []
    start = 0
    index = 0

    while start < len(text):
        # Calculem el final del chunk
        end = start + char_size

        if end < len(text):
            # Busquem el punt de tall natural més proper
            # (final de frase o paràgraf)
            # Prioritzem: \n\n > \n > . > espai
            best_cut = end
            for separator in ["\n\n", "\n", ". ", " "]:
                # Busquem el separador més proper al punt de tall
                sep_pos = text.rfind(separator, start + char_size // 2, end + 100)
                if sep_pos != -1:
                    best_cut = sep_pos + len(separator)
                    break
            end = best_cut

        chunk_text_content = text[start:end].strip()

        if chunk_text_content:  # No afegim chunks buits
            chunks.append(TextChunk(
                content=chunk_text_content,
                start_char=start,
                end_char=end,
                chunk_index=index,
                token_count=len(chunk_text_content) // 4,
            ))
            index += 1

        # Avancem amb overlap
        start = end - char_overlap
        if start >= len(text) or end >= len(text):
            break

    return chunks


# Exemple d'ús amb text wiki
WIKI_EXAMPLE = """
Jinx, the Loose Cannon, is a marksman champion in League of Legends. She was 
released on October 10, 2013. Jinx is known for her chaotic playstyle and 
high attack speed scaling.

Jinx's kit revolves around switching between two weapons: Pow-Pow, a minigun 
that gains attack speed with each hit, and Fishbones, a rocket launcher that 
deals area damage at the cost of mana. This weapon-switching mechanic is her 
passive ability, and it defines her laning phase and teamfight patterns.

In competitive play, Jinx has seen periodic success, particularly in metas 
that favor hypercarry ADCs. Her global ultimate, Super Mega Death Rocket!, 
provides cross-map threat and can secure kills or objectives from anywhere 
on the map. The rocket deals damage based on the target's missing health, 
making it more lethal against low-health targets.

Throughout 2024, Jinx received several balance adjustments. In patch 14.10, 
her base stats were slightly buffed to improve her early laning phase. 
However, by patch 14.15, Riot Games decided to nerf her base AD from 59 to 
57 and reduce Fishbones bonus range as part of mid-season bot lane diversity 
adjustments.
"""

if __name__ == "__main__":
    # Prova el chunker amb mides petites per veure l'overlap
    chunks = chunk_text(WIKI_EXAMPLE, chunk_size=100, overlap=20)

    for chunk in chunks:
        print(f"=== Chunk {chunk.chunk_index} ({chunk.token_count} tokens) ===")
        print(f"Chars {chunk.start_char}-{chunk.end_char}")
        print(chunk.content)
        print()
    
    print(f"Total chunks: {len(chunks)}")
    print(f"Overlap visible: observa com el final d'un chunk ")
    print(f"apareix al principi del següent.")
```

### 4. Verificació de Qualitat dels Chunks (10 min)

```python
# ai-python/src/knowledge/chunk_validator.py
"""
Validador de chunks: comprova que els chunks 
generats compleixen els criteris de qualitat.
"""

from chunk_model import Chunk


def validate_chunks(chunks: list[Chunk]) -> dict:
    """
    Valida una llista de chunks i retorna un resum.
    
    Criteris:
    - Mida: entre 50 i 800 tokens (ni massa petit ni massa gran)
    - Metadades: tots els camps obligatoris presents
    - IDs únics: no hi ha duplicats
    - Contingut: no hi ha chunks buits
    """
    issues = []
    ids_seen = set()

    for i, chunk in enumerate(chunks):
        # Verificar mida
        if chunk.token_count < 50:
            issues.append(f"Chunk {i}: massa petit ({chunk.token_count} tokens)")
        if chunk.token_count > 800:
            issues.append(f"Chunk {i}: massa gran ({chunk.token_count} tokens)")

        # Verificar contingut no buit
        if not chunk.content.strip():
            issues.append(f"Chunk {i}: contingut buit")

        # Verificar metadades
        if not chunk.metadata.entity:
            issues.append(f"Chunk {i}: falta entity a les metadades")
        if not chunk.metadata.source:
            issues.append(f"Chunk {i}: falta source a les metadades")

        # Verificar unicitat d'IDs
        if chunk.id in ids_seen:
            issues.append(f"Chunk {i}: ID duplicat ({chunk.id})")
        ids_seen.add(chunk.id)

    return {
        "total_chunks": len(chunks),
        "total_tokens": sum(c.token_count for c in chunks),
        "avg_tokens": sum(c.token_count for c in chunks) // max(len(chunks), 1),
        "issues": issues,
        "valid": len(issues) == 0,
    }


if __name__ == "__main__":
    # Importem el chunker del pas anterior i validem
    from chunker import chunk_all_patches
    from sample_data import SAMPLE_PATCHES

    chunks = chunk_all_patches(SAMPLE_PATCHES)
    report = validate_chunks(chunks)

    print("=== Informe de Validació ===")
    print(f"Total chunks: {report['total_chunks']}")
    print(f"Total tokens: {report['total_tokens']}")
    print(f"Mitjana tokens/chunk: {report['avg_tokens']}")
    print(f"Vàlid: {'✅' if report['valid'] else '❌'}")

    if report["issues"]:
        print("\nProblemes trobats:")
        for issue in report["issues"]:
            print(f"  ⚠️ {issue}")
    else:
        print("\nTots els chunks passen la validació!")
```

---

## Checklist de Lliurament

- [ ] Fitxer `chunk_model.py` amb les dataclasses `Chunk` i `ChunkMetadata`
- [ ] Fitxer `chunker.py` que converteix patch notes estructurades en chunks
- [ ] Fitxer `text_chunker.py` que divideix text pla amb overlap
- [ ] Fitxer `chunk_validator.py` que valida la qualitat dels chunks
- [ ] Executa `python chunker.py` i verifica que genera chunks correctes
- [ ] Fitxer `chunks_preview.json` generat a `ai-python/data/raw/`
- [ ] Pots explicar per què l'overlap és important i quina mida de chunk és adequada
