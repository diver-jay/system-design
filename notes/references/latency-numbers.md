# Latency Numbers Everyone Should Know

> 출처: Google SRE / Jeff Dean  
> 설계 계산의 출발점. 외울 것이 아니라 **직관을 갖출 것**.

---

## 핵심 수치표

| Operation | ns | ms | 비고 |
|-----------|----|----|------|
| L1 cache reference | 1 | — | |
| Branch misprediction | 3 | — | |
| L2 cache reference | 4 | — | |
| Mutex lock/unlock | 17 | — | |
| Main memory reference | 100 | — | |
| Compress 1kB (Zippy) | 2,000 | 0.002 | |
| Read 1MB sequentially from memory | 10,000 | 0.010 | |
| Send 2kB over 10Gbps network | 1,600 | 0.0016 | |
| SSD 4kB random read | 20,000 | 0.020 | |
| Read 1MB sequentially from SSD | 1,000,000 | 1 | |
| Round trip within same datacenter | 500,000 | 0.5 | |
| Read 1MB sequentially from disk (HDD) | 5,000,000 | 5 | |
| Read 1MB sequentially from 1Gbps network | 10,000,000 | 10 | |
| Disk seek | 10,000,000 | 10 | 최대 병목 |
| TCP packet round trip between continents | 150,000,000 | 150 | 물리적 한계 |

---

## 순차 읽기 처리량

| 저장소 | 처리량 |
|--------|--------|
| HDD | ~200 MB/s |
| SSD | ~1 GB/s |
| Main memory | ~100 GB/s (burst) |
| 10Gbps Ethernet | ~1,000 MB/s |

데이터센터 내부: ~2,000 round trips/sec 가능  
유럽↔미국: 6–7 round trips/sec 한계

---

## 직관적 시간 비교 (1ns = 1초 가정)

| Operation | 실제 | 비유 |
|-----------|------|------|
| L1 cache | 1ns | 1초 (눈 깜빡임) |
| Main memory | 100ns | 1분 40초 (커피 준비) |
| SSD random read | 20,000ns | 5.5시간 (서울→부산) |
| Disk seek | 10,000,000ns | 115일 (계절이 바뀜) |
| Cross-continent | 150ms | 4.7년 (대학 학위) |

**메모리 vs 디스크**: 10만 배 차이

---

## QPS 계산 공식

```
단일 스레드 최대 QPS = 1 / Latency
전체 시스템 QPS    = (1 / Latency) × 병렬 스레드 수
```

### 1M QPS 달성에 필요한 자원

| 병목 | Latency | 단일 스레드 QPS | 1M QPS 달성 조건 |
|------|---------|----------------|-----------------|
| 메모리 조회 | 100ns | 10,000,000 | 단일 서버 캐시로 충분 |
| 데이터센터 네트워크 | 500μs | 2,000 | 500개 병렬 연결 필요 |
| 디스크 DB 조회 | 10ms | 100 | 10,000개 I/O → 샤딩/캐싱 필수 |

---

## Back-of-Envelope 예시

**문제**: 서버 1대에서 256kB 이미지 30장 조회 시 레이턴시?

```
Reads = 30 images / 2 disks = 15 reads
Time per image = (256kB / 1MB) × 5ms + 10ms seek = 11.28ms
Total = 15 × 11.28ms = 169.2ms
Throughput = 1000ms / 169.2ms ≈ 5 pages/sec
```

---

## 설계 원칙

1. **메모리에 넣어라** — 성능이 중요한 데이터는 디스크 대신 메모리에
2. **네트워크 왕복은 비싸다** — 같은 DC 0.5ms, 대륙 간 150ms
3. **순차 읽기가 우선** — 랜덤 I/O보다 배치/순차 처리가 항상 유리
4. **먼저 단순하게** — 수치로 정당화한 후에 복잡성 추가
