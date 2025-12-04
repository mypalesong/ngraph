# Cypher 심화

## 서브쿼리

### CALL 서브쿼리

```cypher
// 각 사람별로 친구 수 계산
MATCH (p:Person)
CALL {
  WITH p
  MATCH (p)-[:FRIEND]-(friend)
  RETURN count(friend) AS friendCount
}
RETURN p.name, friendCount
ORDER BY friendCount DESC
```

### UNION

```cypher
// 두 쿼리 결과 합치기
MATCH (p:Person)-[:WORKS_AT]->(c:Company)
RETURN p.name AS name, 'Employee' AS type
UNION
MATCH (c:Company)
RETURN c.name AS name, 'Company' AS type
```

### EXISTS 서브쿼리

```cypher
// IT 회사에서 일하는 사람
MATCH (p:Person)
WHERE EXISTS {
  MATCH (p)-[:WORKS_AT]->(c:Company)
  WHERE c.industry = 'IT'
}
RETURN p.name
```

### COUNT 서브쿼리

```cypher
// 친구가 5명 이상인 사람
MATCH (p:Person)
WHERE COUNT {
  (p)-[:FRIEND]-()
} >= 5
RETURN p.name
```

---

## 리스트 처리

### 리스트 함수

```cypher
// 리스트 생성
RETURN range(1, 10) AS numbers         // [1,2,3,...,10]
RETURN [x IN range(1,5) | x * 2]       // [2,4,6,8,10]

// 리스트 조작
WITH [1, 2, 3, 4, 5] AS nums
RETURN head(nums),       // 1
       tail(nums),       // [2,3,4,5]
       last(nums),       // 5
       size(nums),       // 5
       reverse(nums)     // [5,4,3,2,1]
```

### 리스트 컴프리헨션

```cypher
// 친구 이름 리스트
MATCH (p:Person {name: 'Alice'})-[:FRIEND]->(f)
RETURN [friend IN collect(f) | friend.name] AS friendNames

// 조건부 필터링
WITH [1, 2, 3, 4, 5, 6, 7, 8, 9, 10] AS nums
RETURN [x IN nums WHERE x % 2 = 0 | x * x] AS evenSquares
// [4, 16, 36, 64, 100]
```

### UNWIND

```cypher
// 리스트를 행으로 펼치기
UNWIND ['Python', 'Java', 'Go'] AS skill
CREATE (s:Skill {name: skill})
RETURN s

// 관계 일괄 생성
MATCH (p:Person {name: 'Alice'})
UNWIND ['Python', 'Neo4j', 'Docker'] AS skillName
MERGE (s:Skill {name: skillName})
MERGE (p)-[:HAS_SKILL]->(s)
```

---

## 맵 처리

### 맵 프로젝션

```cypher
// 노드를 맵으로 변환
MATCH (p:Person)
RETURN p {.name, .age, .city}

// 추가 필드 포함
MATCH (p:Person)-[:WORKS_AT]->(c:Company)
RETURN p {.name, .age, company: c.name}

// 동적 맵
MATCH (p:Person)
RETURN p {.*, type: 'Person', timestamp: datetime()}
```

### 맵 함수

```cypher
WITH {name: '홍길동', age: 30, city: '서울'} AS person
RETURN keys(person),           // ['name', 'age', 'city']
       person.name,            // '홍길동'
       person['age']           // 30
```

---

## 경로 처리

### 경로 함수

```cypher
MATCH path = (a:Person)-[:FRIEND*1..3]->(b:Person)
WHERE a.name = 'Alice'
RETURN path,
       nodes(path),            // 경로의 모든 노드
       relationships(path),    // 경로의 모든 관계
       length(path)            // 경로 길이
```

### 경로 패턴

```cypher
// 특정 조건의 경로
MATCH path = (start:Person {name: 'Alice'})-[:FRIEND*]-(end:Person)
WHERE ALL(node IN nodes(path) WHERE node.age > 20)
RETURN path

// 관계 조건
MATCH path = (a)-[rels:FRIEND*1..5]->(b)
WHERE ALL(r IN rels WHERE r.since > 2018)
RETURN path
```

---

## 날짜/시간

### 현재 시간

```cypher
RETURN datetime() AS now,
       date() AS today,
       time() AS currentTime,
       localdatetime() AS localNow
```

### 날짜 생성

```cypher
RETURN date('2024-01-15') AS specificDate,
       datetime('2024-01-15T10:30:00') AS specificDateTime,
       date({year: 2024, month: 1, day: 15}) AS constructedDate
```

### 날짜 연산

```cypher
// 기간 계산
WITH date('2024-01-01') AS start, date('2024-12-31') AS end
RETURN duration.between(start, end) AS diff

// 날짜 더하기/빼기
WITH date() AS today
RETURN today + duration('P30D') AS after30Days,
       today - duration('P1M') AS oneMonthAgo

// 기간 필터
MATCH (e:Event)
WHERE e.date > date() - duration('P7D')
RETURN e
```

### 날짜 추출

```cypher
WITH datetime() AS now
RETURN now.year, now.month, now.day,
       now.hour, now.minute, now.second,
       now.dayOfWeek, now.dayOfYear
```

---

## CASE 표현식

```cypher
// 단순 CASE
MATCH (p:Person)
RETURN p.name,
       CASE p.age
         WHEN 20 THEN '20대'
         WHEN 30 THEN '30대'
         ELSE '기타'
       END AS ageGroup

// 조건부 CASE
MATCH (p:Person)
RETURN p.name,
       CASE
         WHEN p.age < 20 THEN '청소년'
         WHEN p.age < 30 THEN '청년'
         WHEN p.age < 50 THEN '중년'
         ELSE '장년'
       END AS generation
```

---

## 트랜잭션 처리

### 명시적 트랜잭션

```cypher
// Browser에서
:BEGIN
CREATE (a:Account {id: 'A', balance: 1000})
CREATE (b:Account {id: 'B', balance: 500})
:COMMIT

// 롤백
:BEGIN
CREATE (x:Test)
:ROLLBACK
```

### Python에서

```python
def transfer_money(tx, from_id, to_id, amount):
    tx.run("""
        MATCH (from:Account {id: $from_id})
        MATCH (to:Account {id: $to_id})
        SET from.balance = from.balance - $amount
        SET to.balance = to.balance + $amount
    """, from_id=from_id, to_id=to_id, amount=amount)

with driver.session() as session:
    session.execute_write(transfer_money, 'A', 'B', 100)
```

---

## 쿼리 최적화

### 실행 계획 확인

```cypher
// 예상 실행 계획
EXPLAIN MATCH (p:Person)-[:KNOWS]->(f:Person)
WHERE p.name = 'Alice'
RETURN f.name

// 실제 실행 계획 (데이터 포함)
PROFILE MATCH (p:Person)-[:KNOWS]->(f:Person)
WHERE p.name = 'Alice'
RETURN f.name
```

### 인덱스 활용

```cypher
// 인덱스 힌트
MATCH (p:Person)
USING INDEX p:Person(name)
WHERE p.name = 'Alice'
RETURN p

// 스캔 힌트
MATCH (p:Person)
USING SCAN p:Person
WHERE p.age > 25
RETURN p
```

### 최적화 팁

```cypher
// ❌ 비효율적
MATCH (a), (b)
WHERE a.name = 'Alice' AND (a)-[:KNOWS]->(b)
RETURN b

// ✅ 효율적
MATCH (a:Person {name: 'Alice'})-[:KNOWS]->(b)
RETURN b

// ❌ 모든 속성 반환
MATCH (p:Person) RETURN p

// ✅ 필요한 속성만 반환
MATCH (p:Person) RETURN p.name, p.age
```

---

## 고급 패턴

### 재귀 패턴

```cypher
// 조직도 탐색
MATCH path = (ceo:Employee {title: 'CEO'})-[:MANAGES*]->(emp:Employee)
RETURN emp.name, length(path) AS level
ORDER BY level

// 특정 깊이까지
MATCH (root:Category {name: 'Electronics'})-[:SUBCATEGORY*0..3]->(sub)
RETURN sub.name
```

### 양방향 탐색

```cypher
// 공통 친구 찾기
MATCH (a:Person {name: 'Alice'})-[:FRIEND]-(mutual)-[:FRIEND]-(b:Person {name: 'Bob'})
WHERE a <> b
RETURN mutual.name AS commonFriend
```

### 부정 패턴

```cypher
// 관계가 없는 쌍 찾기
MATCH (a:Person), (b:Person)
WHERE a <> b AND NOT (a)-[:KNOWS]-(b)
RETURN a.name, b.name
LIMIT 10
```

---

## 다음 단계

!!! success "심화 완료!"
    [데이터 모델링](data-modeling.md)에서 효과적인 그래프 설계를 배워보세요.
