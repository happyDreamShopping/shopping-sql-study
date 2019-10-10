# 3장 SQL의 조건 분기 - 구문에서 식으로
- UNION을 사용한 조건 분기
    - WHERE 구만 조금씩 다른 여러 개의 SELECT 구문을 합쳐서, 여러 조건에 일치하는 하나의 결과 집합을 얻고 싶을 때 사용
    - **장점:** 큰 문제를 작은 문제로 나눌 수 있어 생각하기 쉬움
        - 조건 분기와 관련된 문제를 접할 대 가장 처음 생각할 수 있는 기본적인 방법
    - **단점:** 성능이 좋지 않음
        - 외부적으로는 하나의 SQL 구문을 실행하는 것처럼 보이지만, 내부적으로는 여러 개의 SELECT 구문을 실행
        - 따라서 테이블에 접근하는 횟수가 많아져서 I/O 비용이 증가함
- UNION과 CASE를 사용한 조건 분기에 대한 실행 계획을 비교하며 사용해야 함

## 11강 절차 지향형과 선언형

### 1. 구문 기반과 식 기반
- SQL 초보자 - 절차 지향적, 기본 단위 `구문(statement)`
    - UNION 자체가 구문을 바탕으로 하는 절차 지향적인 체계를 사용하기 때문에 초보자는 UNION을 사용해서 조건분기 하는 것.
- SQL 중급자 이상 - 선언형, 기본 단위 `식(expression)`
    - 절차 지향형 언어가 CASE 구문으로 분기하는 것을 SQL은 CASE 식으로 분기함.
    - SQL 구문의 각 부분(SELECT, FROM, WHERE, GROUP BY, HAVING, ORDER BY)에 작성하는 것은 모두 식.
    (SQL 구문 내부에는 식을 작성하지, 구문을 작성하지 않음)

### 2. 선언형의 세계로 도약
- 절차 지향형 세계에서 선언형 세계로 도약하는 것이 SQL 능력 향상의 핵심이므로 패러다임 전환을 연습해야 함.

## 08강 UNION을 사용한 쓸데없이 긴 표현
![스터디1](https://user-images.githubusercontent.com/20595690/66521051-58f0bc80-eb25-11e9-9eb1-795093d93133.PNG)
- 2001년 까지는 세금이 포함되지 않은 가격을, 2002년 이후는 세금이 포함된 가격을 price로 표시

### 1. UNION을 사용했을 경우
```SQL
SELECT item_name, year, price_tax_ex AS price
  FROM Items
 WHERE year <= 2001
UNION ALL
SELECT item_name, year, price_tax_in AS price
  FROM Items
 WHERE year >= 2002;
```
- **실행 계획:** Items 테이블에 2번 접근 (TABLE ACCESS FULL 2회)
    - 테이블의 크기에 따라 읽어들이는 비용도 선형으로 증가
      (캐시에 테이블의 데이터가 있으면 증상이 완화되겠지만, 테이블의 크기가 커지면 캐시 히트율도 낮아지므로 성능을 기대하기 어려움)
- **문제점:** 비슷한 쿼리를 두번이나 실행하고 있어 읽기 어렵고, 길고, 성능이 좋지 않음
- UNION은 레코드 집합을 합칠 수 있기 때문에 편리하나, 비슷한 쿼리를 여러번 사용해서 코드가 길어지면 테이블 접근을 증가시켜 SQL의 성능이 하락함

### 2. SELECT 구를 사용했을 경우
```SQL
SELECT item_name, year,
       CASE WHEN year <= 2001 THEN price_tax_ex
            WHEN year >= 2002 THEN price_tax_in END AS price
  FROM Items;
```
- **실행 계획:** Items 테이블에 1번 접근 (TABLE ACCESS FULL 1회)
    - UNION을 사용한 구문보다 성능이 2배 좋아짐
    (사실 버퍼 캐시의 영향도 생각해야 하고, I/O 비용과 실행 시간은 선형 관계에 있지 않기 때문에 진짜 2배는 아님)
- **좋은점:** 가독성이 높아지고, 성능도 좋아지고, 짧아짐
- SELECT 구문을 사용해서 조건분기를 하면 UNION을 사용했을 때와 같은 결과를 출력하지만 성능면에서는 훨씬 좋 (테이블의 크기가 커질수록 명확한 성능차이가 나타남)

### 3. 결론 
- 조건 분기를 WHERE 구로 하는 사람들은 초보자. 잘하는 사람들은 SELECT 구만으로 조건 분기를 함
- SQL 구문의 성능이 좋은지 나쁜지는 반드시 실행 계획 레벨에서 판단해야 함
    - SQL 구문에는 어떻게 데이터를 검색해야 할지를 나타내는 접근 경로가 쓰여 있지 않기 때문에 이를 알려면 실행 계획을 봐야 함
        - RDB와 SQL이 가진 컨셉은 "사용자가 데이터에 접근 경로라는 물리 레벨의 문제를 의식하지 않도록 하고 싶다"
        - 이런 뜻을 이루기에는 현재의 RDB와 SQL, 하드웨어는 역부족이기 때문에 은폐하고 있는 접속 경로를 엔지니어가 체크해줘야 함
- UNION과 CASE의 쿼리를 구문적인 관점에서 비교하면?
    - UNION을 사용한 분기는 SELECT 구문을 기본 단위로 분기
        - 구문을 기본 단위로 사용하고 있다는 점이 아직 절차 지향형의 발상을 벗어나지 못한 방법
    - CASE 식을 사용한 분기는 문자 그대로 식을 바탕으로 하는 사고
    - 구문에서 식으로 사고를 변경하는 것이 SQL을 마스터하는 열쇠 중 하나
        - 문제를 절차 지향형 언어로 해결한다면 어떤 IF 조건문을 사용해야 할까? → SQL의 CASE로는 어떻게 해결할 수 있지?를 꾸준히 의식해야 함

## 09강 집계와 조건 분기
### 1. 집계 대상으로 조건 분기
![스터디2](https://user-images.githubusercontent.com/20595690/66521757-c8b37700-eb26-11e9-97bb-e30ec23b4c7a.PNG)
- 테이블의 레이아웃을 변경하기
    - 표측/표두 레이아웃 이동 문제: CASE 식의 응용 방법으로 유명
      (SQL은 이런 결과 포맷팅을 목적으로 만들어진 언어가 아니지만 실무에서 자주 사용됨)
    - 표두(head, column): 표의 상부 제목, 표측(stub, row): 표의 좌측 제목

#### 1-1. UNION을 사용했을 경우
```SQL
SELECT prefecture, SUM(pop_men) AS pop_men, SUM(pop_wom) AS pop_wom
  FROM ( SELECT prefecture, pop AS pop_men, NULL AS pop_wom
           FROM Population
          WHERE sex = '1'
          UNION
         SELECT prefecture, NULL AS pop_men, pop AS pop_wom
           FROM Population
          WHERE sex = '2') TMP
 GROUP BY prefecture;
```
- **쿼리 설명:** 서브쿼리 TMP로 인해 남성의 인구를 지역별로 구하고, 여성의 인구를 지역별로 구한 뒤 GROUP BY구를 이용해 하나로 집약
- **실행 계획:** Population 테이블에 2번 접근 (TABLE ACCESS FULL 2회)
- **문제점:** WHERE 구에서 sex 필드로 분기, 결과를 UNION으로 머지하는 절차 지향적인 방식을 사용

#### 1-2. SELECT 구를 사용했을 경우
```SQL
SELECT prefecture,
       SUM(CASE WHEN sex = '1' THEN pop ELSE 0 END) AS pop_men,
       SUM(CASE WHEN sex = '2' THEN pop ELSE 0 END) AS pop_wom
  FROM Population
 GROUP BY prefecture;
```
- **쿼리 설명:** CASE 식을 집약 함수 내부에 포함시켜서 남성인구와 여성인구 필터를 생성
- **실행 계획:** Population 테이블에 1번 접근 (TABLE ACCESS FULL 1회)
    - UNION을 사용했을 때 풀 스캔이 2회인 경우에 비해 I/O 비용이 절반으로 감소함
      (캐시 등을 고려하지 않는다고 가정)
- **좋은점:** 쿼리가 간단해지고, 성능도 향상됨
    - UNION을 사용하지 않고 CASE 식으로 조건 분기를 잘 활용하면 테이블의 접근을 줄일 수 있음
    - CASE 식은 SQL에서 다양한 표현을 할수 있을 뿐만 아니라 성능적으로도 큰 힘을 발휘함

### 2. 집약 결과로 조건 분기
![스터디3](https://user-images.githubusercontent.com/20595690/66521831-f4366180-eb26-11e9-8026-20c6fa29eab4.PNG)
- 다음과 같은 조건에 맞는 테이블 생성하기
    1. 소속된 팀이 1개라면 해당 직원은 팀의 이름을 그대로 출력 → Jim, Carl
    2. 소속된 팀이 2개라면 해당 직원은 '2개를 겸무'라는 문자열을 출력 → Kim
    3. 소속된 팀이 3개 이상이라면 해당 직원은 '3개를 겸무'라는 문자열을 출력 → Bree, Joe

#### 2-1. UNION을 사용했을 경우
```SQL
SELECT emp_name,
       MAX(team) AS team
  FROM Employees
 GROUP BY emp_name
HAVING COUNT(*) = 1
UNION
SELECT emp_name,
       '2개를 겸무' AS team
  FROM Employees
 GROUP BY emp_name
HAVING COUNT(*) = 2
UNION
SELECT emp_name,
       '3개 이상을 겸무' AS team
  FROM Employees
 GROUP BY emp_name
HAVING COUNT(*) >= 3;
```
- **쿼리 설명:** 조건 분기가 레코드값이 아닌, 집합의 레코드 수에 적용되기 때문에 조건 분기가 WHERE 구가 아니라 HAVING 구에 지정됨 
- **실행 계획:** 3개의 쿼리를 머지하므로 Employees 테이블에 3번 접근 (TABLE ACCESS FULL 3회)
- **문제점:** HAVING 구에서 조건 분기(UNION으로 머지하고 있는 이상 구문 레벨의 분기일 뿐 WHERE 구를 사용할 때와 크게 다르지 않음)

#### 2-2. SELECT 구를 사용했을 경우
```SQL
SELECT emp_name,
       CASE WHEN COUNT(*) = 1 THEN MAX(team)
            WHEN COUNT(*) = 2 THEN '2개를 겸무'
            WHEN COUNT(*) >= 3 THEN '3개 이상을 겸무'
       END AS team
  FROM Employees
 GROUP BY emp_name;
```
- **쿼리 설명 & 좋은점** 
    - GROUP BY의 HASH 연산이 3회에서 1회로 감소
    - CASE 식을 사용하면 테이블에 접근 비용을 3분의 1로 줄일 수 있음
    - 위의 두개가 가능했던 것은 집약 결과(COUNT 함수의 리턴값)를 CASE 식의 입력으로 사용했기 때문
    - COUNT 또는 SUM과 같은 집약 함수의 결과는 1개의 레코드로 압축됨
    - 집약 함수의 결과가 스칼라(더 이상 분할 불가능한 값)가 되기 때문에 CASE 식의 매개변수에 집약 함수를 넣을 수 있음
- **실행 계획:** Employees 테이블에 1회 접근 (TABLE ACCESS FULL 1회)
    
### 3. 결론
- WHERE 구에서 조건 분기를 하는 사람은 초보자, HAVING 구에서 조건 분기를 하는 사람도 초보자

## 10장 그래도 UNION이 필요한 경우

### 1. UNION을 사용할 수밖에 없는 경우
- 머지 대상이 되는 SELECT 구문들에서 사용하는 테이블이 다른 경우 (여러개의 테이블에서 검색한 결과를 머지하는 경우)
```SQL
SELECT col_1
  FROM Table_A
 WHERE col_2 = 'A'
UNION ALL
SELECT col_3
  FROM Table_B
 WHERE col_4 = 'B';
```
- FROM 구에서 테이블을 결합하고 CASE 식을 사용할 수도 있지만 그러면 필요 없는 결합이 발생해서 성능이 UNION보다 더 좋지 않음

### 2. UNION을 사용하는 것이 성능적으로 더 좋은 경우
![스터디4](https://user-images.githubusercontent.com/20595690/66532939-273f1c00-eb4c-11e9-9137-752cc000bb14.PNG)
- 날짜 필드 date_1~date_3과 짝을 이루는 플래드 필드 flg_1~flg3가 있는 테이블
    - 레코드는 (date_n, flg_n) 이라는 3개의 짝에서 하나의 짝에만 값이 있고 다른 짝은 모두 (NULL, NULL)
    - date_1~date_3이 특정 날짜를 가지고 있고 대칭되는 플래그 필드의 값이 'T'인 레코드를 선택하는 경우 SELECT 구문을 만드는 방법

#### 2-1. UNION을 사용한 방법 
```SQL
SELECT key, name,
       date_1, flg_1,
       date_2, flg_2,
       date_3, flg_3
  FROM ThreeElements
 WHERE date_1 = '2013-11-01'
   AND flg_1 = 'T'
UNION
SELECT key, name,
       date_1, flg_1,
       date_2, flg_2,
       date_3, flg_3
  FROM ThreeElements
 WHERE date_2 = '2013-11-01'
   AND flg_2 = 'T'
UNION
SELECT key, name,
       date_1, flg_1,
       date_2, flg_2,
       date_3, flg_3
  FROM ThreeElements
 WHERE date_3 = '2013-11-01'
   AND flg_3 = 'T';
```
- 3개의 SELECT 구문을 UNION으로 머지
- 3개의 SELECT 구문에서 다른 부분은 WHERE 구 뿐
- 성능과 실행 계획에서 이 쿼리를 최적의 성능으로 수행하려면 다음과 같은 필드 조합의 인덱스가 필요함

```SQL
CREATE INDEX IDX_1 ON ThreeElements (date_1, flg_1);
CREATE INDEX IDX_2 ON ThreeElements (date_2, flg_2);
CREATE INDEX IDX_3 ON ThreeElements (date_3, flg_3);
```
- 인덱스가 있으면 3개의 SELECT 구문 모두 IDX_1, IDX_2, IDX3 인덱스를 사용함
- ThreeElements 테이블의 레코드 수가 많고, 각각의 WHERE 구의 검색 조건에서 레코드를 많이 압축할수록 풀 스캔보다 빠른 접근 속도를 기대할 수 있음

#### 2-2. OR을 사용한 방법
```SQL
SELECT key, name,
       date_1, flg_1,
       date_2, flg_2,
       date_3, flg_3
  FROM ThreeElements
 WHERE (date_1 = '2013-11-01' AND flg_1 = 'T')
    OR (date_2 = '2013-11-01' AND flg_2 = 'T')
    OR (date_3 = '2013-11-01' AND flg_3 = 'T');
```
- 결과는 UNION을 사용했을 때와 같으나 실행 계획이 다름
- SELECT 구문이 하나로 줄어들었기 때문에 ThreeElements 테이블에 대한 접근이 1회로 감소
- 이때는 인덱스가 사용되지 않고 ThreeElements 테이블에 1번 접근 (TABLE ACCESS FULL 1회)
- WHERE 구문에서 OR을 사용하면 해당 필드에 부여된 인덱스를 사용할 수 없음

#### 2-3. IN을 사용한 방법
```SQL
SELECT key, name,
       date_1, flg_1,
       date_2, flg_2,
       date_3, flg_3
  FROM ThreeElements
 WHERE ('2013-11-01', 'T')
           IN ((date_1, flg_1),
               (date_2, flg_2),
               (date_3, flg_3));
```

#### 2-4. CASE 식을 사용한 방법
```SQL
SELECT key, name,
       date_1, flg_1,
       date_2, flg_2,
       date_3, flg_3
  FROM ThreeElements
 WHERE CASE WHEN date_1 = '2013-11-01' THEN flg_1
            WHEN date_2 = '2013-11-01' THEN flg_2
            WHEN date_3 = '2013-11-01' THEN flg_3
            ELSE NULL END = 'T';
```
