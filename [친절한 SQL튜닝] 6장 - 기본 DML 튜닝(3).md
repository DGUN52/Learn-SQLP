6.3.3 파티션을 활용한 대량 UPDATE 튜닝
- 대량데이터를 I U D 할땐 인덱스를 drop/unusable하게 변경한 뒤 작업하기도 함
  - 손익분기점은 5%정도 (전체데이터의 5%이상을 I U D 하면 인덱스를 조정한 뒤 작업하는게 이득)
- 파티션 Exchange를 이용한 대량 데이터 변경
  - 테이블이 파티셔닝 돼있고 인덱스도 로컬 파티션이라면
    - 수정된 값을 갖는 임시 세그먼트를 원본 파티션과 바꾸는 방식
    1. 임시 테이블 생성
    ```sql
    create table 거래_t nologging
    as
    select * from 거래 where 1 = 2;
    ```
    2. 거래 데이터를 임시 테이블에 입력하면서 UPDATE
    ```sql
    insert /*+ append */ into 거래_t
    select 고객번호, 거래일자, 거래순번, ... , (case when 상태코드<>'zzz' then 'zzz' else 상태코드 end) 상태코드 -- 상태코드가 zzz가 아닌것을 zzz로 변경
    from 거래
    where 거래일자 < '20150101';
    ```
    3. 임시 테이블에 원본 테이블과 같은 구조로 인덱스 생성. 가능하면 nologging모드
    ```sql
    create unique index 거래_t_pk on 거래_t (고객번호, 거래일자, 거래순번) nologging;
    create index 거래_t_x1 on 거래_t(거래일자, 고객번호) nologging;
    create index 거래_t_x2 on 거래_t(상태코드, 거래일자) nologging;
    ```
    4. 임시테이블과 원본테이블 exchange
    ```sql
    alter table 거래
    exchange partition p201412 with table 거래_t
    including indexes without validation;
    ```
    5. 임시테이블 drop
    6. nologging 모드로 작업했을 시 파티션을 logging모드로 전환

6.3.4 파티션을 활용한 대량 DELETE 튜닝
- 대량 데이터의 DELETE또한 막대한 시간자원이 소모된다.
- 테이블 인덱스의 비사용 > 작업 > 재생성도 고비용
- UPDATE는 대상컬럼이 있는 파티션의 인덱스만 재생성하면 되지만 DELETE는 모든 인덱스를 재생성해야함
  - delete가 느린 이유
    1. 테이블 레코드 삭제
    2. Undo Logging
    3. Redo Logging
    4. 인덱스 레코드 삭제
    5. Undo Logging
    6. Redo Logging
    7. 2번과 5번에 대한 Redo Logging
   
- 파티션 Drop을 이용한 대량 데이터 삭제
  - 삭제조건이 파티션과 맞아떨어진다면 간단하게 drop 가능
  ```sql
  alter table 거래 drop partition p201412;
  ```
  - 오라클 11g부터 가능한 값 기준 파티션 지정
  ```sql
  alter table 거래 drop partition for('20141201');
  ```

  - 파티션 Truncate를 이용한 대량 데이터 삭제
    - delete 조건에 해당하는 데이터가 적으면 그대로 통상적인 delete문을 사용하면 된다.
    - 조건에 해당하는 데이터가 대량이면 남길 데이터만 백업한 다음 지우고 재입력하는 방식이 빠름.
    1. 임시 테이블 생성, 남길 데이터 복제
    ```sql
    create table 거래_t
    as
    select *
    from 거래
    where 거래일자 < `20150101`
    and 상태코드 = 'ZZZ'; -- 복제쿼리
    ```
    2. 삭제 대상 테이블 파티션 Truncate(공간은 남겨둠)
    ```sql
    alter table 거래 truncate partition p201412; -- 오라클 11g의 값 기준 파티션 지정 가능
    ```
    3. 임시 테이블의 복제데이터를 원본 테이블에 입력
    ```sql
    insert into 거래
    select * from 거래_t
    ```
    4. 임시 테이블 drop
    ```sql
    drop table 거래_t;
    ```

  - 서비스 중단없이 DROP/TRUNCATE를 하려면 아래 조건 필요
    1. 파티션 키와 커팅 기준 컬럼 일치
    2. 파티션 단위와 커팅 주기 일치
    3. 모든 인덱스가 로컬 파티션 인덱스여야함
   
6.3.5 파티션을 활용한 대량 INSERT 튜닝
- 비파티션 테이블
  - 인덱스 Unusable, INSERT, 인덱스 재생성
  ```sql
  alter table target_t nologging;
  alter index target_t_x01 unusable;
  insert /*+ append */ into target_t
  select * from source_t;
  alter index target_t_x01 rebuild nologging;
  alter table target_t logging;
  alter index target_t_x01 logging;
  ```
- 파티션 테이블
  - 대용량 테이블일 경우 웬만하면 인덱스 Unusable 설정을 하지 않고 INSERT.
    - 하지만 파티션 테이블, 로컬 인덱스 파티션이라면 파티션 단위로 해결 가능
  - 파티션 단위로 인덱스를 unusable, insert, 재생성한다.

 
6.4 Lock과 트랜잭션 동시성 제어
- 사용하는 DB의 고유 Lock 메커니즘을 이해해야 한다.

6.4.1 오라클 Lock
- **DML Lock**, DDL Lock, 래치, 버퍼Lock, 라이브러리 캐시 Lock/Pin 등
  - 래치 : SGA에 공유된 각종 자료구조 보호용
  - 버퍼Lock : 버퍼 블록에 대한 액세스 직렬화
  - 라이브러리 캐시Lock/Pin : 라이브러리 캐시에 공유된 SQL커서, PL/SQL프로그램 보호

- DML 로우 Lock
  - 두 개의 트랜잭션이 같은 로우 변경을 방지
    - 아직 커밋하지 않은 로우를 다른 트랜잭션이 조작(U/D)할 수 없다.
    - INSERT의 경우엔 Unique 인덱스가 있을 때만 Lock 경합 발생
    - 오라클은 SELECT에 Lock을 사용하지 않음(복사본을 만들어서 읽음)
    - MVCC 모델을 사용하지 않는 DBMS는 SELECT문에 공유 Lock 사용.
      - 공유Lock 끼리는 동시에 설정될 수 있지만 배타적 Lock과는 호환되지 않음
      - 공유 Lock과 배타적 Lock이 서로 방해하지 않으려면 Lock의 길이를 짧게 가지도록 커밋 시점을 당겨야 한다. (SQL을 튜닝해야 한다.)
     
- DML 테이블 Lock
  - (오라클)DML Lock 이전에 테이블 Lock을(TM Lock) 설정
    - 로우Lock은 항상 배타적, 테이블 Lock은 다양한 모드
    - <table>
      <th>
        <td>Null</td><td>RS</td><td>RX</td><td>S</td><td>SRX</td><td>X</td>
      </th>
      <tr>
        <td>Null</td><td>O</td><td>O</td><td>O</td><td>O</td><td>O</td><td>O</td>
      </tr>
      <tr>
        <td>RS</td><td>O</td><td>O</td><td>O</td><td>O</td><td>O</td><td></td>
      </tr>
      <tr>
        <td>RX</td><td>O</td><td>O</td><td>O</td><td></td><td></td><td></td>
      </tr>
      <tr>
        <td>S</td><td>O</td><td>O</td><td></td><td>O</td><td></td><td></td>
      </tr>
      <tr>
        <td>SRX</td><td>O</td><td>O</td><td></td><td></td><td></td><td></td>
      </tr>
      <tr>
        <td>X</td><td></td><td></td><td></td><td></td><td></td><td></td>
      </tr>
    </table>
    - RS: row share (or SS: sub share)
    - RX: row exclusive (or SX: sub exclusive)
    - S: share
    - SRX: share row exclusive (or SSX: share/sub exclusive)
    - X: exclusive
  - 이 테이블 lock 종류에 따라서 후행 트랜잭션이 테이블에서 할 수 있는 작업의 범위가 결정됨
  - select for update 문으로 내부적으로 결정되는 lock 메커니즘 외에 사용자가 결정할 수 있다.

- 08.11(금) 456p ~ 470p
