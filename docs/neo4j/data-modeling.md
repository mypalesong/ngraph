# Neo4j 데이터 모델링

## 그래프 모델링 원칙

### 1. 도메인 중심 설계

현실 세계의 개념을 그래프로 직접 매핑합니다.

```
도메인 개념          →  그래프 요소
─────────────────────────────────
명사 (사람, 제품)    →  노드 (레이블)
동사 (구매, 친구)    →  관계 (타입)
형용사/부사          →  속성
```

### 2. 화이트보드 친화적

```
┌─────────────────────────────────────────────────────────────┐
│  화이트보드 다이어그램                                        │
│                                                             │
│    [고객] ──구매──> [주문] ──포함──> [제품]                   │
│       │              │                 │                    │
│       └──거주──> [주소]         [카테고리] <──속함──┘        │
│                                                             │
│  ↓ 그대로 Neo4j로 변환                                       │
│                                                             │
│    (:Customer)-[:PURCHASED]->(:Order)-[:CONTAINS]->(:Product)│
│         │                       │                    │       │
│         └─[:LIVES_IN]->(:Address)    (:Category)<-[:BELONGS]-┘│
└─────────────────────────────────────────────────────────────┘
```

## 노드 vs 관계 결정

### 노드로 만들어야 할 것

- 독립적인 엔티티
- 여러 관계의 대상이 되는 것
- 자체 속성이 풍부한 것

### 관계로 만들어야 할 것

- 두 엔티티 간의 연결
- 동사로 표현되는 것
- 시작과 끝이 명확한 것

### 예시: 주문 모델링

```cypher
// ✅ 좋은 모델: 주문을 노드로
(:Customer)-[:PLACED]->(:Order)-[:CONTAINS]->(:Product)
// 장점: 주문에 여러 상품, 배송 정보 등 연결 가능

// ❌ 나쁜 모델: 주문을 관계로
(:Customer)-[:ORDERED {date, items}]->(:Product)
// 단점: 주문당 하나의 상품만 표현 가능
```

## 레이블 전략

### 단일 vs 다중 레이블

```cypher
// 다중 레이블 활용
CREATE (p:Person:Employee:Manager {name: '김부장'})
CREATE (p:Person:Customer:VIP {name: '이고객'})

// 쿼리 시 적절한 레이블 선택
MATCH (m:Manager) RETURN m                    // 관리자만
MATCH (p:Person) RETURN p                     // 모든 사람
MATCH (v:VIP:Customer) RETURN v               // VIP 고객만
```

### 레이블 네이밍

```
✅ 좋은 예: Person, Company, Product, Order
❌ 나쁜 예: person, PERSON, Persons, PersonNode
```

## 관계 방향성

### 자연스러운 방향

```cypher
// 자연스러운 방향 선택
(person)-[:WORKS_AT]->(company)      // 사람이 회사에서 일한다
(person)-[:PURCHASED]->(product)     // 사람이 제품을 구매한다
(movie)<-[:ACTED_IN]-(actor)         // 배우가 영화에 출연한다

// 쿼리 시 방향 무시 가능
MATCH (p:Person)-[:WORKS_AT]-(c:Company)  // 양방향 탐색
```

### 양방향 관계 피하기

```cypher
// ❌ 중복된 양방향 관계
(a)-[:FRIEND]->(b)
(b)-[:FRIEND]->(a)

// ✅ 단방향 + 양방향 쿼리
(a)-[:FRIEND]->(b)
// 쿼리: MATCH (a)-[:FRIEND]-(b)
```

## 속성 설계

### 인덱스 고려

```cypher
// 자주 검색하는 속성은 인덱스 생성
CREATE INDEX person_email FOR (p:Person) ON (p.email)
CREATE INDEX person_name FOR (p:Person) ON (p.name)

// 복합 인덱스
CREATE INDEX person_name_city FOR (p:Person) ON (p.name, p.city)
```

### 속성 타입

```cypher
// 지원되는 타입
CREATE (p:Person {
  name: '홍길동',           // String
  age: 30,                  // Integer
  height: 175.5,            // Float
  active: true,             // Boolean
  joinDate: date('2020-01-15'),  // Date
  createdAt: datetime(),    // DateTime
  skills: ['Python', 'Neo4j'],   // List
  point: point({x: 1, y: 2})     // Point (공간)
})
```

## 슈퍼노드 문제

### 문제

```
인기 제품에 수백만 개의 PURCHASED 관계 → 성능 저하
```

### 해결책 1: 중간 노드 추가

```cypher
// Before: 직접 연결
(:User)-[:PURCHASED]->(:Product)

// After: 중간 노드
(:User)-[:MADE]->(:Purchase {date, quantity})-[:OF]->(:Product)
```

### 해결책 2: 관계 팬아웃

```cypher
// 시간대별 분산
(:User)-[:PURCHASED_2024_Q1]->(:Product)
(:User)-[:PURCHASED_2024_Q2]->(:Product)
```

## 시간 기반 모델링

### 이벤트 소싱

```cypher
// 모든 변경을 이벤트로 기록
CREATE (e:Event:StatusChange {
  timestamp: datetime(),
  fromStatus: 'pending',
  toStatus: 'shipped'
})

MATCH (order:Order {id: '12345'})
CREATE (order)-[:HAS_EVENT]->(e)
```

### 유효 기간

```cypher
// 관계에 기간 표시
(p:Person)-[:EMPLOYED_AT {
  startDate: date('2020-01-01'),
  endDate: date('2023-12-31')
}]->(c:Company)

// 현재 직원 조회
MATCH (p:Person)-[r:EMPLOYED_AT]->(c:Company)
WHERE r.startDate <= date() AND (r.endDate IS NULL OR r.endDate >= date())
RETURN p, c
```

## 실전 모델 예시

### E-Commerce

```cypher
// 스키마
(:Customer {id, name, email})
(:Order {id, date, status, total})
(:Product {id, name, price, stock})
(:Category {id, name})
(:Address {street, city, zip})

// 관계
(:Customer)-[:PLACED]->(:Order)
(:Order)-[:CONTAINS {quantity, price}]->(:Product)
(:Product)-[:IN_CATEGORY]->(:Category)
(:Customer)-[:LIVES_AT]->(:Address)
(:Order)-[:SHIPPED_TO]->(:Address)
```

### 소셜 네트워크

```cypher
// 스키마
(:User {id, name, email, joinDate})
(:Post {id, content, createdAt})
(:Comment {id, content, createdAt})
(:Tag {name})

// 관계
(:User)-[:FOLLOWS]->(:User)
(:User)-[:POSTED]->(:Post)
(:User)-[:LIKED]->(:Post)
(:User)-[:COMMENTED {text}]->(:Post)
(:Post)-[:TAGGED]->(:Tag)
(:Post)-[:REPLY_TO]->(:Post)
```

### 조직도

```cypher
// 스키마
(:Employee {id, name, title, hireDate})
(:Department {id, name, budget})
(:Project {id, name, deadline})

// 관계
(:Employee)-[:MANAGES]->(:Employee)
(:Employee)-[:BELONGS_TO]->(:Department)
(:Employee)-[:WORKS_ON {role, allocation}]->(:Project)
(:Department)-[:PARENT_OF]->(:Department)
```

## 다음 단계

!!! tip "모델 검증"
    [Python 연동](python-driver.md)에서 설계한 모델을 코드로 구현해보세요.
