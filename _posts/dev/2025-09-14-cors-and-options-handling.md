---
layout: post
title: "Handling of cors and options requests in Spring"
subtitle: "Spring Boot RESTful API에서 CORS와 `OPTIONS` 요청 처리 가이드"
date: 2025-09-14
author: kgi0412
# cover-img: /assets/img/
# thumbnail-img: /assets/img/posts/
# share-img: /assets/img/
categories: [wiki, dev]
tags: [ChatGPT, springboot, CORS, OPTIONS]
language: ko
comments: true
---

# Spring Boot RESTful API 및 Spring MVC에서 CORS와 `OPTIONS` 요청 처리 가이드

이 문서는 Spring Boot 기반 RESTful API 서비스를 중심으로 `Allow`와 `Access-Control-Allow-Methods` 헤더의 차이, `OPTIONS` 요청 처리 방법, 브라우저의 CORS 동작, 그리고 Nginx와의 상호작용을 정리한 가이드입니다. 서버 간 호출 전용 API에서 브라우저 요청을 차단하는 데 초점을 맞췄으며, 비교 참고로 Spring MVC 기반 내용을 추가로 다룹니다.

## 1. `Allow`와 `Access-Control-Allow-Methods`의 차이

- `Allow` **헤더**:

  - **역할**: 서버가 특정 리소스에서 지원하는 HTTP 메서드를 클라이언트에 알림.
  - **상황**: `OPTIONS` 요청이나 `405 Method Not Allowed` 응답에서 사용.
  - **예시**: `Allow: GET, POST, PUT, DELETE`
  - **특징**: HTTP/1.1 표준, CORS와 무관, 정보 제공용.

- `Access-Control-Allow-Methods` **헤더**:

  - **역할**: CORS 정책에서 크로스 오리진 요청으로 허용되는 메서드를 지정.
  - **상황**: `OPTIONS` 요청(프리플라이트) 응답에서 사용, 브라우저가 후속 요청을 허용할지 판단.
  - **예시**: `Access-Control-Allow-Methods: GET, POST`
  - **특징**: CORS 전용, 실제 요청 제어.

**핵심**: `Allow`는 지원 메서드를 알리고, `Access-Control-Allow-Methods`는 크로스 오리진 요청을 제한. `Access-Control-Allow-Methods`는 `Allow`의 하위 집합이어야 함.

## 2. 브라우저의 프리플라이트 요청 흐름

브라우저는 \*\*단순 요청(Simple Request)\*\*이 아닌 경우에만 프리플라이트(`OPTIONS`) 요청을 보냅니다. 아래는 프리플라이트 요청과 응답의 흐름을 정상, 메서드 미지원, 예외 처리 경우로 나눠 시각화한 도표입니다.

### Mermaid Flow 다이어그램

```mermaid
graph TD
    A[Browser: 비단순 요청<br>(예: POST with Content-Type: application/json)] -->|1. 프리플라이트 요청<br>OPTIONS /api/test| B{Nginx}
    B -->|프록시| C[Spring Boot/MVC API]

    subgraph 정상 흐름
        C -->|2. 200 OK<br>Access-Control-Allow-Origin: *<br>Access-Control-Allow-Methods: GET, POST| D[Browser: 프리플라이트 성공]
        D -->|3. 본 요청<br>POST /api/test| E[Spring Boot/MVC API]
        E -->|4. 200 OK<br>JSON 응답 (REST)<br>또는 View (MVC)| F[Browser: 요청 성공]
    end

    subgraph 메서드 미지원 (405)
        C -->|2. 405 Method Not Allowed<br>Allow: GET, POST| G[Browser: 프리플라이트 실패]
        G --> H[본 요청 차단]
    end

    subgraph 예외 처리 (CORS 오류)
        C -->|2. 200 OK<br>Access-Control-Allow-Origin 없음| I[Browser: CORS 정책 위반]
        I --> J[본 요청 차단<br>콘솔 에러: No 'Access-Control-Allow-Origin']
    end

    subgraph 단순 요청 (프리플라이트 없음)
        K[Browser: 단순 요청<br>(예: GET /api/test)] -->|1. 본 요청| L[Spring Boot/MVC API]
        L -->|2. 200 OK<br>Access-Control-Allow-Origin 없음| M[Browser: CORS 정책 위반]
        M --> N[응답 차단<br>콘솔 에러]
    end
```

**설명**:

- **정상 흐름**: 프리플라이트 성공(`200`, CORS 헤더 포함) → 본 요청 허용.
- **메서드 미지원**: `OPTIONS`에 `405` → 프리플라이트 실패 → 본 요청 차단.
- **CORS 오류**: `OPTIONS`에 `Access-Control-Allow-Origin` 없음 → 본 요청 차단.
- **단순 요청**: 프리플라이트 없이 본 요청 → CORS 헤더 없음 → 응답 차단.

### 단순 요청 조건

- **HTTP 메서드**: `GET`, `HEAD`, `POST`.
- **허용 헤더**: `Accept`, `Accept-Language`, `Content-Language`, `Content-Type` (값: `application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain`).
- **예시**:
  - `GET /api/data` → 단순 요청, 프리플라이트 없음.
  - `POST /api/data` (Content-Type: `application/json`) → 비단순 요청, 프리플라이트 발생.

## 3. 서버 간 전용 RESTful API에서 브라우저 요청 차단

서버 간 호출 전용 RESTful API에서 브라우저 요청을 차단하려면:

1. **CORS 비활성화**: `Access-Control-Allow-Origin` 등 CORS 헤더 제거.
2. `OPTIONS`**를** `405`**로 처리**: 프리플라이트 실패로 비단순 요청 차단.

### Spring Boot RESTful API에서 `OPTIONS` 처리

#### 인터셉터로 `OPTIONS` 차단

```java
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@Component
public class BlockOptionsInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
        if ("OPTIONS".equalsIgnoreCase(request.getMethod())) {
            response.setHeader("Allow", "GET, POST, PUT, DELETE");
            response.sendError(HttpServletResponse.SC_METHOD_NOT_ALLOWED);
            return false;
        }
        return true;
    }
}
```

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {
    private final BlockOptionsInterceptor blockOptionsInterceptor;

    public WebConfig(BlockOptionsInterceptor blockOptionsInterceptor) {
        this.blockOptionsInterceptor = blockOptionsInterceptor;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(blockOptionsInterceptor).addPathPatterns("/**");
    }
}
```

**결과**:

- `OPTIONS` → `405 Method Not Allowed`, `Allow: GET, POST, PUT, DELETE`.
- 비단순 요청 차단, 서버 리소스 소모 최소.

#### Spring Security로 `OPTIONS` 차단

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests((requests) -> requests
                .requestMatchers(org.springframework.http.HttpMethod.OPTIONS, "/**").denyAll()
                .anyRequest().permitAll()
            )
            .cors().disable()
            .csrf().disable();
        return http.build();
    }
}
```

**결과**:

- `OPTIONS` → `405`, CORS 헤더 없음.
- 브라우저 요청 차단, 서버 간 호출 정상.

#### RESTful API 컨트롤러 예시

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api")
public class ApiController {
    @GetMapping("/test")
    public String getTest() {
        return "{\"message\": \"GET response\"}";
    }

    @PostMapping("/test")
    public String postTest() {
        return "{\"message\": \"POST response\"}";
    }
}
```

**결과**:

- 단순 요청: `Access-Control-Allow-Origin` 없음 → 브라우저 응답 차단.
- 비단순 요청: `OPTIONS` → `405` → 본 요청 차단.

## 4. Spring MVC와의 비교 (참고)

Spring MVC는 뷰 렌더링(예: JSP, Thymeleaf)을 중심으로 동작하며, RESTful API와 달리 응답이 주로 HTML입니다. 서버 간 호출 전용이 아닌, 웹 애플리케이션에 적합합니다.

### Spring MVC에서 `OPTIONS` 처리

Spring MVC에서는 `@Controller`를 사용하며, `OPTIONS` 요청을 명시적으로 처리하거나 인터셉터로 차단 가능.

#### 컨트롤러에서 `OPTIONS` 처리

```java
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

@Controller
@RequestMapping("/example")
public class ExampleController {
    @RequestMapping(value = "/test", method = RequestMethod.GET)
    public String getTest() {
        return "testView"; // 예: testView.jsp
    }

    @RequestMapping(value = "/test", method = RequestMethod.OPTIONS)
    public ResponseEntity<Void> handleOptions() {
        HttpHeaders headers = new HttpHeaders();
        headers.add("Allow", "GET, POST");
        return new ResponseEntity<>(headers, HttpStatus.METHOD_NOT_ALLOWED); // 405
    }
}
```

**차이점**:

- **RESTful API**: JSON 응답 (`@RestController`), 서버 간 호출 최적화.
- **MVC**: 뷰 렌더링 (`@Controller`), 브라우저 UI 중심.
- **CORS 처리**: MVC는 CORS를 활성화하여 브라우저 요청을 허용하는 경우가 많음. 서버 간 전용 API라면 RESTful API와 동일하게 CORS 비활성화 가능.

#### 인터셉터로 `OPTIONS` 차단 (MVC 동일)

RESTful API와 동일한 인터셉터 사용 가능 (위 `BlockOptionsInterceptor` 참조).

**결과**:

- MVC에서도 `OPTIONS` → `405`로 브라우저 요청 차단 가능.
- 뷰 렌더링이 필요 없으므로 `ResponseEntity`로 응답 제어.

## 5. Nginx와의 상호작용

Nginx가 프록시로 사용될 때, CORS 헤더나 `OPTIONS` 응답 수정에 주의해야 합니다.

### 문제 시나리오: Nginx가 `OPTIONS`를 `200`으로 처리

```nginx
location /api {
    if ($request_method = OPTIONS) {
        add_header Access-Control-Allow-Origin "*";
        add_header Access-Control-Allow-Methods "GET, POST, OPTIONS";
        return 200;
    }
    proxy_pass http://spring-backend;
}
```

**결과**:

- `OPTIONS` → `200 OK`, CORS 헤더 포함.
- 브라우저 프리플라이트 성공 → 후속 요청 허용 → 서버 간 전용 의도 깨짐.

### 해결: Nginx에서 `OPTIONS`를 `405`로 처리

```nginx
location /api {
    if ($request_method = OPTIONS) {
        add_header Allow "GET, POST, PUT, DELETE";
        return 405;
    }
    proxy_pass http://spring-backend;
}
```

**결과**:

- `OPTIONS` → `405`, 프리플라이트 실패.
- 브라우저 요청 차단.

### 최적: Nginx에서 CORS 헤더 제거

```nginx
location /api {
    proxy_pass http://spring-backend;
}
```

**결과**:

- Spring의 `405` 응답과 CORS 비활성화 유지.
- 브라우저 요청 차단.

## 6. 리소스 소모 최소화

단순 요청은 서버 리소스를 소모하므로, 이를 줄이는 방법:

### `Origin` 헤더로 브라우저 요청 차단

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import javax.servlet.http.HttpServletRequest;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;

@RestController
@RequestMapping("/api")
public class ApiController {
    @GetMapping("/test")
    public ResponseEntity<String> getTest(HttpServletRequest request) {
        String origin = request.getHeader("Origin");
        if (origin != null && !origin.isEmpty()) {
            return ResponseEntity.status(HttpStatus.FORBIDDEN)
                                .body("Browser requests are not allowed");
        }
        return ResponseEntity.ok("{\"message\": \"GET response\"}");
    }
}
```

**효과**:

- 브라우저 요청 조기 차단, 리소스 소모 최소화.

### Nginx에서 `Origin` 체크

```nginx
location /api {
    if ($http_origin) {
        return 403 "Browser requests are not allowed";
    }
    proxy_pass http://spring-backend;
}
```

## 7. 결론

- **비단순 요청**: `OPTIONS` → `405` → 프리플라이트 실패 → 요청 차단, 리소스 소모 없음.
- **단순 요청**: 본 요청 → CORS 헤더 없음 → 브라우저 응답 차단, 리소스 소모 있음.
- **Nginx 주의**: `OPTIONS`를 `200`으로 처리하거나 CORS 헤더 추가 시 브라우저 요청 허용 가능.
- **Spring MVC 비교**: RESTful API는 JSON 중심, MVC는 뷰 렌더링 중심. CORS 비활성화 및 `OPTIONS` 처리 방식은 동일 적용 가능.
- **최적화**: CORS 비활성화, `OPTIONS`를 `405`로 처리, `Origin` 체크로 리소스 소모 최소화.

## 8. 테스트 방법

- **비단순 요청**:

  ```javascript
  fetch('http://your-api/api/test', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ data: 'test' })
  }).catch(error => console.error('CORS Error:', error));
  ```

- **단순 요청**:

  ```javascript
  fetch('http://your-api/api/test', { method: 'GET' })
      .catch(error => console.error('CORS Error:', error));
  ```

- **서버 간 호출**:

  ```bash
  curl -X GET http://your-api/api/test
  ```