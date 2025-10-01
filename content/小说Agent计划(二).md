---
title: "小说Agent计划(二): 实现关系图"
date: 2025-09-25T18:09:00+08:00
---

{{< callout icon="sparkles" >}}
  与Blog同步开发的开源项目: [PlotWeave](https://github.com/shadow3aaa/PlotWeave)
{{< /callout >}}

上次我们提出了小说 Agent 的设计思路，接下来可以实际上手试试是否可行了。

<!--more-->

## 设计

### 关系图

关系图的设计是核心，必须要非常慎重。要对图做出规范，我的设计如下

- 实体: 可能是人物、地点、物品、组织等，不可以是事件。
- 边: 主要是事件，但也可以是抽象的关系，比如“朋友”、“母亲”、“成员”

任何事件都必须使用边来表示，不能直接使用实体保存事件。

还有一个需要明确的问题是，实体和边能否被删除？我的意思是，假设一个人物死了，或者情节上某个边永久的改变，能否将这个实体或者边从图中删除？

两方面都考虑一下，假如可以删除，那么允许删除实体，会导致什么？首先，删除实体会导致所有和这个实体相关的边都被删除或者悬垂，就好像整个世界都对它失忆了，这在小说中是很少见的。其次，删除实体会导致更改难以回溯，我认为起码实体是不应该被删除的。

那么边呢？边的删除似乎就不会导致相关实体或者边出现必须删除或者悬垂的问题。但是我依然认为边不应该被删除。因为边代表事件，事件一旦发生就不可能被抹去。即使是“忘记”这个事件，也应该是一个新的边，而不是删除原有的边。

反过来，激进一点，是否应该让边和实体的信息都不可变？初听这个想法似乎很疯狂，假如一个实体的信息改变了怎么办？但是仔细想想，现实中的改变也不会影响历史。比如一个人从穷人变成了富人，这个改变不会影响他过去的经历。**现在**的状态只是在**过去**的基础上**添加信息**，永远不是**减少信息**。

所以，我的结论是，实体和边都不应该被删除或者修改。任何改变只能通过添加新的实体或者边来实现。

#### 实体

实体的设计也很重要。要减少幻觉，我们需要尽可能的形式化表示实体。

对小说来说实体可以完全被形式化表示吗？我认为不可能。小说毕竟是抽象内容的创作，很多东西是无法被完全形式化的。

然而，这并不妨碍我们尽可能的将实体分类，我认为小说中实体大致可以分为以下几类

1. 人物
2. 地点
3. 物品
4. 组织

还是以《诡秘之主》为例，假设我们要表示主角克莱恩·莫雷蒂。

![克喵](克喵.png)

(图片来自网络，侵删)

首先我们需要一个唯一标识符，不然克莱恩·莫雷蒂这个名字可能会有歧义，比如这其实是他“穿越”后原主的名字，如果直接使用就无法指代原主了，这种冲突是不可接受的。

那么唯一标识符可以是什么呢？科幻小说里面可能会给它取一个很酷炫又有意象的名字，比如主角的id是 `Adam` 什么的但现实中我们只能使用 uuid 了 XD。

这里假设我们给克莱恩·莫雷蒂分配的 uuid 是 `123e4567-e89b-12d3-a456-426614174000`，那么目前我们可以这样表示他

- 类型: 人物
- id: 123e4567-e89b-12d3-a456-426614174000

接下来是其它信息，即属性，比如名字、性别、年龄、外貌、性格、背景等。

这里有两个选择，一是干脆放飞自我，让属性直接是Any数值对象，这样就可以表示任何属性了，非常的自由。但是这样就会导致属性之间没有任何约束，可能会出现一些不合理的情况，比如性别是数字，年龄是字符串，甚至忘记创建某个属性，而这属性又是非常重要的。

另一种选择是对属性进行严格的类型约束，比如必须且只能有名字、性别、年龄、归属地等属性等等。这问题更大，因为小说中人物的属性是非常多样化的，可能有些属性是某个小说中特有的，强制要求会导致无法表示这些属性，也许只创建特定题材可以考虑这样设计，但是我希望它是通用的。

除此之外，还有一个问题，根据之前的结论，实体的信息应该是不可变的，那么这个不可变怎么实现呢？我以为应该为属性设计版本号，如果要新增属性，首先看看描述这个属性的键是不是已经存在了，如果存在就不要插入新的键，而是使得这个键对应的，按版本排列的值列表新增一个值。这个值列表还不能只有值内容，我们也需要根据它定位在"历史"中的位置，也就是说它必须要有一个固定的属性用于描述这个值的时间点。就像下面这样

```python
class AttributeValue:
    def __init__(self, value: str, timestamp_desc: str):
        self.value = value
        self.timestamp = timestamp
# 克莱恩当前的序列
value = AttributeValue("占卜家", "在值夜者小队立功之后选择了称为占卜家")
```

这样查询时就能看到属性完整的变化历史而非只有当前值。

同时，我认为刚刚提出的约束问题也可以被此解决，要求增加属性时先检查表示相同概念的键是否存在，存在就新增值，不存在就新增键和值。除此之外不对属性做任何约束。这样就能兼顾灵活性，也避免了属性数量无限制膨胀的问题。

总结一下，实体的设计如下

- id: UUID
- type: str (人物、地点、物品、组织等)
- attributes: dict[str, list[AttributeValue]] (属性字典，键是属性名，值是按时间顺序排列的属性值列表，带有时间关系描述)

#### 边

边的设计，除了不能是实体以外和实体完全一样。这里我就不赘述了。

## 开发

ai 生态还是 python 比较完善，这里使用 uv 创建项目

```bash
uv init PlotWeave
```

Plot (剧情) Weave (编织)，意为剧情织机。

python 中有很多图论的库，最常用的应该是 networkx。这里使用的也是。

```bash
uv add networkx
```

让我们封装一个 class 来管理这个图

```python
class World:
    def __init__(self):
        self.graph: networkx.MultiDiGraph[UUID] = networkx.MultiDiGraph()
```

然后首先是添加实体的方法，首先引入实体类型Enum

```python
@unique
class EntityType(Enum):
    """
    实体类型

    PERSON: 人物
    PLACE: 地点
    ITEM: 物品
    ORGANIZATION: 组织
    """
    PERSON = 1
    PLACE = 2
    ITEM = 3
    ORGANIZATION = 4
```

然后定义实体类型

```python
@dataclass
class AttributeValue:
    value: str
    timestamp_desc: str

@dataclass
class Entity:
    type: EntityType
    id: UUID = field(default_factory=lambda: uuid4())
    attributes: dict[str, list[AttributeValue]] = field(default_factory=dict[str, list[AttributeValue]])
    def __hash__(self):
        return hash(self.id)
```

接下来定义边类型

```python
@dataclass
class Edge:
    id: UUID = field(default_factory=lambda: uuid4())
    attributes: dict[str, list[AttributeValue]] = field(default_factory=dict[str, list[AttributeValue]])

    def __hash__(self):
        return hash(self.id)
```

然后是添加实体的方法

```python
def add_entity(self, entity: Entity):
    if entity.id in self.graph:
        raise ValueError(f"Entity with id {entity.id} already exists.")
    self.graph.add_node(entity.id, entity=entity)
```

接下来是添加边的方法

```python
def add_edge(self, from_entity_id: UUID, to_entity_id: UUID, edge: Edge):
    if from_entity_id not in self.graph or to_entity_id not in self.graph:
        raise ValueError("Both entities must exist in the graph.")
    self.graph.add_edge(from_entity_id, to_entity_id, key=edge.id, edge=edge)
```

接着就是查询实体和边的方法了。简单的做个模糊查询效果大概率不会好，这里我认为应该引入向量搜索。这会是一个比较大的改动。

总结一下目前的设计为下面的python代码

```python
@unique
class EntityType(Enum):
    """
    实体类型枚举

    - PERSON: 人物
    - PLACE: 地点
    - ITEM: 物品
    - ORGANIZATION: 组织
    """

    PERSON = 0  # pyright: ignore[reportCallIssue]
    PLACE = 1  # pyright: ignore[reportCallIssue]
    ITEM = 2  # pyright: ignore[reportCallIssue]
    ORGANIZATION = 3  # pyright: ignore[reportCallIssue]

    def __hash__(self):
        return hash(self.value)


@dataclass
class AttributeValue:
    """
    属性值

    - value: 属性值
    - timestamp_desc: 本属性值开始生效的时间描述
    """

    value: str
    timestamp_desc: str

    def __hash__(self):
        return hash((self.value, self.timestamp_desc))


@dataclass
class Entity:
    """
    实体
    - id: 实体唯一标识符
    - type: 实体类型
    - attributes: 实体属性列表
    """

    type: EntityType
    id: UUID = field(default_factory=lambda: uuid4())
    attributes: dict[str, list[AttributeValue]] = field(
        default_factory=dict[str, list[AttributeValue]]
    )

    def __hash__(self):
        return hash(self.id)


@dataclass
class Edge:
    """
    边

    - id: 边唯一标识符
    - attributes: 边属性列表
    """

    id: UUID = field(default_factory=lambda: uuid4())
    attributes: dict[str, list[AttributeValue]] = field(
        default_factory=dict[str, list[AttributeValue]]
    )

    def __hash__(self):
        return hash(self.id)


class World:
    def __init__(self):
        self.graph: networkx.MultiDiGraph[UUID] = networkx.MultiDiGraph()

    def add_entity(self, entity: Entity):
        """
        添加实体

        如果实体已存在则抛出异常
        """
        if entity.id in self.graph:
            raise ValueError(f"Entity with id {entity.id} already exists.")
        self.graph.add_node(entity.id, entity=entity)
    
    def add_edge(self, from_entity_id: UUID, to_entity_id: UUID, edge: Edge):
        """
        添加边

        如果边已存在，或者起点或终点实体不存在则抛出异常
        """
        if from_entity_id not in self.graph or to_entity_id not in self.graph:
            raise ValueError("Both entities must exist in the graph.")
        if edge.id in self.graph:
            raise ValueError(f"Edge with id {edge.id} already exists.")
        self.graph.add_edge(from_entity_id, to_entity_id, key=edge.id, edge=edge)
```
