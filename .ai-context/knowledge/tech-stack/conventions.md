# 📚 기술 스택 & 코딩 컨벤션

## 기술 스택

### Backend

- **언어**: Kotlin 1.9+
- **프레임워크**: Spring Boot 3.2+
- **빌드 도구**: Gradle (Kotlin DSL)
- **ORM**: Spring Data JPA + QueryDSL
- **DB**: PostgreSQL 15+
- **캐시**: Redis 7+
- **메시지 브로커**: Apache Kafka

### 인프라

- **컨테이너**: Docker
- **오케스트레이션**: Kubernetes
- **CI/CD**: GitHub Actions
- **클라우드**: AWS

## 패키지 구조 (클린 아키텍처)

```text
com.rideryo.{service}/
├── application/          # 애플리케이션 계층
│   ├── port/
│   │   ├── in/           # Input Ports (Use Cases)
│   │   └── out/          # Output Ports (Repository Interfaces)
│   └── service/          # Use Case 구현체
│
├── domain/               # 도메인 계층
│   ├── model/            # 도메인 모델 (Entity, Value Object)
│   └── event/            # 도메인 이벤트
│
├── adapter/              # 어댑터 계층
│   ├── in/
│   │   ├── web/          # REST Controller
│   │   └── kafka/        # Kafka Consumer
│   └── out/
│       ├── persistence/  # JPA Repository 구현
│       └── kafka/        # Kafka Producer
│
└── config/               # 설정
```

## 코딩 컨벤션

### 네이밍 규칙

```kotlin
// 클래스명: PascalCase
class RiderService

// 함수명: camelCase
fun findAvailableRiders(): List<Rider>

// 상수: SCREAMING_SNAKE_CASE
const val MAX_CONCURRENT_DELIVERIES = 3

// 패키지명: lowercase
package com.rideryo.rider.application.service
```

### Use Case 작성 규칙

```kotlin
// Input Port (인터페이스)
interface AssignDeliveryUseCase {
    fun assign(command: AssignDeliveryCommand): DeliveryAssignedResult
}

// Command 객체
data class AssignDeliveryCommand(
    val deliveryId: Long,
    val riderId: Long
)

// Service 구현
@Service
@Transactional
class AssignDeliveryService(
    private val deliveryRepository: DeliveryRepository,
    private val riderRepository: RiderRepository,
    private val eventPublisher: DomainEventPublisher
) : AssignDeliveryUseCase {

    override fun assign(command: AssignDeliveryCommand): DeliveryAssignedResult {
        val delivery = deliveryRepository.findById(command.deliveryId)
            ?: throw DeliveryNotFoundException(command.deliveryId)
        
        val rider = riderRepository.findById(command.riderId)
            ?: throw RiderNotFoundException(command.riderId)
        
        delivery.assignTo(rider)
        
        eventPublisher.publish(DeliveryAssignedEvent(delivery.id, rider.id))
        
        return DeliveryAssignedResult(delivery.id, rider.id)
    }
}
```

### API 응답 규칙

```kotlin
// 성공 응답
data class ApiResponse<T>(
    val success: Boolean = true,
    val data: T? = null,
    val error: ErrorInfo? = null
)

// 에러 응답
data class ErrorInfo(
    val code: String,
    val message: String,
    val details: Map<String, Any>? = null
)
```

### 테스트 컨벤션

```kotlin
// 테스트 클래스명: {대상}Test 또는 {대상}IntegrationTest
class RiderServiceTest {

    @Test
    fun `라이더 상태가 ONLINE일 때 배차가 가능하다`() {
        // given
        val rider = createRider(status = RiderStatus.ONLINE)
        
        // when
        val result = riderService.canBeAssigned(rider.id)
        
        // then
        assertThat(result).isTrue()
    }
}
```

## Git 컨벤션

### 브랜치 전략

```text
main
├── develop
│   ├── feature/{ticket-id}-{description}
│   ├── bugfix/{ticket-id}-{description}
│   └── hotfix/{ticket-id}-{description}
```

### 커밋 메시지

```text
{type}({scope}): {subject}

{body}

{footer}
```

**타입**:

- `feat`: 새로운 기능
- `fix`: 버그 수정
- `refactor`: 리팩토링
- `docs`: 문서 수정
- `test`: 테스트 추가/수정
- `chore`: 빌드, 설정 변경

**예시**:

```text
feat(rider): 라이더 실시간 위치 추적 기능 추가

- WebSocket을 통한 실시간 위치 업데이트
- Redis Geo를 활용한 위치 저장
- 30초 간격 위치 갱신

Closes #123
```
