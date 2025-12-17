---
title: "Hexagonal Architecture 제대로 이해하기 - Port와 Adapter란?"
date: 2025-12-17
categories: [Architecture]
tags: [architecture, hexagonal, ports-and-adapters, clean-code]
---

지난 글에서 여러 아키텍처를 비교하고 Hexagonal을 선택했다고 했는데, 이번에는 Hexagonal Architecture를 제대로 파헤쳐보기로 했다.

처음 접하면 "Port? Adapter? 뭔 소리야?" 싶다. 나도 그랬다.

---

## Hexagonal, 이름의 의미

Hexagonal(육각형)이라는 이름은 사실 큰 의미가 없다. 창시자 Alistair Cockburn이 "포트가 여러 개 붙을 수 있다"는 걸 표현하려고 육각형을 그렸을 뿐이다.

진짜 이름은 **Ports & Adapters**다. 이쪽이 본질을 더 잘 설명해준다.

---

## 핵심 개념: Port와 Adapter

### Port = 규격서 (인터페이스)

Port는 도메인과 외부 세계 사이의 "규격"이다. Java로 치면 인터페이스.

**Input Port (Driving Port)**: 외부에서 도메인을 사용하기 위한 규격

```java
// "쿠폰 생성하려면 이렇게 요청해줘"
public interface CreateCouponUseCase {
    CouponResponse createCoupon(CreateCouponCommand command);
}
```

**Output Port (Driven Port)**: 도메인이 외부를 사용하기 위한 규격

```java
// "쿠폰 저장하려면 이렇게 해줘"
public interface CouponRepository {
    Coupon save(Coupon coupon);
    Optional<Coupon> findById(Long id);
}
```

---

### Adapter = 변환기 (구현체)

Adapter는 Port를 실제로 구현한 것이다. 외부 기술을 Port 규격에 맞게 변환해주는 역할이라고 보면 된다.

**Driving Adapter (Input Adapter)**: 외부 요청을 받아서 Port 호출

```java
@RestController
@RequiredArgsConstructor
public class CouponController {
    
    private final CreateCouponUseCase createCouponUseCase;  // Port 주입
    
    @PostMapping("/api/coupons")
    public ResponseEntity<CouponResponse> createCoupon(
            @RequestBody CreateCouponRequest request) {
        
        CreateCouponCommand command = request.toCommand();
        CouponResponse response = createCouponUseCase.createCoupon(command);
        
        return ResponseEntity.ok(response);
    }
}
```

**Driven Adapter (Output Adapter)**: Port를 구현해서 실제 인프라 연결

```java
@Repository
@RequiredArgsConstructor
public class CouponPersistenceAdapter implements CouponRepository {
    
    private final CouponJpaRepository jpaRepository;
    private final CouponMapper mapper;
    
    @Override
    public Coupon save(Coupon coupon) {
        CouponJpaEntity entity = mapper.toEntity(coupon);
        CouponJpaEntity saved = jpaRepository.save(entity);
        return mapper.toDomain(saved);
    }
    
    @Override
    public Optional<Coupon> findById(Long id) {
        return jpaRepository.findById(id)
                .map(mapper::toDomain);
    }
}
```

---

## 프랜차이즈 비유로 다시 이해하기

프랜차이즈 본사를 생각해보자.

**본사(Domain)**는 "떡볶이 레시피"만 갖고 있다. 주문이 어디서 오는지, 재료가 어디서 오는지 모른다. 알 필요도 없다.

**주문 규격서(Input Port)**가 있어서 배달앱이든, 키오스크든, 전화든 이 규격만 맞추면 주문을 받을 수 있다.

**재료 규격서(Output Port)**가 있어서 CJ든, 오뚜기든, 로컬 농장이든 이 규격만 맞추면 재료를 납품받을 수 있다.

**각 연결 담당자(Adapter)**가 규격에 맞게 변환해준다. 배달앱 연동 담당자, CJ 납품 담당자처럼.

이 구조의 장점은? 배달앱을 쿠팡이츠에서 배민으로 바꿔도 본사 레시피는 안 건드려도 된다. 연결 담당자(Adapter)만 바꾸면 되는 것!

---

## "도메인이 인프라에 의존하지 않는다"의 의미

이게 Hexagonal의 핵심이다. 코드로 비교해보자.

### Layered의 문제

```java
@Service
public class CouponService {
    
    private final CouponJpaRepository repository;  // JPA에 직접 의존!
    
    public Coupon createCoupon(CreateCouponDto dto) {
        CouponEntity entity = new CouponEntity();  // JPA Entity 직접 사용!
        entity.setName(dto.getName());
        return repository.save(entity);
    }
}
```

Service가 JPA를 직접 알고 있다. JPA를 MongoDB로 바꾸면? Service 코드 전체 수정해야 한다.

### Hexagonal의 해결

```java
// Domain - 순수 자바 객체, 아무것도 모름
public class Coupon {
    private Long id;
    private String name;
    private int discountAmount;
    private LocalDateTime expiredAt;
    
    public boolean isExpired() {
        return LocalDateTime.now().isAfter(expiredAt);
    }
    
    public void validate() {
        if (discountAmount <= 0) {
            throw new InvalidCouponException("할인 금액은 0보다 커야 합니다");
        }
    }
}
```

```java
// Service - Port(인터페이스)에만 의존
@Service
@RequiredArgsConstructor
public class CreateCouponService implements CreateCouponUseCase {
    
    private final CouponRepository couponRepository;  // Port! JPA인지 모른다
    
    @Override
    public CouponResponse createCoupon(CreateCouponCommand command) {
        Coupon coupon = Coupon.create(
            command.getName(),
            command.getDiscountAmount(),
            command.getExpiredAt()
        );
        
        coupon.validate();  // 도메인 로직
        
        Coupon saved = couponRepository.save(coupon);
        
        return CouponResponse.from(saved);
    }
}
```

Service는 `CouponRepository`라는 인터페이스(Port)만 안다. 그게 JPA로 구현됐는지, MongoDB로 구현됐는지 모른다.

JPA를 MongoDB로 바꾸고 싶으면? `CouponMongoAdapter`만 새로 만들면 된다. Service는 그대로!

---

## 의존성 역전 (DIP)

Hexagonal의 마법은 **의존성 역전**에서 나온다고 볼 수 있다.

**Layered에서는** Service가 구현체(JPA Repository)에 의존한다. 화살표가 위에서 아래로 흐른다.

**Hexagonal에서는** Service와 JPA Adapter 둘 다 추상화(Port)에 의존한다. Service → Port ← JPA Adapter 이런 모양이다.

이렇게 되면 JPA Adapter를 갈아끼워도 Service는 영향이 없다. 둘 다 Port만 바라보고 있으니까!

---

## 왜 테스트하기 좋은가?

이게 실무에서 체감되는 가장 큰 장점이다.

### Layered에서의 테스트

```java
@SpringBootTest  // Spring 컨텍스트 로딩 필요 → 느림
class CouponServiceTest {
    
    @Autowired
    private CouponService couponService;
    
    @MockBean  // JPA Repository Mocking 필요 → 복잡
    private CouponJpaRepository repository;
    
    @Test
    void createCoupon() {
        when(repository.save(any())).thenReturn(new CouponEntity());
        // 테스트...
    }
}
```

Spring 컨텍스트 로딩에 몇 초씩 걸리고, Mocking도 복잡하다.

### Hexagonal에서의 테스트

```java
// Spring 없이 순수 단위 테스트!
class CreateCouponServiceTest {
    
    private CreateCouponService service;
    private CouponRepository repository;
    
    @BeforeEach
    void setUp() {
        repository = new FakeCouponRepository();  // Fake 구현
        service = new CreateCouponService(repository);
    }
    
    @Test
    void 쿠폰_생성_성공() {
        CreateCouponCommand command = new CreateCouponCommand(
            "10% 할인", 1000, LocalDateTime.now().plusDays(30)
        );
        
        CouponResponse response = service.createCoupon(command);
        
        assertThat(response.getName()).isEqualTo("10% 할인");
    }
    
    @Test
    void 할인금액_0이하면_예외() {
        CreateCouponCommand command = new CreateCouponCommand(
            "잘못된 쿠폰", -1000, LocalDateTime.now().plusDays(30)
        );
        
        assertThatThrownBy(() -> service.createCoupon(command))
            .isInstanceOf(InvalidCouponException.class);
    }
}
```

```java
// 테스트용 Fake Repository - 간단!
class FakeCouponRepository implements CouponRepository {
    
    private Map<Long, Coupon> store = new HashMap<>();
    private Long sequence = 1L;
    
    @Override
    public Coupon save(Coupon coupon) {
        coupon.setId(sequence++);
        store.put(coupon.getId(), coupon);
        return coupon;
    }
    
    @Override
    public Optional<Coupon> findById(Long id) {
        return Optional.ofNullable(store.get(id));
    }
}
```

Spring 없이 순수 자바로 테스트할 수 있다. 빠르고, 명확하고, 도메인 로직에만 집중할 수 있다.

---

## 실제 프로젝트 구조

Spring Boot 프로젝트에 Hexagonal을 적용하면 이런 구조가 나온다.

```
coupon/
├── domain/                      # 핵심 도메인
│   ├── Coupon.java              # 도메인 엔티티
│   └── CouponPolicy.java        # 도메인 정책
│
├── application/                 # 애플리케이션 계층
│   ├── port/
│   │   ├── in/                  # Input Ports
│   │   │   └── CreateCouponUseCase.java
│   │   └── out/                 # Output Ports
│   │       └── CouponRepository.java
│   └── service/
│       └── CreateCouponService.java
│
└── adapter/                     # 어댑터 계층
    ├── in/web/
    │   ├── CouponController.java
    │   └── dto/
    │       └── CreateCouponRequest.java
    └── out/persistence/
        ├── CouponPersistenceAdapter.java
        ├── CouponJpaRepository.java
        ├── CouponJpaEntity.java
        └── CouponMapper.java
```

폴더 구조만 봐도 역할이 명확하게 보인다!

---

## 요청 흐름 정리

쿠폰 생성 요청이 들어오면 이런 흐름으로 처리된다.

1. **HTTP 요청**이 들어옴
2. **CouponController**(Driving Adapter)가 Request DTO를 Command로 변환
3. **CreateCouponUseCase**(Input Port)를 호출
4. **CreateCouponService**(UseCase 구현)가 도메인 객체 생성하고 비즈니스 검증
5. **CouponRepository**(Output Port)를 통해 저장 요청
6. **CouponPersistenceAdapter**(Driven Adapter)가 Domain을 JPA Entity로 변환해서 DB 저장
7. 저장된 Entity를 다시 Domain으로 변환해서 반환
8. **Response** 반환

복잡해 보이지만, 각 단계가 하는 일이 명확하다. 그래서 문제가 생겨도 어디서 생겼는지 바로 찾을 수 있다!

---

## 핵심 정리

| 개념 | 설명 |
|------|------|
| **Port** | 도메인과 외부의 경계를 정의하는 인터페이스 |
| **Adapter** | Port를 구현하여 실제 기술과 연결 |
| **Driving (Input)** | 외부 → 도메인 방향 (Controller 등) |
| **Driven (Output)** | 도메인 → 외부 방향 (Repository 등) |
| **의존성 방향** | 항상 바깥 → 안쪽 (도메인은 아무것도 모름) |

---

## 마무리

Hexagonal Architecture의 핵심은 **도메인을 보호하는 것**이다.

도메인은 순수한 비즈니스 로직만 담고, 외부 기술(DB, API, 프레임워크)과는 Port/Adapter로 연결하면 된다. 이렇게 하면 외부 기술이 바뀌어도 도메인은 안전하고, 테스트도 쉬워진다.

처음엔 파일이 많아져서 복잡해 보일 수 있다. 근데 익숙해지면 "어디에 뭘 넣어야 하지?"라는 고민이 확 줄어든다. 구조가 명확하니까!

다음 글에서는 모듈러 모놀리스에 대해 알아보자. Hexagonal과 조합하면 MSA 전환도 쉬운 깔끔한 구조를 만들 수 있다.

---

**이전 글:** [소프트웨어 아키텍처 비교 - Layered vs Hexagonal vs Clean vs CQRS]

**다음 글:** [모듈러 모놀리스란? - MSA 가기 전 최선의 선택]
