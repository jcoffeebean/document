modules
==

模块

cluster
==

[原文](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/modules-cluster.html)  

基本概念
===

cluster: 集群，一个集群通常由很多节点(node)组成  
node: 节点，比如集群中的每台机器可以看做一个node   
shard: 分片，ES是分布式搜索引擎，会把数据拆分成很多个shard，一个索引默认有5个shard  
replica: 副本，ES是high availability的， 为了数据安全会把同一份数据存放在多个节点，默认情况下一个索引的数据会存两份副本。一份是primary，一份是replica。 
primary: 主节点   
rebalancing: 指数据在集群的节点中重新分配，比如当集群中增加或者移除节点时就会发生rebalancing   

Shards allocation
===

Shards allocation是指在各个节点分配shard的过程。 在初始化恢复, 分配replica, rebalancing, 新增或移除节点时会发生。

一些基本配置如下

`cluster.routing.allocation.allow_rebalance`

根据集群中节点的状态来控制什么时候可以rebalancing， 可以设置三种方式。

* `indices_primaries_active`: 仅当所有的primary shards是active的时候才允许rebalancing。  
* `indices_all_active`: 仅当所有的shards是active的时候才允许rebalancing。  
* `always`: 一组shard、replication是active时就可以rebalancing。   

默认值为`indices_all_active`，可以减少cluster初始化恢复时各节点之间的交互。

`cluster.routing.allocation.cluster_concurrent_rebalance`

设置在cluster中最多可以允许几个rebalancing同时进行，默认为2, 如果设置为-1则意味着不做限制。  

`cluster.routing.allocation.node_initial_primaries_recoveries`

限制每个节点可以同时初始化恢复primary shard数量。

这个设置是为了防止同时进行的recovery进程太多影响节点负载，因为大多数情况下用的是local gateway，速度相当快，所以可以同时执行多个recovery进程而不会造成太多的负荷，默认是4。 

`cluster.routing.allocation.node_concurrent_recoveries`

限制每个节点并行recovery的数量， 默认是2。

`cluster.routing.allocation.disable_new_allocation`

设置是否禁止分配新的新的primary shard，注意, 设置为true会阻止新建的索引的shard分配。 
因为如果primary不存在，replica会自动提升为primary， 所以这个配置通过更新cluster配置的API动态更新才有意义。

`cluster.routing.allocation.disable_allocation`

是否禁止分配primary和replica，这个配置通过更新cluster配置的API动态更新才有意义。

`cluster.routing.allocation.disable_replica_allocation`

是否禁止分配replica，和上面的设置类似， 这个配置通过更新cluster配置的API动态更新才有意义。

`indices.recovery.concurrent_streams`

设置在从对应的shard恢复一个shard时，可以同时打开的数据流的数量(节点级别)，默认是3。


Cluster allocation awareness
===

分片规则

集群分片规则(Cluster allocation awareness)允许我们用一些参数来配置整个集群中shard和replica的分片规则。 下面通过一个例子来解释一下。 

假设我们有几个机架，我们给一节点配置一个名为rack_id的属性(其他名字也可以)， 配置例子如下:

node.rack_id: rack_one

上面例子为这个节点配置了一个名为rack_id的属性， 值为rack_one。   接下来将rack_id配置为分片规则所用的属性（在所有节点都要设置）。

cluster.routing.allocation.awareness.attributes: rack_id

上面的配置意味着分片规则会基于属性rack_id来做shard和replica分配。 例如，我们有两个node.rack_id属性设置为rack_one的节点, 部署了一个有5个shards和1个replica的索引， 索引数据会分布到这两个节点上(每个节点有5个shard， 1个replica, 总共10个shards)。 

如果我们再加入两个节点，这两个节点的node.rack_id属性设置为rack_two, shard会在这些节点上重新分配， 但是一个shard和他的replica不会分配到有相同rack_id属性的节点上。

可以为分片规则设置多个属性， 比如:

cluster.routing.allocation.awareness.attributes: rack_id,zone

注意：启用了分片规则属性后，如果一个节点没有配置这些属性， shard就不会分配到这个节点上。

forced awareness
===

强制行分片规则

有时候我们提前知道用来做分片规则的属性会有更多的值， 我们不希望一些replica被分配到一组特定节点上， 对于这种情况， 我们可以针对这些属性值用强制分片规则。

例如，我们用属性zone来做分片规则属性，并且我们知道会有两个zone：zone1和zone2。 下面是强制分片规则设置的例子：

```
cluster.routing.allocation.awareness.force.zone.values: zone1,zone2
cluster.routing.allocation.awareness.attributes: zone
```

现在我们先启用两个node.zone属性设置成zone1的节点，然后创建一个有5个shard和1个replica的索引。
索引建完后只有5个shard（没有replica），要等到我们再启用更多属性node.zone为zone2的节点时，replica才会被分配。

automatic preference when searching / geting
===

在执行search或者执行get指令时， 接受请求的节点会优先选择与其有相同属性值的节点分片上执行请求。

realtime settings update
===

这些设置可以通过更新cluster配置的api在一个运行着的cluster上实时更新。

shard allocation filtering
===

可以用include/exclude过滤器控制索引部署到哪些节点上，过滤器可以设置在索引级别，也可以设置在集群级别， 我们先看一下设置在索引级别的例子。

假设我们有四个节点， 每个节点配置了一个名为tag的分片规则属性(可以是任何名字)，节点1的tag属性置为value1, 节点2的tag属性设置为value2，以此类推。

我们可以把配置项`index.routing.allocation.include.tag`设置为value1,value2来使创建的索引只部署到哪些tag属性为value1和value2的节点上，例如

```
curl -XPUT localhost:9200/test/_settings -d '{
      "index.routing.allocation.include.tag" : "value1,value2"
}'
```

另一方面， 我们将配置项`index.routing.allocation.exclude.tag`设置为value3， 这样创建的索引会被部署到tag属性为value3之外的那些节点上，例如

```
curl -XPUT localhost:9200/test/_settings -d '{
      "index.routing.allocation.exclude.tag" : "value3"
}'
```

从0.90版开始， 可以用`index.routing.allocation.require.*`来设置一系列规则， 只有符合全部规则的节点才会分配shard， 相对而言`include`则是只要符合任意一条就可以。


`include`, `exclude`和`require`的值都支持通配符, 例如`value1*`。 一个特殊的节点名是`_ip`，可以用来匹配节点的ip地址. 另外一个特殊属性`_host`可以用来匹配节点的主机名和ip地址。

上面说过，一个节点可以配置几个属性， 例如 

```
node.group1: group1_value1
node.group2: group2_value4
```

对应的， `include`, `exclude` 和 `require` 也可以用几个属性, 例如

```
curl -XPUT localhost:9200/test/_settings -d '{
    "index.routing.allocation.include.group1" : "xxx"
    "index.routing.allocation.include.group2" : "yyy",
    "index.routing.allocation.exclude.group3" : "zzz",
    "index.routing.allocation.require.group4" : "aaa"
}'
```

这些设置可以用更新配置的api实时更新, 实时移动索引(索引的分片)。

Cluster级别的过滤器可以用更新cluster设置的api来实时定义和更新，这些设置在解除一个节点时很有用(即使replica数量设置为0)。 下面是根据ip地址解除一个节点的例子:

```
curl -XPUT localhost:9200/_cluster/settings -d '{
    "transient" : {
        "cluster.routing.allocation.exclude._ip" : "10.0.0.1"
    }
}'
```


discovery
==

[原文](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/modules-discovery.html)    

discovery模块负责发现集群(cluster)中的节点，以及选举出主节点。  

注意，ES是一个基于端到端(p2p)的系统，节点之间直接通信，所有主要的API(index, search, delete)不需要和主节点(master node)通信。 主节点的职责是维护整个集群的状态，并且在节点加入或者离开集群时重新分片。 每次集群的状体改变会通知到集群中的其他节点(方式取决于discovery模块的具体实现)。  

settings  
===

配置项`cluster.name`用来给集群设置一个名字，以此把一个集群和其他的集群区分开。 默认的集群名是elasticsearch， 不过推荐改为能反映集群用途的有实际意义的名字。  

ec2 discovery  
===

EC2 discovery机制使用EC2的API来执行自动发现，用不到，不看了。  

zen discovery  
===

Zen发现机制  

zen发现机制是ES默认的内置发现模块。 它提供了多播和单播两种发现方式，并且能够很容易的扩展以支持云环境。  

zen发现机制是和其他模块集成在一起的，比如所有节点之间通讯是用trasport模块来完成。  

Zen发现机制分为几个子模块，接下来分别做详细解释。  

ping  
===

ping是指一个节点用发现机制发现其他节点的过程， 支持多播(multicast)和单播(unicast)两种方式，也可以组合使用。  

multicast
===

multicast是指发送一个或多个多播请求，存在的节点会接受并且相应请求。 它提供了一组以`discovery.zen.ping.multicast`为前缀的配置项。

| Setting       | Description   									|
| --------------|:--------------------------------------------------|
| group      	| group地址，默认值为224.2.2.4      					|
| port        	| 端口，默认为54328   			 	    			|
| ttl          	| 多播消息的ttl，默认是3 			        			| 
| address       | 绑定地址，默认为null，即绑定所有可用的network接口。 	| 


unicast
===

在多播禁用的情况下可以使用unicast发现方式。 它需要一个主机列表， 它提供了下面以`discovery.zen.ping.unicast`为前缀的配置项。

| Setting       | Description   															|
| --------------|:--------------------------------------------------------------------------|
| hosts      	| 一个数组或者以逗号分隔的字符串， 每个值的格式为host:port或者host[port1-port2] 	|

 
unicast发现方式需要借助transport模块来实现。   

master election  
===

主节点选举  

作为ping初始化过程的一部分， 需要选举出一个集群的master节点或者加入到一个已经选出的master节点， 这个过程是自动完成。 可以通过配置项`discovery.zen.ping_timeout`来设置ping的超时时间(默认是3s)以应对网络速度慢或者网络拥堵的情况。 设置一个比较大的值可以减少失败的几率。   

节点可以设置属性`node.master`为false来避免被选举为master节点。 注意， 如果一个节点被设置为客户端节点(`node.client`属性设置为true)， 这个节点不会被选举为master节点(`node.master`自动设置为false)。  

属性`discovery.zen.minimum_master_nodes`设置一个集群中最少的合格master节点数量， 对于2个节点以上的集群，建议设置为大于1的值。  

举个例子， 假设集群有5个节点， `minimum_master_nodes`设置为3, 如果2个节点掉线了，这两个节点不会自己组建一个集群， 而是尝试加入另一个集群。    

这个设置可以避免网络故障时有些节点试图自行组织集群，从而导致整个集群不稳定。  

fault detection
===

错误检测

有两种错误检测方式，一种是master节点ping集群中所有其他的节点来验证他们是否存活，另一种是每个节点ping master节点来验证它是否存活，或者是否需要初始化一个选举。

下面的配置项用于设置错误检测，前缀是`discovery.zen.fd`:

| Setting       	| Description   					|
| ------------------|:----------------------------------|
| ping_interval     | ping的频率， 默认1s					|
| ping_timeout      | ping的超时时间， 默认30s			|
| ping_retries      | 如果ping失败或者超时，重试的次数 	|


external multicast
===

外部多播

multicast 发现机制还支持外部多播请求，外部客户端可以发送多播请求， 格式为：

```
{
    "request" : {
        "cluster_name": "test_cluster"
    }
}
```

响应格式类似节点信息的响应(只有节点信息，包括transport/http地址以及节点的属性):

```
{
    "response" : {
        "cluster_name" : "test_cluster",
        "transport_address" : "...",
        "http_address" : "...",
        "attributes" : {
            "..."
        }
    }
}
```

注意，可以禁用内部multicast发现机制，只启用外部多播发现机制。 方式为将`discovery.zen.ping.multicast.enabled`设为true（默认），但是将`discovery.zen.ping.multicast.ping.enabled`设为false。


gateway
==

[原文](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/modules-gateway.html)  

gateway模块存储集群元数据(meta data)的状态，集群元数据主要包括索引的配置和声明的mapping定义。  

每次集群元数据发生变化时（比如添加或删除索引），会通过gateway来持久化这些变化。 集群启动时会从gateway读取并且应用这些数据。    

设置在节点级别的gateway自动控制索引所用的gateway。 比如节点用`fs` gateway，该节点创建的索引也自动用`fs` gateway。 在这种情况下， 如节点不应该持久化状态数据， 应该明确设置为`none`(唯一可以设置的值)。

ES默认使用的gateway是local gateway。  

recovery after nodes / time
===

大多数场景下，集群的元数据只能在特定的节点已经启动后才能被恢复， 或者等待到超时。 这在集群重启时非常有用，此时每个节点的本地索引存储仍然可用，不需要从gateway恢复（能够减少从gateway恢复的时间）。

`gateway.recover_after_nodes`(数字类型)设置多少个合格的data节点以及master节点启动后触发recovery。 `gateway.recover_after_data_nodes` 和`gateway.recover_after_master_nodes`含义类似，只不过分别设置data节点和master节点的数值。 `gateway.recover_after_time`(事件类型)设置在所有的`gateway.recover_after...nodes`条件满足后，等待多长时间再开始recovery。

`gateway.expected_nodes`设置预期多少个合格的data和master节点启动后就开始recovery，一旦满足条件马上启动recovery，`recover_after_time`设置会被忽略，对应的也支持`gateway.expected_data_nodes`和`gateway.expected_master_nodes`这两个配置项。 一个配置的例子如下:

```
gateway:
    recover_after_nodes: 1
    recover_after_time: 5m
    expected_nodes: 2、
```

这个例子配置了在一个预期两个节点的集群中，在一个节点启动后的5分钟后执行recovery，一旦集群中有已经有两个节点启动了，立即开始recovery（不等待，忽略recover_after_time）。    

注意，一旦元数据从gateway恢复了，那么这个配置就不再有效，直到下次集群完整重启。  

在集群元数据没有恢复时，为了避免和真实的集群元数据冲突，所有操作都会被阻止。  


local gateway
===

本地网关

local gateway从每个节点的本地存储中恢复整个集群状态和索引数据， 并且不需要节点级别的共享存储。  

注意，和共享类的gateway不同， local gateway的持久化不是异步的，一单一个操作被执行， 数据就会被存储以备集群恢复时使用。 

非常重要的一点是在配置`gateway.recover_after_nodes`时要包括大多数在整个集群重启后期望启动的节点， 这可以确保集群恢复到最新的状态。 例如:

```
gateway:
    recover_after_nodes: 1
    recover_after_time: 5m
    expected_nodes: 2
```

注意，为了能够备份/快照完整地集群状态， 建议禁用flush的情况下所有节点的本地存储都要有副本(理论上不需要所有的，只需要确保每个shard的副本被备份，这依赖replication的设置)。 共享存储比如S3可以在一个地方保存不同节点的拷贝，尽管代价是带来了更多的IO。   

shared fs gateway
===

shared FS gateway已经废弃，以后会被移除， 不看了。

hadoop gateway
===

hadoop gateway以后会被移除， 不看了。

s3 gateway
===

s3 gateway以后会被移除， 不看了。


http
==

[原文](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/modules-http.html)  

http模块允许通过http访问ES的接口。

http是完全异步的，意味着等待响应时不会阻塞线程。

如果有可能， 考虑使用HTTP keep alive来获得更好的性能，并且http客户端不要启用HTTP chunking。

settings
===

下面是http模块的一些设置。

| Setting       				| Description   					|
| --------------------------	|:----------------------------------|
| http.port     				| 绑定的端口范围， 默认9200-9300		|
| http.max_content_length   	| http请求大小的上限， 默认100mb		|
| http.max_initial_line_length  | http url的最大长度， 默认4kb 		|
| http.compression      		| 是否支持http压缩， 默认是false 		|
| http.compression_level      	| http压缩的级别， 默认是6 			|

http模块共享通用的network设置。

disable http
===

设置`http.enabled`为false可以禁用http模块，比如创建非数据节点来接收http请求，这些节点利用内部的transport来和数据节点通信。


indices
==

[原文](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/modules-indices.html)  

indices模块可以对所有索引进行全局管理。

indexing buffer
===

索引缓冲的设置可以控制多少内存分配给索引进程。 这是一个全局配置，会应用于一个节点上所有不同的分片上。

`indices.memory.index_buffer_size`接受一个百分比或者一个表示字节大小的值。 默认是10%，意味着分配给节点的总内存的10%用来做索引缓冲的大小。 这个数值被分到不同的分片(shards)上。 如果设置的是百分比，还可以设置`min_index_buffer_size` (默认48mb)和`max_index_buffer_size`（默认没有上限）。

`indices.memory.min_shard_index_buffer_size`设置分配给每个分片的索引缓冲内存的下限，默认4mb。

ttl interval
===

你可以动态设置`indices.ttl.interval`来控制自动删除过期documents的检查间隔，默认是60s。

删除是批量进行的，你可以设置`indices.ttl.bulk_size`来适应你的需要，默认值是10000。

其余的参考_ttl的文档。

recovery
===

以下设置用来管理recovery的策略:

| Setting       					| Description   					|
| ----------------------------------|:----------------------------------|
| indices.recovery.concurrent_streams| 默认是3							|
| indices.recovery.file_chunk_size  | 默认512kb							|
| indices.recovery.translog_ops  	| 默认1000							|
| indices.recovery.translog_size    | 默认512kb 							|
| indices.recovery.compress      	| 默认true							|
| indices.recovery.max_bytes_per_sec| 默认20mb							|
| indices.recovery.max_size_per_sec | 0.90.1去掉，用`indices.recovery.max_bytes_per_sec`代替|

下面的设置对存储进行限流:

| Setting       							| Description   						|
| ------------------------------------------|:--------------------------------------|
| indices.store.throttle.type       		| 可以是`merge` (默认), `not`或者`all`	|
| indices.store.throttle.max_bytes_per_sec  | 默认20mb								|



jmx
==

removed as of v0.90.  

memcached
==

[原文](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/modules-memcached.html)  

memcached模块可以通过memecached协议来访问ES的接口。

memcached模块通过一个名为transport-memcached插件提供，插件的安装说明见[transport-memcached](https://github.com/elasticsearch/elasticsearch-transport-memcached)，也可以下载memcached插件并放在plugins目录下。

memcached协议支持二进制和文本两种协议, 会自动检测应该用哪一种协议。

mapping rest to memcached protocol

Memcached命令会被映射到REST接口，并且会被同样的REST层处理，下面是支持的memcached命令列表:

get

memcached的GET命令映射到REST的GET方法。 用URI (带参数)来做key。 memcached的GET命令不允许在请求中带body(SET不允许返回结果)， 为此大多数REST接口(比如search)允许接受一个"source"作为URI的参数。

set

memcached的SET命令映射为REST的POST。 用URI (带参数)来做key, body映射为REST的body。

delete

memcached的DELETE命令映射为REST的DELETE。 用URI (带参数)来做key。

quit

memcached的QUIT命令用来断开客户端链接。

settings
===

以下设置可以用来配置memcached:

| Setting       			| Description   						|
| --------------------------|:--------------------------------------|
| memcached.port            | 绑定端口范围， 默认11211-11311			|

同样共享通用的network设置。

disable memcached
===

设置`memcached.enabled`为false可以禁用memcached模块， 默认检测到该插件即启用memcached模块。  

network settings
==

[原文](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/modules-network.html)  

一个节点的多个模块都用到了网络基本配置，例如transport模块和HTTP模块。 节点级别的网络配置可以用来设置所有基于网络的模块的通用配置（除了被每个模块明确覆盖的那些配置项）。  

`network.bind_host`用来设置绑定的ip地址， 默认绑定`anyLocalAddress` (0.0.0.0或者::0)。

`network.publish_host`配置其他节点和本节点通信的地址。 这个当然不能是`anyLocalAddress`, 默认是第一个非回环地址或者本机地址。

`network.host`设置是一个简化设置， 它自动设置`network.bind_host`和`network.publish_host`为同一个值。

两个设置都可以配置为主机ip地址或者主机名， 还可以设置为下表中列出来的值。

| Logical Host Setting Value| Description   						|
| --------------------------|:--------------------------------------|
| _local_		            | 本机ip地址								|
| _non_loopback_		    | 第一个非loopback地址					|
| _non_loopback:ipv4_	    | 第一个非loopback的ipv4地址				|
| _non_loopback:ipv6_	    | 第一个非loopback的ipv6地址				|
| _[networkInterface]_	    | 指定网卡的IP地址. 例如 _en0_			|
| _[networkInterface]:ipv4_ | 指定网卡的IPv4地址. 例如 _en0:ipv4_		|
| _[networkInterface]:ipv6_ | 指定网卡的IPv6地址. 例如 _en0:ipv6_		|
| _non_loopback:ipv6_	    | 第一个非loopback的ipv6地址				|

cloud-aws
===

如果安装了`cloud-aws`插件， 下表列出来值也是有效的设置:

| EC2 Host Value            | Description   											|
| --------------------------|:----------------------------------------------------------|
| _ec2:privateIpv4_         | The private IP address (ipv4) of the machine				|
| _ec2:privateDns_		    | The private host of the machines 							|
| _ec2:publicIpv4_		    | The public IP address (ipv4) of the machine 				|
| _ec2:publicDns_ 		    | The public host of the machines 							|
| _ec2_ 	  				| Less verbose option for the private ip address			|
| _ec2:privateIp_ 			| Less verbose option for the private ip address 			|
| _ec2:publicIp_ 			| Less verbose option for the public ip address 			|

tcp settings
===

任何使用TCP的组件 (比如HTTP, Transport和Memcached)共享下面的设置:

| Setting  		            | Description   								|
| --------------------------|:----------------------------------------------|
| network.tcp.no_delay      | 启用或禁用tcp no delay。 默认是true.				|
| network.tcp.keep_alive    | 启用或禁用tcp keep alive。 默认不设置			|
| network.tcp.reuse_address	| 地址是否应该被重用，在非windows的机器上默认是true	|
| network.tcp.send_buffer_size 		| tcp发送缓冲区的大小。 默认不设置			|
| network.tcp.receive_buffer_size 	| tcp接收缓冲区的大小。 默认不设置	 		|



node
==

[原文](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/modules-node.html)  

ES可以设置一个节点是否在本地存储数据，存储数据意味着不同索引的分片可以分配到这个节点上。 默认每个节点都可以作为数据节点(data node)，可以设置`node.data`为false来关闭。  

这是一个很强大的配置， 可以很简单的来创建一个智能负载均衡。  

我们可以启动一个数据节点的集群而不启用http模块， 这可以通过设置`http.enabled`为true做到， 这些节点通过transport模块相互通信， 在集群的前端可以启动一个和或者多个启用了http模块的非数据节点， 所有的http通讯由这些非数据节点来执行。  

这样做的好处是首先能够创建一个智能负载均衡器。 这些非数据节点仍然是集群的一部分， 他们将请求重定向到那些有相关数据的节点上。  另一个好处是对于那些scatter/gather操作(比如search),  这些节点可以执行一部分处理， 因为它们启动scatter处理并且执行实际的gather过程。   
 
这样数据节点可以专注于索引和查询这类大负载的工作，而不需要处理http请求， 占用网络负载，或者执行gather过程。   

plugins
==

[原文](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/modules-plugins.html)   

Plugins提供了以自定义的方式增强ES基本功能的途径， 范围包括添加自定义mapping类型, 自定义分词, 原生脚本， 自定义discovery等等。

installing plugins
===

安装插件可以手工将插件安装包放到`plugins`目录， 也可以用`plugin`脚本来安装。 在github的[elasticsearch](https://github.com/elasticsearch)能找到好几个插件， 名字以"elasticsearch-"开头。  

从0.90.2开始， 插件可以通过执行 `plugin --install <org>/<user/component>/<version>`的形式来安装。 插件会从`download.elasticsearch.org`自动下载, 如果插件不存在的话, 就从maven(central and sonatype)下载。  

注意， 如果插件放在maven central或者sonatype的话, `<org>`是`groupId`，`<user/component>`是`artifactId`。  

对于以前的版本， 老的安装方式是 `plugin -install <org>/<user/component>/<version>`  

一个插件也可以直接通过指定它的URL来安装， 例如  

`bin/plugin --url file://path/to/plugin --install plugin-name `  

或者对于老版本来说是  

`bin/plugin -url file://path/to/plugin -install plugin-name`  

从0.90.2开始, 插件的更新信息可以通过运行 `bin/plugin -h`来获得。  

site plugins
===

插件可以包含一个站点， 任何位于`plugins`目录下的插件如果包含一个`_site`目录， 目录里的内容就可以当做静态内容通过`/_plugin/[plugin_name]/url`来访问，在进程已经开始的情况下也可以向其添加内容。    

安装的插件如果不包含任何java相关的内容， 会被自动检测为site插件， 内容会被移动到`_site`目录下。  

安装github托管的插件非常简单， 比如运行  

```
# From 0.90.2
bin/plugin --install mobz/elasticsearch-head
bin/plugin --install lukas-vlcek/bigdesk

# From a prior version
bin/plugin -install mobz/elasticsearch-head
bin/plugin -install lukas-vlcek/bigdesk
```

会自动安装这两个site插件，elasticsearch-head插件可以通过 `http://localhost:9200/_plugin/head/`访问， bigdesk插件可以通过`http://localhost:9200/_plugin/bigdesk/`访问。  

mandatory plugins
===

如果你依赖一些插件， 你可以通过属性`plugin.mandatory`来定义这些强制性(mandatory)插件， 下面是一个配置的例子:

```
plugin.mandatory: mapper-attachments,lang-groovy
```

出于安全考虑， 如果mandatory插件没有安装， 节点不会启动。   

installed plugins
=== 

当前已加载的插件列表可通过Node Info API获得。  

removing plugins
===  

要删除一个插件，可以手工将它从`plugins`目录移除，也可以用`plugin`脚本。  

删除一个插件通常可以用下面的形式:

```
plugin --remove <pluginname>
```

silent/verbose mode
===

运行`plugin`脚本时, 可以加`--verbose`参数获得更多的信息(调试模式)。 相反的， 如果希望`plugin`脚本静默与运行可以用`--silent`选项。  

注意， 退出码可能是:

```
0: everything was OK
64: unknown command or incorrect option parameter
74: IO error
70: other errors
bin/plugin --install mobz/elasticsearch-head --verbose
plugin --remove head --silent
```

scripting
==

scripting模块可以用脚本来执行自定义表达式， 比如可以用脚本将"script fields"作为查询的一部分返回， 或者用来计算某个查询的自定义评分等。  

脚本模块默认用扩展过的[mvel](http://mvel.codehaus.org/)作为脚本语言， 之所以用是因为它非常快而且用起来很简单， 大多数情况下需要的是简单的表达式(比如数学方程式)。  

其他`lang`插件可以提供执行不同语言脚本的能力， 目前支持的脚本插件有javascript的`lang-javascript`，Groovy的`lang-groovy`， Python的`lang-python`。 所有可以用`script`参数的地方可以设置`lang`参数来定义脚本所用的语言。 `lang`的选项可以是`mvel`, `js`, `groovy`, `python`, 和`native`。  

default scripting language
===

默认的脚本语言是(如果没有指定`lang`参数)`mvel`。 如果要修改默认语言的话可以将设置`script.default_lang`为合适的语言。  

preloaded scripts
===

脚本可以作为相关api的一部分， 也可以将脚本放到`config/scripts`来预加载这些脚本， 这样用到这些脚本的地方可以直接引用脚本的名字而不用提供整个脚本， 这有助于减少客户端和节点间的传输的数据量。  

脚本的名字从其所在的目录结构继承，不需要带文件名的后缀， 例如被放在`config/scripts/group1/group2/test.py`的脚本会被命名为group1_group2_test。  

disabling dynamic scripts
===

建议ES运行在某个应用或者代理的后端，这样可以将ES和外部隔离， 如果用户被授权运行动态脚本(即使在search请求)，那么他们就可以访问ES所在的机器。  

首先， 你不应该用`root`用户来运行ES, 因为这样会允许脚本在你的服务器上没有限制的做任何事， 其次你不应该直接将ES暴露给用户访问， 如果你打算直接将ES暴露给用户， 你必须决定是否足够信任他们在你的服务器上运行脚本。 如果答案是不的话, 即使你有个代理仅允许GET请求， 你也应该在每个节点的`config/elasticsearch.yml`加入如下设置来禁用动态脚本:  

```
script.disable_dynamic: true
```

这样可以仅允许配置过的命名脚本或者通过插件注册的原生Java脚本运行， 防止用户通过接口来运行任意脚本。  

native (java) scripts
===

即使`mvel`已经相当快了，注册的原生java脚本还能执行的更快。  

实现`NativeScriptFactory`接口的脚本才会被执行。 主要有两种形式，一种是扩展`AbstractExecutableScript`，一种是扩展`AbstractSearchScript`(可能大多数用户会选择这种方式, 可以借助`AbstractLongSearchScript`, `AbstractDoubleSearchScript`, `AbstractFloatSearchScript`这几个辅助类来实现扩展)。

可以通过配置来注册脚本， 例如：`script.native.my.type`设为`sample.MyNativeScriptFactory`将注册一个名为my的脚本。 另一个途径是插件中访问`ScriptModule`的`registerScript`方法注册脚本。  

设置`lang`为`native`并且提供脚本的名字就可以执行注册的脚本。  

注意， 脚本需要位于ES的classpath下， 一个简单方法是在plugins目录下创建一个目录(选择一个描述性的名字)，将jar/classes文件放在这个目录，他们就会被自动加载。  

score
===

所有的脚本都可以在facets中使用, 可以通过`doc.score`访问当前文档的评分。  

document fields
===

大多数脚本都会用到document的字段， `doc['field_name']`可以用来访问document中的某个字段(document通常通过脚本的上下文传给脚本)。 访问document的字段非常快， 因为他们会被加载到内存中(所有相关的字段值/token会被加载到内存中)。  

下表是能够从字段上拿到的数据：

| Expression       				| Description   											|
| ------------------------------|:----------------------------------------------------------|
| doc['field_name'].value   	| 字段的原生值， 比如，如果是字段short类型，就返回short类型的值	|
| doc['field_name'].values  	| 字段的原生值的数组， 比如，如果字段是short类型，就返回short[]类型的数组。 记住，单个文档中的一个字段可以有好几个值，如果字段没有值就返回空数组 |
| doc['field_name'].empty   	| boolean值， 表明文档的字段是否有值 							|
| doc['field_name'].multiValued | boolean值， 表明文档的字段是否有多个值 						|
| doc['field_name'].lat 		| geo point类型的维度值				 						|
| doc['field_name'].lon 		| geo point类型的经度值				 						|
| doc['field_name'].lats 		| geo point类型的维度数组				 						|
| doc['field_name'].lons 		| geo point类型的经度数组				 						|
| doc['field_name'].distance(lat, lon)			| geo point类型的字段到给定坐标的plane距离(单位是miles)|
| doc['field_name'].arcDistance(lat, lon)		| geo point类型的字段到给定坐标的arc距离(单位是miles)	|
| doc['field_name'].distanceInKm(lat, lon)		| geo point类型的字段到给定坐标的plane距离(单位是km)	|
| doc['field_name'].arcDistanceInKm(lat, lon)	| geo point类型的字段到给定坐标的arc距离(单位是km)		|
| doc['field_name'].geohashDistance(geohash)	| geo point类型的字段到给定geohash的距离(单位是miles)	|
| doc['field_name'].geohashDistanceInKm(geohash)| geo point类型的字段到给定geohash的距离(单位是km)		|


stored fields
===

执行脚本时也可以访问存储的字段(Stored)， 注意，因为他们不会被记载到内存，所以访问速度与访问document字段相比慢很多。 可以用`_fields['my_fields_name'].value`或`_fields['my_fields_name'].values`的形式来访问。   

source field
===

执行脚本时也可以获取源字段(source)。 每个文档的源字段会被加载，解析，然后提供给脚本计算。 可以通过上下文的`_source`来访问源字段，例如`_source.obj2.obj1.fields3`。  

mvel built in functions
===

以下是脚本中可以使用的内置函数:

| Function  	| Description   											|
| --------------|:----------------------------------------------------------|
| time()   		| The current time in milliseconds.							|
| sin(a)   		| Returns the trigonometric sine of an angle.				|
| cos(a)  		| Returns the trigonometric cosine of an angle.				|
| tan(a)  		| Returns the trigonometric tangent of an angle.			|
| asin(a)  		| Returns the arc sine of a value.							|
| acos(a)  		| Returns the arc cosine of a value.						|
| atan(a) 		| Returns the arc tangent of a value.						|
| toRadians(angdeg) | Converts an angle measured in degrees to an approximately equivalent angle measured in radians.	|
| toDegrees(angrad) | Converts an angle measured in radians to an approximately equivalent angle measured in degrees.	|
| exp(a) 		| Returns Euler’s number e raised to the power of value.	|
| log(a) 		| Returns the natural logarithm (base e) of a value.		|
| log10(a) 		| Returns the base 10 logarithm of a value.					|
| sqrt(a) 		| Returns the correctly rounded positive square root of a value.	|
| cbrt(a) 		| Returns the cube root of a double value.					|
| IEEEremainder(f1, f2)	| Computes the remainder operation on two arguments as prescribed by the IEEE 754 standard.	|
| ceil(a) 		| Returns the smallest (closest to negative infinity) value that is greater than or equal to the argument and is equal to a mathematical integer.	|
| floor(a) 		| Returns the largest (closest to positive infinity) value that is less than or equal to the argument and is equal to a mathematical integer.		|
| rint(a) 		| Returns the value that is closest in value to the argument and is equal to a mathematical integer.|
| atan2(y, x)	| Returns the angle theta from the conversion of rectangular coordinates (x, y) to polar coordinates (r,theta).	|
| pow(a, b) 	| Returns the value of the first argument raised to the power of the second argument. |
| round(a)		| Returns the closest int to the argument.					|
| random()		| Returns a random double value.							|
| abs(a)		| Returns the absolute value of a value.					|
| max(a, b)		| Returns the greater of two values.						|
| min(a, b)		| Returns the smaller of two values.						|
| ulp(d)		| Returns the size of an ulp of the argument.				|
| signum(d)		| Returns the signum function of the argument.				|
| sinh(x)		| Returns the hyperbolic sine of a value.					|
| cosh(x)		| Returns the hyperbolic cosine of a value.					|
| tanh(x)		| Returns the hyperbolic tangent of a value.				|
| hypot(x, y)	| Returns sqrt(x2 + y2) without intermediate overflow or underflow.	|


arithmetic precision in mvel
===

用基于MVEL的脚本做两个数的除法时， 引擎遵循java的默认规则， 这意味着如果你把两个整数相除(你可以在mapping里配置字段为integer类型)， 结果仍然是整数。 也就是说如果你在脚本中计算`1/num`这样的表达式， 如果num是整数8的话，结果是0而不是你可能期望的0.125，你需要明确用doubel来指定精度以获得期望的结果，用比如`1.0/num`。  

thread pool
==

为了提高线程管理和内存使用的效率， 一个节点会用到好几个线程池， 比较重要的是以下几个。  

| Function  	| Description   											|
| --------------|:----------------------------------------------------------|
| index   		| 用于index/delete, 默认是`fixed`, 大小为`# of available processors`, queue_size是`200`		|
| search   		| 用于count/search, 默认是`fixed`, 大小为`3x # of available processors`, queue_size是`1000`	|
| suggest  		| 用于suggest, 默认是`fixed`, 大小为`# of available processors`, queue_size是`1000`			|
| geting  		| 用于get, 默认是`fixed`, 大小为`# of available processors`, queue_size是`1000`				|
| bulk  		| 用于bulk, 默认是`fixed`, 大小为`# of available processors`, queue_size是`50`				|
| percolate  	| 用于percolate, 默认是`fixed`, 大小为`# of available processors`, queue_size是`1000`			|
| warmer	  	| 用于warm-up, 默认是`scaling`, keep-alive是`5m `												|
| refresh  		| 用于refresh, 默认是`fixed`, keep-alive是`5m `												|
| percolate  	| 用于percolate, 默认是`fixed`, 大小为`# of available processors`, queue_size是`1000`			|

可以通过设置线程池的类型以及参数来修改指定的线程池，下面的例子配置index线程池可以用更多的线程：  

```
threadpool:
    index:
        type: fixed
        size: 30
```

注意， 可以通过Cluster Update Settings接口在运行期间修改线程池的设置。  

thread pool types
===

以下是线程池的类型以及对应的参数。  

cache

cache线程池是没有大小限制的， 只要有请求就会启动一个线程， 下面是设置的例子:

```
threadpool:
    index:
        type: cached
```

fixed

fixed线程池有固定的大小， 如果当前没有可用的线程时，就把请求放到一个队列里， 队列可以设置大小。  

`size`参数设置线程的数量， 默认是cpu内核数乘以5。  

`queue_size`设置存放挂起请求的队列的大小， 默认是-1， 即没有大小限制。  如果一个请求进来而队列已经满了的情况下， 这个请求会被舍弃。 

配置例子如下:

```
threadpool:
    index:
        type: fixed
        size: 30
        queue_size: 1000
```

processors setting
===

ES会自动检测处理器的数量， 并且会自动根据处理器的数量来设置线程池。 有时候可能检测出来的处理器数量是错的， 这种情况下可以设置`processors` 来声明处理器的数量。  

可以用带`os`参数的nodes info接口来检查自动检测出来的处理器数量。    

thrift
==

thrift传输模块允许通过thrift协议对外暴露REST接口， Thrift协议可以提供比http协议更好的性能。 因为thrift既提供了协议，也提供了传输的实现方式， 用起来应该很简单(尽管缺乏相关文档)。

使用thrift需要安装[transport-thrift插件](https://github.com/elasticsearch/elasticsearch-transport-thrift) 。 

[thrift schema](https://github.com/elasticsearch/elasticsearch-transport-thrift/blob/master/elasticsearch.thrift)可以用来生成thrift的客户端代码。  

thrift的相关配置如下:  

| Setting    	| Description   											|
| --------------|:----------------------------------------------------------|
| thrift.port   | 绑定的端口， 默认9500-9600									|
| thrift.frame  | 默认-1,  即不分frame, 可以设置比较大的值来指定frame的大小(比如15mb)。|


transport
==

传输模块用于集群内节点间的内部通讯。 每次跨节点的调用都会用到transport模块(比如某个节点接受http GET请求，实际执行处理的是另一个持有数据的节点)。  

transport机制是完全异步的， 即等待响应时不会阻塞线程， 异步通信的好处首先是解决了C10k问题， 另外也是scatter(broadcast)/gather操作(比如搜索)的方案。  

tcp transport
===

TCP transport是用TCP实现传输模块， 允许如下设置:

| Setting    					| Description   											|
| ------------------------------|:----------------------------------------------------------|
| transport.tcp.port    		| 绑定的端口范围， 默认9300-9400								|
| transport.tcp.connect_timeout | socket链接的超时时间， 默认2s 								|
| transport.tcp.compress 		| 设置为true可以启用压缩(LZF)， 默认是false。 					|


tcp transport共享通用网络设置。  

local transport
===

这在JVM内做集成测试时非常有用。 当使用`NodeBuilder#local(true)`时会自动启用。


