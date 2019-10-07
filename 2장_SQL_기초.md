# 2장 SQL 기초

## `SELECT` 구문
```SQL
SELECT -- 조회 결과로 보고싶은 필드
  department,
  name,
  age,
  status
FROM employee -- 조회할 대상이 있는 테이블
WHERE -- 조회 시 조건 지정
  name IS NOT NULL AND -- NULL 은 비교 연산자(=, !=) 대신 IS 로 비교
  department != ' ' AND -- NULL 이 아닌 진짜 공백값은 비교 연산자 사용 가능
  status IN ('HAPPY', 'VERY HAPPY') -- IN 을 사용해서 여러개의 OR 를 대체
;
```
- 데이터 질의 시 사용
- `SELECT` 구문에는 데이터를 어떤 방법으로 선택할 지 쓰여있지 않으며, 이 방법은 DBMS 에게 일임함
- `SELECT` 구문의 입출력 자료형은 테이블(관계)임
    - 입출력 모두 2차원 표
    - 이 이외엔 어떠한 자료형이 존재하지 않기 때문에 관계가 닫혀있다는 의미로 폐쇄성(closure property)이라고도 부름
    - 뷰와 서브쿼리 이해 시 중요한 개념

<br>

## `GROUP BY` 구 & `HAVING` 구
```SQL
SELECT
  department,
  status,
  COUNT(1) happy_count
FROM employee
WHERE status = 'UNHAPPY'
GROUP BY department, status
HAVING COUNT(1) > 0
;
```
- `GROUP BY` 는 합계 또는 평균 등의 집계 연산을 수행
  - `COUNT`: 레코드 수를 계산
  - `SUM`: 숫자의 합
  - `AVG`: 숫자의 평균을 구함
  - `MAX`, `MIN`: 각각 최댓값과 최솟값
- `HAVING`는 집계 연산 결과에 조건을 지정
  - `WHERE` 구는 레코드에 조건을 지정한다면 `HAVING` 구는 집계 에 조건을 지정
  
<br>
  
## `ORDER BY` 구
```SQL
SELECT
  department,
  name,
  age,
  status
FROM employee
ORDER BY age, name DESC
;
```

<br>

## 뷰와 서브쿼리
```SQL
/* 명시적으로 뷰 생성*/
CREATE VIEW happy_department (
  department,
  happy_count
)
AS
SELECT
  department,
  COUNT(1) happy_count
FROM employee
WHERE status IN ('VERY HAPPY', 'HAPPY')
GROUP BY department
;
SELECT *
FROM happy_department
;

/* 익명 뷰 생성 후 사용 */
SELECT
  department,
  happy_count
FROM (
  SELECT
    department,
    COUNT(1) happy_count
  FROM employee
  WHERE status IN ('VERY HAPPY', 'HAPPY')
  GROUP BY department
)
;
```
- `View` 는 SELECT 구문을 DB 안에 저장하는 것과 같은 결과
  - 내부에 데이터를 저장하진 않으며 어디까지나 `SELECT` 구문을 저장한 것 뿐
  - 뷰에서 데이터를 조회하는 `SELECT` 구문은 실제로 내부적으로 추가적인 `SELECT` 구문을 실행함
- 위와 같이 `FROM` 구에 `SELECT`구를 직접 지정하는 것을 서브쿼리라 함.
  - SQL 은 서브쿼리부터 순서대로 실행함
- 또한 서브쿼리는 `WHERE` 구에 사용하여 아래와 같이 사용 가능
  ```SQL
  SELECT name
  FROM employee
  WHERE employee_id IN (
    SELECT employee_id
    FROM employee
    WHERE age < 30
  )
  ;
  ```

<br>

## 예시 코드
- http://sqlfiddle.com/#!4/9a58c/30
  
