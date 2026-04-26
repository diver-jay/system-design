# System Design Study

Hello Interview Pro 기반 4주 커리큘럼. 주말 집중 학습.

## 폴더 구조

```
system-design/
├── notes/        # 주차별 정리
├── problems/     # 문제별 설계 솔루션
├── flashcards/   # 복습용 Q&A
└── mock-logs/    # 모의 인터뷰 기록
```

---

## 4주 커리큘럼

### Week 1: 기초 체계 잡기
> HI 처방: "Delivery Framework → Core Concepts" 순서

| 파일 | 내용 |
|------|------|
| [week-01-delivery-framework.md](notes/week-01-delivery-framework.md) | 면접 구조 (Requirements → Deep Dive) |
| [week-01-core-concepts.md](notes/week-01-core-concepts.md) | 9대 핵심 개념 |
| [week-01-numbers-to-know.md](notes/week-01-numbers-to-know.md) | 2026 하드웨어 수치 치트시트 |

**HI 페이지:**
- https://www.hellointerview.com/learn/system-design/in-a-hurry/delivery
- https://www.hellointerview.com/learn/system-design/in-a-hurry/core-concepts
- https://www.hellointerview.com/learn/system-design/core-concepts/numbers-to-know

---

### Week 2: Key Technologies 실전 이해
> 도메인 지식 확장. "어떤 기술을 왜 쓰는가"까지 설명할 수 있어야 함.

| 기술 | HI 페이지 | 우선순위 |
|------|-----------|---------|
| Redis | /deep-dives/redis | ★★★ |
| PostgreSQL | /deep-dives/postgres | ★★★ |
| Kafka | /deep-dives/kafka | ★★★ |
| Elasticsearch | /deep-dives/elasticsearch | ★★☆ |
| Cassandra | /deep-dives/cassandra | ★★☆ |
| DynamoDB | /deep-dives/dynamodb | ★★☆ |

---

### Week 3: Patterns + 기본 문제 3개 적용
> 패턴을 배우고 바로 문제에 적용.

**7대 패턴:**
- Real-time Updates
- Dealing with Contention
- Multi-step Processes
- Scaling Reads
- Scaling Writes
- Handling Large Blobs
- Managing Long Running Tasks

**연습 문제 3개 (쉬움 → 패턴 적용 연습):**
- [Bit.ly](https://www.hellointerview.com/learn/system-design/problem-breakdowns/bitly) — URL Shortener
- [Rate Limiter](https://www.hellointerview.com/learn/system-design/problem-breakdowns/distributed-rate-limiter) — 인프라 설계
- [YouTube Top K](https://www.hellointerview.com/learn/system-design/problem-breakdowns/top-k) — 데이터 처리

---

### Week 4: Advanced Practice Problems
> 실전 설계 + 모의 인터뷰 1회

**대규모 미디어/검색 시스템 (3개):**
- [YouTube](https://www.hellointerview.com/learn/system-design/problem-breakdowns/youtube) — 대규모 비디오 플랫폼
- [Web Crawler](https://www.hellointerview.com/learn/system-design/problem-breakdowns/web-crawler) — 검색 인덱싱
- [Ad Click Aggregator](https://www.hellointerview.com/learn/system-design/problem-breakdowns/ad-click-aggregator) — 광고 집계 시스템

**인프라 시스템 (1개):**
- [Distributed Cache](https://www.hellointerview.com/learn/system-design/problem-breakdowns/distributed-cache) — 분산 캐시 설계

**모의 인터뷰:** 위 4문제 중 1개 선택해 45분 실전 연습

---

## 학습 방식

1. HI 사이트에서 해당 섹션 읽기
2. notes/ 파일과 대조하며 이해 확인
3. 복습 질문으로 능동적 회상
4. problems/ 에 직접 설계해보기 (Week 3-4)
