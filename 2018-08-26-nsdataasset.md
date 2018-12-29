---
title: NSDataAsset
author: Mattt
category: Cocoa
excerpt: >
  Il existe de nombreuses techniques pour accélérer une
  requête réseau :
  compression et streaming,
  mise en cache et pré-chargement,
  partage et multiplexages de connexions,
  exécution différée ou en arrière-plan.
  Et pourtant, il existe une stratégie d'optimisation
  qui les précède et surpasse toutes :
  _ne simplement pas faire la requête._
status:
  swift: 4.2
---

Sur le web,
la vitesse n'est pas un luxe,
c'est une affaire de survie.

Des études récentes suggèrent que _toute_ latence
perceptible dans la durée de chargement d'une page ---
c'est à dire, supérieure à 400 millisecondes (littéralement
la durée d'un clignement d'oeil) ---
peut négativement impacter les taux de conversion et
d'engagement.
Pour chaque seconde supplémentaire qu'une page web prend
pour se charger, il faut s'attendre à ce que 10% des
utilisateurs reviennent en arrière ou ferment l'onglet.

Pour des grandes entreprises du web comme Google, Amazon et
Netflix, une seconde de plus ici ou là peut représenter
des _milliards_ de dollars de recettes annuelles.
Il n'est donc pas surprenant que ces mêmes entreprises
aient mis en oeuvre tant d'efforts d'ingénierie
pour rendre le web plus rapide.

Il existe de nombreuses techniques pour accélérer une
requête réseau :
compression et streaming,
mise en cache et pré-chargement,
partage et multiplexage des connexions,
exécution différée ou en arrière-plan.
Et pourtant, il existe une stratégie d'optimisation
qui les précède et surpasse toutes :
_ne simplement pas faire la requête._

Les applications mobiles, par le fait qu'elles sont téléchargées
en amont de leur utilisation, possèdent un avantage unique
par rapport aux pages web classiques.
Cette semaine sur NSHipster,
nous allons montrer comment un Asset Catalog peut être utilisé
de manière non-conventionnelle, dans le but d'améliorer l'expérience
utilisateur au premier lancement de votre app.

---

Les Asset Catalogs vous permettent de stocker et d'organiser des 
ressources en fonction des caractéristiques de l'appareil courant.
Pour une même image,
vous pouvez fournir différents fichiers, en fonction de l'appareil
(iPhone, iPad, Apple Watch, Apple TV, Mac), la résolution d'écran
(`@2x` / `@3x`), ou l'espace colorimétrique (sRGB / P3).
Pour d'autres types de ressources,
vous pouvez offrir des alternatives en fonction de la quantité
de mémoire disponible ou de la version de Metal supportée.
Il n'y a besoin que de demander une ressource par son nom,
et sa version la plus appropriée sera automatiquement fournie.

Au delà de proposer API pratique à utiliser,
les Asset Catalogs permettent aux apps d'exploiter le mécanisme
d'[apps allégées](https://help.apple.com/xcode/mac/current/#/devbbdc5ce4f),
qui résulte en des installations plus petites, qui sont optimisées pour
l'appareil de chaque utilisateur.

Les images sont de très loin le type de ressources le plus courant,
mais depuis iOS 9 et macOS El Capitan, des fichiers
JSON, XML et d'autres types données peuvent se joindre à l'aventure
par l'intermédiaire d'un [`NSDataAsset`](https://developer.apple.com/documentation/uikit/nsdataasset).

## Stocker et récupérer des données dans un Asset Catalog

À titre d'exemple,
imaginons une app iOS permettant de créer des palettes numériques de couleurs.

Pour différencier différents niveaux de gris,
nous pourrions charger une liste de couleurs et les noms qui leur sont
associés. Normalement, nous pourrions la télécharger depuis un serveur lors
du premier lancement, mais cela pourrait résulter en une mauvaise expérience
utilisateur si [des conditions réseau défavorables](https://nshipster.com/network-link-conditioner/) bloquent les fonctionnalités de l'app.
Puisqu'il s'agit d'un jeu de données relativement statique,
pourquoi ne pas l'inclure directement dans le contenu même de l'app
via un Asset Catalog ?

### Étape 1. Ajouter un nouveau Data Set à un Asset Catalog

Lorsque vous créez un nouveau projet d'app dans Xcode,
un Asset Catalog est automatiquement généré.
Sélectionnez `Assets.xcassets` dans le navigateur de projet
pour ouvrir l'éditeur d'Asset Catalog.
Cliquez sur l'icône <kbd>+</kbd> dans le coin inférieur gauche
et sélectionnez "New Data Set"

{% asset add-new-data-set.png %}

Cette action crée un nouveau sous-répertoire dans `Assets.xcassets`
qui porte l'extension `.dataset`

{% info do %}

Par défaut,
le Finder considère ces deux paquets comme des répertoires,
ce qui simplifie leur inspection et la modification de leurs contenus
si nécessaire.

{% endinfo %}

### Étape 2. Ajouter un fichier de données

Ouvrez le Finder,
naviguez jusqu'au fichier de données et faites un
glissez-déposer sur votre Data Set dans Xcode.

{% asset asset-catalog-any-any-universal.png %}

Quand vous faites ceci,
Xcode va copier le fichier dans le sous-répertoire `.dataset`
et mettre à jour le fichier de métadonnées `contents.json`
avec le nom du fichier et son [Universal Type Identifier](https://en.wikipedia.org/wiki/Uniform_Type_Identifier).

```json
{
  "info": {
    "version": 1,
    "author": "xcode"
  },
  "data": [
    {
      "idiom": "universal",
      "filename": "colors.json",
      "universal-type-identifier": "public.json"
    }
  ]
}
```

### Étape 3. Accéder aux données via NSDataAsset

Vous pouvez maintenant accéder au contenu du fichier avec
le code suivant :

```swift
guard let asset = NSDataAsset(name: "NamedColors") else {
    fatalError("Missing data asset: NamedColors")
}

let data = asset.data
```

Dans le cas de notre app,
nous pourrions faire cet appel depuis la méthode `viewDidLoad()`
d'un view controller et utiliser le résultat pour décoder un
tableau d'objets qui seront affichés dans une table view :

```swift
let decoder = JSONDecoder()
self.colors = try! decoder.decode([NamedColor].self, from: asset.data)
```

## Aller plus loin

Les jeux de données ne bénéficient habituellement pas du mécanisme de
d'app allégées que permettent les Asset Catalogs (la plupart des
fichiers JSON, par exemple, ne dépendent absolument pas de quelle
version de Metal est supportée par un appareil).

Mais pour notre app de palettes de couleurs,
nous pourrions proposer différentes listes de couleurs sur les appareils
avec un espace colorimétrique étendu.

Pour ce faire,
sélectionnez la ressource dans la barre latérale de l'éditeur d'Asset Catalog
et cliquez sur le menu déroulant intitulé "Gamut" dans l'inspecteur d'attributs.

{% asset select-color-gamut.png %}

Après avoir fourni les jeux de données appropriés pour chaque espace colorimétrique,
le fichier de métadonnées `contents.json` devrait ressembler à quelque chose
comme ça :

```json
{
  "info": {
    "version": 1,
    "author": "xcode"
  },
  "data": [
    {
      "idiom": "universal",
      "filename": "colors-srgb.json",
      "universal-type-identifier": "public.json",
      "display-gamut": "sRGB"
    },
    {
      "idiom": "universal",
      "filename": "colors-p3.json",
      "universal-type-identifier": "public.json",
      "display-gamut": "display-P3"
    }
  ]
}
```

## Conserver des données fraîches

Stocker et récupérer des données depuis un Asset Catalog est
trivial. Ce qui est réellement difficile --- et en fin de compte
plus important --- est de garder ces données à jour.

Rafraîchissez des données en utilisant `curl`, `rsync`, `sftp`,
Dropbox, BitTorrent, ou Filecoin.
Lancez le processus depuis un script Shell
(et appelez le depuis un Xcode Build Phase,
si vous voulez).
Ajoutez-le à votre `Makefile`, `Rakefile`, `Fastfile`,
ou tout autre système de compilation de votre choix.
Déléguez la tâche à Jenkins, Travis ou à un stagiaire.
Déclenchez le depuis une intégration Slack ou créez un
raccourci Siri pour étonner vos collègues avec un 
_"Dis Siri, mets à jour ce jeu de donnée avant qu'il ne pourrisse"_.

**Peu importe comment vous décidez de synchroniser vos données,
l'important est que le mécanisme soit automatisé et fasse parti
de votre processus de livraison.**

Voici un exemple d'un script shell que vous pourriez
utiliser pour télécharger la dernière version des données
en utilisant `curl` :

```shell
#!/bin/sh
CURL='/usr/bin/curl'
URL='https://example.com/path/to/data.json'
OUTPUT='./Assets.xcassets/Colors.dataset/data.json'

$CURL -fsSL -o $OUTPUT $URL
```

## Assemblons les morceaux

Bien que les Asset Catalogs réalisent une compression sans perte
des images, rien dans la documentation, l'aide d'Xcode ou les sessions
WWDC n'indique que de telles optimisations sont réalisées pour les fichiers de données
(ou du moins pas pour le moment).

Pour des jeux de données plus lourds que, disons, quelque centaines de
kilo-octets, vous devriez considérer l'utilisation d'une compression.
Cela est particulièrement vrai pour des fichiers texte comme les JSON,
CSV et XML, qui peuvent habituellement se compresser jusqu'à 60% - 80%
de leur taille d'origine.

Nous pouvons ajouter une étape de compression au script précédent
en redirigeant la sortie de `curl` vers `gzip` :

```shell
#!/bin/sh
CURL='/usr/bin/curl'
GZIP='/usr/bin/gzip'
URL='https://example.com/path/to/data.json'
OUTPUT='./Assets.xcassets/Colors.dataset/data.json.gz'

$CURL -fsSL $URL | $GZIP -c > $OUTPUT
```

Si vous décidez d'utiliser cette compression,
assurez-vous que le champ `"universal-type-identifier"`
possède bien cette valeur :

```json
{
  "info": {
    "version": 1,
    "author": "xcode"
  },
  "data": [
    {
      "idiom": "universal",
      "filename": "colors.json.gz",
      "universal-type-identifier": "org.gnu.gnu-zip-archive"
    }
  ]
}
```

Côté client,
c'est à vous qu'il revient de décompresser les données récupérées
de l'Asset Catalog avant de les utiliser.
Si vous avez un module `Gzip`, vous pourriez faire comme suit :

```swift
do {
    let data = try Gzip.decompress(data: asset.data)
} catch {
    fatalError(error.localizedDescription)
}
```

Ou, si vous faites cela à de multiples reprises dans votre app,
vous pourriez créer une méthode dans une extension de `NSDataAsset` :

```swift
extension NSDataAsset {
    func decompressedData() throws -> Data {
        return try Gzip.decompress(data: self.data)
    }
}
```

{% info do %}

Vous pourriez aussi considérer gérer de grands jeux de données
dans votre contrôle de version en utilisant 
[Git Large File Storage (LFS)](https://git-lfs.github.com).

{% endinfo %}

---

Bien qu'il soit tentant de supposer que tous vos utilisateurs
jouissent d'une connexion réseau rapide et fiable via WiFi ou 4G,
cela n'est pas vrai pour tout le monde, et ce n'est certainement
pas vrai tout le temps.

Prenez un moment pour regarder quels sont les appels réseau que
votre app réalise à son lancement et demandez-vous si certains
d'entre-eux pourraient gagner à être pré-chargés.
Faire une bonne première impression peut être la différence entre
une utilisation fidèle et assidue et une désinstallation au bout
de quelques secondes.
