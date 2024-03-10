# 教學：GraphQL API 設計

這份教學文件是由[Shopify](https://www.shopify.ca/)編撰，原本僅供公司內部使用。但我們認為任何想學建立 GraphQL API 的人都可以從中獲益，所以建立了這份公開版本。

這份文件是奠基在 Shopify 在 schema 開發實踐上，累積迄今近三年的經驗與所學；期間，它的內容不斷演進，而未來也將持續更新。

我們認為，在絕大多數情況下，這些設計準則都能適用；然而，它們也未必完全符合你的需求。即便是在 Shopify 內部，我們也仍不斷地針對這些準則進行討論，或有不完全遵循準則規範的時候。因此，請不要盲目地複製與服從這份文件裡所列的所有規範：選用那些符合你的使用情境的。

目錄
=================
* [引言](#引言)
* [第零步：背景知識](#第零步背景知識)
* [第一步：俯視設計](#第一步俯視設計)
* [第二步：重新設計](#第二步重新設計)
  * [重新設計 `CollectionMemberships`](#重新設計-collectionmembership)
  * [重新設計「商品系列」](#重新設計商品系列)
  * [小結](#小結)
* [第三步：加入細節](#第三步加入細節)
  * [Starting point](#starting-point)
  * [IDs and the Node Interface](#ids-and-the-node-interface)
  * [Rules and Subobjects](#rules-and-subobjects)
  * [Lists and Pagination](#lists-and-pagination)
  * [Strings](#strings)
  * [IDs and Relations](#ids-and-relations)
  * [Naming and Scalars](#naming-and-scalars)
  * [Pagination Again](#pagination-again)
  * [Enums](#enums)
* [第四步：商業邏輯](#第四步商業邏輯)
* [第五步：資料修改（mutation）](#第五步資料修改)
  * [Separate Logical Actions](#separate-logical-actions)
  * [Naming the Mutations](#naming-the-mutations)
  * [Manipulating Relationships](#manipulating-relationships)
  * [Input: Structure, Part 1](#input-structure-part-1)
  * [Input: Scalars](#input-scalars)
  * [Input: Structure, Part 2](#input-structure-part-2)
  * [Output](#output)
* [TLDR：設計準則](#tldr-the-rules)
* [結語](#結語)

## 引言

歡迎！這份文件將引領你走過一趟 GraphQL API 設計之旅。API 設計是一項複雜、困難的任務，它高度仰賴你的業務領域的深入理解，並且必須歷經不斷的疊代與驗證，從而確保設計的良好與嚴謹。

## 第零步：背景知識

這份教學文件中，我們將以電商領域為例：想像你在一家電商公司工作，這個電商平台有一個可以查詢產品資訊的 GraphQL API，但目前僅止於此。你的團隊最近在後端開發新加入了「商品系列」（collections）的功能，希望能把它加入 API 之中。

「商品系列」是用來呈現一組「產品」（products）的必備功能：舉例來說，你可以建立一個 T-shirt 商品系列，把所有的 T-shirt 產品加入其中。透過建立商品系列，你可以在你的網站上分門別類地陳列產品，也可以針對商品系列設定特定指令（例如針對某個商品系列內的所有產品設定折扣促銷）。

在後端，「商品系列」這項新功能包括：

* 每個商品系列都有一些簡單的屬性，例如標題、敘述（允許 HTML 格式化文本）和附圖。
* 分為兩種商品系列：「手動」（"manual"）和「自動」（"automatic"），前者由使用者人工選取產品、加入列表中，後者則是根據使用者所設定的特定條件，由系統自動將符合條件的產品包含於其中。
* 產品跟商品系列之間是多對多（many-to-many）的關係，因此在後端資料庫設有一個名為 `CollectionMembership` 的合併資料表格，用以描述產品跟商品系列兩者的關聯。
* 就像我們可以上、下架產品一樣，我們也可以設定商品系列的狀態，是否發佈在商場網站上。

有了這些背景知識，我們可以開始思考如何設計 API。

## 第一步：俯視設計

一個最初步、粗略的 schema 設計可能類似這樣（這裡暫時不羅列 `Product`等既有的型態）：

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

雖然這個設計只有四種物件跟一個介面，但乍看之下已經頗為複雜。而且這個設計還沒有包括所有所需的功能，例如，我們沒辦法用這個 API 建立在手機應用程式的商品系列功能。

我們先暫時退後一步，試著綜觀全局、重新思考。一個繁複但良好設計的 GraphQL API 通常是由許多物件所組成，再透過多個連結和數十個欄位跟串起物件之間的關聯。想要一步到位、一次完成這麼複雜的設計，往往會造成混亂跟錯誤。正確的作法，應該是由更高階的抽象層級出發，專注於設計型態（types）跟它們之間的關聯，之後再煩惱它們的具體欄位（fields）或資料修改（mutation）等等。原則上，這就像在[實體關係模型](https://zh.wikipedia.org/zh-tw/ER模型)（Entity-Relationship model）的框架之上，加入一些 GraphQL 特有的設計。這麼一來，我們簡化一下原本初步、粗略的 schema 設計，可以得到下面的結果：

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

在這個簡化的版本，我們移除了所有純量的欄位、欄位名稱和是否接受空值等，目前暫時不需要注意的細節。剩下的這個只保有輪廓、類似 GraphQL 的東西，讓我們可以聚焦在更高階的型態和關聯設計。

*準則一：永遠記得從更高階的抽象層級出發，先設計物件與它們的關聯，之後再進入到具體欄位。*

## 第二步：重新設計

接下來，我們接續上一步的初階成果，進一步修正其中最關鍵的問題。

我們目前的設計定義了「手動商品系列」（`ManualCollection`）和「自動商品系列」（`AutomaticCollection`）的架構，以及一個合併資料表格 `CollectionMembership`，用來描述「產品與商品系列之間多對多的關係」。這個初始版本很井然有序，與資料庫的實體關係模型完全契合，但這其實是個錯誤的設計方式。

這種設計的根本問題在於，設計 API 跟設計資料庫所關注的目的並不相同，兩者在實作上也經常是在不同的抽象層級。完全依照資料庫端的架構設計 API 會在後續應用上造成很大的麻煩。

### 重新設計 `CollectionMembership`

你可能已經發現，在 schema 中加入合併資料表格 `CollectionMembership` 型態是個很明顯的失誤。這個表格描述了產品與商品系列之間多對多的關係。注意：是「產品與商品系列」之間的關係，從語意、或業務領域的視角來看，這個從屬關係並不具意義；它只是一個需要被實現的技術細節。

換句話說，這個關係不應該出現在我們的 API 中。我們的 API 應該呈現的是業務領域實際應用所需的物件關係：意即對各個商品系列而言，它與其所屬產品的直接關聯。因此，我們可以進一步把 schema 設計修正成：

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

這樣好多了。

*準則二：不要在 API 設計中納入不必要的技術細節。*

### 重新設計「商品系列」

目前這個 API 版本還有一個致命的缺陷，但除非我們非常熟悉業務領域，不然這個缺陷比較難被發現。在目前的設計中，我們替「自動商品系列」和「手動商品系列」分別設計了 `AutomaticCollection` 和 `ManualCollection` 兩個型態，兩者分別實現了同一個介面 `Collection`。直觀看來，這個設計似乎很符合邏輯：它們有許多相同的欄位，但仍在關聯（`AutomaticCollection` 設有選取所屬產品的規則）和某些特性上有所差異。

但從商業模式來看，這些差異基本上也只是技術細節。「商品系列」的定義是一組產品，至於怎麼挑選哪些產品加入列表則是次要的。我們或許會在某個時間點加入第三種選取產品的方法（例如機器學習？），或新增一種同時允許手動跟自動加入產品的商品系列——但不變的是，它們都還是一種「商品系列」。我們甚至可以說，目前商品系列沒有同時允許手動跟自動加入產品這個選項，是個設計上的錯誤。總而言之，我們的 API 應該長得像這樣才對：

```graphql
type Collection {
  [CollectionRule]
  Image
  [Product]
}

type CollectionRule { }
```

這個設計合理多了。或許你會覺得奇怪，手動商品系列並沒有選取所屬產品的條件啊？但注意，這裡的 `CollectionRule` 欄位是一個串列（list）。在這個新的設計中，我們只要指定 `ManualCollection` 的所屬產品選取條件是一個空的串列，就沒問題了。

### 小結

我們需要對所設計的業務領域有非常深入的理解，才能在這個抽象層級選擇最好的 API 設計。我們很難在這份教學文件中提供更深入的範例，更全面演示在真實應用情境中這些設計需求是多麼複雜；不過希望上述「商品系列」的例子雖然簡單，但已經充分闡述了背後的理論與邏輯。在設計 API 時，深入理解業務領域，設想各式各樣複雜的實際使用情境，反覆地由此自我拷問，是很重要的；切記不要盲目地照本宣科、在 API 設計上直接複製貼上實作的技術細節。

值得一提的是，一個良好的 API 設計也不應該是仿照使用者介面的建模。後端的 API 實作執行和前端的使用者介面都可以是很好的設計參考和靈感來源，但最終 API 設計所關注的是業務領域的應用情境。

更為重要的一點：我們並不必然需要照搬原有的 REST API 設計（如果有的話）。REST 跟 GraphQL 有著不同的設計準則：某個在 REST API 中很棒的設計，並不必然適用在 GraphQL API。

盡可能放下包袱，從頭開始。 

*準則三：以你的業務領域為核心進行 API 設計，而非照抄後端資料庫、前端使用者介面的模型或既有的 API 設計。*

## 第三步：加入細節

現在我們有一個乾淨簡潔的抽象模型，是時候重新加入之前省略的欄位，著手考慮更為細節的層級了。

在開始加入細節之前，捫心自問：我們現在真的需要這個欄位嗎？看到資料庫有某個欄位、前端模型有某個屬性或 REST API 有某個屬性，並不代表我們需要自然而然把它放進 GraphQL schema 中。

我們應該根據實際使用需求，決定是否要在 schema 中納入某個元素（欄位、引數、型態等）。在 GraphQL schema 中加入新的元素非常容易；但變更或移除既有的設計往往非常困難，很容易招致整體設計的損壞。

*準則四：新增一個欄位遠比刪掉一個欄位容易。*

### Starting point

### ID 和 `Node` 介面

在我們的 Collection 類型中，最上面的欄位是一個 ID 欄位，這很正常也很合理；我們需要使用這個 ID 來識別整個 API 中的 collection，特別是在執行修改或刪除它們等操作的時候。
然而，我們設計的這一部分缺少了一個東西：`Node` 介面。這是一個非常常用的介面，已經存在於大多數的 schema 之中，它看起來像這樣：
```graphql
interface Node {
  id: ID!
}
```
它向客戶端提示該物件可以透過給定的 ID 來持久化和取回資料，這允許客戶端精準有效地管理本地快取以及其他技巧。你絕大多數可識別的業務物件（例如，product、collection，等等）應該實作 `Node` 介面。

我們初始的設計現在看起來像這樣：
```graphql
type Collection implements Node {
  id: ID!
}
```

*準則五：主要的業務物件類型都應該實作 `Node` 介面。*

### Rules 和子物件

我們接下來將一起考慮 Collection 類型上的兩個欄位：`rules` 以及 `rulesApplyDisjunctively`。第一個非常直白：一個 rule 的列表。請注意，列表本身和列表中的項目都被標記不可為空值：這樣很好，因為 GraphQL 確實會區分 `null`、`[]` 和 `[null]`對手動 collection 來說，這列表可以是空列表，但它不能為 null 也不能包含 null。

*專業提醒：列表類型的欄位通常總是不可為空值的列表包含著不可為空值的項目。如果你想要一個可以為空值的列表，請確保真的能在語意上區分空列表以及空值。*

第二個欄位有點奇怪：它是一個布林欄位，指出這些規則是否分開套用。它也不允許空值，但是在這裡我們遇到了一個問題：手動 collection 的這個欄位應該取什麼值？不管將它設定成 false 還是 true 感覺會產生誤導，但是使該欄位可以為空值，然後在處理自動 collection 時，讓它變成一個有三種的奇怪狀態這樣也很怪。當我們對此感到困惑時，還有另一件事值得一提：這兩個欄位明顯且錯綜複雜地相關。
這在語義上是正確的，我們選擇了具有共用前綴的名稱的這一事實也暗示了這一點。有沒有辦法以某種方式在 schema 中指出這個關聯？

事實上，要解決這所有問題，我們可以透過進一步脫離底層實作並加入一個新的特殊 GraphQL 類型：`CollectionRuleSet`。當你有一組密切相關的欄位，而且其值和行為相互關聯時，這樣做通常是很合理的。通過在 API 層級將這兩個欄位分組到它們自己的類型中，我們提供了一個清晰的語義指示，也解決了我們所有關於可允許空值的問題：對於手動 collection 來說，這個 rule-set 本身為空值。而布林欄位可以維持不允許空值。這把我們導向了以下的設計：

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

*專業提醒：就像列表一樣，布林欄位通常總是不為空值。如果你想要一個可為空值的布林，請確保真的能在語意上區分這三個狀態（null/false/true）並且這不意味著一個更大的設計缺陷。*

*準則六：把密切相關的欄位抽出來放到子物件。*

### Lists and Pagination

### 字串

下一個要看的是 `title` 欄位。它的設計完全沒問題。它是一個簡單的字串，被標記為不允許空值，因為所有的 collection 都必須有 title。

*專業提醒：跟布林以及列表一樣，值得注意 GraphQL 有區分空字串（`""`）以及空值（`null`），所以如果你需要一個可以為空值的字串，請確保不存在（`null`）和存在但為空（`""`）之間存在合理的語意差異。你通常可以把空字串想成代表「可使用，但未填」，空值代表「不可使用」。*

### IDs and Relations

### Naming and Scalars

### Pagination Again

### 列舉

這將我們帶到了 schema 中的最後一個類型，`CollectionRule`。每條 rule 是由一個要匹配的 column（例如，product title）、一種關聯類型（例如，equality）和要實際使用的值（例如，「Boots」）組成這有點混淆地被稱作為 `condition`。那最後一個欄位可以重新命名，而且 `column` 也是；column 是一個非常特定於資料庫的用詞，而我們是在做 GraphQL。`field` 可能是個更好的選擇。

就類型而言，`field` 和 `relation` 在內部都有可能是用列舉實作（假設你選的程式語言有支援列舉）。幸運的是 GraphQL 也有列舉，所以我們可以將這兩個欄位轉換為列舉。我們完成的 schema 設計現在看起來像這樣：

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

*準則十一：對只能接受一組特定值的欄位使用列舉。*

## 第四步：商業邏輯

## 第五步：資料修改（Mutation）

現在我們的 GraphQL schema 設計還缺少的最後一個部分是實際去更改值的能力：建立、更新、以及刪除 collection 和關聯的部分。
跟 schema 的可讀部分一樣，我們應該從高階概覽開始：在這種情況下，只包含我們想要實作的各種 mutation，而不用擔心它們的個別的 input 跟 output。不經思考的話，我們可能會遵照 CRUD 範式並只有 `create`、`delete`、以及 `update` mutation。
雖然這是一個不錯的起點，但對於合適的 GraphQL API 來說還不夠。

### 拆分合理的操作

如果我們堅持只使用 CRUD，可能會注意到的第一件事是，我們的 `update` mutation 很快會變得
很龐大，不僅僅負責更新簡單的 scalar 值像是 title，也被用來執行複雜的操作，像是發布/取消發布、添加/刪除/重新排序 collection 中的 product、更改自動 collection 的 rule，等等。這讓它在伺服器上很難實作，在客戶端也很難去使用。
因此，我們可以利用 GraphQL 來把它拆分成更細緻、更合理的操作。作為第一步，我們可以拆分發布/取消發布，從而產生以下的 mutation 列表：
- create
- delete
- update
- publish
- unpublish

*準則十四：為資源上獨立的操作撰寫不同的 mutation。*

### 為 Mutation 命名

作為一個起點，不要預設使用 CRUD 動詞名稱。用 CRUD 動詞可以很好的描述資料庫的陳述句，不過它們是應該對 API 使用者隱藏的實作細節。考慮到你的領域、情境以及 mutation 的作用，CRUD 動詞很少能充分描述一個業務操作。如果有更有意義的動詞可以使用，那就傾向使用它。例如，如果主要的作用是取消發布一個 collection，不要使用 `collectionDelete` 當作名字；而將它命名為 `collectionUnpublish`。

### Manipulating Relationships

### Input：結構，第一部分

### Input：Scalars

### Input：結構，第二部分

我們繼續處理 update mutation 的部分，它可能看起來像這樣：

```graphql
type Mutation {
  # ...
  collectionCreate(title: String!, ruleSet: CollectionRuleSetInput, image: ImageInput, description: String)
  collectionUpdate(collectionId: ID!, title: String, ruleSet: CollectionRuleSetInput, image: ImageInput, description: String)
}
```

你會注意到，它跟我們的 create mutation 非常相似，但有兩個不同之處：多了一個 `collectionId` 參數來決定要更新哪個 collection，以及不再需要 `title` 因為這個 collection 已經有了。暫時忽略 title 的必填狀態，我們範例中的 mutation 有四個重複的參數，而一個完整的 collection 模型還會有更多。

雖然有一些爭論是要讓這些 mutation 保持原本那樣，但我們已經決定，像是在參數共用的這種情況，即使是要犧牲欄位是否必填的正確性作為代價，我們也要避免重複。這樣有幾個優點：
- 我們最終得到一個代表 collection 概念的 input 物件，並對應到我們 schema 已經有的 `Collection` 類型。
- 客戶端可以在 create 跟 update 表單之間共享程式碼 （一種常見的模式）因為他們最終操作的是同一種 input 物件。
- Mutation 只有幾個頂層參數，維持輕量以及可讀。

當然，主要的代價是，從 schema 上不再能清楚知道在建立時 title 是必要的。我們的 schema 最後看起來像這樣：

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

*準則二十一：結構化 mutation 的 input 來減少重複，即便這需要放寬某些欄位的必填限制。*

### Output

我們需要處理的最候一個設計問題是 mutation 的回傳值。 通常情況下，mutation 可能成功也可能失敗，雖然 GraphQL 確實有明確地支援查詢層級的錯誤，但這對業務層級的 mutation 失敗並不理想。因此，我們將這些頂層的錯誤保留給客戶端（例如：請求一個不存在的欄位）而不要用在用戶上。因此，每一個 mutation 都應該定義一個「payload」類型，除了其他可能有用的值以外，還要包含一個使用者錯誤的欄位。對於 create 來說，它可能看起來像這樣：

```graphql
type CollectionCreatePayload {
  userErrors: [UserError!]!
  collection: Collection
}
type UserError {
  message: String!
  # 導致錯誤的 input 欄位的路徑。
  field: [String!]
}
```

在這裡，成功的 mutation 會回傳一個空的列表給 `userErrors`，並會回傳剛建立的 collection 給 `collection` 欄位。不成功成功的 mutation 會回傳一個以上的 `UserError` 物件，並回傳 `null` 給 collection。

*準則二十二：Mutation 應該透過 mutation payload 的 `userErrors` 欄位來提供使用者/業務層級錯誤。底層的查詢錯誤欄位是保留給客戶端和伺服器層級的錯誤。*

在許多實作中，大多是自動提供這種結構，而你需要定義的只是 `collection` 回傳欄位。

update mutation 的部分，我們遵循完全相同的模式：

```graphql
type CollectionUpdatePayload {
  userErrors: [UserError!]!
  collection: Collection
}
```

值得注意的是，甚至在這裡，`collection` 仍然可以為空值，因為如果提供的 ID 不代表有效的 collection，則沒有要回傳的 collection。

*準則二十三：大部分 mutation 的 payload 欄位應該要可以接受空值，除非在每種可能的錯誤情況下都確實有要回傳的值。*

## TLDR：設計準則

- 準則一：永遠記得從更高階的抽象層級出發，先設計物件與它們的關聯，之後再進入到具體欄位。
- 準則二：不要在 API 設計中納入不必要的技術細節。
- 準則三：以你的業務領域為核心進行 API 設計，而非照抄後端資料庫、前端使用者介面的模型或既有的 API 設計。
- 準則四：新增一個欄位遠比刪掉一個欄位容易。
- 準則五：主要的業務物件類型都應該實作 `Node` 介面。
- 準則六：把密切相關的欄位抽出來放到子物件。
- 準則七：總是檢查清單欄位是否應該被分頁。
- 準則八：總是使用物件參照而不是 ID 欄位。
- 準則九：根據合理性選擇欄位名稱，而不要根據實作或是舊的 API 怎麼稱呼它。
- 準則十：當你開放查詢欄位的值有特殊語意，使用自定義的 scalar 類型。
- 準則十一：對只能接受一組特定值的欄位使用列舉。
- 準則十二：API 應該提供商業邏輯，而不只是資料。複雜的計算應該在伺服器上統一完成，而不是在許多的客戶端上做。
- 準則十三：也要提供原始資料，即使它周圍已經有商業邏輯。
- 準則十四：為資源上獨立的操作撰寫不同的 mutation。
- 準則十五：操作物件關聯非常複雜，不容易總結成一條簡潔的規則。
- 準則十六：當為關聯撰寫個別的 mutation 時，考慮一次對多個項目進行操作是否會對這個 mutation 有用。
- 準則十七：為了按照字母分組，把 mutation 操作的物件名稱前綴在 mutation 名稱上（例如，使用 `orderCancel` 而不是 `cancelOrder`）。
- 準則十八：只有 mutation 語義上確實需要該 input 欄位才能執行，才把它標為必填。
- 準則十九：當格式明確而且客戶端驗證很複雜時，使用較弱的類型來限制 input（例如：用 `String` 來取代 `Email`）。這讓伺服器可以一次執行所有複雜的驗證並用單一格式統一回傳錯誤，簡化了客戶端的複雜度。
- 準則二十：當格式可能不明確且客戶端驗證很簡單時，使用較強的類型來限制 input（例如：用 `DateTime` 來取代 `String`）。這樣清楚明瞭並鼓勵客戶端使用比較嚴格的輸入控制（例如：用日期選擇器取代自由的文字輸入框）。
- 準則二十一：結構化 mutation 的 input 來減少重複，即便這需要放寬某些欄位的必填限制。
- 準則二十二：Mutation 應該透過 mutation payload 的 `userErrors` 欄位來提供使用者/業務層級錯誤。底層的查詢錯誤欄位是保留給客戶端和伺服器層級的錯誤。
- 準則二十三：大部分 mutation 的 payload 欄位應該要可以接受空值，除非在每種可能的錯誤情況下都確實有要回傳的值。

## 結語

謝謝你的收看！讀到這裡，希望你已經更熟悉、理解如何設計一個良好、嚴謹的 GraphQL API。

如果你已經完成了一個令人滿意的 API 設計，那就開始實作它吧！