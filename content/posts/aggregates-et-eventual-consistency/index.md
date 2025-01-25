---
title: Aggregates et Eventual Consistency
date: 2025-01-25T04:00:00+01:00
lastmod: 2025-01-25T04:00:00+01:00
author: Anthony Cyrille
categories:
  - design
tags:
  - orienté objet
  - domain-driven design
draft: false
---

Si vous êtes familier avec le domain-driven design, vous avez certainement soupé du **Invariant Métiers** et de l'importance 
des Aggregates dans leur modélisation. Seulement, modéliser un invariant métier dans un aggregate n'est pas toujours simple.

## Invariant simple

Prenons un exemple.

Vous développez une application e-commerce classique comme celui d'Auchan et on vous demande de limiter le nombre de commande
par produit à 3. 

En bon développeur, vous avez modélisé un Aggregate `Order` qui contient des `LineItems` correspondant aux différentes produits commandés.

```ts 
class Order {
  private lineItems: LineItem[];
  
  private addItem(productId: string, quantity: number) {
    this.lineItems.push(new LineItem(randomId(), productId, quantity));
  }
}

class LineItem {
  private id: string;
  private productId: string;
  private quantity: number;
  
  constructor(id: string, productId: string, quantity: number) {
    this.id = id;
    this.productId = productId;
    this.quantity = quantity;
  }
}
```

Votre invariant ne concerne qu'un seul `LineItem` item à la fois, et il s'avère que `LineItem` est un `ValueObject` très simple.
Quel est le meilleur endroit pour faire respecter cet invariant ? On va choisir le référentiel le plus proche de l'information, et ici,
il s'agit de `LineItem`.

```ts 
class LineItem {
  private id: string;
  private productId: string;
  private quantity: number;

  private static MAX_QUANTITY = 3;

  constructor(id: string, productId: string, quantity: number) {
    this.id = id;
    this.productId = productId;
    this.quantity = quantity;
    
    if (quantity > LineItem.MAX_QUANTITY) {
      throw new Error("Quantité maximale dépassée");
    }
  }
}
```

On pourrait même implémenter un `AlwaysValid` et se créer un `ValueObject` appelée `Quantity`.

```ts
class Quantity {
  private value: number;
  
  constructor(value: number) {
    if (value > LineItem.MAX_QUANTITY) {
      throw new Error("Quantité maximale dépassée");
    }
    
    this.value = value;
  }
}

class LineItem {
  private id: string;
  private productId: string;
  private quantity: Quantity;

  constructor(id: string, productId: string, quantity: number) {
    this.id = id;
    this.productId = productId;
    this.quantity = new Quantity(quantity);
  }
}
```

Excellent. 

## Invariant racine complexe

Maintenant, on vous demande de **limiter la quantité de produit total de la commande à un certain poids**, par exemple 10kg.
Ah, mais on ne connait pas le poids des éléments, bigre !

Déjà, où placer cet invariant ? Le référentiel le plus bas est la classe `Order` car c'est **l'objet qui a la connaissance de l'ensemble
de la commande**, comme indiqué dans l'intitulé de l'invariant.

Pour l'implémentation, on a deux options, et la première consiste à récupérer les produits au moment où l'on ajoute un `LineItem` à la commande.

```ts
class Order {
  // ...
  
  private addItem(productId: string, quantity: number, productsRepository: ProductRepository) {
    const productIds = [productId].concat(this.lineItems.map(lineItem => lineItem.getProductId()));
    const products = productsRepository.findByIds(productIds);
    
    let totalWeight = 0;
    
    // ... algorithme pour calculer le poids
    
    if (totalWeight > 10) {
      throw new Error("Commande trop lourde")
    }
    
    this.lineItems.push(new LineItem(randomId(), productId, quantity));
  }
}
```

Evidemment, ça fonctionne, Cette solution a au moins l'avantage de ne pas impacter `LineItem` et d'être facile à exécuter,
mais elle a également son lot d'inconvénient :

1) La pureté du Domain Model s'effrite car on injecte un `Repository` dans l'entité. 
On pourrait bien évidemment injecter directement la liste des produits, mais en faisant cela, une partie de la connaissance
s'échappe de l'Aggregate `Order` et fuite vers l'appelant, ce qui est contraire à la pensée Orienté Objet
2) On doit récupérer la totalité des produits de la commande pour ajouter un seul item, ce qui signifie que l'opération
s'alourdit à mesure qu'il se remplit, augmentant la charge et le risque d'erreur de transaction. Imaginez un cas d'usage similaire
où vous auriez peut-être des centaines voir des milliers d'entrées à vérifier.

C'est une méthode qui peut convenir dans certaines circonstances, mais elle ne scale pas en volumétrie.

L'autre approche, beaucoup plus dans la pensée OO mais très difficile à accepter pour nos esprits gangrénés par les bases
de données relationnelle, consiste à **dénormaliser** l'information du poids, c'est à dire la **dupliquer**.

En d'autre termes, **chaque LineItem possède une copie du poids du produit**.

```ts
class LineItem {
  private id: string;
  private productId: string;
  private quantity: Quantity;
  private weight: number;

  constructor(id: string, productId: string, quantity: number, weight: number) {
    this.id = id;
    this.productId = productId;
    this.quantity = new Quantity(quantity);
    this.weight = weight;
  }
}
```

Garantir l'invariant devient un jeu d'enfant.

```ts
class Order {
  // ...
  
  private addItem(productId: string, quantity: number, weight: number) {
    const totalWeight = this.lineItems.reduce((acc, lineItem) => acc + lineItem.getWeight(), 0)
    
    if (totalWeight + weight > 10) {
      throw new Error("Commande trop lourde")
    }
    
    this.lineItems.push(new LineItem(randomId(), productId, quantity, weight));
  }
}
```

{{< notice tip >}}
Si le fait de représenter des masses avec de simple nombres entiers vous choque, eh bien... **félicitations** ! Vous pensez
vraiment en objet. Il faudrait ici créer un autre `ValueObject` appelé `Weight` pour encapsuler la masse, notamment car 
**la masse est une valeur unitaire qui peut s'exprimer en grammes, en kilogrammes**...

Ici, il n'y a rien qui documente l'unité 
de masse, **donc on doit déduire qu'il s'agit de kilos**. Vous allez me dire que c'est évident et qu'il faut être bête pour se tromper,
mais [si la NASA l'a fait, c'est que ça n'est pas si évident que ça.](https://www.simscale.com/blog/nasa-mars-climate-orbiter-metric/).
{{< /notice >}}

Cette méthode a l'énorme avantage de conserver "l'orientation objet" du programme et donc d'encapsuler pleinement l'information
de l'invariant à l'intérieur de l'objet. [Un point en moins pour l'anémisme](https://martinfowler.com/bliki/AnemicDomainModel.html).

Le désavantage, c'est la dénormalisation de l'information : **si on change la masse d'un produit, il faut répercuter cette modification sur 
toutes les commandes qui contiennent ce produit**. Et dans ce cas, on met un pied dans le monde de **l'Eventual Consistency**.

Avant d'avancer plus loin dans ce sujet, notons que peu importe la solution que l'on emploi, avec ou sans dénormalisation, le même problème
se pose : que se passe t'il lorsqu'on met à jour la masse d'un produit ?

En discutant avec les experts métiers, vous apprenez que l'opération ne peut que réussir, on ne peut pas empêcher la mise à jour de la masse
car certains paniers deviendraient subitement trop gros. 

Vous apprenez alors que dans ce cas là, plusieurs choses se passent selon l'état de la commande : 
- **Si elle n'est pas terminée**, alors un message indique à l'utilisateur que la commande est trop lourde et qu'il doit enlever des produits
- **Si elle est terminée et que la commande n'a pas été envoyée**, alors on envoie un message à l'utilisateur pour lui dire de mettre à jour sa commande (quitte à gérer des remboursements)
- **Et si elle a déjà été envoyée,** alors c'est trop tard, on ne peut rien faire.

Finalement, peu importe la façon dont on fait respecter l'invariant, il faudra toujours gérer ces scénarios. Je le précise,
car la dénormalisation de l'information n'a que peu d'impact sur les conséquences du non respect de l'invariant.

## Le problème de la Transactional Consistency

Pour revenir à notre sujet, _que se passe t'il lorsque la masse d'un produit est mis à jour, en partant du principe que notre
solution est dénormalisée ?_

D'abord, une transaction est créée côté administratif pour mettre à jour le produit. 
La question à se poser, c'est : **à quel moment doit-on mettre à jour les commandes ?**

On peut les mettre à jour immédiatement, dans la même transaction, **mais cela pose deux problèmes :** 
- Quand on s'appelle Auchan, **on peut avoir des centaines de milliers de commandes à mettre à jour**, 
ce qui peut prendre du temps, nuire au responsiveness de l'application et rendre le produit inaccessible, ce qui bloquerait
le fonctionnement total de l'application (sachant que l'opération n'est pas parallélisable, à moins que vous ayez découvert
le moyen de distribuer des transactions ?)
- Et si jamais une commande voit son poids dépasser le seuil, une erreur sera émise. Or, notre expert métier nous a bien
précisé que **la mise à jour de la masse d'un produit ne doit pas échouer.**

Au final, mettre à jour la totalité des commandes dans la même transaction n'est tout simplement pas pratiquable.

Il ne nous reste donc plus qu'à les mettre à jour séparémmenent, mais cela implique deux choses : 
- D'abord, il y aura un délai plus ou moins court pendant lequel le poids dans les commandes sera **désynchronisé**
du poids des produits
- Ensuite, **que cette mise à jour peut échouer pour plusieurs raisons** (bug dans le code, problème de réseau, l'électricité qui saute, etc)

Il nous faut donc une solution robuste qui tolère un léger décalage entre les poids des produits et les poids des commandes, 
donc un mécanisme compensatoire pour gérer les violations d'invariant.

Mais également une solution pour garantir la synchronisation du poids des commandes.

## L'Eventual Consistency

OK, maintenant qu'on a posé les problèmes, à quoi doit ressembler la solution ?
Le flow doit être le suivant : 

- La masse du produit est mis à jour
- La transaction est persistée et la mise à jour est effective
- Toutes les commandes possédant ce produit sont récupérés et mis à jour
- Ceux dont l'invariant a été violé font l'objet d'une procédure spéciale comme énoncée par l'expert métier

Rien ne vous choque ?

On parle ici de violer un invariant métier quand même, **celui qu'une commande puisse se retrouver avec un poids
dépassant 10kg** ! Diantre, où est ma croix ?!

C'est là qu'on réalise qu'il y a deux sortes d'invariants :  
- **Ceux qui sont absolus**, que rien ne peut rompre, et dans lequel le système ne doit absolument jamais se trouver.
Par exemple, si on sauvegarde le prix total d'une commande dans l'Aggregate Root, il doit toujours strictement être égal
à la somme du prix des `LineItems`. 
- **Ceux qui sont flexibles**, notamment car le monde est imparfait et donc que dans la réalité, certaines situations que l'on souhaite
éviter peuvent se produire, et donc qu'il faut être capable de les compenser.

Avec l'expérience, vous réaliserez que les invariants absolus sont assez rares car trop inflexibles. Il s'avère que dans
certaines situations, [être flexible sur un invariant est justement un moyen de gagner plus d'argent](https://personal.utdallas.edu/~metin/Research/b2bcargoinform_s.pdf).

Travailler autour d'un invariant flexible est donc **un enjeu métier**, car il permet d'augmenter le profit de l'entreprise.

On doit donc être capable, malgré l'absence de transaction, de garantir que toutes les commandes seront mises à jour. Pour cela,
il faut mettre en place des process répétables et analysable. La solution la plus commune est d'utiliser un **Job Queue**.

La force d'un Job Queue est de pouvoir effectuer **un travail de manière asynchrone**, et de pouvoir le répéter en cas d'échec, ou bien
de notifier des développeurs en cas d'échec afin de corriger le problème et de rejouer le job.

En d'autre terme, on a une trace du travail à effectuer et certaines garanties qu'il sera effectué si le code qui l'exécute
n'échoue pas. C'est une solution extrêmement flexible.

Chaque Job est ensuite exécuté par un Worker, qui est généralement un processus distinct ou à minima un thread distinct.
Ce qui permet de paralléliser l'exécution des jobs.

Dans notre cas, on pourrait imaginer le scénario suivant : 
- Un job est dispatché pour **lister les commandes à mettre à jour**
- Un job est créé pour **chaque commande à mettre à jour**

L'avantage de cette approche et le haut parallélisme de l'opération : si on a 10.000 ou 100.000 commandes à mettre à jour, on peut
adapter le nombre de workers pour palier au besoin et mettre à jour tous les produits en quelques secondes seulement.

{{< notice tip >}}
**Pourquoi ne pas tout faire en un seul job** ? Première car un job est une barrière transactionnelle : imaginez avoir 100.000 commandes et que
pour une raison que l'on ignore, la 50.000e commande fait planter le job. Bien joué, vous avez perdu tout le travail des 50.000 premières commandes.

Vous voudrez avoir des jobs très court et très petits pour maximiser votre parallélisme et minimiser les risques.
{{< /notice >}}

Ce que l'on vient de faire, c'est de **L'Eventual Consistency** : on ne garantit pas que les commandes seront mises à jour immédiatement,
mais on garantit qu'elles le seront à un moment donné, généralement dans un terme court.

## Conclusions

Comme vous pouvez le voir, on peut très rapidement passer d'un invariant simple à un invariant extrêmement complexe qui nécessite
**une approche orienté objet selon la philosophie du Domain-Driven Design**. Vous voyez aussi comment la discussion avec les
experts métiers est le point de départ de toutes les décisions techniques qui ont été prise. Même si la solution finit par devenir
plutôt complexe, elle est à la mesure du problème qu'elle résout.

Notez que tous les Aggregates n'ont pas forcément besoin d'une solution aussi robuste. Ce qui a joué ici, c'est la volumétrie : on sait
qu'il peut y avoir beaucoup de commandes en attente, et qu'une mise à jour transactionnel aurait un impact désastreux sur les performances du produit.
La solution, face à la volumétrie, est d'être plus flexible sur l'invariant et d'ajouter de l'Eventual Consistency.

Mais si dans votre cas votre volumétrie est faible voir inexistante, vous pouvez effectuer toute la mise à jour dans une seule
et même transaction, ce qui est beaucoup plus simple et plus rapide à implémenter.