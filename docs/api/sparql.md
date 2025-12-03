# SPARQL

## 개요

SPARQL은 RDF 데이터를 쿼리하기 위한 W3C 표준 언어입니다.

## 기본 문법

### SELECT

```sparql
PREFIX foaf: <http://xmlns.com/foaf/0.1/>

SELECT ?name ?email
WHERE {
  ?person a foaf:Person .
  ?person foaf:name ?name .
  OPTIONAL { ?person foaf:mbox ?email }
}
ORDER BY ?name
LIMIT 10
```

### CONSTRUCT

```sparql
CONSTRUCT {
  ?person foaf:knows ?friend .
}
WHERE {
  ?person foaf:name "홍길동" .
  ?person foaf:knows ?friend .
}
```

### ASK

```sparql
ASK {
  ?person foaf:name "홍길동" .
  ?person foaf:knows ?someone .
}
```

## Python 사용

```python
from SPARQLWrapper import SPARQLWrapper, JSON

sparql = SPARQLWrapper("http://dbpedia.org/sparql")
sparql.setQuery("""
    SELECT ?person ?name WHERE {
        ?person a dbo:Person .
        ?person foaf:name ?name .
        FILTER(LANG(?name) = 'ko')
    }
    LIMIT 10
""")
sparql.setReturnFormat(JSON)
results = sparql.query().convert()
```

## 다음 단계

- [Cypher](cypher.md)와 비교하세요
