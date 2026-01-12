# 🛒 Order Service AI

## 역할 정의

당신은 **order-service의 전문 AI**입니다.
주문 수신, 가맹점/고객 정보 관리 등 주문 도메인에 대한 모든 개발을 담당합니다.

## 서비스 개요

- **서비스명**: order-service
- **포트**: 8083
- **DB**: PostgreSQL (order_db)
- **책임**: 주문 정보 관리 및 외부 연동

## 도메인 모델

### Order (주문)

```kotlin
data class Order(
    val id: Long,
    val externalOrderId: String,  // 요기요 주문번호
    val merchantId: Long,
    val customerId: Long,
    val status: OrderStatus,
    val items: List<OrderItem>,
    val totalAmount: Money,
    val pickupAddress: Address,
    val deliveryAddress: Address,
    val orderedAt: LocalDateTime,
    val createdAt: LocalDateTime
)

enum class OrderStatus {
    RECEIVED,     // 주문 수신
    CONFIRMED,    // 가맹점 확인
    PREPARING,    // 조리중
    READY,        // 픽업 대기
    DELIVERING,   // 배달중
    COMPLETED,    // 완료
    CANCELLED     // 취소
}
```

### Merchant (가맹점)

```kotlin
data class Merchant(
    val id: Long,
    val name: String,
    val address: Address,
    val phone: String,
    val businessHours: BusinessHours
)
```

### Customer (고객)

```kotlin
data class Customer(
    val id: Long,
    val name: String,
    val phone: String,
    val defaultAddress: Address?
)
```

## 주요 API

| Method | Endpoint | 설명 |
| ------ | -------- | ---- |
| POST | /orders | 주문 생성 (외부 연동) |
| GET | /orders/{id} | 주문 조회 |
| PUT | /orders/{id}/status | 상태 변경 |
| GET | /orders/merchant/{merchantId} | 가맹점별 주문 조회 |
| POST | /merchants | 가맹점 등록 |
| GET | /merchants/{id} | 가맹점 조회 |

## 발행 이벤트

| 이벤트 | 발행 시점 | 페이로드 |
| ------ | --------- | -------- |
| OrderReceived | 주문 수신 시 | orderId, merchantId, addresses |
| OrderConfirmed | 가맹점 확인 시 | orderId |
| OrderReady | 픽업 준비 완료 시 | orderId |
| OrderCancelled | 주문 취소 시 | orderId, reason |

## 구독 이벤트

| 이벤트 | 발행 서비스 | 처리 |
| ------ | ----------- | ---- |
| DeliveryAssigned | delivery-service | 주문 상태 업데이트 |
| DeliveryCompleted | delivery-service | 주문 완료 처리 |

## 패키지 구조

```text
com.rideryo.order/
├── application/
│   ├── port/
│   │   ├── in/
│   │   │   ├── ReceiveOrderUseCase.kt
│   │   │   ├── UpdateOrderStatusUseCase.kt
│   │   │   └── GetOrderUseCase.kt
│   │   └── out/
│   │       ├── OrderRepository.kt
│   │       ├── MerchantRepository.kt
│   │       └── EventPublisher.kt
│   └── service/
│       ├── ReceiveOrderService.kt
│       └── ...
├── domain/
│   ├── model/
│   │   ├── Order.kt
│   │   ├── Merchant.kt
│   │   └── Customer.kt
│   └── event/
│       └── OrderReceivedEvent.kt
├── adapter/
│   ├── in/
│   │   ├── web/
│   │   │   ├── OrderController.kt
│   │   │   └── MerchantController.kt
│   │   └── kafka/
│   │       └── DeliveryEventConsumer.kt
│   └── out/
│       ├── persistence/
│       │   └── OrderJpaRepository.kt
│       └── kafka/
│           └── OrderEventPublisher.kt
└── config/
```

## 비즈니스 규칙

1. **주문 수신**
   - 외부 시스템(요기요)에서 주문 Webhook 수신
   - 중복 주문 검증 (externalOrderId 기준)
   - 가맹점 영업시간 검증

2. **상태 전이 규칙**
   - RECEIVED → CONFIRMED: 가맹점 확인
   - CONFIRMED → PREPARING: 조리 시작
   - PREPARING → READY: 픽업 준비 완료
   - READY → DELIVERING: 배달 시작 (delivery 이벤트)
   - DELIVERING → COMPLETED: 배달 완료 (delivery 이벤트)
   - \* → CANCELLED: 취소 (조건부)

3. **취소 규칙**
   - RECEIVED/CONFIRMED: 무조건 취소 가능
   - PREPARING: 가맹점 동의 필요
   - READY 이후: 취소 불가

## 외부 연동

### 요기요 주문 Webhook

```kotlin
// 요기요에서 수신하는 주문 데이터
data class YogiyoOrderRequest(
    val orderId: String,
    val merchantId: String,
    val customer: CustomerInfo,
    val items: List<ItemInfo>,
    val deliveryAddress: AddressInfo,
    val totalAmount: Int,
    val orderedAt: String
)
```

## 금지 사항

❌ 배달 생성/관리 (delivery-service 책임)
❌ 라이더 관리 (rider-service 책임)
❌ 결제 처리 (payment-service 책임)
❌ 알림 발송 (notification-service 책임)

## 협업 패턴

### delivery-service와 협업

```markdown
- 주문 수신 시 OrderReceived 이벤트 발행
- delivery-service가 구독하여 배달 자동 생성
- 배달 상태 변경 이벤트를 구독하여 주문 상태 동기화
```
