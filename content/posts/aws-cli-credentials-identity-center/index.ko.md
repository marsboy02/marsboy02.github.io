---
title: "AWS CLI는 자격증명을 어떻게 관리하는가 — saml2aws의 종말까지"
date: 2026-05-16
draft: false
tags: ["aws", "iam", "identity-center", "security"]
translationKey: "aws-cli-credentials-identity-center"
summary: "~/.aws/ 디렉토리 구조부터 임시 자격증명을 얻는 5가지 경로, 그리고 Passkey 도입으로 무너진 saml2aws의 마지막 풍경까지 — AWS CLI 자격증명의 원리를 한 번에 정리한다."
---

어느 날부터 갑자기 `saml2aws login`이 안 됐다. 정확히는, 패스워드 매니저를 사용하기 위해서 passkey를 옮기면서 saml2aws가 갑자기 안되기 시작했다.

`--verbose`를 붙여서 다시 실행해보니 "HTML 파싱에 실패했다"는 로그가 떠 있었다. 호기심에 saml2aws의 오픈소스를 직접 열어보니, IdP별 provider가 하나같이 **IdP 로그인 페이지의 HTML을 받아 폼을 파싱하는 구조**였다. 정상적인 비밀번호/OTP 페이지를 가정하고 짜여 있어서, Passkey 인증 페이지가 끼어든 순간 셀렉터가 어긋나며 무너진 것이다. "도구가 깨졌다"기보다는 **헤드리스 자동화가 통하지 않는 방향으로 인증 모델이 옮겨가고 있다**는 신호에 가까웠다.

여기서 두 가지 의문이 따라붙었다. **AWS CLI는 도대체 자격증명을 어떻게 받아오고 어디에 저장하는 걸까? 그리고 saml2aws를 대체해야 한다면 어떤 길이 구조적으로 맞는 답인가?** 이 글은 그 두 질문을 따라가는 기록이다. `~/.aws/` 디렉토리 해부에서 시작해서 임시 자격증명을 얻는 다섯 가지 경로를 비교하고, 마지막으로 IAM Identity Center가 왜 그 답이 되는지로 이어진다.

---

## 1. AWS 자격증명, 결국 어디서 오는가

AWS CLI를 비롯한 모든 AWS SDK는 자격증명이 필요할 때마다 **자격증명 해석 체인**(credential resolution chain)이라는 정해진 순서를 따른다. 환경변수가 있는지 보고, 없으면 프로필 설정을 보고, 그것도 없으면 EC2 인스턴스 메타데이터나 컨테이너 역할로 넘어간다. 어떤 SDK든 이 우선순위는 거의 동일하다.

이 글이 다룰 부분은 그중 **프로필 기반** 흐름, 즉 `~/.aws/` 디렉토리를 통한 인증이다. 로컬에서 CLI를 쓰는 개발자에게 가장 익숙한 길이기도 하다.

자격증명은 크게 두 부류로 나뉜다.

- **장기 자격증명**(long-term credentials) — IAM User의 access key. `AKIA`로 시작하며 무기한 유효하다. 직접 로테이션하지 않으면 영원히 쓸 수 있다.
- **단기 자격증명**(short-term credentials) — STS가 발급한 임시 키. `ASIA`로 시작하며 `session token`이 짝지어 있다. 보통 1시간에서 12시간 사이로 유효하다.

`~/.aws/` 디렉토리 구조는 결국 "이 두 종류의 자격증명을, 어떻게 발급받고, 어디에 저장하고, 어떻게 갱신할 것인가"에 대한 답이다. 각 파일이 무엇을 위해 존재하는지를 이해하면, 문제가 생겼을 때 어디를 봐야 하는지가 자연스럽게 보인다.

---

## 2. `~/.aws/` 디렉토리 해부

### 2.1 `~/.aws/config` — 프로필의 뼈대

가장 기본이 되는 설정 파일. `aws configure`, `aws configure sso`, 또는 직접 편집으로 만들어진다. 프로필 단위로 region, output format, 그리고 자격증명을 얻는 방식을 정의한다.

```ini
[default]
region = ap-northeast-2
output = json

[profile prod]
region = ap-northeast-2
role_arn = arn:aws:iam::123456789012:role/AdminRole
source_profile = default
mfa_serial = arn:aws:iam::111111111111:mfa/hj.kang

[profile prod-sso]
sso_session = teamsparta
sso_account_id = 123456789012
sso_role_name = AdministratorAccess
region = ap-northeast-2

[sso-session teamsparta]
sso_start_url = https://teamsparta.awsapps.com/start
sso_region = ap-northeast-2
sso_registration_scopes = sso:account:access
```

여기서 자주 헷갈리는 디테일이 두 가지 있다.

첫째, `default` 프로필만 `[default]`로 쓰고, 나머지는 `[profile 이름]`처럼 **`profile` 접두사가 필수**다. 이건 뒤에 나올 `credentials` 파일의 규칙과 다르기 때문에 더 헷갈린다.

둘째, `[sso-session]` 섹션은 AWS CLI v2의 IAM Identity Center 통합에서 도입된 비교적 새로운 개념이다. 여러 프로필이 **하나의 SSO access token을 공유**할 수 있게 해준다. 50개 계정에 접근 가능한 사용자라도 SSO 로그인은 한 번만 하면 되는 이유가 여기에 있다.

### 2.2 `~/.aws/credentials` — 평문으로 박혀 있는 키

IAM User access key를 평문으로 저장하는 파일. `aws configure`로 키를 입력하면 생긴다.

```ini
[default]
aws_access_key_id = AKIA...
aws_secret_access_key = ...

[some-temp-profile]
aws_access_key_id = ASIA...
aws_secret_access_key = ...
aws_session_token = ...
```

세 가지를 기억해두면 좋다.

- `AKIA`로 시작하는 키는 IAM User의 **영구 access key**.
- `ASIA`로 시작하는 키는 **STS가 발급한 임시 자격증명**. 이 경우 `aws_session_token`이 함께 들어 있다.
- 섹션 이름은 `[프로필명]` — `config` 파일과 달리 `profile` 접두사가 **없다**.

이 파일은 **ISMS-P 같은 보안 컴플라이언스 관점에서 가장 위험한 파일**이다. 평문으로 장기 키가 디스크에 박혀 있기 때문이다. 노트북을 분실하거나 백업이 새면 그대로 노출된다. 뒤에서 보겠지만, "장기 키를 `credentials` 파일에 두는 운영 방식"은 점점 사라지는 추세다.

### 2.3 `~/.aws/sso/cache/` — SSO 토큰의 집

`aws sso login` 또는 `aws configure sso`로 SSO 로그인을 하면 만들어지는 디렉토리. 두 종류의 JSON 파일이 들어간다.

**파일 1**: `{sha1-of-start-url-or-session-name}.json` — 실제 SSO access token이 담겨 있다.

```json
{
  "startUrl": "https://teamsparta.awsapps.com/start",
  "region": "ap-northeast-2",
  "accessToken": "eyJlbmMi...",
  "expiresAt": "2026-05-16T20:00:00Z",
  "refreshToken": "...",
  "clientId": "...",
  "clientSecret": "...",
  "registrationExpiresAt": "2026-08-14T..."
}
```

- access token은 보통 8시간 유효하다.
- refresh token이 있으면 백그라운드에서 자동 갱신된다.
- **이 토큰 한 장으로 SSO 세션이 접근 가능한 모든 계정과 Role의 임시 자격증명을 발급받을 수 있다.** 그래서 실질적으로는 `credentials` 파일보다도 더 민감한 파일이다.

**파일 2**: `botocore-client-id-{region}.json` — OIDC Dynamic Client Registration의 결과.

```json
{
  "clientId": "...",
  "clientSecret": "...",
  "registrationExpiresAt": "2026-08-14T..."
}
```

AWS SSO OIDC에 CLI를 "클라이언트"로 등록한 정보다. 약 3개월 유효하고, 매 로그인마다 새로 만들어지지 않는다. 이게 무슨 의미인지는 5장 IAM Identity Center 흐름에서 다시 등장한다.

### 2.4 `~/.aws/cli/cache/` — AssumeRole 결과 캐시

`AssumeRole` 류의 STS 호출 결과를 캐시하는 디렉토리. `role_arn` + `source_profile`을 사용하는 프로필을 쓸 때 만들어진다.

```json
{
  "Credentials": {
    "AccessKeyId": "ASIA...",
    "SecretAccessKey": "...",
    "SessionToken": "...",
    "Expiration": "2026-05-16T20:00:00Z"
  },
  "AssumedRoleUser": {
    "AssumedRoleId": "AROA...:session",
    "Arn": "arn:aws:sts::123456789012:assumed-role/..."
  }
}
```

파일명은 프로필 설정(`role_arn`, `role_session_name`, `mfa_serial` 등)을 해시한 값이다. CLI를 쓸 때마다 MFA 토큰을 다시 묻지 않는 것은 바로 이 캐시 덕분이다. 만료 전까지는 같은 임시 자격증명을 재사용한다.

### 2.5 그 외 — 알아두면 좋은 부수 파일

- `~/.aws/cli/alias` — CLI alias 정의 파일.
- `~/.aws/models/` — 커스텀 서비스 모델.
- `*.lock` — 동시성 락 파일. 잠깐 생겼다 사라진다.

이 정도면 디렉토리 구조의 큰 그림은 충분하다. 이제 이 파일들에 어떻게 자격증명이 채워지는지, 그 다섯 가지 길을 비교해보자.

---

## 3. 임시 자격증명을 얻는 5가지 경로

모든 임시 자격증명은 결국 **STS API 또는 그 변형**을 통해 발급된다. 어떤 API를 어떤 인풋으로 호출하는지, 결과가 어디에 저장되는지, 어떻게 갱신되는지가 각 경로의 본질이다.

### 3.1 한눈에 보는 비교표

| 경로 | API | 인풋 | 결과 저장 위치 | 갱신 방식 |
|---|---|---|---|---|
| **AssumeRole** | `sts:AssumeRole` | source_profile의 키 (+ MFA) | `~/.aws/cli/cache/` | 만료 시 재호출 |
| **AssumeRoleWithSAML** | `sts:AssumeRoleWithSAML` | SAML assertion | `~/.aws/credentials` (saml2aws가 직접 씀) | 없음 — 재인증 필요 |
| **AssumeRoleWithWebIdentity** | `sts:AssumeRoleWithWebIdentity` | OIDC JWT | 메모리 또는 토큰 파일 경로 | 토큰 파일 갱신 시 자동 |
| **SSO** | OIDC Device Auth + `sso:GetRoleCredentials` | SSO access token | 자격증명은 메모리 또는 일시 캐시; access token은 `~/.aws/sso/cache/` | refresh token으로 자동 갱신 |
| **credential_process** | 외부 프로세스 stdout 파싱 | 임의 (도구마다 다름) | 도구에 따라 다름 (예: aws-vault는 OS keychain) | 도구가 자체 처리 |

### 3.2 다섯 가지 경로의 본질

**(1) AssumeRole 체인** — 가장 전통적인 방식이다. `[default]`에 IAM User 키를 두고, `[profile prod]`에서 `role_arn` + `source_profile = default`로 역할을 갈아탄다. MFA를 요구하도록 정책을 설정하면 `mfa_serial`을 추가한다. 결과는 `cli/cache/`에 캐시되어 만료 전까지 재사용된다. 장기 키가 디스크에 있어야만 동작한다는 점이 한계다.

**(2) AssumeRoleWithSAML** — SAML 기반 SSO 환경에서 쓰는 STS 전용 엔드포인트. saml2aws가 내부적으로 호출하던 바로 그 API다. SAML assertion을 들고 STS에 가면 임시 자격증명을 받아 `~/.aws/credentials`에 박아 넣는 식이다. refresh token이 없어서 만료되면 처음부터 재인증해야 하고, 인트로에서 본 것처럼 IdP 로그인 페이지가 Passkey 같은 비-HTML 인증으로 바뀌는 순간 자동화 도구는 거기서 멈춘다.

**(3) AssumeRoleWithWebIdentity** — OIDC JWT를 들고 STS에 가서 자격증명을 받는다. EKS의 **IRSA**(IAM Roles for Service Accounts), GitHub Actions OIDC, GCP/Azure 워크로드의 cross-cloud 인증 등이 모두 이 경로다. `~/.aws/config`에 `web_identity_token_file`과 `role_arn`을 지정하면, SDK가 토큰 파일을 읽어 STS에 제출한다. 사람이 아니라 **워크로드가 들고 다니는 인증 방식**이다.

**(4) SSO**(IAM Identity Center) — 두 단계 흐름이다. 첫 단계는 OIDC Device Authorization Grant로 사용자가 브라우저에서 인증해서 access token을 얻는 것(이게 `~/.aws/sso/cache/`에 저장된다). 두 번째는 이 access token으로 `sso:GetRoleCredentials`를 호출해서 특정 계정/Role의 임시 자격증명을 받는 것. 흥미로운 점은 받아온 임시 자격증명 자체는 디스크에 별도로 캐시되지 않는 경우가 많고, **access token만 캐시된다**는 것이다.

**(5) credential_process** — `~/.aws/config`에 `credential_process = /path/to/tool ...`를 적어두면, SDK가 자격증명이 필요할 때마다 그 프로세스를 실행해서 stdout으로 JSON을 받아 쓴다. aws-vault, 1Password CLI 같은 도구가 SDK 표준에 끼어드는 공식 후크다.

### 3.3 한 가지 흐름은 그림으로

표만으로는 머리에 잘 들어오지 않으니, 가장 헷갈리는 **AssumeRoleWithSAML** 흐름만 짧게 풀어두자.

![AssumeRoleWithSAML credential flow](images/assume-role-with-saml-flow.svg)

인트로에서 본 saml2aws는 정확히 이 흐름의 위쪽, **IdP 응답에서 SAMLResponse를 뽑아내는 지점**에서 깨졌다. IdP가 비밀번호 폼 대신 Passkey 페이지를 내려보내자 폼 셀렉터가 어긋났고, assertion 자체를 손에 쥐지 못했으니 그 뒤의 STS 호출까지 가지도 못한 것이다. WebAuthn/Passkey는 피싱 저항성을 위해 origin binding과 user presence를 강제하는데, 그 보안 속성이 정확히 헤드리스 자동화 도구를 막는 속성이기도 하다. saml2aws만의 문제가 아니라, **자동화된 SAML 클라이언트 전반의 시대가 저물고 있다는 신호**다. 그래서 자연스럽게 떠오르는 질문이 다음 장의 주제다 — 그럼 어디로 가야 하나.

---

## 4. 그래서 IAM Identity Center는 무엇이 다른가

### 4.1 OIDC Device Authorization Grant — 인증과 토큰 수집의 분리

IAM Identity Center는 SAML이 아니라 OIDC를 쓴다. 그것도 **Device Authorization Grant**(RFC 8628)라는 조금 특이한 흐름이다.

1. CLI가 "내가 인증해야 한다"고 SSO OIDC 엔드포인트에 요청한다.
2. 응답으로 user code와 verification URL을 받는다.
3. CLI는 사용자에게 **브라우저로 URL을 열라고 안내**하고, 백그라운드에서 token endpoint를 폴링한다.
4. 사용자가 브라우저에서 자기 IdP로 인증한다 — Passkey든, WebAuthn 키든, SAML이든 IdP가 원하는 방식 그대로.
5. 인증이 완료되면 CLI의 폴링이 access token과 refresh token을 받는다.

핵심은 **"인증은 브라우저가, 토큰 수집은 CLI가"** 분리되었다는 점이다. CLI는 인증 과정에 끼어들지 않는다. 그래서 Passkey가 작동한다. saml2aws가 죽은 바로 그 이유, 헤드리스로 IdP 로그인을 가로채려 했다는 그 약점이 여기엔 없다.

### 4.2 갱신 모델 — 하루 한 번이면 충분

- **access token**: 약 8시간 유효, 만료 시 refresh token으로 자동 갱신.
- **refresh token**: 약 90일 유효(registration expiry와 묶임).
- 결과적으로 사실상 **하루에 한 번 정도 브라우저 인증, 그 외엔 자동**이다.

saml2aws가 1시간마다 끊겼던 것과 비교하면 운영 경험 자체가 다르다. 작업 중간에 인증이 끊겨서 흐름이 깨지는 경험이 사라진다.

### 4.3 `sso-session` 섹션의 진짜 의미

`~/.aws/config`의 `[sso-session teamsparta]` 같은 섹션은 단순한 그룹핑이 아니라, **여러 프로필이 같은 access token을 공유한다**는 의미다. 50개 AWS 계정에 접근 가능한 사용자가 50번 로그인하지 않고 한 번의 SSO 로그인으로 모든 계정의 임시 자격증명을 받을 수 있는 이유가 여기에 있다.

조직이 커지고 계정이 늘어나면 이 차이가 점점 더 중요해진다. 한 번의 인증이 한 사용자의 모든 권한 경계로 자연스럽게 펼쳐지는 모델은, 운영자에게도 사용자에게도 훨씬 단순하다.

---

## 마치며

이 글을 쓰면서 다시 확인한 사실은, **`~/.aws/`의 구조는 "어떻게 자격증명을 발급받고, 어디에 저장하고, 어떻게 갱신할까"라는 질문에 대한 답**이라는 것이다. 각 파일이 어떤 흐름을 위해 존재하는지를 이해하면, 문제가 생겼을 때 어디를 봐야 하는지가 자연스럽게 보인다.

saml2aws가 깨졌던 그날을 돌아보면, 그건 단순히 도구 하나가 죽은 게 아니었다. 헤드리스 클라이언트가 IdP 로그인 화면을 자동화하던 시대, 그리고 장기 access key가 평문으로 디스크에 박혀 있어도 별 거부감 없던 시대 — 그 두 시대가 한꺼번에 저무는 풍경이었다. 피싱 저항 인증이 표준이 되고, 단기 자격증명과 refresh token이 기본값이 되는 흐름은 이미 시작됐다.

IAM Identity Center로 옮기는 일이 단순한 도구 교체로 보이지 않는 이유도 거기에 있다. 브라우저-CLI 분리, refresh token 모델, `sso-session` 공유 같은 구조적 개선이 함께 따라오기 때문이다. AWS 위에서 일하는 사람이라면, 늦지 않게 이 흐름에 올라타는 편이 좋다.

운영 관점에서 비용 최적화를 함께 정리한 [이전 글]({{< relref "/posts/aws-cost-optimization" >}})도 같이 읽으면, AWS를 "잘 쓴다"는 게 어떤 모양인지 좀 더 입체적으로 보일 것이다.
