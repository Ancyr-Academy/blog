---
title: Comment tester les requêtes paginées ?
date: 2025-01-20T04:00:00+01:00
lastmod: 2025-01-20T04:00:00+01:00
author: Anthony Cyrille
categories:
  - testing
tags:
  - integration tests
draft: false
---

Dans vos back-ends & APIs, les requêtes paginées et les requêtes de listes sont assez complexes à tester car il y a plusieurs paramètres à prendre en compte :

- La rectitude de la data en elle-même
- Les différents filtres applicables
- L'ordre correct de la liste
- La quantité d'éléments dans la liste
- La pagination
- Et parfois même le filtrage automatique des entrées qu'on a pas le droit de voir 

Pour moi, il y a deux approches possibles :
- **L'approche simple**
- **L'approche complète**

# Approche simple

C'est celle que je favorise dans les circonstances suivantes :
- Le projet est petit ou personnel, sans grand risque ni grand besoin
- La requête en elle-même n'est pas très importante

Dans l'un ou l'autre de ces deux cas, je vais simplement m'assurer...
- Que la route API fonctionne (d'avoir une réponse avec un code de 200 entre autre)
- Que j'ai bien de la data (généralement 2/3 entrées)

```ts
describe('Feature: getting a list of users', () => {
  it('should return the list of users', async () => {
    const response = await app.request("/api/users");
    
    expect(response.statusCode).toBe(200);
    
    expect(response.body.info).toEqual({
      page: 1,
      total: 3,
      countPerPage: 10,
    });
    
    expect(response.body.data).toEqual([
      {
        id: "1",
        name: "Sherlock Holmes",
        age: 32
      },
      {
        id: "2",
        name: "Irène Adler",
        age: 27 
      },
      {
        id: "3",
        name: "John H. Watson",
        age: 31
      }
    ])
  })
})
```

Notez bien les caractéristiques de ce test :
- **C'est un test d'intégration**, il communique avec la vraie API et la vraie base de donnée
- Toutes les assertions sont **regroupées en un seul test** pour diminuer le temps d'exécution
- La liste est testée avec une stricte égalité car les valeurs retournées sont stables et contiennent peu de clés. 

{{< notice tip >}}
Si votre API retourne beaucoup d'information et que ces infos sont sujettes à changement, **évitez les égalités strictes**, sinon
vos tests joueront contre vous. **Testez les objets un à un avec des égalités partielles.**

```ts
expect(response.body.data[0]).toMatchEqual({
  id: "1",
  name: "Sherlock Holmes"
})
```
{{< /notice >}}

En bref, j'ai limité la portée du test à son maximum car je n'ai pas besoin de plus vu l'intérêt et la stabilité de l'API. 
Je peux toujours rajouter quelques tests manuels à l'occasion si besoin.

# Approche Complète

Lorsque le projet est sérieux, que la route API est importante aux yeux du client et que j'ai besoin de garantir une fonctionnalité
complète, je sors l'artillerie lourde.

Dans cet approche, je met largement de côté la performance d'exécution du test car l'important est le fonctionnement de la feature,
quitte à perdre 1s dans la pipeline. Il y a toujours moyen d'optimiser ou de grouper les tests en cas de besoin.

Il y a deux choses extrêmement importantes à prendre en compte dans cette approche :
- Le data-set, l'état de la base de donnée au moment de l'exécution des requêtes
- La décomposition des tests

## Le Data-Set

Sur des requêtes extrêmement complexes, on a tendance à avoir beaucoup de data dans plusieurs tables, notamment car certaines
requêtes font l'aggrégation de la data de plusieurs tables.

{{< notice tip >}}
Il est même possible que la requête SQL effectuée en fond soit une **procédure stockée de plusieurs milliers de lignes de code !**
{{< /notice >}}

C'est très importante de bien concevoir son data-set, et il y a généralement deux approches :
- **Un setup général de la base de donnée**, avant le lancement de tous les tests, auquel cas le même data-set est utilisé pour la totalité des tests
- **Un setup par groupe de test**, on peut grouper les tests par scénario et assigner un data-set par scénario

L'approche la plus simple et la plus optimisée est la première, car elle permet de ne charger le data-set qu'une fois pour l'ensemble des tests, 
ce qui fait économiser du temps d'exécution. Et puis, puisqu'on parle d'une query, la data ne sera pas modifiée, donc elle est réutilisable.

Et il y a plusieurs façons de construire ce data-set
- **Via un script SQL préconfiguré**
- **Via le code**

La méthode SQL est la plus rapide mais la moins simple à maintenir, je ne la recommande pas.
Je préfère créer mon data-set dans le code où j'ai un contrôle total de la création des entrées et où je peux me créer
un superbe **DSL** pour simplifier sa création et sa maintenance.

{{< notice tip >}}
On utilise parfois (_OK, souvent_) le terme **seed** au lieu de data-set, et le fait de remplir la base de données de test
avec cette data de test est appelée **seeding**.
{{< /notice >}}

Et ça, c'est extrêmement important : plus votre data-set est complexe, plus vous avez intérêt à vous créer votre propre langage 
de construction du data-set, même s'il ne sera utilisé **que pour cette suite de test**. 

C'est pour ça que l'approche complète
(qui est très lourde) a besoin d'être justifié par un gros besoin de s'assurer que la feature fonctionne, et que je l'utilise 
précautionneusement _(j'ai peiné à l'écrire celui là :D)_.

Un DSL ressemblerait à ça.

```ts
// tests/api/list-orders/seeds.ts

const seeds = new WarehouseBuilder()
  .withProduct(
    new ProductBuilder("product-1")
      .withPrice(ProductBuilder.Price(10.00, "EUR"))
      .withCostPerProduct(ProductBuilder.Price(3.99, "EUR")),
    new ProductBuilder("product-2")
      .withPrice(ProductBuilder.Price(10.00, "EUR"))
      .withCostPerProduct(ProductBuilder.Price(3.99, "EUR")),
  )
  withInventory(
    new InventoryBuilder("product-1")
      .fromLocation("In House")
      .withQuantity(17)
      .available(10),
    new InventoryBuilder("product-2")
      .fromLocation("Seller")
      .withQuantity(415)
      .available(397),
  )
  .addOrder(
    new OrderBuilder()
      .withLineItem("product-1", 3)
      .withLineItem("product-2", 1)
  )
// ... 
```

On a un seul fichier qui contient toute la configuration pour cette suite de test là et dont le seed est réutilisé
pour la totalité des tests. L'avantage, c'est que ce code est uniquement descriptif et facile à comprendre et à maintenir.

A côté, on a évidemment le code pour construire ce DSL.

```ts
// tests/api/list-orders/dsl.ts

export class WarehouseBuilder {
  // ...
}
```

Ensuite, j'aime regrouper les tests par scénarios en créant un fichier par scénario.

On peut par exemple partir du scénario le plus rudimentaire, dans lequel on applique aucun filtre spécifique ni ordre particulier et l'appeler `default.test.ts`.
A l'intérieur, chaque test enverrait exactement la même requête, mais testerais quelque chose de différent.

```ts
// tests/api/list-orders/default.test.ts

describe('Scenario: default call without any specific parameters', () => {
  test('the list should contain 10 elements', async () => {
    const response = await sendRequest();
    expect(response.body.data).toHaveLength(10);
  })
})

// On créé une fonction qui exécute la requête afin d'exécuter la même requête dans chaque test
// On pourrait utiliser `beforeEach` mais je ne suis pas fan de l'approche.
const sendRequest = () => {
  return app.request("/api/users", {
    page: 1,
    countPerPage: 10,
  })
}
```

Ensuite on déroule : un test par aspect que l'on veut tester.
Par exemple, la taille de la liste.

```ts
// tests/api/list-orders/default.test.ts

test('the list should contain 10 elements', async () => {
  const response = await sendRequest();
  expect(response.body.data).toHaveLength(10);
})
```

Puis le contenu de la liste. A noter qu'on peut tester l'entièreté des propriétés de la liste, ou seulement certaines.
On peut aussi vérifier que les clés attendues soient bien présentes et aient le bon type, mais on approche des tests qui
apportent peu de plus value et sont lourds à maintenir.


```ts
// tests/api/list-orders/default.test.ts

test('the items values must be correct', async () => {
  const response = await sendRequest();
  
  expect(response.body.data[0]).toEqual({
    id: "1",
    lineItems: [
      {
        id: "1",
        product: {
          id: "product-1",
          price: "10.00€",
        },
        quantity: 10,
        // ...
      }
    ]
  })
})
```

Je test le moins possible les valeurs de la liste, uniquement les entrées qui ont un comportement particulier. 
Par exemple, si un utilisateur a acheté un produit qui était à -10% au moment de l'achat, je vais
vérifier la règle pour **cette clé** là et **cette valeur spécifique** pour **cette entrée**, sans tester la totalité de l'entrée.

```ts
// tests/api/list-orders/default.test.ts

test('the second item price must be discounted', async () => {
  // Ce genre de test peut-être très complexe à comprendre.
  // Le seul moyen de savoir pourquoi on utilise "0" et "1" est de consulter le seed.
  // On cherchera à donner le maximum d'information sur le contexte directement dans le code, ici.
  const response = await sendRequest();
  
  const order = response.body.data[0];
  const item = order.lineItems[1];

  expect(item.price).toBe("9.00€")
})
```

Je teste maintenant l'ordre des éléments.

```ts
// tests/api/list-orders/default.test.ts

test('the items should be ordered from the latest created to the oldest', async () => {
  const response = await sendRequest();

  const expectedIds = ["3", "2", "1"];
  const actualIds = response.body.data.map(item => item.id);
  
  expect(actualIds).toEqual(expectedIds);
})
```

Je peux tester des contraintes de sécurité, également.

```ts
// tests/api/list-orders/default.test.ts

test('the private data should only appear on elements owner by the requester', async () => {
  const response = await sendRequest();

  const authorized = response.body.data[1];
  expect(authorized.privateData).toEqual({ /* ... */ });

  const notAuthorized = response.body.data[2];
  expect(notAuthorized.privateData).toEqual(null);
})
```

{{< notice tip >}}
**Pourquoi ne pas regrouper toutes les assertions dans un seul test ? Ce serait beaucoup plus rapide !**

Bonne question ! Il y a au moins 3 raisons qui me viennent en tête
- **Je favorise la lisibilité / maintenance** du test à ses performances car dans ce contexte là, c'est ce qui importe
- Séparer par tests permet de **titrer chaque test** et de lui donner du sens, donc de la lisibilité
- Et enfin le plus important : si un test échoue, **je sais exactement ce qui a échoué et pourquoi.**

Si je regroupe les assertions dans un seul test, **la première assertion a échoué bloquera les suivantes**. Résultat, j'ai
beaucoup moins de visibilité sur ce qui ne fonctionne vraiment plus, et je risque d'introduire un bug en corrigeant le premier.

C'est vraiment super important de tester séparément chaque aspect de la feature, même si ça prend plus de temps.
{{< /notice >}}

Bon, vous avez compris l'idée.
Ensuite, il me reste plus qu'à créer un fichier de test par scénario.

```ts
// tests/api/list-orders/filtering-by-products.test.ts
// tests/api/list-orders/sorting-by-discount.test.ts
// tests/api/list-orders/filtering-by-location.test.ts
```

Vous voyez, ça peut rapidement conduire à une centaine de tests. Donc gardez bien en tête :
- Que rien ne vous oblige à couvrir la totalité des scénarios possibles
- Et que rien ne vous oblige à tester la totalités des aspects d'un scénario

C'est du mix & match, on prend ce dont on a besoin pour gagner la maximum de fiabilité sans sacrifier la productivité. **Dés
que le retour sur investissement est trop faible, on arrête.**

Pour récapituler :
- **Approche Simple** : je veux juste vérifier que l'API fonctionne et est accessible
- **Approche Complète** : je veux vérifier que l'API fonctionne dans toutes les conditions possibles

Gardez en tête que l'approche complète consomme beaucoup de temps et de ressources. En pratique, je l'applique rarement.