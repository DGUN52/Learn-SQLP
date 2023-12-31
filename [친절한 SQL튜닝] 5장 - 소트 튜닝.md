5. 소트 튜닝
- 
5.1 소트 연산에 대해
- SQL 수행 시 가공된 데이터 집합이 필요할 때 오라클은 PGA와 Temp table space를 사용한다.
  - 가공 : 소트 머지 조인, 해시 조인, 데이터 소트, 그룹핑 등

5.1.1 소트 수행 과정
- 소트는 PGA의 Sort Area에서 이루어지고, 초과 시 Temp 테이블스페이스를 이용한다.
  - 메모리 소트 : PGA의 Sort Area만 이용하는 것. Internal Sort
  - 디스크 소트 : 디스크 공간까지 이용하는 것. External Sort

- 소트 과정
  1. SGA의 버퍼캐시를 통해 소트할 대상 집합을 불러들인다.
  2. Sort Area에서 정렬을 수행하며, 양이 많으면 중간집합을 Temp 테이블스페이스에 저장한다. (저장된 집합은 Sort Run 이라 불림)
  3. Sort Run들을 머지하며 PGA -> 클라이언트/쿼리의 다음 수행으로 전송

- 소트연산은 메모리 집약적, CPU집약적이다.
  - 데이터 양이 많으면 디스크IO까지 발생하는 성능 부하의 큰 축
  - 가능하면 소트연산을 피하고, 불가피하면 Internal Sort로 끝내야 한다.
 
5.1.2 소트 오퍼레이션의 종류
1. Sort Aggregate
  - 전체 로우를 대상으로 집계를 수행할 때 발생 (Sort라는 표현과 달리 정렬하진 않음. Sort Area를 사용할 뿐)
  - SUM, MIN, MAX, AVG를 연산하는 과정 -> Sort Area의 각 네가지 공간에 데이터를 하나씩 읽어가며 갱신한다. 단 AVG는 count공간과 SUM공간을 이용한다.
2. Sort Order By
  - 5.1.1의 소트과정과 같다.
3. Sort Group By
  - 그룹별(ex 부서별)로 Sort Aggregate 과정을 수행한다. 마찬가지로 많은 데이터공간이 필요하지 않다.
  - 10gR2부터는 Hash Group By 방식이 도입되어 Group By 뒤에 Order By가 명시되지 않으면 Hash Group By 방식으로 수행한다.
    - Hash Group By는 group을 나눌 때 해시 알고리즘을 사용한다.
4. Sort Unique
  - 서브쿼리 Unnesting에서 Unnesting된 서브쿼리가 M쪽 집합이면(혹은 1쪽 집합이더라도 조인 컬럼에 Unique 인덱스가 없으면) 메인 쿼리와 조인하기 전에 중복 레코드를 제거해야 하고, Sort Unique 오퍼레이션이 수행된다.
  - PK/Unique 제약 또는 Unique 인덱스를 통해 언네스팅된 서브쿼리의 유일성이 보장되면 Sort Unique 오퍼레이션은 생략된다.
  - Union, Minus, Intersect같은 집합 연산자를 사용할 때도 Sort Unique 연산이 수행된다.
  - Dintinct 연산도 Order by가 없을 시 Hash Unique 연산이 수행된다. (10gR2 이후)
5. Sort Join
  - 소트 머지 조인 수행 시 나타남
6. Window Sort
  - 윈도우 함수(분석 함수) 수행 시 나타남


- 08.02(수) 317p ~ 342p


5.2 소트가 발생하지 않도록 SQL 작성
- 집합연산(intersect, minus...)의 사용을 유의하고, 조인방식을 유의하며 사용해야한다.

5.2.1 Union vs Union All
- 되도록 소팅 연산이 없는 Union All을 사용해야 한다.
- 중복 가능성이 없는 두 테이블을 합칠 시에는 Union All을 사용하여 소트연산의 발생을 피해야한다.
- 중복 가능성이 있다면. ex) 결제일자='20XXAABB' union 주문일자='20XXAABB' 대신 결제일자='20XXAABB' union 주문일자 '20XXAABB' and 결제일자 <> '20XXAABB'로 하면 된다.
  - 여기서 결제일자가 만약 null허용 컬럼이라면 마지막 조건항을 and (결제일자 <> '20XXAABB' or 결제일자 is null)로 바꾸어야 한다.
 
5.2.2 Exists활용
- Distinct 연산자는 조건에 해당하는 데이터를 모두 읽어야한다.
  - 부분범위 처리 불가, IO 다량 발생
- 따라서 조건들을 서브쿼리로 변환 후 where 절의 exists 문 안에 넣어서 존재여부만 확인하면 중단되게 한다.
  - 부분범위 처리 가능
- distinct(minus) 연산자는 대부분 (not)exists로 변환 가능

 5.2.3 조인 방식 변경
 - order by 조건이 인덱스에 포함되어 소트연산을 생략할 수 있는 상황에도 해시조인으로 처리되어 소트연산이 실행될 수 있다.
   - 이럴 때는 nl방식으로 유도하여 부분범위 처리의 이점, 소트연산 미실행의 이점을 취해야 한다.
 - 소트 머지 조인도 정렬 기준이 키 컬럼이면 sort order by 연산이 생략된다.

5.3 인덱스를 이용한 소트 연산 생략
- 인덱스를 이용하면 Order by, Group by의 소트 연산을 생략할 수 있고 Top N 쿼리, 최소값, 최대값의 연산 등에서 성능이 매우 빨라진다.

5.3.1 Sort Order By 생략
- where절과 order by 절에 사용되는 조건 순으로 인덱스를 구성

- 3 Tier 환경에서도 부분범위 처리는 효과적이다.

5.3.2 Top N 쿼리
- 3 tier 환경에서도 소트연산이 생략된다면 훨씬 빠른 응답속도를 낸다. (페이징 처리를 위한 Top N 연산 이용)
- 이를 가능하게 하려면 다음 수칙을 지킨다.
  1. 부분범위 처리 가능하도록 SQL 작성
     - NL조인 위주 처리, Order by 생략 가능하게 인덱스 및 쿼리를 구성
  2. 작성한 SQL문을 페이징 처리용 표준 패턴 SQL Body에 넣는다.

- 페이징 처리 패턴에서 일부 조건절을 필요없어보인다고 제거하면 원하는 데이터를 다 출력했음에도 전체범위를 계속 읽을 수도 있다.

5.3.3 최소값/최대값 구하기
- MIN, MAX와 같이 Sort Aggregate 연산을 이용하게 하는 함수는 전체 데이터를 읽는다.
  - 인덱스는 정렬돼있으므로 전체를 읽지 않고도 쉽게 최소/최대를 찾을 수 있다.
  - 이와 같이 생략되려면 조건절 컬럼들과 MIN/MAX 함수의 인자가 모두 인덱스에 포함돼있어야한다.
- Top N 쿼리를 이용하여 N에 1을 넣어서 Top 1 레코드를 찾음으로 MIN/MAX 값을 찾을 수 있다.
  - 쿼리는 좀 더 복잡하지만 성능면에선 MIN/MAX 함수보다 낫다.

5.3.4 이력 조회
- 이력조회가 복잡한 쿼리를 요구한다면, 인덱스 컬럼을 가공하는 대신 여러 서브쿼리로 나누어서라도 First Row Stopkey 알고리즘이 작동하게 하는 것이 성능적으로 좋다.
  - 하지만 쿼리가 더 복잡해진다면 쿼리가 엄청나게 더러워진다.
- INDEX_DESC 힌트와 ronum<=1 조건으로 복잡한 조건에서도 Top N stopkey를 구현할 수 있다.
  - 단 인덱스 구성을 완벽하게 해주어야 한다.
 

- 08.03(목) 343p ~ 370p


- 11g 이상의 오라클에선 메인 쿼리의 컬럼을 서브쿼리 내 인라인 뷰에서도 참조할 수 있다.
- 윈도우 함수가 점점 발전하고 12c부터는 Row Limiting 절도 지원 : 하지만 아직 성능이 뛰어나진 않다
- 이력조회 테이블을 조회하는 쿼리에 윈도우함수 ROW_NUMBER() OVER (order by A DESC, B DESC) 과 같은 함수를 사용할 수 있지만
  - 소트를 생략할 수 있을 때 사용하면 Top N Stopkey 알고리즘이 작동하지 않아서 불리하다.
- 12c부터의 Row Limiting절도 마찬가지
  - Row limiting절은 윈도우 함수 형태로 옵티마이저가 변환한다. (ex. fetch first X Rows → ROW_NUMBER() Over (Order by ~))
- 페이징 처리에 윈도우 함수를 사용하면 Top N Stopkey 작동 가능 (인라인 뷰에 쿼리를 넣는 페이징 처리 패턴을 사용하는 쿼리)
  - 인덱스를 사용하지 않을 수 있음(소트 생략하지 않을 수 있음)
  - 따라서 index/index_desc힌트를 자주 사용
  - + first_rows(n) 힌트 사용 (12c이상)

- 이력 조회 패턴의 다양함 : 예시 ) 장비 이력
  - 일부 장비, 전체 장비, 직전 이력, 최종 이력, 특정상태가 된 최종 이력 등 다양한 조회상황 존재
  - 이렇게 다양한 상황에 맞춰 First Row Stopkey/Top N Stopkey와 같은 실행계획을 통해 소트연산을 생략하고자 한다면 조회패턴도 다양하다.
- 조회량이 많다면 Stopkey 알고리즘의 호출이 핵심요소가 아니게 된다. (인덱스는 다량 조회일수록 가치가 떨어지는 것과 일맥)
- 전체 장비 이력 조회의 경우 윈도우 함수 사용이 효과적 (Full Scan과 해시 조인 사용하게됨)
- +선분이력모델 : 유효시작일자와 유효종료일자로 데이터를 점이 아닌 선 형태로 관리하는 것. 과거데이터 조회, 다량데이터 조회에 유리

5.3.5 Sort Group By 생략
- 소트연산 뿐만 아닌 그루핑 연산도 인덱스를 활용해 생략할 수 있다.
  - group by A인 쿼리일 때 인덱스의 선두컬럼이 A인 경우 -> Sort Group By Nosort 실행계획 등장
  - 이 경우 Group by 연산임에도 부분범위 처리가 가능해진다
 
5.4 Sort Area를 적게 사용하도록 SQL 작성
- 소트연산이 불가피하다면 인메모리 소팅으로 처리되어야 빠르다

5.4.1 소트 데이터 줄이기
- order by A이고 select A, B, C라면 A절만 소트에어리어에 담기는게 아니라 A B C 모두 소트 에어리어에 정렬된다.
  - lpad나 ||등의 연산으로 가공된 데이터도 마찬가지
 
5.4.2 Top N 쿼리의 소트 부하 경감 원리
- 인덱스를 사용하여 sort 생략
- 생략 불가능할 시 top n 소트 알고리즘이 작동되게 한다
  - 작동 시 아무리 많은 데이터라도 PR과 PW, sorts(disk)가 일어나지 않아 성능이 좋다

5.4.3 Top N 쿼리가 아닐 때 발생하는 소트 부하
- 페이지 처리 패턴에서 Rownum 조건을 제거하고 실행 시 top n 쿼리가 작동하지 않을 수 있다.
  - top n 쿼리 미작동을 유도하고 실행 시 pr, pw, sorts(disk) 항목의 통계치가 증가한다. (성능 down)

5.4.4 분석함수에서의 Top N 소트
- rank, row_number를 쓸 수 있는 상황이면 max함수를 사용하는 것보다 성능이 좋다.



- 08.04(금) 371p ~ 392p
