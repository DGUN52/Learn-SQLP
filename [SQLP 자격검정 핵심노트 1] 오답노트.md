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
