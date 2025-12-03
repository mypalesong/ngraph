# 성능 최적화

## 인덱스 전략

```cypher
// 고유성 제약조건 + 인덱스
CREATE CONSTRAINT person_id IF NOT EXISTS
FOR (p:Person) REQUIRE p.id IS UNIQUE;

// 복합 인덱스
CREATE INDEX person_name_dept IF NOT EXISTS
FOR (p:Person) ON (p.name, p.department);

// 전문 검색 인덱스
CREATE FULLTEXT INDEX person_search IF NOT EXISTS
FOR (p:Person) ON EACH [p.name, p.bio];
```

## 쿼리 최적화

```cypher
// ❌ 비효율적
MATCH (p:Person)
WHERE p.name STARTS WITH 'Kim'
RETURN p

// ✅ 인덱스 활용
MATCH (p:Person)
WHERE p.name STARTS WITH 'Kim'
USING INDEX p:Person(name)
RETURN p
```

## 배치 처리

```cypher
// APOC 배치 처리
CALL apoc.periodic.iterate(
  "MATCH (p:Person) RETURN p",
  "SET p.processed = true",
  {batchSize: 10000, parallel: true}
)
```

## 메모리 설정

```properties
# neo4j.conf
dbms.memory.heap.initial_size=4G
dbms.memory.heap.max_size=8G
dbms.memory.pagecache.size=4G
```

## 다음 단계

- [API 레퍼런스](../api/sparql.md)에서 쿼리 언어를 학습하세요
