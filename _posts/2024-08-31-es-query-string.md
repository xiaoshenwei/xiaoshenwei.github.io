# ElasticSearch查询



## Query DSL

DSL (Domain Specific Language)

基于 JSON 的完整查询 DSL（领域特定语言）来定义查询。将查询 DSL 视为查询的 AST（抽象语法树），由两种类型的子句组成：

- **Leaf query clauses** 叶查询子句
  - Leaf 查询子句用于在特定字段中查找特定值，例如匹配（match）、精确值（term）或范围（range）查询。这些查询可以单独使用。
- **Compound query clauses** 复合查询子句： 复合查询子句包装其他叶查询**或**复合查询

### Query context 查询上下文

在查询上下文中，查询子句回答以下问题：“*此文档与此查询子句的匹配程度如何？”*除了判断文档是否匹配之外，查询子句还会计算`_score`数据字段中的相关性分数。

### Filter context 过滤上下文

在过滤器上下文中，查询子句回答“*此文档是否与此查询子句匹配？”的*问题。 ” 答案很简单，是或否——不计算分数。过滤器上下文主要用于过滤结构化数据，例如

- *Does this `timestamp` fall into the range 2015 to 2016?
  该`timestamp`是否在 2015 年至 2016 年范围内？*
- *Is the `status` field set to `"published"`*?
  *`status`字段是否设置为`"published"`* ？

每当查询子句传递给`filter`参数（例如[`bool`](https://www.elastic.co/guide/en/elasticsearch/reference/master/query-dsl-bool-query.html)查询中的`filter`或`must_not`参数、 [`constant_score`](https://www.elastic.co/guide/en/elasticsearch/reference/master/query-dsl-constant-score-query.html)查询中的`filter`参数或[`filter`](https://www.elastic.co/guide/en/elasticsearch/reference/master/search-aggregations-bucket-filter-aggregation.html)聚合）时，过滤器上下文就会生效。

### Example of query and filter contexts

以下是在`search` API 的查询和过滤上下文中使用查询子句的示例。此查询将匹配满足以下所有条件的文档：

- The `title` field contains the word `search`.
  `title`字段包含单词`search` 。
- The `content` field contains the word `elasticsearch`.
  `content`字段包含单词`elasticsearch` 。
- The `status` field contains the exact word `published`.
  `status`字段包含确切的单词`published` 。
- The `publish_date` field contains a date from 1 Jan 2015 onwards.
  `publish_date`字段包含从 2015 年 1 月 1 日起的日期。

```sql
GET /_search
{
  "query": { 1
    "bool": { 2
      "must": [
        { "match": { "title":   "Search"        }},
        { "match": { "content": "Elasticsearch" }}
      ],
      "filter": [ 3
        { "term":  { "status": "published" }},
        { "range": { "publish_date": { "gte": "2015-01-01" }}}
      ]
    }
  }
}
```

1. `query`参数指示查询上下文。
2. `bool`和两个`match`子句在查询上下文中使用，这意味着它们用于对每个文档的匹配程度进行评分。
3. `filter`参数指示过滤器上下文。它的`term`和`range`子句用于过滤器上下文中。它们会过滤掉不匹配的文档，但不会影响匹配文档的分数。

### Compound queries 复合查询

复合查询包装其他复合查询或叶查询，以组合它们的结果和分数，改变它们的行为，或者从查询切换到过滤上下文。

#### [`bool`查询](https://www.elastic.co/guide/en/elasticsearch/reference/master/query-dsl-bool-query.html)

用于组合多个叶或复合查询子句的默认查询，如`must` 、 `should` 、 `must_not`或`filter`子句。 `must`和`should`子句将它们的分数相结合——匹配子句越多越好——而`must_not`和`filter`子句在过滤器上下文中执行。

#### [boosting 查询](https://www.elastic.co/guide/en/elasticsearch/reference/master/query-dsl-boosting-query.html)

用于调整查询结果的相关性评分，从而提升或降低某些文档的排名。它通过对查询结果应用加权来实现这一目标。

Boosting 查询允许你在搜索中强调某些条件，同时抑制其他条件。这是通过定义一个“正向”查询和一个“反向”查询来实现的。正向查询匹配的文档会获得更高的评分，而反向查询匹配的文档则会被降权或从结果中排除。

[constant_score query](https://www.elastic.co/guide/en/elasticsearch/reference/master/query-dsl-constant-score-query.html)

**constant_score 查询**用于创建一个匹配查询，并为所有匹配的文档返回相同的评分。与其他查询不同，constant_score 查询不考虑文档的相关性评分，只关注是否匹配。

[`dis_max` query](https://www.elastic.co/guide/en/elasticsearch/reference/master/query-dsl-dis-max-query.html)

用于在多个查询中找到最高的评分，并将其作为最终结果的评分。它非常适用于当你希望将多个查询的结果结合起来，并只使用这些查询中的最高评分来决定文档的相关性时。

[`function_score` query](https://www.elastic.co/guide/en/elasticsearch/reference/master/query-dsl-function-score-query.html)

修改查询结果的评分。它允许你在基础查询的得分上应用一个或多个自定义评分函数，从而调整文档的相关性评分。



### Bool Query

- must: 子句（查询）必须出现在匹配的文档中，并将有助于得分
- filter: 用于指定必须匹配的查询条件，但与 `must` 不同的是，`filter` 子句的匹配不影响文档的评分。`filter` 子句主要用于过滤文档，而不调整文档的相关性得分。这些子句在过滤上下文中执行，这意味着评分被忽略，但这些子句会被考虑用于缓存，以提高查询效率。
- should: 子句（查询）应出现在匹配的文档中
- must_not: 子句（查询）不得出现在匹配文档中。子句在[过滤器上下文](https://www.elastic.co/guide/en/elasticsearch/reference/master/query-filter-context.html)中执行，这意味着忽略评分并考虑对子句进行缓存。由于评分被忽略，因此所有文档的分数都返回`0` 。

`bool`查询采用*“匹配越多越好”的*方法，因此每个匹配的`must` ”或`should` ”子句的分数将相加，以提供每个文档的最终`_score` 。

### 使用`bool.filter`评分

在`filter`元素下指定的查询对评分没有影响 - 分数返回为`0` 

此`bool`查询有一个`match_all`查询，它为所有文档分配`1.0`的分数。

# Full text queries 全文查询

[`intervals` query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-intervals-query.html): 全文查询，允许对匹配术语的顺序和接近度进行细粒度控制。

[`match`查询](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html) 用于执行全文查询的标准查询，包括模糊匹配和短语或邻近查询

### Match boolean prefix query
### Match phrase query
### Match phrase prefix query






## KQL

## ES|QL



