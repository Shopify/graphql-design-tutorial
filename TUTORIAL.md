# Tutorial: Designing a GraphQL API

이 튜토리얼은 내부적인 목적을 위해 [Shopify](https://www.shopify.ca/)에서 만들어졌습니다. 
누구나 graphQL API를 만들기에 유용하다고 생각했기 때문에 우리는 이것을 공개하기로 했습니다.

거의 3년이 넘는 시간 동안 만들어진 Shopify의 프로덕션 스키마들로부터 배운 것들을 바탕으로 만들었습니다. 
이 튜토리얼도 많이 발전해왔고, 미래에도 계속해서 바뀔 예정입니다. 정해진 것은 없습니다.

우리는 스키마 디자인 가이드가 대부분의 경우에 유용할 것이라고 생각합니다. 어쩌면 당신에게는 맞지 않을 수도 있겠네요. 
대부분의 규칙이 항상 100% 적용되는 것은 아니기 때문에, Shopify 내부에서도 여전히 그 문제에 대한 답을 찾고 있고,
여러 예외 사항도 존재합니다. 그러니, 당신에게 맞는 것을 선택하시길 바랍니다.

## Intro

환영합니다! 이 문서는 당신에게 새로운 graphQL을 (또는 이밎 존재하는 graphQL API에 새 API를) 디자인하는 방법을
차근차근 알려줄 거예요. API 디자인은 반복과 실험, 그리고 당신의 비즈니스 도메인에 대한 이해가 필요한 어려운 일이지만요. 

## Step Zero: Background

이 튜토리얼의 목적을 위해, 당신이 e-commerce 기업에서 일하고 있다고 상상해보시기 바랍니다. 
당신은 이미 존재하는 graphQL API를 갖고 있습니다. 이 API는 제품들에 대한 정보를 외부에 노출하고 있지만, 그렇게 많은 정보는 아닙니다.
그런데, 당신의 팀은 백엔드에서 "collections"를 구현하는 프로젝트를 방금 끝냈고 기존에 존재하는 API에 그 Collections를 노출시키기 원합니다.

Collections는 제품을 그룹핑하는 새로운 go-to 함수입니다. 예를 들어, 당신이 티셔츠 collection을 갖고 있다고 해봅시다.
Collection은 당신의 웹사이트를 볼 때, 화면에 띄워주는 용도로 사용될 수 있습니다. 또한, 프로그래밍 업무를 위해서도 사용될 수 있죠. 
(예를 들면, 당신이 특정 collection에 대해서만 할인을 적용하고 싶을 때 적용할 수 있겠네요.)

백엔드에서는, 다음과 같은 것들을 통해 앞으로의 계획이 그려지고 있습니다.

- 모든 collection들은 제목, 요약(HTML 포맷팅(<역주>html 태그 같은 것들)을 포함하는), 이미지와 같은 간단한 속성들을 갖고 있습니다, 
- Collection에는 두 가지 동류가 있습니다. : 당신이 원하는 것들을 포함시키기 위해 직접 리스트업하는 "수동적인" collection들, 그리고 
 어떤 규칙을 정해놓고, collection들이 알아서 채워질 수 있도록 하는 "자동적인" collection들이 있습니다.
- 제품과 collection 관계는 다대다 관계이므로, 중간에 'CollectionMembership'이라는 조인 테이블을 갖고 있습니다.
- 제품과 같은 collections은 사이트 앞단에 그려질 수도 있고 그렇지 않을 수도 있습니다. (<역주>사이트에 보여주는 용도로 쓰거나, 프로그래밍적 용도로 쓰거나)

이러한 배경을 가지고, 어떻게 API를 설계할 수 있을 지 생각해보기로 합시다.

## Step One: A Bird's-Eye View (<역주> 조감도, 하늘에서 새가 내려다 보듯 전체적인 구성을 봅다)

단순한(또는 무식한) 방법으로 만든 스키마는 다음과 같이 구성할 수 있습니다. 
(`Product`같이 이미 존재하는 타입들은 모두 생략합시다.)


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

한 눈에 보기에도 상당히 복잡해보입니다. 4개의 객체와 하나의 인터페이스만 있을 뿐인데도요.
또한, 우리가 이 API를 이용해 모바일 앱의 collection 특징 같은 것을 구축하려고 한다면,
이 스키마는 우리가 필요로 하는 모든 특징을 명확히 구현하고 있는 것도 아닙니다.

한 발짝 뒤로 가봅시다. 복잡한 graphQL API는 다양한 경로와 몇 십개의 필드를 통해 많은 객체들을 구성하게 됩니다.
이런 API를 모두 한 번에 설계하려고 하는 것은 혼란과 실수를 야기하기 좋은 방법입니다. 
처음부터 구체적으로 시작하기보다는 더 높은 곳에서 바라보는 것부터 시작하는 게 좋습니다. 
구체적인 field나 mutation들은 걱정하지 말고, 일단 type과 그들의 관계에만 집중해보세요.

하지만, 'with a few GraphQL-specific bits thrown in'하는 경우, 기본적으로 ERD([Entity-Relationship model](https://en.wikipedia.org/wiki/Entity%E2%80%93relationship_model))에 대해 생각해봅시다.

만약 위의 단순한 스키마를 줄이고 싶다면, 다음과 같은 방법을 시도해봅시다.

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

더 간단한 스키마를 얻기 위해, 모든 스칼라 field와 field 이름, nullable한 정보는 뺐습니다.
남겨진 것들은 graphQL과 여전히 비슷하지만, 더 높은 수준의 type과 그 관계에 집중할 수 있게 합니다. 

*규칙 #1: 구체적인 필드를 다루기 전에, 항상 객체들과 그 사이의 관계를 높은 수준에서 바라보는 것부터 시작하세요.*
(<역주> 나무를 보기 전에 숲을 먼저 보세요.)

## Step Two: A Clean Slate

우리는 우리 API를 다루기 좀 더 간단한 것으로 만들었고, 이제 이 설계에 있는 큰 결함을 해결할 수 있습니다. 

이전에 말씀드린 것처럼, 우리가 만든 구현은 수동적이고 자동적인 collection들을 정의하고 있습니다. 
단순한(무식한) API 디자인은 우리의 구현에 대해서는 명확히 짜여졌습니다만, 이건 잘못된 방법입니다. 

이 접근 방법의 가장 근본적인 문제는, API는 '구현과는 다른 목적으로, 그리고 종종 다른 추상화 레벨에서 동작한다'는 것입니다.
이 경우, 우리의 구현은 많은 다른 프론트 단에서 실수를 유발하도록 할 수 있습니다.

### Representing `CollectionMembership`s

가장 눈에 띄는 것, 가장 명확한 것은 스키마 안에 `CollectionMembership` type이 포함되어 있다는 것입니다. 
collection memberships table은 product와 collection들 간의 다대다 관계를 표현하기 위해 사용됩니다.
자, 마지막 문장을 다시 읽어봅시다. *product와 collection들 간의* 관계입니다. 
비즈니스 관점에서는 collection membership 그 자체가 하는 것이 아무 것도 없습니다. 
그것은 그저 구현의 세부사항일 뿐입니다.

이것은 collection membership이 우리 API에 포함되지 않는다는 것을 의미합니다. 
우리 API는 실질적인 비즈니스 도메인 관계를 제품에 직접적으로 노출시키길 원합니다. 

우리가 collection membership을 뺀다면, 높은 수준에서의 설계는 다음과 같이 보일 것입니다. 

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

이 API 디자인은 여전히 한 가지 중요한 결함을 갖고 있습니다. 비즈니스 도메인에 대한 이해 없이는, 아마 덜 느껴질 것입니다.
이 설계에서, 우리는  AutomaticCollections와 ManualCollections를 두 개의 다른 type으로 모델링했습니다.
그리고 이 두 type은 각각 공통적으로 Collection interface를 구현합니다. 직관적으로 보면 말이 되는 것처럼 보이기도 합니다.
그들 사이에는 많은 공통의 field가 존재하지만, 여전히 그들의 관계나 동작하는 방식은 많이 다릅니다. (AutomaticCollections는 규칙을 갖고 있죠.)

비즈니스 모델 관점으로부터, 이러한 차이점은 기본적으로 구현상의 디테일일 뿐입니다. collection의 정의는 제품을 그룹핑하는 것이었습니다.
그러한 제품들을 선택하는 함수(자동이냐, 수동이냐)는 두번째입니다. 우리는 제품을 선택하는 세번째 함수를 허용하거나, 함수를 섞도록
(수동적으로 제품을 추가하는 방법과 어떤 규칙들을 섞도록) 어느 시점에서 구현을 확장할 수 있습니다. 
그리고 그것들은 여전히 collections로 정의할 수 있습니다. 그렇다면, 당신은 왜 그 함수들을 지금 당장 섞도록 허용하지 않냐고 말할 수도 있겠네요. 
만약 그렇게 한다면, API의 모양은 다음과 같아집니다.

```graphql
type Collection {
  [CollectionRule]
  Image
  [Product]
}

type CollectionRule { }
```

정말 나이스합니다! 이 시점에서 당신이 걱정할 수도 있는 것은, 우리가 ManualCollections를 
규칙을 갖고 있는 것처럼 만들었다는 것이겠네요. 하지만, 이 관계는 리스트라는 것을 기억하세요. 
우리의 새 API 설계에서는, "ManualCollections"는 단순히 비어있는 규칙 리스트를 가진 collection일 뿐입니다.

### Conclusion

추상화 단계에서 가장 최고의 API 설계를 선택하려면, 모델링하고 있는 도메인에 대해 깊게 이해하고 있어야 합니다.
튜토리얼 환경에서는 구체적인 주제에 대한 깊은 이해를 제공하기는 힘듭니다. 하지만, collection 디자인은 추론이 가능할 정도로 충분히 간단합니다. 
collection에 대해 깊이 이해하고 있지 않더라도, 실제로 모델링하고 있는 영역에 대해서는 깊은 이해가 필요합니다.
당신의 API를 설계할 때, 이런 어려운 질문을 스스로 던져보는 것, 그리고 맹목적으로 구현하지 않는 것은 매우 중요한 일입니다.

관련 레퍼런스 중에서, 좋은 API는 사용자 인터페이스를 모델링하지 않는다고 합니다. 
구현과 UI는 모두 당신의 API 설계에 있어 제공과 입력을 위해 사용될 수 있지만, 
결정의 가장 중요한 동인은 항상 비즈니스 도메인이어야 합니다.

더 중요하게는, 기존의 REST API를 복사하지 않는 것이 좋습니다. REST와 GraphQL의 설계 원칙은 다른 영역입니다. 
그러므로 당신의 REST API에 동작하는 것이 GraphQL에서도 좋은 선택이 될 것이라 가정하시면 안됩니다.

가능한 당신의 짐을 최대한 내려놓고, 처음부터 시작하시길 바랍니다.

*Rule #3: 구현도 UI도 기존 API도 아닌, 비즈니스 도메인에 맞춰 API를 설계하세요.*

## Step Three: Adding Detail

이제 우리는 type을 모델링하기에 깔끔한 구조를 갖게 되었습니다. 우리는 field를 추가할 수 있고, 다시 세부적인 수준에서 시작할 수 있습니다. 

세부사항을 추가하기 전에, 지금 시점에 이것을 추가하는 게 맞는지 스스로에게 물어봅시다. 데이터베이스 컬럼, 모델 속성, 또는 REST 속성이 이미 기존에 있다는 이유로 graphQL 스키마에 추가할 필요는 없습니다.

실질적인 요구와 활용 사례에 의해 스키마 요소(field, argument, type 등)를 추가하는 게 좋습니다. GraphQL 스키마는 요소를 추가함으로써 쉽게 발전될 수 있습니다. 하지만, 요소를 제거하거나 변형하는 것은 변화를 깨뜨리는 것이고, 일을 훨씬 더 어렵게 들 수 있습니다. 

*규칙 #4: 필드를 제거하는 것보다 추가하는 것이 더 쉽습니다.*

### Starting point

새로운 구조에 맞게 조정된 단순한 필드를 복구시켜 봅시다.

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

이제 우리는 해결해야 할 완전히 새로운 설계 문제를 갖고 있습니다. 우리는 우리가 가는 대로 꼭대기부터 바닥까지 필드를 고쳐갈 것입니다. 

### IDs and the `Node` Interface

Collection type에서 가장 첫번째 필드는 ID 필드입니다. 이 필드는 꽤 괜찮고 정상적이네요. 이 ID는 
우리가 API에서 collection들을 식별하기 위해 사용하는 데에 필요합니다. 특히, collection들을 
수정하거나 삭제해야할 때 도움이 될 것입니다. 그러나 우리 설계에는 어떤 한 조각이 빠져있습니다. 
그것은 `Node`라는 인터페이스입니다. 이것은 대부분의 스키마에서 이미 존재하는, 매우 공통적으로 사용되는 인터페이스입니다.
생긴 것은 다음과 같습니다. 

```graphql
interface Node {
  id: ID!
}
```

클라이언트에게 이 객체는 ID가 주어져야 한다는 것을 알려줍니다. 이 ID는 클라이언트가 정확하고 효율적으로 
로컬 캐시나 다른 트릭들을 관리할 수 있도록 해줍니다. 
대부분의 식별 가능한 비즈니스 객체(products, collections 등)는 이 `Node`를 구현해야 합니다.

우리 설계의 시작은 이제 이렇게 됩니다. 
```graphql
type Collection implements Node {
  id: ID!
}
```

*규칙 #5: 주요한 비즈니스 객체 type은 항상 `Node`를 구현해야 합니다.*

### Rules and Subobjects

우리는 collection 필드 중에서 `rules`, `rulesApplyDisjunctively`라는 두 가지 필드를 살펴볼 것입니다.
첫번째는 rules의 리스트라는 꽤 정직한 이름입니다. 리스트 그 자체와 그 안의 요소 모두 non-null(<역주> 필드 값이 null로 저장될 수 없음)로 표시된다는 것을 알아두세요. GraphQL은 `null`과 `[]` 그리고 `[null]`을 구별하기 때문에 괜찮습니다. 
수동적인 collection을 위해, 우리는 이 리스트를 비워둘 수 있습니다. 하지만 그것이 null이 되거나, null 값을 포함할 수는 없습니다.
(<역주> `null`, `[null]`은 안되고, `[]`은 가능합니다.)

*Protip: List-type 필드는 거의 항상 non-null 요소를 가지는 non-null 리스트입니다. 만약 당신이 nullable한 리스트를 원한다면, 리스트에 빈 리스트와 null 리스트를 구별할 수 있도록 하는 \*'시맨틱 값'이 있어야 한다는 것을 확실히 해두세요.*

🧚‍♀️ <역주> 시맨틱 값(semantic value): 의미론적인 값이란, 함수나 값이 어떤 것인지 설명하지 않아도 그 이름만으로 어떤 역할, 어떤 의미를 가지는 지 알아볼 수 있는 값입니다. HTML을 예로 들면, `<p style="font-size: 32px;"> header </p>`라고 표현하는 것보다 `<header> header </header>`라고 표현하는 것이 해당 태그가 무엇인지 더 알아보기 쉬울 것입니다.

두번째 필드는 살짝 이상합니다. 이것은 규칙이 불분명하게 적용되는지 아닌지 알려주는 boolean 타입의 필드입니다. 
이 또한, non-null 필드지만, 여기에는 문제가 있습니다. 수동적인 collection에서는, 어떤 값이 이 필드에 들어와야 할까요?
fales나 true로 두는 것 어떤 것도 잘못된 방법처럼 느껴집니다. 그렇다고 필드를 nullable로 만드는 것 또한, 일종의 이상한
3개주 지역의 깃발처럼 되어(이것도 저것도 아닌 상태가 되어) 자동적인 collection을 다루기에도 어색해집니다. 우리가 이 문제를 
해결하고 있는 동안, 언급할 만한 다른 하나의 사실이 있습니다. 이 두 필드들 사이에는 명확하고, 복잡한 관계가 있다는 것입니다.
이것은 의미론적으로 사실이며, 우리가 공통된 접두사를 붙였다는 것에서도 알 수 있습니다. 그렇다면, 어떤 방법으로 스키마에서
이 관계를 드러낼 수 있을까요? 

사실, 우리는 기본적인 구현으로부터 멀리 벗어나, 직접적인 모델과 동등하지 않은 새로운 graphQL type을 도입함으로써 이런 문제를 단번에 해결할 수 있습니다. 그 type을 `CollectionRuleSet`이라고 합니다. 이것은 당신이 값이나 행위가 연결되어 있는, 근접한 관계에 있는 필드들의 집합을 갖고 있을 때 종종 사용됩니다. 두 필드를 API에서 우리가 만든 type으로 그룹핑함으로써, 우리는 깔끔한 시맨틱 식별자를 제공하고, 또한 우리가 nullability와 관련해서 가졌던 모든 문제를 해결할 수 있습니다. 수동적인 collection에서는, 우리는 rule-set을 그 자체로 null로 둡니다. boolean 필드는 non-null로 남을 수 있습니다. 다음처럼 설계할 수 있습니다. 

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

*Protip: list 같이, boolean 필드도 거의 항상 non-null입니다. 만약 nullable한 boolean을 원한다면, null, false, true의 세 가지 상태를 구별할 수 있는 시맨틱 값을 포함시켜야 합니다. 그리고 이런 행위가 더 큰 설계 결함을 일으키지 않는 지도 확인해야 합니다.*

*규칙 #6: 근접한 관계를 가진 필드는 하위-객체로 그룹핑하세요.*

### Lists and Pagination

다음은 `products` 필드를 볼 차례입니다. 보기에는 안전해 보입니다. `CollectionMembership`을 제거했을 때, 이미 이 관계를 고친 뒤지만, 여기에는 또 다른 문제가 있습니다.   

현재 products 배열을 반환하도록 정의된 필드이지만, collection들은 몇 십, 몇 천의 products를 가질 수 있습니다.
그리고 그 모든 것들을 하나의 배열로 모으는 것은 아주 큰 비용을 치뤄야 하며, 비효율적일 수 있습니다. 이런 상황 때문에,
graphQL은 lists pagination이라는 것을 제공합니다. 

여러 객체를 반환하는 필드나 릴레이션을 구현할 때마다, 항상 스스로에게 그 필드를 페이지네이션할 수 있는지 묻길 바랍니다. 
필드에 얼마나 많은 객체가 들어올 수 있을까요? 최대한으로 생각하는 수량은 어느 정도인가요?

필드를 페이지네이션하기 위해서는 페이지네이션 솔루션을 먼저 구현해야합니다. 이 튜토리얼은 [Connections](https://graphql.org/learn/pagination/#complete-connection-model)을 사용합니다. 이것은 [Relay Connection spec](https://facebook.github.io/relay/graphql/connections.htm)에 정의되어 있습니다. 

이 경우, 우리 설계에서 products 필드를 페이지네이션하는 것은 그것의 정의를 `products: ProductConnection!`로 바꾸는 것 만큼이나 간단합니다. 당신이 connections를 구현한다고 가정하면, type은 다음과 같아집니다.

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

###  Strings

다음은 `title` 필드입니다. 합리적으로 괜찮은 방법입니다. 
간단한 문자열이고, non-null로 표시됩니다. 왜냐하면, collection들은 각각 title을 가져야하니까요.

*Protip: boolean, list와 마찬가지로, graphQL은 빈 문자열과 null을 구분합니다. 그러니 nullable한 문자열을 원할 땐, 합리적으로 표현되지만 비어있는(`""`) 것과 표현되지 않는 것(`null`) 사이에 시맨틱한 차이점이 있는지 확인해보세요. 빈 문자열을 "적용 가능하지만 채워지지 않는다"는 의미로 생각할 수 있으며, null 문자열은 "적용할 수 없다"는 것으로 종종 생각할 수 있습니다.*

### IDs and Relations

이제 `imageId` 필드로 왔습니다. 이 필드는 우리가 REST 설계를 GraphQL에 적용할 때 발생하는 클래식한 예시입니다.
REST API에서는 다른 객체들을 연결하기 위한 방법으로, 당신의 response에 다른 객체의 ID를 포함하는 것은 꽤 흔한 일입니다.
객체에 대한 다른 정보를 얻기 위해, ID를 제공하거나 클라이언트에게 (input, output을 주고 받는) 왕복하도록 강요하는 대신에, 그저 직접적으로 graph에 객체를 포함하는 것이 좋습니다. 그게 바로 GraphQL이 존재하는 목적이죠. REST API에서는 이 패턴이 종종 실용적인 것은 아닙니다. 객체의 사이즈가 클 때는 response의 크기가 상당히 증가할 수 있기 때문입니다. 그러나, GraphQL에서는 괜찮습니다. 왜냐하면, 모든 필드는 반드시 명시적으로 질의되거나, 서버가 이것을 반환하지 않을 것이기 때문입니다.

일반적인 규칙으로서, 설계에서 ID 필드들은 오직 해당 오브젝트의 ID 필드여야합니다(다른 오브젝트의 ID필드 포함 x). 다른 ID 필드를 가질 때는, 아마도 그것은 그 객체의 레퍼런스가 되어야합니다. 이 규칙을 우리 스키마에 적용하면, 다음과 같습니다.

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

*규칙 #8: 다른 ID 필드들을 사용하기보다는, 항상 객체 레퍼런스를 사용하세요.*

### Naming and Scalars

우리의 `Collection` type에서 마지막 필드는 `bodyHtml`입니다. 
collections가 구현되는 방법에 친숙하지 않는 사용자에게는, 이 필드의 역할이 완전히 명확하지는 않을 것입니다.
이것은 구체적인 collection에 대한 body description입니다. 우리가 이 API를 더 나은 것으로 만들기 위해
첫번째로 할 수 있는 것은 이 필드의 이름을 그저 `description`으로 바꾸는 것입니다. 그게 훨씬 더 명확한 이름같습니다.

*규칙 #9: 구현 또는 기존 API에서 그 필드가 무엇으로 불렸는지에 근거하기 보다는 좀 더 명확한 필드 이름을 선택하세요.*

Next, we can make it non-nullable. As we talked about with the title field, it
doesn't make sense to distinguish between the field being null and simply being
an empty string, so we don't expose that in the API. Even if your database
schema does allow records to have a null value for this column, we can hide that
at the implementation layer.

다음으로 우리는 이것을 non-nullable로 만들 수 있습니다. title 필드에 대해서 말했던 것처럼, 
필드가 null이 되는 것과, 단순히 빈 문자열인 것을 구분하는 것은 합당한 일이 아닙니다. 
그래서 우리는 이것을 API에는 노출시키지 않을 것입니다. 
데이터베이스 스키마가 컬럼에서 값이 null을 가지도록 허락한다 해도, 우리는 구현 단에서 이것을 숨길 수 있습니다. 

마지막으로, 우리는 `String`이 이 필드에 실질적으로 맞는 type인지 고려해봐야 합니다. GraphQL은 꽤 괜찮은 
내장된 스칼라 타입의 집합을 갖고 있습니다. 하지만 이것은 당신 스스로 당신의 것을 정의하도록 합니다. 그리고 그것이 이 기능을
사용하는 주요한 활용 사례이기도 합니다. 대부분의 스키마들은 자신의 활용 사례에 스스로의 추가적인 스칼라 집합을 정의합니다. 
이것은 클라이언트에게 시맨틱 값이나 추가적인 context를 제공합니다. 이 경우, 문제의 문자열이 유효한 HTML이어야 하는 때에 여기에(잠재적으로는 다른 곳에도) 사용자 정의 `HTML` 스칼라를 정의하는 것이 타당할 것입니다.

Whenever you're adding a scalar field, it's worth checking your existing list of
custom scalars to see if one of them would be a better fit. If you're adding a
field and you think a new custom scalar would be appropriate, it's worth talking
it over with your team to make sure you're capturing the right concept.

당신이 스칼라 필드를 추가할 때마다, 이미 존재하는 사용자 정의 스칼라 리스트를 확인하는 것이 좋습니다. 
새로 만들기 보다는, 이미 존재하는 것 중에 더 잘 맞는 게 있을 수 있으니까요. 만약, 당신이 필드를 추가하고 있고, 
당신이 생각하기에 새로운 사용자 정의 스칼라가 더 적당하다면, 팀과 상의하여 올바른 개념을 파악하고 있는지 확인해보는 것이 좋습니다.

*규칙 #10: 무언가 구체적인 시맨틱 값을 노출할 때는 사용자 정의 스칼라 타입을 사용하세요.*

### Pagination Again

지금까지 핵심적인 `Collection` type의 모든 필드를 살펴봤습니다. 다음 객체는 `CollectionRuleSet`입니다.
꽤 간단한 객체죠. 여기서의 문제는 그저 rules의 리스트가 페이지네이션 되어야하는가 말아야하는가일 뿐입니다.
이 경우에는, 기존의 배열이 더 타당합니다. rules 리스트를 페이지네이션하는 것은 과잉 행동이 될 수 있습니다. 
대부분의 collection들은 적은 규칙만을 가지기 때문입니다. 그리고 collection에게는 큰 rule set을 가지는 것이 좋은 활용 사례는 아닙니다. 규칙이 십여 가지가 된다면 products를 수동적으로 추가해야하거나, 그 collection이 옳은지 재고해봐야 하는 지표가 될 것입니다.


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
