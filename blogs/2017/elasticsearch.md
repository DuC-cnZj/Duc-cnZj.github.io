---
title: elasticsearch
date: '2021-05-12 12:54:14'
sidebar: false
categories:
 - elasticsearch
tags:
 - elasticsearch
publish: true
---



## docker 安装 &#x1f642;

**elasticsearch：**`docker pull docker.elastic.co/elasticsearch/elasticsearch:6.3.2`

**elastic 图形界面：** `docker pull docker.elastic.co/kibana/kibana:6.3.2` 

> 这里说明一下，iK分词版本最新是 6.3.2 所以 es 最好也是这个版本，否则有可能无法兼容 Ik

```yaml
version: '3'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.3.2
    container_name: elasticsearch
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - discovery.type=single-node
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata1:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
    networks:
      - esnet
  kibana:
    image: docker.elastic.co/kibana/kibana:6.3.2
    networks:
      - esnet
    ports:
      - 5601:5601
    environment:
      XPACK_SECURITY_ENABLED: 'false'
      SERVER_HOST: "0.0.0.0"
      SERVER_NAME: kibana
      ELASTICSEARCH_URL: http://elasticsearch:9200
    depends_on:
      - elasticsearch

volumes:
  esdata1:
    driver: local

networks:
  esnet:

```

## Ik 分词 &#x1f353;

> 国人写的一个包 [github 地址](https://github.com/medcl/elasticsearch-analysis-ik)

*把这个插件安装到 elasticsearch 中*

1. 下载到响应的 `Dockerfile` 目录

2. Dockerfile

3. ```dockerfile
   FROM docker.elastic.co/elasticsearch/elasticsearch:6.3.2
   
   USER root
   COPY elasticsearch-analysis-ik-6.3.2.zip /usr/share/elasticsearch/plugins/elasticsearch-analysis-ik-6.3.2.zip
   RUN mkdir /usr/share/elasticsearch/plugins/ik \
       && unzip /usr/share/elasticsearch/plugins/elasticsearch-analysis-ik-6.3.2.zip -d /usr/share/elasticsearch/plugins/ik \
       && rm /usr/share/elasticsearch/plugins/elasticsearch-analysis-ik-6.3.2.zip
   
   ```

4. `docker build -t duccnzj/elastic:latest .`

5. 你可以直接拉取的我 `docker` 镜像, `docker pull duccnzj/elastic:latest`



## 集成 IK 分词之后的 `docker-compose.yml`

```yaml
version: '3'
services:
  elasticsearch:
    image: duccnzj/elastic:latest
    container_name: elasticsearch
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - discovery.type=single-node
    volumes:
      - esdata1:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
    networks:
      - esnet
  kibana:
    image: docker.elastic.co/kibana/kibana:6.3.2
    networks:
      - esnet
    ports:
      - 5601:5601
    environment:
      XPACK_SECURITY_ENABLED: 'false'
      SERVER_HOST: "0.0.0.0"
      SERVER_NAME: kibana
      ELASTICSEARCH_URL: http://elasticsearch:9200
    depends_on:
      - elasticsearch

volumes:
  esdata1:
    driver: local

networks:
  esnet:

```

## 集成到 laravel

### 安装 es 扩展包 

**别问我为什么选它，因为这个包用着舒服：**[scout-elasticsearch-driver](https://github.com/babenkoivan/scout-elasticsearch-driver#search-rules)

基本配置略。。



常见的 `query` 查询语句

> 地址：[查询语句](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_most_important_queries.html)

```json
// match_all 查询
// 匹配所有文档
{ "match_all": {}}

//  match 查询 (只能单个字段查询) 
{ "match": { "tweet": "About Search" }}
{ "match": { "age":    26           }}
{ "match": { "date":   "2014-09-01" }}
{ "match": { "public": true         }}
{ "match": { "tag":    "full_text"  }}

// multi_match 查询
{
    "multi_match": {
        "query":    "full text search",
        "fields":   [ "title", "body" ]
    }
}

// range 查询
{
    "range": {
        "age": {
            "gte":  20,
            "lt":   30
        }
    }
}

// term 查询被用于精确值 匹配
{ "term": { "age":    26           }}
{ "term": { "date":   "2014-09-01" }}
{ "term": { "public": true         }}
{ "term": { "tag":    "full_text"  }}

// terms 查询和 term 查询一样，但它允许你指定多值进行匹配
{ "terms": { "tag": [ "search", "full_text", "nosql" ] }}

// exists 查询和 missing 查询被用于查找那些指定字段中有值 (exists) 或无值 (missing) 的文档
// title 字段有值的数据
{
    "exists":   {
        "field":    "title"
    }
}

// 多查询组合
// must 文档 必须 匹配这些条件才能被包含进来。
// must_not 文档 必须不 匹配这些条件才能被包含进来。
// should 如果满足这些语句中的任意语句，将增加 _score ，否则，无任何影响。它们主要用于修正每个文档的相关性得分。
// filter 必须 匹配，但它以不评分、过滤模式来进行。这些语句对评分没有贡献，只是根据过滤标准来排除或包含文档。
{
    "bool": {
        "must":     { "match": { "title": "how to make millions" }},
        "must_not": { "match": { "tag":   "spam" }},
        "should": [
            { "match": { "tag": "starred" }},
            { "range": { "date": { "gte": "2014-01-01" }}}
        ]
    }
}

{
    "bool": {
        // 一定有
        "must":     { "match": { "title": "how to make millions" }},
        // 一定不能有
        "must_not": { "match": { "tag":   "spam" }},
        // 有了可以加分
        "should": [
            { "match": { "tag": "starred" }}
        ],
        // 在上面数据中筛选出匹配的
        "filter": {
          "range": { "date": { "gte": "2014-01-01" }} 
        }
    }
}

{
    "bool": {
        "must":     { "match": { "title": "how to make millions" }},
        "must_not": { "match": { "tag":   "spam" }},
        "should": [
            { "match": { "tag": "starred" }}
        ],
        "filter": {
          "bool": { 
              "must": [
                  { "range": { "date": { "gte": "2014-01-01" }}},
                  { "range": { "price": { "lte": 29.99 }}}
              ],
              "must_not": [
                  { "term": { "category": "ebooks" }}
              ]
          }
        }
    }
}

{
    "constant_score":   {
        "filter": {
            "term": { "category": "ebooks" } 
        }
    }
}

// 搜索user字段没有值的数据
{
    "query": {
        "bool": {
            "must_not": {
                "exists": {
                    "field": "user"
                }
            }
        }
    }
}
```





## 以下是案例，仅供参考 &#x1f3c7;

```
SCOUT_QUEUE=true
SCOUT_DRIVER=elastic
SCOUT_ELASTIC_HOST=elasticsearch:9200
```



```php
<?php

namespace App\ES;

use ScoutElastic\IndexConfigurator;
use ScoutElastic\Migratable;

class CompanyIndexConf extends IndexConfigurator
{
    use Migratable;

    // It's not obligatory to determine name. By default it'll be a snaked class name without `IndexConfigurator` part.
    protected $name = 'company';

    // You can specify any settings you want, for example, analyzers.
    protected $settings = [
        "number_of_shards" => "1",
        "number_of_replicas" => "1",
        'analysis' => [
            'analyzer' => [
                'es_std' => [
                    'type' => 'standard',
                    'stopwords' => '_spanish_'
                ]
            ]
        ]
    ];
}

```

```php
<?php

namespace App;

use App\ES\CompanyRule;
use ScoutElastic\Searchable;
use Illuminate\Database\Eloquent\Model;

class Company extends Model
{
    use Searchable;

    protected $appends = ['highlight_vision', 'highlight_name'];
    protected $searchRules = [
        CompanyRule::class
    ];

    /**
     * @var string
     */
    protected $indexConfigurator = \App\ES\CompanyIndexConf::class;

    /**
     * @var array
     */
    protected $mapping = [

      'properties' => [
        'name' => [
            'type' => 'text',
            "analyzer" => "ik_max_word",
            "search_analyzer" => "ik_max_word",
            "boost" => 3,
            'fields' => [
                'raw' => [
                    'type' => 'keyword',
                    'ignore_above' => 256
                    ]
                ]
            ],
            'city' => [
                'type' => 'keyword',
            ],
            'vision' => [
                'type' => 'text',
                "analyzer" => "ik_max_word",
                "search_analyzer" => "ik_max_word",
            ],
            'phone' => [
                'type' => 'long',
            ],
            'email' => [
                "analyzer" => "ik_max_word",
                "search_analyzer" => "ik_max_word",
                'type' => 'text',
                'fields' => [
                 'raw' => [
                    'type' => 'keyword',
                ]
            ]
            ],
        ]
    ];

    public function getHighlightVisionAttribute()
    {
        return optional($this->highlight)->vision[0];
    }

    public function getHighlightNameAttribute()
    {
        return optional($this->highlight)->name[0];
    }
}

```

```php
<?php

namespace App\Http\Controllers;

use App\Company;
use Illuminate\Http\Request;

class CompanyController extends Controller
{
    public function index()
    {
        $results = Company::search(request('q'))->get();

        return response()->json(['success'=>true,'data' => $results]);
    }
}

```



```javascript
<template>
    <div class="container">
          <form class="form-inline" v-on:submit.prevent="searchData">
            <div class="form-group mx-sm-3 mb-2">
                <input type="text"
                    class="form-control"
                    v-model="search"
                    id="searchBox"
                    @click="searchData"
                    @keyup.enter="searchData"
                >
            </div>

        </form>
        <div class="card-columns">
          <div class="card" v-for="(item,index) in companies" :key="index">
            <div class="card-header" v-html="item.highlight_name?item.highlight_name:item.name">
            </div>
             <div class="card-block">
                <blockquote class="card-blockquote">
                    <p v-html="item.highlight_vision?item.highlight_vision:item.name"></p>
                    <p>{{item.phone}}</p>
                    <footer>
                        {{item.city}}
                        <cite title="Source Title">{{item.email}}</cite>
                    </footer>
                </blockquote>
            </div>
          </div>
        </div>
    </div>
</template>

<script>
    export default {
        mounted() {
            console.log('Component mounted.')
        },

        data() {
            return {
                search: "",
                companies: [],
            }
        },
        // computed: {
        //     duc(){
        //         return item.highlighted?item.highlighted:item.name;
        //     }
        // },
        methods: {
            searchData() {
                axios.get('http://localhost:8089/search', {
                    params: {q: this.search}
                }).then(({data})=>{
                    this.companies = data.data
                    // this.search = ''
                })
            },
             duc(item, name){
                return item.highlighted?item.highlighted:item.name;
            }
        },
        watch: {
            search:_.debounce(function () {
                !this.search ?'': this.searchData()
            }, 800)
        }
    }
</script>

```

[高亮规则](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-highlighting.html#control-highlighted-frags)

[query规则](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-bool-query.html)

```php
<?php

namespace App\ES;

use ScoutElastic\SearchRule;

class CompanyRule extends SearchRule
{
 // This method returns an array, describes how to highlight the results.
    // If null is returned, no highlighting will be used.
    public function buildHighlightPayload()
    {
        return [
            'fields' => [
                'name' => [
                    'type' => 'plain',
                    "pre_tags"=>"<span style='color:blue'>",
                    "post_tags"=>"</span>"
                ],
                'vision' => [
                    'type' => 'plain',
                    "fragment_size"=> 150,
                    "number_of_fragments"=> 3
                ]
            ],
            "pre_tags"=>"<span style='color:red'>",
            "post_tags"=>"</span>"
        ];
    }

    // This method returns an array, that represents bool query.
    // public function buildQueryPayload()
    // {
    //     return [
    //         'must' => [
    //             'match' => [
    //                 'name' => $this->builder->query
    //             ]
    //         ]
    //     ];
    // }
}

```



## &#x1f919; 1025434218@qq.com 