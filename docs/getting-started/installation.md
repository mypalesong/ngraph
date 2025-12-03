# 설치 가이드

## 그래프 데이터베이스 설치

### Neo4j

#### Docker (권장)

```bash
# 최신 버전
docker run -d \
  --name neo4j \
  -p 7474:7474 \
  -p 7687:7687 \
  -v $HOME/neo4j/data:/data \
  -v $HOME/neo4j/logs:/logs \
  -e NEO4J_AUTH=neo4j/your_password \
  -e NEO4J_PLUGINS='["apoc", "graph-data-science"]' \
  neo4j:5.15.0
```

#### 직접 설치 (Ubuntu)

```bash
# 저장소 추가
wget -O - https://debian.neo4j.com/neotechnology.gpg.key | sudo apt-key add -
echo 'deb https://debian.neo4j.com stable latest' | sudo tee /etc/apt/sources.list.d/neo4j.list

# 설치
sudo apt-get update
sudo apt-get install neo4j

# 서비스 시작
sudo systemctl start neo4j
sudo systemctl enable neo4j
```

### Apache Jena (RDF/SPARQL)

```bash
# 다운로드
wget https://dlcdn.apache.org/jena/binaries/apache-jena-4.10.0.tar.gz
tar -xzf apache-jena-4.10.0.tar.gz

# 환경 변수 설정
export JENA_HOME=/path/to/apache-jena-4.10.0
export PATH=$PATH:$JENA_HOME/bin

# Fuseki 서버 시작 (SPARQL endpoint)
cd apache-jena-fuseki-4.10.0
./fuseki-server --mem /dataset
```

### Amazon Neptune (AWS)

```bash
# AWS CLI로 Neptune 클러스터 생성
aws neptune create-db-cluster \
  --db-cluster-identifier my-neptune-cluster \
  --engine neptune \
  --vpc-security-group-ids sg-xxxxxxxx \
  --db-subnet-group-name my-subnet-group
```

## Python 환경 설정

### 가상 환경 생성

```bash
python -m venv kg-env
source kg-env/bin/activate  # Linux/Mac
# kg-env\Scripts\activate  # Windows
```

### 필수 패키지 설치

```bash
# requirements.txt
pip install neo4j>=5.0.0
pip install rdflib>=7.0.0
pip install sparqlwrapper>=2.0.0
pip install networkx>=3.0
pip install pyvis>=0.3.0
pip install pandas>=2.0.0
```

### 설치 확인

```python
# test_installation.py
from neo4j import GraphDatabase
from rdflib import Graph
import networkx as nx

print("Neo4j driver version:", neo4j.__version__)
print("RDFLib version:", rdflib.__version__)
print("NetworkX version:", nx.__version__)

# Neo4j 연결 테스트
try:
    driver = GraphDatabase.driver("bolt://localhost:7687",
                                  auth=("neo4j", "password"))
    driver.verify_connectivity()
    print("Neo4j 연결 성공!")
    driver.close()
except Exception as e:
    print(f"Neo4j 연결 실패: {e}")
```

## 개발 도구

### VS Code 확장

- **Neo4j for VS Code**: Cypher 문법 강조, 쿼리 실행
- **RDF/SPARQL**: RDF 파일 편집, SPARQL 쿼리
- **Draw.io Integration**: 그래프 다이어그램 작성

### Jupyter Notebook

```bash
pip install jupyter ipywidgets

# Neo4j 매직 커맨드 설치
pip install ipython-cypher
```

```python
# Jupyter에서 사용
%load_ext cypher
%cypher MATCH (n) RETURN n LIMIT 10
```

## 프로덕션 환경

### Neo4j 클러스터 설정

```yaml
# docker-compose.yml
version: '3.8'
services:
  core1:
    image: neo4j:5.15.0-enterprise
    environment:
      - NEO4J_AUTH=neo4j/password
      - NEO4J_MODE=CORE
      - NEO4J_ACCEPT_LICENSE_AGREEMENT=yes
      - NEO4J_initial_server_mode__constraint=PRIMARY
    ports:
      - "7474:7474"
      - "7687:7687"
    volumes:
      - core1_data:/data

  core2:
    image: neo4j:5.15.0-enterprise
    environment:
      - NEO4J_AUTH=neo4j/password
      - NEO4J_MODE=CORE
      - NEO4J_ACCEPT_LICENSE_AGREEMENT=yes
    volumes:
      - core2_data:/data

volumes:
  core1_data:
  core2_data:
```

## 다음 단계

환경 설정이 완료되었다면:

- [핵심 개념](../concepts/what-is-kg.md)을 학습하세요
- [그래프 데이터베이스](../implementation/graph-databases.md) 비교를 확인하세요
