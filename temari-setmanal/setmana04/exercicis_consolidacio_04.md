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

### 5. Comparativa Java CompletableFuture vs Python asyncio
Implementa el mateix extractor de 50 jocs (amb crides simulades de 300ms) en Java (`CompletableFuture`) i en Python (`asyncio`). Compara: temps total, línies de codi, i gestió d'errors parcials. Quin codi és més llegible? Quin és més fàcil de depurar?

**Fet quan:** Ambdós extractors funcionen, una taula comparativa amb les mètriques, i un paràgraf d'opinió sobre els trade-offs de cada enfocament.
