---
title: Redis 数据类型
date: 2021-06-16 23:44:41
categories: 中间件
tags: [Redis]
---

Redis 可以理解为一个大号的 Map，其中所有的数据都采用 key:value 的形式维护，在 Redis 中，一个 key 总是对应一个 value。其中 key 永远是字符串，而一般提到 Redis 的数据类型，指的是其存储的 value 的数据类型。下面介绍 Redis 常见的数据类型，并简单介绍常见的相关指令。

<!--more-->

# string

string 是最基本的数据类型，其他几个数据类型底层也是通过 string 来实现的。

{% mermaid flowchart LR %}
    subgraph values
        value-one
        value-two
    end
    subgraph keys
        key:one --> value-one
        key:two --> value-two
    end
{% endmermaid %}

string 有以下几个特点：

- string 类型用 nil 表示不存在的值，类似于编程语言中的 null。
- string 类型可存储的最大长度为 512M。
- Redis 中没有专门的数字类型，当一个 string 中存放的数据全部为数字、且该数字的范围在 64位长整型范围中时，Redis 可以将其当作数字进行操作。

## 基本操作

- 添加/修改数据：`set key value`，当 key 已存在时会更新其对应的值，返回 OK 表示执行成功，其他很多命令执行成功时也会返回 OK，不再赘述。
- 获取数据：`get key`，当 key 不存在时返回 nil。
- 添加/修改一个或多个数据：`mset key value [key value ...]`。 `[key value ...]` 表示该命令的参数可以是任意数量的 key、value 对，这种写法与 Redis 官方文档相同，本文也多次使用这种方式。
- 获取一个或多个数据：`mget key [key ...]`，返回的顺序与参数指定的顺序相同，如果参数中传入了不存在的 key，返回结果中对应位置也会有一个 nil。
- 删除一个或多个：`del key [key ...]`，与 MySQL 类似，返回结果是一个 Integer，表示改动的记录数量，其他很多命令执行成功也会返回 OK，不再赘述。
- 获取字符串长度：`strlen key`。
- 追加信息到原数据后：`append key value`，追加完成后返回新的字符串长度。

## 拓展操作

### 数字的自增自减

- 自增一 `incr key`，如果 key 对应的值不是纯数字字符串、或自增后会超过64位长整型范围，该指令会报错。
- 自增指定的整数 `incrby key increment`，increment 自增系数如果是负数也可以实现自减的操作。
- 自增指定的浮点数 `incrbyfloat key increment`
- 自减一 `decr key`
- 自减指定的整数 `decrby key decrement`
- 没有自减指定的浮点数命令

# hash

哈希表 hash 用于分组存储数据，可类比编程语言中的哈希数据结构，一个 key 对应一整个哈希表，哈希表中可以有多个字段 field 和字段的对应值 value，字段名和字段值都是字符串，受 string 类型特点的约束。

{% mermaid flowchart LR %}
    subgraph values
        subgraph hash
            field-one --> value-one
            field-two --> value-two
        end
    end
    subgraph keys
        subgraph key
        end
    end
    key --> hash
{% endmermaid %}

哈希结构有以下几个特点：

- 在一个哈希表中，字段不可重复，字段值可以重复。
- 仍保有哈希表的普遍特点，例如查询效率高等。
- 每个哈希表可以存储 2<sup>32</sup> - 1个键值对。

## 基本操作

- 添加/修改哈希表中字段和对应的值：`hset key field value [field value ...]`。此外还有功能类似的 `hmset` 命令，在 Redis 官方文档中被标记为已废弃。
- 获取哈希表中某个字段的值：`hget key field` 
- 获取哈希表中所有字段的值：`hgetall key`
- 获取哈希表中某些字段的值：`hmget key field [feild ...]`
- 获取哈希表中字段的数量：`hlen key`
- 检查哈希表中是否存在指定的字段：`hexists key field`
- 删除哈希表中某些数据：`hdel key field [field ...]`

## 扩展操作

### 独有操作

- 获取一个哈希表中所有字段名：`hkeys key`
- 获取一个哈希表中所有字段值：`hvals key`

### 数字的自增自减

- 将一个哈希表中某个字段的值自增指定的整数：`hincrby key field increment`，与前面出现的自增指令相同，increment 为负值时可以实现自减的功能。
- 将一个哈希表中某个字段的值自增指定的浮点数：`hincrbyfloat key field increment`

# list

列表 list 用于区分数据进出的顺序，类似与编程语言中的双端队列，一个 key 对应一个列表，列表中存放的是元素 element，每个元素都是由 string 构成的，受 string 类型特点的约束。通过专门的、仅使用一部分命令可分别实现栈或队列。

{% mermaid flowchart LR %}
    subgraph values
        subgraph element
            elementone[[elementone]]
            elementtwo[[elementtwo]]
            elementthree[[elementthree]] 
            elementone<-->elementtwo<-->elementthree           
        end
    end
    subgraph keys
        subgraph key
        end
    end
    key-->element
{% endmermaid %}

- 列表中最多可以存储 2<sup>32</sup> - 1个元素。
- 列表的相关命令较多，因为 Redis 为很多操作都提供了从左侧和右侧获取两种方式，分别以 l 和 r 开头。
- 列表的元素可通过索引获取，索引编号从 0 开始，-1 表示倒数第一个元素。后续命令中，index、start 和 stop 指索引的范围。

## 基本操作

- 添加元素到列表中：左侧添加`lpush key element [element ...]` 和右侧添加  `rpush ley element [element ...]`，**由于列表结构的特殊性，这两条命令不能通过覆盖原始值的方式实现修改的功能**。
- 获取列表中指定位置的元素：`lindex key index`，这条命令没有对应的 `rindex` 命令，可以理解为此处的 l 是 list 的缩写，而不是 left，下面两条同理。
- 获取列表的长度：`llen key`
- 获取列表指定范围内的元素：`lrange key start stop`
- 获取并移除列表中的元素：`lpop key` 和 `rpop key`

## 拓展操作

- 从列表左侧开始，删除指定的元素，删除一定数量后停止：`lrem key count element` 。假如列表 mylist 中有几个元素如下：*a b c a b c a b c*，执行命令 `lrem mylist 2 b ` 后，队列中剩下的元素是 *a c a c a b c*。

# set

集合 set 与哈希结构类似，一个 key 对应一个集合，集合中存放的是成员 member，成员都是由 string 组成的，受 string 类型特点的约束。其底层实现方式就是在哈希的字段名上存放数据、而所有的字段值都是 nil。

{% mermaid flowchart LR %}
    subgraph values
        subgraph set
            member-one
            member-two
        end
    end
    subgraph keys
        subgraph key
        end
    end
    member-one --> nil
    member-two --> nil
    key --> set
{% endmermaid %}

集合结构有以下几个特点：

- 仍保有 set 集合的普遍特点，例如不能存放重复值，高效的查询等。
- 每个集合可以存储 2<sup>32</sup> - 1个值对。

## 基本操作

- 添加成员：`sdd key member [member ...] `，当指定成员存在时，既不必更新也不会报错。
- 获取全部成员：`smembers key`
- 删除数据：`srem key member [member ...]`

## 拓展操作

- 获取集合成员总数：`scard key`
- 判断集合中是否包含指定成员：`sismember key member`，返回 1 表示存在、返回 0 表示不存在。

### 随机获取

- 随机获取集合中指定数量的成员：`srandmember key [count]`
- 随机获取集合中指定数量的成员并移出集合：`spop key [count]` 

### 集合运算

- 求交集并返回结果：`sinter key [key ...]`
- 求并集并返回结果：`sunion key [key ...]`
- 求差集并返回结果：`sdiff key [key ...]`
- 求交集并保存结果到目标集合：`sinterstore destination key [key ...]`，destination 是目标集合的 key。
- 求并集并保存结果到目标集合：`sunionstore destination key [key ...]`
- 求差集并保存结果到目标集合：`sdiffstore destination key [key ...]`
- 将原集合中某个数据转移到目标集合中：`smove source destination member`

# sorted_set

有序集合 sorted_set （也缩写为 zset）是在 set 的基础上，又增加了用于排序的分数 score。

{% mermaid flowchart LR %}
    subgraph values
        subgraph set
            score-one
            score-two
            member-one
            member-two
        end
    end
    subgraph keys
        subgraph key
        end
    end
    score-one --- member-one --> nil
    score-two --- member-two --> nil
    key --> set
{% endmermaid %}

## 基础操作

- 添加/修改成员和对应的分数：`zadd key score member [score member ...]`，当 member 已存在时，这条指令可以用于更新分数。
- 获取集合的成员总数：`zcard key`
- 获取集合中指定成员对应的分数：`score key member`
- 删除集合中指定的成员：`zrem key member [member ...]`

### 根据索引操作

- 获取指定索引范围内的成员，按分数升序排列：`zrange key start stop [WITHSORES]`，如果入参有 withscores 返回结果也会带上成员对应的分数。
- 获取指定索引范围内的成员，按分数降序排列：`zrevrange key start stop [WITHSORES]`
- 按索引删除数据：`zremrangebyrank key start stop`

### 根据分数操作

- 获取指定分数范围内的成员，按分数升序排列：`zrangebyscore key min max [WITHSCORES] [LIMIT offset count]`。操作分数时，默认是闭区间，可加上符号 `(` 表示开区间。`[LIMIT offset count]` 是偏移量和计数的限制。
- 获取指定分数范围内的成员，按分数降序排列：`zrevrangebyscore key min max [WITHSCORES] [LIMIT offset count]`
- 按分数删除数据：`zremrangebyscore key min max`
- 计算某个分数范围内成员总数：`zcount key min max`
- 返回集合中指定成员按分数升序排列后的排名：`zrank key member`
- 返回集合中指定成员按分数降序排列后的排名：`zrevrank key member`

## 拓展操作

### 数字的自增自减

- 将集合中指定成员的数据自增：`zincrby key increment member`

### 集合运算

- 交集：`zinterstore destination numkeys key [key ...]`
- 并集：`zunionstore destination numkeys key [key ...]`
