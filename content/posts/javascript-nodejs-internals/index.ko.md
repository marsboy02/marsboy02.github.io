---
title: "JavaScript & Node.js의 작동 원리: 싱글스레드의 비밀"
date: 2025-01-31
draft: false
tags: ["javascript", "nodejs", "event-loop", "browser"]
translationKey: "javascript-nodejs-internals"
summary: "웹 브라우저의 역사부터 JavaScript 이벤트 루프의 동작 원리, Node.js의 스레딩 모델까지 — 싱글스레드 언어가 어떻게 비동기 처리를 해내는지 깊이 파헤칩니다."
---

![주요 웹 브라우저 로고](images/54_browser_logos.png)

자바스크립트는 **싱글스레드 기반이며 논블로킹 패러다임의 비동기적 동시성 언어**이다. 이 한 줄에 담긴 의미를 제대로 이해하려면, 웹 브라우저의 탄생부터 V8 엔진의 내부 구조, 이벤트 루프의 동작 방식, 그리고 Node.js의 스레딩 모델까지 폭넓게 살펴볼 필요가 있다.

이 글에서는 세 가지 주제를 하나의 흐름으로 엮어 다룬다. 먼저 웹 브라우저의 역사를 통해 자바스크립트와 렌더링 엔진이 어떻게 진화해 왔는지 살펴보고, 자바스크립트의 이벤트 루프가 싱글스레드 환경에서 어떻게 비동기 처리를 구현하는지 분석한다. 마지막으로 Node.js가 "싱글스레드인가, 멀티스레드인가?"라는 오래된 논쟁에 대해 libuv와 함께 답을 찾아본다.

---

## 1. 웹 브라우저의 역사

### 1955년생 전설의 IT 3인방과 WWW의 탄생

컴퓨터 세상에는 전설적인 1955년생 세 명이 있다. 마이크로소프트의 **빌 게이츠**, 애플의 **스티브 잡스**, 그리고 구글의 **에릭 슈미트**가 가장 유명하다. 여기에 **WWW(World Wide Web)**의 창시자인 **팀 버너스 리**를 추가하는 사람들도 있다.

팀 버너스 리는 1989년 스위스 제네바에 위치한 유럽 입자 물리 연구실(CERN)에서 근무하던 도중, 논문 참조 정보를 쉽게 연결하기 위한 시스템을 고안했다. 이것이 바로 WWW의 시작이다. 하이퍼링크를 통해 논문에서 다른 논문으로 바로 연결될 수 있게 하려는 목적이었고, 이 과정에서 **HTML(하이퍼텍스트 마크업 언어)**, **HTTP(하이퍼텍스트 전송 프로토콜)**, **URL(통합 자원 식별자)** 등이 개발되었다.

놀라운 점은, 팀 버너스 리가 이 기술들을 특허도 내지 않고 무료로 공개했다는 것이다. 이를 계기로 정적 콘텐츠를 손쉽게 확인할 수 있는 웹 브라우저가 탄생하기 시작했고, 치열한 브라우저 전쟁의 서막이 열렸다.

### 브라우저 전쟁: Netscape Navigator vs Internet Explorer

![브라우저 전쟁 - Netscape vs IE](images/54_browser_war.png)

1990년대 중반부터 2000년대 초반까지, **Netscape Navigator**와 **Microsoft Internet Explorer(IE)**는 웹 브라우저의 점유율을 놓고 치열하게 경쟁했다. 이 기간은 **브라우저 전쟁**이라고 불리며, 인터넷의 발전과 웹 기술의 진화를 가속화한 시기였다.

1994년, **Netscape Communications**가 Netscape Navigator를 출시하여 웹 브라우저 시장을 점령하기 시작했다. 컴퓨터의 보급이 빠르게 일어나던 시기와 맞물려 약 1년 만에 80%의 점유율을 확보했다.

하지만 인터넷의 잠재력을 깨달은 **Microsoft**가 1995년 Windows 95 출시와 함께 Internet Explorer 1.0을 선보였다. 넷스케이프가 유료로 브라우저를 판매하던 것과 달리, IE는 무료로 제공되었으며 윈도우즈 운영체제에 통합되어 함께 배포되었다. 운영체제 레벨에서 IE를 밀어준 결과, 1997년 이후 IE의 점유율은 급격히 상승하여 2000년대 초반에는 90% 이상을 차지하게 되었다.

### 넷스케이프의 최후의 한 발: 오픈소스

![넷스케이프의 최후의 한 발](images/54_netscape_last_shot.png)

IE의 점유율을 더 이상 이길 수 없다고 판단한 Netscape는 1998년, 브라우저의 **소스 코드를 공개**하기로 결정한다. 이렇게 웹 브라우저의 오픈소스가 풀렸고, **Mozilla 재단**이 이를 이어받아 Mozilla 프로젝트를 시작했다.

이 프로젝트를 통해 탄생한 것이 바로 **Mozilla Firefox**이다. 2002년 Phoenix라는 이름으로 개발을 시작하여, Firebird를 거쳐 2004년 Firefox 1.0이 출시되었다. Firefox는 **최근 닫은 탭 열기, 세션 복구, 피싱 필터, 오디오 및 비디오 기본 지원** 등 당시로서는 혁신적인 기능들을 제공하며 빠르게 사용자를 확보했다.

결정적으로 2008년에 등장한 **Google Chrome**이 IE에 강력한 타격을 입혔다. Chrome은 V8 자바스크립트 엔진을 탑재하여 압도적인 속도를 자랑했고, IE는 가장 무겁고 느린 브라우저로 전락했다. 결국 마이크로소프트는 2022년 6월 15일에 인터넷 익스플로러의 지원을 종료했다.

### 렌더링 엔진과 자바스크립트 엔진

![주요 렌더링 엔진들](images/54_rendering_engines.png)

웹 브라우저 내부에는 **렌더링 엔진**이 탑재되어 있다. HTML, CSS, JavaScript 등을 해석하여 시각적인 요소로 보여주는 역할을 한다. 주요 렌더링 엔진의 계보는 다음과 같다.

- **WebKit**: 2001년 Apple이 KHTML과 KJS 엔진을 기반으로 개발. Safari와 iOS용 모든 브라우저에 사용된다. 모바일 환경에 최적화되어 있으며, Apple의 정책상 iOS에서는 반드시 WebKit을 사용해야 한다.
- **Blink**: 2013년 Google이 WebKit을 포크(fork)하여 개발한 엔진. Chrome, Edge, Opera 등에서 사용되며, V8 자바스크립트 엔진과 함께 **Chromium** 프로젝트를 구성한다.
- **Gecko**: Netscape가 개발을 시작한 후 Mozilla 재단이 이어받은 오픈소스 엔진. Firefox에서 사용되며 강력한 웹 표준 지원과 보안 기능이 특징이다.

![브라우저별 렌더링 엔진과 자바스크립트 엔진 비교](images/54_engine_table.png)

위 표에서 볼 수 있듯이 Chrome은 Blink + V8, Safari는 WebKit + JavaScriptCore, Firefox는 Gecko + SpiderMonkey 조합을 사용한다. 여기서 자주 언급되는 **Chromium**은 Blink 렌더링 엔진과 V8 자바스크립트 엔진의 조합을 의미한다.

> iOS의 Chrome도 겉모습은 Chromium처럼 보이지만, 내부적으로는 Apple 정책에 따라 WebKit을 사용하여 렌더링을 진행한다. 프론트엔드 개발자들이 iOS 환경에서 고생하는 이유가 바로 이 WebKit 렌더링 엔진의 특수성 때문이다.

---

## 2. JavaScript 이벤트 루프

### V8 엔진의 구조

자바스크립트를 실행시키는 **V8 엔진**은 구글에서 제작한 C++로 작성된 자바스크립트 엔진이다. 구글 크롬에 탑재되어 자바스크립트를 작동시키는 역할을 하며, 오픈소스이기 때문에 Node.js에서도 활용한다.

V8 엔진의 작동 원리는 다음과 같다.

1. 자바스크립트 소스 코드를 **파서(Parser)**에게 넘긴다.
2. 파서는 소스 코드를 **AST(Abstract Syntax Tree, 추상 구문 트리)**로 변환한다.
3. AST를 **Ignition** 인터프리터에게 넘기면 자바스크립트를 **바이트 코드**로 변환한다.
4. 바이트 코드를 실행하면서 자주 실행되는 코드는 **TurboFan** 컴파일러로 보내 최적화된 머신 코드로 컴파일한다.

![V8 엔진의 구조](images/23_v8_engine.png)

### 싱글 콜 스택과 비동기 처리

> "싱글 스레드 == 싱글 콜 스택 == 한 번에 한 줄의 코드를 실행한다는 것" - Philip Roberts

V8 엔진은 **하나의 Call Stack**을 가지고 있다. 구조적으로 한 번에 하나밖에 처리하지 못하는 명실상부한 싱글스레드 언어이다. 그런데 어떻게 이런 싱글스레드 언어가 멀티스레드와 유사한 퍼포먼스를 낼 수 있을까?

그 비밀은 바로 **Web APIs**, **이벤트 루프(Event Loop)**, **콜백 큐(Callback Queue)**의 협력에 있다.

![자바스크립트 런타임 환경](images/23_js_runtime.png)

위 그림은 크롬 브라우저에 탑재된 자바스크립트 런타임 환경이다. V8 엔진의 Call Stack 옆에 Web APIs, Callback Queue, Event Loop가 함께 동작한다. 비동기 처리를 지원하는 작업들(예: `setTimeout`, 네트워크 요청)은 **Web APIs**에서 처리되고, 작업이 끝나면 **Callback Queue**로 보내진다. 이후 **Event Loop**가 이를 감지하여 Call Stack이 비었을 때 적절하게 함수를 집어넣는다.

### 동기 함수의 실행 흐름

![동기 함수의 Call Stack 동작](images/23_callstack_sync.png)

비동기 함수가 없는 일반적인 경우를 살펴보자. `init()` 함수를 호출하면 해당 함수 안에 있는 `console.log`가 **Call Stack**에 쌓인다. 쌓인 `console.log`는 바로 처리되며, 모든 처리가 끝나면 마지막으로 `init` 함수가 Call Stack에서 제거되면서 실행이 종료된다.

### 비동기 함수의 실행 흐름

![비동기 함수의 Call Stack 동작](images/23_callstack_async.png)

가장 대표적인 비동기 함수는 `setTimeout`이다. `setTimeout`과 같은 비동기 함수는 먼저 **Call Stack**에 올라간 다음 **Web APIs**로 이동한다. Web APIs 내에서 타이머 동작을 마치면 **콜백 함수**를 Callback Queue에 집어넣고, Event Loop가 이를 하나씩 Call Stack으로 올려 보낸다.

![여러 비동기 함수의 Call Stack 동작](images/23_callstack_multiple_async.png)

`setTimeout`을 여러 개 사용하는 경우, 소스 코드를 한 줄씩 훑으며 차례대로 Call Stack에 갔다가 Web APIs로 보내진다. 그 후 각 콜백 함수가 처리 완료되면 **Callback Queue**에 쌓이고, Event Loop가 하나씩 Call Stack으로 보낸다. 이런 메커니즘 덕분에 **Call Stack, Web APIs, Callback Queue, Event Loop**가 상호작용하면서 싱글스레드 기반의 자바스크립트가 멀티스레드에 뒤지지 않는 퍼포먼스를 발휘할 수 있는 것이다.

---

## 3. Node.js 스레딩 모델

### Node.js란 무엇인가

Node.js 공식 홈페이지의 소개에 따르면 다음과 같다.

> Node.js는 오픈소스 및 크로스 플랫폼 자바스크립트 런타임 환경입니다. Node.js는 구글 크롬의 핵심인 V8 자바스크립트 엔진을 브라우저 외부에서 실행합니다.

핵심은 **브라우저 외부에서 실행한다**는 점이다. 자바스크립트는 원래 웹사이트를 위해 태어난 언어로, 1995년 넷스케이프 커뮤니케이션즈의 브렌던 아이크에 의해 개발되었다. 웹 브라우저에서 HTML, CSS와 함께 동작하는 V8 엔진을, 브라우저 밖에서도 독립적으로 실행할 수 있게 만든 것이 바로 **Node.js**이다.

### 싱글스레드인가, 멀티스레드인가?

자바스크립트는 이견 없이 태생부터 싱글스레드이다. 하지만 Node.js를 싱글스레드로 보느냐 멀티스레드로 보느냐에 대해서는 관점 차이가 있다. 결론부터 말하면 **"둘 다 맞다"** -- 어디에 초점을 맞추느냐에 달려 있다.

![자바스크립트의 Memory Heap과 Call Stack](images/73_js_memory_heap.png)

자바스크립트는 하나의 **콜 스택(Call Stack)**을 가지고 있으며, 콜 스택에 있는 작업을 하나씩 처리해나가는 싱글스레드 방식으로 동작한다. **Memory Heap**에는 변수와 같은 정보가 저장되고, 함수를 실행하면 Call Stack에 하나씩 쌓인다.

### 블로킹 vs 논블로킹, 동기 vs 비동기

자바스크립트의 이벤트 루프를 이해하려면 먼저 **블로킹/논블로킹**과 **동기/비동기**의 차이를 알아야 한다.

![블로킹/논블로킹, 동기/비동기 비교](images/73_blocking_nonblocking.png)

- **블로킹 vs 논블로킹**: 특정 함수를 실행하는 동안 다른 일을 할 수 있는가의 여부이다. 블로킹은 함수가 끝날 때까지 대기해야 하며, 논블로킹은 다른 일을 병행할 수 있다. **제어 권한의 유무**가 핵심이다.
- **동기 vs 비동기**: 작업 완료를 누가 확인하는가의 차이이다. 동기 작업은 호출한 측에서 완료 여부를 직접 확인하며, 비동기 작업은 **콜백(Callback)**이 완료를 알려준다.

자바스크립트가 싱글스레드이면서도 멀티스레드에 뒤처지지 않는 이유는 바로 **논블로킹 비동기 방식**으로 동작하기 때문이다. 특정 작업을 콜백으로 실행한 다음 다른 작업을 수행하고, 비동기 작업이 처리되어 콜백 함수가 돌아오면 그때 그 결과를 처리하는 방식이다.

### 비동기 함수와 비동기 동작의 차이

자바스크립트에서 `async`와 `await` 키워드는 각각 다음과 같은 역할을 한다.

- **async**: 함수 앞에 붙이면 해당 함수는 자동으로 `Promise`를 반환하는 **비동기 함수**가 된다.
- **await**: 비동기 함수가 완료될 때까지 **기다린 후** 결과를 반환한다.

중요한 점은, **비동기 함수 자체가 별도로 처리되는 것이 아니라, 비동기 함수 안의 비동기 동작이 별도로 처리된다**는 것이다.

```javascript
async function example() {
    console.log("1. 함수 시작");

    setTimeout(() => {
        console.log("2. 비동기 작업(setTimeout) 완료");
    }, 2000); // 2초 후 실행

    console.log("3. 함수 종료");
}
```

위 코드에서 `setTimeout()`은 비동기 동작이다. 실행하면 1번과 3번이 먼저 출력되고, 2초 후에 2번이 출력된다. `setTimeout`에서 멈추지 않고 나머지 코드가 먼저 실행되는 것이 핵심이다.

`await`를 사용하면 비동기 동작의 완료를 기다릴 수 있다.

```javascript
function delay(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
}

async function example() {
    console.log("1. 함수 시작");

    await delay(2000); // 2초 기다림 (비동기 동작)

    console.log("2. 2초 후 실행됨");
}

example();
console.log("3. 함수 실행 후 다른 작업 수행");
```

위 코드는 `example()` 함수 실행 중 `await`을 만나면, `example()` 내부에서는 2초를 기다리지만 자바스크립트는 논블로킹 방식으로 동작하므로 `"3. 함수 실행 후 다른 작업 수행"`이 먼저 출력된다. 2초 후에야 `"2. 2초 후 실행됨"`이 출력된다.

만약 `await` 없이 성급하게 비동기 함수의 결과를 확인하면, 아직 완료되지 않은 **Promise** 객체가 반환된다.

```javascript
async function example() {
    return delay(2000); // await을 사용하지 않음
}

console.log(example()); // Promise { <pending> }
```

### 이벤트 루프의 동작 구조

![자바스크립트 이벤트 루프](images/73_js_event_loop.png)

자바스크립트를 실행하면 콜 스택에 있는 함수를 하나씩 싱글스레드로 처리한다. 논블로킹 비동기 방식으로 동작하려면, 비동기 작업을 대신 처리해줄 존재가 필요하다.

- **웹 브라우저 환경**: **Web APIs**가 멀티스레드 형식으로 비동기 작업을 처리한다.
- **Node.js 환경**: **libuv**가 멀티스레드를 지원하여 비동기 작업을 처리한다.

바로 이 지점에서 **자바스크립트를 멀티스레드라고 볼 수 있는 관점**이 나온다. Web APIs와 libuv는 각각 멀티스레드로 동작하며, Java의 스레드풀처럼 libuv 또한 스레드풀을 가지고 있다.

![libuv 공식 사이트](images/73_libuv.webp)

### Callback Queue: Task Queue와 Microtask Queue

![콜백 큐와 이벤트 루프 구조](images/73_callback_queue.png)

자바스크립트에서 실행된 모든 함수는 먼저 콜 스택으로 들어온다. 그중 비동기 함수는 Web APIs(또는 libuv)를 호출하고, 완료된 콜백 함수를 **콜백 큐**에 집어넣는다.

콜백 큐는 사실 하나가 아니라 **두 개**이다.

- **Task Queue**: `setTimeout`, `setInterval` 등이 완료된 후 실행될 콜백이 대기하는 곳
- **Microtask Queue**: `Promise.then()`, `MutationObserver` 등의 작업이 들어가는 큐

**Microtask Queue가 우선순위가 더 높다.** 이벤트 루프가 한 사이클을 돌 때마다 가장 먼저 실행되며, 대부분 즉시 실행될 필요가 있는 중요한 비동기 로직들이 이곳에 들어간다.

### Event Loop의 6단계

![이벤트 루프의 단계](images/73_event_loop_phases.png)

"이벤트 루프가 한 사이클을 돈다"라는 표현에서 **Loop**는 6개의 단계(Phase)를 순환한다는 의미이다. 각 단계에서 비동기 작업의 실행 순서를 결정하며, 그 구성은 다음과 같다.

1. **Timers**: `setTimeout()`, `setInterval()` 콜백 실행
2. **Pending Callbacks**: 일부 시스템 I/O 작업의 콜백 실행 (TCP 오류 등)
3. **idle, prepare**: 내부적인 최적화 작업 수행 (libuv 최적화)
4. **Poll**: 대기 중인 I/O 이벤트를 확인하고 처리
5. **Check**: `setImmediate()` 콜백 실행
6. **Close Callbacks**: `socket.on('close', callback)` 같은 닫기 이벤트 실행

웹 브라우저에서는 **Chromium** 같은 브라우저 엔진이 이벤트 루프의 구현을 담당하고, Node.js에서는 **libuv**가 이 역할을 수행한다.

### Node.js는 공식적으로 싱글스레드

![Node.js 공식 문서](images/73_nodejs_about.png)

Node.js 공식 문서를 보면, Node.js는 **싱글스레드**임을 명확히 내세운다. 가장 결정적인 증거는 **Node.js 사용자에게는 Lock이 없다**는 것이다.

멀티스레드 환경에서는 하나의 자원에 동시에 접근하는 여러 스레드에 대해 **데드락(Deadlock)**을 방지하기 위한 스핀락, 세마포어, 뮤텍스 등의 동기화 메커니즘이 필요하다. 하지만 Node.js에는 이런 것이 없다 -- 이것이 Node.js가 싱글스레드라는 강력한 반증이다.

정리하면 다음과 같다.

| 관점 | 근거 |
|------|------|
| **싱글스레드** | 자바스크립트 자체는 하나의 콜 스택으로 동작하며, Lock 메커니즘이 없다 |
| **멀티스레드** | libuv가 스레드풀을 가지고 비동기 I/O를 멀티스레드로 처리한다 |

둘 다 틀린 말은 아니다. 다만 **메인 실행 흐름(자바스크립트 코드 실행)**은 싱글스레드이고, **비동기 I/O 처리를 돕는 libuv**는 멀티스레드라는 것이 정확한 설명이다.

---

## 4. 비동기 프로그래밍 패러다임의 진화

자바스크립트의 비동기 처리 방식은 언어의 성장과 함께 크게 세 단계로 진화해 왔다.

### Callbacks: 시작점

자바스크립트 비동기 처리의 원형은 **콜백 함수**이다. 비동기 작업이 완료되면 미리 전달해 둔 함수를 호출하는 방식이다.

```javascript
function fetchData(callback) {
    setTimeout(() => {
        callback(null, { id: 1, name: "marsboy" });
    }, 1000);
}

fetchData((error, data) => {
    if (error) {
        console.error("에러 발생:", error);
        return;
    }
    console.log("데이터:", data);
});
```

단순한 경우에는 직관적이지만, 비동기 작업이 중첩되면 이른바 **콜백 지옥(Callback Hell)**이 발생한다.

```javascript
// 콜백 지옥의 예시
getUser(userId, (error, user) => {
    getOrders(user.id, (error, orders) => {
        getOrderDetails(orders[0].id, (error, details) => {
            getShippingInfo(details.shippingId, (error, shipping) => {
                console.log("배송 정보:", shipping);
                // 점점 깊어지는 들여쓰기...
            });
        });
    });
});
```

코드의 가독성이 극단적으로 떨어지고 에러 처리도 각 단계마다 반복해야 하는 문제가 있었다.

### Promises: 체이닝의 등장

ES6(2015)에서 도입된 **Promise**는 콜백 지옥을 해결하기 위해 등장했다. 비동기 작업의 결과를 나타내는 객체로, `then()`과 `catch()`를 통해 체이닝 방식으로 비동기 흐름을 제어한다.

```javascript
function fetchUser(userId) {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve({ id: userId, name: "marsboy" });
        }, 1000);
    });
}

fetchUser(1)
    .then(user => getOrders(user.id))
    .then(orders => getOrderDetails(orders[0].id))
    .then(details => getShippingInfo(details.shippingId))
    .then(shipping => console.log("배송 정보:", shipping))
    .catch(error => console.error("에러 발생:", error));
```

콜백 지옥의 깊은 들여쓰기가 평탄화되었고, `.catch()` 하나로 전체 체인의 에러를 한곳에서 처리할 수 있게 되었다.

Promise의 세 가지 상태는 다음과 같다.
- **Pending**: 아직 처리가 완료되지 않은 상태 (`Promise { <pending> }`)
- **Fulfilled**: 처리가 성공적으로 완료된 상태
- **Rejected**: 처리가 실패한 상태

### async/await: 동기식 코드처럼 작성하기

ES2017에서 도입된 `async/await`는 Promise 기반 코드를 마치 동기 코드처럼 작성할 수 있게 해주는 문법적 설탕(Syntactic Sugar)이다.

```javascript
async function getShippingInfoForUser(userId) {
    try {
        const user = await fetchUser(userId);
        const orders = await getOrders(user.id);
        const details = await getOrderDetails(orders[0].id);
        const shipping = await getShippingInfo(details.shippingId);

        console.log("배송 정보:", shipping);
        return shipping;
    } catch (error) {
        console.error("에러 발생:", error);
    }
}
```

위 코드는 비동기 작업임에도 불구하고 마치 동기 코드처럼 위에서 아래로 순서대로 읽힌다. `try/catch`를 사용한 에러 처리도 동기 코드와 동일한 패턴이다.

<!-- TODO: Callbacks -> Promises -> async/await 진화 과정을 보여주는 다이어그램 추가 -->

### 비동기 패러다임 비교

| 방식 | 도입 시기 | 장점 | 단점 |
|------|-----------|------|------|
| **Callbacks** | ES1 (1997) | 단순하고 직관적 | 콜백 지옥, 에러 처리 복잡 |
| **Promises** | ES6 (2015) | 체이닝, 통합 에러 처리 | `then()` 체인이 길어질 수 있음 |
| **async/await** | ES2017 | 동기식 가독성, `try/catch` 지원 | 최상위 레벨에서의 사용 제한 (ESM 모듈에서 해결) |

---

## 마치며

자바스크립트는 웹 페이지의 동적 기능을 위해 태어난 언어이다. 태생적으로 싱글스레드라는 제약을 안고 시작했지만, 이벤트 루프와 논블로킹 비동기 패러다임이라는 독특한 방식으로 이 한계를 극복해냈다.

Node.js는 이 자바스크립트를 브라우저 밖으로 꺼내어 서버 사이드에서도 동작할 수 있게 만들었고, libuv를 통해 비동기 I/O를 멀티스레드로 처리하면서도 개발자에게는 싱글스레드의 단순함을 유지해주었다. Lock도 없고 데드락도 없으니, 개발자 입장에서는 복잡한 동기화 문제를 걱정하지 않아도 된다.

1995년에 브렌던 아이크가 10일 만에 만들었다는 이 언어가, 30년이 지난 지금까지 웹 생태계의 핵심으로 자리 잡고 있다는 것은 정말 경이로운 일이다. 콜백에서 Promise로, 다시 async/await로 진화해온 비동기 프로그래밍 패러다임의 발전이 그 증거이다.

---

## 참고 자료

- **[maybe.works]** The History of Web Browsers: Full Guide: [https://maybe.works/blogs/browser-wars-the-history-of-browsers-and-chromium-victory](https://maybe.works/blogs/browser-wars-the-history-of-browsers-and-chromium-victory)
- **[Node.js]** Introduction to Node.js: [https://nodejs.org/ko/learn/getting-started/introduction-to-nodejs](https://nodejs.org/ko/learn/getting-started/introduction-to-nodejs)
- **[Evans Library]** V8 엔진은 어떻게 내 코드를 실행하는 걸까?: [https://evan-moon.github.io/2019/06/28/v8-analysis/](https://evan-moon.github.io/2019/06/28/v8-analysis/)
- **[JSConf]** What the heck is the event loop anyway?: [https://www.youtube.com/watch?v=8aGhZQkoFbQ](https://www.youtube.com/watch?v=8aGhZQkoFbQ)
- **[BlaCk_Log]** 동시성, 병렬, 비동기, 논블럭킹과 컨셉들: [https://black7375.tistory.com/90](https://black7375.tistory.com/90)
- **[Alexander Zlatkov]** How JavaScript works: [https://medium.com/sessionstack-blog/how-does-javascript-actually-work-part-1-b0bacc073cf](https://medium.com/sessionstack-blog/how-does-javascript-actually-work-part-1-b0bacc073cf)
- **[helloinyong]** Node.js가 왜 싱글 스레드로 불리는 지 "정확한 이유": [https://helloinyong.tistory.com/350](https://helloinyong.tistory.com/350)
