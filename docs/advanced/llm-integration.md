# LLM 통합

## 개요

대규모 언어 모델(LLM)과 지식 그래프를 통합하여 환각을 줄이고 사실 기반 응답을 생성합니다.

## RAG + 지식 그래프

```python
from langchain.chains import GraphCypherQAChain
from langchain_community.graphs import Neo4jGraph
from langchain_openai import ChatOpenAI

graph = Neo4jGraph(
    url="bolt://localhost:7687",
    username="neo4j",
    password="password"
)

llm = ChatOpenAI(model="gpt-4", temperature=0)

chain = GraphCypherQAChain.from_llm(
    llm=llm,
    graph=graph,
    verbose=True
)

# 자연어 질문 -> Cypher -> 답변
response = chain.invoke({"query": "홍길동이 다니는 회사는?"})
```

## 지식 그래프 기반 프롬프트

```python
def kg_augmented_prompt(question, kg):
    # 1. 질문에서 엔티티 추출
    entities = extract_entities(question)

    # 2. 관련 지식 검색
    context = []
    for entity in entities:
        facts = kg.get_facts(entity)
        context.extend(facts)

    # 3. 프롬프트 구성
    prompt = f"""
다음 지식을 참고하여 질문에 답하세요:

지식:
{chr(10).join(context)}

질문: {question}
"""
    return prompt
```

## GraphRAG

```python
# Microsoft GraphRAG 스타일
from graphrag import GraphRAG

rag = GraphRAG(
    graph_store=neo4j_graph,
    llm=llm,
    embedding_model=embeddings
)

# 커뮤니티 기반 요약 검색
response = rag.query(
    "회사의 주요 프로젝트는?",
    search_type="global"  # local 또는 global
)
```

## 장점

- **환각 감소**: 사실 기반 응답
- **추적 가능성**: 응답의 근거 제시
- **최신 정보**: 실시간 지식 업데이트

## 다음 단계

- [분산 처리](distributed-processing.md)에서 대규모 처리를 학습하세요
