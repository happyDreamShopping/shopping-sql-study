# 1. DBMS 아키텍처

![Main components of a DBMS](/img/1_Main_components_of_a_DBMS.PNG)

## 1. 쿼리 평가 엔진 
사용자로부터 입력받은 SQL구문을 분석하기 위한 계획을 세우고, 이를 실행하는 DBMS 핵심 기능을 담당하는 모듈

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
  ```sql
  SELECT * 
  FROM S
  ```
![Decomposition](/img/1_decomposition_1.PNG)
![Decomposition](/img/1_decomposition_2.PNG)

  - push condition down
  

### 3. Localization (DDB)

### 4. Optimization (DDB)

## 2. 버퍼 매니저
DBMS의 버퍼 영역을 관리하는 역할을 수행함.

## 3. 디스크 용량 매니저
데이터를 어디에 저잘할지 관리하며, 읽고 쓰기를 제어함.

## 4. 트랜잭션 매니저와 락 매니저
트랜잭션에 대한 정합성을 유지와 필요한 경우 데이터에 락을 거는 역할 수행

## 5. 리커버리 매니저
장애 시 정기적으로 백업하고 복구하기 위한 기능을 수행


    
# 2. DBMS와 버퍼
## 1. 기억장치
## 2. 메모리 버퍼
## 3. 메모리 트레이드오프
## 4. 시스템 트레이드오프
## 5. 워킹 메모리
    
# 3. DBMS와 실행계획
## 1. 데이터 접근 절차
### 1. 파서
### 2. 옵티마이저
### 3. 카탈로그 매니저
###  4. 플랜 평가
        
## 2. 옵티마이저 & 통계 정보
## 3. 최적 실행 계획
    
# 4. 실행계획과 SQL 성능
## 1. 실행계획 확인
## 2. 풀 스캔
## 3. 인덱스 스캔
## 4. 테이블 결합
  
# 5. 실행계획의 중요성


# 6. References
  
