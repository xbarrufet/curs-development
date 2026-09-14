# Setmana 15 — Dimarts: Anti-al·lucinació: Forçar l'LLM a Dir "No Ho Sé"

## Objectiu del Dia

Implementar tècniques anti-al·lucinació al pipeline RAG perquè l'LLM mai inventi informació que no estigui als chunks recuperats. Al final del dia, l'LLM ha de respondre "no tinc informació" quan li facis preguntes trampa que no cobreixen les patch notes.

---

## Teoria

### El Problema: L'LLM Sempre Vol Ajudar

Els LLMs estan entrenats per ser útils. Quan els fas una pregunta, generen una resposta encara que no tinguin informació fiable. Això és un problema greu en aplicacions empresarials:

```
Pregunta: "Quins canvis va rebre Teemo al patch 14.7?"

Sense anti-al·lucinació:
"Teemo va rebre un buff al seu Q, augmentant el dany de 80 a 95..."
→ ❌ INVENTAT. El patch 14.7 no toca Teemo, però l'LLM genera una 
     resposta plausible perquè sap coses de Teemo del seu entrenament.

Amb anti-al·lucinació:
"No tinc informació sobre canvis a Teemo al patch 14.7 
 en les patch notes indexades."
→ ✅ CORRECTE. L'LLM reconeix que el context no conté aquesta info.
```

### Per Què Passa? El Mecanisme de l'Al·lucinació

L'al·lucinació en RAG té dues causes principals:

1. **Contaminació del coneixement previ:** L'LLM barreja el que sap del seu entrenament amb el context que li passes. Si li preguntes sobre Teemo, "recorda" coses de Teemo i les barreja amb les patch notes.

2. **Chunks irrellevants com a "prova":** Si el retrieval retorna chunks vagament relacionats (score 0.71), l'LLM els interpreta com a context vàlid i genera respostes basades en fragments fora de context.

```
┌────────────────────────────────────────────┐
│         FONTS D'AL·LUCINACIÓ EN RAG        │
│                                            │
│  Pregunta de l'usuari                      │
│       │                                    │
│       ├─→ Retrieval pobre (chunks dolents) │
│       │     → L'LLM interpreta malament    │
│       │                                    │
│       ├─→ Prompt massa permissiu           │
│       │     → L'LLM omple buits inventant  │
│       │                                    │
│       └─→ Coneixement previ de l'LLM       │
│             → Barreja dades pròpies amb     │
│               el context proporcionat       │
└────────────────────────────────────────────┘
```

### Tècnica 1: System Prompt Restrictiu

El system prompt és la primera línia de defensa. Ha de ser explícit i contundent:

```python
# Prompt DOLENT — massa permissiu
BAD_PROMPT = "Respon preguntes sobre patch notes."
# Problema: no diu què fer quan no té informació

# Prompt BO — restrictiu i clar
GOOD_PROMPT = """Ets un assistent que respon EXCLUSIVAMENT basant-se 
en el CONTEXT proporcionat a continuació.

REGLES ESTRICTES:
1. NO facis servir coneixement previ. NOMÉS el context d'aquest missatge.
2. Si la resposta NO es troba al context, digues EXACTAMENT:
   "No tinc informació sobre això en les patch notes indexades."
3. NO intentis deduir, inferir o extrapolar més enllà del que diu el text.
4. Cada afirmació ha d'estar recolzada per una cita del context.
5. Si el context és ambiguu, digues que la informació és incompleta.
"""
```

Les paraules clau importants: "EXCLUSIVAMENT", "EXACTAMENT", "NO intentis deduir". Els LLMs responen millor a instruccions assertives.

### Tècnica 2: Citació Obligatòria (Citation Enforcement)

Si forces l'LLM a citar la font de cada afirmació, és molt més difícil que inventi:

```python
# Afegir al system prompt:
CITATION_RULES = """
FORMAT DE RESPOSTA OBLIGATORI:
- Cada afirmació ha de tenir una cita entre claudàtors: [Font X]
- Exemple: "Jinx va rebre una reducció d'AD base de 59 a 55 [Font 1]"
- Si no pots citar cap font, NO facis l'afirmació
"""
```

Quan l'LLM ha de citar `[Font 3]`, primer busca si la Font 3 realment diu el que vol escriure. Si no hi és, és més probable que es freni.

### Tècnica 3: Validació Post-Generació

Després de generar la resposta, pots verificar-la programàticament:

```python
def validate_response(answer: str, chunks: list[dict]) -> dict:
    """
    Valida que la resposta no contingui al·lucinacions evidents.
    
    Comprovacions:
    1. Si cita fonts, existeixen al context?
    2. La resposta és "no tinc informació" quan no hi ha chunks?
    3. Hi ha afirmacions sense cita?
    """
    # Verificar que no afirma coses sense chunks
    # Si no hem trobat chunks rellevants, la resposta ha de ser "no tinc info"
    if not chunks and "no tinc informació" not in answer.lower():
        return {
            "valid": False,
            "reason": "Resposta generada sense chunks de context"
        }
    
    return {"valid": True, "reason": "OK"}
```

### Tècnica 4: Ajust del Score Threshold

Pujar el `SCORE_THRESHOLD` redueix al·lucinacions a costa de menys respostes:

| Threshold | Al·lucinacions | Respostes útils | Ús recomanat        |
|-----------|---------------|-----------------|----------------------|
| 0.5       | Altes         | Moltes          | Mai en producció     |
| 0.7       | Mitjanes      | Bones           | Desenvolupament      |
| 0.8       | Baixes        | Menys           | Producció general    |
| 0.85      | Molt baixes   | Poques          | Aplicacions crítiques|

---

## Activitat

### 1. Millorar el system prompt (20 min)

Actualitza `rag_engine.py` amb el prompt anti-al·lucinació:

```python
"""
System prompt millorat amb instruccions anti-al·lucinació.
Força l'LLM a citar fonts i rebutjar preguntes sense context.
"""

RAG_SYSTEM_PROMPT = """Ets un assistent expert en patch notes de League of Legends
per al projecte EsportsPulse.

REGLES ESTRICTES — SEGUEIX-LES SEMPRE:

1. CONTEXT ÚNIC: Respon EXCLUSIVAMENT basant-te en el CONTEXT proporcionat.
   NO facis servir cap coneixement previ sobre el joc, campions o patches.

2. SENSE INFORMACIÓ: Si la resposta NO es troba al context, respon EXACTAMENT:
   "No tinc informació sobre això en les patch notes indexades."
   No afegeixis res més. No intentis ser útil amb informació d'altres fonts.

3. CITACIÓ OBLIGATÒRIA: Cada afirmació ha de citar la font:
   Exemple: "Jinx va rebre una reducció d'AD base [Font 1]"
   Si no pots citar una font concreta, NO facis l'afirmació.

4. SENSE DEDUCCIONS: No infereixis, dedueixis ni extrapolis.
   Només afirma el que diu LITERALMENT el context.

5. AMBIGÜITAT: Si el context és ambigu o insuficient, digues-ho explícitament.
   Exemple: "El context menciona canvis a Jinx però no especifica els valors exactes."
"""
```

### 2. Implementar validació post-generació (25 min)

Afegeix al `rag_engine.py`:

```python
"""
Funcions de validació per detectar al·lucinacions a la resposta generada.
"""

import re


def extract_citations(answer: str) -> list[int]:
    """
    Extreu els números de font citats a la resposta.
    Busca patrons com [Font 1], [Font 2], etc.
    
    Args:
        answer: La resposta generada per l'LLM
    Returns:
        Llista d'enters amb els números de font citats
    """
    # Regex per trobar "[Font N]" a la resposta
    pattern = r'\[Font\s+(\d+)\]'
    matches = re.findall(pattern, answer)
    return [int(m) for m in matches]


def validate_response(answer: str, chunks: list[dict]) -> dict:
    """
    Valida la resposta contra els chunks de context per detectar al·lucinacions.
    
    Comprovacions:
    1. Si no hi ha chunks, la resposta ha de ser "no tinc informació"
    2. Les cites [Font N] han de correspondre a chunks reals
    3. Si hi ha chunks, la resposta hauria de contenir cites
    
    Args:
        answer: La resposta generada
        chunks: Els chunks usats com a context
    Returns:
        Diccionari amb 'valid' (bool), 'reason' (str), 'warnings' (list)
    """
    warnings = []
    
    # Comprovació 1: Resposta sense chunks
    no_info_phrase = "no tinc informació"
    if not chunks:
        if no_info_phrase not in answer.lower():
            return {
                "valid": False,
                "reason": "L'LLM ha generat una resposta sense cap chunk de context",
                "warnings": ["POSSIBLE AL·LUCINACIÓ: resposta sense fonts"]
            }
    
    # Comprovació 2: Cites vàlides
    citations = extract_citations(answer)
    if citations:
        max_valid = len(chunks)
        invalid_citations = [c for c in citations if c > max_valid or c < 1]
        if invalid_citations:
            warnings.append(
                f"Cites invàlides: {invalid_citations} "
                f"(només hi ha {max_valid} chunks)"
            )
    
    # Comprovació 3: Resposta amb chunks però sense cites
    if chunks and not citations and no_info_phrase not in answer.lower():
        warnings.append(
            "La resposta no conté cites [Font N] — "
            "difícil verificar la fidelitat"
        )
    
    return {
        "valid": len(warnings) == 0,
        "reason": "OK" if not warnings else "Validació amb advertències",
        "warnings": warnings
    }


def ask_with_validation(query: str) -> dict:
    """
    Versió millorada de ask() que inclou validació anti-al·lucinació.
    
    Args:
        query: La pregunta de l'usuari
    Returns:
        Diccionari amb resposta, fonts, i resultat de la validació
    """
    # Executar el pipeline RAG
    chunks = retrieve_chunks(query)
    context = format_context(chunks)
    answer = generate_answer(query, context)
    
    # Validar la resposta
    validation = validate_response(answer, chunks)
    
    # Si la validació falla, retornar resposta segura
    if not validation["valid"] and not chunks:
        answer = "No tinc informació sobre això en les patch notes indexades."
    
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
        ],
        "validation": validation
    }
```

### 3. Crear tests amb preguntes trampa (25 min)

Crea `ai-python/src/retrieval/test_anti_hallucination.py`:

```python
"""
Tests anti-al·lucinació: preguntes trampa que l'LLM NO hauria de poder respondre.
L'objectiu és verificar que l'LLM diu "no tinc informació" quan cal.
"""

from rag_engine import ask_with_validation

# Preguntes trampa: preguntes que probablement NO estan a les patch notes
# L'LLM ha de respondre "no tinc informació" per a TOTES
TRICK_QUESTIONS = [
    # Campió/patch que no existeix
    "Quins canvis va rebre el campió Zarkon al patch 99.1?",
    # Pregunta fora de l'àmbit (no és sobre patch notes)
    "Quin és el millor build per a Yasuo a la ranked?",
    # Informació massa específica que no estaria a les notes
    "Quants jugadors van abandonar el joc després del patch 14.3?",
    # Pregunta sobre un joc diferent
    "Quins canvis va rebre Genji a l'última patch d'Overwatch?",
    # Pregunta real però amb una patch inventada
    "Quins campions es van afegir al joc al patch 14.99?",
]

# Preguntes legítimes que SÍ haurien de tenir resposta
VALID_QUESTIONS = [
    "Quins canvis va rebre Jinx?",
    "Hi ha hagut algun canvi a objectes d'ADC?",
]


def test_trick_questions():
    """
    Verifica que l'LLM rebutja correctament les preguntes trampa.
    """
    print("=" * 60)
    print("TEST ANTI-AL·LUCINACIÓ — Preguntes Trampa")
    print("=" * 60)
    
    passes = 0
    fails = 0
    
    for i, question in enumerate(TRICK_QUESTIONS, 1):
        print(f"\n{'─' * 60}")
        print(f"Pregunta trampa {i}: {question}")
        
        result = ask_with_validation(question)
        answer = result["answer"].lower()
        
        # L'esperem és que digui "no tinc informació" o similar
        is_refusal = (
            "no tinc informació" in answer
            or "no es troba" in answer
            or "no he trobat" in answer
            or "no disposo" in answer
        )
        
        if is_refusal:
            print(f"  ✅ PASS — L'LLM ha rebutjat correctament")
            passes += 1
        else:
            print(f"  ❌ FAIL — L'LLM ha al·lucinat!")
            print(f"  Resposta: {result['answer'][:200]}...")
            fails += 1
        
        # Mostrar informació de validació
        validation = result["validation"]
        if validation["warnings"]:
            for w in validation["warnings"]:
                print(f"  ⚠️  {w}")
    
    print(f"\n{'=' * 60}")
    print(f"Resultat: {passes}/{len(TRICK_QUESTIONS)} preguntes trampa rebutjades")
    if fails > 0:
        print(f"ATENCIÓ: {fails} al·lucinacions detectades. Revisa el prompt.")
    print(f"{'=' * 60}")


def test_valid_questions():
    """
    Verifica que les preguntes legítimes SÍ reben resposta amb cites.
    """
    print(f"\n{'=' * 60}")
    print("TEST — Preguntes Legítimes (haurien de tenir resposta)")
    print(f"{'=' * 60}")
    
    for i, question in enumerate(VALID_QUESTIONS, 1):
        print(f"\nPregunta {i}: {question}")
        result = ask_with_validation(question)
        
        has_citations = "[Font" in result["answer"]
        has_chunks = result["chunks_used"] > 0
        
        if has_chunks and has_citations:
            print(f"  ✅ Resposta amb {result['chunks_used']} chunks i cites")
        elif has_chunks:
            print(f"  ⚠️  Resposta amb chunks però sense cites explícites")
        else:
            print(f"  ❌ No s'han trobat chunks rellevants")


if __name__ == "__main__":
    test_trick_questions()
    test_valid_questions()
```

### 4. Executar els tests i iterar (15 min)

```bash
# Executar els tests anti-al·lucinació
cd ai-python/src/retrieval
python test_anti_hallucination.py
```

Si alguna pregunta trampa passa (l'LLM al·lucina):
1. Revisa el system prompt — potser cal ser més restrictiu
2. Puja el `SCORE_THRESHOLD` a 0.8 o 0.85
3. Afegeix validació addicional per al cas específic

L'objectiu: **5/5 preguntes trampa rebutjades** i **preguntes legítimes amb cites**.

---

## Checklist de Lliurament

- [ ] System prompt actualitzat amb regles anti-al·lucinació
- [ ] Funció `validate_response()` implementada i funcional
- [ ] Funció `ask_with_validation()` retorna objecte amb camp `validation`
- [ ] 5/5 preguntes trampa rebutjades correctament (l'LLM diu "no tinc informació")
- [ ] Preguntes legítimes retornen respostes amb cites `[Font N]`
- [ ] Commit: `feat(rag): add anti-hallucination prompts and response validation`
