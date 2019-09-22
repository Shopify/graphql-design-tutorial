# Tutorial: Designing a GraphQL API

本チュートリアルは[Shopify](https://www.shopify.ca/)によって社内向けに作られました。
我々は本チュートリアルがGraphQL APIを利用する全ての方にとって役に立つと考え、公開版を作成するに至りました。

本チュートリアルは、Shopifyのプロダクション環境における過去３年間のスキーマ構築と発展から得た学びに基づいています。
本チュートリアルはこれまでも発展してきましたし、今後も更新され続けるでしょう。
ですから、本チュートリアルに盲目的に従ってすべてを取り込もうとせず、あなたの目的や状況に応じて役に立つ部分を適用してください。

## Intro

ようこそ！本ドキュメントでは、新しいGraphQL APIの（あるいは既存のGraphQL APIの拡張の）設計方法をみていきます。
APIの設計は、繰り返し、実験し、ビジネスドメインの十分な理解を得るに値する挑戦的な仕事です。

## Step Zero: Background

本チュートリアルでは、あるE-コマース運営企業の開発業務を想定します。
あなたはすでに商品情報を公開するための非常に小規模なGraphQL APIをもっています。
あなたのチームは、ちょうどコレクション機能のバックエンド部分の実装を終え、本機能をAPIで公開したいと考えています。

コレクションは新しい主要機能で商品をグループにまとめることができます。
例えばあなたの所有するすべてのTシャツ商品をまとめたコレクションが考えられます。
コレクションはWebサイトにおいて商品をまとめて表示する際に利用できますし、自動化タスクにも利用できます（特定のコレクションの商品に対してのみ、ある割引価格と適用したい場合など。）

バックエンドではコレクション機能は以下のように実装されました。
- すべてのコレクションは、タイトルや説明文（HTMLを含むかもしれません）、画像などのいくつかの属性をもちます。
- コレクションは２種類に分けられます。"手動"コレクションは自分で商品を登録し、"自動"コレクションはいくつかのルールを設定することで自動的に商品が登録されます。
- 商品とコレクションは、中間テーブル `CollectionMembership` を通して他対他のリレーションを持ちます。
- コレクションと商品は公開（店頭に並んでいる）と非公開のいずれかの状態にあります。

以上の背景をふまえ、APIデザインを考えていきます。

## Step One: A Bird's-Eye View

素朴にスキーマを定義すると以下のようになります（`Product` など既存の型は省略します。）
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
  rulesApplyDisjunctively: Bool!
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

たった４つのオブジェクトとインターフェースだけにも関わらず、一見しただけでもすでに複雑だということが分かります。
また、このAPIを用いて今後開発する機能（たとえばモバイルアプリ向けのコレクション機能）はまだ一切実装されていません。

一歩下がって考えてみましょう。GraphQLの複雑性は、多数のオブジェクトを含み、それらが複数の関連を介してつながり、いくつものフィールドをもつことに起因するでしょう。
このような設計を一度に済ませようとすると、混乱や誤った設計を招きます。
そうではなく、高いレベルから考えることからはじめ、フィールドの詳細やミューテーションは考えずに、まずは型とそれらの関連性に注目しましょう。

GraphQL特有の表現が少し入りますが、基本的には[エンティティ・リレーションシップ図](https://en.wikipedia.org/wiki/Entity%E2%80%93relationship_model)を考えてください。
素朴なスキーマ定義を凝縮すると次のようになるでしょう。

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

このシンプルな表現ではすべての値フィールド、フィールド名、Null制約の情報を取り除いてあります。
残ったものはGraphQLのように見える何かに過ぎませんが、型や関連性といった高いレベルで考えさせてくれます。

*ルール #1: 詳細に取り掛かる前に、高いレベルでオブジェクトとそれらの関連性を考えることからはじめよ。*

## Step Two: A Clean Slate

それではこのシンプルな構造を用いて、当初の設計における主要な誤りをみていきましょう。

以前に述べたとおり、我々の詳細実装は、手動コレクションと自動コレクション、そしてコレクションと商品を結合するコレクターテーブルを定義しています。
素朴なAPIデザインは明らかに詳細実装に依存して構築されていますが、これは誤りです。

このアプローチの根本的な問題は、APIは詳細実装とは異なる目的の振る舞いを持ち、また多くの場合は抽象度のレベルも異なることです。
今回のケースでは、我々の詳細実装によって多くの迷いが生じました。

### Representing `CollectionMembership`s

現状のスキーマに `CollectionMembership` が含まれていることに気がついたかもしれません。
コレクションメンバーシップテーブルは商品とコレクションの間の他対他の関連を表現するために使われます。
最後の部分をもう一度読んでください、「*商品とコレクションの間*」の関連とあります。
ビジネスドメインのセマンティクスからすれば、コレクションメンバーシップは意味を持っていないのです。
コレクションメンバーシップは詳細実装です。

これはつまり、我々のAPIに含まれないということを意味します。
その代わりに、商品に関する実際のビジネスドメインの関係性を直接的に表現すべきです。
コレクションメンバーシップを取り除くとすれば、我々のスキーマ設計は以下のようになります。

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

かなり良くなりました。

*ルール #2: 詳細実装をAPI設計において表現してはならない。*

### Representing Collections

ビジネスドメインの十分な理解がないと気づかないかもしれませんが、まだ今のAPI設計には大きな誤りがひとつ残されています。

現状の設計では、手動コレクションと自動コレクションは共通のコレクションインターフェースを実装した異なる２つの型として定義されています。
直感的には理にかなっているようにみえます。
それらは多数の共通するフィールドを持っていますが、関連性（自動コレクションは選択ルールをもちます）と振る舞いにおいて明確な違いがあります。

ビジネスモデルの観点からすれば、これらの差異も基本的には詳細実装です。
コレクションの振る舞いとは、商品をグループにまとめることであり、どのようにしてそれら商品を選ぶかは従属的なものだからです。
我々はいずれは第３の商品選択方法（機械学習？）を許容するかもしれませんし、複数の選択方法を組み合わせるかもしれません（選択ルールもあれば手動で選択するものもある。）
それでも、*それらはコレクションにすぎないでしょう。*

現時点において選択方法の組み合わせを見込んで許容しないのは、詳細実装の落ち度だと主張することもできます。
つまり、APIデザインは次のようにすべきだと主張することもできるということです。

```graphql
type Collection {
  [CollectionRule]
  Image
  [Product]
}

type CollectionRule { }
```

それは良さそうにみえます。
手動コレクションがルールを持っているように振る舞うことを懸念されるかもしれません。
しかし、思い出してください、これは関連のリストなのです。
我々のあたらしいAPI設計では、手動コレクションは単に選択ルールが無い（空）コレクションにすぎません。

### Conclusion

この抽象レベルにおけて最適なAPI設計を行うには、モデリング対象としているビジネスドメインへの深い理解が要求されます。
ュートリアルの設定だけで特定の話題に関するコンテキストの深さをお伝えするのは困難ですが、本節のコレクション設計の例を通して、APIの設計方針を説明できていれば幸いです。

実際にモデリングしている対象が何であれコレクション設計の理解は必ず必要となります。
API設計を行っている際には、どうあるべきかを自分自身に問い続ること、詳細実装に盲目的に従わないことが非常に重要です。

関連する点として、良いAPI設計はユーザインターフェースにも従属しません。
詳細実装とUIのいずれも着想を得るためのインプットとして用いることができるものの、設計方針は常にビジネスドメインに従わなければなりません。

さらに重要な点として、既存のREST APIにおける決定事項を不必要に踏襲しないことです。
REST APIとGraphQLの背後にある設計思想はまったく異なった決定を導く可能性があります。
ですから、REST APIでうまくいったことがGraphQLでもそうであると想定してはいけません。

可能なかぎり、これまでの常識を取り外し、ゼロから考えるようにしましょう。

*ルール #3: 詳細実装でも、ユーザインタフェースでも、あるいはレガシーなAPIでもなく、ビジネスドメインに従ってAPI設計を行うこと。*

## Step Three: Adding Detail

我々の型をモデリングするクリーンな構造を手に入れることができました。  
もう一度、細部に視点を移して省いていたフィールドを追加していきましょう。

詳細情報を追加する前に、現時点においてその情報が本当に必要かどうか自問自答してください。
なぜなら、データベースのカラムやモデルのプロパティ、RESTの属性としてその詳細情報が存在していたとしても、それがそのままGraphQLでも必要になるとは限らないからです。

スキーマの要素（フィールドや引数、型など）を公開する場合、実際の必要性やユースケースによって検討されるべきです。  
GraphQLにおいて要素を追加するという操作は簡単ですが、それを変更したり削除することは破壊的な変更であり、難しい操作です。

*ルール #4: フィールドの追加操作は削除操作よりも簡単。*

### Starting point

Restoring our naive fields adjusted for our new structure, we get:

```graphql
type Collection {
  id: ID!
  rules: [CollectionRule!]!
  rulesApplyDisjunctively: Bool!
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

Now we have a whole new host of design problems to resolve. We'll work through
the fields in order top to bottom, fixing things as we go.

### IDs and the `Node` Interface

The very first field in our Collection type is an ID field, which is fine and
normal; this ID is what we'll need to use to identify our collections throughout
the API, in particular when performing actions like modifying or deleting them.
However there is one piece missing from this part of our design: the `Node`
interface. This is a very commonly-used interface that already exists in most
schemas and looks like this:
```graphql
interface Node {
  id: ID!
}
```
It hints to the client that this object is persisted and retrievable by the
given ID, which allows the client to accurately and efficiently manage local
caches and other tricks. Most of your major identifiable business objects
(e.g. products, collections, etc) should implement `Node`.

The beginning of our design now just looks like:
```graphql
type Collection implements Node {
  id: ID!
}
```

*Rule #5: Major business-object types should always implement `Node`.*

### Rules and Subobjects

We will consider the next two fields in our Collection type together: `rules`,
and `rulesApplyDisjunctively`. The first is pretty straightforward: a list of
rules. Do note that both the list itself and the elements of the list are marked
as non-null: this is fine, as GraphQL does distinguish between `null` and `[]`
and `[null]`. For manual collections, this list can be empty, but it cannot be
null nor can it contain a null.

*Protip: List-type fields are almost always non-null lists with non-null
elements. If you want a nullable list make sure there is real semantic value in
being able to distinguish between an empty list and a null one.*

The second field is a bit weird: it is a boolean field indicating whether the
rules apply disjunctively or not. It is also non-null, but here we run into a
problem: what value should this field take for manual collections? Making it
either false or true feels misleading, but making the field nullable then makes
it a kind of weird tri-state flag which is also awkward when dealing with
automatic collections. While we're puzzling over this, there is one other thing
that is worth mentioning: these two fields are obviously and intricately related.
This is true semantically, and it's also hinted by the fact that we chose names
with a shared prefix. Is there a way to indicate this relationship in the schema
somehow?

As a matter of fact, we can solve all of these problems in one fell swoop by
deviating even further from our underlying implementation and introducing a new
GraphQL type with no direct model equivalent: `CollectionRuleSet`. This is often
warranted when you have a set of closely-related fields whose values and
behaviour are linked. By grouping the two fields into their own type at the API
level we provide a clear semantic indicator and also solve all of our problems
around nullability: for manual collections, it is the rule-set itself which is
null. The boolean field can remain non-null. This leads us to the following
design:

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
  appliesDisjunctively: Bool!
}

type CollectionRule {
  column: String!
  relation: String!
  condition: String!
}
```

*Protip: Like lists, boolean fields are almost always non-null. If you want a
nullable boolean, make sure there is real semantic value in being able to
distinguish between all three states (null/false/true) and that it doesn't
indicate a bigger design flaw.*

*Rule #6: Group closely-related fields together into subobjects.*

### Lists and Pagination

Next on the chopping block is our `products` field. This one might seem safe;
after all we already "fixed" this relation back when we removed our `CollectionMembership`
type, but in fact there's something else wrong here.

The field as currently defined returns an array of products, but collections can
easily have many tens of thousands of products, and trying to gather all of
those into a single array would be incredibly expensive and inefficient. For
situations like this, GraphQL provides lists pagination.

Whenever you implement a field or relation returning multiple objects, always
ask yourself if the field should be paginated or not. How many of this object
can there be? What quantity is considered pathological?

Paginating a field means you need to implement a pagination solution first.
This tutorial uses [Connections](https://graphql.org/learn/pagination/#complete-connection-model)
which is defined by the [Relay Connection spec](https://facebook.github.io/relay/graphql/connections.htm).

In this case, paginating the products field in our design is as simple as
changing its definition to `products: ProductConnection!`. Assuming you have
connections implemented, your types would look like this:

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


*Rule #7: Always check whether list fields should be paginated or not.*

###  Strings

Next up is the `title` field. This one is legitimately fine the way it is. It's
a simple string, and it's marked non-null because all collections must have a
title.

*Protip: As with booleans and lists, it's worth noting that GraphQL does
distinguish between empty strings (`""`) and nulls (`null`), so if you need a
nullable string make sure there is a legitimate semantic difference between
not-present (`null`) and present-but-empty (`""`). You can often think of empty
strings as meaning "applicable, but not populated", and null strings meaning
"not applicable".*

### IDs and Relations

Now we come to the `imageId` field. This field is a classic example of what
happens when you try and apply REST designs to GraphQL. In REST APIs it's
pretty common to include the IDs of other objects in your response as a way to
link together those objects, but this is a major anti-pattern in GraphQL.
Instead of providing an ID, and forcing the client to do another round-trip to
get any information on the object, we should just include the object directly
into the graph - that's what GraphQL is for after all. In REST APIs this pattern
often isn't practical, since it inflates the size of the response significantly
when the included objects are large. However, this works fine in GraphQL because
every field must be explicitly queried or the server won't return it.

As a general rule, the only ID fields in your design should be the IDs of the
object itself. Any time you have some other ID field, it should probably be an
object reference instead. Applying this to our schema so far, we get:

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
  appliesDisjunctively: Bool!
}

type CollectionRule {
  column: String!
  relation: String!
  condition: String!
}
```

*Rule #8: Always use object references instead of ID fields.*

### Naming and Scalars

The last field in our simple `Collection` type is `bodyHtml`. To a user who is
unfamiliar with the way that collections were implemented, it's not entirely
obvious what this field is for; it's the body description of the specific
collection. The first thing we can do to make this API better is just to rename
it to `description`, which is a much clearer name.

*Rule #9: Choose field names based on what makes sense, not based on the
implementation or what the field is called in legacy APIs.*

Next, we can make it non-nullable. As we talked about with the title field, it
doesn't make sense to distinguish between the field being null and simply being
an empty string, so we don't expose that in the API. Even if your database
schema does allow records to have a null value for this column, we can hide that
at the implementation layer.

Finally, we need to consider if `String` is actually the right type for this
field. GraphQL provides a decent set of built-in scalar types (`String`, `Int`,
`Boolean`, etc) but it also lets you define your own, and this is a prime use
case for that feature. Most schemas define their own set of additional scalars
depending on their use cases. These provide additional context and semantic
value for clients. In this case, it probably makes sense to define a custom
`HTML` scalar for use here (and potentially elsewhere) when the string in
question must be valid HTML.

Whenever you're adding a scalar field, it's worth checking your existing list of
custom scalars to see if one of them would be a better fit. If you're adding a
field and you think a new custom scalar would be appropriate, it's worth talking
it over with your team to make sure you're capturing the right concept.

*Rule #10: Use custom scalar types when you're exposing something with specific
semantic value.*

### Pagination Again

That covers all of the fields in our core `Collection` type. The next object is
`CollectionRuleSet`, which is quite simple. The only question here is whether or
not the list of rules should be paginated. In this case the existing array
actually makes sense; paginating the list of rules would be overkill. Most
collections will only have a handful of rules, and there isn't a good use case
for a collection to have a large rule set. Even a dozen rules is probably an
indicator that you need to rethink that collection, or should just be manually
adding products.

### Enums

This brings us to the final type in our schema, `CollectionRule`. Each rule
consists of a column to match on (e.g. product title), a type of relation (e.g.
equality) and an actual value to use (e.g. "Boots") which is confusingly called
`condition`. That last field can be renamed, and so should `column`; column is
very database-specific terminology, and we're working in GraphQL. `field` is
probably a better choice.

As far as types go, both `field` and `relation` are probably implemented
internally as enumerations (assuming your language of choice even has
enumerations). Fortunately GraphQL has enums as well, so we can convert those
two fields to enums. Our completed schema design now looks like this:

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
  appliesDisjunctively: Bool!
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

*Rule #11: Use enums for fields which can only take a specific set of values.*

## Step Four: Business Logic

We now have a minimal but well-designed GraphQL API for collections. There is a
lot of detail to collections that we haven't dealt with - any real
implementation of this feature would need a lot more fields to deal with things
like product sort order, publishing, etc. - but as a rule those fields will all
follow the same design patterns laid out here. However, there are still a few
things which bear looking at in more detail.

For this section, it is most convenient to start with a motivating use case from
the hypothetical client of our API. Let us therefore imagine that the client
developer we have been working with needs to know something very specific:
whether a given product is a member of a collection or not. Of course, this is
something that the client can already answer with our existing API: we expose
the complete set of products in a collection, so the client simply has to
iterate through, looking for the product they care about.

This solution has two problems though. The first, obvious problem is that it's
inefficient; collections can contain millions of products, and having the client
fetch and iterate through them all would be extremely slow. The second, bigger
problem, is that it requires the client to write code. This last point is a
critical piece of design philosophy: the server should always be the single
source of truth for any business logic. An API almost always exists to serve
more than one client, and if each of those clients has to implement the same
logic then you've effectively got code duplication, with all the extra work and
room for error which that entails.

*Rule #12: The API should provide business logic, not just data. Complex
calculations should be done on the server, in one place, not on the client, in
many places.*

Back to our client use-case, the best answer here is to provide a new field
specifically dedicated to solving this problem. Practically, this looks like:
```graphql
type Collection implements Node {
  # ...
  hasProduct(id: ID!): Bool!
}
```
This field takes the ID of a product and returns a boolean based on the server
determining if a product is in the collection or not. The fact that this sort-of
duplicates the data from the existing `products` field is irrelevant. GraphQL
returns only what clients explicitly ask for, so unlike REST it does not cost us
anything to add a bunch of secondary fields. The client doesn't have to write
any code beyond querying an additional field, and the total bandwidth used is a
single ID plus a single boolean.

One follow-up warning though: just because we're providing business logic in a
situation does not mean we don't have to provide the raw data too. Clients
should be able to do the business logic themselves, if they have to. You can’t
predict all of the logic a client is going to want, and there isn't always an
easy channel for clients to ask for additional fields (though you should strive
to ensure such a channel exists as much as possible).

*Rule #13: Provide the raw data too, even when there's business logic around it.*

Finally, don't let business-logic fields affect the overall shape of the API.
The business domain data is still the core model. If you're finding the business
logic doesn't really fit, then that's a sign that maybe your underlying model
isn't right.

## Step Five: Mutations

The final missing piece of our GraphQL schema design is the ability to actually
change values: creating, updating, and deleting collections and related pieces.
As with the readable portion of the schema we should start with a high-level
view: in this case, of just the various mutations we will want to implement,
without worrying about their specific inputs or outputs. Naively we might follow
the CRUD paradigm and have just `create`, `delete`, and `update` mutations.
While this is a decent starting place, it is insufficient for a proper GraphQL
API.

### Separate Logical Actions

The first thing we might notice if we were to stick to just CRUD is that our
`update` mutation quickly becomes massive, responsible not just for updating
simple scalar values like title but also for performing complex actions like
publishing/unpublishing, adding/removing/reordering the products in the
collection, changing the rules for automatic collections, etc. This makes it
hard to implement on the server and hard to reason about for the client.
Instead, we can take advantage of GraphQL to split it apart into more granular,
logical actions. As a very first pass, we can split out publish/unpublish
resulting in the following mutation list:
- create
- delete
- update
- publish
- unpublish

*Rule #14: Write separate mutations for separate logical actions on a resource.*

### Manipulating Relationships

The `update` mutation still has far too many responsibilities so it makes sense
to continue splitting it up, but we will deal with these actions separately
since they're worth thinking about from another dimension as well: the
manipulation of object relationships (e.g. one-to-many, many-to-many). We've
already considered the use of IDs vs embedding, and the use of pagination vs
arrays in the read API, and there are some similar issues to deal with when
mutating these relationships.

For the relationship between products and collections, there are a couple of
styles we could broadly consider:
- Embedding the entire relationship (e.g. `products: [ProductInput!]!`) into the
  update mutation is the CRUD-style default, but of course it quickly becomes
  inefficient when the list is large.
- Embedding "delta" fields (e.g. `productsToAdd: [ID!]!` and
  `productsToRemove: [ID!]!`) into the update mutation is more efficient since
  only the changed IDs need to be specified instead of the entire list, but it
  still keeps the actions tied together.
- Splitting it up entirely into separate mutations (`addProduct`,
  `removeProduct`, etc.) is the most powerful and flexible but also the most
  work.

The last option is generally the safest call, especially since mutations like
this will usually be distinct logical actions anyway. However, there are a lot
of factors to consider:
- Is the relationship large or paginated? If so, embedding the entire list is
  definitely impractical, however either delta fields or separate mutations
  could still work. If the relationship is always small though (especially if
  it's one-to-one), embedding may be the simplest choice.
- Is the relationship ordered? The product-collection relationship is ordered,
  and permits manual reordering. Order is naturally supported by the embedded
  list or by separate mutations (you can add a `reorderProducts` mutation)
  but isn't an option for delta fields.
- Is the relationship mandatory? Products and collections can both exist on
  their own outside of the relationship, with their own create/delete lifecycle.
  If the relationship were mandatory (i.e. products must be in a collection)
  then this would strongly suggest separate mutations because the action would
  actually be to *create* a product, not just to update the relationship.
- Do both sides have IDs? The collection-rule relationship is mandatory (rules
  can't exist without collections) but rules don't even have IDs; they are
  clearly subservient to their collection, and since the relationship is also
  small, embedding the list is actually not a bad choice here. Anything else
  would require rules to be individually identifiable and that feels like
  overkill.

*Rule #15: Mutating relationships is really complicated and not easily
 summarized into a snappy rule.*

 If you stir all of this together, for collections we end up with the following
 list of mutations:
- create
- delete
- update
- publish
- unpublish
- addProducts
- removeProducts
- reorderProducts

Products we split into their own mutations, because the relationship is large
and ordered. Rules we left inline because the relationship is small, and rules
are sufficiently minor to not have IDs.

Finally, you may note our product mutations act on sets of products, for example
`addProducts` and not `addProduct`. This is simply a convenience for the client,
since the common use case when manipulating this relationship will be to add,
remove, or reorder more than one product at a time.

*Rule #16: When writing separate mutations for relationships, consider whether
 it would be useful for the mutations to operate on multiple elements at once.*

### Input: Structure, Part 1

Now that we know which mutations we want to write, we get to figure out what
their input structures look like. If you've been browsing any of the real
production schemas that are publicly available, you may have noticed that many
mutations define a single global `Input` type to hold all of their arguments:
this pattern was a requirement of some legacy clients but is no longer needed
for new code; we can ignore it.

For many simple mutations, an ID or a handful of IDs are all that is needed,
making this step quite simple. Among collections, we can quickly knock out the
following mutation arguments:
- `delete`, `publish` and `unpublish` all simply need a single collection ID
- `addProducts` and `removeProducts` both need the collection ID as well as a
  list of product IDs

This leaves us with only three remaining "complicated" inputs to design:
- create
- update
- reorderProducts

Let's start with create. A very naive input might look kind of like our original
naive collection model when we started, but we can already do better than that.
Based on our final collection model and the discussion of relationships above,
we can start with something like this:

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
  appliesDisjunctively: Bool!
}

input CollectionRuleInput {
  field: CollectionRuleField!
  relation: CollectionRuleRelation!
  value: String!
}
```

First a quick note on naming: you'll notice that we named all of our mutations
in the form `collection<Action>` rather than the more naturally-English
`<action>Collection`. Unfortunately, GraphQL does not provide a method for
grouping or otherwise organizing mutations, so we are forced into
alphabetization as a workaround. Putting the core type first ensures that all of
the related mutations group together in the final list.

*Rule #17: Prefix mutation names with the object they are mutating for
 alphabetical grouping (e.g. use `orderCancel` instead of `cancelOrder`).*

### Input: Scalars

This draft is a lot better than a completely naive approach, but it still isn't
perfect. In particular, the `description` input field has a couple of issues. A
non-null `HTML` field makes sense for the output of a collection's description,
but it doesn't work as well for input for a couple of reasons. First-off, while
`!` denotes non-nullability on output, it doesn't mean quite the same thing on
input; instead it denotes more the concept of whether a field is "required". A
required field is one the client must provide in order for the request to
proceed, and this isn't true for `description`. We don't want to prevent clients
from creating collections if they don't provide a description (or equivalently,
we don't want to force them to provide a useless `""`), so we should make
`description` non-required.

*Rule #18: Only make input fields required if they're actually semantically
 required for the mutation to proceed.*

The other issue with `description` is its type; this may seem counter-intuitive
since it is already strongly-typed (`HTML` instead of `String`) and we've been
all about strong typing so far. But again, inputs behave a little differently.
Validation of strong typing on input happens at the GraphQL layer before any
"userspace" code gets run, which means that realistically clients have to deal
with two layers of errors: GraphQL-layer validation errors, and business-layer
validation errors (for example something like: you've reached the limit of
collections you can create with your current storage). In order to simplify this
process, we intentionally weakly type input fields when it might be difficult
for the client to validate up-front. This lets the business-logic side handle
all of the validation, and lets the client only deal with errors from one spot.

*Rule #19: Use weaker types for inputs (e.g. `String` instead of `Email`) when
 the format is unambiguous and client-side validation is complex. This lets the
 server run all non-trivial validations at once and return the errors in a
 single place in a single format, simplifying the client.*

It is important to note, though, that this is not an invitation to weakly-type
all your inputs. We still use strongly-typed enums for the `field` and
`relation` values on our rule input, and we would still use strong typing for
certain other inputs like `DateTime`s if we had any in this example. The key
differentiating factors are the complexity of client-side validation and the
ambiguity of the format. HTML is a well-defined, unambiguous specification, but
is quite complex to validate. On the other hand, there are hundreds of ways to
represent a date or time as a string, all of them reasonably simple, so it
benefits from a strong scalar type to specify which format we expect.

*Rule #20: Use stronger types for inputs (e.g. `DateTime` instead of `String`)
 when the format may be ambiguous and client-side validation is simple. This
 provides clarity and encourages clients to use stricter input controls (e.g. a
 date-picker widget instead of a free-text field).*

### Input: Structure, Part 2

Continuing on to the update mutation, it might look something like this:

```graphql
type Mutation {
  # ...
  collectionCreate(title: String!, ruleSet: CollectionRuleSetInput, image: ImageInput, description: String)
  collectionUpdate(collectionId: ID!, title: String, ruleSet: CollectionRuleSetInput, image: ImageInput, description: String)
}
```

You'll note that this is very similar to our create mutation, with two
differences: a `collectionId` argument was added, which determines which
collection to update, and `title` is no longer required since the collection
must already have one. Ignoring the title's required status for a moment, our
example mutations have four duplicate arguments, and a complete collections
model would include quite a few more.

While there are some arguments for leaving these mutations as-is, we have
decided that situations like this call for DRYing up the common portions of the
arguments, even at the cost of requiredness. This has a couple of advantages:
- We end up with a single input object representing the concept of a collection
  and mirroring the single `Collection` type our schema already has.
- Clients can share code between their create and update forms (a common
  pattern) because they end up manipulating the same kind of input object.
- Mutations remain slim and readable with only a couple of top-level arguments.

The primary cost, of course, is that it's no longer clear from the schema that
the title is required on creation. Our schema ends up looking like this:

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

*Rule #21: Structure mutation inputs to reduce duplication, even if this
 requires relaxing requiredness constraints on certain fields.*

### Output

The final design question we need to deal with is the return value of our
mutations. Typically mutations can succeed or fail, and while GraphQL does
include explicit support for query-level errors, these are not ideal for
business-level mutation failures. Instead, we reserve these top-level errors for
failures of the client (e.g. requesting a non-existant field) rather than of the
user. As such, each mutation should define a "payload" type which includes a
user-errors field in addition to any other values that might be useful. For
create, that might look like this:

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

Here, a successful mutation would return an empty list for `userErrors` and
would return the newly-created collection for the `collection` field. An
unsuccessful mutation would return one or more `UserError` objects, and `null`
for the collection.

*Rule #22: Mutations should provide user/business-level errors via a
 `userErrors` field on the mutation payload. The top-level query errors entry is
 reserved for client and server-level errors.*

In many implementations, much of this structure is provided automatically, and
all you will have to define is the `collection` return field.

For the update mutation, we follow exactly the same pattern:

```graphql
type CollectionUpdatePayload {
  userErrors: [UserError!]!
  collection: Collection
}
```

It's worth noting that `collection` is still nullable even here, since if the
provided ID doesn't represent a valid collection, there is no collection to
return.

*Rule #23: Most payload fields for a mutation should be nullable, unless there
 is really a value to return in every possible error case.*

## TLDR: The rules

- Rule #1: Always start with a high-level view of the objects and their relationships before you deal with specific fields.
- Rule #2: Never expose implementation details in your API design.
- Rule #3: Design your API around the business domain, not the implementation, user-interface, or legacy APIs.
- Rule #4: It’s easier to add fields than to remove them.
- Rule #5: Major business-object types should always implement Node.
- Rule #6: Group closely-related fields together into subobjects.
- Rule #7: Always check whether list fields should be paginated or not.
- Rule #8: Always use object references instead of ID fields.
- Rule #9: Choose field names based on what makes sense, not based on the implementation or what the field is called in legacy APIs.
- Rule #10: Use custom scalar types when you’re exposing something with specific semantic value.
- Rule #11: Use enums for fields which can only take a specific set of values.
- Rule #12: The API should provide business logic, not just data. Complex calculations should be done on the server, in one place, not on the client, in many places.
- Rule #13: Provide the raw data too, even when there’s business logic around it.
- Rule #14: Write separate mutations for separate logical actions on a resource.
- Rule #15: Mutating relationships is really complicated and not easily summarized into a snappy rule.
- Rule #16: When writing separate mutations for relationships, consider whether it would be useful for the mutations to operate on multiple elements at once.
- Rule #17: Prefix mutation names with the object they are mutating for
 alphabetical grouping (e.g. use `orderCancel` instead of `cancelOrder`).
- Rule #18: Only make input fields required if they're actually semantically required for the mutation to proceed.
- Rule #19: Use weaker types for inputs (e.g. String instead of Email) when the format is unambiguous and client-side validation is complex. This lets the server run all non-trivial validations at once and return the errors in a single place in a single format, simplifying the client.
- Rule #20: Use stronger types for inputs (e.g. DateTime instead of String) when the format may be ambiguous and client-side validation is simple. This provides clarity and encourages clients to use stricter input controls (e.g. a date-picker widget instead of a free-text field).
- Rule #21: Structure mutation inputs to reduce duplication, even if this requires relaxing requiredness constraints on certain fields.
- Rule #22: Mutations should provide user/business-level errors via a userErrors field on the mutation payload. The top-level query errors entry is reserved for client and server-level errors.
- Rule #23: Most payload fields for a mutation should be nullable, unless there is really a value to return in every possible error case.

## Conclusion

Thank you for reading our tutorial! Hopefully by this point you have a solid
idea of how to design a good GraphQL API.

Once you've designed an API you're happy with, it's time to implement it!
