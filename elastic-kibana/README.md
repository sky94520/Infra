# 1. 简介
首先是这几个产品的官方定义：
1. Elasticsearch - Elasticsearch 是一个开源的分布式搜索和分析引擎，用于处理和存储大规模数据。它是基于 Apache Lucene 搜索引擎库构建的，并提供了分布式、实时的数据存储和搜索功能；
2. Kibana - Kibana 是一个开源的数据分析和可视化平台，与 Elasticsearch 紧密集成，用于实时搜索、数据分析、数据可视化和操作 Elasticsearch 中的数据；
3. ECK（Elastic Cloud on Kubernetes） 是一个Kubernetes Operator，它管理和自动化Elastic Stack 的生命周期。

在我的理解来说，Elasticsearch是一个全文搜索数据库，kibana则是一个与之配套的可视化UI。

# 2. 部署
## 2.1 部署ECK
ECK的部署比较简单，跟着[官方示例](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-eck.html)走就行。
## 2.2 部署Elasticsearch
ES是一个数据库，它需要一个配套的PV，这里使用名称为nfs的storage class提供PV（关于NFS的相关内容，可以参考[基于k8s搭建Web项目[3] - 搭建Redis Cluster](#/article/2)）。
:::warning
使用NFS，需要ES所在的机器也是支持NFS的，即需要安装```sudo apt install nfs-common```，否则在安装ES后，会报错
```elasticsearch bad option; for several filesystems (e.g. nfs, cifs) you might need a /sbin/mount.<type> helper program.```
:::
ES的配置文件分为了三部分，第一部分是去掉了SSL，这样是方便后续使用Ingress对接ES；第二部分是描述节点以及节点的功能；第三部分则是声明了一个PV。
```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: es
spec:
  version: 8.11.1
  http:
    tls:
      selfSignedCertificate:
        disabled: true
  nodeSets:
  - name: default
    count: 1
    config:
      node.store.allow_mmap: false
      node.roles: ["master", "data", "ingest", "ml", "transform"]
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data # 这里不要修改，进阶用法参考 ECK 官方文档
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 100Gi # 配置默认大小，allowVolumeExpansion为true后续可以扩展
        storageClassName: nfs # StorageClass 名称
```
在执行```kubectl apply -f es.yaml```后，可以查看部署的状态，等到下面的bash输出变为green后，表示创建成功。
```bash
kubectl get elasticsearch -n elastic-system
# NAME   HEALTH   NODES   VERSION   PHASE   AGE
# es     green    1       8.11.1    Ready   4h6m
```
执行完上面的部署后，集群中会出现ES的deployment和类型为ClusterIP的Service，为了能够让局域网的其他PC访问到它，可以再部署一个Ingress。在部署前，需要先创建一个证书，然后导入到集群中
```bash
# 创建一个三年有效期的私有证书
openssl req -x509 -nodes -days 1095 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=elasticsearch.sky.com/O=sky"
# 导入到集群，secret名字为elasticsearch-secret
kubectl create secret tls elasticsearch-secret --key tls.key --cert tls.crt -n elastic-system
```
接着就是Ingress的配置：
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: elasticsearch-ingress
  namespace: elastic-system
spec:
  tls:
    - hosts:
        - elasticsearch.sky.com
      secretName: elasticsearch-secret
  rules:
    - host: elasticsearch.sky.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: es-es-http
                port:
                  number: 9200
```
然后此时到局域网的不属于集群的PC机器上的hosts文件手动输入域名到IP的映射：
```
192.168.101.21 elasticsearch.sky.com
```
并在浏览器打开，发现它需要账号和密码，默认的账号为```elastic```，密码是由ES自动生成的，被保存在了secret: ```es-es-elastic-user```，通过下述的命令获取到密码
```bash
kubectl get secret es-es-elastic-user -n elastic-system -o go-template='{{.data.elastic | base64decode}}'
```
![image.png](/api/v1/file/download/blog/images/77/screenshot-2023-11-26T07:57:39.png)
## 2.3 部署Kibana
Kibana的部署和Elasticsearch类似，同样是编写YAML，使用Ingress暴露服务。
```kibana.yaml```
```yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana
  namespace: elastic-system
spec:
  version: 8.11.1
  http:
    tls:
      selfSignedCertificate:
        disabled: true
  count: 1
  elasticsearchRef:
    name: es
```
接着修改之前的ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: elasticsearch-ingress
  namespace: elastic-system
spec:
  tls:
    - hosts:
        - elasticsearch.sky.com
      secretName: elasticsearch-secret
  rules:
    - host: elasticsearch.sky.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: es-es-http
                port:
                  number: 9200
    - host: kibana.sky.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kibana-kb-http
                port:
                  number: 5601
```
kibana的账号密码与elasticsearch的相同，稍微等待一会，就可以在UI上查看与编辑咯。
以上的YAML保存于[github](https://github.com/sky94520/Infra/tree/master/elastic-kibana)。
## 2.4 添加插件
### 2.4.1 IK插件
在k8s部署ES数据库，官网推荐了两种方法并列举了优缺点：
1. 创建一个包含所需插件和配置文件的自定义容器镜像。
	优点：
	a. 部署是可重复使用的和可复制的。
	b. 在运行时不需要互联网访问。
	c. 节省带宽，并且启动更快。
	缺点：
	a. 需要容器注册表和构建基础设施来构建和托管自定义镜像。
	b. 版本升级需要构建新的容器镜像。
2. 使用initContainers安装和配置文件。
	优点：
	a. 更容易开始使用和升级版本。
	缺点：
	a. 需要Pod具有Internet访问权限;
	b. 由于网络问题或错误配置，添加新的Elasticsearch节点可能会随机失败;
	c. 每个Elasticsearch节点需要重复下载，浪费带宽并减慢启动速度;
	d. 部署清单更加复杂。


:::tip
ES版本需要和插件的版本一样，否则会启动失败
:::
我主要采用的是方案一，不过并没有使用官方的插件，根据[analysis-ik](https://github.com/medcl/elasticsearch-analysis-ik)说明，只需要把release包放置在plugins/analysis-ik下，重启ES即可。因此这里采用在启动ES之前，先下载ik包，然后启动ES：
```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: es
  namespace: elastic-system
spec:
  version: 8.11.1
  http:
    tls:
      selfSignedCertificate:
        disabled: true
  nodeSets:
  - name: default
    count: 1
    config:
      node.store.allow_mmap: false
      node.roles: ["master", "data", "ingest", "ml", "transform"]
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          command:
          - /usr/bin/env
          - bash
          - -c
          - |
            #!/usr/bin/env bash
            set -e
            cd plugins
						# -L 表示重定向 -O表示输出到文件
            curl -LO https://gitee.com/sky94520/elasticsearch-analysis-ik/releases/download/v8.11.1/elasticsearch-analysis-ik-8.11.1.zip
            unzip elasticsearch-analysis-ik-8.11.1.zip -d analysis-ik
            rm elasticsearch-analysis-ik-8.11.1.zip
            cd -
            /bin/tini -- /usr/local/bin/docker-entrypoint.sh
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data # 这里不要修改，进阶用法参考 ECK 官方文档
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 100Gi # 配置默认大小，allowVolumeExpansion为true后续可以扩展
        storageClassName: nfs # StorageClass 名称
```
这里使用的release包为我在gitee上的fork版本，这么做也是因为国内访问github下载非常慢，如果可以访问github，也可以改为官方release包（这也是官方建议的另外一种方式，即可能在initContainers是无法进行网络访问的，参考[Custom configuration files and plugins-Note when using Istio](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-bundles-plugins.html#k8s-bundles-plugins)）。
接着执行上面的yaml，查看log：
```
Defaulted container "elasticsearch" out of: elasticsearch, elastic-internal-init-filesystem (init), elastic-internal-suspend (init)
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   185    0   185    0     0     67      0 --:--:--  0:00:02 --:--:--    67
100   265    0   265    0     0     89      0 --:--:--  0:00:02 --:--:--    89
 84 4508k   84 3807k    0     0   367k      0  0:00:12  0:00:10  0:00:02  559k{"timestamp": "2023-12-16T09:55:53+00:00", "message": "readiness probe failed", "curl_rc": "7"}
100 4508k  100 4508k    0     0   382k      0  0:00:11  0:00:11 --:--:--  558k
Archive:  elasticsearch-analysis-ik-8.11.1.zip
  inflating: analysis-ik/elasticsearch-analysis-ik-8.11.1.jar  
  inflating: analysis-ik/httpclient-4.5.13.jar  
  inflating: analysis-ik/httpcore-4.4.13.jar  
  inflating: analysis-ik/commons-logging-1.2.jar  
  inflating: analysis-ik/commons-codec-1.11.jar  
   creating: analysis-ik/config/
  inflating: analysis-ik/config/stopword.dic  
  inflating: analysis-ik/config/extra_main.dic  
  inflating: analysis-ik/config/quantifier.dic  
  inflating: analysis-ik/config/extra_single_word.dic  
  inflating: analysis-ik/config/IKAnalyzer.cfg.xml  
  inflating: analysis-ik/config/surname.dic  
  inflating: analysis-ik/config/extra_single_word_low_freq.dic  
  inflating: analysis-ik/config/extra_single_word_full.dic  
  inflating: analysis-ik/config/preposition.dic  
  inflating: analysis-ik/config/extra_stopword.dic  
  inflating: analysis-ik/config/suffix.dic  
  inflating: analysis-ik/config/main.dic  
  inflating: analysis-ik/plugin-descriptor.properties  
  inflating: analysis-ik/plugin-security.policy  
/usr/share/elasticsearch
Dec 16, 2023 9:55:56 AM sun.util.locale.provider.LocaleProviderAdapter <clinit>
WARNING: COMPAT locale provider will be removed in a future release
{"@timestamp":"2023-12-16T09:55:56.552Z", "log.level": "INFO", "message":"Java vector incubator API enabled; uses preferredBitSize=256", "ecs.version": "1.2.0","service.name":"ES_ECS","event.dataset":"elasticsearch.server","process.thread.name":"main","log.logger":"org.apache.lucene.internal.vectorization.PanamaVectorizationProvider","elasticsearch.node.name":"es-es-default-0","elasticsearch.cluster.name":"es"}
```
:::warning
我一开始采用的是官方建议的方案一，但是一直在报错误，而且没有发现解决方案
:::
![image.png](/api/v1/file/download/blog/images/77/screenshot-2023-12-16T10:10:08.png)
在部署完成后，可以简单测试下，首先在kibana创建一个index(没有安装kibana可以使用Postman，只需要把第一行改为method，以及完整的URL即可)：
```json
PUT /accounts
{
  "mappings": {
    "properties": {
      "user": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_max_word"
      }
    }
  }
}
```
然后输出：
```json
{
  "acknowledged": true,
  "shards_acknowledged": true,
  "index": "accounts"
}
```
接着添加一笔记录：
```json
POST /accounts/_doc
{
  "user": "我是小小张三"
}
```
输出
```json
{
  "_index": "accounts",
  "_id": "F_sXcowBM5VUCxgum9wy",
  "_version": 1,
  "result": "created",
  "_shards": {
    "total": 2,
    "successful": 1,
    "failed": 0
  },
  "_seq_no": 0,
  "_primary_term": 1
}
```
最后可以查询一下：
```json
GET /accounts/_search
{
  "query": {
    "match": {
      "user": "张三"
    }
  }
}
```
返回结果
```json
{
  "took": 10,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 1,
      "relation": "eq"
    },
    "max_score": 0.5753642,
    "hits": [
      {
        "_index": "accounts",
        "_id": "F_sXcowBM5VUCxgum9wy",
        "_score": 0.5753642,
        "_source": {
          "user": "我是小小张三"
        }
      }
    ]
  }
}
```
# 3. 参考链接
1. [k8s部署生产级elasticsearch+kibana 步骤、踩坑及解决方案](https://blog.csdn.net/finishy/article/details/122479641)
2. [Why do I get "wrong fs type, bad option, bad superblock" error?](https://askubuntu.com/questions/525243/why-do-i-get-wrong-fs-type-bad-option-bad-superblock-error)
3. [es tls certificates](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-tls-certificates.html)
4. [Custom configuration files and plugins](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-bundles-plugins.html#k8s-bundles-plugins)
5. [使用curl命令下载文件](https://zhuanlan.zhihu.com/p/350613195 )
6. [【elasticsearch】docker下elasticsearch 安装ik分词器](https://cloud.tencent.com/developer/article/2139072)