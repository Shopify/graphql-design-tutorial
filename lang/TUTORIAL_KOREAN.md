# Tutorial: Designing a GraphQL API

처음 이 튜토리얼은 [Shopify](https://www.shopify.ca/) 내부에서 쓰기 위해 만들어졌습니다. 그러나 이 튜토리얼이 다른 사람이 graphQL API를 만드는 데 도움을 줄 것이라 생각했기 때문에 우리는 이것을 공개하기로 했습니다.

거의 3년이 넘는 시간 동안 만들어진 Shopify의 스키마들로부터 배운 것들을 바탕으로 만들었습니다. 이 튜토리얼은 그동안 많이 발전해왔고, 앞으로도 계속해서 바뀔 예정입니다. 정해진 것은 없습니다.

우리는 스키마 디자인 가이드가 대부분의 경우에 유용할 것이라고 생각합니다. 어쩌면 여러분에게는 맞지 않을 수도 있습니다. 대부분의 규칙이 항상 100% 적용되는 것은 아니기 때문에, Shopify 내부에서도 여전히 그 문제에 대한 답을 찾고 있고, 여러 예외 사항도 존재합니다. 그러니 여러분에게 맞는 것을 차용하시길 바랍니다.

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
  * [Naming the Mutations](#naming-the-mutations)
  * [Manipulating Relationships](#manipulating-relationships)
  * [Input: Structure, Part 1](#input-structure-part-1)
  * [Input: Scalars](#input-scalars)
  * [Input: Structure, Part 2](#input-structure-part-2)
  * [Output](#output)
* [TLDR: The rules](#tldr-the-rules)
* [Conclusion](#conclusion-1)
* [Appendix: Polymorphism](#appendix-polymorphism)
  * [No Shared Behaviour](#no-shared-behaviour)
  * [Simple Shared Behaviour](#simple-shared-behaviour)
  * [Complex Shared Behaviour: Base Interfaces](#complex-shared-behaviour-base-interfaces)
  * [Complex Shared Behaviour: Push-Down Polymorphism](#complex-shared-behaviour-push-down-polymorphism)

## Intro

환영합니다! 이 문서는 여러분에게 새로운 graphQL을 (또는 이미 존재하는 graphQL API에 새 API를) 디자인하는 방법을 차근차근 알려줄 겁니다. API 디자인은 반복과 실험, 그리고 여러분의 비즈니스 도메인에 대한 이해가 필요한 어려운 일이지만요.

## Step Zero: Background

여러분이 e-commerce 회사에서 일하고 있다고 상상해보시기 바랍니다. 여러분은 기존 graphQL API를 갖고 있습니다. 이 API는 제품들에 대한 정보를 외부에 노출하고 있지만, 정보가 그렇게 많은 건 아닙니다. 여러분의 팀은 백엔드에서 "collections"를 구현하는 프로젝트를 방금 끝냈고 기존 API에 그 collections를 사용하길 원합니다.

Collection은 제품을 그룹핑하는 데 아주 좋은 방법입니다. 예를 들어, 여러분이 티셔츠 collection을 갖고 있다고 해봅시다. Collection은 여러분의 웹사이트를 볼 때, 화면에 띄워주는 용도로 사용될 수 있습니다. 또한, 프로그래밍 업무를 위해서도 사용될 수 있죠. (예를 들면, 특정 collection에 대해서만 할인을 적용하고 싶을 때 사용할 수 있겠네요.)

백엔드에서는, 다음과 같은 것들을 통해 일을 진행할 수 있습니다.

- 모든 collection들은 title, description(HTML 포맷팅(:html 태그 같은 것들)을 포함하는), image와 같은 간단한 속성들을 갖고 있습니다,
- Collection에는 두 가지 종류가 있습니다. : 여러분이 원하는 것들을 포함시키기 위해 직접 리스트업하는 "수동적인" collection들, 그리고 어떤 규칙을 정해놓고 collection들이 알아서 채워질 수 있도록 하는 "자동적인" collection들이 있습니다.
- product와 collection는 다대다 관계이므로, 중간에 'CollectionMembership'이라는 조인 테이블을 갖고 있습니다.
- product와 같은 collections은 사이트 앞단에 보일 수도 있고 그렇지 않을 수도 있습니다.

이러한 배경을 가지고, 어떻게 API를 설계할 수 있을 지 생각해보기로 합시다.

## Step One: A Bird's-Eye View

단순한 방법으로 만든 스키마는 다음과 같이 구성할 수 있습니다.
(`Product`같이 이미 존재하는 타입들은 모두 생략합니다.)

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

한 눈에 보기에도 상당히 복잡해보입니다. 4개의 객체와 하나의 인터페이스만 있을 뿐인데도요. 또한, 우리가 이 API를 이용해 모바일 앱의 collection 기능 같은 것을 구축하려고 한다면, 이 스키마는 우리가 필요로 하는 모든 특징을 명확히 구현하고 있는 것도 아닙니다.

그러므로 한 발짝 뒤로 가봅시다. 복잡한 graphQL API는 다양한 경로와 몇 십개의 필드를 통해 많은 객체들을 구성하게 됩니다. 이런 API를 모두 한 번에 설계하려고 하는 것은 혼란과 실수를 야기하기 좋은 방법입니다.

처음부터 구체적으로 시작하기보다는 더 높은 곳에서 바라보는 것부터 시작하는 게 좋습니다.
구체적인 field나 mutation들은 걱정하지 말고, 일단 type과 그들의 관계에만 집중해보세요.

기본적으로 E-R 모델([Entity-Relationship model](https://en.wikipedia.org/wiki/Entity%E2%80%93relationship_model))을 떠올릴 수 있지만 몇몇의 GraphQL적인 특징들이 있습니다.

만약 위의 단순한 스키마를 더 간단하게 만들고 싶다면, 다음과 같은 방법을 시도해봅시다.

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

더 간단한 스키마를 얻기 위해, 모든 스칼라 field와 field 이름, nullable한 정보는 뺐습니다. 남겨진 것들은 graphQL과 여전히 비슷하지만, 더 높은 수준(나무가 아닌 숲을 보는 관점으로)의 type과 그 관계에 집중할 수 있게 합니다.

*규칙 #1: 구체적인 필드를 다루기 전에, 항상 객체들과 그 사이의 관계를 높은 수준에서 바라보는 것부터 시작하세요.*

## Step Two: A Clean Slate

API를 다루기 좀 더 간단하게 만들었습니다. 이제 이 설계에 있는 큰 결함을 해결할 수 있습니다.

이전에 말씀드린 것처럼, 우리가 만든 구현은 '수동적인' 또는 '자동적인' collection들을 정의하고 있습니다. 이 단순한 API 디자인은 우리의 구현에 대해서는 명확히 짜여졌습니다만, 사실 이건 잘못된 방법입니다.

이 접근 방법의 가장 근본적인 문제는, API는 '구현과는 다른 목적을 위해 동작하며, 빈번하게 다른 추상화 수준에서 동작한다'는 것입니다. 이 경우, 우리의 구현은 많은 다른 프론트 단에서 실수를 유발하도록 할 수 있습니다.

### Representing `CollectionMembership`s

가장 눈에 띄는 명확한 사실은 스키마 안에 `CollectionMembership` type이 포함되어 있다는 것입니다. collection memberships table은 product와 collection들 간의 다대다 관계를 표현하기 위해 사용됩니다.

자, 마지막 문장을 다시 읽어봅시다. *product와 collection들 간의* 관계입니다.
비즈니스 관점에서는 collection membership 그 자체가 하는 것이 아무 것도 없습니다.
그것은 그저 구현의 세부사항일 뿐입니다.

이것은 collection membership이 우리 API에 포함시킬 필요가 없다는 것을 의미합니다.
우리 API는 실질적인 비즈니스 도메인 관계만을 제품 스키마에 직접적으로 노출시키길 원합니다.

우리가 collection membership을 뺀다면, 높은 수준에서의 설계는 다음과 같아집니다.

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

훨씬 낫네요.

*규칙 #2: API를 설계할 때, 구현상의 디테일은 노출시키지 마세요.*

### Representing Collections

이 API 디자인은 여전히 한 가지 중요한 결함을 갖고 있습니다. 비즈니스 도메인에 대한 이해가 없다면, 잘 느껴지지 않을 것입니다.

이 설계에서, 우리는 AutomaticCollections와 ManualCollections를 두 개의 다른 type으로 모델링했습니다. 그리고 이 두 type은 각각 공통적으로 Collection interface를 구현합니다. 직관적으로 보면 타당해 보입니다: 그들 사이에는 많은 공통 field가 존재하지만, 여전히 그들의 관계나 동작 방식은 많이 다르기 때문입니다. (AutomaticCollections는 규칙을 갖고 있죠.)

비즈니스 모델 관점으로부터, 이러한 차이점은 기본적으로 구현상의 디테일일 뿐입니다. collection의 정의된 행동은 제품을 그룹핑하는 것이며, 제품들을 골라내는 방식은 부차적입니다. 우리는 추후에 우리 구현을 확장하기 위해 제품을 선택하는 제 3의 방식(기계학습?)을 허용하거나, 복합적인 방법(규칙과 상품 수동 추가)도 허용할 수 있지만 *이는 여전히 collections일 뿐입니다*. 현재 복합적인 방법이 허용되지 않는 것이 구현 실패가 아니냐 하실지도 모릅니다. 여태 한 이야기를 종합해서 말하자면 API의 모양은 다음과 같아져야 할 것입니다.

```graphql
type Collection {
  [CollectionRule]
  Image
  [Product]
}

type CollectionRule { }
```

정말 멋져요! 이 시점에서 여러분이 걱정할 수도 있는 것은, 우리가 ManualCollections도
규칙을 갖고 있는 것처럼 만들었다는 것이겠네요. 하지만, 이 관계는 리스트라는 것을 기억하세요.

우리의 새 API 설계에서 "ManualCollections"는 단순히 비어있는 규칙 리스트를 가진 collection일 뿐입니다.

### Conclusion

이 정도의 추상화 단계에서 가장 최고의 API 설계를 선택하려면, 모델링하고 있는 도메인에 대해 깊게 이해하고 있어야 합니다. 튜토리얼 환경에서는 구체적인 주제에 대한 깊은 이해를 제공하기는 힘듭니다. 하지만, collection 디자인은 추론이 가능할 정도로 충분히 간단합니다.

collection에 대해 깊이 이해하고 있지 않더라도, 실제로 모델링하고 있는 비즈니스 영역에 대해서는 깊은 이해가 필요합니다. 여러분의 API를 설계할 때, 이런 어려운 질문을 스스로 던져보고 맹목적으로 구현하지 않는 것은 매우 중요한 일입니다.

관련 레퍼런스 중에서, 좋은 API는 사용자 인터페이스를 모델링하지 않는다고 합니다. 구현과 UI는 모두 여러분의 API 설계에 있어 제공과 입력을 위해 사용될 수 있지만, 설계의 가장 중요한 동인은 항상 비즈니스 도메인이어야 합니다.

더 중요하게는, 기존의 REST API를 복사하지 않는 것이 좋습니다. REST와 GraphQL의 설계 원칙은 다른 영역입니다. 그러므로 여러분의 REST API에 동작하는 것이 GraphQL에서도 좋은 선택이 될 것이라 가정하시면 안됩니다.

가능한 여러분의 짐을 최대한 내려놓고, 처음부터 시작하시길 바랍니다.

*Rule #3: 구현도 UI도 기존 API도 아닌, 비즈니스 도메인에 맞춰 API를 설계하세요.*

## Step Three: Adding Detail

이제 우리는 type을 모델링하기에 깔끔한 구조를 갖게 되었습니다. 우리는 field를 추가할 수 있고, 다시 세부적인 수준에서 시작할 수 있습니다.

세부사항을 추가하기 전에, 지금 시점에 이것을 추가하는 게 맞는지 스스로에게 물어봅시다. 데이터베이스 컬럼, 모델 속성, 또는 REST 속성이 이미 기존에 있다는 이유로 graphQL 스키마에 추가할 필요는 없습니다.

실질적인 요구와 활용 사례에 고려해 스키마 요소(field, argument, type 등)를 추가하는 게 좋습니다. GraphQL 스키마는 요소를 추가함으로써 쉽게 발전할 수 있습니다. 하지만, 요소를 제거하거나 변형하는 것은 변화를 깨뜨리는 것이고, 일을 훨씬 더 어렵게 만들 수 있습니다.

*규칙 #4: 필드를 제거하는 것보다 추가하는 것이 더 쉽습니다.*

### Starting point

새로운 구조에 맞게, 조정된 단순한 필드를 되돌려 봅시다.

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

이제 우리는 완전히 다른 설계 문제를 갖게 되었습니다. 우리는 위에서 아래까지 순서대로 필드를 고쳐갈 것입니다.

### IDs and the `Node` Interface

Collection type에서 가장 첫번째 필드는 ID 필드입니다. 이 필드는 꽤 정상적입니다. 이 ID는 우리가 API에서 collection들을 식별하기 위해 사용하는 데에 필요합니다. 특히, collection들을 수정하거나 삭제해야할 때 도움이 될 것입니다. 그러나 우리 설계에는 어떤 한 부분이 빠져있습니다. 그것은 `Node`라는 인터페이스입니다. 이것은 대부분의 스키마에서 이미 존재하는, 매우 공통적으로 사용되는 인터페이스입니다. 생긴 것은 다음과 같습니다.

```graphql
interface Node {
  id: ID!
}
```

클라이언트에게 이 객체는 ID가 주어져야 한다는 것을 알려줍니다. 이 ID는 클라이언트가 정확하고 효율적으로 로컬 캐시나 다른 트릭들을 관리할 수 있도록 해줍니다. 대부분의 식별 가능한 비즈니스 객체(products, collections 등)는 이 `Node`를 구현해야 합니다.

우리 설계의 시작은 이제 이렇게 됩니다.

```graphql
type Collection implements Node {
  id: ID!
}
```

*규칙 #5: 주요한 비즈니스 객체 type은 항상 `Node`를 구현해야 합니다.*

### Rules and Subobjects

우리는 collection 필드 중에서 `rules`, `rulesApplyDisjunctively`라는 두 가지 필드를 살펴볼 것입니다. 첫번째는 rules의 리스트라는 꽤 정직한 이름입니다. 리스트 그 자체와 그 안의 요소 모두 non-null로 표시된다는 것을 알아두세요. GraphQL은 `null`과 `[]` 그리고 `[null]`을 구별하기 때문에 괜찮습니다.
수동적인 collection을 위해, 우리는 이 리스트를 비워둘 수 있습니다. 하지만 그것이 null이 되거나, null 값을 포함할 수는 없습니다. (<옮긴이> `null`, `[null]`은 안되고, `[]`은 가능합니다.)

*Protip: List-type 필드는 거의 항상 non-null 요소를 가지는 non-null 리스트입니다. 만약 여러분이 nullable한 리스트를 원한다면, 리스트에 빈 리스트와 null 리스트를 구별할 수 있도록 하는 \*'시맨틱 값'이 있어야 한다는 것을 확실히 해두세요.*

<sub>🧚‍♀️ <옮긴이> 시맨틱 값(semantic value): 의미론적인 값이란, 함수나 값이 어떤 것인지 설명하지 않아도 그 이름만으로 어떤 역할, 어떤 의미를 가지는 지 알아볼 수 있는 값입니다. ex) type NullableList { ... }.</sub>

두번째 필드는 살짝 이상합니다. 이것은 규칙이 불분명하게 적용되는지 아닌지 알려주는 boolean 타입의 필드입니다. 이 또한 non-null 필드지만 여기에는 문제가 있습니다. 수동적인 collection에서는 어떤 값이 이 필드에 들어와야 할까요?

fales나 true로 두는 것 어떤 것도 잘못된 방법처럼 느껴집니다. 그렇다고 필드를 nullable로 만드는 것 또한, 일종의 이상한.. 3개주 지역의 깃발처럼 되어(null, false, true) 자동적인 collection을 다루기에도 어색해집니다. 우리가 이 문제를 해결하고 있는 동안, 언급할 만한 다른 하나의 사실이 있습니다. 이 두 필드 사이에는 복잡하지만 명확한 관계가 있다는 것입니다. 이것은 의미론적으로 사실이며, 우리가 공통된 접두사를 붙였다는 것에서도 알 수 있습니다. 그렇다면, 어떤 방법으로 스키마에서 이 관계를 드러낼 수 있을까요?

사실 우리는 기본적인 구현으로부터 멀리 벗어나, 직접적인 모델과는 다른 새로운 graphQL type을 도입함으로써 이런 문제를 단번에 해결할 수 있습니다. 그 type을 `CollectionRuleSet`이라고 합시다. 이것은 여러분이 값이나 행위가 연결되어 있는, 근접한 관계에 있는 필드들의 집합을 갖고 있을 때 종종 사용됩니다. 두 필드를 우리가 만든 type으로 그룹핑함으로써, 우리는 깔끔한 시맨틱 식별자를 제공할 수 있고 nullability와 관련해서 가졌던 모든 문제를 해결할 수 있습니다. 수동적인 collection에서는, 우리는 ruleSet을 null로 둡니다. 그럼 boolean 필드는 non-null로 남을 수 있습니다. 다음처럼 설계할 수 있습니다.

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

*Protip: list 같이, boolean 필드도 거의 항상 non-null입니다. 만약 nullable한 boolean을 원한다면, null, false, true의 세 가지 상태를 구별할 수 있는 시맨틱 값을 포함시켜야 합니다. 그리고 이런 행위가 더 큰 설계 결함을 일으키지 않는 지도 확인해야 합니다.*

*규칙 #6: 근접한 관계를 가진 필드는 하위-객체로 그룹핑하세요.*

### Lists and Pagination

다음은 `products` 필드를 볼 차례입니다. 보기에는 안전해 보입니다. `CollectionMembership`을 제거했을 때, 이미 이 relation을 고친 뒤지만, 여기에는 또 다른 문제가 있습니다.

현재 이 필드는 products 배열을 반환하도록 정의되어있으나, collection들은 몇 십, 몇 천의 products를 가질 수 있습니다. 그리고 그 모든 것들을 하나의 배열로 모으는 것은 아주 큰 비용을 치뤄야 하며 비효율적일 수도 있습니다. 이런 상황 때문에, graphQL은 lists pagination이라는 것을 제공합니다.

여러 객체를 반환하는 필드나 릴레이션을 구현할 때마다, 항상 스스로에게 그 필드를 페이지네이션할 수 있는지 묻길 바랍니다. 필드에 얼마나 많은 객체가 들어올 수 있을까요? 최대한으로 생각하는 수량은 어느 정도인가요?

필드를 페이지네이션하기 위해서는 페이지네이션 솔루션을 먼저 구현해야합니다. 이 튜토리얼은 [Connections](https://graphql.org/learn/pagination/#complete-connection-model)을 사용합니다. 이것은 [Relay Connection spec](https://facebook.github.io/relay/graphql/connections.htm)에 정의되어 있습니다.

이 경우, 우리 설계에서 products 필드를 페이지네이션하는 것은 그것의 정의를 `products: ProductConnection!`로 바꾸는 것 만큼이나 간단합니다. 여러분이 connections를 구현한다고 가정하면, type은 다음과 같아집니다.

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

*규칙 #7: 항상 list 필드가 페이지네이션될 수 있는지 아닌지 확인하세요.*

### Strings

다음은 `title` 필드입니다. 합리적으로 괜찮은 방법입니다. 간단한 문자열이고, non-null로 표시됩니다. 왜냐하면, collection들은 각각 title을 가져야하니까요.

*Protip: boolean, list와 마찬가지로, graphQL은 빈 문자열과 null을 구분합니다. 그러니 nullable한 문자열을 원할 땐, 비어있는(`""`) 것과 표현되지 않는 것(`null`) 사이에 시맨틱한 차이점이 있는지 확인해보세요. 빈 문자열은 "적용 가능하지만 채워지지 않는다"는 의미로 생각할 수 있으며, null 문자열은 "적용할 수 없다"는 것으로 종종 생각할 수 있습니다.*

### IDs and Relations

이제 `imageId` 필드로 왔습니다. 이 필드는 우리가 REST 설계를 GraphQL에 적용할 때 발생하는 클래식한 예시입니다. REST API에서 다른 객체를 연결하기 위해 response에 다른 객체 ID를 포함하는 것은 꽤 흔한 일입니다.

객체에 대한 다른 정보를 얻기 위해, ID를 제공하거나 클라이언트에게 (input, output을 주고 받는) 왕복을 하도록 강요하는 대신에, 직접적으로 graph에 객체를 포함하는 것이 좋습니다. 그게 바로 GraphQL이 존재하는 목적이죠. REST API에서는 이 패턴은 종종 실용적이지 않을 수 있습니다. 객체의 사이즈가 클 때는 response의 크기가 상당히 증가할 수 있기 때문입니다. 그러나 GraphQL에서는 괜찮습니다. 왜냐하면 모든 필드는 반드시 명시적으로 질의되거나, 서버가 이것을 반환하지 않을 것이기 때문입니다.

일반적인 규칙으로서, 설계에서 ID 필드들은 오직 해당 오브젝트 자신의 ID 필드여야합니다. 다른 객체의 ID 필드를 가질 때는, 그것은 아마도 그 객체의 레퍼런스가 되어야할 것입니다. 이 규칙을 우리 스키마에 적용하면, 다음과 같습니다.

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

*규칙 #8: 다른 ID 필드들을 사용하기보다는, 항상 객체 레퍼런스를 사용하세요.*

### Naming and Scalars

우리의 `Collection` type에서 마지막 필드는 `bodyHtml`입니다. collections가 구현되는 방법에 친숙하지 않는 사용자에게는 이 필드의 역할이 완전히 명확하지는 않을 것입니다. 이것은 특정 collection에 대한 body description입니다. 우리가 이 API를 더 나은 것으로 만들기 위해 첫번째로 할 수 있는 것은 이 필드의 이름을 그저 `description`으로 바꾸는 것입니다. 그게 훨씬 더 명확한 이름같습니다.

*규칙 #9: 구현 또는 기존 API에서 그 필드가 무엇으로 불렸는지에 근거하기 보다는 좀 더 명확한 필드 이름을 선택하세요.*

다음으로 우리는 이것을 non-nullable로 만들 수 있습니다. title 필드에 대해서 말했던 것처럼, 필드가 null이 되는 것과, 단순히 빈 문자열인 것을 구분하는 것은 타당한 일이 아닙니다. 그래서 우리는 이것을 API에는 노출시키지 않을 것입니다. 데이터베이스 스키마가 컬럼에서 값이 null을 가지도록 허락한다 해도, 우리는 구현 단에서 이것을 숨길 수 있습니다.

마지막으로, 우리는 `String`이 이 필드에 실질적으로 맞는 type인지 고려해봐야 합니다. GraphQL은 꽤 괜찮은 스칼라 타입의 집합(`String`, `Int`, `Boolean`, 등)을 내장하고 있습니다. 하지만 우리 스스로도 스칼라 타입을 정의할 수 있습니다. 이것이 그 기능을 사용하는 주요한 이유이기도 합니다. 대부분의 스키마들은 스스로 추가적인 스칼라 집합을 정의합니다. 이것은 클라이언트에게 시맨틱 값과 추가적인 context를 제공합니다. 문제의 문자열이 유효한 HTML이어야 하는 경우, 여기에(잠재적으로는 다른 곳에도) 사용자 정의 `HTML` 스칼라를 정의하는 것이 타당할 것입니다.

여러분이 스칼라 필드를 추가할 때마다, 이미 존재하는 사용자 정의 스칼라 리스트를 확인하는 것이 좋습니다. 새로 만들기 보다는, 이미 존재하는 것 중에 더 잘 맞는 게 있을 수 있으니까요. 만약, 여러분이 필드를 추가하고 있고, 여러분이 생각하기에 새로운 사용자 정의 스칼라가 더 적당하다면, 팀과 상의하여 올바른 개념을 파악하고 있는지 확인해보는 것이 좋습니다.

*규칙 #10: 무언가 구체적인 시맨틱 값을 노출할 때는 사용자 정의 스칼라 타입을 사용하세요.*

### Pagination Again

지금까지 핵심적인 `Collection` type의 모든 필드를 살펴봤습니다. 다음 객체는 `CollectionRuleSet`입니다. 꽤 간단한 객체죠. 여기서의 문제는 그저 rules의 리스트가 페이지네이션 되어야하는가 말아야하는가일 뿐입니다. 이 경우에는 기존의 배열이 더 타당합니다. rules 리스트를 페이지네이션하는 것은 오버하는 걸 수도 있습니다.

대부분의 collection들은 적은 규칙만을 가지기 때문입니다. 그리고 collection이 큰 rule set을 가지는 것은 좋은 활용 사례가 아닙니다. 규칙이 십여 가지가 된다면 products를 수동적으로 추가해야하거나, 그 collection이 옳은지 재고해봐야 하는 지표가 될 것입니다.

### Enums

스키마의 마지막 type `CollectionRule`까지 왔습니다. 각각의 규칙은 일치하는 컬럼(예: 제품 제목), 관계 유형(예: 같음), 사용을 위한 실질적인 값(예: "Boots(부츠)")으로 구성됩니다(<옮긴이> 제품제목 = 부츠). 그리고 이것은 `condition`(조건)이라고 혼동되어 불리기도 합니다. 마지막 필드는 이름을 `column`으로 다시 지을 수 있습니다. 하지만, column은 데이터베이스 특유의 용어이며 우리는 GraphQL로 일하고 있습니다. 그러므로 `field`가 좀 더 나은 선택이 되겠네요.

Type에 따라서, `field`와 `relation`은 모두 내부적으로는 열거형(enumeration)으로 구현될 수 있습니다 (선택한 언어에 enum이 있다고 가정할 경우). 다행히 GraphQL은 enum도 갖고 있습니다! 그러므로 우리는 두 필드를 enum으로 전환할 수 있습니다. 우리의 복잡한 스키마 설계는 이제 다음과 같아집니다.

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

*규칙 #11: 오직 특정한 값의 집합만을 취하는 필드에는 enum을 사용하세요.*

## Step Four: Business Logic

우리는 이제 collection들을 위한, 작지만 잘 설계된 GraphQL API를 갖고 있습니다. 우리가 아직 다루지 않은 collection들에 대한 수많은 세부사항이 있습니다 - 제품을 순서대로 정렬하거나 퍼블리싱 하는 등의 문제를 다룰 때, 이 기능을 실질적으로 구현하려면 훨씬 더 많은 필드가 필요할 지도 모릅니다 - 하지만 이러한 필드들은 모두 같은 설계 패턴을 따른다는 규칙이 있습니다. 그러나, 아직 몇 가지 더 들여다 봐야할 것들이 있습니다.

이 섹션을 위해, API를 사용할 가상의 클라이언트로부터 이것을 사용해야 할 동기를 얻으며 시작하는 것이 좋겠습니다. 그러므로 우리와 함께 일하는 클라이언트 개발자가 무언가 구체적인 것을 알고 싶어한다는 상상을 해봅시다: '주어진 product가 collection의 멤버인가 아닌가'

물론 그 클라이언트는 이미 기존의 API를 통해 이 문제에 대한 답을 알 수 있을 것입니다: 우리가 products 집합을 완전히 collection 안에 노출시켰으므로, 클라이언트는 그저 product를 찾고, 그것을 반복시키며(iterate 하여) 확인해보면 됩니다.

그러나 이 방법에는 두 가지 문제가 있습니다. 첫번째, 비효율적입니다. collection은 수백만 개의 product를 가질 수 있습니다. 그리고 클라이언트가 product를 가져오고(fetch), 반복시키는 것은 매우 느린 일입니다. 두번째, (아주 큰 문제죠) 이것은 클라이언트로 하여금 코드를 쓰도록 요구합니다. 이 부분은 설계 철학에 있어 아주 중요한 부분입니다. 서버는 항상 어떤 비즈니스 로직이든 단 하나의 공급자가 되어야 합니다. API는 거의 항상 하나의 클라이언트 이상에게 서비스하기 위해 존재합니다. 그리고 만약 이 클라이언트들 각각이 같은 로직을 구현해야 한다면, 이것은 오류를 수반하는 추가적인 작업과 공간(메모리)과 함께 코드의 중복을 야기합니다.

*규칙 #12: API는 데이터가 아닌 비즈니스 로직을 제공해야 합니다. 복잡한 계산은 클라이언트 여러 곳에서가 아닌, 서버 한 곳에서 처리해야 합니다.*

우리 클라이언트가 이 API를 활용하는 사례로 돌아가봅시다. 가장 최고의 정답은 이 문제를 해결하는 데에만 특정하게 사용될 새로운 필드를 제공하는 것입니다. 다음과 같습니다:

```graphql
type Collection implements Node {
  # ...
  hasProduct(id: ID!): Boolean!
}
```

이 필드는 product의 ID를 취합니다. 그리고 서버에서 이 product가 collection에 있는지 없는 지 결정하는 대로 boolean을 반환합니다. 이미 존재하는 `products` 필드로부터 일종의 중복된 데이터를 만드는 것과는 상관 없습니다. GraphQL은 오직 클라이언트가 명시적으로 요청하는 것만 반환합니다. 따라서, REST와는 달리 부가적인 필드를 생성하는 데에 어떠한 비용도 요구하지 않습니다. 클라이언트는 추가적인 필드를 질의하는 것 이상으로는 어떤 코드도 쓸 필요 없습니다. 필요한 것은 ID와 boolean 뿐입니다.

그러나 한 가지 경고가 따릅니다: 우리가 비즈니스 로직을 제공한다는 상황이 raw data(원시 데이터)를 제공해서는 안된다는 것을 의미하지는 않습니다. 만약 필요하다면, 클라이언트는 스스로 비즈니스 로직을 처리할 수 있어야 합니다. 클라이언트가 원하는 모든 로직을 예측할 수는 없습니다. 그리고 클라이언트가 추가적인 필드를 요청할 수 있는 쉬운 채널이 항상 존재하는 것도 아닙니다 (하지만 그러한 채널이 존재하도록 노력해야 합니다).

*규칙 #13: 비즈니스 로직이 있을 지라도, raw data(원시 데이터)도 함께 제공하세요.*

마지막으로, 비즈니스 로직 필드가 전체적인 API 모양에 영향을 주지 않도록 주의하세요. 비즈니스 도메인의 데이터는 여전히 핵심적인 모델입니다. 만약 비즈니스 로직이 정말 맞지 않는다고 생각한다면, 그것은 근본적인 모델이 적합하지 않다는 신호가 될 수 있습니다.

## Step Five: Mutations

마지막으로 GraphQL 스키마 설계에서 빠진 부분은 실제로 값을 변형시킬 수 있는 기능입니다: collection 또는 그와 관련된 부분을 생성, 수정, 제거하는 것을 말합니다. 스키마와 마찬가지로, 우리는 높은 수준의 관점에서부터 시작해야 합니다: 이 경우, 우리는 단지 구체적인 input, output을 고려할 필요 없이 mutation을 구현하길 원합니다. CRUD(Create, Read, Update, Delete) 패러다임을 따라서 그저 단순하게 `create`, `delete`, `update` mutation을 가질 수 있습니다. 괜찮은 시작이지만, 적절한 GraphQL API에는 충분하지 않습니다.

### Separate Logical Actions

만약 단순히 CRUD를 고집한다면, 우리가 제일 먼저 알 수 있는 것은 `update` mutation이 빠르게 거대해질 것이라는 겁니다. 해당 mutation에는 title과 같은 간단한 스칼라 값을 수정하는 역할만 있는 게 아닙니다. 퍼블리싱/언퍼블리싱, collection 내의 product들을 추가/제거/재정렬, 자동적인 collection의 규칙을 변형하는 등 복잡한 행위(action)를 수행하는 데에도 쓰일 수 있습니다. 이로 인해, 서버에서는 구현하기 어렵고 클라이언트는 이 mutation이 어떤 행위를 하는지 추론하기 어렵습니다. 대신, 우리는 좀 더 알맹이 있는 논리적인 행위(action)로 mutation을 분할하기 위해서 GraphQL의 이점을 취할 수 있습니다. 가장 먼저, 다음과 같은 mutation 리스트를 생성하여 퍼블리싱과 언퍼블리싱을 분리할 수 있습니다.

- create
- delete
- update
- publish
- unpublish

*규칙 #14: 리소스 각각의 논리적 행위(action)에 맞는 각각의 mutation을 써보세요.*

### Naming the Mutations

CRUD 동사를 기본값으로 시작하지 마세요. 데이터베이스 문장은 CRUD를 이용해 잘 
표현할 수 있지만, 이런 것은 API 사용자에게 숨겨야 하는 세부 구현사항입니다.
CRUD 동사가 비즈니스 도메인을 제대로 설명하기는 힘듭니다. 대신에 여러분의 도메인, 맥락,
mutation의 동작을 고려하세요. 이런 의미를 좀 더 적절하게 설명할 수 있는 단어를 사용하세요.
예를 들어, 컬렉션의 게시 철회가 기본 결과라면 `collectionDelete` 대신 `collectionUnpublish`라고 
이름 지어주세요.

### Manipulating Relationships

`update` mutation은 여전히 너무 많은 역할을 갖고 있습니다. 그러니 이것을 분리하는 작업을 계속해봅시다. 하지만, 우리는 이 행위들(actions)을 개별적으로 다룰 것입니다. 다른 차원으로부터 생각해보는 것 또한 가치 있는 일이 될테니까요: 객체의 관계(예: 일대다, 다대다) 조작. 우리는 이미 'ID 사용 vs 객체 끼워넣기(embedding)', 그리고 API 조회에 있어 '페이지네이션 vs 배열' 사용을 고려해봤습니다. 여기에는 그 관계들을 변형시킬 때 다뤄야 할 비슷한 이슈가 있습니다.

products와 collections 사이의 관계에 대해, 우리가 광범위하게 고려해 볼 여러 스타일들이 있습니다.

- 전체적인 관계(예: `products: [ProductInput!]!`)를 update mutation에 끼워넣는 것은 CRUD 기본 스타일입니다. 하지만 리스트가 커진다면 빠르게 비효율적으로 변합니다.
- "delta" 필드(예: `productsToAdd: [ID!]!`와 `productsToRemove: [ID!]!`)를 update mutation에 끼워넣은 것은 전체 리스트 대신 변경된 ID들만 알려주면 되지만, 여전히 행위들(actions)을 함께 묶어두므로 훨씬 더 비효율적입니다.
- 전체적으로 이것을 개별적인 mutation으로 (`addProduct`, `removeProduct` 등) 분리하는 것은 가장 강력하고 유연할 뿐만 아니라 가장 잘 동작합니다.

마지막 옵션이 일반적으로 가장 안전합니다. 특히 이런 mutation이라면 어쨌든 일반적으로 뚜렷한 논리 행위(logical action)가 될 것이기 때문입니다. 여기에는 고려해야될 많은 요인이 있습니다.

- 그 관계가 크거나 혹은 페이지네이션되는 것인가요? 만약 그렇다면, delta 필드 또는 개별적인 mutation가 적절할 수 있기에, 전체 리스트를 끼워넣는 것은 확실히 실용적이지 않습니다. 그러나 만약 관계가 항상 작다면 (특히 이것이 일대일 관계라면), 끼워넣기는 가장 간단한 선택이 될 수 있습니다.
- 그 관계가 정렬되어 있나요? product-collection 관계는 정렬됩니다. 그리고 수동적으로 재정렬하죠. 순서는 embedded 리스트 또는 개별적인 mutation에 의해 자연스럽게 정렬됩니다 (`reorderProducts` mutation을 추가하면 됩니다). 하지만 delta 필드에서는 옵션이 아닙니다.
- 그 관계는 의무적인가요? Products와 collection은 모두 관계를 벗어나 스스로 존재할 수 있습니다. 그들 스스로 create/delete의 생애주기를 거치면서요. 만약 그 관계가 의무적이라면(예를 들어, collection 내에 product가 반드시 존재해야 하는 경우), 개별적인 mutation을 강력하게 제안할 수 있습니다. 그 행위(action)는 단지 그 관계를 수정하는 것뿐만 아니라 실제로 product를 *생성*하니까요.
- 양쪽에서 ID를 가지나요? collection-rule의 관계는 의무적입니다 (rule은 collection 없이는 존재할 수 없기 때문입니다). 하지만, rule은 ID조차도 가지지 않습니다. rule은 명확히 collection에 포함되고 그 관계가 작기 때문에, 여기서 리스트를 끼워넣는 것(embedding)은 실제로 나쁘지 않은 선택입니다. 그 밖에 다른 선택은 rule이 개별적으로 식별될 수 있도록 요구하는 것입니다(<옮긴이> rule마다 ID를 가지도록 요구)만 이것은 과한 것같네요.

*규칙 #15: 관계를 변형시키는 것은 정말 복잡한 작업입니다. 그리고 짧고 분명한 규칙으로 요약하는 것도 쉽지 않습니다.*

이 모든 것을 함께 섞는다면, collection에 대해 우리는 다음의 mutation 리스트를 작성할 수 있습니다.

- create
- delete
- update
- publish
- unpublish
- addProducts
- removeProducts
- reorderProducts

Products는 관계가 크고 순서가 있는 것이기 때문에 그들 자신의 mutation으로 분리했습니다. Rule은 관계가 작고 ID를 가지지 않아도 될 만큼 충분히 작기 때문에 인라인으로 남겨두었습니다.

마지막으로, 우리는 product mutation들이 products 집합에 영향을 준다는 사실을 알 수 있습니다 (예: "addProduct"가 아닌 "addProducts"). 클라이언트에게도 편리합니다. 그 관계를 조작할 때, 일반적으로 사용되는 경우는 한 번에 두 개 이상의 product를 추가, 제거 또는 재정렬하는 것이기 때문입니다.

*규칙 #16: 관계에 대한 개별적인 mutation을 작성할 때, 그 mutation이 여러 개의 요소를 한 번에 작업하는 데 유용한지 고려해보세요.*

### Input: Structure, Part 1

이제 우리가 작성하고 싶은 mutaion이 어떤 것인지 알 것 같습니다. 그러니 지금부터는 mutation의 input 구조가 어떻게 생겼는지 이해해봅시다. 만약 여러분이 공개적으로 사용 가능한 실제 products 스키마를 본 적이 있다면, 많은 mutation이 그들의 인자를 모두 단일 전역 input type으로 정의한다는 것을 알아챌 수 있습니다: 이런 패턴은 기존 고객의 요구일 수도 있으나 새로운 코드에는 더 이상 필요하지 않습니다. 우리는 이런 패턴을 무시하도록 합시다.

많은 간단한 mutation들은 이 단계를 간단하게 만들기 위해 하나 또는 소수의 ID들이 필요합니다. collections 중에서, 우리는 빠르게 다음의 mutation 인자들을 생각해낼 수 있습니다.
- `delete`, `publish` 그리고 `unpublish`는 모두 단일 collection ID를 필요로 합니다.
- `addProducts`와 `removeProducts`는 모두 product IDs뿐만 아니라, collection ID 또한 필요로 합니다.

이렇게 하면, 설계를 위한 오직 3가지의 "복잡한" input만이 남습니다:
- create
- update
- reorderProducts

create부터 시작해봅시다. 아주 단순한 input은 우리가 처음에 만들었던 원래의 단순한 collection 모델과 비슷해보이기도 합니다. 그러나 우리는 이미 그것보다 잘할 수 있습니다. 마지막 collection 모델과 위에서 논의했던 관계(relationships)에 근거해서 우리는 다음과 같은 것부터 시작할 수 있습니다:

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

먼저 네이밍을 빠르게 생각해봅시다: 우리는 모든 mutatation의 이름을 `collection<Action>`와 같은 형식으로 지었습니다. `<action>Collection`이 좀 더 영어에 친숙한 표현이긴 하지만요. 불행히도 GraphQL은 mutation을 조직화하거나 그룹핑하는 함수를 제공하지 않습니다. 따라서 이를 해결하려면 이것을 알파벳으로 문자화해야만 합니다. 핵심 type을 먼저 입력하면, 관련된 모든 mutation 그룹이 최종 리스트에 함께 표시됩니다.

*규칙 #17: 알파벳 그룹핑(예: `cancelOrder` 대신 `orderCancel` 사용)을 위해 변형시키는 객체를 mutation의 접두사로 만드세요.*

### Input: Scalars

이 초안은 단순한 접근법보다는 훨씬 더 낫습니다만, 여전히 완벽하지는 않습니다. 특히, `description`의 input 필드가 여전히 많은 문제를 갖고 있습니다. non-null `HTML` 필드는 collection의 description의 output에는 잘 맞지만, 몇 가지 이유로 input에는 적절하지 않습니다. 첫번째로, `!`은 output이 null이 될 수 없음을 명시하는 데 반해, input 역시 똑같이 null이 될 수 없음을 의미하지는 않습니다. 대신, 그 필드가 "필수적인(required)" 것인지에 대한 개념을 설명하는 것에 가깝습니다. 필수적인 필드는 클라이언트가 API를 요청하기 위해 반드시 제공해야 하는 필드를 의미합니다. 그리고 이것은 `description`에는 잘 맞지 않습니다. 우리는 클라이언트가 description을 제공하지 않는다고 해서 collection을 생성하지 못하도록 하는 것을 원하지 않습니다 (똑같이, 그들이 쓸 데 없는 `""`을 쓰도록 강제하고 싶지도 않습니다). 그러므로 우리는 `description`을 non-required로 만들어야 합니다.

*규칙 #18: 만약 mutation을 만들 때, 의미론적으로 필요한 것이라면 input 필드를 필수적인 필드로 만드세요.*

다른 이슈는 `description`이 그 자체로 type이라는 것입니다. 이미 강한 타입이기 때문에(`String` 대신에 `HTML`을 쓰는 것처럼) 직관적이지 않아 보입니다. 그리고 우리는 지금까지 강한 타이핑을 해왔습니다. 하지만 다시, input은 조금 다르게 동작합니다. input에 대한 강한 타입의 유효성은 "사용자공간"에서 코드가 실행되기 전에 GraphQL 단에서 발생합니다. 이것은 현실적으로 클라이언트가 두 개의 차원에 대한 오류를 다뤄야 함을 의미합니다: GraphQL 차원의 유효성 오류, 그리고 비즈니스 차원의 유효성가 있습니다 오류(예: 현재 저장 공간의 한계로 생성할 수 있는 collection이 제한될 수 있습니다). 이 단계들을 간단하게 만들려면, 클라이언트가 미리 검증하기 어려울 때는 의도적으로 약한 타입의 input 필드를 선택해야 합니다. 이것은 비즈니스 로직 측에서 모든 유효성 검사를 가능케 하며, 클라이언트에게는 오직 한 공간에서의 오류만을 다룰 수 있게 합니다.

*규칙 #19: 형식이 명확하고, 클라이언트 측에서 유효성 검사를 하기에 복잡할 것 같다면, input에 좀 더 약한 타입(예: `Email` 대신 `String`)을 사용하세요. 그러면 서버는 애매했던 유효성 검사를 한 번에 할 수 있고, 클라이언트는 조금 더 간단하게 만들면서 단일 형식으로 한 곳에서 오류를 반환할 수 있습니다.*

하지만, 약한 타입이 모든 input에 적합한 것은 아니라는 것을 알려드리고 싶습니다. 우리는 여전히 rule input에는 `field`와 `relation` 값에 대해 강한 타입의 enums를 사용하고 있습니다. 그리고 이런 사례에서라면 여전히 `DateTime`과 같은 같은 특정한 input에는 강한 타이핑을 사용하고 있을 지도 모릅니다. 뚜렷한 요인은 클라이언트 측의 유효성 검사가 복잡하다는 것과 input의 형식이 모호하다는 것입니다. HTML은 잘 정의되어 있고 명확하지만, 유효성을 검사하기에는 살짝 복잡합니다. 반면에, 날짜와 시간을 문자열로 표현하는 방법에는 수백 가지의 방법이 있습니다. 그것들은 모두 합리적으로 간단하며,어 우리가 기대하는 것이 어떤 형식인지 구체화하기 위해 강한 스칼라 타입으로부터 얻는 이점이 있습니다.

*규칙 #20: 형식이 모호하고, 클라이언트 측에서의 유효성 검사가 간단해보일 땐, input에 더 강한 타입(예: `String` 대신에 `DateTime`)을 사용하세요. 이는 명확성을 제공하고, 클라이언트가 좀 더 엄격하게 input 값을 통제할 수 있도록 만듭니다(예: free-text 필드 애신 날짜 선택 위젯을 사용하는 등).*

### Input: Structure, Part 2

계속해서 update mutation을 보도록 합시다.

```graphql
type Mutation {
  # ...
  collectionCreate(title: String!, ruleSet: CollectionRuleSetInput, image: ImageInput, description: String)
  collectionUpdate(collectionId: ID!, title: String, ruleSet: CollectionRuleSetInput, image: ImageInput, description: String)
}
```

create mutation과 update mutation이 매우 비슷하다는 걸 눈치채셨을 겁니다. 하지만, 두 가지는 다르군요: update mutation에는 어떤 collection이 수정될 지 결정할 `collectionId`라는 인자가 추가됐고, 이미 생성된 collection은 제목을 갖고 있으므로, update에서 `title`은 더 이상 필수 인자가 아닙니다. 'required' 상태의 title을 제외하면, 이 mutation은 네 개의 중복된 인자를 갖습니다. 완전한 collections 모델은 그보다 더 많이 포함할 수 있습니다.

이런 mutation을 그냥 놔두자는 주장도 있었습니다. 그러나, '필수'를 처리해야 하는 문제에도 불구하고, 우리는 인자의 공통된 부분을 줄이기로 했습니다. 이것은 몇 가지 이점이 있습니다:
- 스키마가 이미 갖고 있는 단일 `Collection` type을 반영하며, collection 개념을 표현하는 '단일 input 객체'만을 다루면 됩니다.
- 클라이언트는 (공통 패턴을 가진) create, update 폼 사이에 코드를 공유할 수 있습니다. 그들은 같은 종류의 input 객체를 다루기 때문입니다.
- 몇 개의 최상위 인자만으로 Mutations이 깔끔해지고 읽기 쉬워집니다.

처리할 가장 큰 문제는 물론, collection이 생성될 때 그 제목이 필수적인지 아닌지 스키마에서는 확실하게 알 수 없다는 것이겠죠. 스키마는 결국 다음과 같아집니다.

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

*규칙 #21: 특정 필드에서 \*완화된 '필수' 조건이 필요하다 하더라도, 중복을 줄일 수 있도록 mutation input을 짜보세요.*

<sub>🧚‍♀ <옮긴이> 완화된 필수 조건(relaxing requiredness constraints): collectionUpdate의 title처럼, `!`는 붙지 않았지만 반드시 있을 것이라고 기대되는 필드를 말하는 것 같습니다.</sub>

위의 예시와 같이 `collectionUpdate` 함수는 두 개의 인자를 받습니다. `collectionId`는 업데이트할 컬렉션을 선택하고, `collection`은 업데이트할 데이터를 제공합니다. 
이에 대한 대안은 null이 될 수 있는 id 필드를 가진 하나의 `collection: CollectionInput!` 인자입니다. 
그러나 이 방법은 호출의 어떤 부분이 'select'과 관련이 있는지, 어떤 부분이 'update'와 관련이 있는지 판단하기 어렵게 만들기 때문에 권장하지 않습니다.

*규칙 #22: 업데이트 mutation에 있어 객체선택에 관련한 인자와 변경 데이터를 제공하는 인자는 분리해야 합니다.  객체 선택과 관련한 인자는 필터링 조건을 제외하면 null로 설정할 수 없어야 합니다.*

### Output

우리가 다룰 마지막 설계 문제는 mutatoin의 반환 값입니다. mutatoin은 성공하거나 실패할 수 있습니다. GraphQL은 쿼리 수준의 오류에 대해서는 확실하게 지원해주지만, 비즈니스 수준에서 mutation이 실패하는 것에 대해서는 그렇지 않습니다. 그러니 사용자보다는 클라이언트 측의 실수(예: 존재하지 않는 필드를 요청)에 대한 최상위 수준의 오류에 대비해야 합니다. 마찬가지로, 각 mutation은 유용한 다른 값들과 함께 사용자 오류에 대응하는 필드를 포함하는 "payload" type을 정의해야 합니다. Create라면 다음과 같이 표현할 수 있습니다:

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

mutatoin이 성공한다면 `userErrors`에서는 빈 리스트를, `collection` 필드에서는 새롭게 생성된 collection을 반환할 것입니다. mutation이 성공하지 않는다면 하나 이상의 `userErrors` 객체를, collection 필드에는 `null`을 반환할 것입니다.

*규칙 #23: mutation은 mutation payload에서 `userErrors` 필드를 통해 사용자/비즈니스 수준의 오류를 처리해야 합니다. 최상위 수준의 쿼리 오류 엔트리(The top-level query errors entry)는 클라이언트와 서버 수준의 오류에 대비해야 합니다.*

많은 구현에서, 이런 구조는 대부분 자동으로 제공됩니다. 여러분은 그저 `collection`의 반환 필드만 정의하면 됩니다.

update mutation에서는, 우리는 정확히 그와 똑같은 패턴을 따를 수 있습니다.

```graphql
type CollectionUpdatePayload {
  userErrors: [UserError!]!
  collection: Collection
}
```

`collection`이 null이 가능하다는 것에 주목해주세요. 이는 제공된 ID가 유효한 collection을 나타내지 않는다면, 반환할 collection이 없기 때문입니다.

*규칙 #24: 발생 가능한 모든 오류 케이스에서, 필드 값이 반드시 반환될 것이라는 확신이 들지 않는다면 mutation에 대한 대부분의 payload 필드는 null이 가능하도록 하는 게 좋습니다.*

## TLDR: The rules

- *규칙 #1: 구체적인 필드를 다루기 전에, 항상 객체들과 그 사이의 relation을 높은 수준에서 바라보는 것부터 시작하세요.*
- *규칙 #2: API를 설계할 때, 구현상의 디테일은 노출시키지 마세요.*
- *규칙 #3: 구현도 UI도 기존 API도 아닌, 비즈니스 도메인에 맞춰 API를 설계하세요.*
- *규칙 #4: 필드를 제거하는 것보다 추가하는 것이 더 쉽습니다.*
- *규칙 #5: 주요한 비즈니스 객체 type은 항상 `Node`를 구현해야 합니다.*
- *규칙 #6: 근접한 관계를 가진 필드는 하위-객체로 그룹핑하세요.*
- *규칙 #7: 항상 list 필드가 페이지네이션될 수 있는지 아닌지 확인하세요.*
- *규칙 #8: 다른 ID 필드들을 사용하기보다는, 항상 객체 레퍼런스를 사용하세요.*
- *규칙 #9: 구현 또는 기존 API에서 그 필드가 무엇으로 불렸는지에 근거하기 보다는 좀 더 명확한 필드 이름을 선택하세요.*
- *규칙 #10: 무언가 구체적인 시맨틱 값을 노출할 때는 사용자 정의 스칼라 타입을 사용하세요.*
- *규칙 #11: 오직 특정한 값의 집합만을 취하는 필드에는 enum을 사용하세요.*
- *규칙 #12: API는 데이터가 아닌 비즈니스 로직을 제공해야 합니다. 복잡한 계산은 클라이언트 여러 곳에서가 아닌, 서버 한 곳에서 처리해야 합니다.*
- *규칙 #13: 비즈니스 로직이 있을 지라도, raw data(원시 데이터)도 함께 제공하세요.*
- *규칙 #14: 리소스 각각의 논리적 행위(action)에 맞는 각각의 mutation을 써보세요.*
- *규칙 #15: 관계를 변형시키는 것은 정말 복잡한 작업입니다. 그리고 짧고 분명한 규칙으로 요약하는 것도 쉽지 않습니다.*
- *규칙 #16: 관계에 대한 개별적인 mutation을 작성할 때, 그 mutation이 여러 개의 요소를 한 번에 작업하는 데 유용한지 고려해보세요.*
- *규칙 #17: 알파벳 그룹핑(예: `cancelOrder` 대신 `orderCancel` 사용)을 위해 변형시키는 객체를 mutation의 접두사로 만드세요.*
- *규칙 #18: 만약 mutation을 진행할 때, 의미론적으로 필요한 것이라면 input 필드를 필수적인 필드로 만드세요.*
- *규칙 #19: 형식이 명확하고, 클라이언트 측에서 유효성 검사를 하기에 복잡할 것 같다면, input에 좀 더 약한 타입(예: `Email` 대신 `String`)을 사용하세요. 그러면, 서버가 모든 non-trivial한 유효성 검사를 한 번에 할 수 있고, 클라이언트는 조금 더 간단하게 만들면서 단일 형식으로 한 장소에서 오류를 반환할 수 있습니다.*
- *규칙 #20: 형식이 모호하고, 클라이언트 측에서의 유효성 검사가 간단해보일 땐, input에 더 강한 타입(예: `String` 대신에 `DateTime`)을 사용하세요. 이는 명확성을 제공하고, 클라이언트가 좀 더 엄격하게 input 값을 통제할 수 있도록 만듭니다(예: free-text 필드 애신 날짜 선택 위젯을 사용하는 등).*
- *규칙 #21: 특정 필드에서 \*완화된 '필수' 조건이 필요하다 하더라도, 중복을 줄일 수 있도록 mutation input을 짜보세요.*
- *규칙 #22: 업데이트 mutation에 있어 객체선택에 관련한 인자와 변경 데이터를 제공하는 인자는 분리해야 합니다.  객체 선택과 관련한 인자는 필터링 조건을 제외하면 null로 설정할 수 없어야 합니다.*
- *규칙 #23: mutation은 mutation payload에서 `userErrors` 필드를 통해 사용자/비즈니스 수준의 오류를 처리해야 합니다. 최상위 수준의 쿼리 오류 엔트리(The top-level query errors entry)는 클라이언트와 서버 수준의 오류에 대비해야 합니다.*
- *규칙 #24: 발생 가능한 모든 오류 케이스에서, 필드 값이 반드시 반환될 것이라는 확신이 들지 않는다면 mutation에 대한 대부분의 payload 필드는 null이 가능하도록 하는 게 좋습니다.*

## Conclusion

튜토리얼을 읽어주셔서 감사합니다. 이 튜토리얼을 통해 여러분이 어떻게 하면 좋은 GraphQL API를 설계할 지 명확한 아이디어가 생겼길 희망합니다.

API를 설계했다면, 이제는 그것을 구현해보세요!


## Appendix: Polymorphism

위 튜토리얼의 [Collections](#representing-collectionmemberships) 섹션에서, 다형성의 한 가지 예(원래 제안된 Collection 인터페이스)를 매우 간략하게 다루고 
그걸 API 관련된 개념을 담은 단일 비다형 Collection과 세부적인 구현을 담은 것으로 단순화해봤습니다.

도움이 되었길 바라지만, 이 예시는 GraphQL이 제공하는 다양한 다형 타입들
([Interfaces](https://graphql.org/learn/schema/#interfaces)와 [Unions](https://graphql.org/learn/schema/#union-types) 모두)과
그것들을 사용하는 방법을 다 다루고 있지는 못합니다.

GraphQL에서 다형성을 모델링하는 데는 크게 네 가지 다른 접근법이 있습니다.
그리고 각각은 서로 다른 사용 사례에 적합합니다.
이 부록에서는 기본 자습서보다 좀 더 넓은 범위의 예제를 사용하여 각 접근법을 차례로 설명합니다.

### No Shared Behaviour

아마도 가장 간단한 경우는 객체들이 의미적으로 연관되어 있지만 공유된 필드나 연관된 어떤 동작도 없는 경우일 것입니다.
예를 들어 Product 타입과 Collection 타입은 검색에 관련된 어떤 필드나 동작도 공유하지 않지만, 함께 검색 결과로 반환될 수 있습니다.
(다른 이유로 필드들을 공유할 수는 있겠지만요). 이런 경우, Union을 사용하세요.

```graphql
type Product {
  # some fields
}
type Collection {
  # some other fields
}
union SearchResult = Product | Collection
```

SearchResult union은 어떤 공통 행위도 암시하지 않고 타입 사이의 관계를 담아냅니다.
물론 검색 결과에만 해당하는 *공유 필드가 있는 경우* 스키마에 해당 필드를 담아낼 수 있습니다. 이 경우 Union은 최선의 선택은 아닙니다.

### Simple Shared Behaviour

또 다른 매우 간단한 경우는 단순한 동작을 공유하고 관련한 필드를 몇 개 공유하는 경우입니다. 예를 들어,
Product 및 Collection은 인터넷에서 볼 수 있는 URL이 있으므로 GraphQL에 URL 필드가 있습니다. 
이런 관계는 인터페이스로 표현할 수 있습니다.

```graphql
interface HasPublicUrl {
  publicUrl: Url
}
type Product implements HasPublicUrl {
  publicUrl: Url
  # some fields
}
type Collection implements HasPublicUrl {
  publicUrl: Url
  # some other fields
}
```

이런 접근은 간단한 경우에서는 매우 잘 작동하지만, 상당히 많은 필드와 함께 공유하는 동작이나 공유하는 동작이 많을 경우
제대로 작동하지 않습니다.

### Complex Shared Behaviour: Base Interfaces

복잡한 행동을 공유해야할 때, 사람들은 직관적으로 "base" 인터페이스를 사용해서 문제를 해결하려고 합니다.
많은 객체 지향 프로그래밍 언어에서 클래스 상속을 사용하는 것과 유사하게 말이죠.
예를 들어, 전 세계로 상품을 운송할 때 사용하는 운송 패키지(상자, 튜브, 봉투 등)를 생각해봅시다. 이들에게는 각각 고유한 물리적 치수가 있습니다.
그러나 유형에 관계없이 모든 배송 패키지가 공유하는 많은 속성도 있습니다. 
다음과 같은 인터페이스를 사용해 이 상황을 모델링할 수 있습니다.

```graphql
interface ShippingPackage {
  # Many common fields...
}
type ShippingPackageBox implements ShippingPackage {
  # Many common fields...
  height: Float!
  width: Float!
  depth: Float!
}
type ShippingPackageTube implements ShippingPackage {
  # Many common fields...
  length: Float!
  diameter: Float!
}
```

이 접근 방식은 어떤 경우에는 상당히 잘 작동하지만, 이것보다 더 나은 접근 방법이 있는 경우가 많습니다.
기본 인터페이스에는 유념해야할 몇 가지 단점과 엣지 케이스가 있습니다.

* 유형이 상당히 좁은 한 차원에서만 다른 경우(여기서 배송 패키지는 실제 물리적 치수에 의해서만 다르듯이) 스키마에서 찾아내기는 힘듭니다.
 동일한 여러 필드들 사이에 숨어 있는 하나의 작은 차이점을 발견하는 건 쉽지 않으니까요.
* 공유하는 필드가 많으면 필드 집합이 크면 해당 필드가 모든 인터페이스를 상속한 구체적 타입에 중복되므로 스키마가 다소 비대해집니다. (특히 구체적 타입이 많을 경우)
* 문제가 되는 유형이 여러 독립적인 타입들에 횡단으로 넓게 퍼져있다면, 가능한 모든 조합에 대한 타입을 만들어야 합니다. 결과적으로 지나치게 많은 타입이 생겨납니다.

### Complex Shared Behaviour: Push-Down Polymorphism

Shipping Package 예제와 같은 복잡한 사례에 대한 다른 일반적인 접근 방식은 "푸시다운" 다형성을 사용하는 것입니다. 
기본 객체를 나타내는 단일 콘크리트 유형을 생성하고 
nullability와 union을 사용하여 다형성을 해당 유형의 필드에 "푸시"합니다. 다음과 같습니다.

```graphql
# ShippingPackageBox에 대해 다른 필드들만 가진다
type ShippingBoxDimensions {
  height: Float!
  width: Float!
  depth: Float!
}
# ShippingPackageTube에 대해 다른 필드들만 가진다
type ShippingTubeDimensions {
  length: Float!
  diameter: Float!
}
union ShippingPackageDimensions = ShippingBoxDimensions | ShippingTubeDimensions
# shipping packages를 위해 이 타입만 이용한다
type ShippingPackage {
  # 공통 필드들...
  # 차이들을 field-level로 "푸시다운" 한다
  dimensions: ShippingPackageDimensions!
}
```

이는 기본 인터페이스 접근 방식의 주요 문제를 해결합니다:

* API에는 하나의 종류만 있으므로, 그것들이 어떻게 다른지는 알 필요가 없습니다.
* 다양한 타입들이 공유하는 필드를 반복하지 않아도 됩니다.
* 여러 차원을 따라 타입이 다를 경우 API는 구체적 타입의 폭발 없이 가능한 모든 조합을 자연스럽게 지원합니다.

물론, 이것도 만병통치약은 아닙니다. 가능한 모든 조합을 지원하고 싶지 않다면, 상속과 유사한 접근이 더 효과적일 수 있습니다.
당신의 비즈니스 도메인을 가장 잘 나타내는 방법을 선택하세요.