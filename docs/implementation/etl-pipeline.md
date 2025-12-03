# ETL 파이프라인

## 개요

지식 그래프 구축을 위한 ETL(Extract, Transform, Load) 파이프라인 설계입니다.

## 파이프라인 구조

```mermaid
flowchart LR
    A[소스 데이터] --> B[추출]
    B --> C[변환]
    C --> D[로드]
    D --> E[그래프 DB]
```

## Python 구현

```python
import pandas as pd
from neo4j import GraphDatabase

class KGPipeline:
    def __init__(self, driver):
        self.driver = driver

    def extract(self, source):
        """데이터 추출"""
        return pd.read_csv(source)

    def transform(self, df):
        """그래프 형태로 변환"""
        nodes = []
        edges = []
        for _, row in df.iterrows():
            nodes.append({"label": "Entity", "props": dict(row)})
        return nodes, edges

    def load(self, nodes, edges):
        """그래프 DB에 적재"""
        with self.driver.session() as session:
            for node in nodes:
                session.run(
                    f"CREATE (n:{node['label']} $props)",
                    props=node['props']
                )
```

## Airflow DAG

```python
from airflow import DAG
from airflow.operators.python import PythonOperator

with DAG('kg_etl', schedule_interval='@daily') as dag:
    extract = PythonOperator(task_id='extract', python_callable=extract_data)
    transform = PythonOperator(task_id='transform', python_callable=transform_data)
    load = PythonOperator(task_id='load', python_callable=load_to_neo4j)

    extract >> transform >> load
```

## 다음 단계

- [성능 최적화](performance.md)에서 대용량 처리를 학습하세요
