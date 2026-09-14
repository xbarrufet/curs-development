# Setmana 14 — Dilluns: Per Què la IA Falla: El Problema de la Informació No Estructurada

## Objectiu del Dia

Entendre per què els LLMs "al·lucinen" quan els preguntes sobre dades específiques del teu domini, i per què estructurar la informació és el primer pas per resoldre-ho. Al final del dia tindràs documentats casos reals de fallades i una visió clara del problema que resoldrem aquesta setmana.

---

## Teoria

### Què Saben Realment els LLMs?

Un LLM (Large Language Model) com Claude o GPT no és una base de dades. No "sap" coses — ha après patrons estadístics de text. Quan li preguntes alguna cosa:

1. **Genera text plausible**, no respostes factuals
2. **No té accés a informació posterior** al seu entrenament (cutoff date)
3. **No pot distingir** entre el que "sap" i el que "inventa"

```python
# Exemple conceptual: què passa dins d'un LLM
# (simplificació per entendre el problema)

# El que TU penses que fa:
def llm_ideal(pregunta: str) -> str:
    """Busca la resposta correcta en una base de coneixement."""
    resposta = base_de_dades.cerca(pregunta)  # NO fa això
    return resposta

# El que REALMENT fa:
def llm_real(pregunta: str) -> str:
    """Genera el text més probable donat el context."""
    # Prediu el següent token basant-se en probabilitats
    # No verifica si el que diu és cert
    tokens = []
    for _ in range(max_tokens):
        next_token = model.predict_next(pregunta + "".join(tokens))
        tokens.append(next_token)
    return "".join(tokens)
```

> **Clau:** Un LLM no sap que no sap. Per això inventa respostes amb total confiança — és el que estadísticament "encaixa" millor.

### Al·lucinacions: Quan la IA Inventa

Les al·lucinacions (hallucinations) són respostes que sonen correctes però són falses. Passen especialment quan:

- **Dades específiques del domini:** Pregunta sobre un patch concret d'un joc → inventa números.
- **Informació temporal:** Events recents, actualitzacions, estadístiques canviants.
- **Detalls tècnics precisos:** Noms de mètodes d'una API que no ha vist prou.

```
# Prova-ho tu mateix — pregunta a Claude:
"Quins canvis va rebre Jinx al patch 14.15 de League of Legends?"

# Probable resposta: una llista detallada de canvis que SONEN reals
# però que poden ser parcialment o totalment inventats.
# El model genera el que "estadísticament encaixa" amb
# "patch notes de LoL" — no el que realment va passar.
```

### Per Què Passa? El Gap d'Informació

```
┌─────────────────────────────────────────────────────┐
│           El Problema Fonamental                     │
│                                                      │
│  El que el LLM sap:     Tot internet fins al cutoff  │
│  El que TU necessites:  Dades específiques, actuals  │
│                                                      │
│  ┌──────────────┐         ┌──────────────────┐      │
│  │  Coneixement │         │ Les teves dades  │      │
│  │  del LLM     │   GAP   │ de domini        │      │
│  │  (general,   │◄───────►│ (específiques,   │      │
│  │   estàtic)   │         │  actualitzades)  │      │
│  └──────────────┘         └──────────────────┘      │
│                                                      │
│  Solució: DONAR-LI les dades en el moment adequat   │
└─────────────────────────────────────────────────────┘
```

### Dades Estructurades vs No Estructurades

No totes les dades són iguals per a una IA. El **format** importa enormement:

```python
# NO ESTRUCTURAT — Un bloc de text HTML copiat d'una wiki
text_brut = """
<div class="patch-notes">
  <h2>Patch 14.15</h2>
  <p>Jinx: Base AD reduced from 59 to 57. Rocket damage 
  now scales with... Meanwhile, Ahri received buffs to her 
  charm duration which now lasts 1.5s at all ranks instead 
  of 1/1.1/1.2/1.3/1.4s. Also, the new item Stormsurge 
  had its proc damage nerfed from 100-200 to 75-150...</p>
</div>
"""
# Problemes: tot barrejat, difícil de buscar, difícil de filtrar

# ESTRUCTURAT — La mateixa informació organitzada
patch_note = {
    "patch": "14.15",
    "date": "2024-07-24",
    "game": "League of Legends",
    "changes": [
        {
            "champion": "Jinx",
            "type": "nerf",
            "details": [
                {
                    "stat": "Base AD",
                    "before": 59,
                    "after": 57
                }
            ]
        },
        {
            "champion": "Ahri",
            "type": "buff",
            "details": [
                {
                    "stat": "Charm Duration",
                    "before": "1/1.1/1.2/1.3/1.4s",
                    "after": "1.5s at all ranks"
                }
            ]
        }
    ]
}
# Avantatge: podem buscar per champion, filtrar per tipus, comparar patches
```

### Per Què el Format Importa per a Retrieval

Quan volem que una IA respongui preguntes sobre les nostres dades, el flux és:

```
┌──────────┐     ┌──────────────┐     ┌──────────┐     ┌──────────┐
│ Pregunta │────►│ Cerca en les │────►│ Context  │────►│ Resposta │
│ Usuari   │     │ teves dades  │     │ rellevant│     │ precisa  │
└──────────┘     └──────────────┘     └──────────┘     └──────────┘

# Si les dades estan ben estructurades:
#   - La cerca és precisa (troba exactament el chunk rellevant)
#   - El context és net (sense soroll)
#   - La resposta és fiable (basada en fets, no en probabilitats)

# Si les dades són un blob de text:
#   - La cerca retorna coses vagament relacionades
#   - El context porta soroll (informació irrellevant)
#   - La resposta pot seguir sent inventada
```

### El Pla de la Setmana

Aquesta setmana construirem la **pipeline de Knowledge Engineering** per EsportsPulse:

| Dia | Pas | Què farem |
|-----|-----|-----------|
| Dilluns | Problema | Documentar fallades del LLM |
| Dimarts | Chunking | Partir documents en trossos útils amb metadades |
| Dimecres | Embeddings | Convertir text en vectors i indexar a Qdrant |
| Dijous | Eines | Trobar MCP servers, skills i plugins existents |
| Divendres | Consolidar | Script d'ingestió, @docs, PR final |

---

## Activitat

### 1. Preparar el Repositori (10 min)

Crea l'estructura per a la feina d'aquesta setmana dins del projecte EsportsPulse:

```bash
# Crea la carpeta per al mòdul de knowledge engineering
mkdir -p ai-python/src/knowledge

# Crea el fitxer on documentaràs els experiments d'avui
touch ai-python/src/knowledge/hallucination_tests.md
```

### 2. Experiment: Documentar Al·lucinacions (30 min)

Obre Claude (o un altre LLM) i fes-li preguntes **específiques** sobre dades que hauria de saber però probablement no sap bé. Documenta cada interacció:

```markdown
<!-- ai-python/src/knowledge/hallucination_tests.md -->

# Experiments d'Al·lucinació — Setmana 14

## Test 1: Patch Notes Específics
**Pregunta:** "Quins canvis va rebre Jinx al patch 14.15 de LoL?"
**Resposta del LLM:** [copia la resposta aquí]
**Verificació:** [busca els patch notes reals i compara]
**Resultat:** ✅ Correcte / ⚠️ Parcial / ❌ Inventat

## Test 2: Estadístiques de Torneig
**Pregunta:** "Qui va guanyar la final del Worlds 2023 de LoL i amb quin resultat?"
**Resposta del LLM:** [copia la resposta aquí]
**Verificació:** [busca el resultat real]
**Resultat:** ✅ / ⚠️ / ❌

## Test 3: Dades del Teu Projecte
**Pregunta:** "Quins champions té la taula de la meva base de dades EsportsPulse?"
**Resposta del LLM:** [copia la resposta aquí]
**Verificació:** Impossible — el LLM NO té accés a la teva BD
**Resultat:** ❌ Per definició (no pot saber-ho)

## Test 4: Detalls Tècnics d'API
**Pregunta:** "Quins endpoints té la Riot Games API per obtenir dades de partides?"
**Resposta del LLM:** [copia la resposta aquí]
**Verificació:** [consulta la documentació oficial]
**Resultat:** ✅ / ⚠️ / ❌

## Conclusions
[Escriu 3-4 línies sobre què has après dels tests]
```

### 3. Recollir Dades Reals per la Setmana (20 min)

Necessitarem dades reals per treballar durant la setmana. Busca i guarda:

```bash
# Crea un directori per a les dades en brut
mkdir -p ai-python/data/raw

# Opció A: Descarrega patch notes d'un joc
# (ex: League of Legends patch notes des de la wiki o web oficial)
# Guarda el contingut en un fitxer markdown o HTML

# Opció B: Usa dades d'exemple — crea un fitxer amb patch notes ficticis
# però realistes per practicar
```

```python
# ai-python/src/knowledge/sample_data.py
"""
Dades d'exemple per practicar Knowledge Engineering.
Si no tens accés a patch notes reals, usa aquestes.
Simula el format real de patch notes d'un MOBA.
"""

# Simulem patch notes en format similar al real
# Això ens servirà per practicar chunking i indexació
SAMPLE_PATCHES = [
    {
        "patch": "14.15",
        "date": "2024-07-24",
        "highlights": "Mid-season adjustments focused on bot lane carry diversity.",
        "champion_changes": [
            {
                "name": "Jinx",
                "role": "ADC",
                "change_type": "nerf",
                "changes": [
                    "Base AD: 59 → 57",
                    "Fishbones (Q) bonus range: 100/125/150/175/200 → 75/100/125/150/175",
                ]
            },
            {
                "name": "Ahri",
                "role": "Mid",
                "change_type": "buff",
                "changes": [
                    "Charm (E) duration: 1/1.1/1.2/1.3/1.4s → 1.5s at all ranks",
                    "Fox-Fire (W) mana cost: 40 → 30",
                ]
            },
            {
                "name": "Thresh",
                "role": "Support",
                "change_type": "adjustment",
                "changes": [
                    "Death Sentence (Q) cooldown: 20/18/16/14/12 → 19/17/15/13/11",
                    "Flay (E) slow: 20/25/30/35/40% → 25/30/35/40/45%",
                ]
            },
        ],
        "item_changes": [
            {
                "name": "Stormsurge",
                "change_type": "nerf",
                "changes": ["Proc damage: 100-200 → 75-150"]
            }
        ]
    },
    {
        "patch": "14.16",
        "date": "2024-08-07",
        "highlights": "Worlds preparation patch with pro-play focused adjustments.",
        "champion_changes": [
            {
                "name": "Azir",
                "role": "Mid",
                "change_type": "nerf",
                "changes": [
                    "Arise! (W) soldier damage: 60-160 → 55-150",
                    "Emperor's Divide (R) cooldown: 120/105/90 → 130/110/90",
                ]
            },
            {
                "name": "Yone",
                "role": "Mid/Top",
                "change_type": "buff",
                "changes": [
                    "Way of the Hunter (passive) crit multiplier: 2.5x → 2.6x",
                ]
            },
        ],
        "item_changes": []
    }
]

if __name__ == "__main__":
    # Verifica que les dades es carreguen correctament
    for patch in SAMPLE_PATCHES:
        print(f"Patch {patch['patch']} ({patch['date']})")
        print(f"  Champions modificats: {len(patch['champion_changes'])}")
        print(f"  Items modificats: {len(patch['item_changes'])}")
        print()
```

### 4. Primer Anàlisi: Estructurat vs No Estructurat (15 min)

```python
# ai-python/src/knowledge/format_comparison.py
"""
Compara com de fàcil és respondre preguntes 
amb dades estructurades vs no estructurades.
"""

from sample_data import SAMPLE_PATCHES

# Pregunta: "Quins champions van rebre nerfs al patch 14.15?"

# AMB DADES ESTRUCTURADES — Trivial, exacte, programàtic
def find_nerfs_structured(patch_version: str) -> list[str]:
    """Busca nerfs en les dades estructurades. Fàcil i precís."""
    for patch in SAMPLE_PATCHES:
        if patch["patch"] == patch_version:
            # Podem filtrar exactament el que volem
            return [
                ch["name"]
                for ch in patch["champion_changes"]
                if ch["change_type"] == "nerf"
            ]
    return []

# AMB TEXT PLA — Necessites un LLM o NLP complex
TEXT_BLOB = """
Patch 14.15 brought mid-season adjustments focused on bot lane carry 
diversity. Jinx received a nerf with her base AD going from 59 to 57 
and Fishbones bonus range reduced. Ahri was buffed with increased Charm 
duration and lower Fox-Fire mana cost. Thresh got an adjustment to both 
Q cooldown and E slow values. Stormsurge proc damage was also nerfed.
"""

def find_nerfs_unstructured(text: str) -> str:
    """
    Amb text pla, no podem filtrar programàticament.
    Necessitem un LLM o regex complexos (i fràgils).
    """
    # Això NO és fiable — depèn del format del text
    import re
    # Regex fràgil que pot fallar amb qualsevol canvi de format
    pattern = r"(\w+) received a nerf"
    matches = re.findall(pattern, text)
    return matches  # Pot perdre'n o trobar falsos positius

if __name__ == "__main__":
    print("=== Dades Estructurades ===")
    nerfs = find_nerfs_structured("14.15")
    print(f"Nerfs al 14.15: {nerfs}")
    # Output: ['Jinx'] — correcte, complet, fiable
    
    print("\n=== Text Pla ===")
    nerfs_text = find_nerfs_unstructured(TEXT_BLOB)
    print(f"Nerfs trobats (regex): {nerfs_text}")
    # Output: pot ser incomplet o incorrecte
    
    print("\n=== Conclusió ===")
    print("Les dades estructurades permeten cerques exactes.")
    print("El text pla requereix IA o regex fràgils.")
    print("Knowledge Engineering = passar de text pla a estructurat.")
```

### 5. Reflexió i Documentació (10 min)

Afegeix al fitxer `hallucination_tests.md`:

```markdown
## Reflexió Final

### Què he après avui?
- Els LLMs no "saben" coses — generen text probable
- Les al·lucinacions són inevitables sense dades de domini
- El format de les dades (estructurat vs text pla) afecta directament
  la qualitat de les respostes

### Com ho aplicarem a EsportsPulse?
- Tenim patch notes, estadístiques, wikis → dades de domini
- Cal convertir-les a un format que la IA pugui usar
- Dimarts començarem a "trossejar" (chunking) els documents
```

---

## Checklist de Lliurament

- [ ] Fitxer `hallucination_tests.md` amb mínim 4 tests documentats i verificats
- [ ] Fitxer `sample_data.py` amb dades d'exemple (o dades reals descarregades)
- [ ] Fitxer `format_comparison.py` que demostra la diferència entre cercar en dades estructurades vs text pla
- [ ] Directori `ai-python/data/raw/` creat per a les dades de la setmana
- [ ] Pots explicar amb les teves paraules per què un LLM al·lucina i com l'accés a dades reals ho mitiga
