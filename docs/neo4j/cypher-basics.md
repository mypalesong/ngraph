# Cypher 기초

## Cypher란?

**Cypher**는 Neo4j의 선언적 그래프 쿼리 언어입니다. SQL과 유사하지만 그래프 패턴을 직관적으로 표현할 수 있습니다.

## 기본 문법

### 노드 표현

```cypher
()                          // 익명 노드
(n)                         // 변수 n에 바인딩된 노드
(:Person)                   // Person 레이블을 가진 노드
(p:Person)                  // 변수 p에 바인딩된 Person 노드
(p:Person {name: '홍길동'})  // 속성이 있는 Person 노드
(p:Person:Employee)         // 다중 레이블
```

### 관계 표현

```cypher
-->                              // 방향 있는 관계
-[r]->                           // 변수 r에 바인딩
-[:KNOWS]->                      // KNOWS 타입 관계
-[r:KNOWS]->                     // 변수와 타입
-[r:KNOWS {since: 2020}]->       // 속성 포함
-[*1..3]->                       // 1~3 홉
-[:KNOWS|FRIEND]->               // OR 관계
```

### 패턴 예시

```cypher
// Alice가 Bob을 안다
(alice:Person {name: 'Alice'})-[:KNOWS]->(bob:Person {name: 'Bob'})

// 누군가가 영화에 출연했다
(:Person)-[:ACTED_IN]->(:Movie)

// 친구의 친구
(:Person)-[:FRIEND]->(:Person)-[:FRIEND]->(:Person)
```

---

## CREATE - 생성

### 노드 생성

```cypher
// 단일 노드
CREATE (p:Person {name: '홍길동', age: 30})
RETURN p

// 여러 노드
CREATE (a:Person {name: 'Alice'})
CREATE (b:Person {name: 'Bob'})
CREATE (c:Company {name: 'TechCorp'})
RETURN a, b, c

// 노드와 관계 동시 생성
CREATE (a:Person {name: 'Alice'})-[:FRIEND {since: 2020}]->(b:Person {name: 'Bob'})
RETURN a, b
```

### 관계 생성

```cypher
// 기존 노드에 관계 추가
MATCH (a:Person {name: 'Alice'})
MATCH (b:Person {name: 'Bob'})
CREATE (a)-[:KNOWS {since: 2023}]->(b)

// 여러 관계
MATCH (p:Person {name: 'Alice'})
MATCH (c:Company {name: 'TechCorp'})
CREATE (p)-[:WORKS_AT {role: 'Engineer', since: 2020}]->(c)
CREATE (p)-[:INVESTED_IN {amount: 10000}]->(c)
```

---

## MATCH - 조회

### 기본 조회

```cypher
// 모든 노드
MATCH (n) RETURN n LIMIT 25

// 특정 레이블
MATCH (p:Person) RETURN p

// 속성 조건
MATCH (p:Person {name: '홍길동'})
RETURN p

// WHERE 절
MATCH (p:Person)
WHERE p.age > 25
RETURN p.name, p.age
```

### 관계 조회

```cypher
// 특정 관계
MATCH (a:Person)-[:KNOWS]->(b:Person)
RETURN a.name, b.name

// 양방향
MATCH (a:Person)-[:KNOWS]-(b:Person)
RETURN a.name, b.name

// 관계 속성
MATCH (a)-[r:KNOWS]->(b)
WHERE r.since > 2020
RETURN a.name, b.name, r.since
```

### 경로 조회

```cypher
// 가변 길이 경로
MATCH path = (a:Person)-[:KNOWS*1..3]->(b:Person)
WHERE a.name = 'Alice'
RETURN path

// 최단 경로
MATCH path = shortestPath(
  (a:Person {name: 'Alice'})-[*]-(b:Person {name: 'Bob'})
)
RETURN path, length(path)

// 모든 최단 경로
MATCH path = allShortestPaths(
  (a:Person {name: 'Alice'})-[*]-(b:Person {name: 'Bob'})
)
RETURN path
```

---

## WHERE - 조건

### 비교 연산자

```cypher
WHERE p.age = 30            // 같음
WHERE p.age <> 30           // 다름
WHERE p.age > 25            // 크다
WHERE p.age >= 25           // 크거나 같다
WHERE p.age < 30            // 작다
WHERE p.age <= 30           // 작거나 같다
```

### 논리 연산자

```cypher
WHERE p.age > 25 AND p.age < 35
WHERE p.city = '서울' OR p.city = '부산'
WHERE NOT p.status = 'inactive'
WHERE p.name IS NOT NULL
```

### 문자열 연산

```cypher
WHERE p.name STARTS WITH '홍'
WHERE p.name ENDS WITH '동'
WHERE p.name CONTAINS '길'
WHERE p.name =~ '홍.*'          // 정규식
WHERE toLower(p.name) = 'alice'
```

### 리스트 연산

```cypher
WHERE p.age IN [25, 30, 35]
WHERE 'Python' IN p.skills
WHERE ALL(x IN p.scores WHERE x > 80)
WHERE ANY(x IN p.scores WHERE x > 90)
WHERE NONE(x IN p.tags WHERE x = 'spam')
```

### 패턴 존재 확인

```cypher
// 관계가 존재하는 경우
WHERE (p)-[:KNOWS]->(:Person)

// 관계가 없는 경우
WHERE NOT (p)-[:KNOWS]->(:Person)

// EXISTS 사용
WHERE EXISTS {
  MATCH (p)-[:WORKS_AT]->(c:Company)
  WHERE c.industry = 'IT'
}
```

---

## RETURN - 반환

### 기본 반환

```cypher
// 노드 전체
RETURN p

// 특정 속성
RETURN p.name, p.age

// 별칭
RETURN p.name AS 이름, p.age AS 나이

// 계산
RETURN p.name, p.age, 2024 - p.birthYear AS age
```

### 집계 함수

```cypher
// 카운트
MATCH (p:Person)
RETURN count(p) AS total

// 합계, 평균, 최대, 최소
MATCH (p:Person)
RETURN sum(p.age), avg(p.age), max(p.age), min(p.age)

// 수집
MATCH (p:Person)-[:KNOWS]->(f:Person)
RETURN p.name, collect(f.name) AS friends

// DISTINCT
RETURN DISTINCT p.city
```

### 정렬 및 제한

```cypher
MATCH (p:Person)
RETURN p.name, p.age
ORDER BY p.age DESC, p.name ASC
SKIP 10
LIMIT 5
```

---

## SET / REMOVE - 수정

### 속성 수정

```cypher
// 속성 설정
MATCH (p:Person {name: '홍길동'})
SET p.age = 31
RETURN p

// 여러 속성
SET p.age = 31, p.city = '서울'

// 속성 추가 (기존 유지)
SET p += {city: '서울', active: true}

// 속성 덮어쓰기 (기존 삭제)
SET p = {name: '홍길동', age: 31}
```

### 속성 삭제

```cypher
// 단일 속성 삭제
MATCH (p:Person {name: '홍길동'})
REMOVE p.age
RETURN p

// NULL로 설정 (동일 효과)
SET p.age = null
```

### 레이블 수정

```cypher
// 레이블 추가
SET p:Employee:Manager

// 레이블 삭제
REMOVE p:Manager
```

---

## DELETE - 삭제

### 노드 삭제

```cypher
// 관계 없는 노드 삭제
MATCH (p:Person {name: '홍길동'})
DELETE p

// 관계와 함께 삭제
MATCH (p:Person {name: '홍길동'})
DETACH DELETE p
```

### 관계 삭제

```cypher
// 특정 관계 삭제
MATCH (a:Person)-[r:KNOWS]->(b:Person)
WHERE a.name = 'Alice' AND b.name = 'Bob'
DELETE r
```

### 전체 삭제

```cypher
// 모든 데이터 삭제 (주의!)
MATCH (n) DETACH DELETE n
```

---

## MERGE - Upsert

`MERGE`는 패턴이 존재하면 MATCH, 없으면 CREATE합니다.

```cypher
// 노드 upsert
MERGE (p:Person {name: '홍길동'})
ON CREATE SET p.created = datetime()
ON MATCH SET p.lastSeen = datetime()
RETURN p

// 관계 upsert
MATCH (a:Person {name: 'Alice'})
MATCH (b:Person {name: 'Bob'})
MERGE (a)-[r:KNOWS]->(b)
ON CREATE SET r.since = date()
RETURN a, r, b
```

---

## 실습 예제

### 소셜 네트워크 구축

```cypher
// 1. 사용자 생성
CREATE (alice:Person {name: 'Alice', age: 28, city: '서울'})
CREATE (bob:Person {name: 'Bob', age: 32, city: '부산'})
CREATE (charlie:Person {name: 'Charlie', age: 25, city: '서울'})
CREATE (diana:Person {name: 'Diana', age: 30, city: '대전'})

// 2. 친구 관계
CREATE (alice)-[:FRIEND {since: 2020}]->(bob)
CREATE (alice)-[:FRIEND {since: 2021}]->(charlie)
CREATE (bob)-[:FRIEND {since: 2019}]->(diana)
CREATE (charlie)-[:FRIEND {since: 2022}]->(diana)

// 3. 서울 사는 친구의 친구 찾기
MATCH (me:Person {name: 'Alice'})-[:FRIEND]->(friend)-[:FRIEND]->(fof:Person)
WHERE fof.city = '서울' AND me <> fof AND NOT (me)-[:FRIEND]-(fof)
RETURN DISTINCT fof.name

// 4. 네트워크 분석
MATCH (p:Person)-[:FRIEND]-(friend)
RETURN p.name, count(friend) AS friendCount
ORDER BY friendCount DESC
```

---

## 다음 단계

!!! success "기초 완료!"
    [Cypher 심화](cypher-advanced.md)에서 고급 패턴과 최적화를 배워보세요.
