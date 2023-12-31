# 데이터모델링의 이해
- 데이터 모델링
  - 개념적, 논리적, 물리적
- DB스키마 구조
  - 외부 스키마, 개념 스키마, 내부 스키마
- 엔티티의 구분
  - 유형 엔티티, 개념 엔티티, 사건 엔티티
  - 기본엔티티(키 엔티티), 중심 엔티티, 행위 엔티티
- 속성의 특성에 따른 분류
  - 기본속성, 설계속성, 파생속성
- 속성명을 지을 때는 엔티티별로 중복되는 이름이 없는 것이 좋다.
- ERD에선 관계를 존재에 의한 관계와 행위에 의한 관계로 구분할 수 있으나 존재와 행위를 구분하지 않고 단일화된 표기법을 사용한다.
  - 실선과 점선이 관계를 나타낼때 사용하긴 하지만 존재와 행위를 구분하기보단 의존관계를 표현한다.
- 관계 표기법은 관계명, 관계차수, 선택성(선택사양) 세가지로 표현한다.

--- 
# 데이터모델과 성능
- 성능데이터 모델링
  - 용량 산정이란
- 정규화는 항상 조회 성능저하를 나타내지는 않고, 중복데이터를 제거함으로써 조회성능을 높여준다.
- 정규화
  - 1차 정규화 : 컬럼이 원자성을 가져야 한다.
  - 2차 정규화 : 기본키의 일부분에만 종속되는 종속자가 없어야한다.(부분종속이 없어야한다.)
  - 3차 정규화 : 이행적 함수 종속성이 없어야한다.
  - BCNF : 모든 결정자가 후보키에 속해있어야 한다. -> 후보키가 아닌 컬럼이 결정자가 되어선 안된다.
  - 4차 정규화 : 다치종속이 없어야한다.
    - ![image](https://github.com/DGUN52/Learn-SQLP/assets/125403830/91a67ced-3d40-46cd-8dab-02b90ebec922)
    - to this
    - ![image](https://github.com/DGUN52/Learn-SQLP/assets/125403830/7e29c241-41ff-4639-98f5-3bc8874214a7)
  - 5차 정규화 : 조인종속이 없고 조인 시 데이터 손실이 없어야 한다.
----------
# SQL 기본 및 활용
- Grouping 함수 결과형태 파악하기
  - GROUP BY ROLLUP() : 괄호 안의 컬럼 각각의 소계와 전체 합계 제공, 단 뒷순서의 **컬럼부터 추가해가며 그루핑 **
  - GROUP BY CUBE() : 괄호 안의 컬럼으로 만들 수 있는 모든 그룹핑 경우의 수에 대한 소계 제공
  - GROUP BY GROUPING SETS() : 괄호 안의 컬럼에 대한 그룹핑 소계 제공
- Grouping 함수에서 null값을 다른 값으로 보여줄 다음과 같은 형태 파악하기
  - CASE WHEN GROUPING(col_a) = 1 THEN 'a컬럼전체' ELSE (col_a) END

-----------
# 아키텍처 기반 튜닝
- 공유서버 전용서버
- Buffer pinning
- Log Force at commit
- Write Ahead Logging
- Delayed Block Cleanout
- sql서버 문법 의미
  - from 'table1' with (READPAST)
- Lock 종류 http://wiki.gurubee.net/pages/viewpage.action?pageId=22904833
- DML작업 중 SELECT는 SHARED Lock, 나머지 U D I는 ROW EXCLUSIVE Lock, Transaction은 Exclusive Lock 획득
- 트랜잭션 격리성 수준
- MVCC
