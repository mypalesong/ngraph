# 지식 그래프 완성

## 개요

지식 그래프 완성(KG Completion)은 누락된 링크를 예측하여 그래프를 보완하는 기법입니다.

## 링크 예측

### 임베딩 기반

```python
from pykeen.pipeline import pipeline

# TransE 모델로 링크 예측
result = pipeline(
    model='TransE',
    dataset='FB15k-237',
)

# 누락된 링크 예측
predictions = result.model.predict_scores_all_tails(
    head='Q42',  # 더글러스 애덤스
    relation='nationality'
)
```

### 규칙 기반

```cypher
// 추이적 관계로 누락 링크 추론
MATCH (a)-[:PART_OF]->(b)-[:PART_OF]->(c)
WHERE NOT (a)-[:PART_OF]->(c)
MERGE (a)-[:PART_OF]->(c)
```

## 평가 지표

| 지표 | 설명 |
|------|------|
| MRR | Mean Reciprocal Rank |
| Hits@K | 상위 K개 내 정답 비율 |
| MR | Mean Rank |

## 실습

```python
from pykeen.evaluation import RankBasedEvaluator

evaluator = RankBasedEvaluator()
metrics = evaluator.evaluate(
    model=result.model,
    mapped_triples=result.training.mapped_triples,
)
print(f"MRR: {metrics.get_metric('mean_reciprocal_rank')}")
```

## 다음 단계

- [LLM 통합](llm-integration.md)에서 AI 연동을 학습하세요
