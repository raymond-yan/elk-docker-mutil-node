## ELK Stack 通过Docker的多节点部署 (Elasticsearch、Kibana、Logstash、Fiebeat) - 7.14.0 版本

### 简介

[Elasticsearch](https://www.elastic.co/cn/elasticsearch/)是目前世界上最流行、最强大的开源搜索引擎之一。许多主流的软件厂商们都有使用 Elasticsearch，以及ELK Stack。

Elastic Search 优点

 * Scale: 具备可横行扩展能力，不需要像数据库那样分库、分表扩张
 * Speed: 通过索引 提高查询效率
 * Relevance: 查询相关性，不需要数据库查询通过sql获取数据。

非常适合支持在线、实时搜索、实时分析的需求。

ELK Stack:

* Kibana: 数据可视化工具，可以将保存于在Elasticsearch的数据进行可视化、高级分析
* Elasticsearch: 核心部分，支撑数据存储、搜索、分析
* Beats: 轻量级数据采集器agent,golang 语言，用于收集服务器的日志文件信息、运行监控指标信息、安全行为信息、网络流量信息。
* Logstash: 开源流式 ETL 引擎，数据流引擎，对数据进行采集、处理后输出。

基于 Docker 多节点集群化的部署，可以使得用户非常容易的扩展ELK的部署，基于我们的数据、查询量的增长。ELK Stack 可以在相同的服务器内纵向扩展，或者在不同服务器间横向扩展，通常时基于 Docker Swarm 或者 Kubernetes。

### 安装 Elasticsearch 及 Kibana

在本项目中，会通过`docker-compose`运行有三个es节点、一个kibana节点的集群。

下载 Elasticsearch 的镜像:
``` bash
docker pull docker.elastic.co/elasticsearch/elasticsearch:7.14.0
```

下载 Kibana 镜像
```
docker pull docker.elastic.co/kibana/kibana:7.14.0
```

在本项目根目录下运行以下指令，启动集群。
```
docker-compose up
```
当我们看到以下log信息，即代表Elasticsearch和Kibana运行成功。
```
kibana_1  | {"type":"log","@timestamp":"2021-08-23T02:23:59+00:00","tags":["info","status"],"pid":1215,"message":"Kibana is now available (was unavailable)"}
```

此时我们可以运行以下指令获取Elasticsearch的节点运行情况：
```
curl -X GET "localhost:9200/_cat/nodes?v&pretty"
# 输出
ip            heap.percent ram.percent cpu load_1m load_5m load_15m node.role   master name
192.168.112.2           53          89   7    8.40    5.86     3.28 cdfhilmrstw -      es02
192.168.112.3           60          89   7    8.40    5.86     3.28 cdfhilmrstw -      es03
192.168.112.4           37          89   7    8.40    5.86     3.28 cdfhilmrstw *      es01
```

接着打开本地kibana: [http://localhost:5601/](http://localhost:5601/)，可以看到kibana及es的运行情况。打开Kibana左侧菜单栏，选择 Management -> [堆栈检测](http://localhost:5601/app/monitoring)，即可看到目前的 ES 节点和 Kibana 节点的情况。

![Kibana Monitoring](https://blog-images-1304189995.cos.ap-guangzhou.myqcloud.com/elk-docker/kibana-monitoring.png)


### 安装及运行 Logstash

[Logstash](https://www.elastic.co/guide/en/logstash/current/index.html)是一款开源的数据采集引擎，可以动态的从不同的来源中结构化、统一化数据，并且输出到我们所需要的终端中。在本项目中，则是输出到 Elasticsearch。

下载 Logstash 镜像:
``` bash
docker pull docker.elastic.co/logstash/logstash:7.14.0
```

切换到 logstash 目录并运行：
```
cd ./logstash
docker-compose up
```

### 安装及运行 Filebeat

Filebeat 是一个轻量的数据采集器，用于聚合、传输日志信息。当安装于server中时，Filebeat会监控日志文件或规定的目录，采集日志信息并转发到 Elasticsearch 或者 Logstash。

在示例中，我们将通过 Filebeat 读取`./filebeat/apache_logs.txt`，将日志传输到 Logstash中，经过 Logstash 的结构化、标准化处理后，输入到 Elasticsearch 中，用于日志的展示、检索。

首先下载 Filebeat 的镜像:
``` bash
docker pull docker.elastic.co/beats/filebeat:7.14.0
```

切换到`filebeat`目录并运行:

``` bash
cd ./filebeat

docker-compose up
```

### 结果

当运行完以上指令后，此时打开kibana，在开发工具中输入以下指令获取当前的index:

```
GET /_cat/indices
```

或者通过`curl`直接Elasticsearch中获取:
```
curl -XGET "http://localhost:9200/_cat/indices"
```

结果中应该有一项名为`logstash-apache-log`的索引，这就是我们通过 Filebeat 读取的apache log，转发到 Logstash 中，通过结构化、标准化处理后，从log信息中提取出内容，最终输入到 Elasticsearch中。可以在开发工具运行以下指令查询数量，结果应该为10000条，与log日志的行数相同。

```
GET logstash-apache-log/_count

##输出
{
  "count" : 10000,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  }
}
```

### 通过仪表盘展示信息

首先我们打开Kibana的[索引管理页面](http://localhost:5601/app/management/kibana/indexPatterns)，点击**创建索引模式**，输入`logstash-apache-log`，点击下一步，并且在下一步中选择时间字段为`@timestamp`。

创建完索引后，我们可以尝试在[Discover](http://localhost:5601/app/discover#/?_g=(filters:!(),query:(language:lucene,query:'*'),refreshInterval:(pause:!t,value:0),time:(from:'2015-05-10T19:10:56.990Z',to:'2015-05-25T19:10:56.990Z'))&_a=(columns:!(),filters:!(),index:'693a3a50-0183-11ec-b1da-dfd897cc0ac3',interval:auto,query:(language:kuery,query:''),sort:!(!('@timestamp',desc))))页面中，查询数据。可以看到我们的log信息都已经结构化的展示出来。
![Kibana Dashbord](https://blog-images-1304189995.cos.ap-guangzhou.myqcloud.com/elk-docker/kibana-discover.png)


最后，我们需要可以导入已生成的`Dashbord`对象。在[Kibana 已保存对象](http://localhost:5601/app/management/kibana/objects)中，选择导入，选择`./kibana/kibana-dashbord.ndjson`。

导入完成后，我们切换到[Kibana 仪表盘](http://localhost:5601/app/dashboards)中，算则刚刚创建的[仪表盘](http://localhost:5601/app/dashboards#/view/Sample-Dashboard-for-Apache-Logs)，即可可视化的查询我们刚刚导入的apache日志信息。可以看到我们的总访问人数、访问者在地图上的分布情况、国家、系统类型分布、uv/pv 等信息。

![Kibana Dashbord](https://blog-images-1304189995.cos.ap-guangzhou.myqcloud.com/elk-docker/kibana-dashbord.png)


### 参考文档

 - [Deploy a Multi-node Elasticsearch Cluster with Docker Compose](https://edward-cernera.medium.com/deploy-a-multi-node-elasticsearch-instance-with-docker-compose-ef63625f246e)
 - [开始使用 Elasticsearch](https://www.elastic.co/cn/webinars/getting-started-elasticsearch)
 - [开始使用Logstash](https://www.elastic.co/cn/webinars/getting-started-logstash)
 - [Running the Elastic Stack on Docker](https://www.elastic.co/guide/en/elastic-stack-get-started/current/get-started-docker.html#get-started-docker)
 - [Install Kibana with Docker](https://www.elastic.co/guide/en/kibana/current/docker.html)
