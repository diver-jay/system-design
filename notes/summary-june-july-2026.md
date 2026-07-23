# 📚 6월~7월 시스템 디자인 & 아키텍처 학습 총정리

> **학습 기간**: 2026년 6월 1일 ~ 2026년 7월 31일  
> **주요 주제**: 마이크로서비스 아키텍처(MSA) 데이터 조회 패턴, 초고부하 라이브 스코어 시스템 설계, 클린 아키텍처(경계와 의존성 역전)

---

## 📌 목차
1. [6월 학습 내용: MSA 데이터 조회 패턴 & 시스템 설계](#1-6월-학습-내용-msa-데이터-조회-패턴--시스템-설계)
   - [1.1 MSA 데이터 조회 패턴 (API Composition & CQRS)](#11-msa-데이터-조회-패턴-api-composition--cqrs)
   - [1.2 Live Sports Score System Design (2026 FIFA World Cup)](#12-live-sports-score-system-design-2026-fifa-world-cup)
2. [7월 학습 내용: 클린 아키텍처 (Clean Architecture)](#2-7월-학습-내용-클린-아키텍처-clean-architecture)
   - [2.1 17장 경계: 선 긋기 (Boundaries: Drawing Lines)](#21-17장-경계-선-긋기-boundaries-drawing-lines)
3. [💡 연관 학습 문서 및 reference](#3-연관-학습-문서-및-reference)

---

## 1. 6월 학습 내용: MSA 데이터 조회 패턴 & 시스템 설계

### 1.1 MSA 데이터 조회 패턴 (API Composition & CQRS)
*관련 문서: [`notes/TIL-2026-06-04.md`](file:///home/junhonglee/dev/learning/system-design-public/notes/TIL-2026-06-04.md), [`notes/TIL-2026-06-06.md`](file:///home/junhonglee/dev/learning/system-design-public/notes/TIL-2026-06-06.md)*

#### 1) MSA 환경에서의 조회 문제 배경
- **모놀리식**: 단일 DB의 강력한 SQL `JOIN` 및 ACID 트랜잭션으로 강력한 실시간 일관성(Strong Consistency) 보장.
- **MSA**: **'Database per Service'** 원칙에 의해 DB가 격리되어 DB 레벨의 `JOIN` 불가능.

#### 2) API 조합(API Composition) 패턴과 두 가지 딜레마
- **API 조합기(Composer)**: 클라이언트/API 게이트웨이/BFF가 여러 서비스 provider를 병렬 호출하여 결과를 조립.
- **복잡한 필터링/정렬 시 발생하는 딜레마**:
  1. **인-메모리 조인 (In-memory Join)**: 모든 데이터를 조회하여 메모리에서 조인. CPU 연산은 `O(N+M)` 해시맵으로 줄일 수 있으나, 전송 데이터량(Payload) 폭발로 **Heap OOM(Out Of Memory)** 및 GC 병목 발생.
  2. **ID 기반 대량 조회**: 1차 조회의 ID 목록으로 핀셋 쿼리. 통신 데이터량은 줄지만 조건이 많을수록 **N+1 네트워크 트래픽 폭발**.
  3. **네트워크 지연(Latency)**: DB 내부 메모리 통신(ns/µs)과 달리 물리적 랜선/프로토콜 통신(ms)이므로 잦은 통신 시 성능에 치명적 결함 발생.

#### 3) 단일 서비스 내부에서의 조회 난관
- **OLTP vs OLAP 충돌**: 결제 서비스 등 메인 트랜잭션 처리 DB에 대용량 집계 쿼리를 던지면 DB Lock으로 실시간 서비스 장애 발생.
- **선택한 DB의 기술적 한계**: NoSQL(DynamoDB)의 지리 공간(Geospatial) 검색 한계, RDBMS의 시계열 통계 성능 저하 등.
- **관심사의 분리**: 데이터 소유권(CUD 무결성 관리)과 무거운 검색/조회(Query) 책임을 분리할 필요성 증대.

#### 4) CQRS (Command Query Responsibility Segregation) 패턴
- **핵심**: 상태 변경(Command, CUD) 통로와 상태 조회(Query, Read) 통로를 완벽히 분리.
- **구조**:
  - **Command 측**: CUD 전담, 비즈니스 규칙 검증 및 도메인 이벤트(Domain Event) 발행.
  - **Query 측**: 오직 Read 전담, 오직 쿼리 목적에 최적화된 읽기 전용 뷰 DB(Elasticsearch, Redis, 시계열 DB 등)를 비동기 이벤트 핸들러로 동기화.
- **타협점**: 완벽한 실시간 일관성을 포기하고 **최종 일관성(Eventual Consistency)**을 수용하여 압도적인 조회 성능, 가용성, 관심사 분리 달성.

---

### 1.2 Live Sports Score System Design (2026 FIFA World Cup)
*관련 문서: [`2026-06-14-live-score-system-design.md`](file:///home/junhonglee/dev/learning/system-design-public/2026-06-14-live-score-system-design.md), [`TODO-world-cup-design.md`](file:///home/junhonglee/dev/learning/system-design-public/TODO-world-cup-design.md)*

#### 1) 문제 정의 & 요구사항 (Scale & Latency)
- **동시 시청자**: 인기 경기 1개 기준 **10M Concurrent Users**.
- **목표 지연시간**: 골 발생 후 **1~2초 이내** 반영.
- **일관성 모델**: **AP + Eventual Consistency**, 단 **Monotonic Read(단조 읽기, 점수 역행 금지)** 필수 보장.

#### 2) Fan-out 아키텍처: Hybrid 접근법
- **Polling vs Push**: Polling은 10M / 5s = **2M req/s**, Push는 골 순간 **10M 동시 메시지 폭주 (Thundering Herd)**.
- **선택 = Hybrid**:
  - **CDN Polling 백본**: `Cache-Control: max-age=1, stale-while-revalidate=5`로 롱테일/일반 경기/fallback 처리.
  - **SSE(Server-Sent Events) 가속**: 인기 라이브 핫매치에만 선택적 스트리밍 적용. *(단, External Provider가 Event-Driven Push일 때만 SSE 정당화)*.

#### 3) 컴포넌트 설계 & 핵심 기법
- **Ingestion & Monotonic Sequence Number**:
  - 경기마다 증가하는 monotonic `seq` 부여.
  - VAR 골 취소, 피드 순서 꼬임 시 `incoming.seq > stored.seq`일 때만 적용하여 **점수 역행 방지**.
  - Kafka `partition_key = match_id`로 경기 내 순서 보장 및 스냅샷 기반 멱등(Idempotent) 처리.
- **Storage 2계층**:
  - **Redis**: 핫 상태 스냅샷 저장 및 2M req/s 흡수 (읽기 Authoritative).
  - **PostgreSQL**: 영구 Truth 저장소 및 `match_events` 감사 로그.
- **Thundering Herd 방어 3종 세트**:
  1. **Cache Stampede 방지**: Single-flight (Mutex Lock) & stale-while-revalidate 적용.
  2. **Push 버스트 방지**: Gateway micro-batching (수십 ms 윈도우 압축) 및 L4 LB 기반 SSE Gateway 노드 분산.
  3. **재연결 폭풍 방지**: Exponential Backoff + Jitter 적용, `Last-Event-ID`를 통한 스냅샷 스킵.

---

## 2. 7월 학습 내용: 클린 아키텍처 (Clean Architecture)

### 2.1 17장 경계: 선 긋기 (Boundaries: Drawing Lines)
*관련 문서: [`notes/TIL-2026-07-08.md`](file:///home/junhonglee/dev/learning/system-design-public/notes/TIL-2026-07-08.md)*

#### 1) 경계(Boundary)의 정의 및 목적
- 경계는 소프트웨어 요소를 분리하여 한쪽 요소가 반대쪽 요소를 알지 못하도록 차단하는 방어막.
- **핵심 비즈니스 로직(Business Rules)**과 **세부 사항(DB, UI, Framework 등)**을 분리.

#### 2) 선을 긋는 시점과 기준
- **시점**: 프로젝트의 **가장 초기 단계**(코드 작성 전)에 설계.
- **기준**: **변경의 축 (Axis of Change)**
  - *"서로 다른 시점에, 다른 속도로, 다른 이유로 변경된다면 그 사이에 선을 긋는다."* (SRP의 아키텍처 수준 확장).

#### 3) 기술적 구현: 인터페이스와 DIP (Dependency Inversion Principle)
- 소스 코드 의존성 화살표가 항상 **저수준 세부사항(플러그인) $\rightarrow$ 고수준 추상화(핵심 업무 규칙)**를 향하도록 배치.
- 업무 규칙 쪽에 인터페이스(계약)를 두고, 세부 사항(DB/UI)이 이를 구현하도록 작성.

#### 4) 결정을 미뤄야 하는 이유 (Delaying Decisions)
1. 초기는 프로젝트에 대해 가장 무지한 상태이므로 기술 결정을 확정짓는 위험성 회피.
2. 특정 DB/프레임워크에 결합되는 **기술 종속성(Lock-in)**과 매몰 비용 방지.
3. DB나 UI 없이도 순수 비즈니스 로직을 완벽하게 개발하고 테스트할 수 있음.

#### 5) 플러그인 아키텍처 (Plugin Architecture)
- **Business Rules**: 시스템의 심장이자 독립적인 주체.
- **DB / GUI / Framework**: 언제든 뺐다 꼈다 할 수 있는 부속품(플러그인).
> *"좋은 아키텍처는 결정을 미룰 수 있는 선택지를 최대한 많이, 오래 남겨두는 아키텍처다."*

---

## 3. 💡 연관 학습 문서 및 reference

- 📄 [`notes/TIL-2026-06-04.md`](file:///home/junhonglee/dev/learning/system-design-public/notes/TIL-2026-06-04.md) - MSA 데이터 조회 패턴 (기초)
- 📄 [`notes/TIL-2026-06-06.md`](file:///home/junhonglee/dev/learning/system-design-public/notes/TIL-2026-06-06.md) - MSA 데이터 조회 패턴 (심화/종합)
- 📄 [`2026-06-14-live-score-system-design.md`](file:///home/junhonglee/dev/learning/system-design-public/2026-06-14-live-score-system-design.md) - Live Sports Score System Design Spec
- 📄 [`TODO-world-cup-design.md`](file:///home/junhonglee/dev/learning/system-design-public/TODO-world-cup-design.md) - World Cup System Design TODO
- 📄 [`notes/TIL-2026-07-08.md`](file:///home/junhonglee/dev/learning/system-design-public/notes/TIL-2026-07-08.md) - 클린 아키텍처 17장 경계: 선 긋기
- 📄 [`notes/references/latency-numbers.md`](file:///home/junhonglee/dev/learning/system-design-public/notes/references/latency-numbers.md) - Latency Numbers Reference
