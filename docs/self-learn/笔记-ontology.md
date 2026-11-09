# Ontology 学习笔记：从存储到推理查询

## 概览

Ontology（本体）描述一个领域的概念、属性、关系和公理。它不是一种数据库，也不是大语言模型。应用可以用它统一业务含义，并让推理器根据已有事实推导新的事实。

本教程面向了解数据库查询的读者，以“找出所有安全设备”为例，说明本体保存什么、如何保存、用什么代码查询、返回什么，以及如何接入 Agent。重点是：本体必须参与查询或推理；仅仅建立几张概念定义表，却在业务代码中完全不用它们，不能展示本体的价值。

## 目录

- [1. 区分定义、事实与业务规则](#1-区分定义事实与业务规则)
- [2. 本体如何改变查询结果](#2-本体如何改变查询结果)
- [3. 保存什么数据](#3-保存什么数据)
- [4. 运行完整示例](#4-运行完整示例)
- [5. 查询返回什么](#5-查询返回什么)
- [6. 实际业务价值](#6-实际业务价值)
- [7. 与 DeepSeek Harness 的关系](#7-与-deepseek-harness-的关系)
- [8. 阅读资料](#8-阅读资料)

-----

## 1. 区分定义、事实与业务规则

以采购和订单系统为例，应用中通常有几类不同的信息：

| 部分 | 保存内容 | 例子 |
|---|---|---|
| 本体定义 | 概念、属性、关系及其语义 | N95 口罩是呼吸防护用品的一种 |
| 业务事实 | 具体对象及其实际属性、关系 | 商品 P100 的类型是 N95 口罩 |
| 业务规则 | 可执行的业务判断条件 | 发货前，预留数量不得少于订购数量 |
| 数据映射 | 外部数据如何对应统一概念 | ERP 的商品分类字段对应商品的类型 |

本体定义与具体事实可以一起存储，也可以分开维护。业务规则和数据映射不自动等于本体公理；它们通常需要额外的程序、校验器或规则引擎执行。

知识图谱可以按本体组织，但有实体和关系不等于已经建立完整的本体系统。数据库 schema 规定字段和结构，本体进一步表达共享的领域语义；两者可以配合使用。

### 为什么“判断订单能否发货”不一定需要本体？

如果程序已经写死“查订单表、查库存表、比较数量”，普通 SQL 和业务代码就可以完成任务。即使旁边保存了“订单包含订单项”的定义，只要查询和判断没有使用它，这个定义就没有参与结果生成。

要展示本体的作用，需要一个明确的对照：相同业务事实和查询，在加入本体关系并执行推理后，可以得到原先没有显式记录的结果。

-----

## 2. 本体如何改变查询结果

采购人员问：“有哪些商品属于安全设备？”

业务系统只记录一条商品分类事实：

```text
P100 → 类型是 → N95口罩
```

本体定义两条类别关系：

```text
N95口罩 → 是一种 → 呼吸防护用品
呼吸防护用品 → 是一种 → 安全设备
```

推理器据此可以推导：

```text
P100 → 类型是 → 呼吸防护用品
P100 → 类型是 → 安全设备
```

没有直接插入“P100 是安全设备”这条业务记录。它来自类别继承规则和本体关系，因此本体实际参与了查询结果的生成。

这里使用 RDFS 的类别继承语义，是本体应用的基础例子，不覆盖完整的 OWL 建模能力。

-----

## 3. 保存什么数据

### 三元组代替固定的订单表结构

采用 RDF 时，基本记录单位是“主语—谓语—宾语”三元组。下面是需要保存的主要记录：

| 主语 | 谓语 | 宾语 | 类别 |
|---|---|---|---|
| N95Mask | rdf:type | rdfs:Class | 类别声明 |
| RespiratoryProtection | rdf:type | rdfs:Class | 类别声明 |
| SafetyEquipment | rdf:type | rdfs:Class | 类别声明 |
| N95Mask | rdfs:subClassOf | RespiratoryProtection | 本体关系 |
| RespiratoryProtection | rdfs:subClassOf | SafetyEquipment | 本体关系 |
| P100 | rdf:type | N95Mask | 业务事实 |

`rdf:type` 表示对象的类型，`rdfs:subClassOf` 表示类别之间的继承关系。三元组是逻辑数据模型，不代表所有 RDF 数据库内部都只有一张三列表。

### 文件和数据库如何选择

本示例把数据放在 RDFLib 的内存图中，并保存为 `equipment.ttl`。Turtle 是 RDF 的一种文本序列化格式；文件可以重新读入内存，也可以导入支持 RDF 的数据库。大规模或多人共享场景可由数据库提供持久化和查询接口。

保存的数据等价于下面这份 Turtle 文本，序列化时顺序和前缀名称可能不同：

```turtle
@prefix ex: <https://example.com/> .
@prefix rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .

ex:N95Mask a rdfs:Class ;
    rdfs:subClassOf ex:RespiratoryProtection .

ex:RespiratoryProtection a rdfs:Class ;
    rdfs:subClassOf ex:SafetyEquipment .

ex:SafetyEquipment a rdfs:Class .

ex:P100 a ex:N95Mask .
```

`a` 是 `rdf:type` 的简写。`ex:P100` 展开为 `https://example.com/P100`，这里用于唯一标识对象，不要求该地址必须提供网页。

-----

## 4. 运行完整示例

### 安装依赖

在 Python 虚拟环境中安装：

```sh
python -m pip install rdflib owlrl
```

RDFLib 负责保存 RDF 图和执行 SPARQL 查询；owlrl 提供推理实现。本例选择 `RDFS_Semantics`，不是执行完整 OWL 推理。

### 插入定义、插入事实并查询

将下面代码保存为 `ontology_example.py`：

```python
from rdflib import Graph, Namespace, RDF, RDFS
from owlrl import DeductiveClosure, RDFS_Semantics

EX = Namespace("https://example.com/")
g = Graph()

# 本体：声明类别及继承关系。
g.add((EX.N95Mask, RDF.type, RDFS.Class))
g.add((EX.RespiratoryProtection, RDF.type, RDFS.Class))
g.add((EX.SafetyEquipment, RDF.type, RDFS.Class))

g.add((EX.N95Mask, RDFS.subClassOf, EX.RespiratoryProtection))
g.add((EX.RespiratoryProtection, RDFS.subClassOf, EX.SafetyEquipment))

# 业务事实只登记最具体的商品类型。
g.add((EX.P100, RDF.type, EX.N95Mask))

# 此时尚未推理，文件保存的是原始定义和事实。
g.serialize(destination="equipment.ttl", format="turtle")

query = """
PREFIX ex: <https://example.com/>

SELECT ?product
WHERE {
    ?product a ex:SafetyEquipment .
}
"""


def find_safety_equipment():
    return sorted(str(row.product) for row in g.query(query))


print("推理前：", find_safety_equipment())

# 将可推导的事实加入内存图。
DeductiveClosure(RDFS_Semantics).expand(g)

print("推理后：", find_safety_equipment())
```

执行：

```sh
python ontology_example.py
```

输出：

```text
推理前： []
推理后： ['https://example.com/P100']
```

同一条查询在推理前没有结果，推理后可以找到 P100。这里 RDFLib 的普通查询匹配图中已有记录；`expand(g)` 显式增加推导事实，因此第二次查询可以匹配到它们。

### 怎样重新获取文件中的数据

在新程序中加载保存的原始数据：

```python
from rdflib import Graph
from owlrl import DeductiveClosure, RDFS_Semantics

g = Graph()
g.parse("equipment.ttl", format="turtle")
DeductiveClosure(RDFS_Semantics).expand(g)
```

随后执行同一条 SPARQL 查询即可。文件是在推理前保存的，所以重新加载时需要再次推理；也可以明确选择将推理后的图另存为文件。

### 如何验证确实用到了本体

在一个新的、尚未推理的图中，仅插入类别声明和 `P100 rdf:type N95Mask`，不插入两条 `subClassOf` 关系。执行同样的推理与查询，结果仍为空。

不要在已完成推理的图中只删除类别关系就进行这个对照，因为已经补入的推导事实可能仍然存在。该示例也不包含自动撤销过期推导事实的维护机制。

-----

## 5. 查询返回什么

查询分为两个不同目标：

| 查询目标 | 示例问题 | 返回内容 |
|---|---|---|
| 查询本体定义 | N95 口罩属于哪些上级类别？ | 类别和类别之间的关系 |
| 查询业务事实及推导结果 | 哪些商品是安全设备？ | 符合条件的商品标识及所选字段 |

本例的 `SELECT ?product` 返回查询行，每行绑定一个 `product`。Python 将 RDF 标识转为字符串后得到：

```json
["https://example.com/P100"]
```

使用 SPARQL HTTP 服务时，`SELECT` 可以返回标准 SPARQL JSON 结果，保留变量名和 URI、字面值等类型信息。工具服务可以再把它转换成业务 JSON：

```json
{
  "category": "SafetyEquipment",
  "products": [
    {
      "id": "P100",
      "recordedType": "N95Mask",
      "matchedType": "SafetyEquipment"
    }
  ]
}
```

这份业务 JSON 不是本体规定的格式，而是工具接口自行定义的。上面的查询只返回商品标识；`recordedType` 等额外字段需要服务查询原始事实或保留相应记录。推理后一个对象可能具有多个类型，不能随意把任意一个类型称为原始登记类型。

完整解释链同样需要服务另行提供；简单查询返回 P100，不表示推理器自动返回了“为什么命中”的说明。

-----

## 6. 实际业务价值

采购库中可能登记：

```text
P100：N95口罩
P200：防护眼镜
P300：绝缘手套
```

只要本体明确这些类别如何归属于“安全设备”，查询程序就不需要在每个应用中分别维护一份类别列表。新增“防化面罩”时，维护其“是呼吸防护用品的一种”的关系，并更新相关推导结果，既有的“查找安全设备”查询就可以覆盖新类别。

这种方式适合多个系统共享概念、类别层级和关系的场景，例如采购分类、设备部件管理、产品目录整合。它不意味着其他技术做不到：这个简单例子也能通过 SQL 递归查询实现。本体的价值是将领域语义明确保存下来，让通用推理器和多个应用依据相同定义得到一致结果。

本体不会自动保证输入数据准确，也不会自动获取实时库存。没有查到某个事实，不一定能断言该事实为假；例如缺少预留记录不等于已预留数量明确为零。实时数据获取、完整性校验和业务操作仍需要相应服务负责。

-----

## 7. 与 DeepSeek Harness 的关系

当前仓库检查未发现 Harness 内置领域本体推理系统。Session 事件类型描述运行记录，SQLite FTS5 检索文本，[Typert](../../packages/typert/README.md) 处理程序类型，都不能直接等同于领域本体。

[MCP 记忆接入文档](../user/guide/mcp-memory.zh.md) 提供外部知识图谱记忆服务的参考配置，但知识图谱记忆不自动提供本例中的 RDFS 推理能力。

可以将上述查询程序封装为插件或 MCP 服务，并向模型提供工具。下面的工具名称和返回字段是设计示例，不是仓库内置接口：

```text
用户：“有哪些安全设备？”
           ↓
模型调用 find_products(category="SafetyEquipment")
           ↓
业务服务查询经过本体推理的数据
           ↓
返回匹配商品的 JSON
           ↓
模型回答：“P100 是 N95 口罩，属于安全设备。”
```

本体定义、事实和推理器可以留在业务服务中，不必把整个知识库发给模型。模型通过工具获得当前任务需要的结果；匹配类别来自明确的本体关系，而不是仅凭模型常识猜测。

## 8. 阅读资料

- [W3C RDF 1.1 Primer](https://www.w3.org/TR/rdf11-primer/)：三元组、类别关系和 RDF 文件格式。
- [W3C SPARQL JSON 结果格式](https://www.w3.org/TR/sparql11-results-json/)：查询变量、结果行和类型信息。

## Dev Note

本文件是个人学习教程，使用最小的 RDFS 类别推理展示本体作用。示例文件 `equipment.ttl` 写入运行目录；正式业务系统需要另行设计数据同步、推导更新、访问权限和解释结果接口。
