# Live Sports Score System — Design Spec

> 학습용 설계 연습. 출발점 = `screen.png` (Google식 라이브 스코어 위젯, 2026 FIFA 월드컵).
> 작성일: 2026-06-14. Week 1 Core Concepts 지식 적용.

## 1. 문제 정의 & 스코프

화면 = 검색 결과 내 라이브 스포츠 위젯. 경기 카드(라이브 점수 `브라질 0-1 모로코 23'`, 풀타임, 예정), 탭(개요/경기/순위/대표팀/기록/선수), 하이라이트 썸네일, "실시간 점수 고정".

**설계 대상 (in-scope):** 실시간 스코어 전송 + backend
- 수집(ingestion) → 저장(storage) → fan-out(delivery)

**제외 (out-of-scope, §8에서 바운딩):**
- 검색창, 프론트엔드 렌더링, 순위/대표팀/선수 탭, 하이라이트 영상

## 2. 요구사항

### 기능
- 라이브 경기 점수/시계/상태(scheduled/live/halftime/full_time) 실시간 제공
- 골/카드/VAR 등 이벤트 반영
- 라이브 경기 목록 조회

### 비기능
- **실시간성**: 준실시간 — 골 후 ~1-2s 내 반영
- **규모**: 초대형 — 인기 경기 1개에 동시 시청자 **10M concurrent**
- **가용성 > 강한 일관성**: AP + eventual (단 monotonic read 보장 = 점수 역행 금지)

## 3. Back-of-the-Envelope (peak 블록버스터 경기 1개)

| 항목 | 값 | 근거 |
|------|-----|------|
| 동시 시청자 | 10M concurrent | 인기 경기 위젯 |
| 스코어 payload | ~500 B | 팀/점수/분/상태/최근이벤트 JSON |
| 상태 변경 빈도 | 골 희귀(~3/경기), 시계 ~5s | 골은 드물지만 빨라야 |
| 동시 라이브 경기 | 수개~수십 | 그룹스테이지 동시 킥오프 |

**부하 두 모양:**
- Polling: 10M ÷ 5s = **2M req/s** (한 경기), ~1 GB/s egress
- Push: 골 1번 = **10M 메시지 동시** (thundering herd)

**본질:** read-heavy + 소수 핫키(인기경기) + 작은 데이터 + 약간 stale OK.

## 4. Fan-out 접근법 비교 & 선택

| 접근 | 장점 | 단점 |
|------|------|------|
| A. Polling + CDN | 무한 확장, stateless, 단순 | 무변화에도 전송, latency 2~7s |
| B. SSE Push + Pub/Sub | 낮은 latency, 조용할 때 트래픽 0 | stateful 1000+ 노드, 재연결 폭풍, 골당 10M send |
| **C. Hybrid (선택)** | CDN 백본 + 핫매치 SSE | 두 경로 운영 |

**선택 = C (Hybrid).** CDN-polling을 백본으로 모든 경기·롱테일·fallback 처리, SSE는 라이브 핫매치에만 얹어 latency 절감. 원칙: "먼저 단순 → 수치로 정당화 → 핫패스에만 복잡성 추가".

**SSE 정당화의 load-bearing 조건:** provider 피드가 event-driven push(골 ~1s 내)일 때만. provider가 5-10s 폴링 피드뿐이면 상류가 병목 → **SSE 빼고 CDN-polling 단독이 정답**.

## 5. 하이레벨 아키텍처

```
[외부 스포츠 피드] (Sportradar/Stats Perform 류)
   │ push or poll
   ▼
[Ingestion Service]  정규화 + 경기당 monotonic seq 부여
   │ upsert (seq 가드)
   ▼
[State Store]  Postgres(영구 truth) ◄──► Redis(핫 상태, 읽기 authoritative)
   │ 변경 publish
   ▼
[Pub/Sub bus]  Kafka, partition key = match_id
   │
   ├──► [SSE Gateway 1000+ 노드] ──push──► 라이브 핫매치 client
   └──► [CDN edge] ◄──GET /match/{id}── 롱테일/백그라운드 client
```

### Latency budget (목표 골 후 ~1-2s)
| hop | 예산 | 메모 |
|-----|------|------|
| provider → ingest | 최대 변수 | 5-10s 피드면 여기서 막힘 → SSE 무의미 |
| ingest → store/publish | <100ms | 정규화+seq+upsert |
| publish → SSE push | <200ms | bus + gateway fan |
| publish → CDN | TTL 1-2s | polling 경로 |

## 6. 컴포넌트 상세

### 6.1 Ingestion
- **수신**: provider webhook push + polling reconcile(누락 보정)
- **정규화**: provider 포맷 → 내부 스키마
- **seq 부여**: 경기당 monotonic sequence number
  - 이유: VAR 골취소(0-1→0-0 정상), 피드 재시도/순서꼬임. stale가 최신 덮으면 점수 역행(correctness 버그).
  - 규칙: `incoming.seq > stored.seq`일 때만 적용, 낮으면 drop.
- **순서 보장**: Kafka partition key = `match_id` → 경기 내 순차, 경기 간 병렬
- **full-state snapshot 전송**(delta 아님): payload 작아 idempotent·순서꼬임 회피. cheap + correct > clever.
- **내고장성**: store가 truth, ingest는 stateless 변환기. 죽어도 피드 재생/reconcile로 복구.

### 6.2 Storage & 데이터 모델

2계층: Redis(핫, 읽기 authoritative) / Postgres(영구 truth).

```sql
-- Postgres
matches(
  match_id PK, tournament, group_name,
  home_team, away_team, home_score, away_score,
  status,            -- scheduled|live|halftime|full_time
  minute, kickoff_at, seq,   -- seq = 마지막 적용 버전
  updated_at
)
match_events(          -- append-only 감사 로그
  event_id PK, match_id FK, seq,
  type,                -- goal|card|var_revoke|period
  team, player, minute, payload, created_at
)
```

```
# Redis (full snapshot)
match:{id}   = JSON { home, away, score, minute, status, seq, recentEvents[3] }
live:matches = SET (현재 라이브 경기 id 목록)
```

- **읽기 authoritative = Redis**: 2M req/s 흡수, PG는 못 버팀. PG는 truth/감사/복원.
- **seq 가드(모든 쓰기)**: `if incoming.seq > stored.seq: apply else drop`. Redis는 Lua로 compare-set 원자화.
- **관계형 선택**: match 데이터 구조화·관계·양 적음 → Postgres, 샤딩 불필요(50TB·10k TPS 근처 안 감).
- **match_events**: 스냅샷 재구성·VAR 감사·하이라이트 타임라인·**순위 파생**(full_time 이벤트) 소스.

### 6.3 Delivery — CDN polling 백본
```
client --GET /v1/match/{id} (3-5s)--> CDN edge
  HIT(TTL 1-2s) → 즉시 / MISS → origin Read API → Redis snapshot
```
- TTL 1-2s → origin 부하 = 경기당 × 엣지POP수 × 1/TTL ≈ 수백 req/s
- `Cache-Control: max-age=1, stale-while-revalidate=5`
- 모든 경기·롱테일·모바일 백그라운드·SSE 미지원의 fallback. 항상 동작.

### 6.4 Delivery — SSE 가속 (라이브 핫매치만)
```
client --GET /v1/match/{id}/stream--> SSE Gateway
  1. 접속 즉시 Redis snapshot 1회        ← snapshot
  2. 이후 bus 이벤트 push(새 full snapshot) ← stream
  3. heartbeat
```
- **snapshot-then-stream**: 접속 즉시 현재 상태, 다음 이벤트 안 기다림.
- SSE `id:` = seq. 재연결 시 `Last-Event-ID: {seq}` → 이후만 재생. 클라 seq 가드로 역행 방지.
- **Gateway 티어**: 노드당 ~1만 conn → 10M = 1000+ 노드, stateful, L4 LB.
- **구독 모델**: 노드가 자기 conn이 보는 match_id만 구독 → 골 1번 = bus가 노드별 1메시지 → 각 노드 로컬 fan. bus 부하 작음.
- **켜는 기준**: 동시시청 임계 넘는 핫매치만 stream 광고, 나머지 polling.

## 7. Thundering Herd 방어 (3종)

### 7.1 Origin 미스 폭주 (cache stampede)
- **single-flight**: 동일 match_id 동시 미스 → 1개만 Redis 조회, 공유 (Mutex Lock)
- **stale-while-revalidate**: 만료중 옛값 + 1개만 갱신
- **TTL 지터**: 키별 ±랜덤 → 동시만료 분산

### 7.2 골 순간 push 폭주
- bus는 노드별 1메시지(10M 아님) → 안전. 마지막 노드→client fan만 부하
- gateway **마이크로배칭**(수십~수백ms 윈도우)으로 송신 압축. 준실시간이라 OK
- payload 작음 → 노드당 ~5MB 버스트 견딤. conn 늘면 노드 추가

### 7.3 재연결 폭풍 (2차 herd)
- 재접속 snapshot은 Redis에서 + origin single-flight 재사용(7.1)
- 클라 backoff + 지터 → 동시 복귀 방지
- gateway 롤링 드레인(점진)
- `Last-Event-ID`로 snapshot 스킵 가능 시 부하↓

### 7.4 핫 Redis 키
- 읽기 대부분 CDN/gateway 로컬에서 종료 → Redis 도달 적음
- 부족 시 read replica 팬아웃 또는 키 N개 복제(`match:{id}:{0..N}`)

## 8. 일관성 · 장애 · 글로벌 · Out-of-scope

### 8.1 일관성 (CAP)
- 라이브 스코어 = **AP + eventual**. 파티션 시 약간 stale > 에러/빈화면. 돈/재고 아님.
- **monotonic read 보장**: seq 가드로 점수 역행 금지. "eventual이되 뒤로 안 감".

### 8.2 장애 시나리오
| 죽는 것 | 영향 | 복구 |
|---------|------|------|
| Ingest worker | 신규 멈춤, 기존 서빙 | 재시작 + 피드 재생/reconcile |
| Redis 핫키 | 읽기 미스 | PG에서 복원·재워밍 |
| SSE gateway 노드 | 그 conn 끊김 | 재연결(지터)→다른 노드, 안되면 polling fallback |
| Pub/Sub bus | push 멈춤 | **CDN-polling 백본 계속 동작** |
| CDN POP | 라우팅 전환 | 다중 POP |

**핵심:** SSE/bus 전부 죽어도 polling 백본으로 degrade. 핫패스 복잡도가 기본 가용성 안 해침.

### 8.3 글로벌 배포
- 리전별 edge + gateway. Redis 핫상태 리전 복제(read replica) → 가까운 곳 읽기.
- Ingest/PG truth는 1차 리전, 비동기 복제(점수 eventual이라 OK). 뉴욕↔런던 ~80ms 고려.

### 8.4 Out-of-scope 바운딩
- **순위/대표팀/기록/선수 탭**: 느린·정적 → long-TTL CDN. 순위는 full_time 이벤트에서 파생 재계산.
- **하이라이트 영상**: 별도 VOD/스트리밍(YouTube식, Ch.14).
- **검색창**: 자동완성 Trie + Elasticsearch(Ch.13). 별도 서브시스템.
- **"실시간 점수 고정"**: 클라 UI 상태(구독할 match 선택). 백엔드 영향 = 해당 match 구독.

## 9. 노트 지식 연결 (복습 앵커)
- WebSocket vs SSE: 단방향 점수 push → SSE 적합(WebSocket 과함)
- L4 vs L7 LB: persistent 연결(SSE) → L4
- Cache-aside + stampede(mutex/probabilistic): §7.1, §7.3
- 관계형 vs NoSQL, 샤딩 불필요 정당화: §6.2
- CAP — AP + eventual, monotonic read: §8.1
- Numbers: 50TB·10k TPS·캐시 1ms vs DB 20-50ms: §6.2

## 10. 요약 한 줄
Hybrid(CDN-polling 백본 + 핫매치 SSE) + seq 버전드 full-state + 3종 herd 방어 + polling으로 degrade하는 가용성.
