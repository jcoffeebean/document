mapping
==

Mapping是指定义如何将document映射到搜索引擎的过程，比如一个字段是否可以查询以及如何分词等，一个索引可以存储含有不同"mapping types"的documents，ES允许每个mapping type关联多个mapping定义。

显式声明的mapping是定义在index/type级别， 默认不需要显式的定义mapping， 当新的type或者field引入时，ES会自动创建并且注册有合理的默认值的mapping(毫无性能压力)， 只有要覆盖默认值时才必须要提供mapping定义。  

mapping types

Mapping types是将索引里的documents按逻辑分组的方式， 类似数据中的表， 虽然不同的types之间有些区别， 但他们并不是完全分开的(说到底还是存在相同的Lucene索引里)。  

强烈建议跨types的同名field有相同的类型定义以及相同的mapping特征(比如analysis的设置)， 这在通过type前缀(`my_type.my_field`)来选择字段时非常有效， 但这也不一定， 有些地方就不起作用(比如字段的聚合faceting)。  

实际上在实践中这个限制从来不是问题， field名通常表明了该field的类型(例如"first_name"总是一个字符串)。 还要注意， 这不适用于跨索引的情况。   

mapping api

要创建mapping， 需要用到Put Mapping接口, 或者可以在调用create index接口时附带mapping的定义。  

global settings

全局设置`index.mapping.ignore_malformed`可以在索引级别上设置是否忽略异常内容(异常内容的一个例子是尝试将字符串类型的值作为数字类型索引)， 这个设置是跨mapping types的全局设置。  

fields
===

每一个ampping都有一些关联的字段来控制如何索引的document的元数据(例如_all)。  

_uid
===

每个索引的document会关联一个id和一个type,  内部的`_uid`字段将type和id组合起来作为document的唯一标示(这意味着不同的type可以有相同的id， 组合起来仍然是唯一的)。  

在执行基于type的过滤时， 如果`_type`字段没有被索引，会自动使用`_uid`字段， 并且不需要`_id`字段被索引。  


```
附_uid的java源代码

public static final byte DELIMITER_BYTE = 0x23;

public static void createUidAsBytes(BytesRef type, BytesRef id, BytesRef spare) {
    spare.copyBytes(type);
    spare.append(DELIMITER_BYTES);
    spare.append(id);
}
```

_id
===

每个索引的document会关联一个id和一个type, `_id`字段就是用来索引并且存储(可能)id的，默认是不索引(not indexed)并且不存储的(not stored)。  

注意， 即使`_id`是不索引的, 相关的接口仍然起作用(他们会用`_uid`字段)， 比如用`term`, `terms`或者`prefix`来根据ids过滤(包括用`ids`来查询/过滤)。  

`_id`字段也可以启用索引或者存储, 配置如下:

```
{
    "tweet" : {
        "_id" : {"index": "not_analyzed", "store" : "yes"}
    }
}
```

为了维护向后兼容性， 当升级到0.16时可以在节点级别设置`index.mapping._id.indexed`为true来确保id能被索引, 尽管不建议索引id。  

可以设置`_id`的`path`属性来从源文档中提取id， 例如下面的mapping:

```
{
    "tweet" : {
        "_id" : {
            "path" : "post_id"
        }
    }
}
```

如果提交下面的数据

```
{
    "message" : "You know, for Search",
    "post_id" : "1"
}
```

`1`会提取出来作为id。  

因为要提取id来决定在哪一个shard执行索引，需要在索引时做额外的解析。 

_type
===

每个索引的document会关联一个id和一个type, type在索引时会自动赋给`_type`字段, 默认`_type`字段是需要索引的(但不analyzed)并且不存储的， 这就意味着`_type`字段是可查询的。  


`_type`字段也可以设置为stored, 例如:

```
{
    "tweet" : {
        "_type" : {"store" : "yes"}
    }
}
```

`_type`字段也可以设置为不索引, 并且此时所有用到`_type`字段的接口仍然能用。 

```
{
    "tweet" : {
        "_type" : {"index" : "no"}
    }
}
```

_source
===

`_source`是一个自动生成的字段， 用来存储实际提交的JSON数据， 他是不索引的(不可搜索), 只是用来存储。 在执行"fetch"类的请求时, 比如get或者search, `_source`字段默认也会返回。 

尽管`_source`非常有用， 但它确实会占用索引的存储空间， 所以也可以禁用。 比如:

```
{
    "tweet" : {
        "_source" : {"enabled" : false}
    }
}
```

compression

从0.90开始， 所有存储的字段(包括`_source`)总是被压缩的。 

0.90之前:

如果要将source字段存储在索引中的话，启用压缩(LZF)会显著减少索引的大小， 还可能提升性能(解压缩比从磁盘上加载一个比较大的source的性能要好)。 代码需要特别注意，只有需要的时候才执行解压缩， 例如直接将数据解压缩到REST的结果流。 

要启用压缩的话， 需要将`compress`选项设置为true， 默认设置是false。 注意可以在已经存在的索引上修改，ES支持压缩和未压缩的数据混合存放。 

另外,`compress_threshold`可以控制压缩source的时机，可以设置为表示字节大小的值(比如100b, 10kb)。 注意`compress`应该设置为true。 

includes / excludes

可以用path属性来包含/排除source中要存储的字段，支持*通配符，例如:

```
{
    "my_type" : {
        "_source" : {
            "includes" : ["path1.*", "path2.*"],
            "excludes" : ["pat3.*"]
        }
    }
}
```

_all
===

`_all`字段的设计目的是用来包罗文档的一个或多个字段， 这对一些特定的查询非常有用， 比如我们要查询文档的内容， 但是不确定要具体查询哪一个字段， 这会占用额外的cpu和索引容量。  

`_all`字段可以完全禁止掉， field mapping和object mapping可以声明这个字段是否放到`_all`中。 默认所有的字段都包含在`_all`中。  

禁用`_all`字段时， 推荐为`index.query.default_field`设置一个值(例如， 你的数据有一个"message"字段来存储主要的内容, 就设置为`message`)。  

`_all`字段一个很有用的特征是可以把字段的boost等级考虑进去， 假设title字段的boost等级比content字段高, `_all`中的title值也比`_all`中的content值等级高。  

以下是一个配置的例子:
```
{
    "person" : {
        "_all" : {"enabled" : true},
        "properties" : {
            "name" : {
                "type" : "object",
                "dynamic" : false,
                "properties" : {
                    "first" : {"type" : "string", "store" : "yes", "include_in_all" : false},
                    "last" : {"type" : "string", "index" : "not_analyzed"}
                }
            },
            "address" : {
                "type" : "object",
                "include_in_all" : false,
                "properties" : {
                    "first" : {
                        "properties" : {
                            "location" : {"type" : "string", "store" : "yes", "index_name" : "firstLocation"}
                        }
                    },
                    "last" : {
                        "properties" : {
                            "location" : {"type" : "string"}
                        }
                    }
                }
            },
            "simple1" : {"type" : "long", "include_in_all" : true},
            "simple2" : {"type" : "long", "include_in_all" : false}
        }
    }
}
```

在这个例子里， `_all`字段设置了`store`, `term_vector`和`analyzer`(指定`index_analyzer`和`search_analyzer`)。  

highlighting

任何可以highlighting的字段必须既是stored的，又是`_source`的一部分，默认`_all`字段不符合这个条件， 所以它的highlighting不会返回任何数据。  

尽管可以设置`_all`为stored， 但`_all`从根本上说是所有字段的集合,  也就是说会存储多余的数据，它做highlighting可能产生怪怪的结果。  

_analyzer
===

`_analyzer` mapping可以将document某个字段的值作为索引时所用analyzer的名字，如果一个字段没有显式指定`analyzer`或者`index_analyzer`, 索引时就会用这个analyzer。  

下面是配置的例子:

```
{
    "type1" : {
        "_analyzer" : {
            "path" : "my_field"
        }
    }
}
```

上面的配置用`my_field`字段的值作为analyzer， 比如下面的文档:


```
{
    "my_field" : "whitespace"
}
```

会让所有没有指定analyzer的字段用`whitespace`做索引的analyzer。  

path的默认值是`_analyzer`,  所以可以给`_analyzer`字段赋值来指定一个analyzer， 如果需要自定义为别的json字段， 需要通过path属性来明确指定。  

默认`_analyzer`字段是可索引的, 可以在mapping中将`index`设置为`no`来禁用。   


_boost
===

Boosting是增强文档或者字段关联性的过程，字段级别的mapping可以将boost指定为某个字段。 `_boost`(应用在root object上)可以指定一个字段，这个字段的内容控制文档的boost级别。 例如下面的mapping:

```
{
    "tweet" : {
        "_boost" : {"name" : "my_boost", "null_value" : 1.0}
    }
}
```

上面的定义指定了一个名为字段`my_boost`的字段， 如果要索引的JSON文档包括my_boost字段， 字段的值就作为文档的boost值， 比如下面的JSON文档的boost值为2.2:

```
{
    "my_boost" : 2.2,
    "message" : "This is a tweet!"
}
```

(注:name属性默认是`_boost`)  


_parent
===

`_parent`用来定义子类型所关联的父类型， 比如有一个`blog`类型和一个`blog_tag`子类型， `blog_tag`的mapping应该是:

```
{
    "blog_tag" : {
        "_parent" : {
            "type" : "blog"
        }
    }
}
```

`_parent`默认是stored以及indexed的， 也就是说可以用`_parent`来查询。  

_routing
===

`routing`是索引数据或者需要明确指定路由时routing的设置。

store / index

`_routing`的mapping默认会存储routing的值(`store`设置为`yes`)， 之所以这么做是为了可以在routing值来自外部而不是document一部分时仍然可以重建索引。 

required

另一方面， 可以在`_routing`的mapping中设置`required`属性为`true`来将它设置为必需的，这在使用routing功能是非常重要， 因为很多接口会用到它。 如果没有提供routing值(或者不能从document获取)的话，索引操作就不会执行，再比如如果`_routing`是必须的但是没有提供routing值的话，删除操作就会广播到所有的分片(shards)上。

path

routing的值可以在索引时额外提供(并且作为document的一部分存储， 和`_source`字段存储方式很像)， 也可以根据`path`自动从要索引的document提取， 例如下面的mapping:

```
{
    "comment" : {
        "_routing" : {
            "required" : true,
            "path" : "blog.post_id"
        }
    }
}
```

会使下面的document基于值`111222`来路由： 

```
{
    "text" : "the comment text"
    "blog" : {
        "post_id" : "111222"
    }
}
```

注意， 使用`path`而不是明确提供routing值的话， 索引时需要额外的解析过程(尽管相当快)。  

id uniqueness

如果自定义`_routing`的话， 不保证`_id`在所有分片(shards)的唯一性。 事实上, 如果document的`_id`相同而`_routing`值不同的话， 会被分配到不同分片上。 

_index
===

`_index`存储一个document属于哪一个索引(index)， 该字段默认是禁用的，  如果要启用的话， mapping的定义如下:

```
{
    "tweet" : {
        "_index" : { "enabled" : true }
    }
}
```

_size
===

`_size`字段自动存储原始`_source`的大小， 默认是禁用的， 要启用的话mapping定义如下:

```
{
    "tweet" : {
        "_size" : {"enabled" : true}
    }
}
```

如果还要存储的话， 定义如下:

```
{
    "tweet" : {
        "_size" : {"enabled" : true, "store" : "yes"}
    }
}
```

_timestamp
===


`_timestamp`字段允许自动索引一个document的时间戳， 它可以在索引请求时提供， 也可以从`_source`提取，如果没有提供的话会自动设置为document被处理的时间。  

enabled

`_timestamp`默认是禁用的,  如果要启用， mapping的定义如下所示:

```
{
    "tweet" : {
        "_timestamp" : { "enabled" : true }
    }
}
```

store / index

默认`_timestamp`字段的`store`设置为`no`， `index`设置为`not_analyzed`， 它可以当做一个标准的日期字段来查询。 

path

`_timestamp`的值可以在索引请求时额外提供， 也可以根据`path`自动从document中提取， 例如下面的mapping定义:

```
{
    "tweet" : {
        "_timestamp" : {
            "enabled" : true,
            "path" : "post_date"
        }
    }
}
```

提交的数据为

```
{
    "message" : "You know, for Search",
    "post_date" : "2009-11-15T14:12:12"
}
```

时间戳的值就是`2009-11-15T14:12:12`。


注意， 如果用`path`方式而没有明确提供时间戳的值的话， 索引时需要额外的解析操作(尽管相当快)。

format

你可以定义时间戳的格式， 例如:

```
{
    "tweet" : {
        "_timestamp" : {
            "enabled" : true,
            "path" : "post_date",
            "format" : "YYYY-MM-dd"
        }
    }
}
```

注意, 默认的格式是`dateOptionalTime`， 时间戳的值首先作为数字解析， 如果解析失败的话会尝试用定义的格式解析。 


_ttl
===

很多documents有过期时间， 可以设置`_ttl`(time to live)来自动删除过期的documents。 

enabled

`_ttl`默认是禁用的,  要启用的话， mapping定义如下:

```
{
    "tweet" : {
        "_ttl" : { "enabled" : true }
    }
}
```

store / index

默认`_ttl`字段的`store`设置为`yes`， `index`设置为`not_analyzed`， 注意`index`必须设置为`not_analyzed`。 

default

可以为index/type设置默认的`_ttl`， 比如:

```
{
    "tweet" : {
        "_ttl" : { "enabled" : true, "default" : "1d" }
    }
}
```

这种情况下, 如果你没有明确提供`_ttl`的值， `_source`里也没有`_ttl`的话， 所有的tweets的`_ttl`会被设置为一天。  

如果你没有指定时间的单位，比如d (days), m (minutes), h (hours), ms (milliseconds), (weeks), 默认把毫秒(milliseconds)作为单位。  

如果没有设置默认值， 也没有提供`_ttl`的值， document会有无限的`_ttl`，即永不过期。  

可以用put mapping接口动态更新`default`的值， 这不会改变已有documents的`_ttl`， 只会影响新的documents。 

note on documents expiration

过期的documents会自动定期删除， 可以根据你的需要来设置`indices.ttl.interval`， 默认是60s。 

删除命令是批量处理的， 可以根据你的需要来设置`indices.ttl.bulk_size`， 默认是10000。 

注意， 删除是根据版本来的， 如果document在收集过期的documents和执行删除操作的时间间隔之间被修改了， document是不会被删除的。 
