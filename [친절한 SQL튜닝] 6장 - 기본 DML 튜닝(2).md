6.2 Direct Path I/O 활용
- OLTP에선 특정 데이터를 집중적으로 반복해서 읽기 때문에 버퍼캐시로 성능이 향상된다.
- 반면 정보계 시스템(DW/OLAP)나 배치 프로그램에서 사용하는 SQL은 대량데이터 용도
  - 따라서 버퍼캐시를 사용하는 IO메커니즘이 오히려 성능을 떨어뜨린다.
  - 이럴 때 Direct Path IO방식을 사용하면 효과적이다.

6.2.1 Direct Path IO
- 대량 데이터 R/W작업 시 버퍼캐시 탐색은 비효율적이다.
- Direct Path IO방식이 작동하는 경우
  1. 병렬 쿼리로 Full Scan 수행★
  2. 병렬 DML 수행★
  3. Direct Path Insert 수행★
  4. Temp 세그먼트 블록들을 읽고 쓸 때
  5. direct 옵션 지정 후 export 수행
  6. nocache 옵션 지정 후 LOB 컬럼 읽을 때

- ```
  sql select /*+ full(t) parallel(t 4) */ * from big_table t;
  select /*+ index_ffs(t big_table_x1) parallel_index(t big_table_x1 4) */ count(*) from big_table t;
  ```
  위와 같은 방식으로 힌트를 지정하면 지정한 병렬도(4)만큼 병렬 프로세스로 작업
- 병렬도 지정은 수배가 아닌 수십배로 성능이 향상된다. 버퍼캐시를 탐색, 디스크에서 버퍼캐시에 적재하지 않기 때문
- order by, group by, 해시조인, 소트 머지 조인 등의 작업에는 지정한 병렬도보다 두 배의 프로세스 사용

6.2.2 Direct Path Insert
- 일반적인 INSERT가 느린 이유
  1. 데이터를 새로 넣을 블록을 Freelist에서 찾는다.
     - 테이블 HWM high water mark 아래쪽에 있는 블록 중 데이터 입력이 가능한 블록의 목록을 Freelist라 한다.
  2. Freelist에서 할당받은 블록을 버퍼캐시에서 찾는다.
  3. 버퍼캐시에 없으면 데이터파일에서 읽어 버퍼캐시에 적재한다.
  4. Insert 내용을 Undo 세그먼트에 기록
  5. Insert 내용을 Redo 로그에 기록

- Direct Path Insert 유도 힌트
  1. Insert ... Select 문에 append 힌트 사용
  2. parallel 힌트의 INSERT
  3. direct 옵션 지정 후 SQL Loader(sqlldr)로 데이터 적재
  4. CTAS(create table ... as select)문 수행

- Direct Path Insert 방식이 빠른 이유
  1. Freelist를 참조하지 않고 HWM 바깥 영역에 데이터 순차적 입력
  2. 블록을 버퍼캐시에서 탐색하지 않음
  3. 버퍼캐시에 적재하지 않고 데이터파일에 직접 기록
  4. Undo 로깅을 안 한다
     - HWM뒤에 데이터를 입력하기 때문에 다른 세션이 읽을 일이 없다.
       - INSERT 작업이 끝나면 HWM을 뒤로 이동
       - INSERT 롤백시 해당 익스텐트에 대한 딕셔너리 정보만 롤백하면 됨
       - 할당된 익스텐트 정보만 로깅하면 롤백 가능
  5. Redo 로깅을 최소화 할 수 있다. (NOLOGGING 옵션)
     ```sql
     alter table t NOLOGGING;
     ```
     - 정확히는 데이터 딕셔너리 변경사항만 로깅한다.

- Array Processing도 Direct Path Insert 방식으로 처리 가능.
  - append_values 힌트 이용
  - ```sql
    ...
    procedure insert_target(p_source in typ_source) is
    begin
      forall i in p_source.first..p_source.last
      insert /*+ append_values *. into target values p_source(i);
    end insert_target;
    ...
    ```
- Direct Path Insert 방식 이용의 주의점
  1. 성능은 빨라지지만 Exclusive 모드 TM Lock이 걸림.
     - 커밋하기 전까지 다른 트랜잭션은 해당 테이블에 DML을 수행할 수 없음
     - 따라서 트랜잭션이 많은 테이블에 (혹은 시간대에) 이 옵션을 사용하면 안 된다.
     - 참고 TM Lock(6.4.1)
  2. Freelist를 조회하지 않고 HWM 뒤쪽 바깥 영역에 데이터를 입력하므로 테이블의 여유공간을 재활용하지 않는다.
     - Direct Path Insert 방식으로만 INSERT하는 테이블은 사이즈가 끊임없이 늘어난다.
     - Range 파티션 테이블의 경우 버리는 데이터를 DELETE가 아닌 DROP 방식으로 해야 공간이 반환된다.
     - 비파티션 테이블이면 주기적으로 Reorg 작업 수행으로 공간을 정리해야한다.
    
6.2.3 병렬 DML
- Insert는 append 힌트로 Direct Path Write 방식이 유도되지만 UPDATE, DELETE는 기본적으로 불가능
  - 병렬 DML로만 병렬 처리 가능
- ```sql
  alter session enable parallel dml;
  ```
- ```sql
  insert /*+ parallel(c 4) */ into 고객 c
  select /*+ full(o) parallel(o 4) */ * from 외부가입고객 o;

  update /*+ full(c) parallel(c 4) */ 고객 c set 고객상태코드 = 'WD'
  where 최종거래일시 < `20100101';

  delete /*+ full(c) parallel(c 4) */ from 고객 c
  where 탈퇴일시 < `20100101`;
  ```
- 위와 같이 옵션을 키고 힌트를 지정하면 INSERT의 SELECT쿼리, UPDATE/DELETE의 조건절 검색은 물론 INSERT, UPDATE, DELETE 작업도 병렬로 진행한다.
- 옵션을 켜지 않고 힌트만 기술하면 검색과 SELECT쿼리는 병렬로 진행되지만 INSERT, UPDATE, DELETE는 QC(Query Coordinator)가 혼자 진행하여 병목 발생

- +오라클의 두단계 전략 : 오라클은 Consistent모드로 select/조건절 검색을 수행하고 Current모드로 추가/변경/삭제한다. (기본적으로 QC의 직렬처리)

- 병렬INSERT는 append 힌트 없이도 Direct Path Insert방식을 사용한다.
  - 병렬 DML이 작동하지 않을 경우를 대비해 append 힌트를 기술하여 Direct Path INSERT 방식이 동작하게 유도한다.
  - 12c부터 작동하는 enable_parallel_dml 힌트 (옵션켜는 힌트)
    ```sql
    insert /*+ enable_parallel_dml parallel(c 4) */ into 고객 c
    select /*+ full(o) parallel(o 4) */ * from 외부가입고객 o;
    -- update, delete문도 마찬가지
    ```
  - 병렬DML또한 Direct Path Write방식을 사용하므로 insert,update,delete 시 Exclusive 모드 TM Lock이 걸림 (언제, 어떤 상황에 사용하면 안된다?)

- 병렬 DML의 정상작동을 확인하는 법
  - 실행계획에서 UPDATE(INSERT/DELETE)가 'PX COORDINATOR' 아래에 나타나면 해당 DML을 각 병렬프로세스가 처리한다.
  - 실행계획에서 UPDATE(INSERT/DELETE)가 'PX COORDINATOR' 위에 나타나면 해당 DML을 QC가 처리한다.

 
6.3 파티션을 활용한 DML튜닝
- 파티션은 대량 DML(UPDATE, DELETE, INSERT) 작업을 빠르게 할 수 있다.

6.3.1 테이블 파티션
- 테이블 혹은 인덱스 데이터를 특정 컬럼(파티션 키)값에 따라 별도 세그먼트에 나눠서 저장하는 것
- 일반적으로 시계열에 따라 Range 방식으로 분할
  - 리스트 또는 해시방식으로도 가능
- 파티션이 필요한 이유
  - 관리 측면 : 파티션 단위 백업, 추가, 삭제, 변경으로 가용성 향상
  - 성능 측면 : 파티션 단위 조회 및 DML, 경합·부하 분산
- 파티션의 종류
  - Range, 해시, 리스트

1) Range 파티션
  - 가장 기초적인 방식의 파티션
  - 주로 날짜 컬럼을 기준으로 파티셔닝
  - ```sql
    create tabel 주문 (a, b, c, d, 주문일자)
    partition by range(주문일자) (
      partition P2017_Q1 values less than ('20170401')
      ,partition P2017_Q2 values less than ('20170701')
      ,partition P2017_Q3 ...
      ...
      ,partition P9999_MX values less than (MAXVALUE) -- 주문일자 >= 'x' -- 마지막 파티션 기준 날짜
    );
    ```
  - 해당 파티셔닝을 통해 검색 시에도 조건을 만족하는 파티션만 골라 읽고,
  - 보관 정책에 따라 과거 데이터에 해당하는 파티션만 백업/삭제 하는 등의 데이터 관리 작업을 효율적으로 수행할 수 있게 한다.

  - 파티션 pruning
    - SQL 하드파싱, 읽지 않아도 되는 파티션 세그먼트를 액세스 대상에서 제외하는 기능(가지치기)
    - ```sql
      select *
      from 주문
      where 주문일자 >= '20120401'
      and 주문일자 <= '20120630'
      ```
    - 위와 같은 조건절이 있다면 해당 데이터가 해당되는 P2012_Q2만 읽으면 된다.
    - 병렬처리로 더욱 효과적으로 수행할 수 있다.
    - 파티셔닝은 클러스터링 기술에 속한다.
      - 차이점은 파티션은 세그먼트 단위로 모아서 저장함
      - 클러스터는 블록단위로 모아 저장
      - IOT는 데이터를 정렬된 순서로 저장 (Index Organized Table)

2) 해시 파티션
  - 파티션 키 값을 해시함수에 입력하여 반환값이 같은 데이터를 같은 세그먼트에 저장하는 방식
  - 파티션 개수만 사용자가 결정, 데이터를 나누는 알고리즘은 오라클 내부의 해시함수가 결정
  - 고객ID처럼 변별력이 좋고 분포가 고른 컬럼을 선정
  - ```sql
    create table 고객 (고객ID varchar2(5), 고객명 varchar2(10), ...)
    partition by hash(고객ID) partitions 4;
    ```
  - 검색 시엔 조건절 비교값(상수/변수)에 똑같은 해시 함수를 적용하여 읽은 파티션을 결정
    - = 또는 IN-List 방식으로 검색할 때만 파티션 Pruning 동작 (해시 알고리즘의 특징)

3) 리스트 파티션
  - 사용자가 정의한 그룹핑 기준에 따라 데이터를 분할 저장
  - ```sql
    create table 인터넷매물 (물건코드 varchar, 지역분류 varchar, ...)
    partition by list(지역분류) (
      partition P_지역1 values ('서울')
    , partition P_지역2 values ('경기', '인천')
    , ...
    , partition P_기타 values (DEFAULT)
    );
    ```
  - Range파티션과 달리 파티션 기준 값의 순서와 상관없이 불연속적인 값의 목록에 의해 파티션을 나눈다.
  - 가능한 파티션에 값이 고르게 분산되도록 해야 한다.


6.3.2 인덱스 파티션
- 테이블 파티션과 맞물려 다양한 구성 존재
  - 테이블 파티션의 구분
    - 비파티션 테이블
    - 파티션 테이블

  - 인덱스 파티션의 구분
    - 파티션 인덱스
      - 로컬 파티션 인덱스
      - 글로벌 파티션 인덱스
    - 비파티션 인덱스
   
1) 로컬 파티션 인덱스
  - 테이블 파티션과 인덱스 파티션이 1:1대응관계가 되도록 오라클이 자동으로 관리하는 파티션 인덱스
  - 로컬이 아닌 파티션 인덱스는 모두 글로벌 파티션 인덱스
  - ![image](https://github.com/DGUN52/Learn-SQLP/assets/125403830/d60e5a2b-1e0e-4704-9c9b-bf9d72f95144)
  - 파티션별로 별도 인덱스를 생성하는 것과 같다.
  - create index문 뒤에 local 옵션을 추가하여 사용
    ```sql
    create index 주문_x01 on 주문 (주문일자, 주문금액) LOCAL;
    ```
  - 테이블 파티션 속성을 그대로 상속받음.
    - 테이블 파티션 키가 주문일자면 인덱스 파티션 키도 주문일자
    - 로컬 파티션 인덱스 > 로컬 인덱스라고 줄이기도 함
  - 오라클이 테이블과 정확히 1:1 대응관계를 갖게 관리해줌으로 인덱스를 재생성할 필요가 없다.
    - 트랜잭션이 몰리는 시간대만 아니면 무중단으로도 작업 가능
  - 즉 관리가 매우 편리하다.

2) 글로벌 파티션 인덱스
   - 파티션 테이블과 다르게 구성한 인덱스 (혹은 비파티션 테이블에 구성된 인덱스 파티션)
     - 파티션 테이블과 유형이 다르거나, 키가 다르거나, 기준값 정의가 다른 경우
     - CREATE INDEX문 뒤에 GLOBAL 옵션을 추가하여 사용
     ```sql
     create Index 주문_x02 on 주문 (주문금액, 주문일자) GLOBAL;
     partition by range(주문금액) (
      partition P_01 values less than (10000)
     ,partition P_MX values less than (MAXVALUE) -- 주문금액 >=10000
     );
     ```
  - 글로벌 파티션 인덱스는 테이블 파티션 구성을 변경(drop, exchange, split 등)하는 순간 Unusable상태로 바뀌어 인덱스를 바로 재생성 해주어야 함.
    - 그동안 서비스는 중단됨
  - 직접 테이블 파티션과 1:1 대응되도록 만들 순 있지만 오라클이 자동으로 관리해주진 않음.(Global 파티션이기 때문)

3) 비파티션 인덱스
   - 파티셔닝하지 않은 인덱스
   - 일반적인 CREATE INDEX문으로 생성
   - 여러 테이블을 가리키기 때문에 글로벌 비파티션 인덱스라고 불리기도 함
   - 테이블 파티션 구성을 변경하는 순간 Unusable, 인덱스 재생성 필요. 서비스 중단.

4) Prefixed vs. Nonprefixed
   - 인덱스 파티션 키 컬럼이 인덱스 구성의 왼쪽 선두컬럼에 위치하는지에 따른 구분
   - Prefixed : 인덱스 파티션 키 컬럼이 선두에 위치
   - Nonprefixed : 인덱스 파티션 키 컬럼이 선두에 위치하지 않거나 아예 속하지 않음
   - 즉 로컬이냐 글로벌이냐, prefixed냐 nonprefixed냐에 따라 네가지 구성이 나옴(글로벌 nonprefixed 조합은 지원하지 않음)
   - 비파티션 인덱스를 포함해 네가지 유형 존재
     1. 로컬 Prefixed 파티션 인덱스
     2. 로컬 Nonprefixed 파티션 인덱스
     3. 글로벌 Prefixed 파티션 인덱스
     4. 비파티션 인덱스
    
    - (즉 인덱스파티션 X 테이블파티션의 조합으로 [ 테이블(파티션, 비파티션) X 인덱스(비파티션, 로컬, 글로벌) ] 조합이 있고)
    - (인덱스 파티션의 글로벌 X prefix여부 조합으로 [ (로컬, 글로벌) X (Prefixed, Nonprefixed) + 비파티션 ] 조합이 있다.)

- 인덱스 파티션 제약
  - Unique 인덱스를 파티셔닝하려면 파티션 키가 모두 인덱스 구성 컬럼이어야 한다.
    - 파티션 키가 인덱스 구성 컬럼이 아니면 인덱스의 정렬과 파티셔닝의 나뉨이 규칙없이 흩어지기때문에 파티셔닝이 불가능.
  - 따라서 파티션 구조를 변경하여 PK인덱스가 Unusable로 바뀌고, 인덱스를 재생성하기 위한 Rebuild시 서비스 중단을 막기 위해서는 PK를 포함한 모든 인덱스가 로컬 파티션 인덱스여야 한다.


- 08.07(월) 436p ~ 456p
# (3)에서 계속
