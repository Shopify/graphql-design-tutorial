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

素朴な設計で存在していたフィールドを、我々の新しい構造に戻すと次のスキーマを得ます。

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

これが我々が解決すべき新しい問題です。
上から順にフィールドをみていき、ひとつずつ解決していきましょう。

### IDs and the `Node` Interface

まずはじめに、コレクション型のIDフィールドに注目します。  
良さそうに見えます。IDはAPIを通してとくに変更や削除操作を行う際に、コレクションを特定するために使われます。

しかし、我々の設計にはひとつ欠けているものがあります。それは`Node`インターフェースです。  
`Node`は既に多くのスキーマでひろく利用されているインターフェースで、以下ように与えられます。

```graphql
interface Node {
  id: ID!
}
```

これはクライアントに対して、本オブジェクトは永続化されていてIDを用いて取得できることを知らせるヒントであり、クライアントは正確かつ効率的にローカルキャッシュを管理することができます。  
特定可能な主要なビジネスオブジェクト（例えば商品やコレクションなど）は`Node`インターフェースを実装するべきです。

APIデザインは次のようになります。
```graphql
type Collection implements Node {
  id: ID!
}
```

*ルール #5: 主要なビジネスオブジェクト型は`Node`インターフェースを実装する。*

### Rules and Subobjects

コレクション内の、つぎの２つのフィールド`rules`と`rulesApplyDisjunctively`をみていきましょう。  

`rules` は単純明快にルールのリストです。リスト自身とその要素がともに非Nullと示されていることに注意してください。GraphQLは`null`と`[]`、`[null]`のすべてを区別します。  
手動コレクションについては、これらのリストが空になるかもしれませんが、Nullにはなりませんし、要素としてNullを含むこともありません。

*Protip: リスト型はほとんどいつもに非Nullで非Null要素しか含みません。
Nullableなリストを使う場合は、本当にそれが空リストとNullを区別する必要があるのか確認しましょう。*

２つめのフィールドはすこし厄介です。`rulesApplyDisjunctively`はルールを分けて適用すべきか否かを表すBooleanフィールドです。  
非Nullと示されていますが、ここで問題に遭遇します。手動コレクションの場合にこのフィールドは一体どの値をもつべきでしょうか。  
trueとfalseのいずれもミスリーディングのように感じますが、かといってNullableにしたところで３状態を表現するフラグは、自動コレクションの場合に違和感を覚えます。

この問題のパズルを試してみるのもいいですが、他にももうひとつ考えるに値する方法があります。  
これら２つのフィールドは明らかに密接に関連しています。意味的にも明らかですし、同じ接頭辞を用いているという事実からも確認できます。  
どうにかして、この関係性をスキーマで表現する方法はないでしょうか？

じつ言えば、詳細実装をさらに越えて考えることで、これらの問題を一度に解決できます。  
詳細実装には直接相当するモデルがないかもしれませんが、GraphQLに新しい`CollectionRuleSet`型を追加するのです。

値や振る舞いが密接に関連するフィールドの集まりがあるときにこの手法は有効です。  
APIレベルにおいて複数のフィールドをまとめることで、意味的に明確な指針を提供でき、さらにNullabilityまつわる問題も解決できます。  
手動コレクションではルールセット自体がNullになり、Booleanフィールドは非Nullのままです。
APIは次のようになります。

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

*Protip: リストのように、Booleanもほとんどいつも非Nullです。*
NullableなBooleanを使う場合は、本当に３状態（Null/false/true）を区別する必要があるのかという点と、設計上のより大きな問題を招かないことを確認ましょう。*

*ルール #6: 密接に関連する複数のフィールドはサブオブジェクトにまとめること。*

### Lists and Pagination

つぎは`products`フィールドです。これは問題ないかもしれません。
`CollectionMembership`型を取り除くことで、我々はすでに関連に関する問題を修正しました。
しかし、じつは他にも良くない点があります。

本フィールドは現状では商品の配列として定義されていますが、やがてコレクションは何千という商品を含むようになり、一度にすべての商品を取得するのは驚くほどコストがかかり非効率的になるでしょう。
このような状況のために、GraphQLはリストページネーションを提供しています。

複数のオブジェクトを返すフィールドや関連を実装する際は、ページネーションが必要かどうかいつも自問自答するようにしましょう。
どのくらいの量のオブジェクトがありえるのでしょうか？どの程度であれば例外とみなせるのでしょうか？

ページネーションフィールドを利用するには、そもそもページネーションのための実装も必要となります。
本チュートリアルでは[Relay Connection spec](https://facebook.github.io/relay/graphql/connections.htm)で定義される[Connections](https://graphql.org/learn/pagination/#complete-connection-model)を用います。

今回の場合、商品フィールドをページネーションさせるには、型の定義を`products: ProductConnection!`に変更すればよいでしょう。
すでに実装が済んでいれば、APIデザインは次のようになります。

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

*ルール #7: リストフィールドについてはページネーションの必要有無を常に確認する。*

###  Strings

次に登場するのは`title`フィールドです。これは本当に現状のままで問題ないでしょう。  
すべてのコレクションはタイトルを持っているはずなので、非Nullのシンプルな文字列型です。

*Protip: Booleanやリストと同様に、GraphQLが空文字列(`""`)とNull(`null`)を区別することに注意しましょう。
NullableなStringを使う場合は、存在しないこと(`null`)と、存在しているが空文字列(`""`)の状態について意味的に本当に差異があるか確認ましょう。*
空文字列を”許容可能だが値が埋められていない”ものとし、Nullを”許容不可能”なものと考えることができます*

### IDs and Relations

さて、`imageId`フィールドに取り掛かりましょう。
このフィールドは、RESTの思想をGraphQLに適用しようとしたときに何がおこるかを示す典型的な例です。
REST APIでは、オブジェクト同士を結びつける手段のひとつとして、レスポンスの中に他のオブジェクトのIDを含める方法が一般的です。
しかし、これはGraphQLにおける代表的なアンチパターンです。

IDを与えて、対象となるオブジェクトを取得するための別のリクエストを強いる代わりに、オブジェクトをグラフに直接含めるべきです。
詰まるところ、これこそがGraphQLが求めるものです。  
含まれるオブジェクトが大きい場合にレスポンスのサイズも膨れ上がるため、REST APIにおいてこの設計はしばし実用的ではありません。  
しかしGraphQLでは、フィールドを明示的にクエリで要求しない限りはサーバーは何も行わないため、この設計がうまくいくのです。

一般的な指針として、IDフィールドはオブジェクト自身の識別子を表す場合のみ使うべきです。  
他のオブジェクトIDをフィールドとしてもつ場合、そのオブジェクトへ参照であるべきかもしれません。  
この指針をこれまでの我々のスキーマに適用すると、次のようになります。

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

*ルール #8: IDフィールドの代わりとして常にオブジェクトの参照を使うこと。*

### Naming and Scalars

`Collection`型の最後のフィールドは`bodyHtml`です。  
これはコレクションに対する説明文です。
コレクションの詳細実装に詳しくない人間にとって、このフィールドがどういった目的で存在しているのか明らかではありません。
 
このAPIの改善としてまずはじめにできることは、フィールド名を`description`に変えて意味を明確にすることです。

*ルール #9: 歴史的経緯や詳細実装ではなく、意味のある概念に基づいてフィールド名を選ぶこと。*

つぎにNullableを止めることができます。
タイトルフィールドでも議論したとおり、Nullと空文字を区別する意味がないため、APIにおいてNullを許容しません。  
たとえデータベーススキーマが当該のカラムにNullを許容していたとしても、それは詳細実装のレイヤに留めておくことができます。

最後に、`String`が実際に正しい型なのかどうか考える必要があります。  
GraphQLは標準で十分なスカラー型（`String`, `Int`, `Boolean`など）を提供していますが、一方でユーザ自身でカスタムスカラー型を定義することも可能にしています。
本ケースはまさにその主要な利用場面です。

多くのGraphQLスキーマでは。ユースケースに応じて独自のカスタムスカラー型を定義しています。  
カスタムスカラー型はコンテキストや意味づけを与え、それらをクライアントに伝えます。
今回の場合では、文字列が有効なHTMLであることを示すために、カスタムスカラー型`HTML`を定義すると良いでしょう。

カスタムスカラー型を追加するときはいつでも、既存のカスタムスカラー型を用いてより適切に目的を達成できるかどうかを確認しましょう。  
カスタムスカラー型を追加しょうとしているとき、またその型が適切だと考えているときでも、チーム内でそれが正しい概念を表現しているかどうか話し合うことをおすすめします。

*ルール #10: 特定の意味を持った値を表現する場合にはカスタムスカラー型を使うこと。*

### Pagination Again

これで`Collection`型のすべてのフィールドを確認しました。
つぎのオブジェクトは非常にシンプルな`CollectionRuleSet`です。  

唯一の懸念は、リストをページネーションするかどうかという点です。
この場合には、ページネーションは手の込みすぎた実装でしょうから、現状のリスト定義のほうが適しています。
ほとんどのコレクションは片手で数えられるほどのルールしか含むことはないですし、ひとつのコレクションに対して膨大なルールを設けることは好ましいユースケースではありません。

たとえ数十のルールだとしても、それはコレクションの定義を考え直す兆候かもしれません。
あるいは単に手動で商品を追加すれば済む話かもしれません。

### Enums

最後に残った型は`CollectionRule`です。
それぞれのルールは、フィルタ対象のカラム（例えば商品名）、比較方法（例えば完全一致）、そして`condition`と呼ばれる紛らわしい比較対象値（例えば"Boots"）によって構成されます。

`condition`と`column`のフィールド名は変更されて然るべきです。
`column`という名前はデータベース固有の語用ですが、我々が考えているのはGraphQLです。  
`field`のほうが良い選択でしょう。

型に着目してみると、`field`と`relation`は内部的には列挙型で実装されているかもしれません（実装言語が列挙型を提供していると想定します。）  
幸いなことにGraphQLもまた列挙型をもっていますので、それら２つのフィールドを列挙型に変更しましょう。

完成したスキーマは以下のようになります。

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

*ルール #11: 特定の値しかとることがないなら列挙型を使うこと。*

## Step Four: Business Logic

さて、これでコレクションに対する最小かつ良く設計されたGraphQL APIが得られました。  
まだ手を付けていないコレクションの詳細部分がありますが（商品の並び順や公開・非公開のの管理のためにまだ多くのフィールドが必要でしょう）、それら全てが等しく従うであろう設計パターンはすでに見てきました。  
しかしながら、詳しく検討すべき事項がまだ少しだけあります。

本節については、我々のAPIの仮想的なクライアントにおけるユースケースから議論を始めると一番都合が良いです。  
以前から一緒に開発に取り組んでいるクライアント側開発者が、ある商品があるコレクションに所属しているかどうかを知る必要があるとしましょう。  
もちろんクライアント側は既存のAPIを用いてこの問いに答えを得ることができます。
我々はコレクションに含まれる全ての商品のリストを公開しているので、クライアントは単にリストを走査すれば済むのです。

しかしこの状況には２つの問題が潜んでいます。  
１つは、これは明確ですが、非効率だという点です。
コレクションは何万もの商品を含むかもしれず、クライアントが全てを取得してひとつずつ調べる方法は大変な時間を必要とします。  
２つ目は、こちらのほうが大きな問題です、クライアントに実装を要求するという点です。
これは設計哲学における最も重要なポイントのひとつで、つねにサーバーこそが唯一の正しいビジネスロジックの情報源であるべきです。
ほとんどの場合でAPIは複数のクライアントに対して提供されます。
もしそれぞれのクライアントが同じロジックを実装しなければならないとしたら、それはコードの重複を生み、不必要なタスクとエラーが発生させる余地を伴います。

*ルール #12: APIはデータだけではなくビジネスロジックを提供すべき。
複雑な計算はサーバーで為されるべきで、複数のクライアントではない。*

クライアントのユースケースに戻りましょう。ここでの最適解は、商品の所属有無判定を解決する専用のフィールドを設けることです。
具体的にはこのような定義になるでしょう。

```graphql
type Collection implements Node {
  # ...
  hasProduct(id: ID!): Bool!
}
```

このフィールドは商品IDを受け取り、サーバーが判定したコレクションに所属しているか否かを表すBoolean値を返します。
このようなフィールドが既存の商品フィールドから得られる情報を重複させているという事項は関係ありません。
GraphQLはクライアントから明示的に要求されたものだけを返します。
ですから、RESTとは異なり、二次的なフィールドを追加することは一切のコストを増加させません。
クライアントは新しいフィールドをクエリする以外にはいかなるコードも書く必要がありませんし、使用される転送帯域もただ一つのIDとboolean値だけです。

一点付け加えるとすれば、ビジネスロジックを提供しているからといって、元々のデータを提供しなくて済むことにはならないという点です。
クライアントは、必要があれば、彼ら自身でビジネスロジックを実装できるべきです。
クライアントが要求するであろうビジネスロジックをすべて予見するとこはできませんし、クライアントに追加すべきフィールドを尋ねる簡単な手段があることも期待できません（それでも、可能な限りそのような手段がないか探す努力をすべきですが。）

*ルール #13: ビジネスロジックによって済まされる場合でも、元データを提供すること。*

最後に、ビジネスロジックを提供するフィールドがAPI全体の型に影響を与えないように注意しましょう。  
ビジネスドメインのデータは依然としてコアモデルだということに変わりはありません。
もしビジネスロジックのフィールドがどうしてもフィットしないと感じているなら、それは背後にあるモデル自体が正しくないことを示すサインかもしれません。

## Step Five: Mutations

我々のGraphQLスキーマに足りていない最後のピースはデータを変更する機能です。
コレクションを追加、変更したり、コレクションと関連するオブジェクトを削除するといった操作です。
スキーマのクエリ部分と同様に、高いレベルから見ていくべきです。
ここでは、個々の入出力にとらわれず、まずはどのような種類のmutationが必要か考えてみましょう。
素朴にCRUD操作のパラダイムに従い、`create`、`delete`、そして`update`操作のmutationを設けてみます。
これは議論を始めるにはちょうど良いものの、正しいGraphQL APIとしては不十分です。

### Separate Logical Actions

CRUDに従おうとすると、すぐに`update`が巨大になるということにまず気付くと思います。
タイトルのようなシンプルなスカラー値を更新するだけではなく、コレクションの公開や非公開、商品の追加/削除/並べ替え操作、自動コレクションのルールの変更といったあらゆる複雑な操作がupdateの責任範囲となります。
サーバーは実装が困難になり、クライアントは論理的に考えて利用することが難しくなります。
対案として、`update`操作を論理的に細かい粒度に分解できるというGraphQLの利点を活かすことができます。
手始めに公開・非公開の操作を切り出すことで、以下のmutationのリストを得ます。

- create
- delete
- update
- publish
- unpublish

*ルール #14: リソースに対するロジックの異なる操作にはそれぞれmutationを用意すること。*

### Manipulating Relationships

`update` mutationは依然として大きすぎる責務を負っているため、引き続き分解を進めましょう。
とはいえ、分解した操作もそれぞれ別の次元（例えば１対多や多対多の関係性に関する操作）から考えてみる価値があるので、後ほど順に見ていきます。
我々はすでにIDとオブジェクトの埋め込みの使い方、あるいはページネーションとリストの使い分けについてみてきました。
mutationにおいても類似する問題に対処しなければなりません。

商品とコレクションの関連については、いくつかの検討可能な設計パターンがあります。
- update mutationに関連全体（例えば`products: [ProductInput!]!`）を埋め込むCRUDスタイルの方法。
しかし、当然ながら関連のリストが大きくなるにつれて間もなく非効率になる。
- “差分”フィールド（例えば`productsToAdd: [ID!]!`と`andproductsToRemove: [ID!]!`）をupdate mutationに埋め込む方法。
変更のあるIDだけを必要とするため関連全体の場合よりも効率的。しかし依然として複数の操作をひとつにまとめてしまっている。
- 別々のmutation（`addProduct`や`removeProduct`など）に分ける方法。柔軟、効率的で、最もうまくいく。

本ケースのようなmutationは実際には異なる個別のロジックを持つため、最後の選択肢が一般に最も安全だとされます。
しかし、考慮すべき要素がいくつもあります。
- 関連のリストは巨大であったり、ページネーションされていますか？
  もしそうなら、リスト全体を埋め込む方法は非現実的です。
  しかし差分埋め込みやmutationの分割といった方法はうまくいくかもしれません。
  もし関連のリストがいつも小さいと仮定できるなら（特に１対１の場合）、リスト埋め込みの方法が最もシンプルでしょう。
- 関連のリストは順序づけされていますか？
  商品ーコレクションの関連は順序づけされていて、手動で並び替えることが許されていました。
  順序づけは、リスト埋め込みやmutation分割（例えば`reorderProducts`を追加できます）では自然な方法で実現できますが、差分埋め込みの場合には困難です。
- 関連は必須ですか？商品とコレクションは関連とは別に存在することができ、それぞれが独自の作成/削除のライフサイクルをもっています。
  もし関連が必須があれば（例えば商品はいずれかのコレクションに所属している必要がある、など）、mutation分割を強く推めます。
  なぜなら、その操作は実際に商品を*作成する*はずで、関連の更新にとどまらないはずです。
- 関連をもつ双方のオブジェクトがもちますか？
  コレクションールールの関連は必須（ルールをはコレクションから離れて存在できません）ですが、ルールはIDすらもちません。
  ルールは明らかにコレクションに従属していますし、関連の数も小さいので、ここではリスト埋め込みが悪くない選択です。
  他のオブジェクトがルールを一意に識別する必要があるかもしれませんが、それはやりすぎのように感じられます。

*ルール #15: 関連に対する操作は複雑で、ひとつの便利な指針で語ることはできない。*

以上の議論を踏まえると、コレクションに対するmutationは以下のようになるでしょう。
- create
- delete
- update
- publish
- unpublish
- addProducts
- removeProducts
- reorderProducts

商品については関連の数が多く、順序づけもなされているため、独自のmutationを切り出しました。
ルールはインライン（リスト埋め込み）のままとしました。
なぜなら、関連の数が少なく、IDを持たせなくて済む程度にマイナーな概念だからです。

最後に、商品に対する操作の名前が複数形（`addProduct`ではなく`addProducts`）になっていることに触れておきます。
これは単純にクライアント側の利便性を考慮しています。
一般的なユースケースを考えた場合、商品に対する操作は単一の商品ではなく、複数のものを対象として追加、削除、並べ替えを実行することになるからです。

*ルール #16: 関連に対するmutationを分けて実装するときには、一度に複数の要素に対して実行することが便利かどうか検討すること。*

### Input: Structure, Part 1

さて、これで我々がどのようなmutationが必要なのかが分かりました。
つぎは入力の型を明らかにしていきます。
一般に公開され利用可能な本物のスキーマを見たことがあれば、多くのmutationが、全ての引数を内包したグローバルな`Input`型をもっていることに気がついたかもしれません。
このパターンはいつかのレガシーなクライアントで必要とされた要件でしたが、いまでは不要になったため考慮しないで構いません。

シンプルなmutationに対しては、ひとつのIDか片手で数えられる程度のIDがあれば十分です。
簡単に済ませてしまいましょう。
コレクションに関して言えば、以下のmutation引数に関してはすぐに答えが出せます。

- `delete`と`publish`、`unpublish`はすべて単純にひとつのコレクションのIDが必要
- `addProducts`と`removeProducts`はどちらもコレクションのIDに加えて商品のIDリストが必要

残った他の３つの"複雑な"入力に対処していきます。
- create
- update
- reorderProducts

createからです。
極めて素朴な入力の型は、我々が当初考えていた素朴なコレクションモデルのようなものになるかもしれません。しかし我々はすでにより良い方法を知っています。
最終的なコレクションの型と、関連に関する先の議論をもとにして考えると、以下のような案から考えていくことができます。

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

まずは命名について簡単にみていきます。
すべてのmutationに対して、自然な英語では`<action>Collection`であるはずが、あえて`collection<Action>`の形式の名前をつけています。
残念ながら、GraphQLはmutationをまとめたり整理するための手立てを提供しません。
そこで、ワークアラウンドとして無理やりアルファベット順に並べているのです。
対象となる型名を接頭辞とすることで、関係するmutationをまとめて並べることができます。

*ルール #17: アルファベット順でグループ化するために、mutationの接頭辞に操作対象のオブジェクト名を用いること（例えば`cancelOrder`ではなく`orderCancel`とする。）*

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
