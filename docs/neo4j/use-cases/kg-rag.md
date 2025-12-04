# 지식 그래프 RAG (Knowledge Graph RAG)

## 개요

**Knowledge Graph RAG**는 LLM의 환각(hallucination)을 줄이고 정확한 정보를 제공하기 위해 지식 그래프를 검색 증강 생성(RAG)에 활용하는 시스템입니다.

### 기존 RAG vs Knowledge Graph RAG

| 항목 | 기존 벡터 RAG | Knowledge Graph RAG |
|------|--------------|---------------------|
| 검색 방식 | 유사도 기반 | 관계 기반 + 유사도 |
| 컨텍스트 | 청크 단위 | 구조화된 지식 |
| 추론 | 제한적 | 다중 홉 추론 가능 |
| 설명 가능성 | 낮음 | 높음 (경로 추적) |
| 업데이트 | 재인덱싱 필요 | 실시간 가능 |

---

## 시스템 아키텍처

```
┌──────────────────────────────────────────────────────────────┐
│                      User Query                               │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                  Query Understanding                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐   │
│  │Entity Extract│  │Intent Class │  │ Query Decomposition │   │
│  └─────────────┘  └─────────────┘  └─────────────────────┘   │
└──────────────────────┬───────────────────────────────────────┘
                       │
        ┌──────────────┴──────────────┐
        ▼                             ▼
┌───────────────────┐    ┌────────────────────────┐
│   Vector Search   │    │  Knowledge Graph Search │
│   (Embeddings)    │    │   (Cypher Queries)      │
└────────┬──────────┘    └───────────┬────────────┘
         │                           │
         └─────────┬─────────────────┘
                   ▼
┌──────────────────────────────────────────────────────────────┐
│                   Context Fusion                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐   │
│  │  Ranking    │  │  Filtering  │  │   Graph Context     │   │
│  └─────────────┘  └─────────────┘  └─────────────────────┘   │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                      LLM Generation                           │
│            (with structured knowledge context)                │
└──────────────────────────────────────────────────────────────┘
```

---

## 환경 설정

### Docker Compose

```yaml
version: '3.8'
services:
  neo4j:
    image: neo4j:5.15.0
    ports:
      - "7474:7474"
      - "7687:7687"
    environment:
      - NEO4J_AUTH=neo4j/password123
      - NEO4J_PLUGINS=["apoc", "graph-data-science"]
      - NEO4J_apoc_export_file_enabled=true
      - NEO4J_apoc_import_file_enabled=true
    volumes:
      - neo4j_data:/data
      - neo4j_import:/var/lib/neo4j/import

  rag-api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - NEO4J_URI=bolt://neo4j:7687
      - NEO4J_USER=neo4j
      - NEO4J_PASSWORD=password123
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    depends_on:
      - neo4j

volumes:
  neo4j_data:
  neo4j_import:
```

### 의존성 (requirements.txt)

```
neo4j==5.15.0
openai==1.12.0
langchain==0.1.0
langchain-openai==0.0.5
langchain-community==0.0.20
sentence-transformers==2.3.1
fastapi==0.109.0
uvicorn==0.27.0
pydantic==2.5.3
numpy==1.26.3
```

---

## 지식 그래프 스키마

### 데이터 모델

```cypher
// 엔티티 노드
CREATE CONSTRAINT entity_id FOR (e:Entity) REQUIRE e.id IS UNIQUE;
CREATE CONSTRAINT concept_name FOR (c:Concept) REQUIRE c.name IS UNIQUE;

// 문서 노드
CREATE CONSTRAINT document_id FOR (d:Document) REQUIRE d.id IS UNIQUE;
CREATE CONSTRAINT chunk_id FOR (c:Chunk) REQUIRE c.id IS UNIQUE;

// 벡터 인덱스 (Neo4j 5.11+)
CREATE VECTOR INDEX chunk_embedding FOR (c:Chunk) ON (c.embedding)
OPTIONS {indexConfig: {
  `vector.dimensions`: 1536,
  `vector.similarity_function`: 'cosine'
}};

CREATE VECTOR INDEX entity_embedding FOR (e:Entity) ON (e.embedding)
OPTIONS {indexConfig: {
  `vector.dimensions`: 1536,
  `vector.similarity_function`: 'cosine'
}};
```

### 관계 유형

```cypher
// 문서 구조
(Document)-[:HAS_CHUNK]->(Chunk)
(Chunk)-[:NEXT]->(Chunk)

// 엔티티 관계
(Entity)-[:MENTIONED_IN]->(Chunk)
(Entity)-[:RELATED_TO {type: 'string', weight: float}]->(Entity)
(Entity)-[:INSTANCE_OF]->(Concept)

// 지식 관계
(Entity)-[:HAS_PROPERTY {name: 'string', value: 'any'}]->(Entity)
(Concept)-[:SUBCLASS_OF]->(Concept)
```

---

## 구현

### 기본 설정

```python
# config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    neo4j_uri: str = "bolt://localhost:7687"
    neo4j_user: str = "neo4j"
    neo4j_password: str = "password123"
    openai_api_key: str = ""
    embedding_model: str = "text-embedding-3-small"
    llm_model: str = "gpt-4-turbo-preview"
    chunk_size: int = 500
    chunk_overlap: int = 50
    top_k_vectors: int = 5
    max_graph_hops: int = 2

settings = Settings()
```

### Neo4j 연결 및 유틸리티

```python
# database.py
from neo4j import GraphDatabase
from contextlib import contextmanager
from config import settings

class Neo4jConnection:
    def __init__(self):
        self.driver = GraphDatabase.driver(
            settings.neo4j_uri,
            auth=(settings.neo4j_user, settings.neo4j_password)
        )

    @contextmanager
    def session(self):
        session = self.driver.session()
        try:
            yield session
        finally:
            session.close()

    def close(self):
        self.driver.close()

db = Neo4jConnection()
```

### 문서 처리 및 청킹

```python
# document_processor.py
from typing import List, Dict, Any
from langchain.text_splitter import RecursiveCharacterTextSplitter
from openai import OpenAI
import hashlib
import uuid
from config import settings
from database import db

client = OpenAI(api_key=settings.openai_api_key)

class DocumentProcessor:
    def __init__(self):
        self.text_splitter = RecursiveCharacterTextSplitter(
            chunk_size=settings.chunk_size,
            chunk_overlap=settings.chunk_overlap,
            separators=["\n\n", "\n", ". ", " ", ""]
        )

    def get_embedding(self, text: str) -> List[float]:
        """OpenAI 임베딩 생성"""
        response = client.embeddings.create(
            model=settings.embedding_model,
            input=text
        )
        return response.data[0].embedding

    def process_document(self, title: str, content: str, metadata: Dict = None) -> str:
        """문서를 처리하고 그래프에 저장"""
        doc_id = str(uuid.uuid4())

        # 문서 노드 생성
        with db.session() as session:
            session.run("""
                CREATE (d:Document {
                    id: $doc_id,
                    title: $title,
                    created_at: datetime(),
                    metadata: $metadata
                })
            """, doc_id=doc_id, title=title, metadata=metadata or {})

        # 청킹
        chunks = self.text_splitter.split_text(content)

        # 청크 노드 생성 및 연결
        prev_chunk_id = None
        for i, chunk_text in enumerate(chunks):
            chunk_id = f"{doc_id}_chunk_{i}"
            embedding = self.get_embedding(chunk_text)

            with db.session() as session:
                # 청크 생성
                session.run("""
                    MATCH (d:Document {id: $doc_id})
                    CREATE (c:Chunk {
                        id: $chunk_id,
                        text: $text,
                        embedding: $embedding,
                        position: $position
                    })
                    CREATE (d)-[:HAS_CHUNK]->(c)
                """, doc_id=doc_id, chunk_id=chunk_id, text=chunk_text,
                     embedding=embedding, position=i)

                # 이전 청크와 연결
                if prev_chunk_id:
                    session.run("""
                        MATCH (c1:Chunk {id: $prev_id})
                        MATCH (c2:Chunk {id: $curr_id})
                        CREATE (c1)-[:NEXT]->(c2)
                    """, prev_id=prev_chunk_id, curr_id=chunk_id)

            prev_chunk_id = chunk_id

        # 엔티티 추출 및 연결
        self._extract_and_link_entities(doc_id, chunks)

        return doc_id

    def _extract_and_link_entities(self, doc_id: str, chunks: List[str]):
        """LLM을 사용하여 엔티티 추출"""
        for i, chunk_text in enumerate(chunks):
            chunk_id = f"{doc_id}_chunk_{i}"

            # LLM으로 엔티티 추출
            prompt = f"""다음 텍스트에서 주요 엔티티(사람, 조직, 개념, 기술 등)를 추출하세요.
JSON 형식으로 반환하세요: [{{"name": "엔티티명", "type": "유형", "description": "설명"}}]

텍스트: {chunk_text}"""

            response = client.chat.completions.create(
                model=settings.llm_model,
                messages=[{"role": "user", "content": prompt}],
                response_format={"type": "json_object"}
            )

            try:
                import json
                result = json.loads(response.choices[0].message.content)
                entities = result.get("entities", [])

                for entity in entities:
                    self._create_or_update_entity(
                        entity["name"],
                        entity.get("type", "Unknown"),
                        entity.get("description", ""),
                        chunk_id
                    )
            except Exception as e:
                print(f"Entity extraction error: {e}")

    def _create_or_update_entity(self, name: str, entity_type: str,
                                  description: str, chunk_id: str):
        """엔티티 생성 또는 업데이트"""
        entity_id = hashlib.md5(name.lower().encode()).hexdigest()
        embedding = self.get_embedding(f"{name}: {description}")

        with db.session() as session:
            session.run("""
                MERGE (e:Entity {id: $entity_id})
                ON CREATE SET
                    e.name = $name,
                    e.type = $entity_type,
                    e.description = $description,
                    e.embedding = $embedding,
                    e.mention_count = 1
                ON MATCH SET
                    e.mention_count = e.mention_count + 1,
                    e.description = CASE
                        WHEN size(e.description) < size($description)
                        THEN $description
                        ELSE e.description
                    END

                WITH e
                MATCH (c:Chunk {id: $chunk_id})
                MERGE (e)-[:MENTIONED_IN]->(c)
            """, entity_id=entity_id, name=name, entity_type=entity_type,
                 description=description, embedding=embedding, chunk_id=chunk_id)
```

### 엔티티 관계 추출

```python
# relationship_extractor.py
from typing import List, Dict
from openai import OpenAI
from config import settings
from database import db
import json

client = OpenAI(api_key=settings.openai_api_key)

class RelationshipExtractor:
    def extract_relationships(self, doc_id: str):
        """문서 내 엔티티 간 관계 추출"""
        with db.session() as session:
            # 문서의 모든 청크와 엔티티 가져오기
            result = session.run("""
                MATCH (d:Document {id: $doc_id})-[:HAS_CHUNK]->(c:Chunk)
                OPTIONAL MATCH (e:Entity)-[:MENTIONED_IN]->(c)
                RETURN c.text AS chunk_text, collect(DISTINCT e.name) AS entities
                ORDER BY c.position
            """, doc_id=doc_id)

            chunks_with_entities = list(result)

        # 관계 추출
        for record in chunks_with_entities:
            if len(record["entities"]) < 2:
                continue

            self._extract_chunk_relationships(
                record["chunk_text"],
                record["entities"]
            )

    def _extract_chunk_relationships(self, text: str, entities: List[str]):
        """청크 내 엔티티 관계 추출"""
        entities_str = ", ".join(entities)

        prompt = f"""다음 텍스트에서 엔티티들 간의 관계를 추출하세요.
엔티티 목록: {entities_str}

관계 유형 예시: works_for, develops, uses, contains, related_to, depends_on, competes_with 등

JSON 형식으로 반환:
{{"relationships": [{{"source": "엔티티1", "target": "엔티티2", "type": "관계유형", "description": "관계설명"}}]}}

텍스트: {text}"""

        response = client.chat.completions.create(
            model=settings.llm_model,
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"}
        )

        try:
            result = json.loads(response.choices[0].message.content)
            relationships = result.get("relationships", [])

            for rel in relationships:
                self._create_relationship(
                    rel["source"],
                    rel["target"],
                    rel["type"],
                    rel.get("description", "")
                )
        except Exception as e:
            print(f"Relationship extraction error: {e}")

    def _create_relationship(self, source: str, target: str,
                             rel_type: str, description: str):
        """관계 생성"""
        with db.session() as session:
            session.run("""
                MATCH (e1:Entity) WHERE toLower(e1.name) = toLower($source)
                MATCH (e2:Entity) WHERE toLower(e2.name) = toLower($target)
                MERGE (e1)-[r:RELATED_TO {type: $rel_type}]->(e2)
                ON CREATE SET r.description = $description, r.weight = 1.0
                ON MATCH SET r.weight = r.weight + 0.1
            """, source=source, target=target, rel_type=rel_type,
                 description=description)
```

### 하이브리드 검색 엔진

```python
# retriever.py
from typing import List, Dict, Any, Tuple
from dataclasses import dataclass
from openai import OpenAI
from config import settings
from database import db

client = OpenAI(api_key=settings.openai_api_key)

@dataclass
class RetrievalResult:
    chunks: List[Dict[str, Any]]
    entities: List[Dict[str, Any]]
    subgraph: Dict[str, Any]
    paths: List[Dict[str, Any]]

class HybridRetriever:
    def retrieve(self, query: str) -> RetrievalResult:
        """하이브리드 검색 수행"""
        # 1. 쿼리 임베딩 생성
        query_embedding = self._get_embedding(query)

        # 2. 쿼리에서 엔티티 추출
        query_entities = self._extract_query_entities(query)

        # 3. 벡터 검색 (청크)
        vector_chunks = self._vector_search_chunks(query_embedding)

        # 4. 벡터 검색 (엔티티)
        vector_entities = self._vector_search_entities(query_embedding)

        # 5. 그래프 검색 (추출된 엔티티 기반)
        graph_entities = self._graph_search_entities(query_entities)

        # 6. 서브그래프 추출
        all_entity_ids = list(set(
            [e["id"] for e in vector_entities] +
            [e["id"] for e in graph_entities]
        ))
        subgraph = self._extract_subgraph(all_entity_ids)

        # 7. 경로 검색 (엔티티 간)
        paths = self._find_relevant_paths(all_entity_ids)

        # 8. 결과 통합 및 랭킹
        entities = self._merge_and_rank_entities(
            vector_entities, graph_entities
        )

        return RetrievalResult(
            chunks=vector_chunks,
            entities=entities[:10],
            subgraph=subgraph,
            paths=paths
        )

    def _get_embedding(self, text: str) -> List[float]:
        response = client.embeddings.create(
            model=settings.embedding_model,
            input=text
        )
        return response.data[0].embedding

    def _extract_query_entities(self, query: str) -> List[str]:
        """쿼리에서 엔티티 추출"""
        prompt = f"""다음 질문에서 검색해야 할 주요 엔티티(키워드)를 추출하세요.
JSON 형식: {{"entities": ["엔티티1", "엔티티2"]}}

질문: {query}"""

        response = client.chat.completions.create(
            model=settings.llm_model,
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"}
        )

        try:
            import json
            result = json.loads(response.choices[0].message.content)
            return result.get("entities", [])
        except:
            return []

    def _vector_search_chunks(self, embedding: List[float]) -> List[Dict]:
        """벡터 유사도 기반 청크 검색"""
        with db.session() as session:
            result = session.run("""
                CALL db.index.vector.queryNodes('chunk_embedding', $top_k, $embedding)
                YIELD node, score
                MATCH (d:Document)-[:HAS_CHUNK]->(node)
                OPTIONAL MATCH (node)-[:NEXT]->(next:Chunk)
                OPTIONAL MATCH (prev:Chunk)-[:NEXT]->(node)
                RETURN
                    node.id AS id,
                    node.text AS text,
                    score,
                    d.title AS document_title,
                    prev.text AS prev_text,
                    next.text AS next_text
                ORDER BY score DESC
            """, top_k=settings.top_k_vectors, embedding=embedding)

            return [dict(r) for r in result]

    def _vector_search_entities(self, embedding: List[float]) -> List[Dict]:
        """벡터 유사도 기반 엔티티 검색"""
        with db.session() as session:
            result = session.run("""
                CALL db.index.vector.queryNodes('entity_embedding', $top_k, $embedding)
                YIELD node, score
                RETURN
                    node.id AS id,
                    node.name AS name,
                    node.type AS type,
                    node.description AS description,
                    score,
                    'vector' AS source
                ORDER BY score DESC
            """, top_k=settings.top_k_vectors, embedding=embedding)

            return [dict(r) for r in result]

    def _graph_search_entities(self, entity_names: List[str]) -> List[Dict]:
        """그래프 기반 엔티티 검색"""
        if not entity_names:
            return []

        with db.session() as session:
            result = session.run("""
                UNWIND $names AS name
                MATCH (e:Entity)
                WHERE toLower(e.name) CONTAINS toLower(name)
                   OR toLower(e.description) CONTAINS toLower(name)
                RETURN DISTINCT
                    e.id AS id,
                    e.name AS name,
                    e.type AS type,
                    e.description AS description,
                    e.mention_count AS mentions,
                    'graph' AS source
                ORDER BY e.mention_count DESC
                LIMIT 20
            """, names=entity_names)

            return [dict(r) for r in result]

    def _extract_subgraph(self, entity_ids: List[str]) -> Dict[str, Any]:
        """관련 엔티티 서브그래프 추출"""
        if not entity_ids:
            return {"nodes": [], "relationships": []}

        with db.session() as session:
            result = session.run("""
                MATCH (e:Entity)
                WHERE e.id IN $entity_ids
                OPTIONAL MATCH (e)-[r:RELATED_TO]-(e2:Entity)
                WHERE e2.id IN $entity_ids OR r.weight > 0.5
                WITH e, r, e2
                RETURN
                    collect(DISTINCT {
                        id: e.id,
                        name: e.name,
                        type: e.type
                    }) AS nodes,
                    collect(DISTINCT CASE WHEN r IS NOT NULL THEN {
                        source: startNode(r).name,
                        target: endNode(r).name,
                        type: r.type,
                        description: r.description
                    } END) AS relationships
            """, entity_ids=entity_ids)

            record = result.single()
            if record:
                return {
                    "nodes": record["nodes"],
                    "relationships": [r for r in record["relationships"] if r]
                }
            return {"nodes": [], "relationships": []}

    def _find_relevant_paths(self, entity_ids: List[str]) -> List[Dict]:
        """엔티티 간 경로 검색"""
        if len(entity_ids) < 2:
            return []

        with db.session() as session:
            result = session.run("""
                MATCH (e1:Entity), (e2:Entity)
                WHERE e1.id IN $entity_ids
                  AND e2.id IN $entity_ids
                  AND e1 <> e2
                MATCH path = shortestPath((e1)-[*..3]-(e2))
                WITH path, e1, e2,
                     [n IN nodes(path) | n.name] AS path_nodes,
                     [r IN relationships(path) | type(r) + ': ' + coalesce(r.type, '')] AS path_rels
                RETURN
                    e1.name AS from_entity,
                    e2.name AS to_entity,
                    path_nodes,
                    path_rels,
                    length(path) AS hops
                ORDER BY hops
                LIMIT 10
            """, entity_ids=entity_ids)

            return [dict(r) for r in result]

    def _merge_and_rank_entities(self, vector_entities: List[Dict],
                                  graph_entities: List[Dict]) -> List[Dict]:
        """엔티티 통합 및 랭킹"""
        entity_scores = {}

        # 벡터 검색 결과 (유사도 점수)
        for e in vector_entities:
            entity_scores[e["id"]] = {
                **e,
                "final_score": e.get("score", 0) * 0.6
            }

        # 그래프 검색 결과 (멘션 수 기반)
        for e in graph_entities:
            if e["id"] in entity_scores:
                entity_scores[e["id"]]["final_score"] += 0.4
                entity_scores[e["id"]]["mentions"] = e.get("mentions", 0)
            else:
                entity_scores[e["id"]] = {
                    **e,
                    "final_score": 0.4
                }

        # 정렬
        ranked = sorted(
            entity_scores.values(),
            key=lambda x: x["final_score"],
            reverse=True
        )

        return ranked
```

### RAG 생성기

```python
# generator.py
from typing import Dict, Any, List
from openai import OpenAI
from retriever import HybridRetriever, RetrievalResult
from config import settings

client = OpenAI(api_key=settings.openai_api_key)

class KnowledgeGraphRAG:
    def __init__(self):
        self.retriever = HybridRetriever()

    def generate(self, query: str, stream: bool = False) -> Dict[str, Any]:
        """지식 그래프 기반 RAG 생성"""
        # 1. 검색
        retrieval = self.retriever.retrieve(query)

        # 2. 컨텍스트 구성
        context = self._build_context(retrieval)

        # 3. 프롬프트 구성
        system_prompt = self._build_system_prompt()
        user_prompt = self._build_user_prompt(query, context)

        # 4. LLM 생성
        if stream:
            return self._generate_stream(system_prompt, user_prompt, retrieval)
        else:
            return self._generate_sync(system_prompt, user_prompt, retrieval)

    def _build_context(self, retrieval: RetrievalResult) -> str:
        """검색 결과로 컨텍스트 구성"""
        context_parts = []

        # 1. 관련 문서 청크
        if retrieval.chunks:
            context_parts.append("## 관련 문서 내용")
            for i, chunk in enumerate(retrieval.chunks[:5], 1):
                context_parts.append(f"\n### 문서 {i}: {chunk.get('document_title', 'Unknown')}")
                context_parts.append(chunk["text"])

        # 2. 관련 엔티티 정보
        if retrieval.entities:
            context_parts.append("\n## 관련 엔티티")
            for entity in retrieval.entities[:10]:
                desc = entity.get("description", "")
                context_parts.append(
                    f"- **{entity['name']}** ({entity.get('type', 'Unknown')}): {desc}"
                )

        # 3. 엔티티 관계 (서브그래프)
        if retrieval.subgraph.get("relationships"):
            context_parts.append("\n## 엔티티 간 관계")
            for rel in retrieval.subgraph["relationships"][:10]:
                context_parts.append(
                    f"- {rel['source']} --[{rel['type']}]--> {rel['target']}"
                )

        # 4. 연결 경로
        if retrieval.paths:
            context_parts.append("\n## 개념 연결 경로")
            for path in retrieval.paths[:5]:
                path_str = " -> ".join(path["path_nodes"])
                context_parts.append(f"- {path_str}")

        return "\n".join(context_parts)

    def _build_system_prompt(self) -> str:
        return """당신은 지식 그래프 기반 AI 어시스턴트입니다.
제공된 컨텍스트 정보를 기반으로 정확하고 신뢰할 수 있는 답변을 제공합니다.

지침:
1. 컨텍스트에 있는 정보만 사용하여 답변하세요.
2. 컨텍스트에 없는 정보는 "제공된 정보에서 확인할 수 없습니다"라고 명시하세요.
3. 엔티티 간 관계를 활용하여 논리적인 추론을 제공하세요.
4. 답변의 근거가 되는 엔티티나 관계를 언급하세요.
5. 불확실한 정보는 추측하지 마세요."""

    def _build_user_prompt(self, query: str, context: str) -> str:
        return f"""## 컨텍스트 정보
{context}

## 질문
{query}

위 컨텍스트를 기반으로 질문에 답변해주세요."""

    def _generate_sync(self, system_prompt: str, user_prompt: str,
                       retrieval: RetrievalResult) -> Dict[str, Any]:
        """동기 생성"""
        response = client.chat.completions.create(
            model=settings.llm_model,
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": user_prompt}
            ],
            temperature=0.1
        )

        return {
            "answer": response.choices[0].message.content,
            "sources": {
                "chunks": [c["document_title"] for c in retrieval.chunks],
                "entities": [e["name"] for e in retrieval.entities],
                "relationships": retrieval.subgraph.get("relationships", [])[:5]
            },
            "usage": {
                "prompt_tokens": response.usage.prompt_tokens,
                "completion_tokens": response.usage.completion_tokens
            }
        }

    def _generate_stream(self, system_prompt: str, user_prompt: str,
                         retrieval: RetrievalResult):
        """스트리밍 생성"""
        stream = client.chat.completions.create(
            model=settings.llm_model,
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": user_prompt}
            ],
            temperature=0.1,
            stream=True
        )

        for chunk in stream:
            if chunk.choices[0].delta.content:
                yield {
                    "type": "content",
                    "data": chunk.choices[0].delta.content
                }

        # 소스 정보 반환
        yield {
            "type": "sources",
            "data": {
                "chunks": [c["document_title"] for c in retrieval.chunks],
                "entities": [e["name"] for e in retrieval.entities]
            }
        }
```

### 다중 홉 추론

```python
# multi_hop_reasoning.py
from typing import List, Dict, Any
from openai import OpenAI
from database import db
from config import settings

client = OpenAI(api_key=settings.openai_api_key)

class MultiHopReasoner:
    def reason(self, query: str, max_hops: int = 3) -> Dict[str, Any]:
        """다중 홉 추론 수행"""
        # 1. 쿼리 분해
        sub_questions = self._decompose_query(query)

        # 2. 단계별 추론
        reasoning_chain = []
        accumulated_context = ""

        for i, sub_q in enumerate(sub_questions):
            # 이전 컨텍스트를 포함한 검색
            result = self._retrieve_for_subquery(sub_q, accumulated_context)

            # 부분 답변 생성
            partial_answer = self._generate_partial_answer(sub_q, result)

            reasoning_chain.append({
                "step": i + 1,
                "question": sub_q,
                "evidence": result,
                "answer": partial_answer
            })

            accumulated_context += f"\n단계 {i+1}: {sub_q}\n답변: {partial_answer}"

        # 3. 최종 답변 종합
        final_answer = self._synthesize_answer(query, reasoning_chain)

        return {
            "query": query,
            "reasoning_chain": reasoning_chain,
            "final_answer": final_answer
        }

    def _decompose_query(self, query: str) -> List[str]:
        """복잡한 쿼리를 단계별 질문으로 분해"""
        prompt = f"""다음 질문을 답변하기 위해 필요한 단계별 하위 질문으로 분해하세요.
각 질문은 이전 질문의 답변을 바탕으로 할 수 있습니다.
JSON 형식: {{"sub_questions": ["질문1", "질문2", ...]}}

질문: {query}"""

        response = client.chat.completions.create(
            model=settings.llm_model,
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"}
        )

        import json
        result = json.loads(response.choices[0].message.content)
        return result.get("sub_questions", [query])

    def _retrieve_for_subquery(self, sub_query: str,
                                context: str) -> Dict[str, Any]:
        """하위 쿼리에 대한 그래프 검색"""
        # 이전 컨텍스트에서 엔티티 추출
        combined_query = f"{context}\n{sub_query}" if context else sub_query

        with db.session() as session:
            # 쿼리 키워드로 관련 엔티티 및 경로 검색
            result = session.run("""
                // 텍스트 매칭으로 시작점 찾기
                CALL db.index.fulltext.queryNodes('entity_fulltext', $query)
                YIELD node, score
                WHERE score > 0.5
                WITH node, score
                ORDER BY score DESC
                LIMIT 5

                // 연결된 엔티티 탐색
                MATCH (node)-[r:RELATED_TO*1..2]-(connected:Entity)
                WITH node, connected, r

                // 관련 청크 가져오기
                OPTIONAL MATCH (node)-[:MENTIONED_IN]->(chunk:Chunk)

                RETURN
                    node.name AS entity,
                    node.description AS description,
                    collect(DISTINCT connected.name)[..5] AS connected_entities,
                    collect(DISTINCT chunk.text)[..2] AS related_chunks
            """, query=sub_query)

            return [dict(r) for r in result]

    def _generate_partial_answer(self, question: str,
                                  evidence: List[Dict]) -> str:
        """부분 답변 생성"""
        evidence_text = "\n".join([
            f"- {e['entity']}: {e.get('description', '')} (연결: {', '.join(e.get('connected_entities', []))})"
            for e in evidence
        ])

        prompt = f"""다음 증거를 바탕으로 질문에 간단히 답변하세요.

증거:
{evidence_text}

질문: {question}"""

        response = client.chat.completions.create(
            model=settings.llm_model,
            messages=[{"role": "user", "content": prompt}],
            max_tokens=200
        )

        return response.choices[0].message.content

    def _synthesize_answer(self, query: str,
                           reasoning_chain: List[Dict]) -> str:
        """추론 체인을 종합하여 최종 답변 생성"""
        chain_text = "\n".join([
            f"단계 {step['step']}: {step['question']}\n답변: {step['answer']}"
            for step in reasoning_chain
        ])

        prompt = f"""다음 단계별 추론을 종합하여 원래 질문에 대한 최종 답변을 작성하세요.

원래 질문: {query}

추론 과정:
{chain_text}

종합적이고 정확한 최종 답변을 제공하세요."""

        response = client.chat.completions.create(
            model=settings.llm_model,
            messages=[{"role": "user", "content": prompt}]
        )

        return response.choices[0].message.content
```

### FastAPI 서버

```python
# main.py
from fastapi import FastAPI, HTTPException
from fastapi.responses import StreamingResponse
from pydantic import BaseModel
from typing import Optional, List
import json

from document_processor import DocumentProcessor
from relationship_extractor import RelationshipExtractor
from generator import KnowledgeGraphRAG
from multi_hop_reasoning import MultiHopReasoner

app = FastAPI(title="Knowledge Graph RAG API")

doc_processor = DocumentProcessor()
rel_extractor = RelationshipExtractor()
rag = KnowledgeGraphRAG()
reasoner = MultiHopReasoner()

class DocumentInput(BaseModel):
    title: str
    content: str
    metadata: Optional[dict] = None

class QueryInput(BaseModel):
    query: str
    stream: bool = False
    multi_hop: bool = False

class QueryResponse(BaseModel):
    answer: str
    sources: dict
    reasoning_chain: Optional[List[dict]] = None

# 문서 처리
@app.post("/documents")
async def add_document(doc: DocumentInput):
    """문서 추가 및 지식 그래프 구축"""
    try:
        doc_id = doc_processor.process_document(
            doc.title, doc.content, doc.metadata
        )
        rel_extractor.extract_relationships(doc_id)
        return {"doc_id": doc_id, "message": "Document processed successfully"}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# 질의 응답
@app.post("/query")
async def query(input: QueryInput):
    """지식 그래프 RAG 질의"""
    try:
        if input.multi_hop:
            result = reasoner.reason(input.query)
            return {
                "answer": result["final_answer"],
                "sources": {},
                "reasoning_chain": result["reasoning_chain"]
            }

        if input.stream:
            return StreamingResponse(
                _stream_response(input.query),
                media_type="text/event-stream"
            )

        result = rag.generate(input.query)
        return QueryResponse(**result)
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

async def _stream_response(query: str):
    """스트리밍 응답"""
    for chunk in rag.generate(query, stream=True):
        yield f"data: {json.dumps(chunk)}\n\n"
    yield "data: [DONE]\n\n"

# 그래프 탐색
@app.get("/entities/{entity_name}")
async def get_entity(entity_name: str, depth: int = 1):
    """엔티티 및 연결 정보 조회"""
    from database import db

    with db.session() as session:
        result = session.run("""
            MATCH (e:Entity)
            WHERE toLower(e.name) CONTAINS toLower($name)
            OPTIONAL MATCH (e)-[r:RELATED_TO*1..$depth]-(connected:Entity)
            RETURN e {.*} AS entity,
                   collect(DISTINCT connected {.*}) AS connected
        """, name=entity_name, depth=depth)

        record = result.single()
        if not record:
            raise HTTPException(status_code=404, detail="Entity not found")

        return {
            "entity": record["entity"],
            "connected": record["connected"]
        }

@app.get("/graph/stats")
async def get_graph_stats():
    """그래프 통계"""
    from database import db

    with db.session() as session:
        result = session.run("""
            MATCH (d:Document)
            WITH count(d) AS docs
            MATCH (c:Chunk)
            WITH docs, count(c) AS chunks
            MATCH (e:Entity)
            WITH docs, chunks, count(e) AS entities
            MATCH ()-[r:RELATED_TO]->()
            RETURN docs, chunks, entities, count(r) AS relationships
        """)

        record = result.single()
        return dict(record)

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## 고급 기능

### 그래프 기반 재랭킹

```python
# graph_reranker.py
from typing import List, Dict
from database import db

class GraphReranker:
    def rerank(self, query_entities: List[str],
               candidates: List[Dict],
               alpha: float = 0.5) -> List[Dict]:
        """그래프 중심성을 활용한 재랭킹"""

        # 후보 엔티티의 PageRank 점수 계산
        with db.session() as session:
            # 임시 그래프 프로젝션
            session.run("""
                CALL gds.graph.project(
                    'rerank_graph',
                    'Entity',
                    'RELATED_TO',
                    {relationshipProperties: 'weight'}
                )
            """)

            # PageRank 계산
            result = session.run("""
                CALL gds.pageRank.stream('rerank_graph')
                YIELD nodeId, score
                RETURN gds.util.asNode(nodeId).id AS entity_id, score
            """)

            pagerank_scores = {r["entity_id"]: r["score"] for r in result}

            # 그래프 삭제
            session.run("CALL gds.graph.drop('rerank_graph')")

        # 점수 조합
        for candidate in candidates:
            vector_score = candidate.get("score", 0)
            graph_score = pagerank_scores.get(candidate["id"], 0)

            # 하이브리드 점수
            candidate["hybrid_score"] = (
                alpha * vector_score +
                (1 - alpha) * graph_score
            )

        # 재정렬
        return sorted(candidates, key=lambda x: x["hybrid_score"], reverse=True)
```

### 시간 인식 검색

```python
# temporal_retriever.py
from datetime import datetime, timedelta
from typing import List, Dict, Optional
from database import db

class TemporalRetriever:
    def retrieve_with_time(self, query: str,
                           time_range: Optional[tuple] = None) -> List[Dict]:
        """시간 범위를 고려한 검색"""

        with db.session() as session:
            if time_range:
                start_date, end_date = time_range
                result = session.run("""
                    MATCH (d:Document)-[:HAS_CHUNK]->(c:Chunk)
                    WHERE d.created_at >= $start AND d.created_at <= $end
                    MATCH (e:Entity)-[:MENTIONED_IN]->(c)
                    WHERE toLower(e.name) CONTAINS toLower($query)
                       OR toLower(c.text) CONTAINS toLower($query)
                    RETURN DISTINCT
                        c.text AS chunk,
                        d.title AS document,
                        d.created_at AS timestamp,
                        collect(e.name) AS entities
                    ORDER BY d.created_at DESC
                    LIMIT 10
                """, start=start_date, end=end_date, query=query)
            else:
                result = session.run("""
                    MATCH (d:Document)-[:HAS_CHUNK]->(c:Chunk)
                    MATCH (e:Entity)-[:MENTIONED_IN]->(c)
                    WHERE toLower(e.name) CONTAINS toLower($query)
                       OR toLower(c.text) CONTAINS toLower($query)
                    RETURN DISTINCT
                        c.text AS chunk,
                        d.title AS document,
                        d.created_at AS timestamp,
                        collect(e.name) AS entities
                    ORDER BY d.created_at DESC
                    LIMIT 10
                """, query=query)

            return [dict(r) for r in result]

    def get_entity_timeline(self, entity_name: str) -> List[Dict]:
        """엔티티의 시간순 이벤트"""
        with db.session() as session:
            result = session.run("""
                MATCH (e:Entity)-[:MENTIONED_IN]->(c:Chunk)<-[:HAS_CHUNK]-(d:Document)
                WHERE toLower(e.name) = toLower($name)
                RETURN
                    d.title AS document,
                    d.created_at AS timestamp,
                    c.text AS context
                ORDER BY d.created_at
            """, name=entity_name)

            return [dict(r) for r in result]
```

---

## 평가 및 모니터링

### RAG 품질 평가

```python
# evaluator.py
from typing import List, Dict
from openai import OpenAI
from config import settings

client = OpenAI(api_key=settings.openai_api_key)

class RAGEvaluator:
    def evaluate(self, query: str, answer: str,
                 context: str, ground_truth: str = None) -> Dict:
        """RAG 응답 품질 평가"""

        scores = {}

        # 1. 충실도 (Faithfulness) - 답변이 컨텍스트에 기반하는지
        scores["faithfulness"] = self._evaluate_faithfulness(answer, context)

        # 2. 관련성 (Relevance) - 답변이 질문에 관련되는지
        scores["relevance"] = self._evaluate_relevance(query, answer)

        # 3. 일관성 (Coherence) - 답변이 논리적인지
        scores["coherence"] = self._evaluate_coherence(answer)

        # 4. 정확도 (Accuracy) - Ground truth가 있는 경우
        if ground_truth:
            scores["accuracy"] = self._evaluate_accuracy(answer, ground_truth)

        scores["overall"] = sum(scores.values()) / len(scores)

        return scores

    def _evaluate_faithfulness(self, answer: str, context: str) -> float:
        prompt = f"""답변이 주어진 컨텍스트에만 기반하는지 평가하세요.
컨텍스트에 없는 정보를 포함하면 낮은 점수를 주세요.

컨텍스트: {context[:2000]}
답변: {answer}

0.0~1.0 사이의 점수만 반환하세요."""

        response = client.chat.completions.create(
            model=settings.llm_model,
            messages=[{"role": "user", "content": prompt}],
            max_tokens=10
        )

        try:
            return float(response.choices[0].message.content.strip())
        except:
            return 0.5

    def _evaluate_relevance(self, query: str, answer: str) -> float:
        prompt = f"""답변이 질문에 얼마나 관련되고 직접적으로 대답하는지 평가하세요.

질문: {query}
답변: {answer}

0.0~1.0 사이의 점수만 반환하세요."""

        response = client.chat.completions.create(
            model=settings.llm_model,
            messages=[{"role": "user", "content": prompt}],
            max_tokens=10
        )

        try:
            return float(response.choices[0].message.content.strip())
        except:
            return 0.5

    def _evaluate_coherence(self, answer: str) -> float:
        prompt = f"""답변의 논리적 일관성과 명확성을 평가하세요.

답변: {answer}

0.0~1.0 사이의 점수만 반환하세요."""

        response = client.chat.completions.create(
            model=settings.llm_model,
            messages=[{"role": "user", "content": prompt}],
            max_tokens=10
        )

        try:
            return float(response.choices[0].message.content.strip())
        except:
            return 0.5

    def _evaluate_accuracy(self, answer: str, ground_truth: str) -> float:
        prompt = f"""답변이 정답과 얼마나 일치하는지 평가하세요.
의미적 유사성을 기준으로 평가하세요.

정답: {ground_truth}
답변: {answer}

0.0~1.0 사이의 점수만 반환하세요."""

        response = client.chat.completions.create(
            model=settings.llm_model,
            messages=[{"role": "user", "content": prompt}],
            max_tokens=10
        )

        try:
            return float(response.choices[0].message.content.strip())
        except:
            return 0.5
```

---

## 운영 가이드

### 그래프 최적화

```cypher
// 인덱스 상태 확인
SHOW INDEXES

// 느린 쿼리 분석
PROFILE MATCH (e:Entity)-[:MENTIONED_IN]->(c:Chunk)
WHERE e.name = 'Python'
RETURN c.text

// 통계 업데이트
CALL db.stats.retrieve('GRAPH COUNTS')

// 메모리 사용량 확인
CALL dbms.listPools()
```

### 모니터링 대시보드

```python
# monitoring.py
from prometheus_client import Counter, Histogram, Gauge, start_http_server

# 메트릭스 정의
query_counter = Counter('rag_queries_total', 'Total RAG queries')
query_latency = Histogram('rag_query_latency_seconds', 'Query latency')
graph_nodes = Gauge('graph_nodes_total', 'Total nodes in graph')
graph_relationships = Gauge('graph_relationships_total', 'Total relationships')

def update_graph_metrics():
    """그래프 메트릭스 업데이트"""
    from database import db

    with db.session() as session:
        result = session.run("""
            MATCH (n) WITH count(n) AS nodes
            MATCH ()-[r]->()
            RETURN nodes, count(r) AS rels
        """)
        record = result.single()

        graph_nodes.set(record["nodes"])
        graph_relationships.set(record["rels"])

# Prometheus 서버 시작
start_http_server(8001)
```

---

## 다음 단계

!!! success "Knowledge Graph RAG 완료!"
    [네트워크 인프라 관리](network-infrastructure.md)에서 IT 인프라 관리 시스템을 구현해보세요.
