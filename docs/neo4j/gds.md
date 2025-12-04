# Graph Data Science (GDS)

## GDS란?

**Neo4j Graph Data Science**는 그래프 알고리즘과 머신러닝을 위한 라이브러리입니다. 60개 이상의 알고리즘을 제공합니다.

## 설치

```bash
# Docker
docker run -e NEO4J_PLUGINS='["graph-data-science"]' neo4j:latest

# neo4j.conf
dbms.security.procedures.unrestricted=gds.*
dbms.security.procedures.allowlist=gds.*
```

### 설치 확인

```cypher
RETURN gds.version()
CALL gds.list()
```

---

## 그래프 프로젝션

GDS는 인메모리 그래프를 생성하여 알고리즘을 실행합니다.

### 네이티브 프로젝션

```cypher
// 기본 프로젝션
CALL gds.graph.project(
  'myGraph',           // 그래프 이름
  'Person',            // 노드 레이블
  'KNOWS'              // 관계 타입
)

// 여러 레이블과 관계
CALL gds.graph.project(
  'socialGraph',
  ['Person', 'Company'],
  ['KNOWS', 'WORKS_AT']
)

// 속성 포함
CALL gds.graph.project(
  'weightedGraph',
  'Person',
  {
    KNOWS: {
      properties: 'weight'
    }
  }
)
```

### Cypher 프로젝션

```cypher
CALL gds.graph.project.cypher(
  'customGraph',
  'MATCH (n:Person) WHERE n.active = true RETURN id(n) AS id',
  'MATCH (a:Person)-[r:KNOWS]->(b:Person) RETURN id(a) AS source, id(b) AS target, r.weight AS weight'
)
```

### 그래프 관리

```cypher
// 그래프 목록
CALL gds.graph.list()

// 그래프 정보
CALL gds.graph.list('myGraph')
YIELD graphName, nodeCount, relationshipCount

// 그래프 삭제
CALL gds.graph.drop('myGraph')
```

---

## 중심성 알고리즘

### PageRank

```cypher
// 프로젝션
CALL gds.graph.project('prGraph', 'Page', 'LINKS')

// 실행 (stream)
CALL gds.pageRank.stream('prGraph')
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).name AS name, score
ORDER BY score DESC
LIMIT 10

// 결과를 노드에 저장 (write)
CALL gds.pageRank.write('prGraph', {
  writeProperty: 'pagerank'
})
YIELD nodePropertiesWritten
```

### Betweenness Centrality

```cypher
CALL gds.betweenness.stream('myGraph')
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).name AS name, score
ORDER BY score DESC
LIMIT 10
```

### Degree Centrality

```cypher
CALL gds.degree.stream('myGraph')
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).name AS name, score AS connections
ORDER BY score DESC
```

---

## 커뮤니티 탐지

### Louvain

```cypher
CALL gds.louvain.stream('myGraph')
YIELD nodeId, communityId
RETURN communityId, count(*) AS size, collect(gds.util.asNode(nodeId).name) AS members
ORDER BY size DESC
```

### Label Propagation

```cypher
CALL gds.labelPropagation.stream('myGraph')
YIELD nodeId, communityId
RETURN communityId, count(*) AS size
ORDER BY size DESC
```

### Weakly Connected Components

```cypher
CALL gds.wcc.stream('myGraph')
YIELD nodeId, componentId
RETURN componentId, count(*) AS size
ORDER BY size DESC
```

### Triangle Count

```cypher
CALL gds.triangleCount.stream('myGraph')
YIELD nodeId, triangleCount
WHERE triangleCount > 0
RETURN gds.util.asNode(nodeId).name AS name, triangleCount
ORDER BY triangleCount DESC
```

---

## 유사도 알고리즘

### Node Similarity

```cypher
CALL gds.nodeSimilarity.stream('myGraph')
YIELD node1, node2, similarity
RETURN gds.util.asNode(node1).name AS person1,
       gds.util.asNode(node2).name AS person2,
       similarity
ORDER BY similarity DESC
LIMIT 10
```

### K-Nearest Neighbors (KNN)

```cypher
CALL gds.knn.stream('myGraph', {
  nodeProperties: ['age', 'income'],
  topK: 3
})
YIELD node1, node2, similarity
RETURN gds.util.asNode(node1).name AS person,
       gds.util.asNode(node2).name AS neighbor,
       similarity
```

---

## 경로 찾기

### Shortest Path

```cypher
MATCH (source:Person {name: 'Alice'}), (target:Person {name: 'Bob'})
CALL gds.shortestPath.dijkstra.stream('myGraph', {
  sourceNode: source,
  targetNode: target,
  relationshipWeightProperty: 'cost'
})
YIELD path, totalCost
RETURN path, totalCost
```

### All Shortest Paths

```cypher
CALL gds.allShortestPaths.stream('myGraph')
YIELD sourceNodeId, targetNodeId, distance
RETURN gds.util.asNode(sourceNodeId).name AS source,
       gds.util.asNode(targetNodeId).name AS target,
       distance
```

---

## 노드 임베딩

### FastRP

```cypher
// 임베딩 생성
CALL gds.fastRP.stream('myGraph', {
  embeddingDimension: 128,
  iterationWeights: [0.0, 1.0, 1.0]
})
YIELD nodeId, embedding
RETURN gds.util.asNode(nodeId).name AS name, embedding
LIMIT 5

// 노드에 저장
CALL gds.fastRP.write('myGraph', {
  embeddingDimension: 128,
  writeProperty: 'embedding'
})
```

### Node2Vec

```cypher
CALL gds.node2vec.stream('myGraph', {
  embeddingDimension: 64,
  walkLength: 80,
  walksPerNode: 10
})
YIELD nodeId, embedding
RETURN gds.util.asNode(nodeId).name, embedding
```

---

## 머신러닝 파이프라인

### Link Prediction

```cypher
// 파이프라인 생성
CALL gds.beta.pipeline.linkPrediction.create('lp-pipeline')

// 특성 추가
CALL gds.beta.pipeline.linkPrediction.addFeature('lp-pipeline', 'hadamard', {
  nodeProperties: ['embedding']
})

// 모델 학습
CALL gds.beta.pipeline.linkPrediction.train('myGraph', {
  pipeline: 'lp-pipeline',
  modelName: 'lp-model',
  targetRelationshipType: 'KNOWS',
  metrics: ['AUCPR']
})

// 예측
CALL gds.beta.pipeline.linkPrediction.predict.stream('myGraph', {
  modelName: 'lp-model',
  topN: 100
})
YIELD node1, node2, probability
RETURN gds.util.asNode(node1).name,
       gds.util.asNode(node2).name,
       probability
```

### Node Classification

```cypher
// 파이프라인 생성
CALL gds.beta.pipeline.nodeClassification.create('nc-pipeline')

// 특성 추가
CALL gds.beta.pipeline.nodeClassification.addNodeProperty('nc-pipeline', 'fastRP', {
  embeddingDimension: 64
})

// 학습
CALL gds.beta.pipeline.nodeClassification.train('myGraph', {
  pipeline: 'nc-pipeline',
  modelName: 'nc-model',
  targetProperty: 'label',
  metrics: ['F1_WEIGHTED']
})
```

---

## 실행 모드

| 모드 | 설명 | 용도 |
|------|------|------|
| stream | 결과를 스트림으로 반환 | 탐색, 분석 |
| stats | 통계만 반환 | 요약 확인 |
| mutate | 인메모리 그래프에 저장 | 파이프라인 |
| write | 데이터베이스에 저장 | 영구 저장 |

```cypher
// Stream
CALL gds.pageRank.stream('g') YIELD nodeId, score RETURN *

// Stats
CALL gds.pageRank.stats('g') YIELD centralityDistribution RETURN *

// Mutate (인메모리)
CALL gds.pageRank.mutate('g', {mutateProperty: 'pr'})

// Write (DB)
CALL gds.pageRank.write('g', {writeProperty: 'pr'})
```

---

## 다음 단계

!!! success "GDS 완료!"
    [성능 튜닝](performance.md)에서 최적화 방법을 배워보세요.
