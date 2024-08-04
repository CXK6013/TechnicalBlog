# 为什么ConcurrentHashMap的key和value不能为null



## 前言

上周跟朋友讨论为什么ConcurrentHashMap不能存null，我当时的回答是因为null具备二义性，所谓二义性也就是指不确定这个元素是否在集合里面，还是确实为null，HashMap允许存储value为null的原因在于，我们可以通过Map的containsKey的情况下可以判断这个key在不在，像下面这样:

```java
Map<String,String> map = new HashMap<>();
if (map.containsKey("xxx")){
    String value = map.get("xxx");
    // 接着处理后续的逻辑
}
```

但是如果在并发情况下，可能有这样一种情况，A线程在containsKey条件满足之后，还没拿到对应的value，B线程将这个key被移除，这个value得到为空，现在我们就无法确认这样一种情况这个究竟存储的是null值，还是这个key被移除了。但看起来好像也没什么问题，我们再加个判空? 但是线程安全的问题往往出现在判断然后执行某些逻辑，原因在于可能当你判断的时候条件是满足的，当你开始执行判断里面的逻辑的时候，其他线程将条件变为不满足，这个时候就出现了意料之外的事情。

那value不能为空好像说的通了，现在让我们来看key为什么不能为空，ConcurrentHashMap也不允许key是null:

```java
Map<String,String>  concurrentHashMap = new ConcurrentHashMap<>();
concurrentHashMap.put(null,"hello");
```

然后会抛出空指针异常，在Hashtable倒是没有一开始就校验，但是你如果将key为null的键值对放入，在计算哈希值的时候同样会抛出空指针异常。而HashMap在计算哈希值的时候则进行了判断:

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

但这套逻辑不能移动到并发集合里面就会有点问题原因在于，存在这样的情况



