# 教學：GraphQL API 設計

這份教學文件是由[Shopify](https://www.shopify.ca/)編撰，原本僅供公司內部使用。但我們認為任何想學建立 GraphQL API 的人都可以從中獲益，所以建立了這份公開版本。

這份文件是奠基在 Shopify 在 schema 開發實踐上，累積迄今近三年的經驗與所學；期間，它的內容不斷演進，而未來也將持續更新。

我們認為，在絕大多數情況下，這些設計準則都能適用；然而，它們也未必完全符合你的需求。即便是在 Shopify 內部，我們也仍不斷地針對這些準則進行討論，或有不完全遵循準則規範的時候。因此，請不要盲目地複製與服從這份文件裡所列的所有規範：選用那些符合你的使用情境的。

目錄
=================
* [引言](#引言)
* [第零步：背景知識](#第零步：背景知識)
* [第一步：俯視設計](#第一步：俯視設計)
* [第二步：重新設計](#第二步：重新設計)
  * [重新設計 `CollectionMemberships`](#重新設計-collectionmembership)
  * [重新設計「商品系列」](#重新設計商品系列)
  * [小結](#小結)
* [第三步：Adding Detail](#step-three-adding-detail)
  * [Starting point](#starting-point)
  * [IDs and the Node Interface](#ids-and-the-node-interface)
  * [Rules and Subobjects](#rules-and-subobjects)
  * [Lists and Pagination](#lists-and-pagination)
  * [Strings](#strings)
  * [IDs and Relations](#ids-and-relations)
  * [Naming and Scalars](#naming-and-scalars)
  * [Pagination Again](#pagination-again)
  * [Enums](#enums)
* [第四步：Business Logic](#step-four-business-logic)
* [第五步：Mutations](#step-five-mutations)
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

我們先暫時退後一步，試著綜觀全局、重新思考。一個繁複但良好設計的 GraphQL API 通常是由許多物件所組成，再透過多個連結和數十個欄位跟串起物件之間的關聯。想要一步到位、一次完成這麼複雜的設計，往往會造成混亂跟錯誤。正確的作法，應該是由更高階的抽象層級出發，專注於設計型態（types）跟它們之間的關聯，之後再煩惱它們的具體欄位（fields）或變更（mutation）等等。原則上，這就像在[實體關係模型](https://zh.wikipedia.org/zh-tw/ER模型)（Entity-Relationship model）的框架之上，加入一些 GraphQL 特有的設計。這麼一來，我們簡化一下原本初步、粗略的 schema 設計，可以得到下面的結果：

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

## TLDR：設計準則

- 準則一：永遠記得從更高階的抽象層級出發，先設計物件與它們的關聯，之後再進入到具體欄位。
- 準則二：不要在 API 設計中納入不必要的技術細節。
- 準則三：以你的業務領域為核心進行 API 設計，而非照抄後端資料庫、前端使用者介面的模型或既有的 API 設計。

## 結語

謝謝你的收看！讀到這裡，希望你已經更熟悉、理解如何設計一個良好、嚴謹的 GraphQL API。

如果你已經完成了一個令人滿意的 API 設計，那就開始實作它吧！
