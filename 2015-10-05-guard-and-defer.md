---
title: guard & defer
author: Mattt & Nate Cook
authors:
  - Nate Cook
  - Mattt
category: Swift
excerpt: >
  Swift 2.0 a instauré deux nouvelles structures de contrôle,
  dont l'objectif est de simplifier et d'affiner les programmes
  que nous écrivons.
  Alors que la première, par sa nature, rend notre code plus
  linéaire, la seconde permet l'inverse, en retardant
  l'exécution de son contenu.
revisions:
  "2015-10-05": First Publication
  "2018-08-01": Updated for Swift 4.2
status:
  swift: 4.2
  reviewed: August 1, 2018
---

> "Tels des programmeurs conscients de nos limites,
> nous devrions faire tout notre possible pour […]
> faire en sorte que la relation entre nos
> programmes (exprimés par du texte)
> et leurs exécutions (exprimées par rapport au temps)
> soit aussi évidente que possible."

> —[Edsger W. Dijkstra](https://en.wikipedia.org/wiki/Edsger_W._Dijkstra),
> ["Go To Considered Harmful"](https://homepages.cwi.nl/~storm/teaching/reader/Dijkstra68.pdf)

Il est regrettable que l'article de Dijkstra soit principalement resté
dans la mémoire des développeurs comme l'origine du populaire titre d'article "\_\_\_\_ Consider Harmful".

Car, comme souvent, Dijkstra faisait une remarque pertinente:
**la structure d'un code devrait refléter son comportement**.

Swift 2.0 a instauré deux nouvelles structures de contrôle,
dont l'objectif est de simplifier et d'affiner les programmes
que nous écrivons : `guard` et `defer`.
Alors que la première, par sa nature, rend notre code plus
linéaire, la seconde permet l'inverse, en retardant
l'exécution de son contenu.

Comment devons nous apprivoiser ces nouvelles structures
de contrôle ?
De quelle manière `guard` et `defer` peuvent-ils nous permettre
de simplifier la relation entre un programme et son exécution ?

Remettons `defer` à plus tard, et commençons par nous
intéresser à `guard`.

---

## guard

`guard` est une instruction conditionnelle, qui requiert
une expression s'évaluant à `true` pour poursuivre
l'exécution.
Si l'expression s'évalue à `false`, l'obligatoire clause
`else` est exécuté à la place.

```swift
func sayHello(numberOfTimes: Int) {
    guard numberOfTimes > 0 else {
        return
    }

    for _ in 1...numberOfTimes {
        print("Hello!")
    }
}
```

La clause `else` d'une instruction `guard`
doit entraîner la sortie de la portée
courante en utilisant soit `return` pour
quitter une fonction, soit `continue` ou
`break` pour sortir d'une boucle, ou bien
une fonctionne retournant
[`Never`](https://nshipster.com/never)
telle que `fatalError(_:file:line:)`.

`guard` est particulièrement utile lorsqu'il est combiné
à l'inspection d'un optionnel. Toutes les affectations
de valeurs optionnelles crées dans une instruction `guard`
sont visibles par le reste de la fonction ou portée.

Comparons une affectation d'optionnel réalisée par une
instruction `guard-let` par rapport à une instruction
`if-let` :

```swift
var name: String?

if let name = name {
    // name is nonoptional inside (name is String)
}
// name is optional outside (name is String?)


guard let name = name else {
    return
}

// name is nonoptional from now on (name is String)
```

Si la syntaxe permettant de multiples affections
instaurée par [Swift 1.2](/swift-1.2/) annonçait
une réfection de la [pyramide du malheur](http://www.scottlogic.com/blog/2014/12/08/swift-optional-pyramids-of-doom.html), `guard`
permet de la démolir complètement.

```swift
for imageName in imageNamesList {
    guard let image = UIImage(named: imageName)
        else { continue }

    // do something with image
}
```

### Se prémunir de l'indentation et les erreurs excessives

Regardons un avant/après de la façon dont `guard`
permet d'améliorer notre code et éviter des erreurs.

Comme exemple, nous allons implémenter une fonction
`readBedtimeStory()` :

```swift
enum StoryError: Error {
    case missing
    case illegible
    case tooScary
}

func readBedtimeStory() throws {
    if let url = Bundle.main.url(forResource: "book",
                               withExtension: "txt")
    {
        if let data = try? Data(contentsOf: url),
            let story = String(data: data, encoding: .utf8)
        {
            if story.contains("👹") {
                throw StoryError.tooScary
            } else {
                print("Once upon a time... \(story)")
            }
        } else {
            throw StoryError.illegible
        }
    } else {
        throw StoryError.missing
    }
}
```

Pour lire une histoire,
il nous faut obtenir un livre,
ce livre doit être déchiffrable,
et l'histoire ne doit pas faire
trop peur.

Remarquons comme les instructions `throw` sont éloignées des
conditions qui les déclenchent. Pour comprendre ce qu'il doit
se passer lorsque le livre `book.txt` n'a pas pu être trouvé,
il est nécessaire de descendre jusqu'à la fin de la fonction.

Comme un bon livre, un code devrait raconter une histoire:
un scénario facile à suivre, avec un début, un milieu et une fin
bien identifiés.

Une utilisation appropriée de `guard` nous permet
de structurer notre code afin de rendre sa lecture
plus linéaire.

```swift
func readBedtimeStory() throws {
    guard let url = Bundle.main.url(forResource: "book",
                                  withExtension: "txt")
    else {
        throw StoryError.missing
    }

    guard let data = try? Data(contentsOf: url),
        let story = String(data: data, encoding: .utf8)
    else {
        throw StoryError.illegible
    }

    if story.contains("👹") {
        throw StoryError.tooScary
    }

    print("Once upon a time... \(story)")
}
```

_Beaucoup mieux !_

Chaque erreur est traitée dès sa détection, et nous
pouvons aisément suivre l'exécution.

### Il ne faut pas ne pas éviter les doubles négations

Un travers à éviter à propos de cette nouvelle structure
de contrôle est sa sur-utilisation --- particulièrement avec
une condition déjà inversée.

Par exemple, si l'on souhaite mettre fin à l'exécution lorsque
qu'une chaîne de caractères est vide, il ne faut pas écrire :

```swift
// Huh?
guard !string.isEmpty else {
    return
}
```

Restons simple.
Mieux vaut utiliser une structure de contrôle classique,
et éviter ainsi une double négation.

```swift
// Aha!
if string.isEmpty {
    return
}
```

## defer

Entre `guard` et la nouvelle instruction `throw` pour la gestion
d'erreur, Swift promeut un style de programmation basé sur la fin
d'exécution prématurée plutôt que l'imbrication d'instructions `if`.
Toutefois, ces retours prématurés posent un problème lorsque des
ressources ont été allouées, sont peut-être encore utilisées,
et doivent être libérées avant de mettre fin à l'exécution.

Le mot-clé `defer` fournit un moyen sûr et simple de gérer cette
difficulté en indiquant qu'un bloc de code ne devra être exécuté
que quand l'exécution de la portée courante se terminera.

Considérons la fonction suivante, qui encapsule l'appel système
`gethostname(2)` et retourne le [nom d'hôte](https://en.wikipedia.org/wiki/Hostname)
du système :

```swift
import Darwin

func currentHostName() -> String {
    let capacity = Int(NI_MAXHOST)
    let buffer = UnsafeMutablePointer<Int8>.allocate(capacity: capacity)

    guard gethostname(buffer, capacity) == 0 else {
        buffer.deallocate()
        return "localhost"
    }

    let hostname = String(cString: buffer)
    buffer.deallocate()

    return hostname
}
```

Ici, nous allouons un `UnsafeMutablePointer<Int8>`, et nous
devons nous assurer de le libérer lorsque la condition échoue
_mais aussi_ lorsque nous avons fini de l'utiliser normalement.

Source d'erreur ? _Complètement._
Répétitif et frustrant ? _Absolument._

En utilisant une instruction `defer`,
nous pouvons supprimer l'erreur de programmation potentielle,
tout en simplifiant notre code :

```swift
func currentHostName() -> String {
    let capacity = Int(NI_MAXHOST)
    let buffer = UnsafeMutablePointer<Int8>.allocate(capacity: capacity)
    defer { buffer.deallocate() }

    guard gethostname(buffer, capacity) == 0 else {
        return "localhost"
    }

    return String(cString: buffer)
}
```

Bien que `defer` soit présent immédiatement après l'appel à
`allocate(capacity)`, son exécution attendra le retour de la
fonction, peu importe où il aura lieu.

`defer` est un bon choix lorsque des appels d'API vont de paire,
tels que `allocate(capacity:)` / `deallocate()`,
`wait()` / `signal()`, ou
`open()` / `close()`.
Par cette approche, non seulement une source d'erreurs est éliminée,
mais Dijkstra a également de quoi être fier.
_"Goed gedaan!" peut-il s'exclamer, dans son Danois natif_.

### Utiliser `defer` à répétition

Si vous écrivez plusieurs instructions `defer` dans une même portée,
elles seront exécutées dans l'ordre inverse de leur déclaration ---
comme une pile.
Cette inversion est un détail primordial,
car il assure que toutes les ressources présentes lorsque
l'instruction est écrite le seront encore lorsqu'elle sera
exécutée.

Par exemple, exécuter le code ci-dessous produira le
résultat suivant :

```swift
func procrastinate() {
    defer { print("wash the dishes") }
    defer { print("take out the recycling") }
    defer { print("clean the refrigerator") }

    print("play videogames")
}
```

<samp>
play videogames<br/>
clean the refrigerator<br/>
take out the recycling<br/>
wash the dishes<br/>
</samp>

> Que se passe-t-il si des instructions `defer` sont imbriquées, comme ici ?

```swift
defer { defer { print("clean the gutter") } }
```

> Vous pourriez penser que `print("clean the gutter")` sera exécuté en tout dernier.
> Mais ce n'est pas ce qui se produira.
> Réfléchissez à la solution, et testez la ensuite dans un Playground.

### Utiliser `defer` à bon escient

Si une variable est utilisée dans le corps d'une instruction `defer`,
sa valeur au moment de l'exécution du corps sera utilisée.
Autrement dit : les instructions `defer` ne capturent pas la valeur d'une
variable.

Si vous exécutez le code suivant, vous obtiendrez ce résultat :

```swift
func flipFlop() {
    var position = "It's pronounced /ɡɪf/"
    defer { print(position) }

    position = "It's pronounced /dʒɪf/"
    defer { print(position) }
}
```

<samp>
It's pronounced /dʒɪf/ <br/>
It's pronounced /dʒɪf/
</samp>

### Utiliser `defer` avec parcimonie

Un autre aspect à garder en tête est que `defer` ne permet
pas de mettre fin à l'exécution d'une fonction ou d'une boucle.
Donc si vous y appelez une fonction marquée comme `throws`,
l'erreur ne pourra pas être propagée.

```swift
func burnAfterReading(file url: URL) throws {
    defer { try FileManager.default.removeItem(at: url) }
    // 🛑 Errors not handled

    let string = try String(contentsOf: url)
}
```

A la place, vous pouvez choisir d'ignorer l'erreur via
`try?` ou bien, si cela n'est pas possible, ne pas effectuer
cet appel via l'instruction `defer`

### (Any Other) Defer Considered Harmful

Aussi pratique que `defer` puisse être,
il faut se méfier de sa capacité à produire du code confus et
cryptique.
Il peut-être tentant de recourir à `defer` dans des situations où
une fonction à besoin de retourner une valeur qui doit également être
modifiée, comme, par exemple, dans l'implémentation de l'opérateur
post-fix `++` :

```swift
postfix func ++(inout x: Int) -> Int {
    let current = x
    x += 1
    return current
}
```

Dans un tel cas, `defer` offre une alternative astucieuse.
Pourquoi créer une variable temporaire quand il est possible de
simplement retarder l'incrémentation ?

```swift
postfix func ++(inout x: Int) -> Int {
    defer { x += 1 }
    return x
}
```

Pour autant qu'elle soit astucieuse, cette inversion du flot du programme
porte préjudice à sa lisibilité.
Utiliser `defer` pour intentionnellement altérer le flot d'exécution d'un
programme, au lieu de s'en tenir à la libération de ressources, conduira
à un programme dont l'exécution sera compliquée à démêler.

---

"Tels des programmeurs conscients de nos limites",
nous devons soupeser consciencieusement le rapport bénéfice/risque
de chaque fonctionnalité d'un langage.

Un ajout tel que `guard` permet d'obtenir un code plus linéaire et lisible :
il faut l'utiliser aussi souvent que possible.

De la même manière, `defer` permet également de résoudre des situations
compliquées, mais nous force à garder en tête sa présence et son impact
sur l'exécution du programme : il sera sage de le réserver aux situations
pour lequelles il a réellement été conçu.
