# Tutorial: diseñando una API de GraphQL

Este tutorial fue creado por [Shopify](https://www.shopify.ca/) para 
propósitos internos. Hemos creado una versión pública ya que creemos
que es útil a cualquiera creando una API de GraphQL.

Está basado en las lecciones aprendidas de la creación y la evolución de los esquemas de producción en 
Shopify durante casi 3 años. El tutorial ha evolucionado y continuará 
cambiando en el futuro así que nada está escrito en piedra.

Creemos que estas guías de diseño funcionan en la mayoría de los casos. Puede 
que no todas funcionen para ti. Incluso dentro de la compañía continuamos 
cuestionándolas y tenemos excepciones ya que la mayoría de las reglas
no se pueden aplicar el 100% de las veces. Así que no copies e implementes 
todo ciegamente. Selecciona cuáles hacen sentido para ti y tus casos de uso.

Tabla de contenido
=================
* [Intro](#intro)
* [Paso cero: contexto](#paso-cero-contexto)
* [Paso uno: una vista de pájaro](#paso-uno-una-vista-de-pájaro)
* [Paso dos: borrón y cuenta nueva](#paso-dos-borrón-y-cuenta-nueva)
  * [Representando `CollectionMembership`](#representando-CollectionMembership)
  * [Representando colecciones](#representando-colecciones)
  * [Conclusiones](#conclusiones)
* [Paso tres: agregando detalle](#paso-tres-agregando-detalle)
  * [Punto de partida](#punto-de-partida)
  * [IDs y la interfaz `Node`](#ids-y-la-interfaz-node)
  * [Reglas y subobjetos](#reglas-y-subobjetos)
  * [Listas y paginación](#listas-y-paginación)
  * [Strings](#strings)
  * [IDs y sus relaciones](#ids-y-sus-relaciones)
  * [Nombramiento y escalares](#nombramiento-y-escalares)
  * [Otra vez paginación](#otra-vez-paginación)
  * [Enums](#enums)
* [Paso cuatro: lógica de negocio](#paso-cuatro-lógica-de-negocio)
* [Paso cinco: mutaciones](#paso-cinco-mutaciones)
  * [Separa acciones lógicas](#separa-acciones-lógicas)
  * [Manipulando relaciones](#manipulando-relaciones)
  * [Entradas: estructura, parte 1](#entradas-estructura-parte-1)
  * [Entradas: escalares](#entradas-escalares)
  * [Entradas: estructura, parte 2](#entradas-estructura-parte-2)
  * [Salidas](#salidas)
* [TLDR: las reglas](#tldr-las-reglas)
* [Conclusión](#conclusión)

## Intro

¡Bienvenido! Este documento te guiará a través de los pasos para diseñar 
una nueva API de GraphQL (o una nueva pieza de una API existente de GraphQL). El diseño de una API es una tarea desafiante que recompensa 
fuertemente la iteración, experimentación, y el profundo entendimiento 
de tu dominio de negocio.

## Paso cero: contexto

Para los propósitos de este tutorial, imagina que trabajas en una compañía de e-commerce. 
Tienes una API de GraphQL que expone información acerca de tus productos, pero 
no mucho más. Sin embargo, tu equipo acaba de terminar un proyecto implementando 
"colecciones" en el back-end y quiere exponer las colecciones a través de tu API 
también.

Colecciones es el nuevo método para agrupar productos; por ejemplo, 
podrías tener una colección para todas tus playeras. Las colecciones pueden 
utilizarse para propósitos de visualización cuando se navega en tu sitio web, y también para tareas programáticas 
(ej. quisieras que un descuento solo aplicara a productos de una colección 
específica).

En el back-end, la nueva funcionalidad ha sido implementada de la siguiente manera: 
- Todas las colecciones tienen algunos atributos simples como un título, un cuerpo de descripción 
  (que podría incluir formato HTML), y una imagen.
- Tienes dos tipos específicos de colecciones: colecciones "manuales" donde 
  enlistas todos los productos que quieres incluir, y colecciones "automáticas" donde 
  especificas algunas reglas y permites que la colección se pueble a sí misma. 
- Ya que la relación producto-colección es de muchos-a-muchos, hay en el medio 
  una tabla unión llamada `ColecctionMembership`.
- Las colecciones, al igual que los productos, anteriores a ellas, pueden ser publicadas (visibles en
  el portal) o no.

Con este contexto, estás listo para comenzar a pensar en el diseño de tu API.

## Paso uno: una vista de pájaro

Una versión ingenua del esquema podría verse como algo así (excluyendo 
los tipos preexistentes como `Product`):
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

Esto ya es medianamente complicado a simple vista, a pesar de que solo son cuatro 
objetos y una interfaz. También está claro que no implementa toda la funcionalidad 
que necesitaríamos si vamos a usar esta API para construir, por ejemplo, nuestro
de la colección de aplicaciones para móviles.

Demos un paso atrás. Una API de GraphQL medianamente compleja consistirá en varios 
objetos, relacionados a través de múltiples rutas y docenas de campos. Tratar de diseñar 
algo como esto de un solo golpe es una receta para la confusión y los errores. En lugar de eso, 
deberías comenzar con una vista a alto nivel, enfocándote solamente en los tipos y 
sus relaciones, sin preocuparte acerca de campos específicos o mutaciones. 
Básicamente piensa en un [modelo entidad-relación](https://es.wikipedia.org/wiki/Modelo_entidad-relaci%C3%B3n)
pero con unas partes específicas de GraphQL agregadas. Si achicamos nuestro esquema ingenuo, 
terminamos con lo siguiente:

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

Para obtener esta representación simplificada, saqué todos los campos escalares, todos los campos de 
nombres, y toda la información de nulidad. Lo que se queda aún 
parece GraphQL pero te permite enfocarte a alto nivel en los tipos y sus 
relaciones.

*Regla #1: Siempre empieza con una vista a alto nivel de los objetos y sus 
relaciones antes de encargarte de campos específicos.*

## Paso dos: borrón y cuenta nueva

Ahora que tenemos algo simple con lo que trabajar, podemos abordar los principales defectos 
de este diseño.

Como se mencionó previamente, nuestra implementación definía la existencia de colecciones manuales y 
automáticas, al igual que el uso de una tabla de unión. Nuestro diseño 
ingenuo estaba claramente estructurado alrededor de nuestra implementación, sin embargo 
esto es un error.

El problema raíz con este enfoque es que una API opera para un propósito 
más que para una implementación, y frecuentemente a un nivel diferente de 
abstracción. En este caso, nuestra aplicación nos ha llevado por mal camino en una serie de
diferentes frentes.

### Representando `CollectionMembership`

Lo que puede que ya te haya llamado la atención, y ojalá por obvias razones, 
es la inclusión del tipo `CollectionMembership` en el esquema. La tabla de membresía a colecciones es 
usada para representar la relación de muchos-a-muchos entre los productos y las colecciones. 
Ahora lee la última oración otra vez: la relación es *entre productos y 
colecciones*; desde la perspectiva de la semántica y el dominio de negocio, la membresía a colecciones no tiene nada 
que ver con nada. Son detalles de implementación.

Esto significa que no pertenecen a nuestra API. En cambio, nuestra API debería exponer la verdadera 
relación entre el dominio de negocio y los productos directamente. Si eliminamos 
la membresía a colecciones, el diseño de alto nivel resultante se ve algo así:

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

Esto está mucho mejor.

*Regla #2: Nunca expongas detalles de implementación en el diseño de tu API.*

### Representando colecciones

El diseño de esta API todavía tiene un gran defecto, aunque es uno que probablemente es mucho
menos obvio sin un conocimiento realmente profundo del dominio de negocio. En 
nuestro diseño existente, modelamos AutomaticCollections y ManualCollections como dos 
tipos diferentes, cada uno implementando la interfaz común Collection. Intuitivamente 
esto hace un poco de sentido: ambas tienen varios campos en común, pero son 
distintivamente diferentes en sus relaciones (las colecciones automáticas tienen
reglas) y en algunos de sus comportamientos.

Pero desde una perspectiva de modelo de negocio, estas diferencias son básicamente 
detalles de implementación. El comportamiento determinante de una colección es que agrupa 
productos; el método por el cual se escogen esos productos es secundario. Podríamos expandir 
nuestra implementación en algún punto para permitir algún tercer método de selección de 
productos (¿Aprendizaje de máquina?) o permitir la mezcla de métodos (algunos productos agregados 
por reglas y otros manualmente) y *seguirían siendo colecciones*. Podrías incluso
argumentar que el hecho que actualmente no permitimos la mezcla de reglas es un fracaso de 
implementación. Todo esto es para decir que el diseño de nuestra API en realidad debería verse más 
como algo así:

```graphql
type Collection {
  [CollectionRule]
  Image
  [Product]
}

type CollectionRule { }
```

Eso está muy bonito. La preocupación inmediata que podrías tener en este punto es que 
ahora estamos pretendiendo que las colecciones manuales tienen reglas, pero recuerda que esta 
relación es una lista. En nuestro nuevo diseño de API, una "ManualCollection" es 
solamente una Collection cuya lista de reglas está vacía.

### Conclusiones

Elegir el mejor diseño de API en este nivel de abstracción necesariamente requiere 
un profundo entendimiento del dominio del problema que estás modelando. 
Es difícil en el escenario de un tutorial proveer la profundidad necesaria para un tema específico, 
pero con suerte el diseño de la colección es lo suficientemente simple para que el razonamiento
todavía tenga sentido. Incluso si no cuentas con esta profundidad de compresión
específicamente para colecciones, aun así la necesitas para cualquier dominio que en realidad 
estés modelando. Es de crítica importancia que cuando estés diseñando tu API 
te hagas estás preguntas complicadas, y no solo sigas ciegamente la 
implementación.

En una nota cercanamente relacionada, una buena API tampoco modela la interfaz de usuario. 
La implementación y la IU pueden ser ambas usadas como inspiración y entrada a 
tu diseño de API, pero al final el conductor de tus decisiones debe ser siempre
tu dominio de negocio.

Aún más importantemente, decisiones existentes de APIs REST no deben ser 
necesariamente copiadas. Los principios de diseño detrás de REST y GraphQL pueden llevar 
a decisiones muy diferentes, así que no asumas que lo que funcionó para tu API REST es 
una buena decisión para GraphQL.

En la medida de lo posible, suelta lo que llevas y empieza de nuevo.

*Regla #3: Diseña tu API alrededor de tu diseño de negocio, no alrededor de tu implementación,
la interfaz de usuario, o APIs legacy.*

## Paso tres: agregando detalle

Ahora que tenemos una estructura clara para modelar nuestros tipos, podemos agregar 
otra vez nuestros campos y empezar a trabajar en el detalle otra vez.

Antes de empezar a agregar detalle, pregúntate a ti mismo si es realmente necesario en este 
momento. Solamente porque una tabla de una base de datos, una propiedad del modelo, o un atributo REST puede que exista,
no significa que automáticamente necesita ser agregado al esquema de GraphQL.

Exponer un elemento del esquema (campo, argumento, tipo, etc.) debería ser impulsado por una 
necesidad verdadera y un caso de uso. Los esquemas de GraphQL pueden evolucionar fácilmente al agregar elementos, 
pero cambiarlos o eliminarlos son cambios de ruptura y son mucho más difíciles de 
realizar.

*Regla #4: Es más fácil agregar campos que eliminarlos.*

### Punto de partida

Restaurando los campos ingenuos ajustados para nuestro nuevo esquema obtenemos:

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

Ahora tenemos toda una nueva serie de problemas de diseño que resolver. Trabajaremos a través de
los campos en orden de arriba a abajo, arreglando las cosas a medida que avanzamos.

### IDs y la interfaz `Node`

El primer campo en nuestro tipo Collection es de tipo ID, que está bien y es normal; 
este ID es lo que necesitamos usar para identificar nuestras colecciones a través de 
la API, en particular cuando realicemos acciones como modificarlas o eliminarlas. 
Sin embargo, hay una pieza faltante en esta parte de nuestro diseño: la interfaz 
`Node`. Esta es una interfaz comúnmente usada que ya existe en la mayoría de los 
esquemas y se ve así:
 ```graphql
interface Node {
  id: ID!
}
```
Da una pista al cliente de que este objeto es persistente y extraíble dando un ID, 
lo que permite al cliente manejar cachés locales de forma precisa y eficiente, además 
de poder realizar otros trucos. La mayoría de tus objetos principales e 
identificables de negocio (ej. productos, colecciones, etc.) deben implementar `Node`.

El inicio de nuestro diseño ahora solo se ve así:
```graphql
type Collection implements Node {
  id: ID!
}
```

*Regla #5: Los tipos de los principales objetos de negocio siempre deben implementar `Node`.*

### Reglas y subobjetos

Consideraremos los siguientes dos objetos en nuestro tipo Collection conjuntamente: 
`rules`, y `rulesApplyDisjunctively`. El primero es bastante sencillo: una lista de 
reglas. Ten en cuenta que tanto la lista en sí como los elementos de la lista están marcados
como no nulos: esto está bien, GraphQL sí distingue entre `null`, `[]`  y `[null]`.
Para las colecciones manuales, esta lista puede estar vacía, pero no puede ser nula 
y tampoco puede contener un nulo.

*Tip pro: Campos de tipo lista son casi siempre listas no nulas con elementos no 
nulos. Si quieres que una lista pueda ser nula asegúrate de que hay un verdadero 
valor semántico en distinguir entre una lista vacía y una lista nula.*

El segundo campo es un poco raro: es un campo booleano indicando si las reglas pueden 
aplicar de manera disyuntiva o no. Tampoco puede ser nulo, pero aquí nos encontramos 
con un problema: ¿Qué valor debe tomar este campo para una colección manual? Hacerlo 
falso o verdadero se siente engañoso, pero permitir que sea nulo lo convierte una 
especie de bandera de tres estados que también es raro cuando se trata de colecciones 
automáticas. Mientras estamos intrigados por esto, hay otra cosa que vale la pena 
mencionar: estos dos campos están obvia e intrincadamente relacionados. 
Esto es verdadero semánticamente, y también es sugerido por el hecho de que elegimos 
nombres con un prefijo común ¿Hay alguna forma de indicar esta relación en el 
esquema?

De hecho, podemos resolver todos estos problemas de una sola vez
desviándonos aún más de nuestra aplicación subyacente e introduciendo un nuevo
tipo de GraphQL sin un equivalente en un modelo directo: `CollectionRuleSet`. Esto 
está usualmente garantizado cuando tienes un conjunto de campos estrechamente relacionados cuyos valores 
y comportamiento están vinculados. Al agrupar estos dos campos en solo tipo al nivel 
de API damos un claro indicador semántico y resolvemos todos nuestros problemas 
alrededor de la posibilidad del campo de ser nulo: para las colecciones manuales, es 
el conjunto de reglas en sí que es nulo. El campo booleano puede permanecer nulo. 
Esto nos lleva al siguiente diseño:

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

*Tip pro: Al igual que las listas, los campos booleanos son casi siempre no-nulos. 
Si quieres que un booleano pueda ser nulo, asegúrate de que haya un valor semántico 
real en poder distinguir entre los tres estados (null/false/true) y que eso no 
indica un mayor defecto de diseño.*

*Regla #6: Agrupa campos estrechamente relacionados en subobjetos.*

### Listas y paginación

El siguiente en la podadora es nuestro campo `products`. Este parece uno seguro; 
después de todo ya "arreglamos" esta relación antes cuando quitamos nuestro tipo `CollectionMembership`, 
pero de hecho hay otra cosa mal aquí.

El campo como está actualmente definido regresa un arreglo de productos, pero las 
colecciones pueden 
fácilmente tener muchos miles de productos, y tratar de juntarlos todos en un solo 
arreglo puede ser increíblemente caro e ineficiente. Para situaciones así, GraphQL 
provee paginación de listas.

Siempre que implementes un campo o una relación regresando múltiples objetos, 
pregúntate 
a ti mismo si el campo debería ser paginado o no ¿Cuántos de estos objetos pueden 
encontrarse ahí? ¿Qué cantidad es considerada patológica?

Paginar un campo significa que debes implementar una solución de paginación primero. 
Este tutorial usa [Connections](https://graphql.org/learn/pagination/#complete-connection-model)
que está definido por el [Relay Connection spec](https://facebook.github.io/relay/graphql/connections.htm).

En este caso, paginar el campo de productos en nuestro diseño es tan simple como 
cambiar su definición a `products: ProductConnection!`. Asuminendo que has 
implementado conexiones, tus tipos se verían así:

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

*Regla #7: Siempre verifica si los campos de listas deben paginarse o no.*

### Strings

El siguiente es el campo `title`. Este está legítimamente bien tal como está. Es un 
string simple, y está marcado como no nulo porque todas las colecciones deben tener 
un título.

*Tip pro: Al igual que los booleanos y las listas, es importante notar que GraphQL 
sí distingue entre strings vacíos (`""`) y nulos (`null`), así que si necesitas un 
string nulo asegúrate de que una diferencia semántica legítima entre no presente 
(`null`) y presente pero vacío (`""`). Puedes usualmente pensar en los strings 
vacíos como "aplica, pero no está poblado", y strings nulos como "no aplica".*

### IDs y sus relaciones

Ahora llegamos al campo `imageID`. Este campo es un ejemplo clásico de lo que sucede 
cuando tratas de aplicar diseños de REST a GraphQL. En las APIs REST es muy común 
incluir los IDs de otros objetos en tu respuesta como una forma de unirlos con 
aquellos objetos, pero este es un gran anti-patrón en GraphQL. 
En lugar de proveer un ID, y forzar al cliente a que haga otro viaje de regreso para 
traer la información del objeto, deberíamos solo incluir el objeto directamente 
en el grafo - eso es para lo que es GraphQL al final de todo. En las APIs REST este patrón 
no es práctico muchas veces, ya que infla el tamaño de la respuesta significativamente 
cuando los objetos incluidos son grandes. Sin embargo, esto funciona bien en GraphQL porque 
cada campo debe ser específicamente consultado o el servidor no lo regresará.

Como una regla de dedo, los únicos IDs en tu diseño deben ser los IDs de los objetos 
en sí. Cada vez que tengas algún otro campo ID, debería ser probablemente la 
referencia a un objeto en su lugar. Aplicando esto al esquema que tenemos hasta ahora, obtenemos:

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

*Regla #8: Siempre usa referencias a objetos en lugar de campos ID.*

### Nombramiento y escalares

El último campo en nuestro simple tipo `Collection` es `bodyHtml`. Para un usuario no 
familiarizado con la forma en que las colecciones fueron implementadas, no le es completamente
obvio para qué es este campo; es el cuerpo de la descripción de la colección en específico. 
Lo primero que podemos hacer para mejorar esta API es renombrarlo a `description`, 
que es un nombre mucho más claro.

*Regla #9: Elige nombres para tus campos basados en lo que hace sentido, no basados 
en la implementación o en la forma en que el campo era llamado en APIs legacy.*

Ahora, podemos hacerlo no nulo. Como hablamos anteriormente con el campo del título, 
no hace sentido distinguir entre el campo siendo nulo o simplemente siendo un string 
vacío, así que no expongas eso en tu API. Incluso si el esquema de tu base de datos 
sí permite tener a las entradas un valor nulo para esta columna, podemos esconder 
eso en la capa de la implementación.

Finalmente, debemos considerar si `String` es realmente el tipo adecuado para este 
campo. GraphQL proporciona un conjunto decente de tipos escalares incorporados (`String`, 
`Int`, `Boolean`, etc.) pero también te permite definir los tuyos, y este es un 
primordial de esa característica. La mayoría de los esquemas definen su conjunto de escalares adicionales 
dependiendo de sus casos de uso. Estos proveen un contexto adicional y valor 
semántico para sus clientes. En este caso, probablemente hace sentido definir un 
escalar personalizado `HTML` para usarse aquí (y potencialmente en otro lado) cuando 
el string en cuestión debe de ser HTML válido.

Cuando agregas un tipo escalar, vale la pena checar tu lista existente de escalares 
personalizados y ver si alguno de ellos quedaría mejor. Si estás agregando un campo 
y crees que sería apropiado usar un nuevo escalar personalizado, vale la pena 
hablarlo para asegurarte que estás capturando el concepto adecuado.

*Regla #10: Usa escalares personalizados cuando estés exponiendo algo con un valor 
semántico específico.*

### Otra vez paginación

Eso cubre todos los campos en nuestro tipo principal `Collection`. Es siguiente 
objeto es `CollectionRuleSet`, que es bastante simple. La única pregunta aquí es si 
la lista de reglas debería estar paginada o no. En este caso el arreglo existente 
hace sentido; paginar la lista de reglas sería exagerado. La mayoría de las 
colecciones solo tendrán un puñado de reglas, y no hay un buen caso de uso para que 
una colección tenga una gran cantidad de reglas. Incluso una docena de reglas es 
probablemente un indicador de que necesitas repensar la colección, o que deberías 
simplemente agregar los productos manualmente.

### Enums

Esto nos lleva al tipo final en nuestro esquema, `CollectionRule`. Cada regla
consiste en una columna (ej. título de producto) para hacer coincidir en un tipo de relación 
(ej. igualdad) y un valor real a utilizar (ej. "Botas") que es confusamente llamado 
`condition`. El último campo puede ser renombrado, y al que igual que `column`; columna 
es terminología bastante específica de base de datos, y estamos trabajando en GraphQL. `field` es 
probablemente una mejor elección.

En cuanto a tipos respecta, ambos `field` y `relation` son probablemente implementados 
internamente como enumeraciones (asumiendo que tu lenguaje de 
elección tiene enumeraciones). Afortunadamente GraphQL también tiene enumeraciones, 
así que podemos convertir esos campos a enums. Nuestro diseño de esquema completo ahora se ve así:

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

*Regla #11: Usa enums para campos que solo pueden tomar un conjunto específico de valores.*

## Paso cuatro: lógica de negocio

Ahora tenemos una mínima pero bien diseñada API de GraphQL para colecciones. Hay un montón 
de detalle en las colecciones que no hemos tratado - cualquier implementación 
real de esta funcionalidad necesitaría muchos más campos para lidiar con cosas
como el orden de clasificación de los productos, publicaciones, etc. - pero como una regla, todos esos campos 
seguirán los mismos patrones establecidos aquí. Sin embargo, todavía hay algunas
cosas que hay que mirar con más detalle.

Para esta sección, lo más conveniente es empezar con un caso de uso motivante del 
hipotético cliente de nuestra API. Por lo tanto, imaginémonos que el cliente 
desarrollador con el que hemos estado trabajando necesita saber algo muy específico:
si un producto dado es miembro de una colección o no. Claro, esto es 
algo que el cliente ya puede saber con nuestra API existente: exponemos 
todo el conjunto de productos en una colección, así que el cliente solo necesita
iterar, buscando el producto de su interés.

Sin embargo, esta solución tiene dos problemas. El primer problema obvio es que es 
ineficiente; las colecciones pueden contener millones de productos, y hacer que el cliente
traiga e itere a través de ellos sería extremadamente lento. El segundo, más grande 
problema, es que requiere que el cliente escriba código. Este último punto es una 
pieza crítica de la filosofía de diseño: el servidor debe ser siempre la única 
fuente de verdad para cualquier lógica de negocio. Una API casi siempre existe para servir 
a más de un cliente, y si cada uno de esos clientes tiene que implementar la misma 
lógica entonces efectivamente tienes duplicación de código, con todo el trabajo extra y 
espacio para el error que eso conlleva.

*Regla #12: La API debe proveer lógica de negocio, no solo datos. Cálculos 
complejos deben ser ejecutados en el servidor, en un solo lugar, no en el cliente, en 
muchos lugares.*

De regreso al caso de uso del cliente, aquí la mejor respuesta es proveer un nuevo campo
específicamente dedicado a solucionar este problema. Prácticamente, esto se ve así:
```graphql
type Collection implements Node {
  # ...
  hasProduct(id: ID!): Boolean!
}
```
Este campo toma el ID de un producto y regresa un booleano basado en si el servidor
determinó que un producto está en la colección o no. El hecho de que esto medio
duplica datos existentes del campo existente `products` es irrelevante. GraphQL
regresa solo lo que el cliente explícitamente pidió, así que al contrario de REST no nos cuesta nada 
agregar un par de campos secundarios. El cliente no necesita escribir nada de 
código más que consultar un campo adicional, y el total de ancho de banda utilizado es un 
solo ID más un solo booleano.

Una advertencia de seguimiento: solo porque estamos proporcionado la lógica de negocio 
en una situación no significa que no debemos proporcionar los datos en bruto. Los clientes 
deben poder hacer la lógica de negocio ellos mismos, si lo necesitan. No puedes 
predecir toda la lógica que el cliente va a querer, y no hay siempre un canal 
fácil por el cual los clientes puedan pedir campos adicionales (aunque deberías 
buscar asegurar lo más posible que ese tipo de canal exista).

*Regla #13: Proporciona los datos brutos también, aun cuando hay lógica de negocio alrededor de ellos.*

Finalmente, no dejes que los campos de lógica de negocio afecten la forma general de la API.
El modelo principal sigue siendo los datos del dominio de negocio. Si encuentras 
que la lógica de negocio no encaja realmente bien, entonces eso es una señal de que el modelo subyacente 
no está bien. 

## Paso cinco: mutaciones

La última pieza que falta en nuestro diseño del esquema GraphQL es la habilidad de realmente
cambiar los valores: crear, actualizar, y borrar colecciones y sus piezas relacionadas.
Al igual que con la parte de lectura de nuestro esquema, debemos empezar con una vista
a alto nivel: en este caso, de solo las distintas mutaciones que deseamos implementar,
sin preocuparnos de las entradas y salidas en específico. Ingenuamente podríamos seguir 
el paradigma CRUD y solo tener `create`, `delete`, y `update` como mutaciones.
Aunque este es un buen punto de partida, es insuficiente para una API GraphQL
correcta.

### Separa acciones lógicas

Lo primero que podemos observar si nos ajustamos solamente a CRUD es que nuestra 
mutación `update` se vuelve masiva rápidamente, es responsable no solo de actualizar 
valores escalares simples como el título sino también de realizar acciones complejas como 
publicar/eliminar las publicaciones, agregar/eliminar/reordenar 
productos de la colección, cambiar las reglas para colecciones 
automáticas, etc. Esto lo hace difícil de implementar en el servidor y difícil de razonar para el cliente. 
En lugar de eso, podemos tomar ventaja de GraphQL para dividirlo en más acciones lógicas 
granulares. Como una primera pasada, podemos separar publicar/eliminar publicaciones,
resultando en la siguiente lista de mutaciones:
- create
- delete
- update
- publish
- unpublish

*Regla #14: Escribe mutaciones separadas para acciones lógicas separadas en un recurso.*

### Manipulando relaciones

La mutación `update` todavía tiene demasiadas responsabilidades así que hace sentido continuar 
dividiéndola, pero vamos a tratar con estas acciones separadamente ya 
que vale la pena pensar en ellas en otra dimensión también: la manipulación 
de objetos de relaciones (ej. uno-a-muchos, muchos-a-muchos). Ya hemos 
considerado el uso de IDs vs embeberlos, y el uso de paginación vs 
arreglos en la API de lectura, y hay problemas similares con los que lidiar 
cuando mutemos estas relaciones.

Para la relación entre productos y relaciones, hay un par de estilos 
que podemos considerar en general:
- Embeber la relación entera (ej. `products: [ProductInput!]!`) en la mutación 
de actualización es el estilo de CRUD por defecto, pero claro que rápidamente se vuelve 
ineficiente cuando la lista es grande.
- Embeber campos "delta" (ej. `productsToAdd: [ID!]!` y
  `productsToRemove: [ID!]!`) en la mutación es más eficiente porque 
  solo se necesita especificar los IDs cambiantes en lugar de toda la lista, pero de todas maneras
  mantiene las acciones atadas juntas.
- Dividirlo completamente en mutaciones separadas (`addProduct`,
  `removeProduct`, etc.) es la forma más poderosa y flexible pero 
  también la que implica más trabajo.

La última opción generalmente es la más segura, especialmente porque mutaciones como 
esta usualmente serán acciones distintas de cualquier forma. Sin embargo,
hay muchos factores a considerar:
- ¿Es la relación grande o paginada? En ese caso, embeber la lista entera es 
definitivamente impráctico, sin embargo, ya sean los campos delta o mutaciones separadas 
podrían seguir funcionando. Pero si la relación es siempre pequeña (especialmente si es uno-a-uno), embeberla podría ser la opción más 
simple.
- ¿La relación está ordenada? La relación producto-colección está ordenada,
y permite ordenamiento manual. El orden está apoyado naturalmente por listas 
embebidas o por mutaciones separadas (puedes agregar una mutación `reorderProducts`) pero 
no es una opción para campos delta.
- ¿Es una relación obligatoria? Ambos, los productos y las colecciones pueden 
existir por si mismos fuera de la relación, con sus propios ciclos para ser creados/eliminados.
Si la relación fuera obligatoria (es decir, los productos tienen que estar en una colección)
entonces esto sugeriría fuertemente separar las mutaciones porque la acción sería realmente la de 
*crear* un producto, no solo de actualizar la relación.
- ¿Ambos lados tienen IDs? La relación colección-regla es obligatoria (las reglas 
no pueden existir sin colecciones) pero las reglas no tienen IDs; son 
claramente un sirviente de las colecciones, y ya que además la relación es 
pequeña, embeber la lista no es realmente una mala opción aquí. Cualquier otra cosa
requeriría que las reglas pudieran ser identificables individualmente y eso se siente algo
exagerado.

*Regla #15: Mutar relaciones es algo realmente complicado y no es 
fácilmente resumible en una regla rápida.*

Si mezclas todo esto junto, terminamos con la siguiente lista de 
mutaciones para las colecciones:
- create
- delete
- update
- publish
- unpublish
- addProducts
- removeProducts
- reorderProducts

Separamos los productos en sus propias mutaciones, porque la relación es grande
y ordenada. Las reglas las dejamos en línea porque la relación es pequeña, 
y las reglas son lo suficientemente secundarias para no tener IDs.

Finalmente, podrás notar que nuestras mutaciones de productos actúan en conjuntos de productos, por ejemplo
`addProducts` y no `addProduct`. Esto es simplemente una conveniencia para el cliente,
ya que el caso de uso común para esta relación será agregar, eliminar,
y reordenar más de un producto a la vez.

*Regla #16: Cuando escribas mutaciones separadas para las relaciones, 
considera si sería útil para las mutaciones operar en múltiples elementos a la vez.*

### Entradas: estructura, parte 1

Ahora que sabemos que mutaciones queremos escribir, necesitamos descifrar  
cómo se va a ver la estructura de sus entradas. Si has estado ojeando 
algunos de los esquemas de producción que están públicos, puede que hayas notado 
que muchos definen un tipo único global llamado `Input` que contiene a todos los argumentos:
este patrón era un requerimiento de algunos clientes legacy pero ya no es necesario 
para nuevos códigos; podemos ignorarlo.

Para muchas mutaciones simples, un ID o un puñado de IDs son todo lo que se necesita, 
haciendo este paso bastante sencillo. Entre las colecciones, podemos eliminar rápidamente
los siguientes argumentos de mutación:
- `delete`, `publish` y `unpublish` todos solo necesitan el ID de la colección
- `addProducts` y `removeProducts` ambos necesitan el ID de la colección, así como una lista 
con IDs de productos

Esto no deja solo con tres entradas "complicadas" para diseñar:
- create
- update
- reorderProducts

Empecemos con create. Una muy ingenua entrada podría verse como el modelo de 
nuestra colección original ingenua cuando empezamos, pero ahora podemos hacer algo mejor que eso.
Basados en nuestro modelo de colecciones final y la discusión acerca de relaciones arriba,
podemos empezar con algo como esto:

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

Pero primero una nota rápida acerca de nombramiento: podrás notar que nombramos a todas nuestras mutaciones 
en la forma de `collection<Action>` en lugar de la forma más natural 
del inglés `<action>Collection`. Desafortunadamente, GraphQL no proporciona un método para 
agrupar o en su defecto organizar mutaciones, así que nos vemos forzados 
a usar alfabetización para darle la vuelta. Poniendo el tipo base primero aseguramos 
que todas las mutaciones relacionadas queden agrupadas juntas en la lista final.

*Regla #17: Coloca el nombre del objeto como prefijo de las mutaciones 
para agruparlas alfabéticamente (ej. usa `orderCancel` en lugar de `cancelOrder`).*

### Entradas: escalares

Este borrador está mucho mejor que un acercamiento completamente ingenuo, pero todavía no es 
perfecto. En particular, la entrada del campo `description` tiene algunos problemas. Un 
campo no-nulo `HTML` hace sentido para la salida de la descripción de una colección, 
pero no funciona tan bien para la entrada por algunas razones. Primero que nada, mientas que 
`!` denota una salida no nula, eso no significa lo mismo en la 
entrada; en lugar de eso, denota el concepto de si un campo es "requerido". Un 
campo requerido es un que el cliente debe proveer para que la 
petición prosiga, y esto no es verdadero para `description`. No queremos prevenir a los clientes 
de crear colecciones si no dan una descripción (o equivalentemente, 
no queremos forzarlos a utilizar el inútil `""`), así que deberíamos hacer 
que `description` no requerido.

*Regla #18: Solo haz que los campos de entrada sean requeridos si 
realmente son semánticamente requeridos para que la mutación prosiga.*

El otro problema con `description` es su tipo; esto podría parecer contra intuitivo
ya que es fuertemente tipado (`HTML` en lugar de `String`) y hemos 
hablado bien de tipar fuertemente hasta ahora. Pero otra vez, las entradas han sido algo diferentes. 
Validación de entradas fuertemente tipadas sucede en la capa de GraphQL antes de que cualquier 
se corra cualquier código del "espacio de usuario", lo que significa que, siendo realistas, los clientes tienen que lidiar
con dos capas de errores: errores de validación de la capa de GraphQL, y errores de validación 
de la capa de negocio (por ejemplo algo así: haz alcanzado el límite de 
colecciones que puedes crear con tu capacidad de almacenamiento actual). Para simplificar este 
proceso, intencionalmente tipamos débilmente los campos de entrada en los casos 
en que pueda ser difícil para el cliente validarlos desde antes. Esto permite manejar al lado de la lógica de negocio 
toda la validación, y deja a los clientes solo tratar con errores en un solo punto.

*Regla #19: Usa tipos más débiles para las entradas (ej. `String` en lugar de `Email`) cuando 
el formato no es ambiguo y la validación del lado del cliente es compleja. Esto permite al 
servidor correr todas las validaciones no triviales de una y regresar todos los errores
en un solo lugar en un solo formato, simplificando al cliente.*

Es importante notar, sin embargo, que esto no es una invitación a tipar débilmente 
todas tus entradas. Todavía usamos enums fuertemente tipados para los valores de `field` y
`relation` en nuestra entrada de reglas, y también usaríamos tipados fuertes para ciertas 
entradas como `DateTime`s si las tuviéramos en este ejemplo. Los factores clave diferenciadores 
son la complejidad de validaciones en el lado del cliente y la 
ambigüedad del formato. HTML es una bien-definida y no ambigua especificación, pero 
requiere una validación bastante compleja. Por otro lado, hay cientos de formas de 
representar una fecha o tiempo en un string, todas ellas relativamente simples, así que es un 
beneficio de un campo escalar tipado especificar el formato que esperamos.

*Regla #20: Tipa fuertemente las entradas (ej. `DateTime` en lugar de `String`)
cuando el formato podría ser ambiguo y la validación del lado del cliente es simple. Esto 
provee claridad y anima a los clientes a usar controles de entrada más estrictos (ej. un widget de selección de fechas en lugar de un campo 
de texto libre).*

## Entradas: estructura, parte 2

Continuando con la mutación de actualización, podría verso como algo así: 

```graphql
type Mutation {
  # ...
  collectionCreate(title: String!, ruleSet: CollectionRuleSetInput, image: ImageInput, description: String)
  collectionUpdate(collectionId: ID!, title: String, ruleSet: CollectionRuleSetInput, image: ImageInput, description: String)
}
```

Notarás que esto es muy similar a nuestra mutación create, con dos 
diferencias: el argumento `collectionId` fue añadido, lo que determina 
que colección actualiza, y `title` no es requerido ya que la colección 
debe ya tener alguno. Ignorando el estatus de requerido del título por un momento, nuestro 
ejemplo de mutaciones tiene cuatro argumentos duplicados, y un modelo completo de 
colecciones requeriría bastantes más.

Mientras que hay algunos argumentos para dejar las mutaciones como están, hemos decidido que 
en situaciones como esta aplica DRY en las porciones comunes de los 
argumentos, incluso a costa de restricciones de exigencia. Esto tiene un par de ventajas:
- Terminamos con un solo objeto de entrada que representa el concepto 
de una colección y espejea el tipo único `Collection` que nuestro esquema ya tiene.
- Los clientes pueden compartir código entre las formas para crear y 
actualizar (un patrón común) porque terminan manipulando el mismo tipo de objeto de entrada.
- Las mutaciones permanecen delgadas y legibles con solo un par de argumentos de algo nivel.

El costo primario, por supuesto, es que ya no está claro en el esquema que
el título se requiere en la creación. Nuestro esquema termina viéndose así: 

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

*Regla número 21: Estructurar las entradas de las mutaciones para reducir la duplicación, incluso si esto
 requiere relajar las restricciones de la exigencia en ciertos campos.*

 ### Salidas

La pregunta final de diseño con la que tenemos que tratar es el valor de regreso de nuestras 
mutaciones. Típicamente las mutaciones pueden ser exitosas o fallar, y mientras que GraphQL
incluye soporte explícito para errores al nivel de consulta, estos no son ideales para 
fallos en mutaciones al nivel de negocio. En cambio, reservamos estos errores de alto nivel para
fallas del cliente (ej. solicitar un campo no existente) en lugar de para fallas del 
usuario. Como tal, cada mutación debería definir un tipo "carga útil" que incluya
campos para errores del usuario además de cualquier otro valor que pueda ser útil. Para 
crear, eso podría verse así:

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

Aquí, una mutación exitosa regresaría una lista vacía para `userErrors` y 
regresaría la recién creada colección para el campo `collection`. Una mutación 
fallida regresaría uno o más objetos `userErrors`, y `null` 
para la colección.

*Regla #22: Las mutaciones deben proporcionar errores al nivel de 
usuario/negocio a través de un campo `userErrors` en la carga útil de la mutación. La entrada de errores de consulta de alto nivel está 
reservada para errores a nivel del cliente y el servidor.*

En muchas implementaciones, mucha de esta estructura es proporcionada automáticamente, y 
todo lo que tienes que definir es el campo de regreso `collection`.

Para la mutación de actualización, seguimos exactamente el mismo patrón:

```graphql
type CollectionUpdatePayload {
  userErrors: [UserError!]!
  collection: Collection
}
```

Vale la pena notas que `collection` puede ser nulo incluso aquí,
ya que si el 
ID proporcionado no representa una colección valida, no hay una colección que regresar.

*Regla #23: La mayoría de los campos de carga útil para una mutación deberían poder ser nulos, a menos que haya
 realmente un valor a devolver en cada posible caso de error.*

## TLDR: las reglas

- Regla #1: Siempre empieza con una vista a alto nivel de los objetos y sus relaciones antes de encargarte de campos específicos.
- Regla #2: Nunca expongas detalles de implementación en el diseño de tu API.
- Regla #3: Diseña tu API alrededor de tu diseño de negocio, no alrededor de tu implementación, la interfaz de usuario, o APIs legacy.
- Regla #4: Es más fácil agregar campos que eliminarlos.
- Regla #5: Los tipos de los principales objetos de negocio siempre deben implementar `Node`.
- Regla #6: Agrupa campos estrechamente relacionados en subobjetos.
- Regla #7: Siempre verifica si los campos de listas deben paginarse o no.
- Regla #8: Siempre usa referencias a objetos en lugar de campos ID.
- Regla #9: Elige nombres para tus campos basados en lo que hace sentido, no basados en la implementación o en la forma en que el campo era llamado en APIs legacy.
- Regla #10: Usa escalares personalizados cuando estés exponiendo algo con un valor semántico específico.
- Regla #11: Usa enums para campos que solo pueden tomar un conjunto específico de valores.
- Regla #12: La API debe proveer lógica de negocio, no solo datos. Cálculos complejos deben ser ejecutados en el servidor, en un solo lugar, no en el cliente, en muchos lugares.
- Regla #13: Proporciona los datos brutos también, aun cuando hay lógica de negocio alrededor de ellos.
- Regla #14: Escribe mutaciones separadas para acciones lógicas separadas en un recurso.
- Regla #15: Mutar relaciones es algo realmente complicado y no es fácilmente resumible en una regla rápida.
- Regla #16: Cuando escribas mutaciones separadas para las relaciones, considera si sería útil para las mutaciones operar en múltiples elementos a la vez.
- Regla #17: Coloca el nombre del objeto como prefijo de las 
mutaciones para agruparlas alfabéticamente (ej. usa `orderCancel` en lugar de `cancelOrder`).
- Regla #18: Solo haz que los campos de entrada sean requeridos si realmente son semánticamente requeridos para que la mutación prosiga.
- Regla #19: Usa tipos más débiles para las entradas (ej. `String` en lugar de `Email`) cuando el formato no es ambiguo y la validación del lado del cliente es compleja. Esto permite al servidor correr todas las validaciones no triviales de una y regresar todos los errores en un solo lugar en un solo formato, simplificando al cliente.
- Regla #20: Tipa fuertemente las entradas (ej. `DateTime` en lugar de `String`) cuando el formato podría ser ambiguo y la validación del lado del cliente es simple. Esto provee claridad y anima a los clientes a usar controles de entrada más estrictos (ej. un widget de selección de fechas en lugar de un campo de texto libre).
- Regla número 21: Estructurar las entradas de las mutaciones para reducir la duplicación, incluso si esto requiere relajar las restricciones de la exigencia en ciertos campos.
- Regla #22: Las mutaciones deben proporcionar errores al nivel de usuario/negocio a través de un campo `userErrors` en la carga útil de la mutación. La entrada de errores de consulta de alto nivel está reservada para errores a nivel del cliente y el servidor.
- Regla #23: La mayoría de los campos de carga útil para una mutación deberían poder ser nulos, a menos que haya realmente un valor a devolver en cada posible caso de error.

## Conclusión

¡Gracias por leer nuestro tutorial! Ojalá que para este punto ya tengas una sólida
idea de cómo diseñar una buena API de GraphQL.

¡Una vez que hayas diseñado una API con la que estés contento, es tiempo de implementarla!
