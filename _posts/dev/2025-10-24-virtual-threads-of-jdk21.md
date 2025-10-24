---
layout: post
title: "JDK21의 Virtual Threads"
subtitle: "Spring Boot에서 Virtual Threads와 Platform Threads 비교 및 최적화 가이드"
date: 2025-10-24
author: kgi0412
# cover-img: /assets/img/
# thumbnail-img: /assets/img/posts/
# share-img: /assets/img/
categories: [wiki, dev]
tags: [ChatGPT, springboot, jdk21, virtual_threads]
language: ko
comments: true
---

# Spring Boot에서 Virtual Threads와 Platform Threads 비교 및 최적화 가이드

## 1. 개요

Spring Boot 3.5+와 Undertow를 사용하는 RESTful API 프로젝트에서 **Virtual Threads**와 **Platform Threads**의 성능을 비교하고, **Pinned Threads(핀락)** 메커니즘을 이해하며, **도커 스웜 환경**에서의 최적 설정을 정리한 문서입니다. 이 문서는 실무 적용 사례와 Spring의 동작 방식을 기반으로 작성되었습니다.

### 프로젝트 환경

- **Spring Boot**: 3.5+
- **서버**: Undertow
- **JDK**: 21 (Virtual Threads 지원)
- **도커 스웜**: 4개 컨테이너 (컨테이너당 vCPU 2, 메모리 2GB)
- **주요 작업**: `RestClient` 프록시 API (90%), 동기 JPA 쿼리 (10%)

---

## 2. Virtual Threads vs Platform Threads

### 2.1. 성능 비교

Virtual Threads는 **동시성이 높은 작업**(예: HTTP API 호출, 짧은 I/O 작업)에서 Platform Threads보다 **일반적으로 더 빠르고 메모리 효율적**입니다. 이는 Virtual Threads가 **Pinned Threads(핀락)** 상황을 최소화하며 캐리어 스레드를 효율적으로 재사용하기 때문입니다. 반면, **핀락이 발생하는 작업**(예: 동기 DB 쿼리, `ReentrantLock`)에서는 Platform Threads와 성능 차이가 거의 없습니다.

#### 주요 차이

| 항목 | Virtual Threads | Platform Threads |
| --- | --- | --- |
| **스레드 생성** | 수십만 개 가능 (1KB/스레드) | 제한적 (1MB/스레드) |
| **핀락 상황** | 캐리어 스레드 해방 (효율적) | 스레드 블록킹 (비효율적) |
| **메모리 사용** | 낮음 (작은 스택) | 높음 (큰 스택) |
| **적합 작업** | 논블로킹 I/O (RestClient, HTTP) | CPU 집약적, 동기 작업 (DB 쿼리) |

#### 권장 사용 시나리오

- **Virtual Threads**: 대규모 동시 요청, 논블로킹 I/O 작업
- **Platform Threads**: 동기 DB 쿼리, 파일 I/O, `synchronized` 블록

### 2.2. 테스트 예제

아래 코드는 Virtual Threads와 Platform Threads의 성능을 비교하는 컨트롤러로, **I/O 블록킹 작업**을 시뮬레이션하여 처리 시간과 메모리 사용량을 측정합니다.

#### 테스트 코드

```java
package com.example.restapdemo;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.lang.management.ManagementFactory;
import java.lang.management.MemoryMXBean;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicLong;

@RestController
public class LoadTestController {

    private final MemoryMXBean memoryMXBean = ManagementFactory.getMemoryMXBean();

    @GetMapping("/load-test3")
    public String loadTest3(
            @RequestParam(defaultValue = "10000") int requests,
            @RequestParam(defaultValue = "true") boolean useVirtualThreads,
            @RequestParam(defaultValue = "10") int delayMs) {
        
        // ForkJoinPool 튜닝 (Virtual Threads용)
        if (useVirtualThreads) {
            System.setProperty("jdk.virtualThreadScheduler.maxPoolSize", "16");
            System.setProperty("jdk.virtualThreadScheduler.minRunnable", "8");
        }

        // 스레드 풀 설정
        ExecutorService executor = useVirtualThreads
                ? Executors.newVirtualThreadPerTaskExecutor()
                : Executors.newFixedThreadPool(400, r -> {
                    Thread t = new Thread(r);
                    t.setName("platform-thread-" + t.getId());
                    return t;
                });

        // 활성 스레드 모니터링
        ScheduledThreadPoolExecutor scheduler = new ScheduledThreadPoolExecutor(2);
        scheduler.scheduleWithFixedDelay(() -> {
            System.out.println("Thread.activeCount: " + Thread.activeCount() + 
                               ", Virtual: " + useVirtualThreads);
        }, 300L, 300L, TimeUnit.MILLISECONDS);

        // 메모리 및 시간 측정
        long startTime = System.currentTimeMillis();
        AtomicLong maxHeap = new AtomicLong(memoryMXBean.getHeapMemoryUsage().getUsed());
        AtomicInteger successCount = new AtomicInteger(0);
        AtomicInteger errorCount = new AtomicInteger(0);

        // 요청 처리
        for (int i = 0; i < requests; i++) {
            final int requestId = i;
            executor.submit(() -> {
                try {
                    Thread.sleep(delayMs);
                    successCount.incrementAndGet();
                    System.out.println("Request " + requestId + " on thread: " + 
                                       Thread.currentThread() + 
                                       ", isVirtual: " + Thread.currentThread().isVirtual());
                } catch (InterruptedException e) {
                    errorCount.incrementAndGet();
                    System.err.println("Error in request " + requestId + ": " + e.getMessage());
                }
            });
        }

        // 메모리 모니터링
        Thread memoryMonitor = new Thread(() -> {
            while (!executor.isTerminated()) {
                long currentHeap = memoryMXBean.getHeapMemoryUsage().getUsed();
                maxHeap.updateAndGet(current -> Math.max(current, currentHeap));
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        });
        memoryMonitor.start();

        try {
            executor.shutdown();
            executor.awaitTermination(600, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return "Error: Interrupted";
        } finally {
            scheduler.shutdownNow();
            memoryMonitor.interrupt();
            if (!executor.isShutdown()) {
                executor.shutdownNow();
            }
        }

        long duration = System.currentTimeMillis() - startTime;
        long maxHeapMB = maxHeap.get() / (1024 * 1024);

        return String.format(
            "Processed %d requests (%d success, %d failed) using %s in %d ms, Max heap: %d MB",
            requests,
            successCount.get(),
            errorCount.get(),
            useVirtualThreads ? "Virtual Threads" : "Platform Threads",
            duration,
            maxHeapMB
        );
    }
}
```

#### 테스트 설명

- **목적**: Virtual Threads와 Platform Threads의 **I/O 블록킹 작업 처리 성능** 비교
- **파라미터**:
  - `requests`: 요청 수 (기본 10,000)
  - `useVirtualThreads`: Virtual Threads 사용 여부 (기본 `true`)
  - `delayMs`: 각 요청의 블록킹 시간 (기본 10ms)
- **동작**:
  - `Thread.sleep(delayMs)`로 I/O 블록킹 시뮬레이션
  - 활성 스레드 수(`Thread.activeCount`)와 최대 힙 메모리 측정
  - 성공/실패 카운트 집계
- **실행 방법**:

  ```bash
  # Virtual Threads 테스트
  curl "http://localhost:8080/fit/load-test3?requests=10000&useVirtualThreads=true&delayMs=10"
  
  # Platform Threads 테스트
  curl "http://localhost:8080/fit/load-test3?requests=10000&useVirtualThreads=false&delayMs=10"
  ```

#### 예상 결과

| 설정 | 처리 시간 | 메모리 사용 | 성공률 |
| --- | --- | --- | --- |
| **Virtual Threads** | **100-200ms** | \~50-70MB | 100% |
| **Platform Threads** | **2,000-3,000ms** | \~60-100MB | 100% |

- **Virtual Threads**: 캐리어 스레드 해방으로 동시 처리 효율적
- **Platform Threads**: 400개 스레드 풀 제한으로 큐잉 발생
- **핀락 테스트 추가**:

  ```java
  if (useLock) {
      lock.lock();
      try {
          Thread.sleep(delayMs);
      } finally {
          lock.unlock();
      }
  }
  ```

  → 핀락 발생 시 Virtual Threads 성능이 Platform Threads와 동일(\~2,000-3,000ms)

---

## 3. Pinned Threads(핀락) 메커니즘

### 3.1. 핀락이란?

Virtual Thread가 \*\*캐리어 스레드(실제 CPU 스레드)\*\*에 \*\*고정(Pinned)\*\*되어 다른 Virtual Thread가 캐리어를 사용할 수 없는 상태입니다. 핀락이 발생하면 Virtual Threads의 동시성 이점이 사라지고, Platform Threads와 동일한 성능을 보입니다.

#### 비유

```
놀이기구(CPU 16코어)에 10,000명 탑승
- Platform Threads: 400개 의자 → 나머지 대기
- Virtual Threads: 10,000개 종이표 → **핀락 시 캐리어 고정, 비핀락 시 캐리어 해방**
```

### 3.2. 핀락 발생 상황

| 작업 | 핀락 발생? | Virtual Threads 효과 | 권장 방식 |
| --- | --- | --- | --- |
| `Thread.sleep()` | ✅ | **효율적 (캐리어 해방)** | Virtual Threads |
| `RestClient` | ❌ | **효율적 (논블로킹)** | Virtual Threads |
| `동기 DB 쿼리` | ✅ | **효과 없음** | Platform Threads |
| `ReentrantLock` | ✅ | **효과 없음** | Platform Threads |
| `synchronized` | ✅ | **효과 없음** | 피하기 |

#### 코드 예시

```java
// 빠름: 핀락 없음
Thread.sleep(10);  // 캐리어 스레드 해방

// 느림: 핀락 발생
lock.lock();
try {
    Thread.sleep(10);  // 캐리어 스레드 고정
} finally {
    lock.unlock();
}
```

---

## 4. Spring Boot에서의 동작 방식

### 4.1. `spring.threads.virtual.enabled: true`

- **적용 범위**:
  - Undertow Worker: Virtual Threads (RestClient 호출에서 빠름)
  - JPA 동기 쿼리: Virtual Threads (해락 발생 → Platform과 동일)
  - RestClient: Virtual Threads (논블로킹 → 빠름)
  - `@Async`: 기본적으로 Platform Threads (별도 설정 필요)
- **결론**: RestClient 위주의 프로젝트에서는 Virtual Threads 활성화가 유리

### 4.2. Spring의 자동 최적화

Spring은 내부적으로 작업 특성에 따라 최적 스레드를 선택합니다:

- **RestClient**: 논블로킹 → Virtual Threads 효과 극대화
- **JPA 동기 쿼리**: 해락 발생 → Platform Threads와 동일
- **권장**: 별도 `@Async` 구분 없이 `spring.threads.virtual.enabled: true`로 충분

---

## 5. 도커 스웜 환경 설정

### 5.1. 환경

- **물리서버**: 4코어
- **컨테이너**: 4개 (vCPU 2, 메모리 2GB)

### 5.2. Undertow 설정

| 설정 | 기본값 | 추천값 | 이유 |
| --- | --- | --- | --- |
| `worker` | CPU × 8 (32) | **200** | 동시 요청 800 처리 가능 |
| `io` | CPU (4) | **8** | 네트워크 I/O 최적화 |

### 5.3. 도커 컴포즈

```yaml
version: '3.8'
services:
  app:
    deploy:
      replicas: 4
    resources:
      limits:
        cpus: '2.0'
        memory: 2G
    environment:
      - JAVA_OPTS=-Djdk.virtualThreadScheduler.maxPoolSize=16
```

---

## 6. 실무 적용 설정

### 6.1. 최종 `application.yml`

```yaml
server:
  undertow:
    threads:
      io: 8
      worker: 200
spring:
  threads:
    virtual:
      enabled: true
logging:
  level:
    io.undertow: INFO
```

### 6.2. 코드 예시

```java
@RestController
public class ProxyController {
    @Autowired
    private RestClient restClient;

    @GetMapping("/proxy")
    public String proxy() {
        // Virtual Threads → 논블로킹, 빠름
        return restClient.get().uri("external-api").retrieve().body(String.class);
    }
}

@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;

    public Order findOrder(Long id) {
        // Virtual Threads (해락 발생 → Platform과 동일)
        return orderRepository.findById(id);
    }
}
```

### 6.3. 성능 전망

| 트래픽 | Platform Threads | Virtual Threads |
| --- | --- | --- |
| **1,000 req/s** | 보통 | **매우 빠름** |
| **10,000 req/s** | 느림 또는 실패 | **빠르고 안정적** |
| **DB 100 req/s** | 안정적 | **안정적 (동일)** |

---

## 7. 결론 및 권장사항

- **Virtual Threads 활성화**:
  - RestClient 위주 API → 동시성이 높은 작업에서 빠름
  - 동기 DB 쿼리는 해락으로 Platform Threads와 동일
- **worker: 200**:
  - 4개 컨테이너 → 총 800 동시 요청 처리
  - 메모리: 400MB (2GB 내 20%)
- **해락 관리**:
  - `ReentrantLock`, `synchronized` 사용 최소화
  - RestClient, HTTP 호출은 Virtual Threads로 최대 효과
- **모니터링**:

  ```java
  @GetMapping("/thread-info")
  public String threadInfo() {
      return String.format("Thread: %s, Virtual: %b", Thread.currentThread(), Thread.currentThread().isVirtual());
  }
  ```

---

## 8. 추가 참고

### 8.1. JFR로 해락 디버깅

Java Flight Recorder(JFR)를 사용하여 Virtual Threads의 해락 상황을 모니터링할 수 있습니다.

```bash
# JFR 활성화
docker exec <container-id> java -XX:+FlightRecorder -XX:StartFlightRecording=filename=recording.jfr -jar app.jar

# 해락 이벤트 확인
docker exec <container-id> jfr print --events jdk.VirtualThreadPinned recording.jfr
```

### 8.2. JMeter 테스트

JMeter를 사용해 Virtual Threads와 Platform Threads의 성능을 비교할 수 있습니다. 아래는 테스트 플랜 설정 방법입니다.

#### JMeter 설정

1. **JMeter 설치**: Apache JMeter 다운로드
2. **테스트 플랜**:
   - **Thread Group**:
     - Number of Threads: 1,000
     - Ramp-up Period: 10초
     - Loop Count: 1
   - **HTTP Request 1 (Virtual Threads)**:
     - URL: `http://<swarm-ip>:8080/fit/load-test3?requests=10000&useVirtualThreads=true&delayMs=10`
   - **HTTP Request 2 (Platform Threads)**:
     - URL: `http://<swarm-ip>:8080/fit/load-test3?requests=10000&useVirtualThreads=false&delayMs=10`
   - **Listener**:
     - Summary Report
     - Response Time Graph
     - Aggregate Report
3. **실행**:

   ```bash
   # JMeter 실행
   jmeter -n -t test-plan.jmx -l results.jtl
   ```
4. **결과 분석**:
   - **Virtual Threads**: 낮은 응답 시간(\~100-200ms), 안정적
   - **Platform Threads**: 높은 응답 시간(\~2,000-3,000ms), 큐잉 지연 가능
   - **메모리**: Virtual Threads가 약 20-40% 효율적
5. **권장 추가 테스트**:
   - 동시 요청 수 증가: `requests=50000`
   - 실제 API 호출: `RestClient`로 외부 API 호출 테스트
   - 해락 상황: `useLock=true` 추가하여 해락 성능 비교

#### 테스트 실행 예시

```bash
# Virtual Threads
curl "http://localhost:8080/fit/load-test3?requests=10000&useVirtualThreads=true&delayMs=10"

# Platform Threads
curl "http://localhost:8080/fit/load-test3?requests=10000&useVirtualThreads=false&delayMs=10"
```

---

**작성자**: \[Your Name\]\
**작성일**: 2025년 10월 24일\
**프로젝트**: RESTful API 프록시 서비스\
**목적**: GitHub Pages 배포용 문서