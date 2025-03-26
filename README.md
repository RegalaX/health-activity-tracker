---
### 과제 요구사항을 분석한 문서를 KB헬스케어_과제_분석자료.pdf 로 정리하였고, 이를 기반으로 설계를 진행하였습니다.
### 현재 이용 중인 네트워크 보안 정책으로 인해 Gmail 첨부가 불가능하여, 압축 파일을 GitHub에 업로드했습니다.
### KB헬스케어_백엔드_과제_제출_최종현.zip 파일을 다운로드 부탁드립니다.
---

# Health Activity Tracker

사용자의 **건강 활동 정보**(예: 걸음 수, 칼로리, 거리)를 **모바일 앱에서 서버로 전송**받아 저장하고, 이를 **일별(Daily) 및 월별(Monthly) 통계로 조회**할 수 있는 백엔드 시스템입니다.

## 📌 주요 기능

- **활동 데이터 수집**: 단말에서 전송되는 활동 데이터(JSON)를 API로 수신
- **Redis Stream 기반 비동기 처리**: 활동 데이터를 Redis Stream에 전송하고, Consumer가 DB에 저장
- **걸음 수, 칼로리, 거리 집계**: 저장된 데이터를 기준으로 일별 및 월별 통계 조회
- **Dead Letter Queue 처리**: 실패한 메시지를 DLQ에 저장하여 유실 방지
- **Fallback 처리**: Redis 전송 실패 시 DB로 직접 저장

---

## ⚙️ 기술 스택

| 기술       | 설명 |
|------------|------|
| Java 17    | 주요 개발 언어 |
| Spring Boot 3 | 프레임워크 |
| Spring Data JPA | ORM 및 DB 접근 |
| MySQL 8.x  | 활동 데이터 저장소 (파티셔닝 포함) |
| Redis Streams | 메시지 큐 역할 |
| Docker(Optional) | 로컬 테스트 시 사용 가능 |

---

## 🗂️ 주요 구성

### 1. API 컨트롤러

| 경로 | 설명 |
|------|------|
| `POST /api/activity` | 활동 데이터를 JSON 형태로 수신 |
| `GET /api/statistics/daily` | recordKey별 일별 통계 조회 |
| `GET /api/statistics/monthly` | recordKey별 월별 통계 조회 |

### 2. Redis Stream 처리

- `ActivityStreamProducer`: API에서 수신된 데이터를 Redis Stream에 전송
- `ActivityStreamConsumer`: Stream에서 데이터를 읽어 DB에 저장
- `Dead letter 처리`: 실패 시 `activity.stream.dead.letter`로 전송됨

  #### ✅ DLQ 테스트 방법
  Dead Letter Queue 처리가 잘 되는지 확인하려면 의도적으로 예외를 발생시켜 테스트할 수 있습니다.

  예를 들어 `ActivityStreamHandler` 또는 `DLQStreamHandler`의 `process()` 메서드를 아래와 같이 수정합니다:

  ```java
  @Override
  public void process(ActivityPayload payload) {
      throw new RuntimeException("Dead letter processing test");
      // 정상 처리 시:
      // recordService.saveActivityFromPayload(payload);
  }


### 3. DB 테이블
#### 3-1. `activity_record` (활동 데이터의 고유 단위)

사용자가 단말기에서 전송한 데이터 묶음을 표현하는 테이블입니다.  
동일한 단말기 및 동일한 전송건에 대해 중복 저장을 방지하기 위해 `record_key`를 사용합니다.

| 컬럼명 | 설명                                      |
|--------|-----------------------------------------|
| `record_key` | 각 요청 건에 대해 생성되는 **고유 식별 키**로 중복 방지 역할   |
| `source_id` | 어떤 단말(기기, 앱 등)에서 발생한 데이터인지 구분하는 외래 키    |
| `activity_type` | 활동 종류 예: `STEPS`.... 등                  |
| `last_updated_at` | 단말 기준 마지막 데이터 수집 시점                     |
| `created_at` | 서버 수신 시각                                |
| `memo` | Apple HealthKit에서만 제공되는 주석 등 (Optional) |

> 📝 하나의 `record_key`에는 수십~수천 개의 `entry`가 연결될 수 있습니다.

---

#### 3-2. `activity_record_source` (데이터의 출처)

모바일 단말의 제조사, 모드, 기기명 등 **데이터 수집 출처 정보를 정규화**하여 관리하는 테이블입니다.

| 컬럼명 | 설명 |
|--------|------|
| `name`, `mode`, `type` | 출처를 식별하는 주요 정보 |
| `product_name` | 기기 이름 (예: Galaxy Watch 5) |
| `product_vendor` | 제조사 (예: Samsung, Apple Inc) |
| `created_at` | 최초 등록 시간 |
| `UNIQUE` 제약 | 동일한 출처 정보가 중복 저장되지 않도록 보장 |

> 🧠 많은 활동 데이터가 동일한 출처로부터 오기 때문에 **중복 최소화 및 성능 최적화**를 위해 정규화 처리하였습니다.

---

#### 3-3. `activity_entry` (실제 활동 데이터)

단말에서 수집된 활동 데이터를 표현합니다.  
예를 들어 10분 단위로 측정된 걸음 수, 칼로리, 거리 데이터가 각각의 `entry`로 저장됩니다.

| 컬럼명 | 설명 |
|--------|------|
| `record_id` | 어떤 recordKey에 소속된 entry인지 (논리적 FK) |
| `start_time`, `end_time` | 해당 데이터의 시간 범위 (예: 06:30 ~ 06:40) |
| `steps`, `calories_kcal`, `distance_km` | 건강 지표 수치들 |
| `created_at` | 저장 시각 |
| `PRIMARY KEY` | `(record_id, start_time)` 복합키로 설정하여 **중복 방지** |
| `PARTITION` | `start_time` 기준 Range Partition 적용으로 **쿼리 성능 최적화** |

> 🧩 수백~수천 개의 entry가 한 번에 들어오는 구조이므로,  
> 데이터 중복을 **DB 수준에서 방지**하고 효율적으로 저장될 수 있도록 설계되어 있습니다.
---
## 📦 Redis Stream 실행방법

### 1. 로컬 Redis 실행

시스템은 Redis Stream 기능을 활용하므로 **Redis 서버가 실행 중**이어야 합니다.


```bash
docker run --name redis -p 6379:6379 -d redis:7.0.4
```
---
## 🚀 추후 개선 사항

### 1. `hourly`, `daily`, `monthly` 집계 테이블 도입

현재는 `activity_entry` 테이블에서 통계 데이터를 실시간으로 집계(`SUM`, `GROUP BY`)하고 있으나,  
대량의 데이터가 누적될 경우 **조회 성능 저하**가 발생할 수 있습니다.  
이를 해결하기 위해 아래와 같은 **사전 집계(Aggregation) 테이블**을 도입할 예정입니다:

| 테이블 이름                 | 설명                                    |
|----------------------------|-----------------------------------------|
| `activity_hourly_summary`  | 1시간 단위로 집계된 데이터 (예: 09:00~10:00) |
| `activity_daily_summary`   | 하루 단위로 집계된 데이터 (예: 2024-11-01) |
| `activity_monthly_summary` | 한 달 단위로 집계된 데이터 (예: 2024-11)  |

---

### 2. 집계 전략

- `activity_entry` → `hourly_summary` → `daily_summary` → `monthly_summary`  
  구조로 **하위 집계 데이터를 상위 집계로 점진적으로 누적**하는 방식으로 처리합니다.

- **배치 작업(Scheduler)** 을 통해 주기적으로 집계 작업을 수행합니다. ( 실시간성이 중요하다면 스트리밍으로도 구현이 가능합니다. )

---

### 3. 기대 효과

| 항목               | 개선 전                                      | 개선 후                                              |
|--------------------|-----------------------------------------------|-------------------------------------------------------|
| 실시간 통계 조회    | `activity_entry`에서 직접 `GROUP BY` → 느림   | 요약 테이블에서 바로 조회 → 빠름                      |
| 대용량 데이터 대응 | 데이터가 많을수록 쿼리 속도 저하               | 집계 테이블 기반으로 빠르게 응답 가능                 |
| 스케일 아웃        | 어려움                                        | 캐싱, 샤딩, 파티셔닝 등으로 확장 가능                |

---


