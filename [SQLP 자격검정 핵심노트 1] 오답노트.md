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
