# Setmana 11 — Dijous: MCP Server — Construir l'Eina search_logs

## Objectiu del Dia

Passar de **consumidor** d'eines MCP (Setmana 10) a **creador**. Construiràs un servidor MCP amb FastMCP que exposa 3 eines: `search_logs`, `get_champion` i `get_champion_stats`. Al final del dia, podràs buscar logs i dades de campions des del xat de Cursor o Claude Code.

---

## Teoria

### De Consumidor a Creador de MCP

A la Setmana 10 vas usar eines MCP que altres havien creat. Ara **tu** crearàs les eines. Això et dona superpoders:

- L'LLM pot consultar els teus logs sense que tu hagis de fer grep
- Pot buscar campions a la teva API sense que tu facis curl
- Pot diagnosticar problemes creuant dades de logs i API

### FastMCP: El Framework per Crear Servidors MCP

`FastMCP` és la manera més senzilla de crear un servidor MCP en Python. Cada funció decorada amb `@mcp.tool()` es converteix en una eina que l'LLM pot cridar.

```bash
# Instal·lació de FastMCP.
pip install fastmcp
```

**Estructura del projecte:**

```
ai-python/
├── src/
│   ├── main.py                  ← FastAPI (servei web)
│   ├── logging_config.py        ← Configuració structlog
│   └── mcp_server.py            ← Servidor MCP (NOU)
└── requirements.txt
```

### Tool 1: search_logs — Buscar Logs Estructurats

Aquesta eina permet a l'LLM buscar als logs JSON que generen els nostres serveis:

```python
from fastmcp import FastMCP
import json
from pathlib import Path
from datetime import datetime, timedelta

# Creem el servidor MCP amb un nom descriptiu.
# Aquest nom apareixerà a l'IDE quan llistem les eines disponibles.
mcp = FastMCP("esportspulse")

@mcp.tool()
def search_logs(
    query: str = "",
    severity: str = "all",
    last_minutes: int = 60
) -> str:
    """
    Cerca als logs estructurats JSON d'EsportsPulse.

    Utilitza aquesta eina per diagnosticar errors, trobar patrons
    o investigar el comportament del sistema. Pots filtrar per
    text, gravetat (info/warning/error) i finestra temporal.

    Args:
        query: Text a buscar al missatge del log (ex: "champion", "timeout").
               Buit per veure tots els logs.
        severity: Filtra per gravetat: "info", "warning", "error", o "all".
        last_minutes: Mostra logs dels últims N minuts. Per defecte 60.

    Returns:
        Logs que coincideixen amb els filtres, en format llegible.
    """
    # Ruta als fitxers de log. Ajusta segons la teva configuració.
    log_file = Path("logs/esportspulse.json")

    if not log_file.exists():
        return "No s'han trobat fitxers de log. Verifica que els serveis estan arrencat."

    # Llegim els logs línia per línia (cada línia és un JSON independent).
    results = []
    cutoff = datetime.now() - timedelta(minutes=last_minutes)

    with open(log_file, "r") as f:
        for line in f:
            line = line.strip()
            if not line:
                continue

            try:
                # Parsegem cada línia com a JSON.
                entry = json.loads(line)
            except json.JSONDecodeError:
                # Si una línia no és JSON vàlid, la ignorem.
                continue

            # Filtre per temps: només logs dins de la finestra temporal.
            log_time_str = entry.get("timestamp", entry.get("@timestamp", ""))
            if log_time_str:
                try:
                    # Intentem parsejar el timestamp (format ISO 8601).
                    log_time = datetime.fromisoformat(
                        log_time_str.replace("Z", "+00:00")
                    )
                    if log_time.replace(tzinfo=None) < cutoff:
                        continue
                except ValueError:
                    pass

            # Filtre per gravetat.
            if severity != "all":
                log_level = entry.get("level", "").lower()
                if log_level != severity.lower():
                    continue

            # Filtre per text al missatge.
            if query:
                message = entry.get("message", "").lower()
                # Busquem també en tots els valors del JSON per ser exhaustius.
                all_values = " ".join(str(v).lower() for v in entry.values())
                if query.lower() not in all_values:
                    continue

            # Formatem la sortida per a l'LLM (llegible, no JSON cru).
            results.append(
                f"[{entry.get('timestamp', '?')}] "
                f"{entry.get('level', '?').upper()} "
                f"({entry.get('service_name', '?')}) "
                f"{entry.get('message', '?')} "
                f"| correlation_id={entry.get('correlation_id', 'N/A')}"
            )

    if not results:
        return f"Cap log trobat amb query='{query}', severity='{severity}', last_minutes={last_minutes}"

    # Limitem a 50 resultats per no sobrecarregar el context de l'LLM.
    header = f"Trobats {len(results)} logs (mostrant màx. 50):\n"
    return header + "\n".join(results[:50])
```

### Tool 2: get_champion — Obtenir Dades d'un Campió

```python
import httpx

@mcp.tool()
def get_champion(champion_id: int) -> str:
    """
    Obté les dades d'un campió de League of Legends des de l'API Java d'EsportsPulse.

    Utilitza aquesta eina quan necessitis informació sobre un campió concret:
    nom, rol, estadístiques, winrate, etc.

    Args:
        champion_id: L'ID numèric del campió (ex: 1 per Aatrox, 42 per Jinx).

    Returns:
        Dades del campió en format llegible, o missatge d'error si no existeix.
    """
    try:
        # Cridem a l'API Java directament.
        # Timeout de 5 segons: si Java no respon, no bloquejem l'eina.
        with httpx.Client(timeout=5.0) as client:
            response = client.get(
                f"http://localhost:8080/api/champions/{champion_id}"
            )

        # Si el campió no existeix, informem l'LLM de forma clara.
        if response.status_code == 404:
            return f"Champion amb ID {champion_id} no trobat a la base de dades."

        # Si Java retorna un error, donem detalls per al diagnòstic.
        if response.status_code >= 400:
            return (
                f"Error de l'API Java (HTTP {response.status_code}): "
                f"{response.text}"
            )

        # Formatem les dades per a l'LLM.
        data = response.json()
        return (
            f"Champion: {data.get('name', 'Desconegut')}\n"
            f"Rol: {data.get('role', 'Desconegut')}\n"
            f"Win Rate: {data.get('winRate', 'N/A')}%\n"
            f"Pick Rate: {data.get('pickRate', 'N/A')}%\n"
            f"Ban Rate: {data.get('banRate', 'N/A')}%\n"
            f"Tier: {data.get('tier', 'N/A')}"
        )

    except httpx.ConnectError:
        return (
            "ERROR: No es pot connectar al servei Java (http://localhost:8080). "
            "Verifica que el servei està arrencat."
        )
    except httpx.TimeoutException:
        return (
            "ERROR: Timeout connectant al servei Java. "
            "El servei podria estar sobrecarregat."
        )
```

### Tool 3: get_champion_stats — Estadístiques Agregades

```python
@mcp.tool()
def get_champion_stats() -> str:
    """
    Obté estadístiques agregades de tots els campions d'EsportsPulse.

    Utilitza aquesta eina per obtenir una visió general: quants campions hi ha,
    quin és el que més es juga, el que té millor winrate, distribució per rols, etc.

    Returns:
        Resum estadístic de tots els campions.
    """
    try:
        with httpx.Client(timeout=5.0) as client:
            response = client.get("http://localhost:8080/api/champions")

        if response.status_code != 200:
            return f"Error obtenint campions: HTTP {response.status_code}"

        champions = response.json()

        if not champions:
            return "No hi ha campions a la base de dades."

        # Calculem estadístiques agregades.
        total = len(champions)

        # Distribució per rol.
        roles = {}
        for c in champions:
            role = c.get("role", "Desconegut")
            roles[role] = roles.get(role, 0) + 1

        # Top 5 per winrate.
        sorted_by_wr = sorted(
            champions,
            key=lambda c: c.get("winRate", 0),
            reverse=True
        )
        top_5 = sorted_by_wr[:5]

        # Formatem el resultat de forma llegible per l'LLM.
        result = f"=== Estadístiques EsportsPulse ===\n"
        result += f"Total campions: {total}\n\n"

        result += "Distribució per rol:\n"
        for role, count in sorted(roles.items(), key=lambda x: -x[1]):
            result += f"  {role}: {count} ({count*100//total}%)\n"

        result += f"\nTop 5 Win Rate:\n"
        for i, c in enumerate(top_5, 1):
            result += (
                f"  {i}. {c.get('name', '?')} — "
                f"{c.get('winRate', 0):.1f}% WR, "
                f"{c.get('pickRate', 0):.1f}% PR\n"
            )

        return result

    except httpx.ConnectError:
        return "ERROR: No es pot connectar al servei Java."
    except httpx.TimeoutException:
        return "ERROR: Timeout connectant al servei Java."
```

### Bones Descripcions d'Eines: La Clau del MCP

L'LLM decideix quina eina cridar **basant-se en la descripció**. Si la descripció és vaga, l'LLM no sabrà quan usar-la:

```python
# MALAMENT: descripció vaga
@mcp.tool()
def search(q: str) -> str:
    """Busca coses."""  # L'LLM no sap QUÈ busca ni QUAN usar-ho.

# BÉ: descripció completa amb context
@mcp.tool()
def search_logs(query: str, severity: str = "all") -> str:
    """
    Cerca als logs estructurats JSON d'EsportsPulse.

    Utilitza aquesta eina per diagnosticar errors, trobar patrons
    o investigar el comportament del sistema.
    """
    # L'LLM sap: QUÈ fa, QUAN usar-la, i QUINS paràmetres accepta.
```

**Regles per a bones descripcions:**
1. Primera línia: què fa l'eina (breu)
2. Segon paràgraf: quan usar-la (context)
3. Args: què és cada paràmetre, amb exemples
4. Returns: què retorna (format, contingut)

### Executar el Servidor MCP

```python
# Al final de mcp_server.py:
# Punt d'entrada per executar el servidor MCP.
if __name__ == "__main__":
    # transport="stdio" significa que l'IDE es comunica via stdin/stdout.
    # Això és l'estàndard per a eines MCP locals.
    mcp.run(transport="stdio")
```

### Configurar a l'IDE (Cursor / Claude Code)

Crea `.cursor/mcp.json` a l'arrel del projecte:

```json
{
    "mcpServers": {
        "esportspulse": {
            "command": "python",
            "args": ["ai-python/src/mcp_server.py"],
            "env": {
                "PYTHONPATH": "ai-python/src"
            }
        }
    }
}
```

Per a Claude Code, crea `.claude/mcp.json`:

```json
{
    "mcpServers": {
        "esportspulse": {
            "command": "python",
            "args": ["ai-python/src/mcp_server.py"],
            "env": {
                "PYTHONPATH": "ai-python/src"
            }
        }
    }
}
```

### Depuració d'Eines MCP

Si les eines no apareixen o fallen:

```bash
# 1. Verificar que el servidor MCP arrencar sense errors:
python ai-python/src/mcp_server.py

# 2. Verificar que les eines es llisten correctament:
# A Cursor: obre el panell MCP (Ctrl+Shift+P → "MCP")
# Has de veure: search_logs, get_champion, get_champion_stats

# 3. Si no apareixen, comprova:
#    - Que mcp.json té la ruta correcta
#    - Que les dependències estan instal·lades (fastmcp, httpx)
#    - Que no hi ha errors de sintaxi al mcp_server.py
```

---

## Activitat

### Pas 1: Crear el Servidor MCP

1. Crea `ai-python/src/mcp_server.py` amb les 3 eines de la teoria
2. Afegeix `fastmcp` a `requirements.txt`
3. Verifica que arrencar: `python ai-python/src/mcp_server.py`

### Pas 2: Configurar a l'IDE

1. Crea `.cursor/mcp.json` (o `.claude/mcp.json`) amb la configuració
2. Reinicia l'IDE perquè detecti el nou servidor MCP
3. Verifica que les 3 eines apareixen al panell MCP

### Pas 3: Provar les Eines des del Xat

Arrencar Java i Python, genera alguns logs, i després prova des del xat de l'IDE:

```
Pregunta al xat: "Busca els errors dels últims 30 minuts"
→ L'LLM hauria de cridar search_logs(severity="error", last_minutes=30)

Pregunta al xat: "Quines estadístiques tenen els campions?"
→ L'LLM hauria de cridar get_champion_stats()

Pregunta al xat: "Quin winrate té Jinx?"
→ L'LLM hauria de cridar get_champion(champion_id=42)
```

### Pas 4: Afegir una Eina Extra (Opcional)

Crea una eina addicional que et sembli útil. Exemples:

```python
@mcp.tool()
def get_service_health() -> str:
    """
    Comprova l'estat de salut dels serveis d'EsportsPulse.
    Verifica que Java i Python responen correctament.
    Útil per diagnosticar si un servei està caigut.
    """
    # Implementa: crida /actuator/health de Java,
    # verifica temps de resposta, etc.
    pass
```

---

## Checklist de Lliurament

- [ ] `mcp_server.py` creat amb `FastMCP("esportspulse")`
- [ ] Eina `search_logs` implementada: filtra per query, severity i temps
- [ ] Eina `get_champion` implementada: crida a l'API Java
- [ ] Eina `get_champion_stats` implementada: estadístiques agregades
- [ ] Totes les eines tenen **descripcions completes** (què fan, quan usar-les, paràmetres)
- [ ] `.cursor/mcp.json` o `.claude/mcp.json` configurat
- [ ] Les eines apareixen a l'IDE i responen correctament des del xat
- [ ] Commit amb missatge: `feat(mcp): create EsportsPulse MCP server with log search and champion tools`
