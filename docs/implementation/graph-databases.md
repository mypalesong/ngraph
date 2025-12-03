# 그래프 데이터베이스

## 주요 그래프 DB 비교

| 특성 | Neo4j | Amazon Neptune | JanusGraph |
|------|-------|----------------|------------|
| 모델 | Property Graph | Property + RDF | Property Graph |
| 쿼리 | Cypher | Gremlin/SPARQL | Gremlin |
| 라이선스 | 상용/커뮤니티 | AWS 관리형 | Apache 2.0 |
| 확장성 | 수직/수평 | 자동 | 수평 |

## Neo4j

### 설치 및 연결

```python
from neo4j import GraphDatabase

driver = GraphDatabase.driver(
    "bolt://localhost:7687",
    auth=("neo4j", "password")
)

with driver.session() as session:
    result = session.run("MATCH (n) RETURN count(n)")
    print(result.single()[0])
```

### APOC 확장

```cypher
// 유용한 APOC 프로시저
CALL apoc.meta.graph()  // 스키마 시각화
CALL apoc.export.json.all("export.json", {})  // 데이터 내보내기
CALL apoc.load.json("data.json") YIELD value  // 데이터 가져오기
```

## Amazon Neptune

```python
from gremlin_python.driver import client

gremlin_client = client.Client(
    'wss://your-neptune-endpoint:8182/gremlin',
    'g'
)

result = gremlin_client.submit("g.V().count()").all().result()
```

## 선택 가이드

- **Neo4j**: 빠른 개발, 풍부한 생태계
- **Neptune**: AWS 통합, 관리형 서비스
- **JanusGraph**: 대규모 분산 처리

## 다음 단계

- [ETL 파이프라인](etl-pipeline.md)에서 데이터 적재를 학습하세요
