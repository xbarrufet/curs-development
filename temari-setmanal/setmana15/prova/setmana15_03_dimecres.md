# Setmana 15 — Dimecres: Evals: Mesurar la Qualitat del Retrieval

## Objectiu del Dia

Crear un sistema d'avaluació (evals) automatitzat per mesurar la qualitat del pipeline RAG. Al final del dia has de tenir 10 preguntes amb ground truth, mètriques d'accuracy i faithfulness, i tests integrats al CI amb pytest.

---

## Teoria

### Per Què "Vibes-Based Testing" No Funciona

Fins ara has provat el RAG manualment: fas una pregunta, llegeixes la resposta, dius "pinta bé". Això és vibes-based testing i és perillós:

```
┌────────────────────────────────────────────────┐
│          VIBES vs EVALS SISTEMÀTIQUES          │
│                                                 │
│  Vibes:                                         │
│  "Li he preguntat 3 coses i ha respost bé"     │
│  → No escala. No és reproduïble. No detecta     │
│    regressions quan canvies el prompt.           │
│                                                 │
│  Evals:                                         │
│  10 preguntes × 3 mètriques × cada commit      │
│  → Detecta quan un canvi empitjora el sistema.  │
│    Dóna números concrets. S'integra al CI.      │
└────────────────────────────────────────────────┘
```

El problema real: canvies el system prompt per millorar un cas i, sense adonar-te'n, empitjores tres casos més. Sense evals, no ho sabràs fins que un usuari es queixi.

### Les Tres Mètriques Fonamentals del RAG

**1. Accuracy (Exactitud de la resposta)**
La resposta generada coincideix amb la resposta esperada?

```
Pregunta: "Quin canvi va rebre Jinx al patch 14.5?"
Ground truth: "Reducció d'AD base de 59 a 55"
Resposta LLM: "Jinx va rebre una reducció d'atac base de 59 a 55"
Accuracy: ✅ (conté la informació correcta)
```

**2. Faithfulness (Fidelitat a les fonts)**
La resposta cita correctament els chunks? No afegeix informació inventada?

```
Resposta: "Jinx va rebre reducció d'AD [Font 1] i un buff de velocitat [Font 2]"
Font 1: "Jinx base AD 59→55" ✅ Correcte
Font 2: "Caitlyn range increased..." ❌ No parla de velocitat de Jinx!
Faithfulness: PARCIAL (una cita correcta, una incorrecta)
```

**3. Relevance (Rellevància del retrieval)**
Els chunks recuperats són rellevants per a la pregunta?

```
Pregunta: "Canvis a Jinx?"
Chunk 1: "Jinx base AD reduced..." (score: 0.92) ✅ Rellevant
Chunk 2: "Jinx passive adjusted..." (score: 0.88) ✅ Rellevant  
Chunk 3: "Dragon spawn timer changed..." (score: 0.71) ❌ Irrellevant
Relevance: 2/3 = 67%
```

### Estructura d'un Dataset d'Evals

Un bon eval dataset és una llista de preguntes amb respostes esperades:

```python
# Cada entrada té: pregunta, resposta esperada, chunks esperats
EVAL_DATASET = [
    {
        "question": "Quin canvi va rebre Jinx al patch 14.5?",
        "expected_answer_contains": ["AD base", "59", "55"],
        "expected_patch": "14.5",
        "category": "champion_change"
    },
    # ... 9 més
]
```

Important: no busquem coincidència exacta de text (l'LLM mai repetirà paraula per paraula). Busquem que la resposta **contingui els conceptes clau**.

### Mètriques Agregades

Després d'executar les 10 preguntes, calcules:

| Mètrica          | Càlcul                          | Objectiu |
|------------------|---------------------------------|----------|
| Accuracy         | Respostes correctes / Total     | >= 80%   |
| Faithfulness     | Cites vàlides / Total cites     | >= 90%   |
| Retrieval Prec.  | Chunks rellevants / Chunks tot. | >= 70%   |
| Refusal Rate     | "No tinc info" correctes / Tot. | >= 90%   |

---

## Activitat

### 1. Crear el dataset d'evaluació (25 min)

Crea `ai-python/src/retrieval/eval_dataset.py`:

```python
"""
Dataset d'avaluació per al pipeline RAG d'EsportsPulse.
Conté 10 preguntes amb ground truth per mesurar qualitat.

Cada entrada defineix:
- question: La pregunta a fer al sistema
- expected_keywords: Paraules clau que la resposta ha de contenir
- expected_patch: El patch d'on hauria de venir la informació
- category: Tipus de pregunta per agrupar mètriques
- should_refuse: True si el sistema NO hauria de respondre (pregunta trampa)
"""

EVAL_DATASET = [
    # --- Preguntes amb resposta esperada ---
    {
        "id": "eval_01",
        "question": "Quins canvis va rebre Jinx recentment?",
        "expected_keywords": ["AD", "base", "reduit", "reducc"],
        "expected_patch": "14.5",
        "category": "champion_nerf",
        "should_refuse": False
    },
    {
        "id": "eval_02",
        "question": "Hi va haver algun canvi a objectes d'ADC?",
        "expected_keywords": ["Infinity Edge", "crític", "critical"],
        "expected_patch": "14.5",
        "category": "item_change",
        "should_refuse": False
    },
    {
        "id": "eval_03",
        "question": "Quin campió suport va rebre un buff?",
        "expected_keywords": ["suport", "curació", "escud"],
        "expected_patch": "14.5",
        "category": "champion_buff",
        "should_refuse": False
    },
    {
        "id": "eval_04",
        "question": "Hi ha hagut canvis al sistema de drac?",
        "expected_keywords": ["drac", "dragon", "spawn"],
        "expected_patch": "14.5",
        "category": "system_change",
        "should_refuse": False
    },
    {
        "id": "eval_05",
        "question": "Quins nerfs van rebre els assassins?",
        "expected_keywords": ["assassí", "dany", "damage"],
        "expected_patch": "14.5",
        "category": "champion_nerf",
        "should_refuse": False
    },
    {
        "id": "eval_06",
        "question": "Quin va ser el canvi més important del patch 14.5?",
        "expected_keywords": [],  # Resposta oberta, validem que citi font
        "expected_patch": "14.5",
        "category": "summary",
        "should_refuse": False
    },
    {
        "id": "eval_07",
        "question": "Hi va haver algun bugfix al patch?",
        "expected_keywords": ["bug", "fix", "corregit", "solucion"],
        "expected_patch": "14.5",
        "category": "bugfix",
        "should_refuse": False
    },
    # --- Preguntes trampa (should_refuse = True) ---
    {
        "id": "eval_08",
        "question": "Quin campió nou es va llançar al patch 14.5?",
        "expected_keywords": [],
        "expected_patch": None,
        "category": "trick_new_champion",
        "should_refuse": True
    },
    {
        "id": "eval_09",
        "question": "Quants jugadors actius té League of Legends?",
        "expected_keywords": [],
        "expected_patch": None,
        "category": "trick_out_of_scope",
        "should_refuse": True
    },
    {
        "id": "eval_10",
        "question": "Quins canvis va rebre Valorant al patch 14.5?",
        "expected_keywords": [],
        "expected_patch": None,
        "category": "trick_wrong_game",
        "should_refuse": True
    },
]
```

### 2. Implementar el motor d'evals (30 min)

Crea `ai-python/src/retrieval/eval_runner.py`:

```python
"""
Motor d'avaluació per al pipeline RAG.
Executa el dataset d'evals i calcula mètriques de qualitat.
"""

from dataclasses import dataclass, field
from retrieval.rag_engine import ask_with_validation, retrieve_chunks
from retrieval.eval_dataset import EVAL_DATASET


@dataclass
class EvalResult:
    """Resultat de l'avaluació d'una sola pregunta."""
    eval_id: str
    question: str
    category: str
    # Mètriques individuals
    accuracy_pass: bool = False        # La resposta conté els keywords esperats?
    faithfulness_pass: bool = False     # Les cites són vàlides?
    retrieval_relevant: bool = False    # Els chunks són del patch esperat?
    refusal_correct: bool = False       # Ha rebutjat correctament (si calia)?
    # Detalls
    answer: str = ""
    chunks_used: int = 0
    warnings: list = field(default_factory=list)


def check_accuracy(answer: str, expected_keywords: list[str]) -> bool:
    """
    Verifica si la resposta conté les paraules clau esperades.
    No busquem coincidència exacta, sinó que els conceptes clau hi siguin.
    
    Args:
        answer: La resposta generada per l'LLM
        expected_keywords: Llista de paraules que han d'aparèixer
    Returns:
        True si TOTES les paraules clau són presents (o la llista és buida)
    """
    if not expected_keywords:
        # Si no hi ha keywords definits, acceptem qualsevol resposta amb cites
        return "[Font" in answer
    
    answer_lower = answer.lower()
    # Comprovem que almenys una variant de cada keyword hi sigui
    for keyword in expected_keywords:
        if keyword.lower() not in answer_lower:
            return False
    return True


def check_faithfulness(validation_result: dict) -> bool:
    """
    Verifica la fidelitat de la resposta usant el validador anti-al·lucinació.
    
    Args:
        validation_result: El resultat de validate_response()
    Returns:
        True si no hi ha warnings de cites invàlides
    """
    # Si la validació és OK o les warnings no són sobre cites
    if validation_result["valid"]:
        return True
    
    # Comprovar si les warnings són crítiques
    critical_warnings = [
        w for w in validation_result.get("warnings", [])
        if "invàlides" in w or "AL·LUCINACIÓ" in w
    ]
    return len(critical_warnings) == 0


def check_retrieval_relevance(
    chunks: list[dict], expected_patch: str | None
) -> bool:
    """
    Verifica que els chunks recuperats provenen del patch esperat.
    
    Args:
        chunks: Els chunks retornats pel retrieval
        expected_patch: El número de patch esperat (None si no aplica)
    Returns:
        True si almenys un chunk és del patch esperat
    """
    if expected_patch is None:
        # Per a preguntes trampa, és OK no trobar chunks
        return True
    
    # Almenys un chunk ha de ser del patch esperat
    for chunk in chunks:
        if chunk["metadata"].get("patch") == expected_patch:
            return True
    return False


def check_refusal(answer: str, should_refuse: bool) -> bool:
    """
    Verifica que el sistema ha rebutjat (o no) correctament.
    
    Args:
        answer: La resposta generada
        should_refuse: True si el sistema hauria de dir "no tinc info"
    Returns:
        True si el comportament és correcte
    """
    refusal_phrases = [
        "no tinc informació",
        "no es troba",
        "no he trobat",
        "no disposo",
    ]
    
    is_refusal = any(phrase in answer.lower() for phrase in refusal_phrases)
    
    if should_refuse:
        return is_refusal  # Hauria de rebutjar i ha rebutjat?
    else:
        return not is_refusal  # No hauria de rebutjar i no ho ha fet?


def run_eval(dataset: list[dict] = None) -> list[EvalResult]:
    """
    Executa l'avaluació completa del pipeline RAG.
    
    Args:
        dataset: Llista d'evals a executar (per defecte, EVAL_DATASET)
    Returns:
        Llista de resultats d'avaluació
    """
    if dataset is None:
        dataset = EVAL_DATASET
    
    results = []
    
    for entry in dataset:
        # Executar el pipeline RAG amb validació
        rag_result = ask_with_validation(entry["question"])
        chunks = retrieve_chunks(entry["question"])
        
        # Avaluar cada mètrica
        eval_result = EvalResult(
            eval_id=entry["id"],
            question=entry["question"],
            category=entry["category"],
            answer=rag_result["answer"],
            chunks_used=rag_result["chunks_used"],
        )
        
        # Mètriques
        eval_result.accuracy_pass = check_accuracy(
            rag_result["answer"], entry["expected_keywords"]
        )
        eval_result.faithfulness_pass = check_faithfulness(
            rag_result["validation"]
        )
        eval_result.retrieval_relevant = check_retrieval_relevance(
            chunks, entry["expected_patch"]
        )
        eval_result.refusal_correct = check_refusal(
            rag_result["answer"], entry["should_refuse"]
        )
        eval_result.warnings = rag_result["validation"].get("warnings", [])
        
        results.append(eval_result)
    
    return results


def print_eval_report(results: list[EvalResult]) -> dict:
    """
    Genera i imprimeix un informe resum de les avaluacions.
    
    Args:
        results: Llista de resultats d'avaluació
    Returns:
        Diccionari amb les mètriques agregades
    """
    total = len(results)
    
    # Calcular mètriques agregades
    accuracy = sum(1 for r in results if r.accuracy_pass) / total
    faithfulness = sum(1 for r in results if r.faithfulness_pass) / total
    relevance = sum(1 for r in results if r.retrieval_relevant) / total
    refusal = sum(1 for r in results if r.refusal_correct) / total
    
    print("=" * 60)
    print("INFORME D'AVALUACIÓ RAG — EsportsPulse")
    print("=" * 60)
    
    # Resultats individuals
    for r in results:
        status = "✅" if all([
            r.accuracy_pass, r.faithfulness_pass,
            r.retrieval_relevant, r.refusal_correct
        ]) else "❌"
        print(f"\n{status} [{r.eval_id}] {r.question[:50]}...")
        print(f"   Accuracy: {'✅' if r.accuracy_pass else '❌'}  "
              f"Faithfulness: {'✅' if r.faithfulness_pass else '❌'}  "
              f"Relevance: {'✅' if r.retrieval_relevant else '❌'}  "
              f"Refusal: {'✅' if r.refusal_correct else '❌'}")
        if r.warnings:
            for w in r.warnings:
                print(f"   ⚠️  {w}")
    
    # Resum
    print(f"\n{'=' * 60}")
    print("MÈTRIQUES AGREGADES")
    print(f"{'─' * 60}")
    print(f"  Accuracy:         {accuracy:.0%} (objectiu: >= 80%)")
    print(f"  Faithfulness:     {faithfulness:.0%} (objectiu: >= 90%)")
    print(f"  Retrieval Prec.:  {relevance:.0%} (objectiu: >= 70%)")
    print(f"  Refusal Rate:     {refusal:.0%} (objectiu: >= 90%)")
    print(f"{'=' * 60}")
    
    return {
        "accuracy": accuracy,
        "faithfulness": faithfulness,
        "relevance": relevance,
        "refusal_rate": refusal
    }
```

### 3. Escriure tests pytest (25 min)

Crea `ai-python/tests/test_rag_evals.py`:

```python
"""
Tests pytest per al pipeline RAG.
S'integren al CI per detectar regressions automàticament.

Executar: pytest tests/test_rag_evals.py -v
"""

import pytest
from retrieval.eval_runner import run_eval, print_eval_report
from retrieval.eval_dataset import EVAL_DATASET


# Marca els tests com a "slow" perquè fan crides reals a l'LLM
# Al CI pots executar-los amb: pytest -m "slow" --timeout=120
pytestmark = pytest.mark.slow


@pytest.fixture(scope="module")
def eval_results():
    """
    Executa totes les evals una sola vegada per al mòdul.
    Scope="module" perquè les crides a l'LLM són cares i lentes.
    """
    results = run_eval(EVAL_DATASET)
    return results


@pytest.fixture(scope="module")
def eval_metrics(eval_results):
    """Calcula les mètriques agregades."""
    return print_eval_report(eval_results)


class TestRAGAccuracy:
    """Tests d'exactitud: les respostes contenen la informació correcta."""
    
    def test_overall_accuracy_above_threshold(self, eval_metrics):
        """L'accuracy global ha de ser >= 80%."""
        assert eval_metrics["accuracy"] >= 0.8, (
            f"Accuracy {eval_metrics['accuracy']:.0%} per sota del 80%. "
            "Revisa el retrieval o el prompt."
        )
    
    def test_no_wrong_answers(self, eval_results):
        """Cap pregunta amb resposta esperada hauria de fallar en accuracy."""
        # Filtrar només preguntes que NO són trampa
        real_questions = [
            r for r in eval_results if not any(
                e["should_refuse"] for e in EVAL_DATASET 
                if e["id"] == r.eval_id
            )
        ]
        failed = [r for r in real_questions if not r.accuracy_pass]
        assert len(failed) == 0, (
            f"{len(failed)} preguntes amb accuracy incorrecta: "
            f"{[r.eval_id for r in failed]}"
        )


class TestRAGFaithfulness:
    """Tests de fidelitat: les respostes citen correctament les fonts."""
    
    def test_overall_faithfulness_above_threshold(self, eval_metrics):
        """La faithfulness global ha de ser >= 90%."""
        assert eval_metrics["faithfulness"] >= 0.9, (
            f"Faithfulness {eval_metrics['faithfulness']:.0%} per sota del 90%. "
            "L'LLM està generant cites invàlides."
        )


class TestRAGRetrieval:
    """Tests de retrieval: els chunks recuperats són rellevants."""
    
    def test_overall_relevance_above_threshold(self, eval_metrics):
        """La retrieval precision ha de ser >= 70%."""
        assert eval_metrics["relevance"] >= 0.7, (
            f"Relevance {eval_metrics['relevance']:.0%} per sota del 70%. "
            "El retrieval no troba els chunks correctes."
        )


class TestAntiHallucination:
    """Tests anti-al·lucinació: el sistema rebutja preguntes sense context."""
    
    def test_overall_refusal_rate(self, eval_metrics):
        """La refusal rate ha de ser >= 90%."""
        assert eval_metrics["refusal_rate"] >= 0.9, (
            f"Refusal rate {eval_metrics['refusal_rate']:.0%} per sota del 90%. "
            "L'LLM al·lucina en preguntes que hauria de rebutjar."
        )
    
    def test_all_trick_questions_refused(self, eval_results):
        """TOTES les preguntes trampa han de ser rebutjades."""
        trick_ids = {
            e["id"] for e in EVAL_DATASET if e["should_refuse"]
        }
        trick_results = [r for r in eval_results if r.eval_id in trick_ids]
        
        failed = [r for r in trick_results if not r.refusal_correct]
        assert len(failed) == 0, (
            f"AL·LUCINACIÓ DETECTADA en {len(failed)} preguntes trampa: "
            f"{[r.eval_id for r in failed]}"
        )
```

### 4. Integrar al CI (10 min)

Afegeix al teu `Makefile` o `pyproject.toml`:

```toml
# pyproject.toml — secció de pytest
[tool.pytest.ini_options]
markers = [
    "slow: tests que fan crides a APIs externes (LLM, Qdrant)"
]
# Per defecte, NO executem tests slow al CI ràpid
# Comanda CI completa: pytest -m "slow" --timeout=180
```

```bash
# Executar tots els tests (incloent evals)
pytest tests/test_rag_evals.py -v --timeout=180

# Executar només tests ràpids (sense crides LLM)
pytest tests/ -v -m "not slow"
```

---

## Checklist de Lliurament

- [ ] `eval_dataset.py` amb 10 preguntes (7 reals + 3 trampa)
- [ ] `eval_runner.py` amb mètriques: accuracy, faithfulness, relevance, refusal rate
- [ ] `test_rag_evals.py` amb tests pytest que validen les 4 mètriques
- [ ] Accuracy >= 80%, Faithfulness >= 90%, Relevance >= 70%, Refusal >= 90%
- [ ] Tests integrats al CI (marcats com `@pytest.mark.slow`)
- [ ] Commit: `test(rag): add eval dataset and automated quality metrics`
