6.1 기본 DML 튜닝
- DML 성능에 영향을 미치는 요소
  1. 인덱스
  2. 무결성 제약
  3. 조건절
  4. 서브쿼리
  5. Redo 로깅
  6. Undo 로깅
  7. Lock
  8. 커밋
 
1. 인덱스와 DML성능
  - 레코드 삽입 시
    - 테이블에는 Freelist를 이용한 삽입가능
    - 인덱스는 수직 탐색 후 삽입
  - 레코드 DELETE 시
    - 테이블 레코드 삭제
    - 인덱스 레코드 삭제
  - 레코드 UPDATE 시
    - 테이블에 해당하는 건만 변경
    - 기존 인덱스 삭제 후 변경된 위치에 삽입
  - 따라서 인덱스 개수는 DML성능에 큰 영향을 미친다.

2. 무결성 제약과 DML성능
  - 개체entity 무결성
  - 참조referential 무결성
  - 도메인domain 무결성
  - 사용자 정의 무결성(or 업무 제약 조건)
  - 이 외에도 PK, FK, Check, Not Null제약 설정으로 데이터 무결성 향상 가능

<table>
  <th><td>PK제약/인덱스</td><td>일반 인덱스</td><td>소요시간</td></th>
  <tr><td></td><td>O</td><td>O</td><td>38.98초</td></tr>
  <tr><td></td><td>O</td><td>X</td><td>4.95초</td></tr>
  <tr><td></td><td>X</td><td>X</td><td>1.32초</td></tr>
</table>
  - 이처럼 무결성 제약은 성능은 느리게 만든다.

3. 조건절과 DML성능
  - 2, 3장에서 배운 원리

4. 서브쿼리와 DML성능
  - 4장에서 배운 원리 (※4.4 서브쿼리 조인)

5. Redo 로깅과 DML성능
  - 오라클은 데이터파일, 컨트롤 파일의 모든 변경사항을 Redo로그에 기록
    - Redo로그는 트랜잭션을 재현함으로써 트랜잭션 유실 시 이전 상태로 복구용
      - DB 복구, 캐시 복구(Instance Recovery의 roll forward), Fast commit
        - fast commit : 데이터 변경 시 디스크의 데이터 블록에 반영하는 작업은 랜덤 액세스 방식으로 느리기 때문에, 로그에 append 방식으로 기록해둔 뒤 DBWR같은 방식을 이용 해서 후에 Batch방식으로 일괄 수행할 수 있다.
  - DML수행마다 Redo로그가 생성된다. -> DML 성능 영향
    - INSERT 작업에 Redo 로깅 생략 가능한 이유 -> 뒤의 6.2.2에서 설명

6. Undo(rollback) 로깅과 DML성능
  - Redo : 트랜잭션 재현에 필요한 정보 로깅
  - Undo : 변경된 블록을 이전 상태로 돌리는데 필요한 정보 로깅
  - 용도 : Transaction Rollback, Transaction Recovery(Instance Recovery의 rollback), Read Consistency
    - 읽기 일관성 Read Consistency모드가 성능에 주는 영향 : 동시 트랜잭션이 증가할수록 RC모드에 의해 블록 IO 증가, 성능 저하
    - Consistent모드 : 쿼리가 시작된 이후 다른 트랜잭션이 변경한 블록을 만나면 원본 블록의 사본을 만들고 Undo 데이터를 적용해 블록의 처음 시점을 읽는 방식
      - 이 모드와 Undo 데이터로 인해 원본 블록 하나가 여러 복사본으로 캐시에 존재할 수 있다.
    - Current모드 : 디스크에서 캐시로 적재된 원본 블록을 현재 상태 그대로 읽는 방식
    - 블록 SCN(System Commit Number) : 트랜잭션이 커밋할 때마다 / 백그라운드 프로세서가 조작할 때마다 1씩 증가하고 블록에 각각 저장.
    - 쿼리 SCN : 쿼리의 읽기 작업 시작 시 글로벌 변수인 SCN을 읽고 시작하는 것
    - Consistent 모드(다시 설명) : 쿼리SCN과 블록SCN을 비교하여 블록이 변경됐는지 확인하면서 읽는 모드.
      - 블록SCN>쿼리SCN 이면 블록이 변경된 것
      - Select문은 거의 항상 Consistent 모드로 읽고 Insert/Update/Delete는 Current모드로 수행한다. (데이터 변경은 사본 블록이 아닌 원본 블록에 하기 위해)

7. Lock과 DML성능
  - 데이터 품질, Lock Level, 트랜잭션 격리성 수준 vs 성능 은 Trade off 관계
  - 세밀한 동시성 제어로 둘 다 만족시킬 순 있다.
  - 동시성 제어 Concurrency Control : 동시 실행되는 트랜잭션 수를 최대화하면서 Insert, Update, Delete, Select의 데이터 무결성을 유지하는 것

8. 커밋과 DML성능
  - Commit을 통해 Lock을 풀고 다음 트랜잭션이 실행될 수 있게 한다.
  - 모든 DBMS는 Fast Commit을 구현하고 있다.
  - 커밋의 내부 메커니즘
    1) DB버퍼캐시 : DBWR가 버퍼캐시에 저장된 변경된 블록(dirty block)을 모아 데이터파일에 기록(Batch방식)
    2) Redo 로그버퍼 : 버퍼캐시에 반영된 내용은 Redo 로그에도 기록돼있다.(버퍼캐시가 유실되어도 복구 가능)
      - Redo 로그도 IO가 필요한 느린 방식이기 때문에 로그버퍼에 먼저 기록 후 LGWR가 로그파일에 기록(Batch방식)
    3) 트랜잭션 데이터 저장 과정
      1. Redo 로깅 : DML문 실행, Redo 로그버퍼에 기록
      2. 레코드 변경 : 버퍼블록에 데이터 Manipulation(버퍼캐시에 블록이 없다면 IO 발생)
      3. 커밋
      4. LGWR가 로그버퍼를 로그파일에 일괄 저장
      5. DBWR가 변경된 버퍼블록을 데이터파일에 일괄 저장
      - 즉 커밋은 Redo 로그버퍼와 버퍼블록 조작이 끝났다는 것을 확인하는 것
      - LGWR과 DBWR은 커밋 외에도 주기적으로 로그버퍼, 버퍼블록(Dirty 블록)을 데이터파일에 기록
      - 따라서 버퍼블록이 디스크에 온전히 기록되지 않았더라도 그 전에 저장한 로그파일로 트랜잭션의 영속성 보장

    4) 커밋=저장 버튼
      - 커밋은 그전까지의 작업을 디스크에 기록하라는 명령어
      - LGWR가 작업을 완료하기 전까지 다음 작업 진행 불가(Sync방식)
      - 트랜잭션이 너무 길어 커밋을 늦게하는 것도 문제(Undo 공간 부족), 너무 자주하는 것도 문제.
   
6.1.2 데이터베이스 Call과 성능
- SQL의 수행 세 단계
  1. Parse Call : SQL파싱, 최적화 단계. 라이브러리에 sql과 실행계획이 있다면 단계 생략 가능
  2. Execute Call : SQL 실행 단계. Select문은 3단계까지 계속된다.
  3. Fetch Call : 사용자에게 결과집합을 전송하는 과정(Select문 전용). 데이터가 많으면 여러번 발생.
- Call발생위치에 따른 분류
  1. User Call : 외부에서 네트워크를 경유해서 DBMS로 들어가는 Call.
    - 실제로는 Client가 요청하지만 DBMS입장에선 WAS(ap서버)가 발생시킨다.
  2. Recursive Call : dbms 내부에서 발생하는 Call.
     - SQL파싱, 최적화에서 발생하는 데이터 딕셔너리 조회, PL/SQL 함수/프로시저/트리거에 내장된 SQL 실행시 발생하는 Call
- 모든 종류의 call은 성능을 저하시킨다. User Call은 특히 더욱.

- 절차적 루프 처리
  - loop문 안에 commit;을 둘 경우 성능 저하 (필요한 경우 10만번에 한번과 같은 조건을 두기)
- OneSQL의 중요성
  - Insert Into Select, 수정가능 조인 뷰(6.1.5), Merge문(6.1.6)을 적극 활용하여 One SQL로 구현
 
6.1.3 Array Processing 활용
- Call 발생량이 많을 시 배열에 모아서 한번에 execute

6.1.4 인덱스 및 제약 해제를 통한 대량 DML튜닝
- 온라인 트랜잭션 처리 시스템OLTP에서 제약들을 해제할 순 없지만
- 동시 트랜잭션이 없는 배치 프로그램에선 pk제약 해제, 무결성 체크 생략하도록 설정하여 성능효과를 얻을 수 있다.
- pk제약을 해제하고 배치프로그램을 수행한 후 다시 pk제약을 활성화하면 pk인덱스가 자동으로 생성된다.

- pk제약과 인댁스 해제, 2-PK제약에 Non-Unique 인덱스를 사용한 경우
  - pk인덱스를 Drop하지 않고 Unusable 상태에서 데이터를 입력하고 싶다면 PK제약에 Non-Unique 인덱스를 사용하면 된다.
    - (자동으로 생성되는 pk인덱스 대신 수동으로 인덱스를 생성하여 pk인덱스로 지정해주면 된다)
    - create index target_pk on target(a, b);
    - alter table target add constraint target_pk primary key(a, b) using index target_pk;
    - alter table target modify constraint target_pk disable keep index;
    - alter table index target_pk unusable; -- 원래는 pk를 unusable로 설정할 수 없지만 non-unique index라 가능
    - alter table index target_x1 unusable; -- 추가 인덱스 비활성화


    
- 08.05(토) 393p ~ 421p


6.1.5 수정가능 조인 뷰
- 전통적인 방식의 UPDATE문을 활용하면 비효율을 완전히 해소할 수 없다.
  - 같은 테이블을 두번 조회하는 등의 비효율이 존재
- 수정가능 조인 뷰 활용 시(12c이후)
  - 조인뷰 : from절에 두 개 이상의 테이블을 가진 뷰
    - 이 조인뷰가 입력, 수정, 삭제가 가능하면 수정가능 조인 뷰
    - 단 1:M에서 m쪽에만 수정 가능
      - ex) emp(m)와 dept(1) 테이블을 합친 뷰에서 job(m)="CLERK"인 레코드의 loc(1)="SEOUL'로 수정하면 원하지 않는 레코드의 loc까지 변경될 것이다.
    - pk제약이나 unique index가 설정되지 않으면 조인뷰에 입력/수정/삭제가 불가능하다.(DBMS에러)
      - alter table A add constraint A_pk primary key (col_id);
      - 위처럼 1쪽 집합에 pk를 설정 해준 다음 조작 시 정상작동한다.
      - pk설정을 해준 1쪽집합은 비 키-보존 테이블이 되고 m쪽집합 테이블은 키-보존 테이블로 남는다.
     
- 키 보존 테이블 Key-Preserved Table
  - 조인된 결과집합을 통해서도 중복 값 없이 유일하게 식별 가능한 테이블
  - 조인 뷰의 조인하는 테이블에 pk 혹은 최소 unique 인덱스가 있어야 조인뷰를 수정할 수 있다.
  - (!!!키가 보존된 쪽의 레코드만 수정할 수 있다는 말!!!)
  - 위의 예시에서 dept 테이블에 pk를 걸어주며 조인 시에는 unique한 값들과 조인하는 emp테이블이 키-보존 테이블이 된다.
  - (emp테이블에도 pk/unique index가 설정돼있어야 emp테이블의 키도 보존된다.)
  - (반대로 1쪽에 pk/unique index가 설정돼있어도 m쪽 집합에서 1쪽 레코드를 다량 참조하기 때문에 키가 보존되지 않을 수 있다. 다만 이는 조인뷰가 group by (1쪽 테이블의 pk/unique 컬럼) 이런식으로 조인뷰에서 1쪽 테이블의 pk가 중복되지 않게 쿼링되면 1쪽의 키는 보존된다고 볼 수 있다.)
  - 설정들로 인해 DBMS가 수정불가능한 조인뷰 Non updatable join view라고 판단하면 에러가 발생할 수 있음
    - /*+ byass_ujvc */ 힌트를 사용하여 에러 회피(11g이후)
    - 혹은 MERGE문으로 바꾸어야 한다. 다만 에러 회피시 데이터의 무결성 보존에 유의해야한다.

6.1.6 MERGE문 활용
- Data Warehouse에서 자주 발생하는 오퍼레이션
  - 기간계 시스템에서 가져온 신규 트랜잭션 데이터를 반영하여 두 시스템 간의 데이터 동기화 작업
    - ex) 고객 테이블의 변경데이터를 dw에 반영하는 프로세스 (추출 전송 적재)
      1. 전 일 변경 데이터 추출
         ```sql
         create table customer_delta
         as
         select * from customer
         where mod_dt >= trunc(sysdate)-1
         and mod_dt < trunc(sysdate);
         ```
      2. 생성한 customer_delta 테이블을 DW시스템으로 전송
      3. DW시스템으로 적재
        ```sql
        merge into customer t using customer_delta s on (t.cust_id = s.cust_id)
        when matched then update
          set t.cust_nm = s.cust_nm, t.email = s.email, ...
        when not matched then insert
          (cust_id, cust_nm, email, tel_no, region, addr, reg_dt) values
          (s.cust_id, s.cust_nm, s.email, s.tel_no, s.region, s.addr, s.reg_dt);
        ```
    - 이처럼 update와 insert가 섞인 MERGE문을 UPSERT라고 하기도 한다.

  - Optional Clauses (UPDATE와 INSERT의 선택적 처리)
    - 일괄 처리가 아닌 별개의 처리
    ```sql
    merge into customer t using customer_delta s on (t.cust_id = s.cust_id)
    when matched then update
      set t.cust_nm = s.cust_nm, t.email = s.email, ... ;

    merge into customer t using customer_delta s on (t.cust_id = s.cust_id)
    when not matched -- (insert문 생략)
    ```
  - 이 머지문을 통해 updatable join view를 대체할 수 있다.

- Conditional Operations
  - 위의 on 절에 건 조건 외에도 추가로 조건절을 기술할 수 있다.
  - update/insert문 마지막에 where문 추가를 통해서 조건 추가 기술.
 
- DELETE CLAUSE
  - 이미 저장된 데이터를 조건에 따라 삭제
  - merge문에서 delete문 추가
    - update문 바로 뒤에 delete문을 추가 하는 경우 update문이 반영된 결과 기준으로 delete한다.(null값 주의)
    - 또한 조인이 성공된 데이터만 삭제 가능.(update가 delete 바로 위에 있을 경우 update도 마찬가지로 조인 성공된 데이터만..)

- MERGE문을 활용하면 기존 select ~; if (condition) then insert ~ else update ~ end if; 구문에선 항상 두번씩 쿼리를 수행해야하던 것을 한번 수행으로 끝낼 수 있다.
- INSERT 없는 MERGE문을 사용하는 경우 USING절에 검증용 SELECT문을 만들 경우 테이블을 두번 조회한다.(select 한번 update 한번)
  - 이럴 때 select절 없는 merge문이나 update문(수정가능 조인 뷰 이용)을 이용하여 성능을 향상시키는 것이 바람직하다.



<h2>6장 내용이 방대함으로 md파일을 part 1과 2로 나누어 기록</h2>
- 08.06(일)(1) 422p ~ 435p











