# 1. DBMS 아키텍처

![Main components of a DBMS](/img/1_Main_components_of_a_DBMS.PNG)

## 1. 쿼리 평가 엔진 
사용자로부터 입력받은 SQL구문을 분석하기 위한 계획을 세우고, 이를 실행하는 DBMS 핵심 기능을 담당하는 모듈

<details>
<summary>more</summary>

(Distributed) Query Processing
- Decomposition
  - Normalization
  - Eliminating invalid and redundancy
  - Algebraic rewriting
- Localization
  - Start query
  - Replace relations by fragments
  - Simplify
- Optimization
  - CPU cost
  - I/O cost
  - Communication cost

### 1. Decomposition
- query에 대한 sanity, syntax 검증을 수행하고, 이를 실행 가능한 standard form으로 변환한다.

**1) Normalization**
일반적인 언어에서 실행 가능한 표준언어로 변환하는 단계이다.

  - language form
  ```sql
  SELECT A,C
  FROM R,S
  WHERE (R.B=1 and S.D=2) or (R.C>3 and S.D=2)
  ```

  - algebraic form
  
![Decomposition](/img/1_1.PNG)

  - e.g) https://www.slideshare.net/AliUsman10/database-7-query-localization/12


**2) Eliminating invalid and redundancy**
  - invalid
  ```sql
  SELECT * FROM S, R, WHERE R.B=1
  ```

  - redundancy
  1) (S.A=1) ∩ (S.A>5) => False
  2) (S.A<10) ∩ (S.A<5) => S.A<5


**3) Algebraic rewriting**
  - sub-expressions
  
  ![Decomposition](/img/1_decomposition_1.PNG)
  ![Decomposition](/img/1_decomposition_2.PNG)

  ```sql
  a = b * c + g;
  d = b * c * e;
  ```
  ```
  tmp = b * c;
  a = tmp + g;
  b = tmp * e;
  ```
  
### 3. Localization (DDB)
  - Start query: Algebraic query를 실행한다.
  - Replace relations by fragments: union operation으로 치환한다. 
  - [Push up union](https://www.slideshare.net/AliUsman10/database-7-query-localization/17)
    - selection, prediction down
  - [Simplify](https://www.slideshare.net/AliUsman10/database-7-query-localization/18)
    - eliminate unnecessary operations

### 4. Optimization (DDB)
전체 Execution plan의 cost에 대한 최적화

- CPU: 로컬에서 instruction을 실행하는 비용
- I/O: disk I/O 비용
- Communication: fragment간 message 초기화 + 전송 비용

</details>

## 2. 버퍼 매니저
DBMS의 버퍼 영역을 관리하는 역할을 수행함.

## 3. 디스크 용량 매니저
데이터를 어디에 저잘할지 관리하며, 읽고 쓰기를 제어함.

## 4. 트랜잭션 매니저와 락 매니저
트랜잭션에 대한 정합성을 유지와 필요한 경우 데이터에 락을 거는 역할 수행

## 5. 리커버리 매니저
장애 시 정기적으로 백업하고 복구하기 위한 기능을 수행

# 2. DBMS와 버퍼
메모리는 한정된 희소자원이므로 데이터를 버퍼에 어떠한 식으로 저장 및 확보할 것인지에 따라 성능에 굉장히 중요한 영향을 미친다.

## 1. 기억장치
기억비용(데이터를 저장하는 비용)에 따라 1차, 2차, 3차의 계층으로 나뉘고, 각 저장소를 선택함에 따라 영속성과 속도 간의 트레이드오프가 발생한다.

### 1) 하드디스크
2차 기억장치로 DBMS의 대부분은 HDD를 선택하며, 어떠한 상황에서 평균적인 수치를 가지는 매체이다.

## 2. 메모리 위에 있는 두개의 버퍼
디스크에 비해 기억비용이 매우 비싸고 한정적이므로, 모든 내부 데이터를 메모리에 올리는 것은 불가능하다. 따라서 자주 접근하는 데이터를 
메모리(버퍼 또는 캐시)에 올려둔다면 sql구문의 실행 시간의 대부분을 차지하는 I/O 비용을 큰 폭으로 줄일 수 있다.

### 1) 데이터 캐시
디스크에 있는 데이터의 일부를 메모리에 유지하기 위해 사용하는 메모리 영역

### 2) 로그 버퍼
- 로그 버퍼는 **갱신 처리 (INSERT, DELETE, UPDATE, MERGE)** 와 관련있다. DBMS는 사용자로부터 입력 받은 갱신과 관련된 SQL 구문을 바로 저장소 데이터를 변경하지 않는다. 
- 일단 로그 버퍼에 보내고 이후에 디스크 변경을 수행한다. 
- 갱신 처리는 SQL 구문 실행 시점과 갱신 시점의 차이가 있는 비동기 처리이다.
> 저장소 검색/갱신에 상당한 시간이 소모되므로, 우선 통지한 뒤 비동기로 처리를 계속하는 구조임

<details>
<summary>more</summary>
  
Redo Log Buffer는 아래의 5가지 특성을 지닌다.
- Physiological Logging
- Page Fix Rule
- Write Ahead Log
- Log Force At Commit
- Logical Ordering Of Redo

#### 1) Physiological Logging
최소의 Logging으로 최대의 복구 기능 제공

![Physiological Logging](/img/1_3_2.PNG)

#### 2) Page Fix Rule

![Page Fix Rule](/img/1_3_3.PNG)

- 변경을 시도하는 블럭에 대한 모든 변경이 성공하기까지 다른 Access 방지
- 블록에 변경 과정에서 세마포어가 수행되는 프로세스에 대한 것

#### 3) Write Ahead Log
DML 작업 시 블록의 변경보다 Redo Log를 먼저 생성해 Redo Log Buffer에 기록하는 기법
- Log Buffer Ahead: 변경하기 위한 버퍼를 데이터베이스 버퍼 캐시에 캐싱한 후 실제 변경을 수행하기 전에 Redo를 Log Buffer에 먼저 기록하는 방식
- Log File Ahead: DBWR이 변경된 블록 버퍼를 데이터 파일에 기록하기 전에 LGWR(Log Writer)이 Redo Record를 Redo Log File에 기록하는 방식이다.

![Write Ahead Log](/img/1_3_4.PNG)

- 블록이 변경된 후 Redo Log를 생성하지 못하고 데이터베이스가 비정상 종료되는 경우, 복구가 불가
- 이를 위해 블록을 변경하기 전 반드시 Redo Log를 생성하는 Write-Ahead Logging 기법을 사용함


#### 4) Log Force At Commit
사용자로부터 Commit 요청이 들어오면 관련된 모든 Redo Record들은 Redo Log file에 저장한 후 Commit을 완료

![Log Force At Commit](/img/1_3_5.PNG)

- Piggyback Write/Group Commit: 동시에 여러 사용자에 의해 발생한 트랜잭션에 대한 Commit을 한 번씩 LGWR에 의해 기록하는 경우 디스크 I/O가 과다하게 발생한다. 이에 한 번에 여러 개의 Redo Log를 Redo Log File에 기록하여 디스크 부하를 줄이는 방식이다.


#### 5) Logical Ordering Of Redo
Redo Log File에 Redo Log를 기록하 는 과정에서 트랜잭션별로 물리적인 순서를 정하는 것이 아니라 SCN 에 의해 논리적인 순서를 정해 저장하는 기법
 
</details>


## 3. 메모리 트레이드오프
### 1. 휘발성
- 메모리에는 데이터의 영속성이 없다. 전원을 꺼져버리면 메모리 위에 올라가 있는 모든 데이터가 사라지는데 이러한 성질을 휘발성이라고 한다. (버퍼위의 데이터들이 모두 날아감)
- 영속성이 없는 이상 디스크를 완전히 대체하는 것은 불가능

### 2. 휘발성의 문제점
- 장애 발생 시 데이터 유실로 인한 부정합
- 데이터 캐시라면 디스크위에 데이터가 남아있기 때문에 문제가 되지 않지만 로그 버퍼 위에 존재하는 데이터가 로그 파일에 반영되기 전에 장애가 발생해서 사라져 버린다면 복구 불가
- DBMS가 비동기로 로그 파일에 기록하는 이상 유실 가능성은 항상 존재함
- 이를 위해 커밋 시점에 반드시 갱신 정보를 로그파일에 저장하여, 장애 시에도 정합성을 유지하도록 함

  | 이름   | 데이터 정합성 | 성능 |
  |--------|---------------|------|
  | 동기   | O             | X    |
  | 비동기 | X             | O    |


## 4. 시스템 트레이드오프
### 1. 데이터 캐시와 로그 버퍼 크기
로그 버퍼가 데이터 캐시보다 작다. 이는 기본적으로 검색을 메인으로 처리한다고 가정하기 때문

## 5. 워킹 메모리
- 데이터 캐시와 로그 버퍼와는 다른 영역으로 주로 정렬 또는 해시 관련 처리에 사용되는 작업용 영역이며, 종료되면 해제되는 임시 영역이기도 하다.
  - 정렬: Order by, 집합 연산, 윈도우 함수
  - 해시: 테이블 해시 결합
- 데이터양보다 영역의 크기가 작은 경우 데이터베이스 저장소 간의 SWAP이 발생하여 전체적인 속도가 느려진다. 단 메모리가 부족하더라도 처리가 멈추거나 에러를 발생하지 않는다. 

# 3. DBMS와 실행계획
## 1. 데이터 접근 절차

![query process](/img/3_1_1.PNG)

- 데이터 접근 절차를 결정하는 모듈은 쿼리 평가 엔진이라고 부른다.

### 1. 파서
- 사용자로부터 입력받은 SQL 구문에 대한 문법 검증과 정형적인 형식으로 변환한다.

### 2. 옵티마이저
- 옵티마이저는 인덱스 유무, 데이터 분산 또는 편향 정도, 내부 매개변수 등의 조건을 고려해서, 선택 가능한 많은 실행 계획을 작성한다.
- 실행 계획들의 비용을 연산하고, 가장 낮은 비용을 가진 실행계획을 선택한다.

### 3. 카탈로그 매니저
- 옵티마이저가 실행계획을 세울 때 옵티마이저에게 정보를 제공
- DBMS의 내부 정보를 모아놓은 테이블로, 테이블 또는 인덱스 통계 정보가 저장되어 있음

###  4. 플랜 평가
- 옵티마이저가 SQL 구문에서 여러개의 실행 계획을 세운 후 그것을 받아 최적의 실행결과를 선택하는 것

        
## 2. 옵티마이저 & 통계 정보
옵티마이저가 최적의 플랜을 선택하도록 하기 위해서는 카탈로그에 포함되어있는 통계정보가 충분해야한다.
- 각 테이블의 레코드 수
- 각 테이블의 필드 수와 필드의 크기
- 필드의 카디널리티
- 필드값의 히스토그램
- 필드 내부에 있는 NULL 수
- 인덱스 정보


## 3. 최적 실행 계획
    
# 4. 실행계획과 SQL 성능
## 1. 실행계획 확인
## 2. 풀 스캔
## 3. 인덱스 스캔
## 4. 테이블 결합
  
# 5. 실행계획의 중요성


# 6. References
- http://www.grammaticalframework.org/qconv/qconv-a.html
- https://www.slideshare.net/AliUsman10/database-7-query-localization
- http://www.gurubee.net/lecture/2666
- https://o2sunn.tistory.com/20
