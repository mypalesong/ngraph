# 네트워크 인프라 관리

## 개요

**네트워크 인프라 관리 시스템**은 데이터센터, 클라우드, 온프레미스 환경의 모든 IT 자산과 연결 관계를 그래프로 모델링하여 의존성 분석, 장애 영향도 파악, 용량 계획을 수행합니다.

### 핵심 기능

| 기능 | 설명 |
|------|------|
| 토폴로지 시각화 | 네트워크 구성 요소 간 연결 관계 |
| 의존성 분석 | 서비스-인프라 간 의존성 추적 |
| 장애 영향도 분석 | 특정 장비 장애 시 영향 범위 |
| 경로 분석 | 트래픽 경로 및 대체 경로 탐색 |
| 용량 계획 | 리소스 사용량 및 확장 계획 |

---

## 데이터 모델

### 스키마 설계

```
┌─────────────────────────────────────────────────────────────────┐
│                     INFRASTRUCTURE GRAPH                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────┐    HOSTS     ┌──────────┐    RUNS     ┌────────┐  │
│  │Datacenter│◄────────────►│  Server  │◄───────────►│Service │  │
│  └──────────┘              └──────────┘             └────────┘  │
│       │                         │                        │       │
│       │ CONTAINS               │ CONNECTED_TO           │       │
│       ▼                        ▼                        ▼       │
│  ┌──────────┐           ┌──────────┐            ┌──────────┐   │
│  │   Rack   │           │  Switch  │            │ Database │   │
│  └──────────┘           └──────────┘            └──────────┘   │
│       │                      │                        │         │
│       │ HOUSES              │ LINKS                  │ STORES  │
│       ▼                     ▼                        ▼         │
│  ┌──────────┐          ┌──────────┐            ┌──────────┐   │
│  │  Server  │          │  Router  │            │  Volume  │   │
│  └──────────┘          └──────────┘            └──────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Cypher 스키마 정의

```cypher
// 제약조건
CREATE CONSTRAINT datacenter_id FOR (d:Datacenter) REQUIRE d.id IS UNIQUE;
CREATE CONSTRAINT rack_id FOR (r:Rack) REQUIRE r.id IS UNIQUE;
CREATE CONSTRAINT server_id FOR (s:Server) REQUIRE s.id IS UNIQUE;
CREATE CONSTRAINT switch_id FOR (sw:Switch) REQUIRE sw.id IS UNIQUE;
CREATE CONSTRAINT router_id FOR (r:Router) REQUIRE r.id IS UNIQUE;
CREATE CONSTRAINT service_id FOR (s:Service) REQUIRE s.id IS UNIQUE;
CREATE CONSTRAINT database_id FOR (d:Database) REQUIRE d.id IS UNIQUE;
CREATE CONSTRAINT application_id FOR (a:Application) REQUIRE a.id IS UNIQUE;

// 인덱스
CREATE INDEX server_status FOR (s:Server) ON (s.status);
CREATE INDEX server_hostname FOR (s:Server) ON (s.hostname);
CREATE INDEX service_name FOR (s:Service) ON (s.name);
CREATE INDEX switch_ip FOR (sw:Switch) ON (sw.ip_address);

// 전문 검색 인덱스
CREATE FULLTEXT INDEX infra_search FOR (n:Server|Service|Application|Database)
ON EACH [n.name, n.description, n.hostname];
```

---

## 환경 설정

### Docker Compose

```yaml
version: '3.8'
services:
  neo4j:
    image: neo4j:5.15.0-enterprise
    ports:
      - "7474:7474"
      - "7687:7687"
    environment:
      - NEO4J_AUTH=neo4j/infrastructure123
      - NEO4J_ACCEPT_LICENSE_AGREEMENT=yes
      - NEO4J_PLUGINS=["apoc", "graph-data-science"]
    volumes:
      - neo4j_data:/data

  infra-api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - NEO4J_URI=bolt://neo4j:7687
      - NEO4J_USER=neo4j
      - NEO4J_PASSWORD=infrastructure123
    depends_on:
      - neo4j

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    depends_on:
      - prometheus

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

class ServerStatus(str, Enum):
    ACTIVE = "active"
    MAINTENANCE = "maintenance"
    OFFLINE = "offline"
    DEGRADED = "degraded"

class Settings(BaseSettings):
    neo4j_uri: str = "bolt://localhost:7687"
    neo4j_user: str = "neo4j"
    neo4j_password: str = "infrastructure123"

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

### 인프라 관리 서비스

```python
# infrastructure_service.py
from typing import List, Dict, Any, Optional
from datetime import datetime
from database import db
from config import ServerStatus

class InfrastructureService:
    # ========== 데이터센터 관리 ==========

    def create_datacenter(self, dc_id: str, name: str,
                          location: str, tier: int = 3) -> Dict:
        """데이터센터 생성"""
        with db.session() as session:
            result = session.run("""
                CREATE (d:Datacenter {
                    id: $dc_id,
                    name: $name,
                    location: $location,
                    tier: $tier,
                    created_at: datetime()
                })
                RETURN d {.*} AS datacenter
            """, dc_id=dc_id, name=name, location=location, tier=tier)
            return result.single()["datacenter"]

    def add_rack_to_datacenter(self, dc_id: str, rack_id: str,
                                row: str, position: int) -> Dict:
        """랙 추가"""
        with db.session() as session:
            result = session.run("""
                MATCH (d:Datacenter {id: $dc_id})
                CREATE (r:Rack {
                    id: $rack_id,
                    row: $row,
                    position: $position,
                    capacity_u: 42,
                    used_u: 0
                })
                CREATE (d)-[:CONTAINS]->(r)
                RETURN r {.*} AS rack
            """, dc_id=dc_id, rack_id=rack_id, row=row, position=position)
            return result.single()["rack"]

    # ========== 서버 관리 ==========

    def create_server(self, server_id: str, hostname: str,
                      specs: Dict, rack_id: str, position_u: int) -> Dict:
        """서버 생성 및 랙에 배치"""
        with db.session() as session:
            result = session.run("""
                MATCH (r:Rack {id: $rack_id})
                CREATE (s:Server {
                    id: $server_id,
                    hostname: $hostname,
                    cpu_cores: $cpu_cores,
                    memory_gb: $memory_gb,
                    storage_tb: $storage_tb,
                    os: $os,
                    ip_address: $ip_address,
                    status: $status,
                    position_u: $position_u,
                    height_u: $height_u,
                    created_at: datetime()
                })
                CREATE (r)-[:HOUSES {position: $position_u}]->(s)
                SET r.used_u = r.used_u + $height_u
                RETURN s {.*} AS server
            """,
                server_id=server_id,
                hostname=hostname,
                rack_id=rack_id,
                position_u=position_u,
                cpu_cores=specs.get("cpu_cores", 16),
                memory_gb=specs.get("memory_gb", 64),
                storage_tb=specs.get("storage_tb", 1),
                os=specs.get("os", "Ubuntu 22.04"),
                ip_address=specs.get("ip_address"),
                status=ServerStatus.ACTIVE.value,
                height_u=specs.get("height_u", 1)
            )
            return result.single()["server"]

    def update_server_status(self, server_id: str,
                             status: ServerStatus, reason: str = None) -> Dict:
        """서버 상태 업데이트"""
        with db.session() as session:
            result = session.run("""
                MATCH (s:Server {id: $server_id})
                SET s.status = $status,
                    s.status_changed_at = datetime(),
                    s.status_reason = $reason
                RETURN s {.*} AS server
            """, server_id=server_id, status=status.value, reason=reason)
            return result.single()["server"]

    # ========== 네트워크 장비 ==========

    def create_switch(self, switch_id: str, name: str,
                      ip_address: str, port_count: int,
                      switch_type: str = "access") -> Dict:
        """스위치 생성"""
        with db.session() as session:
            result = session.run("""
                CREATE (sw:Switch:NetworkDevice {
                    id: $switch_id,
                    name: $name,
                    ip_address: $ip_address,
                    port_count: $port_count,
                    type: $switch_type,
                    status: 'active'
                })
                RETURN sw {.*} AS switch
            """, switch_id=switch_id, name=name, ip_address=ip_address,
                 port_count=port_count, switch_type=switch_type)
            return result.single()["switch"]

    def connect_server_to_switch(self, server_id: str, switch_id: str,
                                  port: int, speed_gbps: int = 10) -> Dict:
        """서버를 스위치에 연결"""
        with db.session() as session:
            result = session.run("""
                MATCH (s:Server {id: $server_id})
                MATCH (sw:Switch {id: $switch_id})
                CREATE (s)-[c:CONNECTED_TO {
                    port: $port,
                    speed_gbps: $speed_gbps,
                    connected_at: datetime()
                }]->(sw)
                RETURN s.hostname AS server, sw.name AS switch, c {.*} AS connection
            """, server_id=server_id, switch_id=switch_id,
                 port=port, speed_gbps=speed_gbps)
            return dict(result.single())

    def link_switches(self, switch1_id: str, switch2_id: str,
                      port1: int, port2: int, speed_gbps: int = 40) -> Dict:
        """스위치 간 연결 (업링크)"""
        with db.session() as session:
            result = session.run("""
                MATCH (sw1:Switch {id: $switch1_id})
                MATCH (sw2:Switch {id: $switch2_id})
                CREATE (sw1)-[l:LINKS {
                    port1: $port1,
                    port2: $port2,
                    speed_gbps: $speed_gbps,
                    link_type: 'uplink'
                }]->(sw2)
                RETURN sw1.name AS from_switch, sw2.name AS to_switch, l {.*} AS link
            """, switch1_id=switch1_id, switch2_id=switch2_id,
                 port1=port1, port2=port2, speed_gbps=speed_gbps)
            return dict(result.single())

    # ========== 서비스/애플리케이션 ==========

    def create_service(self, service_id: str, name: str,
                       service_type: str, criticality: str = "medium") -> Dict:
        """서비스 생성"""
        with db.session() as session:
            result = session.run("""
                CREATE (s:Service {
                    id: $service_id,
                    name: $name,
                    type: $service_type,
                    criticality: $criticality,
                    status: 'active',
                    created_at: datetime()
                })
                RETURN s {.*} AS service
            """, service_id=service_id, name=name,
                 service_type=service_type, criticality=criticality)
            return result.single()["service"]

    def deploy_service_to_server(self, service_id: str, server_id: str,
                                  port: int, resources: Dict = None) -> Dict:
        """서비스를 서버에 배포"""
        with db.session() as session:
            result = session.run("""
                MATCH (svc:Service {id: $service_id})
                MATCH (srv:Server {id: $server_id})
                CREATE (srv)-[r:RUNS {
                    port: $port,
                    cpu_allocated: $cpu,
                    memory_allocated_gb: $memory,
                    deployed_at: datetime()
                }]->(svc)
                RETURN srv.hostname AS server, svc.name AS service, r {.*} AS deployment
            """, service_id=service_id, server_id=server_id, port=port,
                 cpu=resources.get("cpu", 2) if resources else 2,
                 memory=resources.get("memory_gb", 4) if resources else 4)
            return dict(result.single())

    def add_service_dependency(self, service_id: str,
                               depends_on_id: str, dependency_type: str) -> Dict:
        """서비스 의존성 추가"""
        with db.session() as session:
            result = session.run("""
                MATCH (s1:Service {id: $service_id})
                MATCH (s2:Service {id: $depends_on_id})
                CREATE (s1)-[d:DEPENDS_ON {
                    type: $dependency_type,
                    created_at: datetime()
                }]->(s2)
                RETURN s1.name AS service, s2.name AS depends_on, d.type AS dependency_type
            """, service_id=service_id, depends_on_id=depends_on_id,
                 dependency_type=dependency_type)
            return dict(result.single())
```

### 의존성 분석 서비스

```python
# dependency_analyzer.py
from typing import List, Dict, Any
from database import db

class DependencyAnalyzer:
    def get_service_dependencies(self, service_id: str,
                                  depth: int = 3) -> Dict[str, Any]:
        """서비스의 전체 의존성 트리 조회"""
        with db.session() as session:
            result = session.run("""
                MATCH (s:Service {id: $service_id})

                // 상위 의존성 (이 서비스에 의존하는 것들)
                OPTIONAL MATCH upstream_path = (upstream:Service)-[:DEPENDS_ON*1..$depth]->(s)

                // 하위 의존성 (이 서비스가 의존하는 것들)
                OPTIONAL MATCH downstream_path = (s)-[:DEPENDS_ON*1..$depth]->(downstream:Service)

                // 인프라 의존성
                OPTIONAL MATCH (srv:Server)-[:RUNS]->(s)
                OPTIONAL MATCH (srv)-[:CONNECTED_TO]->(sw:Switch)

                RETURN
                    s {.*} AS service,
                    collect(DISTINCT {
                        path: [n IN nodes(upstream_path) | n.name],
                        depth: length(upstream_path)
                    }) AS upstream_dependencies,
                    collect(DISTINCT {
                        path: [n IN nodes(downstream_path) | n.name],
                        depth: length(downstream_path)
                    }) AS downstream_dependencies,
                    collect(DISTINCT srv {.*}) AS servers,
                    collect(DISTINCT sw.name) AS network_devices
            """, service_id=service_id, depth=depth)

            record = result.single()
            return {
                "service": record["service"],
                "upstream": [d for d in record["upstream_dependencies"] if d["path"]],
                "downstream": [d for d in record["downstream_dependencies"] if d["path"]],
                "infrastructure": {
                    "servers": record["servers"],
                    "network_devices": record["network_devices"]
                }
            }

    def find_critical_path(self, from_service: str, to_service: str) -> List[Dict]:
        """두 서비스 간 의존성 경로 탐색"""
        with db.session() as session:
            result = session.run("""
                MATCH (s1:Service {name: $from_service})
                MATCH (s2:Service {name: $to_service})
                MATCH path = shortestPath((s1)-[:DEPENDS_ON*]-(s2))
                RETURN
                    [n IN nodes(path) | {
                        name: n.name,
                        type: n.type,
                        criticality: n.criticality
                    }] AS path_nodes,
                    length(path) AS hops
            """, from_service=from_service, to_service=to_service)

            paths = []
            for record in result:
                paths.append({
                    "nodes": record["path_nodes"],
                    "hops": record["hops"]
                })
            return paths

    def get_infrastructure_dependencies(self, server_id: str) -> Dict[str, Any]:
        """서버의 인프라 의존성 조회"""
        with db.session() as session:
            result = session.run("""
                MATCH (s:Server {id: $server_id})

                // 물리적 위치
                OPTIONAL MATCH (rack:Rack)-[:HOUSES]->(s)
                OPTIONAL MATCH (dc:Datacenter)-[:CONTAINS]->(rack)

                // 네트워크 연결
                OPTIONAL MATCH (s)-[:CONNECTED_TO]->(sw:Switch)
                OPTIONAL MATCH (sw)-[:LINKS*1..2]-(upstream:Switch)

                // 실행 중인 서비스
                OPTIONAL MATCH (s)-[:RUNS]->(svc:Service)

                // 스토리지
                OPTIONAL MATCH (s)-[:MOUNTS]->(vol:Volume)

                RETURN
                    s {.*} AS server,
                    dc {.*} AS datacenter,
                    rack {.*} AS rack,
                    collect(DISTINCT sw {.*}) AS switches,
                    collect(DISTINCT upstream.name) AS upstream_switches,
                    collect(DISTINCT svc {.*}) AS services,
                    collect(DISTINCT vol {.*}) AS volumes
            """, server_id=server_id)

            record = result.single()
            return {
                "server": record["server"],
                "location": {
                    "datacenter": record["datacenter"],
                    "rack": record["rack"]
                },
                "network": {
                    "switches": record["switches"],
                    "upstream": record["upstream_switches"]
                },
                "services": record["services"],
                "storage": record["volumes"]
            }
```

### 장애 영향도 분석

```python
# impact_analyzer.py
from typing import List, Dict, Any
from database import db
from datetime import datetime

class ImpactAnalyzer:
    def analyze_server_failure(self, server_id: str) -> Dict[str, Any]:
        """서버 장애 시 영향도 분석"""
        with db.session() as session:
            result = session.run("""
                MATCH (s:Server {id: $server_id})

                // 직접 영향받는 서비스
                OPTIONAL MATCH (s)-[:RUNS]->(direct_svc:Service)

                // 간접 영향 (의존성 체인)
                OPTIONAL MATCH (s)-[:RUNS]->(direct_svc)-[:DEPENDS_ON*1..3]-(indirect_svc:Service)
                WHERE indirect_svc <> direct_svc

                // 같은 랙의 다른 서버 (전원 장애 시)
                OPTIONAL MATCH (rack:Rack)-[:HOUSES]->(s)
                OPTIONAL MATCH (rack)-[:HOUSES]->(sibling:Server)
                WHERE sibling <> s

                // 네트워크 경로
                OPTIONAL MATCH (s)-[:CONNECTED_TO]->(sw:Switch)

                RETURN
                    s {.*} AS failed_server,
                    collect(DISTINCT direct_svc {
                        .*, impact: 'direct', users_affected: direct_svc.estimated_users
                    }) AS direct_impact,
                    collect(DISTINCT indirect_svc {
                        .*, impact: 'indirect'
                    }) AS indirect_impact,
                    collect(DISTINCT sibling.hostname) AS same_rack_servers,
                    sw.name AS connected_switch
            """, server_id=server_id)

            record = result.single()

            # 영향도 점수 계산
            direct_services = record["direct_impact"]
            indirect_services = record["indirect_impact"]

            impact_score = self._calculate_impact_score(
                direct_services, indirect_services
            )

            return {
                "failed_component": record["failed_server"],
                "impact_score": impact_score,
                "direct_impact": {
                    "services": direct_services,
                    "count": len(direct_services)
                },
                "indirect_impact": {
                    "services": indirect_services,
                    "count": len(indirect_services)
                },
                "risk_factors": {
                    "same_rack_servers": record["same_rack_servers"],
                    "network_device": record["connected_switch"]
                },
                "recommendations": self._get_recommendations(
                    direct_services, indirect_services
                )
            }

    def analyze_switch_failure(self, switch_id: str) -> Dict[str, Any]:
        """스위치 장애 시 영향도 분석"""
        with db.session() as session:
            result = session.run("""
                MATCH (sw:Switch {id: $switch_id})

                // 연결된 서버
                OPTIONAL MATCH (server:Server)-[:CONNECTED_TO]->(sw)

                // 영향받는 서비스
                OPTIONAL MATCH (server)-[:RUNS]->(svc:Service)

                // 연결된 다른 스위치 (캐스케이드 장애)
                OPTIONAL MATCH (sw)-[:LINKS]-(connected_sw:Switch)

                // 대체 경로 확인
                OPTIONAL MATCH (server)-[:CONNECTED_TO]->(alt_sw:Switch)
                WHERE alt_sw <> sw

                WITH sw, server, svc, connected_sw, alt_sw
                RETURN
                    sw {.*} AS failed_switch,
                    collect(DISTINCT server {
                        .hostname, .id, has_redundancy: alt_sw IS NOT NULL
                    }) AS affected_servers,
                    collect(DISTINCT svc.name) AS affected_services,
                    collect(DISTINCT connected_sw.name) AS cascade_risk,
                    count(DISTINCT CASE WHEN alt_sw IS NOT NULL THEN server END) AS servers_with_redundancy
            """, switch_id=switch_id)

            record = result.single()

            total_servers = len(record["affected_servers"])
            redundant_servers = record["servers_with_redundancy"]

            return {
                "failed_component": record["failed_switch"],
                "affected_servers": {
                    "list": record["affected_servers"],
                    "total": total_servers,
                    "with_redundancy": redundant_servers,
                    "at_risk": total_servers - redundant_servers
                },
                "affected_services": record["affected_services"],
                "cascade_risk": record["cascade_risk"],
                "redundancy_coverage": (
                    redundant_servers / total_servers * 100
                    if total_servers > 0 else 0
                )
            }

    def analyze_datacenter_failure(self, dc_id: str) -> Dict[str, Any]:
        """데이터센터 장애 시 영향도 분석 (DR 시나리오)"""
        with db.session() as session:
            result = session.run("""
                MATCH (dc:Datacenter {id: $dc_id})
                MATCH (dc)-[:CONTAINS]->(rack:Rack)-[:HOUSES]->(server:Server)

                // 영향받는 서비스
                OPTIONAL MATCH (server)-[:RUNS]->(svc:Service)

                // DR 사이트 확인
                OPTIONAL MATCH (svc)-[:REPLICATED_TO]->(dr_svc:Service)
                OPTIONAL MATCH (dr_server:Server)-[:RUNS]->(dr_svc)
                OPTIONAL MATCH (dr_rack:Rack)-[:HOUSES]->(dr_server)
                OPTIONAL MATCH (dr_dc:Datacenter)-[:CONTAINS]->(dr_rack)
                WHERE dr_dc <> dc

                RETURN
                    dc {.*} AS datacenter,
                    count(DISTINCT server) AS total_servers,
                    count(DISTINCT svc) AS total_services,
                    collect(DISTINCT svc {
                        .name,
                        .criticality,
                        has_dr: dr_svc IS NOT NULL,
                        dr_location: dr_dc.name
                    }) AS services_dr_status
            """, dc_id=dc_id)

            record = result.single()

            services = record["services_dr_status"]
            dr_covered = len([s for s in services if s.get("has_dr")])

            return {
                "datacenter": record["datacenter"],
                "impact": {
                    "total_servers": record["total_servers"],
                    "total_services": record["total_services"]
                },
                "dr_status": {
                    "services_with_dr": dr_covered,
                    "services_without_dr": len(services) - dr_covered,
                    "coverage_percent": dr_covered / len(services) * 100 if services else 0,
                    "details": services
                }
            }

    def _calculate_impact_score(self, direct: List, indirect: List) -> float:
        """영향도 점수 계산"""
        criticality_weights = {
            "critical": 10,
            "high": 5,
            "medium": 2,
            "low": 1
        }

        score = 0
        for svc in direct:
            weight = criticality_weights.get(svc.get("criticality", "medium"), 2)
            score += weight * 2  # 직접 영향은 2배

        for svc in indirect:
            weight = criticality_weights.get(svc.get("criticality", "medium"), 2)
            score += weight

        return min(score, 100)  # 최대 100점

    def _get_recommendations(self, direct: List, indirect: List) -> List[str]:
        """권장 조치 생성"""
        recommendations = []

        critical_services = [s for s in direct + indirect
                           if s.get("criticality") == "critical"]

        if critical_services:
            recommendations.append(
                f"긴급: {len(critical_services)}개의 중요 서비스 영향. 즉시 대응 필요."
            )

        if len(direct) > 3:
            recommendations.append(
                "서비스 분산 검토: 단일 서버에 너무 많은 서비스가 배포됨"
            )

        if len(indirect) > len(direct) * 2:
            recommendations.append(
                "의존성 체인 검토: 간접 영향 범위가 큼. 서비스 분리 고려"
            )

        return recommendations
```

### 네트워크 경로 분석

```python
# path_analyzer.py
from typing import List, Dict, Any, Optional
from database import db

class PathAnalyzer:
    def find_network_path(self, from_server: str,
                          to_server: str) -> Dict[str, Any]:
        """두 서버 간 네트워크 경로 탐색"""
        with db.session() as session:
            result = session.run("""
                MATCH (s1:Server {hostname: $from_server})
                MATCH (s2:Server {hostname: $to_server})

                // 네트워크 경로 찾기
                MATCH path = shortestPath(
                    (s1)-[:CONNECTED_TO|LINKS*]-(s2)
                )

                RETURN
                    [n IN nodes(path) |
                        CASE
                            WHEN 'Server' IN labels(n) THEN {type: 'server', name: n.hostname}
                            WHEN 'Switch' IN labels(n) THEN {type: 'switch', name: n.name, ip: n.ip_address}
                            WHEN 'Router' IN labels(n) THEN {type: 'router', name: n.name}
                            ELSE {type: 'unknown', name: coalesce(n.name, n.hostname)}
                        END
                    ] AS path_nodes,
                    length(path) AS hops
            """, from_server=from_server, to_server=to_server)

            record = result.single()
            if not record:
                return {"error": "No path found", "path": [], "hops": -1}

            return {
                "from": from_server,
                "to": to_server,
                "path": record["path_nodes"],
                "hops": record["hops"]
            }

    def find_all_paths(self, from_server: str, to_server: str,
                       max_paths: int = 5) -> List[Dict]:
        """모든 가능한 경로 탐색 (대체 경로 포함)"""
        with db.session() as session:
            result = session.run("""
                MATCH (s1:Server {hostname: $from_server})
                MATCH (s2:Server {hostname: $to_server})

                MATCH path = (s1)-[:CONNECTED_TO|LINKS*..6]-(s2)

                WITH path,
                     [n IN nodes(path) | coalesce(n.name, n.hostname)] AS path_names,
                     length(path) AS hops,
                     reduce(bw = 1000, r IN relationships(path) |
                        CASE WHEN r.speed_gbps < bw THEN r.speed_gbps ELSE bw END
                     ) AS min_bandwidth

                RETURN path_names, hops, min_bandwidth
                ORDER BY hops, min_bandwidth DESC
                LIMIT $max_paths
            """, from_server=from_server, to_server=to_server, max_paths=max_paths)

            paths = []
            for record in result:
                paths.append({
                    "path": record["path_names"],
                    "hops": record["hops"],
                    "bandwidth_gbps": record["min_bandwidth"]
                })
            return paths

    def check_redundancy(self, server_id: str) -> Dict[str, Any]:
        """서버의 네트워크 이중화 상태 확인"""
        with db.session() as session:
            result = session.run("""
                MATCH (s:Server {id: $server_id})

                // 모든 네트워크 연결
                MATCH (s)-[conn:CONNECTED_TO]->(sw:Switch)

                // 각 스위치의 업링크
                OPTIONAL MATCH (sw)-[:LINKS]->(upstream:Switch)

                WITH s, collect({
                    switch: sw.name,
                    port: conn.port,
                    speed: conn.speed_gbps,
                    uplinks: collect(upstream.name)
                }) AS connections

                RETURN
                    s.hostname AS server,
                    size(connections) AS connection_count,
                    connections,
                    CASE
                        WHEN size(connections) >= 2 THEN 'redundant'
                        WHEN size(connections) = 1 THEN 'single'
                        ELSE 'disconnected'
                    END AS redundancy_status
            """, server_id=server_id)

            record = result.single()
            return {
                "server": record["server"],
                "status": record["redundancy_status"],
                "connection_count": record["connection_count"],
                "connections": record["connections"]
            }

    def find_single_points_of_failure(self) -> List[Dict]:
        """단일 장애점(SPOF) 탐지"""
        with db.session() as session:
            result = session.run("""
                // 단일 연결된 서버
                MATCH (s:Server)-[:CONNECTED_TO]->(sw:Switch)
                WITH s, count(sw) AS switch_count
                WHERE switch_count = 1

                // 해당 서버의 서비스
                OPTIONAL MATCH (s)-[:RUNS]->(svc:Service)

                RETURN
                    s.hostname AS server,
                    s.id AS server_id,
                    collect(svc.name) AS services,
                    'single_network' AS spof_type

                UNION

                // 단일 업링크 스위치
                MATCH (sw:Switch)
                WHERE sw.type = 'access'
                OPTIONAL MATCH (sw)-[:LINKS]->(upstream:Switch)
                WITH sw, count(upstream) AS uplink_count
                WHERE uplink_count <= 1

                // 연결된 서버 수
                OPTIONAL MATCH (server:Server)-[:CONNECTED_TO]->(sw)

                RETURN
                    sw.name AS server,
                    sw.id AS server_id,
                    collect(server.hostname) AS services,
                    'single_uplink' AS spof_type
            """)

            return [dict(r) for r in result]
```

### 용량 계획 서비스

```python
# capacity_planner.py
from typing import List, Dict, Any
from database import db
from datetime import datetime, timedelta

class CapacityPlanner:
    def get_datacenter_capacity(self, dc_id: str) -> Dict[str, Any]:
        """데이터센터 용량 현황"""
        with db.session() as session:
            result = session.run("""
                MATCH (dc:Datacenter {id: $dc_id})
                MATCH (dc)-[:CONTAINS]->(rack:Rack)
                OPTIONAL MATCH (rack)-[:HOUSES]->(server:Server)

                WITH dc, rack,
                     sum(rack.capacity_u) AS total_rack_u,
                     sum(rack.used_u) AS used_rack_u,
                     collect(server) AS servers

                UNWIND servers AS s
                WITH dc, total_rack_u, used_rack_u, servers,
                     sum(s.cpu_cores) AS total_cpu,
                     sum(s.memory_gb) AS total_memory,
                     sum(s.storage_tb) AS total_storage

                RETURN
                    dc.name AS datacenter,
                    size(servers) AS server_count,
                    {
                        total: total_rack_u,
                        used: used_rack_u,
                        available: total_rack_u - used_rack_u,
                        utilization: toFloat(used_rack_u) / total_rack_u * 100
                    } AS rack_space,
                    {
                        total_cores: total_cpu,
                        total_memory_gb: total_memory,
                        total_storage_tb: total_storage
                    } AS compute_capacity
            """, dc_id=dc_id)

            record = result.single()
            return {
                "datacenter": record["datacenter"],
                "server_count": record["server_count"],
                "rack_space": record["rack_space"],
                "compute_capacity": record["compute_capacity"]
            }

    def get_server_utilization(self, server_id: str) -> Dict[str, Any]:
        """서버 리소스 사용률"""
        with db.session() as session:
            result = session.run("""
                MATCH (s:Server {id: $server_id})
                OPTIONAL MATCH (s)-[r:RUNS]->(svc:Service)

                WITH s,
                     sum(r.cpu_allocated) AS cpu_allocated,
                     sum(r.memory_allocated_gb) AS memory_allocated,
                     collect(svc {.name, cpu: r.cpu_allocated, memory: r.memory_allocated_gb}) AS services

                RETURN
                    s.hostname AS server,
                    s.cpu_cores AS total_cpu,
                    s.memory_gb AS total_memory,
                    cpu_allocated,
                    memory_allocated,
                    toFloat(cpu_allocated) / s.cpu_cores * 100 AS cpu_utilization,
                    toFloat(memory_allocated) / s.memory_gb * 100 AS memory_utilization,
                    services
            """, server_id=server_id)

            record = result.single()
            return {
                "server": record["server"],
                "cpu": {
                    "total": record["total_cpu"],
                    "allocated": record["cpu_allocated"],
                    "utilization_percent": record["cpu_utilization"]
                },
                "memory": {
                    "total_gb": record["total_memory"],
                    "allocated_gb": record["memory_allocated"],
                    "utilization_percent": record["memory_utilization"]
                },
                "services": record["services"]
            }

    def find_overloaded_servers(self, cpu_threshold: float = 80,
                                 memory_threshold: float = 80) -> List[Dict]:
        """과부하 서버 탐지"""
        with db.session() as session:
            result = session.run("""
                MATCH (s:Server)
                OPTIONAL MATCH (s)-[r:RUNS]->(svc:Service)

                WITH s,
                     sum(r.cpu_allocated) AS cpu_allocated,
                     sum(r.memory_allocated_gb) AS memory_allocated,
                     count(svc) AS service_count

                WHERE toFloat(cpu_allocated) / s.cpu_cores * 100 > $cpu_threshold
                   OR toFloat(memory_allocated) / s.memory_gb * 100 > $memory_threshold

                RETURN
                    s.hostname AS server,
                    s.id AS server_id,
                    toFloat(cpu_allocated) / s.cpu_cores * 100 AS cpu_utilization,
                    toFloat(memory_allocated) / s.memory_gb * 100 AS memory_utilization,
                    service_count
                ORDER BY cpu_utilization DESC, memory_utilization DESC
            """, cpu_threshold=cpu_threshold, memory_threshold=memory_threshold)

            return [dict(r) for r in result]

    def recommend_placement(self, service_requirements: Dict) -> List[Dict]:
        """서비스 배치 추천"""
        cpu_required = service_requirements.get("cpu", 2)
        memory_required = service_requirements.get("memory_gb", 4)
        criticality = service_requirements.get("criticality", "medium")

        with db.session() as session:
            result = session.run("""
                MATCH (s:Server {status: 'active'})
                OPTIONAL MATCH (s)-[r:RUNS]->(:Service)

                WITH s,
                     coalesce(sum(r.cpu_allocated), 0) AS cpu_used,
                     coalesce(sum(r.memory_allocated_gb), 0) AS memory_used

                WHERE s.cpu_cores - cpu_used >= $cpu_required
                  AND s.memory_gb - memory_used >= $memory_required

                // 위치 정보
                OPTIONAL MATCH (rack:Rack)-[:HOUSES]->(s)
                OPTIONAL MATCH (dc:Datacenter)-[:CONTAINS]->(rack)

                // 이중화 상태
                OPTIONAL MATCH (s)-[:CONNECTED_TO]->(sw:Switch)

                WITH s, cpu_used, memory_used, dc.name AS datacenter, rack.id AS rack,
                     count(sw) AS network_connections,
                     s.cpu_cores - cpu_used AS cpu_available,
                     s.memory_gb - memory_used AS memory_available

                RETURN
                    s.hostname AS server,
                    s.id AS server_id,
                    datacenter,
                    rack,
                    cpu_available,
                    memory_available,
                    network_connections,
                    // 점수 계산 (가용 리소스 + 이중화)
                    cpu_available * 0.3 + memory_available * 0.3 + network_connections * 20 AS score

                ORDER BY score DESC
                LIMIT 5
            """, cpu_required=cpu_required, memory_required=memory_required)

            return [dict(r) for r in result]
```

### FastAPI 서버

```python
# main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional, Dict, List
from enum import Enum

from infrastructure_service import InfrastructureService
from dependency_analyzer import DependencyAnalyzer
from impact_analyzer import ImpactAnalyzer
from path_analyzer import PathAnalyzer
from capacity_planner import CapacityPlanner
from config import ServerStatus

app = FastAPI(title="Network Infrastructure Management API")

infra = InfrastructureService()
dependency = DependencyAnalyzer()
impact = ImpactAnalyzer()
path = PathAnalyzer()
capacity = CapacityPlanner()

# ========== 모델 정의 ==========
class ServerCreate(BaseModel):
    server_id: str
    hostname: str
    rack_id: str
    position_u: int
    specs: Dict = {}

class ServiceCreate(BaseModel):
    service_id: str
    name: str
    service_type: str
    criticality: str = "medium"

class DeploymentCreate(BaseModel):
    service_id: str
    server_id: str
    port: int
    resources: Dict = None

# ========== 인프라 관리 ==========
@app.post("/datacenters/{dc_id}")
async def create_datacenter(dc_id: str, name: str, location: str, tier: int = 3):
    return infra.create_datacenter(dc_id, name, location, tier)

@app.post("/datacenters/{dc_id}/racks")
async def add_rack(dc_id: str, rack_id: str, row: str, position: int):
    return infra.add_rack_to_datacenter(dc_id, rack_id, row, position)

@app.post("/servers")
async def create_server(server: ServerCreate):
    return infra.create_server(
        server.server_id, server.hostname,
        server.specs, server.rack_id, server.position_u
    )

@app.patch("/servers/{server_id}/status")
async def update_server_status(server_id: str, status: ServerStatus, reason: str = None):
    return infra.update_server_status(server_id, status, reason)

@app.post("/services")
async def create_service(service: ServiceCreate):
    return infra.create_service(
        service.service_id, service.name,
        service.service_type, service.criticality
    )

@app.post("/deployments")
async def deploy_service(deployment: DeploymentCreate):
    return infra.deploy_service_to_server(
        deployment.service_id, deployment.server_id,
        deployment.port, deployment.resources
    )

# ========== 의존성 분석 ==========
@app.get("/services/{service_id}/dependencies")
async def get_dependencies(service_id: str, depth: int = 3):
    return dependency.get_service_dependencies(service_id, depth)

@app.get("/servers/{server_id}/infrastructure")
async def get_server_infrastructure(server_id: str):
    return dependency.get_infrastructure_dependencies(server_id)

# ========== 영향도 분석 ==========
@app.get("/impact/server/{server_id}")
async def analyze_server_impact(server_id: str):
    return impact.analyze_server_failure(server_id)

@app.get("/impact/switch/{switch_id}")
async def analyze_switch_impact(switch_id: str):
    return impact.analyze_switch_failure(switch_id)

@app.get("/impact/datacenter/{dc_id}")
async def analyze_dc_impact(dc_id: str):
    return impact.analyze_datacenter_failure(dc_id)

# ========== 경로 분석 ==========
@app.get("/paths/network")
async def find_path(from_server: str, to_server: str):
    return path.find_network_path(from_server, to_server)

@app.get("/paths/all")
async def find_all_paths(from_server: str, to_server: str, max_paths: int = 5):
    return path.find_all_paths(from_server, to_server, max_paths)

@app.get("/servers/{server_id}/redundancy")
async def check_redundancy(server_id: str):
    return path.check_redundancy(server_id)

@app.get("/analysis/spof")
async def find_spof():
    """단일 장애점 탐지"""
    return path.find_single_points_of_failure()

# ========== 용량 계획 ==========
@app.get("/capacity/datacenter/{dc_id}")
async def get_dc_capacity(dc_id: str):
    return capacity.get_datacenter_capacity(dc_id)

@app.get("/capacity/server/{server_id}")
async def get_server_capacity(server_id: str):
    return capacity.get_server_utilization(server_id)

@app.get("/capacity/overloaded")
async def find_overloaded(cpu_threshold: float = 80, memory_threshold: float = 80):
    return capacity.find_overloaded_servers(cpu_threshold, memory_threshold)

@app.post("/capacity/recommend")
async def recommend_placement(requirements: Dict):
    return capacity.recommend_placement(requirements)

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## 샘플 데이터 생성

```cypher
// 데이터센터 생성
CREATE (dc1:Datacenter {id: 'dc-seoul', name: '서울 데이터센터', location: '서울 가산', tier: 3})
CREATE (dc2:Datacenter {id: 'dc-busan', name: '부산 DR센터', location: '부산 해운대', tier: 2})

// 랙 생성
CREATE (r1:Rack {id: 'rack-a01', row: 'A', position: 1, capacity_u: 42, used_u: 0})
CREATE (r2:Rack {id: 'rack-a02', row: 'A', position: 2, capacity_u: 42, used_u: 0})
CREATE (dc1)-[:CONTAINS]->(r1)
CREATE (dc1)-[:CONTAINS]->(r2)

// 서버 생성
CREATE (s1:Server {id: 'srv-001', hostname: 'web-01', cpu_cores: 32, memory_gb: 128, status: 'active'})
CREATE (s2:Server {id: 'srv-002', hostname: 'web-02', cpu_cores: 32, memory_gb: 128, status: 'active'})
CREATE (s3:Server {id: 'srv-003', hostname: 'db-01', cpu_cores: 64, memory_gb: 256, status: 'active'})
CREATE (s4:Server {id: 'srv-004', hostname: 'app-01', cpu_cores: 16, memory_gb: 64, status: 'active'})

CREATE (r1)-[:HOUSES {position: 1}]->(s1)
CREATE (r1)-[:HOUSES {position: 3}]->(s2)
CREATE (r2)-[:HOUSES {position: 1}]->(s3)
CREATE (r2)-[:HOUSES {position: 3}]->(s4)

// 네트워크 장비
CREATE (sw1:Switch:NetworkDevice {id: 'sw-access-01', name: 'Access-SW-01', ip_address: '10.0.1.1', type: 'access'})
CREATE (sw2:Switch:NetworkDevice {id: 'sw-access-02', name: 'Access-SW-02', ip_address: '10.0.1.2', type: 'access'})
CREATE (sw3:Switch:NetworkDevice {id: 'sw-core-01', name: 'Core-SW-01', ip_address: '10.0.0.1', type: 'core'})

// 네트워크 연결
CREATE (s1)-[:CONNECTED_TO {port: 1, speed_gbps: 10}]->(sw1)
CREATE (s2)-[:CONNECTED_TO {port: 2, speed_gbps: 10}]->(sw1)
CREATE (s3)-[:CONNECTED_TO {port: 1, speed_gbps: 10}]->(sw2)
CREATE (s4)-[:CONNECTED_TO {port: 2, speed_gbps: 10}]->(sw2)
CREATE (sw1)-[:LINKS {port1: 48, port2: 1, speed_gbps: 40}]->(sw3)
CREATE (sw2)-[:LINKS {port1: 48, port2: 2, speed_gbps: 40}]->(sw3)

// 서비스
CREATE (svc1:Service {id: 'svc-web', name: 'Web Service', type: 'frontend', criticality: 'critical'})
CREATE (svc2:Service {id: 'svc-api', name: 'API Gateway', type: 'api', criticality: 'critical'})
CREATE (svc3:Service {id: 'svc-db', name: 'Main Database', type: 'database', criticality: 'critical'})
CREATE (svc4:Service {id: 'svc-cache', name: 'Redis Cache', type: 'cache', criticality: 'high'})

// 서비스 배포
CREATE (s1)-[:RUNS {port: 80, cpu_allocated: 8, memory_allocated_gb: 32}]->(svc1)
CREATE (s2)-[:RUNS {port: 80, cpu_allocated: 8, memory_allocated_gb: 32}]->(svc1)
CREATE (s4)-[:RUNS {port: 8080, cpu_allocated: 4, memory_allocated_gb: 16}]->(svc2)
CREATE (s3)-[:RUNS {port: 5432, cpu_allocated: 32, memory_allocated_gb: 128}]->(svc3)
CREATE (s4)-[:RUNS {port: 6379, cpu_allocated: 2, memory_allocated_gb: 8}]->(svc4)

// 서비스 의존성
CREATE (svc1)-[:DEPENDS_ON {type: 'api_call'}]->(svc2)
CREATE (svc2)-[:DEPENDS_ON {type: 'data_read'}]->(svc3)
CREATE (svc2)-[:DEPENDS_ON {type: 'caching'}]->(svc4)
CREATE (svc4)-[:DEPENDS_ON {type: 'persistence'}]->(svc3)
```

---

## 시각화 예제

### 네트워크 토폴로지 조회

```cypher
// 전체 네트워크 토폴로지
MATCH (dc:Datacenter)-[:CONTAINS]->(rack:Rack)-[:HOUSES]->(server:Server)
OPTIONAL MATCH (server)-[:CONNECTED_TO]->(switch:Switch)
OPTIONAL MATCH (switch)-[:LINKS]-(connected_switch:Switch)
RETURN dc, rack, server, switch, connected_switch

// 서비스 의존성 그래프
MATCH (s:Service)
OPTIONAL MATCH (s)-[d:DEPENDS_ON]->(dep:Service)
RETURN s, d, dep
```

---

## 다음 단계

!!! success "네트워크 인프라 관리 완료!"
    [계보/족보 시스템](genealogy.md)에서 가족 관계 및 조직 계보 시스템을 구현해보세요.
