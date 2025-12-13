---
title: "Facade 패턴, 복잡한 시스템을 단순하게 다루는 법"
date: 2025-12-13
categories: [Development, Design Pattern]
tags: [디자인패턴, Facade, Java, Spring, 이커머스]
---

## 들어가며

이커머스 시스템에서 "주문하기" 버튼 하나 누르면 뒤에서 무슨 일이 벌어질까요?

- 재고 확인
- 결제 처리
- 포인트 차감
- 쿠폰 사용 처리
- 주문 내역 저장
- 알림 발송 (카카오톡, SMS, 이메일)
- 외부 물류 시스템 연동

버튼 하나에 이 모든 게 엮여 있습니다. 근데 Controller에서 이걸 다 호출하면 어떻게 될까요?

```java
// 😱 Controller가 이렇게 되면 안 됩니다
@PostMapping("/order")
public ResponseEntity<?> createOrder(@RequestBody OrderRequest request) {
    // 1. 재고 확인
    for (OrderItem item : request.getItems()) {
        int stock = inventoryService.getStock(item.getProductId());
        if (stock < item.getQuantity()) {
            throw new OutOfStockException();
        }
    }
    
    // 2. 재고 차감
    for (OrderItem item : request.getItems()) {
        inventoryService.decreaseStock(item.getProductId(), item.getQuantity());
    }
    
    // 3. 쿠폰 검증 및 사용
    if (request.getCouponId() != null) {
        Coupon coupon = couponService.getCoupon(request.getCouponId());
        if (coupon.isExpired()) {
            throw new CouponExpiredException();
        }
        couponService.useCoupon(request.getCouponId());
    }
    
    // 4. 포인트 차감
    if (request.getUsePoints() > 0) {
        int userPoints = pointService.getPoints(request.getUserId());
        if (userPoints < request.getUsePoints()) {
            throw new InsufficientPointsException();
        }
        pointService.usePoints(request.getUserId(), request.getUsePoints());
    }
    
    // 5. 결제 처리
    PaymentResult paymentResult = paymentService.processPayment(
        request.getPaymentMethod(),
        request.getTotalAmount()
    );
    
    // 6. 주문 저장
    Order order = orderService.createOrder(request, paymentResult);
    
    // 7. 알림 발송
    notificationService.sendKakao(request.getUserId(), order);
    notificationService.sendSms(request.getUserId(), order);
    notificationService.sendEmail(request.getUserId(), order);
    
    return ResponseEntity.ok(order);
}
```

Controller 하나에 70줄이 넘어갑니다. 이런 코드의 문제점은 뭘까요?

1. **Controller가 너무 많은 걸 알고 있음** - 주문 로직의 모든 세부사항을 알아야 함
2. **재사용 불가** - 다른 곳에서 주문 로직 쓰려면 복붙해야 함
3. **테스트 어려움** - 이 Controller 테스트하려면 모든 서비스를 모킹해야 함
4. **변경에 취약** - 알림 채널 하나 추가하면 Controller 수정해야 함

이럴 때 쓰는 게 **Facade 패턴**입니다.

---

## Facade 패턴이란?

### 정의

Facade(퍼사드)는 프랑스어로 "건물의 정면"이라는 뜻입니다. 건물 정면만 보면 깔끔해 보이지만, 뒤에는 복잡한 배관, 전기 시설, 구조물이 숨어있잖아요. 패턴도 마찬가지입니다.

![Facade 패턴 개념도](/assets/img/posts/facade-pattern/facade-concept.png)

**Facade 패턴**은 복잡한 서브시스템들을 하나의 통합된 인터페이스로 감싸서, 클라이언트가 쉽게 사용할 수 있게 해주는 구조적 디자인 패턴입니다.

### GoF의 정의

> "서브시스템의 인터페이스 집합에 대한 통합된 인터페이스를 제공한다. Facade는 서브시스템을 더 쉽게 사용할 수 있게 해주는 고수준 인터페이스를 정의한다."

쉽게 말하면, **"복잡한 건 내가 처리할 테니까, 너는 이것만 호출해"** 입니다.

---

## 주문 처리에 Facade 패턴 적용하기

### Before: Facade 없이

아까 본 코드처럼 Controller가 모든 걸 직접 호출합니다.

![Before: Facade 없이](/assets/img/posts/facade-pattern/facade-before.png)

### After: Facade 적용

![After: Facade 적용](/assets/img/posts/facade-pattern/facade-after.png)

### 코드로 보기

#### OrderFacade.java

```java
@Component
@RequiredArgsConstructor
public class OrderFacade {

    private final InventoryService inventoryService;
    private final CouponService couponService;
    private final PointService pointService;
    private final PaymentService paymentService;
    private final OrderService orderService;
    private final NotificationService notificationService;

    /**
     * 주문 생성
     * - 복잡한 주문 프로세스를 하나의 메서드로 캡슐화
     */
    @Transactional
    public Order createOrder(OrderRequest request) {
        // 1. 재고 확인 및 차감
        validateAndDecreaseStock(request.getItems());

        // 2. 할인 처리 (쿠폰 + 포인트)
        DiscountResult discount = processDiscount(request);

        // 3. 결제
        PaymentResult payment = processPayment(request, discount);

        // 4. 주문 저장
        Order order = saveOrder(request, payment, discount);

        // 5. 알림 발송 (비동기)
        sendNotifications(request.getUserId(), order);

        return order;
    }

    private void validateAndDecreaseStock(List<OrderItem> items) {
        for (OrderItem item : items) {
            inventoryService.validateStock(item.getProductId(), item.getQuantity());
        }
        for (OrderItem item : items) {
            inventoryService.decreaseStock(item.getProductId(), item.getQuantity());
        }
    }

    private DiscountResult processDiscount(OrderRequest request) {
        int couponDiscount = 0;
        int pointDiscount = 0;

        if (request.getCouponId() != null) {
            couponDiscount = couponService.applyCoupon(
                request.getCouponId(), 
                request.getTotalAmount()
            );
        }

        if (request.getUsePoints() > 0) {
            pointDiscount = pointService.usePoints(
                request.getUserId(), 
                request.getUsePoints()
            );
        }

        return new DiscountResult(couponDiscount, pointDiscount);
    }

    private PaymentResult processPayment(OrderRequest request, DiscountResult discount) {
        int finalAmount = request.getTotalAmount() 
                        - discount.getCouponDiscount() 
                        - discount.getPointDiscount();

        return paymentService.process(request.getPaymentMethod(), finalAmount);
    }

    private Order saveOrder(OrderRequest request, PaymentResult payment, DiscountResult discount) {
        return orderService.create(
            request.getUserId(),
            request.getItems(),
            payment,
            discount
        );
    }

    private void sendNotifications(Long userId, Order order) {
        // 실패해도 주문은 성공으로 처리
        try {
            notificationService.sendOrderComplete(userId, order);
        } catch (Exception e) {
            log.warn("알림 발송 실패: orderId={}", order.getId(), e);
        }
    }
}
```

#### OrderController.java

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api/orders")
public class OrderController {

    private final OrderFacade orderFacade;

    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(@RequestBody @Valid OrderRequest request) {
        Order order = orderFacade.createOrder(request);
        return ResponseEntity.ok(OrderResponse.from(order));
    }
}
```

Controller가 엄청 깔끔해졌습니다. 주문의 세부 로직은 전혀 몰라도 되고, Facade에게 위임하면 끝이에요.

---

## Facade 패턴의 특징

### 1. 단방향 의존성

Facade는 여러 Service를 알고 있지만, Service는 Facade를 모릅니다. Controller도 Service를 직접 모르고 Facade만 알고 있어요. 의존성이 `Controller → Facade → Services` 방향으로만 흐르기 때문에 순환 참조 걱정이 없습니다.

Facade가 서브시스템에 의존하지만, 서브시스템은 Facade를 모릅니다. 이게 중요해요. 양방향 의존이 생기면 순환 참조 문제가 발생합니다.

### 2. 서브시스템 직접 접근도 가능

Facade가 있다고 해서 서브시스템에 직접 접근을 막는 건 아닙니다.

```java
// Facade를 통한 접근 (일반적인 주문 플로우)
orderFacade.createOrder(request);

// 서브시스템 직접 접근 (특수한 경우)
// 예: 관리자가 재고만 직접 조회
inventoryService.getStock(productId);
```

필요하면 서브시스템을 직접 쓸 수 있어요. Facade는 **강제가 아니라 편의**를 제공하는 겁니다.

### 3. 여러 Facade 공존 가능

하나의 서브시스템에 여러 Facade가 붙을 수 있습니다.

![여러 Facade 공존](/assets/img/posts/facade-pattern/facade-multiple.png)

---

## 언제 Facade 패턴을 쓸까?

### 쓰면 좋은 경우

**1. 여러 서비스를 조합해야 하는 유스케이스**

```java
// 회원 가입: 회원 생성 + 웰컴 쿠폰 + 포인트 적립 + 알림
memberFacade.signUp(request);

// 상품 등록: 상품 저장 + 이미지 업로드 + 검색 인덱싱 + 카테고리 매핑
productFacade.register(request);

// 환불 처리: 결제 취소 + 재고 복구 + 포인트 환급 + 쿠폰 복원
refundFacade.process(orderId);
```

**2. 외부 시스템 연동을 감싸야 할 때**

```java
// 여러 외부 API를 묶어서 처리
@Component
public class ShippingFacade {
    private final CJLogisticsClient cjClient;
    private final HanjinClient hanjinClient;
    private final LotteLogisticsClient lotteClient;
    
    public TrackingResult getTrackingInfo(String trackingNumber, String carrier) {
        // 택배사별 API 호출 로직을 숨김
    }
}
```

**3. 레거시 시스템을 감싸야 할 때**

```java
// 복잡한 레거시 시스템을 새 인터페이스로 감싸기
@Component
public class LegacyOrderFacade {
    private final OldOrderSystem oldSystem;  // 10년 된 레거시
    
    public Order createOrder(NewOrderRequest request) {
        // 레거시 형식으로 변환해서 호출
        OldOrderFormat oldFormat = convertToLegacy(request);
        OldOrderResult result = oldSystem.processOrder(oldFormat);
        return convertToNew(result);
    }
}
```

### 쓰지 않아도 되는 경우

**1. 단일 서비스만 호출하는 경우**

```java
// 이건 그냥 Service 직접 호출하면 됨
@GetMapping("/products/{id}")
public Product getProduct(@PathVariable Long id) {
    return productService.findById(id);  // Facade 필요 없음
}
```

**2. 서비스 조합이 단순한 경우**

```java
// 이 정도는 Service에서 처리해도 됨
public void updateProfile(ProfileRequest request) {
    memberService.updateProfile(request);
    // 끝
}
```

---

## Facade vs Service vs Controller

"그냥 Service에서 다른 Service 호출하면 되는 거 아니야?" 라는 질문이 나올 수 있어요.

### 역할 구분

각 계층의 역할을 명확히 구분하면:

- **Controller**: HTTP 요청/응답 처리, Request validation, Response 변환. 비즈니스 로직은 없어야 합니다.
- **Facade**: 유스케이스 단위 조율(오케스트레이션), 여러 Service 조합, 트랜잭션 경계 설정. 특정 도메인 로직은 없어야 해요.
- **Service**: 단일 도메인 비즈니스 로직, 해당 도메인만 책임. 다른 Service 호출은 최소화합니다.
- **Repository**: 데이터 접근만 담당.

### Service에서 다른 Service 호출하면?

```java
// 🤔 이렇게 하면 안 되나?
@Service
public class OrderService {
    @Autowired private InventoryService inventoryService;
    @Autowired private PaymentService paymentService;
    @Autowired private CouponService couponService;
    
    public Order createOrder(OrderRequest request) {
        // 모든 로직을 OrderService에서 처리
    }
}
```

할 수는 있지만 문제가 생깁니다.

1. **순환 참조 위험**: OrderService → CouponService → OrderService
2. **책임 과다**: OrderService가 결제, 재고, 쿠폰까지 다 알아야 함
3. **테스트 복잡**: OrderService 테스트에 모든 의존성 필요

Facade를 쓰면:

```java
// OrderService는 주문 도메인만 책임
@Service
public class OrderService {
    public Order create(Long userId, List<OrderItem> items, 
                        PaymentResult payment, DiscountResult discount) {
        // 주문 생성 로직만
    }
}

// 조합은 Facade에서
@Component
public class OrderFacade {
    // 여러 Service를 조율
}
```

각 Service는 자기 도메인만 책임지고, Facade가 조율합니다.

---

## 실무에서 주의할 점

### 1. Facade가 비대해지지 않게

```java
// 😱 이렇게 되면 안 됨
@Component
public class GodFacade {
    // 50개의 Service 주입
    // 100개의 메서드
    // 5000줄의 코드
}
```

Facade도 **단일 책임 원칙**을 지켜야 합니다. 도메인별로 나누세요.

```java
// 😊 도메인별로 분리
OrderFacade        // 주문 관련
MemberFacade       // 회원 관련  
ProductFacade      // 상품 관련
ShippingFacade     // 배송 관련
```

### 2. 비즈니스 로직을 Facade에 넣지 않기

```java
// 😱 Facade에 비즈니스 로직이 들어감
public class OrderFacade {
    public Order createOrder(OrderRequest request) {
        // 할인 계산 로직이 Facade에...
        int discount = 0;
        if (request.getTotalAmount() > 100000) {
            discount = request.getTotalAmount() * 0.1;
        }
        if (isVipMember(request.getUserId())) {
            discount += 5000;
        }
        // ...
    }
}
```

비즈니스 로직은 Service에 있어야 합니다. Facade는 **조율만** 해요.

```java
// 😊 Facade는 조율만
public class OrderFacade {
    public Order createOrder(OrderRequest request) {
        DiscountResult discount = discountService.calculate(request);
        // ...
    }
}
```

### 3. 트랜잭션 범위 고려

```java
@Component
public class OrderFacade {

    @Transactional  // 전체를 하나의 트랜잭션으로
    public Order createOrder(OrderRequest request) {
        validateAndDecreaseStock(request.getItems());  // 재고 차감
        DiscountResult discount = processDiscount(request);  // 쿠폰/포인트
        PaymentResult payment = processPayment(request, discount);  // 결제
        Order order = saveOrder(request, payment, discount);  // 저장
        
        // 알림은 트랜잭션 밖에서 (실패해도 롤백 안 함)
        sendNotificationsAsync(request.getUserId(), order);
        
        return order;
    }
    
    @Async  // 비동기 처리
    public void sendNotificationsAsync(Long userId, Order order) {
        notificationService.sendOrderComplete(userId, order);
    }
}
```

어디까지 트랜잭션으로 묶을지 잘 고려해야 합니다.

---

## 정리

### Facade 패턴 핵심 정리

| 항목 | 내용 |
|------|------|
| **목적** | 복잡한 서브시스템을 단순한 인터페이스로 감싸기 |
| **언제 쓰나** | 여러 서비스를 조합해야 하는 유스케이스 |
| **장점** | 클라이언트 코드 단순화, 서브시스템 캡슐화, 결합도 감소 |
| **단점** | 잘못 쓰면 God Object가 될 수 있음 |
| **주의점** | Facade에 비즈니스 로직 넣지 말기, 도메인별로 분리하기 |

### 면접에서 물어보면

> **Q. Facade 패턴이 뭔가요?**
>
> "복잡한 서브시스템들을 하나의 통합된 인터페이스로 감싸서 클라이언트가 쉽게 사용할 수 있게 해주는 패턴입니다. 예를 들어 주문 처리 시 재고, 결제, 포인트, 알림 등 여러 서비스를 조합해야 하는데, 이걸 OrderFacade로 감싸면 Controller는 하나의 메서드만 호출하면 됩니다. 각 서비스의 세부 구현을 몰라도 되니까 결합도가 낮아지고, 코드도 깔끔해집니다."

> **Q. Service에서 다른 Service를 호출하면 안 되나요?**
>
> "가능하지만 몇 가지 문제가 있습니다. 순환 참조가 발생할 수 있고, 하나의 Service가 너무 많은 책임을 지게 됩니다. Facade를 도입하면 각 Service는 자기 도메인만 책임지고, 유스케이스 단위의 조율은 Facade에서 담당해서 책임이 명확하게 분리됩니다."

---

사실 Facade 패턴은 개념 자체는 단순합니다. "복잡한 걸 감싸서 단순하게 만든다." 근데 실무에서 잘 쓰려면 어디까지 감쌀지, 어디서 트랜잭션을 끊을지, 비즈니스 로직은 어디에 둘지 같은 고민이 필요해요.

다음에 기회가 되면 Template Method 패턴도 다뤄볼게요. Facade랑 자주 같이 쓰이는 패턴이거든요.

그럼 다음에 또 봐요!
