# Neo4j 설치 및 설정

## 설치 방법 선택

| 방법 | 장점 | 적합한 용도 |
|------|------|------------|
| Docker | 빠른 시작, 격리된 환경 | 개발, 테스트 |
| Desktop | GUI 관리, 다중 DB | 학습, 로컬 개발 |
| 직접 설치 | 완전한 제어 | 프로덕션 |
| AuraDB | 관리형 서비스 | 클라우드 프로덕션 |

---

## 1. Docker 설치 (권장)

### 기본 실행

```bash
# 최신 버전 실행
docker run -d \
    --name neo4j \
    -p 7474:7474 \
    -p 7687:7687 \
    -e NEO4J_AUTH=neo4j/your_password \
    neo4j:latest

# 버전 지정
docker run -d \
    --name neo4j \
    -p 7474:7474 \
    -p 7687:7687 \
    -e NEO4J_AUTH=neo4j/your_password \
    neo4j:5.15.0
```

### 데이터 영속화

```bash
# 볼륨 마운트로 데이터 보존
docker run -d \
    --name neo4j \
    -p 7474:7474 \
    -p 7687:7687 \
    -v $HOME/neo4j/data:/data \
    -v $HOME/neo4j/logs:/logs \
    -v $HOME/neo4j/import:/var/lib/neo4j/import \
    -v $HOME/neo4j/plugins:/plugins \
    -e NEO4J_AUTH=neo4j/your_password \
    neo4j:latest
```

### 플러그인 포함 실행

```bash
# APOC + GDS 플러그인 포함
docker run -d \
    --name neo4j \
    -p 7474:7474 \
    -p 7687:7687 \
    -v $HOME/neo4j/data:/data \
    -e NEO4J_AUTH=neo4j/your_password \
    -e NEO4J_PLUGINS='["apoc", "graph-data-science"]' \
    -e NEO4J_dbms_security_procedures_unrestricted=apoc.*,gds.* \
    -e NEO4J_dbms_security_procedures_allowlist=apoc.*,gds.* \
    neo4j:latest
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  neo4j:
    image: neo4j:5.15.0
    container_name: neo4j
    ports:
      - "7474:7474"  # HTTP
      - "7687:7687"  # Bolt
    volumes:
      - neo4j_data:/data
      - neo4j_logs:/logs
      - neo4j_import:/var/lib/neo4j/import
      - neo4j_plugins:/plugins
    environment:
      - NEO4J_AUTH=neo4j/your_secure_password
      - NEO4J_PLUGINS=["apoc", "graph-data-science"]
      - NEO4J_dbms_memory_heap_initial__size=512m
      - NEO4J_dbms_memory_heap_max__size=2G
      - NEO4J_dbms_memory_pagecache_size=1G
    restart: unless-stopped

volumes:
  neo4j_data:
  neo4j_logs:
  neo4j_import:
  neo4j_plugins:
```

실행:
```bash
docker-compose up -d
docker-compose logs -f neo4j
```

---

## 2. Neo4j Desktop

### 다운로드 및 설치

1. [Neo4j Desktop 다운로드](https://neo4j.com/download/)
2. 설치 파일 실행
3. 활성화 키 입력 (무료 등록 필요)

### 프로젝트 및 데이터베이스 생성

```
1. "New Project" 클릭
2. 프로젝트 이름 입력 (예: "Knowledge Graph")
3. "Add" → "Local DBMS" 선택
4. 이름, 비밀번호 설정
5. Neo4j 버전 선택
6. "Create" 클릭
7. "Start" 버튼으로 시작
```

### 플러그인 설치

```
1. 데이터베이스 클릭
2. "Plugins" 탭 선택
3. APOC, Graph Data Science 등 "Install" 클릭
4. 데이터베이스 재시작
```

---

## 3. 직접 설치 (Linux)

### Ubuntu/Debian

```bash
# 저장소 키 추가
wget -O - https://debian.neo4j.com/neotechnology.gpg.key | sudo apt-key add -

# 저장소 추가
echo 'deb https://debian.neo4j.com stable latest' | sudo tee /etc/apt/sources.list.d/neo4j.list

# 설치
sudo apt-get update
sudo apt-get install neo4j

# 서비스 시작
sudo systemctl start neo4j
sudo systemctl enable neo4j

# 상태 확인
sudo systemctl status neo4j
```

### CentOS/RHEL

```bash
# 저장소 설정
sudo rpm --import https://debian.neo4j.com/neotechnology.gpg.key

cat <<EOF | sudo tee /etc/yum.repos.d/neo4j.repo
[neo4j]
name=Neo4j RPM Repository
baseurl=https://yum.neo4j.com/stable/5
enabled=1
gpgcheck=1
EOF

# 설치
sudo yum install neo4j

# 시작
sudo systemctl start neo4j
```

### 설정 파일

```bash
# /etc/neo4j/neo4j.conf

# 네트워크 설정
server.default_listen_address=0.0.0.0
server.bolt.listen_address=:7687
server.http.listen_address=:7474

# 메모리 설정
server.memory.heap.initial_size=512m
server.memory.heap.max_size=2G
server.memory.pagecache.size=1G

# 보안
dbms.security.auth_enabled=true

# 플러그인
dbms.security.procedures.unrestricted=apoc.*,gds.*
```

---

## 4. Neo4j AuraDB (클라우드)

### 무료 티어 생성

1. [Neo4j Aura](https://neo4j.com/cloud/aura/) 접속
2. "Start Free" 클릭
3. Google/GitHub 계정으로 가입
4. "New Instance" → "AuraDB Free" 선택
5. 리전 선택, 인스턴스 이름 입력
6. 생성된 연결 정보 저장

### 연결 정보

```
Connection URI: neo4j+s://xxxxxxxx.databases.neo4j.io
Username: neo4j
Password: (생성 시 제공)
```

### Python 연결

```python
from neo4j import GraphDatabase

URI = "neo4j+s://xxxxxxxx.databases.neo4j.io"
AUTH = ("neo4j", "your_password")

driver = GraphDatabase.driver(URI, auth=AUTH)
driver.verify_connectivity()
print("연결 성공!")
```

---

## 설치 확인

### 1. 웹 브라우저 접속

```
http://localhost:7474
```

- 사용자명: `neo4j`
- 비밀번호: 설정한 비밀번호

### 2. Cypher Shell

```bash
# Docker
docker exec -it neo4j cypher-shell -u neo4j -p your_password

# 직접 설치
cypher-shell -u neo4j -p your_password
```

### 3. 첫 쿼리 실행

```cypher
// 버전 확인
CALL dbms.components() YIELD name, versions
RETURN name, versions;

// 테스트 노드 생성
CREATE (n:Test {message: 'Hello Neo4j!'})
RETURN n;

// 확인
MATCH (n:Test) RETURN n;

// 삭제
MATCH (n:Test) DELETE n;
```

---

## 문제 해결

### 포트 충돌

```bash
# 사용 중인 포트 확인
sudo lsof -i :7474
sudo lsof -i :7687

# 다른 포트 사용
docker run -p 17474:7474 -p 17687:7687 ...
```

### 메모리 부족

```bash
# JVM 힙 크기 조정
NEO4J_dbms_memory_heap_max__size=1G

# 또는 neo4j.conf에서
server.memory.heap.max_size=1G
```

### 연결 실패

```bash
# 방화벽 확인
sudo ufw allow 7474
sudo ufw allow 7687

# Docker 네트워크 확인
docker network ls
docker inspect neo4j
```

---

## 다음 단계

!!! success "설치 완료!"
    [Neo4j Browser 사용법](browser.md)에서 그래프 탐색과 시각화를 배워보세요.
