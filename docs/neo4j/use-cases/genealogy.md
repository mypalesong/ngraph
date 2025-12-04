# 계보/족보 시스템 (Genealogy System)

## 개요

**계보/족보 시스템**은 가족 관계, 조직 계층, 역사적 인물 관계 등을 그래프로 모델링하여 조상 추적, 관계 분석, 유전적 특성 추적을 수행합니다.

### 활용 분야

| 분야 | 활용 사례 |
|------|----------|
| 가계도 | 가문 족보, 조상 탐색, 친척 관계 |
| 유전학 | 유전 질환 추적, DNA 매칭 |
| 역사 연구 | 왕가 계보, 역사적 인물 관계 |
| 기업 조직 | 조직도, 보고 체계, 인수합병 이력 |
| 종교/문화 | 종단 계보, 사제 계승 |

---

## 데이터 모델

### 가족 관계 스키마

```
┌──────────────────────────────────────────────────────────────┐
│                      FAMILY GRAPH                             │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│   ┌────────┐  MARRIED_TO   ┌────────┐                        │
│   │ Person │◄─────────────►│ Person │                        │
│   └────────┘               └────────┘                        │
│       │                         │                            │
│       │ PARENT_OF              │ PARENT_OF                   │
│       ▼                        ▼                             │
│   ┌────────┐              ┌────────┐                         │
│   │ Person │              │ Person │                         │
│   └────────┘              └────────┘                         │
│       │                                                       │
│       │ BELONGS_TO                                           │
│       ▼                                                      │
│   ┌────────┐  BRANCH_OF   ┌────────┐                        │
│   │  Clan  │◄────────────│ Family │                         │
│   └────────┘              └────────┘                         │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Cypher 스키마 정의

```cypher
// 제약조건
CREATE CONSTRAINT person_id FOR (p:Person) REQUIRE p.id IS UNIQUE;
CREATE CONSTRAINT family_id FOR (f:Family) REQUIRE f.id IS UNIQUE;
CREATE CONSTRAINT clan_id FOR (c:Clan) REQUIRE c.id IS UNIQUE;
CREATE CONSTRAINT event_id FOR (e:Event) REQUIRE e.id IS UNIQUE;

// 인덱스
CREATE INDEX person_name FOR (p:Person) ON (p.name);
CREATE INDEX person_birth FOR (p:Person) ON (p.birth_date);
CREATE INDEX person_generation FOR (p:Person) ON (p.generation);
CREATE INDEX family_name FOR (f:Family) ON (f.name);

// 전문 검색
CREATE FULLTEXT INDEX person_search FOR (p:Person)
ON EACH [p.name, p.birth_name, p.alias];
```

### 관계 유형

```cypher
// 가족 관계
(:Person)-[:PARENT_OF]->(:Person)
(:Person)-[:CHILD_OF]->(:Person)
(:Person)-[:MARRIED_TO {date, location, divorce_date?}]->(:Person)
(:Person)-[:SIBLING_OF]->(:Person)

// 조직 소속
(:Person)-[:BELONGS_TO]->(:Family)
(:Person)-[:MEMBER_OF {role, since}]->(:Clan)
(:Family)-[:BRANCH_OF]->(:Clan)

// 이벤트
(:Person)-[:PARTICIPATED_IN {role}]->(:Event)
(:Event)-[:OCCURRED_AT]->(:Location)

// 특성/유전
(:Person)-[:HAS_TRAIT]->(:Trait)
(:Person)-[:HAS_CONDITION]->(:MedicalCondition)
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
      - NEO4J_AUTH=neo4j/genealogy123
      - NEO4J_PLUGINS=["apoc", "graph-data-science"]
    volumes:
      - neo4j_data:/data

  genealogy-api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - NEO4J_URI=bolt://neo4j:7687
      - NEO4J_USER=neo4j
      - NEO4J_PASSWORD=genealogy123
    depends_on:
      - neo4j

volumes:
  neo4j_data:
```

---

## 구현

### 기본 설정

```python
# config.py
from pydantic_settings import BaseSettings
from enum import Enum
from typing import Optional

class Gender(str, Enum):
    MALE = "male"
    FEMALE = "female"
    OTHER = "other"

class RelationType(str, Enum):
    BIOLOGICAL = "biological"
    ADOPTED = "adopted"
    STEP = "step"
    FOSTER = "foster"

class Settings(BaseSettings):
    neo4j_uri: str = "bolt://localhost:7687"
    neo4j_user: str = "neo4j"
    neo4j_password: str = "genealogy123"

settings = Settings()
```

### 데이터베이스 연결

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

### 인물 관리 서비스

```python
# person_service.py
from typing import List, Dict, Any, Optional
from datetime import date
from database import db
from config import Gender, RelationType
import uuid

class PersonService:
    def create_person(self, name: str, gender: Gender,
                      birth_date: Optional[date] = None,
                      birth_place: Optional[str] = None,
                      death_date: Optional[date] = None,
                      generation: Optional[int] = None,
                      family_id: Optional[str] = None) -> Dict:
        """인물 생성"""
        person_id = str(uuid.uuid4())

        with db.session() as session:
            result = session.run("""
                CREATE (p:Person {
                    id: $person_id,
                    name: $name,
                    gender: $gender,
                    birth_date: $birth_date,
                    birth_place: $birth_place,
                    death_date: $death_date,
                    generation: $generation,
                    is_living: $death_date IS NULL,
                    created_at: datetime()
                })

                WITH p
                OPTIONAL MATCH (f:Family {id: $family_id})
                FOREACH (_ IN CASE WHEN f IS NOT NULL THEN [1] ELSE [] END |
                    CREATE (p)-[:BELONGS_TO]->(f)
                )

                RETURN p {.*} AS person
            """, person_id=person_id, name=name, gender=gender.value,
                 birth_date=str(birth_date) if birth_date else None,
                 birth_place=birth_place,
                 death_date=str(death_date) if death_date else None,
                 generation=generation, family_id=family_id)

            return result.single()["person"]

    def add_parent_relationship(self, child_id: str, parent_id: str,
                                 relation_type: RelationType = RelationType.BIOLOGICAL) -> Dict:
        """부모-자녀 관계 설정"""
        with db.session() as session:
            result = session.run("""
                MATCH (child:Person {id: $child_id})
                MATCH (parent:Person {id: $parent_id})

                // 부모 -> 자녀 관계
                CREATE (parent)-[r:PARENT_OF {
                    type: $relation_type,
                    created_at: datetime()
                }]->(child)

                // 자녀 -> 부모 관계
                CREATE (child)-[:CHILD_OF {
                    type: $relation_type
                }]->(parent)

                // 세대 자동 계산
                SET child.generation = COALESCE(parent.generation, 0) + 1

                RETURN parent.name AS parent, child.name AS child, r.type AS relationship
            """, child_id=child_id, parent_id=parent_id,
                 relation_type=relation_type.value)

            return dict(result.single())

    def add_marriage(self, person1_id: str, person2_id: str,
                     marriage_date: Optional[date] = None,
                     location: Optional[str] = None) -> Dict:
        """결혼 관계 설정"""
        with db.session() as session:
            result = session.run("""
                MATCH (p1:Person {id: $person1_id})
                MATCH (p2:Person {id: $person2_id})

                CREATE (p1)-[m:MARRIED_TO {
                    date: $marriage_date,
                    location: $location,
                    is_current: true,
                    created_at: datetime()
                }]->(p2)

                CREATE (p2)-[:MARRIED_TO {
                    date: $marriage_date,
                    location: $location,
                    is_current: true
                }]->(p1)

                RETURN p1.name AS spouse1, p2.name AS spouse2, m {.*} AS marriage
            """, person1_id=person1_id, person2_id=person2_id,
                 marriage_date=str(marriage_date) if marriage_date else None,
                 location=location)

            return dict(result.single())

    def add_sibling_relationship(self, person1_id: str, person2_id: str) -> Dict:
        """형제자매 관계 설정 (자동 추론도 가능)"""
        with db.session() as session:
            result = session.run("""
                MATCH (p1:Person {id: $person1_id})
                MATCH (p2:Person {id: $person2_id})

                CREATE (p1)-[:SIBLING_OF]->(p2)
                CREATE (p2)-[:SIBLING_OF]->(p1)

                RETURN p1.name AS sibling1, p2.name AS sibling2
            """, person1_id=person1_id, person2_id=person2_id)

            return dict(result.single())

    def update_person(self, person_id: str, updates: Dict) -> Dict:
        """인물 정보 업데이트"""
        set_clauses = []
        for key in updates:
            if key not in ['id', 'created_at']:
                set_clauses.append(f"p.{key} = ${key}")

        if not set_clauses:
            return {}

        query = f"""
            MATCH (p:Person {{id: $person_id}})
            SET {', '.join(set_clauses)}, p.updated_at = datetime()
            RETURN p {{.*}} AS person
        """

        with db.session() as session:
            result = session.run(query, person_id=person_id, **updates)
            return result.single()["person"]

    def get_person_with_family(self, person_id: str) -> Dict[str, Any]:
        """인물 및 가족 관계 조회"""
        with db.session() as session:
            result = session.run("""
                MATCH (p:Person {id: $person_id})

                // 부모
                OPTIONAL MATCH (p)-[:CHILD_OF]->(parent:Person)

                // 배우자
                OPTIONAL MATCH (p)-[m:MARRIED_TO]->(spouse:Person)
                WHERE m.is_current = true

                // 자녀
                OPTIONAL MATCH (p)-[:PARENT_OF]->(child:Person)

                // 형제자매
                OPTIONAL MATCH (p)-[:SIBLING_OF]-(sibling:Person)

                RETURN
                    p {.*} AS person,
                    collect(DISTINCT parent {.*}) AS parents,
                    collect(DISTINCT spouse {.*}) AS spouses,
                    collect(DISTINCT child {.*, age: duration.between(date(child.birth_date), date()).years}) AS children,
                    collect(DISTINCT sibling {.*}) AS siblings
            """, person_id=person_id)

            record = result.single()
            return {
                "person": record["person"],
                "parents": record["parents"],
                "spouses": record["spouses"],
                "children": record["children"],
                "siblings": record["siblings"]
            }
```

### 가문/씨족 관리

```python
# clan_service.py
from typing import List, Dict, Any, Optional
from database import db
import uuid

class ClanService:
    def create_clan(self, name: str, origin: str,
                    founding_date: Optional[str] = None,
                    description: Optional[str] = None) -> Dict:
        """씨족/가문 생성"""
        clan_id = str(uuid.uuid4())

        with db.session() as session:
            result = session.run("""
                CREATE (c:Clan {
                    id: $clan_id,
                    name: $name,
                    origin: $origin,
                    founding_date: $founding_date,
                    description: $description,
                    created_at: datetime()
                })
                RETURN c {.*} AS clan
            """, clan_id=clan_id, name=name, origin=origin,
                 founding_date=founding_date, description=description)

            return result.single()["clan"]

    def create_family_branch(self, family_name: str, clan_id: str,
                              branch_ancestor_id: str) -> Dict:
        """분파/분가 생성"""
        family_id = str(uuid.uuid4())

        with db.session() as session:
            result = session.run("""
                MATCH (c:Clan {id: $clan_id})
                MATCH (ancestor:Person {id: $branch_ancestor_id})

                CREATE (f:Family {
                    id: $family_id,
                    name: $family_name,
                    created_at: datetime()
                })

                CREATE (f)-[:BRANCH_OF {
                    ancestor_id: $branch_ancestor_id,
                    established_at: datetime()
                }]->(c)

                CREATE (ancestor)-[:FOUNDER_OF]->(f)

                RETURN f {.*} AS family, c.name AS clan, ancestor.name AS founder
            """, family_id=family_id, family_name=family_name,
                 clan_id=clan_id, branch_ancestor_id=branch_ancestor_id)

            return dict(result.single())

    def get_clan_statistics(self, clan_id: str) -> Dict[str, Any]:
        """씨족 통계"""
        with db.session() as session:
            result = session.run("""
                MATCH (c:Clan {id: $clan_id})
                OPTIONAL MATCH (f:Family)-[:BRANCH_OF]->(c)
                OPTIONAL MATCH (p:Person)-[:BELONGS_TO]->(f)

                WITH c,
                     count(DISTINCT f) AS family_count,
                     count(DISTINCT p) AS member_count,
                     collect(DISTINCT p) AS members

                UNWIND members AS m
                WITH c, family_count, member_count, members,
                     max(m.generation) AS max_generation,
                     count(CASE WHEN m.is_living = true THEN 1 END) AS living_count,
                     count(CASE WHEN m.gender = 'male' THEN 1 END) AS male_count,
                     count(CASE WHEN m.gender = 'female' THEN 1 END) AS female_count

                RETURN
                    c {.*} AS clan,
                    family_count AS branches,
                    member_count AS total_members,
                    max_generation AS generations,
                    living_count AS living_members,
                    male_count,
                    female_count
            """, clan_id=clan_id)

            return dict(result.single())
```

### 조상 탐색 서비스

```python
# ancestry_service.py
from typing import List, Dict, Any, Optional
from database import db

class AncestryService:
    def get_ancestors(self, person_id: str,
                      generations: int = 10) -> Dict[str, Any]:
        """조상 계보 조회"""
        with db.session() as session:
            result = session.run("""
                MATCH (p:Person {id: $person_id})
                MATCH path = (p)-[:CHILD_OF*1..$generations]->(ancestor:Person)

                WITH ancestor, length(path) AS generation_distance
                ORDER BY generation_distance

                RETURN collect({
                    person: ancestor {.*},
                    generation: generation_distance
                }) AS ancestors
            """, person_id=person_id, generations=generations)

            record = result.single()
            return {
                "person_id": person_id,
                "ancestors": record["ancestors"]
            }

    def get_descendants(self, person_id: str,
                        generations: int = 10) -> Dict[str, Any]:
        """후손 계보 조회"""
        with db.session() as session:
            result = session.run("""
                MATCH (p:Person {id: $person_id})
                MATCH path = (p)-[:PARENT_OF*1..$generations]->(descendant:Person)

                WITH descendant, length(path) AS generation_distance
                ORDER BY generation_distance, descendant.birth_date

                RETURN collect({
                    person: descendant {.*},
                    generation: generation_distance
                }) AS descendants
            """, person_id=person_id, generations=generations)

            record = result.single()
            return {
                "person_id": person_id,
                "descendants": record["descendants"]
            }

    def get_lineage(self, person_id: str) -> Dict[str, Any]:
        """직계 혈통 조회 (조상 -> 본인 -> 후손)"""
        with db.session() as session:
            result = session.run("""
                MATCH (p:Person {id: $person_id})

                // 직계 조상
                OPTIONAL MATCH ancestor_path = (p)-[:CHILD_OF*]->(ancestor:Person)
                WHERE NOT (ancestor)-[:CHILD_OF]->()  // 최고 조상

                // 직계 후손 (장자 우선)
                OPTIONAL MATCH descendant_path = (p)-[:PARENT_OF*]->(descendant:Person)

                WITH p, ancestor_path, descendant_path,
                     [n IN nodes(ancestor_path) | n] AS ancestors,
                     [n IN nodes(descendant_path) | n] AS descendants

                RETURN
                    p {.*} AS person,
                    CASE WHEN ancestors IS NOT NULL
                        THEN [a IN reverse(ancestors) WHERE a <> p | a {.*}]
                        ELSE []
                    END AS ancestors,
                    CASE WHEN descendants IS NOT NULL
                        THEN [d IN descendants WHERE d <> p | d {.*}]
                        ELSE []
                    END AS descendants
            """, person_id=person_id)

            record = result.single()
            return dict(record)

    def find_common_ancestor(self, person1_id: str,
                             person2_id: str) -> Dict[str, Any]:
        """공통 조상 찾기"""
        with db.session() as session:
            result = session.run("""
                MATCH (p1:Person {id: $person1_id})
                MATCH (p2:Person {id: $person2_id})

                // 각자의 조상 경로
                MATCH path1 = (p1)-[:CHILD_OF*0..20]->(ancestor:Person)
                MATCH path2 = (p2)-[:CHILD_OF*0..20]->(ancestor)

                WITH ancestor, length(path1) AS dist1, length(path2) AS dist2,
                     path1, path2
                ORDER BY dist1 + dist2

                WITH ancestor, dist1, dist2,
                     [n IN nodes(path1) | n.name] AS path1_names,
                     [n IN nodes(path2) | n.name] AS path2_names
                LIMIT 1

                RETURN
                    ancestor {.*} AS common_ancestor,
                    dist1 AS generations_from_person1,
                    dist2 AS generations_from_person2,
                    dist1 + dist2 AS total_distance,
                    path1_names AS path_from_person1,
                    path2_names AS path_from_person2
            """, person1_id=person1_id, person2_id=person2_id)

            record = result.single()
            if not record:
                return {"error": "No common ancestor found"}
            return dict(record)

    def calculate_relationship(self, person1_id: str,
                               person2_id: str) -> Dict[str, Any]:
        """관계 계산 (촌수)"""
        with db.session() as session:
            result = session.run("""
                MATCH (p1:Person {id: $person1_id})
                MATCH (p2:Person {id: $person2_id})

                // 최단 경로 찾기
                MATCH path = shortestPath(
                    (p1)-[:PARENT_OF|CHILD_OF|SIBLING_OF|MARRIED_TO*..20]-(p2)
                )

                WITH p1, p2, path,
                     length(path) AS distance,
                     [r IN relationships(path) | type(r)] AS rel_types

                // 촌수 계산 로직
                WITH p1, p2, path, distance, rel_types,
                     size([r IN rel_types WHERE r IN ['PARENT_OF', 'CHILD_OF']]) AS blood_steps,
                     'MARRIED_TO' IN rel_types AS includes_marriage

                RETURN
                    p1.name AS person1,
                    p2.name AS person2,
                    distance AS path_length,
                    blood_steps AS chon,  // 촌수
                    includes_marriage,
                    [n IN nodes(path) | n.name] AS relationship_path,
                    rel_types AS relationship_types
            """, person1_id=person1_id, person2_id=person2_id)

            record = result.single()
            if not record:
                return {"error": "No relationship found"}

            # 관계명 결정
            relationship_name = self._determine_relationship_name(
                record["chon"],
                record["relationship_types"],
                record["includes_marriage"]
            )

            return {
                **dict(record),
                "relationship_name": relationship_name
            }

    def _determine_relationship_name(self, chon: int,
                                      rel_types: List[str],
                                      includes_marriage: bool) -> str:
        """촌수에 따른 관계명 결정"""
        if chon == 0:
            return "본인"
        elif chon == 1:
            if "PARENT_OF" in rel_types:
                return "자녀" if rel_types[0] == "PARENT_OF" else "부모"
            return "부모/자녀"
        elif chon == 2:
            if includes_marriage:
                return "배우자"
            return "형제자매 또는 조부모/손자녀"
        elif chon == 3:
            return "삼촌/조카 또는 증조부모"
        elif chon == 4:
            return "사촌 또는 고조부모"
        elif chon <= 8:
            return f"{chon}촌"
        else:
            return f"원친 ({chon}촌)"
```

### 유전/특성 추적

```python
# trait_service.py
from typing import List, Dict, Any, Optional
from database import db

class TraitService:
    def add_trait(self, person_id: str, trait_name: str,
                  trait_type: str, inherited_from: Optional[str] = None) -> Dict:
        """특성 추가"""
        with db.session() as session:
            result = session.run("""
                MATCH (p:Person {id: $person_id})

                MERGE (t:Trait {name: $trait_name})
                ON CREATE SET t.type = $trait_type

                CREATE (p)-[h:HAS_TRAIT {
                    inherited_from: $inherited_from,
                    recorded_at: datetime()
                }]->(t)

                RETURN p.name AS person, t {.*} AS trait
            """, person_id=person_id, trait_name=trait_name,
                 trait_type=trait_type, inherited_from=inherited_from)

            return dict(result.single())

    def add_medical_condition(self, person_id: str, condition_name: str,
                               diagnosed_date: Optional[str] = None,
                               is_hereditary: bool = False) -> Dict:
        """의료 상태 추가"""
        with db.session() as session:
            result = session.run("""
                MATCH (p:Person {id: $person_id})

                MERGE (c:MedicalCondition {name: $condition_name})
                ON CREATE SET c.is_hereditary = $is_hereditary

                CREATE (p)-[h:HAS_CONDITION {
                    diagnosed_date: $diagnosed_date,
                    recorded_at: datetime()
                }]->(c)

                RETURN p.name AS person, c {.*} AS condition
            """, person_id=person_id, condition_name=condition_name,
                 diagnosed_date=diagnosed_date, is_hereditary=is_hereditary)

            return dict(result.single())

    def trace_trait_inheritance(self, person_id: str,
                                 trait_name: str) -> Dict[str, Any]:
        """특성 유전 추적"""
        with db.session() as session:
            result = session.run("""
                MATCH (p:Person {id: $person_id})
                MATCH (t:Trait {name: $trait_name})

                // 같은 특성을 가진 조상 찾기
                OPTIONAL MATCH ancestor_path = (p)-[:CHILD_OF*]->(ancestor:Person)-[:HAS_TRAIT]->(t)

                // 같은 특성을 가진 후손 찾기
                OPTIONAL MATCH descendant_path = (p)-[:PARENT_OF*]->(descendant:Person)-[:HAS_TRAIT]->(t)

                WITH t, p,
                     collect(DISTINCT {
                         person: ancestor {.*},
                         distance: length(ancestor_path)
                     }) AS ancestors_with_trait,
                     collect(DISTINCT {
                         person: descendant {.*},
                         distance: length(descendant_path)
                     }) AS descendants_with_trait

                RETURN
                    t {.*} AS trait,
                    ancestors_with_trait,
                    descendants_with_trait,
                    size(ancestors_with_trait) + size(descendants_with_trait) + 1 AS total_carriers
            """, person_id=person_id, trait_name=trait_name)

            record = result.single()
            return dict(record)

    def find_hereditary_condition_risk(self, person_id: str) -> List[Dict]:
        """유전 질환 위험 분석"""
        with db.session() as session:
            result = session.run("""
                MATCH (p:Person {id: $person_id})

                // 조상의 유전 질환
                MATCH (p)-[:CHILD_OF*1..5]->(ancestor:Person)-[:HAS_CONDITION]->(c:MedicalCondition)
                WHERE c.is_hereditary = true

                WITH c, count(DISTINCT ancestor) AS affected_ancestors,
                     collect(DISTINCT ancestor.name) AS affected_names

                // 위험도 계산 (영향받은 조상 수에 비례)
                RETURN
                    c.name AS condition,
                    affected_ancestors,
                    affected_names,
                    CASE
                        WHEN affected_ancestors >= 3 THEN 'high'
                        WHEN affected_ancestors >= 2 THEN 'medium'
                        ELSE 'low'
                    END AS risk_level
                ORDER BY affected_ancestors DESC
            """, person_id=person_id)

            return [dict(r) for r in result]
```

### 가계도 시각화 서비스

```python
# visualization_service.py
from typing import List, Dict, Any
from database import db

class VisualizationService:
    def get_family_tree(self, person_id: str,
                        generations_up: int = 3,
                        generations_down: int = 3) -> Dict[str, Any]:
        """가계도 데이터 생성"""
        with db.session() as session:
            result = session.run("""
                MATCH (root:Person {id: $person_id})

                // 조상 (위로)
                OPTIONAL MATCH ancestor_path = (root)-[:CHILD_OF*1..$gen_up]->(ancestor:Person)

                // 후손 (아래로)
                OPTIONAL MATCH descendant_path = (root)-[:PARENT_OF*1..$gen_down]->(descendant:Person)

                // 배우자 관계
                OPTIONAL MATCH (related:Person)-[m:MARRIED_TO]-(spouse:Person)
                WHERE related IN nodes(ancestor_path) + nodes(descendant_path) + [root]
                  AND m.is_current = true

                WITH root,
                     collect(DISTINCT {
                         id: ancestor.id,
                         name: ancestor.name,
                         gender: ancestor.gender,
                         birth_date: ancestor.birth_date,
                         death_date: ancestor.death_date,
                         generation: -length(ancestor_path)
                     }) AS ancestors,
                     collect(DISTINCT {
                         id: descendant.id,
                         name: descendant.name,
                         gender: descendant.gender,
                         birth_date: descendant.birth_date,
                         death_date: descendant.death_date,
                         generation: length(descendant_path)
                     }) AS descendants,
                     collect(DISTINCT {
                         person1: related.id,
                         person2: spouse.id
                     }) AS marriages

                RETURN
                    root {.*, generation: 0} AS root_person,
                    ancestors,
                    descendants,
                    marriages
            """, person_id=person_id, gen_up=generations_up, gen_down=generations_down)

            record = result.single()

            # 노드와 엣지 구성
            nodes = [record["root_person"]] + record["ancestors"] + record["descendants"]
            nodes = [n for n in nodes if n.get("id")]  # None 제거

            return {
                "nodes": nodes,
                "marriages": [m for m in record["marriages"] if m.get("person1")],
                "root_id": person_id
            }

    def get_pedigree_chart(self, person_id: str) -> Dict[str, Any]:
        """혈통도 (4대 조상)"""
        with db.session() as session:
            result = session.run("""
                MATCH (p:Person {id: $person_id})

                // 부모
                OPTIONAL MATCH (p)-[:CHILD_OF]->(father:Person {gender: 'male'})
                OPTIONAL MATCH (p)-[:CHILD_OF]->(mother:Person {gender: 'female'})

                // 조부모
                OPTIONAL MATCH (father)-[:CHILD_OF]->(pgf:Person {gender: 'male'})
                OPTIONAL MATCH (father)-[:CHILD_OF]->(pgm:Person {gender: 'female'})
                OPTIONAL MATCH (mother)-[:CHILD_OF]->(mgf:Person {gender: 'male'})
                OPTIONAL MATCH (mother)-[:CHILD_OF]->(mgm:Person {gender: 'female'})

                // 증조부모
                OPTIONAL MATCH (pgf)-[:CHILD_OF]->(ppgf:Person {gender: 'male'})
                OPTIONAL MATCH (pgf)-[:CHILD_OF]->(ppgm:Person {gender: 'female'})
                OPTIONAL MATCH (pgm)-[:CHILD_OF]->(pmgf:Person {gender: 'male'})
                OPTIONAL MATCH (pgm)-[:CHILD_OF]->(pmgm:Person {gender: 'female'})
                OPTIONAL MATCH (mgf)-[:CHILD_OF]->(mpgf:Person {gender: 'male'})
                OPTIONAL MATCH (mgf)-[:CHILD_OF]->(mpgm:Person {gender: 'female'})
                OPTIONAL MATCH (mgm)-[:CHILD_OF]->(mmgf:Person {gender: 'male'})
                OPTIONAL MATCH (mgm)-[:CHILD_OF]->(mmgm:Person {gender: 'female'})

                RETURN
                    p {.*} AS self,
                    father {.*} AS father,
                    mother {.*} AS mother,
                    pgf {.*} AS paternal_grandfather,
                    pgm {.*} AS paternal_grandmother,
                    mgf {.*} AS maternal_grandfather,
                    mgm {.*} AS maternal_grandmother,
                    {
                        ppgf: ppgf {.*}, ppgm: ppgm {.*},
                        pmgf: pmgf {.*}, pmgm: pmgm {.*},
                        mpgf: mpgf {.*}, mpgm: mpgm {.*},
                        mmgf: mmgf {.*}, mmgm: mmgm {.*}
                    } AS great_grandparents
            """, person_id=person_id)

            return dict(result.single())

    def export_gedcom(self, clan_id: str) -> str:
        """GEDCOM 형식으로 내보내기"""
        with db.session() as session:
            # 모든 인물 조회
            persons_result = session.run("""
                MATCH (f:Family)-[:BRANCH_OF]->(c:Clan {id: $clan_id})
                MATCH (p:Person)-[:BELONGS_TO]->(f)
                RETURN p {.*} AS person
            """, clan_id=clan_id)

            persons = [r["person"] for r in persons_result]

            # 가족 관계 조회
            families_result = session.run("""
                MATCH (f:Family)-[:BRANCH_OF]->(c:Clan {id: $clan_id})
                MATCH (p:Person)-[:BELONGS_TO]->(f)
                OPTIONAL MATCH (p)-[m:MARRIED_TO]->(spouse:Person)
                WHERE m.is_current = true
                OPTIONAL MATCH (p)-[:PARENT_OF]->(child:Person)
                RETURN p.id AS person_id,
                       collect(DISTINCT spouse.id) AS spouses,
                       collect(DISTINCT child.id) AS children
            """, clan_id=clan_id)

            families = [dict(r) for r in families_result]

        # GEDCOM 생성
        gedcom_lines = [
            "0 HEAD",
            "1 SOUR Neo4j Genealogy",
            "1 GEDC",
            "2 VERS 5.5.1",
            "1 CHAR UTF-8"
        ]

        # 개인 레코드
        for p in persons:
            gedcom_lines.extend([
                f"0 @I{p['id'][:8]}@ INDI",
                f"1 NAME {p.get('name', 'Unknown')}",
                f"1 SEX {'M' if p.get('gender') == 'male' else 'F' if p.get('gender') == 'female' else 'U'}"
            ])
            if p.get('birth_date'):
                gedcom_lines.append(f"1 BIRT")
                gedcom_lines.append(f"2 DATE {p['birth_date']}")
            if p.get('death_date'):
                gedcom_lines.append(f"1 DEAT")
                gedcom_lines.append(f"2 DATE {p['death_date']}")

        gedcom_lines.append("0 TRLR")

        return "\n".join(gedcom_lines)
```

### FastAPI 서버

```python
# main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional, List
from datetime import date

from person_service import PersonService
from clan_service import ClanService
from ancestry_service import AncestryService
from trait_service import TraitService
from visualization_service import VisualizationService
from config import Gender, RelationType

app = FastAPI(title="Genealogy System API")

person_svc = PersonService()
clan_svc = ClanService()
ancestry_svc = AncestryService()
trait_svc = TraitService()
viz_svc = VisualizationService()

# ========== 모델 정의 ==========
class PersonCreate(BaseModel):
    name: str
    gender: Gender
    birth_date: Optional[date] = None
    birth_place: Optional[str] = None
    death_date: Optional[date] = None
    generation: Optional[int] = None
    family_id: Optional[str] = None

class ParentRelation(BaseModel):
    child_id: str
    parent_id: str
    relation_type: RelationType = RelationType.BIOLOGICAL

class MarriageCreate(BaseModel):
    person1_id: str
    person2_id: str
    marriage_date: Optional[date] = None
    location: Optional[str] = None

# ========== 인물 관리 ==========
@app.post("/persons")
async def create_person(person: PersonCreate):
    return person_svc.create_person(**person.dict())

@app.get("/persons/{person_id}")
async def get_person(person_id: str):
    return person_svc.get_person_with_family(person_id)

@app.post("/relationships/parent")
async def add_parent(relation: ParentRelation):
    return person_svc.add_parent_relationship(
        relation.child_id, relation.parent_id, relation.relation_type
    )

@app.post("/relationships/marriage")
async def add_marriage(marriage: MarriageCreate):
    return person_svc.add_marriage(**marriage.dict())

# ========== 씨족/가문 ==========
@app.post("/clans")
async def create_clan(name: str, origin: str, founding_date: str = None):
    return clan_svc.create_clan(name, origin, founding_date)

@app.get("/clans/{clan_id}/stats")
async def get_clan_stats(clan_id: str):
    return clan_svc.get_clan_statistics(clan_id)

# ========== 계보 탐색 ==========
@app.get("/persons/{person_id}/ancestors")
async def get_ancestors(person_id: str, generations: int = 10):
    return ancestry_svc.get_ancestors(person_id, generations)

@app.get("/persons/{person_id}/descendants")
async def get_descendants(person_id: str, generations: int = 10):
    return ancestry_svc.get_descendants(person_id, generations)

@app.get("/persons/{person_id}/lineage")
async def get_lineage(person_id: str):
    return ancestry_svc.get_lineage(person_id)

@app.get("/relationship")
async def calculate_relationship(person1_id: str, person2_id: str):
    return ancestry_svc.calculate_relationship(person1_id, person2_id)

@app.get("/common-ancestor")
async def find_common_ancestor(person1_id: str, person2_id: str):
    return ancestry_svc.find_common_ancestor(person1_id, person2_id)

# ========== 특성/유전 ==========
@app.post("/persons/{person_id}/traits")
async def add_trait(person_id: str, trait_name: str, trait_type: str):
    return trait_svc.add_trait(person_id, trait_name, trait_type)

@app.get("/persons/{person_id}/hereditary-risks")
async def get_hereditary_risks(person_id: str):
    return trait_svc.find_hereditary_condition_risk(person_id)

# ========== 시각화 ==========
@app.get("/persons/{person_id}/family-tree")
async def get_family_tree(person_id: str, up: int = 3, down: int = 3):
    return viz_svc.get_family_tree(person_id, up, down)

@app.get("/persons/{person_id}/pedigree")
async def get_pedigree(person_id: str):
    return viz_svc.get_pedigree_chart(person_id)

@app.get("/clans/{clan_id}/export/gedcom")
async def export_gedcom(clan_id: str):
    from fastapi.responses import PlainTextResponse
    gedcom = viz_svc.export_gedcom(clan_id)
    return PlainTextResponse(content=gedcom, media_type="text/plain")

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## 샘플 데이터 생성

```cypher
// 씨족 생성
CREATE (clan:Clan {
    id: 'clan-kim-gimhae',
    name: '김해 김씨',
    origin: '김해',
    founding_date: '42',
    description: '가락국 수로왕을 시조로 하는 성씨'
})

// 시조
CREATE (founder:Person {
    id: 'person-001',
    name: '김수로',
    gender: 'male',
    birth_date: '42-01-01',
    death_date: '199-01-01',
    generation: 1,
    title: '가락국 시조'
})

// 가문 분파
CREATE (family:Family {
    id: 'family-main',
    name: '본가'
})
CREATE (family)-[:BRANCH_OF]->(clan)
CREATE (founder)-[:BELONGS_TO]->(family)
CREATE (founder)-[:FOUNDER_OF]->(family)

// 2세대
CREATE (gen2_1:Person {
    id: 'person-002',
    name: '김거등',
    gender: 'male',
    generation: 2
})
CREATE (founder)-[:PARENT_OF]->(gen2_1)
CREATE (gen2_1)-[:CHILD_OF]->(founder)
CREATE (gen2_1)-[:BELONGS_TO]->(family)

// 배우자
CREATE (queen:Person {
    id: 'person-003',
    name: '허황옥',
    gender: 'female',
    birth_place: '아유타국'
})
CREATE (founder)-[:MARRIED_TO {date: '48-07-27', is_current: true}]->(queen)
CREATE (queen)-[:MARRIED_TO {date: '48-07-27', is_current: true}]->(founder)
CREATE (queen)-[:PARENT_OF]->(gen2_1)
CREATE (gen2_1)-[:CHILD_OF]->(queen)

// 더 많은 세대 추가...
CREATE (gen3_1:Person {id: 'person-004', name: '김마품', gender: 'male', generation: 3})
CREATE (gen2_1)-[:PARENT_OF]->(gen3_1)
CREATE (gen3_1)-[:CHILD_OF]->(gen2_1)
CREATE (gen3_1)-[:BELONGS_TO]->(family)

// 특성 추가
CREATE (trait:Trait {name: '왕족 혈통', type: 'lineage'})
CREATE (founder)-[:HAS_TRAIT]->(trait)
CREATE (gen2_1)-[:HAS_TRAIT]->(trait)
```

---

## 고급 쿼리 예제

### 촌수별 친척 찾기

```cypher
// 4촌 이내 친척 찾기
MATCH (p:Person {id: $person_id})
MATCH path = (p)-[:PARENT_OF|CHILD_OF|SIBLING_OF*1..4]-(relative:Person)
WHERE relative <> p

WITH relative, length(path) AS chon,
     [r IN relationships(path) | type(r)] AS rel_path

RETURN DISTINCT
    relative.name AS name,
    chon AS 촌수,
    rel_path AS 관계경로
ORDER BY chon
```

### 세대별 통계

```cypher
MATCH (c:Clan {id: $clan_id})<-[:BRANCH_OF]-(f:Family)<-[:BELONGS_TO]-(p:Person)

WITH p.generation AS generation, collect(p) AS members

RETURN
    generation AS 세대,
    size(members) AS 인원수,
    size([m IN members WHERE m.gender = 'male']) AS 남성,
    size([m IN members WHERE m.gender = 'female']) AS 여성,
    size([m IN members WHERE m.is_living = true]) AS 생존

ORDER BY generation
```

### 가장 많은 후손을 가진 조상

```cypher
MATCH (p:Person)-[:PARENT_OF*]->(descendant:Person)

WITH p, count(DISTINCT descendant) AS descendant_count

RETURN
    p.name AS ancestor,
    p.generation AS generation,
    descendant_count AS 후손수

ORDER BY descendant_count DESC
LIMIT 10
```

---

## 다음 단계

!!! success "계보/족보 시스템 완료!"
    모든 Neo4j 활용 사례 구현 가이드가 완료되었습니다.

    다른 주제들도 확인해보세요:

    - [소셜 네트워크 분석](social-network.md)
    - [실시간 추천 엔진](recommendation-engine.md)
    - [사기 탐지 시스템](fraud-detection.md)
    - [지식 그래프 RAG](kg-rag.md)
    - [네트워크 인프라 관리](network-infrastructure.md)
