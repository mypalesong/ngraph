# Neo4j Browser 사용법

## 개요

**Neo4j Browser**는 Neo4j와 상호작용하기 위한 웹 기반 인터페이스입니다. Cypher 쿼리 실행, 그래프 시각화, 데이터 탐색을 지원합니다.

## 접속

```
http://localhost:7474
```

## 인터페이스 구성

```
┌─────────────────────────────────────────────────────────────┐
│  ⭐ Neo4j Browser                              [DB: neo4j]  │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │  $ MATCH (n) RETURN n LIMIT 25                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                            [▶ Run] [💾]    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│    ┌──────────────────────────────────────────────────┐    │
│    │                                                  │    │
│    │            [그래프 시각화 영역]                   │    │
│    │                                                  │    │
│    │      (●)───────(●)                              │    │
│    │       │         │                               │    │
│    │      (●)───────(●)                              │    │
│    │                                                  │    │
│    └──────────────────────────────────────────────────┘    │
│                                                             │
│  [Graph] [Table] [Text] [Code]              Rows: 25       │
└─────────────────────────────────────────────────────────────┘
```

## 기본 명령어

### 시스템 명령어 (`:` 접두사)

```cypher
:help              -- 도움말
:clear             -- 화면 지우기
:history           -- 명령어 히스토리
:server status     -- 서버 상태
:sysinfo           -- 시스템 정보
:dbs               -- 데이터베이스 목록
:use <database>    -- 데이터베이스 전환
```

### 스키마 탐색

```cypher
// 모든 레이블 보기
CALL db.labels()

// 모든 관계 타입 보기
CALL db.relationshipTypes()

// 모든 속성 키 보기
CALL db.propertyKeys()

// 스키마 시각화
CALL db.schema.visualization()
```

## 샘플 데이터 로드

### Movie Graph (내장)

```cypher
:play movie-graph
```

그 후 "Create" 버튼을 클릭하면 영화 데이터가 로드됩니다.

```cypher
// 로드된 데이터 확인
MATCH (n) RETURN labels(n)[0] AS label, count(*) AS count

// 영화 그래프 탐색
MATCH (m:Movie)<-[:ACTED_IN]-(a:Person)
WHERE m.title = 'The Matrix'
RETURN m, a
```

### Northwind (비즈니스 데이터)

```cypher
:play northwind-graph
```

### 커스텀 데이터 생성

```cypher
// 간단한 소셜 네트워크
CREATE (alice:Person {name: 'Alice', age: 30})
CREATE (bob:Person {name: 'Bob', age: 28})
CREATE (charlie:Person {name: 'Charlie', age: 35})
CREATE (techCorp:Company {name: 'TechCorp', industry: 'IT'})

CREATE (alice)-[:FRIEND {since: 2020}]->(bob)
CREATE (bob)-[:FRIEND {since: 2019}]->(charlie)
CREATE (alice)-[:WORKS_AT {role: 'Engineer'}]->(techCorp)
CREATE (bob)-[:WORKS_AT {role: 'Designer'}]->(techCorp)

RETURN alice, bob, charlie, techCorp
```

## 시각화 기능

### 그래프 뷰 조작

| 동작 | 설명 |
|------|------|
| 드래그 | 노드 이동 |
| 더블클릭 | 노드 확장 (관련 노드 표시) |
| 휠 | 확대/축소 |
| 우클릭 | 컨텍스트 메뉴 |

### 노드 스타일링

```
1. 노드 클릭
2. 하단 패널에서 속성 확인
3. 색상, 크기, 캡션 변경
   - Caption: 표시할 속성 선택
   - Color: 노드 색상
   - Size: 노드 크기
```

### Grass 스타일시트

```css
/* 브라우저 설정에서 편집 가능 */
node {
  diameter: 50px;
  color: #68BDF6;
  border-color: #5CA8DB;
  border-width: 2px;
  text-color-internal: #FFFFFF;
  font-size: 10px;
}

node.Person {
  color: #F79767;
  border-color: #EB7D51;
  caption: '{name}';
}

node.Movie {
  color: #6DCE9E;
  border-color: #60B58B;
  caption: '{title}';
}

relationship {
  color: #A5ABB6;
  shaft-width: 1px;
  font-size: 8px;
  text-color-external: #000000;
}
```

## 유용한 쿼리 예시

### 데이터 분석

```cypher
// 노드/관계 수
MATCH (n) RETURN count(n) AS nodes
UNION ALL
MATCH ()-[r]->() RETURN count(r) AS relationships

// 레이블별 노드 수
MATCH (n)
RETURN labels(n)[0] AS label, count(*) AS count
ORDER BY count DESC

// 관계 타입별 수
MATCH ()-[r]->()
RETURN type(r) AS type, count(*) AS count
ORDER BY count DESC
```

### 그래프 탐색

```cypher
// 특정 노드의 모든 연결
MATCH (n {name: 'Alice'})-[r]-(connected)
RETURN n, r, connected

// 2단계 이내 연결
MATCH path = (n {name: 'Alice'})-[*1..2]-(connected)
RETURN path

// 최단 경로
MATCH path = shortestPath(
  (a:Person {name: 'Alice'})-[*]-(b:Person {name: 'Charlie'})
)
RETURN path
```

### 데이터 정리

```cypher
// 모든 데이터 삭제 (주의!)
MATCH (n) DETACH DELETE n

// 특정 레이블 삭제
MATCH (n:Test) DETACH DELETE n

// 고아 노드 삭제 (관계 없는 노드)
MATCH (n)
WHERE NOT (n)--()
DELETE n
```

## 결과 내보내기

### CSV 내보내기

```cypher
// 테이블 뷰에서 다운로드 버튼 클릭
MATCH (p:Person)
RETURN p.name, p.age, p.email
```

### JSON 내보내기

```cypher
// Code 뷰에서 JSON 복사
MATCH (p:Person)-[:WORKS_AT]->(c:Company)
RETURN p {.name, .age, company: c.name}
```

## 키보드 단축키

| 단축키 | 동작 |
|--------|------|
| `Ctrl + Enter` | 쿼리 실행 |
| `Ctrl + Up/Down` | 히스토리 탐색 |
| `/` | 명령어 팔레트 |
| `Esc` | 편집기 포커스 해제 |

## 다음 단계

!!! tip "실습"
    [Cypher 기초](cypher-basics.md)에서 쿼리 언어를 체계적으로 학습하세요.
