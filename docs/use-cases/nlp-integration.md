# NLP 통합

## 개요

자연어 처리(NLP)와 지식 그래프를 통합하여 텍스트에서 지식을 추출하고 질의응답 시스템을 구축합니다.

## 주요 활용

### 1. 엔티티 추출 (NER)

```python
import spacy

nlp = spacy.load("ko_core_news_lg")

def extract_entities(text):
    doc = nlp(text)
    entities = []
    for ent in doc.ents:
        entities.append({
            "text": ent.text,
            "label": ent.label_,
            "start": ent.start_char,
            "end": ent.end_char
        })
    return entities

# 사용
text = "삼성전자의 이재용 회장이 서울에서 기자회견을 열었다."
entities = extract_entities(text)
# [{"text": "삼성전자", "label": "ORG"}, {"text": "이재용", "label": "PERSON"}, ...]
```

### 2. 관계 추출

```python
from transformers import pipeline

re_pipeline = pipeline("text2text-generation", model="Babelscape/rebel-large")

def extract_relations(text):
    result = re_pipeline(text)
    # 트리플 형태로 파싱
    return parse_triplets(result[0]['generated_text'])

# 결과: [(이재용, 소속, 삼성전자), (이재용, 직책, 회장)]
```

### 3. 지식 그래프 구축

```python
def text_to_knowledge_graph(text, neo4j_driver):
    entities = extract_entities(text)
    relations = extract_relations(text)

    with neo4j_driver.session() as session:
        # 엔티티 생성
        for entity in entities:
            session.run(
                f"MERGE (n:{entity['label']} {{name: $name}})",
                name=entity['text']
            )

        # 관계 생성
        for subj, pred, obj in relations:
            session.run("""
                MATCH (a {name: $subj}), (b {name: $obj})
                MERGE (a)-[:$pred]->(b)
                """, subj=subj, obj=obj, pred=pred)
```

## 질의응답 시스템

```python
def answer_question(question, kg):
    # 1. 질문에서 엔티티 추출
    entities = extract_entities(question)

    # 2. 질문 의도 파악
    intent = classify_intent(question)

    # 3. Cypher 쿼리 생성
    if intent == "find_relation":
        query = generate_relation_query(entities)
    elif intent == "find_path":
        query = generate_path_query(entities)

    # 4. 결과 반환
    return kg.run(query)
```

## 다음 단계

- [LLM 통합](../advanced/llm-integration.md)에서 고급 NLP 활용법을 확인하세요
