# Cypher

## 개요

Cypher는 Neo4j의 선언적 그래프 쿼리 언어입니다.

## 기본 문법

### 노드 생성

```cypher
CREATE (p:Person {name: '홍길동', age: 30})
RETURN p
```

### 관계 생성

```cypher
MATCH (a:Person {name: '홍길동'})
MATCH (b:Company {name: '테크회사'})
CREATE (a)-[:WORKS_AT {since: 2020}]->(b)
```

### 패턴 매칭

```cypher
// 친구의 친구 찾기
MATCH (me:Person {name: '홍길동'})-[:FRIEND*2]-(fof:Person)
WHERE me <> fof
RETURN DISTINCT fof.name
```

### 경로 탐색

```cypher
// 최단 경로
MATCH path = shortestPath(
  (start:Person {name: '홍길동'})-[*]-(end:Person {name: '이영희'})
)
RETURN path
```

## 집계

```cypher
MATCH (p:Person)-[:WORKS_AT]->(c:Company)
RETURN c.name, count(p) AS employees
ORDER BY employees DESC
```

## Python 사용

```python
from neo4j import GraphDatabase

driver = GraphDatabase.driver("bolt://localhost:7687", auth=("neo4j", "password"))

with driver.session() as session:
    result = session.run(
        "MATCH (p:Person {name: $name}) RETURN p",
        name="홍길동"
    )
    for record in result:
        print(record["p"])
```

## 다음 단계

- [GraphQL](graphql.md)에서 API 계층을 학습하세요
