# 整合：spring data redis：lettuce



SpringBoot2.0以前移植使用jedis作为客户端，2.0之后改采用Lettuce

主要还是为了弥补jedis的不足，jedis是直接连接Redis Server，除非使用线程池强行增加物理连接，否则在多线程下是不安全的

Lettuce专为多线程环境下设计，且线程安全，同时支持按需增加连接

## 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
```



## 配置文件

```yml
spring:
  redis:
    host: 127.0.0.1
    port: 6379
    password:
    timeout: 3600ms #超时时间
    lettuce:
      pool:
        max-active: 8 #最大连接数
        max-idle: 8 #最大空闲连接 默认8
        max-wait: -1ms #默认-1 最大连接阻塞等待时间
        min-idle: 0 #最小空闲连接
```



## 配置类

```java
// Redis缓存配置类
@Configuration
@EnableCaching
@AutoConfigureAfter(RedisAutoConfiguration.class)
public class RedisConfig extends CachingConfigurerSupport {
    public RedisConfig() {
        System.out.println("RedisConfig容器启动初始化。。。");
    }
    @Resource
    private LettuceConnectionFactory lettuceConnectionFactory;

    @Bean
    public KeyGenerator keyGenerator() {
        return new KeyGenerator() {
            @Override
            public Object generate(Object target, Method method, Object... params) {
                StringBuffer sb = new StringBuffer();
                sb.append(target.getClass().getName());
                sb.append(method.getName());
                for (Object obj : params) {
                    sb.append(obj.toString());
                }
                return sb.toString();
            }
        };
    }

    // 缓存管理器
    @Bean
    public CacheManager cacheManager() {
        RedisCacheManager.RedisCacheManagerBuilder builder = RedisCacheManager.RedisCacheManagerBuilder.fromConnectionFactory(lettuceConnectionFactory);
        @SuppressWarnings("serial")
        Set<String> cacheNames = new HashSet<String>() {
            {
                add("codeNameCache");
            }
        };
        builder.initialCacheNames(cacheNames);
        return builder.build();
    }
    
    @Bean
    public RedisTemplate<String,Serializable> redisCacheTemplate(LettuceConnectionFactory redisConnectionFactory){
        RedisTemplate<String,Serializable> template = new RedisTemplate<String,Serializable>();
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
		template.setHashKeySerializer(new StringRedisSerializer());// Hash key序列化
    	template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());// Hash value序列化
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }
}
```

## RedisTemplate和StringRedisTemplate

RedisTemplate包含序列化器；将键值序列化为二进制数组后存储在redis，在redis客户端直接用键取不出来值

StringRedisTemplate不包含序列化器，可以在redis客户端用键取值



## 使用

```java
//操作redis的String
redisCacheTemplate.opsForValue().set("qwe", user);//user为对象
User map2 = (User) redisCacheTemplate.opsForValue().get("qwe");


//操作redis的set；整段的逻辑是，在redis中取值未命中，去数据库中取值，取到后再放入redis的set中
Set<Medicine> medicineSet = redisTemplate.opsForSet().members("medicine");//set中取键“medicine”的所有值
if(medicineSet.isEmpty()){
    List<Medicine> medicineList = medicineService.selectAll();
    //将数据库中取出的List<Medicine>放入redis的set中
    for(Medicine medicine:medicineList){
        redisTemplate.opsForSet().add("medicine",medicine);
        //medicineSet.add(medicine);
    }
}
```