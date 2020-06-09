---
layout: post
title: spring boot整合redis
date: 2019-07-18
Author: sssunday
categories: 
tags: [java, redis, 入门]
comments: true
---
### spring boot整合redis
#### maven依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
    <version>2.6.2</version>
</dependency>
```
+ spring-boot-starter-data-redis 是redis依赖
+ commons-pool2是为了支撑lettuce连接池

#### redis yml配置
```yml
spring:
  redis:
    host: localhost
    port: 6380
    database: 1
    lettuce: #lettuce连接池配置
      pool:
        min-idle: 10 
        max-idle: 10
        time-between-eviction-runs: 60000
        max-active: 10
```

#### RedisTemplate配置
```java
@EnableCaching
@Configuration
public class RedisTemplateConfig {

    @PostConstruct
    void init() {
        System.setProperty("es.set.netty.runtime.available.processors", "false");
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();

        //配置连接工厂
        template.setConnectionFactory(factory);

        //使用Jackson2JsonRedisSerializer来序列化和反序列化redis的value值（默认使用JDK的序列化方式）
        Jackson2JsonRedisSerializer jacksonSeial = new Jackson2JsonRedisSerializer(Object.class);
        ObjectMapper om = new ObjectMapper();
        // 指定要序列化的域，field,get和set,以及修饰符范围，ANY是都有包括private和public
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.activateDefaultTyping(LaissezFaireSubTypeValidator.instance ,
                ObjectMapper.DefaultTyping.NON_FINAL, JsonTypeInfo.As.PROPERTY);
        jacksonSeial.setObjectMapper(om);

        // 值采用json序列化
        template.setValueSerializer(jacksonSeial);
        //使用StringRedisSerializer来序列化和反序列化redis的key值
        template.setKeySerializer(new StringRedisSerializer());

        // 设置hash key 和value序列化模式
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(jacksonSeial);
        template.afterPropertiesSet();

        return template;
    }
}
```

#### redisUtil 工具类整理，方便redis api操作
```java
@Slf4j
@Component
public final class RedisUtil {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    /**
     * 指定缓存失效时间
     */
    public boolean expire(String key, long time) {
        try {
            if (time > 0) {
                redisTemplate.expire(key, time, TimeUnit.SECONDS);
            }

            return true;
        } catch (Exception e){
            log.error("Redis设置缓存时间失败，{}.{}", e.getMessage(), e.getStackTrace());

            return false;
        }
    }

    /**
     * 根据key 获取过期时间
     * @param key 键 不能为null
     * @return 时间(秒) 返回0代表为永久有效
     */
    public long getExpire(String key){
        Long expireTime = redisTemplate.getExpire(key,TimeUnit.SECONDS);
        return expireTime != null ? expireTime : 0;
    }

    /**
     * 判断key是否存在
     */
    public boolean hasKey(String key) {
        try {
            Boolean hasKey = redisTemplate.hasKey(key);
            return hasKey != null ? hasKey : false;
        } catch (Exception e){
            log.error("Redis检查key是否存在失败，{}.{}", e.getMessage(), e.getStackTrace());

            return false;
        }
    }

    /**
     * 删除缓存
     */
    public void del(String... key) {
        if (null != key && key.length > 0) {
            if (key.length == 1) {

                redisTemplate.delete(key[0]);
            } else {
                redisTemplate.delete(Arrays.asList(key));
            }
        }
    }

    //============================String操作=============================
    /**
     * 普通缓存获取
     * @param key 键
     * @return 值
     */
    public Object get(String key){
        return key == null ? null : redisTemplate.opsForValue().get(key);
    }

    /**
     * 普通缓存放入
     * @param key 键
     * @param value 值
     * @return true成功 false失败
     */
    public boolean set(String key, Object value) {
        try {
            redisTemplate.opsForValue().set(key, value);
            return true;

        } catch (Exception e) {
            log.error("Redis普通缓存放入失败，{}.{}", e.getMessage(), e.getStackTrace());

            return false;
        }
    }


    /**
     * setnx
     * @param key 键
     * @param value 值
     * @return true成功 false失败
     */
    public boolean setIfAbsent(String key, Object value,long l,TimeUnit unit) {
        return redisTemplate.opsForValue().setIfAbsent(key, value,l,unit);
    }

    /**
     * 普通缓存放入并设置时间
     * @param key 键
     * @param value 值
     * @param time 时间(秒) time要大于0 如果time小于等于0 将设置无限期
     * @return true成功 false 失败
     */
    public boolean set(String key, Object value, long time){
        try {
            if(time > 0){
                redisTemplate.opsForValue().set(key, value, time, TimeUnit.SECONDS);
            }else{
                set(key, value);
            }

            return true;
        } catch (Exception e) {
            log.error("Redis普通缓存放入并设置时间失败，{}.{}", e.getMessage(), e.getStackTrace());

            return false;
        }
    }

    /**
     * 递增
     * @param key 键
     * @param delta 要增加几(大于0)
     * @return long
     */
    public long incr(String key, long delta){
        if(delta < 0){
            throw new RuntimeException("递增因子必须大于0");
        }

        Long incr = redisTemplate.opsForValue().increment(key, delta);
        if (null == incr) {
            throw new RuntimeException("递增因子必须大于0");
        }

        return incr;
    }

    /**
     * 递减
     */
    public long decr(String key, long delta){
        if(delta < 0){
            throw new RuntimeException("递减因子必须大于0");
        }

        Long decr = redisTemplate.opsForValue().increment(key, -delta);
        if (null == decr) {
            throw new RuntimeException("递减因子必须大于0");
        }

        return decr;
    }

    //================================Map=================================
    /**
     * HashGet
     */
    public Object hGet(String key, String item){

        return redisTemplate.opsForHash().get(key, item);
    }

    /**
     * 获取hashKey对应的所有键值
     */
    public Map<Object,Object> hmGet(String key){

        return redisTemplate.opsForHash().entries(key);
    }

    /**
     * HashSet
     */
    public boolean hmSet(String key, Map<String, Object> map){
        try {
            redisTemplate.opsForHash().putAll(key, map);

            return true;
        } catch (Exception e) {
            log.error("Redis执行HashSet失败，{}.{}", e.getMessage(), e.getStackTrace());

            return false;
        }
    }

    /**
     * HashSet 并设置时间
     */
    public boolean hmSet(String key, Map<String, Object> map, long time){
        try {
            redisTemplate.opsForHash().putAll(key, map);
            if(time > 0){
                expire(key, time);
            }

            return true;
        } catch (Exception e) {
            log.error("Redis执行HashSet并设置时间失败，{}.{}", e.getMessage(), e.getStackTrace());

            return false;
        }
    }

    /**
     * 向一张hash表中放入数据,如果不存在将创建
     */
    public boolean hSet(String key,String item,Object value) {
        try {
            redisTemplate.opsForHash().put(key, item, value);

            return true;
        } catch (Exception e) {
            log.error("Redis执行HashSet失败，{}.{}", e.getMessage(), e.getStackTrace());

            return false;
        }
    }

    /**
     * 向一张hash表中放入数据,如果不存在将创建
     */
    public boolean hSet(String key, String item, Object value, long time) {
        try {
            redisTemplate.opsForHash().put(key, item, value);
            if(time > 0){
                expire(key, time);
            }

            return true;
        } catch (Exception e) {
            log.error("Redis执行HashSet失败，{}.{}", e.getMessage(), e.getStackTrace());

            return false;
        }
    }

    /**
     * 删除hash表中的值
     */
    public void hDel(String key, Object... item){

        redisTemplate.opsForHash().delete(key,item);
    }

    /**
     * 判断hash表中是否有该项的值
     */
    public boolean hHasKey(String key, String item){

        return redisTemplate.opsForHash().hasKey(key, item);
    }

    /**
     * hash递增 如果不存在,就会创建一个 并把新增后的值返回
     */
    public double hIncr(String key, String item, double by){

        return redisTemplate.opsForHash().increment(key, item, by);
    }

    /**
     * hash递减
     */
    public double hDecr(String key, String item, double by){

        return redisTemplate.opsForHash().increment(key, item, -by);
    }

    //============================set=============================
    /**
     * 根据key获取Set中的所有值
     */
    public Set<Object> sGet(String key){
        try {

            return redisTemplate.opsForSet().members(key);
        } catch (Exception e) {
            log.error("Redis根据key获取Set中的所有值失败，{}.{}", e.getMessage(), e.getStackTrace());

            return null;
        }
    }

    /**
     * 根据value从一个set中查询,是否存在
     */
    public boolean sHasKey(String key, Object value){
        try {
            Boolean sHasKey = redisTemplate.opsForSet().isMember(key, value);

            return sHasKey != null ? sHasKey : false;
        } catch (Exception e) {
            log.error("Redis根据value从一个set中查询,是否存在失败，{}.{}", e.getMessage(), e.getStackTrace());

            return false;
        }
    }

    /**
     * 将数据放入set缓存
     */
    public long sSet(String key, Object...values) {
        try {
            Long sSet = redisTemplate.opsForSet().add(key, values);

            return sSet != null ? sSet : 0;
        } catch (Exception e) {
            log.error("Redis将数据放入set缓存失败，{}.{}", e.getMessage(), e.getStackTrace());

            return 0;
        }
    }

    /**
     * 将set数据放入缓存
     */
    public long sSetAndTime(String key, long time, Object...values) {
        try {
            Long count = redisTemplate.opsForSet().add(key, values);
            if(time > 0) {
                expire(key, time);
            }
            return count != null ? count : 0;
        } catch (Exception e) {
            log.error("Redis将set数据放入缓存失败，{}.{}", e.getMessage(), e.getStackTrace());

            return 0;
        }
    }

    /**
     * 获取set缓存的长度
     */
    public long sGetSetSize(String key){
        try {
            Long sGetSetSize = redisTemplate.opsForSet().size(key);

            return sGetSetSize != null ? sGetSetSize : 0;
        } catch (Exception e) {
            log.error("Redis获取set缓存的长度失败，{}.{}", e.getMessage(), e.getStackTrace());

            return 0;
        }
    }

    /**
     * 移除值为value
     */
    public long setRemove(String key, Object ...values) {
        try {
            Long count = redisTemplate.opsForSet().remove(key, values);

            return count != null ? count : 0;
        } catch (Exception e) {
            log.error("Redis移除值为value失败，{}.{}", e.getMessage(), e.getStackTrace());

            return 0;
        }
    }
    //===============================list=================================
    /**
     * 获取list缓存的内容
     */
    public List<Object> lGet(String key, long start, long end){
        try {

            return redisTemplate.opsForList().range(key, start, end);
        } catch (Exception e) {
            log.error("Redis获取list缓存的内容失败，{}.{}", e.getMessage(), e.getStackTrace());

            return null;
        }
    }

    /**
     * 获取list缓存的全部内容
     */
    public List<Object> lGetAll(String key){
        try {

            return redisTemplate.opsForList().range(key, 0, -1);
        } catch (Exception e) {
            log.error("Redis获取list缓存的内容失败，{}.{}", e.getMessage(), e.getStackTrace());

            return null;
        }
    }

    /**
     * 获取list缓存的长度
     */
    public long lGetListSize(String key){
        try {
            Long lGetListSize = redisTemplate.opsForList().size(key);

            return lGetListSize != null ? lGetListSize : 0;
        } catch (Exception e) {
            log.error("Redis获取list缓存的长度失败，{}.{}", e.getMessage(), e.getStackTrace());

            return 0;
        }
    }

    /**
     * 通过索引 获取list中的值
     */
    public Object lGetIndex(String key, long index){
        try {

            return redisTemplate.opsForList().index(key, index);
        } catch (Exception e) {
            log.error("Redis通过索引获取list中的值失败，{}.{}", e.getMessage(), e.getStackTrace());

            return null;
        }
    }

    /**
     * 将list放入缓存
     */
    public boolean rightPush(String key, Object value) {
        try {
            redisTemplate.opsForList().rightPush(key, value);

            return true;
        } catch (Exception e) {
            log.error("Redis将list放入缓存失败，{}.{}", e.getMessage(), e.getStackTrace());

            return false;
        }
    }

    /**
     * 将list放入缓存
     */
    public boolean rightPush(String key, Object value, long time) {
        try {
            redisTemplate.opsForList().rightPush(key, value);
            if (time > 0) {
                expire(key, time);
            }

            return true;
        } catch (Exception e) {
            log.error("Redis将list放入缓存失败，{}.{}", e.getMessage(), e.getStackTrace());

            return false;
        }
    }

    /**
     * 将list放入缓存
     */
    public boolean rightPushAll(String key, List<Object> value) {
        try {
            redisTemplate.opsForList().rightPushAll(key, value);

            return true;
        } catch (Exception e) {
            log.error("Redis将list放入缓存失败，{}.{}", e.getMessage(), e.getStackTrace());

            return false;
        }
    }

    /**
     * 将list放入缓存
     */
    public boolean rightPushAll(String key, List<Object> value, long time) {
        try {
            redisTemplate.opsForList().rightPushAll(key, value);
            if (time > 0) {
                expire(key, time);
            }

            return true;
        } catch (Exception e) {
            log.error("Redis将list放入缓存失败，{}.{}", e.getMessage(), e.getStackTrace());

            return false;
        }
    }

    /**
     * 根据索引修改list中的某条数据
     */
    public boolean lUpdateIndex(String key, long index, Object value) {
        try {
            redisTemplate.opsForList().set(key, index, value);

            return true;
        } catch (Exception e) {
            log.error("Redis根据索引修改list中的某条数据失败，{}.{}", e.getMessage(), e.getStackTrace());

            return false;
        }
    }

    /**
     * 移除N个值
     */
    public long lRemove(String key, long count, Object value) {
        try {
            Long remove = redisTemplate.opsForList().remove(key, count, value);
            return remove != null ? remove : 0;
        } catch (Exception e) {
            log.error("Redis根据索引修改list中的某条数据失败，{}.{}", e.getMessage(), e.getStackTrace());

            return 0;
        }
    }

}

```

### test单元测试
```java

@SpringBootTest
class TvshopJobApplicationTests {

    @Autowired
    RedisUtil redisUtil;
    
    void redisTest(){

        Object value = redisUtil.get("test-key");
        System.out.println("value1 ===>" + value);
        redisUtil.set("test-key", "test-value1");
        Object value2 = redisUtil.get("test-key");
        System.out.println("value1 ===>" + value2);

        redisUtil.del("test-key");
        Object value3 = redisUtil.get("test-key");
        System.out.println("value1 ===>" + value3);
    }

}

```