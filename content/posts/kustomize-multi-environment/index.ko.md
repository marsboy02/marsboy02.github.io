---
title: "Kustomize로 멀티 환경 배포 관리하기"
date: 2024-12-20
draft: false
tags: ["kubernetes", "kustomize", "devops", "gitops"]
translationKey: "kustomize-multi-environment"
summary: "선언적 관리라는 개념에서 출발해 Base/Overlay 구조와 패치 전략, 그리고 Helm과의 비교까지 — 환경별로 복제되는 YAML을 상속으로 정리하는 방법을 다룬다."
---

같은 애플리케이션을 dev, staging, prod에 배포하다 보면 YAML이 환경 수만큼 복제된다. 대부분의 내용은 동일하고 replicas 하나, 도메인 하나, ConfigMap 값 몇 개만 다른데, 그 몇 줄 때문에 파일 전체가 세 벌로 늘어난다. 한쪽만 고치고 다른 쪽을 빠뜨리는 사고는 대체로 이 지점에서 난다.

**Kustomize**는 이 문제를 상속으로 푸는 도구다. 쿠버네티스 오브젝트를 원하는 대로 변형할 수 있게 도와주는 선언형 설정 관리 도구이며, 공통 리소스를 base에 두고 환경별 차이만 overlay에 선언하는 구조가 핵심이다.

{{< linkcard url="https://kustomize.io/" title="Kustomize — Kubernetes native configuration management" desc="템플릿 없이 순수 YAML만으로 쿠버네티스 오브젝트를 조합하고 변형하는 공식 설정 관리 도구" image="images/kustomize-logo.png" >}}

Kustomize가 처음부터 kubectl의 일부였던 것은 아니다. 2018년 5월 구글이 SIG-CLI 서브프로젝트로 공개한 별도 CLI였고, 저장소도 `kubernetes-sigs/kustomize`로 따로 있었다. 정확히는 외부 벤더가 만든 서드파티라기보다 쿠버네티스 프로젝트 안에서 출발한 독립 도구였는데, 어느 쪽이든 쓰려면 바이너리를 따로 설치해야 했다는 점은 같다.

이것이 바뀐 시점이 **2019년 3월 kubectl v1.14**다. `kustomize build` 흐름이 kubectl에 통합되면서 `kubectl apply -k` 한 줄로 쓸 수 있게 되었고, 지금은 쿠버네티스 공식 문서가 선언형 관리 방법으로 Kustomize를 안내한다. 별도 설치 없이 바로 쓸 수 있다는 점이 Helm과 갈리는 첫 번째 지점이다.

다만 한 가지 알아둘 것이 있다. kubectl에 내장된 버전과 별도 바이너리의 버전이 오랫동안 어긋나 있었다.

| kubectl 버전  | 내장 kustomize 버전       |
| ------------- | ------------------------- |
| v1.14 ~ v1.20 | v2.0.3 (그대로 고정)      |
| v1.21 이후    | v4.0.5부터 주기적으로 갱신 |

kubectl v1.14에 들어간 것은 kustomize v2.0.3이었고, 이 버전이 kubectl v1.20까지 2년 가까이 고정되어 있었다. kubectl v1.21에 와서야 v4.0.5로 올라갔고 그 뒤로는 정기적으로 갱신되고 있다. 그래서 오래된 클러스터의 kubectl로 최신 문법을 쓰면 문서대로 동작하지 않을 수 있다. 뒤에서 다룰 `patches`나 `replicas` 필드가 대표적인데, 문서와 다르게 동작한다면 `kubectl version`과 `kustomize version`을 먼저 비교해보는 편이 빠르다.

이 글에서는 선언적 관리라는 개념에서 출발해 Base/Overlay 구조와 패치 전략, 실전 프로젝트 구성, 그리고 Helm과의 비교까지 다룬다.

---

## 1. 선언적(Declarative) vs 명령적(Imperative)

Kustomize를 이해하려면 쿠버네티스가 리소스를 다루는 두 가지 방식부터 구분해야 한다. 리소스 관리 방식은 크게 **명령적 방식**과 **선언적 방식**으로 나뉜다.

### 명령적(Imperative) 방식

명령적 방식은 "무엇을 어떻게 해라"고 하나하나 명령을 내리는 방식이다. 예를 들어 다음과 같이 개별 명령어로 리소스를 직접 조작한다.

```bash
kubectl create deployment nginx --image=nginx
kubectl scale deployment nginx --replicas=3
kubectl expose deployment nginx --port=80
```

간단한 작업에는 편리하지만, 환경이 복잡해질수록 어떤 명령이 실행되었는지 추적하기 어렵고 재현성이 떨어진다. 무엇보다 클러스터의 현재 상태를 설명하는 문서가 어디에도 남지 않는다.

### 선언적(Declarative) 방식

선언적 방식은 "최종 상태가 이것이어야 한다"를 YAML 파일로 기술하고, 쿠버네티스가 현재 상태와 비교하여 필요한 변경만 적용하는 방식이다.

```bash
kubectl apply -f deployment.yaml
```

선언적 방식의 장점은 다음과 같다.

- **멱등성**(Idempotency): 같은 명령을 여러 번 실행해도 결과가 동일하다.
- **버전 관리**: YAML 파일을 Git으로 관리할 수 있어 변경 이력 추적이 가능하다.
- **자동 복구**: 현재 클러스터 상태와 선언된 상태가 다르면 변경된 부분만 자동으로 반영된다.

실무에서 쿠버네티스를 다룰 때에는 대체로 명령적 방식보다 선언적 방식이 우세하다. 여럿이 함께 클러스터를 관리하는 상황에서는 동료가 무엇을 바꿨는지 알아야 하는데, 변경이 코드로 남아 있지 않으면 확인할 방법이 없기 때문이다.

Kustomize는 바로 이 선언적 방식을 더욱 강력하게 만들어주는 도구다. `kubectl apply -k` 명령으로 Kustomize가 조립한 리소스를 적용하면, 변경 사항이 없는 리소스는 `unchanged`로, 변경이 필요한 리소스만 `created` 또는 `configured`로 처리된다.

![kubectl apply -k 실행 결과](images/kubectl-apply-kustomize-output.png)

위 스크린샷처럼 이미 존재하는 리소스는 `unchanged`로 표시되고, 새로 추가된 리소스만 `created`로 생성된다. 명령을 몇 번 실행하든 결과가 같다는 것, 이것이 선언형 관리의 핵심이다. (이를 멱등하다고 부른다.)

---

## 2. Base/Overlay 구조

선언적 관리가 무엇인지 정리했으니, 이제 Kustomize가 그 위에서 무엇을 더 해주는지 볼 차례다.

### 왜 멀티 환경 관리가 필요한가

쿠버네티스로 서비스를 운영하다 보면 여러 스테이지에 서버를 배포해야 하는 상황이 생긴다. 조직의 규모와 제품의 특성에 따라 구성은 달라지지만, 일반적으로는 다음과 같이 단순화할 수 있다.

```
Dev → Staging → Prod
```

![Git 브랜치 컨벤션](images/git-branch-convention.png)

위 그림처럼 Git 브랜치 전략에 따라 배포 환경이 결정되는 경우가 많다. Master 브랜치는 Production, Release 브랜치는 Staging, Develop 브랜치는 Dev 환경에 매핑된다. 그리고 각 환경마다 URL, 데이터베이스 연결 정보, 컴퓨팅 리소스 할당 같은 설정이 미묘하게 달라진다.

이 "약간의 차이"를 어떻게 선언적으로 관리할 것인가. 이것이 Kustomize의 Base/Overlay 구조가 푸는 문제다.

### Base/Overlay 개념

Kustomize의 핵심 아이디어는 **상속**이다. 기본이 되는 리소스 정의를 `base` 디렉토리에 두고, 환경별 차이점만 `overlay` 디렉토리에 정의한다.

![Kustomize 디렉토리 구조](images/kustomize-directory-structure.png)

위와 같이 `base`에는 deployment.yaml, service.yaml 등 공통 리소스를 두고, `overlay` 아래 각 환경(prod, stage)에는 해당 환경에 특화된 configmap.yaml, ingress.yaml을 배치한다.

### Base의 kustomization.yaml

`base` 디렉토리의 `kustomization.yaml`은 공통 리소스들을 등록한다.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
```

### Overlay의 kustomization.yaml

각 환경의 `kustomization.yaml`에서는 base를 상속받고, 환경별 리소스를 추가한다.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base
  - configmap.yaml
  - ingress.server.yaml
```

`resources` 필드에 `../../base`를 지정하면 base의 모든 리소스를 상속받는다. 그 위에 해당 환경에만 필요한 ConfigMap이나 Ingress를 추가로 선언하는 식이다.

![Base/Overlay 상속 관계 다이어그램](images/base-overlay-inheritance-diagram.png)

정리하면 하나의 base를 여러 overlay가 공유하는 구조다. 새로운 환경이 필요하면 overlay 아래에 디렉토리를 하나 더 만들고 달라지는 파일 몇 개만 작성하면 된다. 환경이 늘어나도 공통 부분은 한 곳에만 존재한다.

환경별로 갈리는 것은 대개 정해져 있다. prod와 stage가 서로 다른 환경 변수를 쓰기 때문에 ConfigMap과 Secret이 overlay로 분리되고, Ingress는 `api.example.com`과 `api.dev.example.com`처럼 서브도메인을 다르게 할당하기 위해 분리된다.

추가로, base에 있는 deployment에 replicas를 2로 정의해두었으나, overlays한 다른 환경(예를 들어 test)에서는 한 개의 Pod만 있어도 된다면, 상속받은 kustomization.yaml에서 replicas를 변경할 수 있다.

이때 별도의 패치 파일을 만들 필요도 없다. Kustomize는 자주 쓰이는 변형을 `kustomization.yaml`의 필드로 내장하고 있어서, 상속 선언 바로 옆에 몇 줄 적는 것으로 끝난다.

```yaml
# overlays/test/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base
  - configmap.yaml
  - ingress.yaml

namespace: test
namePrefix: test-

replicas:
  - name: nginx-deployment
    count: 1

images:
  - name: nginx
    newTag: 1.25-alpine
```

각 필드가 하는 일은 다음과 같다.

- `namespace`: 이 overlay가 만들어내는 모든 리소스를 `test` 네임스페이스로 보낸다.
- `namePrefix`: 모든 리소스 이름 앞에 `test-`를 붙인다. 이름만 바꾸는 것이 아니라 그 이름을 참조하는 곳(Deployment가 바라보는 ConfigMap 이름 등)까지 함께 갱신해준다.
- `replicas`: base의 `nginx-deployment`를 찾아 레플리카 수만 1로 바꾼다. base의 나머지 정의는 그대로다.
- `images`: 컨테이너 이미지의 태그를 교체한다.

base는 한 글자도 건드리지 않았고, overlay에도 새 YAML 파일이 늘어나지 않았다는 점이 핵심이다. 이 정도의 변형은 전부 `kustomization.yaml` 안에서 끝나고, 여기서 표현할 수 없는 것들만 다음에 볼 패치로 넘어간다.

### 패치 전략

리소스를 새로 추가하는 것이 아니라 base에 이미 있는 값을 바꿔야 할 때는 패치를 사용한다. Kustomize는 두 가지 패치 전략을 제공한다.

#### Strategic Merge Patch

기존 리소스의 특정 필드만 오버라이드하는 방식이다. 쿠버네티스 오브젝트의 구조를 이해하고 병합하기 때문에, 바꾸고 싶은 필드만 적어두면 나머지는 base의 값이 그대로 유지된다.

![Kustomize의 Base와 Patch 적용 과정](images/kustomize-base-overlay-patch.png)

위 그림처럼 base에 정의된 Deployment의 replicas를 환경별로 다르게 지정할 수 있다. Staging에서는 `replicas: 2`, Production에서는 `replicas: 6`처럼, 동일한 base를 두고 차이점만 패치 파일에 기술한다.

```yaml
# overlay/prod/increase-replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 6
```

이 패치는 base의 Deployment에서 `replicas` 값만 6으로 변경하고, 나머지 필드는 건드리지 않는다. 어떤 리소스를 패치할지는 `metadata.name`과 `kind`로 식별한다.

#### JSON Patch

보다 세밀한 제어가 필요할 때 사용한다. JSON Patch(RFC 6902) 형식으로 특정 경로의 값을 추가, 삭제, 교체할 수 있다. 배열의 특정 인덱스를 다루거나 필드를 삭제하는 것처럼 Strategic Merge Patch로 표현하기 어려운 작업에 적합하다.

```yaml
# overlay/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

patches:
  - target:
      kind: Deployment
      name: nginx-deployment
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 6
      - op: add
        path: /metadata/labels/environment
        value: production
```

---

## 3. 실전 프로젝트 예시

개념을 봤으니 실제로 디렉토리를 만들어 배포해본다.

### 시나리오 구성

nginx 이미지를 Deployment로 배포하는 시나리오를 구성해보자. 환경별 차이점은 다음과 같다.

| 설정 항목    | Staging                 | Production             |
| ------------ | ----------------------- | ---------------------- |
| 서브도메인   | `stage.localhost`       | `prod.localhost`       |
| ConfigMap    | `this-is-stage-overlay` | `this-is-prod-overlay` |
| Ingress 이름 | `nginx-stage-ingress`   | `nginx-prod-ingress`   |

### 실행 방법

Kustomize로 구성된 환경을 배포하려면, 해당 환경의 kustomization.yaml이 있는 디렉토리에서 다음 명령을 실행한다.

```bash
kubectl apply -k .
```

배포 전에 어떤 리소스가 생성될지 미리 확인하려면 `kubectl kustomize`를 사용한다. 실제로 클러스터에 적용하지 않고 조립 결과만 출력해주기 때문에, overlay를 수정할 때마다 이 명령으로 확인하는 습관을 들이는 편이 좋다.

```bash
kubectl kustomize .
```

위 명령의 출력 결과는 다음과 같다.

```yaml
apiVersion: v1
data:
  key: this-is-prod-overlay
kind: ConfigMap
metadata:
  name: nginx-prod
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx-deployment
  template:
    metadata:
      labels:
        app: nginx-deployment
    spec:
      containers:
        - image: nginx
          name: nginx
          ports:
            - containerPort: 80
          resources:
            limits:
              cpu: 500m
              memory: 128Mi
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  labels:
    name: nginx
  name: nginx-prod-ingress
spec:
  rules:
    - host: prod.localhost
      http:
        paths:
          - backend:
              service:
                name: nginx-nginx
                port:
                  number: 80
            path: /
            pathType: Prefix
```

base에서 정의한 Deployment와 Service는 그대로 유지되면서, prod overlay에서 추가한 ConfigMap과 Ingress가 함께 출력되는 것을 확인할 수 있다.

### 자주 쓰는 패턴

실무에서 반복적으로 사용하게 되는 패턴들을 정리하면 다음과 같다.

**공통 레이블 추가**

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

labels:
  - pairs:
      app: my-app
      team: platform
    includeSelectors: false

resources:
  - deployment.yaml
  - service.yaml
```

예전 문서에서 흔히 보이는 `commonLabels`는 Kustomize v5에서 deprecated 되었다. `commonLabels`는 레이블을 Deployment의 셀렉터에까지 자동으로 밀어 넣어서, 이미 배포된 워크로드에 적용하면 셀렉터가 불변 필드라는 이유로 배포가 실패하는 문제가 있었다. 현재는 위와 같이 `labels` 필드를 쓰고, 셀렉터까지 바꿀지 여부를 `includeSelectors`로 명시한다.

**네임스페이스 일괄 지정**

```yaml
# overlay/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
  - ../../base
```

**이미지 태그 오버라이드**

```yaml
# overlay/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

images:
  - name: nginx
    newTag: 1.25-alpine
```

이미지 태그 오버라이드는 CI 파이프라인이 건드리는 지점이기도 하다. 빌드가 끝난 뒤 `kustomize edit set image`로 이 필드만 갱신해 커밋하면, 배포는 그 커밋을 따라간다.

### 안티패턴

반대로 피해야 할 패턴도 있다.

- **base에 환경별 설정 포함**: base에는 모든 환경에 공통으로 적용되는 리소스만 두어야 한다. 특정 환경에만 필요한 설정이 base로 올라가면 overlay의 의미가 퇴색된다.
- **과도한 패치 중첩**: 패치가 3단계 이상 중첩되면 최종 결과를 예측하기 어려워진다. 가능하면 base → overlay 2단계로 유지하는 것이 좋다.
- **overlay 간 리소스 공유**: 각 overlay는 독립적이어야 한다. overlay끼리 리소스를 참조하기 시작하면 의존성이 생겨 관리가 복잡해진다.

세 가지 모두 "어디를 봐야 이 환경의 최종 상태를 알 수 있는가"라는 질문에 대한 답을 흐린다는 공통점이 있다.

---

## 4. Kustomize vs Helm

Kustomize를 이야기하면 반드시 따라오는 질문이 Helm과의 비교다. 둘은 목적이 겹치는 듯 보이지만 접근 방식이 다르다.

### Helm Chart의 등장과 한계

**Helm**은 쿠버네티스의 패키지 매니저 역할을 하는 도구로, `apt`나 `pip`처럼 사전에 구성된 프리셋(Chart)을 가져와 애플리케이션을 배포할 수 있게 해준다.

다만 Kustomize가 kubectl에 통합되기 전, Helm v2 시절에는 멀티 스테이지 배포가 까다로웠다. **Tiller**라는 서버 컴포넌트를 클러스터 안에 띄워야 했고, Chart와 Release, Revision 같은 Helm 고유의 개념을 먼저 이해해야 했다.

이런 배경에서 Kustomize가 등장했다. 순수 YAML만으로 동작하고 별도의 서버 컴포넌트 없이 설정을 관리할 수 있었기 때문에 빠르게 자리를 잡았고, 결국 쿠버네티스에 공식 통합되었다.

### 비교표

| 기준            | Kustomize          | Helm                  |
| --------------- | ------------------ | --------------------- |
| 접근 방식       | 순수 YAML 오버레이 | Go 템플릿 기반        |
| 러닝 커브       | 낮음               | 중간~높음             |
| 패키지 배포     | 미지원             | Chart 레포지토리 지원 |
| 쿠버네티스 통합 | kubectl 내장       | 별도 CLI 설치 필요    |
| 복잡한 로직     | 제한적 (패치 기반) | 조건문, 반복문 지원   |
| 템플릿 재사용   | base/overlay 상속  | 차트 의존성 관리      |
| 롤백            | Git 기반 수동 롤백 | `helm rollback` 내장  |

### 언제 어떤 도구를 선택할 것인가

**Kustomize가 적합한 경우**

- 팀 내부 애플리케이션의 환경별 설정 관리
- 단순한 설정 차이만 있는 멀티 스테이지 배포
- GitOps 워크플로우에서 선언적 관리가 필요한 경우
- 순수 YAML을 유지하고 싶은 경우

**Helm이 적합한 경우**

- 외부 배포용 패키지 작성 (오픈소스 프로젝트 등)
- 복잡한 조건 분기가 필요한 설정 관리
- Chart 레포지토리를 통한 버전 관리가 필요한 경우
- 롤백 기능이 자주 필요한 경우

### 함께 사용할 수 있는가

두 도구는 상호 배타적이지 않다. 실무에서는 Helm으로 서드파티 애플리케이션(Prometheus, Nginx Ingress Controller 등)을 설치하고, Kustomize로 팀 내부 애플리케이션의 환경별 설정을 관리하는 조합이 흔하다. ArgoCD 같은 GitOps 도구는 Helm Chart와 Kustomize를 모두 지원하므로, 리소스의 성격에 따라 나눠 쓰면 된다.

---

## 5. GitOps와의 시너지

여기까지가 Kustomize 자체의 이야기라면, 이 도구가 실제로 힘을 발휘하는 자리는 GitOps다.

[GitOps]({{< ref "/posts/about-gitops" >}})는 Git을 **단일 진실 공급원**(Single Source of Truth)으로 삼아 인프라를 관리하는 방법론이다. 모든 인프라 설정이 Git 레포지토리에 선언적으로 기술되면, 변경 이력 추적과 롤백이 Git을 통해 자연스럽게 이루어진다. 이를 실천하려면 전체 시스템이 선언적으로 기술되어야 하는데, Kustomize가 바로 그 역할을 맡는다.

![Kustomize 공식 홈페이지 주요 특징](images/kustomize-overview-features.png)

대표적인 GitOps 도구인 **ArgoCD**는 Git 저장소를 주기적으로 감시하면서 변동 사항을 감지하고, 변경된 내용을 자동으로 클러스터에 반영한다. Webhook을 통한 트리거 방식도 지원한다.

![ArgoCD 아키텍처](images/argocd-architecture.webp)

Kustomize는 별도의 서버 컴포넌트가 없는 독립형 구조이기 때문에 ArgoCD, FluxCD 같은 도구와 매끄럽게 연동된다. 결과적으로 커밋 하나로 인프라 변경이 반영되는 파이프라인이 만들어진다. 다만 이 구조에서는 모든 설정이 Git에 평문으로 올라간다는 점이 문제가 되는데, 비밀번호나 토큰을 어떻게 다룰지는 [GitOps에서 Secret을 다루는 방법]({{< ref "/posts/gitops-secret-management" >}})에서 따로 정리했다.

---

## 마치며

Kustomize는 쿠버네티스의 선언적 관리 철학을 그대로 따르는 도구다. Base/Overlay 구조로 환경별 차이를 정리할 수 있고, kubectl에 내장되어 있어 별도 설치 없이 바로 쓸 수 있다.

직접 써보면서 가장 크게 느낀 것은, Kustomize의 진짜 가치가 YAML 중복을 줄이는 데 있지 않다는 점이다. 중복을 줄이는 것은 결과일 뿐이고, 핵심은 **환경 사이의 차이가 파일 하나로 드러난다**는 데 있다. overlay 디렉토리만 열어보면 이 환경이 다른 환경과 무엇이 다른지 한눈에 읽힌다. 세 벌로 복제된 YAML을 두고 "prod에만 있는 설정이 뭐였지"를 확인하려면 파일 전체를 비교해야 했던 것과 비교하면, 이쪽이 훨씬 빠르게 답을 준다. 반대로 overlay가 비대해지고 있다면 그것은 환경 구성 자체를 다시 봐야 한다는 신호이기도 하다.

다음으로는 ArgoCD의 ApplicationSet처럼, 환경이 더 늘어났을 때 overlay를 자동으로 찍어내는 방식을 정리해보려고 한다. 환경이 서너 개일 때는 디렉토리를 손으로 복사해도 괜찮지만, 고객사별로 클러스터가 나뉘는 상황에서는 그 복사 자체가 다시 중복이 되기 때문이다.

---

### References

- [Kustomize 공식 홈페이지](https://kustomize.io/)
- [Kubernetes Blog — Introducing kustomize (2018-05-29)](https://kubernetes.io/blog/2018/05/29/introducing-kustomize-template-free-configuration-customization-for-kubernetes/)
- [kubernetes-sigs/kustomize — kubectl 내장 버전 대응표](https://github.com/kubernetes-sigs/kustomize)
- [Kubernetes 공식 문서 — Kustomize를 이용한 선언형 관리](https://kubernetes.io/ko/docs/tasks/manage-kubernetes-objects/kustomization/)
- [SIG CLI — Labels and Annotations](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/labels/)
- [ArgoCD 공식 문서](https://argo-cd.readthedocs.io/)
