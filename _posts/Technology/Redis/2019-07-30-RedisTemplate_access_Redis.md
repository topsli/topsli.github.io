---

layout: post
title: 如何使用RedisTemplate访问Redis数据结构
category: 技术
tags: Apollo
keywords: Redis，RedisTemplate

---

## Redis 数据结构简介
https://www.jianshu.com/p/7bf5dc61ca06

Redis 可以存储键与5种不同数据结构类型之间的映射，这5种数据结构类型分别为String（字符串）、List（列表）、
Set（集合）、Hash（散列）和 Zset（有序集合）。

下面来对这5种数据结构类型作简单的介绍：

![](/public/upload/redis/dataType.png)

Redis 5种数据结构的概念大致介绍到这边，下面将结合Spring封装的RedisTemplate来对这5种数据结构的运用进行演示

## RedisTemplate介绍

spring 封装了 RedisTemplate 对象来进行对redis的各种操作，它支持所有的 redis 原生的 api。

RedisTemplate在spring代码中的结构如下：

    org.springframework.data.redis.core
    Class RedisTemplate<K,V>
    java.lang.Object
        org.springframework.data.redis.core.RedisAccessor
            org.springframework.data.redis.core.RedisTemplate<K,V>


Type Parameters:

K: the Redis key type against which the template works (usually a String)
   模板中的Redis key的类型（通常为String）如：RedisTemplate<String, Object>
   注意：如果没特殊情况，切勿定义成RedisTemplate<Object, Object>，否则根据里氏替换原则，使用的时候会造成类型错误 。
V: the Redis value type against which the template works
   模板中的Redis value的类型
   
### RedisTemplate中定义了对5种数据结构操作

    redisTemplate.opsForValue();//操作字符串
    redisTemplate.opsForHash();//操作hash
    redisTemplate.opsForList();//操作list
    redisTemplate.opsForSet();//操作set
    redisTemplate.opsForZSet();//操作有序set
    
### StringRedisTemplate与RedisTemplate

* 两者的关系是StringRedisTemplate继承RedisTemplate。

* 两者的数据是不共通的；也就是说StringRedisTemplate只能管理StringRedisTemplate里面的数据，RedisTemplate只能管理
RedisTemplate中的数据。

* SDR默认采用的序列化策略有两种，一种是String的序列化策略，一种是JDK的序列化策略。

 StringRedisTemplate默认采用的是String的序列化策略，保存的key和value都是采用此策略序列化保存的。

 RedisTemplate默认采用的是JDK的序列化策略，保存的key和value都是采用此策略序列化保存的。

### RedisTemplate配置

    @Bean
        public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory)
        {
            Jackson2JsonRedisSerializer<Object> jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer<Object>(Object.class);
            ObjectMapper om = new ObjectMapper();
            om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
            om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
            jackson2JsonRedisSerializer.setObjectMapper(om);
            RedisTemplate<String, Object> template = new RedisTemplate<String, Object>();
            template.setConnectionFactory(redisConnectionFactory);
            template.setKeySerializer(jackson2JsonRedisSerializer);
            template.setValueSerializer(jackson2JsonRedisSerializer);
            template.setHashKeySerializer(jackson2JsonRedisSerializer);
            template.setHashValueSerializer(jackson2JsonRedisSerializer);
            template.afterPropertiesSet();
            return template;
        }
        
### Redis的String数据结构 （推荐使用StringRedisTemplate）

注意：如果使用RedisTemplate需要更改序列化方式
    RedisSerializer<String> stringSerializer = new StringRedisSerializer();
            template.setKeySerializer(stringSerializer );
            template.setValueSerializer(stringSerializer );
            template.setHashKeySerializer(stringSerializer );
            template.setHashValueSerializer(stringSerializer );

    public interface ValueOperations<K,V>
    Redis operations for simple (or in Redis terminology 'string') values.
    ValueOperations可以对String数据结构进行操作：
 
* set void set(K key, V value);
    使用：redisTemplate.opsForValue().set("name","tom");
    结果：redisTemplate.opsForValue().get("name")  输出结果为tom

* set void set(K key, V value, long timeout, TimeUnit unit);
    




