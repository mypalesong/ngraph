# GraphQL

## 개요

GraphQL을 사용하여 지식 그래프에 대한 유연한 API를 제공합니다.

## 스키마 정의

```graphql
type Person {
  id: ID!
  name: String!
  age: Int
  worksAt: Company
  friends: [Person!]!
}

type Company {
  id: ID!
  name: String!
  employees: [Person!]!
}

type Query {
  person(id: ID!): Person
  persons(name: String): [Person!]!
  company(id: ID!): Company
}
```

## 쿼리 예시

```graphql
query {
  person(id: "1") {
    name
    age
    worksAt {
      name
    }
    friends {
      name
    }
  }
}
```

## Python 구현

```python
from ariadne import QueryType, make_executable_schema
from ariadne.asgi import GraphQL

query = QueryType()

@query.field("person")
def resolve_person(_, info, id):
    with driver.session() as session:
        result = session.run(
            "MATCH (p:Person {id: $id}) RETURN p",
            id=id
        )
        return result.single()["p"]

schema = make_executable_schema(type_defs, query)
app = GraphQL(schema)
```

## 다음 단계

- [REST API](rest-api.md)에서 전통적인 API 설계를 확인하세요
