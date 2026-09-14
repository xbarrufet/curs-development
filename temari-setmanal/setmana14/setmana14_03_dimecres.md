# Setmana 14 — Dimecres: Vector Embeddings i Qdrant: Indexar la Wiki

## Objectiu del Dia

Entendre què són els embeddings, per què serveixen per a cerca semàntica, i usar-los per indexar els chunks de dimarts a Qdrant. Al final del dia podràs fer cerques semàntiques sobre les teves dades de domini.

---

## Teoria

### Què Són els Embeddings?

Un embedding és una representació numèrica (vector) d'un text. Textos amb significat similar tenen vectors propers en l'espai:

```
Text                          →  Vector (simplificat a 3D)
─────────────────────────────────────────────────────────
"Jinx nerf base AD"           →  [0.82, 0.15, 0.91]
"Jinx base attack reduced"   →  [0.80, 0.14, 0.89]  ← MOLT SIMILAR
"Ahri charm duration buff"   →  [0.31, 0.72, 0.45]  ← DIFERENT
"restaurant menu prices"     →  [0.05, 0.95, 0.12]  ← TOTALMENT DIFERENT

# En realitat, els vectors tenen 1536 dimensions (OpenAI)
# o 1024 dimensions (altres models). Nosaltres no els veiem
# directament — els usa Qdrant per calcular similituds.
```

### Per Què Funciona la Cerca Semàntica?

La cerca tradicional (keyword search) busca paraules exactes. La cerca semàntica busca **significat**:

```python
# CERCA PER PARAULES CLAU (keyword search)
query = "ADC nerfs"
# Busca documents que continguin "ADC" i "nerfs" literalment
# NO trobaria: "marksman base stats reduced" 
# (mateix significat, paraules diferents)

# CERCA SEMÀNTICA (amb embeddings)
query = "ADC nerfs"
# Converteix la query a vector → busca vectors similars
# SÍ trobaria: "marksman base stats reduced"
# perquè el SIGNIFICAT és similar, encara que les paraules siguin diferents
```

```
┌──────────────────────────────────────────────────────┐
│              Com funciona la cerca semàntica          │
│                                                       │
│  1. Indexació (un cop):                               │
│     Document → Chunks → Embeddings → Qdrant          │
│                                                       │
│  2. Cerca (cada query):                               │
│     Query → Embedding → Qdrant busca vectors propers  │
│     → Retorna els chunks més similars                 │
│                                                       │
│  3. Resposta:                                         │
│     Chunks rellevants → LLM genera resposta           │
│     amb informació REAL, no inventada                 │
└──────────────────────────────────────────────────────┘
```

### Qdrant: La Base de Dades Vectorial

Qdrant és una base de dades especialitzada en vectors. Ja el tens al `docker-compose.yml` des de la Setmana 8:

```yaml
# Fragment del docker-compose.yml (ja existent)
services:
  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"    # API REST
      - "6334:6334"    # API gRPC
    volumes:
      - qdrant_data:/qdrant/storage
```

Qdrant fa tres coses principals:
1. **Emmagatzema** vectors amb les seves metadades (payload)
2. **Busca** els vectors més propers a un vector de query
3. **Filtra** per metadades abans o durant la cerca

```python
# Conceptualment, Qdrant fa això:
# (simplificació — internament usa algorismes HNSW molt més eficients)

def cerca_similar(query_vector, tots_els_vectors, top_k=5):
    """Busca els K vectors més propers al query."""
    distancies = []
    for vector_id, vector in tots_els_vectors.items():
        # Calcula la distància cosinus entre vectors
        dist = cosine_distance(query_vector, vector)
        distancies.append((vector_id, dist))
    
    # Ordena per distància (menor = més similar)
    distancies.sort(key=lambda x: x[1])
    return distancies[:top_k]
```

### Models d'Embeddings

Necessitem un model per convertir text a vectors. Opcions habituals:

| Model | Dimensions | Preu | Qualitat |
|-------|-----------|------|----------|
| OpenAI `text-embedding-3-small` | 1536 | $0.02/1M tokens | Bona |
| OpenAI `text-embedding-3-large` | 3072 | $0.13/1M tokens | Molt bona |
| `sentence-transformers` (local) | 384-768 | Gratis | Acceptable |
| Voyage AI | 1024 | $0.02/1M tokens | Molt bona |

Per al curs usarem `sentence-transformers` perquè és gratis i funciona en local:

```bash
# Instal·la la dependència
pip install sentence-transformers qdrant-client
```

---

## Activitat

### 1. Arrenca Qdrant (5 min)

```bash
# Assegura't que Qdrant està corrent
docker compose up -d qdrant

# Verifica que respon
curl http://localhost:6333/healthz
# Hauria de retornar: {"title":"qdrant","version":"..."}

# Obre el dashboard (opcional, però útil per visualitzar)
# http://localhost:6333/dashboard
```

### 2. Script d'Embeddings (25 min)

```python
# ai-python/src/knowledge/embeddings.py
"""
Generació d'embeddings amb sentence-transformers.
Converteix text a vectors per a cerca semàntica.

Usem un model local (all-MiniLM-L6-v2) per evitar
dependències d'APIs externes i costos.
El model és petit (80MB) però suficient per al nostre cas.
"""

from sentence_transformers import SentenceTransformer

# Carreguem el model un cop — reutilitzem per a múltiples textos
# all-MiniLM-L6-v2: model lleuger, 384 dimensions, bon per a anglès
# Per a català/castellà, podries usar "paraphrase-multilingual-MiniLM-L12-v2"
MODEL_NAME = "all-MiniLM-L6-v2"
VECTOR_SIZE = 384  # Dimensions del vector resultant


def load_model() -> SentenceTransformer:
    """
    Carrega el model d'embeddings.
    La primera vegada descarrega el model (~80MB).
    Les següents vegades usa la còpia local.
    """
    print(f"Carregant model {MODEL_NAME}...")
    model = SentenceTransformer(MODEL_NAME)
    print("Model carregat.")
    return model


def generate_embeddings(
    texts: list[str],
    model: SentenceTransformer | None = None,
) -> list[list[float]]:
    """
    Genera embeddings per a una llista de textos.
    
    Args:
        texts: Llista de textos a convertir
        model: Model pre-carregat (si None, en carrega un de nou)
    
    Returns:
        Llista de vectors (cada vector és una llista de floats)
    """
    if model is None:
        model = load_model()

    # encode() converteix textos a vectors numpy
    # convert_to_list=True els retorna com a llistes Python
    # (necessari per serialitzar a JSON o enviar a Qdrant)
    embeddings = model.encode(
        texts,
        show_progress_bar=True,    # Mostra progrés si hi ha molts textos
        normalize_embeddings=True,  # Normalitza per a cosine similarity
    )

    return embeddings.tolist()


def cosine_similarity(vec_a: list[float], vec_b: list[float]) -> float:
    """
    Calcula la similitud cosinus entre dos vectors.
    Retorna un valor entre -1 (oposats) i 1 (idèntics).
    Útil per entendre i depurar els resultats.
    """
    dot_product = sum(a * b for a, b in zip(vec_a, vec_b))
    norm_a = sum(a * a for a in vec_a) ** 0.5
    norm_b = sum(b * b for b in vec_b) ** 0.5
    if norm_a == 0 or norm_b == 0:
        return 0.0
    return dot_product / (norm_a * norm_b)


if __name__ == "__main__":
    # Demostració: textos similars tenen vectors propers
    model = load_model()

    textos_prova = [
        "Jinx base AD reduced from 59 to 57",
        "Jinx attack damage nerfed",           # Significat similar
        "Ahri charm duration increased",        # Tema diferent
        "Best pizza restaurants in Barcelona",   # Totalment diferent
    ]

    embeddings = generate_embeddings(textos_prova, model)

    print(f"\nDimensions dels vectors: {len(embeddings[0])}")
    print(f"\nSimilituds amb '{textos_prova[0]}':")

    for i in range(1, len(textos_prova)):
        sim = cosine_similarity(embeddings[0], embeddings[i])
        print(f"  vs '{textos_prova[i]}': {sim:.4f}")
    
    # Esperat:
    # "Jinx attack damage nerfed" → ~0.7-0.9 (molt similar)
    # "Ahri charm duration increased" → ~0.3-0.5 (relacionat)
    # "Best pizza restaurants" → ~0.0-0.1 (no relacionat)
```

### 3. Indexar a Qdrant (25 min)

```python
# ai-python/src/knowledge/indexer.py
"""
Indexador: pren chunks amb embeddings i els puja a Qdrant.
Crea la col·lecció si no existeix, i upserta els punts
(insert or update) per garantir idempotència.
"""

from qdrant_client import QdrantClient
from qdrant_client.models import (
    Distance,
    PointStruct,
    VectorParams,
)

from chunk_model import Chunk
from chunker import chunk_all_patches
from embeddings import load_model, generate_embeddings, VECTOR_SIZE
from sample_data import SAMPLE_PATCHES

# Configuració de Qdrant
QDRANT_HOST = "localhost"
QDRANT_PORT = 6333
COLLECTION_NAME = "esportspulse_wiki"


def get_client() -> QdrantClient:
    """Crea una connexió al servidor Qdrant."""
    return QdrantClient(host=QDRANT_HOST, port=QDRANT_PORT)


def create_collection(client: QdrantClient) -> None:
    """
    Crea la col·lecció a Qdrant si no existeix.
    
    Una col·lecció és com una "taula" en SQL,
    però emmagatzema vectors en lloc de files.
    """
    # Comprovem si ja existeix (idempotència)
    collections = client.get_collections().collections
    existing_names = [c.name for c in collections]

    if COLLECTION_NAME in existing_names:
        print(f"Col·lecció '{COLLECTION_NAME}' ja existeix. Reutilitzant.")
        return

    # Creem la col·lecció amb la configuració adequada
    client.create_collection(
        collection_name=COLLECTION_NAME,
        vectors_config=VectorParams(
            size=VECTOR_SIZE,        # 384 dimensions (all-MiniLM-L6-v2)
            distance=Distance.COSINE  # Usem similitud cosinus
        ),
    )
    print(f"Col·lecció '{COLLECTION_NAME}' creada.")


def index_chunks(
    client: QdrantClient,
    chunks: list[Chunk],
) -> None:
    """
    Indexa una llista de chunks a Qdrant.
    
    Procés:
    1. Genera embeddings per al contingut de cada chunk
    2. Crea punts (PointStruct) amb vector + payload (metadades)
    3. Upserta a Qdrant (insert o update si l'ID ja existeix)
    """
    if not chunks:
        print("No hi ha chunks per indexar.")
        return

    # 1. Genera embeddings
    print(f"Generant embeddings per a {len(chunks)} chunks...")
    model = load_model()
    texts = [chunk.content for chunk in chunks]
    embeddings = generate_embeddings(texts, model)

    # 2. Crea punts per a Qdrant
    points = []
    for i, (chunk, embedding) in enumerate(zip(chunks, embeddings)):
        point = PointStruct(
            # Qdrant necessita IDs numèrics o UUIDs
            # Usem l'índex com a ID simple (per a producció, usa UUIDs)
            id=i,
            vector=embedding,
            payload={
                # El contingut del chunk (el text que es retornarà)
                "content": chunk.content,
                # Les metadades (permeten filtrar)
                **chunk.metadata.to_dict(),
                # L'ID del chunk (per referència)
                "chunk_id": chunk.id,
            },
        )
        points.append(point)

    # 3. Upserta a Qdrant (en batches per eficiència)
    BATCH_SIZE = 100
    for i in range(0, len(points), BATCH_SIZE):
        batch = points[i:i + BATCH_SIZE]
        client.upsert(
            collection_name=COLLECTION_NAME,
            points=batch,
        )
        print(f"  Indexats {min(i + BATCH_SIZE, len(points))}/{len(points)} punts")

    print(f"Indexació completada: {len(points)} punts a '{COLLECTION_NAME}'")


def search(
    client: QdrantClient,
    query: str,
    top_k: int = 3,
) -> list[dict]:
    """
    Cerca semàntica: troba els chunks més similars a la query.
    
    Args:
        query: Text de la pregunta
        top_k: Quants resultats retornar
    
    Returns:
        Llista de resultats amb score, contingut i metadades
    """
    # Genera l'embedding de la query
    model = load_model()
    query_embedding = generate_embeddings([query], model)[0]

    # Busca a Qdrant
    results = client.query_points(
        collection_name=COLLECTION_NAME,
        query=query_embedding,
        limit=top_k,
    )

    # Formata els resultats
    formatted = []
    for hit in results.points:
        formatted.append({
            "score": hit.score,
            "content": hit.payload["content"],
            "entity": hit.payload["entity"],
            "patch": hit.payload["patch_version"],
            "change_type": hit.payload["change_type"],
        })

    return formatted


if __name__ == "__main__":
    # === Pipeline completa: chunk → embed → index → search ===

    # 1. Genera chunks
    print("=== 1. Generant chunks ===")
    chunks = chunk_all_patches(SAMPLE_PATCHES)
    print(f"Chunks generats: {len(chunks)}")

    # 2. Connecta a Qdrant i crea col·lecció
    print("\n=== 2. Connectant a Qdrant ===")
    client = get_client()
    create_collection(client)

    # 3. Indexa
    print("\n=== 3. Indexant chunks ===")
    index_chunks(client, chunks)

    # 4. Cerca
    print("\n=== 4. Provant cerques ===")
    queries = [
        "What nerfs did Jinx receive?",
        "Which champions got buffed?",
        "Stormsurge item changes",
    ]

    for query in queries:
        print(f"\nQuery: '{query}'")
        results = search(client, query, top_k=2)
        for i, r in enumerate(results):
            print(f"  #{i+1} (score: {r['score']:.4f}) [{r['entity']} - {r['patch']}]")
            # Mostra les primeres línies del contingut
            preview = r["content"].split("\n")[0]
            print(f"       {preview}")
```

### 4. Cerca amb Filtre de Metadades (10 min)

```python
# ai-python/src/knowledge/filtered_search.py
"""
Cerca amb filtres: combina cerca semàntica amb filtres de metadades.
Exemple: "nerfs" però NOMÉS del patch 14.15.
"""

from qdrant_client import QdrantClient
from qdrant_client.models import FieldCondition, Filter, MatchValue

from embeddings import generate_embeddings, load_model
from indexer import COLLECTION_NAME, QDRANT_HOST, QDRANT_PORT


def search_with_filter(
    query: str,
    patch_version: str | None = None,
    change_type: str | None = None,
    top_k: int = 3,
) -> list[dict]:
    """
    Cerca semàntica amb filtres opcionals.
    
    Els filtres s'apliquen ABANS de la cerca vectorial,
    reduint l'espai de cerca i millorant la precisió.
    """
    client = QdrantClient(host=QDRANT_HOST, port=QDRANT_PORT)
    model = load_model()
    query_embedding = generate_embeddings([query], model)[0]

    # Construïm el filtre dinàmicament
    conditions = []
    if patch_version:
        conditions.append(
            FieldCondition(
                key="patch_version",
                match=MatchValue(value=patch_version),
            )
        )
    if change_type:
        conditions.append(
            FieldCondition(
                key="change_type",
                match=MatchValue(value=change_type),
            )
        )

    # Si hi ha condicions, creem el filtre; si no, None
    query_filter = Filter(must=conditions) if conditions else None

    results = client.query_points(
        collection_name=COLLECTION_NAME,
        query=query_embedding,
        query_filter=query_filter,
        limit=top_k,
    )

    return [
        {
            "score": hit.score,
            "content": hit.payload["content"],
            "entity": hit.payload["entity"],
            "patch": hit.payload["patch_version"],
        }
        for hit in results.points
    ]


if __name__ == "__main__":
    print("=== Cerca sense filtre ===")
    results = search_with_filter("champion changes")
    for r in results:
        print(f"  {r['entity']} (patch {r['patch']}) - score: {r['score']:.4f}")

    print("\n=== Cerca filtrada per patch 14.15 ===")
    results = search_with_filter("champion changes", patch_version="14.15")
    for r in results:
        print(f"  {r['entity']} (patch {r['patch']}) - score: {r['score']:.4f}")

    print("\n=== Cerca filtrada per nerfs ===")
    results = search_with_filter("what was nerfed", change_type="nerf")
    for r in results:
        print(f"  {r['entity']} (patch {r['patch']}) - score: {r['score']:.4f}")
```

---

## Checklist de Lliurament

- [ ] Qdrant arrencant correctament amb `docker compose up -d qdrant`
- [ ] Fitxer `embeddings.py` que genera vectors i demostra similituds semàntiques
- [ ] Fitxer `indexer.py` que indexa chunks a Qdrant i fa cerques
- [ ] Fitxer `filtered_search.py` que combina cerca semàntica amb filtres
- [ ] Executar `python indexer.py` i veure resultats de cerca coherents
- [ ] Verificar al dashboard de Qdrant (`http://localhost:6333/dashboard`) que la col·lecció té punts
- [ ] Pots explicar la diferència entre keyword search i semantic search
