
JanusGraph支持两种不同的索引来提高查询速度：图索引和以顶点为中心的索引。 大多数图形查询从一些由其属性标识的顶点或边开始遍历。 图索引使这些全局检索操作在大图上有效。 以顶点为中心的索引可加快整个图的实际遍历速度，特别是在遍历具有许多有入射边的顶点时有效。

## 11.1. 图索引

图形索引是整个图形上的全局索引结构，允许通过其属性有效地检索顶点或边缘，以获得足够的选择条件。 例如，请看以下查询：
```
g.V().has('name', 'hercules')
g.E().has('reason', textContains('loves'))
```
第一个查询所有名字为hercules的顶点。 第二个查询所有reason属性中有love关键字的边。 如果没有图表索引执行这些查询，则需要对图中的所有顶点或边进行全面扫描，以找到与给定条件匹配的结果，这对于巨大的图形而言是非常低效且不可行的。

区分JanusGraph两种图索引类型：复合索引和混合索引。 复合索引非常快速且高效，但仅限于特定的，先前定义的属性键组合的等值查找。 混合索引可用于对索引键的任何组合进行查找，并且除了等值之外还支持多个条件谓词查询，具体取决于支持的后端索引存储类型。

两种类型的索引的创建都是通过JanusGraph管理系统和索引构建器JanusGraphManagement.buildIndex（String，Class）的返回结果，其中第一个参数定义索引的名称，第二个参数指定要索引的元素的类型（例如，Vertex.class）。图索引的名称必须是唯一的。针对新定义的属性键构建的图索引，即在与索引相同的管理事务中定义的属性键，可立即使用。这同样适用于受限于与索引在同一管理事务中创建的标签的图索引。针对已经使用但未限制为新创建的标签的属性键构建的图索引需要执行reindex过程以确保索引包含所有先前添加的元素。在reindex过程完成之前，索引将不可用。建议在与初始模式相同的事务中定义图索引。

> *注意*

> *在没有索引的情况下，JanusGraph将默认为完整图形扫描，以便检索所需的顶点列表。 虽然这会产生正确的结果集，但图形扫描效率非常低，导致生产环境中的整体系统性能较差。 *
> *在JanusGraph的生产部署中启用强制索引配置选项以禁止图扫描。*

### 11.1.1. 复合索引

复合索引通过一个或多个（固定）组合的多个键来检索顶点或边。 请考虑以下复合索引定义。
```
graph.tx().rollback() //Never create new indexes while a transaction is active
mgmt = graph.openManagement()
name = mgmt.getPropertyKey('name')
age = mgmt.getPropertyKey('age')
mgmt.buildIndex('byNameComposite', Vertex.class).addKey(name).buildCompositeIndex()
mgmt.buildIndex('byNameAndAgeComposite', Vertex.class).addKey(name).addKey(age).buildCompositeIndex()
mgmt.commit()
//Wait for the index to become available
ManagementSystem.awaitGraphIndexStatus(graph, 'byNameComposite').call()
ManagementSystem.awaitGraphIndexStatus(graph, 'byNameAndAgeComposite').call()
//Reindex the existing data
mgmt = graph.openManagement()
mgmt.updateIndex(mgmt.getGraphIndex("byNameComposite"), SchemaAction.REINDEX).get()
mgmt.updateIndex(mgmt.getGraphIndex("byNameAndAgeComposite"), SchemaAction.REINDEX).get()
mgmt.commit()
```

首先，已经定义了两个属性键名称和年龄。 接下来，构建一个仅对name属性键的简单复合索引。 JanusGraph将使用此索引来回答以下查询。

```
g.V().has('name', 'hercules')
```
其次，复合图索引包括这两个键（name和age）。 JanusGraph将使用此索引来响应以下查询。
```
g.V().has('age', 30).has('name', 'hercules')
```
请注意，必须在查询的相等条件中找到复合图索引的所有键，才能使用此索引。 例如，以下查询无法使用任一索引来回答，因为它仅包含对age的约束，而不包含name。
```
g.V().has('age', 30)
```
再请注意，复合图索引只能用于上述查询中的相等约束。 仅使用name键上定义的简单复合索引来回答以下查询，因为age约束不是等式约束
```
g.V().has('name', 'hercules').has('age', inside(20, 50))
```
 
复合索引不需要配置外部索引后端，并且通过主存储后端支持。 因此，复合索引修改通过与图修改相同的事务持久化，这意味着如果底层存储后端支持原子性and/or一致性，那些更改是原子的and/or一致性的。

> *注意*
>
> *复合索引可以仅包括一个或多个键。 仅具有一个键的复合索引有时被称为键索引。*

##### 11.1.1.1. 索引唯一性

Composite indexes can also be used to enforce property uniqueness in the graph. If a composite graph index is defined as unique() there can be at most one vertex or edge for any given concatenation of property values associated with the keys of that index. For instance, to enforce that names are unique across the entire graph the following composite graph index would be defined.

```
graph.tx().rollback()  //Never create new indexes while a transaction is active
mgmt = graph.openManagement()
name = mgmt.getPropertyKey('name')
mgmt.buildIndex('byNameUnique', Vertex.class).addKey(name).unique().buildCompositeIndex()
mgmt.commit()
//Wait for the index to become available
ManagementSystem.awaitGraphIndexStatus(graph, 'byNameUnique').call()
//Reindex the existing data
mgmt = graph.openManagement()
mgmt.updateIndex(mgmt.getGraphIndex("byNameUnique"), SchemaAction.REINDEX).get()
mgmt.commit()
```
> 注意:
    要对最终一致的存储后端强制唯一性，必须将索引的一致性显式设置为启用锁定。
    
##### 11.1.2. 混合索引
混合索引通过先前添加的任何属性键组合来检索顶点或边。混合索引提供了比复合索引更大的灵活性，并且支持不相等的附加条件谓词。另一方面，对于大多数相等查询，混合索引比复合索引慢。  

与复合索引不同，混合索引需要配置索引后端，并使用该索引后端执行查找操作。Janusgraph可以在单个安装中支持多个索引后端。每个索引后端必须在janusgraph配置中通过名称唯一标识，该配置称为索引后端名称。  

```$xslt
graph.tx().rollback()  //Never create new indexes while a transaction is active
mgmt = graph.openManagement()
name = mgmt.getPropertyKey('name')
age = mgmt.getPropertyKey('age')
mgmt.buildIndex('nameAndAge', Vertex.class).addKey(name).addKey(age).buildMixedIndex("search")
mgmt.commit()
//Wait for the index to become available
ManagementSystem.awaitGraphIndexStatus(graph, 'nameAndAge').call()
//Reindex the existing data
mgmt = graph.openManagement()
mgmt.updateIndex(mgmt.getGraphIndex("nameAndAge"), SchemaAction.REINDEX).get()
mgmt.commit()
```
上面的示例定义了一个包含属性键名称和期限的混合索引。该定义引用了索引后端名称搜索，以便Janusgraph知道应该将哪个配置的索引后端用于此特定索引。在buildMixedIndex调用中指定的搜索参数必须与janusGraph配置定义中的第二个子句匹配，如：index.search.backend。如果索引名为solrsearch，则配置定义将如下所示：index.solrsearch.backend。  

上面指定的mgmt.buildIndex示例使用文本搜索作为其默认行为。将索引显式定义为文本索引的索引语句可以编写如下：
```$xslt
mgmt.buildIndex('nameAndAge',Vertex.class).addKey(name,Mapping.TEXT.getParameter()).addKey(age,Mapping.TEXT.getParameter()).buildMixedIndex("search")
```

有关文本和字符串搜索选项的详细信息，请参阅第24章“索引参数和全文搜索”，有关每个后端如何处理文本和字符串搜索的详细信息，请参阅针对正在使用的索引后端的文档部分。  

虽然索引定义示例与上面的复合索引类似，但它提供了更大的查询支持，并且可以应答以下任何查询。
```$xslt
g.V().has('name', textContains('hercules')).has('age', inside(20, 50))
g.V().has('name', textContains('hercules'))
g.V().has('age', lt(50))
g.V().has('age', outside(20, 50))
g.V().has('age', lt(50).or(gte(60)))
g.V().or(__.has('name', textContains('hercules')), __.has('age', inside(20, 50)))
```

混合索引支持全文搜索、范围搜索、地理搜索等。有关特定索引后端支持的谓词列表，请参阅第23章“搜索谓词和数据类型”。  

>注意:与复合索引不同，混合索引不支持唯一性。

##### 11.1.2.1. Adding Property Keys

属性键可以添加到现有的混合索引中，该索引允许后续查询在查询条件中包含此键。  
```$xslt
graph.tx().rollback()  //Never create new indexes while a transaction is active
mgmt = graph.openManagement()
location = mgmt.makePropertyKey('location').dataType(Geoshape.class).make()
nameAndAge = mgmt.getGraphIndex('nameAndAge')
mgmt.addIndexKey(nameAndAge, location)
mgmt.commit()
//Previously created property keys already have the status ENABLED, but
//our newly created property key "location" needs to REGISTER so we wait for both statuses
ManagementSystem.awaitGraphIndexStatus(graph, 'nameAndAge').status(SchemaStatus.REGISTERED, SchemaStatus.ENABLED).call()
//Reindex the existing data
mgmt = graph.openManagement()
mgmt.updateIndex(mgmt.getGraphIndex("nameAndAge"), SchemaAction.REINDEX).get()
mgmt.commit()
```
为了添加一个新定义的键，我们首先从管理事务中按其名称检索现有的索引，然后调用AddIndexKey方法将该键添加到此索引。  

如果在同一个管理事务中定义了添加的键，则该键将立即可供查询。如果属性键已在使用中，则添加该键需要执行reindex过程，以确保索引包含所有以前添加的元素。在重新索引过程完成之前，该键在混合索引中不可用。

##### 11.1.2.2. Mapping Parameters
当通过索引生成器或AddIndexKey方法向混合索引添加属性键时，可以选择指定参数列表，以调整属性值映射到索引后端的方式。有关每个索引后端支持的参数类型的完整列表，请参阅映射参数概述。

#### 11.1.3. Ordering

可以使用order().by()指令定义图形查询结果的返回顺序。order().by()方法需要两个参数：
- 用于排序结果的属性键的名称。结果将按此属性键的顶点或边的值排序。
- 排序顺序：递增或递减递减

例如: g.V().has('name', textContains('hercules')).order().by('age', decr).limit(10) 以大力神的名义找出十个最老的人。

使用order().by()时，请注意：
- 复合图索引本机不支持排序搜索结果。将检索所有结果，然后在内存中进行排序。对于大型结果集，这可能非常昂贵。
- 混合索引支持本机高效排序。但是，ORDER().BY()方法中使用的属性键必须事先添加到混合索引中，以获得本机结果排序支持。在order().by()键与查询键不同的情况下，这一点很重要。如果属性键不是索引的一部分，那么排序需要将所有结果加载到内存中。

### 11.1.4. Label Constraint
在许多情况下，只需要使用特定标签为顶点或边编制索引。例如，人们可能只想用神的名字来索引神，而不是用每个具有名称属性的顶点。定义索引时，可以使用索引生成器的indexonly方法将索引限制到特定顶点或边缘标签。下面为属性键名称创建一个复合索引，该索引只对标记为“上帝”的顶点进行索引。
```$xslt
graph.tx().rollback()  //Never create new indexes while a transaction is active
mgmt = graph.openManagement()
name = mgmt.getPropertyKey('name')
god = mgmt.getVertexLabel('god')
mgmt.buildIndex('byNameAndLabel', Vertex.class).addKey(name).indexOnly(god).buildCompositeIndex()
mgmt.commit()
//Wait for the index to become available
ManagementSystem.awaitGraphIndexStatus(graph, 'byNameAndLabel').call()
//Reindex the existing data
mgmt = graph.openManagement()
mgmt.updateIndex(mgmt.getGraphIndex("byNameAndLabel"), SchemaAction.REINDEX).get()
mgmt.commit()
```
标签限制同样适用于混合索引。当具有标签限制的复合索引被定义为唯一索引时，唯一性约束仅适用于指定标签的顶点或边上的属性。

### 11.1.5. Composite versus Mixed Indexes

1.使用复合索引进行完全匹配的索引检索。复合索引不需要配置或操作外部索引系统，通常比混合索引快得多。
   - 作为例外，当查询约束的不同值的数目相对较小，或者如果一个值预期与图中的许多元素相关联（即在选择性较低的情况下），则使用混合索引进行精确匹配。
2.对数值范围、全文或地理空间索引使用混合索引。此外，使用混合索引可以加快order().by()查询的速度。

## 11.2. Vertex-centric Indexes
顶点中心索引是每个顶点单独构建的局部索引结构。在大型图中，顶点可以有数千个入射边。遍历这些顶点的速度可能非常慢，因为必须检索大量的事件边缘子集，然后在内存中过滤以匹配遍历的条件。以顶点为中心的索引可以通过使用本地化的索引结构来只检索需要遍历的边来加速这种遍历。  

假设Hercules与数百个怪物作战，除了在神的介绍图中捕获的三个怪物。如果没有以顶点为中心的索引，查询那些在时间点10到20之间战斗的怪物需要检索所有的战斗边缘，即使只有少数匹配边缘。  
```$xslt
h = g.V().has('name', 'hercules').next()
g.V(h).outE('battled').has('time', inside(10, 20)).inV()
```
通过时间构建以顶点为中心的索引可以加快这种遍历查询的速度。注意，这个最初的索引示例已经作为名为边的索引存在于众神图中。因此，运行以下步骤将导致唯一性约束错误。  

```$xslt
graph.tx().rollback()  //Never create new indexes while a transaction is active
mgmt = graph.openManagement()
time = mgmt.getPropertyKey('time')
battled = mgmt.getEdgeLabel('battled')
mgmt.buildEdgeIndex(battled, 'battlesByTime', Direction.BOTH, Order.decr, time)
mgmt.commit()
//Wait for the index to become available
ManagementSystem.awaitRelationIndexStatus(graph, 'battlesByTime').call()
//Reindex the existing data
mgmt = graph.openManagement()
mgmt.updateIndex(mgmt.getGraphIndex("battlesByTime"), SchemaAction.REINDEX).get()
mgmt.commit()
```
这个例子建立了一个以顶点为中心的索引，该索引按时间递减的顺序对两个方向的战斗边缘进行索引。以顶点为中心的索引是根据特定的边缘标签构建的，该标签是索引构造方法的第一个参数。  

这个例子建立了一个以顶点为中心的索引，该索引按时间递减的顺序对两个方向的战斗边缘进行索引。以顶点为中心的索引是根据特定的边缘标签构建的，该标签是索引构造方法janusGraphManagement.buildEdgeIndex（）的第一个参数。该索引仅适用于该标签的边缘-在上面的示例中战斗。第二个参数是索引的唯一名称。第三个参数是建立索引的边缘方向。索引将只适用于沿此方向的边的遍历。在这个例子中，顶点中心索引是在两个方向上构建的，这意味着沿着有战斗边缘的时间限制的遍历可以由这个索引在进出方向上提供服务。Janusgraph将在战斗边缘的进出顶点上保持一个顶点中心索引。或者，我们可以定义索引，只适用于向外的方向，这样可以加速从大力神到怪物的移动，但不能反向移动。这只需要维护一个索引，因此需要一半的索引维护和存储成本。最后两个参数是索引的排序顺序和索引依据的属性键列表。排序顺序是可选的，默认为升序（即order.asc）。属性键列表必须非空，并定义索引给定标签边缘的键。顶点中心索引可以用多个键定义。

```$xslt
graph.tx().rollback()  //Never create new indexes while a transaction is active
mgmt = graph.openManagement()
time = mgmt.getPropertyKey('time')
rating = mgmt.makePropertyKey('rating').dataType(Double.class).make()
battled = mgmt.getEdgeLabel('battled')
mgmt.buildEdgeIndex(battled, 'battlesByRatingAndTime', Direction.OUT, Order.decr, rating, time)
mgmt.commit()
//Wait for the index to become available
ManagementSystem.awaitRelationIndexStatus(graph, 'battlesByRatingAndTime', 'battled').call()
//Reindex the existing data
mgmt = graph.openManagement()
mgmt.updateIndex(mgmt.getRelationIndex(battled, 'battlesByRatingAndTime'), SchemaAction.REINDEX).get()
mgmt.commit()
```
本例通过对有战斗力的边缘进行评级来扩展模式，并构建一个以顶点为中心的索引，该索引按照评级和时间的降序在向外的方向对有战斗力的边缘进行索引。注意，指定属性键的顺序很重要，因为以顶点为中心的索引是前缀索引。这意味着，战斗边缘是由第一次和第二次评级索引。
```$xslt
h = g.V().has('name', 'hercules').next()
g.V(h).outE('battled').property('rating', 5.0) //Add some rating properties
g.V(h).outE('battled').has('rating', gt(3.0)).inV()
g.V(h).outE('battled').has('rating', 5.0).has('time', inside(10, 50)).inV()
g.V(h).outE('battled').has('time', inside(10, 50)).inV()
```
因此，battlesbyratingandtime索引可以加速前两个查询，而不是第三个查询.  
为了支持不同的约束遍历，可以为同一个边缘标签构建多个以顶点为中心的索引。Janusgraph的查询优化器试图为任何给定的遍历选择最有效的索引。以顶点为中心的索引只支持相等和范围/间隔约束。

>注意:以顶点为中心的索引中使用的属性键必须具有显式定义的数据类型（即，不是object.class），该类型支持本机排序顺序。这不仅意味着它们必须实现可比较的，而且它们的序列化程序必须导入orderPreservingSerializer。当前支持的类型有Boolean、Uuid、Byte、Float、Long、String、Integer、Date、Double、Character和Short。  

如果顶点中心索引是根据在同一管理事务中定义的边缘标签构建的，则该索引将立即可用于查询。如果边缘标签已在使用中，则针对其构建以顶点为中心的索引需要执行重新索引过程，以确保索引包含所有以前添加的边缘。在重新索引过程完成之前，索引将不可用。

>注意:janusgraph自动建立索引的顶点为中心的边缘标签和属性键。这意味着，即使有成千上万的事件发生，诸如g.v(h).out(“母亲”)或g.v(h).values(“年龄”)这样的查询也能被本地索引有效地应答。

以顶点为中心的索引不能加速不受约束的遍历，这需要遍历特定标签的所有关联边。随着入射边数的增加，这些遍历将变慢。通常，这样的遍历可以重写为约束遍历，可以使用以顶点为中心的索引来确保可接受的规模性能。

### 11.2.1. Ordered Traversals
以下查询指定事件边缘的遍历顺序。使用locallimit命令为遍历的每个顶点检索边的子集（按给定顺序）。
```$xslt
h = g..V().has('name', 'hercules').next()
g.V(h).local(outE('battled').order().by('time', decr).limit(10)).inV().values('name')
g.V(h).local(outE('battled').has('rating', 5.0).order().by('time', decr).limit(10)).values('place')
```
第一个查询询问10个最近被大力神打败的怪物的名字。第二个问题是关于最近10次大力神之战中被评为5颗星的位置。在这两种情况下，查询都受属性键上的顺序的约束，并且限制了要返回的元素的数量。

如果顺序键与索引键匹配，并且请求的顺序（即增加或减少）与为索引定义的顺序相同，则以顶点为中心的索引也可以有效地回答此类查询。battlesbytime索引将用于回答第一个查询，battlesbyratingandtime将应用于第二个查询。注意，BattlesbyratingandTime索引不能用于回答第一个查询，因为为了使索引中的第二个键有效，必须存在对评级的相等约束。

>注意:有序顶点查询是对gremlin的janusgraph扩展，它导致冗长的语法，需要__()步骤将janusgraph结果转换回gremlin管道。


