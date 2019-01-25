---
title: Method Swizzling
author: Mattt
translator: Vincent Pradeilles
category: Objective-C
excerpt: "Le swizzling de méthode est le processus consistant à changer l'implémentation associée à un sélecteur existant. Cette technique est rendue possible par le fait qu'en Objective-C, l'invocation de méthodes puisse être altérée à l'exécution, en modifiant la façon dont des sélecteurs sont associés à leurs fonctions sous-jacentes dans la table d'aiguillage d'une classe."
status:
  swift: n/a
  reviewed: January 28, 2015
---

> If you could blow up the world with the flick of a switch<br/>
> Would you do it?<br/>
> If you could make everybody poor just so you could be rich<br/>
> Would you do it?<br/>
> If you could watch everybody work while you just lay on your back<br/>
> Would you do it?<br/>
> If you could take all the love without giving any back<br/>
> Would you do it?<br/>
> And so we cannot know ourselves or what we'd really do...<br/>
> With all your power ... What would you do?<br/> > <cite><strong>The Flaming Lips</strong>, <em><a href="https://en.wikipedia.org/wiki/The_Yeah_Yeah_Yeah_Song_(With_All_Your_Power)">"The Yeah Yeah Yeah Song (With All Your Power)"</a></em></cite>

Dans l'article de la semaine précédente sur les [associated objects](https://nshipster.com/associated-objects/), nous avons commencé à explorer les arcanes de l'environnement d'exécution de l'Objective-C. Cette semaine, nous nous aventurons plus avant, pour discuter de ce qui est peut-être la plus controversée des astuces qui tirent parti de cet environnement : le swizzling de méthode.

---

Le swizzling de méthode est le processus consistant à changer l'implémentation associée à un sélecteur existant. Cette technique est rendue possible par le fait qu'en Objective-C, l'invocation de méthodes puisse être altérée à l'exécution, en modifiant la façon dont des sélecteurs sont associés à leurs fonctions sous-jacentes dans la table d'aiguillage d'une classe.

Par exemple, disons que nous souhaitons savoir combien de fois chaque view controller d'une app iOS est présenté à l'utilisateur.

Chaque view controller pourrait ajouter le code approprié à son implémentation de `viewDidAppear:`, mais cela produirait une énorme quantité de code dupliqué. L'héritage pourrait être une autre approche, mais cela nécessiterait de sous-classer `UIViewController`, `UITableViewController`, `UINavigationController`, et tous les autres types de contrôleurs – une approche qui produirait également du code dupliqué.

Fort heureusement, il existe une alternative : le swizzling de méthode depuis une catégorie. Voici comment cela fonctionne :

```objc
#import <objc/runtime.h>

@implementation UIViewController (Tracking)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];

        SEL originalSelector = @selector(viewWillAppear:);
        SEL swizzledSelector = @selector(xxx_viewWillAppear:);

        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);

        // When swizzling a class method, use the following:
        // Class class = object_getClass((id)self);
        // ...
        // Method originalMethod = class_getClassMethod(class, originalSelector);
        // Method swizzledMethod = class_getClassMethod(class, swizzledSelector);

        BOOL didAddMethod =
            class_addMethod(class,
                originalSelector,
                method_getImplementation(swizzledMethod),
                method_getTypeEncoding(swizzledMethod));

        if (didAddMethod) {
            class_replaceMethod(class,
                swizzledSelector,
                method_getImplementation(originalMethod),
                method_getTypeEncoding(originalMethod));
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

#pragma mark - Method Swizzling

- (void)xxx_viewWillAppear:(BOOL)animated {
    [self xxx_viewWillAppear:animated];
    NSLog(@"viewWillAppear: %@", self);
}

@end
```

> En informatique, le [swizzling de pointeur](https://en.wikipedia.org/wiki/Pointer_swizzling) est la conversion de références basées sur un nom ou une position vers de réels pointeurs. Bien que l'usage de ce terme dans le cadre de l'Objective-C ne soit pas complètement clair, il est aisé de comprendre pourquoi il est entré dans les moeurs, puisque le swizzling de méthode implique de remplacer le pointeur de fonction associé à un sélecteur.

Désormais, quand n'importe quelle instance de `UIViewController`, ou d'une de ses sous-classes appellera `viewWillAppear:`, un message sera écrit à la console.

L'injection d'un comportement dans le cycle de vie d'un view controller, la remontée d'un évènement depuis l'UI, le dessin d'une vue ou la couche réseau de Foundation sont autant de bons exemples de comment le swizzling de méthodes peut être utilisé efficacement. Il existe de nombreuses occasions où le swizzling est une technique pertinente, et elles apparaitront d'autant plus clairement qu'un dévelopeur Objective-C sera expérimenté.

Peu importe _pourquoi_ ou _à quel endroit_ l'on choisit de mettre en oeuvre le swizzling, il est impératif de savoir _comment_ s'y prendre :

## +load vs. +initialize

**Le swizzling devrait toujours avoir lieu dans `+load`.**

Il existe deux méthodes qui sont automatiquement appelées pour chaque classe par l'environnement d'exécution de l'Objective-C. `+load` est appelée lorsque la classe est initialement chargée, et `+initialize` juste avant que le premier appel de méthode sur cette classe ou une de ses instances ai lieu. L'implémentation de ces deux méthodes est facultative, et elles ne sont exécutées que si elles sont réellement présentes.

Puisque le swizzling de méthode modifie l'état global, il est important de minimiser la possibilité d'un accès concurrent entre différents threads. `+load` est garantie d'être appelée lors du chargement d'une classe, ce qui minimise les risque lors de la modification de l'état global. A l'inverse, `+initialize` ne fournie pas de telle garantie sur le moment de son appel — de fait, elle pourrait ne _jamais_ être appelée, si sa classe ne reçoit jamais de message de la part de l'app.

## dispatch_once

**Le swizzling devrait toujours avoir lieu dans un `dispatch_once`.**

Une fois encore, puisque le swizzling modifie l'état global, nous nous devons de mettre en oeuvre toutes les précautions que permet l'environnement d'exécution. L'atomicité est une de ces précautions, car elle garantie que le code ne sera exécuté qu'une seule fois, même entre différents threads. La fonction `dispatch_once` de Grand Central Dispatch offre ces garanties, et doit donc être considérée comme aussi indispensable lors d'un swizzling que lors de [l'initialisation d'un singleton](https://nshipster.com/c-storage-classes/).

## Selectors, Methods, & Implementations

En Objective-C, les _sélecteurs_, _méthodes_, et _implémentations_, bien que souvent utilisés de façon interchangeable, font référence à des aspects bien distincts de l'environnement d'exécution.

Voici la façon dont la [documentation d'Apple](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/ObjCRuntimeRef/Reference/reference.html#//apple_ref/c/func/method_getImplementation) les décrits :

> - Sélecteur ((`typedef struct objc_selector *SEL`) : les sélecteurs sont utilisés pour représenter le nom d'une méthode lors de l'exécution. Un sélecteur de méthode est une chaîne de caractères C qui a été enregistrée (ou associée) à l'environnement d'exécution Objective-C. Les sélecteurs générés par le compilateur sont automatiquement associés à l'environnement d'exécution lorsque la classe est chargée.
> - Méthode (`typedef struct objc_method *Method`) : un type opaque représentant une méthode dans la définition d'une classe.
> - Implémentation (`typedef id (*IMP)(id, SEL, ...)`) : ce type de données est un pointeur vers le début d'une fonction qui implémente la méthode. Cette fonction utilise la convention d'appel C standard de l'architecture processeur courante. Le premier paramètre est un pointeur vers self (c'est à dire, l'emplacement en mémoire de l'instance de la classe, ou, pour une méthode de classe, un pointeur vers l'objet métaclasse). Le second paramètre est le sélecteur de la méthode. Suivent les arguments de la méthode.

La meilleure façon de saisir le lien entre ces concepts est le suivant : une classe (`Class`) maintient une table d'aiguillage permettant de résoudre l'envoi de messages à l'exécution ; chaque entrée de cette table est une méthode (`Method`), avec pour valeur associée un sélecteur (`SEL`), vers une implémentation (`IMP`), qui est un pointeur vers la fonction C sous-jacente.

Swizzler une méthode revient à modifier la table d'aiguillage d'une classe de façon à ce que les messages d'un sélecteur existant soient aiguillés vers une implémentation différente, tout en associant l'implémentation originale à un nouveau sélecteur.

## Appeler `_cmd`

Il pourrait sembler que le code suivant résulte en une boucle infinie :

```objc
- (void)xxx_viewWillAppear:(BOOL)animated {
    [self xxx_viewWillAppear:animated];
    NSLog(@"viewWillAppear: %@", NSStringFromClass([self class]));
}
```

De façon surprenante, ce n'est pas le cas. Lors du processus de swizzling, `xxx_viewWillAppear:` a été réassigné à l'implémentation originale de `UIViewController -viewWillAppear:`. Il est tout à fait normal qu'appeler à nouveau une même méthode sur self dans sa propre implémentation soit perçu comme dangereux, cependant, dans ce cas, il est nécessaire de se rappeler ce qui est _réellement_ entrain d'avoir lieu. Toutefois, si nous venions à appeler `viewWillAppear:` dans cette même méthode, cela produirait _effectivement_ une boucle infinie, puisque l'implémentation de cette méthode sera associée au sélecteur `viewWillAppear:` lors de l'exécution.

> Rappelez-vous de préfixer les noms de vos méthodes swizzlées, de la même manière que vous le feriez pour tout autre méthode déclarée dans une catégorie.

## Discussion

Le swizzling est largement considéré comme une forme de magie noire, susceptible de mener à des comportements inattendus et des conséquences imprévues. Bien qu'il ne s'agisse pas de la plus sûre des approches, elle peut l'être raisonnablement, lorsque les précautions suivantes sont prises :

- **Toujours appeler l'implémentation originale de la méthode (à moins d'avoir une bonne raison de ne pas le faire)** : les API fournissent un contrat strict sur leurs entrées et sorties, mais l'implémentation située entre les deux est une boite noire. Swizzler une méthode et ne pas appeler l'implémentation originale peut casser certaines suppositions sur l'état interne, et, avec elles, le reste de l'application.
- **Éviter les collisions** : préfixez les méthodes de catégories, et assurez vous que rien d'autre dans votre code (et dans aucune de vos dépendances) ne cherche à tripoter la même fonctionnalité.
- **Comprenez ce qui est entrain de se dérouler** : réaliser un swizzling à partir d'un simple copier-coller d'un bout de code sans comprendre son fonctionnement n'est pas seulement dangereux, c'est également une opportunité gâchée d'en apprendre beaucoup sur l'environnement d'exécution Objective-C. Lisez [Objective-C Runtime Reference](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/ObjCRuntimeRef/Reference/reference.html#//apple_ref/c/func/method_getImplementation) et parcourez le fichier `<objc/runtime.h>` pour acquérir une bonne compréhension de pourquoi et comment les choses se produisent. _Faites toujours l'effort de substituer la compréhension à la pensée magique_.
- **Agissez avec prudence** : peu importe votre niveau de confiance dans le fait de swizzler des fonctionnalités de Foundation, UIKit ou tout autre framework pré-intégré, soyez conscient que tout peut être cassé dans la prochaine version. Soyez prêt à cela, et assurez vous de faire les efforts nécessaire pour que jouer avec le feu ne se termine par par une `NSBrulûre`.

> Vous ne vous sentez pas capable d'interagir directement avec l'environnement d'exécution Objective-C ? [Jonathan ‘Wolf’ Rentzsch](https://twitter.com/rentzsch) propose une librairie testée et compatible avec CocoaPods appelée [JRSwizzle](https://github.com/rentzsch/jrswizzle), qui s'en chargera pour vous.

---

Tout comme les [associated objects](https://nshipster.com/associated-objects/), le swizzling de méthode est une technique puissante, mais qui devrait être utilisée avec parcimonie.
