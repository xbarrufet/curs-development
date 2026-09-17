# Setmana 05 — Divendres: Demo End-to-End — Domini→Repository→Service→REST Funcionant

## Objectiu del Dia

Verificar, de punta a punta i sense escriure codi nou, que les cinc setmanes construint l'stack (Domini i validació a S1-S2, Repository i Service a S2, persistència real amb JPA a S4, i l'API REST aquesta setmana) formen una aplicació coherent i funcional. Al final del dia tindràs una demo reproduïble de tot el cicle CRUD contra la teva pròpia API, un tag de milestone, i hauràs identificat quins punts de l'arquitectura et costa explicar — que és exactament el que revisaràs a partir de la setmana vinent.

---

## Teoria

### Per Què Aquest Dia No Té Codi Nou

Fins avui, cada dia ha afegit una peça: un `record`, una interfície `Repository`, una entitat JPA, un `@RestController`. És fàcil arribar aquí sense haver vist mai el conjunt sencer funcionant en una sola execució. Avui no aprens cap tecnologia nova — **connectes els punts**:

```
┌─────────────┐   ┌──────────────┐   ┌─────────────┐   ┌──────────────────┐
│ ChampionRecord │→│ ChampionRepository │→│ ChampionService │→│ ChampionController │→ HTTP
│   (Domini, S1-S2) │  (Interfície, S2 · JPA, S4) │  (S2)   │      (REST, S5)      │
└─────────────┘   └──────────────┘   └─────────────┘   └──────────────────┘
```

Aquesta capacitat — mirar un sistema de diverses capes i saber traçar una petició d'un cap a l'altre — és exactament el que et demanaran el primer dia en una feina real, quan et donin accés a un repositori que no has escrit tu. Avui ho practiques amb l'avantatge que sí que l'has escrit tu: si et costa traçar-lo ara, val la pena aturar-se abans de seguir afegint-hi capes.

### Anatomia d'una Petició Completa

Quan un client fa `POST /api/champions`, la petició travessa totes les capes que has construït:

```
1. HTTP arriba a Tomcat (embegut a Spring Boot)
2. ChampionController rep el JSON, el deserialitza a un DTO/record
3. ChampionController delega a ChampionService (mai accedeix al repository directament)
4. ChampionService aplica lògica de negoci i validacions
5. ChampionService crida ChampionRepository (interfície, S2)
6. Spring Data JPA (S4) tradueix la crida a SQL contra H2
7. La resposta puja les mateixes capes en sentit invers
8. ChampionController serialitza el resultat a JSON i respon
```

Si en algun punt d'aquesta cadena no sabries dir quin fitxer del teu projecte s'executa, aquest és el dia per aclarir-ho.

---

## Activitat

### 1. Arrenca i Verifica l'Stack Complet (15 min)

```bash
# Arrenca l'API (H2 en memòria, sense dependències externes)
cd backend-java && mvn spring-boot:run
```

En un altre terminal, comprova que totes les capes responen:

```bash
# Health check bàsic
curl -i http://localhost:8080/api/champions

# Cicle CRUD complet — cada crida travessa Controller→Service→Repository→JPA
curl -X POST http://localhost:8080/api/champions \
  -H "Content-Type: application/json" \
  -d '{"name": "Ahri", "role": "Mage", "winRate": 52.3}'

curl http://localhost:8080/api/champions
curl http://localhost:8080/api/champions/1
curl -X PUT http://localhost:8080/api/champions/1 \
  -H "Content-Type: application/json" \
  -d '{"name": "Ahri", "role": "Mage", "winRate": 54.1}'
curl -X DELETE http://localhost:8080/api/champions/1
curl -i http://localhost:8080/api/champions/1    # Ha de retornar 404
```

### 2. Traça una Petició Capa per Capa (20 min)

Agafa el `POST /api/champions` que acabes d'executar i, sense mirar codi encara, escriu en un full quin creus que és el recorregut exacte: quin mètode de quina classe crida a quin altre. Després obre el projecte i comprova-ho línia per línia, des de `ChampionController` fins al `save()` del repository.

Respon per escrit (2-3 frases cadascuna):
- On es valida que `winRate` estigui entre 0 i 100? Per què allà i no en un altre punt?
- Si demà canviessis H2 per PostgreSQL, quants fitxers hauries de tocar? (Aquesta pregunta té a veure amb per què vam abstraure el Repository a S2 abans de tenir JPA a S4.)
- Quina capa faria fallar el test si algú trenqués la validació sense voler?

### 3. CLI + API en un Sol Flux (15 min)

Executa el CLI de Python (fet ahir) contra l'API i confirma que el mateix flux CRUD funciona des de fora de la JVM:

```bash
python ai-python/src/cli.py create-champion --name "Zed" --role "Assassin" --win-rate 49.8
python ai-python/src/cli.py list-champions
```

Si això funciona, tens dues aplicacions independents (Java i Python) parlant amb el mateix contracte HTTP — la base de tot el treball políglota que ve a partir d'ara.

### 4. Tag de Milestone (10 min)

Marca aquest punt del projecte: és la primera versió amb l'stack complet funcionant.

```bash
git checkout main
git pull

git tag -a v0.1-vertical-slice -m "Vertical slice complet: Domini, Repository, Service i REST API funcionant end-to-end"
git push origin v0.1-vertical-slice
```

> **Lectura recomanada (opcional, no bloquejant):**
> - Martin Fowler: [PresentationDomainDataLayering](https://martinfowler.com/bliki/PresentationDomainDataLayering.html)

---

## Checklist de Lliurament

- [ ] L'API arrenca amb `mvn spring-boot:run` i respon a totes les operacions CRUD via `curl`
- [ ] El CLI de Python fa el mateix cicle CRUD contra la mateixa API
- [ ] Has traçat per escrit el recorregut d'una petició `POST` capa per capa i ho has comprovat contra el codi real
- [ ] Has respost les 3 preguntes sobre on viu la validació i per què abstraure el Repository importa
- [ ] Tag `v0.1-vertical-slice` creat i pujat al repositori remot
