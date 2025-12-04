# 사기 탐지 시스템 구현 가이드

## 프로젝트 개요

Neo4j 기반 금융 사기 탐지 시스템을 구축하는 완전한 가이드입니다.

### 구현 목표
- 실시간 거래 모니터링
- 순환 거래 탐지
- 이상 패턴 탐지
- 네트워크 기반 사기 탐지
- 위험 점수 계산

---

## 1단계: 데이터 모델

### 금융 거래 스키마

```cypher
// 제약조건
CREATE CONSTRAINT account_id FOR (a:Account) REQUIRE a.id IS UNIQUE;
CREATE CONSTRAINT customer_id FOR (c:Customer) REQUIRE c.id IS UNIQUE;
CREATE CONSTRAINT transaction_id FOR (t:Transaction) REQUIRE t.id IS UNIQUE;
CREATE CONSTRAINT device_id FOR (d:Device) REQUIRE d.fingerprint IS UNIQUE;

// 인덱스
CREATE INDEX transaction_timestamp FOR (t:Transaction) ON (t.timestamp);
CREATE INDEX transaction_status FOR (t:Transaction) ON (t.status);
CREATE INDEX account_status FOR (a:Account) ON (a.status);

// 노드 타입
(:Customer {
  id: STRING,
  name: STRING,
  email: STRING,
  phone: STRING,
  ssn: STRING,          // 암호화
  dateOfBirth: DATE,
  address: STRING,
  riskScore: FLOAT,
  kycStatus: STRING,
  createdAt: DATETIME
})

(:Account {
  id: STRING,
  type: STRING,         // checking, savings, credit
  balance: FLOAT,
  currency: STRING,
  status: STRING,       // active, frozen, closed
  openedAt: DATETIME,
  lastActivityAt: DATETIME
})

(:Transaction {
  id: STRING,
  amount: FLOAT,
  currency: STRING,
  type: STRING,         // transfer, withdrawal, deposit, payment
  status: STRING,       // pending, completed, failed, flagged
  timestamp: DATETIME,
  description: STRING,
  channel: STRING,      // online, mobile, atm, branch
  riskScore: FLOAT
})

(:Device {
  fingerprint: STRING,
  type: STRING,         // mobile, desktop, tablet
  os: STRING,
  browser: STRING,
  ip: STRING,
  location: POINT,
  firstSeen: DATETIME,
  lastSeen: DATETIME,
  trusted: BOOLEAN
})

(:Merchant {
  id: STRING,
  name: STRING,
  category: STRING,
  riskLevel: STRING,    // low, medium, high
  location: STRING
})

(:Alert {
  id: STRING,
  type: STRING,
  severity: STRING,
  description: STRING,
  createdAt: DATETIME,
  status: STRING,       // open, investigating, resolved, false_positive
  resolvedAt: DATETIME
})

// 관계
(:Customer)-[:OWNS]->(:Account)
(:Account)-[:SENT]->(t:Transaction)-[:RECEIVED]->(:Account)
(:Transaction)-[:USING]->(:Device)
(:Transaction)-[:AT]->(:Merchant)
(:Customer)-[:USES]->(:Device)
(:Transaction)-[:TRIGGERED]->(:Alert)
(:Alert)-[:ASSIGNED_TO]->(:Analyst)
```

---

## 2단계: 사기 탐지 규칙

### 규칙 기반 탐지 서비스

```python
# services/rule_based_detection.py
from typing import List, Dict, Optional
from datetime import datetime, timedelta

class RuleBasedDetection:
    def __init__(self, db):
        self.db = db

    def check_velocity(self, account_id: str, hours: int = 1,
                      threshold: int = 5) -> Optional[Dict]:
        """거래 속도 검사: 짧은 시간 내 다수 거래"""
        query = """
        MATCH (a:Account {id: $accountId})-[:SENT]->(t:Transaction)
        WHERE t.timestamp > datetime() - duration('PT' + $hours + 'H')
          AND t.status IN ['pending', 'completed']
        WITH a, count(t) AS txCount, sum(t.amount) AS totalAmount,
             collect(t) AS transactions
        WHERE txCount >= $threshold
        RETURN {
            accountId: a.id,
            ruleType: 'velocity',
            severity: CASE
                WHEN txCount >= $threshold * 2 THEN 'high'
                WHEN txCount >= $threshold * 1.5 THEN 'medium'
                ELSE 'low' END,
            details: {
                transactionCount: txCount,
                totalAmount: totalAmount,
                timeWindowHours: $hours,
                threshold: $threshold
            }
        } AS alert
        """
        with self.db.session() as session:
            result = session.run(query,
                accountId=account_id, hours=str(hours), threshold=threshold)
            record = result.single()
            return dict(record['alert']) if record else None

    def check_amount_anomaly(self, account_id: str,
                            multiplier: float = 3.0) -> Optional[Dict]:
        """금액 이상 검사: 평균 대비 비정상적으로 큰 거래"""
        query = """
        MATCH (a:Account {id: $accountId})-[:SENT]->(t:Transaction)
        WHERE t.status = 'completed'
          AND t.timestamp > datetime() - duration('P90D')

        WITH a, avg(t.amount) AS avgAmount, stDev(t.amount) AS stdAmount

        MATCH (a)-[:SENT]->(recent:Transaction)
        WHERE recent.timestamp > datetime() - duration('PT24H')
          AND recent.status = 'pending'
          AND recent.amount > avgAmount + ($multiplier * stdAmount)

        RETURN {
            accountId: a.id,
            ruleType: 'amount_anomaly',
            severity: CASE
                WHEN recent.amount > avgAmount * 10 THEN 'critical'
                WHEN recent.amount > avgAmount * 5 THEN 'high'
                ELSE 'medium' END,
            details: {
                transactionId: recent.id,
                amount: recent.amount,
                averageAmount: avgAmount,
                standardDeviation: stdAmount,
                zScore: (recent.amount - avgAmount) / stdAmount
            }
        } AS alert
        LIMIT 1
        """
        with self.db.session() as session:
            result = session.run(query,
                accountId=account_id, multiplier=multiplier)
            record = result.single()
            return dict(record['alert']) if record else None

    def check_new_device(self, customer_id: str,
                        transaction_id: str) -> Optional[Dict]:
        """새 디바이스 검사: 처음 보는 디바이스에서 거래"""
        query = """
        MATCH (c:Customer {id: $customerId})-[:OWNS]->(a:Account)-[:SENT]->(t:Transaction {id: $txId})
        MATCH (t)-[:USING]->(d:Device)

        // 기존에 사용한 디바이스인지 확인
        OPTIONAL MATCH (c)-[:USES]->(knownDevice:Device)
        WHERE knownDevice.fingerprint = d.fingerprint

        WITH c, t, d, knownDevice
        WHERE knownDevice IS NULL

        // 위치 변화 확인
        OPTIONAL MATCH (c)-[:USES]->(lastDevice:Device)
        WHERE lastDevice.trusted = true
        WITH c, t, d,
             point.distance(d.location, lastDevice.location) / 1000 AS distanceKm

        RETURN {
            customerId: c.id,
            transactionId: t.id,
            ruleType: 'new_device',
            severity: CASE
                WHEN distanceKm > 1000 THEN 'high'
                WHEN distanceKm > 100 THEN 'medium'
                ELSE 'low' END,
            details: {
                deviceFingerprint: d.fingerprint,
                deviceType: d.type,
                ip: d.ip,
                distanceFromLastKnownKm: distanceKm
            }
        } AS alert
        """
        with self.db.session() as session:
            result = session.run(query,
                customerId=customer_id, txId=transaction_id)
            record = result.single()
            return dict(record['alert']) if record else None

    def check_time_anomaly(self, account_id: str,
                          transaction_id: str) -> Optional[Dict]:
        """시간 이상 검사: 비정상적인 시간대 거래"""
        query = """
        MATCH (a:Account {id: $accountId})-[:SENT]->(t:Transaction {id: $txId})

        // 사용자의 일반적인 거래 시간대
        MATCH (a)-[:SENT]->(past:Transaction)
        WHERE past.status = 'completed'
          AND past.timestamp > datetime() - duration('P30D')
        WITH a, t,
             avg(past.timestamp.hour) AS avgHour,
             stDev(past.timestamp.hour) AS stdHour

        WITH a, t, avgHour, stdHour,
             abs(t.timestamp.hour - avgHour) AS hourDiff

        WHERE hourDiff > stdHour * 2 OR t.timestamp.hour IN [0,1,2,3,4,5]

        RETURN {
            accountId: a.id,
            transactionId: t.id,
            ruleType: 'time_anomaly',
            severity: CASE
                WHEN t.timestamp.hour IN [2,3,4] THEN 'high'
                ELSE 'medium' END,
            details: {
                transactionHour: t.timestamp.hour,
                averageHour: avgHour,
                deviation: hourDiff
            }
        } AS alert
        """
        with self.db.session() as session:
            result = session.run(query,
                accountId=account_id, txId=transaction_id)
            record = result.single()
            return dict(record['alert']) if record else None
```

### 네트워크 기반 탐지 서비스

```python
# services/network_detection.py
from typing import List, Dict

class NetworkBasedDetection:
    def __init__(self, db):
        self.db = db

    def detect_circular_transactions(self, min_amount: float = 10000,
                                    max_hops: int = 5) -> List[Dict]:
        """순환 거래 탐지: A→B→C→...→A"""
        query = """
        MATCH path = (start:Account)-[:SENT]->(:Transaction)-[:RECEIVED]->(hop1:Account)
                     -[:SENT*1..""" + str(max_hops-1) + """]->(:Transaction)-[:RECEIVED]->(start)

        // 모든 거래가 기준 금액 이상
        WITH path, nodes(path) AS pathNodes, relationships(path) AS rels
        WHERE ALL(r IN rels WHERE
            (r:SENT OR r:RECEIVED) OR
            (r.amount >= $minAmount)
        )

        // 거래들 추출
        WITH path,
             [n IN pathNodes WHERE n:Transaction] AS transactions,
             [n IN pathNodes WHERE n:Account] AS accounts

        WHERE size(transactions) >= 2

        RETURN {
            ruleType: 'circular_transaction',
            severity: CASE
                WHEN size(accounts) >= 4 THEN 'critical'
                WHEN size(accounts) >= 3 THEN 'high'
                ELSE 'medium' END,
            details: {
                accountIds: [a IN accounts | a.id],
                transactionIds: [t IN transactions | t.id],
                hopCount: size(accounts) - 1,
                totalAmount: reduce(sum = 0.0, t IN transactions | sum + t.amount)
            }
        } AS alert
        LIMIT 100
        """
        with self.db.session() as session:
            result = session.run(query, minAmount=min_amount)
            return [dict(r['alert']) for r in result]

    def detect_structuring(self, account_id: str,
                          threshold: float = 10000,
                          window_hours: int = 24) -> Optional[Dict]:
        """구조화 탐지: 보고 기준 금액 바로 아래로 분할 거래"""
        query = """
        MATCH (a:Account {id: $accountId})-[:SENT]->(t:Transaction)
        WHERE t.timestamp > datetime() - duration('PT' + $hours + 'H')
          AND t.amount >= $threshold * 0.8
          AND t.amount < $threshold

        WITH a, collect(t) AS suspiciousTx,
             sum(t.amount) AS totalAmount,
             count(t) AS txCount

        WHERE txCount >= 3 AND totalAmount >= $threshold

        RETURN {
            accountId: a.id,
            ruleType: 'structuring',
            severity: CASE
                WHEN totalAmount >= $threshold * 3 THEN 'critical'
                WHEN totalAmount >= $threshold * 2 THEN 'high'
                ELSE 'medium' END,
            details: {
                transactionCount: txCount,
                totalAmount: totalAmount,
                averageAmount: totalAmount / txCount,
                threshold: $threshold,
                transactions: [t IN suspiciousTx | {id: t.id, amount: t.amount}]
            }
        } AS alert
        """
        with self.db.session() as session:
            result = session.run(query,
                accountId=account_id, threshold=threshold, hours=str(window_hours))
            record = result.single()
            return dict(record['alert']) if record else None

    def detect_shared_device_network(self, min_accounts: int = 3) -> List[Dict]:
        """공유 디바이스 네트워크: 여러 계정이 같은 디바이스 사용"""
        query = """
        MATCH (d:Device)<-[:USING]-(:Transaction)<-[:SENT]-(a:Account)
        WITH d, collect(DISTINCT a) AS accounts
        WHERE size(accounts) >= $minAccounts

        // 관련 계정들의 거래 패턴 확인
        UNWIND accounts AS acc
        MATCH (acc)-[:SENT]->(t:Transaction)
        WHERE t.timestamp > datetime() - duration('P7D')
        WITH d, accounts,
             sum(t.amount) AS totalAmount,
             count(t) AS totalTx

        RETURN {
            ruleType: 'shared_device_network',
            severity: CASE
                WHEN size(accounts) >= 5 THEN 'critical'
                WHEN size(accounts) >= 4 THEN 'high'
                ELSE 'medium' END,
            details: {
                deviceFingerprint: d.fingerprint,
                deviceIp: d.ip,
                accountCount: size(accounts),
                accountIds: [a IN accounts | a.id],
                recentTransactionCount: totalTx,
                recentTotalAmount: totalAmount
            }
        } AS alert
        ORDER BY size(accounts) DESC
        LIMIT 50
        """
        with self.db.session() as session:
            result = session.run(query, minAccounts=min_accounts)
            return [dict(r['alert']) for r in result]

    def detect_rapid_fund_movement(self, account_id: str,
                                   hours: int = 24) -> Optional[Dict]:
        """급속 자금 이동: 입금 직후 출금"""
        query = """
        MATCH (a:Account {id: $accountId})

        // 최근 입금
        MATCH (a)<-[:RECEIVED]-(inTx:Transaction)
        WHERE inTx.timestamp > datetime() - duration('PT' + $hours + 'H')
          AND inTx.amount >= 5000

        // 입금 직후 출금
        MATCH (a)-[:SENT]->(outTx:Transaction)
        WHERE outTx.timestamp > inTx.timestamp
          AND outTx.timestamp < inTx.timestamp + duration('PT2H')
          AND outTx.amount >= inTx.amount * 0.8

        WITH a, inTx, outTx,
             duration.between(inTx.timestamp, outTx.timestamp).minutes AS gapMinutes

        RETURN {
            accountId: a.id,
            ruleType: 'rapid_fund_movement',
            severity: CASE
                WHEN gapMinutes < 10 THEN 'critical'
                WHEN gapMinutes < 30 THEN 'high'
                ELSE 'medium' END,
            details: {
                incomingTx: {id: inTx.id, amount: inTx.amount, time: inTx.timestamp},
                outgoingTx: {id: outTx.id, amount: outTx.amount, time: outTx.timestamp},
                gapMinutes: gapMinutes,
                amountRatio: outTx.amount / inTx.amount
            }
        } AS alert
        LIMIT 1
        """
        with self.db.session() as session:
            result = session.run(query,
                accountId=account_id, hours=str(hours))
            record = result.single()
            return dict(record['alert']) if record else None
```

---

## 3단계: 위험 점수 계산

```python
# services/risk_scoring.py
class RiskScoringService:
    def __init__(self, db):
        self.db = db

    def calculate_transaction_risk(self, transaction_id: str) -> Dict:
        """거래 위험 점수 계산"""
        query = """
        MATCH (t:Transaction {id: $txId})
        MATCH (sender:Account)-[:SENT]->(t)-[:RECEIVED]->(receiver:Account)
        OPTIONAL MATCH (t)-[:USING]->(d:Device)
        OPTIONAL MATCH (t)-[:AT]->(m:Merchant)

        // 금액 점수 (0-25)
        WITH t, sender, receiver, d, m,
             CASE
                 WHEN t.amount > 50000 THEN 25
                 WHEN t.amount > 20000 THEN 20
                 WHEN t.amount > 10000 THEN 15
                 WHEN t.amount > 5000 THEN 10
                 ELSE 5
             END AS amountScore

        // 시간 점수 (0-20)
        WITH t, sender, receiver, d, m, amountScore,
             CASE
                 WHEN t.timestamp.hour IN [0,1,2,3,4,5] THEN 20
                 WHEN t.timestamp.hour IN [22,23] THEN 10
                 ELSE 0
             END AS timeScore

        // 디바이스 점수 (0-20)
        WITH t, sender, receiver, d, m, amountScore, timeScore,
             CASE
                 WHEN d IS NULL THEN 15
                 WHEN d.trusted = false THEN 20
                 WHEN d.firstSeen > datetime() - duration('P7D') THEN 15
                 ELSE 0
             END AS deviceScore

        // 상대방 점수 (0-20)
        WITH t, sender, receiver, d, m, amountScore, timeScore, deviceScore,
             CASE
                 WHEN receiver.status = 'frozen' THEN 20
                 WHEN receiver.riskScore > 0.7 THEN 15
                 WHEN NOT exists((sender)-[:SENT]->()-[:RECEIVED]->(receiver)) THEN 10
                 ELSE 0
             END AS counterpartyScore

        // 가맹점 점수 (0-15)
        WITH t, amountScore, timeScore, deviceScore, counterpartyScore,
             CASE
                 WHEN m IS NULL THEN 5
                 WHEN m.riskLevel = 'high' THEN 15
                 WHEN m.riskLevel = 'medium' THEN 8
                 ELSE 0
             END AS merchantScore

        WITH t,
             amountScore + timeScore + deviceScore + counterpartyScore + merchantScore AS totalScore

        SET t.riskScore = totalScore / 100.0

        RETURN {
            transactionId: t.id,
            riskScore: t.riskScore,
            riskLevel: CASE
                WHEN totalScore >= 70 THEN 'critical'
                WHEN totalScore >= 50 THEN 'high'
                WHEN totalScore >= 30 THEN 'medium'
                ELSE 'low' END,
            components: {
                amount: amountScore,
                time: timeScore,
                device: deviceScore,
                counterparty: counterpartyScore,
                merchant: merchantScore
            }
        } AS result
        """
        with self.db.session() as session:
            result = session.run(query, txId=transaction_id)
            record = result.single()
            return dict(record['result']) if record else None

    def calculate_account_risk(self, account_id: str) -> Dict:
        """계정 위험 점수 계산"""
        query = """
        MATCH (a:Account {id: $accountId})
        MATCH (c:Customer)-[:OWNS]->(a)

        // 최근 알림 수
        OPTIONAL MATCH (a)-[:SENT]->(:Transaction)-[:TRIGGERED]->(alert:Alert)
        WHERE alert.createdAt > datetime() - duration('P30D')
        WITH a, c, count(alert) AS alertCount

        // 고위험 거래 비율
        MATCH (a)-[:SENT]->(t:Transaction)
        WHERE t.timestamp > datetime() - duration('P30D')
        WITH a, c, alertCount,
             count(t) AS totalTx,
             sum(CASE WHEN t.riskScore > 0.5 THEN 1 ELSE 0 END) AS highRiskTx

        // 네트워크 위험도
        OPTIONAL MATCH (a)-[:SENT]->()-[:RECEIVED]->(connected:Account)
        WHERE connected.status = 'frozen'
        WITH a, c, alertCount, totalTx, highRiskTx,
             count(DISTINCT connected) AS frozenConnections

        WITH a,
             alertCount * 10 +
             (toFloat(highRiskTx) / CASE WHEN totalTx > 0 THEN totalTx ELSE 1 END) * 40 +
             frozenConnections * 15 +
             CASE WHEN c.kycStatus <> 'verified' THEN 20 ELSE 0 END AS riskScore

        SET a.riskScore = riskScore / 100.0

        RETURN {
            accountId: a.id,
            riskScore: a.riskScore,
            riskLevel: CASE
                WHEN riskScore >= 70 THEN 'critical'
                WHEN riskScore >= 50 THEN 'high'
                WHEN riskScore >= 30 THEN 'medium'
                ELSE 'low' END
        } AS result
        """
        with self.db.session() as session:
            result = session.run(query, accountId=account_id)
            record = result.single()
            return dict(record['result']) if record else None
```

---

## 4단계: 실시간 모니터링

```python
# services/realtime_monitor.py
import asyncio
from datetime import datetime
from typing import Callable

class RealtimeMonitor:
    def __init__(self, db):
        self.db = db
        self.rule_detection = RuleBasedDetection(db)
        self.network_detection = NetworkBasedDetection(db)
        self.risk_scoring = RiskScoringService(db)

    async def process_transaction(self, transaction_id: str,
                                 on_alert: Callable) -> Dict:
        """새 거래 처리 및 검사"""

        # 1. 위험 점수 계산
        risk_result = self.risk_scoring.calculate_transaction_risk(transaction_id)

        # 2. 규칙 기반 검사
        alerts = []

        # 거래 정보 조회
        tx_info = self._get_transaction_info(transaction_id)
        account_id = tx_info['sender_account_id']
        customer_id = tx_info['customer_id']

        # 속도 검사
        velocity_alert = self.rule_detection.check_velocity(account_id)
        if velocity_alert:
            alerts.append(velocity_alert)

        # 금액 이상 검사
        amount_alert = self.rule_detection.check_amount_anomaly(account_id)
        if amount_alert:
            alerts.append(amount_alert)

        # 디바이스 검사
        device_alert = self.rule_detection.check_new_device(customer_id, transaction_id)
        if device_alert:
            alerts.append(device_alert)

        # 시간 검사
        time_alert = self.rule_detection.check_time_anomaly(account_id, transaction_id)
        if time_alert:
            alerts.append(time_alert)

        # 구조화 검사
        structuring_alert = self.network_detection.detect_structuring(account_id)
        if structuring_alert:
            alerts.append(structuring_alert)

        # 급속 이동 검사
        rapid_alert = self.network_detection.detect_rapid_fund_movement(account_id)
        if rapid_alert:
            alerts.append(rapid_alert)

        # 3. 알림 생성
        for alert in alerts:
            self._create_alert(transaction_id, alert)
            await on_alert(alert)

        # 4. 고위험 시 거래 보류
        if risk_result['riskLevel'] in ['critical', 'high'] or len(alerts) >= 2:
            self._flag_transaction(transaction_id)

        return {
            'transactionId': transaction_id,
            'riskResult': risk_result,
            'alerts': alerts,
            'action': 'flagged' if risk_result['riskLevel'] in ['critical', 'high'] else 'approved'
        }

    def _get_transaction_info(self, tx_id: str) -> Dict:
        query = """
        MATCH (c:Customer)-[:OWNS]->(a:Account)-[:SENT]->(t:Transaction {id: $txId})
        RETURN {
            sender_account_id: a.id,
            customer_id: c.id,
            amount: t.amount
        } AS info
        """
        with self.db.session() as session:
            result = session.run(query, txId=tx_id)
            return dict(result.single()['info'])

    def _create_alert(self, tx_id: str, alert_data: Dict):
        query = """
        MATCH (t:Transaction {id: $txId})
        CREATE (a:Alert {
            id: randomUUID(),
            type: $alertType,
            severity: $severity,
            description: $description,
            createdAt: datetime(),
            status: 'open'
        })
        CREATE (t)-[:TRIGGERED]->(a)
        RETURN a
        """
        with self.db.session() as session:
            session.run(query,
                txId=tx_id,
                alertType=alert_data['ruleType'],
                severity=alert_data['severity'],
                description=str(alert_data['details'])
            )

    def _flag_transaction(self, tx_id: str):
        query = """
        MATCH (t:Transaction {id: $txId})
        SET t.status = 'flagged'
        """
        with self.db.session() as session:
            session.run(query, txId=tx_id)
```

---

## 5단계: FastAPI 통합

```python
# main.py
from fastapi import FastAPI, BackgroundTasks
from pydantic import BaseModel

app = FastAPI(title="Fraud Detection API")

monitor = RealtimeMonitor(db)

class TransactionEvent(BaseModel):
    transaction_id: str

async def alert_callback(alert: Dict):
    # 알림 시스템 연동 (Slack, Email 등)
    print(f"🚨 Alert: {alert['ruleType']} - {alert['severity']}")

@app.post("/transactions/check")
async def check_transaction(event: TransactionEvent, background_tasks: BackgroundTasks):
    """새 거래 검사"""
    result = await monitor.process_transaction(
        event.transaction_id,
        alert_callback
    )
    return result

@app.get("/accounts/{account_id}/risk")
def get_account_risk(account_id: str):
    """계정 위험 점수 조회"""
    return risk_scoring.calculate_account_risk(account_id)

@app.get("/alerts/circular-transactions")
def get_circular_alerts(min_amount: float = 10000):
    """순환 거래 탐지"""
    return network_detection.detect_circular_transactions(min_amount)

@app.get("/alerts/shared-devices")
def get_shared_device_alerts(min_accounts: int = 3):
    """공유 디바이스 네트워크 탐지"""
    return network_detection.detect_shared_device_network(min_accounts)
```

---

## 다음 단계

[지식 그래프 RAG](kg-rag.md)에서 LLM과 Neo4j를 통합하는 방법을 확인하세요.
