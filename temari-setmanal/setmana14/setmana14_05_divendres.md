# Setmana 14 — Divendres: Configurar @docs i Consolidar la Wiki

## Objectiu del Dia

Configurar Cursor @docs per millorar la qualitat de les respostes de la IA, crear un script d'ingestió reproduïble i idempotent, i consolidar tota la feina de la setmana en un PR. Al final del dia tindràs una pipeline de Knowledge Engineering funcional de principi a fi.

---

## Teoria

### Cursor @docs: Documentació com a Context

Cursor permet apuntar la IA directament a documentació externa amb `@docs`. Quan escrius `@docs Spring Boot`, Cursor:

1. **Indexa** la documentació de Spring Boot (la primera vegada)
2. **Busca** la informació rellevant per la teva pregunta
3. **Inclou** fragments de documentació al context del LLM

```
Sense @docs:
┌────────────┐     ┌─────────┐     ┌──────────────────────┐
│ "Com faig   │────►│  LLM    │────►│ Resposta genèrica,   │
│  pagination │     │         │     │ pot ser incorrecta    │
│  amb Spring │     │         │     │ per a Spring Boot 3   │
│  Boot 3?"   │     └─────────┘     └──────────────────────┘

Amb @docs:
┌────────────┐     ┌──────────────┐     ┌─────────┐     ┌──────────────────┐
│ "Com faig   │────►│ @docs busca  │────►│  LLM +  │────►│ Resposta precisa │
│  pagination │     │ Spring Boot  │     │ context │     │ amb la sintaxi   │
│  amb Spring │     │ docs reals   │     │ real    │     │ actual de SB3    │
│  Boot 3?"   │     └──────────────┘     └─────────┘     └──────────────────┘
```

### Idempotència: Per Què el Script Ha de Ser Reproduïble

Un script d'ingestió idempotent és aquell que pots executar **múltiples vegades** amb el mateix resultat. Per què importa?

```python
# MAL — No idempotent: cada execució duplica dades
def ingest_bad():
    chunks = generate_chunks()
    for chunk in chunks:
        qdrant.insert(chunk)  # Si l'executes 3 cops, tens 3x els chunks!

# BÉ — Idempotent: sempre el mateix resultat
def ingest_good():
    chunks = generate_chunks()
    # Esborra i recrea la col·lecció (o upsert amb IDs determinístics)
    qdrant.recreate_collection("esportspulse_wiki")
    for chunk in chunks:
        qdrant.upsert(id=chunk.id, data=chunk)  # ID basat en contingut
    # Resultat: sempre exactament N chunks, sense duplicats
```

### El Patró ETL (Extract, Transform, Load)

La pipeline de Knowledge Engineering segueix el patró ETL clàssic:

```
┌───────────┐     ┌─────────────────┐     ┌────────────────┐
│  EXTRACT  │     │    TRANSFORM    │     │     LOAD       │
│           │     │                 │     │                │
│ - Patch   │────►│ - Chunking      │────►│ - Embeddings   │
│   notes   │     │ - Metadades     │     │ - Qdrant index │
│ - Wiki    │     │ - Validació     │     │ - Verificació  │
│ - APIs    │     │ - Neteja        │     │                │
└───────────┘     └─────────────────┘     └────────────────┘

# Cada pas és independent i testejable
# Pots canviar la font (Extract) sense tocar Transform ni Load
# Pots canviar Qdrant per Pinecone sense tocar Extract ni Transform
```

---

## Activitat

### 1. Configurar @docs a Cursor (15 min)

Obre Cursor i configura documentació externa:

```
# A Cursor:
# 1. Obre Settings (Cmd+,)
# 2. Cerca "docs" o "context"
# 3. Afegeix documentació:

# Documentació recomanada per al projecte:
# - Spring Boot 3: https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/
# - Spring Data JPA: https://docs.spring.io/spring-data/jpa/docs/current/reference/html/
# - FastAPI: https://fastapi.tiangolo.com/
# - Pydantic: https://docs.pydantic.dev/latest/
# - Qdrant: https://qdrant.tech/documentation/
# - sentence-transformers: https://www.sbert.net/
```

Prova la diferència:

```
# Pregunta SENSE @docs:
"How do I configure pagination in Spring Boot 3?"
→ Pot donar codi de Spring Boot 2 (deprecated)

# Pregunta AMB @docs Spring Boot:
"@docs How do I configure pagination in Spring Boot 3?"
→ Dona codi actualitzat amb Pageable i @PageableDefault
```

Documenta la diferència que observes en 3-4 línies.

### 2. Script d'Ingestió Consolidat (30 min)

Crea un script únic que executa tota la pipeline ETL:

```python
# ai-python/src/knowledge/ingest.py
"""
Script d'ingestió principal.
Executa tota la pipeline: Extract → Transform → Load.

Característiques:
- Idempotent: pots executar-lo múltiples vegades sense duplicar dades
- Configurable: paràmetres per línia de comandes
- Verificable: mostra estadístiques i fa una cerca de prova al final

Ús:
    python ingest.py                    # Ingestió completa
    python ingest.py --verify-only      # Només verifica l'estat actual
    python ingest.py --reset            # Esborra i reindexa tot
"""

import argparse
import json
import sys
from datetime import datetime
from pathlib import Path

from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams

from chunk_model import Chunk
from chunker import chunk_all_patches
from chunk_validator import validate_chunks
from embeddings import load_model, generate_embeddings, VECTOR_SIZE
from sample_data import SAMPLE_PATCHES

# Configuració centralitzada
QDRANT_HOST = "localhost"
QDRANT_PORT = 6333
COLLECTION_NAME = "esportspulse_wiki"


def extract() -> list[dict]:
    """
    EXTRACT: Obté les dades en brut.
    
    Ara usa dades d'exemple. En producció, aquí
    cridaries una API, llegiríes fitxers, o rasparies una web.
    """
    print("[EXTRACT] Carregant dades...")
    data = SAMPLE_PATCHES
    print(f"[EXTRACT] {len(data)} patches carregats.")
    return data


def transform(raw_data: list[dict]) -> list[Chunk]:
    """
    TRANSFORM: Converteix dades en brut a chunks validats.
    """
    print("[TRANSFORM] Generant chunks...")
    chunks = chunk_all_patches(raw_data)
    print(f"[TRANSFORM] {len(chunks)} chunks generats.")

    # Validació
    print("[TRANSFORM] Validant chunks...")
    report = validate_chunks(chunks)
    if not report["valid"]:
        print(f"[TRANSFORM] ⚠️ Problemes trobats:")
        for issue in report["issues"]:
            print(f"  - {issue}")
    else:
        print(f"[TRANSFORM] ✅ Tots els chunks vàlids.")
    
    print(f"[TRANSFORM] Mitjana: {report['avg_tokens']} tokens/chunk")

    return chunks


def load(chunks: list[Chunk], reset: bool = False) -> None:
    """
    LOAD: Genera embeddings i indexa a Qdrant.
    
    Args:
        chunks: Chunks validats
        reset: Si True, esborra la col·lecció i la recrea
    """
    client = QdrantClient(host=QDRANT_HOST, port=QDRANT_PORT)

    # Gestió de la col·lecció
    collections = [c.name for c in client.get_collections().collections]

    if reset and COLLECTION_NAME in collections:
        print(f"[LOAD] Esborrant col·lecció '{COLLECTION_NAME}'...")
        client.delete_collection(COLLECTION_NAME)
        collections.remove(COLLECTION_NAME)

    if COLLECTION_NAME not in collections:
        print(f"[LOAD] Creant col·lecció '{COLLECTION_NAME}'...")
        client.create_collection(
            collection_name=COLLECTION_NAME,
            vectors_config=VectorParams(
                size=VECTOR_SIZE,
                distance=Distance.COSINE,
            ),
        )

    # Genera embeddings
    print(f"[LOAD] Generant embeddings per a {len(chunks)} chunks...")
    model = load_model()
    texts = [chunk.content for chunk in chunks]
    embeddings = generate_embeddings(texts, model)

    # Indexa a Qdrant amb upsert (idempotent)
    from qdrant_client.models import PointStruct

    points = [
        PointStruct(
            id=i,
            vector=embedding,
            payload={
                "content": chunk.content,
                **chunk.metadata.to_dict(),
                "chunk_id": chunk.id,
                "ingested_at": datetime.now().isoformat(),
            },
        )
        for i, (chunk, embedding) in enumerate(zip(chunks, embeddings))
    ]

    # Upsert en batches
    BATCH_SIZE = 100
    for i in range(0, len(points), BATCH_SIZE):
        batch = points[i:i + BATCH_SIZE]
        client.upsert(collection_name=COLLECTION_NAME, points=batch)

    print(f"[LOAD] ✅ {len(points)} punts indexats a '{COLLECTION_NAME}'.")


def verify() -> None:
    """
    Verifica l'estat de la col·lecció i fa cerques de prova.
    """
    client = QdrantClient(host=QDRANT_HOST, port=QDRANT_PORT)

    # Info de la col·lecció
    try:
        info = client.get_collection(COLLECTION_NAME)
        print(f"\n[VERIFY] Col·lecció: {COLLECTION_NAME}")
        print(f"[VERIFY] Punts totals: {info.points_count}")
        print(f"[VERIFY] Vectors: {info.config.params.vectors.size}D")
    except Exception as e:
        print(f"[VERIFY] ❌ Col·lecció no trobada: {e}")
        return

    # Cerques de prova
    print(f"\n[VERIFY] Provant cerques...")
    model = load_model()

    test_queries = [
        ("Jinx nerfs", "Jinx"),
        ("champion buffs", "Ahri"),  # Esperem Ahri com a primer resultat
        ("item changes", "Stormsurge"),
    ]

    all_pass = True
    for query, expected_entity in test_queries:
        query_embedding = generate_embeddings([query], model)[0]
        results = client.query_points(
            collection_name=COLLECTION_NAME,
            query=query_embedding,
            limit=1,
        )

        if results.points:
            top_result = results.points[0]
            entity = top_result.payload.get("entity", "?")
            score = top_result.score
            status = "✅" if entity == expected_entity else "⚠️"
            if entity != expected_entity:
                all_pass = False
            print(f"  {status} '{query}' → {entity} (score: {score:.4f})")
        else:
            print(f"  ❌ '{query}' → cap resultat")
            all_pass = False

    if all_pass:
        print(f"\n[VERIFY] ✅ Totes les cerques retornen resultats esperats.")
    else:
        print(f"\n[VERIFY] ⚠️ Algunes cerques no han retornat l'esperat.")
        print(f"  (Això pot ser normal — la cerca semàntica no és exacta)")


def main():
    parser = argparse.ArgumentParser(
        description="EsportsPulse Knowledge Ingestion Pipeline"
    )
    parser.add_argument(
        "--verify-only",
        action="store_true",
        help="Només verifica l'estat actual sense reingestar",
    )
    parser.add_argument(
        "--reset",
        action="store_true",
        help="Esborra la col·lecció i reindexa tot",
    )
    args = parser.parse_args()

    print("=" * 60)
    print("EsportsPulse — Knowledge Ingestion Pipeline")
    print(f"Data: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    print("=" * 60)

    if args.verify_only:
        verify()
        return

    # Pipeline ETL completa
    raw_data = extract()
    chunks = transform(raw_data)
    load(chunks, reset=args.reset)
    verify()

    # Guarda un log de la ingestió
    log_path = Path("ai-python/data/ingestion_log.json")
    log_path.parent.mkdir(parents=True, exist_ok=True)
    log_entry = {
        "timestamp": datetime.now().isoformat(),
        "patches_processed": len(raw_data),
        "chunks_generated": len(chunks),
        "collection": COLLECTION_NAME,
        "reset": args.reset,
    }
    log_path.write_text(json.dumps(log_entry, indent=2))
    print(f"\n[LOG] Ingestió registrada a {log_path}")


if __name__ == "__main__":
    main()
```

### 3. Crear la Wiki del Projecte (15 min)

Organitza la documentació del coneixement de domini:

```markdown
<!-- ai-python/data/wiki/README.md -->
# EsportsPulse — Knowledge Base

## Estructura

```
data/
├── raw/                    # Dades en brut (patch notes, dumps)
│   └── chunks_preview.json # Preview dels chunks generats
├── wiki/                   # Documentació de domini curada
│   ├── README.md           # Aquest fitxer
│   ├── patch_notes/        # Patch notes per versió
│   └── champions/          # Info per champion
└── ingestion_log.json      # Log de la última ingestió
```

## Com Afegir Dades

1. Col·loca les dades en brut a `raw/`
2. Si cal, crea un parser específic a `src/knowledge/`
3. Actualitza `sample_data.py` amb les noves dades (o crea un nou loader)
4. Executa `python src/knowledge/ingest.py --reset`
5. Verifica amb `python src/knowledge/ingest.py --verify-only`

## Fonts de Dades
- Patch notes: [font]
- Estadístiques: [font]
- Wiki: [font]
```

```bash
# Crea l'estructura de directoris
mkdir -p ai-python/data/wiki/patch_notes
mkdir -p ai-python/data/wiki/champions
```

### 4. Preparar i Crear el PR (20 min)

Consolida tota la feina de la setmana:

```bash
# Verifica l'estat del repositori
git status

# Crea una branca per a la setmana 14
git checkout -b feature/week14-knowledge

# Afegeix tots els fitxers de la setmana
git add ai-python/src/knowledge/
git add ai-python/data/
git add .cursorrules
git add CLAUDE.md
# NO afegeixis fitxers de configuració amb secrets
# (.claude/settings.local.json pot tenir connection strings)

# Revisa què estàs a punt de cometre
git diff --staged

# Commit amb conventional commits
git commit -m "feat(knowledge): add knowledge engineering pipeline

- Chunk model with dataclasses and deterministic IDs
- Patch notes chunker with metadata extraction
- Generic text chunker with overlap support
- Chunk validator for quality assurance
- Embedding generation with sentence-transformers
- Qdrant indexer with upsert (idempotent)
- Filtered semantic search
- ETL ingestion script with verify mode
- Project rules (.cursorrules) and CLAUDE.md
- Sample data and wiki structure"

# Puja la branca
git push -u origin feature/week14-knowledge

# Crea el PR
gh pr create \
  --title "feat: knowledge engineering pipeline (S14)" \
  --body "## Summary
- Pipeline ETL per ingestar patch notes a Qdrant
- Chunking amb metadades i validació
- Cerca semàntica amb embeddings (sentence-transformers)
- Script d'ingestió idempotent amb mode verify
- Configuració MCP server PostgreSQL
- .cursorrules i CLAUDE.md del projecte

## Test Plan
- [ ] docker compose up -d (Qdrant + PostgreSQL)
- [ ] python ingest.py --reset (pipeline completa)
- [ ] python ingest.py --verify-only (verifica resultats)
- [ ] python filtered_search.py (cerca amb filtres)

## Files
- ai-python/src/knowledge/*.py — Pipeline completa
- ai-python/data/ — Dades i estructura wiki
- .cursorrules, CLAUDE.md — Configuració del projecte"
```

### 5. Reflexió Setmanal (10 min)

```markdown
<!-- Afegeix al final de hallucination_tests.md o crea un fitxer nou -->

# Reflexió Setmana 14 — Knowledge Engineering

## Què he après?
1. Per què els LLMs al·lucinen i com les dades de domini ho resolen
2. Com dividir documents en chunks amb mida i overlap adequats
3. Com funcionen els embeddings i la cerca semàntica
4. On trobar i com avaluar eines existents (MCP servers, skills)
5. La importància de la idempotència en pipelines de dades

## Què ha funcionat bé?
[Escriu aquí]

## Què ha estat difícil?
[Escriu aquí]

## Com connecta amb el que ve?
- Setmana 15: Usarem aquesta base de coneixement per construir
  un sistema RAG (Retrieval-Augmented Generation) complet
- Les dades que hem indexat avui alimentaran l'agent de la S16
- El patró ETL es repetirà en contextos més complexos
```

---

## Checklist de Lliurament

- [ ] @docs configurat a Cursor amb almenys Spring Boot docs
- [ ] Diferència documentada entre respostes amb i sense @docs
- [ ] Script `ingest.py` funcional amb modes `--reset` i `--verify-only`
- [ ] Estructura de wiki creada a `ai-python/data/wiki/`
- [ ] Branca `feature/week14-knowledge` creada i pujada
- [ ] PR creat amb tots els fitxers de la setmana
- [ ] Reflexió setmanal escrita
- [ ] Pots executar la pipeline sencera: `python ingest.py --reset` sense errors
