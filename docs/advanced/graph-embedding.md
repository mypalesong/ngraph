# 그래프 임베딩

## 개요

그래프 임베딩은 노드, 엣지, 그래프를 저차원 벡터 공간에 표현하는 기법입니다.

## 주요 알고리즘

### Node2Vec

```python
from node2vec import Node2Vec
import networkx as nx

# 그래프 생성
G = nx.Graph()
G.add_edges_from([(1, 2), (2, 3), (3, 4)])

# Node2Vec 임베딩
node2vec = Node2Vec(G, dimensions=64, walk_length=30, num_walks=200)
model = node2vec.fit(window=10, min_count=1)

# 노드 벡터 조회
vector = model.wv['1']

# 유사 노드 찾기
similar = model.wv.most_similar('1')
```

### TransE (지식 그래프 임베딩)

```python
from pykeen.pipeline import pipeline

result = pipeline(
    model='TransE',
    dataset='FB15k-237',
    training_kwargs=dict(num_epochs=100),
)

# 엔티티 임베딩
entity_embeddings = result.model.entity_representations[0]
```

## Neo4j GDS

```cypher
// FastRP 임베딩
CALL gds.fastRP.stream('myGraph', {
    embeddingDimension: 128,
    iterationWeights: [0.0, 1.0, 1.0]
})
YIELD nodeId, embedding
RETURN gds.util.asNode(nodeId).name, embedding
```

## 활용

- 노드 분류
- 링크 예측
- 유사도 검색
- 클러스터링

## 다음 단계

- [지식 그래프 완성](kg-completion.md)에서 링크 예측을 학습하세요
