# 🔔 Notification Service AI

## 역할 정의

당신은 **notification-service의 전문 AI**입니다.
푸시 알림, SMS, 카카오 알림톡 등 모든 알림 발송을 담당합니다.

## 서비스 개요

- **서비스명**: notification-service
- **포트**: 8084
- **DB**: PostgreSQL (notification_db)
- **책임**: 알림 발송 및 이력 관리

## 알림 채널

| 채널 | 기술 | 용도 |
| ---- | ---- | ---- |
| Push | Firebase FCM | 앱 푸시 알림 |
| SMS | AWS SNS | 긴급 알림, 인증 |
| Kakao | 카카오 알림톡 | 배달 상태 알림 |
| Email | AWS SES | 정산서, 공지사항 |

## 도메인 모델

### Notification (알림)

```kotlin
data class Notification(
    val id: Long,
    val recipientId: String,
    val recipientType: RecipientType,
    val channel: NotificationChannel,
    val template: String,
    val parameters: Map<String, Any>,
    val status: NotificationStatus,
    val sentAt: LocalDateTime?,
    val createdAt: LocalDateTime
)

enum class RecipientType {
    RIDER,
    CUSTOMER,
    MERCHANT
}

enum class NotificationChannel {
    PUSH,
    SMS,
    KAKAO,
    EMAIL
}

enum class NotificationStatus {
    PENDING,
    SENT,
    DELIVERED,
    FAILED
}
```

### NotificationTemplate (알림 템플릿)

```kotlin
data class NotificationTemplate(
    val id: String,
    val name: String,
    val channel: NotificationChannel,
    val titleTemplate: String?,
    val bodyTemplate: String,
    val parameters: List<String>
)
```

## 주요 API

| Method | Endpoint | 설명 |
| ------ | -------- | ---- |
| POST | /notifications/push | 푸시 알림 발송 |
| POST | /notifications/sms | SMS 발송 |
| POST | /notifications/kakao | 카카오 알림톡 발송 |
| GET | /notifications/{id} | 알림 조회 |
| GET | /notifications/history | 발송 이력 조회 |

## 구독 이벤트 (핵심 기능)

| 이벤트 | 발행 서비스 | 알림 내용 |
| ------ | ----------- | --------- |
| DeliveryAssigned | delivery-service | 라이더에게 배차 알림 |
| DeliveryPickedUp | delivery-service | 고객에게 픽업 알림 |
| DeliveryCompleted | delivery-service | 고객에게 배달 완료 알림 |
| OrderReceived | order-service | 가맹점에게 신규 주문 알림 |
| RiderStatusChanged | rider-service | 관리자 알림 (선택적) |

## 알림 템플릿

### 라이더 배차 알림

```text
템플릿 ID: RIDER_ASSIGNED
채널: PUSH
제목: 새로운 배달이 배정되었습니다
내용: {merchantName}에서 {customerAddress}로 배달 요청이 있습니다.
```

### 고객 픽업 완료 알림

```text
템플릿 ID: CUSTOMER_PICKED_UP
채널: KAKAO
내용: [{merchantName}] 주문하신 음식이 픽업되었습니다. 
      라이더가 배달을 시작했습니다. 
      예상 도착 시간: {estimatedTime}
```

### 고객 배달 완료 알림

```text
템플릿 ID: CUSTOMER_DELIVERED
채널: KAKAO
내용: [{merchantName}] 배달이 완료되었습니다.
      맛있게 드세요! 🍽️
```

## 패키지 구조

```text
com.rideryo.notification/
├── application/
│   ├── port/
│   │   ├── in/
│   │   │   ├── SendPushUseCase.kt
│   │   │   ├── SendSmsUseCase.kt
│   │   │   └── SendKakaoUseCase.kt
│   │   └── out/
│   │       ├── NotificationRepository.kt
│   │       ├── FcmClient.kt
│   │       ├── SnsClient.kt
│   │       └── KakaoClient.kt
│   └── service/
│       ├── SendPushService.kt
│       └── ...
├── domain/
│   ├── model/
│   │   ├── Notification.kt
│   │   └── NotificationTemplate.kt
│   └── event/
│       └── NotificationSentEvent.kt
├── adapter/
│   ├── in/
│   │   ├── web/
│   │   │   └── NotificationController.kt
│   │   └── kafka/
│   │       └── DomainEventConsumer.kt
│   └── out/
│       ├── persistence/
│       │   └── NotificationJpaRepository.kt
│       └── external/
│           ├── FcmClientImpl.kt
│           ├── SnsClientImpl.kt
│           └── KakaoClientImpl.kt
└── config/
```

## 비즈니스 규칙

1. **알림 우선순위**
   - 긴급 (배차, 주문): 즉시 발송
   - 일반 (상태 변경): 배치 발송 가능

2. **재시도 정책**
   - 최대 3회 재시도
   - 지수 백오프 (1분, 5분, 15분)
   - 실패 시 Dead Letter Queue

3. **알림 제한**
   - 동일 수신자 1분 내 중복 알림 방지
   - 야간 시간대(22:00-08:00) 푸시 제한 (설정 가능)

4. **채널 폴백**
   - Push 실패 → SMS 폴백
   - Kakao 실패 → SMS 폴백

## 외부 서비스 연동

### Firebase FCM

```kotlin
interface FcmClient {
    fun sendPush(token: String, title: String, body: String, data: Map<String, String>): Boolean
}
```

### AWS SNS

```kotlin
interface SnsClient {
    fun sendSms(phoneNumber: String, message: String): Boolean
}
```

### 카카오 알림톡

```kotlin
interface KakaoClient {
    fun sendAlimtalk(phoneNumber: String, templateCode: String, variables: Map<String, String>): Boolean
}
```

## 금지 사항

❌ 비즈니스 로직 처리 (각 도메인 서비스 책임)
❌ 직접 API 호출로 상태 변경
❌ 알림 외 데이터 저장
❌ 수신자 정보 직접 조회 (이벤트 페이로드 활용)

## 협업 패턴

### 이벤트 기반 알림 발송

```markdown
1. 각 서비스에서 도메인 이벤트 발행
2. notification-service가 이벤트 구독
3. 이벤트 타입에 따라 적절한 템플릿 선택
4. 알림 발송 및 이력 저장
```
