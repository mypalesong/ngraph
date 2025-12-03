# 온톨로지와 스키마

## 온톨로지란?

**온톨로지(Ontology)**는 특정 도메인의 개념과 그들 간의 관계를 형식적으로 정의한 것입니다.

```
온톨로지 = 어휘(Vocabulary) + 의미(Semantics) + 제약조건(Constraints)
```

## 온톨로지 구성 요소

### 1. 클래스 (Classes)

엔티티의 유형을 정의:

```turtle
@prefix ex: <http://example.org/> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .

ex:Person a rdfs:Class ;
    rdfs:label "사람" ;
    rdfs:comment "인간 개체를 나타내는 클래스" .

ex:Employee a rdfs:Class ;
    rdfs:subClassOf ex:Person ;
    rdfs:label "직원" .
```

### 2. 속성 (Properties)

클래스 간의 관계나 데이터 값을 정의:

```turtle
# Object Property (엔티티 간 관계)
ex:worksAt a owl:ObjectProperty ;
    rdfs:domain ex:Person ;
    rdfs:range ex:Organization .

# Datatype Property (리터럴 값)
ex:birthDate a owl:DatatypeProperty ;
    rdfs:domain ex:Person ;
    rdfs:range xsd:date .
```

### 3. 제약조건 (Constraints)

```turtle
# 카디널리티 제약
ex:Person a owl:Class ;
    rdfs:subClassOf [
        a owl:Restriction ;
        owl:onProperty ex:hasMother ;
        owl:cardinality 1
    ] .

# 값 제약
ex:Adult a owl:Class ;
    owl:equivalentClass [
        a owl:Restriction ;
        owl:onProperty ex:age ;
        owl:someValuesFrom [
            a rdfs:Datatype ;
            owl:onDatatype xsd:integer ;
            owl:withRestrictions ([xsd:minInclusive 18])
        ]
    ] .
```

## 표준 온톨로지

### FOAF (Friend of a Friend)

```turtle
@prefix foaf: <http://xmlns.com/foaf/0.1/> .

ex:홍길동 a foaf:Person ;
    foaf:name "홍길동" ;
    foaf:mbox <mailto:hong@example.com> ;
    foaf:knows ex:이영희 .
```

### Schema.org

```json-ld
{
  "@context": "https://schema.org",
  "@type": "Person",
  "name": "홍길동",
  "jobTitle": "소프트웨어 엔지니어",
  "worksFor": {
    "@type": "Organization",
    "name": "테크회사"
  }
}
```

### Dublin Core

```turtle
@prefix dc: <http://purl.org/dc/elements/1.1/> .

ex:논문1 a ex:Paper ;
    dc:title "지식 그래프 연구" ;
    dc:creator ex:홍길동 ;
    dc:date "2024-01-15" .
```

## Property Graph 스키마

Neo4j에서의 스키마 정의:

```cypher
// 제약조건 생성
CREATE CONSTRAINT person_name IF NOT EXISTS
FOR (p:Person) REQUIRE p.name IS NOT NULL;

CREATE CONSTRAINT company_unique_name IF NOT EXISTS
FOR (c:Company) REQUIRE c.name IS UNIQUE;

// 인덱스 생성
CREATE INDEX person_age IF NOT EXISTS
FOR (p:Person) ON (p.age);

// 관계 타입 정의 (문서화 목적)
/*
(:Person)-[:WORKS_AT]->(:Company)
(:Person)-[:KNOWS]->(:Person)
(:Person)-[:LIVES_IN]->(:City)
*/
```

## 온톨로지 설계 원칙

### 1. 명확성 (Clarity)

```turtle
# 좋은 예
ex:birthDate rdfs:comment "사람이 태어난 날짜 (ISO 8601 형식)" .

# 나쁜 예
ex:date1 a rdf:Property .  # 의미 불명확
```

### 2. 확장성 (Extensibility)

```turtle
# 상속을 통한 확장
ex:SoftwareEngineer rdfs:subClassOf ex:Engineer .
ex:Engineer rdfs:subClassOf ex:Employee .
ex:Employee rdfs:subClassOf ex:Person .
```

### 3. 재사용 (Reusability)

```turtle
# 기존 온톨로지 재사용
@prefix foaf: <http://xmlns.com/foaf/0.1/> .
@prefix schema: <https://schema.org/> .

ex:MyPerson rdfs:subClassOf foaf:Person, schema:Person .
```

## 온톨로지 도구

| 도구 | 설명 |
|------|------|
| Protégé | 온톨로지 편집기 (Stanford) |
| TopBraid Composer | 상용 온톨로지 도구 |
| WebVOWL | 온톨로지 시각화 |
| OOPS! | 온톨로지 검증 도구 |

## 다음 단계

- [추론과 규칙](reasoning-rules.md)에서 온톨로지 기반 추론을 학습하세요
