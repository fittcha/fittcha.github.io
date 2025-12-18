---
title: "모듈러 모놀리스란? - MSA 가기 전 최선의 선택"
date: 2025-12-18
categories: [Architecture]
tags: [architecture, modular-monolith, msa, microservices]
---

Hexagonal Architecture를 공부하다 보니 자연스럽게 **모듈러 모놀리스**라는 개념을 알게 됐다.

"MSA 해야 하나?" 고민하는 사람들한테 좋은 대안이 될 수 있을 것 같아서 정리해본다.

---

## 용어부터 정리하자

### 모놀리스 (Monolith)

하나의 덩어리. 주문, 결제, 쿠폰, 회원 기능이 전부 한 애플리케이션에 들어있고, 하나의 JAR/WAR로 배포된다.

### MSA (Microservices)

서비스를 잘게 쪼갠 것. 주문 서비스, 결제 서비스, 쿠폰 서비스가 각각 독립된 서버로 돌아가고, API로 통신한다.

### 모듈러 모놀리스 (Modular Monolith)

**배포는 모놀리스처럼 하나로, 내부 구조는 MSA처럼 모듈로 나눈 것.**

이게 핵심이다.

---

## 비유로 이해하기

### 모놀리스 = 원룸

침대, 책상, 냉장고, 옷장, TV, 세탁기가 전부 한 공간에 있다. 요리하다 냄새나면 침대에서도 냄새난다. 모든 게 섞여있어서 하나 건드리면 다른 데 영향이 간다.

### MSA = 각자 따로 사는 자취방

철수네 집, 영희네 집, 민수네 집이 따로 있다. 완전 독립적인데, 만나려면 카톡하고 이동해야 한다. (= 네트워크 통신 비용)

### 모듈러 모놀리스 = 방 나눠진 아파트

안방, 거실, 주방이 문으로 나눠져 있다. 한 집이라서 이동이 빠르고(같은 JVM, 메서드 호출), 주방 냄새가 안방까지 퍼지지 않는다(모듈 분리). 나중에 독립하고 싶으면 방 단위로 분리할 수 있다(MSA 전환).

---

## 코드로 보면 차이가 확실하다

### 일반 모놀리스 (다 섞여있음)

```
src/
├── controller/
│   ├── OrderController.java
│   ├── PaymentController.java
│   └── CouponController.java
├── service/
│   ├── OrderService.java      ← PaymentService 직접 호출
│   ├── PaymentService.java
│   └── CouponService.java
└── repository/
    └── ...
```

`OrderService`가 `CouponService`를 직접 알고 있다. 경계가 없다.

### 모듈러 모놀리스 (모듈별로 분리)

```
src/
├── order/                    # 주문 모듈
│   ├── domain/
│   ├── application/
│   │   └── OrderService.java  ← CouponService 직접 모름!
│   └── adapter/
│
├── coupon/                   # 쿠폰 모듈
│   ├── domain/
│   ├── application/
│   │   └── port/in/
│   │       └── UseCouponUseCase.java  ← 이것만 공개
│   └── adapter/
│
└── common/                   # 공통
```

`OrderService`는 `CouponService` 내부를 모른다. 공개된 `UseCouponUseCase`(Port)만 사용한다.

---

## 모듈 간 통신 규칙

핵심은 **다른 모듈의 내부는 직접 접근 금지**라는 것.

```java
// ❌ 금지 - 다른 모듈 내부 직접 접근
OrderService → CouponRepository  // 남의 DB 접근
OrderService → Coupon            // 남의 도메인 직접 사용

// ✅ 허용 - 공개된 Port만 사용
OrderService → UseCouponUseCase  // 공개 API만 호출
```

```java
@Service
public class OrderService {
    
    // 쿠폰 모듈의 내부 구현은 모른다
    // 공개된 UseCase(Port)만 사용!
    private final UseCouponUseCase useCouponUseCase;
    
    public void createOrder(CreateOrderCommand command) {
        // 쿠폰 사용 요청 (내부가 어떻게 돌아가는지 몰라도 됨)
        useCouponUseCase.useCoupon(command.getCouponId());
        
        // 주문 처리...
    }
}
```

---

## 왜 모듈러 모놀리스가 좋은가?

MSA의 장점은 가져오고, 단점은 버린다.

| 비교 | 일반 모놀리스 | MSA | 모듈러 모놀리스 |
|------|-------------|-----|----------------|
| 배포 | 하나 | 여러 개 | **하나** |
| 모듈 경계 | 없음 | 명확 | **명확** |
| 통신 비용 | 없음 | 높음 (네트워크) | **없음 (메서드 호출)** |
| 트랜잭션 | 쉬움 | 어려움 (분산 트랜잭션) | **쉬움** |
| 운영 복잡도 | 낮음 | 높음 | **낮음** |
| MSA 전환 | 어려움 | 이미 MSA | **쉬움** |

---

## 서비스 성장 시나리오

실제로 이런 흐름으로 발전하는 경우가 많다:

```
1. 처음: 모듈러 모놀리스로 시작
         ↓
2. 서비스 성장
         ↓
3. 특정 모듈만 트래픽 폭발 (예: 쿠폰)
         ↓
4. 해당 모듈만 MSA로 분리
```

처음부터 MSA로 시작하면 오버엔지니어링이다. 트래픽도 없는데 분산 시스템 복잡도만 떠안게 된다.

모듈러 모놀리스로 시작하면 나중에 필요할 때 모듈 단위로 쉽게 떼어낼 수 있다. 이미 경계가 나눠져 있으니까.

---

## CouponFit Commerce에 적용하면

내 토이 프로젝트는 **Hexagonal + 모듈러 모놀리스** 조합으로 가기로 했다.

```
couponfit-commerce/
├── coupon/                    # 🎫 쿠폰 모듈
│   ├── domain/
│   ├── application/
│   │   └── port/in/           # 다른 모듈이 쓸 수 있는 공개 API
│   │       └── UseCouponUseCase.java
│   └── adapter/
│
├── order/                     # 📦 주문 모듈
│   ├── domain/
│   ├── application/
│   └── adapter/
│
├── payment/                   # 💳 결제 모듈
│   └── ...
│
├── member/                    # 👤 회원 모듈
│   └── ...
│
└── common/                    # 🔧 공통
    ├── exception/
    └── config/
```

각 모듈이 Hexagonal 구조를 갖고 있고, 모듈 간에는 Port(UseCase)로만 통신한다.

---

## 의존성 규칙 정리

```
✅ 허용
- 다른 모듈의 port/in/ (UseCase) 접근

❌ 금지  
- 다른 모듈의 domain/ 직접 접근
- 다른 모듈의 repository/ 직접 접근
- 다른 모듈의 service/ 직접 접근
```

이 규칙만 지키면 모듈 간 결합도가 낮아지고, 나중에 MSA로 전환할 때도 수월하다.

---

## 마무리

모듈러 모놀리스는 **"지금 당장 MSA는 과하고, 그냥 모놀리스는 불안한"** 상황에서 좋은 선택지다.

배포는 하나로 단순하게 가져가면서, 내부 구조는 깔끔하게 모듈로 나눈다. 트랜잭션 처리도 쉽고, 운영 복잡도도 낮다. 그러면서도 나중에 필요하면 MSA로 전환할 수 있는 유연성까지 확보할 수 있다.

처음부터 MSA 하지 말고, 모듈러 모놀리스로 시작해서 필요할 때 쪼개자.

---

**이전 글:** [Hexagonal Architecture 제대로 이해하기 - Port와 Adapter란?]

**다음 글:** [Spring Boot에서 Hexagonal + 모듈러 모놀리스 적용하기 (실전)]
