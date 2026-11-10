---
title: "팀스파르타 인프라 정비 회고 2편 — Datadog IaC, 로깅 패키지, 그리고 환경 변수 주입 방식 개선"
date: 2026-11-10
draft: true
tags: ["terraform", "datadog", "grafana", "observability", "nestjs", "logging", "retrospective"]
translationKey: "teamsparta-infra-retrospective-2"
summary: "Datadog을 Terraform으로 관리하고, nestjs-pino 기반 로깅 패키지로 Datadog/Grafana 이원화 기반을 만들고, 런타임 SSM 조회 방식의 환경 변수를 표준 방식으로 바꾼 과정을 회고한다."
---

<!-- TODO: 인트로 — 1편 요약 한 줄 + 이번 글에서 다룰 것 -->

(예시) [이전 글]({{< ref "/posts/teamsparta-infra-retrospective-1" >}})에서는 Terraform으로 ECS를 들여오고 배포 파이프라인을 하나로 모은 과정을 정리했다. 이번 글에서는 그 위에서 진행한 관측성과 설정 관리 작업을 다룬다.

- Datadog 리소스를 Terraform으로 관리하기
- nestjs-pino 기반 공용 로깅 패키지와 Datadog/Grafana 이원화
- 런타임에 SSM에서 값을 가져오던 환경 변수 구조 개선

---

## 1. Datadog을 코드로 관리하기

<!-- 전환 문장 -->
(예시) 인프라가 코드로 정리되자, 그 인프라를 지켜보는 모니터와 대시보드도 같은 방식으로 관리하고 싶어졌다.

### 1.1 콘솔에서 관리되던 모니터와 대시보드

<!-- TODO: 기존 상황 — 누가 만든지 모르는 모니터, 알림 채널 제각각, 서비스 추가 시 수동 생성 등 -->

### 1.2 Datadog Terraform provider 도입

<!-- TODO: 어떤 리소스를 코드로 옮겼는지 (monitor, dashboard, synthetics, log pipeline 등), 레포 구조, ECS 모듈과 어떻게 엮었는지 -->

```hcl
# (예시) 서비스 공통 모니터를 모듈로 묶기
module "service_a_monitors" {
  source       = "../../modules/datadog-service-monitors"
  service_name = "service-a"
  env          = "prod"
  notify       = "@slack-alerts-prod"
}
```

### 1.3 좋아진 점과 남은 고민

<!-- TODO: 예 — 서비스 추가 시 모니터 자동 생성, 리뷰 가능한 알림 임계값. 고민 — 콘솔에서 튜닝하고 싶은 사람과의 충돌, import 범위 -->

---

## 2. nestjs-pino 기반 로깅 패키지

<!-- 전환 문장 -->
(예시) 모니터를 정리하고 나니, 그 모니터가 바라보는 데이터 — 특히 로그 — 의 형식이 서비스마다 다르다는 문제가 남아 있었다.

### 2.1 서비스마다 다른 로그 형식

<!-- TODO: 기존 로그 상황 — console.log, 다른 로거, 필드 이름 불일치, trace 연결 불가 등 -->

### 2.2 공용 로깅 패키지 설계

<!-- TODO: 패키지가 제공하는 것 — 공통 필드(service, env, version, trace_id), redaction, 요청 로깅, 배포 방식(private npm / GitHub Packages) -->

```typescript
// (예시) 서비스에서의 사용
@Module({
  imports: [
    SpartaLoggerModule.forRoot({
      service: 'service-a',
    }),
  ],
})
export class AppModule {}
```

### 2.3 Datadog과 Grafana 이원화

<!-- TODO: 왜 일부 prod만 Datadog인지 (비용 vs 가치), 나머지는 어떤 스택(Loki/Prometheus 등)으로 Grafana에서 보는지, 패키지가 이 차이를 어떻게 흡수하는지 -->

<!-- TODO: 다이어그램 — App(pino JSON) → Datadog Agent / Grafana 스택 분기 (images/logging-routing.svg) -->

| 환경 | 수집 대상 | 이유 |
|------|-----------|------|
| prod (핵심 서비스) | (예: Datadog) | (예: APM·트레이스 연계가 필요) |
| prod (그 외) / dev | (예: Grafana) | (예: 비용 대비 충분한 가시성) |

---

## 3. 환경 변수 주입 방식 개선

<!-- 전환 문장 -->
(예시) 마지막으로 다룰 것은 애플리케이션이 설정 값을 받아오는 방식이다. 눈에 잘 띄지 않지만, 가장 위험하다고 느꼈던 부분이다.

### 3.1 기존 구조 — 앱이 뜬 뒤에 SSM에서 가져오기

<!-- TODO: 기존 흐름 설명. 앱 부팅 → AWS SDK로 SSM Parameter Store 조회 → 설정 주입 -->

<!-- TODO: 다이어그램 — before/after 부팅 시퀀스 비교 (images/env-injection-before-after.svg) -->

### 3.2 무엇이 위험했나

<!-- TODO: 구체적인 위험 — 예시 아래. 실제 겪은 사례가 있다면 꼭 넣기 -->

- (예: SSM 조회 실패/스로틀링 시 앱이 반쯤 뜬 상태가 됨)
- (예: 헬스체크는 통과했는데 설정이 비어 있는 상태)
- (예: 애플리케이션 코드가 인프라(AWS)에 강하게 결합 — 로컬 실행·테스트가 어려움)
- (예: 어떤 값이 주입되었는지 배포 시점에 알 수 없음)

### 3.3 표준적인 방식으로 — @nestjs/config와 .env

<!-- TODO: 바꾼 구조. 인프라 레이어(ECS task definition secrets / 파이프라인)에서 값을 주입하고, 앱은 환경 변수만 읽는다. 스키마 검증(Joi/zod 등)으로 fail-fast -->

```typescript
// (예시) 부팅 시점 검증
ConfigModule.forRoot({
  isGlobal: true,
  validationSchema: Joi.object({
    DATABASE_URL: Joi.string().required(),
    REDIS_HOST: Joi.string().required(),
  }),
});
```

```hcl
# (예시) ECS task definition에서 SSM 값을 환경 변수로 주입
secrets = [
  { name = "DATABASE_URL", valueFrom = "arn:aws:ssm:...:parameter/service-a/prod/DATABASE_URL" },
]
```

### 3.4 마이그레이션 과정

<!-- TODO: 서비스별 전환 순서, 하위 호환 기간, 로컬 개발 환경(.env) 정리 -->

---

## 마치며

<!-- 요약 → 개인적 인사이트 → 다음 방향 -->

(예시) 두 편에 걸쳐 4월부터 진행한 인프라 정비 작업을 정리했다. Terraform으로 리소스를 모으고, 배포·관측·설정 주입을 모든 서비스가 같은 방식으로 하도록 맞추는 것이 전체를 관통하는 흐름이었다.

<!-- TODO: 개인적 인사이트 — 예: "일관성은 한 번에 만들어지지 않고, 새로 만드는 것부터 표준을 따르게 하는 것이 가장 효과적이었다" -->

<!-- TODO: 다음 방향 — 예: 남은 서비스 이관, SLO 기반 알림, 비용 최적화 등. 관련 글이 있으면 링크: [SLO 글]({{< ref "/posts/slo-sli-sla-error-budget" >}}) -->

### References

<!-- TODO: 참고한 문서 링크 -->
- [Datadog Terraform Provider](https://registry.terraform.io/providers/DataDog/datadog/latest/docs)
- [nestjs-pino](https://github.com/iamolegga/nestjs-pino)
- [NestJS — Configuration](https://docs.nestjs.com/techniques/configuration)
- [Amazon ECS — Passing sensitive data to a container](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/specifying-sensitive-data.html)
