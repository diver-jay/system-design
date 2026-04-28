# Week 1 Deep Dive: 2026 HI Pro Insights

오늘 세션에서 Hello Interview Pro 계정 내용과 대조하며 도출한 심화 학습 포인트입니다.

---

## 1. Delivery Framework (Senior/Staff Signals)

### ⏱️ 시간 관리 및 전략
- **Functional Requirements**: 딱 3개만 잡는 것이 시간 관리와 집중력의 핵심.
- **Verbal Callout**: HLD 단계에서 캐시, 큐 등을 바로 그리지 말고 *"지연시간을 위해 캐시가 필요하지만, 골격 완성을 위해 나중에 다루겠다"*고 말로만 언급하고 지나가는 노련함.

### 🔐 Security (2026 필수)
- **Auth Token vs userId**: API 설계 시 `userId`를 요청 바디나 경로에 두지 말 것. 
- **IDOR 방지**: 서버가 토큰에서 직접 `userId`를 추출하여 권한을 확인하는 설계를 명시적으로 언급. (예: `PATCH /v1/me`)

### 📈 Observability (Operational Excellence)
- **3 Pillars**: Metrics, Logs, Traces.
- **시스템별 강조점**:
  - **Latency-sensitive**: Distributed Tracing (p99 latency).
  - **Write-heavy**: Queue Lag (Kafka Lag).
  - **Consistency-critical**: Audit Logs & DLQ.

---

## 2. Numbers to Know (2026 Hardware Reality)

현대 하드웨어는 과거의 교과서적 수치보다 훨씬 강력합니다.

- **RAM**: 단일 노드 최대 **24 TB**. (대부분의 활성 데이터셋을 메모리에 수용 가능)
- **SSD Storage**: 단일 노드 최대 **60 TB**. (용량 때문에 샤딩할 일은 거의 없음)
- **DB Write TPS**: 잘 튜닝된 단일 Postgres 노드는 **20k+ WPS** 처리 가능.
- **Kafka**: 브로커당 **1M+ msgs/sec**.

**💡 면접 논리**: "100GB니까 샤딩해야 한다"는 감점 요인. 반드시 예상 QPS와 용량을 수치로 계산하여 **"단일 노드로 충분하니 단순하게(Simplicity) 가겠다"**고 정당화할 것.

---

## 3. Scaling & Infrastructure Deep Dive

### Sharding vs. Message Queue
- **Message Queue (Buffering)**: '일시적인 스파크(Spike)' 대응. 큐가 터지는 것은 메모리 부족보다 **Consumer Lag**(DB 쓰기 속도가 느려 큐에 물이 차는 현상) 때문.
- **Sharding (Capacity)**: '지속적인 한계(Sustained Load)' 대응. 처리량 자체를 늘리는 궁극적 해결책.
- **상호보완**: 큐로 스파크를 막고, 샤딩으로 배수구(DB)를 넓혀 전체 처리량을 맞춤.

### Protocols (SSE vs. WebSocket)
- **SSE (Server-Sent Events)**: 서버→클라이언트 단방향. HTTP 기반으로 가볍고 자동 재연결 지원. 알림/피드 업데이트에 최적.
- **WebSocket**: 양방향 실시간. 상태 유지 비용이 크므로 클라이언트 상호작용이 많은 경우(채팅 등)에만 사용.

---

## 4. Database & Search
- **B-Tree Index**: 1억 명의 데이터 검색도 인덱스만 있다면 충분히 빠름.
- **Elasticsearch**: '오타 교정(Fuzzy Search)'이나 복잡한 형태소 분석이 필요할 때만 도입. (Simplicity 우선)
