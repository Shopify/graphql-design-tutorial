# 教學：GraphQL API 設計

這份教學文件是由[Shopify](https://www.shopify.ca/)編撰，原本僅供公司內部使用。但我們認為任何想學建立 GraphQL API 的人都可以從中獲益，所以建立了這份公開版本。

這份文件是奠基在 Shopify 在 schema 開發實踐上，累積迄今近三年的經驗與所學；期間，它的內容不斷演進，而未來也將持續更新。

我們認為，在絕大多數情況下，這些設計準則都能適用；然而，它們也未必完全符合你的需求。即便是在 Shopify 內部，我們也仍不斷地針對這些準則進行討論，或有不完全遵循準則規範的時候。因此，請不要盲目地複製與服從這份文件裡所列的所有規範：選用那些符合你的使用情境的。

目錄
=================
* [引言](#引言)
* [第零步：背景知識](#第零步：背景知識)
* [第一步：俯視設計](#第一步：俯視設計)
* [第二步：A Clean Slate](#step-two-a-clean-slate)
  * [Representing CollectionMemberships](#representing-collectionmemberships)
  * [Representing Collections](#representing-collections)
  * [Conclusion](#conclusion)
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
