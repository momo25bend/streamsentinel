# StreamSentinel

IA d'aide à l'évaluation des cours d'eau urbains — OneAquaHealth IEEE Global Hackathon 2026
**Track 3 : AI-Assisted Stream Assessment**

## Le problème

Les citoyens qui signalent l'état d'un cours d'eau produisent des données
hétérogènes et difficiles à exploiter. StreamSentinel analyse une photo,
pré-remplit le formulaire d'observation et estime un niveau de risque pour les
humains et les animaux, en croisant l'image avec des données publiques (météo,
Hub'Eau). L'humain garde la décision finale : toute alerte est validée par un
gestionnaire avant envoi.

## Jour 1 — Analyse de l'eau

Pipeline **sans aucun entraînement**, fondé sur des modèles pré-entraînés et du
traitement d'image.

| Étape | Méthode |
|---|---|
| Contrôle qualité | Variance du Laplacien (flou), luminosité moyenne |
| Segmentation | SegFormer (ADE20K) → eau / végétation / bâti / sol / ciel |
| Type de berge | Analyse de l'anneau de pixels autour de l'eau |
| Ombrage de la rive | Part de canopée dans la moitié haute de l'image |
| Turbidité, eau verte, mousse, irisation | Analyse HSV + texture sur les pixels d'eau |

Sur 11 photos annotées à la main : berge 90,9 %, turbidité 81,8 %, eau verte
90,9 %, mousse 90,9 %. Le contrôle qualité rejette correctement les photos
floues, sous-exposées et sans cours d'eau (7/7).

## Jour 2 — Validation, RGPD et guidage

| Étape | Méthode |
|---|---|
| Validation de la scène | CLIP zero-shot, avec scènes pièges (piscine, mer, rue, parking, portrait) |
| Vérification de la description | Mots-clés français → test binaire CLIP, élément par élément |
| Anonymisation | Prévue dans le pipeline (voir limites) |
| Messages de reprise | Une consigne concrète par cause de rejet |

Sur 21 photos : 13 acceptées, 8 rejetées, conformément à l'attendu.

Exemple — l'utilisateur déclare « un poisson mort au bord » sur une photo qui
n'en contient pas : la photo est acceptée, mais l'app répond « Nous ne
retrouvons pas sur la photo : poisson. Prenez une seconde photo plus proche de
cet élément. »

## Jour 3 — Détection d'objets

**Détecteur de déchets entraîné** (YOLO11s affiné sur 2 058 images, 40 epochs) :

| Classe | Précision | Rappel | mAP50 |
|---|---|---|---|
| plastic | 0,896 | 0,897 | 0,925 |
| can | 0,928 | 0,815 | 0,868 |
| bottle | 0,849 | 0,806 | 0,837 |
| carton | 0,827 | 0,680 | 0,745 |
| paper | 0,769 | 0,571 | 0,603 |
| **Global** | **0,854** | **0,754** | **0,796** |

Mesuré sur 599 images de validation indépendantes.

**Détection zero-shot (OWLv2)** pour les catégories sans dataset : buse de
rejet, faune, débris sanitaire.

**Garde-fou par masque d'eau** : les détections dont la présence n'a de sens que
dans l'eau sont confrontées au masque de segmentation du J1.

### Décisions de conception

- **Filtrage des détections zero-shot.** OWLv2 produisait 18 à 33 détections par
  photo à seuil 0,15. Trois mesures : seuils relevés à 0,30-0,35 par classe,
  phrases décrivant l'objet et non la scène, rejet des boîtes couvrant plus de
  25 % de l'image. Résultat : 1 à 8 détections par photo, sans perte sur les
  vrais objets.
- **Classe « ouvrage » retirée** : le masque du J1 détecte déjà la berge
  bétonnée avec 90,9 % de précision.
- **Détection de poissons morts écartée.** Le modèle public
  `dead-fish-ye77u/30` (mAP50 annoncé : 88,8 %) signalait un garde-corps (0,54)
  et des nuages (0,66) comme poissons morts sur nos photos. Ses performances
  sont mesurées sur un jeu de test qui ne ressemble pas à nos conditions de
  prise de vue. Une fausse alerte déclencherait une notification aux
  gestionnaires : le coût d'erreur justifie de ne pas l'intégrer sans
  validation. Piste retenue : confirmation par modèle vision-langage.

## Limites connues

- **Échantillon réduit** : les seuils du J1 et du J2 sont calés sur une
  vingtaine de photos. Ces chiffres sont indicatifs, pas une validation. Les
  résultats du J3 (mAP50) sont en revanche mesurés sur un jeu indépendant.
- L'écume d'une cascade est parfois confondue avec de la mousse.
- Une eau sombre et peu profonde est parfois classée comme sol.
- **Floutage RGPD non actif** : les cascades de Haar ont été retirées
  d'OpenCV 5. L'architecture le prévoit (appel avant toute analyse) ; une
  version en production utiliserait un détecteur de visages dédié.
- CLIP est entraîné en anglais : les descriptions françaises passent par une
  table de mots-clés, qui couvre les cas courants mais pas tout le vocabulaire.
- Une photo a été retirée du jeu de test (`eau_verte_00.jpg`) : image satellite
  récupérée par erreur lors de la collecte Wikimedia.

## Suite prévue

- **J4** : modèle vision-langage pour arbitrer les cas ambigus (écume ou mousse,
  poisson mort ou débris) et pré-remplir le formulaire d'observation
- **J5** : croisement avec Open-Meteo et Hub'Eau, score de risque
  humains/animaux, prédiction à 3 jours
- **J6** : évaluation globale et réglage final des seuils
- **Interface** : application mobile de capture guidée, tableau de bord
  gestionnaire avec file de validation des alertes

## Données

- Photos de test : Wikimedia Commons (voir `credits_photos.csv`)
- Déchets flottants : [Roboflow Universe, RF100-VL](https://universe.roboflow.com/rf100-vl/floating-waste-8deje-lrbq) (MIT)

## Exécution

Ouvrir les notebooks dans Google Colab (GPU T4), dans l'ordre J1, J2, J3. Une
clé API Roboflow est requise, à placer dans les Secrets Colab sous le nom
`ROBOFLOW_API_KEY`.
