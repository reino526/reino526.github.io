---
sort: 9
---



# Elasticsearch

​	elasticsearch是一款非常强大的开源分布式搜索引擎，可以帮助我们从海量数据中快速找到需要的内容，elasticsearch结合kibana（数据可视化）、Logstash（数据抓取）、Beats（数据抓取），就是elastic stack(ELK，elastic技术栈)，被广泛应用在日志数据分析、实时监控等领域

​	elasticsearch是基于Lucene开发的，Lucene是一个Java语言的搜索引擎类库



## 倒排索引

​	elasticsearch采用倒排索引：

- 文档（document）：每条数据就是一个文档
- 词条（term）：文档按照语义分成的词语

​	与Mysql进行对比：

| MySQL  | Elasticsearch |                             说明                             |
| :----: | :-----------: | :----------------------------------------------------------: |
| Table  |     Index     |   索引（index），就是文档的集合，类似于数据库的表（table）   |
|  Row   |   Document    | 文档（document），就是一条条的数据，类似于数据库中的行（Row），都是JSON格式 |
| Column |     Field     | 字段（Field），就是JSON文档中的字段，类似于数据库中的列（Column） |
| Schema |    Mapping    | 映射（Mapping）是索引中文档的约束，例如字段类型约束，类似于数据库中的表结构（Schema） |
|  SQL   |      DSL      | DSL是elasticsearch提供的JSON风格的请求语句，用来操作elasticsearch，实现CRUD |

​	Mysql：擅长事务类型操作，可以确保数据的安全和统一性

​	ElasticSearch：擅长海量数据的搜索、分析、计算

​	所以未来Mysql需要给Elasticsearch进行数据同步



## 安装和部署

​	省略，太麻烦了，到时候用到再找



## DSL 操作

​	mapping常见属性：

- type：数据类型
  - 字符串：text（可分词的文本）、keyword（精确值，例如品牌、国家、ip地址）
  - 数值：long、integer、short、byte、double、float
  - 布尔值：boolean
  - 日期：date
  - 对象：object
- index：是否创建索引，默认为true
- analyzer：使用哪种分词器
- properties：该字段的子字段
- copy_to：将当前字段拷贝到指定的字段里



### 索引库操作

​	ES中通过Restful请求操作索引库、文档，请求内容用DSL语句来表示，创建索引库与mapping的DSL语法如下：

```http
PUT /索引库名称
{
	"mappings": {  # 定义mapping
        "properties": {  # 定义字段
            "字段名": {
                "type": "数据类型",
                "analyzer": "分词器选择",
                "index": 是否创建索引
            }
        }
    }
}
```

​	查看索引库：

```http
GET /索引库名称
```

​	删除索引库：

```http
DELETE /索引库名称
```

​	索引库和mapping一旦创建无法修改，但是可以添加新的字段，语法如下：

```http
PUT /索引库名称/_mapping
{
    "properties": {
        "新字段名": {
            "type": "字段数据类型"
        }
    }
}
```



### 文档操作

​	新增文档的DSL语法如下：

```http
POST /索引库名/_doc/文档id
{
    "字段1": "值1",
    "字段2": {
        "子属性1": "值2",
        "子属性2": "值3"
    }
}
```

​	查看文档语法：

```http
GET /索引库名/_doc/文档id
```

​	删除文档语法：

```http
DELETE /索引库名/_doc/文档id
```

​	修改文档：

- 方式一：全量修改，会删除旧文档，添加新文档：

```http
PUT /索引库名/_doc/文档id
{
	"字段1": "值1"
}
```

- 方式二：增量修改，修改指定字段值

```http
POST /索引库名/_update/文档id
{
	"doc": {
		"字段名": "新的值"
	}
}
```



## RestClient 操作

​	ES官方提供了各种不同语言的客户端用来操作ES,，这些客户端的本质就是组装DSL语句，通过http请求发送给ES，RestClient就是一个为Java语言设计的

​	

- 引入依赖

```xml
<dependency>
	<groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-high-level-client</artifactId>
</dependency>
```

- 初始化RestHighLevelClient

```java
RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(
		HttpHost.create("这里填写ES的http地址")
))
```



### 索引库操作

- 创建索引库

```java
void testCreateIndex() throws IOException{
    // 创建Request对象
    CreateIndexRequest request = new CreateIndexRequest("索引库名称");
    // 请求参数	，dsl是创建索引库的DSL语句（注意不需要写POST那些，那些属于是请求路径）
    request.source(dsl, XContentType.JSON);
    // 发起请求,indices方法表示拿到索引库的所有方法
    client.indices().create(request, RequestOptions.DEFAULT);
}
```

- 删除索引库

```java
// 创建Request对象
DeleteIndexRequest request = new DeleteIndexRequest("索引库名称");
// 发起请求
client.indices().delete(request, RequestOptions.DEFAULT);
```

- 判断索引库是否存在

```java
// 创建Request对象
GetIndexRequest request = new GetIndexRequest("索引库名称");
// 发起请求
client.indices().exists(request, RequestOptions.DEFAULT);
```



### 文档操作

- 添加文档

```java
// 创建Request对象
IndexRequest request = new IndexRequest("索引库名称").id("文档id");
// 请求参数	，准备json格式的文档数据（注意不需要写POST那些，那些属于是请求路径）
request.source("{\"字段名\": \"字段值\"}", XContentType.JSON);
// 发起请求
client.index(request, RequestOptions.DEFAULT);
```

- 查询文档

```java
// 创建Request对象
GetRequest request = new GetRequest("索引库名称", "文档id");
// 发起请求,得到结果
GetResponse response = client.get(request, RequestOptions.DEFAULT);
// 解析结果
String json = response.getSourceAsString();
```

- 修改文档（这个是局部更新，全量更新和刚刚添加文档没有区别）

```java
// 创建Request对象
UpdateIndexRequest request = new UpdateRequest("索引库名称", "文档id");
// 请求参数，每两个参数为一对key value
request.doc(
		"字段名1", "字段值1",
    	"字段名2", "字段值2"
);
// 发起请求
client.update(request, RequestOptions.DEFAULT);
```

- 删除文档

```java
// 创建Request对象
DeleteRequest request = new DeleteRequest("索引库名称", "文档id");
// 发起请求
client.delete(request, RequestOptions.DEFAULT);
```

- 批量新增文档

```java
// 创建Bulk请求
BulkRequest request = new BulkRequest();
// 添加要批量提交的请求，这里就添加了一个
request.add(new IndexRequest("索引库名称").id("文档id").source("json格式的source", XContentType.JSON));
// 发起bulk请求
client.bulk(request, RequestOptions.DEFAULT);
```



## DSL 查询

​	Elasticsearch提供了基于JSON的DSL来定义查询，常见的查询类型包括：

- 查询所有：查询出所有数据，一般测试用，也可以指定一些条件，例如：match_all
- 全文检索（full text）查询：利用分词器对用户输入内容分词，然后去倒排索引库中匹配，例如：
  - match_query
  - multi_match_query
- 精确查询：根据精确词条值查找数据，一般是查找keyword、数值、日期、boolean等类型字段，例如：
  - ids
  - range
  - term
- 地理（geo）查询：根据经纬度查询，例如：
  - geo_distance
  - geo_bounding_box
- 复合（compound）查询：复合查询可以将上述各种查询条件组合起来，合并查询条件，例如：
  - bool
  - function_score



### DSL 基本语法


```http
GET /索引库名称/_search
{
    "query": {
        "查询类型": {
			"查询条件": "条件值"
        }
    }
}
```

### 查询所有


```http
GET /索引库名称/_search
{
    "query": {
        "match_all": {}
    }
}
```

### 全文检索查询

常用于搜索框搜索：

- match查询：会对用户输入内容分词，然后去倒排索引库检索

```http
GET /索引库名称/_search
{
    "query": {
        "match": {
			"需要分词检索的字段": "搜索的值"
        }
    }
}
```

- multi_match：与match查询类似，只不过允许同时查询多个字段

```http
GET /索引库名称/_search
{
    "query": {
        "multi_match": {
        	"query": "搜索的值"
			"fields": ["需要分词检索的字段"]
        }
    }
}
```

### 精确查询

不会对搜索条件分词，常见的有term（根据词条精确值查询）、range（根据值的范围查询）

```http
GET /索引库名称/_search
{
    "query": {
        "term": {
			"需要检索的字段": {
				"value": "搜索的值"
			}
        }
    }
}
```

```http
GET /索引库名称/_search
{
    "query": {
        "range": {
			"需要检索的字段": {
				"gte": "大于等于哪个值",		# gt就是大于不等于
				"lte": "小于等于哪个值"
			}
        }
    }
}
```

### 根据经纬度查询

- geo_bounding_box：查询geo_point值落在某个矩形范围的所有文档

```http
GET /索引库名称/_search
{
    "query": {
        "geo_bounding_box": {
			"需要检索的字段": {
				"top_left": {
					"lat": 矩形左上纬度,
					"lon": 矩形左上经度
				},
				"bottom_right": {
					"lat": 矩形左上纬度,
					"lon": 矩形左上经度
				}
			}
        }
    }
}
```

- geo_distance：查询到指定中心点小于某个距离值的所有文档

```http
GET /索引库名称/_search
{
    "query": {
        "geo_distance": {
			"distance": {
				"distance": "距离多少以内",
				"需要检索的字段": "距离哪个点的经纬度"
			}
        }
    }
}
```

### 复合查询：

复合查询（compound）可以将其它简单查询组合起来。实现更复杂的搜索逻辑，如

- function score query：算分函数查询，可以控制文档相关性算分，控制文档排名，ES有几种相关性打分算法

  - TF-IDF：在ES的5.0之前，会随着词频增加而越来越大
  - BM25：在ES的5.0之后，会随着词频增加而增大，但增长曲线会趋于水平

  其中，算分函数结果称为function score，会与query score运算得到新算分，常见的算分函数有：

  - weight：给一个常量值，作为函数结果
  - field_value_factor：用文档中的某个字段值作为函数结果
  - random_score：随机生成一个值，作为函数结果
  - script_score：自定义计算公式，公示结果作为函数结果

  而boost_mode则是加权模式，定义function score与query score的运算模式，包括：

  - multiply：两者相乘，默认是这个
  - replace：用function score替换query score
  - 其它： sum、avg、max、min

```http
GET /索引库名称/_search
{
    "query": {
        "function_score": {
			"query": {
				"match": {
					"需要分词检索的字段": "搜索的值"  # 这里搜索文档后会根据相关性打分（query score）
				}
			},
			"functions": [
				{
					"filter": {
						"term": {
							"用于过滤条件的字段": "符合条件的字段值",
						}
						"weight": 10  # 算分函数
					}
				}
			],
			"boost_mode": "multiply"
        }
    }
}
```

- boolean query：布尔查询，是一个或多个查询子句的组合，子查询的组合方式有：

  - must：必须匹配每个子查询，类似“与”
  - should：选择性匹配子查询，类似“或”
  - must_not：必须不匹配，不参与算分（即查询子句不会算分），类似“非”
  - filter：必须匹配，不参与算分

```http
GET /索引库名称/_search
{
    "query": {
        "bool": {
        	"must": [
        		{"term": {"字段名": "满足条件的字段值"}}
        	],
        	"should": [
        		{"term": {"字段名": "满足条件的字段值"}}
        	]
        }
    }
}
```

​	

### 查询结果处理

- 排序

​	elasticsearch支持对搜索结果排序，默认是根据相关度算分（_score）来排序，可以排序的字段有：keyword类型、数值类型、地理坐标类型、日期类型，日期类型和剩余几种的方法不太一样

```http
GET /索引库名称/_search
{
    "query": {
        "match_all": {}
    },
    "sort":[
    	{
    		"字段名": "排序方式，desc降序，asc升序"
    	}
    ]
}
```

```http
GET /索引库名称/_search
{
    "query": {
        "match_all": {}
    },
    "sort":[
    	{
    		"_geo_distance": {
    			"字段名": "纬度,经度",
    			"order": "排序方式，desc降序，asc升序",
    			"unit": "显示的单位，例如km"
    		}
    	}
    ]
}
```

- 分页

​	elasticsearch默认情况下只返回Top10的数据，而如果要查询更多数据就需要修改分页参数，通过修改from、size参数来控制要返回的分页结果：

```http
GET /索引库名称/_search
{
	"query": {
		"match_all": {}
	},
	"from": 分页开始的位置，默认为0,
	"size": 期望文档总数，默认是10,
	"sort": [
		{"用于排序的字段": asc或desc}
	]
}
```

​	深度分页问题：ES是分布式的，而且对于ES的结构，如果只是想获取第990个开始后的10个，它必须得获取前1000个才能得到，所以会面临深度分页问题，这种时候只能聚合所有节点前1000个的结果再重新排序选取前1000个，但是如果搜索页数过深，或者结果集（from+size）越大，对内存和CPU的消耗也越高，因此ES设定结果集查询的上限是10000

​	深度分页问题解决方案：

​     1. search after：分页时需要排序，原理是从上一次的排序值开始查询下一页数据（就是把起点更换了，所以只能往后翻页不能往前）

​     2. scroll：原理将排序数据形成快照，保存在内存，这种方式耗内存，官方已经不推荐使用


- 高亮

​	高亮就是在搜索结果中把搜索关键字突出显示

```http
GET /索引库名称/_search
{
	"query": {
		"match": {
			"需要检索的字段": "字段值"
		}
	},
	"highlight": {
		"fields": {
			"需要高亮的字段": {
				"pre_tags": "标记高亮字段的前置标签",
				"post_tags": "标记高亮字段的后置标签"
			}
		}
	}
}
```

​	默认情况下，ES搜索字段必须与高亮字段一致，可以通过在post_tags下面加一个这个可以设置不一致也行

```json
"require_field_match": "false"
```



## RestClient 查询



### 查询

- match_all

```java
// 准备Request
SearchRequest request = new SearchRequest("索引库名称");
// 组织DSL参数
request.source().query(QueryBuilders, matchAllQuery());
// 发送请求得到响应结果
SearchResponse response = client.search(request, RequestOptions.DEFAULT);
// 解析结果
SearchHits searchHits = response.getHits();
// 查询的总条数
long total = searchHits.getTotalHits().value;
// 查询的结果数组
SearchHit[] hits = searchHits.getHits();
// 遍历
for (SearchHit hit : hits){
    // 得到source
    String json = hit.getSourceAsString();
}
```

- 全文检索查询

```java
// 和上面差不多，只需要改这一个地方
// 单字段查询
QueryBuilders.matchQuery("需要检索的字段", "字段值");
// 多字段查询
QueryBuilders.multiMatchQuery("字段值", "需要检索的字段1", "需要检索的字段2");
```

- 精确查询

```java
// 词条查询
QueryBuilders.termQuery("需要检索的字段", "字段值");
// 范围查询
QueryBuilders.rangeQuery("需要检索的字段").gte(大于等于的值).lte(小于等于的值);
```

- 复合查询

```java
// boolean query
// 创建布尔条件
BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();
// 添加must条件
boolQuery.must(QueryBuilders.termQuery("需要检索的字段", "字段值"));
// 添加filter条件
boolQuery.filter(QueryBuilders.rangeQuery("需要检索的字段").lte(小于等于的值));
// 添加进请求
request.source().query(boolQuery);
```



### 排序和分页

```java
// 查询
request.source().query(QueryBuilders.matchAllQuery());
// 分页
request.source().from(分页开始的位置).size(期望文档总数);
// 排序
request.source().sort("排序的字段", SortOrder.ASC);
```



### 高亮

```java
request.source().highlighter(new HighlightBuilder().field("需要高亮的字段").requireFieldMatch(false));

// 处理高亮就不能和上面一样，因为高亮的字段是在另一个地方
Map<String, HighlightField> highlightFields = hit.getHighlightFields();
if (!CollectionUtils.isEmpty(highlightFields)) {
	// 获取高亮字段的结果
    HighlightField highlightField = highlightFields.get("需要高亮的字段名");
    if (highlightField != null){
        // 取出第一个就是了
        String value = highlightField.getFragments()[0].string();
    }
}
```



## 数据聚合

​	聚合（aggregations）可以实现对文档数据的统计、分析、运算，参与聚合的字段不能是可分词的



### 分类

- 桶（Bucket）聚合：用来对文档做分组，其实就是分类
  - TermAggregation：按照文档字段值分组
  - Date Histogram：按照日期阶梯分组，例如一周为一组，或者一月为一组
- 度量（Metric）聚合：用以计算一些值，比如：最大值、最小值、平均值等
  - Avg：求平均值
  - Max：求最大值
  - Min：求最小值
  - Stats：同时求max、min、avg、sum等
- 管道（pipeline）聚合：其它聚合的结果为基础做聚合



### DSL 实现聚合

- 桶聚合

```http
GET /索引库名称/_search
{
	"size": 0,	// 设置size为0,结果中不包含文档，只包含聚合结果
	"aggs": {  // 定义聚合，可以定义好多个
		"聚合的名字": {
			"terms": {  // 聚合的类型
			 	"field": "参与聚合的字段",
			 	"size": 希望获取的聚合结果数量，默认是10,
			 	"order": {
			 		"_count": "asc"  // 结果按照_count（桶里的文档数）升序排列
			 	}
			}
		}
	}
	"query": {  // 这里可以设定聚合的范围
	
	}
}
```

- 度量聚合

```http
GET /索引库名称/_search
{
	"size": 0,	// 设置size为0,结果中不包含文档，只包含聚合结果
	"aggs": {  // 定义聚合，可以定义好多个
		"聚合的名字": {
			"terms": {  // 聚合的类型
			 	"field": "参与聚合的字段",
			 	"size": 希望获取的聚合结果数量，默认是10,
			 	"order": {
			 		"_count": "asc"  // 结果按照_count（桶里的文档数）升序排列
			 	}
			}
			"aggs": {  // 是外面那个聚合的子聚合，执行完外面的桶聚合就会执行里面的统计每一个桶
				"子聚合的名字": {
					"stats": {  // 聚合类型
						"field": "参与聚合的字段"
					}
				}
			}
		}
	}
	"query": {  // 这里可以设定聚合的范围
	
	}
}
```



### RestAPI实现聚合

```java
SearchRequest searchRequest = new SearchRequest("索引库名称");
request.source().size(0);
request.source().aggregation(
	AggregationBuilders.term("聚合的名字").field("参与聚合的字段").size(希望获取的聚合结果数量)
);
SearchResponse response = client.search(request, RequestOptions.DEFAULT);
// 解析聚合结果
Aggregations aggregations = response.getAggregations();
// 根据名称获取聚合结果
Terms terms = aggreations.get("聚合的名字");
// 获取桶
List<? extends Terms.Bucket> buckets = brandTerms.getBuckets();
// 遍历
for (Terms.Bucket bucket : buckets) {
	// 获取不同桶的key，即不同分类的不同值
    String key =  bucket.getKeyAsString();
}
```



## 自动补全

​	

### 拼音分词器

​	要实现根据字母做补全，就必须对文档按照拼音分词，需要安装elasticsearch的拼音分词插件

​	elasticsearch中分词器（analyzer）的组成包含三部分：

- character filters：在tokenizer之前对文本进行处理，例如删除字符、替换字符
- tokenizer：将文本按照一定的规则切割成词条（term）
- tokenizer filter：将tokenizer输出的词条做进一步处理，例如大小写转换、同义词处理、拼音处理等

​	只有在创建索引库的时候，通过settings来配置	自定义的analyzer：

```http
PUT /索引库的名称
{
	"settings": {
		"analysis": {
			"analyzer": {	// 自定义分词器
				"分词器名称": {
					"tokenizer": "分词规则，如ik_smart_word是ik分词器的智能分词，在这里可以填入自定义的过滤器名称",
					"filter": "拼音处理就是pinyin"
				}
			}
			"filter": {  // 自定义tokenizer filter
				"过滤器名称": {  
					"type": "过滤器类型，如pinyin",
					然后这里的配置可以去参考插件的文档来设置
				}	
			}
		}
	}
	"mappings": {
		"properties": {
			"字段名": {
				"type": "text",
				"analyzer": "在这里可以填入自定义的分词器名称"
			}
		}
	}
}
```

​	另外拼音分词器适合在创建倒排索引的时候使用，不能在搜索的时候使用，不然会搜到同音字，可以像这样配置

```http
PUT /索引库的名称
{	
	"mappings": {
		"properties": {
			"字段名": {
				"type": "text",
				"analyzer": "设置倒排索引时的分词器",
				"search_analyzer": "设置搜索时的分词器"
			}
		}
	}
}
```



### DSL 实现自动补全

​	elaticsearch提供了Completion Suggester查询来实现自动补全功能，这个查询会匹配以用户输入内容开头的词条并返回，为了提高补全查询的效率，对于文档中字段的类型有一些约束：

- 参与补全查询的字段必须是completion类型
- 字段的内容一般是用来补全的多个词条形成的数组

语法如下：

```http
GET /索引库名称/_search
{
	"suggest": {
		"自动补全的查询名称": {
			"text", "查询的关键字",
			"completion": {
				"field": "补全查询的字段",
				"skip_duplicates": 是否跳过重复的字段,
				"size": 获取前面多少条结果
			}
		}
	}
}
```



### RestAPI 实现自动补全

```java
SearchRequest searchRequest = new SearchRequest("索引库名称");
request.source().suggest(new SuggestBuilder()
                         .addSuggestion("自动补全的查询名称", SuggestBuilders
                                        .completionSuggestion("补全查询的字段")
                                        .prefix("查询的关键字")
                                        .skipDuplicates(true)
                                        .size(获取前面多少条结果)
                         ));
SearchResponse response = client.search(request, RequestOptions.DEFAULT);
// 处理结果
Suggest suggest = response.getSuggest();
// 根据名称获取补全结果
CompletionSuggestion suggestion = suggest.getSuggestion("自动补全的查询名称");
// 获取options并遍历
for (CompletionSuggestion.Entry.Option option : suggestion.getOption()) {
	// 获取option中的text，就是一个补全的词条
    String text = option.getText().string();
}
```



## 数据同步

​	elasticsearch中的数据来自于mysql数据库，因此mysql数据发生改变时，elasticsearch也必须跟着改变，这个就是elasticsearch与mysql之间的数据同步

​	方案一：同步调用：更改sql的操作调用更新索引库的接口

​	方案二：异步通知：更改sql的操作会发布消息，然后更新索引库的微服务监听到消息后就更新

​	方案三：监听binlog：sql更新时会输出binlog，利用类似canal的中间件监听binglog，发现更新后会通知索引库数据变更情况让其更新

​	方案一业务耦合度高，方案二低耦合但是依赖mq的可靠性，方案三完全解除服务间耦合但是开启binlog会增加数据库负担



## ES 集群

​	单机的Elasticsearch做数据存储，必然面临两个问题：海量数据存储问题、单点故障问题

- 海量数据存储问题：将索引库从逻辑上拆分为N个分片（shard），存储到多个节点
- 单点故障问题：将分片数据在不同节点备份（replica）

​	在创建docker-compose时将这几个es的environment中的cluster.name设置为一样的，就是属于同一个集群，然后在discovery.seed_hosts中写入其它几个es的ip地址（容器内互连可以直接用服务名），在cluster.initial_master_nodes中设置候选的主从节点

​	在创建索引库的时候可以在settings设置下加入number_of_shards来设置分片数量，加入number_of_replicas设置副本数量



### ES 集群的节点角色

|    节点类型     |       配置参数        | 默认值 |             节点职责             |
| :-------------: | :-------------------: | :----: | :------------------------------: |
| master eligible |      node.master      |  true  |            备选主节点            |
|      data       |       node.data       |  true  |             数据节点             |
|     ingest      |      node.ingest      |  true  |       数据存储之前的预处理       |
|  coordinating   | 上面都为false就是这个 |   无   | 路由请求到其它节点，合并节点结果 |

​	建议每个节点都有独立的角色，因为不同角色的硬件要求不同，而且同时担任多个角色资源分配不好

​	ES集群的脑裂：默认情况下，每个节点都是master eligible节点，因此一旦master节点宕机，其它候选节点会选举一个成为主节点，当主节点与其他节点网络故障时，可能发生脑裂问题（就是两个master节点），为了避免脑裂，需要要求选票超过(eligible节点数量+1)/2才能当选为主，因此eligible节点数量最好是奇数，早期这个配置需要人为的去修改discovery.zen.minimum_master_nodes，但在es7.0以后已经成为默认配置，因此一般不会发送脑裂问题



### ES 集群的分布式存储

​	当新增文档时，应该保存到不同分片，保证数据均衡，elasticsearch会通过hash算法来计算文档应该存储到哪个分片：

```java
shard = hash(_routing) % number_of_shards
```

​	其中_routing默认是文档的id，算法与分片数量有关，因此索引库一旦创建，分片数量不能修改

​	计算应该存储到哪个分片是主节点干的事，然后由协调节点通知

​	elasticsearch的查询分成两个阶段：

- scatter phase：分散阶段，coordinating node会把请求分发到每一个分片
- gather phase：聚集阶段，coordinating node汇总data node的搜索结果，并处理为最终结果集返回给用户



### ES 集群的故障转移

​	集群的master节点会监控集群中的节点状态，如果发现有节点宕机，会立即将宕机节点的分片数据迁移到其它节点（保证肯定有副本），确保数据安全，这个叫做故障转移
