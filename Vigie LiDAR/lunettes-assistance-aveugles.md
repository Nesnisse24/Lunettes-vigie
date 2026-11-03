---
titre: Lunettes d'assistance pour personnes aveugles
nom-provisoire: Vigie (nom officiel non choisi)
cadre: PFE ECE
statut: cadrage (livrable n°1 — Design Thinking et cahier des charges, nov. 2026)
tags: [PFE, lunettes, LiDAR, vision, Raspberry-Pi, accessibilité]
---

# Lunettes d'assistance pour personnes aveugles

> [!summary] En une phrase
> Des lunettes équipées d'une **caméra** et d'un **[[LiDAR]]** repèrent les dangers que la [[canne blanche]] ne détecte pas. Un **[[Raspberry Pi]]** analyse ce qu'elles captent. Le porteur est prévenu par des **vibreurs** sur les branches et, en cas d'urgence seulement, par une **phrase courte**.

---

## 1. Contexte

- Projet de fin d'études (PFE) de l'équipe, à l'ECE.
- Des livrables sont à rendre au professeur. Le livrable n°1 ([[Design Thinking]] et [[cahier des charges]]) est attendu en novembre 2026.
- Le prototype doit être construit par l'équipe avec des composants peu coûteux (budget étudiant), autour d'un [[Raspberry Pi]].

> [!note] Nos choix de cadrage
> - Pour rester réalisable en un an, on se **concentre sur un seul problème principal**. Les autres restent des objectifs secondaires, non garantis.
> - Le problème prioritaire **n'est pas encore choisi** par l'équipe.
> - Le nom du projet n'est pas encore choisi : « Vigie » est provisoire.

---

## 2. Les quatre problèmes identifiés

| # | Problème | Pourquoi la canne ne suffit pas | Capteur le plus utile | Part d'IA |
|---|---|---|---|---|
| 1 | **Véhicules silencieux** (vélos, trottinettes, voitures électriques) | Ils arrivent vite, sans bruit, et souvent par le côté | LiDAR (vitesse) + caméra (type d'objet) | Moyenne : détection et suivi d'objets |
| 2 | **Ruptures de sol / vide** (marches, bouche de métro, bordure) | La canne les trouve, mais tard (≈ 1 m) | LiDAR + [[IMU]] | Faible : des règles géométriques suffisent |
| 3 | **Obstacles à hauteur de tête** (branches, panneaux, rétroviseurs de camion) | La canne passe dessous | LiDAR | Faible : des règles géométriques suffisent |
| 4 | **Identification d'éléments** (numéros de bus, noms de rue, magasins) | La canne ne lit pas | Caméra | Forte : détection de texte + [[OCR]] |

> [!tip] À garder en tête pour le choix
> Plus on va vers le problème 4, plus le projet devient un projet d'**IA**. Plus on va vers les problèmes 2 et 3, plus il devient un projet **hardware / embarqué**. Le problème 1 est le plus équilibré entre les deux.

---

## 3. Principe de fonctionnement

```
  CAPTER                 COMPRENDRE                    PRÉVENIR
  caméra  ─┐                                     ┌─► vibreur gauche
           ├─► Raspberry Pi ─► dangers par ──────┤
  LiDAR  ──┘   (+ IMU)          secteur          ├─► vibreur droit
                                                 └─► voix (urgence / sur demande)
```

1. **Capter.** La caméra donne une image. Le LiDAR donne une liste de distances, une par zone.
2. **Reconnaître.** Un modèle de vision (type [[YOLO]]) encadre les objets qu'il connaît dans l'image.
3. **Mesurer.** Chaque mesure LiDAR devient un point 3D. Avec l'inclinaison de la tête lue par l'[[IMU]], on calcule la **hauteur** de ce point par rapport au sol.
4. **Fusionner.** Les points LiDAR qui tombent dans le cadre d'un objet reconnu lui donnent sa **distance exacte** ([[fusion de capteurs]]).
5. **Décider.** L'espace devant le porteur est découpé en trois secteurs : gauche, devant (±15°), droite. Dans chaque secteur, on garde le danger le plus proche.
6. **Prévenir.** Le vibreur du côté concerné s'active, avec une intensité qui dépend de la distance et un motif qui dépend du type de danger.

---

## 4. Caméra ou LiDAR ? Les deux

Les lunettes peuvent fonctionner avec **la caméra seule**, **le LiDAR seul** ou **les deux**.

| | Caméra | LiDAR |
|---|---|---|
| Ce qu'elle / il apporte | **Ce que c'est** (personne, vélo, panneau…) | **À quelle distance**, précisément |
| Distance | Estimée, approximative | Mesurée au centimètre près |
| La nuit | Ne voit presque plus rien | Fonctionne pareil (il envoie sa propre lumière) |
| Vitres | Peut reconnaître un abri vitré | Le laser traverse le verre : invisible |
| Surfaces noires | OK | Peu de retour (voiture noire) |
| Objets inconnus | Ne reconnaît que ce qu'on lui a appris | Mesure tout, sans savoir ce que c'est |

> [!important] Idée clé
> La caméra **nomme** l'objet, le LiDAR le **mesure**. Ensemble, ils couvrent les angles morts l'un de l'autre.

### Classement d'un point LiDAR selon sa hauteur

| Hauteur par rapport au sol sous les pieds | Classe |
|---|---|
| < −10 cm | **Marche, trou ou bordure** (vide si < −75 cm) |
| −10 à +10 cm | Sol, ignoré |
| 10 cm à 1,25 m | **Obstacle au sol** (la canne le trouve) |
| 1,25 à 2,1 m | **Obstacle à hauteur de tête**, si rien en dessous (la canne ne le trouve pas) |
| > 2,1 m | Au-dessus, ignoré |

---

## 5. Retour au porteur

### Vibreurs (un sur chaque branche)

| Motif | Signification |
|---|---|
| Impulsions de plus en plus rapides | Obstacle au sol qui se rapproche |
| Double impulsion | Obstacle à hauteur de tête |
| Vibration longue | Marche, trou ou bordure |

- L'intensité passe de 0 % (à 2,5 m) à 100 % (à 0,6 m).
- Le côté qui vibre indique le côté du danger. Un danger juste devant fait vibrer les deux côtés.

### Voix

> [!note] Notre règle
> - Les lunettes ne parlent d'elles-mêmes **qu'en cas d'extrême urgence**.
> - Elles décrivent l'environnement **seulement quand le porteur les interroge**.
> - Ce que la canne détecte (bornes, bancs, marches…) **n'est pas annoncé**.

**Urgences annoncées automatiquement :**
- Un objet **rapide** qui coupe la route (vélo, trottinette) : « Stop ! Vélo qui arrive vite, à 10 heures, à 4 pas. »
- Un obstacle **à hauteur de visage** sur la trajectoire : « Attention, panneau à hauteur de visage, droit devant, à 3 pas. Décalez-vous vers la droite. »
- Un **vide** sur la trajectoire (bouche de métro) : annoncé seulement si le porteur se dirige vers lui.

**Sur demande** (« Qu'y a-t-il devant moi ? ») : description de tout ce qu'il y a à moins de 6 pas.

**Format des phrases**, pensé pour une personne aveugle :
- distance **en pas** (≈ 70 cm) plutôt qu'en mètres ;
- direction **en heures d'horloge** : 12 h = droit devant, 9 h = à gauche, 3 h = à droite. C'est le repère utilisé en locomotion avec les personnes aveugles.

### Évaluer le danger d'un objet qui bouge

Tout ce qui bouge n'est pas une urgence :
- Un **piéton** marche à environ 1,25 m/s, il nous voit et s'écarte, et la canne le signale. → **Pas d'urgence.**
- Un **vélo** roule à plus de 3 m/s et ne s'arrêtera pas. → **Urgence**, le porteur doit s'arrêter.

Le critère retenu est la vitesse mesurée entre deux trames LiDAR : au-delà de **2,2 m/s** (≈ 8 km/h), c'est une urgence. Si la caméra reconnaît l'objet, son type sert aussi : une « personne » n'est jamais une urgence, un « vélo » l'est toujours.

> [!info] Positionnement
> Les lunettes **complètent la canne, elles ne la remplacent pas.**

---

## 6. Matériel envisagé

| Élément | Option | Caractéristiques | Remarque |
|---|---|---|---|
| Calcul | [[Raspberry Pi]] | — | Un accélérateur IA peut être nécessaire pour la vision |
| LiDAR **recommandé** | **VL53L8CX** (ST) | 8 × 8 zones, 45° × 45°, jusqu'à 4 m, [[I2C]] ou [[SPI]] | Garde ≈ 2,85 m de portée en lumière ambiante : adapté à l'extérieur |
| LiDAR champ large | **3 × VL53L8CX en éventail** | ≈ 135° × 45° | Couvre toute la largeur du trottoir ; il faut changer l'adresse I2C de deux capteurs |
| LiDAR intérieur | VL53L7CX | 60° × 60°, jusqu'à 3,5 m, ≈ 20 $ (Pololu) | Perd beaucoup de portée en plein soleil (≈ 65 cm) |
| Profondeur dense | Caméra [[temps de vol\|ToF]] Arducam | 240 × 180 points, jusqu'à 4 m, 30 images/s | Plus encombrante ; comportement au soleil à tester |
| Alternative USB | Sipeed MaixSense A010 | 100 × 100 points, 0,2 à 2,5 m | Portée courte |
| Version minimale | VL53L1X ou TFmini-S | Un seul rayon | Il faut balayer avec la tête |
| Orientation de la tête | [[IMU]] | — | **Indispensable** pour calculer la hauteur des points (marches, trous) |
| Image | Caméra pour Raspberry Pi | ≈ 66° × 41° | — |
| Retour | 2 moteurs vibreurs + sortie audio | — | Type de sortie audio à définir |

---

## 7. La partie IA et data

> [!note] Ce que l'équipe prévoit
> Entraîner des modèles (identification, etc.), avec une partie **data science** : analyse et nettoyage des données.

Pistes concrètes :
- **Modèle de détection spécialisé.** Un modèle générique (entraîné sur des objets courants) reconnaît personne, vélo, voiture, banc, feu… mais **pas** les bornes, les branches, les marches ou les bouches de métro. Il faut constituer un jeu de données de photos de trottoir, l'annoter, le nettoyer et entraîner le modèle.
- **Lecture de texte** ([[OCR]]) : numéros de bus, plaques de rue, enseignes (problème 4).
- **Estimation de profondeur à une seule caméra** (famille *Depth Anything*) : l'IA remplace le LiDAR. C'est plus difficile, et il faut vérifier si c'est assez rapide sur un Raspberry Pi.
- **Suivi d'objets** et estimation de vitesse, pour évaluer le danger des objets mobiles (problème 1).

---

## 8. Simulation 3D

Une simulation interactive a été réalisée pour comprendre ce que voient les capteurs et comment exploiter leurs données :
**[Simulation Vigie](https://claude.ai/artifact/47N2ChEjwnSDc647PpwDoU)** (privée tant qu'elle n'est pas partagée via le menu *Partager*).

**Ce qu'elle montre :**
- Une rue réaliste : immeubles, commerces, abri de bus, bouche de métro, passage piéton avec bande d'éveil de vigilance, feu piéton, piéton, cycliste.
- Les rayons du LiDAR et le nuage de points, avec plusieurs modèles de capteurs à comparer.
- La vue caméra avec les objets reconnus (modèle générique ou spécialisé).
- Une vue de dessus des secteurs, la liste des dangers et les vibreurs animés.
- Un porteur qui **réagit** : il s'arrête pour un vélo, contourne un obstacle puis revient dans son axe, ralentit devant une marche, et sa canne touche les obstacles bas.
- Les annonces vocales dans une bulle, et un bouton pour interroger les lunettes (touche Q).
- Un mode nuit : la caméra devient aveugle, le LiDAR non.

**Situations à tester :** borne basse, branche à hauteur de tête, panneau suspendu, escalier qui descend, vitre et voiture noire, piéton qui arrive, bouche de métro, vélo qui coupe, de nuit.

---

## 9. Code de principe (Python, Raspberry Pi)

### LiDAR → vibreurs

```python
import math
ALERTE, CRITIQUE = 2.5, 0.6          # mètres

def classer(h):
    if h < -0.10: return "descente"
    if h <= 0.10: return "sol"
    if h <= 1.25: return "obstacle"
    if h <= 2.10: return "tete"
    return "au-dessus"

def trame_vers_vibreurs(dist, azim, elev, tangage, h_capteur=1.65):
    # dist en m (None = pas de retour), angles en radians,
    # tangage = inclinaison de la tête lue sur l'IMU
    proche = {"G": 99, "C": 99, "D": 99}
    for d, az, el in zip(dist, azim, elev):
        if d is None: continue
        a = el + tangage
        h  = h_capteur + d * math.sin(a)   # hauteur du point
        dh = d * math.cos(a)               # distance au sol
        if classer(h) in ("obstacle", "tete", "descente"):
            s = "G" if az < -0.26 else "D" if az > 0.26 else "C"
            proche[s] = min(proche[s], dh)
    i = {s: max(0, min(1, (ALERTE - d) / (ALERTE - CRITIQUE)))
         for s, d in proche.items()}
    return max(i["G"], i["C"]), max(i["D"], i["C"])
```

### Fusion caméra + LiDAR

```python
def fusion(detections, points):
    # detections : [(nom, x0, y0, x1, y1)] cadres dans l'image
    # points : [(u, v, distance)] points LiDAR projetés dans l'image
    # (il faut calibrer caméra et LiDAR l'un par rapport à l'autre)
    objets = []
    for nom, x0, y0, x1, y1 in detections:
        d = [p[2] for p in points
             if x0 <= p[0] <= x1 and y0 <= p[1] <= y1]
        objets.append((nom, min(d) if d else None))
    return objets
```

### Phrase pour la voix

```python
def heure(angle_deg):              # 0° = droit devant, positif = à droite
    h = max(-3, min(3, round(angle_deg / 30)))
    return "droit devant" if h == 0 else f"à {h if h > 0 else 12 + h} heures"

def pas(d):
    n = max(1, round(d / 0.7))
    return "à un pas" if n == 1 else f"à {n} pas"

# "Attention, panneau à hauteur de visage, droit devant, à 3 pas."
```

---

## 10. Questions ouvertes et prochaines étapes

- [ ] **Choisir le problème prioritaire** parmi les quatre.
- [ ] Choisir le nom du projet.
- [ ] Rédiger le livrable n°1 : Design Thinking et cahier des charges.
- [ ] Choisir le LiDAR définitif (VL53L8CX seul ou en éventail ?) et la caméra.
- [ ] Définir la sortie audio (écouteur, haut-parleur, conduction osseuse ?).
- [ ] Vérifier les performances de la vision sur Raspberry Pi (images/s, besoin d'un accélérateur).
- [ ] Constituer et nettoyer un premier jeu de données de photos de trottoir.
- [ ] Rencontrer des personnes aveugles ou des instructeurs en locomotion pour valider les règles d'alerte (urgences, format en pas et en heures).

---

## Sources

- [ST — VL53L7CX / VL53L8CX (fiche)](https://www.st.com.cn/resource/en/flyer/vl53l7cx-vl53l8cx-flyer.pdf)
- [Pololu — VL53L7CX carrier](https://www.pololu.com/product/3418)
- [Pimoroni — Arducam Time of Flight Camera](https://shop.pimoroni.com/products/time-of-flight-camera-for-raspberry-pi)
- [Core Electronics — Sipeed MaixSense A010](https://core-electronics.com.au/sipeed-maixsense-a010-with-lcd-screen-3d-sensor-module-based-on-bl702-and-tof-camera.html)
- [Hackaday — « Radar » Glasses (VL53L1X + vibreur)](https://hackaday.com/2024/01/19/radar-glasses-grant-vision-free-distance-sensing)
