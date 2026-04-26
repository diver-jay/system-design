# Week 1: Delivery Framework

> Hello Interview 핵심 처방: "Delivery Framework가 면접 채점 기준 자체"
> 이 섹션을 완전히 내면화하는 것이 가장 중요한 첫 번째 과제.

---

## 전체 구조 (45분 면접 기준)

```
Requirements      ~5분
Core Entities     ~2분
API Design        ~5분
[Data Flow]       ~5분 (선택, 데이터 처리 시스템만)
High Level Design ~10-15분
Deep Dives        ~10분
```

---

## 1. Requirements (~5분)

### Functional Requirements
"Users/Clients should be able to..." 형식으로 작성.
**핵심: 딱 3개만.** 많을수록 불리하다. 집중력 부재로 평가됨.

Twitter 예시:
- Users should be able to post tweets
- Users should be able to follow other users
- Users should be able to see tweets from users they follow

### Non-functional Requirements
"The system should be..." 형식. **반드시 수치화**.

나쁜 예: "The system should be low latency" (의미 없음, 모든 시스템이 그래야 함)
좋은 예: "The system should have low latency search, < 500ms"

**체크리스트 (상위 3-5개 선택):**
- CAP: Consistency vs Availability 중 어느 쪽?
- Scalability: bursty traffic? 읽기 vs 쓰기 비율?
- Latency: 특정 연산의 응답시간 목표?
- Durability: 데이터 손실 허용 수준?
- Fault Tolerance: 장애 복구 요구사항?
- Security: 규정 준수, 접근 제어?

### Capacity Estimation
**HI 핵심 조언: 대부분의 경우 생략하라.**

"계산은 설계 결정에 영향을 줄 때만 하라."
DAU, QPS 계산하고 "OK, it's a lot" 결론 내리는 건 시간 낭비.

언제 필요한가: 예) Top-K 시스템에서 단일 min-heap으로 처리 가능한지 or 샤딩이 필요한지 판단할 때.

---

## 2. Core Entities (~2분)

시스템의 핵심 명사들. 불릿 리스트로 빠르게.

**왜 전체 스키마를 지금 안 짜는가:** 설계하면서 새 엔티티가 발견됨. 먼저 나열하면 반드시 수정하게 됨.

Twitter 예시:
- User
- Tweet
- Follow

유용한 질문:
- 이 시스템의 행위자는 누구인가?
- Functional requirement를 충족하는 데 필요한 명사는?

---

## 3. API Design (~5분)

**프로토콜 선택:**
- REST: 기본값. 90%의 경우 이걸 써라.
- GraphQL: 클라이언트마다 다른 데이터가 필요할 때.
- gRPC: 서비스 간 내부 통신, 성능이 중요할 때.
- WebSocket/SSE: 실시간 기능.

**경고: API 설계에 10분 이상 쓰면 안 됨.** 4-5개 엔드포인트를 2분 안에 스케치하고 넘어가라.

Twitter REST 예시:
```
POST /v1/tweets          { "text": string }
GET  /v1/tweets/{id}  -> Tweet
POST /v1/follows         { "followee_id": string }
GET  /v1/feed         -> Tweet[]
```

**규칙:** user_id는 request body가 아니라 auth token에서 파생. 항상 복수 명사 리소스.

---

## 4. [Optional] Data Flow (~5분)

데이터 처리 시스템(Web Crawler, Ad Click Aggregator 등)에만 적용.
단순 리스트로.

Web Crawler 예시:
1. Fetch seed URLs
2. Parse HTML
3. Extract URLs
4. Store data
5. Repeat

일반 product 설계라면 건너뛰어라.

---

## 5. High Level Design (~10-15분)

핵심 원칙: **API 엔드포인트 하나씩 따라가며 설계를 쌓아라.**

- Client → LB → App Server → DB 흐름을 그리며 설명
- 각 요청마다 "어떤 상태가 바뀌는가" 명시
- DB에 도달했을 때 관련 컬럼/필드 옆에 바로 기록
- **모든 컬럼 기록 X.** 설계에 중요한 것만.

**흔한 실수:** 너무 일찍 복잡성 추가. 캐시, 메시지 큐 보이면 메모만 하고 Deep Dive로 넘기라.

---

## 6. Deep Dives (~10분)

Non-functional requirement를 만족시키는 단계.

다룰 주제들:
- 병목 지점 식별 및 해결
- 엣지 케이스
- 면접관의 프로빙(probe)에 대응

**시니어리티에 따라:**
- Junior: 면접관이 포인트를 짚어줌
- Senior: 본인이 먼저 문제를 찾아 리드해야 함

**실수:** 면접관이 끼어들 틈을 안 주는 것. 면접관도 특정 신호를 보내고 싶어함. 질문받을 공간 남겨라.

Twitter Deep Dive 예시:
- NFR "100M DAU 지원": 수평 확장, 캐시 도입, DB 샤딩 논의
- NFR "피드 저지연": fanout-on-read vs fanout-on-write 논의

---

## 암기 포인트

```
Requirements (5) → Entities (2) → API (5) → [DataFlow (5)] → HLD (15) → DeepDive (10)
```

면접 당일 처음 5분이 나머지를 결정한다. Requirements를 너무 많이 잡으면 시간이 폭발한다.
