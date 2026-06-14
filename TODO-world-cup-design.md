# 🏆 World Cup System Design - Deep Dive TODO

월드컵 라이브 스코어 시스템의 4대 핵심 난제(Why)를 Alex Xu 책의 이론과 연결하여 정복하는 학습 계획입니다.

## 🎯 학습 목표
- 월드컵과 같은 초고부하 실시간 시스템의 설계를 이론적으로 정당화할 수 있다.
- CAP 정리, Fan-out, Thundering Herd 등 핵심 아키텍처 패턴을 실전에 적용한다.

---

## 🗒️ 학습 리스트

### [ ] 1. 골 순간 Push 폭주 (Fan-out & Scalability)
- **핵심 난제**: 1,000만 명에게 동시에 골 소식을 지연 없이 전달하기.
- **공부할 챕터**: 
    - [ ] Alex Xu Ch 10. Design a Notification System
    - [ ] Alex Xu Ch 11. Design a News Feed System
- **체크포인트**:
    - Fan-out on write(Push)와 Fan-out on read(Pull)의 차이 이해하기.
    - 메시지 큐를 이용한 부하 분산 원리 파악.

### [ ] 2. 재연결 폭풍 (Reliability & Connection Management)
- **핵심 난제**: 터널/이동 중 끊긴 수만 명의 모바일 클라이언트가 동시에 재접속할 때의 서버 보호.
- **공부할 챕터**: 
    - [ ] Alex Xu Ch 4. Design a Rate Limiter
    - [ ] Alex Xu Ch 12. Design a Chat System
- **체크포인트**:
    - Exponential Backoff와 Jitter를 이용한 재접속 분산 원리.
    - Persistent Connection(SSE/WebSocket) 서버의 대규모 관리 기법.

### [ ] 3. Correctness & VAR (Consistency & Ordering)
- **핵심 난제**: VAR 판독 등으로 인한 점수 취소/정정 시 데이터 순서 뒤바뀜 방지.
- **공부할 챕터**: 
    - [ ] Alex Xu Ch 6. Design a Key-Value Store
    - [ ] Alex Xu Ch 7. Design a Unique ID Generator
- **체크포인트**:
    - Monotonic Read(단조 읽기)를 보장하기 위한 Sequence Number 활용법.
    - 최종 일관성(Eventual Consistency) 환경에서 정합성 유지하기.

### [ ] 4. 가용성 & Fallback (Availability & Fault Tolerance)
- **핵심 난제**: 결승전과 같은 단판 승부에서 시스템 일부가 죽어도 중단 없는 서빙.
- **공부할 챕터**: 
    - [ ] Alex Xu Ch 1. Scale from Zero to Millions of Users
    - [ ] Alex Xu Ch 6. Design a Key-Value Store (CAP Theorem)
- **체크포인트**:
    - AP(Availability-Partition tolerance) 시스템의 특성과 선택 이유.
    - 하이브리드 설계(SSE + Polling Fallback)를 통한 Graceful Degradation 이해.

---

## 🔗 참고 자료
- **교재**: `source/SystemDesignInterview.pdf`
- **설계 문서**: `docs/superpowers/specs/2026-06-14-live-score-system-design.md`
- **학습 노트**: `notes/TIL-2026-06-14-world-cup-architecture.md`
