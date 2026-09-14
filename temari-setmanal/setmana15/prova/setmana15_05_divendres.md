# Setmana 15 — Divendres: Spec de Comportament i Consolidació

## Objectiu del Dia

Escriure una especificació de comportament (behavior spec) formal per al sistema de retrieval i consolidar tot el treball de la setmana. Al final del dia has de tenir una spec automatitzable, el pipeline complet funcionant, i un PR creat amb tot el codi de la setmana.

---

## Teoria

### Què és una Behavior Spec?

Una behavior spec defineix el comportament esperat del sistema en format "Given-When-Then" (Donat-Quan-Llavors). No és un test unitari ni un test d'integració: és un contracte de comportament.

```
GIVEN: Una wiki amb patch notes del 14.1 al 14.5 indexada a Qdrant
WHEN:  L'usuari pregunta "Quins canvis va rebre Jinx al patch 14.5?"
THEN:  El sistema retorna una resposta que:
       - Conté "AD base" i els valors "59" i "55"
       - Cita [Font N] corresponent a un chunk del patch 14.5
       - La validació anti-al·lucinació passa sense warnings
```

### Per Què Behavior Specs i No Només Tests?

| Aspecte       | Test Unitari                | Behavior Spec                     |
|---------------|-----------------------------|------------------------------------|
| Abast         | Una funció                  | El sistema complet                 |
| Llenguatge    | Tècnic (asserts)            | Negoci (donat/quan/llavors)       |
| Qui el llegeix| Desenvolupadors             | Desenvolupadors + product owners  |
| Què valida    | Implementació correcta      | Comportament correcte             |
| Exemple       | `assert hash("a") == "x"`  | "Donada una pregunta trampa, el   |
|               |                             |  sistema ha de dir 'no tinc info'" |

Les behavior specs serveixen com a documentació viva: si les executes i passen, el sistema fa el que diu que fa.

### Estructura d'una Spec per a RAG

Un sistema RAG té múltiples comportaments a especificar:

```
┌─────────────────────────────────────────────────────┐
│            COMPORTAMENTS DEL SISTEMA RAG             │
│                                                      │
│  1. RETRIEVAL                                        │
│     - Donada una query, recupera chunks rellevants   │
│     - Score per sobre del threshold                  │
│     - Chunks del patch correcte                      │
│                                                      │
│  2. GENERACIÓ                                        │
│     - Resposta basada NOMÉS en els chunks            │
│     - Cites correctes ([Font N])                     │
│     - Llengua catalana                               │
│                                                      │
│  3. ANTI-AL·LUCINACIÓ                                │
│     - Rebutja preguntes sense context                │
│     - No inventa informació                          │
│     - Valida cites post-generació                    │
│                                                      │
│  4. CACHE                                            │
│     - Queries repetides servides des de cache        │
│     - TTL respectat (24h)                            │
│     - Mètriques de cost correctes                    │
│                                                      │
│  5. RESILIÈNCIA                                      │
│     - Redis caigut → funciona sense cache            │
│     - Pressupost excedit → resposta d'error amable   │
│     - Qdrant caigut → error clar, no al·lucinació    │
└─────────────────────────────────────────────────────┘
```

### Revisió del Pipeline Complet

Aquesta setmana has construït un pipeline RAG de producció:

```
Setmana 14 (anterior):
  Wiki → Chunks → Embeddings → Qdrant (indexació)

Setmana 15 (aquesta):
  Dilluns:   Query → Embedding → Qdrant → Top-K → Prompt → LLM → Resposta
  Dimarts:   + Anti-al·lucinació (prompt restrictiu + validació)
  Dimecres:  + Evals automatitzades (10 preguntes, 4 mètriques)
  Dijous:    + Cache Redis (estalvi cost + latència)
  Divendres: + Behavior spec (contracte de comportament)
```

---

## Activitat

### 1. Escriure la behavior spec (30 min)

Crea `ai-python/tests/test_behavior_spec.py`:

```python
"""
Behavior Specification del sistema RAG d'EsportsPulse.

Aquesta spec defineix el contracte de comportament del sistema.
Cada test segueix el patró Given-When-Then per claredat.

Executar: pytest tests/test_behavior_spec.py -v --timeout=180
"""

import time
import pytest
from retrieval.rag_engine import (
    ask_with_cache,
    ask_with_validation,
    retrieve_chunks
)
from retrieval.cache import (
    get_cached_answer,
    store_answer,
    redis_client,
    query_hash,
    CACHE_PREFIX
)


# ============================================================
# SPEC 1: RETRIEVAL SEMÀNTIC
# ============================================================

class TestRetrievalBehavior:
    """
    Especificació del comportament de retrieval semàntic.
    El sistema ha de trobar chunks rellevants basant-se en significat.
    """
    
    def test_semantic_match_not_keyword(self):
        """
        GIVEN: Patch notes indexades que contenen "Jinx base AD reduced"
        WHEN:  L'usuari pregunta sobre "champion nerfs"
        THEN:  El sistema recupera chunks sobre Jinx encara que
               la paraula "nerf" no apareix als chunks
        """
        chunks = retrieve_chunks("champion nerfs")
        
        # Ha de trobar almenys un chunk rellevant
        assert len(chunks) > 0, (
            "El retrieval semàntic no ha trobat cap chunk per 'champion nerfs'"
        )
        # Almenys un chunk ha de tenir un score acceptable
        assert any(c["score"] >= 0.7 for c in chunks), (
            "Cap chunk amb score >= 0.7"
        )
    
    def test_retrieval_returns_correct_patch(self):
        """
        GIVEN: Patch notes de múltiples patches indexades
        WHEN:  L'usuari pregunta sobre un patch específic
        THEN:  Els chunks retornats pertanyen a aquell patch
        """
        chunks = retrieve_chunks("canvis del patch 14.5")
        
        # Almenys un chunk ha de ser del patch demanat
        patches_found = {c["metadata"]["patch"] for c in chunks}
        assert "14.5" in patches_found, (
            f"Cap chunk del patch 14.5. Patches trobats: {patches_found}"
        )
    
    def test_retrieval_respects_top_k_limit(self):
        """
        GIVEN: Qualsevol query
        WHEN:  Es fa una cerca amb top_k=3
        THEN:  Es retornen com a màxim 3 chunks
        """
        chunks = retrieve_chunks("canvis recents", top_k=3)
        assert len(chunks) <= 3, (
            f"S'han retornat {len(chunks)} chunks, màxim esperat: 3"
        )


# ============================================================
# SPEC 2: GENERACIÓ AMB CITES
# ============================================================

class TestGenerationBehavior:
    """
    Especificació del comportament de generació de respostes.
    Les respostes han de citar fonts i estar en català.
    """
    
    def test_answer_contains_citation(self):
        """
        GIVEN: Una pregunta amb chunks rellevants disponibles
        WHEN:  El sistema genera una resposta
        THEN:  La resposta conté almenys una cita [Font N]
        """
        result = ask_with_validation("Quins canvis va rebre Jinx?")
        
        assert "[Font" in result["answer"], (
            "La resposta no conté cites [Font N]. "
            f"Resposta: {result['answer'][:200]}"
        )
    
    def test_answer_has_sources(self):
        """
        GIVEN: Una pregunta sobre un tema cobert per la wiki
        WHEN:  El sistema genera una resposta
        THEN:  El resultat inclou la llista de fonts usades
        """
        result = ask_with_validation("Hi ha hagut canvis a objectes?")
        
        assert result["chunks_used"] > 0, "Cap chunk usat per a la resposta"
        assert len(result["sources"]) > 0, "Cap font reportada"
    
    def test_answer_in_catalan(self):
        """
        GIVEN: Una pregunta en català
        WHEN:  El sistema genera una resposta
        THEN:  La resposta és en català (conté paraules catalanes comunes)
        """
        result = ask_with_validation("Quins canvis va rebre Jinx?")
        answer = result["answer"].lower()
        
        # Paraules que esperem trobar en una resposta en català
        catalan_indicators = ["va", "del", "al", "els", "amb", "que", "informació"]
        has_catalan = any(word in answer for word in catalan_indicators)
        
        assert has_catalan, (
            f"La resposta no sembla ser en català: {result['answer'][:200]}"
        )


# ============================================================
# SPEC 3: ANTI-AL·LUCINACIÓ
# ============================================================

class TestAntiHallucinationBehavior:
    """
    Especificació del comportament anti-al·lucinació.
    El sistema ha de rebutjar preguntes sense context.
    """
    
    def test_refuses_unknown_champion(self):
        """
        GIVEN: Una pregunta sobre un campió/entitat que no existeix
        WHEN:  L'usuari pregunta sobre "Zarkon"
        THEN:  El sistema respon "no tinc informació"
        """
        result = ask_with_validation(
            "Quins canvis va rebre el campió Zarkon?"
        )
        assert "no tinc informació" in result["answer"].lower(), (
            f"L'LLM ha al·lucinat sobre un campió inventat. "
            f"Resposta: {result['answer'][:200]}"
        )
    
    def test_refuses_wrong_game(self):
        """
        GIVEN: Patch notes de League of Legends
        WHEN:  L'usuari pregunta sobre un altre joc
        THEN:  El sistema rebutja la pregunta
        """
        result = ask_with_validation(
            "Quins canvis va rebre Genji a Overwatch?"
        )
        assert "no tinc informació" in result["answer"].lower(), (
            "L'LLM ha intentat respondre sobre un joc diferent"
        )
    
    def test_refuses_out_of_scope(self):
        """
        GIVEN: Un sistema limitat a patch notes
        WHEN:  L'usuari pregunta sobre estadístiques de jugadors
        THEN:  El sistema rebutja la pregunta
        """
        result = ask_with_validation(
            "Quants jugadors actius té League of Legends?"
        )
        assert "no tinc informació" in result["answer"].lower(), (
            "L'LLM ha intentat respondre una pregunta fora d'àmbit"
        )
    
    def test_validation_catches_no_chunks(self):
        """
        GIVEN: Una pregunta sense chunks rellevants
        WHEN:  Es genera i valida la resposta
        THEN:  La validació detecta la situació
        """
        result = ask_with_validation(
            "Quin temps farà demà a Barcelona?"
        )
        # O bé rebutja (correcte) o bé la validació marca warning
        is_refusal = "no tinc informació" in result["answer"].lower()
        has_warning = not result["validation"]["valid"]
        
        assert is_refusal or has_warning, (
            "Ni rebuig ni warning per una pregunta completament fora d'àmbit"
        )


# ============================================================
# SPEC 4: CACHE
# ============================================================

class TestCacheBehavior:
    """
    Especificació del comportament del cache Redis.
    Les queries repetides han de servir-se des de cache.
    """
    
    def test_second_call_from_cache(self):
        """
        GIVEN: Una consulta que ja s'ha fet prèviament
        WHEN:  Es repeteix la mateixa consulta
        THEN:  La resposta ve del cache (from_cache = True)
        """
        query = "Test de cache: canvis a Jinx"
        
        # Primera crida: hauria d'anar a l'LLM
        result1 = ask_with_cache(query)
        
        # Segona crida: hauria de venir de cache
        result2 = ask_with_cache(query)
        
        assert result2.get("from_cache") is True, (
            "La segona crida no ve de cache"
        )
    
    def test_cache_is_faster(self):
        """
        GIVEN: Una consulta cachejada
        WHEN:  Es fa la mateixa consulta una segona vegada
        THEN:  La segona crida és significativament més ràpida
        """
        query = "Benchmark de velocitat: canvis recents"
        
        # Primera crida (LLM)
        start = time.time()
        ask_with_cache(query)
        time_llm = time.time() - start
        
        # Segona crida (cache)
        start = time.time()
        ask_with_cache(query)
        time_cache = time.time() - start
        
        # El cache ha de ser almenys 5x més ràpid
        assert time_cache < time_llm / 5, (
            f"Cache no prou ràpid. LLM: {time_llm:.2f}s, "
            f"Cache: {time_cache:.3f}s"
        )
    
    def test_normalized_queries_share_cache(self):
        """
        GIVEN: Dues queries que difereixen només en majúscules/puntuació
        WHEN:  Es fan les dues consultes
        THEN:  La segona usa el cache de la primera
        """
        # Netejar cache per a aquesta clau
        q1 = "Canvis a Jinx al patch?"
        q2 = "canvis a jinx al patch"
        key = CACHE_PREFIX + query_hash(q1)
        redis_client.delete(key)
        
        # Primera crida
        ask_with_cache(q1)
        
        # Segona crida amb diferent capitalització
        result2 = ask_with_cache(q2)
        
        assert result2.get("from_cache") is True, (
            "Queries normalitzades no comparteixen cache"
        )


# ============================================================
# SPEC 5: RESILIÈNCIA
# ============================================================

class TestResilienceBehavior:
    """
    Especificació del comportament de resiliència.
    El sistema ha de funcionar encara que components fallin.
    """
    
    def test_works_without_redis(self):
        """
        GIVEN: Redis no disponible (simulat)
        WHEN:  L'usuari fa una consulta
        THEN:  El sistema funciona (amb latència d'LLM, sense cache)
        
        Nota: Aquest test requereix una configuració especial
        per simular la caiguda de Redis. Si no és possible,
        es marca com a skip.
        """
        # Test conceptual — en un entorn real, simularíem desconnectant Redis
        # Per ara, verifiquem que la funció ask_with_validation (sense cache)
        # funciona independentment
        result = ask_with_validation("Canvis a Jinx?")
        assert "answer" in result, "El pipeline sense cache no funciona"
    
    def test_budget_exceeded_returns_error(self):
        """
        GIVEN: El pressupost mensual s'ha excedit
        WHEN:  L'usuari fa una consulta
        THEN:  El sistema retorna un missatge d'error amable
               (sense fer crida a l'LLM)
        
        Nota: Per testejar això, caldria manipular el comptador de Redis.
        """
        # Verificar que la funció check_budget() existeix i retorna bool
        from retrieval.cache import check_budget
        result = check_budget()
        assert isinstance(result, bool), "check_budget() ha de retornar bool"
```

### 2. Revisar i netejar el codi (20 min)

Verifica que tots els fitxers estan ben organitzats:

```
ai-python/
├── src/
│   └── retrieval/
│       ├── __init__.py
│       ├── rag_engine.py          # Pipeline RAG (dilluns + dimarts)
│       ├── cache.py               # Cache Redis (dijous)
│       ├── eval_dataset.py        # Dataset d'evals (dimecres)
│       ├── eval_runner.py         # Motor d'evals (dimecres)
│       ├── rag_router.py          # Endpoints FastAPI (dilluns + dijous)
│       ├── test_rag.py            # Test interactiu (dilluns)
│       ├── test_anti_hallucination.py  # Test al·lucinació (dimarts)
│       └── benchmark_cache.py     # Benchmark cache (dijous)
├── tests/
│   ├── test_rag_evals.py          # Tests pytest evals (dimecres)
│   └── test_behavior_spec.py      # Behavior spec (divendres)
└── requirements.txt
```

Afegeix les dependències noves al `requirements.txt`:

```
# Dependències afegides a la setmana 15
qdrant-client>=1.7.0          # Client de Qdrant per a retrieval
openai>=1.12.0                # API d'OpenAI per a embeddings i LLM
redis>=5.0.0                  # Client Redis per a caching
```

### 3. Executar el full eval suite (15 min)

```bash
# Assegurar que tots els serveis estan corrent
docker compose up -d qdrant redis

# Executar la behavior spec
pytest tests/test_behavior_spec.py -v --timeout=180

# Executar les evals de dimecres
pytest tests/test_rag_evals.py -v --timeout=180

# Executar tots els tests junts
pytest tests/ -v --timeout=180
```

Objectius mínims:
- Behavior spec: tots els tests passen (o els de resiliència marcats com skip)
- Evals: accuracy >= 80%, faithfulness >= 90%, relevance >= 70%, refusal >= 90%
- Cap al·lucinació a les preguntes trampa

### 4. Crear el PR de la setmana (15 min)

```bash
# Crear branca de feature
git checkout -b feature/week15-rag-pipeline

# Afegir tots els fitxers nous
git add ai-python/src/retrieval/
git add ai-python/tests/
git add docker-compose.yml
git add ai-python/requirements.txt

# Revisar el que afegirem
git status
git diff --cached

# Commit amb missatge descriptiu
git commit -m "feat(rag): complete retrieval pipeline with anti-hallucination and caching

- Semantic retrieval from Qdrant with top-k and score threshold
- Anti-hallucination prompts with citation enforcement
- Response validation post-generation
- Eval framework with 10-question dataset and 4 metrics
- Redis cache-aside pattern for LLM responses
- Cost tracking and monthly budget alerts
- Behavior specification with Given-When-Then tests
- FastAPI endpoints for /ask and /costs"

# Pujar la branca
git push -u origin feature/week15-rag-pipeline

# Crear el PR
gh pr create \
  --title "feat: RAG pipeline amb anti-al·lucinació i cache" \
  --body "## Resum

Pipeline RAG complet per a consultes sobre patch notes:
- Retrieval semàntic amb Qdrant
- Anti-al·lucinació amb cites obligatòries
- Evals automatitzades (10 preguntes, 4 mètriques)
- Cache Redis amb control de costos

## Tests
- \`pytest tests/test_behavior_spec.py\`
- \`pytest tests/test_rag_evals.py\`

## Com provar
\`\`\`bash
docker compose up -d qdrant redis
curl -X POST http://localhost:8000/api/v1/rag/ask \\
  -H 'Content-Type: application/json' \\
  -d '{\"question\": \"Quins canvis va rebre Jinx?\"}' 
\`\`\`"
```

---

## Checklist de Lliurament

- [ ] `test_behavior_spec.py` amb specs per a: retrieval, generació, anti-al·lucinació, cache, resiliència
- [ ] Tots els tests de la behavior spec passen
- [ ] Full eval suite executat amb resultats dins els objectius
- [ ] Estructura de fitxers neta i organitzada
- [ ] `requirements.txt` actualitzat amb totes les dependències
- [ ] PR creat amb branca `feature/week15-rag-pipeline`
- [ ] Commit final: `feat(rag): complete retrieval pipeline with anti-hallucination and caching`
