modules
==

模块

----------

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

```json
curl -XPUT localhost:9200/test/_settings -d '{
      "index.routing.allocation.include.tag" : "value1,value2"
}'
```

另一方面， 我们将配置项`index.routing.allocation.exclude.tag`设置为value3， 这样创建的索引会被部署到tag属性为value3之外的那些节点上，例如

```json
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

----------

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

```json
{
    "request" : {
        "cluster_name": "test_cluster"
    }
}
```

响应格式类似节点信息的响应(只有节点信息，包括transport/http地址以及节点的属性):

```json
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

------------------

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

----------

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

----------

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


----------

jmx
==

removed as of v0.90.  

----------

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

----------

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


----------

node
==

[原文](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/modules-node.html)  

ES可以设置一个节点是否在本地存储数据，存储数据意味着不同索引的分片可以分配到这个节点上。 默认每个节点都可以作为数据节点(data node)，可以设置`node.data`为false来关闭。  

这是一个很强大的配置， 可以很简单的来创建一个智能负载均衡。  

我们可以启动一个数据节点的集群而不启用http模块， 这可以通过设置`http.enabled`为true做到， 这些节点通过transport模块相互通信， 在集群的前端可以启动一个和或者多个启用了http模块的非数据节点， 所有的http通讯由这些非数据节点来执行。  

这样做的好处是首先能够创建一个智能负载均衡器。 这些非数据节点仍然是集群的一部分， 他们将请求重定向到那些有相关数据的节点上。  另一个好处是对于那些scatter/gather操作(比如search),  这些节点可以执行一部分处理， 因为它们启动scatter处理并且执行实际的gather过程。   
 
这样数据节点可以专注于索引和查询这类大负载的工作，而不需要处理http请求， 占用网络负载，或者执行gather过程。   


----------

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

----------

scripting
==

----------

thread pool
==


----------

thrift
==

----------

transport
==
