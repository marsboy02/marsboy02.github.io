---
title: "팀스파르타 인프라 정비 회고 1편 — Terraform으로 ECS를 들여오고 배포 파이프라인을 하나로 모으기까지"
date: 2026-11-10
draft: true
tone: "polite"
tags:
  ["terraform", "iac", "aws-ecs", "github-actions", "ci-cd", "retrospective"]
translationKey: "teamsparta-infra-retrospective-1"
summary: "4월부터 진행한 인프라 정비 작업 중 Terraform 기반 ECS 이관과 공용 GitHub Actions 파이프라인 구축 과정을 회고합니다."
---

안녕하세요. 팀스파르타 플랫폼개발팀 인프라파트에서 데브옵스 엔지니어로 일하고 있는 강형준입니다. 저는 2026년 4월 인프라파트에 합류해 지금까지 팀스파르타의 인프라를 관리하고 있습니다.

2026년을 돌아보면 꽤 다사다난한 한 해였는데요. 정해진 기준에 맞춰 일을 했다기보다는, 정립된 인프라 표준이 없던 상황에서 일 년 내내 그 표준을 하나씩 세워 나가는 일을 했습니다. 흩어져 있던 인프라를 Terraform으로 모으면서 늘 일관성, 그리고 기초(파운데이션)에 대해 고민했고, 여러 분들과 커피챗을 나누며 어떻게 하면 모두가 편하고 효율적으로 일할 수 있을지 고민한 한 해였습니다.

이번 회고는 내용이 길어 1편과 2편으로 나누어 작성했습니다. 각 편에서 다루는 내용은 다음과 같습니다.

- **1편**(이 글): 기존 인프라의 상황, Terraform으로 ECS를 들여온 과정, 공용 GitHub Actions 파이프라인
- **2편**: Datadog IaC, nestjs-pino 기반 로깅 패키지와 Datadog/Grafana 이원화, 환경 변수 주입 방식 개선

<!-- TODO: 2편 공개 후 링크로 교체 — [2편]({{< ref "/posts/teamsparta-infra-retrospective-2" >}}) -->

---

## 1. 처음 마주한 인프라

![얽히고설킨 보안 그룹과 서버들 앞에서 고민하는 모습](images/tangled-security-groups.png)

안녕하세요. 저는 팀스파르타에서 데브옵스 엔지니어로서 일하는 강형준입니다. 인프라파트는 플랫폼개발팀 소속으로, 팀원들이 편리하고 안전하게 애플리케이션을 개발할 수 있도록 돕는 일을 합니다. 제가 합류했을 때 인프라파트는 QA 엔지니어, 데이터 엔지니어, 그리고 인프라를 주로 맡는 저로 구성되어 있었고, 운영 중인 ECS 서비스는 약 40개 정도였습니다.

합류하자마자 보안 그룹 정리와 신규 서비스 프로비저닝 작업을 맡게 되었는데요. 그중 가장 많은 시간을 쓴 부분은 인프라의 표준을 세우는 일이었습니다. 예를 들어 앞으로 ECS 서비스는 어떤 구성으로 만들지, 데이터베이스가 필요할 때는 어떻게 구축할지 같은 것들을 정해 나갔습니다.

이런 기준을 하나씩 적다 보니 그 기준을 담아 둘 문서나 코드가 필요했고, 마침 Terraform을 도입해 이 문제를 풀어 보기로 했습니다. 이전부터 리소스를 선언형으로 관리하는 방식을 많이 고민해 왔고, 최근에는 AI 에이전트의 성능이 좋아져 Terraform 코드를 손쉽게 작성할 수 있었기 때문에 타이밍도 좋았습니다. 앞으로 더 좋아질 AI의 성능을 생각하면 AWS의 상태를 git의 소스 코드로 나타내는 GitOps 철학이 장기적으로 큰 도움이 될 것이라 판단했고, 팀 내 만장일치로 도입을 진행하게 되었습니다.

![6월부터 11월까지 리소스별로 누적한 Terraform 관리 leaf 수 추이](images/terraform-leaf-growth.png)

위 그래프는 Terraform을 처음 운영하기 시작한 6월부터 이 글을 쓰는 11월까지, 약 반년 동안 Terraform으로 관리하게 된 leaf의 수를 나타낸 것입니다. leaf는 서비스 하나에 해당하는 관리 단위라고 생각하시면 편한데요. ECS, Lambda, EC2 같은 컴퓨팅 리소스를 중심으로, 지금은 231개의 leaf를 Terraform으로 관리하고 있습니다. 이번 글에서는 Terraform을 도입하면서 내렸던 굵직한 결정들과, 그 결정에 이르기까지의 과정을 다뤄 보려고 합니다.

### 1.1 콘솔에서 만들어진 리소스들

데브옵스 엔지니어로서 합류하고나서 가장 먼저 AWS를 정리하는 일을 맡게 되었습니다. 당시 대부분의 리소스는 AWS 콘솔 기반으로 만들어져있는 서비스였기 때문에 추적하기가 어려웠으며, 서비스가 어떻게 영향을 끼치는지 파악하기가 어려웠습니다. 이렇게 콘솔에서 만들거나 AWS CLI를 통해서 리소스를 만들게 되면 명령을 통해서 만들게 되며, 이런 방식을 **명령형(Imperative) 프로그래밍 패러다임**이라고 부르며 일반적으로 안티 패턴(Anti-pattern)에 가깝습니다. 하나씩 명령어를 통해서 리소스를 생성하기 때문에 어떤 변화가 있었는지 추적하기 어려운 문제가 있습니다.

반대는 이상적으로 여거지는 **선언형(Declarative) 프로그래밍 패러다임**입니다. 필요한 리소스를 전부 써두면, 테라폼의 프로바이더(provider)나 쿠버네티스의 엔진이 선언되어있는 상태에 맞게 구현하는 것을 의미합니다. 인프라와 배포 환경에서는 '어떻게 변경할 것인가'보다 '최종적으로 어떤 상태가 되어야 하는가'를 보장하는 선언형 방식이 거의 표준으로 자리잡았고, 이를 도와주는 툴이 많이 존재합니다. 선언형 패러다임을 사용하면 다음과 같은 장점을 얻을 수 있다고 합니다.

1. **멱등성(Idempotency) 보장**: 선언형은 같은 코드를 100번 실행해도 항상 동일한 최종 상태를 유지합니다.
2. **구성 드리프트**: 운영자가 서버에 직접 접속해 명령어로 긴급 수정을 가하면 코드와 실제 인프라 상태가 달라지고, 파악하기가 어렵습니다.
3. **상태 추적 및 롤백의 한계**: 선언형 도구(Terraform, Kubernetes 등)는 현재 상태와 목표 상태를 비교해 변경분만 적용합니다.
4. **GitOps 구현**: 현대 데브옵스의 핵심인 GitOps를 통해서 Git 저장소를 단일 진실 공급원(Single Source of Truth)으로 삼습니다.

![ClickOps와 GitOps 비교](images/clickops-vs-gitops.png)

_출처: [devopsschool.com](https://www.devopsschool.com/blog/how-to-implement-gitops-using-terraform/)_

이 외에도 많은 장점이 있는데, GitOps가 의미하는 것은 단순히 Git으로 이력을 관리할 수 있다는 것을 넘어서, 함께 Git 기반으로 협업을 할 수 있다는 것을 의미합니다. 또한, AI Agent가 많이 발달하여 소스 코드의 형태로 인프라를 나타내는 것이 매우 큰 레버리지를 가지고 있다고 생각하였고, 앞으로 더욱 발전할 것이기 때문에 미래를 생각해서도 지금부터 소스 코드 기반으로 컨텍스트를 관리하는 것이 중요하다고 판단했습니다.

### 1.2 Terraform의 도입과 고민

![Terraform 설계도 한 장으로 여러 서비스를 프로비저닝하는 모습](images/terraform-provisioning.png)

인프라파트에 합류하기 이전에도 소수의 서비스를 테라폼으로 프로비저닝하고 있었고, 도입이 논의되고 있었기 때문에 테라폼을 도입하는 것은 어렵지 않았습니다. 바로 가장 먼저 들어온 서비스 프로비저닝 요청 건을 테라폼으로 처리하기 시작했고, 전사 인프라를 하나의 Github 레포지토리로 관리할 수 있도록 신경을 많이 써야했습니다.

가장 큰 고민은 단순히 인프라의 관리를 소스 코드로 하는 것을 넘어서, 인프라 프로비저닝을 자동화하고 반복되는 프로세스를 없앨 수 있도록 할 수 있는가? 이었습니다. 테라폼을 도입하고 혼자서 AI Agent를 통해서 빠르게 작업을 하기 시작하면서 프로비저닝 요청이 들어오면 빠르게 처리할 수 있는 상황이 되었습니다. 하지만 데브옵스의 역할은 개발(dev)와 운영(ops)의 벽을 허무는 것이기 때문에, 이 테라폼을 통해서 개발자가 스스로 인프라를 프로비저닝할 수 있는 **플랫폼**(platform)을 만들 수 있을지 고민하게 되었습니다.

---

## 2. Terraform으로 ECS 들여오기

![non-prod와 prod 계정, 환경별 칸에 leaf를 정리하는 모습](images/leaf-by-account-env.png)

Terraform을 쓰기로 결정한 뒤, 본격적으로 작업하기 전에 꼭 정해야 할 것들부터 정리했습니다. 수십 개의 서비스와 여러 환경을 하나의 레포에 담으려면 "무엇을 어디에 둘 것인가"라는 규칙이 먼저 필요했기 때문입니다. 이번 장에서는 그 규칙인 leaf 구조와, ECS를 옮기면서 가장 신경 썼던 task definition 관리 방식을 소개하겠습니다.

### 2.1 계정과 환경으로 나눈 leaf 구조

레포의 디렉터리는 `<리소스>/<서비스명>/<환경>_<리전>/` 세 단계로 나뉩니다. 가장 안쪽 디렉터리를 **leaf**라고 부르는데요, leaf 하나가 곧 하나의 Terraform 루트 모듈이자 하나의 state 파일입니다.

```text
terraform/
├── modules/
│   └── ecs_app_service/            # ECS 서비스 공용 모듈
├── variables/
│   └── var_global.tf               # 모든 leaf가 심볼릭 링크로 공유
├── ecs/
│   ├── example-backend/
│   │   ├── dev_apnortheast2/       # leaf (non-prod 계정)
│   │   ├── stage_apnortheast2/     # leaf (non-prod 계정)
│   │   └── prod_apnortheast2/      # leaf (prod 계정)
│   └── test-shared/                # 환경 공용 리소스(클러스터, ALB 리스너, IAM role)
├── rds/
├── elasticache/
├── s3/
│   └── ecs-taskdefs/               # task definition 정본 저장소 (2.4)
└── ...
```

환경이 곧 계정을 결정합니다. 팀스파르타는 prod, non-prod, sandbox 세 개의 계정을 사용하는데요. 처음에는 하나의 AWS 계정에 모든 환경이 섞여 있었지만, 지금은 prod만 prod 계정에 두고 dev·test·stage는 모두 non-prod 계정으로 옮겼습니다. 계정이 다르면 state 버킷도 다르기 때문에 **계정 경계가 곧 state 경계**가 됩니다. 그래서 leaf의 `backend.tf`만 보면 이 leaf가 어느 계정에 속하는지 알 수 있습니다.

```hcl
# ecs/example-backend/prod_apnortheast2/backend.tf
terraform {
  backend "s3" {
    bucket         = "<prod-account>-apnortheast2-tfstate"
    key            = "provisioning/terraform/ecs/example-backend/prod_apnortheast2/terraform.tfstate"
    region         = "ap-northeast-2"
    encrypt        = true
    dynamodb_table = "terraform-lock"
  }
}
```

서비스끼리 공유하는 리소스(클러스터, 공용 ALB 리스너, IAM role, DB 보안 그룹)는 `test-shared`, `stage-shared` 같은 별도 leaf가 소유하고, 각 서비스 leaf는 `terraform_remote_state`로 그 출력값을 읽기만 합니다. 공용 리소스를 서비스 leaf에서 다시 선언하는 일이 없도록 소유자를 한 곳으로 정해 둔 것입니다.

<!-- TODO: 다이어그램 — 계정 2개(prod / non-prod) 안에 shared leaf와 service leaf가 remote_state로 연결되는 그림 (images/leaf-structure.svg) -->

### 2.2 왜 leaf 단위로 쪼갰을까

처음부터 leaf 단위로 정한 것은 아니었습니다. 원래는 더 큰 모듈 단위로 관리하려고 했는데요. 앞으로는 많은 개발자가 AI 에이전트와 함께 직접 Terraform을 다루게 될 것이고, 자잘한 설정 작업은 에이전트가 맡게 되니 잘게 나눠 관리하는 오버헤드도 크지 않을 것이라고 판단했습니다. 그래서 leaf를 작게 가져가기로 했고, 지금까지 다음과 같은 이점을 누리고 있습니다.

1. **폭발 반경이 작습니다** — 한 서비스의 `plan`/`apply`가 다른 서비스의 리소스를 건드릴 수 없습니다. state가 작으니 `plan`도 빠르고, 실수가 나도 그 leaf 안에서 끝납니다.
2. **리뷰 단위와 배포 단위가 같습니다** — PR에서 건드린 leaf가 곧 Atlantis가 `plan`하는 프로젝트입니다. 리뷰어는 코드가 아니라 그 leaf의 plan 결과를 보고 승인합니다.
3. **경로로 권한을 나눌 수 있습니다** — `dev*`·`test*`·`stage*` leaf는 승인 없이 apply할 수 있고, `prod_*` leaf나 DB·네트워크처럼 되돌리기 어려운 경로만 CODEOWNERS로 인프라 담당자의 승인을 받습니다. 디렉터리 이름 자체가 권한 정책이 되는 셈입니다.

대신 leaf가 많아진다는 비용이 있습니다. 이 글을 쓰는 시점에 레포의 leaf는 230개를 넘었고, ECS만 90개가 넘습니다. leaf가 늘수록 Atlantis 설정(`atlantis.yaml`)에 새 leaf를 등록하는 것을 잊기 쉬운데, 등록되지 않은 leaf는 PR에서 아무것도 `plan`되지 않았는데도 체크가 초록불로 뜹니다. 그래서 `atlantis.yaml`은 손으로 고치지 않고 디렉터리를 스캔해 만드는 생성 스크립트로 관리하며, CI에서 누락 여부를 검사하고 있습니다.

![여러 leaf에 걸친 PR에 Atlantis가 leaf별 apply 명령어를 코멘트로 남긴 모습](images/atlantis-apply-comments.png)

위 사진은 PR에 `atlantis apply` 코멘트를 남기면 변경 사항을 적용해 주는 Atlantis를 사용하는 모습입니다. 일본팀 프로덕트 엔지니어분이 올린 PR에 적용 방법을 코멘트로 안내했는데, leaf가 여러 개에 걸쳐 있다 보니 apply 명령어를 무려 9번이나 입력해야 했습니다. 번거로운 것은 사실이지만, 이 작업 또한 AI 에이전트에게 맡길 수 있습니다. 실제로 적용할 때는 에이전트가 test → stage → prod 순서로 하나씩 apply하면서 검증합니다.

### 2.3 ECS 서비스 모듈

leaf가 많아도 내용이 제각각이면 의미가 없습니다. 그래서 ECS 서비스를 구성하는 리소스를 `ecs_app_service` 모듈 하나로 묶었습니다. 이 모듈 하나가 다음 리소스를 모두 만듭니다.

- ECR 레포지토리(server, proxy)와 라이프사이클 정책
- task definition과 ECS 서비스
- ALB 타깃 그룹과 리스너 룰(공용 ALB 모드 / 전용 ALB 모드)
- CloudWatch 로그 그룹, 서비스 보안 그룹, Route 53 레코드

leaf는 이 모듈을 호출하고 `terraform.tfvars`에 서비스별 값만 채웁니다. 새 서비스를 만들 때는 같은 환경의 형제 leaf를 참고해 값만 바꾸면 되는데요, 이 과정을 `/provisioning`이라는 Claude Code 스킬로 만들어 두어 개발자가 직접 PR을 올릴 수 있게 했습니다.

<!-- TODO: 모듈 호출부 예시를 실제 leaf에서 민감 정보를 지우고 붙이기 -->

```hcl
# (예시) 서비스 leaf의 main.tf
module "service" {
  source = "../../../modules/ecs_app_service"

  service_name = "example-backend-prod"
  task_config = {
    cpu    = 1024
    memory = 2048
  }
  shared_alb_config = { ... }
}
```

이 모듈에서 가장 중요한 부분은 오히려 **Terraform이 관리하지 않기로 한 것**입니다.

```hcl
# modules/ecs_app_service/service.tf
resource "aws_ecs_service" "this" {
  # ...
  lifecycle {
    ignore_changes = [desired_count, task_definition]
  }
}

# modules/ecs_app_service/task_definition.tf
resource "aws_ecs_task_definition" "this" {
  # ...
  lifecycle {
    ignore_changes = [container_definitions]
  }
}
```

`desired_count`는 오토스케일링이, 실행 중인 task definition과 이미지는 배포 파이프라인이 바꿉니다. 이 값들까지 Terraform이 들고 있으면 `apply`할 때마다 런타임이 바꿔 둔 값을 코드의 값으로 되돌리려고 합니다. 그래서 Terraform은 서비스의 **뼈대**만 처음 한 번 만들고, 이후의 런타임 상태는 각자의 소유자에게 넘깁니다. 그렇다면 CPU, 메모리, 환경 변수, 사이드카처럼 task definition의 내용은 누가 관리해야 할까요? 이 질문에 대한 답이 바로 다음 절입니다.

### 2.4 task definition을 S3에서 관리하기

![task definition 카드를 S3 상자에 모아 두는 모습](images/taskdefs-in-s3.png)

ECS를 Terraform으로 옮기면서 가장 공을 들인 부분입니다. 원래는 각 애플리케이션 레포가 자기 task definition을 직접 들고 있었기 때문에, 같은 설정을 서비스 10개에 적용하려면 레포 10개에서 따로 작업해야 했습니다. 그래서 모든 서비스의 task definition을 Terraform에서 한 번에 관리하는 것을 목표로 잡았습니다. 결론부터 말씀드리면 최종 구조는 task definition을 S3에 모아 두는 방식이고, 여기까지 오는 데 세 단계의 고민을 거쳤습니다.

#### 1단계 — 레포마다 있는 task-definition.json

![GitHub Actions에서 제공하는 Deploy to Amazon ECS 워크플로 템플릿](images/deploy-to-ecs-template.png)

GitHub Actions로 ECS에 배포할 때 가장 흔한 방법은 GitHub이 제공하는 Deploy to Amazon ECS 템플릿입니다. `amazon-ecs-render-task-definition` 액션이 **애플리케이션 레포에 들어 있는** `task-definition.json`의 이미지만 새 태그로 바꾸고, `amazon-ecs-deploy-task-definition` 액션이 그 파일을 새 revision으로 등록해 배포합니다.

```yaml
# 기존: 앱 레포의 workflow
- uses: aws-actions/amazon-ecs-render-task-definition@v1
  with:
    task-definition: task-definition.json # 앱 레포 안의 파일
    container-name: server
    image: ${{ steps.build.outputs.image }}
- uses: aws-actions/amazon-ecs-deploy-task-definition@v2
  with:
    task-definition: ${{ steps.render.outputs.task-definition }}
    service: example-backend-prod
    cluster: prod-cluster
```

서비스가 몇 개뿐이라면 문제없는 방식입니다. 하지만 서비스가 수십 개가 되면 task definition도 수십 개의 레포에 흩어집니다. 모든 서비스의 Datadog 사이드카 설정을 바꾸거나 로그 설정을 통일하려면, 수십 개의 레포를 열어 각자의 `task-definition.json`을 확인하고 PR을 따로 올려야 합니다. 게다가 이 파일은 인프라 설정인데도 앱 레포에 있어서, 실제로 어떤 값으로 돌고 있는지 인프라 쪽에서는 알 수가 없었습니다.

실제로 Node.js 서비스에서 이벤트 루프나 가비지 컬렉터 같은 런타임 메트릭을 수집하려고 `DD_RUNTIME_METRICS_ENABLED`를 켜야 했을 때, 레포마다 들어가서 직접 작업해야 했습니다. 작업은 오래 걸리고, 나중에 어느 서비스에 적용되었는지 확인하기도 어렵습니다. 애플리케이션 개발자 입장에서도 자기 레포에 인프라 설정 커밋이 계속 쌓이면서 불필요한 컨텍스트가 늘어납니다.

#### 2단계 — Terraform이 등록한 최신 revision을 CI가 가져오기

처음 떠올린 방법은 task definition을 Terraform의 `aws_ecs_task_definition` 리소스로 관리하고, CI는 레포의 파일 대신 AWS CLI로 ECS에 등록된 최신 revision을 가져와 이미지만 바꿔 배포하는 것이었습니다.

```bash
# CI에서 최신 revision을 가져와 이미지만 바꾸는 방식
aws ecs describe-task-definition \
  --task-definition example-backend-prod \
  --query taskDefinition > task-definition.json
```

앱 레포에서 task definition 파일이 사라진다는 점에서는 한 걸음 나아간 방식입니다. 하지만 이 구조에서는 task definition이 ECS 서비스와 같은 leaf 안에 묶이게 됩니다. task definition이 서비스 모듈의 일부라서, 설정 하나를 모든 서비스에 반영하려면 결국 서비스 leaf 수십 개를 하나씩 plan하고 apply해야 합니다. 레포를 돌아다니던 일이 leaf를 돌아다니는 일로 바뀌었을 뿐이었습니다.

소유권도 꼬입니다. Terraform이 만든 revision 위에 CI가 배포할 때마다 새 revision을 등록하기 때문에, "가장 최신 revision"이 Terraform이 만든 것인지 CI가 만든 것인지가 매번 달라집니다. 2.3에서 `container_definitions`를 `ignore_changes`로 막아 둔 이유도 여기에 있습니다. Terraform이 task definition을 직접 들고 있으면, 오래된 이미지 태그를 가진 revision이 배포를 덮어쓸 위험이 생깁니다.

<!-- TODO: 2단계에서 실제로 막혔던 기억이 더 있다면 보충 -->

#### 3단계 — task definition만 모아서 S3에 올리기

결국 task definition만 따로 떼어서 한곳에 모아야 한다는 결론에 이르렀고, 그 한곳으로 고른 것이 S3입니다. 환경마다 `s3/ecs-taskdefs/<환경>_<리전>/` leaf 하나를 두고, 그 아래 `taskdefs/`에 서비스별 JSON 파일을 둡니다. Terraform은 이 파일들을 S3 버킷에 그대로 업로드할 뿐, 렌더링도 가공도 하지 않습니다.

```hcl
# s3/ecs-taskdefs/prod_apnortheast2/main.tf
resource "aws_s3_object" "taskdef" {
  for_each = fileset("${path.module}/taskdefs", "*.json")

  bucket = aws_s3_bucket.taskdefs.id
  key    = "taskdefs/${each.value}"
  source = "${path.module}/taskdefs/${each.value}"
  etag   = filemd5("${path.module}/taskdefs/${each.value}") # 내용이 바뀐 파일만 다시 올린다
}
```

이제 애플리케이션 레포의 CI는 자기 레포의 파일도, ECS의 최신 revision도 아닌 S3의 JSON을 가져와서 이미지 태그만 커밋 SHA로 바꾼 뒤 새 revision을 등록하고 배포합니다.

```text
Developer edits taskdefs/<svc>-<env>.json   (cpu, memory, env, ports, sidecars)
        │  PR → Atlantis plan → apply
        ▼
Terraform uploads JSON to S3               (nothing changes in production yet)
        │
        ▼
App repo push → CI (AWS OIDC)
   ├─ fetch task definition from S3
   ├─ replace only the image → :<commit SHA>
   └─ register new revision → update-service
        ▲
        └─ the new shape and the new image go live together here
```

<!-- TODO: 위 흐름을 SVG 다이어그램으로 (images/taskdef-s3-flow.svg) -->

핵심은 소유권의 분리입니다. task definition의 **모양**(shape)은 Terraform이, 이미지는 CI가 소유합니다. CPU나 환경 변수를 바꾸고 싶으면 Terraform 레포에 PR을 올리고, 코드를 배포하고 싶으면 앱 레포에 push하면 됩니다. 두 경로가 섞이지 않습니다.

여기에 한 가지 결정을 더했습니다. task definition 변경 중 환경 변수를 추가하거나 새 시크릿을 연결하는 작업은 인프라 담당자보다 애플리케이션 개발자가 훨씬 자주 하게 됩니다. 이런 변경마다 인프라 담당자의 리뷰를 기다리게 하면 그 자체가 병목이 되는데요. 그래서 2.2에서 소개한 경로 기반 권한 규칙에 예외를 하나 두었습니다. `s3/ecs-taskdefs/*/taskdefs/*.json` 경로는 prod를 포함해도 별도 코드 리뷰 승인 없이 `atlantis apply`로 바로 적용할 수 있습니다. 어차피 apply만으로는 라이브 서비스가 바뀌지 않고, 실제 반영은 개발자 본인이 앱을 다시 배포하는 시점에 일어나기 때문에 가능한 판단이었습니다.

#### 이렇게 해서 얻은 것

- **설정 변경이 한 곳에서 끝납니다** — 모든 서비스의 task definition이 환경별 leaf 하나에 모여 있으니, 공통 설정을 바꾸는 것도 PR 하나, apply 한 번이면 됩니다.
- **리뷰하기 쉽습니다** — task definition이 git에 JSON 파일로 있기 때문에, PR의 Files changed 탭에서 어떤 컨테이너의 어떤 환경 변수가 어떻게 바뀌었는지 줄 단위로 바로 보입니다.
- **apply가 곧 배포는 아닙니다** — Terraform apply는 S3에 새 shape를 올릴 뿐 라이브 서비스를 건드리지 않습니다. 실제 반영은 다음 배포 때 일어나므로, 인프라 변경이 예상치 못한 재시작을 일으키지 않습니다.
- **이미지는 항상 불변 태그입니다** — CI가 `:latest` 대신 커밋 SHA 태그로 이미지를 바꾸기 때문에, 태스크가 재시작되어도 배포되지 않은 코드가 뜨는 일이 없습니다. 배포마다 새 revision이 생기므로 롤백도 이전 revision을 다시 가리키면 됩니다.
- **계정을 옮기기 쉽습니다** — S3의 객체는 git에 있는 JSON의 파생물일 뿐입니다. 서비스를 다른 계정으로 옮길 때 데이터를 마이그레이션할 필요 없이, 버킷 이름만 바꿔 apply하면 됩니다.

#### 한계 — plan으로는 JSON의 무엇이 바뀌는지 보이지 않는다

아쉬운 점도 하나 있습니다. 처음에는 JSON을 고치고 `terraform plan`을 돌리면 task definition의 어느 부분이 어떻게 바뀌는지 plan에 나올 거라고 기대했는데요. 실제 plan은 이렇게 나옵니다.

```text
  # aws_s3_object.taskdef["example-backend-prod.json"] will be updated in-place
  ~ resource "aws_s3_object" "taskdef" {
      ~ etag   = "a2f9c8bea262e652e0f9f3654a59ca22" -> "afa00ffa93b4edd6f57fa9f1eab09695"
        id     = "taskdefs/example-backend-prod.json"
        # (12 unchanged attributes hidden)
    }
```

이유는 `aws_s3_object`에 파일을 넘기는 방식에 있습니다. `source` 인자에 넘기는 것은 파일의 **내용이 아니라 경로**입니다. Terraform state에는 `source = "./taskdefs/example-backend-prod.json"`이라는 문자열만 저장되고, 파일 내용은 apply할 때 읽어서 S3에 올릴 뿐 state에 남지 않습니다. 경로 문자열은 바뀌지 않으니, Terraform이 "파일이 바뀌었다"는 사실을 알 수 있는 유일한 단서는 `etag`에 넣은 `filemd5()` 값입니다. 결국 plan이 보여 줄 수 있는 것은 **MD5 해시가 바뀌었다**는 사실뿐이고, JSON의 어느 줄이 바뀌었는지는 알 수 없습니다.

반대로 파일 내용을 `content = file(...)`처럼 문자열로 넘기면, Terraform은 그 문자열이 JSON이라는 것을 알아보고 plan에서 구조적인 diff를 보여 줍니다.

```text
      ~ content = jsonencode(
          ~ {
              ~ containerDefinitions = [
                  ~ {
                      ~ environment = [
                            {
                                name  = "NODE_ENV"
                                value = "production"
                            },
                          + {
                              + name  = "DD_RUNTIME_METRICS_ENABLED"
                              + value = "true"
                            },
                        ]
```

다만 이 방식은 서비스 수십 개의 task definition 전문이 state 파일에 그대로 들어가고, plan 출력도 그만큼 길어진다는 비용이 있습니다. 그래서 지금은 `source` + `etag` 방식을 유지하면서, **변경 내용은 GitHub PR diff로 리뷰하고 plan은 "어떤 파일이 다시 올라가는지"만 확인하는 용도**로 역할을 나눠 쓰고 있습니다. 무엇이 바뀌는지는 PR에서, 실제로 무엇이 적용되는지는 plan에서 보는 셈입니다.

<!-- TODO: content 방식으로 바꿀지에 대한 개인적인 생각이 있다면 한 줄 -->

<!-- TODO: 적용 순서 — stage에서 먼저 검증 → prod로 확장 → dev/test 이식 같은 실제 롤아웃 과정 -->

### 2.5 이관하면서 겪은 문제들

모든 것이 순조롭지는 않았습니다. 특히 "Terraform이 소유하는 것과 런타임이 소유하는 것"의 경계에서 문제가 많이 생겼습니다.

<!-- TODO: 아래는 레포 문서에 남아 있는 사례를 요약한 것. 공개해도 되는 범위인지 확인하고 다듬기 -->

- **오래된 이미지로 롤백되는 문제** — leaf의 `tfvars`에 남아 있던 `image_tag`는 처음 만들 때의 값이라, 실제로 돌고 있는 이미지보다 한참 오래된 상태였습니다. Terraform이 만든 task definition revision으로 서비스를 다시 가리키자 몇 시간 동안 예전 코드가 서비스되었습니다. 이후 `ignore_changes = [container_definitions]`로 컨테이너 정의를 CI에 완전히 넘겼습니다.
- **apply가 서비스를 재시작시키는 문제** — 서비스 리소스에 `force_new_deployment = true`가 있어서, 보안 그룹만 바꾸는 apply에도 라이브 태스크가 롤링 재시작되었습니다. 인프라 변경과 재배포를 분리하는 것이 생각보다 중요했습니다.
- **컨테이너 이름 불일치** — CI의 render 단계는 지정한 컨테이너 이름과 JSON 안의 이름이 정확히 같을 때만 이미지를 교체하고, 다르면 에러 없이 넘어갑니다. 서비스마다 컨테이너 이름이 제각각이어서, 전부 `server`/`proxy`로 통일한 뒤에야 이 방식을 붙일 수 있었습니다.
- **조용히 넘어가는 plan** — 새 leaf를 `atlantis.yaml`에 등록하지 않으면 PR 체크가 초록불인데도 실제로는 아무것도 plan되지 않았습니다. task definition JSON처럼 `fileset()`으로 읽는 데이터 파일 변경이 감지되지 않은 적도 있었습니다.

<!-- TODO: import 과정의 경험 — 기존 리소스를 import할 때 겪은 diff/drift 문제가 있다면 추가 -->

---

## 3. 공용 GitHub Actions 파이프라인

<!-- 전환 문장 -->

(예시) 인프라를 코드로 정리하고 나니, 이번에는 그 위에 애플리케이션을 올리는 과정 — 배포 — 이 서비스마다 다르다는 점이 눈에 띄었습니다.

### 3.1 레포마다 복사된 workflow의 문제

<!-- TODO: 각 레포에 비슷하지만 조금씩 다른 workflow 파일이 있던 상황. 하나를 고치면 N개를 고쳐야 했던 경험 -->

### 3.2 Reusable workflow로 파이프라인 모으기

<!-- TODO: 공용 파이프라인 레포 구조, reusable workflow(workflow_call) 또는 composite action 선택 이유, 버전 관리(tag) 전략 -->

```yaml
# (예시) 서비스 레포 쪽 호출부
jobs:
  deploy:
    uses: <org>/<shared-pipeline-repo>/.github/workflows/ecs-deploy.yml@v1
    with:
      service-name: service-a
      environment: prod
    secrets: inherit
```

<!-- TODO: 파이프라인 흐름 다이어그램 — build → push(ECR) → task def render → deploy (images/shared-pipeline-flow.svg) -->

### 3.3 인증과 권한

<!-- TODO: OIDC로 AWS role assume, 환경별 role 분리 등 -->

### 3.4 바뀐 것

<!-- TODO: before/after 비교. 숫자가 있으면 좋다 (배포 시간, 신규 서비스 온보딩 시간, workflow 파일 줄 수 등) -->

| 항목                  | Before                           | After                  |
| --------------------- | -------------------------------- | ---------------------- |
| 신규 서비스 배포 설정 | (예: workflow 파일 복붙 후 수정) | (예: 10줄 호출부 작성) |
| 파이프라인 수정       | (예: N개 레포 PR)                | (예: 공용 레포 1개 PR) |

---

## 마치며

<!-- 요약 → 개인적 인사이트 → 다음 방향(2편 예고) -->

(예시) 1편에서는 콘솔 중심이던 인프라를 Terraform으로 옮기고, 배포 파이프라인을 하나로 모은 과정을 정리해 보았습니다.

<!-- TODO: 개인적 인사이트 — 예: "IaC의 진짜 가치는 자동화보다 '합의된 기록'에 있다고 느꼈습니다" -->

(예시) 다음 글에서는 이렇게 정리된 기반 위에서 Datadog과 로깅, 환경 변수 주입 방식을 어떻게 정비했는지 이어서 소개하겠습니다. 긴 글 읽어 주셔서 감사합니다.

### References

<!-- TODO: 참고한 문서 링크 -->

- [Terraform — Import](https://developer.hashicorp.com/terraform/language/import)
- [GitHub Actions — Reusing workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
