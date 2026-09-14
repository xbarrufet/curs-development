# Setmana 3 — Exercicis de Consolidació

---

## Bàsics (has de saber fer-ho)

### 1. Race condition al GameRepository
Crea un test que demostri una race condition real al `InMemoryGameRepository`: 10 threads afegint jocs simultàniament a un `ArrayList` (no `ConcurrentHashMap`). Verifica que el nombre final de jocs NO és 10 × jocs_per_thread. Després canvia a `ConcurrentHashMap` i verifica que ara sí que és correcte.

**Connexió S2:** Això demostra per què a S2 vam escollir `ConcurrentHashMap` per al repository. No era un capritx — era defensa contra concurrència.

**Fet quan:** Test que falla amb `ArrayList` i passa amb `ConcurrentHashMap`. Comentari d'1 línia explicant per què.

### 2. Extractor amb gestió d'errors parcials
Modifica el `GameDataExtractor` perquè, si una crida a l'API falla (simula amb un `throw new RuntimeException()` aleatori al 10% de les crides), no falli tot l'extractor. Ha de retornar els resultats vàlids + una llista d'errors.

**Fet quan:** Test que verifica: 50 jocs, ~5 fallen aleatòriament, l'extractor retorna ~45 resultats + 5 errors. Cap excepció no controlada.

### 3. Python asyncio extractor
Implementa l'extractor de jocs en Python amb `asyncio`. Ha de fer 50 crides simulades (amb `asyncio.sleep(0.3)`) en paral·lel. Afegeix gestió d'errors parcials (equivalent a l'exercici 2). Mesura que triga ~0.3s i no ~15s.

**Fet quan:** Script Python que extreu 50 jocs en <1s, gestiona errors parcials, i imprimeix resultats + errors.

---

## Avançats (si vas sobrat)

### 4. Optimistic Locking manual (sense JPA)
Implementa optimistic locking a mà al `InMemoryGameRepository`: afegeix un camp `version` al `GameRecord`. El mètode `update(GameRecord game)` comprova que la versió del record que es vol guardar coincideix amb la que hi ha al mapa. Si no coincideix, llança `OptimisticLockException`. Escriu un test amb 2 threads que demostri que funciona.

**Connexió S5:** Això és exactament el que JPA fa amb `@Version`, però entendre-ho sense màgia.

**Fet quan:** Test on un thread actualitza correctament i l'altre rep `OptimisticLockException`.

### 5. Benchmark: threads clàssics vs Virtual Threads vs asyncio
Crea un benchmark que compari els 3 enfocaments (Java threads, Java Virtual Threads, Python asyncio) per a 200 crides I/O simulades de 200ms cadascuna. Mesura temps total i memòria usada. Presenta els resultats en una taula.

**Fet quan:** Taula amb 3 files (enfocament, temps, memòria). El temps hauria de ser similar (~200ms) per als tres; la memòria molt diferent per als threads clàssics.
