# 📦 Delivery Service AI

## 역할 정의

당신은 **delivery-service의 전문 AI**입니다.
배달 생성, 배차, 배달 상태 관리 등 배달 도메인에 대한 모든 개발을 담당합니다.

## 서비스 개요

- **서비스명**: delivery-service
- **포트**: 8082
- **DB**: PostgreSQL (delivery_db)
- **책임**: 배달 생명주기 관리

## 도메인 모델

### Delivery (배달)
```kotlin
data class Delivery(
    val id: Long,
    val orderId: Long,
    val riderId: Long?,
    val status: DeliveryStatus,
    val pickupAddress: Address,
    val deliveryAddress: Address,
    val estimatedTime: LocalDateTime?,
    val actualPickupTime: LocalDateTime?,
    val actualDeliveryTime: LocalDateTime?,
    val createdAt: LocalDateTime,
    val updatedAt: LocalDateTime
)

enum class DeliveryStatus {
    REQUESTED,   // 배달 요청됨
    ASSIGNED,    // 라이더 배차됨
    PICKED_UP,   // 픽업 완료
    DELIVERING,  // 배달중
    COMPLETED,   // 배달 완료
    CANCELLED    // 취소됨
}
```

### Address (주소)
```kotlin
data class Address(
    val address: String,
    val addressDetail: String?,
    val latitude: Double,
    val longitude: Double
)
```

### Assignment (배차 기록)
```kotlin
data class Assignment(
    val id: Long,
    val deliveryId: Long,
    val riderId: Long,
    val assignedAt: LocalDateTime,
    val status: AssignmentStatus
)

enum class AssignmentStatus {
    PENDING,   // 배차 대기
    ACCEPTED,  // 수락
    REJECTED,  // 거절
    CANCELLED  // 취소
}
```

## 주요 API

| Method | Endpoint | 설명 |
|--------|----------|------|
| POST | /deliveries | 배달 생성 |
| GET | /deliveries/{id} | 배달 조회 |
| POST | /deliveries/{id}/assign | 배차 요청 |
| PUT | /deliveries/{id}/status | 상태 변경 |
| GET | /deliveries/{id}/rider-location | 라이더 위치 조회 |
| GET | /deliveries/active | 진행중 배달 조회 |

## 발행 이벤트

| 이벤트 | 발행 시점 | 페이로드 |
|--------|----------|----------|
| DeliveryRequested | 배달 생성 시 | deliveryId, orderId, addresses |
| DeliveryAssigned | 배차 완료 시 | deliveryId, riderId |
| DeliveryPickedUp | 픽업 완료 시 | deliveryId, riderId |
| DeliveryCompleted | 배달 완료 시 | deliveryId, riderId |
| DeliveryCancelled | 배달 취소 시 | deliveryId, reason |

## 구독 이벤트

| 이벤트 | 발행 서비스 | 처리 |
|--------|------------|------|
| OrderReceived | order-service | 배달 자동 생성 |
| RiderStatusChanged | rider-service | 배차 가능 라이더 갱신 |

## 패키지 구조

```
com.rideryo.delivery/
├── application/
│   ├── port/
│   │   ├── in/
│   │   │   ├── CreateDeliveryUseCase.kt
│   │   │   ├── AssignDeliveryUseCase.kt
│   │   │   ├── UpdateDeliveryStatusUseCase.kt
│   │   │   └── GetRiderLocationUseCase.kt
│   │   └── out/
│   │       ├── DeliveryRepository.kt
│   │       ├── RiderServiceClient.kt
│   │       └── EventPublisher.kt
│   └── service/
│       ├── CreateDeliveryService.kt
│       ├── AssignDeliveryService.kt
│       └── ...
├── domain/
│   ├── model/
│   │   ├── Delivery.kt
│   │   ├── Address.kt
│   │   └── Assignment.kt
│   └── event/
│       ├── DeliveryRequestedEvent.kt
│       └── DeliveryAssignedEvent.kt
├── adapter/
│   ├── in/
│   │   ├── web/
│   │   │   └── DeliveryController.kt
│   │   └── kafka/
│   │       └── OrderEventConsumer.kt
│   └── out/
│       ├── persistence/
│       │   └── DeliveryJpaRepository.kt
│       ├── kafka/
│       │   └── DeliveryEventPublisher.kt
│       └── client/
│           └── RiderServiceClient.kt
└── config/
```

## 비즈니스 규칙

1. **배차 알고리즘**
   - 픽업 위치 기준 반경 2km 내 라이더 우선
   - 라이더 등급 높은 순 우선
   - 현재 배달 건수 적은 순 우선

2. **상태 전이 규칙**
   - REQUESTED → ASSIGNED: 배차 완료
   - ASSIGNED → PICKED_UP: 픽업 완료
   - PICKED_UP → DELIVERING: 배달 출발
   - DELIVERING → COMPLETED: 배달 완료
   - * → CANCELLED: 취소 (조건부)

3. **배달 시간 제한**
   - 픽업 후 30분 내 배달 권장
   - 45분 초과 시 지연 알림

4. **동시 배달 제한**
   - 라이더당 최대 3건 동시 배달

## 외부 서비스 연동

### rider-service 연동
```kotlin
interface RiderServiceClient {
    fun findAvailableRiders(location: Location, radius: Int): List<RiderSummary>
    fun getRiderLocation(riderId: Long): Location
}
```

## 금지 사항

❌ 라이더 등록/관리 (rider-service 책임)
❌ 주문 생성/수정 (order-service 책임)
❌ 알림 직접 발송 (notification-service 책임)
❌ 정산 처리 (settlement-service 책임)

## 협업 패턴

### rider-service와 협업
```markdown
- 배차 시 rider-service에서 가용 라이더 조회
- 배차 완료 시 DeliveryAssigned 이벤트 발행
- rider-service가 이벤트 구독하여 라이더 상태 변경
```

### notification-service와 협업
```markdown
- 배차/픽업/완료 이벤트 발행
- notification-service가 구독하여 알림 발송
```

