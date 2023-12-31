# 8.1 디스크 읽기 방식

데이터베이스의 성능 튜닝은 어떻게 디스크 I/O를 줄이느냐가 관건.

## 8.1.1 Hard Disk Drive(HDD) and Solid State Drive(SSD)

- 전자식(cpu, memory) 빠름.
- 기계식(HDD) 느림. -> 병목
- 기계식 HDD를 대체하기 위한 전자식 저장 매체 SSD.
- SSD는 기계식 원판 대신 플래시 메모리 장착해 1000배 이상 빠르다.

순차 io에서는 SSD와 HDD가 비슷한 성능을 보이지만
랜덤 io에서는 SSD가 훨씬 빠르다.

데이터베이스는 랜덤 io를 통해 작은 데이터를
읽고 쓰는 작업이 대부분이므로 DBMS Storage에 최적.

## 8.1.2 Random I/O and Sequential I/O

![8.3](/chap08-1/images/8.3.png)
왼쪽의 순차 io는 1회 발생, 오른쪽의 랜덤 io는 3회 발생. 랜덤 io의 작업 부하가
훨씬 더 크다. 디스크 원판이 없는 SSD에서도 여전히 랜덤 io는 순차 io 보다 성능
떨어짐.

디스크의 성능은 디스크 헤더의 위치 이동 없이 얼마나 많은 데이터를 한 번에
기록하느냐에 의해 결정.

[참고] 온프렘 버전에서는 RAID 컨트롤러의 캐시 메모리가 빈번한 동기화 작업이
호출되는 순차 io를 효율적으로 처리될 수 있도록 변환한다.

쿼리 튜닝은 랜덤 io를 줄이기 위한 것이 목적.(쿼리를 처리하는데 꼭 필요한
데이터만 읽도록 쿼리 개선)

[참고] Index Range Scan은 주로 random io를 통해 데이터를 읽고, Full Table Scan은
Sequencial io를 사용한다. 큰 데이블의 레코드 대부분을 읽는 작업은 인덱스 대신에
풀 테이블 스캔을 사용할 때가 있는데 이는 순차 io가 랜덤 io보다 훨씬 빨리 많은
레코드를 읽어올 수 있기 때문.

# 8.2 인덱스란?

- 색인. 찾아보기.
- 모든 데이터를 검색해서 원하는 결과를 가져오기엔 시간이 오래 걸리기 때문에
  칼럼의 값과 해당 레고드가 저장된 주소를 key-value로 삼아 인덱스를 만들어둔다.
- 인덱스는 정렬 방식이 아주 중요하다.
- SortedList는 인덱스와 같은 자료 구조. 항상 정렬되어 있음. 새 데이터 올때마다
  정렬해야 하므로 저장 과정이 복잡하지만 서치가 빠르다. 인덱스가 커지면서 인덱스용 인덱스도 있다.
- ArrayList는 데이터 파일과 같은 자료구조. 저장되는 순서되로.
- 즉 인덱스란 데이터의 저장(insert, update, delete) 성능을 희생해서 읽기 속도를
  높이는 기능이다.
- 이 책에서 key == index
- 대표적인 인덱스: B-Tree, Hash, Fractal-Tree, Merge-Tree etc.
- B-Tree Algorithm
  - 가장 일반적으로 사용. 칼럼 값을 변형하지 않고 원래의 값을 이용.
  - 위치 기반 검색하는 R-Tree도 B-Tree의 응용형.
- Hash Index Algorithm
  - 칼럼의 해시값으로 인덱싱. 검색이 매우 빠르다. 그러나 해시값이므로 값 일부만
    검색하거나 범위 검색시 사용이 불가능하다. 주로 메모리 기반 DB에서 많이 사용.

# 8.3 B-Tree Index

인덱싱 알고리즘 중 가장 일반적으로 사용되고 가장 먼저 도입됨. B+Tree 또는
B-Tree가 가장 많이 사용됨. Binary (X) Balanced(O)

이진 트리인 경우 검색이 상당히 비효율적일 것. 비트리 인덱스는 자식 노드 개수가
가변적이다.

## 8.3.1 Structure and Properties

- Root node
- Leaf node: 트리의 가장 하단 노드. 실제 데이터 레코드를 찾아가기 위한 주솟값 가짐.
- Branch node: 루트도 리프도 아닌 중간 노드

데이터베이스에서 인덱스와 실제 데이터가 저장된 공간은 따로 관리된다.

![8.4](/chap08-1/images/8.4.png)

- 인덱스의 키 값은 모두 정렬되어있지만, 데이터 파일의 레코드는 임의의 순서
  되어있음.
- 인덱스는 테이블의 키 칼럼만 가지고 있고 나머지 칼럼을 읽으려면 데이터
파일에서 해당 레코드를 찾아야 한다. 

[참고] InnoDB 테이블에서 레코드는 클러스터되어서 디스크에 저장되므로 프라이머리
키 순서로 정렬되어 저장되는것이 디폴트값이다.

![8.6](/chap08-1/images/8.6.png)
InnoDB 테이블은 프라이머리 키가 ROWID(물리주소) 역할을 한다. 프라이머리 키를
주소처럼 사용하기 때문에 논리적인 주소를 가진다고 볼 수 있다. 따라서 데이터
파일을 바로 찾아가지 못하고 프라이머리 키 인덱스를 한번 더 검색해서 프라이머리
키 인덱스의 리프 페이지에 저장되어있는 레코드를 읽는다. 

즉, InnoDB 테이블은 세컨데리 인덱스 검색할 때 반드시 프라이머리 키 인덱스를 한번
더 거쳐야 한다. (성능이 떨어지는 것 처럼 보이나 장단점이 있다)

## 8.3.2 Insertion and Deletion of B-Tree Index Key

### 8.3.2.1 Index Key Insertion

1. 키를 추가하기 위해서 먼저 비트리에서 적절한 위치 검색.
2. 저장될 위치 결정 후 레코드의 키 값과 주소 정보를 리프 노드에 저장. (리프 노드
   꽉 찬 경우 비용이 좀 든다.)

이노디비는 즉시 새로운 키 값을 인덱스에 추가하는 게 아니라 지연스켜 나중에
처리할 수도 있다. 하지만 프라이머리 키나 유니크 인덱스는 즉시 처리된다.

### 8.3.2.2 Index Key Deletion

삭제는 간단하다.

1. 삭제해야할 리프 노드를 찾아 삭제 마킹을 한다.
2. 삭제 마킹된 노드는 방치되거나 재활용한다. 지연 처리 가능.

### 8.3.2.3 Index Key Update

1. 키를 삭제한다.
2. 다시 새로운 키를 추가한다.

키 값이 변함에 따라 당연히 위치도 변해야 하므로 삭제후 추가한다.

### 8.3.2.4 Index Key Search

B-Tree인덱스를 이용한 검색은 100% 일치하거나 prefix만 일치해도 가능하다. InnoDB
데이블에서 지원하는 레코드 잠김이나 넥스트 키 락은 검색을 수행한 인덱스를 잠근
후에 테이블 레코드를 잠근다. 따라서 인덱스 설계 방식이 중요하다.

## 8.3.3 B-Tree Index 성능에 영향을 미치는 요소

- 칼럼 크기
- 레코드 건수
- 유니크 인덱스 키 개수 등등

### 8.3.3.1 Index Key Value Size

![8.7](/chap08-1/images/8.7.png)
Page or Block: InnoDB스토리지 엔진에서 디스크에 데이터를 저장하는 가장 기본
단위. 디스크의 모든 읽기 및 쓰기 작업의 최소 작업 단위이다. 버퍼 풀에서
버퍼링하는 기본 단위. 인덱스도 페이지로 관리된다.

비트리 인덱스는 자식 노드 개수가 가변적이다. 인덱스 페이지 크기와 키 값의 크기에
따라 최대 자식 노드 개수가 정해진다. `innodb_page_size` 변수로 정함.

인덱스를 구성하는 키 값이 커지면, 가질 수 있는 자식 노드는 줄어들고, 같은 양의 데이터를
읽을 때 더 여러번 나누어 읽어야 한다. 즉 그만큼 느려진다.

### 8.3.3.2 B-Tree Depth

깊이는 딱히 제어할 방법이 없다. 인덱스 키 값의 크기가 커질수록 하나의 인덱스
페이지가 담을 수 있는 개수가 적어진다. 같은 레코드 건수라도 트리 깊이가 깊어져
디스크 읽기를 더 많이 해야 함.

즉 인덱스 키 크키는 작을수록 좋다. 아무리 대용량이라도 트리 깊이가 5 이상
깊어지는 일은 잘 없다.  

### 8.3.3.3 Selectivity(Cardinality) 선택도(기수성)

모든 인덱스 키 값 가운데 유니크한 값의 수. 인덱스 키 값 100개중에 유니크한 값이 10개라면 기수성은 10.

중복된 인덱스 키 값이 많아질수록 기수성/선택도는 떨어진다.

인덱스는 선택도가 높을수록 검색 대상이 줄어들기 때문에 그만큼 빠르다.

[참고] 인덱스가 항상 검색에만 사용되는 것은 아니라서 선택도가 좋지 않아도 정렬이나 그루핑같은 작업을 위해 인덱스를 만드는 것이 좋다.

[예시] 1만개의 레코드가 있는 tb_test 테이블에 country 칼럼으로만 인덱스가 생성된
상태에서 인덱스 유니크 값이 10일떄와 1000일 때의 차이를 보자.

```sql
SELECT *
FROM tb_test
WHERE country='KOREA' AND city='SEOUL';
```

기수성 10일때: 1건의 레코드를 위해 999개의 레코드 더 읽음

기수성 1000일떄: 1건의 레코드를 위해 9개의 레코드 더 읽음

### 8.3.3.4 Number of records to read

레코드가 100만건이 저장된 테이블에서 50만건을 읽어야 하는 쿼리가 있다고 할 때,
다음과 같은 선택 가능한데 무엇이 효율적일지 판단해야 한다.

1. 전체 테이블을 모두 읽은 후 필요 없는 50만건을 버린다.
2. 인덱스를 통해 필요한 50만 건만 읽어온다.

일반적인 데이터베이스의 옵티마이저에서는 인덱스를 통해 레코드 1건을 읽는 것이
테이블에서 직접 레코드 1건을 읽는 것 보다 4~5배 더 많은 비용이 듬. 즉 인덱스를
통해 읽어야 할 레코드의 건수가 전제 테이블 레코드의 20~25%를 넘어서면 인텍스를
이용하지 않고 모두 직접 읽은 후 필터링 방식으로 처리하는 것이 효율적이다. 

## 8.3.4 Data Read with B-Tree Index

인덱스를 이용하는 대표적인 방법 세 가지.

### 8.3.4.1 Index Range Scan 인덱스 레인지 스캔

가장 대표적. 셋 중 가장 빠름. 검색해야 할 인덱스의 범위가 결정 됐을 때 사용.

![8.8](/chap08-1/images/8.8.png)
그림 8.8에서 보듯이 루트 노드부터 비교를 시작해서 브랜치 노드, 리프노드까지 찾아
들어가야만 필요한 레코드의 시작을 찾을 수 있다. 그리고 그 다음부터 리프 노드의
레코드만 순서대로 읽는다. (인덱스만 익는 경우)

![8.9](/chap08-1/images/8.9.png)
그림 8.9 실제 데이터 파일의 레코드를 읽어와야 하는 경우 스캔 시작 위치를 검색 한
후 리프노드에 저장된 레코드 주소로 데이터 파일의 레코드를 읽어오는데 한건 단위로
랜덤 io가 한 번씩 일어난다. 따라서 인덱스를 통해 데이터 레코드를 읽는 작업은
비용이 많이 든다.

1. 인덱스 탐색(index seek): 인덱스에서 조건을 만족하는 값이 저장된 위치를
   찾는다.
2. 인덱스 스캔(index scan): 1에서 찾은 위치부터 필요한 범위만큼 차례대로 쭉
   읽는다.
3. 2에서 읽은 인덱스 키와 레코드 주소로 데이터가 실제로 저장된 페이지를 가져오고 최종 레코드를
   읽는다. (쿼리가 필요로 하는 데이터에 따라 필요하지 않을 수도. 커버링 인덱스는
   디스크의 레코드를 읽지 않아도 되기 때문에 아주 빠르다.)

1번과 2번 작업이 얼마나 수행됐는지 확인 할 수 있는 쿼리도 있다.

### 8.3.4.2 Index Full Scan

인덱스의 처음부터 끝까지 모두 읽는 방식. 쿼리의 조건절에 사용된 칼럼이 인덱스의
첫 번째 칼럼이 아닌 경우 풀 스캔을 한다. 

![8.10](/chap08-1/images/8.10.png)
먼저 인덱스 리프 노드의 제일 앞/뒤로 이동한 후 인덱스 리프 노드의 링크드
리스트를 따라 처음부터 끝까지 스캔하는 방식. 레인지 스캔보다는 빠르지 않지만
테이블 풀 스캔보다는 효율적이다. 인덱스 크기는 테이블 크기보다는 작으므로 테이블
전체를 읽는 것보다 적은 디스크 io로 쿼리를 처리할 수 있다.

[주의] 이 책에서 "인덱스를 사용한다" 라는 것은 레인지 스캔이나 루스 인덱스
스캔을 사용하는 것을 의마한다. 풀 스캔은 효율적은 방식은 아니며, 인덱스 생성
목적도 아니다. 

### 8.3.4.3 Loose Index Scan

오라클의 인덱스 스킵 스캔과 비슷함. MySQL에서는 루스 인덱스 스캔이라 부른다.
앞선 두 가지 레인지/풀 스캔은 Loose와 상반된 의미의 Tight 인덱스 스캔이라
불린다. 루스 인덱스 스캔은 말 그대로 듬성듬성하게 인덱스를 읽는 것.

![8.11](/chap08-1/images/8.11.png)
레인지 스캔과 비슷하지만 중간에 필요치 않은 인덱스 키 값은 무스하고 다음으로
넘어간다. 일반적으로 `GROUP BY` `MAX()` `MIN()` 등에 사용.

```sql
...
WHERE dep_no BETWEEN 'a' and 'b'
GROUP BY dept_no;
```

### 8.3.4.4 Index Skip Scan

employees 테이블에 ix_gender_birthdate (gender, birth_date) 인덱스가 있다고 가정하자. 이 인덱스를
사용하려면 WHERE 절에 gender 칼럼에 대한 비교 조건이 필수이다. 만약 그렇지 않은
경우에는 인덱스를 새로 생성해야 했다. 그러나 8.0부터 옵티마이저가 gender 칼럼을 건너
뛰어서 birth_date 칼럼만으로도 인덱스 검색이 가능한 Index Skip Scan 최적화
기능이 생김.

```sql
SELECT gender, birth_date
FROM employees
WHERE birth_date >= '1965-02-01';
```

쿼리를 실행할떄 기존의 ix_gender_birthdate 인덱스에 인덱스 스킵 스캔 사용 가능.

1. 우선 gender 칼럼에서 유니크한 값을 모두 조회
2. 1에서 gender 칼럼 조건을 추가해서 쿼리를 다시 실행

![8.12](/chap08-1/images/8.12.png)

단점:

 - WHERE 조건 절에 저건이 없는 인덱스(e.g. gender)의 선행 컬럼의 유니크한 값의 개수가 적어야
   함.
 - 쿼리가 인덱스에 존재하는 칼럼만으로 처리 가능해야 함. (커버링 인덱스) `SELECT
   *` 사용 시 나머지 칼럼도 필요해 풀 테이블 스캔 사용하게 됨.

## 8.3.5 Multi-Coulmn Index 다중 칼럼 인덱스/복합 칼럼 인덱스

2개 이상 칼럼을 포함하는 인덱스가 더 많이 사용된다. Concatenated Index 라고도
함.

![8.13](/chap08-1/images/8.13.png)두 번째 칼럼은 첫 번째 칼럼에 의존해서 정렬되어있다. 따라서 다중 칼럼
인덱스에서는 인덱스 내에서 각 칼럼의 순서가 상당히 중요하다.

## 8.3.6 B-Tree Index의 정렬 및 스캔 방향

인덱스 오름차순/내림차순 정렬 상관 없다. 인덱스 거꾸로 읽기도 가능.

### 8.3.6.1 Index 정렬

8.0부터 혼합 순서 인덱스도 가능하다.

```sql
CREATE INDEX ix_teamname_userscore ON employees (team_name ASC, user_score DESC);
```

#### 8.3.6.1.1 Index Scan 방향

#### 8.3.6.1.2 Descending Index 내림차순 인덱스

![8.15](/chap08-1/images/8.15.png)

- 오름차순 인덱스(Ascending index) : 작은 값이 왼쪽
- 내림차순 인덱스(Descending index) : 큰 값이 왼쪽
- Forward Index Scan: 리프노드 왼쪽부터 오른쪽으로 스캔
- Backward Index Scan: 리프노드 오른쪽부터 왼쪽으로 스캔

실험 테이블 t1. 1천만건 데이터

```sql
(1) SELECT * FROM t1 ORDER BY tid ASC LIMIT 12619775, 1; -> 4.15 sec
(2) SELECT * FROM t1 ORDER BY tid DESC LIMIT 12619775, 1; -> 5.35 sec
```

역순 쿼리가 28.9% 더 시간이 걸렸다. InnoDB 스토리지 엔진에서 정순 스캔과 역순
스캔은 페이지간의 `Double linked list` 를 통해 전진/후진 차이만 있지만 실제
역순 스캔이 느린 이유가 있다.

1. 패이지 락이 인덱스 정순 스캔에 적합한 구조.
2. InnoDB 페이지 내에서 인덱스 레코드가 단방향으로만 연결되어 있다. 그림 8.16
![8.16](/chap08-1/images/8.16.png)

실제로 InnoDB 페이지는 힙(Heap: 완전 이진 트리의 일종으로 우선순위 큐를 위하여 만들어진 자료구조)처럼 사용되기 때문에 물리적으로
순서대로 저장되지 않는다.

일반적으로 소량의 레코드에 드물게 실행되는 DESC라면 굳이 내림차순 인덱스는 필요
없을 것이다. 하지만 많은 레코드/빈번하게 실행된다면 내림차순 인덱스가 더
효율적일 것.

## 8.3.7 B-Tree Index의 가용성과 효율성

쿼리의 WHERE 절이나 GROUP BY, ORDER BY 절이 어떤 인덱스를 어떤 방식으로
사용하는지 알아야 쿼리를 최적화하거나, 쿼리에 맞게 인덱스를 최적으로 생성할 수
있다.

### 8.3.7.1 비교 조건의 종류와 효율성

다중 칼럼 인덱스에서 각 칼럼의 순서와 그 칼럼에 사용된 조건의 operator(`=`, `>`,
`<` 등) 에 따라 각 인덱스 칼럼의 활용 형태와 효율이 바뀐다.

```sql
SELECT * FROM dept_emp WHERE dept_no='d002' AND emp_no >=10114;
```

이 쿼리를 실행할때, 다음 두 인덱스에 따라 성능이 다르다.

- A: Index(dept_no, emp_no) # 효율적
- B: Index(emp_no, dept_no) # 비효율적. 다중 칼럼 인덱스의 두번째 인덱스가
  비교 작업의 범위를 좁히는데 전혀 도움이 되지 않았다.

### 8.3.7.2 인덱스의 가용성

B-Tree 인덱스의 특징은 왼쪽 값에 기준해서 오른쪽 값들이 정렬되어 있다는 것이다.(
Left-most)
따라서 왼쪽 값을 모르면 레인지 스캔 방식을 쓸 수 없다.
다중 인덱스 칼럼에서도 똑같이 적용된다.

- Index(first_name)

```sql
SELECT * FROM employees WHERE first_name LIKE '%mer';
```

이 쿼리는 레인지 스캔을 할 수 없다. 그 이유는 상수값의 왼쪽 부분이 고정되지
않았기 때문. 따라서 왼쪽 기준 정렬 기반 인덱스인 비트리에서 인덱스 효과를 얻을
수 없다.

- Index(dept_no, emp_no)

```sql
SELCET * FROM dept_emp WHERE emp_no >= 10144;
```

다중 인덱스의 선행 칼럼인 dept_no 조건 없이는 인덱스 효율적 사용 불가능.

### 8.3.7.3 가용성과 효율성 판단

다음 조건에서는 b-tree 인덱스를 사용할 수 없다.

1. NOT-EQUAL 로 비교된 경우.(`<>`,`NOT IN`, `NOT BETWEEN`, `IS NOT NULL`)
2. LIKE '%??' 뒷부분 일치로 문자열 패턴 비교시
3. 인덱스 칼럼이 변형된 후 비교된 경우. 
    - WHERE SUBSTRING(column, 1, 1) = 'something'
    - WHERE DAYOFMONTH(column) = 1

등등

MySQL에서는 null값도 인덱스에 저장된다. WHERE column IS NULL 도 인덱스 사용.

# 8.4 R-Tree Index

공간 인덱스(Spatial Index)는 R-Tree 인덱스 알고리즘을 이용해 2차원의 데이터를
인덱싱/검색 하는 것. B-Tree와 유사. B-Tree는 1차원 스칼라값인 반면, R-Tree는
2차원 공간 개념.

MySQL의 Spatial Index 기능

1. 공간 데이터를 저장할 수 있는 데이터 타입
2. 공간 데이터 검색을 위한 공간 인덱스(R-Tree)
3. 공간 데이터의 연산 함수(거리 또는 포함 관계 처리)

## 8.4.1 Structure and Properties

`GEOMETRY` data type. 도형 정보 관리. `POINT`, `LINE`, `POLYGON` 객체 모두 저장 가능

MBR(Mininum Bounding Rectangle): 최소경계상자. 해당 도형을 감싸는 최소 크기의
사각형.
![8.20](/chap08-1/images/8.20.png)

이 사각형들의 포험 관계를 B-Tree 형태로 구현한 인덱스가 R-Tree 인덱스이다.

![8.22](/chap08-1/images/8.22.png)

![8.23](/chap08-1/images/8.23.png)
최상위 레벨의 MBR은 루드 노드에 저장되고, 차상위 MBR그룹은 브랜치
노드에, 마지막으로 각 도형 객체는 리프 노드에 저장된다.

## 8.4.2 Usecase of R-Tree Index

R-Tree는 MBR정보를 B-Tree 형태로 인덱스를 구축하므로 Rectecgle의 R과 B-Tree
트리를 따와 이름붙임. Spatial Index라고도 부른다.

일반적으로 위도 경도 좌표나 좌표 시스템에 기반을 둔 정보 저장에 주로 사용.

R-Tree 는 각 도형의 MBR의 포함 관계를 이용해 만들어진 인덱스이므로
`ST_Contains()` 또는 `ST_Within()` 등과 같은 포함 관계를 비교하는 함수로 검색을
수행하는 경우에만 사용 가능. e.g. 현재 사용자의 위치로부터 반경 5Km 이내의
음식점 검색
![8.24](/chap08-1/images/8.24.png)

# 8.5 전문 검색 인덱스

Full Text Search (전문검색): 문서 내용 전체를 인덱스화해서 특정 키워드가 포함된
문서를 검색. InnoDB에서 제공하는 일반적인 B-Tree 인덱스를 사용할 수 없다. 이를
위해서는 fulltext search index를 사용해야함.

## 8.5.1 인덱스 알고리즘

풀텍스트 인덱스는 문서의 키워드를 인덱싱하는 기법에 따라 크게 어근 분석과 n-gram
분석으로 나눈다.

### 8.5.1.1 어근 분석 알고리즘

MySQL 서버의 풀텍스트 인덱스는 다음과 같은 과정을 거쳐 색인 작업을 수행한다,

- 불용어(Stop Word) 처리: 별 가치 없는 단어를 모두 필터링해 제거. 불용어의
  개수는 많지 않아서 상수로 정의하는 경우가 대다수. 불용어 자체를 DB화해서
  사용자가 추가 삭제 가능.
- 어근 분석 (Stemming): 검색어로 선정된 단어의 원형을 찾는 작업. MySQL에서
  오픈소스 형태소 분석 라이브러리(MeCab: 일본어/한국어)을 플러그인 형태로 사용.
  MongoDB(Snowball: 서구권 언어) 사실 모델을 학습시킨 뒤 사용해야 함.

### 8.5.1.2 n-gram 알고리즘

- 형태소 분석 등은 많은 노력과 시간 필요. 범용적 사용을 위해 n-gram 사용. 형태소
분석보다 알고리즘이 단순하고 국가별 언어에 대한 준비 작업이 필요 없으나,
인덱스의 크기가 큰 편.
- 본문을 몇 글자씩 잘라서 인덱싱. `n` 은 인덱싱할 키워드의 최소 글자 수를 의미.
  일반적으로 2-gram(Bi-gram) 사용.

각 단어는 공백과 마침표를 기준으로 단어가 된다. **2글자씩 중첩해서**
토큰으로 분리된다. 10글자인 경우 2-gram 알고리즘에서 (10-1)개의 토큰으로
구븐된다. 중복된 토큰은 1개의 엔트리로 병합됨.

이렇게 생성된 토큰들에 대해 불용어를 걸러내는 작업 수행.

```md
To be or not to be. That is the question.
```
![2-gram](/chap08-1/images/2-gram.png)

### 8.5.1.3 불용어 변경 및 삭제

불용어 처리 시 더 혼란스러운 경우도 있어 설정을 변경하거나 직접 등록하는것을 권장.

## 8.5.2 전문 검색 인덱스의 가용성

풀텍스트 인덱스를 사용하려면 반드시 갖춰야할 조건:

1. 쿼리 문장이 풀텍스트 서치를 위한 문법 사용해야 한다. `MATCH ... AGAINST ...`
2. 테이블이 풀텍스트 서치 대상 칼럼에 대해서 풀텍스트 인덱스를 보유하고 있어야.

```sql
SELECT * FROM tb_test WHERE doc_body LIKE '%apple%';
```

이 경우 풀 테이블 스캔으로 쿼리 처리.

```sql
SELECT * FROM tb_test WHERE MATCH(doc_body) AGAINST ('apple' IN BOOLEAN MODE);
```
