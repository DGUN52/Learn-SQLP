## SQL수행구조
- 로그버퍼에 기록하는건 사용자의 DML프로세스
- DML → **[로그버퍼(redo)]** - LGWR → **[redo-log]** - DBWR/Checkpoint → **[DATA FILE]**
- redo log
  - 두 개의 파일로 구성되며, 하나의 파일이 꽉 찰 때마다 log switch 및 디스크에 기록
  - 용도
    1. Instance Recovery
    2. DB Recovery
    3. Fast Commit
  - 생성원리
    1. Write ahead logging
    2. log force at commit
- undo log
  - 용도
    1. Transaction Rollback
    2. Transaction Recovery
    3. Read Consistency
    - Snapshot too old
- 버퍼의 상태 3가지 : Dirty, Free, pinned

## SQL수행도구
- **실행계획 생성 및 조회**(오라클)
  - ```sql
    explain plan for
    select ~ -- 실행계획을 확인하고자하는 쿼리
    ```
  - 위 명령어를 실행하면 plan_table에 저장됨
  - ```sql
    select * from table( dbms_xplan.display(null, null, 'typical') ); -- typical 대신 'alias', 'outline', 'advanced' 등 지정 가능
    ```
  - 위 명령어로 plan_table 포매팅, 조회
- **실행계획 생성 및 조회**(SQL Server)
  - ```sql
    go
    set showplan_text on
    go
    select ~ -- 실행계획 확인하고자 하는 쿼리
    go
    ```
- **SQL TRACE 분석 및 리포트파일 생성**(오라클)
  - alter session set sql_trace = true
  - tkprof tracefile.trc reportfile.prf sys=no
  - ```sql
    go
    set statistics profile on
    set statistics io on
    set statistics time on
    go
    select * from ~
    go
    ```
- 성능 분석 방법론
  1. Ratio 기반
  2. 응답 시간 분석
  3. Statspack
  4. AWR Automatic Workload Repository
  5. ASH Active Session History - dba_hist_active_ses_history : 오래전에 사용된 세션 히스토리 정보
 
## 인덱스 튜닝 - 인덱스 기본 원리
- 인덱스 구성 컬럼이 전부 not null 이 아닌 nullable이라면 index Range scan을 사용할 수 없다.
  - 전부 null인 경우가 인덱스에 저장되지 않아 쿼리결과에서 값이 누락되기 때문

## 인덱스 튜닝 - 테이블 액세스 최소화

## 조인 튜닝
### 고급 조인 기법
- decode함수나 case문법을 사용하여 조건을 만족하는 값이 없을 경우 null값을 반환하게 하면 **조건을 만족하는 경우에만 조인을 시도하는 쿼리**를 작성할 수 있다!

- 유용한 인라인 뷰 사용법(nl조인이나 hash조인, 부분범위 처리, 소트부하 감소 등 다양한 용도로 사용)
- ```sql
  SELECT /*+ LEADING(A) USE_NL(B)*/
    A.~, A.~, A.~, ...
    B.~, B.~, ... 
  FROM (SELECT /*+ FULL(A) INDEX_FFS(B) LEADING(B) USE_HASH(A) */ /* 필요에 따라 알맞은 힌트 제시, 인라인 뷰에는 모든 기술이 들어간 쿼리 기재 */
          A.~, MAX(B.~), SUM(A), ...
        FROM tableA A, tableB B
        WHERE A.colA = ~ /* 인덱스와 요구사항에 알맞은 where절 기술 */
        GROUP BY A.colA
        ORDER BY) A, tableB B2
  WHERE A.ROWID = B.ROWID /* 혹은 A.colA = B.colB 와 같은 조인 */
  ORDER BY A.colD ASC/DESC
  ;
  ```

- 이력테이블에서 가장 최근 상태를 구하는 쿼리 형태
  - ```sql
    SELECT A.colA, A.colC, ..., B.colD, ... /* 문제에서 요구하는 필요한 인자들*/
    FROM tableA A, tableB B
    WHERE colB = :colB -- 특정 인자에 대한 사전 조건(들)
      and B.ROWID = (
      SELECT rid
      FROM(
        SELECT ROWID rid
        FROM tableB
        WHERE A.colA = colA -- 인덱스 고려 조인컬럼
        ORDER BY /* 부분범위 처리 가능하게 할 인덱스 인자 */
      )
      WHERE A.rownum <= 1
    )
    ORDER BY A.colA
    ```

- 이력테이블에서 가장 최근 상태의 직전 상태를 구하는 쿼리 형태
  - ```sql
    SELECT A.colA, A.colC, ..., B.colD, ... /* 문제에서 요구하는 필요한 인자들*/
    FROM tableA A, tableB B
    WHERE colB = :colB -- 특정 인자에 대한 사전 조건(들)
      and B.ROWID = (
      SELECT rid
      FROM(
        SELECT ROWID rid
        FROM tableB
        WHERE 변경일자 < A.최종상태변경일자 -- 인덱스 고려 조인컬럼
        ORDER BY /* 부분범위 처리 가능하게 할 인덱스 인자 */
      )
      WHERE A.rownum <= 1
    )
    ORDER BY A.colA
    ```

- 선분이력테이블에서 가장 최근 상태를 구하는 쿼리 형태
  - ```sql
    SELECT A.colA, A.colB, ... /* 필요한 인자*/
    FROM tableA A, tableB B
    WHERE A.colA = B.colA -- 조인
      and B.유효시작일자 < sysdate
      and B.유효종료일자 >= sysdate
    ```
- 선분이력테이블에서 가장 최근 상태의 직전 상태를 구하는 쿼리 형태
  - ```sql
    SELECT /* 필요한 인자 */
    FROM tableA A, tableB B
    WHERE A.colA = B.colA --조인
      and B.유효시작일자 < A.최종변경일자
      and B.유효종료일자 >= A.최종변경일자-1/24/60/60
    
  
