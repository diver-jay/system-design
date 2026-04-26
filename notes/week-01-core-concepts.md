# Week 1: Core Concepts

> Hello Interview의 실제 Core Concepts 9개 기반. 이전 파일 완전 대체.
> 각 개념의 "면접에서 언제, 어떻게 쓰는가"에 집중.

---

## 1. Networking Essentials

### 프로토콜 선택 원칙
- **HTTP/REST**: 기본값. 90% 커버.
- **gRPC**: 내부 서비스 간 통신, 성능 중요할 때. 바이너리 직렬화 + HTTP/2.
- **WebSocket**: 양방향 실시간 통신 필요할 때만 (채팅, 협업 도구).
- **SSE (Server-Sent Events)**: 서버→클라이언트 단방향 실시간. 더 단순. (알림, 라이브 피드)

**면접 실수:** WebSocket은 SSE로 충분할 때 과도하게 쓰인다. WebSocket은 클라이언트가 서버로도 데이터를 자주 보낼 때만 필요.

### Load Balancer
- **L7 (Application)**: HTTP 내용 기반 라우팅 가능. API vs 웹페이지 분리 가능.
- **L4 (TCP)**: 더 빠르지만 내용 모름. **WebSocket은 L4 필요** (persistent TCP connection).

### 레이턴시 현실
- 뉴욕→런던: 광섬유 기준 최소 80ms (처리 전).
- 글로벌 저레이턴시 → 리전별 배포 + CDN 필수.

---

## 2. API Design

**면접 팁:** 4-5개 핵심 엔드포인트를 2분 안에. 더 이상 쓰지 마라.

- **REST**: 기본값.
- **페이지네이션**: 대용량 결과셋은 필수. cursor-based(실시간 데이터) vs offset-based(일반).
- **인증**: 사용자 세션 → JWT. 서비스 간 → API Key.
- **Rate Limiting**: 남용 방어. 면접관이 물어볼 때만 깊이 파라.

현재 사용자는 항상 auth token에서 파생. request body의 user_id 신뢰 금지.

---

## 3. Data Modeling

### 관계형 vs NoSQL 선택

| | 관계형 (PostgreSQL) | NoSQL (DynamoDB, Cassandra) |
|--|--|--|
| 언제 | 구조화된 데이터, 복잡한 관계, 강한 일관성 | 유연한 스키마, 수평 확장, 단순 액세스 패턴 |
| 장점 | SQL 복잡 쿼리, 트랜잭션, 외래키 | 대규모 수평 확장, 빠른 단순 조회 |
| 단점 | 수평 확장 어려움 | Join 불가, 액세스 패턴 미리 설계해야 함 |

### 정규화 vs 비정규화

- **정규화**: 데이터 중복 없음, Join 필요, 업데이트 단순.
- **비정규화**: Join 없이 빠른 읽기, 업데이트 시 여러 곳 수정 필요.

**면접 기본값:** 정규화 시작 → 핫 패스에서 읽기 성능 문제 발생 시 비정규화.

### DynamoDB 설계 원칙
액세스 패턴 먼저 정의 → partition key, sort key 결정.
"유저 X의 모든 포스트" → user_id를 partition key로.
"해시태그 Y 포함 포스트" → 별도 인덱스(GSI) 필요.

---

## 4. Database Indexing

### B-tree Index (기본값)
- 정렬된 트리 구조.
- Exact match + Range query 둘 다 지원.
- 대부분의 DB에서 기본 인덱스.

### 특수 인덱스
- **Hash Index**: Exact match만. Range 불가. 더 빠름.
- **Full-text Index**: 문서 내 단어 검색 (Elasticsearch).
- **Geospatial Index**: 위치 기반 쿼리 (PostGIS).

### 면접 패턴
- 자주 쿼리하는 필드 → 인덱스 제안.
- 전문 검색 필요 → Elasticsearch (CDC로 primary DB와 싱크).
- CDC 싱크는 약간의 지연 발생 → 검색 인덱스는 약간 stale 가능. 이건 acceptable.

---

## 5. Caching

**성능 차이:** DB 쿼리 20-50ms vs 캐시 조회 < 1ms.

### Cache-Aside (기본값, 90%)
```
Read: Cache miss → DB 조회 → Cache 저장 (TTL 설정) → 반환
Write: DB 업데이트 → Cache 무효화
```

### 무효화 전략
- **TTL**: 단순. 약간의 stale 허용.
- **Event-based**: DB 변경 시 즉시 삭제. 정확하지만 복잡.
- 보통 둘을 조합.

### Cache Stampede
TTL 만료 시 대량 요청이 동시에 DB 직격 → 시스템 다운 가능.
해결: Mutex Lock, Probabilistic Early Expiration.

### 언제 캐시 안 써도 되나
단순 인덱스 row lookup은 이미 서브 밀리초~수 밀리초. 캐시 불필요한 경우 많음.
자주 읽히고 잘 안 바뀌는 데이터에만 적용.

---

## 6. Sharding

### 언제 샤딩하는가
- **스토리지**: 50TB 근접
- **쓰기 처리량**: 10k TPS 초과 (단일 DB 한계)

먼저 단일 DB + read replica로 충분한지 확인하라.

### Shard Key 선택이 핵심

| 전략 | 방법 | 장단점 |
|------|------|-------|
| **Hash Sharding** | hash(key) % N | 균등 분배, 샤드 추가 시 재분배 필요 |
| **Range Sharding** | 키 범위별 | 범위 쿼리 효율적, Hot spot 위험 |
| **Consistent Hashing** | 링 구조 | 샤드 추가/제거 시 이동 최소화 |

Instagram 예시: user_id 샤드 키 → 유저별 쿼리 빠름, 글로벌 trending 쿼리는 느림.

### 면접 포인트
샤딩 필요성을 수치로 먼저 정당화하라. 그 다음 shard key 선택과 trade-off를 설명.

---

## 7. Consistent Hashing

**풀려는 문제:** 일반 hash(key) % N에서 서버 추가/제거 시 거의 모든 키가 재배치됨.

**해결:** 서버와 키를 가상 링에 배치. 키는 시계 방향 다음 서버에 할당.
- 서버 추가: 해당 범위의 키만 이동 (~1/N)
- 서버 제거: 해당 서버의 키만 다음 서버로 이동

**사용처:** Redis Cluster, Cassandra, DynamoDB, 일부 CDN.

**면접에서:** 동작 원리 설명보다 "consistent hashing 사용해 elastic scaling 시 데이터 이동 최소화"라고 말하는 게 충분한 경우가 많다.

---

## 8. CAP Theorem

**핵심:** 분산 시스템에서 네트워크 파티션은 불가피. 실제 선택은 **CP vs AP**.

| 선택 | 특징 | 예시 |
|------|------|------|
| CP | 파티션 시 일부 요청 거부 (가용성 포기), 데이터 항상 정확 | HBase, Zookeeper |
| AP | 파티션 시도 응답, 일시적 stale 가능 | Cassandra, DynamoDB |

### Consistency 수준
- **Eventual**: 언젠가 동기화됨 (SNS 피드, 추천)
- **Strong**: 모든 노드 즉시 동일 (은행, 재고, 예약)

**면접 기본답:** Eventual consistency. 단, 돈/재고/좌석 예약 → Strong consistency.

**PACELC:** 파티션 없을 때도 Latency vs Consistency 트레이드오프 존재.

---

## 9. Numbers to Know

→ [week-01-numbers-to-know.md](week-01-numbers-to-know.md) 참조

**요약:** 단일 DB 50TiB, 캐시 1TB, Kafka 1M msgs/sec. 먼저 단순하게, 수치로 정당화 후 복잡성 추가.

---

## 복습 질문

1. WebSocket vs SSE: 언제 각각 쓰는가?
2. 정규화 → 비정규화 언제 전환하는가?
3. Cache Stampede란 무엇이고 어떻게 막는가?
4. Consistent Hashing이 일반 modulo hashing보다 나은 이유는?
5. 단일 Postgres DB를 샤딩해야 하는 수치 기준은?
6. AP vs CP: 재고 시스템은 어느 쪽이고 이유는?
