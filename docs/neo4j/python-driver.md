# Python 연동

## 드라이버 설치

```bash
pip install neo4j
```

## 기본 연결

### 연결 생성

```python
from neo4j import GraphDatabase

# 기본 연결
driver = GraphDatabase.driver(
    "bolt://localhost:7687",
    auth=("neo4j", "password")
)

# 연결 확인
driver.verify_connectivity()
print("연결 성공!")

# 종료
driver.close()
```

### Context Manager 사용

```python
from neo4j import GraphDatabase

URI = "bolt://localhost:7687"
AUTH = ("neo4j", "password")

with GraphDatabase.driver(URI, auth=AUTH) as driver:
    driver.verify_connectivity()
    # 작업 수행
```

### AuraDB 연결

```python
# 클라우드 연결 (암호화 필수)
driver = GraphDatabase.driver(
    "neo4j+s://xxxxxxxx.databases.neo4j.io",
    auth=("neo4j", "your_password")
)
```

## 세션과 트랜잭션

### 세션 생성

```python
with driver.session() as session:
    result = session.run("MATCH (n) RETURN count(n)")
    count = result.single()[0]
    print(f"노드 수: {count}")
```

### 데이터베이스 지정

```python
with driver.session(database="mydb") as session:
    # mydb 데이터베이스에서 작업
    pass
```

## 쿼리 실행

### 기본 실행

```python
def get_all_persons(driver):
    with driver.session() as session:
        result = session.run("MATCH (p:Person) RETURN p.name, p.age")
        for record in result:
            print(f"{record['p.name']}: {record['p.age']}세")
```

### 파라미터 바인딩

```python
def find_person(driver, name):
    with driver.session() as session:
        result = session.run(
            "MATCH (p:Person {name: $name}) RETURN p",
            name=name  # 파라미터 바인딩
        )
        return result.single()

# 딕셔너리로 전달
def find_persons_by_age(driver, min_age, max_age):
    params = {"min_age": min_age, "max_age": max_age}
    with driver.session() as session:
        result = session.run(
            "MATCH (p:Person) WHERE p.age >= $min_age AND p.age <= $max_age RETURN p",
            **params
        )
        return list(result)
```

## 트랜잭션 함수

### 읽기 트랜잭션

```python
def get_friends(tx, name):
    result = tx.run("""
        MATCH (p:Person {name: $name})-[:FRIEND]->(friend)
        RETURN friend.name AS name
    """, name=name)
    return [record["name"] for record in result]

with driver.session() as session:
    friends = session.execute_read(get_friends, "Alice")
    print(f"친구들: {friends}")
```

### 쓰기 트랜잭션

```python
def create_person(tx, name, age):
    result = tx.run("""
        CREATE (p:Person {name: $name, age: $age})
        RETURN p
    """, name=name, age=age)
    return result.single()["p"]

with driver.session() as session:
    person = session.execute_write(create_person, "홍길동", 30)
    print(f"생성됨: {person}")
```

### 재시도 로직

```python
from neo4j import GraphDatabase
from neo4j.exceptions import ServiceUnavailable, TransientError

def robust_write(tx, query, **params):
    return tx.run(query, **params)

with driver.session() as session:
    # execute_write는 자동으로 일시적 오류 시 재시도
    session.execute_write(
        lambda tx: tx.run("CREATE (n:Test) RETURN n")
    )
```

## 결과 처리

### Record 접근

```python
result = session.run("MATCH (p:Person) RETURN p.name AS name, p.age AS age")

for record in result:
    # 딕셔너리 스타일
    print(record["name"])

    # 인덱스 스타일
    print(record[0])

    # values() 메서드
    name, age = record.values()

    # data() 메서드 (딕셔너리 반환)
    data = record.data()
    print(data)  # {'name': 'Alice', 'age': 30}
```

### 노드/관계 객체

```python
result = session.run("MATCH (p:Person)-[r:FRIEND]->(f) RETURN p, r, f")

for record in result:
    person = record["p"]
    relation = record["r"]
    friend = record["f"]

    # 노드 속성
    print(person["name"])       # 속성 접근
    print(person.labels)        # 레이블 (frozenset)
    print(person.element_id)    # 고유 ID

    # 관계 속성
    print(relation.type)        # 관계 타입
    print(relation["since"])    # 관계 속성
```

### 결과 소비

```python
# 단일 결과
result = session.run("MATCH (p:Person {name: 'Alice'}) RETURN p")
record = result.single()  # 하나만 기대, 없거나 여러개면 예외

# 첫 번째만
record = result.peek()

# 모든 결과를 리스트로
records = list(result)
records = result.values()  # 값만 리스트로
records = result.data()    # 딕셔너리 리스트로

# 요약 정보
summary = result.consume()
print(f"쿼리 실행 시간: {summary.result_available_after}ms")
print(f"노드 생성: {summary.counters.nodes_created}")
```

## 완전한 예제

### Repository 패턴

```python
from neo4j import GraphDatabase
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class Person:
    name: str
    age: int
    email: Optional[str] = None

class PersonRepository:
    def __init__(self, driver):
        self.driver = driver

    def create(self, person: Person) -> Person:
        def _create(tx, p):
            result = tx.run("""
                CREATE (person:Person {
                    name: $name,
                    age: $age,
                    email: $email
                })
                RETURN person
            """, name=p.name, age=p.age, email=p.email)
            return result.single()["person"]

        with self.driver.session() as session:
            node = session.execute_write(_create, person)
            return Person(
                name=node["name"],
                age=node["age"],
                email=node["email"]
            )

    def find_by_name(self, name: str) -> Optional[Person]:
        def _find(tx, name):
            result = tx.run("""
                MATCH (p:Person {name: $name})
                RETURN p
            """, name=name)
            record = result.single()
            return record["p"] if record else None

        with self.driver.session() as session:
            node = session.execute_read(_find, name)
            if node:
                return Person(
                    name=node["name"],
                    age=node["age"],
                    email=node.get("email")
                )
            return None

    def find_friends(self, name: str) -> List[Person]:
        def _find(tx, name):
            result = tx.run("""
                MATCH (p:Person {name: $name})-[:FRIEND]->(friend:Person)
                RETURN friend
            """, name=name)
            return [record["friend"] for record in result]

        with self.driver.session() as session:
            nodes = session.execute_read(_find, name)
            return [
                Person(name=n["name"], age=n["age"], email=n.get("email"))
                for n in nodes
            ]

    def add_friend(self, name1: str, name2: str) -> bool:
        def _add(tx, n1, n2):
            result = tx.run("""
                MATCH (a:Person {name: $name1})
                MATCH (b:Person {name: $name2})
                MERGE (a)-[:FRIEND]->(b)
                RETURN a, b
            """, name1=n1, name2=n2)
            return result.single() is not None

        with self.driver.session() as session:
            return session.execute_write(_add, name1, name2)


# 사용 예시
if __name__ == "__main__":
    driver = GraphDatabase.driver("bolt://localhost:7687", auth=("neo4j", "password"))

    repo = PersonRepository(driver)

    # 생성
    alice = repo.create(Person("Alice", 28, "alice@example.com"))
    bob = repo.create(Person("Bob", 32))

    # 친구 추가
    repo.add_friend("Alice", "Bob")

    # 조회
    friends = repo.find_friends("Alice")
    print(f"Alice의 친구: {[f.name for f in friends]}")

    driver.close()
```

### 비동기 드라이버

```python
import asyncio
from neo4j import AsyncGraphDatabase

async def main():
    driver = AsyncGraphDatabase.driver(
        "bolt://localhost:7687",
        auth=("neo4j", "password")
    )

    async with driver.session() as session:
        result = await session.run("MATCH (n) RETURN count(n)")
        record = await result.single()
        print(f"노드 수: {record[0]}")

    await driver.close()

asyncio.run(main())
```

## 다음 단계

!!! success "Python 연동 완료!"
    [APOC 라이브러리](apoc.md)에서 강력한 확장 기능을 배워보세요.
