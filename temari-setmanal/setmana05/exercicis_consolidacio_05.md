# Setmana 4 — Exercicis de Consolidació

---

## Bàsics (has de saber fer-ho)

### 1. Troba el bug (sense pistes)
El formador proporciona un segon projecte Spring Boot petit amb un bug diferent al de dilluns. L'estudiant ha de: llegir el codi, reproduir el bug, escriure un test que falla, corregir-lo, verificar que el test passa. **Sense cap pista sobre on és el bug.**

**Connexió S1-S3:** El bug podria estar en qualsevol capa (model, repository, service) i podria ser de rendiment (O(n²) amagat), de concurrència (race condition), o de lògica.

**Fet quan:** Bug identificat, test que falla, fix aplicat, test que passa. Commit amb missatge que explica el bug.

### 2. Code review escrita d'una PR real
Busca una PR oberta a un projecte open source de Spring Boot a GitHub (ex: qualsevol PR petita amb 3-5 fitxers canviats). Escriu una code review simulada: 3-5 comentaris constructius (format: Observació + Impacte + Suggeriment). No cal publicar-la — l'objectiu és practicar el format.

**Fet quan:** Document markdown amb la URL de la PR i 3-5 comentaris en format professional.

### 3. Spec → Agent → Verificació
Escull una funcionalitat senzilla que falta al projecte (ex: un mètode `exportGamesToCsv()` al `GameManagementService`). Escriu una spec de 10-15 línies (input, output, edge cases, tests esperats). Dona-la a Cursor en conversa nova. Avalua el resultat: quants tests de la spec passa? Si algun falla, itera la spec (no el codi) i regenera.

**Connexió transversal:** Primer exercici complet del cicle spec → agent → review → iteració.

**Fet quan:** Spec documentada, codi generat per agent, tests que passen. Nota de quantes iteracions van ser necessàries.

---

## Avançats (si vas sobrat)

### 4. git bisect sobre el teu propi historial
Introdueix un bug deliberat en un commit antic del teu projecte (ex: canvia un `>` per un `<` en una condició). Fes 5-10 commits més per sobre. Ara usa `git bisect` per trobar-lo automàticament, usant `mvn test` com a criteri.

**Fet quan:** `git bisect` identifica el commit culpable correctament.

### 5. Auditoria de seguretat del projecte
Passa per tot el codi del projecte EsportsPulse (S1-S4) i busca problemes de seguretat reals: secrets hardcodejats, SQL injection potencial, null handling absent, excepcions silenciades. Escriu un informe breu (màx 1 pàgina) amb els problemes trobats i les correccions proposades.

**Fet quan:** Document markdown amb problemes + fixes. Aplica els fixes i verifica que els tests passen.
