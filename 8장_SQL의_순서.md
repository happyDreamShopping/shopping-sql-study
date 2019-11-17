# 8장 SQL의 순서

## 23강 레코드에 순번 붙이기
### 1. 기본 키가 한 개의 필드일 경우
기본 키가 한개인 Weights 테이블에 순번 붙이기 [`SQL1`](http://sqlfiddle.com/#!15/45d46/3)

|<U>student_id</U>|weight|    |<U>student_id</U>|seq|
|:---------------:|:----:|:--:|:---------------:|:-:|
|A100             |	50  |    |A100             | 1 |
|A101             |  55  |    |A101             | 2 |
|A124             |  55  |    |A124             | 3 |
|A346             |  60  | &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;→&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; |A346             | 4 |
|B343             |  72  |    |B343             | 5 |
|C345             |  72  |    |C345             | 6 |
|C563             |  72  |    |C563             | 7 |

**윈도우 함수 사용**

```sql
SELECT student_id,
       ROW_NUMBER() OVER (ORDER BY student_id) AS seq
FROM Weights;
```
- Index Only Scan으로 테이블에 직접적인 접근 회피

> **QUERY PLAN**  
> WindowAgg (cost=0.15..93.45 rows=1510 width=20)  
> &nbsp;&nbsp;&nbsp;&nbsp; -> Index Only Scan using weights_pkey on weights (cost=0.15..70.80 rows=1510 width=20)

**상관 서브쿼리 사용** (MySQL처럼 ROW_NUMBER 함수를 사용할 수 없는 경우)❓
- 서브쿼리가 재귀 집합을 만들고 요소 수를 COUNT 함수로 셈
- 기본키인 student_id를 비교 키로 사용하므로, 재귀 집합의 요소가 한 개씩 상승

```sql
SELECT student_id,
       (SELECT COUNT(*)
        FROM Weights W2
        WHERE W2.student_id <= W1.student_id) AS seq
FROM Weights W1;
```
- Sequence Scan이 2회 발생

> **QUERY PLAN (Fiddle)** ❓  
> Seq Scan on weights w1 (cost=0.00..38689.78 rows=1510 width=20)  
> &nbsp;&nbsp;&nbsp;&nbsp; SubPlan 1  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Aggregate (cost=25.60..25.61 rows=1 width=0)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Bitmap Heap Scan on weights w2 (> cost=8.05..24.34 rows=503 width=0)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; Recheck Cond: (student_id <= w1.student_id)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Bitmap Index Scan on weights_pkey (cost=0.00..7.92 rows=503 width=0)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; Index Cond: (student_id <= w1.student_id)  

> **QUERY PLAN (책)**    
> Seq Scan on weights w1 (cost=0.00..8.79 rows=7 width=5)  
> &nbsp;&nbsp;&nbsp;&nbsp; Subplan 1  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Aggregate (cost=1.09..1.10 rows=1 width=0)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; -> Seq Scan on weights w2 (cost=0.00..1.09 rows=2 width=0)  
> &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; Filter: (student_id <= w1.student_id)


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

**윈도우 함수 사용**
- ORDER BY의 키에 필드를 추가

```sql
SELECT class, student_id,
       ROW_NUMBER() OVER (ORDER BY class, student_id) AS seq
FROM Weights2;
```

**상관 서브쿼리 사용**
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

**윈도우 함수 사용**
- class 필드에 PARTITION BY 적용

```sql
SELECT class, student_id,
       ROW_NUMBER() OVER (PARTITION BY class ORDER BY student_id) AS seq
FROM Weights2;
```

**상관 서브쿼리 사용**

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

**윈도우 함수 사용**
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

**상관 서브쿼리 사용**
- 순번 할당 쿼리를 그대로 SET 구에 넣음

```sql
UPDATE Weights3
   SET seq = (SELECT COUNT(*)
              FROM Weights3 W2
              WHERE W2.class = Weights3.class
              AND W2.student_id <= Weights3.student_id);
```

## 24강 레코드에 순번 붙이기 응용
### 1. 중앙값 구하기
데이터 개수가 **홀수**인 Weights5 테이블 (중앙값: 60) [`SQL5`](http://sqlfiddle.com/#!15/62b20/2)

|<U>student_id</U>|weight|      |
|:---------------:|:----:|:----:|
|A100             |	50  |      |
|A101             |  55  |      |
|A124             |  55  |      |
|B343             |  60  | 중앙값 |
|B346             |  72  |      |
|C563             |  72  |      |
|C345             |  72  |      |

|avg|
|---|
|60 |

데이터 개수가 **짝수**인 Weights6 테이블 (중앙값: 66) [`SQL6`](http://sqlfiddle.com/#!9/ac2b00/1)

|<U>student_id</U>|weight|      |
|:---------------:|:----:|:----:|
|A100             |	50  |      |
|A101             |  55  |      |
|A124             |  55  |      |
|B343             |  60  | 중앙값 |
|B346             |  72  | 중앙값 |
|C563             |  72  |      |
|C345             |  72  |      |
|C478             |  90  |      |

|avg|
|---|
|66 |

```sql
SELECT AVG(weight)
FROM (SELECT W1.weight
      FROM Weights W1, Weights W2
      GROUP BY W1.weight
      -- S1(하위 집합)의 조건)
      HAVING SUM(CASE WHEN W2.weight >= W1.weight THEN 1 ELSE 0 END)
              >= COUNT(*)/2
              -- S2(상위 집합)의 조건
      AND SUM(CASE WHEN W2.weight <= W1.weight THEN 1 ELSE 0 END)
            >= COUNT(*)/2) TMP;
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

[`SQL7`](http://sqlfiddle.com/#!17/a60c4e/5)

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

[`SQL8`](http://sqlfiddle.com/#!17/a60c4e/7)
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

### 2. 순번을 사용한 테이블 분할

|num|
|:-:|
|1  |
|3  |
|4  |
|7  |
|8  |
|9  |
|12 |

|gap_start|~|gap_end|
|:-------:|-|:-----:|
|    2    |~|   2   |
|    5    |~|   6   |
|    10   |~|   11  |

[`SQL9`](http://sqlfiddle.com/#!17/cd63e/1)
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

[`SQL10`](http://sqlfiddle.com/#!17/cd63e/2)
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

### 3. 테이블에 존재하는 시퀀스 구하기

|num|
|:-:|
|1  |
|3  |
|4  |
|7  |
|8  |
|9  |
|12 |

|low |~|high|
|:--:|-|:--:|
| 1  |~|  1 |
| 3  |~|  4 |
| 7  |~|  9 |
| 12 |~| 12 |

[`SQL11`](http://sqlfiddle.com/#!17/cd63e/4)

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

[`SQL12`](http://sqlfiddle.com/#!17/cd63e/7)
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

[`SQL_D`](http://sqlfiddle.com/#!17/cd63e/17)

TMP1

|num|next_diff|prev_diff|seq|
|:-:|:-------:|:-------:|:-:|
| 1 |   2     |   null  | 1 |
| 3 |   1     |    2    | 2 |
| 4 |   3     |    1    | 3 |
| 7 |   1     |    3    | 4 |
| 8 |   1     |    1    | 5 |
| 9 |   3     |    1    | 6 |
| 12|  null   |    3    | 7 |

TMP2

| low  | high |seq|
|:----:|:----:|:-:|
|  1   |  1   | 1 |
|  3   | null | 2 |
| null |  4   | 3 |
|  7   | null | 4 |
| null | null | 5 |
| null |  9   | 6 |
|  12  |  12  | 7 |

TMP3

| low  | high |
|:----:|:----:|
|  1   |  1   |
|  3   |  4   |
| null |  4   |
|  7   |  9   |
| null |  9   |
| null |  9   |
|  12  |  12  |

최종

| low  | high |
|:----:|:----:|
|  1   |  1   |
|  3   |  4   |
|  7   |  9   |
|  12  |  12  |

</details>
</br>

> **QUERY PLAN**  
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

## 25강 시퀀스 객체, IDENTITY 필드, 채번 테이블
### 1. 시퀀스 객체
### 2. IDENTITY 필드
### 3. 채번 테이블
