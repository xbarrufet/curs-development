# Setmana 23 — Dijous: Exercici Capstone: Troubleshooting amb Logs

## Objectiu del Dia

Resoldre un exercici de troubleshooting on has de diagnosticar un problema al sistema utilitzant NOMES logs, metriques i traces, sense accedir al codi font. Aixo simula una situacio real d'incident en produccio i un exercici habitual en entrevistes tecniques.

---

## Teoria

### Per Que Troubleshooting amb Logs?

En produccio, quan un sistema falla:
- No pots obrir el codi i posar breakpoints.
- No pots reiniciar el servei per "provar coses".
- Necessites diagnosticar RAPID (cada minut de caiguda te un cost).
- Les teves eines son: logs, metriques, traces, i dashboards.

```
# Habilitats que es valoren en entrevistes:
#
# 1. Metodologia: segueixes un proces logic o vas a cegues?
# 2. Eines: saps consultar logs, metriques, i traces?
# 3. Hipotesis: formes hipotesis i les valides amb dades?
# 4. Comunicacio: pots explicar el problema i la solucio clarament?
# 5. Priorització: distingim entre el simptoma i la causa arrel?
```

### Metodologia de Troubleshooting

Segueix un proces sistematic, no provis coses a l'atzar:

```
# Pas 1: OBSERVAR
# Que esta passant exactament?
# - Quins simptomes veuen els usuaris?
# - Quin error veiem als logs?
# - Quines metriques son anomales?

# Pas 2: HIPOTESI
# Quina podria ser la causa?
# - Llistar 2-3 hipotesis plausibles
# - Ordenar per probabilitat

# Pas 3: VALIDAR
# Com puc confirmar o descartar cada hipotesi?
# - Quins logs hauria de buscar?
# - Quina metrica em donaria la resposta?

# Pas 4: RESOLDRE
# Quin es el fix mes rapid i segur?
# - Solucio temporal (mitigation) vs solucio definitiva (fix)

# Pas 5: DOCUMENTAR
# Que ha passat, per que, i com evitar-ho
# - Post-mortem: timeline, causa, impacte, accions
```

### Llegir Logs Eficientment

```bash
# Patrons de cerca als logs:

# 1. Buscar errors recents (ultims 100 errors)
docker-compose logs --tail=1000 backend-java 2>&1 | grep -i "error\|exception" | tail -100

# 2. Buscar per timestamp (ultims 5 minuts)
docker-compose logs --since 5m backend-java

# 3. Seguir els logs en temps real
docker-compose logs -f backend-java

# 4. Buscar un correlation ID especific
docker-compose logs backend-java ai-python 2>&1 | grep "correlation-id-abc123"

# 5. Comptar errors per tipus
docker-compose logs backend-java 2>&1 | grep "ERROR" | \
  sed 's/.*ERROR//' | sort | uniq -c | sort -rn | head -10
```

### Metriques que Delaten Problemes

```
# CPU alta + latencia alta = el servei esta saturat
# CPU baixa + latencia alta = esperant algo extern (BD, API)
# Errors 500 creixents = bug al codi o dependencia fallant
# Errors 503 = el servei esta caigut o sobrecarregat
# Temps de resposta de BD alt = queries lentes o BD sobrecarregada
# Connexions a BD al maxim = connection pool exhaurit
# Memoria creixent sense baixar = memory leak
```

### L'Exercici: Escenaris de Troubleshooting

Prepararem 3 escenaris de bugs injectats. Per a cada un, has de:
1. Identificar el simptoma.
2. Formar hipotesis.
3. Trobar la causa arrel als logs/metriques.
4. Proposar la solucio.

---

## Activitat

### Escenari 1: "El Dashboard Triga Molt" (30 min)

**Simptoma**: El dashboard Streamlit triga 15-20 segons a carregar la pagina d'equips, quan normalment triga menys de 2 segons.

**Instruccions**: Diagnostica el problema usant NOMES aquestes eines:

```bash
# 1. Comprova el temps de resposta del backend
time curl -s -o /dev/null -w "%{time_total}" \
  -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/api/v1/teams

# 2. Comprova les metriques de Prometheus
curl -s http://localhost:9090/api/v1/query?query=http_server_requests_seconds_sum | \
  python3 -m json.tool

# 3. Revisa els logs del backend
docker-compose logs --tail=200 backend-java 2>&1 | grep -i "slow\|timeout\|warn"

# 4. Comprova les metriques de la BD
curl -s "http://localhost:9090/api/v1/query?query=hikaricp_connections_active" | \
  python3 -m json.tool

# 5. Revisa els slow query logs de PostgreSQL
docker-compose logs --tail=100 postgres 2>&1 | grep -i "duration\|slow"
```

**Documenta**: 
- Quina es la teva hipotesi inicial?
- Quines dades t'han confirmat o descartat la hipotesi?
- Quina es la causa arrel?
- Quina solucio proposes?

### Escenari 2: "L'Agent No Respon" (30 min)

**Simptoma**: Quan un usuari fa una pregunta a l'agent quantitatiu, rep un error 500 intermitent. Funciona 7 de cada 10 vegades.

```bash
# 1. Reprodueix el problema (fes 10 peticions)
for i in $(seq 1 10); do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
    -X POST -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"query":"Analitza T1"}' \
    http://localhost:8000/api/v1/agents/quantitatiu/query --max-time 30)
  echo "Peticio $i: HTTP $STATUS"
done

# 2. Revisa els logs del servei Python
docker-compose logs --tail=200 ai-python 2>&1 | grep -i "error\|traceback\|exception"

# 3. Busca patrons als errors
docker-compose logs ai-python 2>&1 | grep "ERROR" | \
  awk '{print $NF}' | sort | uniq -c | sort -rn

# 4. Comprova la connexio amb Claude API
docker-compose logs ai-python 2>&1 | grep -i "anthropic\|claude\|api_error\|rate_limit"

# 5. Comprova Redis (cache)
docker-compose logs ai-python 2>&1 | grep -i "redis\|connection\|refused"
```

**Documenta** el teu proces de diagnosi pas a pas.

### Escenari 3: "Logins Fallits Massius" (30 min)

**Simptoma**: El sistema de logging mostra centenars de logins fallits en els ultims 30 minuts, pero els usuaris legitims no reporten problemes.

```bash
# 1. Quantifica els logins fallits
docker-compose logs backend-java 2>&1 | \
  grep -i "login.*fail\|auth.*fail\|401" | wc -l

# 2. Busca patrons (IP, username, timing)
docker-compose logs backend-java 2>&1 | \
  grep -i "login.*fail" | \
  awk '{print $1, $NF}' | sort | uniq -c | sort -rn | head -20

# 3. Comprova si hi ha rate limiting
curl -s "http://localhost:9090/api/v1/query?query=http_server_requests_seconds_count{uri='/api/v1/auth/login',status='401'}" | \
  python3 -m json.tool

# 4. Mira si els logins exitosos segueixen funcionant
docker-compose logs backend-java 2>&1 | \
  grep -i "login.*success\|auth.*success\|200.*login" | tail -10

# 5. Comprova l'estat del JWT
docker-compose logs backend-java 2>&1 | \
  grep -i "jwt\|token.*invalid\|token.*expired" | tail -20
```

**Pistes per investigar**:
- Es un atac de brute force?
- Es un servei intern que te credencials caducades?
- Hi ha un cron job o bot amb credencials incorrectes?

### Documentar el Proces (30 min)

Crea `docs/exercises/troubleshooting-s23.md`:

```markdown
# Exercici de Troubleshooting — Setmana 23

## Escenari 1: Dashboard Lent

### Simptoma
<!-- Descripcio del que es veia -->

### Investigacio
<!-- Pas a pas del que has fet -->

#### Hipotesi 1: [La teva hipotesi]
- Dades consultades: ...
- Resultat: confirmada / descartada
- Evidencia: [copia dels logs o metriques rellevants]

#### Hipotesi 2: [Si la primera era incorrecta]
- ...

### Causa Arrel
<!-- Que causava el problema exactament -->

### Solucio Proposada
<!-- Com es resoldria -->

### Llicons Apreses
<!-- Que has apres d'aquest escenari -->

---

## Escenari 2: Agent amb Errors Intermitents
<!-- Mateixa estructura -->

---

## Escenari 3: Logins Fallits Massius
<!-- Mateixa estructura -->
```

### Commit (10 min)

```bash
git add docs/exercises/
git commit -m "docs: add troubleshooting capstone exercise with diagnosis documentation"
```

---

## Checklist de Lliurament

- [ ] Escenari 1 investigat i documentat (causa arrel + solucio)
- [ ] Escenari 2 investigat i documentat (causa arrel + solucio)
- [ ] Escenari 3 investigat i documentat (causa arrel + solucio)
- [ ] Cada escenari te: simptoma, hipotesis, evidencia, causa, solucio
- [ ] Proces de diagnosi logic i metodic (no aleatori)
- [ ] Comandes de consulta de logs correctes
- [ ] Llicons apreses documentades per a cada escenari
- [ ] Commit amb la documentacio de l'exercici
