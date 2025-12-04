# 소셜 네트워크 분석 구현 가이드

## 프로젝트 개요

소셜 네트워크 분석 시스템을 Neo4j로 구축하는 완전한 가이드입니다.

### 구현 목표
- 사용자 관계 그래프 구축
- 친구 추천 알고리즘
- 인플루언서 탐지
- 커뮤니티 분석
- 바이럴 확산 시뮬레이션

---

## 1단계: 환경 설정

### Docker Compose 구성

```yaml
# docker-compose.yml
version: '3.8'

services:
  neo4j:
    image: neo4j:5.15.0
    container_name: social-network-db
    ports:
      - "7474:7474"
      - "7687:7687"
    environment:
      - NEO4J_AUTH=neo4j/socialnet123
      - NEO4J_PLUGINS=["apoc", "graph-data-science"]
      - NEO4J_dbms_memory_heap_max__size=2G
      - NEO4J_dbms_memory_pagecache_size=1G
    volumes:
      - neo4j_data:/data
      - neo4j_logs:/logs
      - ./import:/var/lib/neo4j/import

  api:
    build: ./api
    ports:
      - "8000:8000"
    depends_on:
      - neo4j
    environment:
      - NEO4J_URI=bolt://neo4j:7687
      - NEO4J_USER=neo4j
      - NEO4J_PASSWORD=socialnet123

volumes:
  neo4j_data:
  neo4j_logs:
```

### 프로젝트 구조

```
social-network/
├── docker-compose.yml
├── api/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── main.py
│   ├── models/
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── relationship.py
│   ├── services/
│   │   ├── __init__.py
│   │   ├── user_service.py
│   │   ├── recommendation_service.py
│   │   └── analytics_service.py
│   └── routers/
│       ├── __init__.py
│       ├── users.py
│       └── analytics.py
├── import/
│   └── sample_data.csv
└── tests/
    └── test_recommendations.py
```

---

## 2단계: 데이터 모델 설계

### 스키마 정의

```cypher
// 제약조건 및 인덱스 생성
CREATE CONSTRAINT user_id IF NOT EXISTS FOR (u:User) REQUIRE u.id IS UNIQUE;
CREATE CONSTRAINT user_email IF NOT EXISTS FOR (u:User) REQUIRE u.email IS UNIQUE;
CREATE INDEX user_name IF NOT EXISTS FOR (u:User) ON (u.name);
CREATE INDEX user_location IF NOT EXISTS FOR (u:User) ON (u.location);
CREATE INDEX post_created IF NOT EXISTS FOR (p:Post) ON (p.createdAt);

// 전문 검색 인덱스
CREATE FULLTEXT INDEX user_search IF NOT EXISTS
FOR (u:User) ON EACH [u.name, u.bio, u.interests];
```

### 노드 타입

```cypher
// User 노드
(:User {
  id: STRING,           // UUID
  email: STRING,        // 고유
  name: STRING,
  bio: STRING,
  avatar: STRING,
  location: STRING,
  interests: [STRING],
  joinedAt: DATETIME,
  lastActiveAt: DATETIME,
  isVerified: BOOLEAN,
  followerCount: INTEGER,
  followingCount: INTEGER
})

// Post 노드
(:Post {
  id: STRING,
  content: STRING,
  mediaUrls: [STRING],
  createdAt: DATETIME,
  likeCount: INTEGER,
  commentCount: INTEGER,
  shareCount: INTEGER,
  visibility: STRING    // public, friends, private
})

// Comment 노드
(:Comment {
  id: STRING,
  content: STRING,
  createdAt: DATETIME
})

// Hashtag 노드
(:Hashtag {
  name: STRING,
  postCount: INTEGER
})
```

### 관계 타입

```cypher
// 팔로우 관계
(:User)-[:FOLLOWS {since: DATETIME}]->(:User)

// 게시물 관계
(:User)-[:POSTED {at: DATETIME}]->(:Post)
(:User)-[:LIKED {at: DATETIME}]->(:Post)
(:User)-[:SHARED {at: DATETIME}]->(:Post)
(:User)-[:COMMENTED]->(:Comment)-[:ON]->(:Post)

// 해시태그
(:Post)-[:TAGGED]->(:Hashtag)

// 멘션
(:Post)-[:MENTIONS]->(:User)
```

---

## 3단계: Python API 구현

### requirements.txt

```
fastapi==0.109.0
uvicorn==0.27.0
neo4j==5.15.0
pydantic==2.5.0
python-jose==3.3.0
passlib==1.7.4
python-multipart==0.0.6
```

### 데이터베이스 연결 (database.py)

```python
# api/database.py
from neo4j import GraphDatabase
from contextlib import contextmanager
import os

class Neo4jConnection:
    def __init__(self):
        self.driver = GraphDatabase.driver(
            os.getenv("NEO4J_URI", "bolt://localhost:7687"),
            auth=(
                os.getenv("NEO4J_USER", "neo4j"),
                os.getenv("NEO4J_PASSWORD", "password")
            )
        )

    def close(self):
        self.driver.close()

    @contextmanager
    def session(self):
        session = self.driver.session()
        try:
            yield session
        finally:
            session.close()

db = Neo4jConnection()
```

### 사용자 서비스 (user_service.py)

```python
# api/services/user_service.py
from typing import List, Optional
from datetime import datetime
import uuid

class UserService:
    def __init__(self, db):
        self.db = db

    def create_user(self, email: str, name: str, bio: str = "",
                   interests: List[str] = None) -> dict:
        """새 사용자 생성"""
        query = """
        CREATE (u:User {
            id: $id,
            email: $email,
            name: $name,
            bio: $bio,
            interests: $interests,
            joinedAt: datetime(),
            lastActiveAt: datetime(),
            isVerified: false,
            followerCount: 0,
            followingCount: 0
        })
        RETURN u
        """
        with self.db.session() as session:
            result = session.run(query,
                id=str(uuid.uuid4()),
                email=email,
                name=name,
                bio=bio,
                interests=interests or []
            )
            return dict(result.single()["u"])

    def follow_user(self, follower_id: str, followee_id: str) -> bool:
        """사용자 팔로우"""
        query = """
        MATCH (follower:User {id: $follower_id})
        MATCH (followee:User {id: $followee_id})
        WHERE follower <> followee
        MERGE (follower)-[r:FOLLOWS]->(followee)
        ON CREATE SET r.since = datetime()
        WITH follower, followee, r
        SET follower.followingCount = follower.followingCount + 1,
            followee.followerCount = followee.followerCount + 1
        RETURN r
        """
        with self.db.session() as session:
            result = session.run(query,
                follower_id=follower_id,
                followee_id=followee_id
            )
            return result.single() is not None

    def get_followers(self, user_id: str, skip: int = 0,
                     limit: int = 20) -> List[dict]:
        """팔로워 목록 조회"""
        query = """
        MATCH (u:User {id: $user_id})<-[:FOLLOWS]-(follower:User)
        RETURN follower {
            .id, .name, .avatar, .bio, .isVerified,
            .followerCount, .followingCount
        } AS follower
        ORDER BY follower.followerCount DESC
        SKIP $skip LIMIT $limit
        """
        with self.db.session() as session:
            result = session.run(query,
                user_id=user_id, skip=skip, limit=limit)
            return [dict(r["follower"]) for r in result]

    def get_mutual_friends(self, user1_id: str, user2_id: str) -> List[dict]:
        """공통 친구 조회"""
        query = """
        MATCH (u1:User {id: $user1_id})-[:FOLLOWS]-(mutual:User)-[:FOLLOWS]-(u2:User {id: $user2_id})
        WHERE u1 <> u2 AND mutual <> u1 AND mutual <> u2
        RETURN DISTINCT mutual {.id, .name, .avatar}
        LIMIT 50
        """
        with self.db.session() as session:
            result = session.run(query,
                user1_id=user1_id, user2_id=user2_id)
            return [dict(r["mutual"]) for r in result]
```

### 추천 서비스 (recommendation_service.py)

```python
# api/services/recommendation_service.py
from typing import List

class RecommendationService:
    def __init__(self, db):
        self.db = db

    def get_friend_recommendations(self, user_id: str,
                                   limit: int = 10) -> List[dict]:
        """친구 추천 (친구의 친구 기반)"""
        query = """
        MATCH (me:User {id: $user_id})-[:FOLLOWS]->(friend)-[:FOLLOWS]->(fof:User)
        WHERE NOT (me)-[:FOLLOWS]->(fof)
          AND me <> fof
        WITH fof, count(DISTINCT friend) AS mutualFriends

        // 공통 관심사 점수
        MATCH (me:User {id: $user_id})
        WITH fof, mutualFriends,
             size([i IN me.interests WHERE i IN fof.interests]) AS commonInterests

        // 최종 점수 계산
        WITH fof,
             mutualFriends * 10 + commonInterests * 5 AS score,
             mutualFriends,
             commonInterests

        RETURN fof {
            .id, .name, .avatar, .bio, .isVerified,
            .followerCount
        } AS user,
        score,
        mutualFriends,
        commonInterests
        ORDER BY score DESC
        LIMIT $limit
        """
        with self.db.session() as session:
            result = session.run(query, user_id=user_id, limit=limit)
            return [dict(r) for r in result]

    def get_content_recommendations(self, user_id: str,
                                    limit: int = 20) -> List[dict]:
        """콘텐츠 추천 (관심사 기반)"""
        query = """
        // 사용자의 관심 해시태그
        MATCH (me:User {id: $user_id})-[:POSTED|LIKED]->(:Post)-[:TAGGED]->(t:Hashtag)
        WITH me, collect(DISTINCT t.name) AS myTags

        // 비슷한 태그의 인기 게시물
        MATCH (p:Post)-[:TAGGED]->(tag:Hashtag)
        WHERE tag.name IN myTags
          AND NOT (me)-[:LIKED]->(p)
          AND p.createdAt > datetime() - duration('P7D')

        MATCH (author:User)-[:POSTED]->(p)

        WITH p, author,
             count(DISTINCT tag) AS tagMatches,
             p.likeCount + p.commentCount * 2 + p.shareCount * 3 AS engagement

        RETURN p {
            .id, .content, .mediaUrls, .createdAt,
            .likeCount, .commentCount, .shareCount,
            author: author {.id, .name, .avatar, .isVerified}
        } AS post,
        tagMatches,
        engagement,
        tagMatches * 10 + engagement AS score
        ORDER BY score DESC
        LIMIT $limit
        """
        with self.db.session() as session:
            result = session.run(query, user_id=user_id, limit=limit)
            return [dict(r) for r in result]

    def get_trending_hashtags(self, hours: int = 24,
                              limit: int = 10) -> List[dict]:
        """트렌딩 해시태그"""
        query = """
        MATCH (p:Post)-[:TAGGED]->(t:Hashtag)
        WHERE p.createdAt > datetime() - duration('PT' + $hours + 'H')
        WITH t, count(p) AS recentPosts,
             sum(p.likeCount + p.shareCount) AS engagement
        RETURN t.name AS hashtag,
               recentPosts,
               engagement,
               recentPosts * 2 + engagement AS trendScore
        ORDER BY trendScore DESC
        LIMIT $limit
        """
        with self.db.session() as session:
            result = session.run(query, hours=str(hours), limit=limit)
            return [dict(r) for r in result]
```

### 분석 서비스 (analytics_service.py)

```python
# api/services/analytics_service.py
from typing import List, Dict

class AnalyticsService:
    def __init__(self, db):
        self.db = db

    def find_influencers(self, min_followers: int = 1000,
                        limit: int = 50) -> List[dict]:
        """인플루언서 탐지 (PageRank 기반)"""
        # 먼저 그래프 프로젝션 생성
        projection_query = """
        CALL gds.graph.project.cypher(
            'influencer-graph',
            'MATCH (u:User) WHERE u.followerCount >= $minFollowers RETURN id(u) AS id',
            'MATCH (a:User)-[:FOLLOWS]->(b:User) RETURN id(a) AS source, id(b) AS target',
            {parameters: {minFollowers: $minFollowers}}
        )
        """

        pagerank_query = """
        CALL gds.pageRank.stream('influencer-graph', {
            maxIterations: 20,
            dampingFactor: 0.85
        })
        YIELD nodeId, score
        WITH gds.util.asNode(nodeId) AS user, score
        RETURN user {
            .id, .name, .avatar, .bio, .isVerified,
            .followerCount, .followingCount
        } AS user,
        score AS influenceScore,
        user.followerCount AS followers
        ORDER BY score DESC
        LIMIT $limit
        """

        cleanup_query = "CALL gds.graph.drop('influencer-graph', false)"

        with self.db.session() as session:
            try:
                session.run(projection_query, minFollowers=min_followers)
                result = session.run(pagerank_query, limit=limit)
                influencers = [dict(r) for r in result]
            finally:
                session.run(cleanup_query)

            return influencers

    def detect_communities(self) -> List[dict]:
        """커뮤니티 탐지 (Louvain)"""
        projection_query = """
        CALL gds.graph.project(
            'community-graph',
            'User',
            {FOLLOWS: {orientation: 'UNDIRECTED'}}
        )
        """

        louvain_query = """
        CALL gds.louvain.stream('community-graph')
        YIELD nodeId, communityId
        WITH communityId, collect(gds.util.asNode(nodeId)) AS members
        WITH communityId,
             size(members) AS memberCount,
             [m IN members | m {.id, .name, .avatar}][0..5] AS sampleMembers
        WHERE memberCount >= 3
        RETURN communityId,
               memberCount,
               sampleMembers
        ORDER BY memberCount DESC
        LIMIT 20
        """

        cleanup_query = "CALL gds.graph.drop('community-graph', false)"

        with self.db.session() as session:
            try:
                session.run(projection_query)
                result = session.run(louvain_query)
                communities = [dict(r) for r in result]
            finally:
                session.run(cleanup_query)

            return communities

    def analyze_viral_spread(self, post_id: str) -> dict:
        """바이럴 확산 분석"""
        query = """
        MATCH (original:Post {id: $post_id})<-[:POSTED]-(author:User)

        // 직접 공유
        OPTIONAL MATCH (sharer:User)-[:SHARED]->(original)
        WITH original, author, collect(DISTINCT sharer) AS directSharers

        // 2차 확산 (공유자의 팔로워가 본 경우)
        UNWIND directSharers AS sharer
        OPTIONAL MATCH (sharer)<-[:FOLLOWS]-(secondDegree:User)

        WITH original, author,
             directSharers,
             collect(DISTINCT secondDegree) AS secondDegreeReach

        RETURN {
            postId: original.id,
            author: author {.id, .name},
            directShares: size(directSharers),
            potentialReach: size(secondDegreeReach),
            engagementRate: toFloat(original.likeCount + original.shareCount) /
                           CASE WHEN author.followerCount > 0
                                THEN author.followerCount ELSE 1 END,
            viralCoefficient: toFloat(size(directSharers)) /
                             CASE WHEN author.followerCount > 0
                                  THEN author.followerCount ELSE 1 END
        } AS analysis
        """
        with self.db.session() as session:
            result = session.run(query, post_id=post_id)
            record = result.single()
            return dict(record["analysis"]) if record else None

    def get_network_metrics(self, user_id: str) -> dict:
        """개인 네트워크 메트릭"""
        query = """
        MATCH (u:User {id: $user_id})

        // 기본 메트릭
        OPTIONAL MATCH (u)-[:FOLLOWS]->(following)
        OPTIONAL MATCH (u)<-[:FOLLOWS]-(follower)

        WITH u, count(DISTINCT following) AS followingCount,
                count(DISTINCT follower) AS followerCount

        // 2단계 도달 범위
        OPTIONAL MATCH (u)-[:FOLLOWS]->()-[:FOLLOWS]->(reach2:User)
        WHERE reach2 <> u

        WITH u, followingCount, followerCount,
             count(DISTINCT reach2) AS reach2Count

        // 네트워크 밀도 (친구들 간 연결)
        OPTIONAL MATCH (u)-[:FOLLOWS]->(f1)-[:FOLLOWS]->(f2)<-[:FOLLOWS]-(u)
        WHERE f1 <> f2

        WITH u, followingCount, followerCount, reach2Count,
             count(DISTINCT f1) + count(DISTINCT f2) AS connectedFriends

        RETURN {
            userId: u.id,
            followers: followerCount,
            following: followingCount,
            reach2Degree: reach2Count,
            networkDensity: CASE WHEN followingCount > 1
                THEN toFloat(connectedFriends) / (followingCount * (followingCount - 1))
                ELSE 0 END,
            engagementRatio: CASE WHEN followerCount > 0
                THEN toFloat(followingCount) / followerCount
                ELSE 0 END
        } AS metrics
        """
        with self.db.session() as session:
            result = session.run(query, user_id=user_id)
            record = result.single()
            return dict(record["metrics"]) if record else None
```

### FastAPI 라우터 (main.py)

```python
# api/main.py
from fastapi import FastAPI, HTTPException, Query
from pydantic import BaseModel, EmailStr
from typing import List, Optional
from database import db
from services.user_service import UserService
from services.recommendation_service import RecommendationService
from services.analytics_service import AnalyticsService

app = FastAPI(title="Social Network API", version="1.0.0")

user_service = UserService(db)
recommendation_service = RecommendationService(db)
analytics_service = AnalyticsService(db)

# Pydantic 모델
class UserCreate(BaseModel):
    email: EmailStr
    name: str
    bio: Optional[str] = ""
    interests: Optional[List[str]] = []

class FollowRequest(BaseModel):
    followee_id: str

# 엔드포인트
@app.post("/users", response_model=dict)
def create_user(user: UserCreate):
    return user_service.create_user(
        email=user.email,
        name=user.name,
        bio=user.bio,
        interests=user.interests
    )

@app.post("/users/{user_id}/follow")
def follow_user(user_id: str, request: FollowRequest):
    success = user_service.follow_user(user_id, request.followee_id)
    if not success:
        raise HTTPException(400, "Failed to follow user")
    return {"status": "followed"}

@app.get("/users/{user_id}/followers")
def get_followers(user_id: str, skip: int = 0, limit: int = 20):
    return user_service.get_followers(user_id, skip, limit)

@app.get("/users/{user_id}/recommendations/friends")
def get_friend_recommendations(user_id: str, limit: int = 10):
    return recommendation_service.get_friend_recommendations(user_id, limit)

@app.get("/users/{user_id}/recommendations/content")
def get_content_recommendations(user_id: str, limit: int = 20):
    return recommendation_service.get_content_recommendations(user_id, limit)

@app.get("/trending/hashtags")
def get_trending_hashtags(hours: int = 24, limit: int = 10):
    return recommendation_service.get_trending_hashtags(hours, limit)

@app.get("/analytics/influencers")
def get_influencers(min_followers: int = 1000, limit: int = 50):
    return analytics_service.find_influencers(min_followers, limit)

@app.get("/analytics/communities")
def get_communities():
    return analytics_service.detect_communities()

@app.get("/analytics/posts/{post_id}/viral")
def analyze_viral(post_id: str):
    result = analytics_service.analyze_viral_spread(post_id)
    if not result:
        raise HTTPException(404, "Post not found")
    return result

@app.get("/users/{user_id}/metrics")
def get_user_metrics(user_id: str):
    result = analytics_service.get_network_metrics(user_id)
    if not result:
        raise HTTPException(404, "User not found")
    return result

@app.on_event("shutdown")
def shutdown():
    db.close()
```

---

## 4단계: 샘플 데이터 생성

```cypher
// 샘플 사용자 생성
UNWIND range(1, 100) AS i
CREATE (u:User {
  id: 'user_' + toString(i),
  email: 'user' + toString(i) + '@example.com',
  name: 'User ' + toString(i),
  bio: 'Bio for user ' + toString(i),
  interests: CASE i % 5
    WHEN 0 THEN ['tech', 'ai', 'startup']
    WHEN 1 THEN ['music', 'art', 'design']
    WHEN 2 THEN ['sports', 'fitness', 'health']
    WHEN 3 THEN ['food', 'travel', 'photography']
    ELSE ['gaming', 'movies', 'books']
  END,
  joinedAt: datetime() - duration('P' + toString(toInteger(rand() * 365)) + 'D'),
  isVerified: rand() > 0.9,
  followerCount: 0,
  followingCount: 0
});

// 랜덤 팔로우 관계 생성
MATCH (a:User), (b:User)
WHERE a <> b AND rand() < 0.05
CREATE (a)-[:FOLLOWS {since: datetime()}]->(b);

// 팔로워/팔로잉 카운트 업데이트
MATCH (u:User)
SET u.followerCount = size((u)<-[:FOLLOWS]-()),
    u.followingCount = size((u)-[:FOLLOWS]->());
```

---

## 5단계: 실행 및 테스트

```bash
# 실행
docker-compose up -d

# API 테스트
curl -X POST http://localhost:8000/users \
  -H "Content-Type: application/json" \
  -d '{"email": "test@example.com", "name": "Test User", "interests": ["tech", "ai"]}'

# 친구 추천
curl http://localhost:8000/users/user_1/recommendations/friends

# 인플루언서 조회
curl http://localhost:8000/analytics/influencers
```

---

## 다음 단계

이 가이드를 기반으로:
- 실시간 알림 시스템 추가
- 그래프 시각화 프론트엔드 구현
- 스팸/악용 탐지 기능 추가

[실시간 추천 엔진](recommendation-engine.md)에서 더 고도화된 추천 시스템을 구현해보세요.
