# Numbers to Know (2026 기준 업데이트)

> HI 2026 핵심 메시지: "과거의 100GB 기준 샤딩은 잊어라. 현대 하드웨어는 단일 노드로 수십 테라바이트를 처리한다."

---

## 2026 현대 하드웨어 한계 (Single Instance)

| Component | Modern Limit (e.g. AWS x1e/i3en) | 면접 활용 포인트 |
|-----------|----------------------------------|----------------|
| **RAM (Memory)** | **24 TB** (U-24tb1.metal) | 웬만한 서비스의 '전체 활성 데이터셋'이 메모리 한 대에 다 들어감. |
| **SSD Storage** | **60 TB** (i3en.24xlarge) | 용량 때문에 샤딩할 일은 거의 없음. (Yelp 전체 데이터가 수백 GB 수준) |
| **Network BW** | **100 Gbps** | 네트워크가 새로운 병목. 서비스 간 통신 최적화(gRPC 등)가 중요해짐. |
| **CPU Cores** | **128+ vCPUs** | 병렬 처리 능력이 비약적으로 상승. |

---

## 컴포넌트별 성능 지표

| Component | Key Metrics | "진짜" 확장/샤딩 고려 시점 |
|-----------|-------------|----------------------|
| **Cache (Redis)** | ~1ms latency, 100k+ ops/sec | Hit rate < 80% 또는 **단일 코어 CPU 병목** 발생 시 |
| **Database (Postgres)** | **20k+ TPS (Write)**, 50k+ (Read) | **Sustained Write > 20k TPS** 또는 지역 분산 필요 시 |
| **Message Queue** | 1M+ msgs/sec/broker | Consumer Lag이 지속적으로 증가할 때 |

---

## 레이턴시 감각 (Cheat Sheet)

| Operation | Latency | 비고 |
|------|---------|------|
| Memory Access | 나노초 (ns) | - |
| **SSD Read (Row lookup)** | **1ms 미만 ~ 수 ms** | 인덱스 있으면 캐시 없이도 충분히 빠름 |
| Same AZ Network | < 1ms | 마이크로서비스 간 통신 기본값 |
| Cross-AZ Network | 1-2ms | 가용성(High Availability) 확보 비용 |
| **Cross-Region (Global)** | **50-150ms** | 뉴욕-런던 광속 한계. CDN/Edge Computing 필수 근거 |

---

## 면접에서 살아남는 "2026 논리"

### 1. "용량(Storage) 때문에 샤딩하지 않습니다"
- 60TB SSD는 텍스트 위주 DB(사용자, 포스트, 메타데이터)를 담기에 차고 넘칩니다.
- 샤딩의 진짜 이유는 **쓰기 처리량(Write Throughput)** 한계나 **장애 반경(Blast Radius) 격리**여야 합니다.

### 2. "캐시는 은탄환이 아닙니다"
- SSD가 충분히 빠르기 때문에, 단순 Primary Key 조회에 무조건 캐시를 붙이는 건 오버엔지니어링입니다.
- **무거운 쿼리(Join, Aggregation)**나 **초고빈도 핫스팟 데이터**에만 캐시를 제안하세요.

### 3. "계산은 정당화(Justification) 수단입니다"
- DAU로부터 QPS를 도출하는 목적은 "이 시스템이 복잡해져야 하는가?"를 결정하기 위함입니다.
- 결과가 10k TPS 미만이라면: **"단일 노드로 충분하므로 단순하게 설계하겠습니다"**라고 선언하는 것이 실력입니다.
