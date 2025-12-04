# Neo4j 운영 및 백업

## 백업과 복원

### 오프라인 백업 (Community)

```bash
# Neo4j 중지
sudo systemctl stop neo4j

# 데이터 디렉토리 복사
cp -r /var/lib/neo4j/data/databases/neo4j /backup/neo4j_$(date +%Y%m%d)

# Neo4j 시작
sudo systemctl start neo4j
```

### 온라인 백업 (Enterprise)

```bash
# 전체 백업
neo4j-admin database backup --to-path=/backup neo4j

# 증분 백업
neo4j-admin database backup --to-path=/backup --incremental neo4j
```

### 복원

```bash
# Neo4j 중지
sudo systemctl stop neo4j

# 복원
neo4j-admin database restore --from-path=/backup/neo4j-2024-01-15 neo4j

# Neo4j 시작
sudo systemctl start neo4j
```

### APOC 내보내기/가져오기

```cypher
// 전체 데이터베이스 내보내기
CALL apoc.export.cypher.all('/backup/full_backup.cypher', {format: 'cypher-shell'})

// 특정 노드만
CALL apoc.export.cypher.query(
  'MATCH (p:Person) RETURN p',
  '/backup/persons.cypher',
  {format: 'cypher-shell'}
)

// 복원
CALL apoc.cypher.runFile('/backup/full_backup.cypher')
```

---

## 보안 설정

### 인증 설정

```properties
# neo4j.conf
dbms.security.auth_enabled=true
```

### 사용자 관리

```cypher
// 사용자 생성
CREATE USER alice SET PASSWORD 'secure_password' SET PASSWORD CHANGE NOT REQUIRED

// 사용자 목록
SHOW USERS

// 비밀번호 변경
ALTER USER alice SET PASSWORD 'new_password'

// 사용자 삭제
DROP USER alice
```

### 역할 관리 (Enterprise)

```cypher
// 역할 생성
CREATE ROLE developer

// 권한 부여
GRANT MATCH {*} ON GRAPH * TO developer
GRANT CREATE ON GRAPH * NODES * TO developer

// 역할 할당
GRANT ROLE developer TO alice

// 권한 확인
SHOW ROLE developer PRIVILEGES
```

### 네트워크 보안

```properties
# neo4j.conf

# 특정 IP만 허용
server.default_listen_address=0.0.0.0
server.bolt.listen_address=:7687
server.http.listen_address=:7474

# SSL 설정
dbms.ssl.policy.bolt.enabled=true
dbms.ssl.policy.bolt.base_directory=/var/lib/neo4j/certificates
dbms.ssl.policy.bolt.private_key=private.key
dbms.ssl.policy.bolt.public_certificate=public.crt
```

---

## 클러스터 구성 (Enterprise)

### Causal Clustering

```yaml
# docker-compose.yml
version: '3.8'
services:
  core1:
    image: neo4j:5.15.0-enterprise
    environment:
      - NEO4J_AUTH=neo4j/password
      - NEO4J_ACCEPT_LICENSE_AGREEMENT=yes
      - NEO4J_initial_server_mode__constraint=PRIMARY
      - NEO4J_dbms_cluster_discovery_endpoints=core1:5000,core2:5000,core3:5000
    ports:
      - "7474:7474"
      - "7687:7687"

  core2:
    image: neo4j:5.15.0-enterprise
    environment:
      - NEO4J_AUTH=neo4j/password
      - NEO4J_ACCEPT_LICENSE_AGREEMENT=yes
      - NEO4J_initial_server_mode__constraint=PRIMARY
      - NEO4J_dbms_cluster_discovery_endpoints=core1:5000,core2:5000,core3:5000

  core3:
    image: neo4j:5.15.0-enterprise
    environment:
      - NEO4J_AUTH=neo4j/password
      - NEO4J_ACCEPT_LICENSE_AGREEMENT=yes
      - NEO4J_initial_server_mode__constraint=PRIMARY
      - NEO4J_dbms_cluster_discovery_endpoints=core1:5000,core2:5000,core3:5000
```

### 클러스터 상태 확인

```cypher
// 클러스터 개요
SHOW SERVERS

// 데이터베이스 상태
SHOW DATABASES

// 라우팅 테이블
CALL dbms.routing.getRoutingTable({}, 'neo4j')
```

---

## 모니터링

### 시스템 정보

```cypher
// 시스템 상태
CALL dbms.listConfig()
YIELD name, value
WHERE name STARTS WITH 'server.memory'
RETURN name, value

// 스토어 크기
CALL apoc.monitor.store()
YIELD logSize, stringStoreSize, nodeStoreSize, relStoreSize

// 트랜잭션 통계
CALL dbms.queryJmx('org.neo4j:instance=kernel#0,name=Transactions')
```

### 로그 설정

```properties
# neo4j.conf

# 쿼리 로그
db.logs.query.enabled=INFO
db.logs.query.threshold=2s
db.logs.query.parameter_logging_enabled=true

# 보안 로그
dbms.logs.security.level=INFO

# 디버그 로그
server.logs.debug.level=INFO
```

### 메트릭스 (Prometheus)

```properties
# neo4j.conf
server.metrics.enabled=true
server.metrics.prometheus.enabled=true
server.metrics.prometheus.endpoint=0.0.0.0:2004
```

---

## 유지보수

### 데이터베이스 일관성 검사

```bash
# Neo4j 중지 후 실행
neo4j-admin database check neo4j
```

### 인덱스 재구축

```cypher
// 인덱스 삭제 후 재생성
DROP INDEX person_name
CREATE INDEX person_name FOR (p:Person) ON (p.name)
```

### 스토어 압축

```bash
# 오프라인에서 실행
neo4j-admin database copy --compact-node-store neo4j neo4j_compact
```

### 로그 정리

```bash
# 오래된 트랜잭션 로그 삭제
find /var/lib/neo4j/data/transactions -name "*.tx" -mtime +7 -delete

# 또는 neo4j.conf에서 자동 정리
db.tx_log.rotation.retention_policy=2 days
```

---

## 재해 복구

### 복구 절차

```
1. 마지막 백업 확인
2. Neo4j 서비스 중지
3. 데이터 디렉토리 이동/삭제
4. 백업에서 복원
5. 일관성 검사
6. Neo4j 서비스 시작
7. 데이터 검증
```

### 자동 백업 스크립트

```bash
#!/bin/bash
# /etc/cron.daily/neo4j-backup

BACKUP_DIR=/backup/neo4j
DATE=$(date +%Y%m%d_%H%M%S)

# Enterprise: 온라인 백업
neo4j-admin database backup --to-path=$BACKUP_DIR/$DATE neo4j

# 7일 이상 된 백업 삭제
find $BACKUP_DIR -type d -mtime +7 -exec rm -rf {} \;

# 알림
echo "Neo4j backup completed: $DATE" | mail -s "Backup Complete" admin@example.com
```

---

## 업그레이드

### 업그레이드 절차

```bash
# 1. 백업
neo4j-admin database backup --to-path=/backup neo4j

# 2. Neo4j 중지
sudo systemctl stop neo4j

# 3. 새 버전 설치
sudo apt-get update
sudo apt-get install neo4j=5.16.0

# 4. 마이그레이션 (필요시)
neo4j-admin database migrate neo4j

# 5. Neo4j 시작
sudo systemctl start neo4j

# 6. 검증
cypher-shell -u neo4j -p password "RETURN 1"
```

---

## 다음 단계

!!! success "운영 가이드 완료!"
    [실전 프로젝트](real-world-projects.md)에서 종합 예제를 확인하세요.
