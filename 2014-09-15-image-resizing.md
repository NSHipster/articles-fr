---
title: Image Resizing Techniques
author: Mattt
category: ""
excerpt: "Depuis des temps immémoriaux, les développeurs iOS sont restés perplexes
face à une interrogation singulière : "Comment peut-on redimensionner une
image ?". Cet article fera tout son possible pour fournir une réponse
claire à cette éternelle question."
status:
    swift: 2.0
    reviewed: September 30, 2015
revisions:
    "2014-09-15": Original publication.
    "2015-09-30": Revised for Swift 2.0, `vImage` method added.
---

Depuis des temps immémoriaux, les développeurs iOS sont restés perplexes
face à une interrogation singulière : "Comment peut-on redimensionner une
image ?". C'est une question d'une séduisante simplicité, stimulée par
une défiance mutuelle entre les développeurs et les plates-formes.
Un millier d'extraits de code jonchent les résultats d'une recherche
sur le web, chacun clamant être la Seule Vraie Solution, toutes
les autres étant de faux prophètes.

C'est véritablement embarrassant.

L'article de cette semaine fera tout son possible pour fournir
une explication claire des différentes approches au redimensionnement
d'images sur iOS (et sur OS X, en faisant les conversions `UIImage` → `NSImage`
appropriées), en apportant des preuves empiriques pour donner un aperçu 
des performances de chaque approche, plutôt que de d'imposer une unique 
manière peu importe les situations.

**Avant de continuer la lecture, merci de prendre en compte ceci :**

Lorsque qu'une `UIImage` est affectée à une `UIImageView`, un redimensionnement
manuel est inutile dans la vaste majorité des cas d'utilisation. À la place,
il est possible de simplement définir la propriété `contentMode` à, soit
`.ScaleAspectFit` pour s'assurer que l'image sera visible en entier dans la
frame de l'image view, soit `.ScaleAspectFill` pour que l'image view soit
entièrement remplie par l'image, en la rognant si nécessaire à partir de
son centre.

```swift
imageView.contentMode = .ScaleAspectFit
imageView.image = image
```

---

## Déterminer la nouvelle taille

Avant de procéder à un redimensionnement, il faut tout d'abord déterminer
quelle doit être la nouvelle taille.

### Redimensionner par coefficient

La manière la plus simple de réduire une image est par un coefficient
constant. Généralement, cela implique de diviser par un nombre entier la taille
originale (plutôt que de la multiplier par un tel nombre pour l'agrandir).

Une nouvelle `CGSize` peut être calculée en multipliant les composantes
de longueur et hauteur individuellement :

```swift
let size = CGSize(width: image.size.width / 2, height: image.size.height / 2)
```

...ou en appliquant une `CGAffineTransform` :

```swift
let size = CGSizeApplyAffineTransform(image.size, CGAffineTransformMakeScale(0.5, 0.5))
```

### Redimensionner par ratio d'image

Il est souvent utile de réduire la taille originale de telle manière qu'elle
s'adapte à un rectangle donné, sans modifier le ratio original de l'image.
`AVMakeRectWithAspectRatioInsideRect` est une fonction utile du framework
AVFoundation qui s'occupe de faire ce calcul à notre place :

```swift
import AVFoundation
let rect = AVMakeRectWithAspectRatioInsideRect(image.size, imageView.bounds)
```

## Redimensionner des images

Il existe nombre de méthodes différentes pour redimensionner une image,
chacune ayant des capacités et des performances propres.

### `UIGraphicsBeginImageContextWithOptions` & `UIImage -drawInRect:`

Les APIs de plus haut niveau pour redimensionner une image se trouvent dans le
framework UIKit. À partir d'une `UIImage`, un contexte graphique temporaire
peut être utilisé pour générer une version réduite, en utilisant
`UIGraphicsBeginImageContextWithOptions()` et `UIGraphicsGetImageFromCurrentImageContext()` :

```swift
let image = UIImage(contentsOfFile: self.URL.absoluteString!)

let size = CGSizeApplyAffineTransform(image.size, CGAffineTransformMakeScale(0.5, 0.5))
let hasAlpha = false
let scale: CGFloat = 0.0 // Utilise automatiquement le facteur d'agrandissement de l'écran principal

UIGraphicsBeginImageContextWithOptions(size, !hasAlpha, scale)
image.drawInRect(CGRect(origin: CGPointZero, size: size))

let scaledImage = UIGraphicsGetImageFromCurrentImageContext()
UIGraphicsEndImageContext()
```

`UIGraphicsBeginImageContextWithOptions()` crée un contexte temporaire dans lequel l'image originale est dessinée. Le premier paramètre `size`, est la taille de
l'image redimensionnée. Le second paramètre `isOpaque` sert à indiquer si un canal
alpha doit être rendu. Le définir à `false` pour les images sans transparence
(c'est à dire, sans canal alpha) peut résulter en une image avec une teinte rosée.
Le troisième paramètre `scale` est le facteur d'agrandissement de l'écran. Quand
il vaut `0.0`, le facteur d'agrandissement de l'écran principal est utilisée, qui,
pour des écrans rétina, vaut `2.0` ou plus (`3.0` sur l'iPhone 6 plus).

### `CGBitmapContextCreate` & `CGContextDrawImage`

Core Graphics / Quartz 2D propose un ensemble d'APIs bas-niveau qui autorisent
des configurations plus avancées. À partir d'une `CGImage`, un contexte bitmap
temporaire est utilisé pour générer la version réduite de l'image, en utilisant
`CGBitmapContextCreate()` et `CGBitmapContextCreateImage()` :

```swift
let cgImage = UIImage(contentsOfFile: self.URL.absoluteString!).CGImage

let width = CGImageGetWidth(cgImage) / 2
let height = CGImageGetHeight(cgImage) / 2
let bitsPerComponent = CGImageGetBitsPerComponent(cgImage)
let bytesPerRow = CGImageGetBytesPerRow(cgImage)
let colorSpace = CGImageGetColorSpace(cgImage)
let bitmapInfo = CGImageGetBitmapInfo(cgImage)

let context = CGBitmapContextCreate(nil, width, height, bitsPerComponent, bytesPerRow, colorSpace, bitmapInfo.rawValue)

CGContextSetInterpolationQuality(context, kCGInterpolationHigh)

CGContextDrawImage(context, CGRect(origin: CGPointZero, size: CGSize(width: CGFloat(width), height: CGFloat(height))), cgImage)

let scaledImage = CGBitmapContextCreateImage(context).flatMap { UIImage(CGImage: $0) }
```

`CGBitmapContextCreate` prend plusieurs paramètres pour construire un contexte
ayant les dimensions et la quantité de mémoire par canal d'un espace de couleur souhaitées. Dans cet exemple, ces valeurs sont obtenues depuis la `CGImage`. Ensuite,
`CGContextSetInterpolationQuality` permet au contexte d'interpoler les pixels à
différent niveaux de fidélités. Dans le cas présent, `kCGInterpolationHigh` est
passé pour obtenir les meilleurs résultats. `CGContextDrawImage` permet à l'image
d'être dessinées à une taille et position données, autorisant l'image a être rognée
à partir d'un de ses bords ou en fonction de son contenu, tel que des visages.
Finalement, `CGBitmapContextCreateImage` crée une `CGImage` à partir de ce contexte.

### `CGImageSourceCreateThumbnailAtIndex`

Image I/O est un puissant, bien que méconnu, framework pour travailler sur des images.
Indépendant de Core Graphics, il peut lire et écrire de nombreux formats 
différents, accéder aux métadonnées de photos, et réaliser des opérations courantes
de traitement d'images. Le framework fournit les encodeurs et décodeurs d'images
les plus rapides de la plateforme, avec des mécanismes avancés de cache, et même
la capacité de charger des images incrémentalement .

`CGImageSourceCreateThumbnailAtIndex` expose une API concise avec des options
différentes de celles des appels Core Graphics équivalents :

```swift
import ImageIO

if let imageSource = CGImageSourceCreateWithURL(self.URL, nil) {
    let options: [NSString: NSObject] = [
        kCGImageSourceThumbnailMaxPixelSize: max(size.width, size.height) / 2.0,
        kCGImageSourceCreateThumbnailFromImageAlways: true
    ]

    let scaledImage = CGImageSourceCreateThumbnailAtIndex(imageSource, 0, options).flatMap { UIImage(CGImage: $0) }
}
```

À partir d'une `CGImageSource` et d'un ensemble d'options,
`CGImageSourceCreateThumbnailAtIndex` crée une image miniature.
Le redimensionnement est accompli par `kCGImageSourceThumbnailMaxPixelSize`.
Spécifiez la dimension maximum divisée par un facteur constant réduit
l'image tout en conservant le ratio d'image original. En spécifiant
`kCGImageSourceCreateThumbnailFromImageIfAbsent` ou
`kCGImageSourceCreateThumbnailFromImageAlways`, Image I/O va automatiquement
mettre en cache les résultats pour de futurs appels.


### Ré-échantillonnage de Lanczos avec Core Image

Core Image fournit une fonctionnalité pré-intégrée de
[ré-échantillonnage de Lanczos](https://en.wikipedia.org/wiki/Lanczos_resampling)
avec le filtre `CILanczosScaleTransform`. Bien qu'étant possiblement une
API de plus haut niveau que UIKit, l'usage envahissant de key-value coding
dans Core Image le rend peu agréable à utiliser.

Ceci dit, son utilisation est au moins cohérente. Le processus de création d'un
filtre, sa configuration, et la génération de l'image finale se déroule comme
n'importe quel autre flux de travail de Core Image :

```swift
let image = CIImage(contentsOfURL: self.URL)

let filter = CIFilter(name: "CILanczosScaleTransform")!
filter.setValue(image, forKey: "inputImage")
filter.setValue(0.5, forKey: "inputScale")
filter.setValue(1.0, forKey: "inputAspectRatio")
let outputImage = filter.valueForKey("outputImage") as! CIImage

let context = CIContext(options: [kCIContextUseSoftwareRenderer: false])
let scaledImage = UIImage(CGImage: self.context.createCGImage(outputImage, fromRect: outputImage.extent()))
```

`CILanczosScaleTransform` prend en paramètres une `inputImage`, une `inputScale`
et un `inputAspectRatio`, dont les noms sont assez explicites. Un `CIContext`
est utilisé pour créer une `UIImage` par l'intermédiaire d'une `CGImageRef`,
puisque `UIImage(CIImage:)` ne fonctionne souvent pas comme attendu.

Créer un `CIContext` est une opération couteuse, ainsi un contexte mis en cache
devrait toujours être utilisé pour des redimensionnements répétés. Un `CIContext`
peut être créé en utilisant soit le GPU, soit le CPU (qui est bien plus lent)
pour les opérations de calcul - utilisez la clé `kCIContextUseSoftwareRenderer`
dans le dictionnaire d'options pour indiquer lequel doit être utilisé.


### `vImage` dans Accelerate

Le [framework Accelerate](https://developer.apple.com/library/prerelease/ios/documentation/Accelerate/Reference/AccelerateFWRef/index.html#//apple_ref/doc/uid/TP40009465) inclu une série de fonctions `vImage` de traitements d'images, avec
un [ensemble de fonctions](https://developer.apple.com/library/prerelease/ios/documentation/Performance/Reference/vImage_geometric/index.html#//apple_ref/doc/uid/TP40005490-CH212-145717) capables de mettre à l'échelle un buffer d'image. Ces
APIs de bas-niveau promettent des performances élevées pour une consommation
électrique minimale, mais au prix d'une gestion manuelle des buffers. Le code
ci-dessous est une version Swift d'une méthode [suggérée par Nyx0uf sur GitHub](https://gist.github.com/Nyx0uf/217d97f81f4889f4445a) :

```swift
let cgImage = UIImage(contentsOfFile: self.URL.absoluteString!).CGImage

// création d'un buffer source
var format = vImage_CGImageFormat(bitsPerComponent: 8, bitsPerPixel: 32, colorSpace: nil,
    bitmapInfo: CGBitmapInfo(rawValue: CGImageAlphaInfo.First.rawValue),
    version: 0, decode: nil, renderingIntent: CGColorRenderingIntent.RenderingIntentDefault)
var sourceBuffer = vImage_Buffer()
defer {
    sourceBuffer.data.dealloc(Int(sourceBuffer.height) * Int(sourceBuffer.height) * 4)
}

var error = vImageBuffer_InitWithCGImage(&sourceBuffer, &format, nil, cgImage, numericCast(kvImageNoFlags))
guard error == kvImageNoError else { return nil }

// création d'un buffer de destination
let scale = UIScreen.mainScreen().scale
let destWidth = Int(image.size.width * 0.5 * scale)
let destHeight = Int(image.size.height * 0.5 * scale)
let bytesPerPixel = CGImageGetBitsPerPixel(image.CGImage) / 8
let destBytesPerRow = destWidth * bytesPerPixel
let destData = UnsafeMutablePointer<UInt8>.alloc(destHeight * destBytesPerRow)
defer {
    destData.dealloc(destHeight * destBytesPerRow)
}
var destBuffer = vImage_Buffer(data: destData, height: vImagePixelCount(destHeight), width: vImagePixelCount(destWidth), rowBytes: destBytesPerRow)

// mise à l'échelle de l'image
error = vImageScale_ARGB8888(&sourceBuffer, &destBuffer, nil, numericCast(kvImageHighQualityResampling))
guard error == kvImageNoError else { return nil }

// création d'une CGImage depuis un vImage_Buffer
let destCGImage = vImageCreateCGImageFromBuffer(&destBuffer, &format, nil, nil, numericCast(kvImageNoFlags), &error)?.takeRetainedValue()
guard error == kvImageNoError else { return nil }

// création d'une UIImage
let scaledImage = destCGImage.flatMap { UIImage(CGImage: $0, scale: 0.0, orientation: image.imageOrientation) }
```

Les APIs d'Accelerate utilisées ici opèrent clairement à un plus bas niveau que
d'autres méthodes de redimensionnement. Pour utiliser cette méthode, vous devez
d'abord créer un buffer source depuis votre `CGImage` en utilisant un `vImage_CGImageFormat` via `vImageBuffer_InitWithCGImage()`. Le buffer de destination
est alloué à partir de résolution d'image désirée, puis `vImageScale_ARGB8888`
réalise le travail de redimensionnement de l'image. Gérer ses propres buffers
lorsque l'on manipule des images plus grandes que la limite en mémoire de l'app
est laissé en exercice pour le lecteur.

---

## Analyse des performances

Alors comment ces différentes méthodes se comparent-elles les unes aux autres ?

Voice les résultats d'un ensemble de [benchmarks de performances](https://nshipster.com/benchmarking/) réalisés sur un iPhone 6 exécutant iOS 8.4, via [ce projet](https://github.com/natecook1000/Image-Resizing) :

### JPEG

Charger, redimensionner, et afficher une image haute-résolution (12000 ⨉ 12000 px 20 Mo JPEG) depuis [une galerie de la NASA](http://visibleearth.nasa.gov/view.php?id=78314) à 1/10<sup>ème</sup> de sa taille :

| Operation                    | Time _(sec)_ | σ   |
| ---------------------------- | ------------ | --- |
| `UIKit`                      | 0.612        | 14% |
| `Core Graphics` <sup>1</sup> | 0.266        | 3%  |
| `Image I/O`                  | 0.255        | 2%  |
| `Core Image` <sup>2</sup>    | 3.703        | 33% |
| `vImage` <sup>3</sup>        | --           | --  |

### PNG

Charger, redimensionner et afficher une image raisonnablement grande (1024 ⨉ 1024 px 1Mo PNG) affichant l'icône de [Postgres.app](http://postgresapp.com) à 1/10<sup>ème</sup> de sa taille :

| Operation                    | Time _(sec)_ | σ   |
| ---------------------------- | ------------ | --- |
| `UIKit`                      | 0.044        | 30% |
| `Core Graphics` <sup>4</sup> | 0.036        | 10% |
| `Image I/O`                  | 0.038        | 11% |
| `Core Image` <sup>5</sup>    | 0.053        | 68% |
| `vImage`                     | 0.050        | 25% |

<sup>1, 4</sup> Les résultats furent constants pour différentes valeurs de `CGInterpolationQuality`, avec des écarts de performances négligeables.

<sup>3</sup> La taille de l'image de la NASA était supérieure à ce qui pouvait
être traité en une seule passe par l'appareil.

<sup>2, 5</sup> Définir `kCIContextUseSoftwareRenderer` à `true` dans les options
passées à la création du `CIContext` a conduit des résultats un ordre de grandeur
inférieurs à ceux de référence.

## Conclusions

- **UIKit**, **Core Graphics**, et **Image I/O** offrent tous de bonnes performances
pour des opérations de redimensionnement sur la plupart des images.
- **Core Image** est devancé pour les opérations de mise à l'échelle d'images. En fait,
il est spécifiquement recommandé dans le document [Performance Best Practices section of the Core Image Programming Guide](https://developer.apple.com/library/mac/documentation/graphicsimaging/Conceptual/CoreImaging/ci_performance/ci_performance.html#//apple_ref/doc/uid/TP30001185-CH10-SW1) d'utiliser en amont les fonctions de Core Graphics ou Image I/O pour recadrer ou sous-échantillonner les images.
- Pour un redimensionnement d'images sans besoin de fonctionnalités additionnelles,
**`UIGraphicsBeginImageContextWithOptions`** est probablement la meilleure option.
- Si la qualité d'image est importante, considérez l'utilisation de **`CGBitmapContextCreate`** en conjonction de **`CGContextSetInterpolationQuality`**.
- Lorsque le redimensionnement se fait avec l'intention d'afficher des miniatures,
**`CGImageSourceCreateThumbnailAtIndex`** propose une solution convaincante qui prend
en charge à la fois la génération ainsi que la mise en cache.
- À moins que vous ne soyez déjà entrain de manipuler des **`vImage`**, la charge de
travail additionnelle pour utiliser le framework bas-niveau Accelerate n'est pas rentable.
