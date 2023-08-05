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
    - INSERT 작업에 Redo 로깅 생략 가능한 이유

6. Undo(rollback) 로깅과 DML성능
  - Redo : 트랜잭션 재현에 필요한 정보 로깅
  - Undo : 변경된 블록을 이전 상태로 돌리는데 필요한 정보 로깅
  - 용도 : Transaction Rollback, Transaction Recovery(Instance Recovery의 rollback), Read Consistency
    - 읽기 일관성 Read Consistency : 동시 트랜잭션이 증가할수록 블록 IO 증가, 성능 저하
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
  - Insert Ito Select, 수정가능 조인 뷰(6.1.5), Merge문(6.1.6)을 적극 활용하여 One SQL로 구현
 
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





  
