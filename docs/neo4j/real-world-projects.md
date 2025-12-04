# Neo4j 실전 프로젝트

## 프로젝트 1: 영화 추천 시스템

### 데이터 모델

```cypher
// 스키마
(:User {id, name, email})
(:Movie {id, title, year, genres})
(:Actor {id, name})
(:Director {id, name})
(:Genre {name})

// 관계
(:User)-[:RATED {rating, timestamp}]->(:Movie)
(:User)-[:WATCHLIST]->(:Movie)
(:Actor)-[:ACTED_IN {role}]->(:Movie)
(:Director)-[:DIRECTED]->(:Movie)
(:Movie)-[:IN_GENRE]->(:Genre)
```

### 데이터 로드

```cypher
// CSV에서 영화 로드
LOAD CSV WITH HEADERS FROM 'file:///movies.csv' AS row
CREATE (m:Movie {
  id: row.movieId,
  title: row.title,
  year: toInteger(row.year)
})

// 장르 연결
LOAD CSV WITH HEADERS FROM 'file:///movies.csv' AS row
MATCH (m:Movie {id: row.movieId})
UNWIND split(row.genres, '|') AS genreName
MERGE (g:Genre {name: genreName})
MERGE (m)-[:IN_GENRE]->(g)

// 평점 로드
LOAD CSV WITH HEADERS FROM 'file:///ratings.csv' AS row
MATCH (u:User {id: row.userId})
MATCH (m:Movie {id: row.movieId})
CREATE (u)-[:RATED {
  rating: toFloat(row.rating),
  timestamp: toInteger(row.timestamp)
}]->(m)
```

### 추천 알고리즘

```cypher
// 1. 협업 필터링: 비슷한 취향 사용자가 좋아한 영화
MATCH (me:User {id: $userId})-[r1:RATED]->(m:Movie)<-[r2:RATED]-(other:User)
WHERE r1.rating >= 4 AND r2.rating >= 4
WITH other, count(m) AS commonMovies
ORDER BY commonMovies DESC
LIMIT 10

MATCH (other)-[r:RATED]->(rec:Movie)
WHERE r.rating >= 4 AND NOT (me)-[:RATED]->(rec)
RETURN rec.title, count(*) AS score, avg(r.rating) AS avgRating
ORDER BY score DESC, avgRating DESC
LIMIT 20

// 2. 콘텐츠 기반: 좋아한 영화와 같은 장르/배우
MATCH (me:User {id: $userId})-[r:RATED]->(m:Movie)-[:IN_GENRE]->(g:Genre)
WHERE r.rating >= 4
WITH me, g, count(*) AS genreScore
ORDER BY genreScore DESC
LIMIT 3

MATCH (rec:Movie)-[:IN_GENRE]->(g)
WHERE NOT (me)-[:RATED]->(rec)
WITH rec, sum(genreScore) AS score
ORDER BY score DESC
LIMIT 20
RETURN rec.title, score

// 3. 하이브리드: GDS 활용
CALL gds.graph.project('movieGraph',
  ['User', 'Movie'],
  {RATED: {properties: 'rating'}}
)

CALL gds.nodeSimilarity.stream('movieGraph')
YIELD node1, node2, similarity
WHERE gds.util.asNode(node1):User AND gds.util.asNode(node2):User
RETURN gds.util.asNode(node1).name AS user1,
       gds.util.asNode(node2).name AS user2,
       similarity
ORDER BY similarity DESC
```

### Python API

```python
from fastapi import FastAPI, HTTPException
from neo4j import GraphDatabase

app = FastAPI()
driver = GraphDatabase.driver("bolt://localhost:7687", auth=("neo4j", "password"))

@app.get("/users/{user_id}/recommendations")
def get_recommendations(user_id: str, limit: int = 10):
    query = """
    MATCH (me:User {id: $userId})-[r:RATED]->(m:Movie)<-[r2:RATED]-(other:User)
    WHERE r.rating >= 4 AND r2.rating >= 4
    WITH other, count(m) AS overlap
    ORDER BY overlap DESC LIMIT 5

    MATCH (other)-[r3:RATED]->(rec:Movie)
    WHERE r3.rating >= 4 AND NOT EXISTS((me)-[:RATED]->(rec))
    RETURN rec.id AS id, rec.title AS title,
           count(*) AS score, avg(r3.rating) AS avgRating
    ORDER BY score DESC, avgRating DESC
    LIMIT $limit
    """
    with driver.session() as session:
        result = session.run(query, userId=user_id, limit=limit)
        return [dict(r) for r in result]
```

---

## 프로젝트 2: 사기 탐지 시스템

### 데이터 모델

```cypher
(:Account {id, type, balance, createdAt})
(:Transaction {id, amount, timestamp, status})
(:Device {id, fingerprint, ip})
(:Location {city, country, lat, lon})

(:Account)-[:SENT]->(Transaction)-[:RECEIVED]->(:Account)
(:Transaction)-[:USING]->(:Device)
(:Transaction)-[:FROM_LOCATION]->(:Location)
(:Account)-[:OWNED_BY]->(:Customer)
```

### 사기 패턴 탐지

```cypher
// 1. 순환 거래 탐지
MATCH path = (a:Account)-[:SENT]->(:Transaction)-[:RECEIVED]->(b:Account)-[:SENT*1..5]->(:Transaction)-[:RECEIVED]->(a)
WHERE ALL(t IN [n IN nodes(path) WHERE n:Transaction] WHERE t.amount > 10000)
RETURN a.id AS account, length(path) AS hops,
       reduce(sum=0, t IN [n IN nodes(path) WHERE n:Transaction] | sum + t.amount) AS totalAmount

// 2. 급격한 거래 증가
MATCH (a:Account)-[:SENT]->(t:Transaction)
WHERE t.timestamp > datetime() - duration('P7D')
WITH a, count(t) AS recentCount, sum(t.amount) AS recentAmount

MATCH (a)-[:SENT]->(t2:Transaction)
WHERE t2.timestamp > datetime() - duration('P30D')
  AND t2.timestamp <= datetime() - duration('P7D')
WITH a, recentCount, recentAmount, count(t2) AS prevCount, sum(t2.amount) AS prevAmount

WHERE recentCount > prevCount * 3 OR recentAmount > prevAmount * 5
RETURN a.id, recentCount, prevCount, recentAmount, prevAmount

// 3. 의심스러운 네트워크 (공유 디바이스/위치)
MATCH (a1:Account)-[:SENT]->(:Transaction)-[:USING]->(d:Device)<-[:USING]-(:Transaction)<-[:SENT]-(a2:Account)
WHERE a1 <> a2
WITH a1, a2, d, count(*) AS sharedTransactions
WHERE sharedTransactions > 3
RETURN a1.id, a2.id, d.id, sharedTransactions
ORDER BY sharedTransactions DESC

// 4. 실시간 위험 점수 계산
MATCH (a:Account {id: $accountId})
CALL {
  WITH a
  MATCH (a)-[:SENT]->(t:Transaction)-[:RECEIVED]->(target)
  WHERE t.timestamp > datetime() - duration('PT1H')
  RETURN count(t) * 10 AS velocityScore
  UNION ALL
  WITH a
  MATCH path = (a)-[:SENT*2..4]->(:Transaction)-[:RECEIVED]->(a)
  RETURN count(path) * 50 AS circularScore
  UNION ALL
  WITH a
  MATCH (a)-[:SENT]->(:Transaction)-[:FROM_LOCATION]->(l:Location)
  WHERE l.country <> 'KR'
  RETURN count(*) * 20 AS foreignScore
}
RETURN a.id, sum(velocityScore) + sum(circularScore) + sum(foreignScore) AS riskScore
```

### 실시간 모니터링

```python
import asyncio
from neo4j import AsyncGraphDatabase

async def monitor_transactions():
    driver = AsyncGraphDatabase.driver("bolt://localhost:7687", auth=("neo4j", "password"))

    async with driver.session() as session:
        while True:
            result = await session.run("""
                MATCH (t:Transaction)
                WHERE t.timestamp > datetime() - duration('PT1M')
                AND t.status = 'pending'
                AND t.amount > 50000
                MATCH (sender:Account)-[:SENT]->(t)-[:RECEIVED]->(receiver:Account)
                RETURN t.id AS txId, sender.id AS from, receiver.id AS to, t.amount
            """)

            async for record in result:
                risk_score = await calculate_risk(session, record['from'])
                if risk_score > 100:
                    await flag_transaction(session, record['txId'])
                    print(f"🚨 High risk transaction: {record['txId']}")

            await asyncio.sleep(10)

asyncio.run(monitor_transactions())
```

---

## 프로젝트 3: 기업 지식 그래프

### 데이터 모델

```cypher
(:Employee {id, name, email, title, department})
(:Project {id, name, status, deadline})
(:Skill {name, category})
(:Document {id, title, type, content})
(:Meeting {id, date, topic})
(:Topic {name})

(:Employee)-[:WORKS_ON {role, allocation}]->(:Project)
(:Employee)-[:HAS_SKILL {level, since}]->(:Skill)
(:Employee)-[:AUTHORED]->(:Document)
(:Employee)-[:ATTENDED]->(:Meeting)
(:Document)-[:ABOUT]->(:Topic)
(:Project)-[:REQUIRES]->(:Skill)
(:Employee)-[:REPORTS_TO]->(:Employee)
```

### 지식 검색

```cypher
// 1. 전문가 찾기
MATCH (skill:Skill {name: $skillName})<-[r:HAS_SKILL]-(expert:Employee)
WHERE r.level IN ['expert', 'advanced']
OPTIONAL MATCH (expert)-[:WORKS_ON]->(p:Project)
WHERE p.status = 'active'
RETURN expert.name, expert.email, r.level,
       collect(DISTINCT p.name) AS activeProjects
ORDER BY CASE r.level WHEN 'expert' THEN 1 ELSE 2 END

// 2. 프로젝트 인력 갭 분석
MATCH (p:Project {name: $projectName})-[:REQUIRES]->(required:Skill)
OPTIONAL MATCH (e:Employee)-[r:HAS_SKILL]->(required)
WHERE (e)-[:WORKS_ON]->(p) AND r.level IN ['expert', 'advanced']
WITH required, collect(e.name) AS experts
RETURN required.name AS skill,
       CASE WHEN size(experts) = 0 THEN '❌ GAP' ELSE '✅ ' + toString(size(experts)) + ' experts' END AS status,
       experts

// 3. 지식 경로 탐색
MATCH path = shortestPath(
  (start:Topic {name: $from})-[:ABOUT|RELATED_TO*]-(end:Topic {name: $to})
)
RETURN [n IN nodes(path) |
  CASE WHEN n:Topic THEN n.name
       WHEN n:Document THEN '📄 ' + n.title
       ELSE labels(n)[0] END
] AS knowledgePath

// 4. 조직 네트워크 분석
CALL gds.graph.project('orgGraph', 'Employee',
  {WORKS_ON: {type: 'WORKS_ON'}, REPORTS_TO: {type: 'REPORTS_TO'}}
)

CALL gds.betweenness.stream('orgGraph')
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).name AS employee, score AS influence
ORDER BY score DESC
LIMIT 10
```

### Slack/Teams 통합

```python
from slack_bolt import App
from neo4j import GraphDatabase

app = App(token="xoxb-...")
driver = GraphDatabase.driver("bolt://localhost:7687", auth=("neo4j", "password"))

@app.command("/find-expert")
def find_expert(ack, command, respond):
    ack()
    skill = command['text']

    with driver.session() as session:
        result = session.run("""
            MATCH (s:Skill)<-[r:HAS_SKILL]-(e:Employee)
            WHERE toLower(s.name) CONTAINS toLower($skill)
            AND r.level IN ['expert', 'advanced']
            RETURN e.name AS name, e.email AS email, s.name AS skill, r.level AS level
            ORDER BY r.level
            LIMIT 5
        """, skill=skill)

        experts = list(result)

    if experts:
        blocks = [{"type": "section", "text": {"type": "mrkdwn",
            "text": f"*{skill} 전문가:*\n" + "\n".join(
                f"• {e['name']} ({e['email']}) - {e['level']}" for e in experts
            )}}]
        respond(blocks=blocks)
    else:
        respond(f"'{skill}' 전문가를 찾을 수 없습니다.")
```

---

## 프로젝트 4: 공급망 최적화

### 데이터 모델

```cypher
(:Supplier {id, name, location, leadTime})
(:Warehouse {id, name, location, capacity})
(:Product {id, name, sku, category})
(:Store {id, name, location})

(:Supplier)-[:SUPPLIES {cost, minOrder}]->(:Product)
(:Warehouse)-[:STOCKS {quantity, reorderPoint}]->(:Product)
(:Store)-[:SELLS {price}]->(:Product)
(:Warehouse)-[:SHIPS_TO {distance, cost, time}]->(:Store)
(:Supplier)-[:DELIVERS_TO {cost, time}]->(:Warehouse)
```

### 최적화 쿼리

```cypher
// 1. 최적 배송 경로
MATCH (s:Store {id: $storeId})<-[ship:SHIPS_TO]-(w:Warehouse)-[stock:STOCKS]->(p:Product {id: $productId})
WHERE stock.quantity >= $requiredQty
RETURN w.name AS warehouse,
       stock.quantity AS available,
       ship.cost AS shippingCost,
       ship.time AS deliveryTime
ORDER BY ship.cost + (ship.time * 10)  // 비용과 시간 가중치
LIMIT 1

// 2. 재고 부족 예측
MATCH (w:Warehouse)-[s:STOCKS]->(p:Product)
WHERE s.quantity < s.reorderPoint
MATCH (sup:Supplier)-[sup_rel:SUPPLIES]->(p)
RETURN w.name AS warehouse,
       p.name AS product,
       s.quantity AS current,
       s.reorderPoint AS reorderAt,
       sup.name AS supplier,
       sup_rel.cost AS unitCost,
       sup.leadTime AS leadDays
ORDER BY (s.reorderPoint - s.quantity) DESC

// 3. 공급망 리스크 분석
MATCH (sup:Supplier)-[:SUPPLIES]->(p:Product)<-[:STOCKS]-(w:Warehouse)
WITH sup, count(DISTINCT p) AS products, count(DISTINCT w) AS warehouses
MATCH (sup)-[:SUPPLIES]->(p2:Product)
WHERE NOT EXISTS {
  MATCH (other:Supplier)-[:SUPPLIES]->(p2)
  WHERE other <> sup
}
WITH sup, products, warehouses, collect(p2.name) AS singleSourceProducts
WHERE size(singleSourceProducts) > 0
RETURN sup.name AS supplier,
       products AS totalProducts,
       singleSourceProducts AS riskProducts,
       size(singleSourceProducts) AS riskLevel
ORDER BY riskLevel DESC
```

---

## 핵심 패턴 요약

| 사용 사례 | 핵심 패턴 | Neo4j 기능 |
|----------|----------|-----------|
| 추천 | 협업 필터링, 콘텐츠 유사도 | GDS nodeSimilarity |
| 사기 탐지 | 순환 탐지, 네트워크 분석 | 가변 경로, GDS |
| 지식 관리 | 경로 탐색, 중심성 분석 | shortestPath, Betweenness |
| 공급망 | 최적 경로, 리스크 분석 | Dijkstra, 패턴 매칭 |

---

!!! success "실전 프로젝트 완료!"
    이제 Neo4j로 실제 비즈니스 문제를 해결할 준비가 되었습니다.
    [활용 사례](../use-cases/enterprise-knowledge.md)에서 더 많은 예시를 확인하세요.
