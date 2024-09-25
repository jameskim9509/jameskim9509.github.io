---
title: Redis 사용하기 ( feat. Spring )
categories: [스프링, Redis]
tags: [대용량 트래픽]
description: ""
---

<span class="md-title"> Spring을 통해 Redis 사용하기 </span>   
<span class="md-content"> 문서를 참고하여 Spring Data Redis 라이브러리를 사용해보았다. </span>   
<span class="md-sub"> 1. Connection Factory 설정 </span>   

> RedisConnection provides the core building block for Redis communication, as it handles the communication with the Redis backend.
> Active RedisConnection objects are created through RedisConnectionFactory.
> RedisConnection classes are not Thread-safe. While the underlying native connection, such as Lettuce’s StatefulRedisConnection, may be Thread-safe, Spring Data Redis’s LettuceConnection class itself is not. Therefore, you should not share instances of a RedisConnection across multiple Threads.
> LettuceConnectionFactory share the same thread-safe native connection for all non-blocking and non-transactional operations.

<span class="md-content"> Redis의 lettuce API를 사용하기 위해서는 RedisConnection(Lettuce Connection) 또는 Connection을 생성해주는 RedisConnectionFactory를 등록해주어야한다. RedisConnection은 thread-safe하지 않으므로 ConnectionFactory를 생성하여 내부적으로 관리할 수 있게 하는 것이 바람직해 보였다.

```java
@Bean
LettuceConnectionFactory connectionFactory() {
    return new LettuceConnectionFactory(
            new RedisStandaloneConfiguration(host, port)
    );
}
```

<span class="md-content"> 문서를 보면 ConnectionFactory를 등록하는 과정에서 RedisStandAlone, Sentinel, Cluster 등 댜양한 설정들을 적용할 수 있다는 것을 알 수 있다.</span>   

<span class="md-sub"> 2. RedisTemplate 설정 </span>

> The template offers a high-level abstraction for Redis interactions. While [Reactive]RedisConnection offers low-level methods that accept and return binary values (byte arrays), the template takes care of serialization and connection management, freeing the user from dealing with such details.
> RedisTemplate uses a Java-based serializer for most of its operations.

<span class="md-content"> RedisConnection을 통해 Redis에 접근할 수 있는 low-level 함수를 제공하지만 RedisConnection에 직접 접근하는 것은 위험할 수 있기 때문에 RedisTemplate을 활용한다. RedisTemplate은 serializer를 활용하여 여러 자료구조의 데이터를 쉽게 직렬화하여 Redis에 저장하는 것을 가능하게 한다. </span>

```java
@Bean
RedisTemplate<String, String> redisTemplate(RedisConnectionFactory connectionFactory)
{
    RedisTemplate<String, String> redisTemplate = new RedisTemplate<>();
    redisTemplate.setConnectionFactory(connectionFactory);
    return redisTemplate;
}
```

<span class="md-content"> String 기반의 key와 value를 serialize할 수 있는 serializer를 정의하였다. </span>

<span class="md-sub"> 3. Redis 연산 수행 </span>   
<span class="md-content"> String data를 set하고 get할 수 있는 RedisService를 정의하였다. </span>

```java
RedisService(@Autowired RedisTemplate<String, String> redisTemplate)
{
    this.redisTemplate = redisTemplate;
    this.valueOperations = redisTemplate.opsForValue();
}

public String set(String key, String value)
{
    valueOperations.set(key, value);
    return "ok";
}

public String get(String key)
{
    return valueOperations.get(key);
}
```

<span class="md-content"> opsForValue함수를 통해 Redis Strings 자료구조에 접근할 수 있는 연산을 획득하였다. RestAPI를 활용하여 웹서비스를 구현한 결과는 다음과 같다. </span>

<div class="img-box">
<img src="/assets/img/Redis/Redis 사용하기 ( feat. Spring )/PostKeyValue.png" alt="Redis">
<img src="/assets/img/Redis/Redis 사용하기 ( feat. Spring )/GetKeyValue.png" alt="Redis">
</div>   
<div class="img-box">
<img src="/assets/img/Redis/Redis 사용하기 ( feat. Spring )/RedisKeyCheck.png" width="80%" height="80%" alt="Redis">
</div>

<span class="md-sub"> 4. Redis Cache를 통한 세션 저장소 생성 </span>

> Spring Data Redis provides an implementation of Spring Framework’s Cache Abstraction in the org.springframework.data.redis.cache package. To use Redis as a backing implementation, add RedisCacheManager to your configuration
> The behavior of RedisCache created by RedisCacheManager is defined with RedisCacheConfiguration. The configuration lets you set key expiration times, prefixes, and RedisSerializer implementations for converting to and from the binary storage format

<span class="md-content"> Redis를 Spring Cache와 연동해서 사용할 수 있다는 뜻이다. sessionId를 저장하고 sessionId에 대한 유저의 정보를 저장하는 세션 저장소가 Cache와 연동하여 사용하기 좋다고 생각했다. Redis와 연동된 Cache를 생성하기 위해서는 RedisCacheManager 설정이 필요하고 Cache 데이터 만료시간 및 캐시에 저장하고 꺼내기 위한 Serializer 등 Cache 설정을 위해서는 RedisCacheConfiguration이 필요하다고 한다. </span>

```java
@Bean
RedisCacheManager redisCacheManager(RedisConnectionFactory redisConnectionFactory)
{
    RedisCacheConfiguration defaluts = RedisCacheConfiguration.defaultCacheConfig()
    //                .entryTtl(Duration.ofMinutes(30))
            .entryTtl(Duration.ofSeconds(30))
            .prefixCacheNameWith("")
            .enableTimeToIdle();

    return RedisCacheManager.builder(redisConnectionFactory)
            .cacheDefaults(defaluts)
        .build();
}
```

<span class="md-content"> Cache 저장 시간을 테스트 용으로 30초로 설정하였고, enableTimeToIdle() 설정을 활용하여 접근 시마다 시간이 갱신되도록 설정하였다. </span>

```java
@CachePut(cacheNames = "sessionStore", key="#sessionId")
public String login(String username, String sessionId)
{
    return username;
}

@Cacheable(cacheNames = "sessionStore", key = "#sessionId", sync=true)
public String getInfo(String sessionId)
{
    throw new IllegalArgumentException("접근이 제한되었습니다.");
}
```
<span class="md-content"> 위의 코드는 Spring Caching에 대한 내용을 바탕으로 만든 Cache기반의 세션저장소이다. HttpServlet을 통해 세션 ID를 가진 세션 쿠키를 생성하고, 세션 ID를 기반으로 username을 반환하도록 구현하였다. 이때 세션 저장소에 저장되지 않은 세션 ID를 전달할 경우 오류를 발생시키도록 제한하였다. 결과는 다음과 같다. </span>

<div class="img-box">
<img src="/assets/img/Redis/Redis 사용하기 ( feat. Spring )/PostCache.png" alt="Redis">
<img src="/assets/img/Redis/Redis 사용하기 ( feat. Spring )/GetCache.png" alt="Redis">
</div>   
<div class="img-box">
<img src="/assets/img/Redis/Redis 사용하기 ( feat. Spring )/RedisCacheCheck.png" alt="Redis">
</div>

<span class="md-content"> 일정 시간이 지난 후 </span>   
<div class="img-box">
<img src="/assets/img/Redis/Redis 사용하기 ( feat. Spring )/GetCacheAfter.png" width="70%" height="70%" alt="Redis">
</div>
<div class="img-box">
<img src="/assets/img/Redis/Redis 사용하기 ( feat. Spring )/RedisCacheCheckAfter.png" alt="Redis">
</div>

<span class="md-refs"> refs:   
"백엔드 개발자를 위한 한 번에 끝내는 대용량 데이터 & 트래픽 처리 초격차 패키지"  [\< fastCampus \>](https://fastcampus.co.kr/)   
"Spring Data Redis" [\< Spring Data Redis Reference \>](https://docs.spring.io/spring-data/redis/reference/index.html)   
"Spring Cache" [\< Spring Cache Abstraction Reference \>](https://docs.spring.io/spring-framework/reference/integration/cache.html)   
"Spring API" [\< Spring framework API Reference \>](https://docs.spring.io/spring-framework/docs/current/javadoc-api/)
</span>
