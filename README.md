# streamsentinel
IA d'aide à l'évaluation des cours d'eau urbains — OneAquaHealth IEEE Hackathon 2026
# StreamSentinel

IA d'aide à l'évaluation des cours d'eau urbains — OneAquaHealth IEEE Global Hackathon 2026
**Track 3 : AI-Assisted Stream Assessment**

## Le problème

Les citoyens qui signalent l'état d'un cours d'eau produisent des données
hétérogènes et difficiles à exploiter. StreamSentinel analyse une photo,
pré-remplit le formulaire d'observation et estime un niveau de risque pour
les humains et les animaux, en croisant l'image avec des données publiques
(météo, Hub'Eau). L'humain garde la décision finale : toute alerte est
validée par un gestionnaire avant envoi.

## État d'avancement — Jour 1

Pipeline d'analyse d'image **sans aucun entraînement**, fondé sur des modèles
pré-entraînés et du traitement d'image.

| Étape | Méthode |
|---|---|
| Contrôle qualité | Variance du Laplacien (flou), luminosité moyenne |
| Segmentation de la scène | SegFormer (ADE20K) → eau / végétation / bâti / sol / ciel |
| Type de berge | Analyse de l'anneau de pixels autour de l'eau |
| Ombrage de la rive | Part de canopée dans la moitié haute de l'image |
| Turbidité, eau verte, mousse, irisation | Analyse HSV + texture sur les pixels d'eau |

## Résultats

Sur 11 photos de rivières annotées à la main :

| Détection | Précision |
|---|---|
| Type de berge | 90,9 % |
| Turbidité | 81,8 % |
| Eau verte | 90,9 % |
| Mousse | 90,9 % |

Le contrôle qualité rejette correctement les photos floues, sous-exposées et
sans cours d'eau (7/7).

## Limites connues

- **Échantillon réduit** : les seuils ont été calés sur ces 11 photos. Ces
  chiffres sont indicatifs et ne constituent pas une validation.
- L'écume d'une cascade est parfois confondue avec de la mousse.
- Une eau sombre et peu profonde est parfois classée comme sol.
- Les reflets de végétation peuvent ressembler à une efflorescence algale.

Ces cas ambigus seront arbitrés par un modèle vision-langage (étape suivante).

## Suite prévue

- Validation de la photo (cours d'eau ? cohérence avec la description)
- Floutage des visages et plaques (RGPD)
- Détection d'objets : macrodéchets, poissons morts, buses de rejet
- Croisement avec Open-Meteo et Hub'Eau, score de risque, prédiction à 3 jours

## Données

- Photos de test : Wikimedia Commons (voir `credits_photos.csv`)
- Déchets flottants : [Roboflow Universe, RF100-VL](https://universe.roboflow.com/rf100-vl/floating-waste-8deje-lrbq) (MIT)
- Poissons morts : [Roboflow Universe](https://universe.roboflow.com/test-7elgs/dead-fish-ye77u) (CC BY 4.0)

## Exécution

Ouvrir `streamsentinel_J1_fondations.ipynb` dans Google Colab (GPU T4).
Une clé API Roboflow est requise, à placer dans les Secrets Colab sous le nom
`ROBOFLOW_API_KEY`.
