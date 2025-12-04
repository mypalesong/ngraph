# Neo4j 성능 튜닝

## 인덱스 전략

### 인덱스 유형

| 유형 | 용도 | 생성 |
|------|------|------|
| Range | 일반 검색 | `CREATE INDEX` |
| Text | 전문 검색 | `CREATE TEXT INDEX` |
| Point | 공간 검색 | `CREATE POINT INDEX` |
| Fulltext | 복합 전문 검색 | `CREATE FULLTEXT INDEX` |

### 인덱스 생성

```cypher
// 단일 속성 인덱스
CREATE INDEX person_name FOR (p:Person) ON (p.name)

// 복합 인덱스
CREATE INDEX person_name_age FOR (p:Person) ON (p.name, p.age)

// 관계 인덱스 (Neo4j 5.7+)
CREATE INDEX knows_since FOR ()-[r:KNOWS]-() ON (r.since)

// 전문 검색 인덱스
CREATE FULLTEXT INDEX person_search FOR (p:Person) ON EACH [p.name, p.bio]

// 고유 제약조건 (인덱스 자동 생성)
CREATE CONSTRAINT person_email_unique FOR (p:Person) REQUIRE p.email IS UNIQUE

// 존재 제약조건
CREATE CONSTRAINT person_name_exists FOR (p:Person) REQUIRE p.name IS NOT NULL
```

### 인덱스 관리

```cypher
// 인덱스 목록
SHOW INDEXES

// 인덱스 삭제
DROP INDEX person_name

// 인덱스 사용 확인
EXPLAIN MATCH (p:Person {name: 'Alice'}) RETURN p
// NodeIndexSeek이 표시되면 인덱스 사용 중
```

---

## 쿼리 최적화

### 실행 계획 분석

```cypher
// 예상 계획
EXPLAIN MATCH (p:Person)-[:KNOWS]->(f) RETURN p, f

// 실제 계획 (실행 포함)
PROFILE MATCH (p:Person)-[:KNOWS]->(f) RETURN p, f
```

### 주요 연산자

| 연산자 | 설명 | 성능 |
|--------|------|------|
| NodeIndexSeek | 인덱스로 노드 검색 | 🟢 빠름 |
| NodeByLabelScan | 레이블 스캔 | 🟡 보통 |
| AllNodesScan | 전체 노드 스캔 | 🔴 느림 |
| Expand | 관계 탐색 | 🟢 빠름 |
| Filter | 조건 필터링 | 상황에 따라 |

### 최적화 패턴

```cypher
// ❌ 전체 스캔
MATCH (a), (b)
WHERE a.name = 'Alice' AND (a)-[:KNOWS]->(b)
RETURN b

// ✅ 인덱스 활용 + 패턴 매칭
MATCH (a:Person {name: 'Alice'})-[:KNOWS]->(b)
RETURN b

// ❌ 모든 속성 반환
MATCH (p:Person) RETURN p

// ✅ 필요한 속성만
MATCH (p:Person) RETURN p.name, p.age

// ❌ 불필요한 변수
MATCH (a:Person)-[r:KNOWS]->(b:Person)-[r2:KNOWS]->(c:Person)
WHERE a.name = 'Alice'
RETURN c.name

// ✅ 필요없으면 생략
MATCH (a:Person {name: 'Alice'})-[:KNOWS]->()-[:KNOWS]->(c:Person)
RETURN c.name
```

### 슈퍼노드 처리

```cypher
// ❌ 슈퍼노드 전체 탐색
MATCH (:User)-[:FOLLOWS]->(:Celebrity {name: '유명인'})
RETURN count(*)  // 수백만 관계 탐색

// ✅ 제한 사용
MATCH (c:Celebrity {name: '유명인'})<-[:FOLLOWS]-(u:User)
WITH u LIMIT 1000
RETURN count(*)

// ✅ 카운트 저장
// 관계 수를 노드 속성으로 유지
SET celebrity.followerCount = size((celebrity)<-[:FOLLOWS]-())
```

---

## 메모리 설정

### neo4j.conf

```properties
# JVM Heap (객체 저장)
server.memory.heap.initial_size=2G
server.memory.heap.max_size=4G

# Page Cache (디스크 캐시)
server.memory.pagecache.size=4G

# 트랜잭션 로그
db.tx_log.rotation.retention_policy=2 days
```

### 메모리 계산

```
총 RAM = Heap + Page Cache + OS 여유분

예시 (16GB RAM):
- Heap: 4GB
- Page Cache: 8GB (데이터 크기에 비례)
- OS: 4GB
```

### Page Cache 크기

```cypher
// 데이터 크기 확인
CALL apoc.meta.stats()
YIELD nodeCount, relCount

// 추정: 노드당 ~15KB, 관계당 ~10KB
// Page Cache >= 데이터 크기 권장
```

---

## 쓰기 성능

### 배치 처리

```cypher
// ❌ 개별 트랜잭션
UNWIND range(1, 100000) AS i
CREATE (n:Node {id: i})

// ✅ APOC 배치 처리
CALL apoc.periodic.iterate(
  "UNWIND range(1, 100000) AS i RETURN i",
  "CREATE (n:Node {id: i})",
  {batchSize: 10000, parallel: true}
)
```

### 제약조건 비활성화

```cypher
// 대량 로드 시 임시로 제약조건 제거
DROP CONSTRAINT person_email_unique

// 데이터 로드
LOAD CSV ...

// 제약조건 재생성
CREATE CONSTRAINT person_email_unique ...
```

### 인덱스 지연 생성

```cypher
// 데이터 로드 전 인덱스 생성하면 느림
// 1. 먼저 데이터 로드
// 2. 그 다음 인덱스 생성

LOAD CSV ...
CREATE INDEX ... // 로드 후 생성
```

---

## 읽기 성능

### 쿼리 캐싱

```properties
# neo4j.conf
# 쿼리 계획 캐시
db.query_cache_size=1000
```

### 결과 제한

```cypher
// 항상 LIMIT 사용
MATCH (p:Person)
RETURN p
LIMIT 100

// 스킵과 리밋 (페이지네이션)
MATCH (p:Person)
RETURN p
SKIP 100 LIMIT 20
```

### 프로젝션 활용

```cypher
// 필요한 데이터만 메모리에 로드
MATCH (p:Person)-[:WORKS_AT]->(c:Company)
RETURN p {.name, .email, company: c.name}
```

---

## 모니터링

### 쿼리 로그

```properties
# neo4j.conf
db.logs.query.enabled=INFO
db.logs.query.threshold=1s  # 1초 이상 쿼리 기록
```

### 실행 중 쿼리

```cypher
// 현재 실행 중인 쿼리
SHOW TRANSACTIONS

// 느린 쿼리 종료
TERMINATE TRANSACTION 'transaction-id'
```

### 메트릭스

```cypher
// 데이터베이스 상태
CALL dbms.queryJmx('org.neo4j:*')

// 스토어 크기
CALL apoc.monitor.store()
```

---

## 체크리스트

- [ ] 자주 검색하는 속성에 인덱스 생성
- [ ] 고유 속성에 제약조건 설정
- [ ] EXPLAIN/PROFILE로 쿼리 분석
- [ ] 슈퍼노드 패턴 확인
- [ ] 적절한 메모리 할당
- [ ] 배치 처리 활용
- [ ] 느린 쿼리 로깅 활성화

---

## 다음 단계

!!! success "성능 튜닝 완료!"
    [운영 및 백업](operations.md)에서 프로덕션 관리를 배워보세요.
