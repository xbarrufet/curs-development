# Setmana 16 — Dimecres: L'Agent de Knowledge: Connectar amb el Retrieval

## Objectiu del Dia

Construir un segon agent — l'Agent de Knowledge — que connecta amb el sistema de retrieval que vas crear a S15 (Qdrant + wiki + anti-hallucinacio). A diferencia de l'Agent Quantitatiu (dades estructurades), aquest agent treballa amb informacio no estructurada: patch notes, canvis de meta, histories de champions. L'agent decideix QUAN buscar, QUE buscar, i pot fer multiples cerques en una conversa.

---

## Teoria

### L'Agent de Knowledge vs L'Agent Quantitatiu

| | Agent Quantitatiu (dimarts) | Agent de Knowledge (avui) |
|---|---|---|
| **Tipus de dades** | Estructurades (numeros, stats) | No estructurades (text, patch notes) |
| **Eines** | `query_champions`, `get_stats` | `search_patches`, `get_patch_detail` |
| **Complexitat** | 1 crida = 1 resposta | Pot necessitar 2-3 cerques combinades |
| **Font** | Diccionari Python | Sistema de retrieval (Qdrant + embeddings) |
| **Risc** | Dades incorrectes | Hallucinacio (inventar informacio) |

### Eines que Connecten amb el Retrieval

Les eines de l'Agent de Knowledge son un "pont" entre l'agent i el sistema RAG de S15:

```python
# L'eina search_patches crida al teu sistema de retrieval existent
# No reinventem res — reutilitzem el que ja funciona

TOOL_SEARCH_PATCHES = {
    "name": "search_patches",
    "description": (
        "Cerca informacio als patch notes de League of Legends. "
        "Retorna els fragments mes rellevants ordenats per similitud semantica. "
        "Usa-la per trobar informacio sobre canvis, nerfs, buffs i ajustos de champions."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": (
                    "Cerca en llenguatge natural. Ex: 'nerfs a Ahri al patch 14.10' "
                    "o 'canvis a objectes de support'"
                )
            },
            "top_k": {
                "type": "integer",
                "description": "Nombre de resultats (defecte 3, maxim 10)",
                "default": 3
            }
        },
        "required": ["query"]
    }
}
```

**Clau:** L'eina `search_patches` NO busca per paraules clau — usa embeddings (el sistema que ja tens de S15). Aixo vol dir que "nerfs a Ahri" i "han debilitat Ahri" retornaran resultats similars.

### Multi-Step Reasoning: L'Agent que Pensa

La gran diferencia d'avui: l'agent pot necessitar MULTIPLES cerques per respondre una pregunta.

```
Pregunta: "Com ha canviat Ahri entre el patch 14.8 i el 14.12?"

Pas 1: search_patches("canvis Ahri patch 14.8")  → Resultats sobre patch 14.8
Pas 2: search_patches("canvis Ahri patch 14.12") → Resultats sobre patch 14.12
Pas 3: Claude compara els resultats i sintetitza → Resposta coherent
```

El model DECIDEIX sol que necessita dues cerques. Tu nomes has de tenir el bucle implementat (que ja tens de dimarts).

### Anti-Hallucinacio: El Guardrail Mes Important

Amb text no estructurat, el risc de hallucinacio es alt. El system prompt ha d'incloure guardrails explicits:

```python
SYSTEM_PROMPT = """Ets l'Agent de Knowledge d'EsportsPulse. El teu rol es respondre
preguntes sobre patch notes, canvis de meta i histories de champions.

REGLES CRITIQUES (anti-hallucinacio):
1. MAI invents informacio. Si no la trobes amb les eines, digues "No tinc
   informacio sobre aixo als patch notes indexats"
2. SEMPRE cita la font: indica de quin patch prové la informacio
3. Si els resultats de la cerca no son rellevants a la pregunta, digues-ho
4. Distingeix entre FETS (dades del patch) i OPINIONS (la teva interpretacio)
5. Si l'usuari pregunta per un patch que no tens indexat, digues-ho

EINES:
- search_patches: Cerca semantica als patch notes indexats
- get_patch_detail: Detall complet d'un patch concret

ESTRATEGIA:
- Preguntes sobre un champion → search_patches amb el nom del champion
- Preguntes comparatives → multiples cerques (una per cada element a comparar)
- Preguntes sobre un patch concret → get_patch_detail amb l'ID del patch
- SEMPRE revisa que els resultats de la cerca son rellevants abans de respondre"""
```

> **Lectura recomanada (no bloquejant):**
> - [Building effective agents — Anthropic](https://docs.anthropic.com/en/docs/build-with-claude/agentic) — Seccio "Agentic patterns"
> - [RAG and agents](https://docs.anthropic.com/en/docs/build-with-claude/citations) — Com citar fonts amb Claude

---

## Activitat

### 1. Preparar les Dades de Patch Notes (15 min)

Si el teu sistema de retrieval de S15 esta operatiu, el reutilitzaras directament. Si no, crea dades de prova:

```python
# fitxer: ai-python/src/agents/patch_data.py
# Dades simulades de patch notes per a l'Agent de Knowledge
# En un sistema complet, aixo es reemplacaria per crides al retrieval real (Qdrant)

PATCH_NOTES_DB = [
    {
        "id": "patch-14.8",
        "version": "14.8",
        "date": "2024-04-17",
        "changes": [
            {
                "champion": "Ahri",
                "type": "nerf",
                "detail": "Charm (E): durada del stun reduida de 1.4s a 1.2s. "
                          "Orb of Deception (Q): dany base reduit de 80-200 a 70-190."
            },
            {
                "champion": "Jinx",
                "type": "buff",
                "detail": "Switcheroo! (Q): velocitat d'atac en forma Pow-Pow "
                          "augmentada de 30-70% a 35-75%. Flame Chompers (E): "
                          "temps d'armament reduit de 0.7s a 0.5s."
            },
            {
                "champion": "Thresh",
                "type": "adjust",
                "detail": "Dark Passage (W): shield augmentat de 60-180 a 70-190, "
                          "pero cooldown augmentat de 22-16s a 24-18s."
            }
        ]
    },
    {
        "id": "patch-14.10",
        "version": "14.10",
        "date": "2024-05-15",
        "changes": [
            {
                "champion": "Ahri",
                "type": "buff",
                "detail": "Fox-Fire (W): dany per foc augmentat de 60-160 a 70-170. "
                          "Velocitat de moviment passiva augmentada de 20% a 25%."
            },
            {
                "champion": "Darius",
                "type": "nerf",
                "detail": "Hemorrhage (passiva): dany per stack reduit de 13-30 a "
                          "10-27. Noxian Guillotine (R): cooldown augmentat de "
                          "120-80s a 120-100s."
            },
            {
                "champion": "Lux",
                "type": "buff",
                "detail": "Final Spark (R): cooldown reduit de 80-50s a 70-40s. "
                          "Illumination (passiva): dany bonus augmentat un 10%."
            }
        ]
    },
    {
        "id": "patch-14.12",
        "version": "14.12",
        "date": "2024-06-12",
        "changes": [
            {
                "champion": "Ahri",
                "type": "adjust",
                "detail": "Spirit Rush (R): carregues reduides de 3 a 2, pero dany "
                          "per carrega augmentat de 60-180 a 80-220. Essence Theft "
                          "(passiva): curacio augmentada un 15%."
            },
            {
                "champion": "Lee Sin",
                "type": "nerf",
                "detail": "Safeguard (W): shield reduit de 55-175 a 45-155. "
                          "Dragon's Rage (R): dany base reduit de 175-475 a 150-450."
            },
            {
                "champion": "Kai'Sa",
                "type": "buff",
                "detail": "Icathian Rain (Q): nombre de projectils augmentat de "
                          "6 a 7 contra objectius aillats. Supercharge (E): "
                          "velocitat d'atac bonus augmentada un 5%."
            }
        ]
    }
]


def search_patches(query: str, top_k: int = 3) -> list:
    """Cerca semantica simulada als patch notes.

    En un sistema real, aquesta funcio cridaria al Qdrant amb embeddings.
    Aqui fem una cerca simple per paraules clau com a substitut.

    Args:
        query: Cerca en llenguatge natural
        top_k: Nombre de resultats a retornar

    Returns:
        Llista de canvis rellevants amb metadades del patch
    """
    query_lower = query.lower()
    results = []

    for patch in PATCH_NOTES_DB:
        for change in patch["changes"]:
            # Cerca simple per paraules — en produccio, seria embedding similarity
            text = f"{change['champion']} {change['type']} {change['detail']}".lower()

            # Puntuacio basica: quantes paraules de la query apareixen al text
            score = sum(1 for word in query_lower.split() if word in text)

            if score > 0:
                results.append({
                    "patch_version": patch["version"],
                    "patch_date": patch["date"],
                    "champion": change["champion"],
                    "change_type": change["type"],
                    "detail": change["detail"],
                    "relevance_score": score
                })

    # Ordena per rellevancia i retorna els top_k
    results.sort(key=lambda x: x["relevance_score"], reverse=True)
    return results[:top_k]


def get_patch_detail(patch_id: str) -> dict | None:
    """Obte el detall complet d'un patch.

    Args:
        patch_id: ID del patch (ex: 'patch-14.10')

    Returns:
        Diccionari amb totes les dades del patch, o None si no existeix
    """
    for patch in PATCH_NOTES_DB:
        if patch["id"] == patch_id:
            return patch
    return None
```

### 2. Implementar l'Agent de Knowledge (35 min)

```python
# fitxer: ai-python/src/agents/knowledge_agent.py
# Agent de Knowledge: respon preguntes sobre patch notes i canvis de meta
# Connecta amb el sistema de retrieval (simulat o real de S15)

import json
import anthropic
from patch_data import search_patches, get_patch_detail

# --- Eines ---

TOOLS = [
    {
        "name": "search_patches",
        "description": (
            "Cerca informacio als patch notes de League of Legends. "
            "Retorna fragments rellevants amb la versio del patch i data. "
            "Usa-la per trobar canvis de champions, nerfs, buffs i ajustos."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "Cerca en llenguatge natural"
                },
                "top_k": {
                    "type": "integer",
                    "description": "Nombre de resultats (defecte 3)",
                    "default": 3
                }
            },
            "required": ["query"]
        }
    },
    {
        "name": "get_patch_detail",
        "description": (
            "Obte el detall complet d'un patch especific amb tots els canvis. "
            "Usa-la quan l'usuari pregunta per un patch concret."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "patch_id": {
                    "type": "string",
                    "description": "ID del patch (ex: 'patch-14.10')"
                }
            },
            "required": ["patch_id"]
        }
    }
]

# --- System prompt amb guardrails anti-hallucinacio ---

SYSTEM_PROMPT = """Ets l'Agent de Knowledge d'EsportsPulse. El teu rol es respondre
preguntes sobre patch notes, canvis de meta i histories de champions de League of Legends.

REGLES CRITIQUES:
1. MAI invents informacio. Si no la trobes amb les eines, digues:
   "No tinc informacio sobre aixo als patch notes indexats"
2. SEMPRE cita la font: indica la versio del patch i la data
3. Si els resultats de la cerca NO son rellevants, digues-ho — no forcis una resposta
4. Distingeix FETS (dades del patch) d'INTERPRETACIO (la teva analisi)
5. Per comparacions entre patches, fes cerques separades per cada un
6. Respon en catala

ESTRATEGIA:
- Pregunta sobre un champion → search_patches amb el nom del champion
- Pregunta comparativa → multiples search_patches (un per element)
- Pregunta sobre un patch concret → get_patch_detail amb l'ID
- Pregunta d'evolucio temporal → multiples cerques per cada patch"""


def execute_tool(tool_name: str, tool_input: dict) -> str:
    """Executa una eina i retorna el resultat com a JSON.

    Punt central de control — aqui pots afegir logging, cache,
    o redirigir al sistema de retrieval real de S15.
    """
    if tool_name == "search_patches":
        result = search_patches(
            query=tool_input["query"],
            top_k=tool_input.get("top_k", 3)
        )
        if not result:
            return json.dumps({"message": "Cap resultat trobat per aquesta cerca"})
        return json.dumps(result, ensure_ascii=False)

    elif tool_name == "get_patch_detail":
        result = get_patch_detail(tool_input["patch_id"])
        if result is None:
            return json.dumps({"error": f"Patch '{tool_input['patch_id']}' no trobat"})
        return json.dumps(result, ensure_ascii=False)

    else:
        return json.dumps({"error": f"Eina desconeguda: {tool_name}"})


def run_knowledge_agent(user_question: str, verbose: bool = True) -> dict:
    """Executa l'Agent de Knowledge i retorna resposta + metriques.

    Args:
        user_question: Pregunta de l'usuari
        verbose: Mostra cada pas del bucle

    Returns:
        Diccionari amb 'answer', 'tool_calls' (nombre), 'total_tokens',
        i 'tools_used' (llista de crides)
    """
    client = anthropic.Anthropic()
    messages = [{"role": "user", "content": user_question}]

    # Metriques per avaluacio
    tool_call_count = 0
    total_input_tokens = 0
    total_output_tokens = 0
    tools_used = []

    max_steps = 10
    step = 0

    while step < max_steps:
        step += 1

        if verbose:
            print(f"\n--- Pas {step} ---")

        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            system=SYSTEM_PROMPT,
            tools=TOOLS,
            messages=messages
        )

        # Acumula tokens
        total_input_tokens += response.usage.input_tokens
        total_output_tokens += response.usage.output_tokens

        if response.stop_reason == "tool_use":
            messages.append({"role": "assistant", "content": response.content})

            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    tool_call_count += 1
                    tools_used.append({
                        "tool": block.name,
                        "input": block.input,
                        "step": step
                    })

                    if verbose:
                        print(f"Eina: {block.name}")
                        print(f"  Input: {json.dumps(block.input, ensure_ascii=False)}")

                    result = execute_tool(block.name, block.input)

                    if verbose:
                        print(f"  Resultat: {result[:150]}...")

                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result
                    })

            messages.append({"role": "user", "content": tool_results})

        else:
            # Resposta final
            answer = ""
            for block in response.content:
                if hasattr(block, "text"):
                    answer += block.text

            return {
                "answer": answer,
                "tool_calls": tool_call_count,
                "total_tokens": total_input_tokens + total_output_tokens,
                "input_tokens": total_input_tokens,
                "output_tokens": total_output_tokens,
                "tools_used": tools_used
            }

    return {
        "answer": "ERROR: Maxim de passos superat",
        "tool_calls": tool_call_count,
        "total_tokens": total_input_tokens + total_output_tokens,
        "input_tokens": total_input_tokens,
        "output_tokens": total_output_tokens,
        "tools_used": tools_used
    }


# --- Punt d'entrada ---
if __name__ == "__main__":
    preguntes = [
        # Pregunta simple (1 cerca)
        "Quins canvis ha tingut Ahri recentment?",
        # Pregunta comparativa (multiples cerques)
        "Com ha canviat Ahri entre el patch 14.8 i el 14.12?",
        # Pregunta sobre un patch concret
        "Que va passar al patch 14.10?",
        # Pregunta multi-champion
        "Quins champions de mid han rebut buffs recentment?",
        # Pregunta sense resposta (test anti-hallucinacio)
        "Quins canvis ha tingut Yasuo al patch 14.15?",
    ]

    for pregunta in preguntes:
        print(f"\n{'='*60}")
        print(f"PREGUNTA: {pregunta}")
        print('='*60)

        result = run_knowledge_agent(pregunta)

        print(f"\nRESPOSTA:\n{result['answer']}")
        print(f"\nMETRIQUES:")
        print(f"  Crides a eines: {result['tool_calls']}")
        print(f"  Tokens totals: {result['total_tokens']}")
        print(f"  Eines usades: {[t['tool'] for t in result['tools_used']]}")
```

### 3. Provar Multi-Step Reasoning (15 min)

Executa l'agent i observa especifiquement les preguntes que requereixen multiples cerques:

```bash
cd esportspulse-engine/ai-python/src/agents
python3 knowledge_agent.py
```

**Que has d'observar:**

1. **"Com ha canviat Ahri entre el patch 14.8 i el 14.12?"**
   - L'agent hauria de fer 2-3 cerques (una per cada patch)
   - Hauria de sintetitzar la informacio de totes les cerques

2. **"Quins canvis ha tingut Yasuo al patch 14.15?"**
   - No tenim dades de Yasuo ni del patch 14.15
   - L'agent hauria de dir que no te informacio — NO inventar

3. Compara el nombre de crides a eines per cada pregunta

### 4. Connectar amb el Retrieval Real (Opcional, 20 min)

Si el teu sistema de retrieval de S15 esta operatiu, reemplaca les funcions simulades:

```python
# A execute_tool, canvia la implementacio de search_patches:

# En lloc de:
#   result = search_patches(query=..., top_k=...)
# Fes servir el teu retrieval real:

from retrieval_system import search  # El modul que vas crear a S15

def execute_tool(tool_name: str, tool_input: dict) -> str:
    if tool_name == "search_patches":
        # Crida al retrieval real amb Qdrant
        results = search(
            query=tool_input["query"],
            collection="patch_notes",  # La coleccio que vas crear a S15
            top_k=tool_input.get("top_k", 3)
        )
        return json.dumps(results, ensure_ascii=False)
    # ... resta igual
```

**Aquesta es la potencia del patro agent:** el bucle i el system prompt no canvien. Nomes canvia la implementacio de les eines. L'agent no sap (ni li importa) si les dades venen d'un diccionari o de Qdrant.

---

## Checklist de Lliurament

- [ ] `patch_data.py` funciona amb `search_patches()` i `get_patch_detail()`
- [ ] `knowledge_agent.py` executa el bucle agent complet
- [ ] L'agent fa multiples cerques per preguntes comparatives (2+ crides a eines)
- [ ] L'agent respon "no tinc informacio" per preguntes sense dades (anti-hallucinacio)
- [ ] Les metriques mostren nombre de crides, tokens i eines usades per consulta
- [ ] (Opcional) L'agent connecta amb el retrieval real de S15
- [ ] Commit: `feat(agents): implement knowledge agent with retrieval tools`
