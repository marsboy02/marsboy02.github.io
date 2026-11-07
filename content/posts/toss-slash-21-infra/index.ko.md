---
title: "토스 SLASH 21 — 서버 기술, 쿠버네티스 운영, 인프라 모니터링, SRE 사례"
date: 2026-11-05
draft: false
tags: ["sre", "kubernetes", "monitoring", "toss", "meetup-notes"]
translationKey: "toss-slash-21-infra"
summary: "토스 SLASH 21 발표 중 인프라와 관련된 네 개의 발표를 보고 남기는 기록이다."
---

[이전 글]({{< ref "/posts/daangn-sre-meetup-1-eks-grpc" >}})에서는 당근 SRE 밋업 1회를 정리했다. 이번에는 2021년에 열린 토스의 개발자 컨퍼런스 **SLASH 21**에서 인프라와 관련된 네 개의 발표를 골라 정리한다.

---

## 1. 토스 서비스를 구성하는 서버 기술

이항령 님 (토스 Server Platform Team Leader)

{{< youtube YBXFRSAXScs >}}

가장 먼저 볼 발표는 토스 서비스를 구성하는 서버 기술이다. 사실 이 영상은 당근/토스에서 나온 컨퍼런스 영상을 처음부터 보게 된 이유이기도 하다. 24년도 토스 발표 중간에 21년도 영상을 참고하면 도움이 된다는 언급이 있어서 해당 영상을 찾아보게 되었고, 이참에 21년도 발표부터 정리하면 좋겠다고 생각했다. 정말 토스 인프라 기술의 뼈대를 여실히 보여주는 발표 영상이다.

![토스 서비스를 구성하는 서버 기술 스택 — 데이터 센터 2개(Active-Active) 위의 K8s(Calico) + Istio + Ceph를 중심으로 Spring 기반 애플리케이션, ArgoCD·Vault 등의 배포 도구, Grafana·Kibana·Thanos 등의 모니터링 도구, AWS 서비스가 배치되어 있다](images/toss-server-tech-stack.png)

토스는 이렇게 K8s와 Istio 기반으로 서비스를 운영한다. 당근과 비교하면 특이한 점은 Java나 Kotlin을 사용하는 비중이 압도적으로 많아 보인다는 것이다. 대부분의 서비스를 Spring Boot로 개발하고 있으며, 토스 내에서는 일관성을 중요하게 생각하는 듯하다.

![DC/OS에서 K8s로의 마이그레이션 — vamp service discovery의 비효율과 한계 때문에, 데이터 센터별로 DC/OS와 K8s를 동시에 운영하면서 하나씩 전환했다](images/dcos-to-k8s-migration.png)

토스는 원래 DC/OS라는 기술을 사용하고 있었고, vamp라는 도구를 통해서 온프레미스 데이터 센터의 서비스를 제어하고 있었다고 한다. 다만, 19년도에 vamp의 service discovery 비효율과 한계를 느끼고 서서히 마이그레이션을 진행하였으며, 데이터 센터 2개를 관리하고 있기 때문에 위 그림처럼 하나씩 전환했다고 한다. 이를 진행하면서 Istio + K8s 기반의 인프라를 갖추게 되었다.

한 가지 토스만의 특이한 점이 있다. Istio를 도입하면 보통 retry나 서킷 브레이커 등을 Istio로 구현하는 것을 떠올리기 쉬우나, 세밀한 제어가 어렵기 때문에 발표일 기준으로는 애플리케이션에서 처리한다고 한다. 트랜잭션의 성격에 따라서 retry를 다르게 줘야 하는 로직이 있기 때문에 Istio로 전부 제어하기는 어려웠고, 대신 Istio의 mTLS 등으로 A 서버에서 B 서버로 요청할 수 없도록 접근 제어를 한다고 한다. 그 외에 canary 배포와 같은 기능은 적극적으로 사용한다고 한다.

![api-gateway 슬라이드 — 늘어나는 Client 종류, Gateway 관리 주체, 여러 Gateway의 공통 로직 처리 문제를 짚고, 공통 로직은 모듈화해서 각 Gateway에 넣는다](images/api-gateway-shared-logic.png)

다음으로는 api-gateway에 대한 설명이다. 예전에 넷플릭스처럼 큰 MSA를 운영하는 서비스는 인증을 위해서 큰 api-gateway를 둔다는 글을 읽은 적이 있다. passport라는 이름이었는데, 관련 레퍼런스를 읽고서 api-gateway 방식이 꽤 필요하겠다고 느꼈다. 토스도 마찬가지로 Spring Cloud Gateway를 사용한다고 한다. 이 부분 또한 알고는 있었지만 실제로 어떤 식으로 사용하는지는 자세하게 알아두지 못해서 한 번 공부해 봐야겠다고 느꼈다.

바로 이어서는 WebFlux와 coroutine 등의 이야기가 나온다. 예전에 Spring으로 개발을 해본 적이 있어서 WebFlux에 대해서는 대강 알고 있었지만, coroutine은 그냥 그런 게 있구나 하고 넘어갔었다. 최근에 큰 기업에서 Spring 기반으로 개발하는 백엔드 엔지니어분과 이야기했을 때는 virtual thread도 꽤 좋다는 이야기를 들었는데, 이렇게 Spring 안에서도 퍼포먼스를 위해 할 수 있는 시도들이 많다고 느꼈다.

![로그 모니터링 구조 — 데이터 센터마다 container → kafka → ELK로 로그를 흘려보내고, 장기 보관용 로그는 kafka에서 hadoop(DW)으로 보낸다. APM은 Pinpoint를 사용한다](images/logging-pipeline-kafka-elk.png)

로그의 경우에는 Kafka를 사용하며, ELK 스택을 통해서 확인한다고 한다. 장기 보관이 필요한 로그 데이터는 Hadoop으로 쏘는 파이프라인을 사용한다고 한다.

![메트릭 구조 — 데이터 센터마다 thanos + ceph + grafana를 두고, global thanos가 두 데이터 센터의 thanos를 묶어 이중화한다](images/metric-thanos-ceph.png)

메트릭은 주로 Prometheus를 하드하게 사용하는 것으로 보이며, 여러 Prometheus를 모아서 관리할 수 있게 해주는 Thanos도 많이 사용하는 것 같다. 예전에 토스의 서류 과제에서도 Prometheus 기반으로 엄청나게 많은 트래픽을 견뎌야 하는 상황을 어떻게 처리할 것인지와 같은 내용이 나왔던 것 같고, 이후의 발표에서도 Prometheus 성능 최적화와 관련된 내용이 꽤 나온다.

이후에는 Kafka나 Redis를 클러스터 형태로 어떻게 관리하는지에 대한 내용과 함께 발표를 마친다.

---

## 2. 실수 없이 안전하게 쿠버네티스 운영하기

김형록 님 (토스 DevOps Engineer)

{{< youtube gF1wfTCDyI8 >}}

이번 발표는 앞 발표에 이어서 보면 좋을 내용으로, 쿠버네티스 클러스터를 어떻게 관리하는지 소개한다. ArgoCD로 GitOps를 어떻게 운영하는지와 OPA(Open Policy Agent)로 실수를 막는 정책을 어떻게 적용하는지를 다룬다.

![DevOps/SRE가 관리하는 K8s 클러스터 — Testbed → Dev → Live(부분) → Live(전체) 순서로 변경 사항을 적용한다](images/k8s-cluster-promotion.png)

우선 기본적으로는 위와 같이 클러스터를 구성해서, Testbed와 Dev 클러스터에서 확인하고 나서 Live로 배포하는 구조를 갖고 있다고 한다. 이 구조에서 ArgoCD로 GitOps를 운영하는데, 한 가지 특이한 점은 Auto Sync를 사용하지 않는다는 것이다. 안정성 때문인 것 같은데, 최대한 빠르게 검토하고 나서 sync를 진행하는 것 같다. 그래서 토스에서는 GitOps를 git의 상태와 쿠버네티스의 상태가 동일한 것이 아니라, git의 상태 = 바라는 쿠버네티스 상태로 정의한다고 한다. 개인적으로 쿠버네티스나 테라폼을 관리할 때에도 auto-approve를 할 만한 지점과 아닌 지점을 의도적으로 잘 나눠보는 것이 중요할 것 같다.

![App-of-Apps 구조 — ArgoCD Charts Repository 아래의 App-of-Apps 차트가 Istio, Prometheus, Vault 각각을 가리키는 ArgoCD Application 매니페스트를 템플릿으로 갖는다](images/argocd-app-of-apps.png)

토스에서는 Helm chart를 통해서 서비스들을 전개하는데, Istio 같은 것들을 Helm chart로 벤더링해서 app-of-apps 방식으로 사용한다고 한다. 앞서 이야기한 GitOps 원칙을 지키는 방식으로, ArgoCD에 애플리케이션을 띄우는 것마저도 git에 설정 파일로 '선언형'으로 관리하는 방식이다.

![K8s 시크릿 관리 — 토스가 자체 정의한 VaultSecret 리소스(kv, path 지정)를 만들면 이를 소유자로 하는 쿠버네티스 Secret이 생성된다](images/vault-secret-crd.png)

GitOps를 운영하면 누구나 시크릿을 어떻게 관리할 것인지 고민하게 될 텐데, 일반적으로 Vault나 AWS Secrets Manager를 사용하는 것 같다. AWS Secrets Manager는 시크릿 하나당 월 0.4달러나 하기 때문에 수많은 시크릿을 환경별로 저장한다면 매달 꽤 많은 비용이 나오게 된다. 이런 이유로 일반적으로 Vault를 많이 사용하는 것 같다.

개인적으로는 ESO(External Secrets Operator)를 Vault와 함께 사용하고 있는데, 토스는 신기하게도 자체 제작한 VaultSecret이라는 API를 활용하는 것으로 보인다. 어떤 이유로 자체 제작한 툴을 사용해서 시크릿을 가져오는지 궁금하다.

![Open Policy Agent 구조 — Service가 요청이나 이벤트를 받으면 OPA에 JSON으로 질의하고, OPA는 Rego로 작성된 Policy와 JSON Data를 바탕으로 결정을 돌려준다(Policy Decoupling)](images/open-policy-agent.png)

다음으로는 OPA(Open Policy Agent)다. 휴먼 에러를 방지하기 위해 정책을 적용하여 실수를 막을 수 있다고 한다. 쿠버네티스에서는 Admission Controller가 허가되지 않은 요청을 막는데, OPA라는 툴을 통해서 이러한 정책을 유연하게 설정할 수 있다고 한다.

![OPA 도입 결과 — 장점은 설정상의 Known Issue 재발 방지와, RBAC으로 개발자 권한을 제한해 생산성을 낮추지 않고도 문제가 될 수 있는 특정 부분만 컨트롤할 수 있다는 점이다. 단점은 발생할 수 있는 휴먼 에러에 비해 Admission Controller 방식으로 막을 수 있는 범위가 제한적이라는 점이다](images/opa-pros-and-cons.png)

OPA에 대한 설명을 보면서 딱 느꼈던 점은 쿠버네티스 생태계에 있는 RBAC 같다는 생각이었다. AWS를 쓰든, API 서버를 구축하든 요청자의 Role에 따라서 허가하거나 반려하는 컨트롤 센터가 존재하게 되는데, 쿠버네티스에서는 OPA가 딱 그 역할을 수행하는 것 같다. Rego라는 자체 언어로 간단하게 스크립트를 만들어서 정책을 설정할 수 있는데, 정말 많은 엔지니어가 쿠버네티스 기반의 클러스터를 관리하게 된다면 이러한 접근 제어가 보안 측면에서도, 휴먼 에러 측면에서도 좋아 보인다.

---

## 3. 토스의 서버 인프라 모니터링

이재성 님 (토스 DevOps Engineer)

{{< youtube rxurfKT2lD8 >}}

앞의 발표들과 흐름이 꽤 잘 이어진다고 느꼈다. 토스의 전반적인 인프라 기술을 소개하고, 쿠버네티스를 어떻게 관리하는지 다룬 데 이어, 이번에는 토스에서 모니터링을 어떻게 하는지 알려준다. 영상 초반에 2019년과 2021년 사이에 많은 변화가 있었다며, 약 9개월 동안 DC/OS + Vamp 조합에서 K8s + Istio 조합으로 IDC 전체를 전환하는 데 성공했다고 한다.

![Application Layer, Network Layer, OS Layer 세 가지 레이어로 메트릭을 나눈 슬라이드](images/monitoring-three-layers.png)

토스에서 어떻게 모니터링하는지는 크게 3가지 레이어의 지표로 나누어 살펴본다. 네트워크 레이어에서는 주로 Istio 서비스 메시 덕분에 네트워크를 파악할 수 있다는 이점을 살려 모니터링하고 있으며, 다음과 같은 구조로 2개의 IDC에 걸쳐 auto failover를 구성했다고 한다.

![Network Layer Metric — User 요청이 GSLB를 거쳐 데이터 센터별 L7 → L7 → Istio → App으로 흐르고, Route 53 Health Check로 한쪽 데이터 센터에 장애가 나면 다른 쪽으로 failover한다](images/network-layer-gslb-failover.png)

개인적으로는 GSLB 뒷단에 L7을 2단으로 두고 있는데, 정확히 어떤 목적으로 나누었는지 파악하기 어려워서 꽤 고민했다. 앞단의 L7은 좀 더 넓은 수준에서 말 그대로 라우팅을 해주고, 뒷단의 L7은 로드밸런싱을 하는 것이 아닐까 싶다.

![OS Layer Metric — Brendan Gregg의 USE method 플로우차트로, 리소스마다 Errors, Utilization, Saturation을 차례로 확인하며 문제를 배제해 나간다](images/os-layer-use-method.png)

다음은 OS 레이어의 모니터링과 트러블슈팅이다. OS 레이어는 CPU나 메모리 사용률 등을 보고 어디서 문제가 발생했는지 확인하는 것인데, 아무래도 경험에 의지하는 경향이 크다고 한다. 개별 엔지니어의 역량에 기대게 되기 때문에, OS 레이어에서 문제가 발생하면 브렌던 그레그(Brendan Gregg)가 만든 플로우차트 같은 것을 따라 순차적으로 하나씩 점검하는 ritual이 있다고 한다.

구체적으로는 CPU 사용률은 `container_cpu_usage_seconds_total`로, CPU saturation은 `container_cpu_cfs_throttled_periods_total`과 `container_cpu_cfs_periods_total`로 확인하는 식이다. 참고로 토스는 프로메테우스를 하드하게 쓰고 있기 때문에 프로메테우스 지표를 많이 살펴볼 것으로 생각된다.

![수집하는 메트릭량을 줄이자 — hashmod로 타겟을 샤딩해 여러 Prometheus가 나눠 수집하고, thanos query로 통합 쿼리한다](images/prometheus-hashmod-thanos-query.png)

마지막은 모니터링을 하는 주체, 즉 모니터링 시스템 자체의 모니터링이다. 토스에서는 주로 Prometheus로 메트릭을 수집하고 모니터링하는데, 그 Prometheus가 내려갈 수 있는 상황에 대비하는 내용을 다룬다. 실제로 out-of-memory가 많이 발생해서, Prometheus를 병렬로 두고 hashmod로 타겟을 나누어 수집한다고 한다. 그리고 Thanos로 여러 Prometheus를 하나로 묶어 관리하며, 장기 스토리지로 보내 메트릭을 오래 보관할 수도 있다고 한다.

---

## 4. SRE 사례 소개

조규희 님 (토스 Site Reliability Engineer)

{{< youtube Tu0MMBgdyTw >}}

마지막 발표는 토스의 SRE 사례 소개다. 이 발표에서는 Redis Cluster Rebalancing, Memcached Redistribute, Prometheus Scraping 순서로 3가지 사례를 다룬다. 실제로 문제가 발생한 사례를 들어 어떻게 해결해 나갔는지 상세하게 알려준다는 점이 특징이다. 참고로 토스에서는 SRE 팀이 애플리케이션에 가까운 인프라를 다루고, DevOps 팀이 쿠버네티스 쪽 인프라를 다루는 것으로 알고 있다. 쿠버네티스나 Istio 설정은 DevOps가 전담하고, Spring에 커스텀 메트릭을 심는 것 같은 일은 SRE가 담당하는 식이다.

![source부터 set slot을 진행할 경우의 시퀀스 — 슬롯 상태를 IMPORTING, MIGRATING으로 바꾸고 key를 migrate하는 사이, client는 source와 destination 사이를 redirect하다가 max redirect에 도달하면 에러가 발생한다](images/redis-cluster-set-slot-redirect.png)

첫 번째는 Redis Cluster를 증설하면서 리밸런싱 명령어를 수행하는 과정에서 발생한 문제다. 타임아웃이 한 건 났을 때는 사이즈가 꽤 큰 Redis 컬렉션을 옮기려다 타임아웃이 났다는 것을 쉽게 확인할 수 있었지만, 다른 상황에서는 ASK redirect가 꽤 발생해서 그 원인을 추적해야 했다. 결과적으로는 Spring에서 Redis를 쉽게 다룰 수 있게 해주는 Lettuce의 버그였는데, 이를 찾아내는 과정이 정말 클래식한 엔지니어링이라고 느꼈다.

위 사진처럼 Redis의 흐름을 하나씩 설명하면서 가설을 세우고 검증해 나가는 과정이 꽤 인상 깊었다. 이 글을 쓰는 지금은 AI가 꽤 많은 트러블슈팅을 대신해 주고 앞으로도 더 발전하겠지만, 이렇게 플로우차트를 그려 가며 기술적으로 검증하는 것이야말로 엔지니어에게 근본적으로 필요한 능력이라고 생각한다. 특히 Claude Code 같은 터미널 기반 코딩 에이전트를 쓰기 어려운 베어메탈 환경에서 네트워크 툴로 트러블슈팅해야 할 때는 전반적인 흐름을 잘 알고 있어야 해서, DevOps에게도 꽤 필요한 능력이라고 생각한다.

![Memcached 노드에 문제가 생길 경우 요약 — Timeout threshold만큼 실패하면 노드를 비활성화하고 operation을 redistribute하지만, 1/2^7(약 0.78%) 확률로 실패한 노드로 요청이 간다](images/memcached-redistribute-summary.png)

두 번째는 Memcached다. 토스에서는 Memcached에 failover를 적용해서 노드에 문제가 생기면(발표에서는 hang 상태에 빠졌다고 한다) 다른 노드로 요청을 옮겨 내결함성을 갖추는데, 정말 간헐적으로 실패하는 상황이 있었다고 한다. 찾아보니 코드에 2의 7승 분의 1 확률(약 0.78%)로 실패한다는 주석이 남겨져 있었다고 한다. 발표에서도 왜 그런지는 알 수 없었다고 했고, iptables 같은 네트워크 정책으로 dead node로 요청이 가지 않도록 설정했다고 한다.

![지금까지 내용 정리 — 1월 17일부터 한쪽 데이터 센터에서 해당 애플리케이션에 Prometheus 메트릭 수집을 시작했다](images/prometheus-scrape-jan17-summary.png)

세 번째는 Prometheus 쪽에서 발생한 문제다. 처음에는 Memcached에서 타임아웃이 발생하는 문제로 보였다. 원래 1ms 수준으로 응답하고 타임아웃은 500ms로 설정되어 있었는데, 타임아웃이 간헐적으로 발생해서 확인해 보니 Prometheus 쪽에 원인이 있었다.

데이터 센터에 따라 한쪽으로 가비지 컬렉션이 치우치는 skew 현상이 있었고, 확인해 보니 1월 17일부터 해당 문제가 발생하고 있었다고 한다. 발표에서도 특정 날짜를 기준으로 문제를 분석해 보는 습관이 중요하다고 설명하는데, 바꿔 말하면 진작 해당 지표를 모니터링하고 있었다면 무언가 배포가 나간 직후에 바로 알아차릴 수 있지 않았을까 하는 생각이 들었다. 요즘은 서비스 배포가 워낙 잦고 많다 보니 배포 직후에 대시보드를 꼼꼼히 살펴보는 일이 많지 않은데, 습관적으로 살펴보는 것도 필요하다고 생각한다.

원인은 Humongous Allocation이었다. 힙 리전보다 큰 객체를 할당하면 GC가 예상대로 동작하지 않는 현상이다. 해결 방안으로는 메트릭 개수나 사이즈를 제한하거나, Heap Region Size를 늘리거나, Stream 방식으로 Response를 전달하는 방법이 있다.

개인적으로 이런 문제를 마주했다면 '메트릭 사이즈를 조금 줄여볼까?' 하는 생각을 먼저 했을 것 같다. 그렇게 해서 문제가 사라졌다면 근본 원인은 해결하지 못한 채 오히려 덮어버리는 상황이 생길 수 있다. 그래서 OS 레벨, 혹은 코드(더 나아가 오픈소스) 레벨에서 근본적으로 문제를 해결하는 것이 필요하겠다는 생각이 들었다.

---

## 마치며

토스 SLASH 21은 SRE 발표까지 포함되어 있어서 좀 더 근본적인 엔지니어링이 보이는 느낌이었다. DevOps 엔지니어링을 하다 보면 많은 툴을 넓고 얕게 다루다 보니 공식 문서를 보고 설정을 on/off하는 방식이 일반적이고, 코드를 깊게 들여다봐야 하는 경우는 많지 않았다. 정말 엔지니어링적으로 뭔가를 해야 할 때는 대부분 네트워크 트러블슈팅이었고, 직접 Pod에 exec로 붙어서 확인하거나 netshoot 등을 사이드카로 띄우는 정도였던 것 같다. 이마저도 최근에는 AI 에이전트가 발전하면서 많이 편해졌다고 느낀다.

나름대로 네트워크 공부를 꾸준히 하고 있고 좋아하기도 해서, 문제가 생겼을 때 감으로 꽤 해결하는 편이다. 하지만 지금보다 더 복잡하고 더 다양한 기술을 쓰는 환경이라면 꽤 힘들 수도 있겠다는 생각이 든다. tcpdump나 Wireshark 등으로 정말 근본적으로 하나씩 들여다봐야 한다면 아직은 자신 있게 하기 어려운데, 쿠버네티스와 Istio 기반의 서비스를 제대로 운영하려면 기본적으로 갖춰두면 매우 좋을 역량이라고 생각하게 되었다. 더 열심히 해야겠다.

### References

- [토스 서비스를 구성하는 서버 기술 — 토스 SLASH 21 (YouTube)](https://www.youtube.com/watch?v=YBXFRSAXScs)
- [실수 없이 안전하게 쿠버네티스 운영하기 — 토스 SLASH 21 (YouTube)](https://www.youtube.com/watch?v=gF1wfTCDyI8)
- [토스의 서버 인프라 모니터링 — 토스 SLASH 21 (YouTube)](https://www.youtube.com/watch?v=rxurfKT2lD8)
- [SRE 사례 소개 — 토스 SLASH 21 (YouTube)](https://www.youtube.com/watch?v=Tu0MMBgdyTw)
- [토스 SLASH 다시보기 재생목록](https://www.youtube.com/playlist?list=PL1DJtS1Hv1PiGXmgruP1_gM2TSvQiOsFL)
