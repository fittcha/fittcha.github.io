---
title: "IN절의 한계와 대안 - 대용량 조건 조회 2편"
date: 2025-12-25
categories: [Database]
tags: [sql, mssql, performance, optimization]
---

지난 글에서 한방 쿼리를 분리하고 Java HashMap으로 조합해서 50초를 3초로 줄였다.

끝인 줄 알았지만, 운영 데이터 테스트를 여러 번 하다 보니 바로 새로운 문제를 찾았다.

바로,, 주문이 많은 행사 기간에는 1시간에 많게는 15,000여 건의 주문이 생성된 적도 있다는 것!

내 쿼리 분리 구조는 IN절을 사용하고 있었다. 15,000건을 IN절에 넣으면 어떻게 될까?

---

## IN절의 한계: 2,100개 파라미터 제한

```sql
SELECT * FROM OrderMst 
WHERE OrderNo IN ('ORD001', 'ORD002', ... , 'ORD15000')
```

이 쿼리를 실행하면? SQL Server에서 에러가 난다.

---

## IN절 내부에서 일어나는 일

IN절은 내부적으로 **OR 조건으로 풀린다.**

```sql
-- 이게
WHERE OrderNo IN ('ORD001', 'ORD002', 'ORD003')

-- 내부적으로 이렇게 변환됨
WHERE OrderNo = 'ORD001' OR OrderNo = 'ORD002' OR OrderNo = 'ORD003'
```

그리고 각 값은 **파라미터로 바인딩**된다.

```
@P1 = 'ORD001'
@P2 = 'ORD002'
@P3 = 'ORD003'
...
```

SQL Server는 쿼리를 실행하기 전에 **파싱(구문 분석)**을 하는데, 이 파라미터 개수에 제한이 있다. 바로 **2,100개**.

왜 2,100개일까? SQL Server의 TDS(Tabular Data Stream) 프로토콜에서 파라미터를 전송할 때의 한계인데, 정확히는 내부 버퍼 크기와 관련이 있다고 한다.

15,000건? 당연히,, 에러가 났다.

---

## 그럼 2,000건씩 끊으면 되지 않나?

처음엔 그렇게 생각했다.

```java
// 2000건씩 나눠서 IN절 호출
List<List<String>> chunks = Lists.partition(orderNos, 2000);
for (List<String> chunk : chunks) {
    mapper.selectOrderMstByOrderNos(chunk);
}
```

근데 문제가 있다.

만약 쿼리 분리 구조를 사용하여, IN절을 사용하는 쿼리가 한 사이클에 5개라고 하자. 15,000건이면?

```
15,000건 ÷ 2,000건 = 8번
8번 × 5개 쿼리 = 40번 DB 호출
```

쿼리 수가 폭발한다. DB 커넥션 획득/반환 비용도 40번, 네트워크 왕복도 40번. 오히려 더 느려졌다.

---

## 그럼 임시 테이블에 넣고 JOIN하면?

IN절 대신 임시 테이블을 만들어서 JOIN하면 쿼리 수를 줄일 수 있다.

```sql
-- 임시 테이블 생성
CREATE TABLE #TempOrderNos (OrderNo VARCHAR(20))

-- 15,000건 INSERT
INSERT INTO #TempOrderNos VALUES ('ORD001'), ('ORD002'), ...

-- JOIN으로 조회
SELECT m.* 
FROM OrderMst m
INNER JOIN #TempOrderNos t ON m.OrderNo = t.OrderNo
```

이러면 INSERT 1번 + SELECT 5번 = 6번으로 끝난다. 40번보다 훨씬 낫지.

근데... 우리 시스템에서는 이 방법을 쓸 수 없었다.

---

## 임시 테이블 내부에서 일어나는 일

임시 테이블(#temp)은 어디에 생길까? 바로 **tempdb**다.

tempdb는 SQL Server의 **시스템 데이터베이스**로, 모든 세션이 공유한다.

```
세션 A: #TempOrderNos 생성 → tempdb
세션 B: #TempProducts 생성 → tempdb
세션 C: #TempUsers 생성 → tempdb
...
모든 임시 테이블이 tempdb 한 곳에 몰림
```

---

## pagelatch 경합이 뭔데?

SQL Server는 데이터를 **페이지(8KB)** 단위로 관리한다.

여러 세션이 동시에 tempdb에 임시 테이블을 만들면, 같은 페이지에 접근하게 된다. 이때 **페이지 단위로 잠금(latch)**이 걸린다.

```
세션 A: tempdb 페이지 100번에 쓰기 시도 → 잠금 획득
세션 B: tempdb 페이지 100번에 쓰기 시도 → 대기...
세션 C: tempdb 페이지 100번에 쓰기 시도 → 대기...
```

이게 **pagelatch 경합**이다. 락이 걸린 것처럼 줄줄이 대기하게 된다.

특히 트래픽이 몰리는 시간대에 심하다. 우리 시스템도 상품-주문 테이블 조인에서 임시 테이블을 많이 쓰고 있었는데, 피크 시간에 이 문제로 지연이 발생하고 있었다.

여기에 내 배치까지 tempdb를 쓰면? 불난 집에 기름 붓는 격이지,,

---

## 최종 해결: 물리 테이블 임시 적재

결국 물리 테이블을 사용하기로 했다.

```sql
-- 물리 테이블 (미리 만들어둠)
CREATE TABLE TempOrderNos (
    OrderNo VARCHAR(20),
    BatchId VARCHAR(36)  -- 배치 실행 구분용
)

-- 1. INSERT
INSERT INTO TempOrderNos (OrderNo, BatchId) 
VALUES ('ORD001', 'batch-uuid'), ('ORD002', 'batch-uuid'), ...

-- 2. JOIN으로 조회
SELECT m.* 
FROM OrderMst m
INNER JOIN TempOrderNos t ON m.OrderNo = t.OrderNo
WHERE t.BatchId = 'batch-uuid'

-- 3. 사용 후 삭제
DELETE FROM TempOrderNos WHERE BatchId = 'batch-uuid'
```

임시 테이블과 뭐가 다를까?

| 구분 | 임시 테이블 (#temp) | 물리 테이블 |
|-----|-------------------|------------|
| 저장 위치 | tempdb | 일반 DB |
| 경합 | tempdb에 몰림 | 분산됨 |
| 세션 | 세션 종료 시 삭제 | 직접 삭제 필요 |

---

## JOIN 내부에서 일어나는 일

그러면 IN절 대신 JOIN을 쓰면 뭐가 다를까?

SQL Server는 JOIN할 때 세 가지 방식 중 하나를 선택한다.

```
1. Nested Loop Join
   - 한쪽 테이블을 한 건씩 읽으면서 다른 테이블 탐색
   - 소량 데이터에 적합

2. Hash Join
   - 한쪽 테이블로 해시 테이블 생성
   - 다른 테이블을 해시로 매칭
   - 대량 데이터에 적합

3. Merge Join
   - 양쪽 테이블이 정렬되어 있을 때
   - 순서대로 비교하며 합침
   - 인덱스 있으면 빠름
```

15,000건 JOIN이면? SQL Server가 알아서 Hash Join을 선택해준다. IN절처럼 OR 조건 15,000개를 파싱할 필요 없이, 해시 테이블 한 번 만들고 매칭하면 끝이니까 더 효율적인 방식을 사용하게 되는 것이다.

---

## 상황에 맞는 방법 선택하기

| 상황 | 방법 | 이유 |
|-----|-----|-----|
| 소량 (100건 이하) | IN절 | 단순하고 빠름 |
| 중량 (1,000건 내외) | Java HashMap 조합 | tempdb 부하 없이 처리 |
| 대량 (10,000건 이상) | 물리 테이블 임시 적재 | IN절 한계, 쿼리 수 폭발 방지 |

---

## 마무리

1편에서 쿼리 분리 + Java HashMap으로 50초를 3초로 만들었다. 근데 대량 데이터 앞에서는 또 다른 벽이 있었다.

IN절 2,100개 제한, 쿼리 수 폭발, tempdb pagelatch 경합... 하나씩 부딪히면서 결국 물리 테이블 임시 적재까지 오게 되었다.

결국 정답은 하나가 아니었다. 데이터 규모와 시스템 상황에 맞게 선택하는 것이 중요하다는 걸 다시 한번 느꼈다.
