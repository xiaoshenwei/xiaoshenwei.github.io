---
toc: true
title: "Logs With ElasticSearch"
categories:
  - log
tags:
  - elastic search
---
# Logs With ElasticSearch

## 概述

​    服务日志存储到elastic search 中，同时为每条日志添加字段app=服务名称，从而实现根据应用名称搜索相关日志。

## 部署

当前主流的日志系统选型:

- elastic search + logstash + kibana 
- grafana + loki + promtail
- clickhouse

 由于 当前系统 日志量 较小 单日日志量小于100G，同时满足`快速`的日志检索 需求， 将选用ELK作为日志系统。

### 架构

k8s 类型:

app-log > filebeat > logstash > es

非k8s：

app-log > syslog > logstash > es

## 管理

使用数据流自动管理es 索引，logstash 关键配置如下

```yaml
input {
  beats {
    port => 5044
  }
}

filter {
  if [kubernetes][namespace] !~ "dev|test|prd" {
    drop { }
  }
  if [kubernetes][container][name] =~ "etcd|fluent-bit" {
    drop { }
  }
  mutate {
    add_field => { "app" => "%{[kubernetes][container][name]}" }
  }
}

output {
  # file { path => "/tmp/debug.file" }
  elasticsearch {
    hosts => ["es1:9200", "es2:9200", "es3:9200"]
    user => "logstash"
    password => "xxx"
    data_stream => true
    data_stream_type => "logs"
    data_stream_dataset => "k8s"
    data_stream_namespace => "sf"
  }
}
```

建立索引模版:

logs-k8s 并设置 声明周期 7-days (数据保存7days)

## 搜索

默认情况下，Elasticsearch 按**相关性得分**对匹配的搜索结果进行排序，相关性得分衡量每个文档与查询的匹配程度。

在简单的日志搜索的场景下，我们并不关心相关性评分

DSL vs EQL vs KQL vs ES|QL ?

### DSL

Domain Specific Language based on JSON 

```json
{
  "query": { 
    "bool": { 
      "must": [
        { "match": { "title":   "Search"        }},
        { "match": { "content": "Elasticsearch" }}
      ],
      "filter": [ 
        { "term":  { "status": "published" }},
        { "range": { "publish_date": { "gte": "2015-01-01" }}}
      ]
    }
  }
}
```

Query context 查询上下文: *文档与此查询子句的匹配程度如何？* 计算文档的相关性分数

Filter context 过滤上下文： *此文档是否与此查询子句匹配？* 不计算文档相关性分数

Elasticsearch 将自动缓存常用的过滤器，以提高性能。

每当查询子句传递给`filter`参数（例如[`bool`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-bool-query.html)查询中的`filter`或`must_not`参数）时，过滤器上下文就会生效。

####  Bool query

- must 
- filter
- should
- must_not

##### must vs should

`must` 子句

- **功能**：文档必须满足 `must` 子句中的所有查询条件才能匹配。所有 `must` 子句的条件都需要被满足。

- 示例

  ```json
  {
    "query": {
      "bool": {
        "must": [
          { "match": { "title": "Elasticsearch" } },
          { "range": { "publish_date": { "gte": "2023-01-01" } } }
        ]
      }
    }
  }
  ```
  
  这个查询要求文档的标题中必须包含 “Elasticsearch” 并且发布日期必须在 2023 年 1 月 1 日或之后。

`should` 子句

- **功能**：文档可以满足 `should` 子句中的一个或多个条件。

- 示例

  ```
  json
  复制代码
  {
    "query": {
      "bool": {
        "should": [
          { "match": { "title": "Elasticsearch" } },
          { "match": { "description": "search engine" } }
        ]
      }
    }
  }
  ```
  
  这个查询会返回标题包含 “Elasticsearch” 或描述中包含 “search engine” 的文档。如果这两个条件都匹配，文档的相关性评分将更高。

演示：

```bash
PUT /my_blog
{
  "mappings": {
    "properties": {
      "title": { "type": "text" },
      "content": { "type": "text" },
      "tags": { "type": "keyword" }
    }
  }
}

POST /_bulk
{ "index": { "_index": "my_blog", "_id": "1" } }
{ "title": "Elasticsearch Basics", "content": "Learn about Elasticsearch and its features.", "tags": ["tech", "search"] }
{ "index": { "_index": "my_blog", "_id": "2" } }
{ "title": "Advanced Elasticsearch", "content": "Deep dive into advanced features and optimizations.", "tags": ["tech", "database"] }
{ "index": { "_index": "my_blog", "_id": "3" } }
{ "title": "Introduction to Kibana", "content": "Explore how Kibana integrates with Elasticsearch.", "tags": ["tech"] }

# match 
POST /my_blog/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "Elasticsearch" } }
      ]
    }
  }
}
POST /my_blog/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "Elasticsearch" } },
        { "match": { "content": "advanced" } }
      ]
    }
  }
}

# should
POST /my_blog/_search
{
  "query": {
    "bool": {
      "should": [
        { "term": { "tags": "tech" } },
        { "term": { "tags": "database" } }
      ]
    }
  }
}

DELETE /my_blog
```

#### Full text queries 全文检索

##### intervals query

你需要在文档中查找包含特定短语或词组的文本，并且要求词组中的词语之间的顺序和位置符合特定的要求。

场景: 在法律合同中查找包含“违约”并且“赔偿”出现在“违约”之后的条款，以便找到处理违约赔偿的相关条款。

##### match query

`match`查询是用于执行全文搜索的标准查询，包括模糊匹配选项。

```bash
GET /_search
{
  "query": {
    "match": {
      "message": {
        "query": "this is a testt",
        "operator": "and",
        "fuzziness": "AUTO", # 设置模糊性 testt > test
        "auto_generate_synonyms_phrase_query" : true # 设置同义词
      }
    }
  }
}
```

##### Match boolean prefix query

```
GET /_search
{
  "query": {
    "match_bool_prefix" : {
      "message" : "quick brown f"
    }
  }
}

# 等同于
GET /_search
{
  "query": {
    "bool" : {
      "should": [
        { "term": { "message": "quick" }},
        { "term": { "message": "brown" }},
        { "prefix": { "message": "f"}}
      ]
    }
  }
}
```

**场景描述**： 在搜索框中实现自动补全功能，用户输入一个词的前缀时，系统可以提供可能的补全选项。例如，在电商网站中，当用户输入“iph”时，系统应该建议“iPhone”、“iPhone 13”等可能的补全选项。

##### Match phrase query

**`match_phrase`**：要求词语按指定的顺序出现

##### Match phrase prefix query

词语按照顺序出现，并且最后一个视为前缀查询

##### Multi-match query

多个字段中搜索

```bash
GET /_search
{
  "query": {
    "multi_match" : {
      "query":    "this is a test", 
      "fields": [ "subject", "message" ] 
    }
  }
}
```

##### Combined fields

合并多个字段为一个虚拟字段进行搜索。 可视为 multi-match 的一个特殊设置。

##### Query string query

使用具有严格语法的解析器，根据提供的查询字符串返回文档。

```bash
PUT /my_index
{
  "mappings": {
    "properties": {
      "content": {
        "type": "text"
      }
    }
  }
}

POST /_bulk
{ "index": { "_index": "my_index", "_id": "1" } }
{ "content": "New York City is known as the Big Apple." }
{ "index": { "_index": "my_index", "_id": "2" } }
{ "content": "The Big Apple is a popular nickname for New York City." }
{ "index": { "_index": "my_index", "_id": "3" } }
{ "content": "Many tourists visit New York City to see the Statue of Liberty." }
{ "index": { "_index": "my_index", "_id": "4" } }
{ "content": "The Big Apple is also famous for its Broadway shows." }
{ "index": { "_index": "my_index", "_id": "5" } }
{ "content": "San Francisco is known for its Golden Gate Bridge." }



GET /my_index/_search
{
  "query": {
    "query_string": {
      "query": "(new york city) OR (big apple)",
      "default_field": "content"
    }
  }
}

GET /my_index/_search
{
  "query": {
    "query_string": {
      "query": "content: apple"
    }
  }
}

DELETE /my_index
```

参数:

query: 查询语句

default_field： 如果查询字符串中未提供字段，则默认搜索字段

allow_leading_wildcard： （可选，布尔值）如果`true` ，则通配符`*`和`?`允许作为查询字符串的第一个字符。默认为`true` 。

auto_generate_synonyms_phrase_query： 同义词查询，默认为true

default_operator: 如果未指定运算符，则用于解释查询字符串中的文本的默认布尔逻辑, 默认OR

fields：指定搜索的字段

语法示例：

- 其中`status`字段包含`active`

  ```
  status:active
  ```

- 其中`title`字段包含`quick`或`brown`

  ```
  title:(quick OR brown)
  ```

- 其中`author`字段包含确切的短语`"john smith"`

  ```
  author:"John Smith"
  ```

- `first name`字段包含`Alice` （注意我们需要如何用反斜杠转义空格）

  ```
  first\ name:Alice
  ```

- 其中`book.title` 、 `book.content`或`book.date`中的任何字段包含`quick`或`brown` （注意我们需要如何用反斜杠转义`*` ）：

  ```
  book.\*:(quick OR brown)
  ```

- 其中字段`title`具有任何非空值：

  ```
  _exists_:title
  ```

######  Wildcards 通配符

通配符搜索可以使用`?`对单个术语运行。替换单个字符， `*`替换零个或多个字符：

```
qu?ck bro*
```

###### Regular expressions 正则表达式

正则表达式模式可以通过将它们包裹在正斜杠（ `"/"` ）中来嵌入到查询字符串中：

```
name:/joh?n(ath[oa]n)/
```

##### Simple query string query

使用具有有限但容错语法的解析器，根据提供的查询字符串返回文档。

`simple_query_string`查询支持以下运算符：

- `+` signifies AND operation
  `+`表示 AND 运算
- `|` signifies OR operation
  `|`表示或运算
- `-` negates a single token
  `-`否定单个标记
- `"` wraps a number of tokens to signify a phrase for searching
  `"`包含多个标记来表示要搜索的短语
- `*` at the end of a term signifies a prefix query
  `*`位于术语末尾表示前缀查询
- `(` and `)` signify precedence
  `(`和`)`表示优先级

```json
GET /_search
{
  "query": {
    "simple_query_string": {
      "query": "foo | bar + baz*",
      "flags": "OR|AND|PREFIX"
    }
  }
}
```

演示

```json
GET /logs-k8s-sf/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "query_string": {
            "query": "message: *post*"
          }
        }
      ],
      "filter": [
        {
          "term": {
            "app.keyword": "alert-go-test"
          }
        }
      ]
    }
  },
  "size": 5
}
```



#### Term-level queries

您可以使用**术语级查询**根据结构化数据中的精确值查找文档。 不处理文本，直接匹配原始值。

##### exist query 

返回包含字段的任何索引值的文档。

```bash
{
  "query": {
    "exists": {
      "field": "user"
    }
  }
}
```

##### Fuzzy query

返回包含与搜索词相似的词的文档

- 更改字符 ( **b** ox → **f** ox)
- 删除一个字符(**b**lack → lack)
- 插入字符 (sic → sic **k** )
- 调换两个相邻字符 ( **ac** t → **cat** t)

##### Prefix query

```json
{
  "query": {
    "prefix": {
      "user.id": {
        "value": "ki"
      }
    }
  }
}
```

##### Range query

返回包含给定范围内的术语的文档。

```json
GET /_search
{
  "query": {
    "range": {
      "age": {
        "gte": 10,
        "lte": 20
      }
    }
  }
}
```

##### Regexp query

返回包含与正则表达式匹配的术语的文档。 [语法](https://www.elastic.co/guide/en/elasticsearch/reference/current/regexp-syntax.html)

```json
GET /_search
{
  "query": {
    "regexp": {
      "user.id": {
        "value": "k.*y"
      }
    }
  }
}
```

##### Term query 

返回在提供的字段中包含**确切**术语的文档。

**避免对text字段使用term查询**

```json
{
  "query": {
    "term": {
      "user.id": {
        "value": "kimchy"
      }
    }
  }
}
```

##### Terms query

返回在所提供的字段中包含一个或多个**精确**术语的文档

```json
GET /_search
{
  "query": {
    "terms": {
      "user.id": [ "kimchy", "elkbee" ],
      "boost": 1.0
    }
  }
}
```

##### Terms set query

`terms_set`查询与[`terms`查询](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-terms-query.html)相同，只是您可以定义返回文档所需的匹配术语的数量。例如：

例如：字段`programming_languages`包含已知编程语言的列表，例如求职者的`c++` 、 `java`或`php` 。您可以使用`terms_set`查询返回至少与其中两种语言匹配的文档。

```json
GET /job-candidates/_search
{
  "query": {
    "terms_set": {
      "programming_languages": {
        "terms": [ "c++", "java" ],
        "minimum_should_match": 2
      }
    }
  }
}
```

##### Wildcard query

返回包含与通配符模式匹配的术语的文档。

```json
GET /_search
{
  "query": {
    "wildcard": {
      "user.id": {
        "value": "ki*y"
      }
    }
  }
}
```

**`term`**：精确匹配单个值，用于确定字段值是否等于特定值。

**`terms`**：匹配多个可能的值中的任意一个，用于过滤多个值。

**`terms_set`**：确保字段值集与提供的值集合交集匹配，用于复杂的集合匹配需求。

### EQL

事件查询语言 (EQL) 是一种针对基于事件的时间序列数据（例如日志、指标和跟踪）的查询语言。

EQL 搜索要求搜索的数据流或索引包含*时间戳*字段。默认情况下，EQL 使用[Elastic Common Schema (ECS)](https://www.elastic.co/guide/en/ecs/8.11)中的`@timestamp`字段。

EQL 搜索还需要*事件类别*字段，除非您使用[`any`关键字](https://www.elastic.co/guide/en/elasticsearch/reference/current/eql-syntax.html#eql-syntax-match-any-event-category)搜索没有事件类别字段的文档。默认情况下，EQL 使用 ECS `event.category`字段。

```json
GET /logs-k8s-sf/_eql/search
{
  "query": """
    any where app.keyword == "alert-go-test"
  """
}
```

使用 EQL 的[序列语法](https://www.elastic.co/guide/en/elasticsearch/reference/current/eql-syntax.html#eql-sequences)来搜索一系列有序事件。按时间升序列出事件项，最近的事件列在最后：

```json
GET /logs-k8s-sf/_eql/search
{
  "query": """
    sequence 
    [any where app.keyword == "alert-go-test"]
    [any where app.keyword == "alert-go-prd"]
  """
}
```

[`with maxspan`](https://www.elastic.co/guide/en/elasticsearch/reference/current/eql-syntax.html#eql-with-maxspan-keywords)使用将匹配序列限制在一个时间跨度内：

```json
GET /logs-k8s-sf/_eql/search
{
  "query": """
    sequence with maxspan=1h
    [any where app.keyword == "alert-go-test"]
    [any where app.keyword == "alert-go-prd"]
  """
}
```

使用`!`匹配[缺失事件](https://www.elastic.co/guide/en/elasticsearch/reference/current/eql-syntax.html#eql-missing-events)：序列中在给定时间跨度内不满足条件的事件：

```json
GET /my-data-stream/_eql/search
{
  "query": """
    sequence with maxspan=1d
      [ process where process.name == "cmd.exe" ]
      ![ process where stringContains(process.command_line, "ocx") ]
      [ file where stringContains(file.name, "scrobj.dll") ]
  """
}
```

使用[`by`关键字](https://www.elastic.co/guide/en/elasticsearch/reference/current/eql-syntax.html#eql-by-keyword)来匹配共享相同字段值的事件：

```json
GET /my-data-stream/_eql/search
{
  "query": """
    sequence by process.pid with maxspan=1h
      [ process where process.name == "regsvr32.exe" ]
      [ file where stringContains(file.name, "scrobj.dll") ]
  """
}
```

使用场景:识别同一 IP 地址上的异常流量模式。使用 `by` 关键字可以帮助你分析在同一 IP 地址上发生的异常事件。

```bash
sequence by source_ip
  [event where event_type = "failed_login"]
  [event where event_type = "successful_login"]
```



使用[`until`关键字](https://www.elastic.co/guide/en/elasticsearch/reference/current/eql-syntax.html#eql-until-keyword)指定序列的过期事件。匹配序列必须在此事件之前结束

```json
GET /my-data-stream/_eql/search
{
  "query": """
    sequence by process.pid with maxspan=1h
      [ process where process.name == "regsvr32.exe" ]
      [ file where stringContains(file.name, "scrobj.dll") ]
    until [ process where event.type == "termination" ]
  """
}
```

**场景**：检测网络攻击行为，例如暴力破解攻击。你希望找到某个 IP 地址上的攻击行为，并确保这些攻击在网络安全防护事件发生之前完成。

```json
GET /my-data-stream/_eql/search
{
  "query": """
    sequence by source_ip with maxspan=1h
      [ failed_login where event.type == "failed_login" ]
      [ port_scan where event.type == "port_scan" ]
    until [ security_alert where event.type == "security_alert" ]
  """
}

```

您可以使用[`filter_path`](https://www.elastic.co/guide/en/elasticsearch/reference/current/common-options.html#common-options-response-filtering)查询参数来过滤API响应

```json
GET /logs-k8s-sf/_eql/search?filter_path=hits.sequences.events._source.message,hits.sequences.events._source.@timestamp
{
  "query": """
    sequence with maxspan=1h
    [any where app.keyword == "alert-go-test"]
    [any where stringContains(message,"post")]
  """
}
```

使用`runtime_mappings`参数在搜索期间提取和创建[运行时字段](https://www.elastic.co/guide/en/elasticsearch/reference/current/runtime.html)。使用`fields`参数在响应中包含运行时字段。

```json
GET /logs-k8s-sf/_eql/search?filter_path=hits.events._source.app,hits.events.fields.day_of_week
{
  "runtime_mappings": {
    "day_of_week": {
      "type": "keyword",
      "script": "emit(doc['@timestamp'].value.dayOfWeekEnum.toString())"
    }
  },
  "query": """
    any where app.keyword == "alert-go-test"
  """,
  "fields": [
    "@timestamp",
    "day_of_week"
  ]
}
```

`filter`参数使用Query DSL来限制运行 EQL 查询的文档。

```json
GET /my-data-stream/_eql/search
{
  "filter": {
    "range": {
      "@timestamp": {
        "gte": "now-1d/d",
        "lt": "now/d"
      }
    }
  },
  "query": """
    file where (file.type == "file" and file.name == "cmd.exe")
  """
}
```



EQL 搜索不支持[`text`](https://www.elastic.co/guide/en/elasticsearch/reference/current/text.html)字段全文检索。要搜索`text`字段，请使用 EQL 搜索 API 的[查询 DSL `filter`](https://www.elastic.co/guide/en/elasticsearch/reference/current/eql.html#eql-search-filter-query-dsl)参数。

支持函数

- add +
- between 之间
- cidrMatch： cidrMatch(source.address, "192.168.0.0/16") 
- concat 合并
- divide 除法
- endsWith
- indexOf
- length
- modulo 求模
- multiply 乘法
- number
- startsWith
- string
- stringContains
- substring 字符串切除
- subtract 减法

管道:

```json
process where process.name == "powershell.exe"
| head 3
```

```
process where process.name == "svchost.exe"
| tail 5
```



https://www.elastic.co/guide/en/elasticsearch/reference/current/eql-syntax.html#eql-basic-syntax

### KQL

Kibana 查询语言 (KQL) 是一种简单的基于文本的查询语言，用于过滤数据。

KQL 仅过滤数据，不参与聚合、转换或排序数据。

#### 过滤存在的字段

 http.request.method: *

#### 过滤匹配的文档

 http.request.method: GET

查询关键字、数字、日期或布尔字段时，值必须完全匹配，包括标点符号和大小写。但是，在查询文本字段时，Elasticsearch 会根据[字段的映射设置](https://www.elastic.co/guide/en/elasticsearch/reference/8.15/analysis.html)来分析提供的值。例如，要搜索`http.request.body.content` （ `text`字段）包含文本“空指针”的文档：

```json
http.request.body.content: null pointer
```

因为这是一个`text`字段，所以这些搜索词的顺序并不重要，甚至会返回包含“pointer null”的文档。要搜索其中术语按提供的顺序排列的`text`字段，请将值用引号引起来，如下所示

```
http.request.body.content: "null pointer"
```

#### 使用通配符过滤文档

 http.response.status_code: 4*

#### 否定查询

NOT http.request.method: GET

#### 组合查询

 http.request.method: GET OR http.response.status_code: 400

#### 匹配多个字段

datastream.*: logs

#### 查询嵌套字段

```json
{
  "user" : [
    {
      "first" : "John",
      "last" :  "Smith"
    },
    {
      "first" : "Alice",
      "last" :  "White"
    }
  ]
}
```

```
user:{ first: "Alice" and last: "White" }
```

### ES|QL

Elasticsearch 查询语言 (ES|QL) 提供了一种强大的方法来过滤、转换和分析存储在 Elasticsearch 中以及未来其他运行时中的数据

用户可以编写 ES|QL 查询来查找特定事件、执行统计分析并生成可视化效果。它支持广泛的命令和功能，使用户能够执行各种数据操作，例如过滤、聚合、时间序列分析等。

Elasticsearch 查询语言 (ES|QL) 使用“管道”(|) 逐步操作和转换数据。这种方法允许用户组合一系列操作，其中一个操作的输出成为下一个操作的输入，从而实现复杂的数据转换和分析。

#### 快速开始

```json
PUT sample_data
{
  "mappings": {
    "properties": {
      "client_ip": {
        "type": "ip"
      },
      "message": {
        "type": "keyword"
      }
    }
  }
}

PUT sample_data/_bulk
{"index": {}}
{"@timestamp": "2023-10-23T12:15:03.360Z", "client_ip": "172.21.2.162", "message": "Connected to 10.1.0.3", "event_duration": 3450233}
{"index": {}}
{"@timestamp": "2023-10-23T12:27:28.948Z", "client_ip": "172.21.2.113", "message": "Connected to 10.1.0.2", "event_duration": 2764889}
{"index": {}}
{"@timestamp": "2023-10-23T13:33:34.937Z", "client_ip": "172.21.0.5", "message": "Disconnected", "event_duration": 1232382}
{"index": {}}
{"@timestamp": "2023-10-23T13:51:54.732Z", "client_ip": "172.21.3.15", "message": "Connection error", "event_duration": 725448}
{"index": {}}
{"@timestamp": "2023-10-23T13:52:55.015Z", "client_ip": "172.21.3.15", "message": "Connection error", "event_duration": 8268153}
{"index": {}}
{"@timestamp": "2023-10-23T13:53:55.832Z", "client_ip": "172.21.3.15", "message": "Connection error", "event_duration": 5033755}
{"index": {}}
{"@timestamp": "2023-10-23T13:55:01.543Z", "client_ip": "172.21.3.15", "message": "Connected to 10.1.0.1", "event_duration": 1756467}


POST /_query?format=txt
{
  "query": """
FROM sample_data
  """
}
```

查询示例

``` 
FROM sample_data
| LIMIT 3

FROM sample_data
| SORT @timestamp DESC

FROM sample_data
| WHERE event_duration > 5000000

FROM sample_data
| WHERE message LIKE "Connected*"

FROM sample_data
| SORT @timestamp DESC
| LIMIT 3

FROM sample_data
| EVAL duration_ms = event_duration/1000000.0

使用STATS ... BY命令计算统计数据。例如，中位持续时间：
FROM sample_data
| STATS median_duration = MEDIAN(event_duration)

FROM sample_data
| STATS median_duration = MEDIAN(event_duration), max_duration = MAX(event_duration)

FROM sample_data
| STATS median_duration = MEDIAN(event_duration) BY client_ip



```

为了跟踪一段时间内的统计数据，ES|QL 允许您使用BUCKET函数创建直方图。 BUCKET创建人性化的存储桶大小，并为每行返回一个与该行所属的结果存储桶相对应的值。

将`BUCKET`与[`STATS ... BY`](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-commands.html#esql-stats-by)结合起来创建直方图。例如，要计算每小时的事件数：

```json
FROM sample_data
| STATS c = COUNT(*) BY bucket = BUCKET(@timestamp, 24, "2023-10-23T00:00:00Z", "2023-10-23T23:59:59Z")
```

数据处理

您的数据可能包含非结构化字符串，您希望将其[结构化](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-process-data-with-dissect-and-grok.html)以便更轻松地分析数据。例如，示例数据包含如下日志消息：

```txt
"Connected to 10.1.0.3"
```

```json
FROM sample_data
| DISSECT message "Connected to %{server_ip}"

FROM sample_data
| WHERE STARTS_WITH(message, "Connected to")
| DISSECT message "Connected to %{server_ip}"
| STATS COUNT(*) BY server_ip
```

#### 命令

ES|QL 支持以下源命令：

- [`FROM`](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-commands.html#esql-from)
- [`ROW`](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-commands.html#esql-row)
- [`SHOW`](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-commands.html#esql-show)

##### DISSECT

```
DISSECT input "pattern" [APPEND_SEPARATOR="<separator>"]
```

```
ROW a = "2023-01-23T12:15:00.000Z - some text - 127.0.0.1"
| DISSECT a "%{date} - %{msg} - %{ip}"
| KEEP date, msg, ip
```

##### Drop

删除某一列

```
FROM employees
| DROP height
```

##### Eval

```
ROW first_name = "hello", last_name = "world"
| EVAL full_name = concat(first_name, last_name)
```

##### GROK

```
GROK input "pattern"
```

```
ROW a = "2023-01-23T12:15:00.000Z 127.0.0.1 some.email@foo.com 42"
| GROK a "%{TIMESTAMP_ISO8601:date} %{IP:ip} %{EMAILADDRESS:email} %{NUMBER:num}"
| KEEP date, ip, email, num
```

##### KEEP

```
KEEP columns
```

```json
FROM employees
| KEEP emp_no, first_name, last_name, height
```

##### LIMIT

```json
LIMIT max_number_of_rows
```

要返回的最大行数。

##### RENAME

```json
RENAME old_name1 AS new_name1[, ..., old_nameN AS new_nameN]
```

```json
FROM employees
| KEEP first_name, last_name
| RENAME first_name AS fn, last_name AS ln
```

##### SORT

```json
SORT column1 [ASC/DESC][NULLS FIRST/NULLS LAST][, ..., columnN [ASC/DESC][NULLS FIRST/NULLS LAST]]
```

```json
FROM employees
| KEEP first_name, last_name, height
| SORT height DESC, first_name ASC
```

##### STATS ... BY

```json
STATS [column1 =] expression1[, ..., [columnN =] expressionN]
[BY grouping_expression1[, ..., grouping_expressionN]]
```

`STATS ... BY`处理命令根据公共值对行进行分组，并计算分组行上的一个或多个聚合值。如果省略`BY` ，则输出表仅包含一行，并且聚合应用于整个数据集。

```json
FROM employees
| STATS count = COUNT(emp_no) BY languages
| SORT languages
```

##### WHERE

```json
FROM sample_data
| WHERE @timestamp > NOW() - 1 hour
```

#### Data processing with DISSECT and GROK

您的数据可能包含您想要结构化的非结构化字符串。这使得分析数据变得更加容易。例如，日志消息可能包含您想要提取的 IP 地址，以便您可以找到最活跃的 IP 地址。

![](https://raw.githubusercontent.com/xiaoshenwei/xiaoshenwei.github.io/master/assets/images/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_e1645641-d5e1-4435-946d-34b5d88343f8.png)

`DISSECT`工作原理是使用基于分隔符的模式分解字符串。 `GROK`工作原理类似，但使用正则表达式。这使得`GROK`更强大，但通常也更慢。当数据可靠地重复时， `DISSECT`效果很好。当您确实需要正则表达式的强大功能时，例如当文本的结构因行而异时， `GROK`是更好的选择。

修饰符

##### 右填充修饰符 ( `->` )

```
ROW message="1998-08-10T17:15:42          WARN"
| DISSECT message "%{ts->} %{level}"

ROW message="[1998-08-10T17:15:42]          [WARN]"
| DISSECT message "[%{ts}]%{->}[%{level}]"
```

##### 附加修饰符 ( + )

```json
ROW message="john jacob jingleheimer schmidt"
| DISSECT message "%{+name} %{+name} %{+name} %{+name}" APPEND_SEPARATOR=" "
```

##### 添加顺序修饰符（ +和/n ）

```json
ROW message="john jacob jingleheimer schmidt"
| DISSECT message "%{+name/2} %{+name/4} %{+name/3} %{+name/1}" APPEND_SEPARATOR=","
```

##### 命名的跳过键 ( ? )

Dissect 支持忽略最终结果中的匹配项。这可以使用空键`%{}`来完成，但为了可读性，可能需要为该空键命名。

```json
ROW message="1.2.3.4 - - 30/Apr/1998:22:00:52 +0000"
| DISSECT message "%{clientip} %{?ident} %{?auth} %{@timestamp}"
```

