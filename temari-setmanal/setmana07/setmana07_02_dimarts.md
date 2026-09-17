# Setmana 07 — Dimarts: Git Rebase, Conflictes i Workflow Professional

## Objectiu del Dia

Dominar el flux de treball Git que fan servir els equips professionals: branques feature, rebase per mantenir un historial lineal, resolucio de conflictes manual, i eines de diagnòstic com `git bisect`. Al final del dia sabras gestionar conflictes sense por i mantenir un historial de commits net.

---

## Teoria

### Git Flow Simplificat

En un equip professional, mai es treballa directament a `main`. El flux estàndard:

```
main (estable, sempre desplegable)
  │
  ├── feature/add-champion-search    ← Tu treballes aquí
  ├── feature/price-format           ← Un company treballa aquí
  └── feature/import-csv             ← Un altre company aquí
```

**Cicle de vida d'una feature:**

```
1. Crear branca    → git checkout -b feature/champion-filter
2. Desenvolupar    → commits petits i freqüents
3. Rebase          → git rebase main (agafar canvis nous de main)
4. Push            → git push origin feature/champion-filter
5. Pull Request    → Revisió per un company
6. Merge           → El reviewer aprova i fa merge a main
7. Netejar         → git branch -d feature/champion-filter
```

---

### Per Què Rebase i No Merge?

Quan fas `git merge`, Git crea un "merge commit" extra que embruteix l'historial:

```
  Amb MERGE (historial brut):
  
  * (merge commit) Merge branch 'feature/search'    ← Commit extra sense valor
  |\
  | * feat: add search by role
  | * feat: add search by name
  * | fix: correct winRate calculation
  |/
  * feat: add champion repository
```

```
  Amb REBASE (historial lineal i net):

  * feat: add search by role
  * feat: add search by name
  * fix: correct winRate calculation
  * feat: add champion repository
```

**Regla:** `rebase` per actualitzar la teva branca. `merge` (via PR) per integrar a main.

---

### Com Funciona Rebase

Rebase "mou" els teus commits al final de la branca objectiu:

```
  ABANS del rebase:

       A---B---C  feature/search      ← els teus commits (A, B, C)
      /
  D---E---F---G  main                 ← main ha avançat (F, G són nous)
```

```
  DESPRÉS de git rebase main:

                   A'--B'--C'  feature/search   ← mateixos canvis, nous commits
                  /
  D---E---F---G  main                           ← la teva branca comença des de G
```

**Important:** A', B', C' son commits **nous** (diferent hash). Per això **mai** fas rebase de branques compartides (main, develop).

---

### Comandes Essencials

#### `git rebase main` — Actualitzar la teva branca

```bash
# Situació: estàs a feature/search i main ha avançat
git checkout feature/search
git fetch origin              # Descarrega canvis del remot sense aplicar-los
git rebase origin/main        # Mou els teus commits al final de main

# Si hi ha conflictes:
# 1. Git s'atura i et diu quin fitxer té conflictes
# 2. Edita el fitxer, resol els conflictes
# 3. git add <fitxer_resolt>
# 4. git rebase --continue
# Si vols cancel·lar: git rebase --abort
```

#### `git rebase -i HEAD~3` — Netejar commits (Interactive Rebase)

```bash
# Obre un editor amb els últims 3 commits per reordenar/ajuntar/editar
# Útil per ajuntar commits "WIP" abans d'un PR
git rebase -i HEAD~3

# L'editor mostra:
# pick abc1234 feat: add search method
# pick def5678 WIP: fix typo           ← canvia 'pick' per 'squash'
# pick ghi9012 WIP: another fix        ← canvia 'pick' per 'squash'
#
# Resultat: els 3 commits es fusionen en un sol commit net
```

#### `git stash` — Guardar canvis temporalment

```bash
# Tens canvis sense commit però has de canviar de branca urgentment
git stash                     # Guarda els canvis en una "pila" temporal
git checkout main             # Canvia de branca tranquil·lament
git checkout feature/search   # Torna a la teva branca
git stash pop                 # Recupera els canvis guardats

# Veure la pila de stash:
git stash list
# stash@{0}: WIP on feature/search: abc1234 feat: add search
```

#### `git log --oneline --graph` — Visualitzar l'historial

```bash
# Mostra l'arbre de commits de forma visual i compacta
git log --oneline --graph --all

# Exemple de sortida:
# * a1b2c3d (HEAD -> feature/search) feat: add search by role
# * d4e5f6g feat: add search by name
# | * 7h8i9j0 (origin/main) fix: correct winRate
# |/
# * k1l2m3n feat: add champion repository
```

#### `git cherry-pick <commit>` — Agafar un commit específic

```bash
# Aplica un commit concret d'una altra branca a la teva
# Útil quan un company ha fet un fix que necessites
git cherry-pick a1b2c3d       # Copia el commit a1b2c3d a la branca actual

# Si hi ha conflictes, resol-los igual que amb rebase
```

#### `git reflog` — L'historial secret (recuperar commits "perduts")

```bash
# Mostra TOTES les accions que has fet, incloent les que semblen "perdudes"
# Útil quan un rebase ha anat malament i vols tornar enrere
git reflog

# Exemple de sortida:
# a1b2c3d HEAD@{0}: rebase finished
# d4e5f6g HEAD@{1}: rebase: feat: add search
# 7h8i9j0 HEAD@{2}: checkout: moving from main to feature/search
# k1l2m3n HEAD@{3}: commit: feat: original commit before rebase

# Per tornar a un estat anterior:
git reset --hard HEAD@{3}     # Torna a l'estat abans del rebase
```

---

### Resolucio de Conflictes

Un conflicte passa quan **dues branques modifiquen la mateixa linia** del mateix fitxer.

#### Anatomia d'un Conflicte

```java
// Git marca el conflicte dins del fitxer:
public class ChampionManagementService {

<<<<<<< HEAD
    // Versió de la TEVA branca (feature/search)
    public List<ChampionRecord> searchByName(String keyword) {
        return repository.findByNameContaining(keyword);
    }
=======
    // Versió de MAIN (o la branca on fas rebase)
    public List<ChampionRecord> searchByNameIgnoreCase(String keyword) {
        return repository.findByNameContainingIgnoreCase(keyword);
    }
>>>>>>> main
}
```

#### Com Resoldre'l

**MAI acceptis cegament "Accept Incoming" o "Accept Current".** Entén les dues versions:

```java
// ✅ RESOLUCIÓ CORRECTA — Agafa el millor de les dues versions
// La versió de main tenia IgnoreCase (millora), la teva tenia el nom correcte
public List<ChampionRecord> searchByName(String keyword) {
    // Combinem: el nom del mètode de la nostra branca
    // + el comportament IgnoreCase de main (és una millora)
    return repository.findByNameContainingIgnoreCase(keyword);
}
```

Després de resoldre:

```bash
git add src/main/java/com/esportspulse/service/ChampionManagementService.java
git rebase --continue    # Continua el rebase amb el conflicte resolt
```

---

### Git Bisect — Trobar Bugs amb Cerca Binària

Tens 50 commits i un bug. En lloc de mirar-los tots un per un:

```bash
# 1. Comença el bisect
git bisect start

# 2. Marca l'estat actual com a "dolent" (el bug existeix)
git bisect bad

# 3. Marca un commit antic on saps que funcionava bé
git bisect good v0.1    # o un hash de commit: git bisect good a1b2c3d

# 4. Git fa checkout al commit del MIG
# Tu proves si el bug existeix o no:
#   Si existeix: git bisect bad
#   Si no existeix: git bisect good

# 5. Git repeteix (cerca binària) fins a trobar el commit exacte
# Bisecting: 3 revisions left to test after this (roughly 2 steps)

# 6. Quan acaba:
# abc1234 is the first bad commit
# Author: Joan <joan@example.com>
# Date:   Mon Sep 7 10:30:00 2026
# feat: change winRate calculation

# 7. Acaba el bisect
git bisect reset
```

**Amb 50 commits, bisect troba el bug en ~6 passos** (log2(50) ≈ 6).

---

## Activitat

### Exercici 1: Simular i Resoldre un Conflicte (45 min)

Segueix aquests passos exactament:

```bash
# 1. Assegura't que estàs a main amb tot commitejat
git checkout main

# 2. Crea una branca feature
git checkout -b feature/price-format

# 3. Modifica ChampionRecord.java — afegeix un mètode formattedWinRate()
# Fes commit: git commit -m "feat: add formattedWinRate method"

# 4. Torna a main
git checkout main

# 5. Modifica el MATEIX fitxer — afegeix un mètode displayName()
#    a la MATEIXA zona del fitxer (per provocar conflicte)
# Fes commit: git commit -m "feat: add displayName method"

# 6. Torna a la feature branch i fes rebase
git checkout feature/price-format
git rebase main

# 7. CONFLICTE! Resol-lo manualment:
#    - Obre el fitxer amb conflictes
#    - Entén les dues versions
#    - Combina-les correctament
#    - git add <fitxer>
#    - git rebase --continue
```

### Exercici 2: Interactive Rebase per Netejar Commits (30 min)

```bash
# 1. Crea una branca nova
git checkout -b feature/cleanup-practice

# 2. Fes 4 commits petits (un per cada canvi):
#    - "WIP: start search method"
#    - "WIP: add filter logic"
#    - "WIP: fix typo"
#    - "feat: complete champion search"

# 3. Fes interactive rebase per ajuntar els 3 primers en un:
git rebase -i HEAD~4

# 4. Canvia 'pick' per 'squash' als commits WIP
# 5. Edita el missatge final del commit combinat
# 6. Verifica amb: git log --oneline
```

### Exercici 3: Git Bisect (20 min)

```bash
# 1. Crea 5 commits al teu projecte (un per cada petit canvi)
# 2. Al commit 3, introdueix un bug intencionadament
#    (per exemple, canvia un assertEquals esperat)
# 3. Usa git bisect per trobar quin commit ha introduït el bug
# 4. Documenta quants passos ha necessitat bisect
```

---

## Checklist de Lliurament

- [ ] He simulat i resolt un conflicte de rebase manualment
- [ ] He fet un interactive rebase per ajuntar commits WIP
- [ ] He practicat git bisect per trobar un bug
- [ ] Entenc la diferencia entre `rebase` i `merge`
- [ ] Se quan usar `git stash`, `git cherry-pick` i `git reflog`
- [ ] L'historial de la meva branca es lineal (verificat amb `git log --oneline --graph`)
- [ ] Commit amb missatge: `docs: add git workflow practice exercises`
