# 추천 시스템

## 개요

지식 그래프 기반 추천 시스템은 사용자-아이템 관계를 그래프로 모델링하여 정교한 추천을 제공합니다.

## 그래프 기반 추천의 장점

| 기존 추천 | 그래프 기반 추천 |
|----------|----------------|
| Cold Start 문제 | 관계 기반 해결 |
| 블랙박스 | 설명 가능한 추천 |
| 단일 신호 | 다중 관계 활용 |

## 스키마 설계

```cypher
// 사용자
CREATE (u:User {id: 'user1', name: '홍길동'})

// 아이템
CREATE (m:Movie {id: 'movie1', title: '인셉션', year: 2010})
CREATE (g:Genre {name: 'SF'})
CREATE (d:Director {name: '크리스토퍼 놀란'})

// 관계
CREATE (u)-[:WATCHED {rating: 5, date: date()}]->(m)
CREATE (m)-[:HAS_GENRE]->(g)
CREATE (m)-[:DIRECTED_BY]->(d)
```

## 추천 알고리즘

### 협업 필터링

```cypher
// 비슷한 취향의 사용자가 본 영화 추천
MATCH (me:User {id: $userId})-[:WATCHED]->(m:Movie)<-[:WATCHED]-(other:User)
MATCH (other)-[:WATCHED]->(rec:Movie)
WHERE NOT (me)-[:WATCHED]->(rec)
WITH rec, count(DISTINCT other) AS score
ORDER BY score DESC
RETURN rec.title, score
LIMIT 10
```

### 콘텐츠 기반

```cypher
// 좋아하는 장르의 다른 영화 추천
MATCH (me:User {id: $userId})-[w:WATCHED]->(m:Movie)-[:HAS_GENRE]->(g:Genre)
WHERE w.rating >= 4
MATCH (rec:Movie)-[:HAS_GENRE]->(g)
WHERE NOT (me)-[:WATCHED]->(rec)
WITH rec, count(g) AS genreMatch
ORDER BY genreMatch DESC
RETURN rec.title, genreMatch
LIMIT 10
```

### 하이브리드

```cypher
// 협업 + 콘텐츠 결합
MATCH (me:User {id: $userId})
CALL {
    // 협업 필터링 점수
    MATCH (me)-[:WATCHED]->(m)<-[:WATCHED]-(other)-[:WATCHED]->(rec)
    WHERE NOT (me)-[:WATCHED]->(rec)
    RETURN rec, count(*) * 0.6 AS score
    UNION
    // 콘텐츠 기반 점수
    MATCH (me)-[:WATCHED]->(m)-[:HAS_GENRE]->(g)<-[:HAS_GENRE]-(rec)
    WHERE NOT (me)-[:WATCHED]->(rec)
    RETURN rec, count(*) * 0.4 AS score
}
RETURN rec.title, sum(score) AS finalScore
ORDER BY finalScore DESC
LIMIT 10
```

## 설명 가능한 추천

```cypher
// 추천 이유 제공
MATCH (me:User {id: $userId})-[:WATCHED]->(liked:Movie)-[r*1..2]-(rec:Movie)
WHERE NOT (me)-[:WATCHED]->(rec)
WITH rec, liked, r
RETURN rec.title AS recommendation,
       liked.title AS because,
       [rel in r | type(rel)] AS path
LIMIT 5
```

## Python 구현

```python
class GraphRecommender:
    def __init__(self, driver):
        self.driver = driver

    def recommend_movies(self, user_id: str, limit: int = 10):
        query = """
        MATCH (me:User {id: $userId})-[w:WATCHED]->(m:Movie)
        WHERE w.rating >= 4
        MATCH (m)-[:HAS_GENRE]->(g:Genre)<-[:HAS_GENRE]-(rec:Movie)
        WHERE NOT (me)-[:WATCHED]->(rec)
        WITH rec, collect(DISTINCT g.name) AS genres, count(*) AS score
        ORDER BY score DESC
        LIMIT $limit
        RETURN rec.title AS title, genres, score
        """
        with self.driver.session() as session:
            return list(session.run(query, userId=user_id, limit=limit))
```

## 다음 단계

- [NLP 통합](nlp-integration.md)으로 리뷰 분석 추가
- [그래프 임베딩](../advanced/graph-embedding.md)으로 정확도 향상
