# 教程: GraphQL API 的准则

本教程最初由 [Shopify](https://www.shopify.ca/) 创建，仅供内部使用。但考虑到它对于任何
准备基于 GraphQL API 开发的人有帮助因此我们创建了这个公开版本。

基于最近三年来 Shopify 在生产环境中开发和迭代 GraphQL API 所获得的经验和教训，并将持续
更新此文档，并且将会不断修正，不会一成不变。

我们认为这些设计准则在绝大多数情况下都是适用的。但它们可能未必完全适用于您的需求，即使在 Shopify 内部
这些规则也未必 100% 的适用于所有场景。因此请不要盲目的照搬和实施下述的所有设计准则。

目录
=================
- [教程: GraphQL API 的准则](#教程-graphql-api-的准则)
- [目录](#目录)
  - [简介](#简介)
  - [步骤0: 假设背景](#步骤0-假设背景)
  - [步骤1: 全局概览](#步骤1-全局概览)
  - [步骤2: 少既是多](#步骤2-少既是多)
    - [避免暴露 `CollectionMembership` 表](#避免暴露-collectionmembership-表)
    - [重新思考合集](#重新思考合集)
    - [结论](#结论)
  - [步骤3: 增加细节](#步骤3-增加细节)
    - [最初实现](#最初实现)
    - [ID 和 `Node` 接口](#id-和-node-接口)
    - [`Rules` 字段和子对象](#rules-字段和子对象)
    - [列表和分页](#列表和分页)
    - [字符串](#字符串)
    - [关联对象的 ID](#关联对象的-id)
    - [命名与标量（Scalar）](#命名与标量scalar)
    - [分页的再思考](#分页的再思考)
    - [枚举（Enum）](#枚举enum)
  - [步骤4: 商业逻辑](#步骤4-商业逻辑)
  - [步骤5: 变更（Mutation）](#步骤5-变更mutation)
    - [根据具体业务来划分支持的操作种类](#根据具体业务来划分支持的操作种类)
    - [思考对象与对象间的关系](#思考对象与对象间的关系)
    - [Input: 结构 - 第一部分](#input-结构---第一部分)
    - [Input: 标量](#input-标量)
    - [Input: 结构 - 第二部分](#input-结构---第二部分)
    - [Mutation 的返回值](#mutation-的返回值)
  - [TLDR: 设计准则总结](#tldr-设计准则总结)
  - [结尾](#结尾)

## 简介

欢迎您！本文档将引导设计新的 GraphQL API（或现有 GraphQL API 的扩展和迭代）。 API 设计是一项极具挑战性的任务，它需要你对于业务场景拥有深刻地理解并且不断的进行迭代和验证。

## 步骤0: 假设背景

就本教程而言，假设的想象您在一家电子商务公司工作。目前这一电商平台已经拥有一个可以查询商品信息的 GraphQL API。在最近的产品开发迭代中，你的团队刚刚完成了「商品合集」功能的后端开发，你被指派负责开发该功能对应的 API 接口。

商品合集本质上是一个类似于商品收藏夹的功能；例如，可能拥有所有 t-shirts 的合集分组。允许商家对于商品进行分组 —— 从而实现例如在某个专题促销页中仅展示某个特定合集中的商品或「某个特定合集中的商品打折」之类的程序化任务。

你的后端同事已经完成了这一需求的业务逻辑设计，具体如下：
* 所有的合集都包含一些简单属性诸如：标题、详情描述（可能包括 HTML 片段）、缩略图 等基础字段。
* 合集有两种类型，分别是 需要人工添加商品的「手动合集」（ManualCollections） 和 能够按照规则生成的「自动合集」（AutomaticCollections）—— 例如可以创建一个自动合集，该合集会将店铺内的存在 XL 码库存的所有男装都加入进去。
* 商品和合集之间是多对多（many-to-many）关系，在数据库层面存在一个叫做 `CollectionMembership` 的中间关联表。
* 就像商品可以设置上下架一样，合集也有一个字段可以设置其是否生效。

我们将基于上述的假定需求来思考如何进行 API 设计。

## 步骤1: 全局概览

这一功能最为简单粗暴的 GraphQL Schema 设计可能看起来是这样（ 忽略所有已存在的类型，比如 Product ）：

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

尽管只有四个对象和一个接口，但乍看之下已经相当复杂。而且，它似乎并没有完全地满足业务需求，
例如我们需要使用这样的 API 来完成移动端 App 上的合集功能似乎就不太够用了。

让我们先好好的思考一下，用一个由多个对象和数十个字段揉杂在一起所构成的 Graphql API 来实现某一业务需求往往是混乱和错误的开端。你应该首先从更高的抽象层级进行思考，并着眼于类型（GraphQL Type）及各个类型之间的关系，而非数据库层面的具体字段或对 CRUD 接口进行简单罗列。从[实体关系模型（Entity-Relationship model）](https://zh.wikipedia.org/zh-hans/ER%E6%A8%A1%E5%9E%8B)开始思考会是一个好的选择。如果我们像这样收敛和简化一下范围，我们将得到以下结果：

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

为了获得这种简化的表示形式，当前设计中所有不需要关注的细节隐藏起来。移除了所有字段类型，所有字段名称和所有可能为空的信息。剩下的东西看起来仍然有点像 GraphQL，但是可以让您关注更高级别的类型及其关系。

*规则一：永远先从更高的抽象层级进行设计，先考虑类型与类型之间的关系，再去考虑具体的字段。*

## 步骤2: 少既是多

接下来，让我们着力于解决简单粗暴版本中的核心问题。

如前所述，在数据库中本需求由通过 「自动合集表」、「手动合集表」以及用于「实现商品与合集间多对多关系的中间表」这三张表来实现。我们现有的 GraphQL API 设计完全照搬了这一关系模型，但这实际上是错误的。

这一错误的核心问题在于，API 设计和数据库设计有着不同的抽象层级和使用目的。在 API 设计中不加思考的照搬数据库表结构，往往会把我们引入歧途。

### 避免暴露 `CollectionMembership` 表

你可能已经注意到 `CollectionMembership` 表实质上是一种技术细节，对于业务场景而言它实质上应该是一种黑盒实现。
现在再读一遍最后一句话：产品与合集之间的关系；从业务域的维度来看，它们并不需要明确暴露出来它们之间的数据关系。这是一个需要实现的技术细节。

也就是说我们不应该在 API 设计中将它暴露出来，因为我们的 API 是对于业务模型而非技术细节的抽象。因此我们可以进一步将 Schema 重构成下述版本:

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

看起来好多了。

*规则二: 永远不要在 API 中暴露不必要的实现细节。*

### 重新思考合集

现有的 API Schema 设计依然着一个因为我们没有深入理解业务知识而产生的缺陷。 我们把「自动合集」和「手动合集」设计成两种不同的 GraphQL 类型，他们都继承于同一个公共的集合接口。从直觉上讲，这设计似乎是符合逻辑的 —— 他们有许多相同的字段，但本质上它们的功能和行为存在着明显差异（自动合集有规则）。

但从业务场景角度来看，这些差异基本上也是实现细节。它们在业务场景中所实现的功能是完全一致的 —— 将若干个商品聚合在一起，至于怎么样的方式选择产放入合集都是次要的。而且未来，可能会有新的需求导致第三种甚至第四种合集类型的出现，例如可能会新增一种主要由规则生成同时允许手工加入商品的合集类型。但无论需求怎么变更，有一点是不会变的 —— 它们从逻辑上永远是一种用于将若干个商品聚合在一起的合集。因此或许我们可以进一步的这样重构 API ：

```graphql
type Collection {
  [CollectionRule]
  Image
  [Product]
}

type CollectionRule { }
```
这样就看起来简洁多了，也许你会担心对于「手动合集」而言，聚合规则（CollectionRule）字段是不存在的。但实际不用担心对于「手动合集」只要返回一个空数组，那么这一设计完全是符合逻辑和满足需求的。

### 结论

在较高的抽象层级下进行 API 设计，要求你必须对于你所建模的业务场景有着非常深刻的认知。本教程提供的范例很难有你真实工作环境中的需求那么具体和复杂。但是，也足够用来阐述和讲解这样设计的意义。对于你实际工作中业务领域建模，也是一样的原则 —— 不要急于进行细节实现，花费足够时间理解业务场景及其上下文对于你而言至关重要。

与此同时，一个好的 API 设计也不应该是对于某个 UI 设计稿的建模 —— 你不应该在设计 API 时只考虑设计稿中要求体现哪些字段。即便 UI 设计稿和数据库表结构对于 API 设计而言非常有参考价值，但你务必记得你的核心关注点应该是更为抽象的业务领域场景。

更重要的是，务必不要照搬现有的 REST API设计（如果有的话）。 REST 和 GraphQL 背后是不同的思考逻辑，你在 REST API 领域的设计经验对于 GraphQL API 而言未必是能够适用的。 

既往不恋，纵情向前，当下不杂，未来不迎。

*规则三: 围绕着业务领域背景重新思考你的 GraphQL API，切忌直接照搬数据库表结构、视觉稿或已有遗留的 API。*

## 步骤3: 增加细节

现在我们已经有了一个大体合适的抽象模型，我们可以逐步开始考虑细节 —— 把隐去的具体字段加回来。

在我们开始增加每一个字段时，我们都需要仔细思考这个字段是否真的有存在的必要。我们的 GraphQL 类型中存在某个字段是因为我们的业务场景需要用到，而不应该是因为这个字段在数据库中存在或在过去的 REST API 中存在。

在 GraphQL 中暴露一个字段、参数或类型，非常简单。但一旦发布上线后，你想要将之和改名将变得异常苦难，GraphQL 的灵活意味着你很难预测哪些地方会用到它们。

*规则四：永远记得在 GraphQL 中去掉一个字段要比新增一个字段困难的多。*

### 最初实现

如果不逐一考虑每个字段存在的必要性而是先全部加回来的话，那么当前的 GraphQL Schema 大概是这样的:

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

接下来，我们来从上到下的顺序具体思考每一个字段是否有其存在的必要。并且逐步解决许多新的问题。

### ID 和 `Node` 接口
在我们的合集类型中，第一个字段是 ID 字段。非常合理，而且确实是必须的——我们在做增删该查时都会用的到这一字段。
不过，在 GraphQL API 中往往会存在一个`Node` 接口，它的具体结构如下：

```graphql
interface Node {
  id: ID!
}
```
在 GraphQL 中，它用来告示客户端基于其实现的对象是可以基于唯一 ID 进行持久保存和检索，这有助与客户端更高效的实现本地缓存和其他功能。

>  译者注：`Node` 接口实际上 Relay 规范的一部分，可以在 [GraphQL Sever Specification](https://relay.dev/docs/en/graphql-server-specification.html) 中了解具体信息。

因此我们的业务对象应该继承于 `Node`接口:

```graphql
type Collection implements Node {
  id: ID!
}
```

*规则五： 绝大多数业务对象都应该集成自 `Node` 接口。*

### `Rules` 字段和子对象

接下来我们开始审视 `rules` 和 `rulesApplyDisjunctively` 这两个字段。

第一个字段 `rules` 非常直观，返回一个包含自动匹配规则的列表。不过请注意，这一字段并标记为了非空字段——这是一个不错的设计。在 GraphQL 中 `null`、`[]` 和 `[null]`是不同的，因此这一设计有助于我们确保当合集的类型是手动合集时 `rules` 的值可以为 `[]`，但不能为 `null` 或者 `[null]`。

*小贴士: 一个空数组和`null`在逻辑上有着不同的语义。*

第二个字段 `rulesApplyDisjunctively` 返回的是布尔值——用来表示`rules`数组中的多条规则之间是取交集还是并集（or 还是 and）。但对于不适用于规则的手动合集而言，这一字段的值应该是什么似乎是个问题。

对此而言，引入一种的 GraphQL 类型——`CollectionRuleSet`可能是个好的选择。当你在同一GraphQL类型中有一组值和行为密切相关的字段时，将他们单独抽出来作为一个新的类型往往是很有必要的，因为这可以避免语义上出现歧义。对于我们现在关注的这一场景而言，采用如何下的设计可以确保允许 `CollectionRuleSet` 为空并同时保证布尔类型的 `appliesDisjunctively` 字段的非空性：

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

*小贴士: 确保布尔类型的字段永远是非空的。如果你想要一种允许为空的布尔类型，请确保该字段的（空/真/假）三种状态在语义上存在区别的并请务必三思*

*规则六：将同一类型中互相密切相关的几个字段单独抽出来作为子对象。*

### 列表和分页

接下来我们开始关注 `products` 字段，这个字段看起来是合理的——毕竟我们在移除`CollectionMembership`以及梳理它的关联逻辑，但实际这里还是存在着一些问题。

从语义上来说，该字段返回的是一个包含集合中所有商品的数组——一个集合中可能包含着成千上万款商品。直接罗列全部商品显然是非常低效和代价高昂地，因此我们非常有必要考虑分页问题。

请记得在设计 GraphQL API 时，一旦遇到数组数字就记得考虑下其是否存在分页的必要。尤其是，记得确认下在真实的业务场景中，这一数字通常会包含多少个元素。

在 GraphQL API 中实现分页功能存在着许多不同的设计方案。本教程中使用的是由  [Relay Connection spec](https://facebook.github.io/relay/graphql/connections.htm) 所提出的 [Connections](https://graphql.org/learn/pagination/#complete-connection-model) 方案。

对我们的业务场景来说，将商品字段重新定义为 `products: ProductConnection!` 即可实现分页——假设你已经实现了 Relay 规范中所定义的 Connections 的话：

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


*规则七：始终记得检查数组字段是否有必要支持分页。*

###  字符串

接下来我们开始关注 `title`字段，它的设计完全没有问题。考虑到业务上要求所有合集都有1个名字，所以它被标记为非空是合理的。

*小提示: 在 GraphQL 中 `""` 和 `null` 是不同的——就像前述的 `[]` 、`[null]` 和 `null`一样。因为从逻辑上来说，它们有着不同的语义：`”“`意味着这个字段是存在的，只是恰好值是空白的，而`null`则意味着这个字段对于当前实例来说是不适用的。*

### 关联对象的 ID

接着是 `imageId` 字段，这个字段是一个用来说明如果你完全把 REST API 的设计照搬到 GraphQL 中会发生什么的典型例子。在 REST API中返回其他对象的 ID 是非常常见的行为，但在 GraphQL 中这是一个反模式。因为仅提供其他对象的 ID，意味着我们需要再发起一条新的 GraphQL 查询来查找它们。在 GraphQL 中服务器会仅返回查询中显式列出的字段而不是一股脑的返回该对象所包含的所有字段，因此不必像 REST API 那样担心返回的冗余字段过多。

作为普遍性规则，我们通常在 GraphQL Schema 设计中尽可能地返回它的关联对象而不仅是关联对象的 ID ：

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

*规则八: 尽可能直接返回关联对象本身而不是仅仅返回关联对象的ID。*

### 命名与标量（Scalar）

最后，我们来关注下 `Collection` 类型中的 `bodyHtml` 字段。对于一个不熟悉技术细节的用户而言，这个字段名可能会存在歧义，因此将之改名为 `description` 会是更为合理的选择。

*规则九： 给字段起名时尽可能体现其在业务上的语义，而不是简单照搬数据库中的字段名。*

另外，我们其实应该思考下 `String` 对于这个字段而言是否是合理的类型。GraphQL 内置了一些标量类型（例如： `String`、`Int`、`Boolean` 等等），但与此同时它也允许你定义更多的标量类型。基于我们的场景自定义一个叫做 `HTML`的标量类型，来说明这个字段的值应该是有效的 HTML 片段，要比直接把它视为一个叫做 `bodyHtml` 的字符串字段更为符合语义。

不过每当你想要添加一个标量字段时，请记得先检查下现有的自定义标量字段中是否已经存在符合语义的选择。如果计划增加一个新的标量类型，最好确保和团队中其他成员达成共识。

*规则十：使用自定义的标量类型，有助于更好地说明字段隐含的上下文。*

### 分页的再思考

审视完 `Collection` 之后，我们接下来思考下 `CollectionRuleSet` 类型作为一个数组是否有进行分页的必要。

虽然在大多数场景下分页是必要的，但就 `CollectionRuleSet` 而言分页却会是一个极低性价比的选择。因为通常来说一个集合中往往之后包含极少数的规则，甚至对于手动集合来说这直接会上一个空数组。

### 枚举（Enum）

`CollectionRule` 类型有着 `column`、`relation`、`condition` 三个字段，分别表示要匹配的属性（值如商品表的 `title`字段）、操作符（例如可能是 `=` 或者 `start_with`）以及期望的值。

考虑到 `column` 是一个数据库术语，因此我们先把这个字段重新命名为 GraphQL 术语—— `field` 会比较符合语义。

接下来，由于 `fields` 和 `relation` 这两个字段的值完全是能够枚举的，我们可以在 GraphQL 中用枚举类型实现它们来确保 API 拥有更高的健壮性：

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

*规则十一：对于值可以被穷举的字段，尽可能使用枚举类型。*

## 步骤4: 商业逻辑

我们现在已经有了一个基本可用且设计良好的 GraphQL Schema。即便像如何处理商品排序之类的细节问题还没有处理，但完全可以照着前文列出的规则来逐一实现。接下来，我们需要做的是审视现有的 API 设计是否完全满足了产品需求。

在这一步骤中最为简单的思考方式或许是——想象下假设你同时需要使用这些 API 来开发客户端，那么对于实现客户端中的功能而言目前所能提供的 API 是否能够满足需求。

不过这种思考范式存在着一些问题——对于大规模应用或者对外提供公开 API 的应用来说，你很难想象客户端中的哪些功能会用到这一接口。一个 GraphQL Type 应该是为很多种不同客户端的很多种不同业务场景提供服务的，你必须同时兼顾到其通用性。因为通用性不足导致的接口冗余意味着额外的工作量和更多的出错概率。

*规则十二：GraphQL API提供的应该是业务逻辑而非数据。尽可能把业务逻辑放在 API 中实现，而非任由各个客户端自行实现。*

例如对于我们的例子而言，很有必要实现如下的方法：
```graphql
type Collection implements Node {
  # ...
  hasProduct(id: ID!): Boolean!
}
```

其用于帮助判断某一商品是否在合集中存在。不同于 Rest API，在 GraphQL 中某个类型存在大量的辅助字段（或方法）不会带来任何的额外开销。

不过也请注意，即便我们可能尽可能多地提供辅助字段但我们不可能穷举出所有的使用场景。因此提供辅助字段的同时，请千万不要将原始数据隐藏起来。
> 译者注： 也就是说假设 User 类型存在 `lastName` 和 `firstName`，额外提供1个 `fullName` 字段很有帮助，但这不意味着你可以只提供 `fullName` 字段。

*规则十三: 记得同时提供原始字段与业务相关的计算字段。*

最后，如果在基于业务逻辑设计 GraphQL API 时发现现有的数据库结构或底层实现难以满足，那么不要因此对业务模型有所妥协。一切都应该为业务模型服务的，你可能要做的是改动你的数据库结构或技术实现。

## 步骤5: 变更（Mutation）

我们现有的 GraphQL Schema 只满足了查询需求，接下来我们需要来设计一些 Mutation 以满足对于合集进行创建、修改和删除的需求。

如同我们在一开始进行 Query 设计时一样，我们先从更高的抽象层级开始思考——这意味着我们先去思考我们需要实现哪些种类的变更，而先不考虑每种 Mutation 的具体实现细节。

最为「大力出奇迹」的思路是——照着 CRUD 的范式我们给所有需要的类型都创建 `增`、`删`、`改` 三种形式的变更。在 REST API 中这是不错的选择，但对于 GraphQL API 来说这样做是远远不够用的。

### 根据具体业务来划分支持的操作种类

你可能会注意到如果仅按照 CRUD 范式来做的话，用于实现更新操作的 Mutation 会变得极为臃肿且承担了太多职能：
- 更新合集的名称或描述
- 设置合集生效与否（publish/unpublish）
- 修改规则
- 添加、移除、重新排序商品

等等。 这样的设计对于服务器端和客户端来说显然都是一种麻烦的困扰。因此将 GraphQL Mutation 拆分成更细粒度会是个好主意。 比方说，我们可以想把 `publish`和` unpublish` 作为两种不同的操作给拆分出来：

- create
- delete
- update
- publish
- unpublish

*规则十四： 根据真实的业务需要思考 GraphQL 类型支持哪些种类的操作*

### 思考对象与对象间的关系

在我们拆出 `publish` & `unpublish` 之后，`update` Mutation 仍然显得非常臃肿因此有必要做进一步的拆分。，我们可以从对象与对象间的关系作为思考的切入点。
具体到商品与合集直接的关系，我们可以作出如下结论：
1. 按照 CRUD 范式，当我们需要改变合集中所保护的商品就需要提供一个新的商品数组（形如 `products: [ProductInput!]!`），但假设某一集合中包含了非常多种商品，显然这样做会存在不小的性能问题。
2. 为了实现增量更新，在 `update` Mutation 中提供诸如  `productsToAdd: [ID!]!` 、`productsToReorder: [ID!]!` 和 `productsToRemove: [ID!]!` 的字段会是一个好的选择。
3. 但与其将之作为 `update` 的3个 input 字段，直接将它们拆分成3个新的 Mutation 显然会来的更为简单直观。

不过当我们将上述原则用于其他的场景时，请记得考虑如下问题而不是教条地照搬：

- 这个数组是否需要分页？对于小数组而言，提供单独的增量更新很可能是一种过度设计。
- 数组中包含的元素是否拥有自己的 ID？例如在 合集和合集规则的关系中，`CollectionRule` 类型并没有 `id` 字段。因此它并不应该被抽离成单独的 Mutation。

*规则十五： 设计 Mutation 很复杂，不能教条式地进行照搬。*

在完成上述改动之后，现在我们拥有这些 Mutation:

- create
- delete
- update
- publish
- unpublish
- addProducts
- removeProducts
- reorderProducts

你可能已经注意到了和商品相关的那3个 Mutation 都是复数，因为如果我们能够提供在统一合集内同时增加或删除多个商品的方法，显然能满足更多地客户端实际使用场景——除非 PRD 中明确禁止我们这么做。

*规则十六：尽可能让 Mutation 支持批量操作。*

### Input: 结构 - 第一部分

在明确了我们需要实现哪些 Mutation 之后，我们还需要明确下每个 Mutation 要支持的入参（Input）有哪些。如果你看过其他人写的 GraphQL Schema，你可能会注意到很多 Mutation 统称都只定义了一个名叫 input 的入参并单独为之定义了一个全局唯一的 Input 类型来描述它实际所需要的入参。**实际上这是传统的 Relay 客户端（Relay Classic API）所要求遵循的规范，我们不建议你在新代码中继续遵循这种约定。**

对于一些简单的 Mutation 而言提供 ID 或 ID 数组就完全足够了，比方说`delete`、`publish` 和 `removeProducts` 等等.

我们只需要重点考虑下述 3 个 Mutation 的 Input 要如何设计：
- create
- update
- reorderProducts

首先是 `create`。仍旧是先写一个「大力出奇迹」的版本出来：
```graphql
type Mutation {
  collectionDelete(collectionId: ID!)
  collectionPublish(collectionId: ID!)
  collectionUnpublish(collectionId: ID!)
  collectionAddProducts(collectionId: ID!, productIds: [ID!]!)
  collectionRemoveProducts(collectionId: ID!, productIds: [ID!])
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

你可能注意到这些 Mutation 的名字都是以形如 `类型-行为`(例如 `collectionDelete`) 而非 `行为-对象` (例如 deleteCollection) 的形式命名，即便后者更符合英文语法。

这是因为 GraphQL 目前没有提供对于 Mutation 进行分组的办法，因此将类型前置有助于我们在 Schema 中更显眼的看到某一类型支持哪些种类的 Mutation。

*规则十七： 使用形如 `orderCancel` 而不是 `cancelOrder`的命名风格来个 Mutation 命名 *

### Input: 标量

在现有的方案中 `description` 字段存在这几个问题，首先我们有必要将其设为允许为空。即便是同一类型其被作为 Mutation 的 Input 时和在 Query 中调用时，其是否必填可能是不一致的。需要根据实际情况仔细审视。
*规则十八： 仅将真的必填的字段在 Input 中设为必填*

我们还需要根据实际情况审视`descripition`的数据类型，例如我们很可能允许用户在输入时仅提供 String 而不是 HTML 片段，而由服务器端在将之保存到数据库前从字符串格式化为 HTML 片段。因此除非你希望这一 字符串到HTML片段的格式化逻辑在客户端实现，否则 `String` 会是个更好的选择。

*规则十九：当 Input 类型比较复杂导致客户端进行验证过于复杂时，可以将之弱化成更通用的类型以便于由服务器进行验证。例如用 `string` 标量 替代 `email` 标量，然后在服务器端进行验证并将所有的错误提示一次性返回给客户端。*

但需要注意的是，这并不是要求你在所有场景下都对输入值使用更弱的类型。例如在本教程的例子中 `field` 和 `relation` 的值显然必须依然是枚举类型。此外诸如 DateTime 之类的类型显然也不应该被弱化成字符串。关键的区别因素的客户端进行强类型验证的成本以及格式本身的模糊性。

*规则二十：当输入的格式可能有歧义而且客户端验证并不困难的时候，应该有限考虑使用强类型（例如 `DateTime` 优于 `String`）*

### Input: 结构 - 第二部分

接下来我们把关注点放在 `update` Mutation 上：

```graphql
type Mutation {
  # ...
  collectionCreate(title: String!, ruleSet: CollectionRuleSetInput, image: ImageInput, description: String)
  collectionUpdate(collectionId: ID!, title: String, ruleSet: CollectionRuleSetInput, image: ImageInput, description: String)
}
```

你会注意到，这与我们的 `create` Mutation 非常相似，但有两个不同之处：增加了一个 `collectionId` 参数，它决定了要更新哪个合集同时`title`不再是必需的，因为`title`可能不需要被更新。即便不算上允许为空的 `title`，在 `create` 和 `update` 中有四个 Input 参数是重复的。

考虑到「DRY（不要重复你自己）」原则，我们觉得在 `create` 和 `update` Mutation 共享一部分逻辑会是更好的选择：

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

尽管这样一来 `create` 时的 `title` 也变成了允许为空，但如果需要的话我们完全可以在 `create` 的 resolver 中再行验证。

*规则二十一： 结构化 Mutation 的 Input 以减少重复，即使是以在类型层面上放宽对于某些字段的要求性约束为代价。*

### Mutation 的返回值

我们需要处理的最后一个设计问题是 Mutation 的返回值。通常情况下，Mutation 是可能成功但也可能报错，虽然GraphQL 确实包含了对查询层面错误的明确支持，但这些错误对于业务层面的 Mutation 报错来说并不够用。因此每个 Mutation 都应该定义一个包含有用户错误字段的 "payload " 类型。对于`create` 来说，它可能是这样的：

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

对于执行成功的 Mutation，它将返回一个空的 `UserErros` 数组，和一个包含了新加入商品的 `collection` 数组。而执行失败的 Mutation 则恰恰与之相反。


*规则二十二： Mutation的中应该包含一个标识业务层面错误的数组。*

对与 `update` 来说也是相似的:

```graphql
type CollectionUpdatePayload {
  userErrors: [UserError!]!
  collection: Collection
}
```
值得注意的是，即使是在`update`里 Payload 的 `collection`仍然是可空的，因为如果提供 input 中提供的 ID数组 不代表一个有效的集合，就没有要返回的集合。

*规则二十三：大多数的 Payload .字段都应该是可以为空的，除非确保其在错误的情况下也有返回值。*

## TLDR: 设计准则总结

- 规则一：永远先从更高的抽象层级进行设计，先考虑类型与类型之间的关系，再去考虑具体的字段。
- 规则二：永远不要在 API 中暴露不必要的实现细节。
- 规则三：围绕着业务背景重新思考你的 GraphQL API，切忌直接照搬数据库表结构、视觉稿或已有的 REST API。
- 规则四：永远记得在 GraphQL 中去掉一个字段要比新增一个字段困难的多。
- 规则五：绝大多数业务对象都应该集成自 `Node` 接口。
- 规则六：将同一类型中互相密切相关的几个字段单独抽出来作为子对象。
- 规则七：始终记得检查数组字段是否有必要支持分页。
- 规则八：尽可能直接返回关联对象本身而不是仅仅返回关联对象的ID。
- 规则九：给字段起名时尽可能体现其在业务上的语义，而不是简单照搬数据库中的字段名。
- 规则十：使用自定义的标量类型，有助于更好地说明字段隐含的上下文。
- 规则十一：对于值可以被穷举的字段，尽可能使用枚举类型。
- 规则十二：GraphQL API提供的应该是业务逻辑而非数据。尽可能把业务逻辑放在 API 中实现，而非任由各个客户端自行实现。
- 规则十三：记得同时提供原始字段与业务相关的计算字段。
- 规则十四： 根据真实的业务需要思考 GraphQL 类型支持哪些种类的操作。
- 规则十五： 设计 Mutation 很复杂，不能教条式地进行照搬。
- 规则十六：尽可能让 Mutation 支持批量操作。
- 规则十七： 使用形如 `orderCancel` 而不是 `cancelOrder`的命名风格来个 Mutation 命名。
- 规则十八： 仅将真的必填的字段在 Input 中设为必填。
- 规则十九：当 Input 类型比较复杂导致客户端进行验证过于复杂时，可以将之弱化成更通用的类型以便于由服务器进行验证。例如用 `string` 标量 替代 `email` 标量，然后在服务器端进行验证并将所有的错误提示一次性返回给客户端。
- 规则二十：当输入的格式可能有歧义而且客户端验证并不困难的时候，应该有限考虑使用强类型（例如 `DateTime` 优于 `string`）。
- 规则二十一： 结构化 Mutation 的 Input 以减少重复，既是是以在类型层面上放宽对于某些字段的要求性约束为代价。
- 规则二十二： Mutation的中应该包含一个标识业务层面错误的数组。
- 规则二十三：大多数的 Payload .字段都应该是可以为空的，除非确保其在错误的情况下也有返回值。

## 结尾
感谢你阅读我们的教程。 希望它有助于你设计一个好的 GraphQL API。
