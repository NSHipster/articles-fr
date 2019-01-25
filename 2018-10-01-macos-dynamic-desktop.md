---
title: macOS Dynamic Desktop
author: Mattt
translator: Vincent Pradeilles
category: ""
excerpt: >
  Le mode d'apparence sombre est un des ajouts à macOS le plus
  populaire --- tout particulièrement chez les développeurs.
  Suite logique de cette fonctionnalité et de Night Shift,
  qui existe depuis déjà 2 ans, les fonds d'écran dynamiques
  font leur apparition sur macOS Mojave.
status:
  swift: 4.2
---

Le mode d'apparence sombre est un des ajouts à macOS le plus
populaire --- tout particulièrement chez les développeurs,
qui ont une préférence naturelle pour les thèmes sombres
des éditeurs de texte et apprécieront la cohérence visuelle
à travers le système d'exploitation.

Il y a deux ans de cela, Night Shift avait généré un engouement
similaire, car il permettait de réduire la fatigue oculaire lors
d'une utilisation tard dans la nuit (ou plutôt, très tôt le matin).

Suite logique de ces deux fonctionnalités de macOS, les fonds d'écran
dynamiques débarquent sur macOS Mojave.
Désormais, lorsque vous accédez à "Préférences Système > Bureau et économiseur d'écran",
vous avez l'option de sélectionner un fond d'écran "dynamique", qui évolue
tout au long de la journée, en fonction de votre position géographique.

{% asset desktop-and-screen-saver-preference-pane.png %}

Le résultat est à la fois subtil et satisfaisant.
Avoir un arrière-plan qui suit l'écoulement du temps rend le bureau
plus vivant, plus en phase avec le monde extérieur.
(De plus, cela permet un effet visuel très réussi lors du passage du
thème clair au thème sombre)

_Mais comment cela fonctionne-t-il, exactement ?_<br/>
C'est la question que cherche à resoudre cet article de NSHipster.

La réponse va nous demander d'explorer en profondeur les formats
d'image, de pratiquer l'ingénierie inversée, et fera même intervenir
de un peu trigonométrie dans l'espace.

---

La première étape pour comprendre comment marchent les fonds d'écran
dynamiques est de récupérer une de leurs images dynamiques.

Si votre Mac tourne sous macOS Mojave, ouvrez le Finder,
sélectionnez "Aller > Aller au dossier..." (<kbd>⇧</kbd><kbd>⌘</kbd><kbd>G</kbd>),
puis saisissez "/Library/Desktop Pictures/".

{% asset go-to-library-desktop-pictures.png %}

Dans ce répertoire, vous devriez trouver un fichier nommé "Mojave.heic".
Double-cliquez dessus pour l'ouvrir dans Aperçu.

{% asset mojave-heic.png %}

Dans Aperçu, la barre latérale montre une liste de miniatures, numérotées
de 1 à 16, montrant chacune une différente vue de cette scène désertique.

{% asset mojave-dynamic-desktop-images.png %}

Si nous sélectionnons "Outils > Afficher l'inspecteur" (<kbd>⌘</kbd><kbd>I</kbd>),
nous obtenons quelques informations d'ordre général :

{% asset mojave-heic-preview-info.png %}

Malheureusement, c'est à peu près tout ce qu'Aperçu est capable de nous fournir
(du moins au moment où cet article est écrit).
Si nous cliquons sur l'onglet "En savoir plus", nous n'en apprenons pas
davantage :

|                   |             |
| ----------------- | ----------- |
| Mode de couleurs  | RGB         |
| Profondeur:       | 8           |
| Hauteur en pixels | 2 880       |
| Largeur en pixels | 5 120       |
| Nom du profil     | Afficher P3 |

{% info %}

L'extension de fichier `.heic` correspond à un container d'images
encodées en utilisant <abbr title="High-Efficiency Image File Format">HEIF</abbr>
ou High-Efficiency Image File Format (qui est lui même basé sur <abbr title="High-Efficiency Video Compression">HEVC</abbr>
ou High-Efficiency Video Compression, plus connu sous l'appellation H.265 vidéo).
Pour plus d'informations sur le sujet, vous pouvez consulter : [WWDC 2017 Session 503 "Introducing HEIF and HEVC"](https://developer.apple.com/videos/play/wwdc2017/503/)

{% endinfo %}

Si nous souhaitons en apprendre plus,
nous allons devoir retrousser nos manches,
et mettre les mains dans le camboui des API de bas niveau.

## En Apprendre Plus Grâce à CoreGraphics

Commençons notre enquête par la création d'un nouveau Playground sur Xcode.
Pour plus de facilité, nous pouvons écrire en dur une URL vers le fichier "Mojave.heic".

```swift
import Foundation
import CoreGraphics

// macOS 10.14 Mojave Required
let url = URL(fileURLWithPath: "/Library/Desktop Pictures/Mojave.heic")
```

Ensuite, créons une `CGImageSource`,
copions ses méta-données,
et énumérons un par un ses attributs :

```swift
let source = CGImageSourceCreateWithURL(url as CFURL, nil)!
let metadata = CGImageSourceCopyMetadataAtIndex(source, 0, nil)!
let tags = CGImageMetadataCopyTags(metadata) as! [CGImageMetadataTag]
for tag in tags {
    guard let name = CGImageMetadataTagCopyName(tag),
        let value = CGImageMetadataTagCopyValue(tag)
    else {
        continue
    }

    print(name, value)
}
```

Quand nous exécutons ce code, nous obtenons deux résultats :
`hasXMP`, qui possède la valeur `"True"`,
et `solar`, qui possède une valeur bien plus cryptique :

```
YnBsaXN0MDDRAQJSc2mvEBADDBAUGBwgJCgsMDQ4PEFF1AQFBgcICQoLUWlRelFh
UW8QACNAcO7vOubr3yO/1e+pmkOtXBAB1AQFBgcNDg8LEAEjQFRxqCKOFiAjwCR6
waUkDgHUBAUGBxESEwsQAiNAVZV4BI4c+CPAEP2uFrMcrdQEBQYHFRYXCxADI0BW
tALKmrjwIz/2ObLnx6l21AQFBgcZGhsLEAQjQFfTrJlEjnwjQByrLle1Q0rUBAUG
Bx0eHwsQBSNAWPrrmI0ISCNAKiwhpSRpc9QEBQYHISIjCxAGI0BgJff9KDpyI0BE
NTOsilht1AQFBgclJicLEAcjQGbHdYIVQKojQEq3fAg86lXUBAUGBykqKwsQCCNA
bTGmpC2YRiNAQ2WFOZGjntQEBQYHLS4vCxAJI0BwXfII2B+SI0AmLcjfuC7g1AQF
BgcxMjMLEAojQHCnF6YrsxcjQBS9AVBLTq3UBAUGBzU2NwsQCyNAcTcSnimmjCPA
GP5E0ASXJtQEBQYHOTo7CxAMI0BxgSADjxK2I8AoalieOTyE1AQFBgc9Pj9AEA0j
QHNWsnnMcWIjwEO+oq1pXr8QANQEBQYHQkNEQBAOI0ABZpkFpAcAI8BKYGg/VvMf
1AQFBgdGR0hAEA8jQErBKblRzPgjwEMGElBIUO0ACAALAA4AIQAqACwALgAwADIA
NAA9AEYASABRAFMAXABlAG4AcAB5AIIAiwCNAJYAnwCoAKoAswC8AMUAxwDQANkA
4gDkAO0A9gD/AQEBCgETARwBHgEnATABOQE7AUQBTQFWAVgBYQFqAXMBdQF+AYcB
kAGSAZsBpAGtAa8BuAHBAcMBzAHOAdcB4AHpAesB9AAAAAAAAAIBAAAAAAAAAEkA
AAAAAAAAAAAAAAAAAAH9
```

### Éclairons le Rôle de `solar`

Confrontés à cette énigme, certains pourraient être tentés
d'abandonner.
Mais, comme d'autres pourront le remarquer, ce texte ressemble
étrangement à un [encodage en Base64](https://en.wikipedia.org/wiki/Base64).

Testons notre hypothèse avec ce code :

```swift
if name == "solar" {
    let data = Data(base64Encoded: value)!
    print(String(data: data, encoding: .ascii))
}
```

<samp>
bplist00Ò\u{01}\u{02}\u{03}...
</samp>

Qu'est-ce donc ?
`bplist`, suivi d'un contenu des plus confus ?

Bon sang, ils s'agit de l'[identificateur](https://en.wikipedia.org/wiki/File_format#Magic_number) d'un fichier [`.plist` binaire](https://en.wikipedia.org/wiki/Property_list).

Voyons si `PropertyListSerialization` peut nous aider à y voir plus clair...

```swift
if name == "solar" {
    let data = Data(base64Encoded: value)!
    let propertyList = try PropertyListSerialization
                            .propertyList(from: data,
                                          options: [],
                                          format: nil)
    print(propertyList)
}
```

```
(
    ap = {
        d = 15;
        l = 0;
    };
    si = (
        {
            a = "-0.3427528387535028";
            i = 0;
            z = "270.9334057827345";
        },
        ...
        {
            a = "-38.04743388682423";
            i = 15;
            z = "53.50908581251309";
        }
    )
)
```

_Les affaires reprennent !_

Nous avons deux clés de premier niveau :

La clé `ap` correspond à un dictionnaire d'entiers associés aux
clés `d` et `l`.

La clé `si` correspond à un tableau de dictionnaires, contenants
des nombres entiers et flottants.
Dans ces dictionnaires, `i` est la valeur la plus simple à comprendre :
allant de 0 à 15, il s'agit de l'index de l'image dans la séquence.
Il serait difficile de deviner le sens de `a` et `z` sans information
additionnelle, mais il se trouve qu'ils représentent l'altitude (`a`)
et l'azimuth (`z`) du soleil pour chacune des images.

### Calculer la Position du Soleil

Au moment où cet article est écrit,
ceux d'entre-nous qui sont dans l'hémisphère nord
voient arriver le début de l'hiver, et de ses jours
plus courts et plus froids, alors que ceux de
l'hémisphère sud se préparent pour des températures
plus chaudes et des jours plus longs.
Le changement de saison nous rappelle que la durée
d'une journée dépend de l'endroit de la planète où
l'on se trouve, ainsi que de la position de la planète
sur son orbite autour du soleil.

La bonne nouvelle est que les astronomes sont capables
de prédire --- avec une précision parfaite --- la
position du soleil dans le ciel à tout moment et en
tout lieu.
La mauvaise nouvelle est que les calculs nécessaires
sont pour le moins [complexes](https://en.wikipedia.org/wiki/Position_of_the_Sun).

Toutefois, nous n'avons pas réellement besoin de les
comprendre par nous-mêmes, car nous nous contenterons
de reprendre du code trouvé sur Internet.
Après quelques recherches, nous sommes parvenus à
mettre la main sur [quelque chose qui semble fonctionner](https://github.com/NSHipster/DynamicDesktop/blob/master/SolarPosition.playground)
(les contributions sont les bienvenues !) :

```swift
import Foundation
import CoreLocation

// Apple Park, Cupertino, CA
let location = CLLocation(latitude: 37.3327, longitude: -122.0053)
let time = Date()

let position = solarPosition(for: location, at: time)
let formattedDate = DateFormatter.localizedString(from: time,
                                                    dateStyle: .medium,
                                                    timeStyle: .short)
print("Solar Position on \(formattedDate)")
print("\(position.azimuth)° Az / \(position.elevation)° El")
```

<samp>
Solar Position on Oct 1, 2018 at 12:00
180.73470025840783° Az / 49.27482549913847° El
</samp>

À midi, le 1er octobre 2018,
le soleil brillait sur Apple Park depuis le sud,
à peu près à mi-chemin entre l'horizon et la verticale.

Si nous suivons l'évolution de la position du soleil durant
une journée, nous obtenons une sinusoïde, qui ne manquera pas
de nous rappeler un certain cadran de l'Apple Watch.

{% asset solar-position-watch-faces.jpg %}

### Mieux Comprendre le Format XMP

Trêves d'astronomie pour le moment.
Intéressons nous plutôt à quelque chose de plus terre-à-terre :
les (pseudo-)standards XML de métadonnées.

Vous vous rappelez de cette clé `hasXMP` vue précédemment ?

<abbr title="Extensible Metadata Platform">XMP</abbr>,
ou Extensible Metadata Platform,
est un format standard pour annoter des fichiers avec
des métadonnées.
A quoi XMP peut-il ressembler ?
Préparez-vous à l'impact :

```swift
let xmpData = CGImageMetadataCreateXMPData(metadata, nil)
let xmp = String(data: xmpData as! Data, encoding: .utf8)!
print(xmp)
```

```xml
<x:xmpmeta xmlns:x="adobe:ns:meta/" x:xmptk="XMP Core 5.4.0">
   <rdf:RDF xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#">
      <rdf:Description rdf:about=""
            xmlns:apple_desktop="http://ns.apple.com/namespace/1.0/">
         <apple_desktop:solar>
            <!-- (Base64-Encoded Metadata) -->
        </apple_desktop:solar>
      </rdf:Description>
   </rdf:RDF>
</x:xmpmeta>
```

_Ouch._

Mais cela ne fait pas de mal d'être aller voir.
Nous aurons besoin d'exploiter le namespace `apple_desktop`
pour produire nos propres fonds d'écran dynamiques.

## Créer Nos Propres Fonds d'Écran Dynamiques

Commençons par créer un modèle de données qui représentera un fond
d'écran dynamique :

```swift
struct DynamicDesktop {
    let images: [Image]

    struct Image {
        let cgImage: CGImage
        let metadata: Metadata

        struct Metadata: Codable {
            let index: Int
            let altitude: Double
            let azimuth: Double

            private enum CodingKeys: String, CodingKey {
                case index = "i"
                case altitude = "a"
                case azimuth = "z"
            }
        }
    }
}
```

Chaque fond d'écran dynamique comporte une liste numérotée d'images,
chacune possédant les données binaires de l'image, stockées dans un objet `CGImage`,
ainsi que les métadonnées discutées précédemment.
Nous faisons se conformer `Metadata` à `Codable` dès sa déclaration,
afin de permettre au compilateur de générer le code nécessaire à
l'adoption de ce protocole.
Nous nous en servirons lorsqu'il faudra les représenter sous forme
Base64 binaire.

### Writing to an Image Destination

En premier, nous créons un `CGImageDestination`, en spécifiant
l'URL de sortie.
Le format de fichier est `heic` et le paramètre `sourceCount`
correspond au nombre d'images qui doivent être inclues.

```swift
guard let imageDestination = CGImageDestinationCreateWithURL(
                                outputURL as CFURL,
                                AVFileType.heic as CFString,
                                dynamicDesktop.images.count,
                                nil
                             )
else {
    fatalError("Error creating image destination")
}
```

Ensuite, nous itérons sur chaque image de notre fond d'écran
dynamique.
En utilisant la méthode `enumerated()`, nous obtenons également
l'`index` de chaque image, ce qui nous permet d'écrire les
métadonnées propres à la première image :

```swift
for (index, image) in dynamicDesktop.images.enumerated() {
    if index == 0 {
        let imageMetadata = CGImageMetadataCreateMutable()
        guard let tag = CGImageMetadataTagCreate(
                            "http://ns.apple.com/namespace/1.0/" as CFString,
                            "apple_desktop" as CFString,
                            "solar" as CFString,
                            .string,
                            try! dynamicDesktop.base64EncodedMetadata() as CFString
                        ),
            CGImageMetadataSetTagWithPath(
                imageMetadata, nil, "xmp:solar" as CFString, tag
            )
        else {
            fatalError("Error creating image metadata")
        }

        CGImageDestinationAddImageAndMetadata(imageDestination,
                                              image.cgImage,
                                              imageMetadata,
                                              nil)
    } else {
        CGImageDestinationAddImage(imageDestination,
                                   image.cgImage,
                                   nil)
    }
}
```

En dépit de leur nature brute de décoffrage, les API
CoreGraphics sont plutôt simple à suivre.
La seule partie qui requiert une explication poussée est
l'appel à `CGImageMetadataTagCreate(_:_:_:_:_:)`.

A cause de la différence entre la façon dont les métadonnées sont représentées avant et après sérialisation, nous devons écrire notre propre implémentation de `Encodable` pour le type `DynamicDesktop` :

```swift
extension DynamicDesktop: Encodable {
    private enum CodingKeys: String, CodingKey {
        case ap, si
    }

    private enum NestedCodingKeys: String, CodingKey {
        case d, l
    }

    func encode(to encoder: Encoder) throws {
        var keyedContainer =
            encoder.container(keyedBy: CodingKeys.self)

        var nestedKeyedContainer =
            keyedContainer.nestedContainer(keyedBy: NestedCodingKeys.self,
                                           forKey: .ap)

        // FIXME: Not sure what `l` and `d` keys indicate
        try nestedKeyedContainer.encode(0, forKey: .l)
        try nestedKeyedContainer.encode(self.images.count, forKey: .d)

        var unkeyedContainer =
            keyedContainer.nestedUnkeyedContainer(forKey: .si)
        for image in self.images {
            try unkeyedContainer.encode(image.metadata)
        }
    }
}
```

Une fois ceci en place, nous pouvons implémenter la méthode
`base64EncodedMetadata()` comme suit :

```swift
extension DynamicDesktop {
    func base64EncodedMetadata() throws -> String {
        let encoder = PropertyListEncoder()
        encoder.outputFormat = .binary

        let binaryPropertyListData = try encoder.encode(self)
        return binaryPropertyListData.base64EncodedString()
    }
}
```

Une fois la boucle `for-in` arrivée à son terme,
toutes les images et métadonnées ont été enregistrées,
nous appelons alors `CGImageDestinationFinalize(_:)` pour
finaliser l'opération et enregistrer sur le disque.

```swift
guard CGImageDestinationFinalize(imageDestination) else {
    fatalError("Error finalizing image")
}
```

Si tout s'est déroulé comme prévu, vous devriez être l'heureux
propriétaire d'un fond d'écran dynamique flambant neuf.
Bien joué !

---

Nous adorons les fonds d'écran dynamiques de Mojave,
et nous some impatients de les voir rentrer dans les moeurs,
de la même manière que les fonds d'écran classiques
à l'époque de Windows 95.

Si cela vous intéresse,
voici quelques pistes pour aller plus loin :

### Générer Automatiquement un Fond d'Écran Dynamique depuis Photos

C'est incroyable de réaliser que quelque chose d'aussi transcendant
que le mouvement des corps célestes peut-être réduit à un système
d'équations ne faisant appel qu'à deux paramètres : le temps et le lieu.

Dans l'exemple précédent, cette information est écrite en dur, mais elle
pourrait bien sûr être automatiquement extraite depuis des images.

Par défaut, les appareils photo de la majorité des téléphones enregistrent
des [métadonnées Exif](https://en.wikipedia.org/wiki/Exif) à chaque fois qu'une
photo est prise.
Ces métadonnées incluent l'heure de la prise de vue, ainsi que les coordonnées
GPS de l'appareil.

En exploitant ces informations directement depuis les métadonnées des
images, il est possible de déterminer la position du soleil et donc
de simplifier le processus de création d'un fond d'écran dynamique à
partir d'une série de photos.

### Réaliser un Time Lapse avec son iPhone

Vous voulez mettre votre nouvel iPhone XS à profit ?
(Ou plutôt, "Vous voulez utiliser votre ancien iPhone
pour quelque chose d'utile pendant que vous trainez à
le revendre ?")

Placez votre téléphone contre une fenêtre,
branchez lui son chargeur,
passez l'app Appareil photo sur le mode "Accéléré"
et démarrez l'enregistrement.
En extrayant les images-clés de la vidéo ainsi produite,
vous pourrez réaliser votre propre et unique fond d'écran
dynamique.

Vous serez peut-être intéressé par [Skyflow](https://itunes.apple.com/us/app/skyflow-time-lapse-shooting/id937208291?mt=8)
ou tout autre app permettant la prise de photo à intervalles
réguliers.

### Générer un Paysage depuis des Données GIS

Si vous ne pouvez pas vous passer de votre téléphone pendant toute
une journée (ce qui est triste), ou si vous n'avez rien de valable à
prendre en photo (ce qui est également triste), vous n'avez qu'à
créer votre propre paysage (ce qui semble plus triste qu'il ne l'est).

En utilisant une app comme [Terragen](https://planetside.co.uk),
vous pouvez produire des paysages 3D photo-réalistes, avec un grand
contrôle sur la Terre, le Soleil et le ciel.

Vous pouvez même vous simplifier les choses en téléchargeant une
carte altimétrique depuis [le site](https://viewer.nationalmap.gov/basic/)
du département américain d'études géologiques et en l'utilisant comme
modèle pour votre projet de rendu 3D.

### Télécharger des Fond D'Écran Dynamiques Prêts à l'Emploi

Mais si vous êtes déjà bien occupé par votre travail et que vous n'avez
pas le loisir de passer du temps à prendre de belles images, vous pouvez
toujours rémunérer quelqu'un pour le faire à votre place.

Nous apprécions particulièrement l'app [24 Hour Wallpaper](https://www.jetsoncreative.com/24hourwallpaper/).
Si vous avez d'autres recommandations, [faites nous le savoir sur Twitter !](https://twitter.com/NSHipster/).
