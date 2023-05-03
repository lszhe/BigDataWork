# 本地缓存HashMap、Guava Cache、Caffeine、Encache

## 1. HashMap

LinkedHashMap维持了一个链表结构，用来存储节点的插入顺序或者访问顺序（二选一），并且内部封装了一些业务逻辑，只需要覆盖removeEldestEntry方法，便可以实现缓存的LRU淘汰策略。此外我们利用读写锁，保障缓存的并发安全性。需要注意的是，这个示例并不支持过期时间淘汰的策略。

自实现缓存的方式，优点是实现简单，不需要引入第三方包，比较适合一些简单的业务场景。缺点是如果需要更多的特性，需要定制化开发，成本会比较高，并且稳定性和可靠性也难以保障。对于比较复杂的场景，建议使用比较稳定的开源工具。



## 2. 基于Guava Cache实现本地缓存

Guava是Google团队开源的一款 Java 核心增强库，包含集合、并发原语、缓存、IO、反射等工具箱，性能和稳定性上都有保障，应用十分广泛。Guava Cache支持很多特性：

- 支持最大容量限制
- 支持两种过期删除策略（插入时间和访问时间）
- 支持简单的统计功能
- 基于LRU算法实现

1. 需要引入maven包：

   ```xml
   <dependency>
       <groupId>com.google.guava</groupId>
       <artifactId>guava</artifactId>
       <version>18.0</version>
   </dependency>
   ```

   

```java
public class GuavaCacheTest {

    public static void main(String[] args) throws Exception {
        //创建guava cache
        Cache<String, String> loadingCache = CacheBuilder.newBuilder()
                //cache的初始容量
                .initialCapacity(5)
                //cache最大缓存数
                .maximumSize(10)
                //设置写缓存后n秒钟过期
                .expireAfterWrite(17, TimeUnit.SECONDS)
                //设置读写缓存后n秒钟过期,实际很少用到,类似于expireAfterWrite
                //.expireAfterAccess(17, TimeUnit.SECONDS)
                .build();
        String key = "key";
        // 往缓存写数据
        loadingCache.put(key, "v");

        // 获取value的值，如果key不存在，调用collable方法获取value值加载到key中再返回
        String value = loadingCache.get(key, new Callable<String>() {
            @Override
            public String call() throws Exception {
                return getValueFromDB(key);
            }
        });

        // 删除key
        loadingCache.invalidate(key);
    }

    private static String getValueFromDB(String key) {
        return "v";
    }
}

```



## 3. Caffeine

Caffeine是基于java8实现的新一代缓存工具，缓存性能接近理论最优。可以看作是Guava Cache的增强版，功能上两者类似，不同的是Caffeine采用了一种结合LRU、LFU优点的算法：W-TinyLFU，在性能上有明显的优越性。Caffeine的使用，首先需要引入maven包：

```xml
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
    <version>2.5.5</version>
</dependency>

```

```java
public class CaffeineCacheTest {

    public static void main(String[] args) throws Exception {
        //创建guava cache
        Cache<String, String> loadingCache = Caffeine.newBuilder()
                //cache的初始容量
                .initialCapacity(5)
                //cache最大缓存数
                .maximumSize(10)
                //设置写缓存后n秒钟过期
                .expireAfterWrite(17, TimeUnit.SECONDS)
                //设置读写缓存后n秒钟过期,实际很少用到,类似于expireAfterWrite
                //.expireAfterAccess(17, TimeUnit.SECONDS)
                .build();
        String key = "key";
        // 往缓存写数据
        loadingCache.put(key, "v");

        // 获取value的值，如果key不存在，获取value后再返回
        String value = loadingCache.get(key, CaffeineCacheTest::getValueFromDB);

        // 删除key
        loadingCache.invalidate(key);
    }

    private static String getValueFromDB(String key) {
        return "v";
    }
}

```

#### 4. Encache

Encache是一个纯Java的进程内缓存框架，具有快速、精干等特点，是Hibernate中默认的CacheProvider。同Caffeine和Guava Cache相比，Encache的功能更加丰富，扩展性更强：

- 支持多种缓存淘汰算法，包括LRU、LFU和FIFO
- 缓存支持堆内存储、堆外存储、磁盘存储（支持持久化）三种
- 支持多种集群方案，解决数据共享问题

Encache的使用，首先需要导入maven包：

```xml
<dependency>
    <groupId>org.ehcache</groupId>
    <artifactId>ehcache</artifactId>
    <version>3.8.0</version>
</dependency>
```

```tsx
public class EncacheTest {

    public static void main(String[] args) throws Exception {
        // 声明一个cacheBuilder
        CacheManager cacheManager = CacheManagerBuilder.newCacheManagerBuilder()
                .withCache("encacheInstance", CacheConfigurationBuilder
                        //声明一个容量为20的堆内缓存
                        .newCacheConfigurationBuilder(String.class,String.class, ResourcePoolsBuilder.heap(20)))
                .build(true);
        // 获取Cache实例
        Cache<String,String> myCache =  cacheManager.getCache("encacheInstance", String.class, String.class);
        // 写缓存
        myCache.put("key","v");
        // 读缓存
        String value = myCache.get("key");
        // 移除换粗
        cacheManager.removeCache("myCache");
        cacheManager.close();
    }
}
```

## 总结

- 从易用性角度，Guava Cache、Caffeine和Encache都有十分成熟的接入方案，使用简单。
- 从功能性角度，Guava Cache和Caffeine功能类似，都是只支持堆内缓存，Encache相比功能更为丰富
- 从性能上进行比较，Caffeine最优、GuavaCache次之，Encache最差(下图是三者的性能对比结果）



作者：摩V羯座
链接：https://www.jianshu.com/p/0fd4468055a9
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

