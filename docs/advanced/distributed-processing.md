# 분산 처리

## 개요

대규모 지식 그래프를 위한 분산 처리 아키텍처와 도구입니다.

## 분산 그래프 DB

### Neo4j Fabric

```cypher
// 여러 데이터베이스 연합 쿼리
USE fabric.graphA
MATCH (p:Person)
RETURN p.name AS name, 'A' AS source
UNION
USE fabric.graphB
MATCH (p:Person)
RETURN p.name AS name, 'B' AS source
```

### JanusGraph + Cassandra

```java
JanusGraph graph = JanusGraphFactory.build()
    .set("storage.backend", "cql")
    .set("storage.hostname", "cassandra-cluster")
    .set("storage.cql.replication-factor", 3)
    .open();
```

## Apache Spark GraphX

```python
from pyspark.sql import SparkSession
from graphframes import GraphFrame

spark = SparkSession.builder.appName("KG").getOrCreate()

# 노드와 엣지 DataFrame
vertices = spark.createDataFrame([
    ("1", "홍길동", 30),
    ("2", "이영희", 28),
], ["id", "name", "age"])

edges = spark.createDataFrame([
    ("1", "2", "FRIEND"),
], ["src", "dst", "relationship"])

# GraphFrame 생성
g = GraphFrame(vertices, edges)

# PageRank
results = g.pageRank(resetProbability=0.15, maxIter=10)
results.vertices.show()
```

## 병렬 ETL

```python
from pyspark.sql import SparkSession

def parallel_load(partition):
    driver = GraphDatabase.driver(...)
    with driver.session() as session:
        for row in partition:
            session.run("CREATE (n:Node $props)", props=row)
    driver.close()

# 병렬 처리
rdd.foreachPartition(parallel_load)
```

## 확장 전략

| 전략 | 사용 사례 |
|------|----------|
| 샤딩 | 노드 ID 기반 분할 |
| 복제 | 읽기 성능 향상 |
| 파티셔닝 | 시간/지역 기반 분할 |

## 모니터링

```python
# Prometheus 메트릭
from prometheus_client import Counter, Histogram

query_counter = Counter('kg_queries_total', 'Total KG queries')
query_latency = Histogram('kg_query_latency_seconds', 'Query latency')

@query_latency.time()
def execute_query(query):
    query_counter.inc()
    return driver.session().run(query)
```

## 결론

지식 그래프는 데이터의 관계를 활용하여 새로운 가치를 창출합니다. 이 가이드를 통해 개념부터 구현까지 체계적으로 학습하셨기를 바랍니다.

---

문서에 대한 피드백이나 기여는 [GitHub](https://github.com/mypalesong/ngraph)에서 환영합니다.
