# 추론과 규칙

## 추론(Reasoning)이란?

**추론**은 명시적으로 저장된 지식으로부터 새로운 지식을 논리적으로 도출하는 과정입니다.

```
명시적 지식: (홍길동)-[아버지]->(홍판서)
           (홍판서)-[아버지]->(홍할아버지)
추론된 지식: (홍길동)-[조부]->(홍할아버지)
```

## 추론 유형

### 1. RDFS 추론

```turtle
# 스키마 정의
ex:Employee rdfs:subClassOf ex:Person .
ex:worksAt rdfs:domain ex:Employee .

# 인스턴스
ex:홍길동 ex:worksAt ex:테크회사 .

# 추론 결과
ex:홍길동 a ex:Employee .  # domain 추론
ex:홍길동 a ex:Person .    # subClassOf 추론
```

### 2. OWL 추론

```turtle
# 동치 클래스
ex:성인 owl:equivalentClass [
    a owl:Restriction ;
    owl:onProperty ex:나이 ;
    owl:someValuesFrom [
        owl:onDatatype xsd:integer ;
        owl:withRestrictions ([xsd:minInclusive 18])
    ]
] .

# 역관계
ex:부모 owl:inverseOf ex:자녀 .

# 추이적 관계
ex:조상 a owl:TransitiveProperty .
```

### 3. 규칙 기반 추론

SWRL (Semantic Web Rule Language):

```
Person(?p) ∧ hasAge(?p, ?age) ∧ greaterThan(?age, 65) → Senior(?p)
```

## 추론 엔진

### Apache Jena

```java
// Jena에서 RDFS 추론
Model model = ModelFactory.createDefaultModel();
model.read("data.ttl");

InfModel infModel = ModelFactory.createRDFSModel(model);
// 추론된 트리플 포함
```

### Python RDFLib + OWL-RL

```python
from rdflib import Graph
from owlrl import DeductiveClosure, RDFS_Semantics

g = Graph()
g.parse("data.ttl", format="turtle")

# RDFS 추론 적용
DeductiveClosure(RDFS_Semantics).expand(g)

# 추론된 트리플 확인
for s, p, o in g:
    print(s, p, o)
```

## Neo4j에서의 규칙

### Cypher 쿼리로 규칙 구현

```cypher
// 규칙: 친구의 친구는 추천 대상
MATCH (me:Person {name: '홍길동'})-[:FRIEND]-(friend)-[:FRIEND]-(fof)
WHERE NOT (me)-[:FRIEND]-(fof) AND me <> fof
RETURN DISTINCT fof.name AS 추천친구

// 규칙: 같은 회사 동료
MATCH (me:Person)-[:WORKS_AT]->(company)<-[:WORKS_AT]-(colleague)
WHERE me <> colleague
MERGE (me)-[:COLLEAGUE]->(colleague)
```

### APOC 절차로 규칙 실행

```cypher
// 주기적 규칙 실행
CALL apoc.periodic.commit(
  "MATCH (p:Person)-[:PARENT]->(parent)-[:PARENT]->(grandparent)
   WHERE NOT EXISTS((p)-[:GRANDPARENT]->(grandparent))
   WITH p, grandparent LIMIT $limit
   MERGE (p)-[:GRANDPARENT]->(grandparent)
   RETURN count(*)",
  {limit: 1000}
)
```

## 추론 패턴

### 전이적 폐쇄 (Transitive Closure)

```cypher
// 모든 상위 카테고리 찾기
MATCH (item:Product)-[:IN_CATEGORY*]->(category:Category)
RETURN item.name, collect(category.name) AS allCategories
```

### 역관계 추론

```cypher
// 역관계 자동 생성
MATCH (child:Person)-[:PARENT]->(parent:Person)
MERGE (parent)-[:CHILD]->(child)
```

### 동치 추론

```cypher
// sameAs 관계 처리
MATCH (a)-[:SAME_AS*]-(b)
WITH a, collect(DISTINCT b) AS equivalents
// 동일 엔티티 통합 처리
```

## 추론 성능 최적화

### 1. 구체화 (Materialization)

```cypher
// 추론 결과를 미리 저장
MATCH (a)-[:ANCESTOR*]->(b)
MERGE (a)-[:HAS_ANCESTOR]->(b)
```

### 2. 증분 추론

```python
def incremental_reasoning(new_triple, knowledge_graph):
    """새 트리플 추가 시 영향받는 추론만 업데이트"""
    affected = find_affected_rules(new_triple)
    for rule in affected:
        apply_rule(rule, knowledge_graph)
```

### 3. 온디맨드 추론

```cypher
// 쿼리 시점에 추론
MATCH path = (start)-[:PARENT*1..10]->(ancestor)
WHERE start.name = '홍길동'
RETURN ancestor.name, length(path) AS generation
```

## 다음 단계

- [활용 사례](../use-cases/enterprise-knowledge.md)에서 실제 적용 예시를 확인하세요
