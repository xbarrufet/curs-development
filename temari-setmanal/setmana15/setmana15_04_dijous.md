# Setmana 15 — Dijous: Cache Redis per a Respostes d'LLM i Cost Management

## Objectiu del Dia

Implementar un sistema de cache amb Redis per a les respostes del pipeline RAG i establir controls de cost. Al final del dia, les consultes repetides es serviran des de cache (latència < 50ms vs ~2s), i tindràs un dashboard bàsic de cost mensual estimat.

---

## Teoria

### El Problema del Cost i la Latència

Cada crida a l'LLM costa diners i temps:

```
Crida típica a GPT-4o-mini:
  - Prompt: ~800 tokens (system + context + pregunta)
  - Resposta: ~200 tokens
  - Cost: ~$0.0003 per crida
  - Latència: 1-3 segons

Escala real:
  - 100 consultes/dia × 30 dies = 3.000 crides/mes
  - Cost: 3.000 × $0.0003 = $0.90/mes (barat)
  
  PERÒ amb GPT-4o (no mini):
  - Cost: 3.000 × $0.01 = $30/mes
  - I si el producte creix a 10.000 consultes/dia...
  - Cost: 300.000 × $0.01 = $3.000/mes 💸
```

Moltes consultes són repetides. "Quins canvis va rebre Jinx?" la pregunten 50 persones. Per què pagar 50 vegades per la mateixa resposta?

### El Patró Cache-Aside

El patró cache-aside (o "lazy loading") és el més comú per a caching:

```
┌──────────────────────────────────────────────────┐
│              PATRÓ CACHE-ASIDE                    │
│                                                   │
│  Consulta → Hash de la query                      │
│     │                                             │
│     ├─→ Buscar a Redis (cache)                    │
│     │     │                                       │
│     │     ├─ HIT → Retornar resposta cached ⚡    │
│     │     │        (latència: < 50ms)             │
│     │     │                                       │
│     │     └─ MISS → Executar pipeline RAG         │
│     │                │                            │
│     │                ├─ Guardar a Redis amb TTL   │
│     │                └─ Retornar resposta          │
│     │                   (latència: 1-3s)          │
└──────────────────────────────────────────────────┘
```

**Per què funciona:**
- Molts usuaris fan les mateixes preguntes (o molt similars)
- Les patch notes no canvien constantment
- Un TTL de 24h garanteix frescura quan es publica una nova patch

### Hash de la Query: Normalització

No pots fer servir la query directament com a clau. "Quins canvis va rebre Jinx?" i "quins canvis va rebre jinx?" haurien de ser la mateixa clau:

```python
import hashlib

def normalize_query(query: str) -> str:
    """
    Normalitza la query per maximitzar cache hits.
    Converteix a minúscules, elimina espais extra i puntuació.
    """
    # Minúscules i sense espais extra
    normalized = query.lower().strip()
    # Eliminar puntuació que no afecta el significat
    normalized = normalized.rstrip("?!.")
    return normalized

def query_hash(query: str) -> str:
    """Genera un hash determinista de la query normalitzada."""
    normalized = normalize_query(query)
    return hashlib.sha256(normalized.encode()).hexdigest()[:16]
```

### TTL (Time-To-Live): Frescura vs Estalvi

El TTL determina quan caduca una entrada de cache:

| TTL       | Frescura | Cache Hits | Ús recomanat                |
|-----------|----------|------------|-----------------------------|
| 1 hora    | Alta     | Baixos     | Dades molt canviants        |
| 24 hores  | Bona     | Alts       | Patch notes (canvien setmanalment) |
| 7 dies    | Baixa    | Molt alts  | Dades estàtiques            |
| 0 (sense) | Sempre   | Sempre     | Mai fer-ho en producció     |

Per a les patch notes, 24h (86400 segons) és ideal: prou temps per aprofitar el cache, prou curt per reflectir noves patches.

### Cost Management: Calcular i Controlar

Fórmula de cost mensual:

```
Cost = (Crides totals - Cache hits) × Cost per crida
     = Crides totals × (1 - Cache hit ratio) × Cost per crida

Exemple:
  3.000 crides/mes × (1 - 0.60) × $0.01 = $12/mes
  (sense cache seria $30/mes → estalvi del 60%)
```

Per controlar els costos, implementem:
1. **Rate limiting**: Màxim N crides per minut per usuari
2. **Budget alert**: Notificació quan arribes al 80% del pressupost mensual
3. **Circuit breaker**: Tallar crides si superes el pressupost

### Tècniques de Minimització de Tokens

Abans de pagar menys amb cache, pots **enviar menys** amb cada crida. Aquestes tècniques redueixen els input tokens:

#### 1. Prompt Compression: Dir el Mateix amb Menys

```python
# ABANS — 180 tokens de system prompt:
system_verbose = """
Ets un analista expert de League of Legends amb anys d'experiència.
La teva feina és analitzar campions i proporcionar informació detallada
sobre les seves estadístiques, punts forts, punts febles i el seu estat
actual al meta del joc. Quan responguis, assegura't de ser precís
i objectiu, basant-te en dades reals i no en opinions subjectives.
Respon sempre en català.
"""

# DESPRÉS — 45 tokens, mateixa informació:
system_compact = """Analista LoL. Dades objectives de campions.
Resposta en català. Si no tens dades, digues-ho."""
```

**Regla:** Cada token del system prompt es paga a TOTES les crides. Un prompt 4x més curt estalvia un 75% en system prompt tokens al llarg de milers de crides.

#### 2. Sliding Window: Limitar l'Historial de Conversa

En agents multi-pas, l'historial creix a cada iteració. Sense control, el context window s'omple:

```python
def trim_conversation(messages: list, max_messages: int = 10) -> list:
    """Manté només els últims N missatges de la conversa."""
    if len(messages) <= max_messages:
        return messages
    # Sempre conserva el primer missatge (context inicial)
    # i els últims max_messages-1
    return [messages[0]] + messages[-(max_messages - 1):]
```

#### 3. Summarization: Comprimir Resultats de Tools

Quan un tool retorna molt de text (100 resultats d'una query), resumir abans d'enviar al model:

```python
def summarize_tool_results(results: list, max_items: int = 5) -> list:
    """Retorna només els top-N resultats més rellevants."""
    sorted_results = sorted(results, key=lambda x: x.get("relevance_score", 0), reverse=True)
    return sorted_results[:max_items]
```

#### 4. Structured Output: JSON Compacte

```python
# ABANS — demanar text lliure genera 200+ output tokens
# "Jinx és una Marksman amb un win rate del 51.5%..."

# DESPRÉS — forçar JSON amb tool_use genera 50 output tokens
# {"name": "Jinx", "role": "Marksman", "win_rate": 51.5}
```

### Prompt Caching d'Anthropic: Pagar Menys pel System Prompt

Anthropic ofereix **prompt caching** — si el teu system prompt (o qualsevol prefix dels missatges) és idèntic entre crides, el servidor el cacheja i el cobres amb descompte:

```python
# Amb prompt caching, el system prompt llarg es paga 1 cop complet
# i les crides posteriors paguen només el 10% pel prefix cachejat

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": system_prompt_llarg,  # 2000 tokens
            "cache_control": {"type": "ephemeral"}  # Activa el caching
        }
    ],
    messages=[{"role": "user", "content": query}]
)

# Primera crida: pagues 2000 input tokens complets
# Crides següents (dins de 5 min): pagues 200 tokens (90% descompte)
```

| Escenari | Sense cache | Amb prompt caching |
|----------|-------------|-------------------|
| System prompt (2000 tokens) × 100 crides | 200K tokens input | 20K + 2K = 22K tokens |
| Cost (Sonnet, $3/1M) | $0.60 | $0.066 |
| **Estalvi** | — | **89%** |

> **Quan usar-lo:** Sempre que el system prompt sigui > 1000 tokens i facis múltiples crides en poc temps. Perfecte per agents (mateix system prompt, múltiples passos) i per al pipeline RAG (system prompt + few-shot exemples fixes).

### Resum: On Estalviar Tokens

```
┌─────────────────────────────────────────────────────────────┐
│              ESTRATÈGIES DE MINIMITZACIÓ                     │
│                                                              │
│  1. Prompt compression     → Menys input tokens per crida   │
│  2. Sliding window         → Menys historial acumulat       │
│  3. Summarize tool results → Menys context de tools         │
│  4. Structured output      → Menys output tokens            │
│  5. Prompt caching         → 90% descompte en prefix fixe   │
│  6. Model routing          → Haiku per tasques simples      │
│  7. Redis cache (avui)     → 0 tokens per queries repetides │
│                                                              │
│  Combinant tot: reducció de cost del 80-95% vs naïf         │
└─────────────────────────────────────────────────────────────┘
```

---

## Activitat

### 1. Configurar Redis amb Docker (10 min)

Afegeix Redis al teu `docker-compose.yml`:

```yaml
# Servei Redis per a caching de respostes LLM
# S'afegeix a la configuració existent de Docker Compose
services:
  # ... serveis existents (qdrant, etc.) ...
  
  redis:
    image: redis:7-alpine           # Versió lleugera de Redis
    container_name: esportspulse-redis
    ports:
      - "6379:6379"                 # Port estàndard de Redis
    volumes:
      - redis_data:/data            # Persistir dades entre reinicis
    command: redis-server --maxmemory 100mb --maxmemory-policy allkeys-lru
    # maxmemory: límit de memòria (100MB és suficient per a cache)
    # allkeys-lru: quan es plena, elimina les claus menys usades recentment
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

volumes:
  redis_data:
```

```bash
# Arrencar Redis
docker compose up -d redis

# Verificar que funciona
docker exec esportspulse-redis redis-cli ping
# Hauria de respondre: PONG
```

### 2. Implementar el cache layer (30 min)

Crea `ai-python/src/retrieval/cache.py`:

```python
"""
Cache layer per a respostes del pipeline RAG.
Utilitza Redis amb el patró cache-aside per reduir crides a l'LLM.
"""

import json
import hashlib
import time
from datetime import datetime

import redis

# Connexió a Redis
# decode_responses=True per treballar amb strings (no bytes)
redis_client = redis.Redis(
    host="localhost",
    port=6379,
    db=0,
    decode_responses=True
)

# Configuració del cache
CACHE_TTL = 86400           # 24 hores en segons
CACHE_PREFIX = "rag:answer:" # Prefix per a les claus de cache

# Configuració de cost
COST_PER_CALL = 0.0003      # Cost per crida amb GPT-4o-mini ($)
COST_PREFIX = "rag:cost:"    # Prefix per a mètriques de cost
MONTHLY_BUDGET = 10.0        # Pressupost mensual en $
BUDGET_ALERT_THRESHOLD = 0.8 # Alertar al 80% del pressupost


def normalize_query(query: str) -> str:
    """
    Normalitza la query per maximitzar cache hits.
    Dues preguntes equivalents han de generar la mateixa clau.
    
    Args:
        query: La pregunta original de l'usuari
    Returns:
        Versió normalitzada de la query
    """
    # Convertir a minúscules
    normalized = query.lower().strip()
    # Eliminar puntuació final que no canvia el significat
    normalized = normalized.rstrip("?!.,;:")
    # Eliminar espais duplicats
    normalized = " ".join(normalized.split())
    return normalized


def query_hash(query: str) -> str:
    """
    Genera un hash determinista per a una query.
    Usem SHA-256 truncat a 16 caràcters (prou per evitar col·lisions).
    
    Args:
        query: La query (ja normalitzada o no)
    Returns:
        String hexadecimal de 16 caràcters
    """
    normalized = normalize_query(query)
    return hashlib.sha256(normalized.encode("utf-8")).hexdigest()[:16]


def get_cached_answer(query: str) -> dict | None:
    """
    Busca una resposta al cache per a la query donada.
    
    Args:
        query: La pregunta de l'usuari
    Returns:
        El diccionari de resposta si existeix al cache, None si no
    """
    key = CACHE_PREFIX + query_hash(query)
    
    try:
        cached = redis_client.get(key)
        if cached:
            result = json.loads(cached)
            result["from_cache"] = True
            return result
    except redis.ConnectionError:
        # Si Redis no està disponible, continuar sense cache
        print("WARN: Redis no disponible, saltant cache")
    
    return None


def store_answer(query: str, result: dict, ttl: int = CACHE_TTL) -> None:
    """
    Guarda una resposta al cache amb un TTL.
    
    Args:
        query: La pregunta original
        result: El diccionari de resposta del pipeline RAG
        ttl: Temps de vida en segons (per defecte 24h)
    """
    key = CACHE_PREFIX + query_hash(query)
    
    try:
        # Serialitzar a JSON per guardar a Redis
        redis_client.setex(
            name=key,
            time=ttl,
            value=json.dumps(result, ensure_ascii=False)
        )
    except redis.ConnectionError:
        print("WARN: Redis no disponible, resposta no cached")


def track_llm_call(cost: float = COST_PER_CALL) -> None:
    """
    Registra una crida a l'LLM per al seguiment de costos.
    Utilitza un comptador diari a Redis.
    
    Args:
        cost: Cost de la crida en dòlars
    """
    today = datetime.now().strftime("%Y-%m-%d")
    month = datetime.now().strftime("%Y-%m")
    
    try:
        # Comptador diari de crides
        daily_key = f"{COST_PREFIX}calls:{today}"
        redis_client.incr(daily_key)
        redis_client.expire(daily_key, 86400 * 35)  # Retenir 35 dies
        
        # Acumulador mensual de cost (en centaus per evitar floats)
        monthly_key = f"{COST_PREFIX}total:{month}"
        cost_cents = int(cost * 10000)  # Guardem en dècimes de centau
        redis_client.incrby(monthly_key, cost_cents)
        redis_client.expire(monthly_key, 86400 * 35)
        
    except redis.ConnectionError:
        pass  # No bloquejar si Redis falla


def track_cache_hit() -> None:
    """Registra un cache hit per a mètriques."""
    today = datetime.now().strftime("%Y-%m-%d")
    try:
        key = f"{COST_PREFIX}cache_hits:{today}"
        redis_client.incr(key)
        redis_client.expire(key, 86400 * 35)
    except redis.ConnectionError:
        pass


def get_cost_summary() -> dict:
    """
    Retorna un resum del cost mensual actual.
    
    Returns:
        Diccionari amb cost actual, pressupost, i mètriques de cache
    """
    month = datetime.now().strftime("%Y-%m")
    today = datetime.now().strftime("%Y-%m-%d")
    
    try:
        # Cost mensual acumulat
        monthly_key = f"{COST_PREFIX}total:{month}"
        cost_raw = redis_client.get(monthly_key) or "0"
        monthly_cost = int(cost_raw) / 10000  # Convertir de centaus a dòlars
        
        # Crides i cache hits d'avui
        daily_calls = int(redis_client.get(
            f"{COST_PREFIX}calls:{today}"
        ) or "0")
        daily_hits = int(redis_client.get(
            f"{COST_PREFIX}cache_hits:{today}"
        ) or "0")
        
        # Cache hit ratio
        total_requests = daily_calls + daily_hits
        hit_ratio = daily_hits / total_requests if total_requests > 0 else 0
        
        # Alerta de pressupost
        budget_used = monthly_cost / MONTHLY_BUDGET
        alert = budget_used >= BUDGET_ALERT_THRESHOLD
        
        return {
            "monthly_cost_usd": round(monthly_cost, 4),
            "monthly_budget_usd": MONTHLY_BUDGET,
            "budget_used_pct": round(budget_used * 100, 1),
            "budget_alert": alert,
            "today_llm_calls": daily_calls,
            "today_cache_hits": daily_hits,
            "today_hit_ratio": round(hit_ratio * 100, 1),
            "estimated_monthly_savings_usd": round(
                daily_hits * COST_PER_CALL * 30, 2
            )
        }
    except redis.ConnectionError:
        return {"error": "Redis no disponible"}


def check_budget() -> bool:
    """
    Comprova si estem dins del pressupost mensual.
    
    Returns:
        True si podem fer més crides, False si hem superat el pressupost
    """
    summary = get_cost_summary()
    if "error" in summary:
        return True  # Si no podem verificar, permetre (fail open)
    return summary["monthly_cost_usd"] < MONTHLY_BUDGET
```

### 3. Integrar el cache al pipeline RAG (20 min)

Actualitza `rag_engine.py`:

```python
"""
Afegir caching al pipeline RAG existent.
Integra les funcions de cache.py dins del flux ask_with_validation().
"""

from retrieval.cache import (
    get_cached_answer,
    store_answer,
    track_llm_call,
    track_cache_hit,
    check_budget
)


def ask_with_cache(query: str) -> dict:
    """
    Pipeline RAG complet amb caching i control de costos.
    
    Flux:
    1. Comprovar cache → si HIT, retornar immediatament
    2. Comprovar pressupost → si excedit, retornar error
    3. Executar pipeline RAG normal
    4. Guardar resultat al cache
    5. Registrar crida per a mètriques de cost
    
    Args:
        query: La pregunta de l'usuari
    Returns:
        Diccionari amb resposta, fonts, validació, i info de cache
    """
    # Pas 1: Comprovar cache
    cached = get_cached_answer(query)
    if cached:
        track_cache_hit()
        return cached
    
    # Pas 2: Comprovar pressupost
    if not check_budget():
        return {
            "answer": "S'ha superat el pressupost mensual d'IA. "
                      "Contacta l'administrador.",
            "query": query,
            "chunks_used": 0,
            "sources": [],
            "validation": {"valid": True, "reason": "Budget exceeded"},
            "from_cache": False,
            "budget_exceeded": True
        }
    
    # Pas 3: Executar pipeline RAG amb validació
    result = ask_with_validation(query)
    result["from_cache"] = False
    
    # Pas 4: Guardar al cache (només si la validació és OK)
    if result["validation"]["valid"]:
        store_answer(query, result)
    
    # Pas 5: Registrar cost
    track_llm_call()
    
    return result
```

### 4. Afegir endpoint de mètriques (15 min)

Actualitza el router FastAPI:

```python
"""
Endpoints addicionals per a mètriques de cost i cache.
"""

from retrieval.cache import get_cost_summary


@router.get("/costs", response_model=dict)
async def get_costs():
    """
    Retorna les mètriques de cost i cache del pipeline RAG.
    Útil per monitoritzar l'ús i detectar si cal ajustar el TTL.
    """
    return get_cost_summary()


@router.post("/ask", response_model=AnswerResponse)
async def ask_question(request: QuestionRequest) -> AnswerResponse:
    """
    Endpoint principal de consulta RAG (actualitzat amb cache).
    """
    try:
        # Usem ask_with_cache en lloc de ask_with_validation
        result = ask_with_cache(request.question)
        return AnswerResponse(**result)
    except Exception as e:
        raise HTTPException(
            status_code=500,
            detail="Error processant la consulta."
        )
```

### 5. Mesurar l'impacte del cache (15 min)

Crea `ai-python/src/retrieval/benchmark_cache.py`:

```python
"""
Benchmark per mesurar l'impacte del cache en latència i cost.
Executa les mateixes preguntes dues vegades: la primera sense cache, 
la segona amb cache. Compara temps de resposta.
"""

import time
from retrieval.rag_engine import ask_with_cache
from retrieval.cache import get_cost_summary

# Preguntes de benchmark
BENCHMARK_QUESTIONS = [
    "Quins canvis va rebre Jinx?",
    "Hi va haver canvis a objectes d'ADC?",
    "Quin campió suport va rebre un buff?",
    "Hi ha hagut canvis al sistema de drac?",
    "Quins nerfs van rebre els assassins?",
]


def run_benchmark():
    """Executa el benchmark de cache."""
    print("=" * 60)
    print("BENCHMARK DE CACHE — EsportsPulse RAG")
    print("=" * 60)
    
    # Primera passada: tot serà MISS (no hi ha cache)
    print("\n--- Primera passada (cache MISS esperat) ---")
    miss_times = []
    for q in BENCHMARK_QUESTIONS:
        start = time.time()
        result = ask_with_cache(q)
        elapsed = time.time() - start
        miss_times.append(elapsed)
        cached = "CACHE" if result.get("from_cache") else "LLM"
        print(f"  [{cached}] {elapsed:.2f}s — {q[:40]}...")
    
    # Segona passada: tot hauria de ser HIT
    print("\n--- Segona passada (cache HIT esperat) ---")
    hit_times = []
    for q in BENCHMARK_QUESTIONS:
        start = time.time()
        result = ask_with_cache(q)
        elapsed = time.time() - start
        hit_times.append(elapsed)
        cached = "CACHE" if result.get("from_cache") else "LLM"
        print(f"  [{cached}] {elapsed:.3f}s — {q[:40]}...")
    
    # Resum
    avg_miss = sum(miss_times) / len(miss_times)
    avg_hit = sum(hit_times) / len(hit_times)
    speedup = avg_miss / avg_hit if avg_hit > 0 else float("inf")
    
    print(f"\n{'=' * 60}")
    print("RESULTATS")
    print(f"{'─' * 60}")
    print(f"  Latència mitjana sense cache: {avg_miss:.2f}s")
    print(f"  Latència mitjana amb cache:   {avg_hit:.3f}s")
    print(f"  Speedup:                      {speedup:.0f}x")
    print(f"{'─' * 60}")
    
    # Mètriques de cost
    summary = get_cost_summary()
    print(f"\n  Cost mensual acumulat:   ${summary.get('monthly_cost_usd', 0)}")
    print(f"  Cache hit ratio (avui):  {summary.get('today_hit_ratio', 0)}%")
    print(f"  Estalvi estimat/mes:     ${summary.get('estimated_monthly_savings_usd', 0)}")
    print(f"{'=' * 60}")


if __name__ == "__main__":
    run_benchmark()
```

```bash
# Executar el benchmark
cd ai-python/src/retrieval
python benchmark_cache.py
```

Hauries de veure:
- Primera passada: ~1-3s per pregunta (crida real a l'LLM)
- Segona passada: < 0.05s per pregunta (resposta des de cache)
- Speedup de 20-60x

---

## Checklist de Lliurament

- [ ] Redis configurat al `docker-compose.yml` i funcionant
- [ ] `cache.py` amb funcions `get_cached_answer`, `store_answer`, `track_llm_call`
- [ ] `ask_with_cache()` integra cache-aside al pipeline RAG
- [ ] Benchmark mostra speedup > 10x per a cache hits
- [ ] Endpoint `/api/v1/rag/costs` retorna mètriques de cost i cache
- [ ] Control de pressupost: el sistema rebutja crides si supera el budget
- [ ] Commit: `feat(rag): add Redis cache for LLM responses and cost tracking`
