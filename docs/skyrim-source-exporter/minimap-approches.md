# Minimap : pistes d'amélioration (herbe, arbres, objets, glace)

Ce document liste les approches possibles pour enrichir la minimap générée par SkyrimSourceExporter
à partir des données du jeu : masque d'herbe via `GRAS`, arbres via `TREE`, objets placés via `REFR`,
glace et neige.

## Avertissement sur les sources

La documentation du projet (`skyrimsourceexporter.pages.forge.superkooka.com`), le dépôt
`forge.superkooka.com/Superkooka/SkyrimSourceExporter` et UESP n'étaient pas accessibles depuis
l'environnement où ce document a été rédigé. Conséquences :

- Le pipeline actuel de la minimap est **supposé** : rendu par cellule depuis `LAND` (hauteurs,
  couleurs de vertex, couches de textures `LTEX`), sans objets placés.
- Les layouts binaires cités ici viennent de connaissances générales du format. Les points marqués
  **[à vérifier]** doivent être confirmés sur
  [UESP Mod File Format](https://en.uesp.net/wiki/Skyrim_Mod:Mod_File_Format) avant implémentation.

Adapte les sections « Point de départ » si le pipeline réel diffère.

---

## 1. Point de départ supposé

```
WRLD ─► CELL (XCLC grille x,y) ─► LAND
                                   ├─ VHGT  hauteurs 33×33
                                   ├─ VNML  normales 33×33
                                   ├─ VCLR  couleurs de vertex 33×33
                                   └─ BTXT/ATXT/VTXT  couches LTEX par quadrant
LTEX ─► TXST ─► TX00 (diffuse .dds dans les BSA)
```

Ce que ce rendu ne montre pas :

1. **La végétation générée** (herbe, buissons) : elle n'existe pas en tant qu'objets, le moteur la
   génère à partir des `LTEX`.
2. **Les objets placés** : arbres, rochers, falaises, bâtiments, routes en mesh. Dans Skyrim, une
   grande partie du relief visible (falaises, montagnes, rochers) est faite de `STAT` placés, pas
   de terrain `LAND`.
3. **La glace et la neige portées par les objets** : banquises, glaciers, rochers enneigés via
   matériau directionnel.

---

## 2. Rappel des records utiles

| Record | Rôle pour la minimap | Champs clés |
|---|---|---|
| `WRLD` | Bornes, hauteurs par défaut | `DNAM` (hauteur terrain / eau par défaut), `NAM0`/`NAM9` (bornes), `NAM2` (eau par défaut) **[à vérifier]** |
| `CELL` | Position, eau | `XCLC` (x, y de grille), `XCLW` (hauteur d'eau), `XCWT` (type d'eau `WATR`) |
| `LAND` | Terrain | `VHGT`, `VNML`, `VCLR`, `BTXT`, `ATXT`, `VTXT` |
| `LTEX` | Type de sol | `TNAM` → `TXST`, `GNAM` → `GRAS` (répétable), `MNAM`/`HNAM` (matériau) **[à vérifier]** |
| `TXST` | Textures | `TX00` diffuse, `TX01` normal |
| `GRAS` | Herbe générée | `MODL`, `OBND`, `DATA` (densité, pentes, eau, plages) |
| `TREE` | Arbres | `MODL`, `OBND`, `CNAM` (données de branches) |
| `STAT` | Objets statiques | `MODL`, `OBND`, `DNAM` (angle max + `MATO` matériau directionnel) **[à vérifier]** |
| `REFR` | Instances placées | `NAME` (base), `DATA` (pos xyz + rot xyz en radians), `XSCL` (échelle) |
| `MATO` | Matériau directionnel (neige, mousse) | EDID parlant (`SnowMaterial...`) |
| `WATR` | Type d'eau | couleurs, EDID |

### Rappels de géométrie `LAND`

- Une cellule mesure 4096 unités, soit 33×33 vertex espacés de 128 unités.
- `VHGT` : un `float` d'offset puis 33×33 deltas `int8`. Le premier delta de chaque ligne se cumule
  sur la ligne précédente, les suivants sur le vertex à gauche. Hauteur réelle = cumul × 8.
  **[à vérifier : facteur 8 et 3 octets de padding final]**
- `BTXT`/`ATXT` : `formid LTEX`, `uint8 quadrant` (0 bas-gauche, 1 bas-droit, 2 haut-gauche,
  3 haut-droit), `uint8` inutilisé, `uint16 layer`.
- `VTXT` (suit chaque `ATXT`) : entrées de 8 octets `uint16 position` (0 à 288 dans la grille 17×17
  du quadrant), 2 octets inutilisés, `float opacité`.

Le poids d'une `LTEX` en un vertex s'obtient en empilant la couche de base puis les couches
additionnelles par ordre de `layer`, chacune avec son opacité (alpha blending classique). Ce calcul
existe sans doute déjà pour la couleur du terrain : le masque d'herbe le réutilise tel quel.

---

## 3. Masque d'herbe via `GRAS`

### 3.1 Principe

Le moteur place l'herbe là où une `LTEX` porte des `GNAM`, pondéré par le poids de cette texture au
point considéré, filtré par la pente et la distance à l'eau définies dans `GRAS.DATA`. On reproduit
ce calcul sous forme de **carte de couverture** (valeur 0 à 1 par texel) au lieu de placer des
brins.

### 3.2 Layout `GRAS.DATA` **[à vérifier, 32 octets attendus]**

| Offset | Type | Champ |
|---|---|---|
| 0 | `uint8` | Densité (0 à 100) |
| 1 | `uint8` | Pente min (degrés) |
| 2 | `uint8` | Pente max (degrés) |
| 3 | `uint8` | inutilisé |
| 4 | `uint16` | Unités depuis l'eau |
| 6 | `uint16` | inutilisé |
| 8 | `uint32` | Type de distance à l'eau (enum : au-dessus au moins / au plus, en dessous au moins / au plus, les deux...) |
| 12 | `float` | Position range |
| 16 | `float` | Height range |
| 20 | `float` | Color range |
| 24 | `float` | Wave period |
| 28 | `uint8` | Flags : `0x01` vertex lighting, `0x02` uniform scaling, `0x04` fit to slope |
| 29 | 3 octets | inutilisés |

### 3.3 Algorithme proposé

Pour chaque texel de la minimap (ou chaque vertex `LAND`, puis interpolation) :

```
coverage = 0
for (ltex, w) in poids_ltex(texel):            # déjà calculé pour la couleur
    for gras in ltex.GNAM:
        if not pente_ok(texel, gras.minSlope, gras.maxSlope): continue
        if not eau_ok(hauteur(texel), hauteur_eau(cell), gras.unitsFromWater, gras.waterType): continue
        coverage += w * gras.density / 100
coverage = min(coverage, 1)
```

- **Pente** : `acos(normale.z)` depuis `VNML` (décodée en vecteur normalisé) ou recalculée depuis
  `VHGT`. Recalculer depuis `VHGT` évite les écarts si `VNML` est mal régénéré dans un plugin.
- **Eau** : `CELL.XCLW` si présent, sinon hauteur d'eau par défaut du `WRLD`. Attention au sentinel
  « pas d'eau » (valeur très grande) **[à vérifier]**.
- **Couleur de l'herbe** : moyenne de la diffuse du mesh `GRAS.MODL` (texture lue dans le NIF, ou
  table par EDID en première itération). Si le flag vertex lighting est actif, moduler par `VCLR`
  comme le moteur le fait.

### 3.4 Rendu

Trois niveaux, du moins cher au plus fidèle :

1. **Teinte** : `couleur = lerp(terrain, couleur_herbe, coverage * k)` avec `k` autour de 0.3 à 0.5.
   Assombrit et verdit les zones herbeuses, suffisant pour lire le relief végétal à petite échelle.
2. **Texture de détail** : multiplier la zone par un motif de bruit (ou une petite texture de
   touffes vue de dessus) masqué par `coverage`. Donne du grain aux prairies de Whiterun.
3. **Masque exporté à part** : écrire `coverage` en canal séparé (PNG niveaux de gris ou canal alpha
   d'une texture annexe) pour que le viewer décide du rendu et puisse le désactiver.

L'option 3 est la plus souple et ne coûte presque rien en plus si l'option 1 est faite.

### 3.5 Pièges

- **Herbe sous les objets** : le moteur vanilla place de l'herbe sous les rochers et les maisons.
  Sur la carte, cela donne des halos verts dans les villes. Soustraire les empreintes des `REFR`
  (section 4) du masque corrige le problème, comme le fait le mod NGIO (No Grass In Objects).
- **Cache NGIO** : si des fichiers de cache d'herbe (`.cgid`) sont disponibles, ils contiennent les
  positions réelles. Le format n'est pas documenté officiellement : piste à garder pour plus tard,
  pas pour une première version.
- **Surcharges de plugins** : une `LTEX` modifiée par `Update.esm` ou un DLC peut changer ses
  `GNAM`. Résoudre le record gagnant selon l'ordre de chargement avant de lire les `GNAM`.
- **Solstheim** (`DLC2SolstheimWorld`) : herbe de cendre et `LTEX` de cendre, la couleur ne doit pas
  être verte. D'où l'intérêt d'une couleur par `GRAS` plutôt qu'une constante.

---

## 4. Objets placés : `REFR` vers `TREE`, `STAT` et autres

### 4.1 Collecte

- Parcourir les `CELL` du `WRLD` (groupes de blocs et sous-blocs extérieurs), puis les enfants
  temporaires et persistants de chaque cellule **[à vérifier : types de GRUP 1, 4, 5, 6, 8, 9]**.
- Pour chaque `REFR` : résoudre `NAME` vers son record de base, lire `DATA` (position, rotation) et
  `XSCL` (échelle, 1.0 par défaut).
- Ignorer les `REFR` désactivés par défaut (flag de record `Initially Disabled`) et ceux dont la
  base est un marqueur (`XMarker`, `MapMarker`, lumières, sons).
- Le volume est élevé sur `Tamriel` (plusieurs centaines de milliers de références) : indexer par
  cellule ou par tuile de minimap dès la collecte.

### 4.2 Trois niveaux de représentation

| Approche | Données nécessaires | Coût | Rendu |
|---|---|---|---|
| A. Empreinte `OBND` | `OBND` de la base, `DATA`, `XSCL` | Faible | Rectangles tournés ou ellipses |
| B. Maillages LOD | NIF `_lod` ou fichiers `.bto` | Moyen | Silhouettes simplifiées |
| C. Maillages complets | NIF complets depuis les BSA | Élevé | Silhouettes exactes, vue de dessus |

**A. Empreinte `OBND`** : `OBND` donne une boîte englobante locale (6 `int16`). On applique échelle
puis rotation Z, et on rastérise le rectangle obtenu. Suffit pour les maisons, murs, rochers moyens.
Les rotations X et Y sont ignorées, acceptable pour une carte.

**B. LOD existant** : le jeu embarque déjà des versions vues de loin.
- Objets : `meshes/terrain/<worldspace>/objects/*.bto` combinent les gros objets par tuile, avec un
  atlas de textures. Un rendu orthographique vertical de ces fichiers donne directement une couche
  « objets » cohérente avec ce que voit le joueur de loin.
- Terrain : `textures/terrain/<worldspace>/<worldspace>.<lod>.<x>.<y>.dds` sont des diffuses
  pré-calculées vues de dessus. Elles peuvent servir de référence visuelle pour valider le rendu,
  ou de source directe à basse résolution.
- **[à vérifier]** le format `.bto`/`.btr` (NIF allégé) et la couverture en cellules de chaque
  niveau de LOD (4, 8, 16, 32).

**C. NIF complets** : nécessite un parseur NIF (blocs `BSTriShape` en SE, `NiTriShape` en LE) et un
rasterizer orthographique avec z-buffer. C'est la seule approche qui rend correctement les falaises
et montagnes (`Mountain*`, `RockCliff*`), qui sont des `STAT` et pas du terrain. À réserver à une
seconde phase.

### 4.3 Arbres (`TREE`)

Les arbres sont les objets qui changent le plus la lecture de la carte : forêts de Falkreath, de la
Faille, pins enneigés.

- **Canopée** : cercle de rayon `max(|OBND.x|, |OBND.y|) × XSCL`, couleur par famille d'arbre.
- **Famille** : dériver de l'EDID de la base (`TreePine*`, `TreeAspen*`, `TreeBirch*`,
  `TreeReach*`, variantes `Snow`) ou du chemin `MODL`. Une table EDID vers couleur couvre le vanilla,
  un repli sur une couleur moyenne de la texture de feuilles couvre les mods.
- **Ombre portée** : dupliquer le disque décalé et assombri, dans une direction fixe (nord-ouest par
  exemple, cohérente avec l'ombrage du relief). Donne du volume à coût quasi nul.
- **Échelle** : à fort dézoom, les disques individuels deviennent du bruit. Calculer une densité
  (noyau gaussien sur les positions) et la rendre comme une teinte « forêt » continue. Choisir entre
  disques et densité selon le niveau de zoom de la tuile.
- Certains arbres et buissons sont des `STAT` ou des `FLOR` (plantes récoltables), pas des `TREE`.
  Filtrer par chemin de modèle (`meshes/landscape/trees/`, `meshes/landscape/plants/`) en plus du
  type de record.

### 4.4 Ordre de composition

```
terrain (LAND + LTEX)
  → ombrage du relief
  → masque d'herbe (moins les empreintes d'objets)
  → eau
  → objets au sol (routes en mesh, rochers)
  → glace / neige d'objets
  → bâtiments
  → canopées d'arbres (+ ombres)
  → marqueurs de carte
```

---

## 5. Glace et neige

### 5.1 Glace

La glace n'est pas un type de terrain dans `LAND` : ce sont des `STAT` placés (banquises au nord de
Winterhold, glaciers, stalactites de glace).

- **Identification** : EDID ou chemin `MODL` qui correspond à `Ice`, `Glacier`, `IceFloe`,
  `IceBerg` (ou dossier `meshes/landscape/ice/` **[à vérifier]**). Construire une liste explicite
  plutôt qu'une regex trop large (`Dice`, `Justice`...).
- **Rendu** : empreinte (approche A ou C) remplie d'un blanc bleuté, contour légèrement plus foncé
  pour détacher la banquise de l'eau. Les banquises flottent au niveau de l'eau : les dessiner au-
  dessus de la couche eau.
- **Eau gelée** : vérifier aussi `CELL.XCWT` et les `WATR` dont l'EDID indique une eau glacée ou
  arctique pour teinter l'eau différemment.

### 5.2 Neige sur le terrain

- `LTEX` dont la diffuse `TXST.TX00` ou l'EDID contient `Snow` : déjà rendues en blanc si la
  couleur vient de la texture. Si la couleur vient d'une table, ajouter une catégorie neige.
- Le masque d'herbe doit rester à 0 sur ces zones (normalement déjà le cas via `GNAM` vide).

### 5.3 Neige sur les objets

Les rochers et montagnes enneigés utilisent un matériau directionnel : `STAT.DNAM` pointe vers un
`MATO` qui applique la neige sur les faces orientées vers le haut **[à vérifier : layout `DNAM`,
angle max + formid `MATO`]**.

- Approche A (empreinte) : si la base a un `MATO` de type neige, remplir l'empreinte en blanc au
  lieu de la couleur rocher.
- Approche C (NIF) : pour chaque triangle, si `normale.z > cos(angle_max)` alors neige, sinon
  couleur de la texture. C'est ce que fait le shader en jeu, et vu de dessus quasi tout le dessus
  d'un rocher enneigé est blanc.

---

## 6. Améliorations annexes à coût faible

- **Ombrage du relief** (hillshade) depuis `VNML` ou `VHGT` : `lambert(normale, lumière)` multiplié
  sur la couleur. Si déjà fait, ajouter une occlusion ambiante approximative (différence entre la
  hauteur locale et la moyenne sur un voisinage).
- **Profondeur de l'eau** : `hauteur_eau - hauteur_terrain` pour dégrader la couleur de l'eau, ce
  qui fait apparaître les hauts-fonds et les rives.
- **Routes** : les `LTEX` de route (EDID contenant `Road`) peuvent recevoir une couleur plus
  contrastée pour rester lisibles sous le masque d'herbe.
- **Couleurs de vertex** : si `VCLR` n'est pas encore utilisé, le multiplier sur la diffuse. C'est
  ce qui donne les variations de teinte locales (terre brûlée, zones sombres).

---

## 7. Proposition de découpage

| Étape | Contenu | Dépendances | Effort estimé |
|---|---|---|---|
| 1 | Parser `GRAS`, lire `LTEX.GNAM`, calculer `coverage`, rendu teinte + export du masque | poids `LTEX` existants | Faible |
| 2 | Collecte des `REFR` par cellule, empreintes `OBND` | parser `REFR`, `STAT`, `TREE` | Moyen |
| 3 | Arbres : disques, familles, ombres, densité au dézoom | étape 2 | Faible |
| 4 | Soustraction des empreintes dans le masque d'herbe | étapes 1 et 2 | Faible |
| 5 | Glace et neige d'objets via EDID et `MATO` | étape 2 | Faible |
| 6 | Rendu orthographique des `.bto` LOD ou des NIF | parseur NIF | Élevé |

Les étapes 1 à 5 n'exigent aucun parsing de mesh. L'étape 6 apporte les falaises et montagnes, qui
sont la principale différence restante avec la carte en jeu.

## 8. Validation

Zones de test qui couvrent tous les cas :

| Zone | Ce qu'elle vérifie |
|---|---|
| Plaines de Whiterun | Densité d'herbe, pentes, routes lisibles |
| Falkreath / Faille | Forêts, familles d'arbres, densité au dézoom |
| Mer des Fantômes au nord de Winterhold | Banquises, eau glacée |
| L'Aube (the Pale) | Neige terrain + rochers enneigés |
| Blancherive (ville) | Absence d'herbe sous les bâtiments |
| Solstheim | Cendre, couleur d'herbe non verte, worldspace DLC |

Comparer chaque tuile à la diffuse LOD du jeu (`textures/terrain/...`) au même emplacement donne une
référence objective sans capture d'écran en jeu.

## 9. Questions ouvertes

- Le viewer de la minimap peut-il consommer des couches séparées (herbe, arbres, objets) ou faut-il
  tout cuire dans une seule image ?
- Quelle résolution cible par cellule ? Au-delà de 32 px par cellule, les disques d'arbres
  individuels deviennent pertinents ; en dessous, seule la densité est lisible.
- Le projet doit-il supporter les plugins de mods (ordre de chargement complet) ou seulement le
  vanilla + DLC ? Cela change la stratégie de couleur par EDID.
