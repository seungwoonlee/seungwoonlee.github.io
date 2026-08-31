---
title: 'Track1 Maestro 플로우 작성 심층 가이드 — YAML 구문·선택자·assertion·스크린샷·하위 플로우'
description: '리서치 Track 1 — Maestro 플로우 작성'
pubDate: 'Aug 31 2026'
heroImage: '../../assets/astro-banner.svg'
---

> 이 글은 Hermes 리서치 Track 1(Maestro) 작업결과물을 블로그로 옮긴 것입니다.


# Track 1 — Maestro 플로우 작성 심층 가이드

> 리서치 일자: 2026-08-31 · 생성: Hermes Research Orchestrator · 상태: 완료
> 연구 영역: **Maestro 플로우 작성** (YAML 구문, 선택자, assertion, 스크린샷, 하위 플로우)
> 참고: 같은 날짜의 `2026-08-31_T1_Maestro-CLI-리서치.md`(winClaw)는 CLI 명령어 체계·CI/CD·병렬 실행을 폭넓게 다뤘으므로, 본 문서는 **플로우 작성 실무**에 집중해 중복을 피했다.

## 📌 개요

Maestro의 핵심은 **선언적 YAML 플로우**다. 테스트는 코드가 아니라 사람이 읽을 수 있는 YAML 파일이며, `appId`(앱 식별자)와 명령어 목록으로 구성된다. 이 문서는 "플로우를 어떻게 잘 쓰는가"에 집중한다. 즉 ① YAML 파일의 정확한 구조와 문법, ② 안정적인 선택자(selector) 고르는 법, ③ assertion(검증) 작성법, ④ 스크린샷 캡처와 활용, ⑤ 하위 플로우(subflow)로 재사용성을 높이는 패턴을 다룬다.

핵심 원칙 하나를 먼저 기억하자: **Maestro는 모든 명령어에 자동 대기(wait)와 재시도(retry)가 내장되어 있어 명시적 sleep을 거의 쓰지 않는다.** 이 설계 덕분에 플로우가 짧고, flakiness(불안정성)의 가장 흔한 원인인 타이밍 문제가 사라진다.

## 🔍 조사 키워드

- Maestro YAML flow 구조 (appId + commands)
- Maestro selectors (text / id / index / regex / enabled)
- Maestro assertion (assertVisible / assertNotVisible / assertTrue)
- Maestro screenshots (takeScreenshot / screenshotOnFailure)
- Maestro subflow (runFlow / env 전달 / 파라미터화)
- Maestro runScript / evalScript (동적 데이터)

## 📚 조사 소스

1. QASkills.sh — "Maestro Mobile Testing: The Complete YAML-Based Guide for 2026" (2026-06-14)
2. Maestro 공식 블로그 — "Build Reusable YAML Test Flows: Write Once, Test Everywhere" (2026-03-13)
3. Maestro 공식 문서 (docs.maestro.dev) — 플로우/선택자/하위 플로우 참조


## 1. YAML 플로우의 기본 구조

Maestro 플로우는 두 부분으로 나뉜다.

1. **헤더(선택)**: `appId`, `name`, `env`(환경 변수) 등 메타데이터
2. **본문**: `---` 구분자 뒤에 나열되는 명령어 목록

```yaml
appId: com.example.myapp          # Android 패키지명 / iOS 번들 ID
name: 로그인 플로우                # (선택) 플로우 이름
- launchApp                        # 앱 시작
- tapOn: "Login"                   # 텍스트로 요소 탭
- tapOn: id: "email_input"         # id로 요소 탭
- inputText: "qa@example.com"     # 포커스된 필드에 입력
- tapOn: id: "password_input"
- inputText: "SuperSecret123"
- tapOn: "Sign In"
- assertVisible: "Welcome back"    # 화면에 요소가 보이는지 검증
```

**핵심 문법 규칙**

- `appId`는 Android에서는 패키지명(`com.example.myapp`), iOS에서는 번들 식별자(`com.example.myapp`)다.
- `---`는 헤더와 명령어 목록을 구분하는 YAML 문서 구분자다.
- 명령어는 리스트(`-`)로 위에서 아래로 순차 실행된다.
- 들여쓰기는 **2칸 공백**을 표준으로 쓴다. YAML은 들여쓰기에 민감하므로 탭을 섞지 말 것.
- 명령어마다 자동 대기·재시도가 내장되어 있어 `sleep`을 쓸 필요가 없다.

### 1-1. 자주 쓰는 명령어 요약

| 명령어 | 용도 | 예시 |
|--------|------|------|
| `launchApp` | 앱 시작(또는 재시작) | `- launchApp` |
| `tapOn` | 요소 탭 | `- tapOn: "Sign In"` |
| `inputText` | 포커스된 필드에 입력 | `- inputText: "hello"` |
| `assertVisible` | 요소가 화면에 보이는지 검증 | `- assertVisible: "Welcome"` |
| `assertNotVisible` | 요소가 화면에 없는지 검증 | `- assertNotVisible: "Error"` |
| `scroll` | 뷰 스크롤 | `- scroll` |
| `swipe` | 방향 스와이프 | `- swipe: { direction: LEFT }` |
| `back` | 시스템 뒤로가기 | `- back` |
| `takeScreenshot` | 스크린샷 캡처 | `- takeScreenshot: home` |
| `runFlow` | 하위 플로우 실행 | `- runFlow: login-flow.yaml` |
| `repeat` | 반복 실행 | `- repeat: { times: 3, commands: [...] }` |
| `runScript` / `evalScript` | JS 실행 | `- runScript: generate-user.js` |


## 2. 선택자(Selector): 안정적인 요소 찾기

Maestro는 요소를 찾는 **세 가지 기본 방식**을 제공한다. 어떤 선택자를 쓰느냐가 플로우의 안정성을 좌우한다.

### 2-1. 텍스트(text) — 가장 단순하지만 깨지기 쉬움

사용자가 보는 텍스트로 요소를 찾는다. 버튼·라벨에 적합하지만, **카피가 바뀌거나 다국어(localization)가 적용되면 깨진다.**

```yaml
- tapOn: "Add to cart"
```

### 2-2. id — 가장 안정적 (권장)

개발자가 요소에 설정한 **accessibility identifier / resource id / test id**로 찾는다. 내가 통제할 수 있는 요소라면 무조건 id를 우선하자.

```yaml
- tapOn: id: "add_to_cart_button"
```

### 2-3. index — 중복 요소 구분용 (가급적 지양)

같은 선택자에 여러 요소가 매칭될 때 **0부터 시작하는 위치(index)**로 구분한다. 레이아웃 순서가 바뀌면 깨지므로 가장 취약하다. 꼭 필요할 때만 쓴다.

```yaml
- tapOn:
    text: "Buy now"
    index: 1
```

### 2-4. 선택자 조합과 수식어

여러 매처(matcher)를 조합하고 수식어를 붙일 수 있다.

```yaml
# 텍스트 + enabled 수식어 (비활성 요소 건너뛰기)
- tapOn:
    text: "Submit"
    enabled: true

# 정규식(regex) — 동적 콘텐츠 매칭
- assertVisible:
    text: "Order #[0-9]+ confirmed"
```

### 2-5. 선택자 안정성 요약표

| 선택자 형태 | 안정성 | 사용 시점 |
|-------------|--------|-----------|
| `text` | 중간 | 카피가 고정된 버튼/라벨 |
| `id` | 높음 | test id가 있는 모든 요소 (권장) |
| `index` | 낮음 | 중복 매칭 구분용 |
| `text` + `enabled: true` | 높음 | 비활성 요소 건너뛰기 |
| `text` 정규식 | 중간 | "Item 42" 같은 동적 콘텐츠 |

**실무 팁:** 안정적인 선택자를 고르는 가장 빠른 방법은 **Maestro Studio**의 요소 검사기(element inspector)를 쓰는 것이다. 앱 요소를 클릭하면 해당 요소의 id·텍스트·accessibility 속성이 표시되고, 그에 맞는 명령어 제안까지 복사할 수 있다. 추측으로 선택자를 쓰지 말고 Studio로 확인하자.


## 3. Assertion(검증) 작성

검증은 테스트의 핵심이다. Maestro는 선언적으로 "이 요소가 보이는가 / 보이지 않는가"를 검증한다.

### 3-1. assertVisible / assertNotVisible

```yaml
- assertVisible: "Welcome back"          # 성공 화면 확인
- assertNotVisible: "Error"              # 에러가 없는지 확인
- assertVisible:
    id: "order_confirmation_badge"       # id로 검증
- assertVisible:
    text: "Order #[0-9]+ confirmed"     # 정규식으로 동적 값 검증
```

### 3-2. assertTrue (JS 표현식 검증)

더 복잡한 조건(값 비교, 계산 결과)은 `assertTrue`로 JS 표현식을 평가한다.

```yaml
- assertTrue: ${output.total > 0}        # output 변수 값 검증
- assertTrue: ${output.items.length == 3}
```

### 3-3. 검증 작성 원칙

- **성공 경로와 실패 경로를 모두 검증**한다. (성공 화면 + 에러 메시지 부재)
- **동적 값은 정규식**으로 매칭한다. (주문번호, 타임스탬프 등)
- **검증은 플로우 끝에 모으기보다** 각 단계 직후에 배치해 어디서 깨졌는지 즉시 파악한다.


## 4. 스크린샷 캡처와 활용

### 4-1. takeScreenshot

원하는 시점에 스크린샷을 캡처한다. 파일명을 지정하면 그 이름으로 저장된다.

```yaml
- takeScreenshot: home_screen
- takeScreenshot: checkout_confirm
```

### 4-2. 실패 시 자동 스크린샷

Maestro는 **테스트 실패 시 자동으로 화면 상태를 캡처**한다. 실패 지점을 시각적으로 바로 확인할 수 있어 디버깅이 빠르다. 별도 설정 없이 기본 동작이다.

### 4-3. 스크린샷 활용 시나리오

- **회귀 확인**: 주요 화면(홈, 장바구니, 결제 완료)을 캡처해 시각적 회귀 테스트의 기초 자료로 활용.
- **디버깅**: 실패 시 자동 캡처된 스크린샷으로 "Maestro가 어떤 요소를 찾지 못했는지"를 화면에서 직접 확인.
- **문서화**: 스크린샷을 테스트 리포트에 첨부해 QA·개발자 간 커뮤니케이션에 활용.


## 5. 하위 플로우(Subflow)로 재사용성 높이기

하위 플로우는 Maestro의 **Page Object 패턴에 해당하는 재사용 단위**다. 로그인·검색·내비게이션 같은 공통 동작을 별도 파일로 추출하고 `runFlow`로 호출한다.

### 5-1. 기본 하위 플로우

```yaml
# login-subflow.yaml
appId: com.example.myapp
- tapOn: id: "email_input"
- inputText: ${EMAIL}
- tapOn: id: "password_input"
- inputText: ${PASSWORD}
- tapOn: "Sign In"
```

```yaml
# checkout.yaml (메인 플로우)
appId: com.example.myapp
- launchApp
- runFlow:
    file: login-subflow.yaml
    env:
      EMAIL: "qa@example.com"
      PASSWORD: "secret"
- tapOn: "Checkout"
- assertVisible: "Order confirmed"
```

**핵심:** `runFlow`의 `env`로 하위 플로우에 환경 변수를 전달하고, 하위 플로우 안에서 `${EMAIL}`처럼 참조한다. 이렇게 하면 **하나의 로그인 플로우를 여러 테스트에서 파라미터만 바꿔 재사용**할 수 있다.

### 5-2. 파라미터화된 하위 플로우 (데이터 주도 테스트)

`env` 블록을 플로우 헤더에 정의하면 기본값을 줄 수 있고, CLI에서 `--env` 파일로 덮어쓸 수 있다.

```yaml
# login-flow.yaml
appId: com.yourapp.android
env:
  username: "test_user@example.com"
  password: "secure_password"
  expected_welcome: "Welcome, Test User"
- launchApp
- tapOn: "Login"
- inputText:
    text: ${username}
    index: 0
- inputText:
    text: ${password}
    index: 1
- tapOn: "Sign In"
- assertVisible: ${expected_welcome}
```

```bash
# 환경별 파라미터 파일로 실행
maestro test login-flow.yaml --env admin-params.yaml
```

### 5-3. 모듈식 플로우 조합 예시

```yaml
# complete-shopping-flow.yaml
appId: com.yourapp.android
- runFlow: login-flow.yaml
- runFlow: product-search.yaml
- tapOn: "Add to Cart"
- assertVisible: "Item Added"
```

**이점:** 로그인 프로세스가 바뀌면 `login-flow.yaml` 하나만 수정하면 된다. 이 파일을 쓰는 모든 테스트에 자동 반영된다. 중복 제거와 유지보수 비용 절감이 핵심 가치다.

### 5-4. 하위 플로우 설계 원칙

- **자주 쓰는 사용자 동작**을 모듈화한다. (로그인, 메뉴 내비게이션, 폼 제출, 결제)
- **파일명을 목적에 맞게** 짓는다. (`flow1.yaml` 대신 `user-registration-flow.yaml`)
- **주석**으로 복잡한 로직이나 특이한 선택자를 설명한다.
- **기본값**을 넣어 파라미터 누락 시에도 동작하게 한다.
- **2칸 들여쓰기**를 일관되게 유지한다.


## 6. 동적 데이터: JavaScript 연동

선언적 YAML만으로 부족할 때(동적 테스트 데이터 생성, 값 계산, HTTP 호출로 상태 시드) JS로 전환할 수 있다.

### 6-1. evalScript (인라인 표현식)

```yaml
appId: com.example.myapp
- launchApp
- evalScript: ${output.timestamp = Date.now()}
- inputText: ${"user_" + output.timestamp + "@example.com"}
```

### 6-2. runScript (JS 파일 실행)

```javascript
// generate-user.js
const id = Math.floor(Math.random() * 100000);
output.email = `qa+${id}@example.com`;
output.password = `Pass${id}!aB`;
```

```yaml
appId: com.example.myapp
- launchApp
- runScript: generate-user.js
- tapOn: id: "email_input"
- inputText: ${output.email}
- tapOn: id: "password_input"
- inputText: ${output.password}
- tapOn: "Register"
- assertVisible: "Welcome"
```

**핵심:** JS에서 `output` 객체에 할당한 값은 이후 YAML 명령어에서 `${output.xxx}`로 참조할 수 있다. 동적 데이터 생성과 상태 시드에 유용하다.


## 7. 조건부·반복 (플로우 작성에서 자주 쓰는 제어)

### 7-1. 조건부 실행 (when)

특정 조건이 성립할 때만 명령을 실행한다. 예: 알림 권한 팝업이 뜰 때만 "Allow" 탭.

```yaml
- runFlow:
    when:
      visible: "Allow notifications?"
    commands:
      - tapOn: "Allow"
```

### 7-2. 반복 (repeat)

```yaml
# 정해진 횟수 반복
- repeat:
    times: 3
    commands:
      - tapOn: "Load more"
      - scroll

# 조건이 참인 동안 반복
- repeat:
    while:
      visible: "Next"
    commands:
      - tapOn: "Next"
```


## 8. 실무 적용 체크리스트

플로우를 작성할 때 다음을 점검하면 안정적이고 유지보수 가능한 테스트가 된다.

1. **id 선택자 우선** — 통제 가능한 요소는 text 대신 id를 쓴다.
2. **명시적 sleep 금지** — Maestro의 자동 대기·재시도를 신뢰한다.
3. **동적 값은 정규식** — 주문번호·타임스탬프는 `text: "Order #[0-9]+"` 형태로.
4. **공통 동작은 하위 플로우로** — 로그인·검색·결제를 `runFlow`로 재사용.
5. **파라미터화** — `env`와 `--env` 파일로 환경별(스테이징/프로덕션) 데이터 분리.
6. **성공·실패 경로 모두 검증** — `assertVisible` + `assertNotVisible` 병행.
7. **Studio로 선택자 확인** — 추측 대신 요소 검사기로 정확한 id/속성 파악.
8. **2칸 들여쓰기 + 주석** — 팀 협업을 위해 가독성 유지.
9. **스크린샷 활용** — 주요 화면 캡처 + 실패 시 자동 캡처로 디버깅.
10. **JS는 꼭 필요할 때만** — 동적 데이터·상태 시드에만 `runScript`/`evalScript` 사용.


## 9. 요약

Maestro 플로우 작성의 핵심은 **선언적 YAML + 자동 대기/재시도 + 재사용 가능한 하위 플로우**다.

- **구조**: `appId` + `---` + 명령어 리스트. 2칸 들여쓰기.
- **선택자**: `id`(가장 안정) > `text`(중간) > `index`(취약). 정규식·`enabled` 수식어 지원.
- **검증**: `assertVisible`/`assertNotVisible`/`assertTrue`. 동적 값은 정규식.
- **스크린샷**: `takeScreenshot` + 실패 시 자동 캡처.
- **하위 플로우**: `runFlow` + `env`로 파라미터화. Page Object 패턴의 대안.
- **동적 데이터**: `runScript`/`evalScript`로 JS 연동, `output` 객체로 값 전달.
- **제어**: `when`(조건부), `repeat`(반복).

이 원칙을 지키면 짧고, 읽기 쉽고, flaky하지 않고, 유지보수 비용이 낮은 E2E 테스트 스위트를 구축할 수 있다.


*작성: Hermes Research Orchestrator · 2026-08-31 · Track 1 (Maestro 플로우 작성)*
