# Single-SPA vs Module Federation: 진짜 핵심 차이

## 1줄 요약

| 기술                  | 핵심 정체성                                                         |
| --------------------- | ------------------------------------------------------------------- |
| **Single-SPA**        | **애플리케이션 오케스트레이터** - 여러 독립 앱을 한 페이지에서 실행 |
| **Module Federation** | **모듈 공유 시스템** - JavaScript 코드 조각을 런타임에 공유         |

---

## 핵심 차이 (3가지만)

### 1. 통합 레벨이 다르다

#### Single-SPA: Application-Level

```
┌─────────────────────────────────────┐
│        Browser (1 Page)             │
│                                     │
│  ┌──────────┐  ┌──────────┐       │
│  │ Vue App  │  │ React App│       │  ← 완전히 독립된 애플리케이션
│  │ (완전체) │  │ (완전체) │       │
│  │          │  │          │       │
│  │ ✓ Router │  │ ✓ Router │       │
│  │ ✓ Store  │  │ ✓ Store  │       │
│  │ ✓ UI     │  │ ✓ UI     │       │
│  └──────────┘  └──────────┘       │
│        ↑              ↑            │
│        └──── Root Config ──────┘   │  ← URL 기반 앱 전환
└─────────────────────────────────────┘

/auth   → Vue App 전체 실행
/main   → React App 전체 실행
```

**핵심**: 각 앱이 **독립적으로 실행되는 완전한 SPA**

#### Module Federation: Module-Level

```
┌─────────────────────────────────────┐
│         Main App (Host)             │
│                                     │
│  ┌───────────────────────────────┐ │
│  │                               │ │
│  │   <LoginForm />  ← Remote    │ │  ← 컴포넌트 조각만 가져옴
│  │   <Button />     ← Remote    │ │
│  │   <Header />     ← Local     │ │
│  │                               │ │
│  └───────────────────────────────┘ │
└─────────────────────────────────────┘
          ↓
    import('authApp/LoginForm')  ← 코드 조각만
```

**핵심**: **컴포넌트/함수 같은 코드 조각**을 공유

---

### 2. 목적이 다르다

#### Single-SPA: "마이크로 프론트엔드 아키텍처"

```
문제: 레거시 jQuery 앱 + 새로운 React 앱을 한 도메인에서 실행하고 싶다
해결: Single-SPA로 URL별로 다른 앱 실행

example.com/old    → jQuery 앱 (2015년 코드)
example.com/new    → React 앱 (2024년 코드)

- 점진적 마이그레이션
- 완전히 다른 기술 스택 공존
```

#### Module Federation: "코드 공유 + 중복 제거"

```
문제: 10개 앱에서 같은 LoginForm을 사용하는데 각각 번들에 포함됨
해결: Module Federation으로 한 번만 로딩

App A ────┐
App B ────┼──→ LoginForm (1번만 다운로드)
App C ────┘

- 번들 크기 감소
- 캐시 효율
- 중복 제거
```

---

### 3. 아키텍처 패턴이 다르다

#### Single-SPA: "URL 기반 앱 전환"

```typescript
// 각 앱은 코드 레벨에서 독립적
// 서로 import 불가능 (같은 브라우저 내에서 실행)

// auth-app (Vue)
createApp(AuthApp).mount("#auth");

// main-app (React)
ReactDOM.render(<MainApp />);

// Root Config가 URL 기반으로 앱 전환
registerApplication(
    "auth",
    () => import("auth-app"),
    (url) => url.startsWith("/auth") // /auth면 auth-app 실행
);
registerApplication(
    "main",
    () => import("main-app"),
    (url) => url.startsWith("/") // /면 main-app 실행
);
```

**패턴**: **URL로 앱 분리** (브라우저 내 코드 수준 분리)

#### Module Federation: "서비스 기반 컴포넌트 공유"

```typescript
// 각 앱이 기능을 "서비스"로 제공 (MSA의 API처럼)

// auth-app (서비스 제공자) = DDD의 Bounded Context
export { LoginForm, Button }; // exposes = API 엔드포인트

// main-app (서비스 소비자) = 다른 Bounded Context
import { LoginForm } from "authApp/LoginForm"; // Context 간 통신
```

**패턴**: **서비스 경계 기반 통합** (DDD Bounded Context + MSA API 호출)

---

## 실제 차이가 나타나는 시나리오

### 시나리오 1: LoginForm 컴포넌트 공유

#### Single-SPA 방식

```typescript
// 불가능!
// main-app에서 auth-app의 LoginForm을 직접 import할 수 없음

// 해결책: Auth 앱 전체를 iframe처럼 렌더링
registerApplication({
    name: "auth-widget",
    app: () => System.import("auth-app"),
    activeWhen: () => true, // 항상 렌더링
});
```

**결과**: Auth 앱 **전체**를 로딩해야 함 (불필요한 라우터, 스토어 포함)

#### Module Federation 방식

```typescript
// 가능!
import LoginForm from "authApp/LoginForm";

<LoginForm />; // 컴포넌트만 사용
```

**결과**: LoginForm **컴포넌트만** 로딩 (5KB)

---

### 시나리오 2: 같은 페이지에 2개 프레임워크

#### Single-SPA 방식

```
URL: /dashboard

┌──────────────────┐
│ Vue Header       │  ← Vue App 실행 중
├──────────────────┤
│ React Content    │  ← React App 실행 중
└──────────────────┘

자연스럽게 지원
registerApplication('header', vueApp, () => true);
registerApplication('content', reactApp, () => true);
```

#### Module Federation 방식

```
URL: /dashboard

┌──────────────────┐
│ Vue App          │
│                  │
│ <ReactContent /> │  ← Wrapper 필요
└──────────────────┘

Wrapper 코드 필요 (React → Vue 변환)
```

---

### 시나리오 3: 레거시 jQuery 코드 통합

#### Single-SPA 방식

```typescript
// 간단
export const mount = () => {
    $("#root").html("<div>jQuery App</div>");
};

registerApplication("legacy", jqueryApp, "/legacy");
```

#### Module Federation 방식

```typescript
// 거의 불가능
// jQuery는 ES Module이 아니므로 expose 불가
```

---

## 진짜 핵심 (기술적 차이)

### Single-SPA의 정체

```
URL-based Application Router
= iframe 대체품 (더 스마트한)

같은 브라우저에서 URL에 따라 다른 앱 실행
Root Config = URL 라우터

특징: 브라우저 내에서 앱 간 코드 수준 분리
     → 같은 브라우저, 같은 DOM, 같은 메모리 공간
```

### Module Federation의 정체

```
Runtime Service Integrator
= Frontend의 MSA (Micro-Services Architecture)
= Frontend의 DDD (Domain-Driven Design)

각 팀이 컴포넌트를 "서비스"로 배포
remoteEntry.js = API Gateway
exposes = API 엔드포인트
import = API 호출

각 앱 = Bounded Context (도메인 경계)
auth-app = Auth Context
main-app = Main Context
→ Context 간 명확한 경계와 계약 기반 통신

→ Backend MSA/DDD 철학을 Frontend에 적용한 아키텍처
```

---

## 언제 뭘 써야 하나?

### Single-SPA 사용 케이스

```
1. 레거시 통합 (jQuery → React 마이그레이션)
2. 완전히 다른 팀이 완전히 다른 앱 개발
3. URL 기반으로 다른 앱 보여주기
   /admin  → Admin App
   /user   → User App

예시: 기업 포털 (10년 된 jQuery + 신규 React)
```

### Module Federation 사용 케이스

```
1. 같은 프레임워크 사용
2. 컴포넌트/유틸 공유 필요
3. Monorepo 대체 (독립 배포 + 코드 공유)

예시: E-commerce (Product, Cart, Checkout 앱들이 Button 컴포넌트 공유)
```

---

## 최종 정리

|                   | Single-SPA            | Module Federation         |
| ----------------- | --------------------- | ------------------------- |
| **정체**          | 애플리케이션 컨테이너 | 모듈 공유 시스템          |
| **비유**          | Docker Compose        | NPM (런타임)              |
| **통합 단위**     | 전체 앱               | 컴포넌트/함수             |
| **사용 케이스**   | 다른 프레임워크 공존  | 같은 프레임워크 코드 공유 |
| **레거시 지원**   | 핵심 강점             | 불가능                    |
| **컴포넌트 공유** | 어려움                | 핵심 강점                 |

---

## 결론: 왜 둘 다 존재하는가?

**Single-SPA (2018)**:

-   문제: "10년 된 jQuery 앱을 어떻게 React로 점진적으로 마이그레이션?"
-   해결: "각 앱을 독립적으로 실행하되 한 페이지에서"

**Module Federation (2020)**:

-   문제: "10개 앱이 같은 컴포넌트를 중복해서 번들링하고 있음"
-   해결: "컴포넌트를 런타임에 공유하자"

**→ 전혀 다른 문제를 해결하기 위해 만들어진 기술**

---

## 당신의 프로젝트 (Vue 3)

```
레거시 있음? → NO
다른 프레임워크 필요? → NO
컴포넌트 공유 필요? → YES
독립 배포 필요? → YES

→ Module Federation
```
