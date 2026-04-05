---
title: "AWS 프로덕션 배포: 프론트엔드부터 백엔드까지"
date: 2024-10-21
draft: true
tags: ["aws", "deployment", "ecs", "cloudfront"]
translationKey: "aws-production-deployment"
summary: "S3 + CloudFront로 프론트엔드를, ECS + ALB로 백엔드를 배포하고, Route53으로 도메인을 연결하는 AWS 프로덕션 배포 전 과정을 다룹니다."
---

팀 프로젝트에서 PM 겸 팀장 포지션으로 AWS 기반의 프론트엔드와 백엔드 CI/CD 파이프라인 전체를 구축했다. Free Tier 수준에서 처리할 수 있는 범위 내에서 다양한 AWS 서비스를 활용하여 아키텍처를 구성하였으며, 이 글에서는 그 전 과정을 정리한다.

![전체 아키텍처 개요](images/fe_img.png)

<!-- TODO: 다이어그램 필요 - 프론트엔드(S3 + CloudFront + Route53)와 백엔드(ECS + ALB + Route53)를 포함하는 전체 AWS 아키텍처 다이어그램 -->

## 준비물

본격적인 배포에 앞서 다음 항목이 준비되어야 한다.

- **도메인 네임**: 가비아, Route53 등에서 구매
- **AWS 계정**: 본 글은 Free Tier 기반을 상정하고 작성되었음

---

## 1. 프론트엔드: S3 + CloudFront + Route53

![프론트엔드 배포 흐름](images/fe_img_1.png)

### 프론트엔드 배포 이론

프론트엔드 배포의 핵심은 **정적 콘텐츠 서빙**이다. React 프로젝트를 예로 들면, `npm run build`를 통해 생성되는 압축된 HTML, CSS, JS 파일들을 사용자에게 제공하기만 하면 된다.

```bash
# React 프로젝트 생성
npm install -g create-react-app
npx create-react-app my-app
```

![React 프로젝트 디렉토리 구조](images/fe_img_2.png)

개발 환경에서는 `npm start`로 실행하지만, 프로덕션 배포를 위해서는 정적 콘텐츠를 빌드해야 한다.

```bash
npm run build
```

![빌드 결과물](images/fe_img_3.png)

빌드가 완료되면 `build` 디렉토리에 압축된 정적 콘텐츠가 생성된다. 이 파일들을 nginx나 Apache HTTP Server 등으로 서빙하면 프론트엔드 배포가 완료된다. 이제 이 과정을 AWS와 GitHub Actions로 자동화해 보자.

### IAM 생성 및 S3 버킷 설정

AWS에서 정적 콘텐츠 저장을 담당하는 서비스는 **S3 Bucket**이다. 그리고 GitHub Actions에서 AWS 리소스에 접근하기 위해 **IAM(Identity and Access Management)** 사용자를 생성해야 한다.

#### Amazon IAM

IAM 콘솔에서 사용자를 생성한다. 이름은 `frontend-deploy`로 지정했다.

![IAM 사용자 생성](images/fe_img_4.png)

![IAM 권한 설정](images/fe_img_5.png)

권한 설정에서 **직접 정책 연결 - AmazonS3FullAccess**를 연결한다.

사용자 생성 후, 해당 사용자를 클릭하여 액세스 키를 발급한다.

![IAM 사용자 목록](images/fe_img_6.png)

![액세스 키 만들기](images/fe_img_7.png)

**액세스 키 만들기**를 눌러 키를 생성한다.

![액세스 키 유형 선택](images/fe_img_8.png)

![액세스 키 발급 완료](images/fe_img_9.png)

다음 정보를 안전하게 저장해 둔다.

- **액세스 키**
- **비밀 액세스 키**

#### Amazon S3

S3 콘솔에서 버킷을 생성한다.

![S3 버킷 생성](images/fe_img_10.png)

적당한 이름과 함께 버킷을 만들되, **퍼블릭 액세스 차단 설정을 해제**한다.

![퍼블릭 액세스 차단 해제](images/fe_img_11.png)

버킷 생성 후 두 가지 설정이 필요하다.

**1) 정적 웹사이트 호스팅 활성화**

버킷 속성 탭에서 정적 웹사이트 호스팅을 활성화한다.

![정적 웹사이트 호스팅](images/fe_img_12.png)

![호스팅 설정](images/fe_img_13.png)

**2) 버킷 정책 설정**

권한 탭에서 버킷 정책을 설정한다.

![버킷 정책 편집](images/fe_img_14.png)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
        }
    ]
}
```

S3에 올라간 정적 콘텐츠를 모든 사용자가 조회할 수 있도록 허용하는 정책이다.

### CloudFront (CDN)

S3 버킷을 서울 리전에 생성한 경우, 해외 사용자가 정적 콘텐츠를 요청하면 트래픽이 해외로 나가면서 지연이 발생한다. **CloudFront**를 통해 배포하면 전 세계 216개 엣지 로케이션에 캐싱된 콘텐츠를 제공할 수 있어 S3와의 조합이 매우 효율적이다.

CloudFront 콘솔에서 배포를 생성한다.

![CloudFront 오리진 설정](images/fe_img_15.png)

![웹 엔드포인트 설정](images/fe_img_16.png)

**Origin domain**에서 S3 버킷을 지정하고, 웹 엔드포인트로 설정하여 80번 포트로 데이터를 서빙하도록 한다.

![보안 및 가격 설정](images/fe_img_17.png)

보안 보호(WAF, Shield)는 비용이 발생할 수 있으므로 상황에 맞게 설정한다. 가격 분류에서는 엣지 로케이션 수를 선택할 수 있는데, 비용 절감을 위해 적절한 옵션을 선택한다.

![CloudFront 배포 도메인](images/fe_img_18.png)

생성 후 **배포 도메인 이름**의 CDN URL로 접속하면 페이지를 확인할 수 있다.

### Route53 도메인 설정

`cloudfront.net`으로 끝나는 도메인 대신, 구매한 커스텀 도메인으로 접근할 수 있도록 Route53을 설정한다.

#### ACM 인증서 발급

도메인에 대한 SSL 인증서를 **ACM(AWS Certificate Manager)**으로 발급받아야 한다. 중요한 점은 CloudFront와 Route53을 연결하기 위해 ACM 인증서를 반드시 **버지니아 북부(us-east-1)** 리전에 생성해야 한다는 것이다.

> **참고**: [AWS 공식 문서 - CloudFront 배포에 사용할 SSL 인증서를 미국 동부(버지니아 북부) 리전으로 마이그레이션](https://repost.aws/ko/knowledge-center/migrate-ssl-cert-us-east)

#### Route 53 호스팅 영역 생성

Route 53 콘솔에서 호스팅 영역을 생성한다.

![Route 53 호스팅 영역 생성](images/fe_img_19.png)

![호스팅 영역 레코드](images/fe_img_20.png)

NS 유형(네임서버)에 해당하는 **값/트래픽 라우팅 대상**을 도메인을 구매한 곳(가비아 등)의 네임서버 설정에 입력해야 한다.

![가비아 네임서버 설정](images/fe_img_21.png)

가비아의 경우 **로그인 - my가비아 - 도메인 - 네임서버 설정**에서 변경할 수 있다.

#### ACM 요청 및 연결

ACM 콘솔(버지니아 북부 리전)에서 **퍼블릭 인증서 요청**을 선택한다.

![ACM 인증서 요청](images/fe_img_22.png)

![도메인 이름 입력](images/fe_img_23.png)

도메인 이름을 입력하고 요청하면 몇 분 이내에 발급이 완료된다.

![인증서 발급 완료](images/fe_img_24.png)

발급된 인증서를 클릭한다.

![Route 53 레코드 생성](images/fe_img_25.png)

**Route 53에서 레코드 생성** 버튼을 클릭해 호스팅 영역과 연결한다. 마지막으로 Route 53에서 A 유형 레코드를 추가하여 CloudFront 배포를 연결한다.

![A 레코드 추가](images/fe_img_26.png)

### 프론트엔드 GitHub Actions

GitHub 리포지토리의 Settings에서 Secrets를 설정한다.

![GitHub Secrets 설정](images/fe_img_27.png)

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `S3_BUCKET_NAME`

`.github/workflows/deployment.yaml` 파일을 생성한다.

```yaml
name: deploy to aws

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

env:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  S3_BUCKET_NAME: ${{ secrets.S3_BUCKET_NAME }}
  AWS_REGION: ap-northeast-2

jobs:
  cicd:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20.x'

      - name: Cache yarn dependencies
        uses: actions/cache@v3
        with:
          path: |
            node_modules
            ~/.yarn/cache
          key: ${{ runner.os }}-yarn-${{ hashFiles('yarn.lock') }}
          restore-keys: |
            ${{ runner.os }}-yarn-

      - name: Install dependencies
        run: yarn install

      - name: Build the project
        run: yarn build

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ env.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ env.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Upload to S3
        run: |
          aws s3 sync ./dist s3://${{ env.S3_BUCKET_NAME }} --delete
```

main 브랜치에 머지하면 자동으로 빌드 및 배포가 진행된다.

![GitHub Actions 실행 결과](images/fe_img_28.png)

프론트엔드 배포 아키텍처를 요약하면 다음과 같다.

![프론트엔드 배포 아키텍처](images/fe_img_29.png)

---

## 2. 백엔드: ECS + ALB + Route53

![백엔드 전체 아키텍처](images/be_img.png)

백엔드는 GitHub Actions를 통해 IAM으로 접근하여, 코드 푸시 시 ECS를 통해 자동 배포되도록 구성했다. 프론트엔드와 마찬가지로 완전한 서버리스 아키텍처를 채택하여 스케일 아웃이 설정에 따라 자동으로 진행된다.

![백엔드 CI/CD 흐름](images/be_img_1.png)

### 사전 유의사항

1. **비용**: ECS Fargate는 CPU와 메모리 사용량에 따라 비용을 지불한다. 적절한 수준으로 설정하는 것이 비용 절감의 핵심이다.
2. **Health Check**: 서버리스 아키텍처에서 로드 밸런서가 서버 상태를 확인할 수 있도록 health check API 엔드포인트가 필요하다 (예: `PREFIX/utils/health`).

### 백엔드 배포 이론 (Spring)

프론트엔드가 정적 콘텐츠를 배포하는 것과 달리, 백엔드는 **컨테이너 기반**으로 배포한다. Spring 서버의 경우 Gradle이나 Maven으로 프로젝트를 컴파일하면 `.jar` 파일이 생성되고, `java -jar` 명령어로 JVM 위에서 실행하는 구조다.

```dockerfile
FROM openjdk:17

ARG JAR_FILE_PATH=build/libs/*.jar

WORKDIR /apps

COPY $JAR_FILE_PATH app.jar

EXPOSE 8080

CMD ["java", "--enable-preview", "-jar", "app.jar"]
```

여기서 중요한 점은 **환경 변수 주입 타이밍**이다. Node.js 서버와 달리 Spring은 `.env`와 `application.yml`을 빌드 전에 주입해야 한다. 따라서 GitHub Actions에서 빌드 후 도커라이징하는 순서로 진행해야 한다.

### IAM 생성

백엔드 배포용 IAM 사용자를 생성한다.

![백엔드 IAM 생성](images/be_img_2.png)

다음 두 가지 권한을 부여한다.

![IAM 권한 설정 - ECS, ECR](images/be_img_3.png)

- **AmazonECS_FullAccess**
- **AmazonElasticContainerRegistryPublicPowerUser**

![액세스 키 발급](images/be_img_4.png)

액세스 키와 시크릿 액세스 키를 발급받아 저장한다.

### ECR 생성

**ECR(Elastic Container Registry)**은 Docker 이미지를 저장하는 리포지토리다.

![ECR 콘솔](images/be_img_5.png)

프라이빗 리포지토리를 생성한다.

![ECR 리포지토리 생성](images/be_img_6.png)

![ECR 리포지토리 생성 완료](images/be_img_7.png)

### ECS 서비스 구성 이해

ECS의 서비스들을 ALB를 포함하여 다음 순서로 구성한다.

![ECS 서비스 구조](images/be_img_8.png)

- **ECR (Elastic Container Registry)**: 도커라이징된 이미지가 올라오는 허브
- **ECS (Elastic Container Service)**: ECR을 포함하여 클러스터, 서비스, 작업 정의 등이 모여 있는 서비스
- **태스크 정의 (Task Definition)**: 클러스터가 어떤 작업을 수행할지 정의하는 JSON 형식의 설정
- **클러스터 (Cluster)**: 태스크 정의 내용을 바탕으로 컨테이너를 관리하는 ECS의 핵심
- **서비스 (Service)**: 태스크 정의를 링크하여 클러스터가 해당 내용대로 오케스트레이션하도록 하는 단위

> AWS 콘솔에서 **서비스를 생성하는 과정에서만 ALB를 연결**할 수 있다. 따라서 ALB를 먼저 만든 후, 태스크 정의 -> 클러스터 -> 서비스 순서로 진행한다.

### ALB (Application Load Balancer)

ALB가 필요한 이유는 다음과 같다.

1. ECS 컨테이너는 생성될 때마다 다른 IP를 할당받으므로, 로드밸런싱이 필요하다
2. ECS는 프라이빗 서브넷에서 운영하고, ALB만 외부에 노출하는 것이 보안상 좋다
3. Route53에서 ALB에 도메인을 연결할 수 있다 (Route53 -> ALB -> ECS)

![로드 밸런서 콘솔](images/be_img_9.png)

EC2 콘솔의 로드 밸런서에서 **Application Load Balancer**를 선택한다.

![ALB 유형 선택](images/be_img_10.png)

#### VPC 및 가용 영역

![VPC 및 서브넷 설정](images/be_img_11.png)

기본 VPC를 사용하고, 가용 영역은 두 개 이상을 선택한다.

#### 보안 그룹

![보안 그룹 설정](images/be_img_12.png)

![HTTP/HTTPS 인바운드 규칙](images/be_img_13.png)

HTTP(80)와 HTTPS(443) 트래픽을 허용하는 보안 그룹을 생성하여 ALB에 연결한다.

#### 대상 그룹 (Target Group)

![리스너 및 라우팅 설정](images/be_img_14.png)

대상 그룹을 생성한다. Fargate의 awsvpc 네트워크 모드를 사용하므로 **IP 주소** 유형으로 만든다.

![대상 그룹 유형 선택](images/be_img_15.png)

![대상 그룹 설정](images/be_img_16.png)

**상태 검사(Health Check)** 경로를 API 서버의 URL로 지정한다. 로드 밸런서가 이 경로를 통해 서버 상태를 확인한다.

![Health Check 경로 설정](images/be_img_17.png)

대상 그룹을 생성하고 ALB에 등록한다.

![타겟 그룹 등록 완료](images/be_img_18.png)

### ECS 생성

#### 태스크 정의 (Task Definition)

![태스크 정의 콘솔](images/be_img_19.png)

**새 태스크 정의 생성**을 클릭한다.

![인프라 요구 사항 설정](images/be_img_20.png)

- **시작 유형**: Fargate (EC2보다 저렴하고 관리가 편리)
- **CPU/메모리**: 적절한 수준으로 설정 (Spring Boot의 경우 메모리가 너무 낮으면 Gradle 빌드가 실패할 수 있음)

![컨테이너 설정](images/be_img_21.png)

- **이미지 URI**: `{ECR URL}/{리포지토리명}:latest`
- **포트 매핑**: API 서버에서 사용하는 포트 (예: 8080)

#### 클러스터 (Cluster)

![클러스터 생성](images/be_img_22.png)

이름을 지정하고 **AWS Fargate(서버리스)**로 생성한다.

#### 서비스 생성

![클러스터 상세 페이지](images/be_img_23.png)

클러스터 상세 페이지에서 **서비스 생성** 버튼을 클릭한다.

![시작 유형 선택](images/be_img_24.png)

**시작 유형**을 선택한다.

![태스크 정의 연결](images/be_img_25.png)

이전에 만든 **태스크 정의**를 등록한다.

![서비스 설정 섹션들](images/be_img_26.png)

**네트워킹**과 **로드 밸런싱** 설정이 핵심이다.

**네트워킹 설정:**

![네트워킹 설정](images/be_img_27.png)

- 서브넷은 ALB와 동일하게 두 개 선택
- 보안 그룹은 8080 포트로 ALB의 보안 그룹에서만 접근 가능하도록 설정
- 퍼블릭 IP 켜짐

**로드 밸런싱 설정:**

![로드 밸런싱 설정](images/be_img_28.png)

**기존 로드 밸런서 사용**, **기존 리스너 사용**, **기존 대상 그룹 사용**을 선택하여 사전에 생성한 ALB 리소스를 연결한다.

서비스 생성 후 상태를 확인한다.

![서비스 실행 상태](images/be_img_29.png)

### 백엔드 GitHub Actions

Spring 서버는 빌드 전에 `.env` 및 `application.yml`을 주입해야 하므로, GitHub Actions에서 이를 처리한다.

```yaml
name: Deploy to Amazon ECS

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

env:
  AWS_REGION: ap-northeast-2
  ECR_REPOSITORY: my-ecr-repo
  ECS_SERVICE: my-ecs-service
  ECS_CLUSTER: my-ecs-cluster
  ECS_TASK_DEFINITION: my-task-definition
  CONTAINER_NAME: my-container

permissions:
  contents: read

jobs:
  deploy:
    name: Deploy
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: adopt

      - name: Cache Gradle Dependencies
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
          restore-keys:
            ${{ runner.os }}-gradle-

      - name: Before build, inject .env file
        run: echo "${{ secrets.PROD_ENV }}" > ./src/main/resources/.env

      - name: Build with Gradle
        run: ./gradlew clean build --refresh-dependencies -x test

      - name: Build, tag, and push image to Amazon ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

      - name: Download Task Definition Template
        run: |
          aws ecs describe-task-definition \
            --task-definition ${{ env.ECS_TASK_DEFINITION }} \
            --query taskDefinition \
            > task-definition.json

      - name: Fill in the new image ID in the Amazon ECS task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ steps.build-image.outputs.image }}

      - name: Deploy Amazon ECS task definition
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
```

이 워크플로우에서 주목할 점은 **task-definition**을 AWS에서 직접 가져온다는 것이다. `task-definition.json` 파일에는 민감 정보가 포함되어 있으므로 리포지토리에 커밋하지 않고 런타임에 다운로드하는 방식을 택했다.

GitHub Secrets에 올바른 IAM 키와 `.env` 내용을 등록하면 자동 배포가 동작한다.

![GitHub Actions 배포 성공](images/be_img_30.png)

### Route 53 및 ALB 설정 마무리

배포가 완료되면 Route53과 ALB의 최종 설정을 진행한다.

#### 서울 리전 ACM 인증서

CDN 배포를 위해 버지니아 북부에 ACM을 발급받았듯이, 백엔드는 **서울(ap-northeast-2)** 리전에 ACM 인증서를 별도로 요청한다.

![서울 리전 ACM 요청](images/be_img_31.png)

![ACM 발급 완료](images/be_img_32.png)

#### Route 53 A 레코드 추가

Route 53에서 `api.` 서브도메인에 대한 A 레코드를 추가한다. **별칭**을 선택하고 ALB를 연결한다.

![Route 53 A 레코드 - ALB 연결](images/be_img_32.png)

#### ALB HTTPS 리스너 추가

ALB에서 **리스너 추가**를 선택하여 HTTPS 리스너를 설정한다.

![ALB 리스너 현황](images/be_img_33.png)

![HTTPS 리스너 설정](images/be_img_34.png)

![ACM 인증서 연결](images/be_img_35.png)

서울 리전에서 발급받은 ACM 인증서와 함께 HTTPS 연결을 대상 그룹으로 연결한다.

![ALB 리소스 맵](images/be_img_36.png)

로드 밸런서의 리소스 맵에서 정상 작동을 확인할 수 있다.

![배포 완료 확인](images/be_img_37.png)

도메인으로 접속하면 정상적으로 배포된 것을 확인할 수 있다.

---

## 3. CI/CD 통합

전체 CI/CD 아키텍처를 정리하면 다음과 같다.

![전체 CI/CD 아키텍처](images/fe_img_30.png)

<!-- TODO: 다이어그램 필요 - 프론트엔드와 백엔드를 아우르는 통합 CI/CD 파이프라인 다이어그램 -->

### 프론트엔드 파이프라인

```
GitHub (main merge) -> GitHub Actions -> npm build -> S3 sync -> CloudFront 캐시 무효화
```

- **트리거**: main 브랜치 push 또는 PR
- **빌드**: Node.js 환경에서 `yarn build`
- **배포**: `aws s3 sync`로 빌드 결과물을 S3에 업로드
- **CDN**: CloudFront가 S3의 콘텐츠를 전 세계 엣지 로케이션에 배포

### 백엔드 파이프라인

```
GitHub (main merge) -> GitHub Actions -> .env 주입 -> Gradle build -> Docker build -> ECR push -> ECS 태스크 정의 업데이트 -> 서비스 롤링 업데이트
```

- **트리거**: main 브랜치 push 또는 PR
- **환경 변수**: GitHub Secrets에서 `.env` 파일을 주입
- **빌드**: JDK 17로 Gradle 빌드 후 Docker 이미지 생성
- **배포**: ECR에 이미지 푸시 후 ECS 태스크 정의를 업데이트하여 롤링 배포

### GitHub Secrets 관리

프론트엔드와 백엔드에서 공통으로 필요한 Secrets는 다음과 같다.

| Secret 이름 | 용도 |
|---|---|
| `AWS_ACCESS_KEY_ID` | IAM 액세스 키 |
| `AWS_SECRET_ACCESS_KEY` | IAM 시크릿 액세스 키 |
| `S3_BUCKET_NAME` | 프론트엔드 S3 버킷 이름 |
| `PROD_ENV` | 백엔드 .env 파일 내용 |

---

## 4. 경험에서 배운 것들

### 서버리스 아키텍처의 장점

- **자동 스케일링**: ECS Fargate를 통해 트래픽에 따라 컨테이너가 자동으로 증감한다
- **관리 편의성**: EC2 인스턴스를 직접 관리할 필요 없이 컨테이너 수준에서 배포가 이루어진다
- **배포 자동화**: CI/CD 파이프라인을 한 번 구축하면 개발자는 코드 푸시만으로 배포가 완료된다
- **팀 생산성 향상**: 배포 자동화를 통해 다른 업무(API Gateway, Lambda를 활용한 보조 서버 등)에 리소스를 투입할 수 있었다

### 주요 삽질 포인트

1. **ACM 리전 이슈**: CloudFront + Route53 조합에서는 ACM을 반드시 **us-east-1(버지니아 북부)**에 생성해야 한다. ALB의 경우에는 해당 리소스가 있는 리전(서울 등)에 생성해야 한다.
2. **ECS 서비스와 ALB 연결 타이밍**: ALB는 ECS **서비스 생성 시에만** 연결할 수 있으므로, ALB를 먼저 생성한 후 ECS 리소스를 순서대로 만들어야 한다.
3. **Spring 환경 변수 주입**: 인터프리터 언어 기반 서버와 달리 Spring은 빌드 전에 환경 변수를 주입해야 한다. GitHub Actions에서 빌드 스텝 전에 `.env` 파일을 생성하는 과정이 필요하다.
4. **ECS Fargate 메모리 제약**: Spring Boot 서버의 경우 메모리를 너무 낮게 설정하면 Gradle 빌드 자체가 실패할 수 있다. 충분한 메모리를 할당해야 한다.
5. **Task Definition 관리**: `task-definition.json`을 리포지토리에 커밋하면 민감 정보가 노출될 수 있으므로, AWS CLI를 통해 런타임에 다운로드하는 방식이 안전하다.

### 비용 및 온프레미스 전환

Free Tier 범위 내에서 프론트엔드 배포 비용은 크게 발생하지 않았다. 하지만 ECS Fargate의 경우 CPU와 메모리 사용량에 따른 비용이 지속적으로 발생하며, 서비스 규모가 커질수록 비용 부담이 증가할 수 있다.

비용 문제로 인해 이후 온프레미스 Jenkins로 전환하게 되었는데, 이에 대해서는 [Jenkins CI/CD 파이프라인 구축]({{< relref "jenkins-cicd-pipeline" >}}) 글에서 다루고 있습니다.

---

## 마치며

AWS 기반의 프론트엔드와 백엔드 CI/CD 파이프라인을 전체적으로 구축하면서, 클라우드 아키텍처의 기본기를 확실하게 다질 수 있었다. CI/CD 파이프라인 구성은 단순히 코드 배포 주기를 단축하는 것을 넘어, 팀원들이 각자의 개발에 집중할 수 있는 환경을 만들어 준다는 점에서 프로젝트 전반에 걸쳐 큰 효과를 발휘했다.

특히 프로젝트나 해커톤과 같은 상황에서는 빠른 개발 주기가 경쟁력이 되기 때문에, 초기에 CI/CD를 잘 갖추어 놓는 것이 장기적으로 많은 시간을 절약해 준다. 다만 서비스가 성장함에 따라 비용과 운영 효율성을 지속적으로 검토하여, 상황에 맞는 인프라 전략을 선택하는 것이 중요하다.
