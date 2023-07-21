# BuildFarm配置

先简单介绍下当前可用的BuildFarm配置是怎么做的，采用了10台BuildFarm Server维护集群的可用，24台固定的CPU Buildfarm Worker节点参与编译工作，6台GPU Buildfarm Worker参与GPU Test。除上述机器之外，还有自动扩缩容的机器参与到CPU Buildfarm Worker的工作当中。针对Buildfarm 2.3.1的版本，采用配置如下即可。

### Server

backplane用来指定通信的媒介，拆分为两个不同的queues，用来方便任务的分发，比方说CPU任务包括编译和TEST，GPU任务则只包含TEST任务，可以看到GPU队列必须要满足相关的properties才会被正确的depatched。

CPU队列允许不匹配的规则，其中min-cores和max-cores都为\*，就能保证没命中GPU类型任务都会命中到CPU队列当中

Server的类型为shard，这个是目前唯一支持的类型了

```
backplane:
  redisUri: "redis://xxxxxxxxxxxxxxxxxxxxxxxx:6379"
  queues:
    - name: "gpu"
      allowUnmatched: false
      properties:
        - name: "gpu"
          value: "1"
    - name: "cpu"
      allowUnmatched: true
      properties:
        - name: "min-cores"
          value: "*"
        - name: "max-cores"
          value: "*"
digestFunction: SHA256
maxEntrySizeBytes: 123456
server:
  name: "shard"
  recordBesEvents: true
```

### CPU Worker

CPU Worker的配置和Server端非常类似，其中executeStageWidth标记可同时并发的任务数量，inputFetchStageWidth则标记可以并发获取Input的数量。我们使用FILESYSTEM来标记该CPU节点既执行缓存操作，又有执行能力。realInputDirectories用来标记external，如果不写这个配置可能造成外部文件hard link过多的问题，

```
backplane:
  redisUri: "redis://xxxxxxxxxxxxxxxxxxxxxxxx:6379"
  queues:
    - name: "cpu"
      allowUnmatched: true
      properties:
        - name: "min-cores"
          value: "*"
        - name: "max-cores"
          value: "*"
digestFunction: SHA256
maxEntrySizeBytes: 123456     
worker:
  port: 8982
  publicName: "localhost:8982"
  executeStageWidth: 80
  inputFetchStageWidth: 8
  realInputDirectories:
    - "external"
  storages:
    - type: FILESYSTEM
      maxSizeBytes: 123124124124123123
```

### GPU Worker

可以看到GPU Worker的配置和CPU Worker类似，但是属性那一部分并不一致，借此来保证任务的正常调度。

```
backplane:
  redisUri: "redis://xxxxxxxxxxxxxxxxxxxxxxxx:6379"
  queues:
    - name: "gpu"
      allowUnmatched: false
      properties:
        - name: "gpu"
          value: "1"
digestFunction: SHA256
maxEntrySizeBytes: 12345678      
worker:
  port: 8982
  publicName: "localhost:8982"
  executeStageWidth: 8
  inputFetchStageWidth: 8
  realInputDirectories:
    - "external"
  storages:
    - type: FILESYSTEM
      maxSizeBytes: 123123123123123
  dequeueMatchSettings:
    acceptEverything: true
    allowUnmatched: false
    properties:
      - name: "gpu"
        value: "1"
```

### 如何使用

将export的端口暴露为相应的grpc服务，然后直接在用户的BAZEL配置文件里面写下如下地址即可，其中GPU TEST的Build文件里面需要添加对应poperties类型，来保证这个test被正确地调度到对应的Worker

````
```python
    exec_properties = {"gpu": "1"},
```
````

````
```shellscript
build --remote_executor=服务地址
build --remote_cache=
build --remote_timeout=5m"
build --jobs=30 #控制并发数量
```
````

