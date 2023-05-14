ElasticSearch 版本关系
--------------

| Elasticsearch Version | Elasticsearch-PHP Branch | PHP Version |
| --------------------- | ------------------------ | ----------- |
| >= 6.0                | 6.0                      | >= 7.0.0    |
| >= 5.0, < 6.0         | 5.0                      | >= 5.6.6    |
| >= 2.0, < 5.0         | 1.0 or 2.0               | >= 5.4.0    |
| >= 1.0, < 2.0         | 1.0 or 2.0               | >= 5.3.9    |
| <= 0.90.x             | 0.4                      | ...         |


下载ElasticSearch 镜像
--------------

- 下载
```bash
docker pull docker.elastic.co/elasticsearch/elasticsearch:5.6.16
```
- 运行
```Bash
docker run -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:5.6.16
```

设置容器虚拟内存
--------------

- Linux
```Bash
sysctl -w vm.max_map_count=262144

/etc/sysctl.conf
vm.max_map_count=262144
```
- otherSystem [Document](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/docker.html#docker-cli-run-prod-mode)


docker-compose 运行
--------------
docker-compose运行和docker运行均为单例模式

- env说明
    * cluster.name 节点名称  
    * discovery.type 搜寻模式(单例 开发环境中使用)
    * xpack.security.enabled=false(关闭安全验证)

```yaml
elasticsearch:
  build: ./elasticsearch
  volumes:
    - elasticsearch:/usr/share/elasticsearch/data
  environment:
    - cluster.name=laradock-cluster
    - discovery.type=single-node
    - xpack.security.enabled=false
    - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
  ulimits:
    memlock:
      soft: -1
      hard: -1
  ports:
    - "9200:9200"
    - "9300:9300"
```

运行成功后 会出现9200 端口
--------------
默认账号密码
elastic
changeme

使用看命令
```bash
curl http://elastic:changeme@127.0.0.1:9200
```

详细数据会根据具体变化 如果出现证明elasticsearch 成功运行
```json
{
  "name" : "jbwSzfM",
  "cluster_name" : "laradock-cluster",
  "cluster_uuid" : "PIVWsauhQQaGn5e-W7c9lw",
  "version" : {
    "number" : "5.6.16",
    "build_hash" : "3a740d1",
    "build_date" : "2019-03-13T15:33:36.565Z",
    "build_snapshot" : false,
    "lucene_version" : "6.6.1"
  },
  "tagline" : "You Know, for Search"
}
```










