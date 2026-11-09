# Part 3. 가짜 서버와 API 계층

> **이 Part의 목표**: 백엔드 없이도 "진짜 서버에 요청하는 코드"로 화면을 개발할 수 있게 합니다.
> 이 Part가 끝나면 브라우저 Network 탭에 진짜 요청이 찍히고, 화면 코드는 서버가 진짜인지 가짜인지 모릅니다.
> 나중에 서버를 연결할 때 **화면 코드는 한 줄도 바꾸지 않는 것**이 목표입니다.

---

## Step 3-1. 공통 HTTP 클라이언트

> 🎯 **할 일**: 앱의 모든 서버 요청이 지나가는 단 하나의 통로를 만듭니다.

### 큰 그림: 계층을 나눈다

서버 요청을 화면에서 바로 하지 않고 네 단계로 나눕니다.

```
화면 (UsersPage)
  ↓ useUserList(condition)        features/users/api/queries.ts   캐시 규칙
  ↓ usersApi.list(condition)      features/users/api/usersApi.ts  URL과 파라미터
  ↓ httpClient.get("/users", …)   shared/api/httpClient.ts        fetch, 토큰, 오류
  ↓ fetch("/api/users?name=김")
  ├─ [개발] MSW가 가로챔 → mocks/handlers/users.ts → mocks/db.ts
  └─ [운영] 진짜 백엔드 서버
```

왜 이렇게 나눌까요? 각 층이 **바로 아래층만** 알기 때문에, 무언가 바뀌면 한 층만 고치면 됩니다.

| 바뀌는 것               | 고치는 곳                                    |
| ----------------------- | -------------------------------------------- |
| 가짜 → 진짜 서버        | 환경 변수 `VITE_USE_MOCK=false` (코드 수정 0) |
| 응답 형식 (감싸기 여부) | `httpClient.ts` 한 곳                        |
| fetch → axios           | `httpClient.ts` 한 곳                        |
| 토큰 방식 (헤더 → 쿠키) | `httpClient.ts` 한 곳                        |
| 특정 API의 URL          | 해당 `xxxApi.ts` 한 곳                       |

이것이 요구사항 "서버 API는 추후에 연결 예정이라 나중에 쉽게 변경 가능하도록"의 답입니다.

### API 규약

서버가 없어도 **응답 모양은 미리 정해야** 합니다. 한국 SI 백엔드는 대부분 Spring Boot라서 Spring 기본 형식에 맞춥니다.
실제 프로젝트 규약이 달라도 `httpClient.ts`에서 흡수할 수 있습니다.

| 동작      | 메서드   | URL                  | 성공 상태 |
| --------- | -------- | -------------------- | --------- |
| 목록 조회 | `GET`    | `/api/users?name=김` | 200       |
| 단건 조회 | `GET`    | `/api/users/1`       | 200       |
| 등록      | `POST`   | `/api/users`         | 201       |
| 수정      | `PUT`    | `/api/users/1`       | 200       |
| 삭제      | `DELETE` | `/api/users/1`       | 204       |

성공 응답은 감싸지 않고 데이터를 그대로 줍니다. 오류는 표준 형식인 **RFC 9457 Problem Details**(Spring의 `ProblemDetail`)를 씁니다.

```json
{
  "status": 409,
  "title": "Conflict",
  "detail": "이미 사용 중인 이메일입니다.",
  "code": "USER_EMAIL_DUPLICATED",
  "errors": [{ "field": "email", "message": "이미 사용 중인 이메일입니다." }]
}
```

| 상태 | 의미             | 화면 처리                   |
| ---- | ---------------- | --------------------------- |
| 400  | 입력값 오류      | `errors`를 폼 입력칸에 표시 |
| 401  | 로그인 필요·만료 | 로그인 화면으로             |
| 403  | 권한 없음        | "권한이 없습니다" 알림      |
| 404  | 대상 없음        | 알림                        |
| 409  | 충돌(중복 등)    | 입력칸 또는 알림            |
| 500  | 서버 오류        | "잠시 후 다시 시도" 알림    |

### 따라 하기

📄 **`src/shared/api/types.ts`** (새 파일)

```ts
export type UseYn = "Y" | "N";

/** 서버 오류 응답 (RFC 9457 Problem Details) */
export interface ProblemDetail {
  status: number;
  title?: string;
  detail?: string;
  code?: string;
  errors?: { field: string; message: string }[];
}

/** 페이징 응답 (Spring Data PagedModel 형식). page.number는 0부터 시작한다. */
export interface PageResponse<T> {
  content: T[];
  page: { size: number; number: number; totalElements: number; totalPages: number };
}
```

📄 **`src/shared/api/httpClient.ts`** (새 파일)

```ts
import type { ProblemDetail } from "@/shared/api/types";

const BASE_URL = import.meta.env.VITE_API_BASE_URL;

// tsconfig의 erasableSyntaxOnly 때문에 constructor(public status) 같은 축약 문법은 쓸 수 없다. 필드를 직접 선언한다.
export class ApiError extends Error {
  readonly status: number;
  readonly problem: ProblemDetail;

  constructor(status: number, problem: ProblemDetail) {
    super(problem.detail ?? problem.title ?? `HTTP ${status}`);
    this.status = status;
    this.problem = problem;
  }
}

interface RequestOptions {
  params?: object;
  body?: unknown;
  signal?: AbortSignal;
}

// 값이 비어 있는 조건은 빼고 쿼리스트링을 만든다. { name: "김", role: undefined } → "?name=김"
function toQueryString(params: object = {}) {
  const search = new URLSearchParams();
  for (const [key, value] of Object.entries(params)) {
    if (value !== undefined && value !== null && value !== "") search.append(key, String(value));
  }
  const text = search.toString();
  return text ? `?${text}` : "";
}

async function request<T>(method: string, url: string, options: RequestOptions = {}): Promise<T> {
  const response = await fetch(BASE_URL + url + toQueryString(options.params), {
    method,
    headers: { "Content-Type": "application/json" },
    body: options.body === undefined ? undefined : JSON.stringify(options.body),
    signal: options.signal,
  });

  if (!response.ok) {
    // 서버가 JSON이 아닌 오류를 줄 수도 있다 (프록시의 502 등). 그래도 앱이 죽지 않게 상태코드만 담는다.
    const problem = await response.json().catch(() => ({ status: response.status }));
    throw new ApiError(response.status, problem);
  }
  if (response.status === 204) return undefined as T; // 204는 본문이 없다
  return response.json();
}

export const httpClient = {
  get: <T>(url: string, options?: RequestOptions) => request<T>("GET", url, options),
  post: <T>(url: string, body?: unknown) => request<T>("POST", url, { body }),
  put: <T>(url: string, body?: unknown) => request<T>("PUT", url, { body }),
  delete: <T = void>(url: string) => request<T>("DELETE", url),
};
```

### 코드 읽어 보기

**제네릭 함수 `request<T>`**: 응답의 타입은 호출하는 쪽만 압니다. 그래서 타입을 매개변수처럼 받습니다.
`httpClient.get<User[]>("/users")`라고 쓰면 결과가 `User[]`로 잡힙니다.

**`async`/`await`**: `fetch`는 시간이 걸리는 작업이라 결과 대신 "나중에 결과를 주겠다는 약속"(Promise)을 돌려줍니다.
`await`는 그 약속이 끝날 때까지 기다렸다가 결과를 꺼냅니다.

**`response.ok`**: `fetch`는 500 오류가 와도 **실패로 치지 않습니다.** 네트워크가 끊겼을 때만 실패합니다.
상태코드는 직접 확인해야 합니다. 초보자가 가장 많이 놓치는 부분입니다.

> 💡 **팁** 이름을 `http`가 아니라 `httpClient`로 지은 이유: 곧 쓸 MSW도 `http`라는 이름을 내보냅니다. 핸들러 파일에서 둘이 헷갈리지 않게 합니다.

> 💡 **팁** axios를 안 쓴 이유: 요즘 브라우저의 `fetch`와 얇은 래퍼면 충분하고, 의존성이 하나 줄어듭니다.
> 나중에 팀 표준이 axios여도 이 파일 내부만 바꾸면 됩니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 공통 HTTP 클라이언트 추가"
```

---

## Step 3-2. MSW 설치와 시작

> 🎯 **할 일**: 브라우저 안에 가짜 서버를 띄웁니다.

### 원리: Service Worker

**Service Worker**는 브라우저와 네트워크 사이에 서 있는 "중간 관리자" 스크립트입니다.
MSW(Mock Service Worker)는 이 관리자에게 "`/api/users`로 가는 요청은 내가 대신 답할게"라고 시킵니다.

```
fetch("/api/users")  →  Service Worker가 가로챔  →  MSW 핸들러가 응답  →  앱은 진짜 응답인 줄 앎
```

| 방식               | 장점                                                       | 단점                                   |
| ------------------ | ---------------------------------------------------------- | -------------------------------------- |
| 함수로 흉내        | 가장 단순                                                  | HTTP 계층이 없어서 서버 전환 시 재작성 |
| **MSW (채택)**     | 진짜 fetch가 나간다. 화면·API 코드가 실서버용과 똑같다     | Service Worker 개념을 알아야 한다      |
| json-server        | 명령 하나로 REST 서버                                      | 별도 프로세스, 권한 로직 표현이 어렵다 |

### 따라 하기

▶️ **터미널**

```powershell
npm install -D msw@3.0.4
npx msw init public --save
```

`npx msw init`은 `public/mockServiceWorker.js`를 만듭니다. 브라우저에 등록될 Service Worker 파일입니다.

> ⚠️ **주의** 이 파일은 **Git에 커밋합니다.** 앱이 실행될 때 필요하기 때문입니다. msw를 업데이트하면 `npx msw init`으로 다시 만듭니다.

자동 생성 파일이므로 ESLint 검사에서 뺍니다.

✏️ **`eslint.config.js`** — `public` 폴더를 검사에서 제외합니다.

```js
  globalIgnores(["dist"]),
```

바꿀 코드

```js
  globalIgnores(["dist", "public"]),
```

✅ **커밋**

```powershell
git add .
git commit -m "chore: msw 의존성 추가 및 서비스 워커 초기화"
```

가짜 서버 전용 도우미 함수를 만듭니다.

📄 **`src/mocks/utils.ts`** (새 파일)

```ts
import { HttpResponse } from "msw";

const BASE_URL = import.meta.env.VITE_API_BASE_URL;

export const MOCK_DELAY = Number(import.meta.env.VITE_MOCK_DELAY);

/** httpClient와 같은 규칙으로 주소를 만든다. 규칙이 다르면 요청이 핸들러에 걸리지 않는다. */
export const apiUrl = (path: string) => `${BASE_URL}${path}`;

/** Problem Details 형식의 오류 응답을 만든다. */
export function problem(status: number, detail: string, extra: object = {}) {
  return HttpResponse.json({ status, detail, ...extra }, { status });
}
```

📄 **`src/mocks/handlers.ts`** (새 파일)

```ts
import { delay, http } from "msw";
import { apiUrl, MOCK_DELAY } from "@/mocks/utils";

export const handlers = [
  // 모든 API에 지연을 넣어 로딩 화면을 확인할 수 있게 한다.
  // 아무것도 돌려주지 않는 핸들러는 다음 핸들러로 넘어간다.
  http.all(apiUrl("/*"), async () => {
    await delay(MOCK_DELAY);
  }),
];
```

📄 **`src/mocks/browser.ts`** (새 파일)

```ts
import { setupWorker } from "msw/browser";
import { handlers } from "@/mocks/handlers";

export const worker = setupWorker(...handlers);
```

이제 앱을 그리기 **전에** 가짜 서버를 켭니다.

✏️ **`src/main.tsx`** — 렌더 코드를 함수 안으로 옮깁니다.

```tsx
createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <AppProviders>
      <App />
    </AppProviders>
  </StrictMode>,
);
```

바꿀 코드

```tsx
async function enableMocking() {
  if (import.meta.env.VITE_USE_MOCK !== "true") return;
  // 동적 import: 가짜 서버를 끈 빌드에는 mocks 코드가 아예 포함되지 않는다.
  const { worker } = await import("@/mocks/browser");
  // 핸들러가 없는 요청은 그대로 네트워크로 보낸다. API를 하나씩 진짜 서버로 옮길 수 있다.
  await worker.start({ onUnhandledFrame: "bypass" });
}

// 가짜 서버가 켜진 뒤에 앱을 그린다. 순서가 바뀌면 첫 요청이 가짜 서버를 못 탄다.
enableMocking().then(() => {
  createRoot(document.getElementById("root")!).render(
    <StrictMode>
      <AppProviders>
        <App />
      </AppProviders>
    </StrictMode>,
  );
});
```

> ⚠️ **주의** MSW 3부터 옵션 이름이 바뀌었습니다. 인터넷 예제의 `onUnhandledRequest`는 2.x 시절 이름이고, 지금은 `onUnhandledFrame`입니다.
> 라이브러리는 메이저 버전이 바뀌면 이렇게 이름이 달라지곤 합니다. 예제가 빨간 줄이 나면 라이브러리의 타입 정의(`F12`)가 가장 정확한 문서입니다.

### 코드 읽어 보기: 동적 import

파일 맨 위의 `import`는 **무조건** 번들에 포함됩니다. 함수 안의 `await import()`는 **그 코드가 실행될 때** 파일을 내려받습니다.
게다가 Vite는 빌드할 때 `import.meta.env.VITE_USE_MOCK`을 실제 값으로 바꿔 넣습니다. 그래서 운영 빌드에서는 `if`가 항상 참이 되어
그 아래 `import()`가 통째로 제거됩니다. 가짜 서버가 운영에 섞여 들어갈 일이 없습니다.

### 확인하기

- [ ] `npm run dev` → F12 → **Console** 탭에 `[MSW] Mocking enabled.`가 보입니다.
- [ ] F12 → **Application** → **Service workers**에 `mockServiceWorker.js`가 활성 상태로 보입니다.

> ⌨️ **단축키** 브라우저 `F12`: 개발자 도구. `Ctrl+Shift+C`: 요소 선택 모드. `Ctrl+Shift+R`: 캐시를 무시하고 새로고침.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 개발 모드에서 가짜 서버 시작"
```

---

## Step 3-3. 가짜 DB

> 🎯 **할 일**: 새로고침해도 데이터가 남는 간단한 "테이블"을 만듭니다.

### 왜 필요한가요?

등록·수정·삭제를 연습하려면 데이터가 어딘가에 저장되어야 합니다. 브라우저의 `localStorage`를 DB처럼 씁니다.
핸들러는 메모리에 있어서, 새로고침하면 사라지기 때문입니다.

이 Step의 `createTable`에는 세심한 장치가 두 개 들어 있습니다.

1. **매번 localStorage에서 읽습니다.** 메모리에 들고 있지 않습니다. 그래서 브라우저 탭을 두 개 열어 다른 계정으로 로그인해도 같은 데이터를 봅니다.
   Part 12에서 "A가 일으킨 이벤트의 알림이 B에게 도착"을 시연할 때 이 덕분에 동작합니다.
2. **시드(초기 데이터)가 바뀌면 자동으로 초기화합니다.** 예전 데이터가 남아서 "시드를 고쳤는데 화면이 그대로"인 혼란을 막습니다.

### 따라 하기

📄 **`src/mocks/db.ts`** (새 파일)

```ts
interface Row {
  id: number;
}

/** localStorage에 저장되는 가짜 테이블을 만든다. */
export function createTable<T extends Row>(name: string, seed: T[]) {
  const dataKey = `mockdb:${name}`;
  const seedKey = `mockdb:${name}:seed`;
  const seedText = JSON.stringify(seed);

  const read = (): T[] => {
    // 시드가 바뀌었으면 저장된 데이터를 버린다. 코드와 데이터가 어긋나는 것을 막는다.
    if (localStorage.getItem(seedKey) !== seedText) {
      localStorage.setItem(seedKey, seedText);
      localStorage.removeItem(dataKey);
    }
    const saved = localStorage.getItem(dataKey);
    return saved ? JSON.parse(saved) : structuredClone(seed);
  };

  const write = (rows: T[]) => localStorage.setItem(dataKey, JSON.stringify(rows));

  return {
    all: read,
    find: (id: number) => read().find((row) => row.id === id),
    insert: (data: Omit<T, "id">) => {
      const rows = read();
      const id = Math.max(0, ...rows.map((row) => row.id)) + 1;
      const row = { ...data, id } as T;
      write([...rows, row]);
      return row;
    },
    update: (id: number, patch: Partial<T>) => {
      const rows = read().map((row) => (row.id === id ? { ...row, ...patch } : row));
      write(rows);
      return rows.find((row) => row.id === id);
    },
    remove: (id: number) => write(read().filter((row) => row.id !== id)),
    reset: () => localStorage.removeItem(dataKey),
  };
}
```

### 코드 읽어 보기: 불변 업데이트

`update`를 보면 배열을 고치지 않고 `map`으로 **새 배열**을 만듭니다. `{ ...row, ...patch }`도 객체를 고치지 않고 새 객체를 만듭니다.

```ts
rows[0].name = "새 이름";                                        // ❌ 원본을 직접 고침
rows.map((row) => (row.id === id ? { ...row, ...patch } : row)); // ✅ 새 배열을 만듦
```

React 세계에서는 이 습관이 아주 중요합니다. React는 **"값이 새 객체인가"** 로 변화를 알아채기 때문입니다.
이후 Zustand 저장소에서도 같은 방식을 씁니다.

**`Omit<T, "id">`, `Partial<T>`**: TypeScript의 유틸리티 타입입니다. 기존 타입에서 새 타입을 만듭니다.

- `Omit<User, "id">`: `User`에서 `id`를 뺀 타입 (등록할 때는 id가 없으니까)
- `Partial<User>`: `User`의 모든 속성을 선택 사항으로 (일부만 수정하니까)

**`<T extends Row>`**: "T는 어떤 타입이든 되지만 `id`는 꼭 있어야 한다"는 제약입니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: localStorage 기반 가짜 DB 추가"
```

---

## Step 3-4. 사용자 조회 API

> 🎯 **할 일**: 코드에 박혀 있던 사용자 데이터를 가짜 서버로 옮기고, 화면이 쓸 API 함수를 만듭니다.

### 따라 하기

먼저 데이터의 모양을 정합니다. 기능 폴더 안에 둡니다.

📄 **`src/features/users/types.ts`** (새 파일)

```ts
import type { UseYn } from "@/shared/api/types";

export interface User {
  id: number;
  username: string; // 로그인 아이디
  name: string;
  email: string;
  phone: string;
  roleCodes: string[]; // 한 사람이 역할을 여러 개 가질 수 있다
  useYn: UseYn;
  createdAt: string; // YYYY-MM-DD
}

export interface UserSearchCondition {
  name?: string;
  roleCode?: string;
  useYn?: UseYn;
}
```

시드 데이터입니다.

📄 **`src/mocks/data/users.ts`** (새 파일)

```ts
import type { User } from "@/features/users/types";

export const userSeed: User[] = [
  {
    id: 1,
    username: "admin",
    name: "김민수",
    email: "minsu.kim@example.com",
    phone: "010-1234-0001",
    roleCodes: ["ADMIN"],
    useYn: "Y",
    createdAt: "2026-09-01",
  },
  {
    id: 2,
    username: "manager",
    name: "박지훈",
    email: "jihoon.park@example.com",
    phone: "010-1234-0002",
    roleCodes: ["MANAGER"],
    useYn: "Y",
    createdAt: "2026-09-02",
  },
  {
    id: 3,
    username: "user",
    name: "이서연",
    email: "seoyeon.lee@example.com",
    phone: "010-1234-0003",
    roleCodes: ["USER"],
    useYn: "Y",
    createdAt: "2026-09-03",
  },
  {
    id: 4,
    username: "yujin",
    name: "최유진",
    email: "yujin.choi@example.com",
    phone: "010-1234-0004",
    roleCodes: ["USER"],
    useYn: "Y",
    createdAt: "2026-09-10",
  },
  {
    id: 5,
    username: "doyun",
    name: "정도윤",
    email: "doyun.jung@example.com",
    phone: "010-1234-0005",
    roleCodes: ["MANAGER", "USER"],
    useYn: "Y",
    createdAt: "2026-09-12",
  },
  {
    id: 6,
    username: "haeun",
    name: "강하은",
    email: "haeun.kang@example.com",
    phone: "010-1234-0006",
    roleCodes: ["USER"],
    useYn: "N",
    createdAt: "2026-09-20",
  },
];
```

테이블을 만드는 곳을 한 파일로 모읍니다. 앞으로 테이블이 늘어나면 여기에 추가합니다.

✏️ **`src/mocks/db.ts`** — 파일 맨 위에 import를 넣습니다.

```ts
interface Row {
  id: number;
}
```

바꿀 코드

```ts
import { userSeed } from "@/mocks/data/users";

interface Row {
  id: number;
}
```

✏️ **`src/mocks/db.ts`** — 파일 맨 아래에 테이블을 추가합니다.

```ts
    reset: () => localStorage.removeItem(dataKey),
  };
}
```

바꿀 코드

```ts
    reset: () => localStorage.removeItem(dataKey),
  };
}

export const usersTable = createTable("users", userSeed);
```

가짜 API 핸들러입니다.

📄 **`src/mocks/handlers/users.ts`** (새 파일)

```ts
import { http, HttpResponse } from "msw";
import { usersTable } from "@/mocks/db";
import { apiUrl } from "@/mocks/utils";

export const userHandlers = [
  http.get(apiUrl("/users"), ({ request }) => {
    const params = new URL(request.url).searchParams;
    const name = params.get("name") ?? "";
    const roleCode = params.get("roleCode");
    const useYn = params.get("useYn");

    const rows = usersTable
      .all()
      .filter(
        (user) =>
          user.name.includes(name) &&
          (!roleCode || user.roleCodes.includes(roleCode)) &&
          (!useYn || user.useYn === useYn),
      );
    return HttpResponse.json(rows);
  }),
];
```

✏️ **`src/mocks/handlers.ts`** — 핸들러를 등록합니다.

```ts
import { apiUrl, MOCK_DELAY } from "@/mocks/utils";
```

바꿀 코드

```ts
import { userHandlers } from "@/mocks/handlers/users";
import { apiUrl, MOCK_DELAY } from "@/mocks/utils";
```

✏️ **`src/mocks/handlers.ts`** — 배열 끝에 추가합니다.

```ts
    await delay(MOCK_DELAY);
  }),
];
```

바꿀 코드

```ts
    await delay(MOCK_DELAY);
  }),
  ...userHandlers,
];
```

이제 앱 쪽입니다. URL을 아는 함수 모음을 만듭니다.

📄 **`src/features/users/api/usersApi.ts`** (새 파일)

```ts
import { httpClient } from "@/shared/api/httpClient";
import type { User, UserSearchCondition } from "@/features/users/types";

// URL과 파라미터만 안다. 토큰, 오류 처리, 서버가 진짜인지는 모른다.
export const usersApi = {
  list: (condition: UserSearchCondition, signal?: AbortSignal) =>
    httpClient.get<User[]>("/users", { params: condition, signal }),
};
```

> ⚠️ **주의** `mocks`가 `features`의 **타입**을 가져오는 것은 괜찮습니다. `import type`은 빌드하면 사라지기 때문입니다.
> 반대로 앱 코드가 `mocks`를 import 하면 안 됩니다. 규칙 3번입니다.

아직 화면과 연결하지 않았으니, 브라우저 콘솔에서 먼저 시험해 봅니다.

### 확인하기

- [ ] `npm run dev` → F12 → Console 탭에 아래를 붙여 넣고 Enter.

  ```js
  await (await fetch("/api/users?name=김")).json();
  ```

  `김민수` 한 명이 담긴 배열이 나옵니다. **서버가 없는데도 HTTP 응답이 왔습니다!**

- [ ] F12 → **Network** 탭에서 `users?name=...` 요청을 클릭해 응답을 봅니다.
- [ ] F12 → **Application** → **Local Storage**에 아직 `mockdb:users`가 없습니다. 쓰기가 일어나면 생깁니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 사용자 조회 가짜 API 추가"
```

---

## Step 3-5. TanStack Query로 화면 연결

> 🎯 **할 일**: 서버 데이터를 화면에 표시합니다.

### 왜 `useEffect`로 직접 안 하나요?

서버 데이터를 직접 가져오면 이런 코드를 화면마다 써야 합니다.

```tsx
const [users, setUsers] = useState<User[]>([]);
const [loading, setLoading] = useState(true);
const [error, setError] = useState<Error | null>(null);

useEffect(() => {
  let ignore = false;  // 화면을 떠난 뒤 도착한 응답을 무시하기 위한 장치
  setLoading(true);
  usersApi.list(condition)
    .then((data) => { if (!ignore) setUsers(data); })
    .catch((e) => { if (!ignore) setError(e); })
    .finally(() => { if (!ignore) setLoading(false); });
  return () => { ignore = true; };
}, [condition]);
```

게다가 캐시도, 중복 요청 방지도, 저장 후 재조회도 없습니다. **TanStack Query**는 이걸 전부 대신합니다.

```tsx
const { data, isFetching } = useUserList(condition);
```

### 핵심 개념: 서버 상태

앱의 데이터는 "원본이 어디 있나"로 나뉩니다.

| 종류             | 예                      | 도구           | 이유                                     |
| ---------------- | ----------------------- | -------------- | ---------------------------------------- |
| **서버 상태**    | 사용자 목록, 메뉴       | TanStack Query | 원본은 서버. 화면의 것은 복사본(캐시)    |
| **전역 UI 상태** | 테마, 열린 탭, 토큰     | Zustand        | 여러 곳에서 쓰고, 새로고침 후에도 남음   |
| **화면 상태**    | 조회 조건, 모달 열림    | `useState`     | 그 화면에서만 의미 있음                  |
| **폼 상태**      | 입력값, 검증 오류       | antd Form      | 검증·필드 관리를 antd가 담당             |

판단 순서는 이렇습니다.

1. 서버에서 온 데이터인가? → **Query**
2. 여러 화면이 쓰거나 새로고침 후에도 남아야 하나? → **Zustand**
3. 아니면 → **useState**

> ⚠️ **주의** 가장 흔한 실수는 서버에서 받은 데이터를 Zustand에 복사해 두는 것입니다. 그러면 "언제 다시 받아 올지"를 직접 관리해야 합니다.

**queryKey**: Query는 데이터를 **키**로 구분해 캐시합니다. 조회 조건을 키에 넣으면, 조건이 바뀔 때 자동으로 새로 조회합니다.

```
["users", "list", { name: "김" }]   → 이 키의 캐시
["users", "list", { name: "이" }]   → 저 키의 캐시
```

### 따라 하기

▶️ **터미널**

```powershell
npm install @tanstack/react-query@5.106.1
```

✅ **커밋**

```powershell
git add .
git commit -m "chore: @tanstack/react-query 의존성 추가"
```

📄 **`src/app/queryClient.ts`** (새 파일)

```ts
import { QueryClient } from "@tanstack/react-query";

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: false, // 실패 시 자동 재시도 끔. 오류를 바로 보여 주는 쪽이 업무 화면에 맞다.
      refetchOnWindowFocus: false, // 창 전환만으로 그리드가 다시 조회되면 사용자가 혼란스럽다.
    },
  },
});
```

✏️ **`src/app/AppProviders.tsx`** — Provider를 추가합니다.

```tsx
    <ConfigProvider locale={koKR}>
      <AntdApp style={{ height: "100%" }}>
        <BrowserRouter>{children}</BrowserRouter>
      </AntdApp>
    </ConfigProvider>
```

바꿀 코드

```tsx
    <QueryClientProvider client={queryClient}>
      <ConfigProvider locale={koKR}>
        <AntdApp style={{ height: "100%" }}>
          <BrowserRouter>{children}</BrowserRouter>
        </AntdApp>
      </ConfigProvider>
    </QueryClientProvider>
```

✏️ **`src/app/AppProviders.tsx`** — import를 추가합니다.

```tsx
import { BrowserRouter } from "react-router";
import { App as AntdApp, ConfigProvider } from "antd";
import koKR from "antd/locale/ko_KR";
```

바꿀 코드

```tsx
import { BrowserRouter } from "react-router";
import { QueryClientProvider } from "@tanstack/react-query";
import { App as AntdApp, ConfigProvider } from "antd";
import koKR from "antd/locale/ko_KR";
import { queryClient } from "@/app/queryClient";
```

📄 **`src/features/users/api/queries.ts`** (새 파일)

```ts
import { useQuery } from "@tanstack/react-query";
import { usersApi } from "@/features/users/api/usersApi";
import type { UserSearchCondition } from "@/features/users/types";

// 캐시 키를 한 곳에서 만든다. "users"로 시작하는 키는 사용자 관련 캐시 전부를 뜻한다.
export const userKeys = {
  all: ["users"] as const,
  list: (condition: UserSearchCondition) => [...userKeys.all, "list", condition] as const,
};

export function useUserList(condition: UserSearchCondition) {
  return useQuery({
    queryKey: userKeys.list(condition),
    // signal: 화면을 떠나면 진행 중인 요청을 취소한다.
    queryFn: ({ signal }) => usersApi.list(condition, signal),
  });
}
```

📄 **`src/features/users/pages/UsersPage.tsx`** (전체 교체)

```tsx
import { AgGridReact } from "ag-grid-react";
import type { ColDef } from "ag-grid-community";
import { useUserList } from "@/features/users/api/queries";
import type { User } from "@/features/users/types";

const columnDefs: ColDef<User>[] = [
  { headerName: "아이디", field: "username" },
  { headerName: "이름", field: "name" },
  { headerName: "이메일", field: "email", flex: 2 },
  { headerName: "전화번호", field: "phone" },
  // 값이 배열이면 그리드는 어떻게 보여 줄지 모른다. 글자로 바꾸는 방법을 알려 준다.
  { headerName: "역할", field: "roleCodes", valueFormatter: ({ value }) => value?.join(", ") },
  { headerName: "사용", field: "useYn", width: 90, flex: 0 },
  { headerName: "등록일", field: "createdAt" },
];

const defaultColDef: ColDef<User> = { sortable: true, flex: 1 };

function UsersPage() {
  const { data, isFetching } = useUserList({});

  return (
    <div style={{ height: "100%", display: "flex", flexDirection: "column" }}>
      <h2 style={{ marginTop: 0 }}>사용자 관리</h2>
      <div style={{ flex: 1, minHeight: 0 }}>
        <AgGridReact<User>
          rowData={data}
          columnDefs={columnDefs}
          defaultColDef={defaultColDef}
          loading={isFetching}
        />
      </div>
    </div>
  );
}

export default UsersPage;
```

### 코드 읽어 보기

`useUserList`처럼 `use`로 시작하는 함수를 직접 만든 것을 **커스텀 훅**이라고 합니다. 로직을 묶어서 여러 화면에서 재사용합니다.

**`valueFormatter`**: 셀에 보여 줄 글자를 만듭니다. 데이터는 배열 그대로 두고 표시만 바꿉니다.
빼면 콘솔에 `AG Grid: warning #48`이 뜹니다. 그리드 관련 경고는 콘솔에 링크로 나오니, 눌러 보면 원인이 설명되어 있습니다.

`as const`는 배열을 "바뀌지 않는 정확한 값"으로 고정합니다. `string[]`이 아니라 `readonly ["users"]`가 됩니다.

`data`와 `isFetching`의 차이도 알아 둡시다.

| 값          | 뜻                                           |
| ----------- | -------------------------------------------- |
| `data`      | 데이터. 처음에는 `undefined`                 |
| `isPending` | 아직 데이터가 하나도 없음                    |
| `isFetching`| 요청 진행 중 (재조회 포함)                   |
| `isError`   | 실패함                                       |

### 확인하기

- [ ] `/users`에 들어가면 로딩 표시 후 6명이 보입니다.
- [ ] F12 → Network 탭에 `users` 요청이 찍힙니다.
- [ ] 홈에 갔다가 돌아오면 **즉시** 목록이 보이고, 뒤에서 조용히 재조회가 일어납니다. 이것이 캐시입니다.

### 스스로 답해 보기

<details>
<summary>Q. 지금까지의 방식에서 필요했던 <code>ignore</code> 플래그는 왜 없어졌나요?</summary>

Query가 요청을 키로 관리하기 때문입니다. 조건이 바뀌면 이전 키의 응답은 현재 화면과 무관해지고,
`signal`로 취소까지 됩니다. "늦게 도착한 옛날 응답이 화면을 덮어쓰는" 문제가 구조적으로 사라집니다.

</details>

✅ **커밋**

```powershell
git add .
git commit -m "feat: 사용자 화면을 서버 데이터로 전환"
```

---

## Step 3-6. 조회 조건

> 🎯 **할 일**: 이름·역할로 검색하는 폼을 붙입니다.

### 따라 하기

✏️ **`src/features/users/pages/UsersPage.tsx`** — import를 고칩니다.

```tsx
import { AgGridReact } from "ag-grid-react";
import type { ColDef } from "ag-grid-community";
import { useUserList } from "@/features/users/api/queries";
import type { User } from "@/features/users/types";
```

바꿀 코드

```tsx
import { useState } from "react";
import { Button, Form, Input, Select, Space } from "antd";
import { AgGridReact } from "ag-grid-react";
import type { ColDef } from "ag-grid-community";
import { useUserList } from "@/features/users/api/queries";
import type { User, UserSearchCondition } from "@/features/users/types";
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — 상태와 조회 함수를 추가합니다.

```tsx
  const { data, isFetching } = useUserList({});
```

바꿀 코드

```tsx
  const [form] = Form.useForm<UserSearchCondition>();
  const [condition, setCondition] = useState<UserSearchCondition>({});
  const { data, isFetching, refetch } = useUserList(condition);

  const handleSearch = (values: UserSearchCondition) => {
    // 조건이 같으면 queryKey도 같아서 캐시만 보고 끝난다.
    // 하지만 SI 사용자는 [조회] = "서버에서 다시 가져오기"로 기대하므로 직접 재조회한다.
    if (JSON.stringify(values) === JSON.stringify(condition)) refetch();
    else setCondition(values);
  };

  const handleReset = () => {
    form.resetFields();
    handleSearch({});
  };
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — 폼을 넣습니다.

```tsx
      <h2 style={{ marginTop: 0 }}>사용자 관리</h2>
      <div style={{ flex: 1, minHeight: 0 }}>
```

바꿀 코드

```tsx
      <h2 style={{ marginTop: 0 }}>사용자 관리</h2>
      <Form form={form} layout="inline" onFinish={handleSearch} style={{ marginBottom: 16 }}>
        <Form.Item name="name" label="이름">
          <Input placeholder="이름 입력" allowClear />
        </Form.Item>
        <Form.Item name="roleCode" label="역할">
          <Select
            placeholder="전체"
            allowClear
            style={{ width: 140 }}
            options={[
              { value: "ADMIN", label: "관리자" },
              { value: "MANAGER", label: "매니저" },
              { value: "USER", label: "사용자" },
            ]}
          />
        </Form.Item>
        <Space>
          <Button type="primary" htmlType="submit">
            조회
          </Button>
          <Button onClick={handleReset}>초기화</Button>
        </Space>
      </Form>
      <div style={{ flex: 1, minHeight: 0 }}>
```

### 코드 읽어 보기: antd Form

`Form.Item`의 `name`이 곧 폼 값의 키입니다. 입력값을 `useState`로 하나하나 관리할 필요가 없습니다.
`htmlType="submit"` 버튼을 누르거나 입력칸에서 Enter를 치면 `onFinish`가 폼 전체 값을 받아서 실행됩니다.

조회의 흐름을 따라가 봅시다.

```
① [조회] → onFinish → setCondition({ name: "김" })
② condition이 바뀜 → 컴포넌트 재실행 → useUserList의 queryKey가 바뀜
③ Query가 "새 키네?" → 자동으로 요청 → data가 바뀜 → 그리드에 표시
```

`fetch`를 부르는 코드가 화면에 한 줄도 없습니다. **상태를 바꾸면 데이터가 따라옵니다.**

> ⌨️ **단축키** `Alt+↑/↓`: 현재 줄을 위아래로 이동. `Shift+Alt+↓`: 현재 줄 복제. `Ctrl+D`: 같은 단어를 하나씩 추가 선택(여러 곳 동시 수정).
> `Ctrl+/`: 주석 켜기/끄기. JSX 안에서도 알맞은 주석으로 바꿔 줍니다.

### 확인하기

- [ ] 이름에 `김` 입력 후 Enter → 1명만 보입니다. Network 탭 요청이 `users?name=%EA%B9%80`입니다. (한글이 인코딩된 것)
- [ ] 같은 조건으로 [조회]를 다시 누르면 Network 탭에 요청이 또 찍힙니다.
- [ ] [초기화] → 전체 목록으로 돌아옵니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 사용자 조회 조건 추가"
```

---

## Step 3-7. API 오류 공통 처리

> 🎯 **할 일**: 서버 오류가 나면 화면마다 처리하지 않아도 알림이 뜨게 합니다.

### 왜 까다로운가요?

오류 알림을 띄우려면 antd의 `message`가 필요합니다. 그런데 antd의 `message.error()` 정적 메서드는 `ConfigProvider` 설정을 못 읽어서
다크 모드나 언어가 적용되지 않습니다. 올바른 방법은 `App.useApp()` 훅인데, **훅은 컴포넌트 안에서만** 쓸 수 있습니다.
그런데 Query의 전역 오류 처리기는 컴포넌트 **밖**에 있습니다.

해결책은 간단합니다. 화면에 아무것도 그리지 않는 작은 컴포넌트가 `useApp()`의 결과를 바깥 변수에 넘겨주면 됩니다.

### 따라 하기

📄 **`src/shared/ui/antdStatic.ts`** (새 파일)

```ts
import type { useAppProps } from "antd/es/app/context";

// 컴포넌트 밖(Query 오류 처리기 등)에서 antd 알림을 쓰기 위한 보관함.
export const antdStatic: Partial<useAppProps> = {};

export function setAntdStatic(app: useAppProps) {
  Object.assign(antdStatic, app);
}
```

📄 **`src/shared/ui/AntdStaticBridge.tsx`** (새 파일)

```tsx
import { useEffect } from "react";
import { App } from "antd";
import { setAntdStatic } from "@/shared/ui/antdStatic";

// 화면에는 아무것도 그리지 않는다. App.useApp()의 결과를 컴포넌트 밖으로 전달만 한다.
function AntdStaticBridge() {
  const app = App.useApp();

  useEffect(() => {
    setAntdStatic(app);
  }, [app]);

  return null;
}

export default AntdStaticBridge;
```

📄 **`src/shared/api/showApiError.ts`** (새 파일)

```ts
import { ApiError } from "@/shared/api/httpClient";
import { antdStatic } from "@/shared/ui/antdStatic";

/** API 오류를 사용자에게 알린다. 화면마다 처리하지 않아도 되도록 Query 전역 처리기에서 호출한다. */
export function showApiError(error: unknown) {
  if (!(error instanceof ApiError)) {
    // 요청 취소는 오류가 아니다. 그 외에는 네트워크 단절 등이다.
    if (error instanceof DOMException && error.name === "AbortError") return;
    antdStatic.message?.error("서버에 연결할 수 없습니다.");
    return;
  }
  if (error.status >= 500) {
    antdStatic.message?.error("잠시 후 다시 시도해 주세요.");
    return;
  }
  antdStatic.message?.error(error.message);
}
```

📄 **`src/app/queryClient.ts`** (전체 교체)

```ts
import { MutationCache, QueryCache, QueryClient } from "@tanstack/react-query";
import { showApiError } from "@/shared/api/showApiError";

// meta에 적을 수 있는 값의 타입을 알려 준다.
declare module "@tanstack/react-query" {
  interface Register {
    mutationMeta: { skipGlobalError?: boolean };
  }
}

export const queryClient = new QueryClient({
  queryCache: new QueryCache({ onError: showApiError }),
  mutationCache: new MutationCache({
    onError: (error, _variables, _context, mutation) => {
      // 화면이 직접 처리하는 오류(예: 입력칸에 표시)는 meta로 전역 알림을 끈다. (Part 9)
      if (!mutation.meta?.skipGlobalError) showApiError(error);
    },
  }),
  defaultOptions: {
    queries: {
      retry: false, // 실패 시 자동 재시도 끔. 오류를 바로 보여 주는 쪽이 업무 화면에 맞다.
      refetchOnWindowFocus: false, // 창 전환만으로 그리드가 다시 조회되면 사용자가 혼란스럽다.
    },
  },
});
```

✏️ **`src/app/AppProviders.tsx`** — `AntdStaticBridge`를 넣습니다.

```tsx
          <BrowserRouter>{children}</BrowserRouter>
```

바꿀 코드

```tsx
          <AntdStaticBridge />
          <BrowserRouter>{children}</BrowserRouter>
```

✏️ **`src/app/AppProviders.tsx`** — import를 추가합니다.

```tsx
import { queryClient } from "@/app/queryClient";
```

바꿀 코드

```tsx
import { queryClient } from "@/app/queryClient";
import AntdStaticBridge from "@/shared/ui/AntdStaticBridge";
```

### 확인하기

오류를 일부러 만들어 봅니다. `src/mocks/handlers/users.ts`의 `http.get` 첫 줄에 잠깐 아래를 넣습니다.

```ts
return problem(500, "일부러 만든 오류");  // 맨 위에 import { apiUrl, problem } from "@/mocks/utils"; 로 고칩니다
```

- [ ] `/users`를 새로고침하면 화면 위쪽에 "잠시 후 다시 시도해 주세요."가 뜹니다.
- [ ] Network 탭에서 응답 상태가 500입니다.
- [ ] 확인했으면 넣었던 코드를 **되돌립니다.** (`Ctrl+Z`)

> 💡 **팁** Git을 쓰면 되돌리기가 쉽습니다. 터미널에서 `git status`로 바뀐 파일을 보고, `git restore src/mocks/handlers/users.ts`로 마지막 커밋 상태로 돌립니다.
> VS Code 왼쪽 **소스 제어**(`Ctrl+Shift+G`)에서 파일을 클릭하면 변경 내용을 나란히 볼 수도 있습니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: API 오류 공통 알림 추가"
```

---

## Step 3-8. Query Devtools ○

> 🎯 **할 일**: 캐시 상태를 눈으로 보는 도구를 켭니다.

▶️ **터미널**

```powershell
npm install -D @tanstack/react-query-devtools@5.106.1
```

✏️ **`src/app/AppProviders.tsx`** — Devtools를 넣습니다.

```tsx
    </QueryClientProvider>
```

바꿀 코드

```tsx
      <ReactQueryDevtools buttonPosition="bottom-left" />
    </QueryClientProvider>
```

✏️ **`src/app/AppProviders.tsx`** — import를 추가합니다.

```tsx
import { QueryClientProvider } from "@tanstack/react-query";
```

바꿀 코드

```tsx
import { QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";
```

화면 왼쪽 아래 꽃 모양 버튼을 누르면 캐시 목록이 나옵니다. (오른쪽 아래는 보통 [저장] 같은 버튼이 오는 자리라서 가리지 않도록 왼쪽에 둡니다) 키별로 `fresh`(신선함), `stale`(낡음), `fetching`(조회 중)을 볼 수 있습니다.
조회 조건을 바꿔 가며 캐시가 쌓이는 모습을 구경해 보세요. Devtools는 운영 빌드에서는 자동으로 빠집니다.

✅ **커밋**

```powershell
git add .
git commit -m "chore: TanStack Query Devtools 추가"
```

---

## 🏁 Part 3을 마치며

마일스톤입니다. 지금 앱은 서버가 없는데도 진짜 HTTP 요청을 주고받습니다.

```
화면 코드에 fetch가 없다  →  서버가 바뀌어도 화면은 그대로
```

| 배운 것               | 한 줄 요약                                   |
| --------------------- | -------------------------------------------- |
| 계층 분리             | 각 층은 바로 아래층만 안다                   |
| Service Worker, MSW   | 브라우저 안의 가짜 서버                      |
| 제네릭, 유틸리티 타입 | `get<T>()`, `Omit`, `Partial`                |
| 불변 업데이트         | 고치지 말고 새로 만든다                      |
| 서버 상태             | 원본은 서버, 화면의 것은 캐시                |
| queryKey              | 조건을 키에 넣으면 자동으로 다시 조회한다    |

다음은 눈에 확 보이는 재미있는 구간입니다. → [Part 4. 테마와 라이트/다크](part-04-theme.md)
