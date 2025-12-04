# 실시간 추천 엔진 구현 가이드

## 프로젝트 개요

Neo4j 기반 실시간 추천 엔진을 구축하는 완전한 가이드입니다.

### 구현 목표
- 협업 필터링 추천
- 콘텐츠 기반 추천
- 하이브리드 추천
- 실시간 개인화
- A/B 테스트 지원

---

## 1단계: 데이터 모델

### E-Commerce 추천 스키마

```cypher
// 제약조건
CREATE CONSTRAINT product_id FOR (p:Product) REQUIRE p.id IS UNIQUE;
CREATE CONSTRAINT user_id FOR (u:User) REQUIRE u.id IS UNIQUE;
CREATE CONSTRAINT category_name FOR (c:Category) REQUIRE c.name IS UNIQUE;

// 인덱스
CREATE INDEX product_name FOR (p:Product) ON (p.name);
CREATE INDEX product_price FOR (p:Product) ON (p.price);

// 노드 구조
(:User {
  id: STRING,
  age: INTEGER,
  gender: STRING,
  location: STRING,
  preferences: [STRING],
  createdAt: DATETIME
})

(:Product {
  id: STRING,
  name: STRING,
  description: STRING,
  price: FLOAT,
  brand: STRING,
  imageUrl: STRING,
  rating: FLOAT,
  reviewCount: INTEGER,
  stock: INTEGER
})

(:Category {
  name: STRING,
  level: INTEGER
})

(:Session {
  id: STRING,
  startedAt: DATETIME,
  device: STRING,
  source: STRING
})

// 관계
(:User)-[:VIEWED {at: DATETIME, duration: INTEGER}]->(:Product)
(:User)-[:PURCHASED {at: DATETIME, quantity: INTEGER, price: FLOAT}]->(:Product)
(:User)-[:RATED {score: FLOAT, at: DATETIME}]->(:Product)
(:User)-[:ADDED_TO_CART {at: DATETIME}]->(:Product)
(:User)-[:WISHLISTED {at: DATETIME}]->(:Product)
(:Product)-[:IN_CATEGORY]->(:Category)
(:Category)-[:SUBCATEGORY_OF]->(:Category)
(:Product)-[:SIMILAR_TO {score: FLOAT}]->(:Product)
(:User)-[:HAS_SESSION]->(:Session)
(:Session)-[:CLICKED]->(:Product)
```

---

## 2단계: 추천 알고리즘 구현

### 협업 필터링 서비스

```python
# services/collaborative_filtering.py
from typing import List, Dict
from datetime import datetime, timedelta

class CollaborativeFilteringService:
    def __init__(self, db):
        self.db = db

    def user_based_recommendations(self, user_id: str,
                                   limit: int = 10) -> List[Dict]:
        """사용자 기반 협업 필터링"""
        query = """
        // 1. 타겟 사용자의 구매/평가 이력
        MATCH (me:User {id: $userId})-[r:PURCHASED|RATED]->(p:Product)
        WITH me, collect(DISTINCT p) AS myProducts

        // 2. 비슷한 취향의 사용자 찾기
        MATCH (me)-[r1:PURCHASED|RATED]->(common:Product)<-[r2:PURCHASED|RATED]-(other:User)
        WHERE other <> me

        // 3. 유사도 계산 (Jaccard)
        WITH me, myProducts, other,
             count(DISTINCT common) AS intersection

        MATCH (other)-[:PURCHASED|RATED]->(otherProducts:Product)
        WITH me, myProducts, other, intersection,
             count(DISTINCT otherProducts) AS otherCount

        WITH me, myProducts, other,
             toFloat(intersection) / (size(myProducts) + otherCount - intersection) AS similarity
        WHERE similarity > 0.1
        ORDER BY similarity DESC
        LIMIT 20

        // 4. 비슷한 사용자가 좋아한 상품 추천
        MATCH (other)-[r:PURCHASED|RATED]->(rec:Product)
        WHERE NOT rec IN myProducts
          AND (r:RATED AND r.score >= 4 OR r:PURCHASED)

        WITH rec, sum(similarity * coalesce(r.score, 5)) AS score,
             count(DISTINCT other) AS supporters

        RETURN rec {
            .id, .name, .price, .imageUrl, .rating, .brand
        } AS product,
        score,
        supporters
        ORDER BY score DESC
        LIMIT $limit
        """
        with self.db.session() as session:
            result = session.run(query, userId=user_id, limit=limit)
            return [dict(r) for r in result]

    def item_based_recommendations(self, product_id: str,
                                   limit: int = 10) -> List[Dict]:
        """아이템 기반 협업 필터링"""
        query = """
        // 1. 이 상품을 구매/평가한 사용자
        MATCH (target:Product {id: $productId})<-[r1:PURCHASED|RATED]-(u:User)

        // 2. 같은 사용자가 구매/평가한 다른 상품
        MATCH (u)-[r2:PURCHASED|RATED]->(other:Product)
        WHERE other <> target

        // 3. 동시 구매 빈도로 유사도 계산
        WITH target, other,
             count(DISTINCT u) AS cooccurrence

        // 타겟 상품 구매자 수
        MATCH (target)<-[:PURCHASED|RATED]-(buyer)
        WITH target, other, cooccurrence,
             count(DISTINCT buyer) AS targetBuyers

        // 다른 상품 구매자 수
        MATCH (other)<-[:PURCHASED|RATED]-(otherBuyer)
        WITH other, cooccurrence, targetBuyers,
             count(DISTINCT otherBuyer) AS otherBuyers,
             toFloat(cooccurrence) / sqrt(targetBuyers * count(DISTINCT otherBuyer)) AS similarity

        WHERE similarity > 0.05

        RETURN other {
            .id, .name, .price, .imageUrl, .rating, .brand
        } AS product,
        similarity,
        cooccurrence
        ORDER BY similarity DESC
        LIMIT $limit
        """
        with self.db.session() as session:
            result = session.run(query, productId=product_id, limit=limit)
            return [dict(r) for r in result]

    def frequent_together(self, product_id: str,
                         limit: int = 5) -> List[Dict]:
        """함께 구매한 상품"""
        query = """
        MATCH (p:Product {id: $productId})<-[:PURCHASED]-(u:User)-[:PURCHASED]->(other:Product)
        WHERE p <> other
        WITH other, count(DISTINCT u) AS frequency

        // 같은 카테고리 우선
        OPTIONAL MATCH (p)-[:IN_CATEGORY]->(c:Category)<-[:IN_CATEGORY]-(other)
        WITH other, frequency, count(c) AS categoryMatch

        RETURN other {
            .id, .name, .price, .imageUrl, .rating
        } AS product,
        frequency,
        categoryMatch,
        frequency * (1 + categoryMatch * 0.5) AS score
        ORDER BY score DESC
        LIMIT $limit
        """
        with self.db.session() as session:
            result = session.run(query, productId=product_id, limit=limit)
            return [dict(r) for r in result]
```

### 콘텐츠 기반 추천 서비스

```python
# services/content_based.py
from typing import List, Dict

class ContentBasedService:
    def __init__(self, db):
        self.db = db

    def category_based(self, user_id: str, limit: int = 10) -> List[Dict]:
        """카테고리 기반 추천"""
        query = """
        // 사용자가 선호하는 카테고리
        MATCH (u:User {id: $userId})-[r:PURCHASED|VIEWED|RATED]->(p:Product)-[:IN_CATEGORY]->(c:Category)
        WITH u, c,
             sum(CASE
                 WHEN r:PURCHASED THEN 10
                 WHEN r:RATED THEN r.score * 2
                 ELSE 1 END) AS categoryScore

        ORDER BY categoryScore DESC
        LIMIT 5

        // 선호 카테고리의 인기 상품
        MATCH (rec:Product)-[:IN_CATEGORY]->(c)
        WHERE NOT (u)-[:PURCHASED]->(rec)
          AND rec.stock > 0

        WITH rec, sum(categoryScore) AS relevance,
             rec.rating * rec.reviewCount AS popularity

        RETURN rec {
            .id, .name, .price, .imageUrl, .rating, .brand
        } AS product,
        relevance,
        popularity,
        relevance + popularity * 0.1 AS score
        ORDER BY score DESC
        LIMIT $limit
        """
        with self.db.session() as session:
            result = session.run(query, userId=user_id, limit=limit)
            return [dict(r) for r in result]

    def similar_products(self, product_id: str, limit: int = 10) -> List[Dict]:
        """유사 상품 추천 (속성 기반)"""
        query = """
        MATCH (p:Product {id: $productId})-[:IN_CATEGORY]->(c:Category)
        MATCH (similar:Product)-[:IN_CATEGORY]->(c)
        WHERE similar <> p AND similar.stock > 0

        // 가격대 유사도
        WITH p, similar, c,
             1 - abs(p.price - similar.price) / (p.price + similar.price) AS priceSimilarity

        // 브랜드 보너스
        WITH similar, priceSimilarity,
             CASE WHEN p.brand = similar.brand THEN 0.2 ELSE 0 END AS brandBonus

        // 평점 가중치
        WITH similar,
             (priceSimilarity + brandBonus) * similar.rating AS score

        RETURN similar {
            .id, .name, .price, .imageUrl, .rating, .brand
        } AS product,
        score
        ORDER BY score DESC
        LIMIT $limit
        """
        with self.db.session() as session:
            result = session.run(query, productId=product_id, limit=limit)
            return [dict(r) for r in result]

    def brand_affinity(self, user_id: str, limit: int = 10) -> List[Dict]:
        """브랜드 선호도 기반 추천"""
        query = """
        // 사용자의 브랜드 선호도 계산
        MATCH (u:User {id: $userId})-[r:PURCHASED|RATED]->(p:Product)
        WHERE p.brand IS NOT NULL
        WITH u, p.brand AS brand,
             sum(CASE
                 WHEN r:PURCHASED THEN 5
                 WHEN r:RATED THEN r.score
                 ELSE 1 END) AS brandScore
        ORDER BY brandScore DESC
        LIMIT 3

        // 선호 브랜드의 신상품/인기 상품
        MATCH (rec:Product)
        WHERE rec.brand = brand
          AND NOT (u)-[:PURCHASED]->(rec)
          AND rec.stock > 0

        RETURN rec {
            .id, .name, .price, .imageUrl, .rating, .brand
        } AS product,
        brandScore,
        rec.rating AS rating
        ORDER BY brandScore DESC, rec.rating DESC
        LIMIT $limit
        """
        with self.db.session() as session:
            result = session.run(query, userId=user_id, limit=limit)
            return [dict(r) for r in result]
```

### 하이브리드 추천 서비스

```python
# services/hybrid_recommendation.py
from typing import List, Dict
from services.collaborative_filtering import CollaborativeFilteringService
from services.content_based import ContentBasedService

class HybridRecommendationService:
    def __init__(self, db):
        self.db = db
        self.cf = CollaborativeFilteringService(db)
        self.cb = ContentBasedService(db)

    def personalized_recommendations(self, user_id: str,
                                     limit: int = 20) -> List[Dict]:
        """개인화 하이브리드 추천"""
        query = """
        // 사용자 활동 수준 확인
        MATCH (u:User {id: $userId})
        OPTIONAL MATCH (u)-[:PURCHASED]->(purchased)
        OPTIONAL MATCH (u)-[:VIEWED]->(viewed)
        WITH u,
             count(DISTINCT purchased) AS purchaseCount,
             count(DISTINCT viewed) AS viewCount

        // Cold Start 여부 판단
        WITH u, purchaseCount, viewCount,
             CASE
                 WHEN purchaseCount >= 5 THEN 'active'
                 WHEN purchaseCount >= 1 OR viewCount >= 10 THEN 'moderate'
                 ELSE 'cold'
             END AS userType

        // Active: 협업 필터링 가중치 높음
        CALL {
            WITH u, userType
            WHERE userType = 'active'

            // 협업 필터링 (70%)
            MATCH (u)-[r1:PURCHASED|RATED]->(p1:Product)<-[r2:PURCHASED|RATED]-(other:User)
            WHERE other <> u
            WITH u, other, count(DISTINCT p1) AS common
            ORDER BY common DESC LIMIT 10

            MATCH (other)-[r:PURCHASED|RATED]->(rec:Product)
            WHERE NOT (u)-[:PURCHASED]->(rec) AND r.score >= 4
            WITH rec, count(*) * 0.7 AS cfScore

            // 콘텐츠 기반 (30%)
            MATCH (u)-[:PURCHASED]->(bought)-[:IN_CATEGORY]->(c)<-[:IN_CATEGORY]-(rec)
            WITH rec, cfScore, count(c) * 0.3 AS cbScore

            RETURN rec, cfScore + cbScore AS score, 'hybrid-active' AS method

            UNION

            // Moderate: 균형
            WITH u, userType
            WHERE userType = 'moderate'

            MATCH (u)-[:VIEWED|PURCHASED]->(p)-[:IN_CATEGORY]->(c)<-[:IN_CATEGORY]-(rec:Product)
            WHERE NOT (u)-[:PURCHASED]->(rec)
            WITH rec, count(*) * 0.5 AS score

            RETURN rec, score, 'content-moderate' AS method

            UNION

            // Cold Start: 인기 상품
            WITH u, userType
            WHERE userType = 'cold'

            MATCH (rec:Product)
            WHERE rec.rating >= 4 AND rec.reviewCount >= 10
            WITH rec, rec.rating * log(rec.reviewCount + 1) AS score

            RETURN rec, score, 'popularity-cold' AS method
        }

        RETURN rec {
            .id, .name, .price, .imageUrl, .rating, .brand
        } AS product,
        score,
        method
        ORDER BY score DESC
        LIMIT $limit
        """
        with self.db.session() as session:
            result = session.run(query, userId=user_id, limit=limit)
            return [dict(r) for r in result]

    def contextual_recommendations(self, user_id: str, context: Dict,
                                   limit: int = 10) -> List[Dict]:
        """컨텍스트 기반 추천"""
        query = """
        MATCH (u:User {id: $userId})

        // 시간대별 가중치
        WITH u,
             CASE
                 WHEN $hour >= 6 AND $hour < 12 THEN 'morning'
                 WHEN $hour >= 12 AND $hour < 18 THEN 'afternoon'
                 WHEN $hour >= 18 AND $hour < 22 THEN 'evening'
                 ELSE 'night'
             END AS timeOfDay

        // 같은 시간대에 많이 구매한 카테고리
        MATCH (u)-[r:PURCHASED]->(p:Product)-[:IN_CATEGORY]->(c:Category)
        WHERE datetime(r.at).hour >= $hour - 2 AND datetime(r.at).hour <= $hour + 2
        WITH u, timeOfDay, c, count(*) AS timeBasedScore
        ORDER BY timeBasedScore DESC
        LIMIT 3

        // 해당 카테고리의 추천
        MATCH (rec:Product)-[:IN_CATEGORY]->(c)
        WHERE NOT (u)-[:PURCHASED]->(rec)
          AND rec.stock > 0
          AND ($minPrice IS NULL OR rec.price >= $minPrice)
          AND ($maxPrice IS NULL OR rec.price <= $maxPrice)

        RETURN rec {
            .id, .name, .price, .imageUrl, .rating
        } AS product,
        timeBasedScore AS score,
        timeOfDay AS context
        ORDER BY score DESC, rec.rating DESC
        LIMIT $limit
        """
        with self.db.session() as session:
            from datetime import datetime
            result = session.run(query,
                userId=user_id,
                hour=context.get('hour', datetime.now().hour),
                minPrice=context.get('minPrice'),
                maxPrice=context.get('maxPrice'),
                limit=limit
            )
            return [dict(r) for r in result]

    def real_time_recommendations(self, user_id: str, session_id: str,
                                  limit: int = 5) -> List[Dict]:
        """실시간 세션 기반 추천"""
        query = """
        // 현재 세션에서 본 상품
        MATCH (u:User {id: $userId})-[:HAS_SESSION]->(s:Session {id: $sessionId})
        MATCH (s)-[:CLICKED]->(viewed:Product)
        WITH collect(viewed) AS sessionViewed

        // 세션에서 본 상품과 유사한 상품
        UNWIND sessionViewed AS v
        MATCH (v)-[:IN_CATEGORY]->(c:Category)<-[:IN_CATEGORY]-(rec:Product)
        WHERE NOT rec IN sessionViewed

        // 가격대 유사
        WITH rec, v,
             abs(rec.price - v.price) / v.price AS priceDiff
        WHERE priceDiff < 0.3

        WITH rec, count(DISTINCT v) AS relevance, avg(priceDiff) AS avgPriceDiff

        RETURN rec {
            .id, .name, .price, .imageUrl, .rating
        } AS product,
        relevance,
        1 - avgPriceDiff AS priceSimilarity,
        relevance * (1 - avgPriceDiff) * rec.rating AS score
        ORDER BY score DESC
        LIMIT $limit
        """
        with self.db.session() as session:
            result = session.run(query,
                userId=user_id, sessionId=session_id, limit=limit)
            return [dict(r) for r in result]
```

---

## 3단계: FastAPI 엔드포인트

```python
# main.py
from fastapi import FastAPI, Query
from services.hybrid_recommendation import HybridRecommendationService

app = FastAPI(title="Recommendation Engine API")
recommender = HybridRecommendationService(db)

@app.get("/recommendations/{user_id}")
def get_recommendations(
    user_id: str,
    limit: int = Query(20, ge=1, le=100),
    context_hour: int = Query(None, ge=0, le=23),
    min_price: float = Query(None, ge=0),
    max_price: float = Query(None, ge=0)
):
    """개인화 추천 조회"""
    context = {}
    if context_hour is not None:
        context['hour'] = context_hour
    if min_price is not None:
        context['minPrice'] = min_price
    if max_price is not None:
        context['maxPrice'] = max_price

    if context:
        return recommender.contextual_recommendations(user_id, context, limit)
    return recommender.personalized_recommendations(user_id, limit)

@app.get("/recommendations/{user_id}/session/{session_id}")
def get_session_recommendations(user_id: str, session_id: str, limit: int = 5):
    """실시간 세션 기반 추천"""
    return recommender.real_time_recommendations(user_id, session_id, limit)

@app.get("/products/{product_id}/similar")
def get_similar_products(product_id: str, limit: int = 10):
    """유사 상품 추천"""
    return recommender.cb.similar_products(product_id, limit)

@app.get("/products/{product_id}/bought-together")
def get_bought_together(product_id: str, limit: int = 5):
    """함께 구매한 상품"""
    return recommender.cf.frequent_together(product_id, limit)
```

---

## 4단계: 성능 최적화

### 유사도 사전 계산

```cypher
// 상품 간 유사도 사전 계산 (배치)
CALL apoc.periodic.iterate(
  "MATCH (p:Product) RETURN p",
  "
  MATCH (p)<-[:PURCHASED]-(u:User)-[:PURCHASED]->(other:Product)
  WHERE p <> other
  WITH p, other, count(DISTINCT u) AS cooccurrence
  WHERE cooccurrence >= 3

  MATCH (p)<-[:PURCHASED]-(pb)
  MATCH (other)<-[:PURCHASED]-(ob)
  WITH p, other, cooccurrence,
       count(DISTINCT pb) AS pCount, count(DISTINCT ob) AS oCount

  WITH p, other, toFloat(cooccurrence) / sqrt(pCount * oCount) AS similarity
  WHERE similarity > 0.1

  MERGE (p)-[r:SIMILAR_TO]->(other)
  SET r.score = similarity, r.updatedAt = datetime()
  ",
  {batchSize: 100, parallel: true}
)
```

### 캐싱 레이어

```python
# services/cache.py
import redis
import json
from functools import wraps

redis_client = redis.Redis(host='localhost', port=6379, db=0)

def cached_recommendation(ttl_seconds=300):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            cache_key = f"rec:{func.__name__}:{args}:{kwargs}"
            cached = redis_client.get(cache_key)
            if cached:
                return json.loads(cached)

            result = func(*args, **kwargs)
            redis_client.setex(cache_key, ttl_seconds, json.dumps(result))
            return result
        return wrapper
    return decorator

# 사용
@cached_recommendation(ttl_seconds=60)
def get_trending_products(limit: int = 10):
    # ...
```

---

## 다음 단계

[사기 탐지 시스템](fraud-detection.md)에서 금융 분야 활용 사례를 확인하세요.
