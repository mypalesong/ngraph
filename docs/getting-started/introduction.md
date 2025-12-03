# 지식 그래프 소개

## 개요

**지식 그래프(Knowledge Graph)**는 현실 세계의 엔티티들과 그들 간의 관계를 그래프 구조로 표현한 지식 표현 체계입니다. 2012년 Google이 검색 엔진 개선을 위해 "Knowledge Graph"라는 용어를 대중화한 이후, 다양한 산업 분야에서 핵심 기술로 자리잡았습니다.

## 핵심 구성 요소

### 1. 엔티티 (Entity)

엔티티는 지식 그래프의 노드로, 현실 세계의 개체를 나타냅니다.

```
예시:
- 사람: 홍길동, 이순신
- 조직: 삼성전자, 서울대학교
- 장소: 서울, 뉴욕
- 개념: 인공지능, 머신러닝
```

### 2. 관계 (Relationship)

관계는 엔티티 간의 연결을 나타내는 엣지입니다.

```
예시:
- (홍길동)-[WORKS_AT]->(삼성전자)
- (서울)-[CAPITAL_OF]->(대한민국)
- (머신러닝)-[SUBFIELD_OF]->(인공지능)
```

### 3. 속성 (Property)

엔티티나 관계가 가지는 추가 정보입니다.

```
예시:
- Person {name: "홍길동", age: 30, email: "hong@example.com"}
- WORKS_AT {since: 2020, position: "Engineer"}
```

## 트리플 (Triple)

지식 그래프의 기본 단위는 **트리플(Triple)**입니다:

```
(Subject) - [Predicate] -> (Object)
(주어)    - [술어]      -> (목적어)
```

예시:
```
(서울) - [수도] -> (대한민국)
(홍길동) - [나이] -> (30)
(삼성전자) - [본사위치] -> (서울)
```

## RDF vs Property Graph

지식 그래프를 표현하는 두 가지 주요 모델이 있습니다:

| 특성 | RDF | Property Graph |
|------|-----|----------------|
| **표준** | W3C 표준 | 사실상 표준 |
| **스키마** | RDFS/OWL | 유연한 스키마 |
| **쿼리 언어** | SPARQL | Cypher, Gremlin |
| **관계 속성** | 제한적 (Reification) | 네이티브 지원 |
| **사용 사례** | 시맨틱 웹, 온톨로지 | 소셜 네트워크, 추천 |

## 지식 그래프의 장점

### 1. 유연성
```python
# 새로운 엔티티 타입을 동적으로 추가
CREATE (:NewEntityType {property: "value"})
```

### 2. 연결성
```cypher
# 복잡한 관계 패턴 탐색
MATCH path = (start)-[*1..5]-(end)
WHERE start.name = "홍길동"
RETURN path
```

### 3. 추론 능력
```
규칙: IF (X)-[부모]->(Y) AND (Y)-[부모]->(Z) THEN (X)-[조부모]->(Z)
```

## 다음 단계

- [빠른 시작](quickstart.md)에서 실습을 시작하세요
- [설치 가이드](installation.md)에서 환경을 구성하세요
