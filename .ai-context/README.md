# 🤖 Rideryo AI Context

> Kurly OMS팀의 AI 협업 방식을 참고하여 설계된 다중 AI Agent 협업 환경

## 📁 폴더 구조

```text
.ai-context/
├── knowledge/          # 지식 기반 (What to know)
│   ├── domain/         # 도메인 지식 (비즈니스 로직, 용어)
│   ├── architecture/   # 아키텍처 정보 (MSA 구조, 시스템 설계)
│   └── tech-stack/     # 기술 스택 (프레임워크, 라이브러리, 컨벤션)
│
├── agents/             # AI Agent 역할 정의
│   ├── tpm/            # TPM AI (Technical Project Manager)
│   └── msa/            # MSA별 전문 AI Agents
│       ├── rider/      # Rider Service AI
│       ├── delivery/   # Delivery Service AI
│       ├── order/      # Order Service AI
│       └── notification/ # Notification Service AI
│
├── workflows/          # 행동 기반 (How to act)
│   ├── development/    # 개발 워크플로우
│   ├── review/         # 코드 리뷰 워크플로우
│   └── deployment/     # 배포 워크플로우
│
└── templates/          # 문서/코드 템플릿
    ├── design/         # 설계 문서 템플릿
    ├── pr/             # PR 템플릿
    └── code/           # 코드 템플릿
```

## 🎭 AI Agent 역할

### TPM AI (Technical Project Manager)

- **책임**: 전체 아키텍처 설계, MSA 간 책임 분배, 영향도 분석
- **호출**: `@tpm` 또는 "TPM AI에게 설계 요청"
- **산출물**: 아키텍처 설계서, 배포 계획, 영향도 분석서

### MSA AI (Service-specific AI)

- **책임**: 각 MSA 내부의 상세 설계 및 구현
- **호출**: `@rider`, `@delivery`, `@order`, `@notification`
- **산출물**: 상세 설계서, 코드, 테스트

## 🔄 협업 흐름

```text
요구사항 → TPM AI (1차 설계) → 팀 검토 → MSA AI (집중 설계/구현) → 코드 리뷰
```

1. **요구사항 분석**: TPM AI가 전체 영향도 분석
2. **1차 설계**: TPM AI가 MSA 간 책임 분배 설계
3. **팀 검토**: 엔지니어가 설계 검증 (가장 중요!)
4. **집중 설계**: 각 MSA AI가 상세 설계
5. **구현**: MSA AI가 코드 작성
6. **리뷰**: 엔지니어 + AI 코드 리뷰

## 📝 사용 방법

### TPM AI 호출 예시

```text
@tpm 새로운 실시간 위치 추적 기능을 추가하려고 합니다.
어떤 MSA들이 영향을 받는지 분석하고 설계해주세요.
```

### MSA AI 호출 예시

```text
@rider 라이더의 출퇴근 상태 관리 기능을 구현해주세요.
TPM AI의 설계를 참고해서 상세 구현해주세요.
```

## ⚠️ 중요 원칙

> **AI가 아무리 똑똑해도, AI 답변을 꼼꼼히 검증하고 최종 책임을 지는 것은 인간 엔지니어의 몫입니다.**

1. TPM AI의 설계는 반드시 팀 검토 후 진행
2. MSA AI의 코드는 항상 코드 리뷰 수행
3. AI를 맹신하지 않고 비판적으로 검토
