---
layout: post
title: "DB Connection & HikariCP"
subtitle: "DB Connection & JPA 설정 가이드 (HikariCP / Spring Boot)"
date: 2025-12-05
author: kgi0412
# cover-img: /assets/img/
# thumbnail-img: /assets/img/posts/
# share-img: /assets/img/
categories: [wiki, dev]
tags: [ChatGPT, springboot, HikariCP, DBConnection]
language: ko
comments: true
---

## 🚀 DB Connection & JPA 설정 가이드 (HikariCP / Spring Boot)

본 문서는 프로젝트의 데이터베이스 연결 풀(`HikariCP`) 및 JPA/Hibernate 관련 핵심 설정(`application.yml` 기준)의 **의미와 배경**을 설명합니다. 이는 **안정적인 REST API 운영**을 위한 필수 안전장치들입니다.

---

### 📝 최종 YAML 설정 요약 (공통 설정)

```yaml
# ----------------------------------------
# 1. 공통 설정 (Common) - 모든 환경 적용
# ----------------------------------------
spring:
  datasource:
    hikari:
      pool-name: HikariPool-Common
      auto-commit: true           # 무조건 true! (스프링이 완벽 관리)
      connection-timeout: 20000   # 풀 꽉 차도 20초 대기 → 눈사태 방지
      # validation-timeout: 1000    # 개발: 1000 / 운영: 아래 프로필에서 오버라이드
      # leak-detection-threshold: 3000          # 3초 이상 잡으면 무조건 로그+스택 / 운영 : 0
      maximum-pool-size: 40       # 기본값 (운영에서 필요시 오버라이드 가능)
      minimum-idle: 5
      idle-timeout: 600000        # 10분
      max-lifetime: 1740000       # 29분 (오라클 방화벽/세션 타임아웃 방지)
      connection-test-query: SELECT 1       # 오라클 전용 검증 쿼리
  jpa:
    open-in-view: false
    properties:
      hibernate:
        connection:
          provider_disables_autocommit: false
    hibernate:
      ddl-auto: validate

---
# ----------------------------------------
# 2. 개발 환경 설정 (Dev)
# ----------------------------------------
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    hikari:
      pool-name: HikariPool-Dev
      # [개발] 누수 감지 활성화 (3초 이상 점유 시 경고 로그 + 스택트레이스)
      leak-detection-threshold: 3000
      # [개발] 빠른 오류 발견을 위해 짧게 설정
      validation-timeout: 1000 

---
# ----------------------------------------
# 3. 운영 환경 설정 (Prod)
# ----------------------------------------
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    hikari:
      pool-name: HikariPool-Prod
      # [운영] 성능 최적화를 위해 누수 감지 비활성화 (0 = off)
      # 추후 이슈 발생 시 이 값을 수정하고 재배포하면 추적 가능
      leak-detection-threshold: 0
      # [운영] 안정성을 고려해 합의된 3초 유지 (또는 절충안 1초 고려)
      validation-timeout: 3000
```

## 🚀 DB Connection & JPA 설정 가이드 (HikariCP / Spring Boot)

본 문서는 프로젝트의 데이터베이스 연결 풀(`HikariCP`) 및 JPA/Hibernate 관련 핵심 설정의 **의미와 배경**을 설명합니다. 이는 **안정적인 REST API 운영**을 위한 필수 안전장치들입니다.

---

## 1. HikariCP (Connection Pool) 설정

우리는 성능과 안정성 측면에서 검증된 **HikariCP**를 사용하며, 다음 설정을 통해 커넥션 누수와 부하를 방지합니다.

| 설정 항목 | 현재 값 | 의미 | 설정 배경 (Why?) |
| :--- | :--- | :--- | :--- |
| `pool-name` | `HikariPool-Prod` | 커넥션 풀을 식별하기 위한 이름입니다. | 운영 환경에서 로그를 확인할 때 **어떤 풀**에서 문제가 발생했는지 빠르게 파악하기 위함입니다. |
| `auto-commit` | `true` | SQL 작업 완료 후 DB에 자동 커밋을 시도할지 여부입니다. | **Spring Transaction Manager가 트랜잭션을 완전히 관리**하도록 하기 위해 `true`로 설정합니다. JPA 사용 시 권장되는 표준 설정입니다. |
| `connection-timeout` | 20000ms (20초) | 풀이 고갈되었을 때, 새 커넥션을 얻기 위해 최대로 대기하는 시간입니다. | **서버 눈사태(Snowball Effect) 방지**가 주 목적입니다. 풀 고갈 시 즉시 실패하는 대신, 20초 동안 대기하여 일시적인 부하가 해소될 기회를 줍니다. |
| **`validation-timeout`** | **1000ms (1초)** | 커넥션이 유효한지 검사하는 쿼리(`SELECT 1`)가 응답하기를 기다리는 최대 시간입니다. | **빠른 실패(Fail-Fast) 전략**과 안정성을 모두 고려한 절충안입니다. 불필요한 장기 지연을 방지하고 빠른 응답 속도를 유지합니다. 자세한 내용은 아래 **섹션 4**를 참고하세요. |
| `leak-detection-threshold` | 0 (비활성화) | 커넥션이 반환되지 않고 풀 밖에서 일정 시간 머무를 때 경고 로그를 출력합니다. | **[중요]** 운영 환경에서는 오버헤드 방지를 위해 **`0`으로 비활성화**했습니다. 개발/운영 전략에 대한 자세한 내용은 아래 **섹션 5**를 참고하세요. |
| `maximum-pool-size` | 40 | 풀이 가질 수 있는 최대 커넥션 수입니다. | **동시 접속자 수, 애플리케이션 서버 코어 수, DB 성능**을 고려하여 설정한 값입니다. 운영 중 모니터링하여 DB 부하에 맞춰 조정될 수 있습니다. |
| `minimum-idle` | 5 | 풀에서 최소한으로 유지할 유휴(`Idle`) 커넥션 수입니다. | 갑작스러운 요청에도 **즉시 사용할 수 있는 커넥션**을 확보하여 지연 시간을 줄입니다. |
| `max-lifetime` | 1740000ms (29분) | 커넥션이 풀에서 존재할 수 있는 최대 수명입니다. | **DB 서버(예: Oracle)나 방화벽이 보통 30분으로 세션 타임아웃**을 설정하는 경우가 많습니다. 29분으로 설정하여 DB가 커넥션을 강제로 끊기 전에 **미리 안전하게 커넥션을 재발급**하여 오류를 방지합니다. |
| `connection-test-query` | `SELECT 1` | 커넥션 유효성을 검사할 때 사용하는 쿼리입니다. | Oracle DB를 사용하는 경우 표준 검증 쿼리입니다. 이 쿼리가 `validation-timeout` 내에 응답해야 유효한 커넥션으로 판단됩니다. |

---

## 2. Spring JPA & Hibernate 설정

HikariCP 설정 외에도 JPA 레이어에서 데이터 일관성과 커넥션 관리를 강화하는 핵심 안전장치를 적용합니다.

| 설정 항목 | 현재 값 | 의미 | 설정 배경 (Why?) |
| :--- | :--- | :--- | :--- |
| `open-in-view` | `false` | **Open-Session-In-View (OSIV)** 패턴 사용 여부입니다. | **핵심 안전장치 1 (커넥션 누수 방지).** `true`일 경우, HTTP 요청이 끝날 때까지 DB 세션(커넥션)을 물고 있습니다. 이를 **`false`로 꺼서** 트랜잭션 종료 즉시 커넥션을 풀에 반환하도록 강제합니다. |
| `provider_disables_autocommit` | `false` | Hibernate가 커넥션의 `auto-commit` 설정을 임의로 변경하지 못하도록 합니다. | **핵심 안전장치 2.** HikariCP에서 `auto-commit: true`로 설정했으므로, Hibernate가 이를 변경하려 할 때 발생할 수 있는 충돌이나 예상치 못한 동작을 방지합니다. |
| `ddl-auto` | `validate` | 애플리케이션 시작 시 DB 스키마와 엔티티 간의 매핑을 확인하는 설정입니다. | **운영 환경 안전성 확보.** 스키마가 일치하지 않으면 서버 구동을 중단하고 경고합니다. `update`나 `create`를 사용하지 않아 **운영 DB의 데이터 손실 위험**을 완전히 차단합니다. |

---

## 3. 🚨 핵심 안전장치 요약 및 비유

### OSIV 끄기 (`open-in-view: false`)

> **비유:** DB 커넥션은 '**은행 대출 창구**'입니다. OSIV를 끄는 것은 업무(트랜잭션)가 끝나면 **즉시 창구를 비우고** 다음 고객에게 넘겨주어 풀 고갈 위험을 최소화하는 것입니다.

### Leak Detection (`leak-detection-threshold`)

> **비유:** 누수 감지는 '**택시 미반납 경고등**'입니다. 개발자가 커넥션 반환을 잊었을 때(누수), 해당 시간이 초과되면 경고 로그와 함께 **스택 트레이스**를 출력해 즉시 문제를 찾게 돕습니다.

---

## 4. ⚖️ [추가 설명] validation-timeout 값 선정 배경 (1초)

`validation-timeout` 설정은 '빠른 사용자 응답'과 '일시적 지연 회복' 사이의 **균형**을 잡는 핵심 결정입니다. 우리는 이 둘 사이의 장점을 취하기 위해 **1초를 절충안**으로 선택했습니다.

### 1. **"빠른 실패" (Fail-Fast) 관점 (짧은 타임아웃: 1초 이하)**

**전략:** **사용자 경험(Latency) 최우선.** 커넥션이 무효하면 즉시 실패하고 다음 조치를 취해 불필요한 대기를 방지합니다.

**선택 이유:** 커넥션이 이미 끊어졌거나 DB가 응답 불가능할 경우, 3초나 5초를 기다리는 대신 **1초 만에 실패**를 알릴 수 있어 **서비스 응답 지연을 방지**합니다.

### 2. **"안정성 및 회복" (Resilience) 관점 (긴 타임아웃: 3초 ~ 5초)**

**전략:** **일시적인 부하 상황에서의 회복력 최우선.** 긴 대기 시간을 허용하여 정상 커넥션의 오인을 방지합니다.

**주의점:** 커넥션이 **이미 죽은 상태**임에도 불구하고 3초나 5초를 **불필요하게 대기**하게 됩니다. 이 불필요한 대기 시간은 **고객에게 직접적인 서비스 지연**으로 이어질 수 있습니다.

### 3. **결론: 1초 절충안 선택**

* $1$초는 안정적인 환경에서 유효성 검증에 충분한 시간입니다.
* $1$초를 초과하는 지연은 DB나 네트워크에 심각한 문제가 발생했을 가능성이 높으므로, 불필요한 장기 대기(Resilience)보다 **빠른 실패 처리(Fail-Fast)**를 통해 전체 시스템의 응답 속도 안정성을 유지하는 것을 선택했습니다.

---

## 5. 🛠️ [심층 분석] leak-detection-threshold 운영 전략

`leak-detection-threshold`는 커넥션 누수를 진단하는 강력한 도구이지만, 성능 오버헤드를 유발하므로 **환경별 전략**이 필수입니다.

### 1. 개발/운영 환경별 운영 전략

| 환경 | 설정 값 (예시) | 운영 전략 | 이유 |
| :--- | :--- | :--- | :--- |
| **개발 (`dev`)** | **3000ms** (3초) | **경고 모드 (활성화)**. 3초 이상 커넥션 점유 시 상세 로그 출력. | 커넥션 누수를 **개발 단계에서 조기에 발견**하여 운영 배포를 막는 것이 최우선입니다. 스택 트레이스 출력은 문제 코드를 정확히 짚어줍니다. |
| **운영 (`prod`)** | **0** (비활성화) | **성능 최적화 모드.** 누수 감지 기능 비활성화. | 누수 감지 시 발생하는 **로그 I/O 및 스택 트레이스 생성 오버헤드**가 트래픽이 많은 운영 서버의 성능을 저하시킬 수 있습니다. |

### 2. 개발 환경 모니터링 및 알림 가이드 (Grafana 연동 시)

HikariCP는 JMX MBean을 통해 메트릭을 제공하며, 이를 Prometheus/Micrometer로 수집하여 Grafana로 시각화할 수 있습니다.

| 메트릭 항목 | 모니터링/알림 전략 | 설정 목표 |
| :--- | :--- | :--- |
| **`hikari_connections_leak_detection_count`** | **Critical Alert.** 이 카운터 값이 0 이상으로 증가할 때 즉시 슬랙/이메일로 알림 설정. | 누수가 실제로 발생했음을 의미합니다. 개발자는 이 알림 발생 시 해당 시간대 로그를 확인해 스택 트레이스를 분석해야 합니다. |
| **`hikaricp_connections_active`** | **Warning Alert.** 전체 풀 크기의 80% 이상 지속될 때 알림 설정. | 풀 고갈 임박을 경고합니다. 누수 초기 증상일 수 있으며, `maximum-pool-size` 조정 필요성을 나타낼 수도 있습니다. |

---

## 6. 🔗 참고 자료 및 권장 학습 자료

* **HikariCP 공식 GitHub Wiki**: HikariCP의 모든 설정 항목에 대한 공식 문서입니다.
    * `[https://github.com/brettwooldridge/HikariCP/wiki/Configuration](https://github.com/brettwooldridge/HikariCP/wiki/Configuration)`
* **Spring Boot 공식 문서 (Data Access)**: Spring Boot 환경에서 데이터 접근 및 트랜잭션 관리에 대한 공식 가이드입니다.
    * `[https://docs.spring.io/spring-boot/docs/current/reference/html/data.html](https://docs.spring.io/spring-boot/docs/current/reference/html/data.html)`
* **Hibernate 공식 문서**: JPA 구현체인 Hibernate의 상세 동작 및 설정에 대해 참고할 수 있습니다.
    * (버전에 맞게 검색: `Hibernate connection pooling settings`)
* **OSIV (Open Session In View) 논쟁 관련 자료**: OSIV를 끄는 것이 왜 REST API 환경에서 모범 사례인지 이해하는 데 도움이 됩니다.
    * (검색어: `Spring Boot OSIV false rationale`)