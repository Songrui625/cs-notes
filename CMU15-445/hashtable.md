## Hash Scheme

#### 为什么Hash Table需要保存entry<key, value>，而不能单单只保存value？
在linear probe hash scheme中，当我们需要插入一对key&value时，我们对key进行hash，返回的offset对应的slot中
可能存在了别的key&value，也就是发生了hash collision，这时我们需要往下遍历直到找到一个可用的slot才能插入。  
而当我们需要在hash table中查找一个key对应的value时，我们对key进行hash，返回的offset对应的slot不一定
是我们key对应的value，所以我们需要往下遍历直至匹配key，才能找到key对应的value。

#### Hash Table 删除操作会产生什么问题？如何解决？
