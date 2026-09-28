<img width="2016" height="1140" alt="case1_architecture" src="https://github.com/user-attachments/assets/c64e1314-8b75-408c-845b-22f1aaaecd9a" />
# Case 1. AI 데이터센터 대용량 수집: Scale-up → Scale-out

AI 데이터센터 수집 구조를 N+2 멀티 클러스터 Scale-out으로 전환해 **처리량 4.7배(초당 50만 → 235만 포인트)** 를 달성한 실무 사례입니다.

| 문제 | 설계 결정 | 트레이드오프 | 결과 |
|---|---|---|---|
| 초당 50만 포인트 구조의 병목·레이턴시 | N+2 수집 그룹 × 4개 클러스터로 무중단 Scale-out, ALB 재구성 | 서버 비용 **4배** | 처리량 **4.7배**, CPU **25% 이하** |


## 제약 조건
- **환경**: AI GW급 GPU 서버가 들어선 코로케이션 IDC. 설비가 **매초 수백만 포인트**의 운영 데이터를 만들어 냅니다.
- **문제**: 기존 구조의 처리 한계는 **초당 50만 포인트**였고, 데이터 병목과 레이턴시가 반복됐습니다.
- **요구**: 운영 중단 없이 초당 200만 포인트 이상을 처리할 수 있어야 했습니다.

## 검토한 대안
| 대안 | 방식 | 판단 |
|---|---|---|
| Scale-up | 서버 성능을 올리거나 한 서버에 Collector를 더 띄움 | 한 서버에 Collector를 추가해도 예상 80만 → **실측 70만 포인트/초**로, 성능이 비례해 늘지 않음. 원인은 OS 내부 Network I/O 처리 병목 |
| **Scale-out (채택)** | 수집 그룹을 클러스터 단위로 수평 확장 | 클러스터를 더하는 만큼 처리량이 늘고, 운영 중에도 무중단으로 확장 가능 |

## 설계 결정
- **N+2 수집 그룹 × 4개 클러스터**: 클러스터마다 Collector 3개 (개당 초당 약 20만, 클러스터당 약 60만 포인트)
- **ALB 재구성**: 늘어난 클러스터에 부하 분산
- **데이터 처리·저장 계층 HA**: Kafka 2-partition, Redis Shard/Replica, ClickHouse Replica

## 트레이드오프
- **비용**: 서버 비용 **4배** 증가
- **효과**: 처리량 **4.7배** 증가
- 비용 증가율보다 처리량 증가율이 커서 **포인트당 처리 비용은 약 15% 감소** (4 ÷ 4.7 ≈ 0.85)


## 개선 효과
| 지표 | Before | After |
|---|---|---|
| 처리량 | 초당 50만 포인트 | **초당 235만 포인트 (4.7배)** |
| CPU 사용률 | 병목 발생 | **25% 이하** (동시 Modbus TCP 세션 2,400개) |
| 확장 방식 | Scale-up (한계 도달) | **무중단 Scale-out** |

## 구조도 (Mermaid)
```mermaid
flowchart LR
    D[설비<br/>Modbus TCP 세션 2,400개] --> ALB[ALB 재구성]
    ALB --> C1[클러스터 #1<br/>N+2 · Collector ×3]
    ALB --> C2[클러스터 #2<br/>N+2 · Collector ×3]
    ALB --> C3[클러스터 #3<br/>N+2 · Collector ×3]
    ALB --> C4[클러스터 #4<br/>N+2 · Collector ×3]
    C1 & C2 & C3 & C4 --> P[Kafka 2-partition HA]
    P --> R[(Redis Shard/Replica)]
    P --> CH[(ClickHouse Replica)]
<img width="2542" height="1005" alt="case1_tradeoff" src="https://github.com/user-attachments/assets/33285c1a-b398-4085-a276-5bd2d773122c" />
<img width="2016" height="1140" alt="case1_architecture" src="https://github.com/user-attachments/assets/dbe2e3c1-7587-46de-b03a-6a2f39b1db20" />
