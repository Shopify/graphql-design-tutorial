# Tutoriel : Conception d'une API GraphQL

Ce tutoriel a été créé par [Shopify](https://www.shopify.ca/) à des fins internes. Nous en avons créé une version publique car nous pensons qu'il est utile à toute personne créant une API GraphQL.

Il est basé sur les leçons tirées de la création et de l'évolution des schémas de production chez Shopify pendant près de 3 ans. Le tutoriel a évolué et continuera à changer à l'avenir, donc rien n'est figé.

Nous pensons que ces directives de conception fonctionnent dans la plupart des cas. Elles peuvent ne pas toutes fonctionner pour vous. Même au sein de l'entreprise, nous continuons à les remettre en question et à prévoir des exceptions, car la plupart des règles ne peuvent pas s'appliquer 100 % du temps. Ne vous contentez donc pas de les copier et de les appliquer aveuglément. Choisissez celles qui ont un sens pour vous et pour vos cas d'utilisation.

Table of Contents
=================
* [Intro](#intro)
* [Step Zero: Background](#step-zero-background)
* [Step One: A Bird's\-Eye View](#step-one-a-birds-eye-view)
* [Step Two: A Clean Slate](#step-two-a-clean-slate)
  * [Representing CollectionMemberships](#representing-collectionmemberships)
  * [Representing Collections](#representing-collections)
  * [Conclusion](#conclusion)
* [Step Three: Adding Detail](#step-three-adding-detail)
  * [Starting point](#starting-point)
  * [IDs and the Node Interface](#ids-and-the-node-interface)
  * [Rules and Subobjects](#rules-and-subobjects)
  * [Lists and Pagination](#lists-and-pagination)
  * [Strings](#strings)
  * [IDs and Relations](#ids-and-relations)
  * [Naming and Scalars](#naming-and-scalars)
  * [Pagination Again](#pagination-again)
  * [Enums](#enums)
* [Step Four: Business Logic](#step-four-business-logic)
* [Step Five: Mutations](#step-five-mutations)
  * [Separate Logical Actions](#separate-logical-actions)
  * [Manipulating Relationships](#manipulating-relationships)
  * [Input: Structure, Part 1](#input-structure-part-1)
  * [Input: Scalars](#input-scalars)
  * [Input: Structure, Part 2](#input-structure-part-2)
  * [Output](#output)
* [TLDR: The rules](#tldr-the-rules)
* [Conclusion](#conclusion-1)

## Intro

Bienvenue ! Ce document vous guidera dans la conception d'une nouvelle API GraphQL (ou d'une nouvelle partie d'une API GraphQL existante). La conception d'API est une tâche difficile qui récompense fortement l'itération, l'expérimentation et une compréhension approfondie de votre domaine d'activité.

## Étape zéro : Contexte

Pour les besoins de ce tutoriel, imaginez que vous travaillez dans une entreprise de commerce électronique.
Vous avez une API GraphQL existante qui expose des informations sur vos produits, mais très peu d'autres choses. Cependant, votre équipe vient de terminer un projet d'implémentation de "collections" dans le back-end et souhaite également exposer les collections via l'API.

Les collections sont la nouvelle méthode de prédilection pour regrouper des produits ; par exemple, vous pourriez avoir une collection de tous vos t-shirts. Les collections peuvent être utilisées à des fins d'affichage lors de la navigation sur votre site Web, mais aussi pour des tâches programmatiques (par exemple, vous pouvez souhaiter qu'une remise ne s'applique qu'aux produits d'une certaine collection).

En back-end, votre nouvelle fonctionnalité a été mise en œuvre comme suit :
- Toutes les collections ont des attributs simples comme un titre, un corps de description (qui peut inclure un formatage HTML) et une image.
- Vous avez deux types de collections spécifiques : les collections "manuelles", dans lesquelles vous dressez la liste des produits que vous souhaitez inclure, et les collections "automatiques", dans lesquelles vous spécifiez certaines règles et laissez la collection se remplir d'elle-même.
- Puisque la relation produit-collection est de type many-to-many, vous avez une table de jonction au milieu appelée `CollectionMembership`.
- Les collections, comme les produits avant elles, peuvent être publiées (visibles sur la vitrine) ou non.

Avec ce contexte, vous êtes prêt à commencer à réfléchir à la conception de votre API.

## Première étape : une vue à vol d'oiseau

Une version naïve du schéma pourrait ressembler à ceci (en laissant de côté tous les types préexistants comme `Product`) :
```graphql
interface Collection {
  id: ID!
  memberships: [CollectionMembership!]!
  title: String!
  imageId: ID
  bodyHtml: String
}

type AutomaticCollection implements Collection {
  id: ID!
  rules: [AutomaticCollectionRule!]!
  rulesApplyDisjunctively: Boolean!
  memberships: [CollectionMembership!]!
  title: String!
  imageId: ID
  bodyHtml: String
}

type ManualCollection implements Collection {
  id: ID!
  memberships: [CollectionMembership!]!
  title: String!
  imageId: ID
  bodyHtml: String
}

type AutomaticCollectionRule {
  column: String!
  relation: String!
  condition: String!
}

type CollectionMembership {
  collectionId: ID!
  productId: ID!
}
```

À première vue, c'est déjà assez compliqué, même s'il ne s'agit que de quatre objets et d'une interface. De plus, il est clair qu'elle n'implémente pas toutes les fonctionnalités dont nous aurions besoin si nous devions utiliser cette API pour développer, par exemple, la fonctionnalité de collecte de notre application mobile.

Prenons un peu de recul. Une API GraphQL relativement complexe se compose de nombreux objets, reliés par plusieurs chemins et comportant des dizaines de champs. Essayer de concevoir quelque chose comme cela d'un seul coup est une recette pour la confusion et les erreurs. Vous devriez plutôt commencer par une vue de plus haut niveau, en vous concentrant uniquement sur les types et leurs relations sans vous soucier des champs ou des mutations spécifiques.
En gros, pensez à un [modèle entité-relation] (https://en.wikipedia.org/wiki/Entity%E2%80%93relationship_model) mais avec quelques éléments spécifiques à GraphQL. Si nous réduisons notre schéma naïf de la sorte, nous obtenons ce qui suit :

```graphql
interface Collection {
  Image
  [CollectionMembership]
}

type AutomaticCollection implements Collection {
  [AutomaticCollectionRule]
  Image
  [CollectionMembership]
}

type ManualCollection implements Collection {
  Image
  [CollectionMembership]
}

type AutomaticCollectionRule { }

type CollectionMembership {
  Collection
  Product
}
```

Pour obtenir cette représentation simplifiée, j'ai supprimé tous les champs scalaires, tous les noms de champs et toutes les informations de nullité. Ce qui reste ressemble toujours un peu à GraphQL, mais vous permet de vous concentrer sur les types et leurs relations à un niveau plus élevé.

*Règle n° 1 : commencez toujours par une vue de haut niveau des objets et de leurs relations avant de vous occuper de champs spécifiques.*

## Deuxième étape : une table rase

Maintenant que nous avons quelque chose de simple sur lequel travailler, nous pouvons nous attaquer aux principaux défauts de cette conception.

Comme nous l'avons mentionné précédemment, notre implémentation définit l'existence de collections manuelles et automatiques, ainsi que l'utilisation d'une table de jonction de collecteurs. La conception naïve de notre API était clairement structurée autour de notre implémentation, mais c'était une erreur.

Le problème fondamental de cette approche est qu'une API fonctionne dans un but différent de celui d'une implémentation, et fréquemment à un niveau d'abstraction différent. Dans ce cas, notre mise en œuvre nous a égarés sur plusieurs fronts différents.

### Representing `CollectionMembership`s

Celui qui vous a peut-être déjà sauté aux yeux, et qui est, je l'espère, assez évident, est l'inclusion du type `CollectionMembership` dans le schéma. La table des membres de la collection est utilisée pour représenter la relation many-to-many entre les produits et les collections.
Relisez la dernière phrase : la relation est *entre les produits et les collections* ; d'un point de vue sémantique et commercial, les appartenances à une collection n'ont rien à voir avec quoi que ce soit. Il s'agit d'un détail d'implémentation.

Cela signifie qu'ils n'ont pas leur place dans notre API. Au lieu de cela, notre API doit exposer directement la relation réelle du domaine commercial avec les produits. Si nous supprimons les appartenances à une collection, la conception de haut niveau qui en résulte ressemble maintenant à ceci :

```graphql
interface Collection {
  Image
  [Product]
}

type AutomaticCollection implements Collection {
  [AutomaticCollectionRule]
  Image
  [Product]
}

type ManualCollection implements Collection {
  Image
  [Product]
}

type AutomaticCollectionRule { }
```

C'est beaucoup mieux.

*Règle n°2 : ne jamais exposer les détails de mise en œuvre dans la conception de votre API.*

### Représentation des collections

Cette conception d'API présente encore un défaut majeur, bien qu'il soit probablement moins évident sans une compréhension vraiment approfondie du domaine d'activité. Dans notre conception actuelle, nous modélisons les collections automatiques et les collections manuelles comme deux types différents, chacun implémentant une interface de collection commune. Intuitivement, cela est assez logique : ils ont beaucoup de champs en commun, mais sont encore nettement différents dans leurs relations (les AutomaticCollections ont des règles) et certains de leurs comportements.

Mais du point de vue du modèle d'entreprise, ces différences sont aussi essentiellement un détail d'implémentation. Le comportement déterminant d'une collection est qu'elle regroupe des produits ; la méthode de sélection de ces produits est secondaire. Nous pourrions étendre notre mise en œuvre à un moment donné pour permettre une troisième méthode de sélection des produits (apprentissage automatique ?) ou pour permettre de mélanger les méthodes (certaines règles et certains produits ajoutés manuellement) et *ils resteraient des collections*. On pourrait même dire que le fait que nous ne permettions pas le mélange à l'heure actuelle est un échec de la mise en œuvre. Tout cela pour dire que la forme de notre API devrait ressembler davantage à ceci :

```graphql
type Collection {
  [CollectionRule]
  Image
  [Product]
}

type CollectionRule { }
```

C'est vraiment bien. La préoccupation immédiate que vous pouvez avoir à ce stade est que nous prétendons maintenant que les ManualCollections ont des règles, mais n'oubliez pas que cette relation est une liste. Dans notre nouvelle API, une "ManualCollection" est simplement une collection dont la liste de règles est vide.

### Conclusion

Pour choisir la meilleure conception d'API à ce niveau d'abstraction, vous devez nécessairement avoir une compréhension très approfondie du domaine problématique que vous modélisez. Il est difficile, dans le cadre d'un tutoriel, de fournir un contexte aussi approfondi pour un sujet spécifique, mais nous espérons que la conception de la collection est suffisamment simple pour que le raisonnement ait un sens. Même si vous n'avez pas cette compréhension approfondie spécifiquement pour les collections, vous en avez absolument besoin pour le domaine que vous êtes en train de modéliser. Il est essentiel, lors de la conception de votre API, de vous poser ces questions difficiles et de ne pas vous contenter de suivre aveuglément la mise en œuvre.

Dans le même ordre d'idées, une bonne API ne modélise pas non plus l'interface utilisateur.
La mise en œuvre et l'interface utilisateur peuvent toutes deux servir de source d'inspiration et de contribution à la conception de votre API, mais le moteur final de vos décisions doit toujours être le domaine commercial.

Plus important encore, les choix existants en matière d'API REST ne doivent pas nécessairement être copiés. Les principes de conception qui sous-tendent REST et GraphQL peuvent conduire à des choix très différents. Ne partez pas du principe que ce qui a fonctionné pour votre API REST est un bon choix pour GraphQL.

Dans la mesure du possible, laissez votre bagage de côté et partez de zéro.

*Règle n° 3 : concevez votre API en fonction du domaine d'activité, et non de la mise en œuvre, de l'interface utilisateur ou des anciennes API.

## Étape 3 : Ajout de détails

Maintenant que nous avons une structure propre pour modéliser nos types, nous pouvons ajouter nos champs et recommencer à travailler à ce niveau de détail.

Avant de commencer à ajouter des détails, demandez-vous si vous en avez vraiment besoin pour le moment. Ce n'est pas parce qu'une colonne de base de données, une propriété de modèle ou un attribut REST existe qu'il faut automatiquement l'ajouter au schéma GraphQL.

L'exposition d'un élément du schéma (champ, argument, type, etc.) doit être motivée par un besoin réel et un cas d'utilisation. Les schémas GraphQL peuvent facilement évoluer par l'ajout d'éléments, mais les modifier ou les supprimer sont des changements brutaux et beaucoup plus difficiles.

*Règle n°4 : Il est plus facile d'ajouter des champs que de les supprimer.*

### Point de départ

En rétablissant nos champs naïfs ajustés pour notre nouvelle structure, nous obtenons :

```graphql
type Collection {
  id: ID!
  rules: [CollectionRule!]!
  rulesApplyDisjunctively: Boolean!
  products: [Product!]!
  title: String!
  imageId: ID
  bodyHtml: String
}

type CollectionRule {
  column: String!
  relation: String!
  condition: String!
}
```

Nous avons maintenant une toute nouvelle série de problèmes de conception à résoudre. Nous allons travailler sur les champs dans l'ordre, de haut en bas, en corrigeant les choses au fur et à mesure.

### Les IDs et l'interface `Node`.

Le tout premier champ de notre type de collection est un champ d'identification, ce qui est bien et normal ; cet ID est ce que nous devrons utiliser pour identifier nos collections dans l'API, en particulier lorsque nous effectuons des actions comme les modifier ou les supprimer.
Cependant, il manque un élément dans cette partie de notre conception : l'interface `Node`. Il s'agit d'une interface très courante qui existe déjà dans la plupart des schémas et qui ressemble à ceci :
```graphql
interface Node {
  id: ID!
}
```
Il indique au client que cet objet est persistant et récupérable par l'ID donné, ce qui permet au client de gérer précisément et efficacement les caches locaux et autres astuces. La plupart de vos principaux objets commerciaux identifiables (par exemple, les produits, les collections, etc) devraient implémenter `Node`.

Le début de notre conception ressemble maintenant juste à :
```graphql
type Collection implements Node {
  id: ID!
}
```

*Règle n°5 : Les principaux types d'objets commerciaux doivent toujours implémenter `Node`.*

### Règles et sous-objets

Nous allons considérer ensemble les deux champs suivants de notre type Collection : `rules`, et `rulesApplyDisjunctively`. Le premier est assez simple : une liste de règles. Notez que la liste elle-même et les éléments de la liste sont marqués comme non nuls : c'est bien, car GraphQL fait la distinction entre `null` et `[]` et `[null]`. Pour les collections manuelles, cette liste peut être vide, mais elle ne peut pas être nulle ni contenir un null.

*Astuce : Les champs de type liste sont presque toujours des listes non nulles avec des éléments non nuls. Si vous voulez une liste nullable, assurez-vous qu'il y a une réelle valeur sémantique à pouvoir faire la distinction entre une liste vide et une liste null.*

Le deuxième champ est un peu bizarre : c'est un champ booléen indiquant si les règles s'appliquent de manière disjonctive ou non. Il est également non nul, mais nous rencontrons ici un problème : quelle valeur doit prendre ce champ pour les collections manuelles ? Le fait qu'il soit soit faux ou vrai semble trompeur, mais le fait que le champ puisse être nul en fait une sorte d'indicateur tri-state bizarre qui est également gênant lorsqu'il s'agit de collectes automatiques. Pendant que nous nous interrogeons sur ce point, il y a une autre chose qui mérite d'être mentionnée : ces deux champs sont manifestement et intimement liés. C'est vrai d'un point de vue sémantique, et c'est également suggéré par le fait que nous avons choisi des noms avec un préfixe commun. Existe-t-il un moyen d'indiquer cette relation dans le schéma d'une manière ou d'une autre ?

En fait, nous pouvons résoudre tous ces problèmes d'un seul coup en nous écartant encore plus de notre implémentation sous-jacente et en introduisant un nouveau type GraphQL sans équivalent direct dans le modèle : `CollectionRuleSet`. Cela est souvent justifié lorsque vous avez un ensemble de champs étroitement liés dont les valeurs et le comportement sont liés. En regroupant les deux champs dans leur propre type au niveau de l'API, nous fournissons un indicateur sémantique clair et résolvons également tous nos problèmes de nullité : pour les collections manuelles, c'est l'ensemble de règles lui-même qui est nul. Le champ booléen peut rester non nul. Cela nous conduit à la conception suivante :

```graphql
type Collection implements Node {
  id: ID!
  ruleSet: CollectionRuleSet
  products: [Product!]!
  title: String!
  imageId: ID
  bodyHtml: String
}

type CollectionRuleSet {
  rules: [CollectionRule!]!
  appliesDisjunctively: Boolean!
}

type CollectionRule {
  column: String!
  relation: String!
  condition: String!
}
```

*Astuce : Comme les listes, les champs booléens sont presque toujours non nuls. Si vous voulez un booléen annulable, assurez-vous qu'il y a une réelle valeur sémantique à pouvoir distinguer les trois états (null/false/true) et que cela n'indique pas un défaut de conception plus important*.

*Règle n°6 : regroupez les champs étroitement liés dans des sous-objets.

### Listes et pagination

Le champ "produits" est le prochain à passer à la trappe. Celui-ci pourrait sembler sûr ; après tout, nous avons déjà "corrigé" cette relation lorsque nous avons supprimé notre type `CollectionMembership`, mais en fait, il y a quelque chose d'autre qui ne va pas ici.

Le champ tel qu'il est défini actuellement renvoie un tableau de produits, mais les collections peuvent facilement avoir plusieurs dizaines de milliers de produits, et essayer de tous les rassembler dans un seul tableau serait incroyablement coûteux et inefficace. Pour des situations comme celle-ci, GraphQL fournit une pagination des listes.

Chaque fois que vous implémentez un champ ou une relation retournant plusieurs objets, demandez-vous toujours si le champ doit être paginé ou non. Combien d'exemplaires de cet objet peut-il y avoir ? Quelle quantité est considérée comme pathologique ?

Paginer un champ signifie que vous devez d'abord mettre en œuvre une solution de pagination.
Ce tutoriel utilise [Connections] (https://graphql.org/learn/pagination/#complete-connection-model) qui est défini par le [Relay Connection spec] (https://facebook.github.io/relay/graphql/connections.htm).

Dans ce cas, la pagination du champ products dans notre conception est aussi simple que de changer sa définition en `products : ProductConnection!`. En supposant que vous avez implémenté les connexions, vos types ressembleront à ceci :

```graphql
type ProductConnection {
  edges: [ProductEdge!]!
  pageInfo: PageInfo!
}

type ProductEdge {
  cursor: String!
  node: Product!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
}
```


*Règle n°7 : Toujours vérifier si les champs de la liste doivent être paginés ou non.*

### Chaînes de caractères

Le suivant est le champ `title`. Celui-ci est légitimement bien tel qu'il est. C'est une simple chaîne de caractères, et il est marqué non-null parce que toutes les collections doivent avoir un titre.

*Astuce : Comme pour les booléens et les listes, il est important de noter que GraphQL fait la distinction entre les chaînes vides (`""`) et les nulles (`null`), donc si vous avez besoin d'une chaîne nullable, assurez-vous qu'il y a une différence sémantique légitime entre non-présent (`null`) et présent-mais-vide (`""`). Vous pouvez souvent penser que les chaînes vides signifient "applicable, mais pas présent", et les chaînes nulles signifient "non applicable".*

#### IDs et relations

Nous en arrivons maintenant au champ `imageId`. Ce champ est un exemple classique de ce qui se passe lorsque vous essayez d'appliquer les conceptions REST à GraphQL. Dans les API REST, il est assez courant d'inclure les ID d'autres objets dans votre réponse comme un moyen de relier ces objets entre eux, mais c'est un anti-modèle majeur dans GraphQL.
Au lieu de fournir un ID et de forcer le client à faire un autre aller-retour pour obtenir des informations sur l'objet, nous devrions simplement inclure l'objet directement dans le graphe - c'est à cela que sert GraphQL après tout. Dans les API REST, ce modèle n'est souvent pas pratique, car il gonfle considérablement la taille de la réponse lorsque les objets inclus sont volumineux. Cependant, cela fonctionne bien en GraphQL, car chaque champ doit être explicitement interrogé, sinon le serveur ne le renverra pas.

En règle générale, les seuls champs d'identification dans votre conception doivent être les identifiants de l'objet lui-même. Chaque fois que vous avez un autre champ ID, il devrait probablement être une référence d'objet à la place. En appliquant cela à notre schéma jusqu'à présent, nous obtenons :

```graphql
type Collection implements Node {
  id: ID!
  ruleSet: CollectionRuleSet
  products: ProductConnection!
  title: String!
  image: Image
  bodyHtml: String
}

type Image {
  id: ID!
}

type CollectionRuleSet {
  rules: [CollectionRule!]!
  appliesDisjunctively: Boolean!
}

type CollectionRule {
  column: String!
  relation: String!
  condition: String!
}
```

*Règle n°8 : Toujours utiliser des références d'objets au lieu de champs d'identification.*

### Nommage et scalaires

Le dernier champ de notre simple type `Collection` est `bodyHtml`. Pour un utilisateur qui n'est pas familier avec la façon dont les collections ont été implémentées, il n'est pas évident de savoir à quoi sert ce champ ; c'est la description du corps de la collection spécifique. La première chose que nous pouvons faire pour améliorer cette API est de la renommer en `description`, un nom beaucoup plus clair.

*Règle n°9 : choisissez les noms des champs en fonction de ce qui est logique, et non en fonction de l'implémentation ou du nom du champ dans les anciennes API.*

Ensuite, nous pouvons le rendre non nul. Comme nous l'avons vu avec le champ titre, il n'est pas utile de faire la distinction entre un champ nul et une simple chaîne vide, c'est pourquoi nous ne l'exposons pas dans l'API. Même si le schéma de votre base de données permet aux enregistrements d'avoir une valeur nulle pour cette colonne, nous pouvons le cacher au niveau de l'implémentation.

Enfin, nous devons nous demander si `String` est réellement le bon type pour ce champ. GraphQL fournit un ensemble décent de types scalaires intégrés (`String`, `Int`, `Boolean`, etc) mais il vous permet également de définir vos propres types, et c'est un cas d'utilisation privilégié pour cette fonctionnalité. La plupart des schémas définissent leur propre ensemble de scalaires supplémentaires en fonction de leur utilisation. Ceux-ci fournissent un contexte supplémentaire et une valeur sémantique pour les clients. Dans ce cas, il est probablement judicieux de définir un scalaire `HTML` personnalisé à utiliser ici (et potentiellement ailleurs) lorsque la chaîne en question doit être du HTML valide.

Chaque fois que vous ajoutez un champ scalaire, cela vaut la peine de vérifier votre liste existante de scalaires personnalisés pour voir si l'un d'entre eux ne serait pas mieux adapté. Si vous ajoutez un champ et que vous pensez qu'un nouveau scalaire personnalisé serait approprié, il est bon d'en discuter avec votre équipe pour vous assurer que vous saisissez le bon concept.

*Règle n°10 : Utilisez des types de scalaires personnalisés lorsque vous exposez quelque chose ayant une valeur sémantique spécifique.

### Pagination à nouveau

Cela couvre tous les champs de notre type principal `Collection`. L'objet suivant est `CollectionRuleSet`, qui est assez simple. La seule question ici est de savoir si la liste des règles doit être paginée ou non. Dans ce cas, le tableau existant a du sens ; paginer la liste des règles serait trop compliqué. La plupart des collections n'auront qu'une poignée de règles, et il n'y a pas de bon cas d'utilisation pour une collection d'avoir un grand ensemble de règles. Même une douzaine de règles est probablement un indicateur que vous devez repenser cette collection, ou que vous devriez simplement ajouter des produits manuellement.

### Enums

Ceci nous amène au dernier type de notre schéma, `CollectionRule`. Chaque règle est constituée d'une colonne sur laquelle elle doit s'appuyer (par exemple, le titre du produit), d'un type de relation (par exemple, l'égalité) et d'une valeur réelle à utiliser (par exemple, "Boots") qui est confusément appelée `condition`. Ce dernier champ peut être renommé, de même que `column` ; column est une terminologie très spécifique aux bases de données, et nous travaillons en GraphQL. `field` est probablement un meilleur choix.

En ce qui concerne les types, `field` et `relation` sont probablement implémentés en interne comme des énumérations (en supposant que votre langage de prédilection ait même des énumérations). Heureusement GraphQL a aussi des énumérations, donc nous pouvons convertir ces deux champs en énumérations. Notre schéma complet ressemble maintenant à ceci :

```graphql
type Collection implements Node {
  id: ID!
  ruleSet: CollectionRuleSet
  products: ProductConnection!
  title: String!
  image: Image
  description: HTML!
}

type CollectionRuleSet {
  rules: [CollectionRule!]!
  appliesDisjunctively: Boolean!
}

type CollectionRule {
  field: CollectionRuleField!
  relation: CollectionRuleRelation!
  value: String!
}

enum CollectionRuleField {
  TAG
  TITLE
  TYPE
  INVENTORY
  PRICE
  VENDOR
}

enum CollectionRuleRelation {
  CONTAINS
  ENDS_WITH
  EQUALS
  GREATER_THAN
  LESS_THAN
  NOT_CONTAINS
  NOT_EQUALS
  STARTS_WITH
}
```

*Règle n°11 : Utilisez les enums pour les champs qui ne peuvent prendre qu'un ensemble spécifique de valeurs*.

## Étape 4 : Logique d'entreprise

Nous disposons maintenant d'une API GraphQL minimale mais bien conçue pour les collections. Les collections comportent de nombreux détails que nous n'avons pas abordés - toute mise en œuvre réelle de cette fonctionnalité nécessiterait beaucoup plus de champs pour gérer des éléments tels que l'ordre de tri des produits, la publication, etc. - mais en règle générale, ces champs suivront tous les mêmes modèles de conception que ceux présentés ici. Cependant, il y a encore quelques éléments qui méritent d'être examinés plus en détail.

Pour cette section, il est plus pratique de commencer par un cas d'utilisation motivant du client hypothétique de notre API. Imaginons donc que le développeur client avec lequel nous avons travaillé ait besoin de savoir quelque chose de très spécifique : si un produit donné est membre d'une collection ou non. Bien sûr, le client peut déjà répondre à cette question avec notre API existante : nous exposons l'ensemble des produits d'une collection, et le client n'a plus qu'à les parcourir, à la recherche du produit qui l'intéresse.

Cette solution présente toutefois deux problèmes. Le premier, évident, est qu'elle est inefficace : les collections peuvent contenir des millions de produits et il serait extrêmement lent pour le client de les récupérer et de les parcourir. Le second problème, plus important, est que le client doit écrire du code. Ce dernier point est un élément essentiel de la philosophie de conception : le serveur doit toujours être la source unique de vérité pour toute logique commerciale. Une API existe presque toujours pour servir plus d'un client, et si chacun de ces clients doit mettre en œuvre la même logique, vous avez effectivement une duplication du code, avec tout le travail supplémentaire et la marge d'erreur que cela implique.

*Règle n° 12 : l'API doit fournir une logique commerciale, et pas seulement des données. Les calculs complexes doivent être effectués sur le serveur, à un seul endroit, et non sur le client, à plusieurs endroits.

Pour en revenir au cas d'utilisation de notre client, la meilleure solution consiste à créer un nouveau champ spécifiquement dédié à la résolution de ce problème. En pratique, cela ressemble à ceci :
```graphql
type Collection implements Node {
  # ...
  hasProduct(id: ID!): Boolean!
}
```
Ce champ prend l'ID d'un produit et renvoie un booléen en fonction de la détermination par le serveur de la présence ou non d'un produit dans la collection. Le fait que ce champ duplique en quelque sorte les données du champ `products` existant n'est pas pertinent. GraphQL ne renvoie que ce que les clients demandent explicitement, donc contrairement à REST, cela ne nous coûte rien d'ajouter un tas de champs secondaires. Le client n'a pas à écrire de code au-delà de la requête d'un champ supplémentaire, et la bande passante totale utilisée est un seul ID plus un seul booléen.

Une mise en garde s'impose toutefois : ce n'est pas parce que nous fournissons la logique métier dans une situation donnée que nous ne devons pas également fournir les données brutes. Les clients doivent être en mesure d'effectuer la logique métier eux-mêmes, s'ils le doivent. Vous ne pouvez pas prédire toute la logique qu'un client va vouloir, et il n'y a pas toujours un canal facile pour les clients de demander des champs supplémentaires (bien que vous devriez vous efforcer d'assurer qu'un tel canal existe autant que possible).

*Règle n°13 : Fournissez également les données brutes, même si elles sont entourées d'une logique commerciale.

Enfin, ne laissez pas les champs de logique métier affecter la forme générale de l'API.
Les données du domaine métier restent le modèle de base. Si vous trouvez que la logique métier ne s'adapte pas vraiment, c'est le signe que votre modèle sous-jacent n'est peut-être pas le bon.

### Cinquième étape : Mutations

La dernière pièce manquante de notre conception du schéma GraphQL est la capacité à modifier réellement les valeurs : création, mise à jour et suppression des collections et des éléments connexes.
Comme pour la partie lisible du schéma, nous devrions commencer par une vue de haut niveau : dans ce cas, des diverses mutations que nous voulons mettre en œuvre, sans nous soucier de leurs entrées ou sorties spécifiques. Naïvement, nous pourrions suivre le paradigme CRUD et n'avoir que des mutations `create`, `delete`, et `update`.
Bien que ce soit un bon point de départ, c'est insuffisant pour une API GraphQL correcte.

### Actions logiques séparées

La première chose que nous pourrions remarquer si nous nous en tenions au CRUD est que notre mutation `update` devient rapidement massive, responsable non seulement de la mise à jour de simples valeurs scalaires comme le titre, mais aussi de l'exécution d'actions complexes comme la publication/dé-publication, l'ajout/suppression/réorganisation des produits dans la collection, la modification des règles pour les collections automatiques, etc. Cela rend l'implémentation difficile sur le serveur et le raisonnement difficile pour le client.
Au lieu de cela, nous pouvons tirer parti de GraphQL pour le diviser en actions logiques plus granulaires. Dans un premier temps, nous pouvons séparer les opérations de publication et de dépublication, ce qui donne la liste de mutations suivante :
- créer
- supprimer
- mettre à jour
- publier
- dépublier

*Règle n°14 : écrire des mutations distinctes pour des actions logiques distinctes sur une ressource.*

### Manipulation des relations

La mutation `update` a encore beaucoup trop de responsabilités, il est donc logique de continuer à la diviser, mais nous allons traiter ces actions séparément car elles valent la peine d'être considérées sous une autre dimension : la manipulation des relations entre objets (par exemple, one-to-many, many-to-many). Nous avons déjà examiné l'utilisation des ID par rapport à l'intégration, et l'utilisation de la pagination par rapport aux tableaux dans l'API de lecture, et il y a des problèmes similaires à traiter lors de la modification de ces relations.

En ce qui concerne la relation entre les produits et les collections, il existe plusieurs styles que nous pourrions envisager de manière générale :
- Intégrer la relation entière (par exemple `produits : [EntréeProduit !]!`) dans la mutation de mise à jour est le style CRUD par défaut, mais bien sûr, cela devient rapidement inefficace lorsque la liste est grande.
- Intégrer des champs "delta" (par exemple `produitsToAdd : [ID !]!` et `produitsToRemove : [ID !]!`) dans la mutation de mise à jour est plus efficace puisque seuls les ID modifiés doivent être spécifiés au lieu de la liste entière, mais cela permet de garder les actions liées entre elles.
- La division complète en mutations séparées (`addProduct`, `removeProduct`, etc.) est la plus puissante et la plus flexible mais aussi la plus laborieuse.

La dernière option est généralement la plus sûre, d'autant plus que les mutations de ce type sont généralement des actions logiques distinctes de toute façon. Cependant, il y a beaucoup de facteurs à prendre en compte :
- La relation est-elle volumineuse ou paginée ? Si c'est le cas, l'intégration de la liste entière n'est certainement pas pratique, mais les champs delta ou les mutations séparées peuvent toujours fonctionner. Si la relation est toujours petite (surtout si elle est de type un à un), l'incorporation peut être le choix le plus simple.
- La relation est-elle ordonnée ? La relation produit-collection est ordonnée et permet un réordonnancement manuel. L'ordre est naturellement pris en charge par la liste intégrée ou par des mutations séparées (vous pouvez ajouter une mutation `reorderProducts`) mais ce n'est pas une option pour les champs delta.
- La relation est-elle obligatoire ? Les produits et les collections peuvent tous deux exister indépendamment de la relation, avec leur propre cycle de vie de création/suppression.
  Si la relation était obligatoire (c'est-à-dire que les produits doivent faire partie d'une collection), cela suggérerait fortement des mutations séparées, car l'action consisterait en fait à *créer* un produit, et pas seulement à mettre à jour la relation.
- Les deux parties ont-elles des identifiants ? La relation collection-règle est obligatoire (les règles ne peuvent pas exister sans les collections) mais les règles n'ont même pas d'ID ; elles sont clairement subordonnées à leur collection, et puisque la relation est également petite, l'intégration de la liste n'est pas un mauvais choix ici. Toute autre solution nécessiterait que les règles soient identifiables individuellement et cela semble excessif.

*Règle n°15 : La mutation des relations est vraiment compliquée et ne se résume pas facilement en une règle rapide.*

Si vous mélangez tout cela, pour les collections, nous obtenons la liste suivante de mutations :
- créer
- supprimer
- mettre à jour
- publier
- dépublier
- ajouterProduits
- supprimer des produits
- reorderProducts

Les produits sont séparés en leurs propres mutations, car la relation est importante et ordonnée. Les règles sont laissées en ligne parce que la relation est petite, et les règles sont suffisamment mineures pour ne pas avoir d'ID.

Enfin, vous pouvez noter que nos mutations de produits agissent sur des ensembles de produits, par exemple `addProducts` et non `addProduct`. Il s'agit simplement d'une commodité pour le client, puisque le cas d'utilisation le plus courant lors de la manipulation de cette relation sera d'ajouter, de supprimer ou de réorganiser plus d'un produit à la fois.

*Règle n°16 : Lorsque vous écrivez des mutations distinctes pour les relations, demandez-vous s'il serait utile que les mutations opèrent sur plusieurs éléments à la fois.

### Entrée : Structure, 1ère partie

Maintenant que nous savons quelles mutations nous voulons écrire, nous devons déterminer à quoi ressemblent leurs structures d'entrée. Si vous avez parcouru l'un des schémas de production réels qui sont disponibles publiquement, vous avez peut-être remarqué que de nombreuses mutations définissent un seul type global `Input` pour contenir tous leurs arguments : ce modèle était une exigence de certains anciens clients mais n'est plus nécessaire pour le nouveau code ; nous pouvons l'ignorer.

Pour de nombreuses mutations simples, un identifiant ou une poignée d'identifiants suffisent, ce qui rend cette étape assez simple. Parmi les collections, nous pouvons rapidement éliminer les arguments de mutation suivants :
- `delete`, `publish` et `unpublish` n'ont besoin que d'un seul ID de collection.
- `addProducts` et `removeProducts` ont tous deux besoin de l'identifiant de la collection ainsi que d'une liste d'identifiants de produits.

Il ne nous reste donc que trois entrées "compliquées" à concevoir :
- créer
- mettre à jour
- réorganiser les produits

Commençons par create. Une entrée très naïve pourrait ressembler au modèle de collection naïf que nous avons utilisé au départ, mais nous pouvons déjà faire mieux que cela.
Sur la base de notre modèle de collection final et de la discussion sur les relations ci-dessus, nous pouvons commencer avec quelque chose comme ceci :

```graphql
type Mutation {
  collectionDelete(collectionId: ID!)
  collectionPublish(collectionId: ID!)
  collectionUnpublish(collectionId: ID!)
  collectionAddProducts(collectionId: ID!, productIds: [ID!]!)
  collectionRemoveProducts(collectionId: ID!, productIds: [ID!]!)
  collectionCreate(title: String!, ruleSet: CollectionRuleSetInput, image: ImageInput, description: HTML!)
}

input CollectionRuleSetInput {
  rules: [CollectionRuleInput!]!
  appliesDisjunctively: Boolean!
}

input CollectionRuleInput {
  field: CollectionRuleField!
  relation: CollectionRuleRelation!
  value: String!
}
```

Tout d'abord, une petite remarque sur le nommage : vous remarquerez que nous avons nommé toutes nos mutations sous la forme `collection<Action>` plutôt que sous la forme plus naturelle `<action>Collection`. Malheureusement, GraphQL ne fournit pas de méthode pour regrouper ou organiser les mutations, nous sommes donc obligés d'utiliser l'ordre alphabétique comme solution de rechange. En plaçant le type principal en premier, on s'assure que toutes les mutations liées sont regroupées dans la liste finale.

*Règle n°17 : Préfixez les noms des mutations avec l'objet qu'elles mutent pour un regroupement alphabétique (par exemple, utilisez `orderCancel` au lieu de `cancelOrder`).*

### Entrée : Scalars

Cette ébauche est bien meilleure qu'une approche complètement naïve, mais elle n'est toujours pas parfaite. En particulier, le champ de saisie `description` a quelques problèmes. Un champ `HTML` non nul a du sens pour la sortie de la description d'une collection, mais il ne fonctionne pas aussi bien en entrée pour plusieurs raisons. Tout d'abord, alors que `!` indique la non-nullité en sortie, il ne signifie pas la même chose en entrée ; il indique plutôt si un champ est "requis". Un champ obligatoire est un champ que le client doit fournir pour que la requête soit traitée, et ce n'est pas le cas pour `description`. Nous ne voulons pas empêcher les clients de créer des collections s'ils ne fournissent pas de description (ou de manière équivalente, nous ne voulons pas les forcer à fournir un `""` inutile), donc nous devrions rendre `description` non requis.

*Règle n°18 : Ne rendez les champs de saisie obligatoires que s'ils sont sémantiquement nécessaires à la poursuite de la mutation.*

L'autre problème avec `description` est son type ; cela peut sembler contre-intuitif puisqu'il est déjà fortement typé (`HTML` au lieu de `String`) et que nous avons tout fait pour un typage fort jusqu'ici. Mais encore une fois, les entrées se comportent un peu différemment.
La validation du typage fort sur les entrées se fait au niveau de la couche GraphQL avant que le code "userspace" ne soit exécuté, ce qui signifie que les clients doivent faire face à deux couches d'erreurs : Les erreurs de validation de la couche GraphQL, et les erreurs de validation de la couche métier (par exemple quelque chose comme : vous avez atteint la limite des collections que vous pouvez créer avec votre stockage actuel). Afin de simplifier ce processus, nous avons intentionnellement utilisé des champs de saisie faiblement typés lorsqu'il pourrait être difficile pour le client de les valider en amont. Cela permet au côté logique de l'entreprise de gérer toute la validation et au client de ne traiter les erreurs qu'à un seul endroit.

*Règle n° 19 : Utilisez des types plus faibles pour les entrées (par exemple `String` au lieu de `Email`) lorsque le format est sans ambiguïté et que la validation côté client est complexe. Cela permet au serveur d'exécuter toutes les validations non triviales en une seule fois et de retourner les erreurs à un seul endroit et dans un seul format, ce qui simplifie le client.

Il est important de noter, cependant, que ce n'est pas une invitation à typer faiblement toutes vos entrées. Nous utilisons toujours des enums fortement typés pour les valeurs `field` et `relation` de notre entrée de règle, et nous utiliserions toujours le typage fort pour certaines autres entrées comme `DateTime`s si nous en avions dans cet exemple. Les principaux facteurs de différenciation sont la complexité de la validation côté client et l'ambiguïté du format. Le HTML est une spécification bien définie et sans ambiguïté, mais il est assez complexe à valider. D'autre part, il existe des centaines de façons de représenter une date ou une heure sous forme de chaîne de caractères, toutes raisonnablement simples, et il est donc utile d'utiliser un type scalaire fort pour spécifier le format attendu.

*Règle n°20 : Utilisez des types plus forts pour les entrées (par exemple `DateTime` au lieu de `String`) lorsque le format peut être ambigu et que la validation côté client est simple. Cela apporte de la clarté et encourage les clients à utiliser des contrôles de saisie plus stricts (par exemple, un widget de sélection de date au lieu d'un champ de texte libre).

### Entrée : Structure, partie 2

Si l'on poursuit avec la mutation de mise à jour, cela pourrait ressembler à quelque chose comme ceci :

```graphql
type Mutation {
  # ...
  collectionCreate(title: String!, ruleSet: CollectionRuleSetInput, image: ImageInput, description: String)
  collectionUpdate(collectionId: ID!, title: String, ruleSet: CollectionRuleSetInput, image: ImageInput, description: String)
}
```

Vous remarquerez que cette mutation est très similaire à notre mutation `create`, avec deux différences : un argument `collectionId` a été ajouté, qui détermine quelle collection doit être mise à jour, et `title` n'est plus requis puisque la collection doit déjà en avoir un. Si l'on ignore pour l'instant le statut obligatoire du titre, nos exemples de mutations comportent quatre arguments en double, et un modèle de collection complet en comporterait bien d'autres.

Bien que certains arguments plaident en faveur du maintien de ces mutations telles quelles, nous avons décidé que dans des situations comme celle-ci, il est préférable d'éliminer les parties communes des arguments, même au prix de l'exigence. Cela présente quelques avantages :
- Nous nous retrouvons avec un seul objet d'entrée représentant le concept de collection et reflétant le type unique `Collection` que notre schéma possède déjà.
- Les clients peuvent partager du code entre leurs formulaires de création et de mise à jour (un modèle commun) car ils manipulent le même type d'objet d'entrée.
- Les mutations restent légères et lisibles avec seulement quelques arguments de haut niveau.

Le coût principal, bien sûr, est qu'il n'est plus clair dans le schéma que le titre est requis à la création. Notre schéma finit par ressembler à ceci :

```graphql
type Mutation {
  # ...
  collectionCreate(collection: CollectionInput!)
  collectionUpdate(collectionId: ID!, collection: CollectionInput!)
}

input CollectionInput {
  title: String
  ruleSet: CollectionRuleSetInput
  image: ImageInput
  description: String
}
```

*Règle n°21 : Structurer les entrées de mutation afin de réduire les doublons, même si cela nécessite de relâcher les contraintes d'exigence sur certains champs*.

### Sortie

La dernière question de conception que nous devons traiter concerne la valeur de retour de nos mutations. En général, les mutations peuvent réussir ou échouer, et bien que GraphQL comprenne un support explicite pour les erreurs au niveau de la requête, celles-ci ne sont pas idéales pour les échecs de mutation au niveau de l'entreprise. Nous réservons plutôt ces erreurs de haut niveau aux échecs du client (par exemple, la demande d'un champ inexistant) plutôt qu'à ceux de l'utilisateur. Ainsi, chaque mutation devrait définir un type de "charge utile" qui inclut un champ d'erreurs de l'utilisateur en plus de toute autre valeur qui pourrait être utile. Pour créer, cela pourrait ressembler à ceci :

```graphql
type CollectionCreatePayload {
  userErrors: [UserError!]!
  collection: Collection
}

type UserError {
  message: String!

  # Path to input field which caused the error.
  field: [String!]
}
```

Ici, une mutation réussie renverrait une liste vide pour `userErrors` et renverrait la collection nouvellement créée pour le champ `collection`. Une mutation non réussie renverrait un ou plusieurs objets `UserError`, et `null` pour la collection.

*Règle n°22 : Les mutations doivent fournir des erreurs au niveau de l'utilisateur ou de l'entreprise via un champ `userErrors` dans les données utiles de la mutation. L'entrée d'erreurs de requête de niveau supérieur est réservée aux erreurs de niveau client et serveur.*

Dans de nombreuses implémentations, une grande partie de cette structure est fournie automatiquement, et tout ce que vous aurez à définir est le champ de retour `collection`.

Pour la mutation de mise à jour, nous suivons exactement le même schéma :

```graphql
type CollectionUpdatePayload {
  userErrors: [UserError!]!
  collection: Collection
}
```

Il est intéressant de noter que `collection` est toujours nullable même ici, puisque si l'ID fourni ne représente pas une collection valide, il n'y a pas de collection à retourner.

*Règle n°23 : La plupart des champs payload d'une mutation devraient être nullables, à moins qu'il n'y ait vraiment une valeur à retourner dans tous les cas d'erreur possibles*.

## TLDR : Les règles

- Règle n° 1 : commencez toujours par une vue de haut niveau des objets et de leurs relations avant de vous occuper des champs spécifiques.
- Règle n° 2 : n'exposez jamais les détails de mise en œuvre dans la conception de votre API.
- Règle n° 3 : Concevez votre API en fonction du domaine de l'entreprise, et non de la mise en œuvre, de l'interface utilisateur ou des anciennes API.
- Règle n° 4 : il est plus facile d'ajouter des champs que de les supprimer.
- Règle n° 5 : les principaux types d'objets commerciaux doivent toujours implémenter Node.
- Règle n° 6 : regroupez les champs étroitement liés dans des sous-objets.
- Règle n° 7 : vérifiez toujours si les champs de liste doivent être paginés ou non.
- Règle n° 8 : utilisez toujours des références d'objet au lieu de champs d'identification.
- Règle n° 9 : choisissez les noms de champ en fonction de ce qui est logique, et non en fonction de l'implémentation ou du nom du champ dans les anciennes API.
- Règle n° 10 : utilisez des types scalaires personnalisés lorsque vous exposez quelque chose ayant une valeur sémantique spécifique.
- Règle n° 11 : Utilisez des enums pour les champs qui ne peuvent prendre qu'un ensemble spécifique de valeurs.
- Règle n° 12 : l'API doit fournir une logique d'entreprise, et pas seulement des données. Les calculs complexes doivent être effectués sur le serveur, à un seul endroit, et non sur le client, à plusieurs endroits.
- Règle n° 13 : Fournissez également les données brutes, même si elles sont accompagnées d'une logique métier.
- Règle n° 14 : écrivez des mutations distinctes pour des actions logiques distinctes sur une ressource.
- Règle n° 15 : La mutation des relations est vraiment compliquée et ne se résume pas facilement à une règle rapide.
- Règle n° 16 : lorsque vous écrivez des mutations distinctes pour des relations, demandez-vous s'il serait utile que les mutations opèrent sur plusieurs éléments à la fois.
- Règle n° 17 : Préfixez les noms de mutations par l'objet qu'elles mutent pour un regroupement alphabétique (par exemple, utilisez `orderCancel` au lieu de `cancelOrder`).
- Règle n° 18 : Ne rendez les champs d'entrée obligatoires que s'ils sont réellement nécessaires, d'un point de vue sémantique, à la poursuite de la mutation.
- Règle n°19 : Utilisez des types plus faibles pour les entrées (par exemple String au lieu de Email) lorsque le format est sans ambiguïté et que la validation côté client est complexe. Cela permet au serveur d'exécuter toutes les validations non triviales en une seule fois et de renvoyer les erreurs à un seul endroit et dans un seul format, ce qui simplifie le client.
- Règle n° 20 : utilisez des types plus forts pour les entrées (par exemple, DateTime au lieu de String) lorsque le format peut être ambigu et que la validation côté client est simple. Cela apporte de la clarté et encourage les clients à utiliser des contrôles de saisie plus stricts (par exemple, un widget de sélection de date au lieu d'un champ de texte libre).
- Règle n° 21 : Structurez les entrées de mutation de manière à réduire les doublons, même si cela exige de relâcher les contraintes de rigueur sur certains champs.
- Règle n° 22 : les mutations doivent fournir des erreurs au niveau de l'utilisateur ou de l'entreprise via un champ userErrors dans les données utiles de la mutation. L'entrée d'erreurs de requête de niveau supérieur est réservée aux erreurs de niveau client et serveur.
- Règle n° 23 : La plupart des champs de données utiles d'une mutation doivent être annulables, à moins qu'il n'y ait vraiment une valeur à renvoyer dans tous les cas d'erreur possibles.

## Conclusion

Merci d'avoir lu notre tutoriel ! Nous espérons qu'à ce stade, vous avez une idée solide de la façon de concevoir une bonne API GraphQL.

Une fois que vous avez conçu une API dont vous êtes satisfait, il est temps de la mettre en œuvre !
