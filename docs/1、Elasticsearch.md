[TOC]

## 全文检索

​				先对内容进行分词处理，根据空格对文章内容进行字符串拆分。合并重复的单词、去除标点符号、去除停用词，转换大小写，最终得到一个单词列表，需要记录单词和文章的对应关系。然后再根据单词列表创建索引。查询时直接在索引中查询，找到对应的关键词，再根据关键词和文章的对应关系展示文章列表。这个过程就叫做全文检索。

## Mapping

​			就是索引中文档格式的定义。
​			文档中包含的字段的名称、数据类型、是否索引、是否存储、是否分词。

- 创建索引库时定义mapping

  方法：put
  url：http://192.168.68.129:9200/{索引名称}

```json
{
    "mappings": {
        "properties": {
            "id": {
                "type": "long"
            },
            "title": {
                "type": "text",
                "analyzer": "standard",
                "store": true,
                "index": true
            },
            "mobile": {
                "type": "keyword",
                "store": true,
                "index": true
            },
            "comment": {
                "type": "text",
                "analyzer": "standard",
                "store": true,
                "index": true
            }
        }
    }
}
```

- 先创建索引库然后设置mappings

  方法：put、post
  url：http://192.168.68.129:9200/{索引名称}/_mapping

  ```
  {
      "properties": {
          "id": {
              "type": "long"
          },
          "title": {
              "type": "text",
              "analyzer": "standard",
              "store": "true",
              "index": true
          },
          "mobile": {
              "type": "keyword",
              "store": "true",
              "index": true
          },
          "comment": {
              "type": "text",
              "analyzer": "standard",
              "store": "true",
              "index": true
          }
      }
  }
  ```

  

## Setting

创建索引之后修改settings
				注意：分片数量在创建索引库之后就不能再修改了。
				可以修改副本的数量。

​			方法：
​					PUT
​				url：
​					http://192.168.68.129:9200/{索引名称}/_settings
​				例如：
​					http://192.168.68.129:9200/blog/_settings
​				请求体：
​					{
​						"number_of_replicas": "1"
​					}

## 文档管理

```
1、添加文档
	向索引中添加一行数据。
	使用json来表示。
	使用restful形式的api来实现。
		put：添加
		post：修改
		delete：删除
	
	方法：
		put
	url：
		http://192.168.68.129:9200/{索引}/_doc/{_id}
		文档的id（_id）推荐和真正数据的id保持一致。
	请求体：
		尽量和mapping设置的文档格式保持一致。
		{
			"id":1,
			"title":"这是一篇文章",
			"content":"xxxxx",
			"comment":"备注信息",
			"mobile":"13344556677"
		}
2、修改文档
	方法：
		POST
	url：
		http://192.168.68.129:9200/{索引}/_doc/{_id}
	请求体：
		{
			"id":1,
			"title":"这是一篇文章",
			"content":"xxxxx",
			"comment":"备注信息",
			"mobile":"13344556677"
		}
	修改的原理是，先删除后添加
3、删除文档
	方法：
		DELETE
	ulr:
		http://192.168.68.129:9200/{索引}/_doc/{_id}
4、根据_id取文档
	方法：
		GET
	url：
		http://192.168.68.129:9200/{索引}/_doc/{_id}
5、使用批处理_bulk
	方法：
		PUT、POST
	url：
		http://192.168.68.129:9200/{索引}/_bulk
	请求体：
		action对应的取值：
			create：创建一个文档，如果文档不存在就创建。
			index：创建一个新的文档，如果文档存在就更新。
			update：批量更新文档
			delete：批量删除，不需要有请求体。
		元数据：
			_index:要写入的索引信息
			_type:要写入的type
			_id:要写入文档的id
		{"index":{"_id":1}}
		{"id":1, "title":"这是一篇文章", "content":"xxxxx", "comment":"备注信息", "mobile":"13344556677"}
		{"index":{"_id":2}}
		{"id":2, "title":"这是一篇文章", "content":"xxxxx", "comment":"备注信息", "mobile":"13344556677"}
		{"index":{"_id":3}}
		{"id":3, "title":"这是一篇文章", "content":"xxxxx", "comment":"备注信息", "mobile":"13344556677"}
```

```
二、查询数据
1、查询的语法
	方法：
		POST
	url：
		http://192.168.68.129:9200[/{blog}][/{type}]/_search
	请求体：
		json形式的查询语句
		{
			"query":{
				"xxxx"
			}
		}
2、查询全部数据
	match_all查询

	{
		"query":{
			"match_all":{}
		}
	}
3、termQuery 关键词查询
	是所有查询中最级基本的一个查询。
	根据关键词进行查询，如果关键词在索引中存在那么就有结果，
	如果关键词不存在就查询不到结果。ES不会再次对查询的内容进行分词处理。
	需要指定两部分内容：
		1）要查询的关键词
		2）要查询的字段
	{
		"query":{
			"term":{
				"title":"java"
			}
		}
	}

	默认使用的是standard分词器。处理英文根据空格进行分词处理。如果处理中文，是一个汉字一个关键词。
		原文：传苹果正开发新Apple TV 或集成音响和摄像头
		分词结果：
			传
			苹
			果
			正
			开
			发
			新
			Apple
			TV
			或
			集
			成
			音
			响
			和
			摄
			像
			头
4、QueryString查询，根据查询字符串查询
	查询条件可以指定一个字符串，在查询之前，可以对查询条件进行分词处理，然后基于分词之后的结果再次查询。
	{
		"query":{
			"query_string":{
				"default_field":"title",
				"query":"传苹果正开发新Apple TV 或集成音响和摄像头"
			}
		}
	}
5、match查询
	功能和query_string相同。
	{
		"query":{
			"match":{
				"title":"传苹果正开发新Apple TV 或集成音响和摄像头"
			}
		}
	}
6、multi_match查询
	可以指定在多个字段中查询
	{
		"query":{
			"multi_match":{
				"query":"苹果",
				"fields":["title","content"]
			}
		}
	}
7、bool查询
	1)组合条件查询
	
	逻辑关系：
		must：必须满足，相当于是AND
		should：应该满足，相当于OR
		must_not:必须不能满足，相当于NOT
	语法：
		{
			"query":{
				"bool":{
					"must":[],
					"should":[],
					"must_not":[],
					"filter":[]
				}
			}
		}

		案例：
		{
			"query":{
				"bool":{
					"must":[
						{
							"match":{
								"title":"apple"
							}
						},
						{
							"match":{
								"content":"apple"
							}
						}
					]
				}
			}	
		}
		
		2)filter过滤查询
			在bool查询的filter节点中可以包含多个查询条件，条件之间层层过滤。
			也可以直接使用filter进行数据的查询。filter查询是不进行打分处理。查询性能好于query。

			相关度排序：
				SEO：搜索引擎优化。

				两个指标：
					TF：关键词在文章中出现的频率。TF越大相关度越高。
					DF：所有文档中关键词出现的频率。DF越大相关度越低。例如 and
				根据TF和DF计算出一个相关度的得分，得分越高相关度越高，文档根据相关度得分进行降序排列。

		{
			"query":{
				"bool":{
					"filter":[
						{
							"term":{
								"title":"apple"
							}
						}
					]
				}
			}	
		}
8、高亮处理
	在查询结果中将查询的关键词左右两边分别加上成对的html标签。
	高亮的处理在查询条件中指定。
	{
		"query":{
			"bool":{
				"must":[
					{
						"term":{
							"title":"apple"
						}
					}
				]
			}
		},
		"heightligth":{
			--设置高亮显示的字段
			"fields":{
				"title":{},
				"content":{}
			}
			--设置关键词的前缀
			"pre_tag":"<em>",
			--设置关键词的后缀
			"post_tag":"</em>"
		}	
	}
9、查询结果分页
	在query查询条件中增加两个属性
		from：起始的行号，从0开始
		size：每页显示的记录数量
	POST /blog/_search
	{
		"query":{
			"multi_match":{
				"query":"苹果正开发",
				"fields":["title","content"]
			}
		},
		"highlight": {
		  "fields": {
		    "title": {}
		    , "content": {}
		  },
		  "pre_tags": "<em>",
		  "post_tags": "</em>"
		},
		"from": 10,
		"size": 5
	}
```

```
三、中文分词器
中文分词器都是国产的。
Ik-analyzer

1、Ik的使用方法
	1）下载ES对应版本的ik分词器
	2）把分词器解压缩
	3）把解压之后的目录放到{ES}/plugin目录下
	4）重启ES
2、分词器的测试方法
	方法：
		POST
	url：
		http://192.168.68.129:9200/_analyze
	方法体：
		{
			"analyzer":"standard",
			"text":"and productivity has made it the world's most popular Java framework."
		}

	IK一旦安装之后有两个分词算法：
		ik_smart：快速分词，速度快，粒度比较粗。
		ik_max_word：最大数量分词，速度慢，粒度细。
3、索引一旦创建完毕不能修改分词器的
	如果使用中文分词，应该在创建索引时，设置mapping的过程中指定使用中文分词器。
	
四、field的数据类型
	数值类型：
		int
		long
		float
		double
	字符串：
		text：需要分词的字段必须使用text，只有text类型才能支持分词器。
		keyword：不需要对字段的内容进行分词处理时，可以使用keyword数据类型。
			例如：身份证号、手机号、订单号等。
	日期：
		data

	字段的三个属性：
		是否分词：
			是否是text类型。例如文章的title、content都需要分词。
		是否索引：
			是否对field的内容进行索引。如果text数据类型一定需要创建索引，分词之后一定要创建索引。
			不分词也可以把field的内容添加到索引中，使用keyword数据类型。
			也可以索引field中的内容。
				例如文件的path，不需要分词，不需要索引，只需要存储即可。
				"path":{
					"type":"keyword",
					"index":false,
					"store":true
				}
		是否存储
			定义field时，store属性是否是true。
			如果是true那么就存储，false不存储。
			无论是否存储，不影响分词、创建索引、搜索。

			影响的范围就是是否能在查询结果中看到原始内容。
```



##  安装Elastticsearch	

### docker可视化管理工具Portainer 

```
#拉取镜像
docker pull docker.io/portainer/portainer

#根据镜像id创建容器
docker run -d -p 9000:9000 --name portainer --privileged=true  --restart always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data 580c0e4e98b0
```

### 使用docker安装

```
#拉取镜像 
docker pull elasticsearch:7.10.1 

#创建容器 
docker create --name elasticsearch --net host -e "discovery.type=single-node" -e "network.host=192.168.232.130" elasticsearch:7.10.1 

#启动 docker start elasticsearch 

#查看日志 
docker logs elasticsearch
```

### ES客户端工具elasticsearch-head安装

```
#拉取镜像 
docker pull mobz/elasticsearch-head:5 

#创建容器 
docker create --name elasticsearch-head -p 9100:9100 mobz/elasticsearch-head:5 

#启动容器 
docker start elasticsearch-head
```

### 使用Docker Compose启动多节点集群

1. ###### 创建一个docker-compose.yml文件

```yaml
version: '2.2'
services:
  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.12.0
    container_name: es01
    environment:
      - node.name=es01
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es02,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data01:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
    networks:
      - elastic
  es02:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.12.0
    container_name: es02
    environment:
      - node.name=es02
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data02:/usr/share/elasticsearch/data
    networks:
      - elastic
  es03:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.12.0
    container_name: es03
    environment:
      - node.name=es03
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es02
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data03:/usr/share/elasticsearch/data
    networks:
      - elastic

volumes:
  data01:
    driver: local
  data02:
    driver: local
  data03:
    driver: local

networks:
  elastic:
    driver: bridge
```

