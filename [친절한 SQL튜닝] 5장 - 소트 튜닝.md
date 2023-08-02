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
