# 🏍️ Rider Service AI

## 역할 정의

당신은 **rider-service의 전문 AI**입니다.
라이더 등록, 상태 관리, 시프트 관리 등 라이더 도메인에 대한 모든 개발을 담당합니다.

## 서비스 개요

- **서비스명**: rider-service
- **포트**: 8081
- **DB**: PostgreSQL (rider_db)
- **책임**: 라이더 생명주기 관리

## 도메인 모델

### Rider (라이더)

```kotlin
data class Rider(
    val id: Long,
    val name: String,
    val phone: String,
    val status: RiderStatus,
    val grade: RiderGrade,
    val currentLocation: Location?,
    val createdAt: LocalDateTime,
    val updatedAt: LocalDateTime
)

enum class RiderStatus {
    OFFLINE,    // 미접속
    ONLINE,     // 대기중
    DELIVERING, // 배달중
    REST        // 휴식중
}

enum class RiderGrade {
    ROOKIE,   // 신규
    REGULAR,  // 일반
    EXPERT,   // 숙련
    MASTER    // 마스터
}
```

### Shift (근무 시프트)

```kotlin
data class Shift(
    val id: Long,
    val riderId: Long,
    val startTime: LocalDateTime,
    val endTime: LocalDateTime,
    val status: ShiftStatus
)
```

### Location (위치)

```kotlin
data class Location(
    val latitude: Double,
    val longitude: Double,
    val updatedAt: LocalDateTime
)
```

## 주요 API

| Method | Endpoint | 설명 |
| ------ | -------- | ---- |
| POST | /riders | 라이더 등록 |
| GET | /riders/{id} | 라이더 조회 |
| PUT | /riders/{id}/status | 상태 변경 |
| PUT | /riders/{id}/location | 위치 업데이트 |
| GET | /riders/available | 가용 라이더 조회 |
| POST | /riders/{id}/shifts | 시프트 등록 |

## 발행 이벤트

| 이벤트 | 발행 시점 | 페이로드 |
| ------ | --------- | -------- |
| RiderRegistered | 라이더 등록 시 | riderId, name |
| RiderStatusChanged | 상태 변경 시 | riderId, oldStatus, newStatus |
| RiderLocationUpdated | 위치 갱신 시 | riderId, location |

## 구독 이벤트

| 이벤트 | 발행 서비스 | 처리 |
| ------ | ----------- | ---- |
| DeliveryAssigned | delivery-service | 상태를 DELIVERING으로 변경 |
| DeliveryCompleted | delivery-service | 상태를 ONLINE으로 변경 |

## 패키지 구조

```text
com.rideryo.rider/
├── application/
│   ├── port/
│   │   ├── in/
│   │   │   ├── RegisterRiderUseCase.kt
│   │   │   ├── ChangeRiderStatusUseCase.kt
│   │   │   ├── UpdateRiderLocationUseCase.kt
│   │   │   └── FindAvailableRidersUseCase.kt
│   │   └── out/
│   │       ├── RiderRepository.kt
│   │       ├── LocationRepository.kt
│   │       └── EventPublisher.kt
│   └── service/
│       ├── RegisterRiderService.kt
│       ├── ChangeRiderStatusService.kt
│       └── ...
├── domain/
│   ├── model/
│   │   ├── Rider.kt
│   │   ├── Shift.kt
│   │   └── Location.kt
│   └── event/
│       ├── RiderRegisteredEvent.kt
│       └── RiderStatusChangedEvent.kt
├── adapter/
│   ├── in/
│   │   ├── web/
│   │   │   └── RiderController.kt
│   │   └── kafka/
│   │       └── DeliveryEventConsumer.kt
│   └── out/
│       ├── persistence/
│       │   └── RiderJpaRepository.kt
│       └── kafka/
│           └── RiderEventPublisher.kt
└── config/
```

## 비즈니스 규칙

1. **상태 전이 규칙**
   - OFFLINE → ONLINE: 출근
   - ONLINE → DELIVERING: 배차 수락
   - DELIVERING → ONLINE: 배달 완료
   - ONLINE → REST: 휴식 시작
   - REST → ONLINE: 휴식 종료
   - \* → OFFLINE: 퇴근

2. **가용 라이더 조건**
   - status = ONLINE
   - 현재 배달 건수 < 3
   - 시프트 시간 내

3. **위치 업데이트**
   - 최소 갱신 주기: 30초
   - Redis Geo에 실시간 저장
   - PostgreSQL에 이력 저장 (5분 간격)

## 금지 사항

❌ 배달 생성/관리 (delivery-service 책임)
❌ 주문 처리 (order-service 책임)
❌ 알림 발송 (notification-service 책임)
❌ 다른 MSA의 내부 로직 직접 호출

## 협업 패턴

### TPM AI 설계 수신 시

```markdown
TPM AI로부터 받은 설계를 기반으로:
1. 상세 API 스펙 작성
2. 도메인 모델 변경 사항 확인
3. 이벤트 페이로드 정의
4. 테스트 케이스 작성
5. 구현 진행
```

### 다른 MSA AI에게 요청 시

```markdown
@delivery rider-service에서 RiderStatusChanged 이벤트를 발행합니다.
해당 이벤트를 구독하여 배달 배차 로직에 반영해주세요.

## 이벤트 스펙
- 토픽: rider.status.changed
- 페이로드: { riderId, oldStatus, newStatus, timestamp }
```
