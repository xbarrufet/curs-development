# Setmana 15 — Dilluns: Retrieval Semàntic: Buscar per Significat, No per Paraules

## Objectiu del Dia

Construir la funció de retrieval semàntic que busca chunks rellevants a Qdrant i els injecta dins un prompt per a l'LLM. Al final del dia has de poder fer una pregunta en llenguatge natural sobre patch notes i rebre una resposta fonamentada en les dades reals de la teva wiki.

---

## Teoria

### Keyword Search vs Semantic Search

La cerca clàssica (keyword search) funciona per coincidència exacta de paraules. Si busques "champion nerfs in patch 14.5", el sistema busca documents que continguin exactament aquestes paraules. Però les patch notes diuen coses com "Jinx: base attack damage reduced from 59 to 55". No hi ha cap paraula en comú.

**Keyword search:**
```
Query: "champion nerfs in patch 14.5"
Cerca: LIKE '%champion%' AND LIKE '%nerfs%' AND LIKE '%14.5%'
Resultat: 0 documents trobats ❌
```

**Semantic search:**
```
Query: "champion nerfs in patch 14.5"
Embedding: [0.23, -0.41, 0.87, ...]  ← vector que captura el SIGNIFICAT
Cerca: cosine_similarity(query_vector, chunk_vectors)
Resultat: "Jinx: base AD reduced..." (score: 0.91) ✅
```

La diferència clau: el model d'embeddings entén que "nerf" i "reduced" signifiquen el mateix en context de gaming. No compara lletres, compara conceptes.

### El Patró RAG (Retrieval-Augmented Generation)

RAG és el patró més important en aplicacions d'IA empresarial. Resol el problema fonamental dels LLMs: **al·lucinen quan no tenen informació**.

```
┌──────────────────────────────────────────────────┐
│                    PATRÓ RAG                      │
│                                                   │
│  1. RETRIEVE                                      │
│     Pregunta → Embedding → Qdrant → Top-K chunks  │
│                                                   │
│  2. AUGMENT                                       │
│     System prompt + Chunks rellevants + Pregunta   │
│                                                   │
│  3. GENERATE                                      │
│     LLM genera resposta basada en els chunks       │
│                                                   │
│  Resultat: Resposta fonamentada en dades REALS     │
└──────────────────────────────────────────────────┘
```

**Sense RAG:** L'LLM respon amb el que "recorda" del seu entrenament (que pot ser incorrecte o desactualitzat).

**Amb RAG:** L'LLM respon NOMÉS amb la informació que li passes. Si les patch notes diuen que Jinx té 55 AD base, l'LLM dirà exactament això.

### Top-K i Score Threshold

Quan fas una cerca a Qdrant, recuperes els K chunks més similars. Però no tots són útils:

- **Top-K massa alt** (k=20): Inclouràs chunks irrellevants que confondran l'LLM.
- **Top-K massa baix** (k=1): Potser et perds informació rellevant.
- **Score threshold**: Filtra chunks amb similitud per sota d'un llindar (ex: 0.7).

```python
# Exemple de cerca amb threshold
# Només retornem chunks amb score >= 0.7 per evitar soroll
SCORE_THRESHOLD = 0.7
TOP_K = 5
```

Un bon punt de partida: `k=5` amb `score_threshold=0.7`. Ajustaràs aquests valors amb les evals de dimecres.

### Prompt Engineering per RAG

El prompt que envies a l'LLM és crític. Ha d'indicar clarament:
1. Quin és el context (els chunks recuperats)
2. Quina és la pregunta de l'usuari
3. Quines restriccions té (no inventar, citar fonts)

```python
# Template del prompt RAG
RAG_PROMPT_TEMPLATE = """Ets un assistent expert en patch notes de League of Legends.
Respon ÚNICAMENT basant-te en el context proporcionat.

CONTEXT:
{context}

PREGUNTA: {question}

INSTRUCCIONS:
- Respon en català
- Si el context no conté la informació, digues "No tinc informació sobre això"
- Cita el patch d'on treus la informació
"""
```

---

## Activitat

### 1. Crear el mòdul de retrieval (30 min)

Crea el fitxer `ai-python/src/retrieval/rag_engine.py`:

```python
"""
Motor RAG per a consultes semàntiques sobre patch notes.
Connecta Qdrant (retrieval) amb l'LLM (generation).
"""

import hashlib
from qdrant_client import QdrantClient
from qdrant_client.models import Filter
from openai import OpenAI

# Configuració de clients
# Qdrant per a la cerca vectorial, OpenAI per a embeddings i generació
qdrant = QdrantClient(host="localhost", port=6333)
openai_client = OpenAI()

# Constants de configuració del retrieval
COLLECTION_NAME = "patch_notes"       # Col·lecció creada a la setmana 14
EMBEDDING_MODEL = "text-embedding-3-small"  # Model d'embeddings d'OpenAI
LLM_MODEL = "gpt-4o-mini"            # Model de generació (barat i ràpid)
TOP_K = 5                             # Nombre de chunks a recuperar
SCORE_THRESHOLD = 0.7                 # Llindar mínim de similitud

# Template del prompt RAG
# Inclou instruccions clares per evitar al·lucinacions
RAG_SYSTEM_PROMPT = """Ets un assistent expert en patch notes de League of Legends.
Respon ÚNICAMENT basant-te en el context proporcionat a continuació.
Si la informació no es troba al context, digues exactament:
"No tinc informació sobre això en les patch notes indexades."
Cita sempre el número de patch d'on treus la informació."""


def get_embedding(text: str) -> list[float]:
    """
    Genera l'embedding vectorial d'un text utilitzant l'API d'OpenAI.
    
    Args:
        text: El text a convertir en vector
    Returns:
        Llista de floats representant l'embedding
    """
    response = openai_client.embeddings.create(
        model=EMBEDDING_MODEL,
        input=text
    )
    return response.data[0].embedding


def retrieve_chunks(query: str, top_k: int = TOP_K) -> list[dict]:
    """
    Busca els chunks més rellevants a Qdrant per a una query donada.
    
    Procés:
    1. Converteix la query en un vector (embedding)
    2. Busca els top_k vectors més propers a Qdrant
    3. Filtra per score_threshold per eliminar resultats irrellevants
    
    Args:
        query: La pregunta de l'usuari en llenguatge natural
        top_k: Nombre màxim de chunks a retornar
    Returns:
        Llista de diccionaris amb 'text', 'score' i 'metadata' de cada chunk
    """
    # Pas 1: Convertir la pregunta a vector
    query_vector = get_embedding(query)
    
    # Pas 2: Cercar a Qdrant els vectors més similars
    results = qdrant.search(
        collection_name=COLLECTION_NAME,
        query_vector=query_vector,
        limit=top_k,
        score_threshold=SCORE_THRESHOLD
    )
    
    # Pas 3: Formatar els resultats filtrats
    chunks = []
    for result in results:
        chunks.append({
            "text": result.payload.get("text", ""),
            "score": result.score,
            "metadata": {
                "patch": result.payload.get("patch_number", "desconegut"),
                "section": result.payload.get("section", "general"),
                "chunk_id": result.id
            }
        })
    
    return chunks


def format_context(chunks: list[dict]) -> str:
    """
    Formata els chunks recuperats en un text llegible per a l'LLM.
    Cada chunk s'etiqueta amb el seu patch i secció per facilitar les cites.
    
    Args:
        chunks: Llista de chunks amb text i metadata
    Returns:
        Text formatat amb tots els chunks numerats
    """
    if not chunks:
        return "No s'han trobat chunks rellevants."
    
    context_parts = []
    for i, chunk in enumerate(chunks, 1):
        # Etiquetem cada chunk amb origen per facilitar que l'LLM citi
        patch = chunk["metadata"]["patch"]
        section = chunk["metadata"]["section"]
        context_parts.append(
            f"[Font {i} — Patch {patch}, Secció: {section}]\n{chunk['text']}"
        )
    
    return "\n\n---\n\n".join(context_parts)


def generate_answer(query: str, context: str) -> str:
    """
    Genera una resposta utilitzant l'LLM amb el context dels chunks.
    
    Args:
        query: La pregunta original de l'usuari
        context: El text dels chunks formatat
    Returns:
        La resposta generada per l'LLM
    """
    response = openai_client.chat.completions.create(
        model=LLM_MODEL,
        messages=[
            {"role": "system", "content": RAG_SYSTEM_PROMPT},
            {"role": "user", "content": f"CONTEXT:\n{context}\n\nPREGUNTA: {query}"}
        ],
        temperature=0.1  # Temperatura baixa per respostes més deterministes
    )
    return response.choices[0].message.content


def ask(query: str) -> dict:
    """
    Funció principal del pipeline RAG.
    Orquestra tot el procés: retrieve → augment → generate.
    
    Args:
        query: La pregunta de l'usuari
    Returns:
        Diccionari amb 'answer', 'sources' i 'chunks_used'
    """
    # Pas 1: RETRIEVE — Buscar chunks rellevants
    chunks = retrieve_chunks(query)
    
    # Pas 2: AUGMENT — Formatar context per al prompt
    context = format_context(chunks)
    
    # Pas 3: GENERATE — Generar resposta amb l'LLM
    answer = generate_answer(query, context)
    
    return {
        "answer": answer,
        "query": query,
        "chunks_used": len(chunks),
        "sources": [
            {
                "patch": c["metadata"]["patch"],
                "section": c["metadata"]["section"],
                "score": round(c["score"], 3)
            }
            for c in chunks
        ]
    }
```

### 2. Crear un script de test interactiu (15 min)

Crea `ai-python/src/retrieval/test_rag.py`:

```python
"""
Script de test interactiu per al motor RAG.
Executa 5 preguntes predefinides i mostra els resultats.
"""

from rag_engine import ask

# 5 preguntes de test que cobreixen diferents aspectes
# Cada pregunta busca informació que hauria d'estar a les patch notes
TEST_QUESTIONS = [
    "Quins canvis va rebre Jinx a la última patch?",
    "Quin campió va ser nerfejat més recentment?",
    "Hi ha hagut canvis a l'objecte Infinity Edge?",
    "Quins buffs van rebre els suports?",
    "Quin és l'estat actual del meta d'ADC?",
]

def run_tests():
    """Executa les 5 preguntes i mostra resultats detallats."""
    print("=" * 60)
    print("TEST DEL MOTOR RAG — Patch Notes EsportsPulse")
    print("=" * 60)
    
    for i, question in enumerate(TEST_QUESTIONS, 1):
        print(f"\n{'─' * 60}")
        print(f"Pregunta {i}: {question}")
        print(f"{'─' * 60}")
        
        # Executar el pipeline RAG complet
        result = ask(question)
        
        # Mostrar la resposta
        print(f"\nResposta: {result['answer']}")
        print(f"\nChunks usats: {result['chunks_used']}")
        
        # Mostrar les fonts
        if result["sources"]:
            print("Fonts:")
            for src in result["sources"]:
                print(f"  - Patch {src['patch']}, "
                      f"Secció: {src['section']}, "
                      f"Score: {src['score']}")
        else:
            print("  Cap font trobada (score per sota del llindar)")
        
        print()

if __name__ == "__main__":
    run_tests()
```

### 3. Executar i verificar (15 min)

```bash
# Assegura't que Qdrant està corrent (hauries de tenir-lo des de la S14)
docker ps | grep qdrant

# Si no està corrent, arrencar-lo
docker compose up -d qdrant

# Executar els tests
cd ai-python/src/retrieval
python test_rag.py
```

Verifica que:
- Les respostes es basen en les patch notes reals (no inventa informació)
- El score dels chunks és >= 0.7
- Les fonts citades corresponen a patchs reals de la teva wiki
- Si una pregunta no té resposta, l'LLM ho indica

### 4. Integrar amb FastAPI (20 min)

Afegeix un endpoint a la teva API existent:

```python
"""
Endpoint RAG per a l'API FastAPI d'EsportsPulse.
Permet fer preguntes sobre patch notes via HTTP.
"""

from fastapi import APIRouter, HTTPException
from pydantic import BaseModel, Field
from retrieval.rag_engine import ask

# Router dedicat per als endpoints de retrieval
router = APIRouter(prefix="/api/v1/rag", tags=["RAG"])


class QuestionRequest(BaseModel):
    """Model de la petició: una pregunta en llenguatge natural."""
    question: str = Field(
        ...,
        min_length=5,
        max_length=500,
        description="La pregunta sobre patch notes",
        examples=["Quins canvis va rebre Jinx?"]
    )


class SourceInfo(BaseModel):
    """Informació d'una font citada a la resposta."""
    patch: str
    section: str
    score: float


class AnswerResponse(BaseModel):
    """Model de la resposta del pipeline RAG."""
    answer: str
    query: str
    chunks_used: int
    sources: list[SourceInfo]


@router.post("/ask", response_model=AnswerResponse)
async def ask_question(request: QuestionRequest) -> AnswerResponse:
    """
    Endpoint principal de consulta RAG.
    Rep una pregunta i retorna una resposta fonamentada en les patch notes.
    """
    try:
        result = ask(request.question)
        return AnswerResponse(**result)
    except Exception as e:
        # Log l'error intern però no exposar detalls al client
        raise HTTPException(
            status_code=500,
            detail="Error processant la consulta. Torna-ho a provar."
        )
```

Registra el router al teu `main.py` de FastAPI:

```python
# Al fitxer main.py, afegeix:
from retrieval.rag_router import router as rag_router

app.include_router(rag_router)
```

Prova amb curl:

```bash
# Provar l'endpoint RAG
curl -X POST http://localhost:8000/api/v1/rag/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Quins canvis va rebre Jinx?"}'
```

---

## Checklist de Lliurament

- [ ] `rag_engine.py` creat amb funcions `retrieve_chunks`, `format_context`, `generate_answer` i `ask`
- [ ] Les 5 preguntes de test retornen respostes fonamentades en chunks reals
- [ ] Els scores dels chunks retornats estan per sobre de 0.7
- [ ] Endpoint `/api/v1/rag/ask` funcional i respon correctament
- [ ] Commit: `feat(rag): implement semantic retrieval pipeline with Qdrant and LLM`
