---
title: Redis 사용하기 ( feat. RedisCLI )
categories: [스프링, Redis]
tags: [대용량 트래픽]
description: ""
---

<span class="md-title"> Redis 사용하기 </span>   
<span class="md-content"> 다양한 Spring 프로젝트를 진행하며 Redis를 사용해보았지만 깊게 사용해본 경험이 없다. FastCampus의 강의를 들으며 Redis 실습의 방향성을 잡고, Redis 및 Spring 공식 문서를 참고하며 문서를 보는 방법 및 깊은 실습에 대한 이해를 도모한다. </span>   

<span class="md-title"> 1. Redis 환경 구축하기 </span>   
<span class="md-sub"> 1) Redis vs Redis Stack </span>   
<span class="md-content"> Redis 공식 문서를 보면 Redis 설치와 Redis Stack 설치가 있다. </span>   
<div class="img-box">
<img src="/assets/img/Redis/Redis 사용하기/RedisInstall_1.png" width="70%" height="70%" alt="RedisInstall">
</div>   
<span class="md-content"> 공식 문서에서 Redis와 Redis Stack에 대한 내용을 찾지 못하여서 Chatgpt를 활용하였다.</span>   

> Redis는 오픈 소스 인메모리 데이터 구조 스토어로, 키-값 저장소로 주로 사용됩니다. 캐싱, 세션 관리, 실시간 분석 등 다양한 용도로 활용되며, 데이터를 메모리에 저장하기 때문에 매우 빠른 읽기/쓰기 성능을 제공합니다.
> Redis Stack은 기본 Redis의 기능에 추가로 여러 확장 기능과 모듈을 포함한 확장판입니다. Redis Stack은 Redis를 더 다재다능하게 만들어주는 모듈을 함께 제공하여 보다 복잡한 애플리케이션 요구사항을 처리할 수 있습니다.

<span class="md-content"> 확장 모듈이 추가된 Redis를 Redis Stack이라고 하는 것 같다. 다양한 사용법을 익히는 과정에서 확장 모듈을 이용하는 것이 유리할 것 같아서 확장 모듈을 설치하기로 결정했다. </span>   

<span class="md-sub"> 2) Redis 설치 환경 </span>   

<div class="img-box">
<img src="/assets/img/Redis/Redis 사용하기/RedisInstall_2.png" width="70%" height="70%" alt="RedisInstall">
</div>   

<span class="md-content"> Redis Stack을 설치할 수 있는 다양한 환경이 보인다. Windows 컴퓨터를 쓰고 있기에 Windows 위에서 설치하는 것도 하나의 방법이겠지만, Redis 서버를 여러 개 띄울 경우가 있을 것이기에 docker를 사용하기로 결정했다. </span>   

```powershell
docker run -d --name redis-stack -p 6379:6379 -p 8001:8001 redis/redis-stack:latest
```

<span class="md-content"> 공식문서에 나온 대로 redis/redis-stack:latest 이미지를 로컬로 설치하고 컨테이너를 생성하여 redis 서버와 redis insight에 접속할 수 있도록 하였다. redis-cli와 redis insight 접속 결과는 다음과 같다. </span>   

<div class="img-box">
<img src="/assets/img/Redis/Redis 사용하기/RedisConnect_1.png" width="80%" height="80%" alt="RedisConnect">
</div>   
<div class="img-box">
<img src="/assets/img/Redis/Redis 사용하기/RedisConnect_2.png" width="70%" height="70%" alt="RedisConnect">
</div>   

<span class="md-title"> 2. Redis Data Type의 이해 </span>   
<span class="md-sub"> 1) Strings </span>   

> Redis strings store sequences of bytes, including text, serialized objects, and binary arrays. As such, strings are the simplest type of value you can associate with a Redis key. They're often used for caching, but they support additional functionality that lets you implement counters and perform bitwise operations, too.
> 
> Since Redis keys are strings, when we use the string type as a value too, we are mapping a string to another string. The string data type is useful for a number of use cases, like caching HTML fragments or pages.

<span class="md-content"> Strings은 바이트 배열을 담는 Data Type으로써 텍스트, 직렬화된 객체, 바이너리 배열 등의 거의 대부분의 데이터 타입을 담을 수 있는 가장 간단한 타입이다. </span>   

<span class="md-content"> **① GET, SET** </span>   
<span class="md-content"> GET과 SET은 키에 대한 값을 저장하고 가져오기 위한 명령어이다. SET 명령어의 nx 옵션을 통해 키가 존재하지 않을 경우에만 SET하거나 XX 옵션을 통해 키가 존재할 경우에만 SET하게 할 수도 있다.
또한 MSET MGET을 통해 한번의 접근으로 많은 키를 설정하거나 가져올 수 있다. </span>   

<div class="img-box">
<img src="/assets/img/Redis/Redis 사용하기/RedisGetSet.png" width="70%" height="70%" alt="GetSet">
</div>   

<span class="md-content"> **② INCR. DECR** </span>   
<span class="md-content"> INCR, DECR 명령어는 키의 값을 1 증가 혹은 감소시키는 명령어이다. 이 명령어는 Atomic한 연산이기 때문에 멀티 클라이언트 환경에서도 일관성있는 데이터를 유지할 수 있다는 장점이 있다. INCRBY와 DECRBY 명령어를 통해 증가 혹은 감소시킬 양을 결정할 수 있다. </span>   

<div class="img-box">
<img src="/assets/img/Redis/Redis 사용하기/RedisIncrDecr.png" width="70%" height="70%" alt="RedisConnect">
</div>   

<span class="md-sub"> 2) Lists </span>   

> Redis lists are implemented via Linked Lists. This means that even if you have millions of elements inside a list, the operation of adding a new element in the head or in the tail of the list is performed in constant time. The speed of adding a new element with the LPUSH command to the head of a list with ten elements is the same as adding an element to the head of list with 10 million elements.

<span class="md-content"> Redis의 lists는 연결리스트로 구현된 자료구조를 말한다. 연결 리스트는 삽입 삭제가 빠른 반면에 조회시에 느리다. Redis List는 LPUSH, LPOP 명령 등을 사용하여 Stack, Queue를 구현하기 위해 일반적으로 사용한다. 아래는 LPUSH를 통해 존재하지 않는 키를 지정하면 자동으로 List를 만들어준다는 공식 문서를 참고하여 실험해본 결과이다.  </span>   

<div class="img-box">
<img src="/assets/img/Redis/Redis 사용하기/RedisList.png" width="70%" height="70%" alt="RedisConnect">
</div>   

<span class="md-sub"> 3) Sets </span>   

> A Redis set is an unordered collection of unique strings (members). You can use Redis sets to efficiently:
> 
> Track unique items (e.g., track all unique IP addresses accessing a given blog post).
> Represent relations (e.g., the set of all users with a given role).
> Perform common set operations such as intersection, unions, and differences.

<span class="md-content"> distint한 순서가 없는 strings 집합이라고 한다. 해당 값이 존재하는지, role 적용 및 집합 연산을 적용하기 위해 사용하는 자료구조이다. </span>   

<div class="img-box">
<img src="/assets/img/Redis/Redis 사용하기/ReidsSets.png" width="70%" height="70%" alt="RedisConnect">
</div>   

<span class="md-sub"> 4) Hashes </span>   

> Redis hashes are record types structured as collections of field-value pairs. You can use hashes to represent basic objects and to store groupings of counters, among other things.

<span class="md-content"> 필드와 값의 쌍으로 이루어진 collections 자료구조이다. Java의 해시맵이라고 생각하면 편할 것 같다. HSET을 통해 단일 해시, HMSET을 통해 다중 해시 쌍을 결정할 수 있다. </span>   

<div class="img-box">
<img src="/assets/img/Redis/Redis 사용하기/hset.png" width="70%" height="70%" alt="RedisConnect">
</div>   

<span class="md-sub"> 5) Sorted Sets </span>   

> A Redis sorted set is a collection of unique strings (members) ordered by an associated score. When more than one string has the same score, the strings are ordered lexicographically. Some use cases for sorted sets include:
>
> Leaderboards. For example, you can use sorted sets to easily maintain ordered lists of the highest scores in a massive online game.
Rate limiters. In particular, you can use a sorted set to build a sliding-window rate limiter to prevent excessive API requests.

<span class="md-sub"> Sorted Set의 각 요소는 score라는 값을 가지고 score에 의해 정렬된 Set 자료구조이다. 이는 순위 계산 및 리더보드, sliding window 방식에 기반한 ( 일정 시간동안 요청 제한 ) 요청 수 제한기 등을 쉽게 구현하도록 도와준다. </span>   

<div class="img-box">
<img src="/assets/img/Redis/Redis 사용하기/RedisSortedSets.png" width="70%" height="70%" alt="RedisConnect">
</div>   

<span class="md-sub"> 5) Bitmaps </span>   

> Bitmaps are not an actual data type, but a set of bit-oriented operations defined on the String type which is treated like a bit vector. Since strings are binary safe blobs and their maximum length is 512 MB, they are suitable to set up to 2^32 different bits.

<span class="md-content"> Bitmaps는 strings 자료구조로 정의된 bit 단위( 0, 1의 값을 가짐 )의 집합이다. 2^32의 비트 수를 지원하므로 상당히 많은 회원들의 존재 여부( 1:present, 0:not present )등을 표현할 때 사용될 수 있다. </span>   

<div class="img-box">
<img src="/assets/img/Redis/Redis 사용하기/RedisBitmaps.png" width="70%" height="70%" alt="RedisConnect">
</div>   

<span class="md-sub"> 6) HyperLogLog </span>   

> HyperLogLog is a probabilistic data structure that estimates the cardinality of a set. As a probabilistic data structure, HyperLogLog trades perfect accuracy for efficient space utilization.
>
> The Redis HyperLogLog implementation uses up to 12 KB and provides a standard error of 0.81%.

<span class="md-content"> 집합의 크기를 작은 메모리로 추정하기 위한 확률적 자료구조이다. 2^64개의 유니크 값을 99.2 퍼센트의 정확도를 가지고 최대 12KB의 작은 메모리를 사용한다는 특징이 있어 매우 큰 데이터를 다룰 때 사용할 수 있다. </span>   

<div class="img-box">
<img src="/assets/img/Redis/Redis 사용하기/RedisHyperLogLog.png" width="70%" height="70%" alt="RedisConnect">
</div>   

<span class="md-refs"> refs:   
"백엔드 개발자를 위한 한 번에 끝내는 대용량 데이터 & 트래픽 처리 초격차 패키지"  [\< fastCampus \>](https://fastcampus.co.kr/)   
"Redis 공식 사이트" [\< Redis Docs \>](https://redis.io/docs/latest/develop/) 
</span>
