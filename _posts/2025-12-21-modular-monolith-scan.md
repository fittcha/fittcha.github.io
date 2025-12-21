---
title: "모듈러 모놀리스 첫 적응기 - 컴포넌트 스캔 오류"
date: 2025-12-22
categories: [Spring]
tags: [spring, modular-monolith, component-scan]
---

CouponFit Commerce 프로젝트에서 Hexagonal 아키텍처 + 모듈러 모놀리스 구조를 처음 시도했다.

Product 도메인 개발 후 Controller까지 만들고 API 테스트를 하려는데...

응? 403?

---

## 프로젝트 구조

```
coupon-fit/
├── cf-app/          ← 메인 실행 모듈 (com.fittcha.couponfit)
├── cf-common/       ← 공통 모듈 (com.fittcha.common)
└── cf-product/      ← 상품 모듈 (com.fittcha.product)
```

---

## 문제 1: 403 Forbidden

```bash
curl -X POST http://localhost:8080/api/products \
  -H "Content-Type: application/json" \
  -d '{"brandId":1,"categoryId":1,"name":"테스트","price":10000}'

# 결과: HTTP/1.1 403
```

당연하지 403!

Spring Security로 인해 인증이 필요했다. `SecurityConfig`를 만들고, `/api/**`는 전부 허용해주었다.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/**").permitAll()
                .anyRequest().authenticated()
            );
        return http.build();
    }
}
```

그런데... 여전히 403??!!

---

## 원인 분석: 컴포넌트 스캔 범위

`@SpringBootApplication`은 **같은 패키지와 하위 패키지만** 스캔한다.

```
CouponFitApplication: com.fittcha.couponfit
SecurityConfig:       com.fittcha.couponfit   ← 같은 패키지, 스캔됨
ProductController:    com.fittcha.product     ← 다른 패키지, 스캔 안 됨!
```

SecurityConfig는 인식되었지만, ProductController가 스캔되지 않아 `/api/products` 엔드포인트 자체가 없었던 것!

---

## 해결 1: scanBasePackages 추가

```java
@SpringBootApplication(scanBasePackages = "com.fittcha")
public class CouponFitApplication {
    // ...
}
```

이제 `com.fittcha` 하위를 전부 스캔하겠지?

---

## 문제 2: JPA Repository Bean 못 찾음

다시 실행했더니 이번엔 다른 에러가 떴다.

```
Parameter 0 of constructor in ProductPersistenceAdapter 
required a bean of type 'ProductJpaRepository' that could not be found.
```

뭐야, `scanBasePackages` 설정했는데 왜 또 못 찾아?

알고 보니 `scanBasePackages`는 일반 Bean(`@Controller`, `@Service`, `@Component`)만 스캔한다. JPA Entity와 Repository는 별도 설정이 필요했던 것이다.

---

## 해결 2: EntityScan, EnableJpaRepositories 추가

```java
@SpringBootApplication(scanBasePackages = "com.fittcha")
@EntityScan(basePackages = "com.fittcha")
@EnableJpaRepositories(basePackages = "com.fittcha")
public class CouponFitApplication {

    public static void main(String[] args) {
       SpringApplication.run(CouponFitApplication.class, args);
    }

}
```

| 어노테이션 | 역할 |
|-----------|------|
| `scanBasePackages` | @Controller, @Service, @Component 등 스캔 |
| `@EntityScan` | @Entity 클래스 스캔 |
| `@EnableJpaRepositories` | JpaRepository 인터페이스 스캔 |

---

## 고민: 선언이 너무 많은 거 아닌가?

이 시점에서 의문이 들었다.

선언이 너무 많은 거 아닌가? `scanBasePackages`, `@EntityScan`, `@EnableJpaRepositories`... 세 개나 붙여야 하다니.

다른 모듈에도 다 똑같이 세 개씩 붙여야 하잖아.

cf-app이 모든 모듈의 상위 패키지에 있으면 이런 설정 필요 없지 않나?

이러면 어떨까? 패키지 계층 맞추기

```
com.fittcha.couponfit           ← cf-app (최상위)
com.fittcha.couponfit.product   ← cf-product
com.fittcha.couponfit.common    ← cf-common
```

이렇게 하면 `@SpringBootApplication` 하나로 전부 스캔 가능하니까 매 모듈마다 선언이 필요 없이 깔끔해지지.

---

## 결론: 패키지 분리 유지

패키지를 `com.fittcha.couponfit` 하위로 통일하면 편하긴 하다. 근데 그러면 모듈러 모놀리스가 가지는 장점을 잃어버리는 거 아닌가?

모듈러 모놀리스의 핵심은 각 모듈이 독립적으로 존재하는 것이다. 나중에 트래픽이 몰리는 모듈만 MSA로 분리할 수 있도록. 근데 패키지가 `com.fittcha.couponfit.product`처럼 app 하위에 있으면, 분리할 때 패키지 구조를 다 바꿔야 한다.

처음부터 `com.fittcha.product`로 독립적인 패키지를 가지면? 나중에 그대로 떼어내면 된다.

```
com.fittcha.couponfit   ← cf-app
com.fittcha.product     ← cf-product
com.fittcha.common      ← cf-common
```

선언 3줄 추가하는 건 처음 한 번이면 끝이고, 그 정도는 독립적인 모듈 분리를 위해 당연히 해야 할 일이었다.

---

## 배운 점

1. **모듈러 모놀리스는 패키지 구조가 중요하다**
   - 기본 스캔 범위를 이해해야 삽질을 줄일 수 있다

2. **편의성 vs 독립성 트레이드오프**
   - 패키지 통일하면 설정 편함
   - 패키지 분리하면 모듈 독립성 유지
   - 목적에 맞게 선택!

3. **Spring의 3가지 스캔 설정**
   - `scanBasePackages`: 일반 Bean
   - `@EntityScan`: JPA Entity
   - `@EnableJpaRepositories`: JPA Repository
