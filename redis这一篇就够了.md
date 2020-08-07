
- [redis 应用+底层实现](#redis--------)
  * [redis数据结构：](#redis-----)
    + [String（int(整形)\emstr(字符长度小于44字节)\row（字符串大于44字节））](#string-int-----emstr-------44----row------44----)
    + [命令:](#---)
  * [hash(ziplist\hashtable)](#hash-ziplist-hashtable-)
    + [ziplist整体数据结构：](#ziplist-------)
    + [zlentry数据结构：](#zlentry-----)
    + [hashtable 结构(转换条件：任何一个节点大于64个字节或个数大于512)](#hashtable-----------------64--------512-)
      - [扩容 缩容](#-----)
      - [命令](#--)
  * [list(quicklist)](#list-quicklist-)
    + [数据结构：](#-----)
    + [命令](#---1)
  * [set 基于hashtable value 为 null实现](#set---hashtable-value---null--)
    + [命令](#---2)
  * [zset](#zset)
    + [命令](#---3)
  * [redis　lua 用来做事务](#redis-lua------)
    + [命令](#---4)
    + [LUＡ 脚本语法](#lu------)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>
# redis 应用+底层实现

>redis 整体为 key value形式存储　结构为 dictEntry，dictEntry结构如下
>> key为ＳDS     
>> value　为 redisObject，redisObject结构如下
```json
{
    "type":"数据类型",//String list hash set zset
    "enconding":"具体数据结构",
    "lru":"对象最后访问时间",
    "refcount":"引用个数",
    "*ptr":"集体数据结构"
}
```
>通过 object encoding key　查询key 原始类型
## redis数据结构： 
### String（int(整形)\emstr(字符长度小于44字节)\row（字符串大于44字节））
数据结构:SDS（简单动态字符串）数据结构如下：
```json
{
    "len":"数组长度",
    "alloc":"分配总共内存大小",//2的　5\8\16\32次幂个字节
    "flags":"字符数组属性",//sdshdr 8 \sdshdr 16\sdshdr 32
    "buf":"数组内容"
}
```
### 命令:

![avatar](pic/String.png)

## hash(ziplist\hashtable)
### ziplist整体数据结构：
```json
{
    "zlbytes":"list 大小",
    "ztail":"链表头",
    "zllen":"联表长度",
    "zlend":"链表结尾",
    "zlentry":"链表内容"
}
```
### zlentry数据结构：
```json
{
    "prevrawlensize":"上一个节点长度",
    "prevrawlen":"上一个节点所用字节数",
    "lensize":"当前节点长度",
    "len":"当前所用字节数",
    "headersize":"当前非数据区域大小",
    "encoding":"编码方式",
    "*p":"数据内容指针 sds"
}
```
### hashtable 结构(转换条件：任何一个节点大于64个字节或个数大于512)
```json
{
    "dict":{
        "type":"字典类型",
        "privdata":"私有数据",
        "rehashidx":"rehash索引",
        "iterators":"迭代器数量",
        "dicht":[{
            "**table":"类似于hashMap",
            "size":"大小",
            "sizemask":"",
            "used":""
        }
        {
            //用于扩容
            "**table":"类似于hashMap",
            "size":"大小",
            "sizemask":"",
            "used":""
        }]
    }
}
```
#### 扩容 缩容      

#### 命令

![avatar](pic/hash.png)

## list(quicklist)

### 数据结构：

基于双向链表　借鉴二分法或查找树实现思想的跳表

### 命令 
![avatar](pic/list.png)

## set 基于hashtable value 为 null实现

### 命令

![avatar](pic/set.png)

## zset 

### 命令

![avatar](pic/zset.png)

##　发布订阅实现

//订阅频道

subcribe 具体名称1　具体名称2　具体名称3　

//订阅通配符频道

psubcribe 名称1　名称2　名称3　

//发布消息

publish 名称1　消息内容

//取消订阅

unsubscribe　名称

demo
```java
   static Jedis jedis = new Jedis("127.0.0.1",6379);
    public static void main(String[] args) {
        JedisPubSub jedisPubSub = new JedisPubSub() {
            //通配符订阅实现
//            @Override
//            public void onPMessage(String pattern, String channel, String message) {
//                System.out.println(channel+" : "+message);
//            }
            //订阅具体事件
            @Override
            public void onMessage(String channel, String message) {
                System.out.println(channel+" : "+message);
            }
        };
        //通配符订阅实现
//        jedis.psubscribe(jedisPubSub,"a.*");
        //订阅具体事件
        jedis.subscribe(jedisPubSub,"a");
    }
```

## redis　lua 用来做事务

### 命令
eval "lua 脚本"　参数个数　key key key , value value

实例　eval "return redis.call('hset',KEYS[1],KEYS[2],ARGV[2],KEYS[3],ARGV[3])" 3 "HAISHAN" "AGE" "NAME" , "18" "HAISHAN"

java demo

```java
        List<String> key = new LinkedList<>();
        key.add("POPLE");
        key.add("NAME");
        key.add("AGE");
        
        List<String> value = new LinkedList<>();
        value.add("");
        value.add("HAISHAN");
        value.add("11");
        jedis.eval("return redis.call('hset',KEYS[1],KEYS[2],ARGV[2],KEYS[3],ARGV[3])",key,value); 
```

### LUＡ 脚本语法
[lua w3c school教程](https://www.runoob.com/lua/lua-loops.html)

##＃ lua　脚本缓存

script load 设置缓存

例如：
```r
script load "return redis.call('hset',KEYS[1],KEYS[2],ARGV[2],KEYS[3],ARGV[3])"
```

java

```java
   String id =   jedis.scriptLoad("return redis.call('hset',KEYS[1],KEYS[2],ARGV[2],KEYS[3],ARGV[3])");
```
evalsha id 参数执行缓存脚本

```r
evalsha 17bc634f73aa7be6451cf553540fa2523bb29482 3 "HAISHAN" "AGE" "NAME" , "18" "HAISHAN" 
```

java
```java
  List<String> key = new LinkedList<>();
        key.add("POPLE");
        key.add("NAME");
        key.add("AGE");
        List<String> value = new LinkedList<>();
        value.add("");
        value.add("HAISHAN");
        value.add("11");
        jedis.evalsha("17bc634f73aa7be6451cf553540fa2523bb29482",key,value);
```

lua脚本设置超时时间

在redis.conf中　配置

```
lua-time-limit 5000
```

