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
  - select for update 문으로 내부적으로 결정되는 lock 메커니즘 외에 사용자가 결정할 수 있다. (nowait, wait 3)



- 08.11(금) 456p ~ 470p



- 커밋; Lock 해제
  - 블로킹 : 선행 트랜잭션의 Lock으로 인해 작업 진행을 대기하는 상태. 커밋/롤백으로 해소
  - 교착상태 : 두 개 이상의 트랜잭션이 하나의 리소스를 점유한 채 다른 리소스에 락을 설정하려 하는 상태가 무한히 지속되는 것
    - 발생 필요조건
      - 상호 배제 : 한 자원은 하나의 트랜잭션만 사용할 수 있다.
      - 점유와 대기 : 트랜잭션은 자원을 점유한 채 다른 자원을 기다릴 수 있다.
      - 비선점 : 다른 트랜잭션이 점유한 자원을 강제로 가져올 수 없다.
      - 순환 대기 : 트랜잭션 간 자원을 대기하는 순환이 있다.
    - 발생 방지를 위한 방법
      - 타임아웃 : 대기시간을 정함. 대기시간 후의 처리에 유의
      - 우선순위, 선점 : 설정에 유의
      - 데드락 탐지, 회복 : 데드락을 탐지하면 특정 트랜잭션을 롤백/재시작 -> 처리에 유의

  - 오라클은 데이터를 읽을 때 Lock을 설정하지 않지만, 그래도 트랜잭션을 불필요하게 길게 설정해선 안된다.
    - 롤백 수행이 길어짐
    - Undo 세그먼트 고갈/경합
    - 동시 트랜잭션을 최소화하는 어플리케이션 설계
    - DML Lock으로 인한 동시성 저하 없도록 적절한 시점의 커밋
  - 반대로 너무 많은 커밋은 다음과 같은 단점을 가진다.
    - LGWR이 로그버퍼를 비우는 동안 동기방식으로 기다리는 횟수가 늘어남
    - 10gR2부터 비동기식 커밋, 배치 커밋 존재
      - WAIT(default) : LGWR가 로그버퍼를 파일에 기록 완료할때까지 기다림(동기식)
      - NOWAIT : LGWR의 완료를 기다리지 않고 다음 트랜잭션 진행(비동기식)
      - IMMEDIATE(default) : 커밋 명령을 받을때마다 LGWR가 로그 버퍼에 파일 기록
      - BATCH : 세션에 트랜잭션 데이터를 버퍼링 했다가 일괄 처리
      - 다음과 같은 조합 존재
        ```sql
        COMMIT WRITE IMMEDIATE WAIT ;
        COMMIT WRITE IMMEDIATE NOWAIT ;
        COMMIT WRITE BATCH WAIT ;
        COMMIT WRITE BATCH NOWAIT ;
        ```

6.4.2 트랜잭션 동시성 제어
- 비관적 동시성 제어
  - 사용자들이 데이터를 동시수정을 가정. Lock 사용; 동시성에 악영향 줄 수 있음
  - for update를 사용하면 기본값은 무한정 기다리는 것이고 wait 3/ nowait 옵션을 사용하여 대기하는것을 관리할 수 있다.
- 낙관적 동시성 제어
  - 데이터를 동시에 수정하지 않을 것이라 가정. Lock 사용하지 않음
  - 단 읽는 시점엔 Lock을 사용하진 않아도 수정할 때는 읽었던 데이터가 변경돼었는지 확인해야한다.
  - ```sql
    select A, B, C, D into :a, :b, :c, :d
    from 고객
    where 고객번호 = :cust_num;

    -- 수정할 데이터 계산하는 쿼리

    update 고객 set 수정컬럼 = :입력값
    where 고객번호 = :cust_num
    and A = :a and B = :b and C = :c and D = :d;

    if sql%rowcount = 0 then alert("변경됨"); enf if;
    ```
  - 위와 같이 select의 많은 조건으로 수정 전 변경되었는지 파악하는 것은 매우 번거롭다.
  - a,b,c,d 조건을 '최종변경일시'컬럼으로 대체하여 수정전 최종변경일시가 최초 변경일시와 같은지 비교하여 갱신여부를 판단할 수 있다.
  - update전에 select문을 한번 더 실행함으로써 Lock 예외처리를 하면 다른 트랜잭션의 Lock을 기다리지 않을 수 있다.
  - ```sql
    select 고객번호
    from 고객
    where 고객번호 = :cust_num
    and 변경일시 = :mod_dt
    for update nowait;
    ```

- 동시성 제어 없는 낙관적 프로그래밍
  - 다양한 상황에서 원하지 않는 결과가 도출될 수 있다.
 
- 조인문에서 for update구문을 쓸 때
```sql
for update of t.column
```
이렇게 사용해야 select절에서 조회하고 조건에 해당되는 테이블에만 Lock이 설정된다.
- 큐Queue 테이블 동시성 제어
  - SKIP LOCKED옵션으로 lock이 걸린 레코드는 생략하고 진행할 수 있다.
    - rownum 조건을 제거하고 클라이언트가 정해진 개수를 읽으면 중단되게 해야함. Array Fetch 권장)

- 데이터 품질과 동시성 향상
  - 성능향상도 중요하지만 데이터 품질이 제일 중요하다.
  - FOR UPDATE의 정확한 사용 권장
  - 동시성을 해치지 않게 wait, nowait 옵션을 적극 활용, 예외처리
  - Lock을 적절한 길이로 유지하고 트랜잭션의 원자성이 유지되는 범위 내에서 빠른 커밋
  - 주간에 수행할 필요 없는 배치프로그램은 야간에 수행
  - 낙관적 동시성 제어를 시도하다가 다른 트랜잭션에 의한 변경이 감지되면 비관적 동시성 제어를 사용할 수도 있다.
  - 동시성을 개선하는 가장 기본적인 방법은 SQL튜닝. 올바른 조인방법과 효율적인 인덱스 구성. Array Processing, OneSQL... 그 후에 Lock에 대한 고민


----------------------

6.3.4 채번 방식에 따른 INSERT 성능 비교
- I U D M중 튜닝 요소가 가장 많은 것은 INSERT
  - 채번 방식에 따른 성능 차이가 매우 큼
 
- 채번 방식
  - 채번 테이블
  - 시퀀스 오브젝트
  - MAX + 1 조회
 
- 구분 속성 : t1_PK(상담원ID + 상담일자 + 상담순번) 에서 순번을 제외한 ID,일자를 구분속성이라 부르자.
- 채번 테이블
  - 구분 속성별 순번을 채번하기 위해 별도 테이블을 관리하는 방식
  - 채번 레코드을 읽어서 1을 더한 다음 다음 채번에 사용하는 방식
    - 방식으로 인한 자연스러운 직렿화 가능 : 중복 레코드 채번 원천 방지
    - 장점
      1. 범용성이 좋다.
      2. INSERT과정에 중복 레코드 발생에 대한 예외처리 하지 않아도 됨(편리)
      3. INSERT과정에 결번 방지
      4. PK가 복합컬럼이여도 사용 가능
    - 단점
      1. 다른 채번 방식에 비해 성능이 별로다.
        - 채번 레코드 변경 시 로우 Lock이 설정된다.
        - 동시INSERT가 많으면 테이블 블록에도 경합이 발생한다.(서로 다른 데코드를 변경하더라도 경합할 가능성이 존재)
     
    - 구분 속성의 레코드 수가 소수일 때만 채번방식을 사용한다.  (여전히 Lock 경합 발생 가능성 높음)




- ※ 8.14(월) ~481p



  - PL/SQL의 자율 트랜잭션
    - 메인 트랜잭션에 영향을 주지 않고 서브 트랜잭션에서 일부 자원만 Lock을 해제할 수 있음
      ```sql
      create or replace function seq_funcname(l_gubun number) return number
      as
        pragma autonomous_transaction; -- 자율트랜잭션으로 선언하여 일부 자원만 lock 해제하는 구문
        l_new_seq seq_tab.seq%type;
      begin
        update seq_tab
        set seq = seq + 1 -- 채번테이블
        where ~;

        select seq into l_new_Seq
        from seq_tab
        where ~;

        commit;
        return l_new_seq;
      end;
      ```
    - 자율트랜잭션의 내부에서 커밋을 수행해도 메인 트랜잭션은 커밋되지 않은 상태로 남는다.
    - 반면 채번테이블의 Lock은 해제한 상태이므로 다른 트랜잭션에 영향을 주지 않는다.
      ```sql
      insert into target_tab values (seq_nextval(123), :x, :y, :z);

- 시퀀스 오브젝트
  - 성능이 빠름
  - 중복 레코드 대비 예외처리 하지 않아도 됨
  - 반면 테이블별로 시퀀스 오브젝트를 생성하고 관리해야함
  - 시퀀스 채번 과정에서 Lock 발생으로 성능 이슈
  - 시퀀스 오브젝트는 오라클 내부에서 관리하는 채번테이블(SYS_SEQ$ 테이블, DBA_SEQUENCES뷰)
    - Lock메커니즘 작동
  - 캐시사이즈를 설정하면 가장 빠름
  - 자율 트랜잭션 구현됨
 
- 오라클의 시퀀스 오브젝트 Lock
  1. 로우 캐시 Lock
     - 딕셔너리 캐시, 로우단위로 IO하기 때문에 로우 Lock이라 부름
     - (딕셔너리 : 테이블, 인덱스, 테이블스페이스, 데이터파일, 세그먼트, 익스텐트, 사용자, 제약, 시퀀스, DB Link 등에 관한 정보)
     - 공유캐시(SGA)의 구성요소인 로우 캐시를 사용할 때 액세스를 직렬화 해야하고 이 때 Lock을 사용한다.
     - 채번빈도가 낮다면 CACHE설정을 높이고 채번빈도가 낮다면 NOCACHE옵션 사용
       ```sql
       create sequence MYSEQ cache 1000;
       ```
  2. 시퀀스 캐시 Lock
    - SGA에 위치, 액세스 직렬화용 
  3. SV Lock
    - 시퀀스 캐시는 한 인스턴스 내에서 공유되어 인스턴스 내에서의 번호 순서 보장
    - 인스턴스가 여러개인 RAC환경에서는 인스턴스마다 시퀀스 캐시를 따로갖고, 인스턴스 간 번호 순서 보장되지 않음
    - 이런 경우에 ORDER옵션을 사용하면 인스턴스 간에도 순서가 보장되며 시퀀스 캐시 하나로 모든 RAC 노드가 사용한다.
    - 이런 RAC환경에선 SV Lock을 통해 액세스 직렬화
    - SV Lock은 네트워크를 통해 공유되는 시퀀스 캐시 락
    - 성능적으로는 바람직하지 않다.

- 시퀀스는 기본적으로 PK가 단일컬럼이어야 한다.(단점)
  - 복합컬럼에 사용시 레코드의 최소성 위반
  - PK가 복합컬럼인데 동시 트랜잭션이 높아 시퀀스가 필요하다면 cycle옵션을 가진 시퀀스 활용 가능
    - 하루에 도달할 수 없는 값을 maxvalue로 설정하고 도달 시 다시 1부터 시작하는 순환옵션
    - 컬럼 길이를 최소화할 수 있음. 권장되진 않으나 성능문제가 있을 시 고려
- 다음과 같은 경우에 결번 발생
  1. 시퀀스 채번 이후 롤백 시
  2. CACHE옵션을 설정한 시퀀스가 캐시에서 밀려난 경우
    - 자주 사용하지 않아서 캐시에서 밀려나거나 인스턴스를 재기동하면 캐시돼 있던 번호가 사라지며 디스크상에서의 다음 번호부터 읽음
      - 인스턴스 재기동을 피하려 NOCACAHE옵션을 사용하면 성능문제
      - 사용 빈도가 낮아서 생기는 결번은 다음 코드를 실행하여 회피
        ```sql
        EXEC SYS.DBMS_SHARED_POOL.KEEP('table.my_seq', 'Q');
        ```
      - 하지만 결번 자체가 문제가 되는 시스템은 잘 없으며 데이터 삭제시의 결번도 피할 수 없다.
     
---------------
- MAX+1 조회
  - 대상테이블의 최종 일련번호에 +1 하여 INSERT하는 방식
    ```sql
    insert into 상품거래(거래일련번호, a, b, c, ...)
    values ( (select max(거래일련번호) +1 from 상품거래 )
            , :a, sysdate, :c, :d, ... );
    ```
  - 장점
    1. 시퀀스/채번테이블을 관리해야하는 부담이 없음
    2. 동시 트랜잭션에 의한 충돌이 많지 않으면 성능이 매우 빠름
    3. PK가 복합컬럼인 경우, 구분 속성별 순번을 채번할 때도 사용 가능.
       - 값의 수가 아무리 많아도 상관 없고, 오히려 많을수록 성능이 좋아짐
         - 입력 값 중복에 의한 로우 Lock 경합이 줄고 재실행 횟수도 감소
  - 단점
    1. 레코드 중복에 대비한 세밀한 예외처리 필요
    2. 다중 트랜잭션에 의한 동시 채번이 심할 시 성능이 매우 감소
       - 레코드 중복에 의한 로우 Lock 경합 때문.
         - 선행 트랜잭션이 커밋/롤백할 때 까지 로우 Lock은 지속됨
       - INSERT 실패 시 채번과 INSERT를 반복실행해야하므로 성능 감소
       - 구분속성별 값의 수가 많으면 구분 속성 값별로 채번이 분산되므로 로우 Lock 경합 및 재실행 가능성 감소
    3. MAX값 조회에 최적화된 인덱스 필요(5.3.3)
  - 각 채번 방식에서 발생하는 Lock 경합 요소(표6-3)
![image](https://github.com/DGUN52/Learn-SQLP/assets/125403830/245acbe5-a54d-48b9-a789-65a819704d5c)
![image](https://github.com/DGUN52/Learn-SQLP/assets/125403830/033df30c-ac31-490f-922c-65e4e1803f16)

  - 선택기준
    1. 다중 트랜잭션에 의한 동시 채번이 적으면 셋 다 비슷하지만 MAX+1방식이 관리부담이 없다.
    2. ~에 의한 동시 채번이 많고 PK가 단일컬럼 일련번호면 시퀀스 방식이 best
    3. ~에 의한 동시 채번이 많고 PK 구분 속성의 값 종류 개수가 많으면 MAX+1 방식이 right.(Lock 경합 및 재실행 가능성 낮음)
    4. ~에 의한 동시 채번이 많고 PK 구분 속성의 값 종류 개수가 적으면 cycle 옵션을 가진 시퀀스 오브젝트 활용 고려
   
- 12c 신기능
  1. 컬럼 기본값으로 시퀀스 지정
     ```sql
     create sequence my_seq;

     create table t(
       c1 number default my_seq.nextval not null
     , c2 varchar2(5)
     );
     ```
     - default값이 있으므로 직접 입력하지 않아도 되지만 직접 입력 가능
       ```sql
       insert into t(c1,c2) values (my_seq.nextval, 'X');
       insert into t(c2) values('X');
       ```
  2. IDENTITY 컬럼
     - 특정 컬럼을 IDENTITY컬럼으로 지정 가능
       ```sql
       create table t (c1 number generated always as identity, c2 varchar2(5));

       insert into t (c2) values('X');
       insert into t (c1, c2) values (default, 'X');
       ```
     - identity 컬럼으로 지정된 c1에 직접 값 입력할 시 에러발생
     - 아래 옵션으로 지정하면 직접 입력 가능
       ```sql
       create table t (c1 number generated by default as identity, c2 varchar2(5));
       ```
  3. 세션 시퀀스
    - 여러 세션이 공유할 수 없는 세션 레벨 시퀀스
      ```sql
      create sequence s_seq SESSION;
      ```
    - 세션이 종료되면 초기화됨
    - Lock 메커니즘이 불필요하여 글로벌 시퀀스보다 성능이 좋음
      - 따라서 여러 세션이 호출하지 않는 테이블에 적용하여 성능 향상

- 시퀀스보다 좋은 솔루션
  - 한 개 이상의 구분속성과 함께 뒤쪽에 순번 대신 입력일시(주문일시, 계약일시...) 를 두는 방식으로 PK구조를 설계하면 채번/INSERT과정에 생기는 Lock 이슈가 거의 해소됨
    - 채번과정 대신 SYSDATE/SYSTIMESTAMP만 호출하면 되기 때문
    - 구분 속성이 많으면 DATE타입만으로도 중복되지 않지만, 구분속성이 적으면 TIMESTAMP 사용으로 더 세밀한 구분가능.(중복 방지)
      - 그래도 중복에 대한 예외처리를 해주어야함
  - 빠르게 입력하고 빠르게 삭제하고 시스템에 공간을 반납할 수 있는 구조를 갖춰야한다.
    - 입력일시가 PK에 포함되는것은 바람직하다
      - 파티션활용 대량 DELETE에서 서비스 중단없이 대량 DELETE를 수행하려면
      - 파티션을 커팅하는 기준과 삭제 기준 컬럼이 PK에 포함되있어야하고(=인덱스가 로컬파티션이고 삭제기준 컬럼이 PK에 포함되있어야하고)
      - 많이 사용되는 삭제기준은 입력일시 컬럼이기 땜ㄴ
- 인덱스 블록 경합
  - INSERT성능이 너무 빨라도 인덱스 경합으로 성능 이슈 발생
  - 입력일시가 인덱스 선두컬럼인 경우 맨 우측 블록에만 데이터가 입력됨(right growing index)
  - 다량 INSERT시 블록을 갱신하려는 프로세스 간 버퍼 Lock 경합 발생으로 트랜잭션 성능 감소
  - RAC환경에서 특히 심각한 성능저하
    - 여러 노드가 동시에 Current 블록 하나를 주고받기때문
  - 인덱스 블록 경합을 줄이는 일반적인 방법은 **인덱스를 해시 파티셔닝 하는 것**
    - 혹은 인덱스를 리버스 키 인덱스로 전환
   
- 12c 시퀀스 신기능
  ```sql
  create sequence g_seq global;
  create sequence s_seq session;
  ```
  - 글로벌 시퀀스는 데몬프로세스/커넥션 풀 프로세스가 DB에 접속하는 순간 g_seq.nextval을 호출
  - 세션 시퀀스는 INSERT를 수행할때마다 호출
  - 글로벌 시퀀스의 crrval값과 세션 시퀀스의 nextval을 조합한 값으로 INSERT
    ```sql
    insert into t(id, c1, c2) values
    (to_char(g_seq.currval, 'fm0000') || to_char(s_seq.nextval, 'fm0000'), 'A', 'B');
    ```
  - 서로다른 리프 블록에 값을 입력하므로 인덱스 경합 발생하지 않음
  - 18c에선 Scalable 시퀀스 옵션
    - 인스턴스 번호, 세션id, 시퀀스 번호를 조합한 번호를 반환
      ```sql
      create sequence my_seq maxvalue 9999 SCALE EXTEND;
      ```
      - EXTEND 생략 시 leading zero 없는 숫자 반환
    - 하위 버전에서도 해당 값들을 조합하여 같은 기능 구현 가능
   

- 08.15(화) 482p ~ 500p
