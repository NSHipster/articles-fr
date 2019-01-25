---
title: OptionSet
author: Mattt
translator: Vincent Pradeilles
category: Swift
excerpt: >-
  L'Objective-C utilise la macro `NS_OPTIONS`
  pour définir des ensembles de valeurs qui
  peuvent être combinées entre elles. Swift
  importe ces types sous la forme de structures
  implémentant le protocole `OptionSet`.
  Mais les nouvelles fonctionnalités de Swift
  permettraient-elles une meilleure alternative ?
revisions:
  "2014-09-09": First Publication
  "2018-11-07": Updated for Swift 4.2
status:
  swift: 4.2
  reviewed: November 7, 2018
---

L'Objective-C utilise la macro
[`NS_OPTIONS`](https://nshipster.com/ns_enum-ns_options/)
pour définir des types d'<dfn>options</dfn>, c'est à dire
des ensembles de valeurs qui peuvent être combinées en une
seule.
Par exemple, les valeurs du type `UIViewAutoresizing` de
UIKit peuvent être combinées via l'opérateur OU binaire
(`|`) et le résultat affecté à la propriété
`autoresizingMask` d'une `UIView`, pour spécifier quelles
marges et dimensions doivent être automatiquement
redimensionnées :

```objc
typedef NS_OPTIONS(NSUInteger, UIViewAutoresizing) {
    UIViewAutoresizingNone                 = 0,
    UIViewAutoresizingFlexibleLeftMargin   = 1 << 0,
    UIViewAutoresizingFlexibleWidth        = 1 << 1,
    UIViewAutoresizingFlexibleRightMargin  = 1 << 2,
    UIViewAutoresizingFlexibleTopMargin    = 1 << 3,
    UIViewAutoresizingFlexibleHeight       = 1 << 4,
    UIViewAutoresizingFlexibleBottomMargin = 1 << 5
};
```

Swift importe ce type, ainsi que tous les autres types définis
via la macro `NS_OPTIONS`, sous la forme d'une structure
qui se conforme au protocole `OptionSet`.

```swift
extension UIView {
    struct AutoresizingMask: OptionSet {
        init(rawValue: UInt)

        static var flexibleLeftMargin: UIView.AutoresizingMask
        static var flexibleWidth: UIView.AutoresizingMask
        static var flexibleRightMargin: UIView.AutoresizingMask
        static var flexibleTopMargin: UIView.AutoresizingMask
        static var flexibleHeight: UIView.AutoresizingMask
        static var flexibleBottomMargin: UIView.AutoresizingMask
    }
}
```

{% info %}
Le renommage et l'imbrication des types
importés est le résultat d'un autre
mécanisme.
{% endinfo %}

À l'époque où `OptionSet` a été introduit (et, avant lui,
`RawOptionSetType`), il s'agissait de la meilleure encapsulation
que le langage pouvait fournir.
Vers la fin de cet article, nous démontrerons qu'il est
possible d'exploiter les fonctionnalités apportées par Swift 4.2
pour améliorer `OptionSet`

...mais n'allons pas trop vite.

Cette semaine sur NSHipster,
faisons un tour d'horizon de l'utilisation des types `OptionSet`
automatiquement importés et de comment nous pouvons créer les nôtres.
Ensuite, nous proposerons une approche différente pour définir
des configurations.

## Utiliser les Types `OptionSet` importés

[Selon la documentation](https://developer.apple.com/documentation/swift/optionset),
il existe plus de 300 types dans les SDKs d'Apple qui implémentent
le protocole `OptionSet`, allant de `ARHitTestResult.ResultType` à `XMLNode.Options`.

Peut importe celui que vous utilisez,
la manière de procéder est toujours identique :

Pour spécifier une unique option, il suffit
de la fournir directement (Swift est capable
d'inférer le type lorsqu'une propriété est
affectée, pas besoin, donc, de mentionner
tout ce qui précède le dernier `.`) :

```swift
view.autoresizingMask = .flexibleHeight
```

`OptionSet` implémente le protocole [`SetAlgebra`](https://developer.apple.com/documentation/swift/setalgebra),
il est donc possible de spécifier de multiples options
via un tableau littéral --- aucune manipulation sur les
bits n'est requise :

```swift
view.autoresizingMask = [.flexibleHeight, .flexibleWidth]
```

Pour ne spécifier aucune option,
il suffit de passer un tableau littéral vide (`[]`) :

```swift
view.autoresizingMask = [] // aucune option
```

## Déclarer son propre Type `OptionSet`

Vous pourriez envisager de créer votre propre type
`OptionSet` si vous avez une propriété qui stocke
une combinaison de valeurs issues d'un ensemble
non-extensible, et que vous souhaitez que cette
combinaison soit stockée efficacement, sous la
forme d'un vecteur de bits.

Pour cela,
déclarez une nouvelle structure qui implémente le
protocole `OptionSet` via la propriété d'instance
`rawValue`, ainsi que des propriétés statiques
pour chacunes des options que vous souhaitez représenter.
Les valeurs brutes de celles-ci sont initialisées avec
des puissances croissantes de 2, qui peuvent être
construites en utilisant un décalage binaire à gauche
(`<<`) dont le membre droit est incrémenté avec chaque
nouvelle option.
Vous pouvez également définir des alias pour des
combinaisons spécifiques.

Par exemple, voici comment vous pourriez représenter
les garnitures d'une pizza :

```swift
struct Toppings: OptionSet {
    let rawValue: Int

    static let pepperoni    = Toppings(rawValue: 1 << 0)
    static let onions       = Toppings(rawValue: 1 << 1)
    static let bacon        = Toppings(rawValue: 1 << 2)
    static let extraCheese  = Toppings(rawValue: 1 << 3)
    static let greenPeppers = Toppings(rawValue: 1 << 4)
    static let pineapple    = Toppings(rawValue: 1 << 5)

    static let meatLovers: Toppings = [.pepperoni, .bacon]
    static let hawaiian: Toppings = [.pineapple, .bacon]
    static let all: Toppings = [
        .pepperoni, .onions, .bacon,
        .extraCheese, .greenPeppers, .pineapple
    ]
}
```

Ici utilisées dans un contexte plus concret :

```swift
struct Pizza {
    enum Style {
        case neapolitan, sicilian, newHaven, deepDish
    }

    struct Toppings: OptionSet { ... }

    let diameter: Int
    let style: Style
    let toppings: Toppings

    init(inchesInDiameter diameter: Int,
         style: Style,
         toppings: Toppings = [])
    {
        self.diameter = diameter
        self.style = style
        self.toppings = toppings
    }
}

let dinner = Pizza(inchesInDiameter: 12,
                   style: .neapolitan,
                   toppings: [.greenPeppers, .pineapple])
```

Un autre bénéfice de la conformance d'`OptionSet` à `SetAlgebra`
est que les opérations ensemblistes standards peuvent être
utilisées pour déterminer une appartenance, ajouter ou
supprimer des éléments, et calculer des unions ou des
intersections.
Cela permet, par exemple, de facilement
déterminer quelles garnitures de pizza sont végétariennes :

```swift
extension Pizza {
    var isVegetarian: Bool {
        return toppings.isDisjoint(with: [.pepperoni, .bacon])
    }
}

dinner.isVegetarian // true
```

## Nouveau Regard sur un Vieux Classique

Très bien, maintenant que nous savons utiliser `OptionSet`,
essayons de montrer comment ne pas utiliser `OptionSet`.

Comme indiqué précédemment,
les nouvelles fonctionnalités de Swift 4.2 permettent
d'avoir <del>le beurre</del> <ins>la pizza</ins> et l'argent
<del>du beurre</del> <ins>de la pizza</ins>.

Premièrement, déclarez un nouveau protocole `Option`
qui hérite de `RawRepresentable`, `Hashable`, et `CaseIterable`.

```Swift
protocol Option: RawRepresentable, Hashable, CaseIterable {}
```

Ensuite, déclarez une `enum` basée sur des `String`
qui se conforme au protocole `Option` :

```swift
enum Topping: String, Option {
    case pepperoni, onions, bacon,
         extraCheese, greenPeppers, pineapple
}
```

Comparons les déclarations de structures vues précédemment
avec celle ci-dessus.
Bien mieux, n'est-ce pas ?
Et attendez --- ce n'est pas fini.

L'implémentation automatique de `Hashable` permet une utilisation
avec `Set` sans effort supplémentaire, ce qui nous amène
à mi-chemin des fonctionnalités d'`OptionSet`.
En utilisant la conformance conditionnelle,
nous pouvons créer une extension pour n'importe quel `Set` dont
les éléments sont des `Topping` et définir ainsi des combinaisons
de garnitures.
En bonus, `CaseIterable` permet de facilement représenter
la combinaison de toutes les possibilités :

```swift
extension Set where Element == Topping {
    static var meatLovers: Set<Topping> {
        return [.pepperoni, .bacon]
    }

    static var hawaiian: Set<Topping> {
        return [.pineapple, .bacon]
    }

    static var all: Set<Topping> {
        return Set(Element.allCases)
    }
}

typealias Toppings = Set<Topping>
```

Et ce n'est pas le seul tour que `CaseIterable` a
dans son sac ! En itérant sur la propriété statique
`allCases`, nous pouvons générer le vecteur de bits
propre à chaque cas, que nous pouvons ensuite combiner
pour produire la `rawValue` associée à n'importe quel
`Set` contenant des `Option` :

```swift
extension Set where Element: Option {
    var rawValue: Int {
        var rawValue = 0
        for (index, element) in Element.allCases.enumerated() {
            if self.contains(element) {
                rawValue |= (1 << index)
            }
        }

        return rawValue
    }
}
```

Puisque `OptionSet` et `Set` implémentent tous deux
`SetAlgebra`, notre nouvelle implémentation de `Topping`
peut-être substituée à la précédente sans avoir besoin de
modifier quoique que ce soit dans le type `Pizza`.

{% warning %}
Cette approche suppose que l'ordre de déclaration dans
lequel `CaseIterable` fournit les différents cas reste
le même d'une exécution à l'autre.
Si ce n'est pas le cas, la `rawValue` calculée à partir
d'une combinaison d'options risque d'être incohérente.
{% endwarning %}

---

Alors, pour résumer :
vous avez de grandes chances de croiser la route d'`OptionSet`
lorsque vous utilisez les SDKs d'Apple en Swift.
Et bien que vous _puissiez_ créer vos propres structures
qui implémentent `OptionSet`, cela n'est probablement pas
nécessaire.
Vous pouvez utiliser l'astucieuse méthode décrite à la fin
de cet article ou bien vous satisfaire de l'approche standard.
