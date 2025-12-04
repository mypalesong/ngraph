# APOC 라이브러리

## APOC란?

**APOC (Awesome Procedures On Cypher)**는 Neo4j의 표준 라이브러리로, 450개 이상의 프로시저와 함수를 제공합니다.

## 설치

### Docker

```bash
docker run -d \
  --name neo4j \
  -e NEO4J_PLUGINS='["apoc"]' \
  -e NEO4J_dbms_security_procedures_unrestricted=apoc.* \
  neo4j:latest
```

### Neo4j Desktop

1. Database → Plugins 탭
2. APOC → Install 클릭
3. 데이터베이스 재시작

### 수동 설치

```bash
# APOC JAR 다운로드
wget https://github.com/neo4j/apoc/releases/download/5.15.0/apoc-5.15.0-core.jar

# plugins 폴더에 복사
cp apoc-*.jar /var/lib/neo4j/plugins/

# neo4j.conf 설정
dbms.security.procedures.unrestricted=apoc.*
dbms.security.procedures.allowlist=apoc.*
```

### 설치 확인

```cypher
RETURN apoc.version()

// 사용 가능한 프로시저 목록
CALL apoc.help('apoc')
```

---

## 데이터 가져오기/내보내기

### JSON

```cypher
// JSON 파일 로드
CALL apoc.load.json('https://api.example.com/users')
YIELD value
CREATE (u:User {name: value.name, email: value.email})

// 로컬 파일
CALL apoc.load.json('file:///data/users.json')
YIELD value
RETURN value

// 결과를 JSON으로 내보내기
CALL apoc.export.json.query(
  'MATCH (p:Person) RETURN p',
  'persons.json',
  {}
)
```

### CSV

```cypher
// CSV 로드 (헤더 포함)
LOAD CSV WITH HEADERS FROM 'file:///users.csv' AS row
CREATE (u:User {id: row.id, name: row.name})

// APOC로 대용량 CSV 처리
CALL apoc.periodic.iterate(
  "CALL apoc.load.csv('file:///big_data.csv') YIELD map AS row RETURN row",
  "CREATE (n:Record) SET n = row",
  {batchSize: 10000, parallel: true}
)
```

### 전체 데이터베이스

```cypher
// 내보내기
CALL apoc.export.cypher.all('backup.cypher', {format: 'cypher-shell'})

// GraphML 형식
CALL apoc.export.graphml.all('backup.graphml', {})

// 가져오기
CALL apoc.cypher.runFile('backup.cypher')
```

---

## 배치 처리

### apoc.periodic.iterate

대용량 데이터 처리의 핵심:

```cypher
// 기본 사용법
CALL apoc.periodic.iterate(
  "MATCH (p:Person) RETURN p",           // 소스 쿼리
  "SET p.processed = true",               // 처리 쿼리
  {batchSize: 1000, parallel: true}       // 옵션
)
YIELD batches, total, errorMessages
RETURN batches, total, errorMessages

// 관계 일괄 생성
CALL apoc.periodic.iterate(
  "MATCH (a:Person), (b:Person) WHERE a.company = b.company AND a <> b RETURN a, b",
  "MERGE (a)-[:COLLEAGUE]->(b)",
  {batchSize: 5000}
)
```

### apoc.periodic.commit

트랜잭션 자동 커밋:

```cypher
CALL apoc.periodic.commit(
  "MATCH (p:Person)
   WHERE NOT exists(p.updated)
   WITH p LIMIT $limit
   SET p.updated = true
   RETURN count(*)",
  {limit: 10000}
)
```

### 스케줄링

```cypher
// 주기적 실행 (5분마다)
CALL apoc.periodic.repeat(
  'cleanup',
  'MATCH (n:Temp) WHERE n.created < datetime() - duration("PT1H") DELETE n',
  300  // 초 단위
)

// 스케줄 취소
CALL apoc.periodic.cancel('cleanup')

// 활성 스케줄 확인
CALL apoc.periodic.list()
```

---

## 텍스트 함수

### 문자열 처리

```cypher
// 기본 함수
RETURN apoc.text.join(['a', 'b', 'c'], ', ')     // "a, b, c"
RETURN apoc.text.split('a,b,c', ',')              // ["a", "b", "c"]
RETURN apoc.text.capitalize('hello world')         // "Hello World"
RETURN apoc.text.camelCase('hello_world')          // "helloWorld"
RETURN apoc.text.snakeCase('helloWorld')           // "hello_world"

// 정규식
RETURN apoc.text.regexGroups('abc123def', '([a-z]+)(\d+)')
// [["abc123", "abc", "123"]]
```

### 유사도 계산

```cypher
// 레벤슈타인 거리
RETURN apoc.text.levenshteinDistance('hello', 'hallo')  // 1

// 유사도 (0~1)
RETURN apoc.text.levenshteinSimilarity('hello', 'hallo')  // 0.8

// 음성 유사도
RETURN apoc.text.phonetic('Smith')  // "S530"
RETURN apoc.text.doubleMetaphone('Smith')  // ["SM0", "XMT"]

// 퍼지 매칭
MATCH (p:Person)
WHERE apoc.text.levenshteinSimilarity(p.name, '홍길동') > 0.7
RETURN p.name
```

---

## 컬렉션 함수

```cypher
// 리스트 조작
RETURN apoc.coll.zip([1,2,3], ['a','b','c'])  // [[1,'a'],[2,'b'],[3,'c']]
RETURN apoc.coll.flatten([[1,2],[3,4]])        // [1,2,3,4]
RETURN apoc.coll.partition([1,2,3,4,5], 2)     // [[1,2],[3,4],[5]]
RETURN apoc.coll.shuffle([1,2,3,4,5])          // 무작위 순서

// 집합 연산
RETURN apoc.coll.union([1,2,3], [3,4,5])       // [1,2,3,4,5]
RETURN apoc.coll.intersection([1,2,3], [2,3,4]) // [2,3]
RETURN apoc.coll.subtract([1,2,3], [2])         // [1,3]

// 통계
WITH [1, 2, 3, 4, 5, 5, 5] AS nums
RETURN apoc.coll.avg(nums),                     // 3.57
       apoc.coll.min(nums),                     // 1
       apoc.coll.max(nums),                     // 5
       apoc.coll.frequencies(nums)              // {1:1, 2:1, 3:1, 4:1, 5:3}
```

---

## 그래프 알고리즘 (경량)

### 경로 탐색

```cypher
// 모든 경로 (DFS)
MATCH (start:Person {name: 'Alice'})
CALL apoc.path.expandConfig(start, {
  relationshipFilter: 'FRIEND>',
  minLevel: 1,
  maxLevel: 3,
  uniqueness: 'NODE_PATH'
})
YIELD path
RETURN path

// 최단 경로
MATCH (a:Person {name: 'Alice'}), (b:Person {name: 'Bob'})
CALL apoc.algo.dijkstra(a, b, 'ROAD', 'distance')
YIELD path, weight
RETURN path, weight

// A* 알고리즘
CALL apoc.algo.aStar(a, b, 'ROAD', 'distance', 'lat', 'lon')
YIELD path, weight
RETURN path, weight
```

### 커뮤니티 탐지

```cypher
// 연결 컴포넌트
CALL apoc.algo.unionFind('Person', 'FRIEND')
YIELD nodeId, setId
RETURN setId, count(*) AS size
ORDER BY size DESC
```

---

## 리팩토링

### 노드/관계 변환

```cypher
// 관계를 노드로 변환
MATCH (a:Person)-[r:TRANSFERRED]->(b:Person)
CALL apoc.refactor.extractNode([r], ['Transaction'], 'FROM', 'TO')
YIELD input, output
RETURN input, output

// 노드 병합
MATCH (a:Person {name: 'Alice'}), (b:Person {name: 'Alice_duplicate'})
CALL apoc.refactor.mergeNodes([a, b], {properties: 'combine'})
YIELD node
RETURN node

// 레이블 변경
MATCH (n:OldLabel)
CALL apoc.refactor.rename.label('OldLabel', 'NewLabel', [n])
YIELD committedOperations
RETURN committedOperations
```

### 속성 변환

```cypher
// 속성명 변경
MATCH (n:Person)
CALL apoc.refactor.rename.nodeProperty('oldName', 'newName', [n])
YIELD committedOperations
RETURN committedOperations

// 관계 타입 변경
MATCH ()-[r:OLD_TYPE]->()
CALL apoc.refactor.rename.type('OLD_TYPE', 'NEW_TYPE', [r])
YIELD committedOperations
RETURN committedOperations
```

---

## 메타 정보

```cypher
// 스키마 정보
CALL apoc.meta.schema()
YIELD value
RETURN value

// 그래프 통계
CALL apoc.meta.stats()
YIELD labelCount, relTypeCount, nodeCount, relCount
RETURN *

// 노드 타입 분석
CALL apoc.meta.nodeTypeProperties()
YIELD nodeType, propertyName, propertyTypes
RETURN *
```

---

## 다음 단계

!!! success "APOC 완료!"
    [Graph Data Science](gds.md)에서 고급 그래프 알고리즘을 배워보세요.
