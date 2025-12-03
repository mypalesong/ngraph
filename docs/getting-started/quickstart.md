# 빠른 시작

이 가이드에서는 10분 안에 지식 그래프를 구축하고 쿼리하는 방법을 배웁니다.

## 1. Neo4j 시작하기

### Docker로 실행

```bash
docker run -d \
  --name neo4j \
  -p 7474:7474 -p 7687:7687 \
  -e NEO4J_AUTH=neo4j/password123 \
  neo4j:latest
```

### 브라우저 접속

`http://localhost:7474`에서 Neo4j Browser에 접속합니다.

## 2. 첫 번째 그래프 생성

### 노드 생성

```cypher
// 사람 노드 생성
CREATE (alice:Person {name: '앨리스', age: 28, city: '서울'})
CREATE (bob:Person {name: '밥', age: 32, city: '부산'})
CREATE (charlie:Person {name: '찰리', age: 25, city: '서울'})

// 회사 노드 생성
CREATE (techCorp:Company {name: '테크코퍼레이션', industry: 'IT'})
CREATE (dataInc:Company {name: '데이터인크', industry: '데이터분석'})
```

### 관계 생성

```cypher
// 친구 관계
MATCH (a:Person {name: '앨리스'}), (b:Person {name: '밥'})
CREATE (a)-[:FRIENDS_WITH {since: 2020}]->(b)

// 직장 관계
MATCH (a:Person {name: '앨리스'}), (c:Company {name: '테크코퍼레이션'})
CREATE (a)-[:WORKS_AT {role: 'Engineer', since: 2021}]->(c)
```

## 3. 그래프 쿼리

### 기본 쿼리

```cypher
// 모든 사람 조회
MATCH (p:Person)
RETURN p.name, p.age, p.city
```

### 관계 탐색

```cypher
// 앨리스의 친구 찾기
MATCH (alice:Person {name: '앨리스'})-[:FRIENDS_WITH]-(friend)
RETURN friend.name
```

### 경로 탐색

```cypher
// 2단계 이내의 모든 연결 찾기
MATCH path = (p:Person {name: '앨리스'})-[*1..2]-(connected)
RETURN path
```

## 4. Python에서 사용하기

### 설치

```bash
pip install neo4j
```

### 연결 및 쿼리

```python
from neo4j import GraphDatabase

class KnowledgeGraph:
    def __init__(self, uri, user, password):
        self.driver = GraphDatabase.driver(uri, auth=(user, password))

    def close(self):
        self.driver.close()

    def create_person(self, name, age, city):
        with self.driver.session() as session:
            session.run(
                "CREATE (p:Person {name: $name, age: $age, city: $city})",
                name=name, age=age, city=city
            )

    def find_friends(self, name):
        with self.driver.session() as session:
            result = session.run(
                """
                MATCH (p:Person {name: $name})-[:FRIENDS_WITH]-(friend)
                RETURN friend.name AS name
                """,
                name=name
            )
            return [record["name"] for record in result]

# 사용 예시
kg = KnowledgeGraph("bolt://localhost:7687", "neo4j", "password123")
kg.create_person("다영", 27, "서울")
friends = kg.find_friends("앨리스")
print(f"앨리스의 친구들: {friends}")
kg.close()
```

## 5. 시각화

### Neo4j Browser

Neo4j Browser에서 직접 그래프를 시각화할 수 있습니다:

```cypher
// 전체 그래프 시각화
MATCH (n)-[r]->(m)
RETURN n, r, m
```

### Python 시각화

```python
import networkx as nx
import matplotlib.pyplot as plt

# NetworkX 그래프 생성
G = nx.DiGraph()

# 노드와 엣지 추가
G.add_node("앨리스", type="Person")
G.add_node("테크코퍼레이션", type="Company")
G.add_edge("앨리스", "테크코퍼레이션", relation="WORKS_AT")

# 시각화
pos = nx.spring_layout(G)
nx.draw(G, pos, with_labels=True, node_color='lightblue',
        node_size=2000, font_size=10, arrows=True)
plt.show()
```

## 다음 단계

- [설치 가이드](installation.md)에서 프로덕션 환경을 구성하세요
- [핵심 개념](../concepts/what-is-kg.md)에서 이론을 심화 학습하세요
