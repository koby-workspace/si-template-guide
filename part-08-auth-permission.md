# Part 8. 로그인과 권한

> **이 Part의 목표**: 로그인해야 앱을 쓸 수 있고, 로그인한 사람의 **역할**에 따라 보이는 메뉴와 버튼이 달라지게 합니다.
> 이 Part가 끝나면 `admin`, `manager`, `user` 계정으로 각각 로그인해서 서로 다른 화면을 보게 됩니다.

---

## 먼저 개념 정리

| 용어           | 질문               | 이 템플릿에서                          |
| -------------- | ------------------ | -------------------------------------- |
| 인증 (로그인)  | 누구인가?          | 로그인 → 토큰                          |
| 인가 (권한)    | 무엇을 할 수 있나? | 역할별 메뉴 권한 → 메뉴·화면·버튼 제어 |

### RBAC (역할 기반 권한)

```
사용자 ──(여러 개)── 역할 ──(역할별 메뉴 권한)── 메뉴 × 동작
김민수 ─ ADMIN   ─ 모든 메뉴 × 모든 동작
박지훈 ─ MANAGER ─ 사용자 관리 × 조회, 공통코드·공지사항 × 전부 ...
이서연 ─ USER    ─ 공지사항 × 조회 ...
```

사람에게 권한을 직접 주면 100명의 권한을 하나하나 관리해야 합니다. 역할에 주면 역할 3~5개만 관리하면 됩니다.

**동작(Action)** 은 메뉴 안에서 할 수 있는 일입니다. 이 템플릿은 다섯 가지로 시작합니다.

| 동작     | 뜻          | 예                         |
| -------- | ----------- | -------------------------- |
| `READ`   | 조회        | 메뉴가 보이고 화면이 열림  |
| `CREATE` | 등록        | [등록] 버튼                |
| `UPDATE` | 수정        | [저장] 버튼                |
| `DELETE` | 삭제        | [삭제] 버튼                |
| `EXPORT` | 내보내기    | [엑셀] 버튼                |

프로젝트에 따라 `PRINT`(출력), `APPROVE`(승인) 등을 추가합니다.

### 세 단계로 막는다

| 단계        | 어디서        | 방법                                                         |
| ----------- | ------------- | ------------------------------------------------------------ |
| ① 메뉴 표시 | 사이드바      | 서버가 **권한 있는 메뉴만** 내려준다                         |
| ② 화면 진입 | 탭 열기       | 주소를 직접 입력해도 내 메뉴에 없으면 열리지 않는다          |
| ③ 버튼·기능 | 각 화면       | `<Can action="DELETE">`, `usePermission().can("DELETE")`     |

> ⚠️ **주의** 프론트엔드 권한은 **편의**입니다. 버튼을 숨겨도 개발자 도구로 요청을 직접 보내면 그만입니다.
> **진짜로 막는 것은 서버**입니다. 그래서 이 템플릿은 가짜 서버도 권한을 검사하게 만듭니다. 연습 단계부터 "서버도 막는다"는 감각을 익히기 위해서입니다.

---

## Step 8-1. 가짜 로그인 API와 역할·권한 데이터

> 🎯 **할 일**: 가짜 서버에 계정·역할·권한 데이터와 로그인 API를 만듭니다.

### 테스트 계정

| 아이디    | 비밀번호      | 역할    | 볼 수 있는 것                                          |
| --------- | ------------- | ------- | ------------------------------------------------------ |
| `admin`   | `admin1234`   | ADMIN   | 전부                                                   |
| `manager` | `manager1234` | MANAGER | 사용자(조회·내보내기), 공통코드, 공지사항, 샘플 등     |
| `user`    | `user1234`    | USER    | 대시보드, 공지사항(조회), 알림 테스트, 내 정보         |

### 따라 하기

권한 관련 타입을 추가합니다. 동작(`Action`)은 메뉴와 짝을 이루므로 메뉴 타입 파일에 둡니다.

✏️ **`src/shared/menu/types.ts`** — 파일 끝에 추가합니다.

```ts
// 트리로 바꾼 메뉴. Menu가 유니온이라 interface extends를 쓸 수 없어서 type으로 만든다.
export type MenuNode = Menu & { children: MenuNode[] };
```

바꿀 코드

```ts
// 트리로 바꾼 메뉴. Menu가 유니온이라 interface extends를 쓸 수 없어서 type으로 만든다.
export type MenuNode = Menu & { children: MenuNode[] };

/** 메뉴 안에서 할 수 있는 동작. 프로젝트에 따라 "PRINT", "APPROVE" 등을 추가한다. */
export type Action = "READ" | "CREATE" | "UPDATE" | "DELETE" | "EXPORT";

export const ACTIONS: Action[] = ["READ", "CREATE", "UPDATE", "DELETE", "EXPORT"];

/** 내 메뉴: 메뉴 + 그 메뉴에서 내가 할 수 있는 동작 */
export type MyMenu = Menu & { actions: Action[] };
```

📄 **`src/shared/auth/types.ts`** (새 파일)

```ts
import type { UseYn } from "@/shared/api/types";
import type { LocalizedText } from "@/shared/i18n/locales";
import type { Action } from "@/shared/menu/types";

/** 로그인한 내 정보 */
export interface Me {
  id: number;
  username: string;
  name: string;
  email: string;
  roleCodes: string[];
}

export interface Role {
  id: number;
  code: string; // 예: "ADMIN"
  names: LocalizedText;
  description: string;
  useYn: UseYn;
}

/** 역할별 메뉴 권한. "이 역할은 이 메뉴에서 이 동작들을 할 수 있다" */
export interface Permission {
  id: number;
  roleCode: string;
  menuCode: string;
  actions: Action[];
}
```

> 💡 **팁** `User`(사용자 관리 화면의 데이터)가 아니라 `Me`를 따로 만든 이유: `shared`는 `features`를 import 할 수 없습니다(규칙 1).
> 헤더처럼 모든 화면이 쓰는 곳은 `shared`의 타입을 씁니다.

역할과 권한 시드입니다.

📄 **`src/mocks/data/roles.ts`** (새 파일)

```ts
import type { Role } from "@/shared/auth/types";

export const roleSeed: Role[] = [
  {
    id: 1,
    code: "ADMIN",
    names: { ko: "관리자", en: "Administrator" },
    description: "시스템 전체 관리",
    useYn: "Y",
  },
  {
    id: 2,
    code: "MANAGER",
    names: { ko: "매니저", en: "Manager" },
    description: "업무 담당자",
    useYn: "Y",
  },
  {
    id: 3,
    code: "USER",
    names: { ko: "사용자", en: "User" },
    description: "일반 사용자",
    useYn: "Y",
  },
];
```

📄 **`src/mocks/data/permissions.ts`** (새 파일)

```ts
import { menuSeed } from "@/mocks/data/menus";
import type { Permission } from "@/shared/auth/types";
import { ACTIONS, type Action } from "@/shared/menu/types";

const ALL = ACTIONS;
const READ: Action[] = ["READ"];

// 역할 → 메뉴 코드 → 동작. 폴더는 권한을 주지 않는다. (하위 화면 권한이 있으면 서버가 폴더도 함께 준다)
const grants: Record<string, Record<string, Action[]>> = {
  ADMIN: Object.fromEntries(
    menuSeed.filter((menu) => menu.type !== "FOLDER").map((menu) => [menu.code, ALL]),
  ),
  MANAGER: {
    DASHBOARD: READ,
    USER_MGMT: ["READ", "EXPORT"],
    CODE_MGMT: ALL,
    NOTICE: ALL,
    NOTICE_VIEW: READ,
    NOTICE_EDIT: READ,
    SAMPLE_GRID_EDIT: ALL,
    SAMPLE_FORM: ALL,
    SAMPLE_NOTIFY: ["READ", "CREATE"],
    SAMPLE_FILE: ALL,
    MY_NOTIFICATIONS: ["READ", "UPDATE"],
    MY_PROFILE: ["READ", "UPDATE"],
    MANUAL: READ,
  },
  USER: {
    DASHBOARD: READ,
    NOTICE: READ,
    NOTICE_VIEW: READ,
    SAMPLE_NOTIFY: ["READ", "CREATE"],
    MY_NOTIFICATIONS: ["READ", "UPDATE"],
    MY_PROFILE: ["READ", "UPDATE"],
    MANUAL: READ,
  },
};

export const permissionSeed: Permission[] = Object.entries(grants)
  .flatMap(([roleCode, menus]) =>
    Object.entries(menus).map(([menuCode, actions]) => ({ roleCode, menuCode, actions })),
  )
  .map((permission, index) => ({ ...permission, id: index + 1 }));
```

테이블을 추가합니다. 비밀번호는 사용자 데이터와 **따로** 둡니다. 사용자 목록 API가 실수로 비밀번호를 내보내는 일이 없게 하기 위해서입니다.

✏️ **`src/mocks/db.ts`** — import를 추가합니다.

```ts
import { menuSeed } from "@/mocks/data/menus";
import { userSeed } from "@/mocks/data/users";
```

바꿀 코드

```ts
import { menuSeed } from "@/mocks/data/menus";
import { permissionSeed } from "@/mocks/data/permissions";
import { roleSeed } from "@/mocks/data/roles";
import { userSeed } from "@/mocks/data/users";
```

✏️ **`src/mocks/db.ts`** — 테이블을 추가합니다.

```ts
export const menusTable = createTable("menus", menuSeed);
```

바꿀 코드

```ts
export const menusTable = createTable("menus", menuSeed);
export const rolesTable = createTable("roles", roleSeed);
export const permissionsTable = createTable("permissions", permissionSeed);
// 비밀번호는 사용자 데이터와 따로 두고 아이디로 찾는다. 초기 비밀번호는 "아이디1234"
export const credentialsTable = createTable(
  "credentials",
  userSeed.map((user) => ({
    id: user.id,
    username: user.username,
    password: `${user.username}1234`,
  })),
);
```

가짜 서버의 인증·권한 검사 도우미입니다. 모든 핸들러가 이 함수 하나로 검사합니다.

📄 **`src/mocks/auth.ts`** (새 파일)

```ts
import type { User } from "@/features/users/types";
import { permissionsTable, usersTable } from "@/mocks/db";
import { problem } from "@/mocks/utils";
import type { Action } from "@/shared/menu/types";

const TOKEN_PREFIX = "mock-token-";

// 진짜 서버라면 서명된 JWT나 세션 ID를 발급한다. 가짜 서버는 사용자 id를 넣은 문자열로 충분하다.
export const createToken = (userId: number) => `${TOKEN_PREFIX}${userId}`;

/** 요청 헤더의 토큰으로 사용자를 찾는다. */
export function findUserByToken(request: Request) {
  const token = request.headers.get("Authorization")?.replace("Bearer ", "") ?? "";
  if (!token.startsWith(TOKEN_PREFIX)) return undefined;
  const user = usersTable.find(Number(token.slice(TOKEN_PREFIX.length)));
  return user?.useYn === "Y" ? user : undefined;
}

/** 사용자가 한 메뉴에서 할 수 있는 동작. 역할이 여러 개면 합친다(합집합). */
export function actionsOf(user: User, menuCode: string): Action[] {
  const actions = permissionsTable
    .all()
    .filter((item) => item.menuCode === menuCode && user.roleCodes.includes(item.roleCode))
    .flatMap((item) => item.actions);
  return [...new Set(actions)];
}

type AuthResult = { user: User; error?: undefined } | { user?: undefined; error: Response };

/** 로그인과 권한을 검사한다. 통과하면 user를, 아니면 401·403 응답을 error로 돌려준다. */
export function authorize(
  request: Request,
  menuCode?: string,
  action: Action = "READ",
): AuthResult {
  const user = findUserByToken(request);
  if (!user) return { error: problem(401, "로그인이 필요합니다.") };
  if (menuCode && !actionsOf(user, menuCode).includes(action)) {
    return { error: problem(403, "권한이 없습니다.") };
  }
  return { user };
}
```

로그인 API입니다.

📄 **`src/mocks/handlers/auth.ts`** (새 파일)

```ts
import { http, HttpResponse } from "msw";
import { authorize, createToken } from "@/mocks/auth";
import { credentialsTable, usersTable } from "@/mocks/db";
import { apiUrl, problem } from "@/mocks/utils";

interface LoginBody {
  username: string;
  password: string;
}

export const authHandlers = [
  http.post(apiUrl("/auth/login"), async ({ request }) => {
    const { username, password } = (await request.json()) as LoginBody;
    const user = usersTable.all().find((item) => item.username === username && item.useYn === "Y");
    const credential = credentialsTable.all().find((item) => item.username === username);
    // 아이디가 틀렸는지 비밀번호가 틀렸는지 알려 주지 않는다. (계정 존재 여부 노출 방지)
    if (!user || credential?.password !== password) {
      return problem(401, "아이디 또는 비밀번호가 올바르지 않습니다.");
    }
    return HttpResponse.json({ accessToken: createToken(user.id) });
  }),

  http.post(apiUrl("/auth/logout"), () => new HttpResponse(null, { status: 204 })),

  http.get(apiUrl("/me"), ({ request }) => {
    const { user, error } = authorize(request);
    if (error) return error;
    return HttpResponse.json(user);
  }),
];
```

✏️ **`src/mocks/handlers.ts`** — 등록합니다.

```ts
import { menuHandlers } from "@/mocks/handlers/menus";
```

바꿀 코드

```ts
import { authHandlers } from "@/mocks/handlers/auth";
import { menuHandlers } from "@/mocks/handlers/menus";
```

✏️ **`src/mocks/handlers.ts`** — 배열에 추가합니다.

```ts
  ...userHandlers,
  ...menuHandlers,
];
```

바꿀 코드

```ts
  ...authHandlers,
  ...userHandlers,
  ...menuHandlers,
];
```

### 코드 읽어 보기: 구별된 유니온으로 결과 돌려주기

`authorize`는 성공이면 `{ user }`, 실패면 `{ error }`를 돌려줍니다. 쓰는 쪽은 이렇게 씁니다.

```ts
const { user, error } = authorize(request, "USER_MGMT", "DELETE");
if (error) return error; // 여기서 걸러지면
user.name; // 이 아래에서 TypeScript는 user가 반드시 있다고 안다
```

Part 6의 메뉴 타입과 같은 원리입니다. `error`가 있는지로 타입이 좁혀집니다.

### 확인하기

- [ ] F12 → Console:

  ```js
  await (await fetch("/api/auth/login", { method: "POST", body: JSON.stringify({ username: "admin", password: "admin1234" }) })).json();
  ```

  `{ accessToken: "mock-token-1" }`이 나옵니다. 비밀번호를 틀리면 `status: 401` 응답이 옵니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 가짜 로그인 API와 역할·권한 데이터 추가"
```

---

## Step 8-2. 토큰 저장소와 인증 헤더

> 🎯 **할 일**: 받은 토큰을 저장하고, 모든 요청에 자동으로 붙입니다.

### 원리: 토큰은 Zustand, 사용자 정보는 Query

| 데이터           | 어디에             | 이유                                          |
| ---------------- | ------------------ | --------------------------------------------- |
| 토큰             | Zustand (저장소)   | 앱이 직접 만든 상태. 새로고침해도 남아야 함   |
| 내 이름·역할 등  | Query (`useMe`)    | 원본이 서버. 관리자가 바꾸면 다시 받아야 함   |

**토큰을 어디에 저장하나?** 이 템플릿은 **sessionStorage**에 둡니다. 브라우저 탭을 닫으면 로그아웃됩니다.

| 저장 위치         | 장점                         | 단점                                              |
| ----------------- | ---------------------------- | ------------------------------------------------- |
| sessionStorage    | 탭을 닫으면 사라짐. 단순     | JS로 읽을 수 있어 XSS에 취약                      |
| localStorage      | 창을 닫아도 유지             | JS로 읽을 수 있어 XSS에 취약, 공용 PC에서 위험    |
| **httpOnly 쿠키** | JS로 읽을 수 없어 가장 안전  | 서버 설정 필요 (CSRF 대책 포함)                   |

> ⚠️ **주의** 실제 프로젝트에서는 **httpOnly 쿠키**를 권장합니다. 그때는 `httpClient`에서 헤더를 붙이는 대신 `credentials: "include"`를 넣으면 됩니다.
> 계층을 나눠 둔 덕분에 그 한 파일만 바꾸면 됩니다.

### 따라 하기

📄 **`src/shared/auth/authStore.ts`** (새 파일)

```ts
import { create } from "zustand";
import { createJSONStorage, persist } from "zustand/middleware";

interface AuthState {
  accessToken: string | null;
  loggedOutByUser: boolean; // 사용자가 직접 로그아웃했는가 (로그인 만료와 구별한다)
  login: (accessToken: string) => void;
  logout: (byUser?: boolean) => void;
}

// 토큰만 둔다. 사용자 정보는 서버 상태이므로 useMe(Query)로 받는다.
export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      accessToken: null,
      loggedOutByUser: false,
      login: (accessToken) => set({ accessToken, loggedOutByUser: false }),
      logout: (byUser = false) => set({ accessToken: null, loggedOutByUser: byUser }),
    }),
    { name: "auth", storage: createJSONStorage(() => sessionStorage) },
  ),
);
```

✏️ **`src/shared/api/httpClient.ts`** — 토큰을 붙입니다.

```ts
async function request<T>(method: string, url: string, options: RequestOptions = {}): Promise<T> {
  const response = await fetch(BASE_URL + url + toQueryString(options.params), {
    method,
    headers: { "Content-Type": "application/json", "Accept-Language": i18n.language },
```

바꿀 코드

```ts
async function request<T>(method: string, url: string, options: RequestOptions = {}): Promise<T> {
  const token = useAuthStore.getState().accessToken;
  const response = await fetch(BASE_URL + url + toQueryString(options.params), {
    method,
    headers: {
      "Content-Type": "application/json",
      "Accept-Language": i18n.language,
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
    },
```

✏️ **`src/shared/api/httpClient.ts`** — 401이면 로그아웃시킵니다.

```ts
  if (!response.ok) {
```

바꿀 코드

```ts
  // 로그인이 만료되면 토큰을 지운다. 화면 이동은 RequireAuth가 상태를 보고 알아서 한다.
  if (response.status === 401) useAuthStore.getState().logout();

  if (!response.ok) {
```

✏️ **`src/shared/api/httpClient.ts`** — import를 추가합니다.

```ts
import type { ProblemDetail } from "@/shared/api/types";
```

바꿀 코드

```ts
import type { ProblemDetail } from "@/shared/api/types";
import { useAuthStore } from "@/shared/auth/authStore";
```

만료 안내 문구를 추가합니다.

✏️ **`src/shared/i18n/locales/ko/errors.json`** — 문구를 추가합니다.

```json
    "forbidden": "권한이 없습니다."
```

바꿀 코드

```json
    "forbidden": "권한이 없습니다.",
    "unauthorized": "로그인이 만료되었습니다. 다시 로그인해 주세요."
```

✏️ **`src/shared/i18n/locales/en/errors.json`** — 문구를 추가합니다.

```json
    "forbidden": "You do not have permission."
```

바꿀 코드

```json
    "forbidden": "You do not have permission.",
    "unauthorized": "Your session has expired. Please sign in again."
```

✏️ **`src/shared/api/showApiError.ts`** — 401 안내를 추가합니다.

```ts
  if (error.status === 403) {
```

바꿀 코드

```ts
  if (error.status === 401) {
    antdStatic.message?.warning(i18n.t("errors:api.unauthorized"));
    return;
  }
  if (error.status === 403) {
```

### 코드 읽어 보기

**401에서 화면 이동을 하지 않는 이유**: `httpClient`는 화면을 모릅니다(계층 규칙). 토큰만 지우면 다음 Step의 `RequireAuth`가
"토큰이 없네?" 하고 로그인 화면으로 보냅니다. **상태를 바꾸면 화면이 따라오는** React의 방식입니다.

> ⚠️ **주의** 아직 로그인 화면이 없어서, 지금 앱을 열면 사용자 목록 조회가 실패합니다. 사용자 API는 아직 권한 검사를 안 하므로 실제로는 보이지만,
> 다음 Step까지 진행해야 정상 흐름이 됩니다. 커밋 지점에서 잠깐 어색한 상태가 생기는 것은 괜찮습니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 인증 토큰 저장소 및 요청 헤더 추가"
```

---

## Step 8-3. 로그인 화면과 로그인 보호

> 🎯 **할 일**: 로그인 화면을 만들고, 로그인하지 않으면 앱에 들어올 수 없게 합니다.

### 원리

```
① 아무 주소나 접속 → RequireAuth: 토큰 없음 → /login (가려던 주소를 기억해 둔다)
② 로그인 성공 → 토큰 저장 → 기억해 둔 주소로 (없으면 시작 화면)
③ 이후 모든 요청에 토큰이 붙는다
```

**로그인 화면에 들어오면 이전 사람의 흔적을 지웁니다.** Query 캐시와 열린 탭입니다.
로그아웃 버튼이든 로그인 만료(401)든 결국 로그인 화면으로 오므로, 정리하는 곳이 한 곳이면 빠뜨릴 일이 없습니다.

> ⚠️ **주의** Query 캐시를 지우지 않으면, A가 로그아웃하고 B가 로그인했을 때 **A가 보던 데이터가 B에게 잠깐 보입니다.** 공용 PC가 많은 SI 현장에서는 보안 사고입니다.

### 따라 하기

로그인 직후 내 메뉴를 미리 받아서 시작 화면 주소를 알아내야 합니다. 그래서 Query 설정을 훅 밖에서도 쓸 수 있게 분리합니다.
동시에 반환 타입을 권한이 붙은 `MyMenu`로 바꿉니다.

📄 **`src/shared/menu/useMyMenus.ts`** (전체 교체)

```ts
import { queryOptions, useQuery } from "@tanstack/react-query";
import { httpClient } from "@/shared/api/httpClient";
import type { MyMenu } from "@/shared/menu/types";

// queryOptions로 설정을 한 곳에 묶어 두면 훅(useQuery)에서도, 훅 밖(queryClient.query)에서도 똑같이 쓸 수 있다.
export const myMenusQuery = queryOptions({
  queryKey: ["me", "menus"],
  queryFn: () => httpClient.get<MyMenu[]>("/me/menus"),
  // 메뉴는 메뉴·권한 관리에서 저장할 때만 바뀐다. 그때 직접 무효화하므로 저절로 낡지 않게 한다.
  staleTime: Infinity,
});

export const myMenusKey = myMenusQuery.queryKey;

/** 내가 쓸 수 있는 메뉴 목록 + 메뉴별 권한. 사이드바·탭·화면 제목·권한이 모두 이 데이터 하나를 본다. */
export function useMyMenus() {
  return useQuery(myMenusQuery);
}
```

문구 파일입니다.

📄 **`src/shared/i18n/locales/ko/auth.json`** (새 파일)

```json
{
  "title": "로그인",
  "username": "아이디",
  "password": "비밀번호",
  "submit": "로그인",
  "loginFailed": "아이디 또는 비밀번호가 올바르지 않습니다.",
  "devAccounts": "개발용 계정 (개발 모드에서만 보입니다)",
  "profile": "내 정보 수정",
  "logout": "로그아웃"
}
```

📄 **`src/shared/i18n/locales/en/auth.json`** (새 파일)

```json
{
  "title": "Sign in",
  "username": "User ID",
  "password": "Password",
  "submit": "Sign in",
  "loginFailed": "Invalid user ID or password.",
  "devAccounts": "Test accounts (visible in development only)",
  "profile": "Edit profile",
  "logout": "Sign out"
}
```

로그인 API 함수입니다. 로그인 화면만 쓰므로 `features/auth`에 둡니다.

📄 **`src/features/auth/authApi.ts`** (새 파일)

```ts
import { httpClient } from "@/shared/api/httpClient";

export interface LoginRequest {
  username: string;
  password: string;
}

export const authApi = {
  login: (body: LoginRequest) => httpClient.post<{ accessToken: string }>("/auth/login", body),
};
```

로그인 화면입니다. 메뉴에 연결되면 안 되는 화면이라 `pages/` 폴더 **밖**에 둡니다. (화면 레지스트리가 `pages/` 폴더만 찾기 때문입니다.)

📄 **`src/features/auth/LoginPage.tsx`** (새 파일)

```tsx
import { useEffect } from "react";
import { useTranslation } from "react-i18next";
import { useLocation, useNavigate } from "react-router";
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { Alert, Button, Card, Flex, Form, Input, Space, Typography, theme } from "antd";
import { LockOutlined, UserOutlined } from "@ant-design/icons";
import { authApi, type LoginRequest } from "@/features/auth/authApi";
import { ApiError } from "@/shared/api/httpClient";
import { useAuthStore } from "@/shared/auth/authStore";
import { myMenusQuery } from "@/shared/menu/useMyMenus";
import { START_MENU_CODE, useTabStore } from "@/shared/tabs/tabStore";

const DEV_ACCOUNTS = ["admin", "manager", "user"];

function LoginPage() {
  const { t } = useTranslation("auth");
  const { token } = theme.useToken();
  const navigate = useNavigate();
  const location = useLocation();
  const queryClient = useQueryClient();
  const login = useAuthStore((state) => state.login);
  const [form] = Form.useForm<LoginRequest>();

  // 로그인 화면에 오면 이전 사람의 흔적(데이터 캐시, 열린 탭)을 지운다.
  // 로그아웃 버튼이든 로그인 만료(401)든 결국 여기로 오므로 한 곳에서 처리한다.
  useEffect(() => {
    queryClient.clear();
    const { tabs, close } = useTabStore.getState();
    close(tabs.map((tab) => tab.key));
  }, [queryClient]);

  const loginMutation = useMutation({
    mutationFn: authApi.login,
    meta: { skipGlobalError: true }, // 실패 안내는 이 화면이 직접 한다
    onSuccess: async ({ accessToken }) => {
      login(accessToken);
      // 가려던 주소가 있으면 그리로, 없으면 시작 화면으로 간다. 시작 화면 주소는 내 메뉴에서 찾는다.
      const from = (location.state as { from?: string } | null)?.from;
      const menus = await queryClient.query(myMenusQuery); // 캐시에 있으면 그대로, 없으면 받아 온다
      const start = menus.find((menu) => menu.code === START_MENU_CODE);
      const startPath = start?.type === "PAGE" ? start.path : "/";
      navigate(from && from !== "/" ? from : startPath, { replace: true });
    },
  });

  // 401은 아이디·비밀번호가 틀린 것이고, 그 밖의 실패(서버 오류·연결 실패)는 서버 문제로 안내한다.
  // "HTTP 502" 같은 내부 메시지를 사용자에게 그대로 보여 주지 않는다.
  const error = loginMutation.error;
  const errorMessage = !error
    ? undefined
    : error instanceof ApiError && error.status === 401
      ? t("loginFailed")
      : t("errors:api.server");

  return (
    <Flex
      align="center"
      justify="center"
      style={{ height: "100%", background: token.colorBgLayout }}
    >
      <Card style={{ width: 360 }}>
        <Typography.Title level={3} style={{ textAlign: "center" }}>
          {import.meta.env.VITE_APP_TITLE}
        </Typography.Title>
        <Form form={form} layout="vertical" onFinish={loginMutation.mutate} requiredMark={false}>
          <Form.Item name="username" label={t("username")} rules={[{ required: true }]}>
            <Input prefix={<UserOutlined />} autoComplete="username" autoFocus />
          </Form.Item>
          <Form.Item name="password" label={t("password")} rules={[{ required: true }]}>
            <Input.Password prefix={<LockOutlined />} autoComplete="current-password" />
          </Form.Item>
          {errorMessage && (
            <Alert type="error" title={errorMessage} showIcon style={{ marginBottom: 16 }} />
          )}
          <Button type="primary" htmlType="submit" block loading={loginMutation.isPending}>
            {t("submit")}
          </Button>
        </Form>
        {/* import.meta.env.DEV는 운영 빌드에서 false로 바뀌어 이 블록이 통째로 빠진다 */}
        {import.meta.env.DEV && (
          <div style={{ marginTop: 24 }}>
            <Typography.Text type="secondary">{t("devAccounts")}</Typography.Text>
            <Space wrap style={{ marginTop: 8 }}>
              {DEV_ACCOUNTS.map((username) => (
                <Button
                  key={username}
                  size="small"
                  onClick={() => {
                    form.setFieldsValue({ username, password: `${username}1234` });
                    form.submit();
                  }}
                >
                  {username}
                </Button>
              ))}
            </Space>
          </div>
        )}
      </Card>
    </Flex>
  );
}

export default LoginPage;
```

로그인하지 않았으면 로그인 화면으로 보내는 컴포넌트입니다.

📄 **`src/shared/auth/RequireAuth.tsx`** (새 파일)

```tsx
import type { ReactNode } from "react";
import { Navigate, useLocation } from "react-router";
import { useAuthStore } from "@/shared/auth/authStore";

interface RequireAuthProps {
  children: ReactNode;
}

/** 토큰이 없으면 로그인 화면으로 보낸다. 가려던 주소는 state에 담아 둔다. */
function RequireAuth({ children }: RequireAuthProps) {
  const accessToken = useAuthStore((state) => state.accessToken);
  const loggedOutByUser = useAuthStore((state) => state.loggedOutByUser);
  const location = useLocation();

  if (!accessToken) {
    // 북마크로 들어왔거나 로그인이 만료됐으면, 다시 로그인한 뒤 보던 화면으로 돌아간다.
    // 직접 로그아웃했다면 기억하지 않는다. 다음 사람이 이전 사람의 화면으로 가면 안 되기 때문이다.
    const from = loggedOutByUser ? undefined : location.pathname + location.search;
    return <Navigate to="/login" replace state={{ from }} />;
  }
  return children;
}

export default RequireAuth;
```

📄 **`src/app/App.tsx`** (전체 교체)

```tsx
import { Route, Routes } from "react-router";
import MainLayout from "@/app/layout/MainLayout";
import LoginPage from "@/features/auth/LoginPage";
import RequireAuth from "@/shared/auth/RequireAuth";

function App() {
  return (
    <Routes>
      <Route path="/login" element={<LoginPage />} />
      {/* 그 외 모든 주소는 로그인이 필요하다. 어떤 화면을 보여 줄지는 메뉴 데이터와 탭이 정한다. */}
      <Route
        path="*"
        element={
          <RequireAuth>
            <MainLayout />
          </RequireAuth>
        }
      />
    </Routes>
  );
}

export default App;
```

### 코드 읽어 보기

**`loginMutation.mutate`를 `onFinish`에 바로 넘기기**: `onFinish`는 폼 값(`{ username, password }`)을 넘겨주고, `mutate`는 그 값을 `mutationFn`에 넘깁니다. 모양이 같아서 그대로 연결됩니다.

**`autoComplete`**: 브라우저의 비밀번호 저장 기능이 정확히 동작하게 해 줍니다. 로그인 폼에는 꼭 넣습니다.

**`loggedOutByUser`**: "가려던 주소를 기억했다가 돌려보내기"는 북마크나 로그인 만료 때는 고마운 기능이지만,
A가 로그아웃하고 B가 로그인했을 때 A가 보던 화면으로 가면 안 됩니다. 그래서 **왜 로그아웃됐는지**를 상태로 남겨 둡니다.

> 💡 **팁** 처음에는 "로그아웃 버튼에서 먼저 `/login`으로 이동한 뒤 토큰을 지우면 되지 않을까?"라고 생각하기 쉽습니다. 그런데 React Router의 이동은
> 낮은 우선순위로 처리되어서, 토큰 삭제가 먼저 화면에 반영되면 `RequireAuth`가 이전 주소를 기억해 버립니다. **실행 순서에 기대지 말고 의도를 상태로 표현하는 것**이 안전합니다.

**`import.meta.env.DEV`**: 개발 서버에서는 `true`, 운영 빌드에서는 `false`입니다. 빌드할 때 값이 박히므로 `false` 쪽 코드는 아예 빠집니다. 테스트 계정 버튼이 운영에 나갈 일이 없습니다.

### 확인하기

- [ ] `http://localhost:5173/system/users`로 접속 → 로그인 화면으로 이동합니다.
- [ ] 비밀번호를 틀리면 "아이디 또는 비밀번호가 올바르지 않습니다."
- [ ] [admin] 버튼 → 로그인되고 원래 가려던 **사용자 관리**가 열립니다.
- [ ] F12 → Network → 아무 요청 → Request Headers에 `Authorization: Bearer mock-token-1`
- [ ] F12 → Application → Session Storage → `auth`에 토큰이 보입니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 로그인 화면 및 로그인 보호 추가"
```

---

## Step 8-4. 사용자 메뉴와 로그아웃

> 🎯 **할 일**: 헤더에 로그인한 사람의 이름과 로그아웃 메뉴를 보여 줍니다.

### 따라 하기

📄 **`src/shared/auth/useMe.ts`** (새 파일)

```ts
import { useQuery } from "@tanstack/react-query";
import { httpClient } from "@/shared/api/httpClient";
import type { Me } from "@/shared/auth/types";

/** 로그인한 내 정보. 서버 데이터이므로 Zustand가 아니라 Query로 받는다. */
export function useMe() {
  return useQuery({
    queryKey: ["me", "profile"],
    queryFn: () => httpClient.get<Me>("/me"),
    staleTime: Infinity, // 내 정보 수정 화면에서 저장할 때 무효화한다
  });
}
```

📄 **`src/shared/auth/useLogout.ts`** (새 파일)

```ts
import { httpClient } from "@/shared/api/httpClient";
import { useAuthStore } from "@/shared/auth/authStore";

/** 로그아웃. 화면 이동은 RequireAuth가, 캐시·탭 정리는 로그인 화면이 한다. */
export function useLogout() {
  return async () => {
    await httpClient.post("/auth/logout").catch(() => undefined); // 서버 쪽 정리. 실패해도 로그아웃은 진행한다
    useAuthStore.getState().logout(true); // "직접 로그아웃"으로 표시한다
  };
}
```

📄 **`src/app/layout/UserMenu.tsx`** (새 파일)

```tsx
import { useTranslation } from "react-i18next";
import { Button, Dropdown } from "antd";
import { LogoutOutlined, ProfileOutlined, UserOutlined } from "@ant-design/icons";
import { useLogout } from "@/shared/auth/useLogout";
import { useMe } from "@/shared/auth/useMe";
import { useOpenPage } from "@/shared/page/useOpenPage";

function UserMenu() {
  const { t } = useTranslation("auth");
  const { data: me } = useMe();
  const logout = useLogout();
  const openPage = useOpenPage();

  return (
    <Dropdown
      menu={{
        items: [
          { key: "profile", icon: <ProfileOutlined />, label: t("profile") },
          { type: "divider" },
          { key: "logout", icon: <LogoutOutlined />, label: t("logout") },
        ],
        onClick: ({ key }) => {
          if (key === "profile") openPage("MY_PROFILE");
          if (key === "logout") logout();
        },
      }}
    >
      <Button type="text" icon={<UserOutlined />}>
        {me?.name}
      </Button>
    </Dropdown>
  );
}

export default UserMenu;
```

✏️ **`src/app/layout/AppHeader.tsx`** — 맨 오른쪽에 넣습니다.

```tsx
      <SettingsDrawer open={settingsOpen} onClose={() => setSettingsOpen(false)} />
```

바꿀 코드

```tsx
      <SettingsDrawer open={settingsOpen} onClose={() => setSettingsOpen(false)} />
      <UserMenu />
```

✏️ **`src/app/layout/AppHeader.tsx`** — import를 추가합니다.

```tsx
import SettingsDrawer from "@/app/layout/SettingsDrawer";
```

바꿀 코드

```tsx
import SettingsDrawer from "@/app/layout/SettingsDrawer";
import UserMenu from "@/app/layout/UserMenu";
```

### 확인하기

- [ ] 헤더 오른쪽에 `김민수`가 보입니다. 누르면 메뉴가 나옵니다.
- [ ] 사용자 관리를 연 채로 로그아웃 → 로그인 화면. Session Storage의 `auth` 토큰이 `null`, `tabs`가 비었습니다.
- [ ] 다시 로그인하면 사용자 관리가 아니라 **시작 화면(대시보드)** 이 열립니다. 이전 사람의 화면으로 가지 않습니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 사용자 메뉴와 로그아웃 추가"
```

---

## Step 8-5. 역할별 메뉴와 서버 권한 검사

> 🎯 **할 일**: 로그인한 사람의 역할에 따라 메뉴가 달라지게 하고, 가짜 서버도 권한을 검사하게 합니다.

### 원리

`/api/me/menus`가 **권한 있는 메뉴만** 내려주고, 메뉴마다 할 수 있는 동작(`actions`)을 붙입니다.

```json
[
  { "code": "SYSTEM", "type": "FOLDER", "actions": [] },
  { "code": "USER_MGMT", "type": "PAGE", "path": "/system/users", "actions": ["READ", "EXPORT"] }
]
```

- 폴더는 권한을 따로 주지 않습니다. 하위 화면 중 하나라도 볼 수 있으면 서버가 폴더도 함께 내려줍니다.
- 역할이 여러 개면 동작을 **합집합**으로 합칩니다.
- 메뉴와 권한을 한 번에 받으므로 사이드바·탭·버튼이 **같은 데이터 하나**를 봅니다.

앞에서 만든 화면들은 이미 이 데이터만 보고 있으므로 **화면 코드는 고칠 게 없습니다.** 서버만 바꾸면 됩니다.

### 따라 하기

📄 **`src/mocks/handlers/menus.ts`** (전체 교체)

```ts
import { http, HttpResponse } from "msw";
import { actionsOf, authorize } from "@/mocks/auth";
import { menusTable } from "@/mocks/db";
import { apiUrl } from "@/mocks/utils";

export const menuHandlers = [
  // 내가 쓸 수 있는 메뉴 + 메뉴별 동작
  http.get(apiUrl("/me/menus"), ({ request }) => {
    const { user, error } = authorize(request);
    if (error) return error;

    const menus = menusTable.all().filter((menu) => menu.useYn === "Y");
    // 화면·링크 메뉴 중 조회(READ) 권한이 있는 것
    const allowed = menus
      .filter((menu) => menu.type !== "FOLDER")
      .map((menu) => ({ ...menu, actions: actionsOf(user, menu.code) }))
      .filter((menu) => menu.actions.includes("READ"));

    // 그 메뉴들의 상위 폴더를 모두 찾는다
    const folderIds = new Set<number>();
    const addParents = (parentId: number | null) => {
      const parent = menus.find((menu) => menu.id === parentId);
      if (!parent || folderIds.has(parent.id)) return;
      folderIds.add(parent.id);
      addParents(parent.parentId);
    };
    allowed.forEach((menu) => addParents(menu.parentId));
    const folders = menus
      .filter((menu) => folderIds.has(menu.id))
      .map((menu) => ({ ...menu, actions: [] }));

    return HttpResponse.json([...folders, ...allowed]);
  }),
];
```

사용자 목록 API에도 권한 검사를 붙입니다. 앞으로 만드는 모든 가짜 API는 이렇게 첫 줄에서 검사합니다.

✏️ **`src/mocks/handlers/users.ts`** — 권한을 검사합니다.

```ts
  http.get(apiUrl("/users"), ({ request }) => {
    const params = new URL(request.url).searchParams;
```

바꿀 코드

```ts
  http.get(apiUrl("/users"), ({ request }) => {
    const { error } = authorize(request, "USER_MGMT", "READ");
    if (error) return error;

    const params = new URL(request.url).searchParams;
```

✏️ **`src/mocks/handlers/users.ts`** — import를 추가합니다.

```ts
import { usersTable } from "@/mocks/db";
```

바꿀 코드

```ts
import { authorize } from "@/mocks/auth";
import { usersTable } from "@/mocks/db";
```

권한이 없는 주소로 직접 들어왔을 때의 안내도 고칩니다. "권한이 없다"고 말하면 그런 화면이 있다는 사실이 드러나므로,
**"없거나 권한이 없다"** 로 묶어서 말합니다.

✏️ **`src/app/errors/NotFoundPage.tsx`** — 안내 문구를 바꿉니다.

```tsx
      subTitle={t("page.notFound")}
```

바꿀 코드

```tsx
      subTitle={t("page.notFoundOrForbidden")}
```

### 확인하기

- [ ] `user`로 로그인 → 사이드바에 대시보드, 게시판 > 공지사항, 샘플 > 알림 발생 테스트, 내 정보, 사용자 매뉴얼만 보입니다.
- [ ] `user`로 로그인한 채 주소창에 `/system/users` 입력 → "화면이 없거나 접근 권한이 없습니다."
- [ ] `manager`로 로그인 → 시스템 관리에 사용자 관리, 공통코드 관리만 보입니다.
- [ ] F12 → Network → `menus` 응답에서 메뉴마다 `actions`가 붙어 있습니다.

### 스스로 답해 보기

<details>
<summary>Q. 화면 코드를 하나도 안 고쳤는데 메뉴가 달라진 이유는?</summary>

사이드바·탭·화면 진입이 모두 `useMyMenus()` 데이터 하나만 보고 있기 때문입니다. 데이터가 바뀌면 화면이 따라옵니다.
Part 6에서 "화면은 데이터가 정한다"고 설계해 둔 덕분입니다.

</details>

✅ **커밋**

```powershell
git add .
git commit -m "feat: 역할별 메뉴 접근 권한 적용"
```

---

## Step 8-6. 버튼 권한

> 🎯 **할 일**: 권한에 따라 버튼을 숨기는 도구를 만듭니다.

### 따라 하기

📄 **`src/shared/auth/usePermission.ts`** (새 파일)

```ts
import { use } from "react";
import type { Action } from "@/shared/menu/types";
import { useMyMenus } from "@/shared/menu/useMyMenus";
import { PageContext } from "@/shared/page/PageContext";

/**
 * 메뉴에서 내가 할 수 있는 동작을 확인한다.
 * menuCode를 생략하면 지금 화면의 메뉴를 쓴다. 탭 밖(헤더 등)에서는 menuCode를 꼭 넘긴다.
 */
export function usePermission(menuCode?: string) {
  const page = use(PageContext); // 탭 밖이면 null
  const code = menuCode ?? page?.menuCode;
  const { data: menus = [] } = useMyMenus();
  const actions = menus.find((menu) => menu.code === code)?.actions ?? [];
  return { can: (action: Action) => actions.includes(action) };
}
```

📄 **`src/shared/auth/Can.tsx`** (새 파일)

```tsx
import type { ReactNode } from "react";
import { usePermission } from "@/shared/auth/usePermission";
import type { Action } from "@/shared/menu/types";

interface CanProps {
  action: Action;
  menuCode?: string;
  children: ReactNode;
}

/** 권한이 있을 때만 children을 그린다. */
function Can({ action, menuCode, children }: CanProps) {
  const { can } = usePermission(menuCode);
  return can(action) ? children : null;
}

export default Can;
```

사용자 화면에 [등록] 버튼을 붙여 봅니다. 동작은 Part 9에서 연결합니다.

✏️ **`src/features/users/pages/UsersPage.tsx`** — 화면 틀에 버튼을 넣습니다.

```tsx
    <PageContainer>
```

바꿀 코드

```tsx
    <PageContainer
      extra={
        <Can action="CREATE">
          <Button type="primary">{t("common:button.create")}</Button>
        </Can>
      }
    >
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — import를 추가합니다.

```tsx
import type { User, UserSearchCondition } from "@/features/users/types";
```

바꿀 코드

```tsx
import type { User, UserSearchCondition } from "@/features/users/types";
import Can from "@/shared/auth/Can";
```

### 숨김 vs 비활성

프로젝트마다 정책이 다릅니다. 기본은 숨김(`<Can>`)으로 하고, "있는 줄은 알아야 한다"는 요구가 있으면 비활성으로 합니다.

```tsx
const { can } = usePermission();
<Button danger disabled={!can("DELETE")}>삭제</Button>
```

### 확인하기

- [ ] `admin`으로 로그인 → 사용자 관리에 [등록] 버튼이 보입니다.
- [ ] `manager`로 로그인 → 사용자 관리에 [등록] 버튼이 **없습니다.** (`USER_MGMT`에 `READ`, `EXPORT`만 있음)

### 스스로 답해 보기

<details>
<summary>Q. 버튼을 숨겼으니 이제 안전한가요?</summary>

아닙니다. 버튼은 편의일 뿐, 요청은 개발자 도구나 다른 도구로 얼마든지 보낼 수 있습니다.
진짜 보호는 서버가 모든 API에서 권한을 검사하는 것입니다. 가짜 서버의 `authorize()`가 그 역할을 연습합니다.

</details>

✅ **커밋**

```powershell
git add .
git commit -m "feat: 버튼 단위 권한 제어 추가"
```

---

## 🏁 Part 8을 마치며

마일스톤입니다. 로그인한 사람에 따라 메뉴와 버튼이 달라집니다.

| 증상                                                          | 원인과 해결                                              |
| ------------------------------------------------------------- | -------------------------------------------------------- |
| 다른 계정으로 로그인했는데 이전 사람 목록이 잠깐 보인다       | 로그인 화면에서 `queryClient.clear()`를 하지 않았다      |
| 권한을 바꿨는데 화면이 그대로다                               | 저장 후 `["me", "menus"]` 무효화 누락 (Part 10)          |
| 사용자 정보를 Zustand에 복사해 뒀더니 이름을 바꿔도 안 바뀐다 | 서버 데이터는 Query(`useMe`)로                           |
| `usePermission`이 헤더에서 항상 false                         | 탭 밖에서는 `menuCode`를 직접 넘겨야 한다                |

| 보안 체크리스트    | 이 템플릿                                     | 실제 프로젝트                                     |
| ------------------ | --------------------------------------------- | ------------------------------------------------- |
| 토큰 저장 위치     | sessionStorage                                | httpOnly 쿠키 권장                                |
| 로그인 실패 메시지 | 어느 쪽이 틀렸는지 알려주지 않음              | 같음. 실패 횟수 잠금 추가                         |
| 로그아웃           | 토큰 + Query 캐시 + 탭 정리                   | 서버 세션·리프레시 토큰도 폐기                    |
| 테스트 계정 버튼   | `import.meta.env.DEV`일 때만                  | 운영 빌드에 절대 포함 금지                        |
| 권한 검사          | 가짜 서버도 401·403                           | 서버가 모든 API에서 검사                          |

다음은 등록·수정·삭제입니다. SI 화면의 기본기를 익힙니다. → [Part 9. 업무 화면 기본기 (CRUD)](part-09-crud.md)
