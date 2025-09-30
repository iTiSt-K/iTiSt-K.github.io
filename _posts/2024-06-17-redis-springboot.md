---
title: "Redis with Springboot"
date: 2024-06-17 10:15:00
categories: wiki
author: kgi0412
tags: wiki redis springboot
---
with ChatGPT-4o

## Redis
Redis (REmote DIctionary Server)는 오픈 소스 인메모리 데이터 구조 저장소로, 다양한 데이터 구조(문자열, 해시, 리스트, 셋, 정렬된 셋, 비트맵, 하이퍼로그로그, 지오스페이셜 인덱스 및 스트림)를 지원합니다. Redis는 주로 데이터베이스, 캐시 및 메시지 브로커로 사용됩니다.

- Only 1 Committer

### 주요 기능과 특징
#### 1. 인메모리 데이터 저장소:

- Redis는 모든 데이터를 메모리에 저장하고, 필요에 따라 디스크에 비동기적으로 저장합니다. 이로 인해 매우 빠른 데이터 접근 속도를 제공합니다.

#### 2. 다양한 데이터 구조:

![redis-data-structures](/assets/images/posts/redis-data-structures.png)

- String: 단순한 키-값 쌍.
- List: 순서가 있는 문자열 컬렉션.
- Set: 중복이 없는 문자열의 집합.
- Sorted Set(ZSet): 점수를 기준으로 정렬된 문자열의 집합.
- Hashe: 필드와 값의 쌍으로 이루어진 맵.
- Bitmap: 비트 단위의 조작.
- Hypperloglogs: 근사 카디널리티 계산을 위한 확률적 데이터 구조.
- Geospatial Indexes: 지리적 좌표를 기반으로 한 데이터 구조.
- Streams: 타임 시퀀스 데이터 구조.

#### 3. 고가용성:

- Redis는 레플리케이션을 통해 데이터를 여러 인스턴스에 복제하여 고가용성을 보장합니다.
- Redis Sentinel을 사용하여 자동 장애 조치 및 모니터링을 제공합니다.

#### 4. 분산 처리:

- Redis Cluster를 통해 데이터를 여러 노드에 분산시켜 처리량과 데이터 저장 용량을 확장할 수 있습니다.

#### 5. 퍼시스턴스:

- RDB 스냅샷: 특정 간격으로 메모리 데이터를 디스크에 덤프.
- AOF(Append-Only File): 모든 쓰기 작업을 로그로 기록하여 복구 시 재생.

#### 6. 트랜잭션:

- 여러 명령을 원자적으로 실행하기 위한 트랜잭션을 지원합니다.
- MULTI, EXEC, DISCARD, WATCH 명령어를 사용하여 구현.

#### 7. 스크립팅:

- Lua 스크립팅을 통해 서버 측에서 로직을 실행할 수 있습니다. 이는 원자적 명령어 실행과 로직 복잡성 감소에 유리합니다.

#### 8. Pub/Sub 메시징:

- Redis는 발행/구독 메시징 패턴을 지원하여, 메시지를 특정 채널에 발행하고 여러 구독자가 이를 수신할 수 있습니다.

### 사용 사례
#### 1. 캐싱:

- 웹 애플리케이션에서 자주 조회되는 데이터를 캐시하여 데이터베이스 부하를 줄이고 응답 시간을 단축.

#### 2. 세션 저장소:

- 사용자 세션 데이터를 저장하여, 빠르고 효율적인 세션 관리를 가능하게 함.

#### 3. 실시간 분석:

- 실시간 데이터 분석 및 통계에 활용.

#### 4. 메시지 브로커:

- Pub/Sub 기능을 활용하여 실시간 채팅, 알림 시스템 등에서 메시지 전달.

#### 5. 지오스페이셜 데이터:

- 위치 기반 서비스에서 지리적 데이터 저장 및 질의에 활용.

### Redis의 장단점
#### 장점:
- 매우 빠른 데이터 접근 속도.
- 다양한 데이터 구조 지원.
- 쉬운 설정과 사용.
- 다양한 언어에 대한 클라이언트 라이브러리 지원.

#### 단점:
- 모든 데이터를 메모리에 저장하므로, 대용량 데이터를 처리할 때 메모리 비용이 높아질 수 있음.
- 복잡한 쿼리를 처리하는 기능이 제한적.

## Springboot에서 Redis
Spring Boot와 Redis의 결합은 애플리케이션의 성능 향상과 데이터 관리를 위해 매우 유용합니다. Spring Boot는 간단하고 신속하게 생산성을 높이는 Java 기반 프레임워크이고, Redis는 고속의 인메모리 데이터 저장소입니다. Spring Boot와 Redis의 통합은 다양한 방식으로 이루어질 수 있으며, 주로 캐싱, 세션 관리, 메시지 브로커 등의 용도로 사용됩니다.

### 의미
#### 1. 캐싱(Caching)

- Spring Boot 애플리케이션에서는 데이터를 캐시하여 데이터베이스 조회를 최소화하고 성능을 최적화할 수 있습니다. Redis는 메모리 기반 데이터 저장소로 매우 빠른 데이터 접근 속도를 제공하므로 캐싱 솔루션으로 적합합니다.
- Spring Cache Abstraction과 함께 Redis를 사용하여 캐시 레이어를 손쉽게 구성할 수 있습니다.

```java
@Service
public class UserService {

    @Cacheable(value = "users", key = "#userId")
    public User getUserById(Long userId) {
        // 데이터베이스에서 사용자 정보 조회
    }
}
```

#### 2. 세션 관리(Session Management)

- Spring Session을 사용하여 Redis를 세션 저장소로 구성할 수 있습니다. 이는 분산 환경에서 세션 데이터를 공유하는 데 유리하며, 세션의 지속성과 확장성을 높입니다.
- 예를 들어, 여러 인스턴스의 애플리케이션 서버가 Redis를 통해 세션 데이터를 공유할 수 있습니다.

```properties
spring.session.store-type=redis
spring.redis.host=localhost
```

#### 3. 메시지 브로커(Message Broker)

- Redis의 Pub/Sub 기능을 사용하여 Spring Boot 애플리케이션 간에 메시지를 전달할 수 있습니다. 이를 통해 애플리케이션 간의 통신과 이벤트 처리를 효율적으로 관리할 수 있습니다.

```java
@Component
public class RedisMessageSubscriber implements MessageListener {

    @Override
    public void onMessage(Message message, byte[] pattern) {
        // To-do 메시지 처리 로직
    }
}

@Configuration
public class RedisConfig {

    @Bean
    RedisMessageListenerContainer container(RedisConnectionFactory connectionFactory,
                                            MessageListenerAdapter listenerAdapter) {
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        container.addMessageListener(listenerAdapter, new PatternTopic("my-topic"));
        return container;
    }

    @Bean
    MessageListenerAdapter listenerAdapter(RedisMessageSubscriber subscriber) {
        return new MessageListenerAdapter(subscriber);
    }
}
```

#### 4. 데이터 저장 및 조회

- Redis를 데이터 저장소로 사용하여 간단한 키-값 데이터 또는 복잡한 데이터 구조를 저장하고 조회할 수 있습니다. Spring Data Redis를 사용하면 Spring 애플리케이션에서 Redis와의 상호작용을 간편하게 할 수 있습니다.
```java
@Repository
public class UserRepository {

    private final RedisTemplate<String, User> redisTemplate;

    public UserRepository(RedisTemplate<String, User> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public void saveUser(User user) {
        redisTemplate.opsForValue().set(user.getId(), user);
    }

    public User getUserById(String userId) {
        return redisTemplate.opsForValue().get(userId);
    }
}
```

### Spring Boot와 Redis의 통합 장점
#### 1. 성능 향상:

- Redis의 빠른 데이터 접근 속도를 활용하여 애플리케이션의 응답 시간을 줄이고 데이터베이스 부하를 줄입니다.

#### 2. 확장성:

- Redis는 분산 환경에서 세션 데이터와 캐시 데이터를 공유하는 데 유리하여, 애플리케이션의 확장성을 높입니다.

#### 3. 유연한 데이터 구조:

- Redis의 다양한 데이터 구조를 활용하여 다양한 용도로 데이터를 효율적으로 관리할 수 있습니다.

#### 4. 편리한 설정:

- Spring Boot의 자동 구성 기능을 통해 Redis와의 통합을 쉽게 설정할 수 있습니다.

Spring Boot에서 Redis를 사용하는 것은 애플리케이션의 성능을 최적화하고 확장성을 높이는 데 매우 유용합니다. Redis의 고속 데이터 접근 속도와 다양한 기능을 통해 Spring Boot 애플리케이션의 다양한 요구사항을 충족할 수 있습니다.

## Lettuce와 Jedis
Lettuce와 Jedis는 모두 Redis와 상호작용하기 위한 Java 클라이언트 라이브러리입니다. 두 라이브러리 모두 Spring Boot와 잘 통합될 수 있지만, 각각의 특징과 장단점이 있습니다.

### Lettuce

#### 개요
Lettuce는 Netty 기반의 비동기 및 반응형 Redis 클라이언트입니다. 비동기 및 동기 방식의 API를 제공하며, 고성능과 확장성을 제공합니다.

#### 주요 특징
- 비동기 및 반응형 프로그래밍: Lettuce는 비동기 API를 제공하여 더 높은 성능과 낮은 대기 시간을 목표로 합니다. 리액티브 스트림을 지원하여 반응형 프로그래밍을 구현할 수 있습니다.
- 동기 및 비동기 지원: 필요에 따라 동기 또는 비동기 방식으로 Redis 명령을 실행할 수 있습니다.
- 고성능: Netty 기반으로 설계되어 높은 성능을 제공합니다.
- 멀티 스레드 환경: 안전하게 멀티 스레드 환경에서 사용할 수 있습니다.
- 클러스터 및 레플리케이션 지원: Redis 클러스터와 마스터-슬레이브 구조를 지원합니다.

#### 장점
- 고성능: 비동기 및 반응형 프로그래밍 지원 덕분에 높은 성능을 제공합니다.
- 유연성: 동기, 비동기, 반응형 프로그래밍 모델을 모두 지원합니다.
- 확장성: Netty 기반으로 확장성과 성능이 우수합니다.
- 멀티 스레드 지원: 멀티 스레드 환경에서 안전하게 사용할 수 있습니다.

#### 단점
- 복잡성: 비동기 및 반응형 프로그래밍에 익숙하지 않은 개발자에게는 다소 복잡할 수 있습니다.
- 추가적인 학습 필요: 비동기 및 반응형 API를 사용하는 데 추가적인 학습이 필요합니다.

### Jedis
#### 개요
Jedis는 Redis를 위한 동기식 Java 클라이언트입니다. 간단하고 직관적인 API를 제공하여 사용하기 쉽고, 널리 사용되고 있는 라이브러리입니다.

#### 주요 특징
- 동기식 프로그래밍: 동기식 API를 제공하여 직관적이고 사용하기 쉽습니다.
- 간편한 사용법: 설정과 사용법이 간단하여 빠르게 Redis와 상호작용할 수 있습니다.
- 커넥션 풀링: 다중 스레드 환경에서 커넥션 풀을 관리할 수 있습니다.
- 클러스터 및 레플리케이션 지원: Redis 클러스터와 마스터-슬레이브 구조를 지원합니다.

#### 장점
- 단순성: 동기식 API로 직관적이고 사용하기 쉬워, 빠르게 배울 수 있습니다.
- 안정성: 안정적이고 널리 사용되어 왔으며, 많은 사용자 기반을 가지고 있습니다.
- 풍부한 문서화: 풍부한 문서와 예제가 있어 학습과 사용이 용이합니다.

#### 단점
- 동기식 한계: 동기식 API로 인해 비동기 및 반응형 프로그래밍에 비해 성능이 떨어질 수 있습니다.
- 멀티 스레드 이슈: 기본적으로 Jedis 인스턴스는 스레드 안전하지 않으며, 멀티 스레드 환경에서 안전하게 사용하려면 커넥션 풀을 사용해야 합니다.

#### 비교
| 특징	| Lettuce	| Jedis|
| --- | --- | --- |
|프로그래밍 모델	|비동기 및 반응형, 동기	|동기|
|기반 기술	|Netty	|직접 소켓|
|멀티 스레드 지원	|네이티브 지원	|커넥션 풀 필요|
|성능	|높은 성능	|중간 성능|
|복잡성	|다소 복잡	|간단|
|유연성	|높은 유연성	|제한적 유연성|
|학습 곡선	|비교적 높음	|낮음|
|사용 사례	|고성능, 비동기 및 반응형 애플리케이션	|간단하고 직관적인 애플리케이션|

#### 결론
- Lettuce는 높은 성능과 확장성을 제공하며, 비동기 및 반응형 프로그래밍을 지원합니다. 멀티 스레드 환경에서 안전하게 사용할 수 있고, 유연한 프로그래밍 모델을 제공합니다. 그러나 비동기 및 반응형 프로그래밍에 익숙하지 않은 개발자에게는 다소 복잡할 수 있습니다.

- Jedis는 사용이 간편하고 직관적인 동기식 API를 제공하여, 빠르게 배울 수 있습니다. 다중 스레드 환경에서 사용하려면 커넥션 풀을 관리해야 하며, 비동기 및 반응형 프로그래밍에 비해 성능이 떨어질 수 있습니다.

두 라이브러리 중 어느 것을 사용할지는 애플리케이션의 요구사항과 개발자의 선호도에 따라 달라질 수 있습니다. 고성능이 필요하거나 비동기 작업이 많은 경우 Lettuce를, 간단하고 직관적인 사용이 필요한 경우 Jedis를 선택하는 것이 좋습니다.

## Lettuce와 Jedis의 사용
Spring Boot에서 Redis를 사용하기 위해 spring-boot-starter-data-redis 의존성을 추가하면, 기본적으로 Lettuce 클라이언트가 사용됩니다. 하지만, Jedis 클라이언트로 변경할 수도 있습니다.

### Lettuce를 사용할 때의 코드
기본적으로 spring-boot-starter-data-redis를 사용하면 Lettuce 클라이언트가 사용됩니다. 별도의 설정 없이 Lettuce가 기본 클라이언트로 사용되며, 비동기 및 반응형 프로그래밍이 가능합니다.

#### 예시: Lettuce 기본 설정 및 사용
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.ValueOperations;
import org.springframework.stereotype.Service;

@Service
public class RedisService {

    private final RedisTemplate<String, String> redisTemplate;

    @Autowired
    public RedisService(RedisTemplate<String, String> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public void setValue(String key, String value) {
        ValueOperations<String, String> ops = redisTemplate.opsForValue();
        ops.set(key, value);
    }

    public String getValue(String key) {
        ValueOperations<String, String> ops = redisTemplate.opsForValue();
        return ops.get(key);
    }
}
```
#### 설정 파일
```properties
spring.redis.host=localhost
spring.redis.port=6379
```

### Jedis를 사용할 때의 코드
Jedis 클라이언트를 사용하려면 spring-boot-starter-data-redis 외에도 jedis 의존성을 추가하고, Spring Boot 설정에서 클라이언트를 Jedis로 변경해야 합니다.

#### 의존성 추가
```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

#### Jedis를 사용하는 설정
Spring Boot에서 Redis 클라이언트를 Lettuce에서 Jedis로 변경하려면, application.properties나 application.yml에서 클라이언트를 지정해야 합니다.

```properties
spring.redis.client-type=jedis
```

#### 예시: Jedis 기본 설정 및 사용
코드상에서 사용하는 RedisTemplate과 그 사용법은 Lettuce와 동일합니다. 차이점은 Redis 클라이언트 설정에 있습니다.

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.ValueOperations;
import org.springframework.stereotype.Service;

@Service
public class RedisService {

    private final RedisTemplate<String, String> redisTemplate;

    @Autowired
    public RedisService(RedisTemplate<String, String> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public void setValue(String key, String value) {
        ValueOperations<String, String> ops = redisTemplate.opsForValue();
        ops.set(key, value);
    }

    public String getValue(String key) {
        ValueOperations<String, String> ops = redisTemplate.opsForValue();
        return ops.get(key);
    }
}
```

#### 설정 파일
```properties
spring.redis.host=localhost
spring.redis.port=6379
spring.redis.client-type=jedis
```

### Lettuce와 Jedis의 차이점
- Lettuce: 기본 클라이언트로 설정되어 있으며, 비동기 및 반응형 프로그래밍을 지원합니다. Netty 기반으로 고성능을 자랑합니다.
- Jedis: 전통적인 동기식 클라이언트로, 사용법이 직관적이고 간단합니다. 다중 스레드 환경에서 연결 풀링을 통해 성능을 최적화할 수 있습니다.

### 요약
- spring-boot-starter-data-redis를 사용하면 기본적으로 Lettuce 클라이언트가 사용됩니다.
- Jedis 클라이언트를 사용하려면 추가적인 의존성 설정과 application.properties에서 클라이언트를 지정해야 합니다.
- 코드상에서 RedisTemplate과 그 사용법은 동일하지만, 클라이언트의 설정 및 사용 방식에서 차이가 발생합니다. Lettuce는 비동기 및 반응형 프로그래밍을 지원하고, Jedis는 동기식 클라이언트로 간단하고 직관적인 사용법을 제공합니다.

## 참고
[Redis-Docs](https://redis.io/docs/latest/)

[[NHN FORWARD 2021] Redis 야무지게 사용하기](https://youtu.be/92NizoBL4uA?si=AwZJGS7V7nMuO9Ph)

[[우아한테크세미나] 191121 우아한레디스 by 강대명님](https://youtu.be/mPB2CZiAkKM?feature=shared)

[우아한테크세미나 Redis 후기](https://ict-nroo.tistory.com/133)

[[Redis] 레디스를 파헤쳐 보자!](https://velog.io/@dodozee/Redis)
