# 금융 서비스

## 개요

금융 분야에서 지식 그래프는 사기 탐지, 리스크 관리, 규제 준수에 활용됩니다.

## 주요 활용 사례

### 1. 사기 탐지

```cypher
// 의심스러운 거래 패턴 탐지
MATCH (a:Account)-[t:TRANSFER]->(b:Account)
WHERE t.amount > 10000000
AND t.timestamp > datetime() - duration('P1D')
MATCH path = (a)-[:TRANSFER*1..5]->(a)
RETURN path, reduce(sum=0, r in relationships(path) | sum + r.amount) AS totalAmount
```

### 2. 자금세탁 네트워크

```cypher
// 순환 거래 탐지
MATCH path = (start:Account)-[:TRANSFER*3..10]->(start)
WHERE all(r in relationships(path) WHERE r.amount > 5000000)
RETURN path
```

### 3. 기업 관계 분석

```cypher
// 복잡한 소유 구조 파악
MATCH path = (company:Company)-[:OWNS*]->(subsidiary:Company)
RETURN company.name, collect(subsidiary.name) AS subsidiaries
```

## Python 구현

```python
class FraudDetector:
    def __init__(self, driver):
        self.driver = driver

    def detect_circular_transfers(self, min_amount=1000000):
        query = """
        MATCH path = (a:Account)-[:TRANSFER*3..]->(a)
        WHERE all(r in relationships(path) WHERE r.amount >= $minAmount)
        RETURN a.id AS account,
               length(path) AS hops,
               reduce(s=0, r in relationships(path) | s + r.amount) AS total
        """
        with self.driver.session() as session:
            return list(session.run(query, minAmount=min_amount))
```

## 다음 단계

- [헬스케어](healthcare.md)에서 다른 산업 적용 사례를 확인하세요
