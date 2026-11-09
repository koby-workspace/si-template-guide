# Part 14. 백엔드 연결과 배포

> **이 Part의 목표**: 진짜 백엔드 서버가 생겼을 때 어떻게 옮겨 가는지, 백엔드 없이 고객에게 시연하는 법, 그리고 배포할 때 꼭 알아야 할 것을 정리합니다.
> 마지막으로 이 템플릿을 새 프로젝트에 가져갈 때 볼 README를 씁니다.

---

## Step 14-1. 개발 서버 프록시

> 🎯 **할 일**: 개발 중에 진짜 백엔드(예: `http://localhost:8080`)로 요청을 전달하는 설정을 넣습니다.

### 원리: CORS와 프록시

화면은 `http://localhost:5173`, 백엔드는 `http://localhost:8080`에서 돕니다. 포트가 다르면 브라우저는 **다른 사이트(출처, origin)** 로 봅니다.
보안 규칙(CORS) 때문에, 백엔드가 허락하지 않으면 브라우저가 응답을 막습니다.

**프록시**는 이 문제를 피하는 가장 쉬운 방법입니다. 브라우저는 같은 주소(`localhost:5173/api/...`)로 요청하고, Vite 개발 서버가 대신 백엔드에 전달합니다.

```
브라우저 ─(같은 출처)─→ Vite 개발 서버 :5173 ─(서버끼리, CORS 없음)─→ 백엔드 :8080
```

운영에서는 보통 Nginx 같은 웹 서버가 같은 역할을 합니다. (Step 14-4)

### 따라 하기

백엔드 주소는 사람마다 다를 수 있으니 환경 변수로 둡니다. `vite.config.ts`는 Node에서 실행되므로 `loadEnv`로 `.env`를 직접 읽습니다.

📄 **`vite.config.ts`** (전체 교체)

```ts
import react from "@vitejs/plugin-react";
import { defineConfig, loadEnv } from "vite";

// https://vite.dev/config/
export default defineConfig(({ mode }) => {
  // vite.config.ts는 브라우저가 아니라 Node에서 실행된다. import.meta.env 대신 loadEnv로 .env를 읽는다.
  const env = loadEnv(mode, process.cwd());

  return {
    plugins: [react()],
    resolve: {
      tsconfigPaths: true,
    },
    server: {
      proxy: {
        // /api로 시작하는 요청을 백엔드로 전달한다. (가짜 서버가 켜져 있으면 MSW가 먼저 가로챈다)
        "/api": {
          target: env.VITE_PROXY_TARGET || "http://localhost:8080",
          changeOrigin: true,
        },
      },
    },
  };
});
```

✏️ **`.env`** — 프록시 대상을 추가합니다.

```
VITE_MOCK_DELAY=300
```

바꿀 코드

```
VITE_MOCK_DELAY=300
# 개발 서버 프록시가 /api 요청을 전달할 백엔드 주소 (Step 14-1)
VITE_PROXY_TARGET=http://localhost:8080
```

> 💡 **팁** `VITE_PROXY_TARGET`은 개발 서버 설정에만 쓰이고 화면 코드에서는 읽지 않으므로 `env.d.ts`에 넣지 않아도 됩니다.
> 백엔드 주소가 PC마다 다르면 각자 `.env.local`에 적습니다.

### 백엔드로 옮겨 가는 순서

한꺼번에 옮기지 않습니다. **완성된 API부터 하나씩** 옮깁니다.

```
① 백엔드 API 명세(Swagger 등)와 Part 3의 규약을 비교한다
   → 응답 모양이 다르면 httpClient.ts에서, URL이 다르면 각 기능의 API 함수에서 맞춘다 (Step 14-3)
② 백엔드를 켜고 npm run dev
③ 완성된 API의 MSW 핸들러를 지운다(또는 주석 처리)
   → onUnhandledFrame: "bypass" 덕분에 그 요청만 프록시를 타고 진짜 서버로 간다
④ 전부 옮겨지면 .env.development의 VITE_USE_MOCK=false
⑤ src/mocks 폴더는 지우지 않는다 → 시연 모드(Step 14-2)와 나중에 테스트에서 다시 쓴다
```

### 확인하기

- [ ] `npm run dev`가 이전과 똑같이 동작합니다. (가짜 서버가 켜져 있어서 프록시까지 가지 않음)
- [ ] (선택) `.env.local`에 `VITE_USE_MOCK=false`를 넣고 개발 서버를 다시 켜면, 백엔드가 없으니 로그인에서 "잠시 후 다시 시도해 주세요."가 나옵니다.
      F12 → Network에서 `login` 요청이 `502 Bad Gateway`이고, 터미널에 `http proxy error`가 찍히면 프록시가 동작한 것입니다. 확인 후 `.env.local`을 지웁니다.

✅ **커밋**

```powershell
git add .
git commit -m "chore: 개발 서버 API 프록시 설정"
```

---

## Step 14-2. 백엔드 없이 시연하는 빌드

> 🎯 **할 일**: 가짜 서버를 켠 채로 배포 파일을 만드는 "시연 모드"를 추가합니다. 백엔드가 준비되기 전에 고객에게 화면을 보여 줄 때 씁니다.

### 원리: 빌드 모드

Vite는 `--mode 이름`을 주면 `.env.이름` 파일을 읽습니다. 시연용 모드에서 `VITE_USE_MOCK=true`로 두면, 운영 빌드처럼 최적화된 파일이지만 가짜 서버가 들어 있습니다.

| 명령                 | 읽는 파일                       | 가짜 서버 |
| -------------------- | ------------------------------- | --------- |
| `npm run dev`        | `.env`, `.env.development`      | 켬        |
| `npm run build`      | `.env`, `.env.production`(없음) | 끔        |
| `npm run build:demo` | `.env`, `.env.demo`             | 켬        |

### 따라 하기

📄 **`.env.demo`** (새 파일)

```
# 백엔드 없이 시연할 때. npm run build:demo
VITE_USE_MOCK=true
VITE_APP_TITLE=SI Template (Demo)
```

✏️ **`package.json`** — 실행 명령을 추가합니다.

```json
    "build": "tsc -b && vite build",
```

바꿀 코드

```json
    "build": "tsc -b && vite build",
    "build:demo": "tsc -b && vite build --mode demo",
```

### 확인하기

- [ ] `npm run build:demo` → `npm run preview` → `http://localhost:4173`에서 로그인·메뉴·탭이 전부 동작합니다. 백엔드가 없는데도!
- [ ] 반대로 `npm run build` → `npm run preview`는 로그인이 실패합니다("잠시 후 다시 시도해 주세요."). 운영 빌드에는 가짜 서버가 없고, 프록시가 가리키는 백엔드도 없기 때문입니다.

> ⚠️ **주의** 시연 빌드에는 테스트 계정 정보와 가짜 데이터가 들어 있습니다. **고객 운영 서버에 올리면 안 됩니다.** 시연 전용 주소에만 올립니다.
> 로그인 화면의 테스트 계정 버튼은 `import.meta.env.DEV` 조건이라 시연 빌드에서도 보이지 않습니다. 계정은 따로 알려 줍니다.

✅ **커밋**

```powershell
git add .
git commit -m "chore: 백엔드 없는 시연용 빌드 모드 추가"
```

---

## Step 14-3. 서버 응답 모양이 다를 때 (읽기만)

> 🎯 **할 일**: 실제 백엔드의 규약이 Part 3에서 정한 것과 다를 때 어디를 고치는지 알아 둡니다. 이 Step은 코드를 입력하지 않습니다.

계층을 나눠 둔 덕분에, 대부분 **`httpClient.ts` 한 파일**에서 끝납니다.

### 경우 1. 응답을 `{ code, message, data }`로 감싸는 규약

SI 현장에서 아주 흔합니다. `httpClient`의 `request` 함수에서 `data`만 꺼내 주면 위층은 그대로입니다.

```ts
async function request<T>(method: string, url: string, options: RequestOptions = {}): Promise<T> {
  const response = await send(method, url, options);
  if (response.status === 204) return undefined as T;
  const body = await response.json(); // { code: "0000", message: "성공", data: {...} }
  if (body.code !== "0000") {
    // HTTP 200인데 업무 오류인 규약도 있다. 이때도 ApiError로 바꿔 던지면 화면의 오류 처리가 그대로 동작한다.
    throw new ApiError(400, { status: 400, detail: body.message, code: body.code });
  }
  return body.data as T;
}
```

### 경우 2. 페이징 모양이 다를 때

예를 들어 `{ list: [...], totalCount: 120 }`이라면 각 기능의 API 함수에서 `PageResponse` 모양으로 바꿔 줍니다. 화면은 그대로입니다.

```ts
list: async (condition) => {
  const body = await httpClient.get<{ list: User[]; totalCount: number }>("/users", { params: condition });
  return { content: body.list, page: { number: condition.page, size: condition.size, totalElements: body.totalCount, totalPages: Math.ceil(body.totalCount / condition.size) } };
},
```

### 경우 3. 토큰 대신 쿠키 세션

`send` 함수에서 `Authorization` 헤더를 빼고 `credentials: "include"`를 넣습니다. 로그인 화면은 토큰 대신 "성공 여부"만 저장하게 바꿉니다.

### 경우 4. 팀 표준이 axios

`send` 함수 안만 axios로 바꿉니다. `httpClient.get/post/...`의 모양은 그대로 둡니다.

### 경우 5. 백엔드가 Swagger(OpenAPI)를 제공

`openapi-typescript`로 타입을 자동 생성하거나, `orval`로 API 함수·Query 훅·MSW 핸들러까지 생성할 수 있습니다. 손으로 타입을 맞추는 일이 사라집니다.
프로젝트 규모가 크면 꼭 검토하세요.

---

## Step 14-4. 배포

> 🎯 **할 일**: 빌드 결과물을 웹 서버에 올릴 때 꼭 필요한 설정을 알아 두고, 하위 경로 배포를 지원하게 합니다.

### 빌드 결과물

`npm run build`를 하면 `dist` 폴더에 정적 파일만 생깁니다. 이 폴더를 아무 웹 서버(Nginx, Apache, IIS, Spring Boot의 static 폴더 등)에 올리면 됩니다.

```
dist/
├─ index.html              ← 매번 새로 받게 (캐시 금지)
├─ mockServiceWorker.js    ← (public 폴더 파일. 운영에서는 쓰이지 않음)
└─ assets/
   ├─ index-a1b2c3.js      ← 파일 이름에 내용 해시가 붙어 있다 → 오래 캐시해도 안전
   └─ UsersPage-d4e5f6.js  ← 화면별로 나뉜 파일 (코드 분할)
```

### 반드시 필요한 설정 1: SPA 새로고침 404

`/system/users`에서 새로고침하면, 웹 서버는 `system/users`라는 **파일**을 찾다가 404를 냅니다. 그런 파일은 없고, 모든 주소는 `index.html` 하나가 처리하기 때문입니다.
그래서 **"없는 파일이면 `index.html`을 주라"** 는 설정(SPA fallback)이 반드시 필요합니다.

**Nginx**

```nginx
server {
  listen 80;
  root /var/www/si-template;

  location / {
    try_files $uri $uri/ /index.html;   # 없는 파일이면 index.html
  }

  location /api/ {
    proxy_pass http://backend:8080;      # Step 14-1의 프록시와 같은 역할
  }

  location = /index.html {
    add_header Cache-Control "no-cache"; # index.html은 항상 새로 받는다
  }

  location /assets/ {
    add_header Cache-Control "public, max-age=31536000, immutable"; # 해시가 붙은 파일은 1년 캐시
  }
}
```

**Spring Boot에 같이 넣는 경우**: `dist` 내용을 `src/main/resources/static`에 복사하고, API가 아닌 경로를 `index.html`로 넘기는 컨트롤러를 둡니다.

```java
@Controller
public class SpaController {
  // 점(.)이 없는 경로 = 파일이 아닌 화면 주소 → index.html
  @GetMapping({"/{path:[^\\.]*}", "/**/{path:[^\\.]*}"})
  public String forward() {
    return "forward:/index.html";
  }
}
```

### 반드시 필요한 설정 2: 하위 경로에 배포할 때

`https://회사.com/`이 아니라 `https://회사.com/admin/`처럼 하위 경로에 올릴 때가 있습니다.
Vite에 `base`를, 라우터에 `basename`을 알려 줘야 합니다. 라우터는 Vite가 넣어 주는 `import.meta.env.BASE_URL`을 그대로 쓰게 해 두면, 설정은 빌드 명령 한 곳에만 남습니다.

✏️ **`src/app/AppProviders.tsx`** — 라우터가 배포 경로를 알게 합니다.

```tsx
        <BrowserRouter>{children}</BrowserRouter>
```

바꿀 코드

```tsx
        {/* 하위 경로 배포(vite build --base=/admin/) 시 라우터도 그 경로를 기준으로 동작한다 */}
        <BrowserRouter basename={import.meta.env.BASE_URL}>{children}</BrowserRouter>
```

하위 경로로 빌드할 때는 이렇게 합니다.

```powershell
npx vite build --base=/admin/
```

> 💡 **팁** `import.meta.env.BASE_URL`은 기본값이 `/`라서, 지금처럼 루트에 배포할 때는 아무것도 바뀌지 않습니다.

### 배포 전 체크리스트

| 항목                       | 확인                                                               |
| -------------------------- | ------------------------------------------------------------------ |
| 가짜 서버 꺼짐             | `.env.production` 또는 `.env`의 `VITE_USE_MOCK=false`              |
| 비밀값 없음                | `VITE_` 변수에 비밀번호·키가 없는가 (빌드 파일에 그대로 박힌다)    |
| SPA fallback               | 아무 화면에서 새로고침해도 404가 아닌가                            |
| 캐시                       | `index.html`은 no-cache, `assets`는 장기 캐시                      |
| API 프록시 / CORS          | `/api`가 백엔드로 전달되는가                                       |
| 타입·린트                  | `npm run build`, `npm run lint`가 깨끗한가                         |
| 개발 전용 기능             | 개발자 패널·테스트 계정 버튼이 운영 빌드에 없는가 (`DEV` 조건)     |

### 확인하기

- [ ] `npm run build` → `npm run preview` → 아무 화면이나 열고 `F5`. Vite preview 서버는 SPA fallback이 기본으로 켜져 있어서 404가 나지 않습니다.
- [ ] `dist/assets`에 `DevPanel` 파일이 없습니다. (`Get-ChildItem dist/assets -Name | Select-String Dev`)

✅ **커밋**

```powershell
git add .
git commit -m "feat: 하위 경로 배포를 위한 라우터 basename 설정"
```

---

## Step 14-5. README — 템플릿 사용 설명서

> 🎯 **할 일**: 이 템플릿을 받은 사람(미래의 나 포함)이 가장 먼저 볼 README를 씁니다.

### 따라 하기

📄 **`README.md`** (전체 교체)

````markdown
# SI Template

React + TypeScript 기반의 업무 시스템(SI) 프론트엔드 템플릿입니다.
백엔드 없이도 가짜 서버(MSW)로 모든 화면이 동작하고, 백엔드가 준비되면 환경 변수와 API 함수만 바꿔 연결합니다.

## 기능

- MDI 탭 (여러 화면 동시 사용, 탭 전환 시 상태 유지, 고정 탭 없음)
- 데이터로 그리는 메뉴 (최대 3단계), 메뉴 관리에서 화면 파일 매핑
- 역할 기반 권한 (메뉴 접근 + 버튼 단위), 권한 관리 화면
- 다국어 (한국어 / English), 라이트·다크·시스템 모드, 테마 5종, 밀도
- 공지사항 (서식 에디터, XSS 방어), 알림 (미확인 표시, 이벤트 규칙)
- 공통코드, 서버 페이징, 엑셀 다운로드, 파일 업로드, 메뉴 검색(Ctrl+K), 세션 만료, 개발자 패널

## 시작하기

```powershell
npm install
npm run dev
```

http://localhost:5173 — 테스트 계정: `admin`/`admin1234`, `manager`/`manager1234`, `user`/`user1234`

| 명령                 | 설명                     |
| -------------------- | ------------------------ |
| `npm run dev`        | 개발 서버 (가짜 서버 켬) |
| `npm run build`      | 운영 빌드 (가짜 서버 끔) |
| `npm run build:demo` | 시연 빌드 (가짜 서버 켬) |
| `npm run preview`    | 빌드 결과 미리보기       |
| `npm run lint`       | ESLint                   |
| `npm run format`     | Prettier                 |

## 폴더 구조

```
src/
├─ app/        조립: Provider, 라우트, 레이아웃(헤더·사이드바·탭)
├─ features/   업무 기능. 화면은 features/<기능>/pages/<이름>Page.tsx
├─ shared/     공통: API, 인증·권한, 메뉴, 탭, 테마, 다국어, 공통 UI
└─ mocks/      가짜 서버 (MSW). 앱 코드는 import 하지 않는다
```

의존 방향: `app → features → shared`. feature끼리는 import 하지 않는다.

## 새 화면 추가

1. `src/features/<기능>/types.ts` — 데이터 타입
2. `src/mocks/data/<기능>.ts`, `src/mocks/handlers/<기능>.ts` — 가짜 데이터·API (+ `mocks/handlers.ts`에 등록)
3. `src/features/<기능>/api/queries.ts` — 요청 함수와 Query 훅
4. `src/features/<기능>/pages/<이름>Page.tsx` — `features/users/pages/UsersPage.tsx`를 복사해서 시작 (default export)
5. `src/shared/i18n/locales/{ko,en}/<기능>.json` — 문구
6. 메뉴 관리에서 메뉴 등록 + 화면 파일 선택
7. 권한 관리에서 역할별 권한 부여

## 새 프로젝트에 가져갈 때 바꿀 것

- [ ] `.env`의 `VITE_APP_TITLE`, `index.html`의 파비콘
- [ ] `src/shared/theme/presets.ts`에 고객사 색 프리셋 추가, 기본값 변경
- [ ] `src/mocks/data/menus.ts` 메뉴 구성, `src/mocks/data/permissions.ts` 권한
- [ ] 쓰지 않는 샘플 정리: `src/features/samples`
- [ ] 백엔드 규약 확인 후 `src/shared/api/httpClient.ts` 조정
- [ ] 토큰 저장 방식(httpOnly 쿠키 권장), 세션 만료 시간(`VITE_SESSION_TIMEOUT_MINUTES`)

## 백엔드 연결

1. `.env.local`에 `VITE_PROXY_TARGET=http://백엔드주소`
2. 완성된 API의 MSW 핸들러를 지운다 → 그 요청만 백엔드로 간다
3. 전부 옮기면 `.env.development`의 `VITE_USE_MOCK=false`

## 배포

`dist` 폴더를 웹 서버에 올린다. **SPA fallback**(없는 경로 → `index.html`) 설정이 반드시 필요하다.
하위 경로 배포는 `vite build --base=/경로/`.
````

> 💡 **팁** README 자체에 코드 블록(```)이 들어 있어서, 가이드에서는 바깥을 백틱 4개(````)로 감쌌습니다. 여러분의 파일에는 안쪽 내용만 넣으면 됩니다.

✅ **커밋**

```powershell
git add .
git commit -m "docs: README에 템플릿 사용법 추가"
```

---

## 🏁 완주를 축하합니다

처음 `npm create vite`를 쳤던 빈 폴더가, 어느 SI 프로젝트에 가져가도 바로 시작할 수 있는 템플릿이 되었습니다.
`git log --oneline | Measure-Object -Line`을 쳐 보세요. 작은 커밋들이 학습 일지처럼 쌓여 있습니다.

### 다음에 공부할 것

| 주제                 | 내용                                                                                       | 시작할 때                               |
| -------------------- | ------------------------------------------------------------------------------------------ | --------------------------------------- |
| **테스트**           | Vitest + Testing Library(단위·컴포넌트), Playwright(E2E). `src/mocks` 핸들러를 그대로 재사용 | 화면이 늘고 공통 로직이 복잡해질 때     |
| 첫 테스트 대상       | `shared/lib/format.ts`, `validators.ts`, `hangul.ts` (순수 함수라 가장 쉽다)               | 테스트를 시작하는 첫날                  |
| 커밋 규칙 자동 검사  | commitlint로 type 8개만 허용, lint-staged로 커밋 전 lint·format                            | 협업자가 생길 때                        |
| React Compiler       | `useMemo`·`useCallback`을 컴파일러가 자동 처리                                             | 성능 문제가 체감될 때                   |
| OpenAPI 코드 생성    | 백엔드 Swagger로 타입·API 함수 자동 생성 (`openapi-typescript`, `orval`)                   | 백엔드 API 명세가 나올 때               |
| 웹 접근성            | 공공 SI의 웹 접근성 인증(KWCAG) 대비. 키보드 조작, 대비, 화면 낭독기                        | 공공 프로젝트에 적용할 때               |
| 탭 드래그 정렬       | `@dnd-kit` + antd Tabs                                                                     | 사용자 요청이 있을 때                   |
| `<Activity>`         | 숨은 탭을 React 공식 기능으로 (Part 7 팁)                                                  | 탭이 많아 느려질 때                     |

막혔을 때는 [부록 B. 문제 해결과 용어집](appendix-b-troubleshooting.md)을, 단축키는 [부록 A](appendix-a-shortcuts.md)를 보세요.
