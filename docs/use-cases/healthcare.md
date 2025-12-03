# 헬스케어

## 개요

의료 분야에서 지식 그래프는 질병-증상-치료 관계 모델링, 약물 상호작용 분석, 임상 의사결정 지원에 활용됩니다.

## 핵심 스키마

```cypher
// 질병
CREATE (d:Disease {name: '당뇨병', icd10: 'E11'})

// 증상
CREATE (s:Symptom {name: '다뇨', severity: 'moderate'})

// 약물
CREATE (m:Medication {name: '메트포르민', type: 'oral'})

// 관계
CREATE (d)-[:HAS_SYMPTOM {frequency: 0.8}]->(s)
CREATE (m)-[:TREATS]->(d)
CREATE (m)-[:CONTRAINDICATED_WITH]->(otherMed:Medication)
```

## 주요 활용

### 1. 진단 지원

```cypher
// 증상으로 가능한 질병 찾기
MATCH (s:Symptom)<-[:HAS_SYMPTOM]-(d:Disease)
WHERE s.name IN ['다뇨', '다갈', '체중감소']
WITH d, count(s) AS matchCount
ORDER BY matchCount DESC
RETURN d.name, matchCount
```

### 2. 약물 상호작용

```cypher
// 처방된 약물 간 상호작용 확인
MATCH (m1:Medication)-[i:INTERACTS_WITH]->(m2:Medication)
WHERE m1.name IN $prescribedMeds AND m2.name IN $prescribedMeds
RETURN m1.name, m2.name, i.severity, i.description
```

### 3. 치료 경로 분석

```cypher
// 성공적인 치료 패턴 발견
MATCH (p:Patient)-[:DIAGNOSED_WITH]->(d:Disease)
MATCH (p)-[:RECEIVED]->(t:Treatment)-[:FOR]->(d)
WHERE t.outcome = 'success'
RETURN t.protocol, count(*) AS successCount
ORDER BY successCount DESC
```

## Python 구현

```python
class ClinicalKG:
    def __init__(self, driver):
        self.driver = driver

    def check_drug_interactions(self, medications: list):
        query = """
        MATCH (m1:Medication)-[i:INTERACTS_WITH]->(m2:Medication)
        WHERE m1.name IN $meds AND m2.name IN $meds
        RETURN m1.name AS drug1, m2.name AS drug2,
               i.severity AS severity, i.description AS warning
        """
        with self.driver.session() as session:
            return list(session.run(query, meds=medications))

    def suggest_diagnosis(self, symptoms: list):
        query = """
        MATCH (s:Symptom)<-[r:HAS_SYMPTOM]-(d:Disease)
        WHERE s.name IN $symptoms
        WITH d, sum(r.frequency) AS score, count(s) AS matches
        ORDER BY score DESC
        RETURN d.name, score, matches
        LIMIT 5
        """
        with self.driver.session() as session:
            return list(session.run(query, symptoms=symptoms))
```

## 다음 단계

- [구현 가이드](../implementation/architecture.md)에서 시스템 설계를 학습하세요
