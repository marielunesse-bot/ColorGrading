# ColorGrading

**Auteur** : TEUGUIA TADJUIDJE RODOLF SÉDÉRIS  
**Projet** : #24 - Color Grading Tool  
**Cours** : Conception d'Outils  
**Date** : Janvier 2026  

---

## Description du Projet

Un outil professionnel de retouche d'image permettant d'appliquer des transformations de couleur en temps réel. L'application offre une interface intuitive pour ajuster la luminosité, le contraste, la saturation, éditer des courbes de couleur personnalisées et appliquer des LUT 3D.

---

## Fonctionnalités

### Chargement et Export d'Images
- **Formats supportés** : PNG, JPG, JPEG, BMP
- **Dialogue de fichier natif** : Parcourir le PC pour sélectionner n'importe quelle image
- **Export personnalisé** : Choisir le nom et l'emplacement de sauvegarde

### Transformations de Base
- **Brightness** (Luminosité) : -1.0 à +1.0
- **Contrast** (Contraste) : -1.0 à +1.0
- **Saturation** : 0.0 (noir et blanc) à 2.0 (oversaturé)
- **Application en temps réel** avec prévisualisation instantanée

### Éditeur de Courbes
- **4 courbes indépendantes** : RGB (Master), Red, Green, Blue
- **Éditeur visuel interactif** :
  - Double-clic pour ajouter un point de contrôle
  - Glisser-déposer pour déplacer les points
  - Suppression de points
  - Reset à la courbe linéaire
- **Interpolation linéaire** entre les points
- **Affichage coloré** : courbes en rouge, vert, bleu selon le canal

### Support LUT 3D
- **Format** : Fichiers .cube (Adobe standard)
- **Interpolation trilinéaire** pour application précise
- **Chargement** depuis n'importe quel emplacement
- **Affichage** des informations (nom, taille)

---

## Architecture Technique

### Structure du Projet
```
color_grading_tool/
├── src/
│   ├── main.cpp              # Point d'entrée
│   ├── App.hpp/cpp           # Contrôleur principal (MVC)
│   ├── GUI.hpp/cpp           # Interface utilisateur (Vue)
│   ├── Core/                 # Logique métier (Modèle)
│   │   ├── ColorGrading.hpp/cpp    # Moteur de transformations
│   │   ├── CurveEditor.hpp/cpp     # Gestion des courbes
│   │   └── LUTManager.hpp/cpp      # Gestion des LUT 3D
│   └── Utils/                # Utilitaires
│       ├── ImageLoader.hpp/cpp     # Chargement/sauvegarde images
│       └── (autres helpers)
├── thirdparty/               # Bibliothèques externes
│   ├── imgui/                # Interface graphique
│   ├── SDL3/                 # Fenêtrage et rendu
│   ├── stb/                  # stb_image, stb_image_write
│   └── tinyfiledialogs       # Dialogues de fichiers natifs
├── build.py                  # Système de build Python
└── README.md                 # Ce fichier
```

### Pattern MVC (Model-View-Controller)

- **Model** : `Core/` - Gère les données et la logique métier
- **View** : `GUI.cpp` - Interface utilisateur ImGui
- **Controller** : `App.cpp` - Coordination entre Model et View

### Principes de Conception

**1. Séparation des Responsabilités**
- GUI ne contient aucune logique métier
- Core ne connaît pas l'interface
- Utils fournit des services réutilisables

**2. RAII (Resource Acquisition Is Initialization)**
```cpp
std::unique_ptr<ColorGrading> m_colorGrading;  // Gestion automatique de la mémoire
```

**3. Structures de Données Justifiées**
- **Image en `float[0,1]`** : Évite les pertes de précision lors des transformations multiples
- **LUT en vecteur 1D** : Meilleure localité de cache (`index = (r*size² + g*size + b)*3`)
- **Points de contrôle triés** : Recherche O(n) pour évaluation de courbe

---

## Compilation et Exécution

### Prérequis

- **Python 3.x** (pour le build system)
- **clang++** (compilateur)
- **Windows** (pour cette version)

### Build System

Le projet utilise un **système de build Python** multiplateforme :
```bash
# Compiler le projet
python build.py build

# Lancer l'application
python build.py run

# Nettoyer les fichiers de build
python build.py clean

# Recompiler from scratch
python build.py rebuild
```

### Détails du Build System

**Fichier** : `build.py`

**Caractéristiques** :
- Détection automatique de la plateforme (Windows/macOS/Linux)
- Utilisation exclusive de `clang++` (requis par le prof)
- Gestion automatique des dépendances
- Scripts simples : `build`, `run`, `clean`, `rebuild`

**Compilation** :
```bash
clang++ -std=c++17 -O2 -g \
  -I src -I thirdparty/imgui -I thirdparty/SDL3/include \
  [tous les .cpp] \
  -lSDL3 -lcomdlg32 [autres libs] \
  -o ColorGradingTool.exe
```

---

## Algorithmes Implémentés

### 1. Transformations de Couleur

**Brightness** (Luminosité)
```cpp
// Addition simple sur les canaux RGB
r += brightness;
g += brightness;
b += brightness;
```

**Contrast** (Contraste)
```cpp
// Multiplication autour du point médian (0.5)
float factor = (contrast + 1.0f);
r = (r - 0.5f) * factor + 0.5f;
g = (g - 0.5f) * factor + 0.5f;
b = (b - 0.5f) * factor + 0.5f;
```

**Saturation**
```cpp
// Conversion RGB → HSV → modification S → RGB
rgbToHsv(r, g, b, h, s, v);
s *= saturation;  // Multiplier la saturation
hsvToRgb(h, s, v, r, g, b);
```

### 2. Courbes de Couleur

**Interpolation Linéaire**
```cpp
// Entre deux points de contrôle p0 et p1
float t = (x - p0.x) / (p1.x - p0.x);
return p0.y + t * (p1.y - p0.y);
```

**Génération de LUT**
```cpp
// Pré-calcul d'une LUT 256 valeurs pour performance
std::vector<float> lut(256);
for (int i = 0; i < 256; ++i) {
    float x = i / 255.0f;
    lut[i] = evaluate(x);  // Évalue la courbe en x
}
```

### 3. LUT 3D

**Interpolation Trilinéaire**
```cpp
// Dans un cube 3D de taille size×size×size
// 8 coins interpolés pour obtenir la valeur finale
// Complexité : O(1)
```

**Formule d'index** :
```
index = (r * size² + g * size + b) * 3
```

---

## Utilisation

### Interface

L'application présente **3 onglets** :

#### Onglet "Basic"
- Sliders pour Brightness, Contrast, Saturation
- Bouton "Reset All" pour revenir aux valeurs par défaut

#### Onglet "Curves"
- Sélection du canal (RGB, Red, Green, Blue)
- **Grille interactive** avec courbe affichée
- **Double-clic** sur la courbe → Ajouter un point
- **Clic + glisser** → Déplacer un point (devient jaune)
- **Bouton "Remove Point"** → Supprimer le point sélectionné
- **Bouton "Reset Curve"** → Remettre la courbe linéaire

#### Onglet "LUT"
- Affichage des informations sur la LUT chargée
- Bouton "Clear LUT" pour désactiver la LUT

### Menu "File"

- **Open Image...** : Dialogue natif pour charger une image
- **Load LUT...** : Charger un fichier .cube
- **Export Image...** : Dialogue natif pour sauvegarder (choix du nom et emplacement)
- **Reset Image** : Revenir à l'image originale
- **Quit** : Fermer l'application

---

## Bibliothèques Utilisées

| Bibliothèque | Version | Usage |
|--------------|---------|-------|
| **ImGui** | 1.90+ | Interface graphique |
| **SDL3** | 3.x | Fenêtrage et rendu |
| **stb_image** | 2.28 | Chargement d'images |
| **stb_image_write** | 1.16 | Sauvegarde d'images |
| **tinyfiledialogs** | 3.15+ | Dialogues de fichiers natifs |

### Pourquoi ces choix ?

- **ImGui** : Léger, rapide, parfait pour des outils
- **SDL3** : Cross-platform, moderne, bien documenté
- **stb** : Header-only, facile à intégrer, fiable
- **tinyfiledialogs** : Dialogues natifs Windows sans dépendances lourdes

---

## Points Clés pour la Présentation

### Architecture (30% de la note)

**Séparation claire** : GUI / Core / Utils selon le principe de Single Responsibility  
**RAII** : Smart pointers (`std::unique_ptr`) pour éviter les fuites mémoire  
**Algorithmes documentés** : Chaque transformation expliquée  
**Structures de données justifiées** :
- Image en float pour la précision
- LUT 1D pour la performance (cache-friendly)
- Points de contrôle triés pour l'évaluation O(n)

### Qualité du Code (25%)

**Code auto-documenté** : Noms explicites (`m_brightness`, `applyTransformations`)  
**Commentaires pertinents** : Explication du "pourquoi", pas du "quoi"  
**Gestion d'erreurs robuste** : Vérifications, messages d'erreur clairs  
**Style cohérent** : Indentation, conventions de nommage

### Interface Utilisateur (20%)

**Intuitivité** : Organisation logique en onglets  
**Feedback visuel** : Point jaune pour sélection, courbes colorées  
**Performance** : 60+ FPS grâce aux LUT pré-calculées  
**Thème cohérent** : ImGui Dark avec couleurs personnalisées

### Système de Build (15%)

**Python multiplateforme** : Détection automatique Windows/macOS/Linux  
**Utilisation exclusive de clang++** (requis)  
**Gestion automatique des dépendances**  
**Commandes simples** : `build`, `run`, `clean`, `rebuild`

### Innovation (10%)

**Problème** : Les outils de color grading sont souvent complexes ou payants  
**Solution** : Outil léger, rapide, avec dialogues natifs pour UX fluide  
**Extensions possibles** : Histogramme, undo/redo, presets de courbes

---

## Choix Techniques Détaillés

### Pourquoi float[0,1] pour les images ?

**Problème avec unsigned char** :
```
Original: 128
Brightness +0.3: 128 + 77 = 205
Contrast +0.5: (205 - 128) * 1.5 + 128 = 243
// Perte de précision à chaque transformation
```

**Avec float** :
```
Original: 0.5
Brightness +0.3: 0.5 + 0.3 = 0.8
Contrast +0.5: (0.8 - 0.5) * 1.5 + 0.5 = 0.95
// Aucune perte jusqu'à l'export final
```

### Pourquoi générer des LUT pour les courbes ?

**Sans LUT** :
```cpp
// Pour chaque pixel (1920×1080 = 2M pixels)
for (pixel in image) {
    value = evaluateCurve(pixel.r);  // Recherche dans les points
}
// Complexité : O(pixels × points) = millions d'opérations
```

**Avec LUT** :
```cpp
// Pré-calcul une seule fois
lut = generateLUT(256);

// Pour chaque pixel
for (pixel in image) {
    value = lut[pixel.r];  // Simple lookup O(1)
}
// Complexité : O(256) + O(pixels) = beaucoup plus rapide !
```

---

## Performance

- **Chargement d'image** : < 100ms pour une image 4K
- **Application de transformations** : Temps réel (< 16ms pour 60 FPS)
- **Génération de LUT** : < 1ms
- **Mémoire** : ~50 MB pour une image 4K en float

---

## Extensions Possibles

1. **Histogramme** : Affichage de la distribution RGB
2. **Undo/Redo** : Pile de transformations
3. **Presets** : Sauvegarder/charger des configurations
4. **Courbes de Bézier** : Interpolation plus smooth
5. **Batch Processing** : Appliquer les mêmes réglages sur plusieurs images
6. **Export de LUT** : Sauvegarder les courbes en .cube

---

## Licence

Projet académique - TEUGUIA TADJUIDJE RODOLF SÉDÉRIS © 2026

---

## Remerciements

- **ImGui** (Omar Cornut) : Interface graphique
- **SDL** (Sam Lantinga) : Framework multimédia
- **stb** (Sean Barrett) : Bibliothèques image
- **tinyfiledialogs** (Guillaume Vareille) : Dialogues de fichiers

---

**Projet réalisé dans le cadre du cours de Conception d'Outils**  
**TEUGUIA TADJUIDJE RODOLF SÉDÉRIS - 2026**
