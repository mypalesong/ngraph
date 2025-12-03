# REST API

## 개요

지식 그래프를 위한 RESTful API 설계 가이드입니다.

## 엔드포인트 설계

```
GET    /api/nodes/{type}           # 노드 목록
GET    /api/nodes/{type}/{id}      # 노드 상세
POST   /api/nodes/{type}           # 노드 생성
PUT    /api/nodes/{type}/{id}      # 노드 수정
DELETE /api/nodes/{type}/{id}      # 노드 삭제

GET    /api/nodes/{id}/relations   # 관계 조회
POST   /api/relations              # 관계 생성
```

## FastAPI 구현

```python
from fastapi import FastAPI, HTTPException
from neo4j import GraphDatabase

app = FastAPI()
driver = GraphDatabase.driver("bolt://localhost:7687", auth=("neo4j", "password"))

@app.get("/api/nodes/person/{id}")
def get_person(id: str):
    with driver.session() as session:
        result = session.run(
            "MATCH (p:Person {id: $id}) RETURN p",
            id=id
        )
        record = result.single()
        if not record:
            raise HTTPException(404, "Not found")
        return dict(record["p"])

@app.post("/api/nodes/person")
def create_person(data: dict):
    with driver.session() as session:
        result = session.run(
            "CREATE (p:Person $props) RETURN p",
            props=data
        )
        return dict(result.single()["p"])

@app.get("/api/nodes/{id}/relations")
def get_relations(id: str):
    with driver.session() as session:
        result = session.run("""
            MATCH (n {id: $id})-[r]-(m)
            RETURN type(r) AS type, m.id AS target
        """, id=id)
        return [dict(r) for r in result]
```

## 다음 단계

- [고급 주제](../advanced/graph-embedding.md)에서 심화 내용을 학습하세요
