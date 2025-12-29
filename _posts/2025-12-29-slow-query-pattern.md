---
title: "쿼리로 내 어플리케이션 거북이 만드는 법 6가지"
date: 2025-12-29
categories: [Database]
tags: [sql, mssql, performance, optimization]
---

모든 게 빨리빨리인 세상! 슬로우 푸드가 더 건강해보이지 않나요? 사람들은 여유를 가질 때 건강해집니다. 
따라하면 당신의 어플리케이션도 거북이가 될 수 있다! 특별편입니다.

어떻게 하면 느려지는지도 알아야 잘 피할 수 있는 법.
쿼리 최적화 글은 많이 봤지만, 오늘은 반대로 내가 실무에서 겪거나 봤던 느린 쿼리 패턴들을 정리해봤다.

---

## 1. OUTER APPLY 남발하기

```sql
SELECT 
    o.OrderNo,
    pay.PayMethod,
    dtl.TotalQty
FROM OrderMst o
OUTER APPLY (
    SELECT TOP 1 PayMethod FROM OrderPay WHERE OrderNo = o.OrderNo
) pay
OUTER APPLY (
    SELECT SUM(Qty) AS TotalQty FROM OrderDtl WHERE OrderNo = o.OrderNo
) dtl
```

이거 뭐가 문제일까?

### 내부에서 일어나는 일

OUTER APPLY는 **행마다 서브쿼리를 실행**한다. 실행 계획을 보면 Nested Loop으로 처리되는 걸 볼 수 있다.

```
OrderMst 1행 읽음 → OrderPay 서브쿼리 실행 → OrderDtl 서브쿼리 실행
OrderMst 2행 읽음 → OrderPay 서브쿼리 실행 → OrderDtl 서브쿼리 실행
OrderMst 3행 읽음 → OrderPay 서브쿼리 실행 → OrderDtl 서브쿼리 실행
...
```

1만 건 조회하면? OUTER APPLY가 2개니까 2만 번의 서브쿼리가 돈다.

### 왜 Nested Loop일까?

SQL Server가 JOIN 방식을 선택할 때 세 가지 옵션이 있다.

```
Nested Loop: 한쪽을 한 건씩 읽으면서 다른 쪽 탐색
Hash Join: 한쪽으로 해시 테이블 만들고 매칭
Merge Join: 양쪽 정렬해서 순서대로 합침
```

OUTER APPLY는 구조상 **바깥 행 하나에 대해 안쪽 결과를 계산**해야 한다. 그래서 SQL Server가 선택의 여지 없이 Nested Loop을 쓴다.

### 실행 계획에서 보이는 것

실행 계획을 열어보면 이런 식으로 보인다.

```
Nested Loops (Left Outer Join)
├── Clustered Index Scan (OrderMst)  -- 1만 건
└── Clustered Index Seek (OrderPay)  -- 1만 번 반복 실행
```

"1만 번 반복 실행"이 핵심이다. 인덱스가 있어도 1만 번 탐색이고, 없으면 1만 번 풀스캔이다.

---

## 2. NOT IN 대량 데이터에 사용하기

```sql
-- IN
SELECT * FROM Product
WHERE ProductNo IN (
    SELECT ProductNo FROM OrderDtl
)

-- NOT IN
SELECT * FROM Product
WHERE ProductNo NOT IN (
    SELECT ProductNo FROM OrderDtl
)
```

둘 다 비슷해 보이는데, 성능 차이가 크다.

### 내부에서 일어나는 일

**IN은 최적화가 된다.** SQL Server가 Semi Join으로 변환한다.

```
Semi Join이란?
- 서브쿼리에 "존재하는지만" 확인
- 매칭되면 바로 다음 행으로 넘어감
- 해시 테이블로 O(1) 탐색 가능
```

실행 계획을 보면 Hash Match (Right Semi Join)으로 처리되는 걸 볼 수 있다.

**NOT IN은 Anti Semi Join으로 변환되어야 하는데**, NULL 처리 때문에 최적화가 어렵다.

### NULL의 함정

NOT IN에 NULL이 껴있으면 논리가 꼬인다.

```sql
-- OrderDtl에 ProductNo가 NULL인 행이 하나라도 있으면?
WHERE ProductNo NOT IN ('A', 'B', NULL)

-- 이건 내부적으로 이렇게 풀린다
WHERE ProductNo != 'A' AND ProductNo != 'B' AND ProductNo != NULL
```

`ProductNo != NULL`은 항상 UNKNOWN이다. NULL과 비교는 참도 거짓도 아니니까.

AND 조건에서 UNKNOWN이 하나라도 있으면? 전체가 UNKNOWN이 되어 결과에서 제외된다.

그래서 NOT IN 서브쿼리에 NULL이 하나만 있어도 결과가 0건이 나오는 것이다.

### 실행 계획 비교

```
IN:     Hash Match (Right Semi Join) → 빠름
NOT IN: Nested Loops (Left Anti Semi Join) → 느림
        또는 Row by Row 비교 → 더 느림
```

---

## 3. IN절에 뭐 많이 넣기

```sql
SELECT * FROM OrderMst
WHERE OrderNo IN ('ORD001', 'ORD002', 'ORD003', ... , 'ORD15000')
```

간단해 보이지만, 생각보다 무겁다.

### 내부에서 일어나는 일

IN절은 내부적으로 **OR 조건으로 풀린다.**

```sql
WHERE OrderNo = 'ORD001' OR OrderNo = 'ORD002' OR OrderNo = 'ORD003' ...
```

그리고 각 값이 파라미터로 바인딩된다.

```
@P1 = 'ORD001'
@P2 = 'ORD002'
@P3 = 'ORD003'
...
@P15000 = 'ORD15000'
```

SQL Server는 쿼리 실행 전에 이걸 전부 파싱해야 한다. 값이 많아질수록 파싱 시간도 길어지고.

### 실행 계획이 바뀐다

IN절 값이 적으면 Index Seek로 빠르게 처리된다. 근데 값이 많아지면?

```
값 적을 때: Index Seek (빠름)
값 많아질 때: Index Scan 또는 Table Scan (느림)
```

SQL Server 옵티마이저가 "이거 하나씩 찾는 것보다 그냥 다 스캔하는 게 낫겠다"고 판단해버린다.

그리고 IN절 값이 많으면 실행 계획 캐싱도 안 된다. 값 조합이 매번 다르니까 매번 새로 컴파일하게 되는 것이다.

---

## 아무도 캐치 못하게 사소한 걸로 느리게 만들기

---

## 4. SELECT * 쓰기

```sql
-- 이렇게 쓰면 편하긴 하지
SELECT * FROM OrderMst WHERE OrderNo = 'ORD001'

-- 근데 이게 낫다
SELECT OrderNo, OrderDate, CustomerNo FROM OrderMst WHERE OrderNo = 'ORD001'
```

뭐가 다를까?

### 내부에서 일어나는 일

테이블에 컬럼이 30개 있다고 하자. 근데 나한테 필요한 건 3개뿐이다.

SELECT *를 쓰면 27개의 필요 없는 컬럼까지 읽어온다.

```
디스크 I/O: 30개 컬럼 데이터 읽기
네트워크: 30개 컬럼 전송
메모리: 30개 컬럼 적재
```

### Covering Index를 못 쓴다

이게 진짜 문제다.

```sql
-- OrderNo, OrderDate에 복합 인덱스가 있다고 하자
CREATE INDEX IX_Order ON OrderMst (OrderNo, OrderDate)
```

```sql
-- 이건 인덱스만 읽으면 끝 (Index Only Scan)
SELECT OrderNo, OrderDate FROM OrderMst WHERE OrderNo = 'ORD001'

-- 이건 인덱스 읽고 → 테이블도 읽어야 함 (Key Lookup 발생)
SELECT * FROM OrderMst WHERE OrderNo = 'ORD001'
```

SELECT *는 인덱스에 없는 컬럼까지 요구하니까, 결국 테이블까지 다시 가야 한다.

### 그럼 30개 다 필요하면 *써도 되나?

```sql
-- 이거랑
SELECT * FROM OrderMst

-- 이거랑 차이 있을까?
SELECT OrderNo, OrderDate, CustomerNo, ... (30개 전부) FROM OrderMst
```

성능 차이는 없다. 어차피 같은 데이터를 읽으니까.

근데 *는 다른 문제가 있다.

```
1. 누가 테이블에 컬럼 추가하면 갑자기 더 많은 데이터를 읽게 됨
2. 컬럼 순서가 바뀌면 코드가 깨질 수 있음
3. 뭘 조회하는지 코드만 보고 알 수 없음
```

그래서 습관적으로 컬럼을 명시하는 게 좋다.

---

## 5. OR 조건의 함정

```sql
SELECT * FROM OrderMst
WHERE CustomerNo = 'C001' OR OrderDate = '2024-12-25'
```

이거 뭐가 문제일까?

### 내부에서 일어나는 일

CustomerNo에 인덱스가 있고, OrderDate에도 인덱스가 있다고 하자.

AND 조건이면 두 인덱스를 같이 활용할 수 있다. 근데 OR은?

```
AND: CustomerNo로 찾고 → 그 안에서 OrderDate 필터
OR: CustomerNo로 찾은 것 + OrderDate로 찾은 것 합치기
```

SQL Server가 OR을 처리하는 방식:

```
1. 인덱스 두 개 따로 스캔해서 합치기 (Index Union)
2. 그냥 테이블 풀스캔하기
```

옵티마이저가 "합치는 거 귀찮은데 그냥 풀스캔하자"고 판단하는 경우가 많다.

### 왜 풀스캔을 선택할까?

옵티마이저는 비용(Cost)을 계산해서 방식을 선택한다.

```
Index Union 비용:
- Index Seek 1번 (CustomerNo)
- Index Seek 1번 (OrderDate)
- 두 결과 합치기 (Sort + Merge 또는 Hash)
- 중복 제거
- 각 행마다 Key Lookup (테이블에서 나머지 컬럼 가져오기)

Table Scan 비용:
- 테이블 한 번 쭉 읽기
- 끝
```

데이터가 많아지면 Key Lookup 비용이 커진다. 1만 건 매칭되면 1만 번 테이블 왔다갔다 해야 하니까. 그래서 옵티마이저가 "그냥 풀스캔이 낫겠다"고 판단해버리는 것이다.

### 실행 계획에서 확인

```
기대: Index Seek + Index Seek → Union
현실: Table Scan (옵티마이저의 배신)
```

인덱스 잘 걸어놨는데 왜 안 타지? 하면 OR 조건부터 의심해봐야 한다.

---

## 6. 함수로 컬럼 감싸기

```sql
SELECT * FROM OrderMst
WHERE YEAR(OrderDate) = 2024 AND MONTH(OrderDate) = 12
```

날짜에서 연도, 월 뽑아서 조건 거는 것. 꽤 흔하게 볼 수 있는 패턴인데, 이러면 인덱스를 못 탄다.

### 내부에서 일어나는 일

OrderDate에 인덱스가 있다고 하자.

```sql
-- 인덱스 탄다
WHERE OrderDate >= '2024-12-01' AND OrderDate < '2025-01-01'

-- 인덱스 못 탄다
WHERE YEAR(OrderDate) = 2024 AND MONTH(OrderDate) = 12
```

왜?

인덱스는 **컬럼 원본 값 기준**으로 정렬되어 있다.

```
인덱스 구조:
2024-11-28 → 위치 A
2024-11-29 → 위치 B
2024-12-01 → 위치 C
2024-12-02 → 위치 D
...
```

`WHERE OrderDate >= '2024-12-01'`은 인덱스에서 바로 찾을 수 있다. 정렬되어 있으니까.

근데 `YEAR(OrderDate) = 2024`는? SQL Server가 모든 행을 하나씩 꺼내서 YEAR() 함수를 실행해봐야 안다. 그래서 풀스캔이 되는 것이다.

### 이것도 마찬가지

```sql
-- 인덱스 못 탄다
WHERE CONVERT(VARCHAR, OrderDate, 112) = '20241225'
WHERE DATEADD(DAY, 7, OrderDate) > GETDATE()
WHERE UPPER(CustomerName) = 'HONG'

-- 인덱스 탄다
WHERE OrderDate = '2024-12-25'
WHERE OrderDate > DATEADD(DAY, -7, GETDATE())
WHERE CustomerName = 'Hong'
```

컬럼을 건드리지 말고, 비교 값을 가공하는 게 맞다.

---

## 마무리

| 패턴 | 왜 느린가 |
|-----|---------|
| OUTER APPLY 남발 | 행마다 서브쿼리 실행 (Nested Loop) |
| NOT IN 대량 데이터 | 최적화 안 됨 + NULL 함정 |
| IN절에 뭐 많이 넣기 | 파싱 비용 + 실행 계획 캐싱 안 됨 |
| SELECT * | Covering Index 못 씀 |
| OR 조건 | 옵티마이저가 풀스캔 선택 |
| 함수로 컬럼 감싸기 | 인덱스 못 탐 |

이제 실무에서 자칫 내가 느리게 만들고 있지는 않았는지 한번 확인해보자.

혹시 데이터가 작아 큰 영향이 없었을 수도 있다. 

지금 당장은 영향이 없어 보이더라도 이 요소들을 피하고, 더 나아가서 더 빠른 방법을 탐구하다 보면 우리의 어플리케이션은 거북이보다 빠를 수 있다!
