# 8장 SQL의 순서

> 환경에 따라서 Sequential 대신 Primary Key의 인덱스를 사용한 Index Scan(또는 Index Only Scan)이 나타날 수 있음.  
> Query Plan이 차이가 존재 (여기서는 SQL Fiddle의 Query Plan 결과를 가져오나 설명은 책을 사용)

## 23강 레코드에 순번 붙이기
### 1. 기본 키가 한 개의 필드일 경우
기본 키가 한개인 Weights 테이블에 순번 붙이기 [`SQL1`](http://sqlfiddle.com/#!15/45d46/3)

|<U>student_id</U>|weight|    |<U>student_id</U>|seq|
|:---------------:|:----:|:--:|:---------------:|:-:|
|A100             |	 50  |    |A100             | 1 |
|A101             |  55  |    |A101             | 2 |
|A124             |  55  |    |A124             | 3 |
|A346             |  60  | &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;→&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; |A346             | 4 |
|B343             |  72  |    |B343             | 5 |
|C345             |  72  |    |C345             | 6 |
|C563             |  72  |    |C563             | 7 |

#### 1-1) 윈도우 함수 사용

```sql
SELECT student_id,
       ROW_NUMBER() OVER (ORDER BY student_id) AS seq
FROM Weights;
```

> **QUERY PLAN**  
> WindowAgg (cost=0.15..93.45 rows=1510 width=20)  
> &nbsp;&nbsp;&nbsp;&nbsp; -> Index Only Scan using weights_pkey on weights (cost=0.15..70.80 rows=1510 width=20)
> <br>

> - Index Only Scan으로 테이블에 직접적인 접근 회피

#### 1-2) 상관 서브쿼리 사용
- MySQL처럼 ROW_NUMBER 함수를 사용할 수 없는 경우
- 서브쿼리가 재귀 집합을 만들고 요소 수를 COUNT 함수로 셈
- 기본키인 student_id를 비교 키로 사용하므로, 재귀 집합의 요소가 한 개씩 상승

```sql
SELECT student_id,
       (SELECT COUNT(*)
        FROM Weights W2
        WHERE W2.student_id <= W1.student_id) AS seq
FROM Weights W1;
```

> **QUERY PLAN**  
> Seq Scan on weights w1 (cost=0.00..38689.78 rows=1510 width=20)  
> &nbsp;&nbsp;&nbsp;&nbsp; SubPlan 1  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Aggregate (cost=25.60..25.61 rows=1 width=0)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Bitmap Heap Scan on weights w2 (> cost=8.05..24.34 rows=503 width=0)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; Recheck Cond: (student_id <= w1.student_id)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Bitmap Index Scan on weights_pkey (cost=0.00..7.92 rows=503 width=0)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; Index Cond: (student_id <= w1.student_id)  

> - Sequential Scan이 2회 발생

### 2. 기본 키가 여러 개의 필드로 구성되는 경우
기본 키가 두개인 Weights2 테이블에 순번 붙이기 [`SQL2`](http://sqlfiddle.com/#!15/1c027/3)

|<U>class</U>|<U>student_id</U>|weights|    |<U>class</U>|<U>student_id</U>|seq|
|:----------:|:---------------:|:-----:|:--:|:----------:|:---------------:|:-:|
|1           |100              |   50  |    |1           |100              | 1 |
|1           |101              |   55  |    |1           |101              | 2 |
|1           |102              |   56  |    |1           |102              | 3 |
|2           |100              |   60  | &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;→&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; |2           |100              | 4 |
|2           |101              |   72  |    |2           |101              | 5 |
|2           |102              |   73  |    |2           |102              | 6 |
|2           |103              |   73  |    |2           |103              | 7 |

#### 2-1) 윈도우 함수 사용
- ORDER BY의 키에 필드를 추가

```sql
SELECT class, student_id,
       ROW_NUMBER() OVER (ORDER BY class, student_id) AS seq
FROM Weights2;
```

#### 2-2) 상관 서브쿼리 사용
- 다중 필드 비교(복합적인 필드를 하나의 값으로 연결하고 한꺼번에 비교)를 사용
    - 장점
        - 필드 자료형을 원하는대로 지정 가능 → 숫자&문자열, 문자열&숫자
        - 암묵적인 자료형 변환이 발생하지 않아 기본 키 인덱스도 사용 가능
        - 필드가 3개 이상일 때도 간단하게 확장 가능

```sql
SELECT class, student_id,
       (SELECT COUNT(*)
        FROM Weights2 W2
        WHERE (W2.class, W2.student_id)
                <= (W1.class, W1.student_id)) AS seq
FROM Weights2 W1;
```

### 3. 그룹마다 순번을 붙이는 경우
Weights2 테이블에 학급마다 순번 붙이기 [`SQL3`](http://sqlfiddle.com/#!17/1c027/2)  
(테이블을 그룹으로 나누고 그룹마다 내부 레코드에 순번을 붙이기)

|<U>class</U>|<U>student_id</U>|weights|    |<U>class</U>|<U>student_id</U>|seq|
|:----------:|:---------------:|:-----:|:--:|:----------:|:---------------:|:-:|
|1           |100              |   50  |    |1           |100              | 1 |
|1           |101              |   55  |    |1           |101              | 2 |
|1           |102              |   56  |    |1           |102              | 3 |
|2           |100              |   60  | &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;→&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; |2           |100              | 1 |
|2           |101              |   72  |    |2           |101              | 2 |
|2           |102              |   73  |    |2           |102              | 3 |
|2           |103              |   73  |    |2           |103              | 4 |

#### 3-1) 윈도우 함수 사용
- class 필드에 PARTITION BY 적용

```sql
SELECT class, student_id,
       ROW_NUMBER() OVER (PARTITION BY class ORDER BY student_id) AS seq
FROM Weights2;
```

#### 3-2) 상관 서브쿼리 사용

```sql
SELECT class, student_id,
       (SELECT COUNT(*)
        FROM Weights2 W2
        WHERE W2.class = W1.class
        AND W2.student_id <= W1.student_id) AS seq
FROM Weights2 W1;
```

### 4. 순번과 갱신
Weights3 테이블에 seq(순번) 필드를 UPDATE (채우기) [`SQL4`](http://sqlfiddle.com/#!15/f14dd5/1)

|<U>class</U>|<U>student_id</U>|weights|  seq |    |<U>class</U>|<U>student_id</U>|weights|seq|
|:----------:|:---------------:|:-----:|:----:|:--:|:----------:|:---------------:|:-----:|:-:|
|1           |100              |   50  | null |    |1           |100              |   50  | 1 |
|1           |101              |   55  | null |    |1           |101              |   55  | 2 |
|1           |102              |   56  | null |    |1           |102              |   56  | 3 |
|2           |100              |   60  | null | &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;→&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|2           |100              |   60  | 1 |
|2           |101              |   72  | null |    |2           |101              |   72  | 2 |
|2           |102              |   73  | null |    |2           |102              |   73  | 3 |
|2           |103              |   73  | null |    |2           |103              |   73  | 4 |

#### 4-1) 윈도우 함수 사용
- 순번 할당 쿼리를 서브쿼리(SeqTbl)와 함께 사용하여 SET 구에 넣음

```sql
UPDATE Weights3
   SET seq = (SELECT seq
              FROM (SELECT class, student_id,
                           ROW_NUMBER()
                              OVER (PARTITION BY class
                                        ORDER BY student_id) AS seq
                    FROM Weights3) SeqTbl
              WHERE Weights3.class=SeqTbl.class
              AND Weights3.student_id = SeqTbl.student_id);
```

#### 4-2) 상관 서브쿼리 사용
- 순번 할당 쿼리를 그대로 SET 구에 넣음

```sql
UPDATE Weights3
   SET seq = (SELECT COUNT(*)
              FROM Weights3 W2
              WHERE W2.class = Weights3.class
              AND W2.student_id <= Weights3.student_id);
```

<br>

## 24강 레코드에 순번 붙이기 응용
### 1. 중앙값 구하기
- **중앙값(median)**: 숫자를 정렬하고 양 끝에서부터 수를 세는 경우 정중앙에 오는 값
    - 단순 평균(mean)과 다르게 아웃라이어에 영향을 받지 않음
        - 아웃라이어: 숫자 집합 내부의 중앙에서 극단적으로 벗어나 있는 예외적인 값  
        ex) {1, 0, 2, 1, 3, 9999} 숫자 집합에서 9999가 아웃라이어
    - 데이터가 홀수인 경우 중앙의 값을 그대로 사용, 짝수인 경우 중앙에 있는 두 개 값의 평균을 사용

Weights5 테이블은 데이터 개수가 **홀수** (중앙값: 60)

|<U>student_id</U>|weight|&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;→&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|avg|
|:---------------:|:----:|:----:|:-:|
|A100             |	 50  |      |60 |
|A101             |  55  |      |   |
|A124             |  55  |      |   |
|B343             |  60  | 중앙값 |   |
|B346             |  72  |      |   |
|C563             |  72  |      |   |
|C345             |  72  |      |   |

Weights6 테이블은 데이터 개수가 **짝수** (중앙값: 66)

|<U>student_id</U>|weight|&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;→&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|avg|
|:---------------:|:----:|:----:|:-:|
|A100             |	 50  |      |66 |
|A101             |  55  |      |   |
|A124             |  55  |      |   |
|B343             |  60  | 중앙값 |   |
|B346             |  72  | 중앙값 |   |
|C563             |  72  |      |   |
|C345             |  72  |      |   |
|C478             |  90  |      |   |

<br>

#### 1-1) 집합 지향적 방법
- 테이블을 상위 집합과 하위 집합으로 분할하고 공통 부분을 검색 [`SQL5`](http://sqlfiddle.com/#!15/a60c4e/5)
- 단점: 코드가 복잡하고 성능이 좋지 않음

```sql
SELECT AVG(weight)
FROM (SELECT W1.weight
      FROM Weights W1, Weights W2
      GROUP BY W1.weight
      -- 포인트: HAVING 구
      HAVING SUM(CASE WHEN W2.weight >= W1.weight THEN 1 ELSE 0 END)
             >= COUNT(*)/2
              -- S1(하위 집합)의 조건
      AND SUM(CASE WHEN W2.weight <= W1.weight THEN 1 ELSE 0 END)
          >= COUNT(*)/2) TMP;
          -- S2(상위 집합)의 조건
```

> **QUERY PLAN**  
> Aggregate (cost=68463.48..68463.49 rows=1 width=4)  
> &nbsp;&nbsp;&nbsp;&nbsp; -> HashAggregate (cost=68456.98..68460.98 rows=200 width=8)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; Filter: ((sum(CASE WHEN (w2.weight >= w1.weight)   
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; THEN 1 ELSE 0 END) >= (count(\*) / 2))   
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; AND (sum(CASE WHEN (w2.weight <= w1.weight)   
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; THEN 1 ELSE 0 END) >= (> count(*) / 2)))  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Nested Loop (cost=0.00..28555.22 rows=2280100 width=8)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Seq Scan on weights5 w1 (cost=0.00..25.10 rows=1510 width=4)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Materialize (cost=0.00..32.65 rows=1510 width=4)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Seq Scan on weights5 w2 (cost=0.00..25.10 rows=1510 width=4)

> - Nested Loop 발생 (결합은 비용이 높고 불안정함)

<br>

#### 1-2) 절차 지향적 방법 1
- 자연수의 특징을 활용하여 양쪽 끝부터 숫자 세기 [`SQL6`](http://sqlfiddle.com/#!17/a60c4e/5)
- 홀수와 짝수일 경우의 조건 분기를 IN 연산자로 한꺼번에 수행
    - 홀수의 경우에는 `hi=lo`로 중심점이 하나만 존재
    - 짝수의 경우에는 `hi=lo+1`, `hi=lo-1`
- 주의할점
    1. 순번을 붙일 때는 반드시 `ROW_NUMBER` 함수를 사용해야 레코드 집합에 자연수의 집합을 할당해서 연속성과 유일성을 가질 수 있음
        - 비슷한 함수인 `RANK`, `DENSE_RANK`는 숫번을 붙일 때 중복이 발생할 수 있음
    2. `ORDER BY`의 정렬 키에 weight 필드와 함께 기본키인 student_id도 포함해야 함 (포함하지 않으면 결과가 NULL이 될 수도 있음)
        - weight로만 지정하면 hi와 lo를 산출할 때 같은 체중의 학생들이 언제나 같은 순서로 정렬된다는 보장이 없음

```sql
SELECT AVG(weight) AS median
FROM (SELECT weight,
             ROW_NUMBER() OVER (ORDER BY weight ASC, student_id ASC) AS hi,
             ROW_NUMBER() OVER (ORDER BY weight DESC, student_id DESC) AS lo
      FROM Weights) TMP
WHERE hi IN (lo, lo +1, lo -1);
```

> **QUERY PLAN**  
> Aggregate (cost=290.56..290.57 rows=1 width=32)  
> &nbsp;&nbsp;&nbsp;&nbsp; -> Subquery Scan on tmp (cost=223.78..290.50 rows=23 width=4)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; Filter: ((tmp.hi = tmp.lo)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; OR (tmp.hi = (tmp.lo + 1))  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; OR (tmp.hi = (tmp.lo - 1)))  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> WindowAgg (cost=223.78..255.18 rows=1570 width=40)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Sort (cost=223.78..227.70 rows=1570 width=32)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; Sort Key: weights5.weight DESC, weights5.student_id DESC  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> WindowAgg (cost=109.04..140.44 rows=1570 width=32)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Sort (cost=109.04..112.96 rows=1570 width=24)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp; Sort Key: weights5.weight, weights5.student_id  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp; -> Seq Scan on weights5 (cost=0.00..25.70 rows=1570 width=24)  

> - weight 테이블에 대한 접근이 1회로 감소
> - 결합이 사용되지 않음
> - 정렬이 2회로 증가 (ROW_NUMBER에서 사용하는 정렬 순서가 오름차순과 내림차순으로 다르기 때문)
>     - 결합을 제거한 대신 정렬이 1회 늘어났지만, 이러한 트레이드오프는 테이블이 클 경우 훨씬 이득

<br>

#### 1-3) 절차 지향적 방법 2
- 성능적으로 가장 좋은 방법 [`SQL7`](http://sqlfiddle.com/#!17/a60c4e/7)
- `ROW_NUMBER` 함수로 구한 순번을 2배 한 다음 `COUNT(*)` 값을 구해 0~2 사이인 값을 찾아 평균을 구함
- `COUNT OVER` 구문에 `ORDER BY`가 없음
    - 옵티마이저는 단순히 ROW_NUMBER 함수의 OVER 구문의 ORDER BY만 정렬로 계획

<details>
    <summary> 세부 과정 살펴보기 </summary>

<br>

|weight|ROW_NUMBER()|2*ROW_NUMBER()|COUNT(*)|diff|
|:----:|:----------:|:------------:|:------:|:--:|
|  50  |      1     |       2      |    7   | -5 |
|  55  |      2     |       4      |    7   | -3 |
|  55  |      3     |       6      |    7   | -1 |
|  60  |      4     |       8      |    7   |  1 |
|  72  |      5     |      10      |    7   |  3 |
|  72  |      6     |      12      |    7   |  5 |
|  72  |      7     |      14      |    7   |  7 |

</details>

```sql
SELECT AVG(weight)
FROM (SELECT weight,
             2*ROW_NUMBER() OVER (ORDER BY weight)
               - COUNT(*) OVER() AS diff
      FROM Weights) TMP
WHERE diff BETWEEN 0 AND 2;
```

> **QUERY PLAN**  
> Aggregate (cost=187.56..187.57 rows=1 width=32)  
> &nbsp;&nbsp;&nbsp;&nbsp; -> Subquery Scan on tmp (cost=109.04..187.54 rows=8 width=4)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; Filter: ((tmp.diff >= 0) AND (tmp.diff <= 2))  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> WindowAgg (cost=109.04..163.99 rows=1570 width=12)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> WindowAgg (cost=109.04..136.51 rows=1570 width=12)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Sort (cost=109.04..112.96 rows=1570 width=4)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; Sort Key: weights5.weight  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Seq Scan on weights5 (cost=0.00..25.70 rows=1570 width=4)  

> - 정렬이 1회로 줄어듦
> - 벤더의 독자적인 확장 기능에서 제공하는 중앙값 함수를 제외하면, SQL 표준으로 가장 빠른 방법임

### 2. 순번을 사용한 테이블 분할
- 테이블을 여러 개의 그룹으로 분할
- 테이블에 존재하지 않는 수열을 찾아 그룹화하기

|num|&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;→&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|gap_start|~|gap_end|
|:-:|:--:|:-------:|-|:-----:|
|1  |    |    2    |~|   2   |
|3  |    |    5    |~|   6   |
|4  |    |    10   |~|   11  |
|7  |    |         | |       |
|8  |    |         | |       |
|9  |    |         | |       |
|12 |    |         | |       |

#### 2-1) 집합 지향적 방법 (집합의 경계선)
- 레코드 단위가 아닌 집합 단위로 생각해보기 [`SQL8`](http://sqlfiddle.com/#!17/cd63e/1)
- 특정 레코드의 값 `N1.num` 보다 큰 숫자의 집합을 조건 `ON N2.num > N1.num` 으로 지정
- 세부 과정에서 S1, S3, S6을 주목
    - 특정한 숫자 `N1.num`의 다음의 숫자 `N1.num+1`가 `MIN(N2.num)`과 일치하지 않으면 단절임을 의미
    - `N1.num` 다음의 숫자가 비어있는 숫자의 시작값(gap_start)
    - `N2.num` 바로 앞에 있는 숫자가 종료값(gat_end)

<details>
    <summary> 세부 과정 살펴보기 </summary>

<br>

|    |N1.num|N2.num|                |
|:--:|:----:|:----:|:--------------:|
| S1 |  1   |  3   | 단절 있음(1+1≠3) |
|    |  1   |  4   |                |
|    |  1   |  7   |                |
|    |  1   |  8   |                |
|    |  1   |  9   |                |
|    |  1   |  12  |                |
| S2 |  3   |  4   | 단절 없음(3+1=4) |
|    |  3   |  7   |                |
|    |  3   |  8   |                |
|    |  3   |  9   |                |
|    |  3   |  12  |                |
| S3 |  4   |  7   | 단절 있음(4+1≠7) |
|    |  4   |  8   |                |
|    |  4   |  9   |                |
|    |  4   |  12  |                |
| S4 |  7   |  8   | 단절 없음(7+1=8) |
|    |  7   |  9   |                |
|    |  7   |  12  |                |
| S5 |  8   |  9   | 단절 없음(8+1=9) |
|    |  8   |  12  |                |
| S6 |  9   |  12  | 단절 있음(9+1≠12)|

</details>

```sql
SELECT (N1.num+1) AS gap_start,
       '~',
       (MIN(N2.num) - 1) AS gap_end
FROM Numbers N1 INNER JOIN Numbers N2
ON N2.num > N1.num
GROUP BY N1.num
HAVING (N1.num + 1) < MIN(N2.num);
```
> **QUERY PLAN**  
> GroupAggregate (cost=0.31..76430.40 rows=2550 width=44)  
> &nbsp;&nbsp;&nbsp;&nbsp; Group Key: n1.num  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; Filter: ((n1.num + 1) < min(n2.num))  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Nested Loop (cost=0.31..60135.90 rows=2167500 width=8)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Index Only Scan using numbers_pkey on numbers n1 (cost=0.15..86.41 rows=2550 width=4)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Index Only Scan using numbers_pkey on numbers n2 (cost=0.15..15.05 rows=850 width=4)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; Index Cond: (num > n1.num)  

> - 집합 지향적인 방법은 자기 결합을 꼭 사용해야 하므로 Nested Loop 결합 발생
>     - 결합을 사용하는 쿼리는 비용이 높고 실행 계획 변동 위험이 존재

#### 2-2) 절차 지향적 방법 (다음 레코드와 비교)
- 레코드의 순서를 활용하는 방식 [`SQL9`](http://sqlfiddle.com/#!17/cd63e/2)
    - 현재 레코드와 다음 레코드의 숫자 차이를 비교 후 차이가 1이 아니라면 사이에 비어있는 숫자가 존재
- 윈도우 함수로 현재 레코드의 다음 레코드를 구하고 두 레코드의 숫자 차이를 diff 필드에 저장해 연산

<details>
    <summary> 세부 과정 살펴보기 </summary>

<br>

|num|next_num|
|:-:|:------:|
| 1 |    3   |
| 3 |    4   |
| 4 |    7   |
| 7 |    8   |
| 8 |    9   |
| 9 |   12   |
| 12|        |

- 윈도우 함수 부분의 실행 결과
- num과 next_num 차이가 1이 아니라면 사이에 비어있는 숫자가 존재
    - `diff <> 1`이 조건

</details>

```sql
SELECT NUM+1 AS gap_start,
       '~',
       (num+diff-1) AS gap_end
FROM (SELECT num,
             MAX(num)
               OVER(ORDER BY num
                        ROWS BETWEEN 1 FOLLOWING
                                 AND 1 FOLLOWING) - num
      FROM Numbers) TMP(num, diff)
WHERE diff<>1;
```

> **QUERY PLAN**  
> Subquery Scan on tmp (cost=0.15..181.93 rows=2537 width=40)  
> &nbsp;&nbsp;&nbsp;&nbsp; Filter: (tmp.diff <> 1)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> WindowAgg (cost=0.15..131.03 rows=2550 width=8)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Index Only Scan using numbers_pkey on numbers (cost=0.15..86.41 rows=2550 width=4)  

> - 테이블에 접근이 한 번만 발생
> - 윈도우 함수에서 정렬이 실행됨
> - 결합을 사용하지 않기 때문에 성능이 굉장히 안정적
> - 집합 지향적인 방법에서는 데이터베이스 내부에서 반복이 사용되지만, 절차 지향적인 방법에서는 반복이 사용되지 않음

### 3. 테이블에 존재하는 시퀀스 구하기
- 테이블을 여러 개의 그룹으로 분할
- 테이블에 존재하는 수열을 찾아 그룹화하기

|num|&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;→&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|low |~|high|
|:-:|:----:|:--:|-|:--:|
|1  |      | 1  |~|  1 |
|3  |      | 3  |~|  4 |
|4  |      | 7  |~|  9 |
|7  |      | 12 |~| 12 |
|8  |      |    | |    |
|9  |      |    | |    |
|12 |      |    | |    |

#### 3-1) 집합 지향적인 방법 (집합의 경계선)
- 존재하지 않는 시퀀스를 구하는 것보다 존재하는 시퀀스를 구하는 것이 훨씬 간단 [`SQL10`](http://sqlfiddle.com/#!17/cd63e/4)
    - MAX/MIN 함수를 사용해서 시퀀스의 경계를 직접적으로 구할 수 있기 때문
- 자기 결합으로 num 필드의 조합을 만들고 최대값과 최소값으로 집합의 경계를 구하는 방식 (이전과 크게 다르지 않음)

```sql
SELECT MIN(num) AS low,
       '~',
       MAX(num) AS high
FROM (SELECT N1.num,
             COUNT(N2.num) - N1.num
      FROM Numbers N1 INNER JOIN Numbers N2
      ON N2.num <= N1.num
      GROUP BY N1.num) N(num, gp)
GROUP BY gp;
```

> **QUERY PLAN**  
> GroupAggregate (cost=71175.06..71202.56 rows=200 width=48)  
> &nbsp;&nbsp;&nbsp;&nbsp; Group Key: n.gp  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Sort (cost=71175.06..71181.44 rows=2550 width=12)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; Sort Key: n.gp  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Subquery Scan on n (cost=0.31..71030.78 rows=2550 width=12)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> GroupAggregate (cost=0.31..71005.28 rows=2550 width=12)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; Group Key: n1.num  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Nested Loop (cost=0.31..60135.90 rows=2167500 width=8)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Index Only Scan using numbers_pkey on numbers n1 (cost=0.15..86.41 rows=2550 width=4)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Index Only Scan using numbers_pkey on numbers n2 (cost=0.15..15.05 rows=850 width=4)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; Index Cond: (num <= n1.num)  

> - 자기 결합을 수행
> - 극치 함수(MAX/MIN)로 집약을 수행 (2개의 HashAggregate)
>     - 최근의 DBMS는 집약 함수 또는 극치 함수를 사용할 때 정렬이 아니라 해시를 사용하는 알고리즘을 활용(4장에서 설명)

#### 3-2) 절차 지향형 방법 (다음 레코드 하나와 비교)
- 현재 레코드와 전후의 레코드를 비교 [`SQL11`](http://sqlfiddle.com/#!17/cd63e/7)
    - 기본적인 방식은 이전과 비슷

```sql
SELECT low, high
FROM (SELECT low,
             CASE WHEN high IS NULL
                  THEN MIN(high)
                         OVER (ORDER BY seq
                               ROWS BETWEEN CURRENT ROW
                                              AND UNBOUNDED FOLLOWING)
                  ELSE high END AS high
      FROM (SELECT CASE WHEN COALESCE(prev_diff, 0) <> 1
                        THEN num ELSE NULL END AS low,
                   CASE WHEN COALESCE(next_diff, 0) <> 1
                        THEN num ELSE NULL END AS high,
                   seq
            FROM (SELECT num,
                         MAX(num)
                         OVER(ORDER BY num
                              ROWS BETWEEN 1 FOLLOWING
                                       AND 1 FOLLOWING) - num AS next_diff,
                         num - MAX(num)
                                 OVER(ORDER BY num
                                      ROWS BETWEEN 1 PRECEDING
                                               AND 1 PRECEDING) AS prev_diff,
                         ROW_NUMBER() OVER (ORDER BY num) AS seq
                  FROM Numbers) TMP1) TMP2) TMP3
WHERE low IS NOT NULL;
```

<details>
    <summary> 세부 과정 살펴보기 </summary>

<br>

[`SQL_D`](http://sqlfiddle.com/#!17/cd63e/17)

|num|next_diff|prev_diff|seq|
|:-:|:-------:|:-------:|:-:|
| 1 |   2     |   null  | 1 |
| 3 |   1     |    2    | 2 |
| 4 |   3     |    1    | 3 |
| 7 |   1     |    3    | 4 |
| 8 |   1     |    1    | 5 |
| 9 |   3     |    1    | 6 |
| 12|  null   |    3    | 7 |

- **TMP1**
    - 현재 레코드와 전후의 레코드에 있는 num의 차이를 구함
    - `next_diff`가 1보다 크면 현재 레코드와 다음 레코드 사이에 비어있는 부분이 존재한다는 뜻
    - `prev_diff`가 1보다 크면 현재 레코드와 이전 레코드 사이에 비어있는 부분이 존재한다는 뜻

| low  | high |seq|
|:----:|:----:|:-:|
|  1   |  1   | 1 |
|  3   | null | 2 |
| null |  4   | 3 |
|  7   | null | 4 |
| null | null | 5 |
| null |  9   | 6 |
|  12  |  12  | 7 |

- **TMP2**
    - `next_diff`와 `prev_diff`의 성질을 사용해서 시퀀스의 단절이 되는 양쪽 지점의 num을 구하기
        - `next_diff`와 `prev_diff`가 1인지를 확인해 경계값을 확인
    - low 필드와 high 필드는 각 시퀀스의 양쪽 지점이 되는 값을 나타냄
    - low 필드와 high 필드가 존재하지 않는 시퀀스는 제거

| low  | high |
|:----:|:----:|
|  1   |  1   |
|  3   |  4   |
| null |  4   |
|  7   |  9   |
| null |  9   |
| null |  9   |
|  12  |  12  |

- **TMP3**
    - `low IS NOT NULL`로 불필요한 레코드를 제거

| low  | high |
|:----:|:----:|
|  1   |  1   |
|  3   |  4   |
|  7   |  9   |
|  12  |  12  |

- **최종결과**

</details>

> **QUERY PLAN - PostgreSQL**   
> Subquery Scan on tmp3 (cost=383.69..479.31 rows=2537 width=8)  
> &nbsp;&nbsp;&nbsp;&nbsp; Filter: (tmp3.low IS NOT NULL)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> WindowAgg (cost=383.69..453.81 rows=2550 width=16)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Sort (cost=383.69..390.06 rows=2550 width=20)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; Sort Key: tmp1.seq  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Subquery Scan on tmp1 (cost=0.15..239.41 rows=2550 width=20)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> WindowAgg (cost=0.15..213.91 rows=2550 width=20)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> WindowAgg (cost=0.15..162.91 rows=2550 width=12)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> WindowAgg (cost=0.15..124.66 rows=2550 width=8)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Index Only Scan using numbers_pkey on numbers (cost=0.15..86.41 rows=2550 width=4)  

> **QUERY PLAN - Oracle**  
> \-------------------------------------------------------------------------------------------------  
> |&nbsp;Id&nbsp;&nbsp;|
&nbsp;Operation&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|
&nbsp;Name&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|
&nbsp;Rows&nbsp;|
&nbsp;Bytes&nbsp;&nbsp;|
&nbsp;Cost&nbsp;&nbsp;|
&nbsp;Time&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|  
> \-------------------------------------------------------------------------------------------------  
> |&nbsp;&nbsp;&nbsp;0&nbsp;|
&nbsp;SELECT&nbsp;STATEMENT&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|
&nbsp;&nbsp;&nbsp;&nbsp;7&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|
&nbsp;&nbsp;&nbsp;182&nbsp;&nbsp;&nbsp;|
&nbsp;&nbsp;&nbsp;&nbsp;3&nbsp;&nbsp;&nbsp;&nbsp;|
&nbsp;00:00:01&nbsp;|  
> |&nbsp;*&nbsp;1&nbsp;|
&nbsp;&nbsp;&nbsp;VIEW&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|
&nbsp;&nbsp;&nbsp;&nbsp;7&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|
&nbsp;&nbsp;&nbsp;182&nbsp;&nbsp;&nbsp;|
&nbsp;&nbsp;&nbsp;&nbsp;3&nbsp;&nbsp;&nbsp;&nbsp;|
&nbsp;00:00:01&nbsp;|  
> |&nbsp;&nbsp;&nbsp;2&nbsp;|
&nbsp;&nbsp;&nbsp;&nbsp;WINDOW&nbsp;SORT&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|
&nbsp;&nbsp;&nbsp;&nbsp;7&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|
&nbsp;&nbsp;&nbsp;364&nbsp;&nbsp;&nbsp;|
&nbsp;&nbsp;&nbsp;&nbsp;3&nbsp;&nbsp;&nbsp;&nbsp;|
&nbsp;00:00:01&nbsp;|  
> |&nbsp;&nbsp;&nbsp;3&nbsp;|
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;VIEW&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|
&nbsp;&nbsp;&nbsp;&nbsp;7&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|
&nbsp;&nbsp;&nbsp;364&nbsp;&nbsp;&nbsp;|
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2&nbsp;&nbsp;&nbsp;|
&nbsp;00:00:01&nbsp;|  
> |&nbsp;&nbsp;&nbsp;4&nbsp;|
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;WINDOW&nbsp;BUFFER&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|
&nbsp;&nbsp;&nbsp;&nbsp;7&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|
&nbsp;&nbsp;&nbsp;&nbsp;91&nbsp;&nbsp;&nbsp;&nbsp;|
&nbsp;&nbsp;&nbsp;&nbsp;2&nbsp;&nbsp;&nbsp;|
&nbsp;00:00:01&nbsp;|  
> |&nbsp;&nbsp;&nbsp;5&nbsp;|
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;INDEX&nbsp;FULL&nbsp;SCAN&nbsp;&nbsp;&nbsp;&nbsp;|
&nbsp;SYS_C007067&nbsp;|
&nbsp;&nbsp;&nbsp;&nbsp;7&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|
&nbsp;&nbsp;&nbsp;&nbsp;91&nbsp;&nbsp;&nbsp;&nbsp;|
&nbsp;&nbsp;&nbsp;&nbsp;2&nbsp;&nbsp;&nbsp;|
&nbsp;00:00:01&nbsp;|  
> \-------------------------------------------------------------------------------------------------  

> - TMP1과 TMP3에서 윈도우 함수를 사용하므로 정렬도 2회 발생
> - PostgreSQL에서 Subquery Scan on tmp1 처럼 서브쿼리 스캔이 Tmp1과 Tmp2에서 발생  
>   (오라클은 VIEW가 서브쿼리를 의미)
>    - 서브쿼리의 결과를 일시 테이블에 전개하는 것으로, 일시 테이블의 크기가 크면 비용이 높아질 가능성이 존재  
>      (오라클도 중간 결과를 메모리에 유지하므로 결과가 크면 저장소를 사용)
> - 이 쿼리의 성능은 서브쿼리의 크기에도 의존하므로 집합 지향 쿼리에 비해 좋다고 단언할 수 없음



## 25강 시퀀스 객체, IDENTITY 필드, 채번 테이블
- 시퀀스 객체, IDENTITY 필드, 채번 테이블은 순번을 다루는 기능들
    - 시퀀스 객체: MySQL에서 지원하지 않음
    - IDENTITY 필드: 오라클에서 지원하지 않음
- 채번 테이블보다 IDENTITY 필드를, IDENTITY 필드보다 시퀀스 객체를 사용
- 이 책에서는 셋다 모두 사용하지 않기를 권함

### 1. 시퀀스 객체
### 2. IDENTITY 필드
### 3. 채번 테이블
