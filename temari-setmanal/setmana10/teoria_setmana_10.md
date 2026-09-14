# Setmana 10 - Teoria: Streamlit, Spec-Driven UI i Testing End-to-End

## 1. Per què Streamlit en aquest curs

La setmana 10 no busca convertir l’estudiant en frontend specialist. La idea és molt més pràctica:

- fer visible el sistema en una interfície ràpida,
- practicar especificació visual,
- i validar que la API funciona des d’una perspectiva real d’usuari.

Streamlit és perfecte per això perquè permet construir un dashboard molt ràpidament sense necessitat d’estructura frontend complexa.

---

## 2. Streamlit: prototipatge ràpid, no framework web

Streamlit funciona amb un model molt diferent del web tradicional:

- cada interacció re-executa l’script,
- l’UI es declara en ordre i no en components alienats,
- la lògica d’interfície és simple i directa.

Això és ideal per:
- prototips,
- dashboards operatius,
- proves ràpides de conceptes,
- i validació de les dades del backend.

No és la millor base per construir un producte web complex de zero, però sí per iterar amb velocitat.

---

## 3. Spec-driven UI: del wireframe a la implementació

La diferència clau entre un prototype mediocre i un prototype útil és la spec.

Una spec de dashboard ha de definir:
- layout general,
- components,
- flux d’interacció,
- restriccions,
- i criteris de validació.

Exemple conceptual:

```text
┌─────────────────────────────────────┐
│ GamePulse Dashboard      [Refresh]  │
├────────────────────────────┬────────┤
│ Filtres                   │ Taula  │
│ Buscar...                 │ Jocs   │
│ Preu min/max              │        │
│ Només gratuïts            │        │
├────────────────────────────┴────────┤
│ Detall del joc seleccionado          │
└─────────────────────────────────────┘
```

Aquesta especificació és molt útil perquè l’agent sap exactament què ha de generar i com ha de respondre visualment. El resultat és una UI més coherent i menys dependent de l’interpretació de l’agent.

---

## 4. Separar lògica de presentació

Una de les lliçons clau d’aquesta setmana és que Streamlit no ha de contenir tota la lògica de negoci.

És millor separar:
- `filter_games(...)`
- `calculate_kpis(...)`
- `format_price(...)`
- i la capa d’UI `st.*`

Això permet:
- tests automatitzats,
- menys bugs,
- i un codi més mantenible.

La regla és la mateixa que amb Spring:
- la capa d’exposició no té tota la lògica,
- la lògica es testea per separat.

---

## 5. Testing del dashboard: no és la UI, és la lògica

Una UI de Streamlit és difícil de provar com a frontend tradicional. El patró recomanat és:
- provar la lògica pura,
- no el render de l’UI,
- i fer E2E manual o semi-automatitzat per veure el flux complet.

Exemples de tests útils:
- filtrar per nom,
- filtrar per rang de preu,
- mostrar KPIs amb llista buida,
- format de preus,
- error si l’API no respon.

L’objectiu és fer que el dashboard sigui observabilitat i verificació visual, no una “cosa de clickar i veure”.

---

## 6. CI per Python: igual que Java, però en el stack del dashboard

Aquesta setmana reforça l’ideologia del curs: un projecte real no només executa backend; també valora la qualitat d’altres capes.

Per això el CI s’amplia amb:
- `pytest` per a la lògica Python,
- `ruff` per format i lints,
- validació de tests del dashboard,
- i integració amb el pipeline Java existent.

Això és la base del desenvolupament modern: no hi ha “feature feta” si els tests i el linter no passen.

---

## 7. End-to-end testing: la vista de l’usuari

La millor validació del dashboard no és un screenshot bonic; és que l’usuari real pot:
- obrir el dashboard,
- cercar un joc,
- crear un joc,
- veure canvis reals,
- i llegir errors si el backend falla.

Aquest flux és el que representa un sistema real de producte: l’usuari no veu només un formulari, veu un sistema complet.

---

## 8. Objectiu d’aprenentatge de la setmana

Al final de S10, l’estudiant ha de ser capaç de:
- construir un dashboard funcional amb Streamlit,
- escriure una spec visual clara,
- integrar-lo amb una API REST,
- separar la lògica de presentació,
- i validar el sistema amb tests i fluxos E2E.

Aquesta setmana és la transició del “software de backend” al “software amb feedback visible per l’usuari”.
