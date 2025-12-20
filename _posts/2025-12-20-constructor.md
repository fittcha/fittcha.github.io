---
title: "생성자 완벽 정리"
date: 2025-12-21
categories: [Java, Spring]
tags: [java, spring, constructor, lombok, di]
---

실무에서 `@Autowired`, `@RequiredArgsConstructor` 늘 쓰면서도 생성자가 정확히 왜 필요한 지 모르고 쓰는 주니어들 많을 것이다.

나도 어노테이션 따라 쓰기만 했지, 왜 생성자 주입이 좋은지, 롬복이 뭘 만들어주는지 제대로 인지하지 않은 채 사용하니 점점 잊어버리고 있었다.

그래서 이번에 생성자부터 정적 팩토리 메서드까지 쭉 정리해보기로 했다.

---

## 생성자란?

생성자는 객체를 만들 때 초기화해주는 것이다.

비유하면:
- **클래스** = 붕어빵 틀
- **객체** = 실제 붕어빵
- **생성자** = 붕어빵 만들 때 팥을 넣을지, 슈크림을 넣을지 정하는 과정

```java
public class Coupon {
    private String name;
    private int discount;
    
    // 생성자
    public Coupon(String name, int discount) {
        this.name = name;
        this.discount = discount;
    }
}

// 객체 생성
Coupon coupon = new Coupon("여름할인", 1000);
```

`new Coupon(...)` 할 때마다 생성자가 호출되면서 그 값들로 초기화된 새 객체가 만들어진다.

---

## 근데 왜 스프링이 대신 만들어줄까?

실무에서 `new CouponService(...)` 이렇게 직접 쓴 적 있나? 없다.

우리는 그냥 늘 그래왔듯 자연스럽게 주입받아서 쓴다.

```java
@RestController
public class CouponController {
    
    private final CouponService couponService;
    
    public CouponController(CouponService couponService) {
        this.couponService = couponService;
    }
}
```

스프링이 하는 일:
1. `CouponRepository` 객체 생성
2. `CouponService` 객체 생성하면서 1번을 넣어줌
3. `CouponController` 객체 생성하면서 2번을 넣어줌

이걸 **의존성 주입(DI, Dependency Injection)**이라고 한다.

생성자에 "나 이거 필요해"라고 정의해놓으면, 스프링이 보고 알아서 넣어주는 것이다.

---

## 왜 생성자 주입이 좋을까?

주입 방식은 크게 두 가지가 있다.

```java
// 필드 주입
@Autowired
private CouponService couponService;

// 생성자 주입
private final CouponService couponService;

public CouponController(CouponService couponService) {
    this.couponService = couponService;
}
```

생성자 주입이 좋은 이유는 세 가지다.

### 1. final 가능

```java
// 필드 주입 - final 못 붙임
@Autowired
private CouponService couponService;

// 생성자 주입 - final 가능
private final CouponService couponService;
```

`final`이 붙으면 한번 주입되면 끝. 누가 실수로 바꿀 수 없다.

### 2. 테스트 쉬움

```java
// 생성자 주입이면
CouponService fakeService = new FakeCouponService();
CouponController controller = new CouponController(fakeService);  // 끝!

// 필드 주입이면
CouponController controller = new CouponController();
// controller.couponService = ???  private이라 못 넣음
// @SpringBootTest, @MockBean 써야 함... 무거워짐
```

### 3. 누락 방지

생성자 주입은 안 넣으면 컴파일 에러. 필드 주입은 일단 되고 런타임에 터진다.

---

## 순환 참조 주의

생성자 주입이 좋은 이유가 하나 더 있다. 순환 참조를 빨리 발견할 수 있다.

```java
@Service
public class OrderService {
    private final CouponService couponService;
}

@Service
public class CouponService {
    private final OrderService orderService;
}
```

서로가 서로를 필요로 하면 스프링이 이러는 거다:

> "OrderService 만들려면 CouponService 필요하네"
> "CouponService 만들려면 OrderService 필요하네"
> "근데 OrderService 만들려면..."
> 🤯 무한루프

생성자 주입이면 앱 실행할 때 바로 에러 터진다. 필드 주입이면 일단 앱은 뜨고, 나중에 호출될 때 터진다. 더 위험하다.

1년차 때 이거 터져본 적 있다. 주문, 주문 하위 클래스들, 정산 클래스들... 필요한 거 다 생성자에 넣다 보니 서로가 서로를 물고 있었다. 그때 처음 알았다. 순환 참조가 생기면 설계를 다시 봐야 한다는 걸. 이래서 처음부터 의존성의 방향이 명확하도록 구조를 잘 짜고, 모든 개발자들이 내 플젝의 구조를 이해하고 있어야 한다는 것을.

---

## 롬복 생성자 어노테이션

우리는 대 롬복이 있지. 매번 생성자를 직접 쓰지 않고 롬복이 대신 만들어주는 것을 이용하면 편하다.

```java
// @NoArgsConstructor - 파라미터 없는 기본 생성자
public Coupon() {}

// @AllArgsConstructor - 모든 필드를 받는 생성자
public Coupon(String name, int discount, String description) {
    this.name = name;
    this.discount = discount;
    this.description = description;
}

// @RequiredArgsConstructor - final 필드만 받는 생성자
public Coupon(String name, int discount) {  // final 붙은 것만
    this.name = name;
    this.discount = discount;
}
```

| 어노테이션 | 생성자 | 주로 쓰는 곳 |
|-----------|--------|-------------|
| `@NoArgsConstructor` | 빈 생성자 | JPA Entity |
| `@AllArgsConstructor` | 모든 필드 | 테스트, DTO |
| `@RequiredArgsConstructor` | final 필드만 | Service, Controller (DI용) |

---

### 생성자가 여러 개면?

스프링은 어떤 생성자로 만들어야 할 지 헷갈려 한다. 이럴 때는 `@Autowired`로 지정해줄 수 있다.

```java
@Autowired
public CouponService(CouponRepository couponRepository, EventPublisher eventPublisher) {
    this.couponRepository = couponRepository;
    this.eventPublisher = eventPublisher;
}
```

근데 생성자가 하나뿐이면 `@Autowired` 생략 가능하다. 스프링이 알아서 그거 쓰니까.

그래서 `@RequiredArgsConstructor`가 편한 것이다. 생성자 하나만 만들어주니 `@Autowired`도 필요 없지.

---

## JPA Entity에서는?

JPA Entity에는 `@NoArgsConstructor`가 필수다. 그 이유는 JPA가 DB에서 데이터를 가져올 때 순서를 보면 알 수 있다.

1. 일단 빈 객체 생성 (`new Coupon()`)
2. 그 다음 필드에 값 채워넣기

```java
@Entity
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Coupon {
    @Id
    private Long id;
    private String name;
}
```

근데 왜 `PROTECTED`일까?

- `PRIVATE`이면 → JPA도 못 씀
- `PUBLIC`이면 → 아무나 `new Coupon()` 해버림
- `PROTECTED`면 → JPA는 쓸 수 있고, 외부에서는 막힘

---

### @Builder 쓰려면?

아마 객체에 필드 주입하기 편해서 많이들 `@Builder`패턴을 사용할 것이다. `@Builder`는 내부적으로 모든 필드를 받는 생성자를 사용하기 때문에, `@NoArgsConstructor`만 두고 같이 쓰면 에러가 난다.

```java
// ❌ 에러
@Builder
@NoArgsConstructor
public class Coupon { }

// ✅ 해결
@Builder
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@AllArgsConstructor
public class Coupon { }
```

그래서 JPA Entity에서 흔히 보는 조합이 바로 이거다.

`@Builder`는 편하게 객체 필드 주입, JPA만 빈 객체 생성하도록 `@NoArgsConstructor(access = AccessLevel.PROTECTED)`, `@Builder`사용하려면 `@AllArgsConstructor`!

```java
@Entity
@Getter
@Builder
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@AllArgsConstructor
public class Coupon {
    @Id
    private Long id;
    private String name;
    private int discount;
}
```

---

## 정적 팩토리 메서드

생성자 대신 객체를 만들어주는 static 메서드다.

```java
// 생성자로 만들기
Coupon coupon = new Coupon("여름할인", 1000);

// 정적 팩토리 메서드로 만들기
Coupon coupon = Coupon.createDiscount("여름할인", 1000);
```

뭐가 좋냐면, 이름을 줄 수 있다.

```java
public class Coupon {
    private String name;
    private int discount;
    private boolean isRate;
    
    public static Coupon createFixedDiscount(String name, int amount) {
        Coupon coupon = new Coupon();
        coupon.name = name;
        coupon.discount = amount;
        coupon.isRate = false;
        return coupon;
    }
    
    public static Coupon createRateDiscount(String name, int percent) {
        Coupon coupon = new Coupon();
        coupon.name = name;
        coupon.discount = percent;
        coupon.isRate = true;
        return coupon;
    }
}
```

```java
// 뭘 만드는지 명확함
Coupon fixedCoupon = Coupon.createFixedDiscount("천원할인", 1000);
Coupon rateCoupon = Coupon.createRateDiscount("10%할인", 10);
```

생성자는 `new Coupon(...)` 밖에 못 쓰는데, 정적 팩토리 메서드는 의도에 맞는 이름을 지을 수 있다.

---

이름 지어서 위와 같이 만들어본 적은 없어도, 다들 이 패턴을 사용한 적이 있다.

```java
List<String> list = List.of("a", "b", "c");
Optional<Coupon> opt = Optional.of(coupon);
String str = String.valueOf(123);
```

`new ArrayList<>()` 대신 `List.of()`. 이것도 정적 팩토리 메서드다.

---

### 상황에 따라 다른 구현체를 반환할 수 있다

```java
public class List {
    public static List of(Object... elements) {
        if (elements.length == 0) {
            return new EmptyList();
        } else if (elements.length == 1) {
            return new SingletonList(elements[0]);
        } else {
            return new RegularList(elements);
        }
    }
}
```

```java
List list1 = List.of();           // EmptyList 반환
List list2 = List.of("a");        // SingletonList 반환
List list3 = List.of("a", "b");   // RegularList 반환
```

생성자는 `new ArrayList()` 하면 무조건 `ArrayList`만 나온다. 선택권이 없지.

정적 팩토리 메서드는 내부에서 상황에 맞는 최적의 구현체를 골라서 반환해줄 수 있다.

---

## 마무리

정리하면:

| 개념 | 설명 |
|-----|-----|
| 생성자 | 객체 초기화 |
| DI | 스프링이 대신 만들어서 넣어줌 |
| 생성자 주입 | final 가능, 테스트 쉬움, 누락 방지 |
| `@RequiredArgsConstructor` | final 필드 생성자 자동 생성 |
| `@NoArgsConstructor` | JPA용 빈 생성자 |
| `@AllArgsConstructor` | `@Builder`랑 같이 쓸 때 |
| 정적 팩토리 메서드 | 이름 있는 생성, 유연한 반환 |

기본기는 자주 다시 정리해봐야 잃지 않는다. 어노테이션 따라 쓰기만 하면 결국 왜 쓰는지 잊어버리게 된다.
