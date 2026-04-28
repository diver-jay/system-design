## 세션 시작 전 필수 복습: Latency Numbers

> 모든 학습 세션은 아래 수치를 확인한 후 시작한다.  
> 전체 레퍼런스: `notes/references/latency-numbers.md`

| Operation | ns | ms |
|-----------|----|----|
| L1 cache | 1 | — |
| Main memory | 100 | — |
| SSD 4kB random read | 20,000 | 0.02 |
| Datacenter round trip | 500,000 | 0.5 |
| Disk seek | 10,000,000 | 10 |
| Cross-continent TCP | 150,000,000 | 150 |

**핵심 직관**: 메모리 vs 디스크 = 10만 배 차이  
**설계 원칙**: 먼저 단순하게 → 수치로 정당화 → 복잡성 추가

---

## 📖 PDF 참고 가이드: System Design Interview (Alex Xu, 2nd Ed.)

**파일**: `source/SystemDesignInterview.pdf` (269 pages)  
**읽기**: PDF 읽기 시 반드시 페이지 범위 지정 (없으면 실패)

### 참고 트리거

1. **챕터 매핑 있는 토픽**: 아래 챕터 맵에 해당하면 자동으로 PDF 해당 챕터 읽기
2. **명시적 요청**: "책 참고해서", "책에서는", "Alex Xu 기준" 등 유저가 직접 요청할 때

### 챕터 맵

| Chapter | Topic | 키워드 |
|---------|-------|--------|
| Ch 1 | Scale from Zero to Millions of Users | 스케일링, 로드밸런서, 캐시, CDN, 수평/수직 확장, DB 복제 |
| Ch 2 | Back-of-the-Envelope Estimation | 추정, QPS, 스토리지 계산, 용량 계획, 2의 거듭제곱 |
| Ch 3 | Framework for System Design Interviews | 인터뷰 프레임워크, 4단계, 요구사항 명확화 |
| Ch 4 | Design a Rate Limiter | 처리율 제한, 토큰 버킷, 슬라이딩 윈도우, Leaky bucket |
| Ch 5 | Design Consistent Hashing | 일관성 해싱, 가상 노드, 해시 링 |
| Ch 6 | Design a Key-Value Store | 키-값 저장소, CAP 정리, 일관성 모델, Gossip |
| Ch 7 | Design a Unique ID Generator | 분산 ID, Snowflake, UUID, 티켓 서버 |
| Ch 8 | Design a URL Shortener | URL 단축, 해시 함수, 301 vs 302 리다이렉션 |
| Ch 9 | Design a Web Crawler | 웹 크롤러, BFS, politeness, 중복 제거 |
| Ch 10 | Design a Notification System | 알림 시스템, 푸시, 이메일, SMS |
| Ch 11 | Design a News Feed System | 뉴스피드, fanout-on-write, fanout-on-read |
| Ch 12 | Design a Chat System | 채팅, WebSocket, 메시지 동기화, 온라인 상태 |
| Ch 13 | Design a Search Autocomplete System | 검색 자동완성, Trie, typeahead |
| Ch 14 | Design YouTube | 비디오 스트리밍, CDN, 인코딩, DAG |
| Ch 15 | Design Google Drive | 파일 스토리지, 동기화, 블록 스토리지, 충돌 해결 |
| Ch 16 | The Learning Continues | 추가 학습 리소스 |

### 답변 규칙

- **챕터 매핑 있음**: PDF 해당 챕터를 읽고, 인용 시 `(Ch.X, p.YY)` 형식 사용
- **챕터 매핑 없음**: "이 토픽은 책에 포함되어 있지 않아 일반 지식으로 답변합니다" 먼저 명시 후 답변
- **강의식 금지**: 유저 질문에만 답변, 먼저 설명하거나 요약 나열하지 않기
