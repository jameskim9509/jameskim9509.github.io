---
title: Redis 사용하기 ( feat. Java )
categories: [스프링, Redis]
tags: [대용량 트래픽]
description: ""
---

<span class="md-title"> Java를 통해 Redis 사용하기 </span>   
<span class="md-content"> Spring Data Redis 라이브러리가 얼마나 쉽게 사용할 수 있게 하였는지 알기 위해 Redis가 제공하는 API를 직접 사용해보았다. </span>   
<span class="md-sub"> 1. Jedis vs Lettuce </span>   
<span class="md-content"> 다음은 공식 문서의 내용이다. </span>

> You have two choices of Java clients that you can use with Redis:
>
> Jedis, for synchronous applications.
> Lettuce, for synchronous, asynchronous, and reactive applications.

<span class="md-content"> Redis를 사용하기 위한 java client는 Jedis와 Lettuce가 있다. Jedis는 동기식 API로써 사용하기 편하다는 장점이 있지만 멀티 스레드 환경에서 커넥션 풀을 통해 커넥션을 직접 관리해야한다는 불편함이 있다. 반면 Lettuce는 동기식, 비동기식, Reactive 방식을 모두 제공하는 API로써 커넥션 풀을 자동으로 관리해준다. Jedis보다 상위 호환의 API라고 생각하여 Lettuce를 사용하여 client 프로그래밍을 해보았다.   

<span class="md-sub"> 2. client 설정하기 </span>

```gradle
dependencies {
    compileOnly 'io.lettuce:lettuce-core:6.3.2.RELEASE'
}
```

<span class="md-content"> 라이브러리 추가는 간단하다. 공식 문서에서는 compileOnly를 통해 컴파일 시에만 종속성을 가지고 있지만 실행을 위해서는 implementation을 사용하는 것이 맞는 것 같다. </span>

```gradle
dependencies {
    implementation 'io.lettuce:lettuce-core:6.3.2.RELEASE'
}
```

<span class="md-sub"> 3. 동기식 비동기식으로 Redis client 사용하기 </span>   
<span class="md-content"> Redis 클라이언트를 동기식으로 사용하게 되면 Redis 서버로부터 응답이 올 때까지 대기하게 되는 반면 비동기식은 Redis 서버에 응답이 올 때까지 나머지 프로세스를 수행한다. 이는 값을 set하고 get할 경우 이전 값을 get할 수 있기 때문에 주의해서 사용해야한다. 다음은 공식 문서의 예제 프로그래밍을 수정하여 동기식과 비동기식 방법을 혼합한 코드를 작성해본 것이다. </span>

```java
RedisClient redisClient = RedisClient.create("redis://localhost:6379");

try (StatefulRedisConnection<String, String> connection = redisClient.connect()) {
    RedisAsyncCommands<String, String> asyncCommands = connection.async();

    // 비동기식 strings 값 set 
    RedisFuture<String> future = asyncCommands.set("foo", "bar");
    future.thenAccept(new Consumer<String>() {
    @Override
    public void accept(String s) {
        System.out.println(s);
    }
    });

    // 람다식을 활용한 비동기식 strings 값 get
    future = asyncCommands.get("foo");
    future.thenAccept(System.out::println);

    // Asynchronously store key-value pairs in a hash directly
    Map<String, String> hash = new HashMap<>();
    hash.put("name", "John");
    hash.put("surname", "Smith");
    hash.put("company", "Redis");
    hash.put("age", "29");
    // 비동기식 방식으로 hash 값 set ( await을 통해 Redis 서버에 정상적으로 적용되도록 기다린다. )
    future = asyncCommands.hmset("user-session:123", hash);
    future.await(1, TimeUnit.MINUTES);

    // 동기식 방식으로 hash 값 get
    System.out.println(asyncCommands.hgetall("user-session:123").get());
    // Prints: {name=John, surname=Smith, company=Redis, age=29}
} catch (ExecutionException | InterruptedException e) {
    throw new RuntimeException(e);
} finally {
    redisClient.shutdown();
}
```

<span class="md-refs"> refs:   
"백엔드 개발자를 위한 한 번에 끝내는 대용량 데이터 & 트래픽 처리 초격차 패키지"  [\< fastCampus \>](https://fastcampus.co.kr/)   
"Redis 공식 사이트" [\< Redis Docs \>](https://redis.io/docs/latest/develop/)   
"Redis Lettuce API Reference" [\< lettuce Reference \>](https://www.javadoc.io/doc/io.lettuce/lettuce-core/latest/index.html)   
"lettuce Asynchronous API Doc"  [\< lettuce Asynchronous API \>](https://redis.github.io/lettuce/user-guide/async-api/)
</span>
