# 데이터 모델링

## 모델링 원칙

### 1. 도메인 중심 설계

```cypher
// 비즈니스 도메인 반영
(:Customer)-[:PLACED]->(:Order)-[:CONTAINS]->(:Product)
(:Product)-[:BELONGS_TO]->(:Category)
(:Customer)-[:LIVES_IN]->(:Address)
```

### 2. 쿼리 패턴 기반

```cypher
// 자주 사용되는 쿼리 최적화
// "고객의 최근 주문 상품 카테고리"
MATCH (c:Customer {id: $id})-[:PLACED]->(o:Order)-[:CONTAINS]->(p:Product)-[:BELONGS_TO]->(cat:Category)
WHERE o.date > date() - duration('P30D')
RETURN cat.name, count(*) AS frequency
```

## 노드 설계

```cypher
// 속성 정규화
CREATE (p:Person {
  id: 'person_001',           // 고유 식별자
  name: '홍길동',              // 필수 속성
  email: 'hong@example.com',  // 인덱스 대상
  createdAt: datetime(),      // 메타데이터
  updatedAt: datetime()
})

// 복합 레이블
CREATE (e:Person:Employee:Manager {name: '김부장'})
```

## 관계 설계

```cypher
// 관계에 속성 추가
CREATE (a)-[:WORKS_AT {
  since: date('2020-01-01'),
  role: 'Engineer',
  department: '개발팀'
}]->(b)

// 시간 기반 관계
CREATE (p)-[:EMPLOYED_AT {
  startDate: date('2020-01-01'),
  endDate: date('2023-12-31')
}]->(c)
```

## 안티패턴

### 피해야 할 것

```cypher
// ❌ 슈퍼노드 (너무 많은 관계)
(:PopularProduct)<-[:VIEWED]-(:User)  // 수백만 관계

// ✅ 중간 노드로 분리
(:User)-[:HAS_VIEW]->(:View {date})-[:OF]->(:Product)
```

## 다음 단계

- [그래프 데이터베이스](graph-databases.md) 비교를 확인하세요
