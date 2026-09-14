# Setmana 16 — Dijous: Evals d'Agents i Observabilitat amb LangFuse

## Objectiu del Dia

Crear un sistema d'avaluacio (evals) per als dos agents i afegir observabilitat amb LangFuse. Al final del dia has de tenir: un dataset de 15+ preguntes amb respostes de referencia, evals automatitzats amb pytest, traces visibles a LangFuse (cada pas del bucle agent), i una regla de CI que bloqueja merge si la fidelitat baixa del 80%.

---

## Teoria

### Per Que Necessites Evals d'Agents

Un agent es mes dificil d'avaluar que una funcio normal perque:

1. **No-determinisme:** La mateixa pregunta pot generar respostes lleugerament diferents
2. **Multi-pas:** L'agent pot arribar al resultat per camins diferents
3. **Cost acumulat:** Cada crida a eina costa tokens — un agent ineficient es car
4. **Regressio silenciosa:** Un canvi al system prompt pot millorar un cas i empitjorar-ne tres

```
Funcio classica:  input → output (determinista, facil de testejar)
Agent:            input → [raona → actua → observa] x N → output
                          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                          Cada iteracio pot variar
```

### Tres Dimensions d'Avaluacio

| Dimensio | Que mesura | Metrica |
|---|---|---|
| **Fidelitat** | La resposta es correcta? | % de respostes que coincideixen amb la referencia |
| **Eficiencia** | Quants passos necessita? | Nombre de crides a eines per consulta |
| **Cost** | Quant costa cada consulta? | Tokens totals, dolars per consulta |

### Dataset d'Avaluacio: Estructura

Cada entrada del dataset te:
- **Pregunta:** El que l'usuari demana
- **Resposta de referencia:** El que l'agent HAURIA de respondre (o contenir)
- **Eines esperades:** Quines eines hauria de cridar
- **Max crides:** Maxim de crides a eines acceptable

```python
# Estructura d'una entrada d'eval
{
    "id": "q001",
    "question": "Quin champion te el win rate mes alt?",
    "agent": "quantitative",  # Quin agent avaluem
    "expected_contains": ["Lux", "53.1"],  # Paraules que la resposta HA de contenir
    "expected_not_contains": ["no tinc informacio"],  # NO hauria de contenir
    "expected_tools": ["query_champions"],  # Eines que hauria d'usar
    "max_tool_calls": 2  # Maxim de crides acceptables
}
```

### LangFuse: Observabilitat per Agents

LangFuse es una plataforma d'observabilitat per aplicacions LLM. Registra:
- **Traces:** Cada conversa completa (de pregunta a resposta)
- **Spans:** Cada pas dins la conversa (cada crida a l'API, cada eina)
- **Metriques:** Latencia, tokens, cost per cada pas
- **Scores:** Resultats d'avaluacio associats a cada trace

```
Trace: "Quin champion te el win rate mes alt?"
├── Span: LLM call #1 (324 tokens, 0.8s) → decideix usar query_champions
├── Span: Tool: query_champions (12ms) → retorna llista
├── Span: LLM call #2 (518 tokens, 1.2s) → genera resposta final
└── Score: fidelity=1.0, tool_calls=1, cost=$0.0025
```

### Integrar LangFuse amb l'Agent

LangFuse s'integra com a "decorador" — envolta les crides existents sense canviar la logica:

```python
from langfuse import Langfuse
from langfuse.decorators import observe, langfuse_context

# Inicialitza el client LangFuse
langfuse = Langfuse(
    public_key="pk-lf-...",     # De les variables d'entorn
    secret_key="sk-lf-...",
    host="https://cloud.langfuse.com"  # O el teu servidor local
)

@observe()  # Aixo crea un trace automatic per cada crida
def run_agent(question: str) -> dict:
    # ... el teu bucle agent existent ...
    # LangFuse registra cada crida automaticament
    pass
```

> **Lectura recomanada (no bloquejant):**
> - [LangFuse Python SDK](https://langfuse.com/docs/sdk/python) — Documentacio oficial
> - [LangFuse Tracing](https://langfuse.com/docs/tracing) — Com funcionen els traces

---

## Activitat

### 1. Crear el Dataset d'Avaluacio (20 min)

```python
# fitxer: ai-python/src/agents/eval_dataset.py
# Dataset d'avaluacio per als agents d'EsportsPulse
# 15+ preguntes amb respostes de referencia

EVAL_DATASET = [
    # --- Agent Quantitatiu (8 preguntes) ---
    {
        "id": "quant-001",
        "question": "Quin champion te el win rate mes alt?",
        "agent": "quantitative",
        "expected_contains": ["Lux", "53.1"],
        "expected_not_contains": ["no tinc informacio"],
        "expected_tools": ["query_champions"],
        "max_tool_calls": 2
    },
    {
        "id": "quant-002",
        "question": "Dona'm les estadistiques de Jinx",
        "agent": "quantitative",
        "expected_contains": ["Jinx", "51.8", "adc"],
        "expected_not_contains": [],
        "expected_tools": ["get_champion_stats"],
        "max_tool_calls": 1
    },
    {
        "id": "quant-003",
        "question": "Quins supports tenen mes del 50% de win rate?",
        "agent": "quantitative",
        "expected_contains": ["Leona", "51.5"],
        "expected_not_contains": ["Thresh"],  # Thresh te 49.7
        "expected_tools": ["query_champions"],
        "max_tool_calls": 2
    },
    {
        "id": "quant-004",
        "question": "Compara Ahri i Lux al mid",
        "agent": "quantitative",
        "expected_contains": ["Ahri", "Lux", "52.3", "53.1"],
        "expected_not_contains": [],
        "expected_tools": ["get_champion_stats"],
        "max_tool_calls": 3
    },
    {
        "id": "quant-005",
        "question": "Quin champion te el pick rate mes alt?",
        "agent": "quantitative",
        "expected_contains": ["Kai'Sa", "15.2"],
        "expected_not_contains": [],
        "expected_tools": ["query_champions"],
        "max_tool_calls": 2
    },
    {
        "id": "quant-006",
        "question": "Quins champions de jungle hi ha?",
        "agent": "quantitative",
        "expected_contains": ["Lee Sin"],
        "expected_not_contains": [],
        "expected_tools": ["query_champions"],
        "max_tool_calls": 1
    },
    {
        "id": "quant-007",
        "question": "Quin es el millor restaurant de Barcelona?",
        "agent": "quantitative",
        "expected_contains": [],
        "expected_not_contains": [],
        "expected_tools": [],  # No hauria d'usar eines
        "max_tool_calls": 0,
        "expect_out_of_scope": True  # Hauria de rebutjar la pregunta
    },
    {
        "id": "quant-008",
        "question": "Quin champion te el ban rate mes alt?",
        "agent": "quantitative",
        "expected_contains": ["Darius", "12.3"],
        "expected_not_contains": [],
        "expected_tools": ["query_champions"],
        "max_tool_calls": 2
    },

    # --- Agent de Knowledge (7 preguntes) ---
    {
        "id": "know-001",
        "question": "Quins canvis ha tingut Ahri recentment?",
        "agent": "knowledge",
        "expected_contains": ["Ahri"],
        "expected_not_contains": [],
        "expected_tools": ["search_patches"],
        "max_tool_calls": 3
    },
    {
        "id": "know-002",
        "question": "Que va passar al patch 14.10?",
        "agent": "knowledge",
        "expected_contains": ["14.10"],
        "expected_not_contains": [],
        "expected_tools": ["get_patch_detail"],
        "max_tool_calls": 2
    },
    {
        "id": "know-003",
        "question": "Com ha canviat Ahri entre el patch 14.8 i el 14.12?",
        "agent": "knowledge",
        "expected_contains": ["14.8", "14.12", "Ahri"],
        "expected_not_contains": [],
        "expected_tools": ["search_patches"],
        "max_tool_calls": 4  # Pot necessitar 2-3 cerques
    },
    {
        "id": "know-004",
        "question": "Quins champions han rebut nerfs al patch 14.12?",
        "agent": "knowledge",
        "expected_contains": ["Lee Sin"],
        "expected_not_contains": [],
        "expected_tools": ["search_patches", "get_patch_detail"],
        "max_tool_calls": 3
    },
    {
        "id": "know-005",
        "question": "Quins canvis ha tingut Yasuo al patch 14.15?",
        "agent": "knowledge",
        "expected_contains": ["no tinc informacio"],  # Anti-hallucinacio
        "expected_not_contains": ["Yasuo va rebre"],  # No ha d'inventar
        "expected_tools": ["search_patches"],
        "max_tool_calls": 2
    },
    {
        "id": "know-006",
        "question": "Jinx va rebre buffs o nerfs recentment?",
        "agent": "knowledge",
        "expected_contains": ["Jinx", "buff"],
        "expected_not_contains": [],
        "expected_tools": ["search_patches"],
        "max_tool_calls": 2
    },
    {
        "id": "know-007",
        "question": "Quin champion de support va tenir canvis al patch 14.8?",
        "agent": "knowledge",
        "expected_contains": ["Thresh"],
        "expected_not_contains": [],
        "expected_tools": ["search_patches"],
        "max_tool_calls": 3
    }
]
```

### 2. Implementar els Evals amb pytest (30 min)

```python
# fitxer: ai-python/src/agents/test_agents.py
# Evals automatitzats per als agents d'EsportsPulse
# Executa amb: pytest test_agents.py -v

import pytest
import json
import time
from eval_dataset import EVAL_DATASET
from quantitative_agent import run_agent as run_quant_agent
from knowledge_agent import run_knowledge_agent

# --- Funcions d'avaluacio ---

def evaluate_answer(result: dict | str, eval_case: dict) -> dict:
    """Avalua una resposta d'agent contra el cas d'avaluacio.

    Args:
        result: Resposta de l'agent (string o dict amb 'answer')
        eval_case: Cas d'eval amb expected_contains, etc.

    Returns:
        Diccionari amb puntuacions per cada dimensio
    """
    # Extrau la resposta com a text
    if isinstance(result, dict):
        answer = result.get("answer", "")
        tool_calls = result.get("tool_calls", 0)
        total_tokens = result.get("total_tokens", 0)
    else:
        answer = result
        tool_calls = 0
        total_tokens = 0

    answer_lower = answer.lower()

    # --- Fidelitat: conte les paraules esperades? ---
    contains_score = 1.0
    if eval_case["expected_contains"]:
        matches = sum(
            1 for term in eval_case["expected_contains"]
            if term.lower() in answer_lower
        )
        contains_score = matches / len(eval_case["expected_contains"])

    # --- Absencia: NO conte paraules prohibides? ---
    not_contains_score = 1.0
    if eval_case.get("expected_not_contains"):
        violations = sum(
            1 for term in eval_case["expected_not_contains"]
            if term.lower() in answer_lower
        )
        if violations > 0:
            not_contains_score = 0.0

    # --- Eficiencia: ha usat un nombre raonable d'eines? ---
    efficiency_score = 1.0
    max_calls = eval_case.get("max_tool_calls", 5)
    if tool_calls > max_calls:
        efficiency_score = max(0, 1.0 - (tool_calls - max_calls) * 0.25)

    # --- Puntuacio global ---
    fidelity = contains_score * not_contains_score

    return {
        "id": eval_case["id"],
        "fidelity": fidelity,
        "efficiency": efficiency_score,
        "tool_calls": tool_calls,
        "total_tokens": total_tokens,
        "contains_score": contains_score,
        "not_contains_score": not_contains_score
    }


# --- Tests per l'Agent Quantitatiu ---

@pytest.fixture(scope="module")
def quant_results():
    """Executa l'agent quantitatiu per tots els casos del dataset.
    scope=module assegura que nomes s'executa un cop per tot el modul.
    """
    results = {}
    quant_cases = [c for c in EVAL_DATASET if c["agent"] == "quantitative"]

    for case in quant_cases:
        answer = run_quant_agent(case["question"], verbose=False)
        results[case["id"]] = {
            "answer": answer,
            "tool_calls": 0,  # Afegeix metriques si el teu agent les retorna
            "total_tokens": 0
        }
        time.sleep(1)  # Rate limiting basic

    return results


@pytest.fixture(scope="module")
def knowledge_results():
    """Executa l'agent de knowledge per tots els casos del dataset."""
    results = {}
    know_cases = [c for c in EVAL_DATASET if c["agent"] == "knowledge"]

    for case in know_cases:
        result = run_knowledge_agent(case["question"], verbose=False)
        results[case["id"]] = result
        time.sleep(1)

    return results


class TestQuantitativeAgent:
    """Tests per a l'Agent Quantitatiu."""

    def test_fidelity_above_threshold(self, quant_results):
        """La fidelitat global ha de ser >= 80%."""
        quant_cases = [c for c in EVAL_DATASET if c["agent"] == "quantitative"]
        scores = []

        for case in quant_cases:
            result = quant_results[case["id"]]
            evaluation = evaluate_answer(result, case)
            scores.append(evaluation["fidelity"])

        avg_fidelity = sum(scores) / len(scores)
        assert avg_fidelity >= 0.80, (
            f"Fidelitat de l'Agent Quantitatiu ({avg_fidelity:.2%}) "
            f"per sota del llindar (80%)"
        )

    def test_highest_winrate(self, quant_results):
        """Ha d'identificar Lux com el champion amb mes win rate."""
        result = quant_results["quant-001"]
        answer = result["answer"].lower() if isinstance(result, dict) else result.lower()
        assert "lux" in answer, "No ha identificat Lux com el champion amb mes win rate"

    def test_out_of_scope_rejection(self, quant_results):
        """Ha de rebutjar preguntes fora d'ambit."""
        result = quant_results["quant-007"]
        answer = result["answer"].lower() if isinstance(result, dict) else result.lower()
        # No hauria de parlar de restaurants
        assert "restaurant" not in answer or "ambit" in answer or "estadistiques" in answer


class TestKnowledgeAgent:
    """Tests per a l'Agent de Knowledge."""

    def test_fidelity_above_threshold(self, knowledge_results):
        """La fidelitat global ha de ser >= 80%."""
        know_cases = [c for c in EVAL_DATASET if c["agent"] == "knowledge"]
        scores = []

        for case in know_cases:
            result = knowledge_results[case["id"]]
            evaluation = evaluate_answer(result, case)
            scores.append(evaluation["fidelity"])

        avg_fidelity = sum(scores) / len(scores)
        assert avg_fidelity >= 0.80, (
            f"Fidelitat de l'Agent de Knowledge ({avg_fidelity:.2%}) "
            f"per sota del llindar (80%)"
        )

    def test_anti_hallucination(self, knowledge_results):
        """Ha de dir que no te informacio quan no la te."""
        result = knowledge_results["know-005"]
        answer = result["answer"].lower()
        assert "no" in answer and ("informacio" in answer or "dades" in answer), (
            "L'agent hauria de dir que no te informacio sobre Yasuo al patch 14.15"
        )

    def test_multi_step_comparison(self, knowledge_results):
        """Ha de fer multiples cerques per preguntes comparatives."""
        result = knowledge_results["know-003"]
        assert result["tool_calls"] >= 2, (
            f"Una comparacio entre patches hauria de necessitar 2+ crides, "
            f"pero nomes n'ha fet {result['tool_calls']}"
        )


class TestOverallMetrics:
    """Tests globals de rendiment."""

    def test_print_eval_summary(self, quant_results, knowledge_results):
        """Imprimeix un resum de totes les avaluacions (sempre passa)."""
        all_cases = EVAL_DATASET
        results = {}
        results.update(quant_results)
        results.update(knowledge_results)

        print("\n" + "="*60)
        print("RESUM D'AVALUACIO")
        print("="*60)

        for case in all_cases:
            if case["id"] in results:
                evaluation = evaluate_answer(results[case["id"]], case)
                status = "PASS" if evaluation["fidelity"] >= 0.8 else "FAIL"
                print(f"  [{status}] {case['id']}: "
                      f"fidelitat={evaluation['fidelity']:.0%} "
                      f"eines={evaluation['tool_calls']}")

        print("="*60)
```

### 3. Configurar LangFuse (20 min)

Registra't a [LangFuse Cloud](https://cloud.langfuse.com) (gratis per ús personal) o executa localment amb Docker:

```bash
# Opcio A: LangFuse Cloud (recomanat per simplicitat)
# 1. Registra't a https://cloud.langfuse.com
# 2. Crea un projecte "esportspulse-agents"
# 3. Copia les claus a .env

# Opcio B: LangFuse local amb Docker
docker run -d --name langfuse \
  -p 3000:3000 \
  -e DATABASE_URL="postgresql://postgres:postgres@host.docker.internal:5432/langfuse" \
  langfuse/langfuse
```

Afegeix les claus al `.env`:

```bash
# fitxer: .env (NO el comitis!)
LANGFUSE_PUBLIC_KEY=pk-lf-xxxxxxxx
LANGFUSE_SECRET_KEY=sk-lf-xxxxxxxx
LANGFUSE_HOST=https://cloud.langfuse.com
```

Instala el paquet:

```bash
pip install langfuse
```

Afegeix observabilitat a l'agent:

```python
# fitxer: ai-python/src/agents/knowledge_agent.py (modificacio)
# Afegeix al principi del fitxer:

import os
from langfuse import Langfuse
from langfuse.decorators import observe, langfuse_context

# Inicialitza LangFuse (llegeix les claus de les variables d'entorn)
langfuse = Langfuse()

# Afegeix @observe() a la funcio principal de l'agent
@observe(name="knowledge_agent")
def run_knowledge_agent(user_question: str, verbose: bool = True) -> dict:
    # ... (codi existent) ...

    # Dins el bucle, quan executem una eina, creem un span:
    # (dins el for block de tool_use)
    langfuse_context.update_current_observation(
        metadata={"tool_name": block.name, "tool_input": block.input}
    )
    # ... (resta del codi) ...

    # Abans de retornar, afegim les metriques com a score:
    langfuse_context.score_current_trace(
        name="tool_calls",
        value=tool_call_count
    )
    langfuse_context.score_current_trace(
        name="total_tokens",
        value=total_input_tokens + total_output_tokens
    )

    return result_dict
```

### 4. Afegir Evals a CI (15 min)

Crea o modifica el workflow de GitHub Actions:

```yaml
# fitxer: .github/workflows/agent-evals.yml
# Executa evals d'agents a cada PR — bloqueja merge si fidelitat < 80%

name: Agent Evals

on:
  pull_request:
    paths:
      - 'ai-python/src/agents/**'

jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install dependencies
        run: |
          pip install anthropic langfuse pytest

      - name: Run agent evals
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          LANGFUSE_PUBLIC_KEY: ${{ secrets.LANGFUSE_PUBLIC_KEY }}
          LANGFUSE_SECRET_KEY: ${{ secrets.LANGFUSE_SECRET_KEY }}
          LANGFUSE_HOST: https://cloud.langfuse.com
        run: |
          cd ai-python/src/agents
          pytest test_agents.py -v --tb=short

      # Si els evals fallen (fidelitat < 80%), el job falla
      # i la PR no es pot fer merge
```

---

## Checklist de Lliurament

- [ ] `eval_dataset.py` conte 15+ preguntes amb respostes de referencia
- [ ] `test_agents.py` executa evals i comprova fidelitat >= 80%
- [ ] `pytest test_agents.py -v` passa (o identifica clarament que falla)
- [ ] LangFuse configurat (cloud o local) amb traces visibles al dashboard
- [ ] Pots veure a LangFuse: latencia per pas, tokens per crida, crides a eines
- [ ] `.github/workflows/agent-evals.yml` creat i configurat
- [ ] Commit: `feat(agents): add eval dataset, pytest evals and LangFuse observability`
