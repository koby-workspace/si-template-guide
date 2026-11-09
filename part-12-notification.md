# Part 12. 알림

> **이 Part의 목표**: 헤더(Top 영역)의 🔔에 **미확인 공지**와 **업무 알림** 개수를 표시하고, 특정 화면에서 특정 이벤트가 일어나면
> **정해진 대상**(역할 또는 사람)에게 알림이 가게 합니다. 누구에게 무엇을 보낼지는 관리자가 화면에서 **규칙**으로 정합니다.

---

## 먼저 큰 그림

```
[이벤트 발생]                         [규칙]                                  [알림]
사용자 관리 화면에서 사용자 등록  ─→  USER_CREATED → ADMIN 역할에게      ─→  관리자들의 알림함에 1건씩
알림 테스트 화면에서 결재 요청    ─→  SAMPLE_APPROVAL_REQUESTED → MANAGER
                                       제목 템플릿: "결재 요청: {{item}}"
```

| 개념        | 뜻                                                                       | 누가 정하나          |
| ----------- | ------------------------------------------------------------------------ | -------------------- |
| **이벤트**  | "무슨 일이 일어났다". 코드(`USER_CREATED`)와 발생 화면, 쓸 수 있는 변수 | 개발자 (코드)        |
| **규칙**    | "그 일이 일어나면 누구에게 어떤 문구로 알린다"                           | 관리자 (화면)        |
| **알림**    | 사람 한 명에게 간 메시지 한 건. 읽음 여부를 가진다                       | 서버가 규칙대로 생성 |

**이벤트를 발생시키는 방법은 두 가지**입니다.

1. **서버가 업무 처리 중에** 발생시킨다. 예: 사용자 등록 API가 저장 후 `USER_CREATED`를 발생 (가장 일반적)
2. **화면이 직접** 발생시킨다. 예: 화면에서 `emitEvent("SAMPLE_APPROVAL_REQUESTED", { item })` 한 줄

**도착은 어떻게 아나요?** 이 템플릿은 15초마다 서버에 "내 미확인 알림"을 묻습니다(**폴링**). 가장 단순하고 어떤 서버와도 동작합니다.
나중에 실시간이 필요하면 SSE·WebSocket으로 바꿀 수 있게 **한 파일에 가둬** 둡니다. (Step 12-7)

---

## Step 12-1. 알림 데이터와 발송 엔진

> 🎯 **할 일**: 이벤트 목록, 알림 규칙, 알림 데이터를 만들고, 이벤트가 들어오면 규칙대로 알림을 만드는 "발송 엔진"을 가짜 서버에 만듭니다.

### 따라 하기

타입입니다. 헤더(🔔)가 모든 화면에서 쓰므로 `shared`에 둡니다.

📄 **`src/shared/notification/types.ts`** (새 파일)

```ts
import type { UseYn } from "@/shared/api/types";
import type { LocalizedText } from "@/shared/i18n/locales";

// 브라우저에 이미 Notification이라는 타입이 있어서 이름을 AppNotification으로 짓는다.
export interface AppNotification {
  id: number;
  userId: number; // 받는 사람
  eventCode: string; // 어떤 이벤트로 생겼나
  title: string;
  message: string;
  linkMenuCode?: string; // 누르면 열 화면
  linkParams?: Record<string, string>;
  createdAt: string;
  readAt: string | null; // null이면 안 읽음
}

/** 이벤트 목록(카탈로그). 개발자가 코드로 정의하고 서버가 내려준다. */
export interface NotificationEvent {
  code: string; // 예: "USER_CREATED"
  menuCode: string; // 어느 화면에서 일어나는 이벤트인가. 화면 권한이 있어야 발생시킬 수 있다
  names: LocalizedText;
  variables: string[]; // 문구 템플릿에 쓸 수 있는 변수. 예: ["name", "username"]
}

export type TargetType = "ROLE" | "USER";

/** 알림 규칙. 관리자가 화면에서 정한다. */
export interface NotificationRule {
  id: number;
  name: string;
  eventCode: string;
  targetType: TargetType;
  targets: string[]; // 역할 코드들 또는 사용자 id들
  titleTemplate: string; // 예: "결재 요청: {{item}}"
  messageTemplate: string;
  linkMenuCode?: string;
  useYn: UseYn;
}

export interface UnreadNotice {
  id: number;
  title: string;
  type: string;
  createdAt: string;
}

/** 헤더 🔔에 보여 줄 요약 */
export interface NotificationSummary {
  noticeCount: number;
  notices: UnreadNotice[]; // 최근 몇 건만
  notificationCount: number;
  notifications: AppNotification[]; // 최근 몇 건만
}
```

이벤트 목록과 규칙 시드입니다.

📄 **`src/mocks/data/notifications.ts`** (새 파일)

```ts
import type { NotificationEvent, NotificationRule } from "@/shared/notification/types";

// 이벤트 목록. 새 이벤트가 필요하면 여기(진짜 서버에서는 서버 코드)에 추가한다.
export const notificationEvents: NotificationEvent[] = [
  {
    code: "USER_CREATED",
    menuCode: "USER_MGMT",
    names: { ko: "사용자 등록", en: "User created" },
    variables: ["name", "username"],
  },
  {
    code: "SAMPLE_APPROVAL_REQUESTED",
    menuCode: "SAMPLE_NOTIFY",
    names: { ko: "결재 요청 (샘플)", en: "Approval requested (sample)" },
    variables: ["requester", "item", "amount"],
  },
  {
    code: "SAMPLE_JOB_DONE",
    menuCode: "SAMPLE_NOTIFY",
    names: { ko: "작업 완료 (샘플)", en: "Job finished (sample)" },
    variables: ["job"],
  },
];

export const notificationRuleSeed: NotificationRule[] = [
  {
    id: 1,
    name: "신규 사용자 등록 알림",
    eventCode: "USER_CREATED",
    targetType: "ROLE",
    targets: ["ADMIN"],
    titleTemplate: "새 사용자 등록",
    messageTemplate: "{{name}}({{username}}) 님이 등록되었습니다.",
    linkMenuCode: "USER_MGMT",
    useYn: "Y",
  },
  {
    id: 2,
    name: "결재 요청 알림",
    eventCode: "SAMPLE_APPROVAL_REQUESTED",
    targetType: "ROLE",
    targets: ["MANAGER", "ADMIN"],
    titleTemplate: "결재 요청: {{item}}",
    messageTemplate: "{{requester}} 님이 {{amount}}원 결재를 요청했습니다.",
    linkMenuCode: "SAMPLE_NOTIFY",
    useYn: "Y",
  },
  {
    id: 3,
    name: "작업 완료 알림 (이서연에게)",
    eventCode: "SAMPLE_JOB_DONE",
    targetType: "USER",
    targets: ["3"],
    titleTemplate: "작업 완료",
    messageTemplate: "{{job}} 작업이 끝났습니다.",
    useYn: "Y",
  },
];
```

✏️ **`src/mocks/db.ts`** — import를 추가합니다.

```ts
import { noticeSeed } from "@/mocks/data/notices";
```

바꿀 코드

```ts
import { noticeSeed } from "@/mocks/data/notices";
import { notificationRuleSeed } from "@/mocks/data/notifications";
import type { AppNotification } from "@/shared/notification/types";
```

✏️ **`src/mocks/db.ts`** — 테이블을 추가합니다.

```ts
export const noticesTable = createTable("notices", noticeSeed);
```

바꿀 코드

```ts
export const noticesTable = createTable("notices", noticeSeed);
export const notificationRulesTable = createTable("notificationRules", notificationRuleSeed);
export const notificationsTable = createTable<AppNotification>("notifications", []);
```

발송 엔진입니다. 이벤트 코드와 변수를 받아 규칙을 찾고, 대상 한 명 한 명에게 알림을 만듭니다.

📄 **`src/mocks/notificationEngine.ts`** (새 파일)

```ts
import { notificationRulesTable, notificationsTable, usersTable } from "@/mocks/db";

/** "{{name}} 님" + { name: "홍길동" } → "홍길동 님" */
function fill(template: string, variables: Record<string, string>) {
  return template.replace(/\{\{(\w+)\}\}/g, (_, key: string) => variables[key] ?? "");
}

/**
 * 이벤트를 발생시킨다. 이 이벤트에 걸린 규칙마다 대상자를 찾아 알림을 만들고, 만든 알림 수를 돌려준다.
 * 진짜 서버에서는 이 부분이 서버 코드(서비스 계층)에 있다.
 */
export function emitEvent(eventCode: string, variables: Record<string, string>) {
  const rules = notificationRulesTable
    .all()
    .filter((rule) => rule.eventCode === eventCode && rule.useYn === "Y");
  const now = new Date().toISOString();
  let count = 0;

  for (const rule of rules) {
    // 대상 → 사용자 id 목록. 역할이면 그 역할을 가진 사용 중인 사용자 전부
    const userIds =
      rule.targetType === "ROLE"
        ? usersTable
            .all()
            .filter(
              (user) =>
                user.useYn === "Y" && user.roleCodes.some((code) => rule.targets.includes(code)),
            )
            .map((user) => user.id)
        : rule.targets.map(Number);

    for (const userId of new Set(userIds)) {
      notificationsTable.insert({
        userId,
        eventCode,
        title: fill(rule.titleTemplate, variables),
        message: fill(rule.messageTemplate, variables),
        linkMenuCode: rule.linkMenuCode,
        linkParams: variables.id ? { id: variables.id } : undefined,
        createdAt: now,
        readAt: null,
      });
      count += 1;
    }
  }
  return count;
}
```

> 💡 **팁** 이 템플릿은 이벤트를 일으킨 **본인도** 대상에 포함되면 알림을 받습니다. 혼자서도 동작을 확인하기 쉽게 하기 위해서입니다.
> 실제 프로젝트에서는 "본인은 제외" 같은 규칙 옵션을 추가하는 경우가 많습니다.

가짜 API입니다. "내 알림"과 이벤트 발생, 규칙 관리를 한 파일에 둡니다.

📄 **`src/mocks/handlers/notifications.ts`** (새 파일)

```ts
import { http, HttpResponse } from "msw";
import { actionsOf, authorize } from "@/mocks/auth";
import { notificationEvents } from "@/mocks/data/notifications";
import {
  noticeReadsTable,
  noticesTable,
  notificationRulesTable,
  notificationsTable,
  usersTable,
} from "@/mocks/db";
import { emitEvent } from "@/mocks/notificationEngine";
import { apiUrl, paginate, problem } from "@/mocks/utils";
import type { NotificationRule } from "@/shared/notification/types";

const RULE_MENU = "NOTI_RULE";
const SUMMARY_SIZE = 5; // 🔔에 보여 줄 최근 건수

/** 최신순 정렬 */
const byNewest = <T extends { createdAt: string }>(a: T, b: T) =>
  b.createdAt.localeCompare(a.createdAt);

export const notificationHandlers = [
  // ── 이벤트 ──
  http.get(apiUrl("/notification-events"), ({ request }) => {
    const { error } = authorize(request);
    if (error) return error;
    return HttpResponse.json(notificationEvents);
  }),

  // 화면에서 이벤트를 발생시킨다. 그 이벤트가 일어나는 화면을 쓸 수 있는 사람만 발생시킬 수 있다.
  http.post(apiUrl("/notification-events/:code/emit"), async ({ request, params }) => {
    const event = notificationEvents.find((item) => item.code === params.code);
    if (!event) return problem(404, "없는 이벤트입니다.");
    const { error } = authorize(request, event.menuCode, "READ");
    if (error) return error;
    const { variables } = (await request.json()) as { variables: Record<string, string> };
    return HttpResponse.json({ count: emitEvent(event.code, variables) });
  }),

  // ── 내 알림 ──
  // 🔔 요약: 안 읽은 공지 + 안 읽은 알림. 공지는 공지사항 조회 권한이 있을 때만 센다.
  http.get(apiUrl("/me/notifications/summary"), ({ request }) => {
    const { user, error } = authorize(request);
    if (error) return error;

    const readIds = new Set(
      noticeReadsTable
        .all()
        .filter((item) => item.userId === user.id)
        .map((item) => item.noticeId),
    );
    const unreadNotices = actionsOf(user, "NOTICE").includes("READ")
      ? noticesTable
          .all()
          .filter((notice) => !readIds.has(notice.id))
          .sort(byNewest)
      : [];
    const unreadNotifications = notificationsTable
      .all()
      .filter((item) => item.userId === user.id && !item.readAt)
      .sort(byNewest);

    return HttpResponse.json({
      noticeCount: unreadNotices.length,
      notices: unreadNotices.slice(0, SUMMARY_SIZE).map(({ id, title, type, createdAt }) => ({
        id,
        title,
        type,
        createdAt,
      })),
      notificationCount: unreadNotifications.length,
      notifications: unreadNotifications.slice(0, SUMMARY_SIZE),
    });
  }),

  // 알림함 목록. unreadOnly=true면 안 읽은 것만
  http.get(apiUrl("/me/notifications"), ({ request }) => {
    const { user, error } = authorize(request);
    if (error) return error;
    const params = new URL(request.url).searchParams;
    const unreadOnly = params.get("unreadOnly") === "true";
    const rows = notificationsTable
      .all()
      .filter((item) => item.userId === user.id && (!unreadOnly || !item.readAt))
      .sort(byNewest);
    return HttpResponse.json(paginate(rows, params));
  }),

  http.post(apiUrl("/me/notifications/:id/read"), ({ request, params }) => {
    const { user, error } = authorize(request);
    if (error) return error;
    const item = notificationsTable.find(Number(params.id));
    if (item?.userId === user.id && !item.readAt) {
      notificationsTable.update(item.id, { readAt: new Date().toISOString() });
    }
    return new HttpResponse(null, { status: 204 });
  }),

  http.post(apiUrl("/me/notifications/read-all"), ({ request }) => {
    const { user, error } = authorize(request);
    if (error) return error;
    const now = new Date().toISOString();
    notificationsTable
      .all()
      .filter((item) => item.userId === user.id && !item.readAt)
      .forEach((item) => notificationsTable.update(item.id, { readAt: now }));
    return new HttpResponse(null, { status: 204 });
  }),

  // ── 알림 규칙 관리 ──
  http.get(apiUrl("/notification-rules"), ({ request }) => {
    const { error } = authorize(request, RULE_MENU, "READ");
    if (error) return error;
    return HttpResponse.json(notificationRulesTable.all());
  }),

  http.post(apiUrl("/notification-rules"), async ({ request }) => {
    const { error } = authorize(request, RULE_MENU, "CREATE");
    if (error) return error;
    const input = (await request.json()) as Omit<NotificationRule, "id">;
    return HttpResponse.json(notificationRulesTable.insert(input), { status: 201 });
  }),

  http.put(apiUrl("/notification-rules/:id"), async ({ request, params }) => {
    const { error } = authorize(request, RULE_MENU, "UPDATE");
    if (error) return error;
    const input = (await request.json()) as Omit<NotificationRule, "id">;
    const rule = notificationRulesTable.update(Number(params.id), input);
    return rule ? HttpResponse.json(rule) : problem(404, "규칙을 찾을 수 없습니다.");
  }),

  http.delete(apiUrl("/notification-rules/:id"), ({ request, params }) => {
    const { error } = authorize(request, RULE_MENU, "DELETE");
    if (error) return error;
    notificationRulesTable.remove(Number(params.id));
    return new HttpResponse(null, { status: 204 });
  }),

  // 알림 대상 선택용 사용자 목록 (이름만). 사용자 관리 권한이 없어도 고를 수 있게 따로 둔다.
  http.get(apiUrl("/user-options"), ({ request }) => {
    const { error } = authorize(request);
    if (error) return error;
    return HttpResponse.json(
      usersTable
        .all()
        .filter((user) => user.useYn === "Y")
        .map(({ id, name, username }) => ({ id, name, username })),
    );
  }),
];
```

✏️ **`src/mocks/handlers.ts`** — 등록합니다.

```ts
import { noticeHandlers } from "@/mocks/handlers/notices";
```

바꿀 코드

```ts
import { noticeHandlers } from "@/mocks/handlers/notices";
import { notificationHandlers } from "@/mocks/handlers/notifications";
```

✏️ **`src/mocks/handlers.ts`** — 배열에 추가합니다.

```ts
  ...noticeHandlers,
];
```

바꿀 코드

```ts
  ...noticeHandlers,
  ...notificationHandlers,
];
```

마지막으로 **서버 쪽 이벤트 발생**입니다. 사용자 등록 API가 저장을 마친 뒤 `USER_CREATED`를 발생시킵니다.

✏️ **`src/mocks/handlers/users.ts`** — 등록 후 이벤트를 발생시킵니다.

```ts
    credentialsTable.insert({ username: user.username, password: `${user.username}1234` });
    return HttpResponse.json(user, { status: 201 });
```

바꿀 코드

```ts
    credentialsTable.insert({ username: user.username, password: `${user.username}1234` });
    // 업무 처리가 끝나면 이벤트를 발생시킨다. 누구에게 알릴지는 규칙이 정한다. (서버가 일으키는 이벤트)
    emitEvent("USER_CREATED", { name: user.name, username: user.username });
    return HttpResponse.json(user, { status: 201 });
```

✏️ **`src/mocks/handlers/users.ts`** — import를 추가합니다.

```ts
import { credentialsTable, usersTable } from "@/mocks/db";
```

바꿀 코드

```ts
import { credentialsTable, usersTable } from "@/mocks/db";
import { emitEvent } from "@/mocks/notificationEngine";
```

### 확인하기

- [ ] `admin`으로 사용자를 하나 등록한 뒤, F12 → Console에서:

  ```js
  const t = JSON.parse(sessionStorage.auth).state.accessToken;
  await (await fetch("/api/me/notifications/summary", { headers: { Authorization: `Bearer ${t}` } })).json();
  ```

  `notificationCount: 1`과 "새 사용자 등록" 알림이 보입니다. 공지 5건도 안 읽음으로 셉니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 알림 발송 엔진과 API 추가"
```

---

## Step 12-2. 헤더의 미확인 알림 🔔

> 🎯 **할 일**: 헤더에 🔔와 미확인 개수를 표시하고, 누르면 최근 알림·미확인 공지 목록이 나오게 합니다. 15초마다 저절로 갱신됩니다.

### 원리: 폴링

```ts
useQuery({ queryKey, queryFn, refetchInterval: 15_000 });
```

`refetchInterval` 한 줄이면 TanStack Query가 15초마다 다시 조회합니다. 브라우저 탭이 **보이지 않을 때는 멈춥니다**(기본값). 쓸데없이 서버를 부르지 않습니다.

### 따라 하기

📄 **`src/shared/notification/queries.ts`** (새 파일)

```ts
import { keepPreviousData, useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import { httpClient } from "@/shared/api/httpClient";
import type { PageRequest, PageResponse } from "@/shared/api/types";
import type { AppNotification, NotificationSummary } from "@/shared/notification/types";

/** 알림 도착을 확인하는 주기. 실시간(SSE·WebSocket)으로 바꾸려면 Step 12-7 참고 */
export const NOTIFICATION_POLL_MS = 15_000;

export const notificationKeys = {
  all: ["me", "notifications"] as const,
  summary: () => [...notificationKeys.all, "summary"] as const,
  list: (condition: object) => [...notificationKeys.all, "list", condition] as const,
};

/** 🔔 요약. 일정 주기로 저절로 다시 조회한다. */
export function useNotificationSummary() {
  return useQuery({
    queryKey: notificationKeys.summary(),
    queryFn: () => httpClient.get<NotificationSummary>("/me/notifications/summary"),
    refetchInterval: NOTIFICATION_POLL_MS,
  });
}

export function useMyNotifications(condition: { unreadOnly: boolean } & PageRequest) {
  return useQuery({
    queryKey: notificationKeys.list(condition),
    queryFn: () =>
      httpClient.get<PageResponse<AppNotification>>("/me/notifications", { params: condition }),
    placeholderData: keepPreviousData,
  });
}

export function useMarkNotificationRead() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (id: number) => httpClient.post(`/me/notifications/${id}/read`),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: notificationKeys.all }),
  });
}

export function useMarkAllNotificationsRead() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: () => httpClient.post("/me/notifications/read-all"),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: notificationKeys.all }),
  });
}

/** 화면에서 이벤트를 발생시킨다. 받는 사람은 서버의 규칙이 정한다. 나에게 온 알림이 있을 수 있으니 요약을 다시 받는다. */
export function useEmitEvent() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: ({
      eventCode,
      variables,
    }: {
      eventCode: string;
      variables: Record<string, string>;
    }) =>
      httpClient.post<{ count: number }>(`/notification-events/${eventCode}/emit`, { variables }),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: notificationKeys.all }),
  });
}
```

공지를 읽거나 새로 쓰면 🔔의 미확인 공지 수도 바뀌어야 합니다. 공지 쪽 훅에서 알림 요약도 무효화합니다.

✏️ **`src/features/notice/api/queries.ts`** — 읽음 기록 후 🔔도 갱신합니다.

```ts
    mutationFn: (id: number) => httpClient.post(`/notices/${id}/read`),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: [...noticeKeys.all, "list"] }),
```

바꿀 코드

```ts
    mutationFn: (id: number) => httpClient.post(`/notices/${id}/read`),
    onSuccess: () =>
      Promise.all([
        queryClient.invalidateQueries({ queryKey: [...noticeKeys.all, "list"] }),
        queryClient.invalidateQueries({ queryKey: notificationKeys.all }), // 🔔 미확인 공지 수
      ]),
```

✏️ **`src/features/notice/api/queries.ts`** — import를 추가합니다.

```ts
import type { PageRequest, PageResponse } from "@/shared/api/types";
```

바꿀 코드

```ts
import type { PageRequest, PageResponse } from "@/shared/api/types";
import { notificationKeys } from "@/shared/notification/queries";
```

문구 파일입니다. 이 Part 전체 문구를 한 번에 넣습니다.

📄 **`src/shared/i18n/locales/ko/notification.json`** (새 파일)

```json
{
  "bell": {
    "title": "알림",
    "notifications": "알림 {{count}}",
    "notices": "공지 {{count}}",
    "empty": "새 알림이 없습니다.",
    "emptyNotice": "읽지 않은 공지가 없습니다.",
    "readAll": "모두 읽음",
    "viewAll": "알림함"
  },
  "center": {
    "all": "전체",
    "unread": "안 읽음",
    "createdAt": "받은 시각",
    "title": "제목",
    "message": "내용"
  },
  "rule": {
    "name": "규칙 이름",
    "event": "이벤트",
    "eventMenu": "발생 화면: {{menu}}",
    "targetType": "대상 유형",
    "targets": "대상",
    "ROLE": "역할",
    "USER": "사용자",
    "titleTemplate": "제목 템플릿",
    "messageTemplate": "내용 템플릿",
    "variables": "쓸 수 있는 변수: {{variables}}",
    "link": "누르면 열 화면",
    "createTitle": "알림 규칙 등록",
    "editTitle": "알림 규칙 수정"
  },
  "test": {
    "approvalTitle": "결재 요청 보내기",
    "approvalDesc": "SAMPLE_APPROVAL_REQUESTED 이벤트를 발생시킵니다. 기본 규칙은 매니저·관리자에게 알립니다.",
    "item": "품목",
    "amount": "금액",
    "send": "결재 요청",
    "jobTitle": "작업 완료 알리기",
    "jobDesc": "SAMPLE_JOB_DONE 이벤트를 발생시킵니다. 기본 규칙은 이서연(user)에게 알립니다.",
    "jobName": "월말 정산",
    "jobSend": "작업 완료",
    "sent": "{{count}}명에게 알림을 보냈습니다."
  }
}
```

📄 **`src/shared/i18n/locales/en/notification.json`** (새 파일)

```json
{
  "bell": {
    "title": "Notifications",
    "notifications": "Alerts {{count}}",
    "notices": "Notices {{count}}",
    "empty": "No new notifications.",
    "emptyNotice": "No unread notices.",
    "readAll": "Mark all read",
    "viewAll": "Open inbox"
  },
  "center": {
    "all": "All",
    "unread": "Unread",
    "createdAt": "Received",
    "title": "Title",
    "message": "Message"
  },
  "rule": {
    "name": "Rule name",
    "event": "Event",
    "eventMenu": "Raised on: {{menu}}",
    "targetType": "Target type",
    "targets": "Targets",
    "ROLE": "Role",
    "USER": "User",
    "titleTemplate": "Title template",
    "messageTemplate": "Message template",
    "variables": "Available variables: {{variables}}",
    "link": "Page to open",
    "createTitle": "New notification rule",
    "editTitle": "Edit notification rule"
  },
  "test": {
    "approvalTitle": "Send an approval request",
    "approvalDesc": "Raises SAMPLE_APPROVAL_REQUESTED. The default rule notifies managers and administrators.",
    "item": "Item",
    "amount": "Amount",
    "send": "Request approval",
    "jobTitle": "Notify job completion",
    "jobDesc": "Raises SAMPLE_JOB_DONE. The default rule notifies the user 'user'.",
    "jobName": "Month-end closing",
    "jobSend": "Job done",
    "sent": "Notified {{count}} people."
  }
}
```

🔔 컴포넌트입니다. 헤더에 붙으므로 `app/layout`에 둡니다.

📄 **`src/app/layout/NotificationBell.tsx`** (새 파일)

```tsx
import { useState, type ReactNode } from "react";
import { useTranslation } from "react-i18next";
import { Badge, Button, Empty, Flex, Popover, Tabs, Typography, theme } from "antd";
import { BellOutlined } from "@ant-design/icons";
import { formatDateTime } from "@/shared/lib/format";
import {
  useMarkAllNotificationsRead,
  useMarkNotificationRead,
  useNotificationSummary,
} from "@/shared/notification/queries";
import type { AppNotification } from "@/shared/notification/types";
import { useOpenPage } from "@/shared/page/useOpenPage";

interface ItemProps {
  title: string;
  description?: string;
  time: string;
  onClick: () => void;
}

// 목록의 한 줄. 마우스를 올리면 배경이 바뀐다.
function Item({ title, description, time, onClick }: ItemProps) {
  const { token } = theme.useToken();
  const [hover, setHover] = useState(false);
  return (
    <div
      role="button"
      onClick={onClick}
      onMouseEnter={() => setHover(true)}
      onMouseLeave={() => setHover(false)}
      style={{
        padding: "8px 12px",
        cursor: "pointer",
        borderRadius: token.borderRadius,
        background: hover ? token.colorFillTertiary : undefined,
      }}
    >
      <Typography.Text strong ellipsis style={{ display: "block" }}>
        {title}
      </Typography.Text>
      {description && (
        <Typography.Text type="secondary" ellipsis style={{ display: "block" }}>
          {description}
        </Typography.Text>
      )}
      <Typography.Text type="secondary" style={{ fontSize: 12 }}>
        {time}
      </Typography.Text>
    </div>
  );
}

function ItemList({ children, empty }: { children: ReactNode[]; empty: string }) {
  if (children.length === 0)
    return <Empty image={Empty.PRESENTED_IMAGE_SIMPLE} description={empty} />;
  return <div style={{ maxHeight: 360, overflow: "auto" }}>{children}</div>;
}

/** 헤더의 🔔. 안 읽은 알림 + 안 읽은 공지 개수를 보여 주고, 누르면 최근 목록이 나온다. */
function NotificationBell() {
  const { t } = useTranslation("notification");
  const { data } = useNotificationSummary();
  const markRead = useMarkNotificationRead();
  const markAllRead = useMarkAllNotificationsRead();
  const openPage = useOpenPage();
  const [open, setOpen] = useState(false);

  const notificationCount = data?.notificationCount ?? 0;
  const noticeCount = data?.noticeCount ?? 0;

  const openNotification = (item: AppNotification) => {
    markRead.mutate(item.id);
    if (item.linkMenuCode) openPage(item.linkMenuCode, item.linkParams);
    setOpen(false);
  };

  const content = (
    <div style={{ width: 340 }}>
      <Tabs
        size="small"
        items={[
          {
            key: "notifications",
            label: t("bell.notifications", { count: notificationCount }),
            children: (
              <ItemList empty={t("bell.empty")}>
                {(data?.notifications ?? []).map((item) => (
                  <Item
                    key={item.id}
                    title={item.title}
                    description={item.message}
                    time={formatDateTime(item.createdAt)}
                    onClick={() => openNotification(item)}
                  />
                ))}
              </ItemList>
            ),
          },
          {
            key: "notices",
            label: t("bell.notices", { count: noticeCount }),
            children: (
              <ItemList empty={t("bell.emptyNotice")}>
                {(data?.notices ?? []).map((notice) => (
                  <Item
                    key={notice.id}
                    title={notice.title}
                    time={formatDateTime(notice.createdAt)}
                    onClick={() => {
                      openPage("NOTICE_VIEW", { id: String(notice.id) }); // 상세 화면이 읽음을 기록한다
                      setOpen(false);
                    }}
                  />
                ))}
              </ItemList>
            ),
          },
        ]}
      />
      <Flex justify="space-between" style={{ marginTop: 8 }}>
        <Button
          size="small"
          type="link"
          disabled={notificationCount === 0}
          onClick={() => markAllRead.mutate()}
        >
          {t("bell.readAll")}
        </Button>
        <Button
          size="small"
          type="link"
          onClick={() => {
            openPage("MY_NOTIFICATIONS");
            setOpen(false);
          }}
        >
          {t("bell.viewAll")}
        </Button>
      </Flex>
    </div>
  );

  return (
    <Popover
      trigger="click"
      placement="bottomRight"
      open={open}
      onOpenChange={setOpen}
      content={content}
    >
      <Badge count={notificationCount + noticeCount} size="small" offset={[-6, 6]}>
        <Button type="text" title={t("bell.title")} icon={<BellOutlined />} />
      </Badge>
    </Popover>
  );
}

export default NotificationBell;
```

✏️ **`src/app/layout/AppHeader.tsx`** — 사용자 메뉴 앞에 넣습니다.

```tsx
      <UserMenu />
```

바꿀 코드

```tsx
      <NotificationBell />
      <UserMenu />
```

✏️ **`src/app/layout/AppHeader.tsx`** — import를 추가합니다.

```tsx
import SettingsDrawer from "@/app/layout/SettingsDrawer";
```

바꿀 코드

```tsx
import NotificationBell from "@/app/layout/NotificationBell";
import SettingsDrawer from "@/app/layout/SettingsDrawer";
```

### 확인하기

- [ ] 헤더에 🔔와 빨간 숫자가 보입니다. (안 읽은 공지 5건 + Step 12-1에서 생긴 알림)
- [ ] 🔔 → [알림] 탭에 "새 사용자 등록", [공지] 탭에 안 읽은 공지 목록
- [ ] 공지를 누르면 상세 탭이 열리고, 🔔 숫자가 1 줄어듭니다.
- [ ] 알림을 누르면 사용자 관리 탭이 열리고 숫자가 줄어듭니다.
- [ ] F12 → Network에 `summary` 요청이 **15초마다** 찍힙니다. 다른 브라우저 탭으로 가 있으면 멈춥니다.
- [ ] `user`로 로그인하면 사용자 등록 알림 규칙의 대상(ADMIN)이 아니라서 [알림]은 비어 있습니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 헤더 미확인 알림 표시"
```

---

## Step 12-3. 새 알림 도착 토스트

> 🎯 **할 일**: 폴링으로 새 알림을 받으면 화면 오른쪽 위에 토스트를 띄웁니다. 누르면 그 화면이 열립니다.

### 원리

"새 알림"을 알려면 **이전에 본 가장 큰 id**를 기억해 두고, 그보다 큰 id가 오면 새것입니다.
처음 화면을 열었을 때는 기존 알림이 전부 "새것"으로 보이므로, **첫 조회에서는 기억만 하고 토스트를 띄우지 않습니다.**
기억하는 값은 화면에 보여 줄 것이 아니므로 state가 아니라 `useRef`에 둡니다.

### 따라 하기

✏️ **`src/app/layout/NotificationBell.tsx`** — 토스트 Effect를 추가합니다.

```tsx
  const notificationCount = data?.notificationCount ?? 0;
  const noticeCount = data?.noticeCount ?? 0;
```

바꿀 코드

```tsx
  const notificationCount = data?.notificationCount ?? 0;
  const noticeCount = data?.noticeCount ?? 0;

  // 새로 도착한 알림을 토스트로 띄운다. 처음 받은 목록은 "이미 있던 것"으로 보고 띄우지 않는다.
  const lastSeenId = useRef<number | null>(null);
  useEffect(() => {
    if (!data) return;
    const items = data.notifications;
    if (lastSeenId.current !== null) {
      const seen = lastSeenId.current;
      items
        .filter((item) => item.id > seen)
        .forEach((item) =>
          notification.info({
            title: item.title,
            description: item.message,
            placement: "topRight",
            onClick: () => {
              markRead.mutate(item.id);
              if (item.linkMenuCode) openPage(item.linkMenuCode, item.linkParams);
            },
          }),
        );
    }
    lastSeenId.current = Math.max(lastSeenId.current ?? 0, ...items.map((item) => item.id));
  }, [data, notification, markRead, openPage]);
```

✏️ **`src/app/layout/NotificationBell.tsx`** — 알림 함수를 꺼냅니다.

```tsx
  const [open, setOpen] = useState(false);

  const notificationCount
```

바꿀 코드

```tsx
  const [open, setOpen] = useState(false);
  const { notification } = App.useApp();

  const notificationCount
```

✏️ **`src/app/layout/NotificationBell.tsx`** — import를 고칩니다.

```tsx
import { useState, type ReactNode } from "react";
import { useTranslation } from "react-i18next";
import { Badge, Button, Empty, Flex, Popover, Tabs, Typography, theme } from "antd";
```

바꿀 코드

```tsx
import { useEffect, useRef, useState, type ReactNode } from "react";
import { useTranslation } from "react-i18next";
import { App, Badge, Button, Empty, Flex, Popover, Tabs, Typography, theme } from "antd";
```

> ⚠️ **주의** 위 Effect의 의존성에 `markRead`(mutation 객체)가 들어 있습니다. mutation 객체는 렌더마다 바뀌어서 Effect가 자주 다시 실행되지만,
> `lastSeenId`로 이미 본 알림을 걸러내므로 토스트가 중복으로 뜨지 않습니다. 이렇게 **Effect가 여러 번 실행돼도 결과가 같게** 만들어 두면 안전합니다.

### 확인하기

이 기능은 **두 사람**으로 확인하는 게 제일 재미있습니다. sessionStorage는 브라우저 탭마다 따로라서 탭마다 다른 계정으로 로그인할 수 있습니다.

- [ ] 브라우저 탭 A: `admin`으로 로그인
- [ ] 새 브라우저 탭 B를 열고 주소창에 `http://localhost:5173` 입력 → `admin`으로 로그인 → 사용자 등록
- [ ] 탭 A로 돌아가면 15초 안에 오른쪽 위에 "새 사용자 등록" 토스트가 뜹니다. 누르면 사용자 관리 탭이 열립니다.

> 💡 **팁** 탭 복제(`Ctrl+Shift+K` 등)는 sessionStorage까지 복사해서 같은 계정이 됩니다. **새 탭에서 주소를 직접 입력**하세요.
> 가짜 DB는 localStorage라서 두 탭이 같은 데이터를 봅니다. Part 3에서 "매번 localStorage에서 읽게" 만든 덕분입니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 새 알림 도착 시 토스트 표시"
```

---

## Step 12-4. 알림함

> 🎯 **할 일**: 받은 알림 전체를 보고 읽음 처리하는 화면을 만듭니다. 메뉴의 `notification/NotificationCenterPage`에 연결됩니다.

### 따라 하기

📄 **`src/features/notification/pages/NotificationCenterPage.tsx`** (새 파일)

```tsx
import { useMemo, useState } from "react";
import { useTranslation } from "react-i18next";
import { Button, Flex, Pagination, Segmented } from "antd";
import type { ColDef } from "ag-grid-community";
import type { PageRequest } from "@/shared/api/types";
import { formatDateTime } from "@/shared/lib/format";
import {
  useMarkAllNotificationsRead,
  useMarkNotificationRead,
  useMyNotifications,
} from "@/shared/notification/queries";
import type { AppNotification } from "@/shared/notification/types";
import { useOpenPage } from "@/shared/page/useOpenPage";
import DataGrid from "@/shared/ui/DataGrid";
import PageContainer from "@/shared/ui/PageContainer";

function NotificationCenterPage() {
  const { t } = useTranslation("notification");
  const openPage = useOpenPage();
  const [unreadOnly, setUnreadOnly] = useState(false);
  const [paging, setPaging] = useState<PageRequest>({ page: 0, size: 20, sort: "" });
  const { data, isFetching } = useMyNotifications({ unreadOnly, ...paging });
  const markRead = useMarkNotificationRead();
  const markAllRead = useMarkAllNotificationsRead();

  const columnDefs = useMemo<ColDef<AppNotification>[]>(
    () => [
      {
        headerName: t("center.createdAt"),
        field: "createdAt",
        width: 170,
        flex: 0,
        valueFormatter: ({ value }) => formatDateTime(value),
      },
      { headerName: t("center.title"), field: "title" },
      { headerName: t("center.message"), field: "message", flex: 3, tooltip: true },
    ],
    [t],
  );

  return (
    <PageContainer
      extra={
        <Button onClick={() => markAllRead.mutate()} loading={markAllRead.isPending}>
          {t("bell.readAll")}
        </Button>
      }
    >
      <Segmented<boolean>
        value={unreadOnly}
        onChange={(value) => {
          setUnreadOnly(value);
          setPaging((prev) => ({ ...prev, page: 0 }));
        }}
        options={[
          { value: false, label: t("center.all") },
          { value: true, label: t("center.unread") },
        ]}
        style={{ alignSelf: "flex-start" }}
      />
      <DataGrid<AppNotification>
        rowData={data?.content}
        columnDefs={columnDefs}
        loading={isFetching}
        getRowId={({ data: item }) => String(item.id)}
        getRowStyle={({ data: item }) => (item?.readAt ? undefined : { fontWeight: 600 })}
        onRowDoubleClicked={({ data: item }) => {
          if (!item) return;
          markRead.mutate(item.id);
          if (item.linkMenuCode) openPage(item.linkMenuCode, item.linkParams);
        }}
      />
      <Flex justify="end">
        <Pagination
          size="small"
          current={paging.page + 1}
          pageSize={paging.size}
          total={data?.page.totalElements ?? 0}
          showTotal={(total) => t("common:label.total", { count: total })}
          onChange={(page, size) => setPaging((prev) => ({ ...prev, page: page - 1, size }))}
        />
      </Flex>
    </PageContainer>
  );
}

export default NotificationCenterPage;
```

### 확인하기

- [ ] 🔔 → [알림함] 또는 내 정보 → 알림함. 받은 알림이 최신순으로, 안 읽은 것은 굵게 보입니다.
- [ ] [안 읽음]만 보기, 더블클릭으로 읽음 + 화면 열기, [모두 읽음]이 동작하고 🔔 숫자도 함께 바뀝니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 알림함 화면 추가"
```

---

## Step 12-5. 알림 규칙 관리

> 🎯 **할 일**: "어떤 이벤트가 일어나면, 누구에게, 어떤 문구로" 알릴지 관리자가 화면에서 정하게 합니다.

### 따라 하기

📄 **`src/features/notification/api/ruleQueries.ts`** (새 파일)

```ts
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import { httpClient } from "@/shared/api/httpClient";
import type { NotificationEvent, NotificationRule } from "@/shared/notification/types";

export type RuleInput = Omit<NotificationRule, "id">;

export interface UserOption {
  id: number;
  name: string;
  username: string;
}

const rulesKey = ["notification-rules"] as const;

export function useNotificationRules() {
  return useQuery({
    queryKey: rulesKey,
    queryFn: () => httpClient.get<NotificationRule[]>("/notification-rules"),
  });
}

export function useNotificationEvents() {
  return useQuery({
    queryKey: ["notification-events"],
    queryFn: () => httpClient.get<NotificationEvent[]>("/notification-events"),
    staleTime: Infinity, // 이벤트 목록은 코드로 정해져 있어서 바뀌지 않는다
  });
}

export function useUserOptions() {
  return useQuery({
    queryKey: ["user-options"],
    queryFn: () => httpClient.get<UserOption[]>("/user-options"),
  });
}

export function useSaveRule() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: ({ id, input }: { id?: number; input: RuleInput }) =>
      id
        ? httpClient.put<NotificationRule>(`/notification-rules/${id}`, input)
        : httpClient.post<NotificationRule>("/notification-rules", input),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: rulesKey }),
  });
}

export function useDeleteRule() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (id: number) => httpClient.delete(`/notification-rules/${id}`),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: rulesKey }),
  });
}
```

규칙 입력 모달입니다. 이벤트를 고르면 그 이벤트에서 쓸 수 있는 변수(`{{name}}` 등)를 안내합니다.
대상 유형을 바꾸면 대상 선택지가 역할 목록 ↔ 사용자 목록으로 바뀝니다.

📄 **`src/features/notification/components/RuleFormModal.tsx`** (새 파일)

```tsx
import { useTranslation } from "react-i18next";
import { App, Form, Input, Modal, Radio, Select } from "antd";
import {
  useNotificationEvents,
  useSaveRule,
  useUserOptions,
  type RuleInput,
} from "@/features/notification/api/ruleQueries";
import { useRoles } from "@/shared/auth/useRoles";
import CodeRadio from "@/shared/codes/CodeRadio";
import { useLocalizedText } from "@/shared/i18n/useLocalizedText";
import { useMyMenus } from "@/shared/menu/useMyMenus";
import type { NotificationRule } from "@/shared/notification/types";

interface RuleFormModalProps {
  rule?: NotificationRule;
  onClose: () => void;
}

function RuleFormModal({ rule, onClose }: RuleFormModalProps) {
  const { t } = useTranslation("notification");
  const { message } = App.useApp();
  const localize = useLocalizedText();
  const [form] = Form.useForm<RuleInput>();
  const saveRule = useSaveRule();
  const { data: events = [] } = useNotificationEvents();
  const { data: roles = [] } = useRoles();
  const { data: users = [] } = useUserOptions();
  const { data: menus = [] } = useMyMenus();

  const eventCode = Form.useWatch("eventCode", form);
  const targetType = Form.useWatch("targetType", form);
  const event = events.find((item) => item.code === eventCode);
  const menuName = (code?: string) => {
    const menu = menus.find((item) => item.code === code);
    return menu ? localize(menu.names) : (code ?? "");
  };

  const required = (field: string) => ({
    required: true,
    message: t("common:validation.required", { field }),
  });

  const handleFinish = (input: RuleInput) => {
    saveRule.mutate(
      { id: rule?.id, input },
      {
        onSuccess: () => {
          message.success(t("common:message.saved"));
          onClose();
        },
      },
    );
  };

  return (
    <Modal
      open
      width={640}
      title={rule ? t("rule.editTitle") : t("rule.createTitle")}
      okText={t("common:button.save")}
      cancelText={t("common:button.cancel")}
      onOk={form.submit}
      onCancel={onClose}
      confirmLoading={saveRule.isPending}
      mask={{ closable: false }}
    >
      <Form
        form={form}
        name="ruleForm"
        layout="vertical"
        initialValues={rule ?? { targetType: "ROLE", targets: [], useYn: "Y" }}
        onFinish={handleFinish}
      >
        <Form.Item name="name" label={t("rule.name")} rules={[required(t("rule.name"))]}>
          <Input />
        </Form.Item>
        <Form.Item
          name="eventCode"
          label={t("rule.event")}
          rules={[required(t("rule.event"))]}
          extra={event && t("rule.eventMenu", { menu: menuName(event.menuCode) })}
        >
          <Select
            options={events.map((item) => ({
              value: item.code,
              label: `${localize(item.names)} (${item.code})`,
            }))}
          />
        </Form.Item>
        <Form.Item name="targetType" label={t("rule.targetType")}>
          <Radio.Group
            // 유형을 바꾸면 이전 대상 목록은 의미가 없으니 비운다
            onChange={() => form.setFieldValue("targets", [])}
            options={[
              { value: "ROLE", label: t("rule.ROLE") },
              { value: "USER", label: t("rule.USER") },
            ]}
          />
        </Form.Item>
        <Form.Item name="targets" label={t("rule.targets")} rules={[required(t("rule.targets"))]}>
          <Select
            mode="multiple"
            options={
              targetType === "USER"
                ? users.map((user) => ({
                    value: String(user.id),
                    label: `${user.name} (${user.username})`,
                  }))
                : roles.map((role) => ({ value: role.code, label: localize(role.names) }))
            }
          />
        </Form.Item>
        <Form.Item
          name="titleTemplate"
          label={t("rule.titleTemplate")}
          rules={[required(t("rule.titleTemplate"))]}
          extra={
            event &&
            t("rule.variables", {
              variables: event.variables.map((name) => `{{${name}}}`).join(", "),
            })
          }
        >
          <Input />
        </Form.Item>
        <Form.Item name="messageTemplate" label={t("rule.messageTemplate")}>
          <Input.TextArea rows={2} />
        </Form.Item>
        <Form.Item name="linkMenuCode" label={t("rule.link")}>
          <Select
            allowClear
            options={menus
              .filter((menu) => menu.type === "PAGE")
              .map((menu) => ({ value: menu.code, label: localize(menu.names) }))}
          />
        </Form.Item>
        <Form.Item name="useYn" label={t("common:label.use")}>
          <CodeRadio group="USE_YN" />
        </Form.Item>
      </Form>
    </Modal>
  );
}

export default RuleFormModal;
```

> 💡 **팁** `t("rule.variables", { variables: "{{name}}" })`처럼 **번역 값 안에** `{{...}}`가 들어가도 괜찮습니다.
> i18next는 번역 문장(템플릿)의 `{{변수}}`만 바꾸고, 넣어 준 값 안의 중괄호는 그대로 둡니다.

📄 **`src/features/notification/pages/NotificationRulePage.tsx`** (새 파일)

```tsx
import { useMemo, useState } from "react";
import { useTranslation } from "react-i18next";
import { App, Button, Space } from "antd";
import type { ColDef } from "ag-grid-community";
import {
  useDeleteRule,
  useNotificationEvents,
  useNotificationRules,
} from "@/features/notification/api/ruleQueries";
import RuleFormModal from "@/features/notification/components/RuleFormModal";
import Can from "@/shared/auth/Can";
import { usePermission } from "@/shared/auth/usePermission";
import { useCodeName } from "@/shared/codes/useCodes";
import { useLocalizedText } from "@/shared/i18n/useLocalizedText";
import type { NotificationRule } from "@/shared/notification/types";
import DataGrid from "@/shared/ui/DataGrid";
import PageContainer from "@/shared/ui/PageContainer";

function NotificationRulePage() {
  const { t } = useTranslation("notification");
  const { message, modal } = App.useApp();
  const { can } = usePermission();
  const localize = useLocalizedText();
  const formatUseYn = useCodeName("USE_YN");
  const rules = useNotificationRules();
  const { data: events = [] } = useNotificationEvents();
  const deleteRule = useDeleteRule();
  const [selected, setSelected] = useState<NotificationRule>();
  const [editing, setEditing] = useState<{ rule?: NotificationRule } | null>(null);

  const columnDefs = useMemo<ColDef<NotificationRule>[]>(() => {
    const eventName = (code: string) => {
      const event = events.find((item) => item.code === code);
      return event ? localize(event.names) : code;
    };
    return [
      { headerName: t("rule.name"), field: "name", flex: 2 },
      {
        headerName: t("rule.event"),
        field: "eventCode",
        valueFormatter: ({ value }) => eventName(value),
      },
      {
        headerName: t("rule.targets"),
        valueGetter: ({ data: rule }) =>
          rule && `${t(`rule.${rule.targetType}`)}: ${rule.targets.join(", ")}`,
      },
      { headerName: t("rule.titleTemplate"), field: "titleTemplate", flex: 2 },
      {
        headerName: t("common:label.use"),
        field: "useYn",
        width: 90,
        flex: 0,
        valueFormatter: ({ value }) => formatUseYn(value),
      },
    ];
  }, [t, events, localize, formatUseYn]);

  const handleDelete = () => {
    if (!selected) return;
    modal.confirm({
      title: t("common:message.confirmDelete", { count: 1 }),
      okButtonProps: { danger: true },
      onOk: () =>
        deleteRule
          .mutateAsync(selected.id)
          .then(() => message.success(t("common:message.deleted")))
          .catch(() => undefined),
    });
  };

  return (
    <PageContainer
      extra={
        <Space>
          <Can action="CREATE">
            <Button type="primary" onClick={() => setEditing({})}>
              {t("common:button.create")}
            </Button>
          </Can>
          <Can action="DELETE">
            <Button danger disabled={!selected} onClick={handleDelete}>
              {t("common:button.delete")}
            </Button>
          </Can>
        </Space>
      }
    >
      <DataGrid<NotificationRule>
        rowData={rules.data}
        columnDefs={columnDefs}
        loading={rules.isFetching}
        getRowId={({ data: rule }) => String(rule.id)}
        rowSelection={{ mode: "singleRow", checkboxes: false, enableClickSelection: true }}
        onSelectionChanged={({ api }) => setSelected(api.getSelectedRows()[0])}
        onRowDoubleClicked={({ data: rule }) => rule && can("UPDATE") && setEditing({ rule })}
      />
      {editing && <RuleFormModal rule={editing.rule} onClose={() => setEditing(null)} />}
    </PageContainer>
  );
}

export default NotificationRulePage;
```

### 확인하기

- [ ] 시스템 관리 → 알림 규칙. 기본 규칙 3개가 보입니다.
- [ ] "신규 사용자 등록 알림"을 더블클릭 → 대상에 `MANAGER`를 추가하고 저장
- [ ] 사용자를 하나 등록 → `manager`로 로그인하면 🔔에 "새 사용자 등록" 알림이 있습니다.
- [ ] 대상 유형을 [사용자]로 바꾸면 선택지가 사람 이름 목록으로 바뀝니다.
- [ ] 이벤트를 고르면 아래에 `쓸 수 있는 변수: {{name}}, {{username}}` 안내가 나옵니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 알림 규칙 관리 화면 추가"
```

---

## Step 12-6. 화면에서 이벤트 발생시키기

> 🎯 **할 일**: "특정 화면에서 특정 이벤트가 일어나면 알림"을 화면 코드 한 줄로 하는 예제를 만듭니다. 메뉴의 `samples/NotifyTestPage`에 연결됩니다.

### 원리

화면은 **누구에게 보낼지 모릅니다.** 이벤트 코드와 변수만 서버에 알립니다. 받는 사람과 문구는 관리자가 정한 규칙이 결정합니다.

```ts
const emitEvent = useEmitEvent();
emitEvent.mutate({ eventCode: "SAMPLE_APPROVAL_REQUESTED", variables: { item: "노트북", amount: "1500000" } });
```

이렇게 나누면 "결재 요청 알림을 팀장에게도 보내 주세요" 같은 요청에 **코드를 고치지 않고** 규칙만 바꾸면 됩니다.

### 따라 하기

📄 **`src/features/samples/pages/NotifyTestPage.tsx`** (새 파일)

```tsx
import { useTranslation } from "react-i18next";
import { App, Button, Card, Flex, Form, Input, InputNumber, Typography } from "antd";
import { useMe } from "@/shared/auth/useMe";
import { useEmitEvent } from "@/shared/notification/queries";
import PageContainer from "@/shared/ui/PageContainer";

interface ApprovalForm {
  item: string;
  amount: number;
}

function NotifyTestPage() {
  const { t } = useTranslation("notification");
  const { message } = App.useApp();
  const { data: me } = useMe();
  const emitEvent = useEmitEvent();

  // 이벤트를 발생시키고, 몇 명에게 갔는지 알려 준다.
  const emit = (eventCode: string, variables: Record<string, string>) =>
    emitEvent.mutate(
      { eventCode, variables },
      { onSuccess: ({ count }) => message.success(t("test.sent", { count })) },
    );

  return (
    <PageContainer>
      <Flex gap={16} wrap align="start">
        <Card title={t("test.approvalTitle")} style={{ width: 420 }}>
          <Typography.Paragraph type="secondary">{t("test.approvalDesc")}</Typography.Paragraph>
          <Form<ApprovalForm>
            name="approvalForm"
            layout="vertical"
            initialValues={{ item: "노트북", amount: 1500000 }}
            onFinish={({ item, amount }) =>
              emit("SAMPLE_APPROVAL_REQUESTED", {
                requester: me?.name ?? "",
                item,
                amount: amount.toLocaleString(),
              })
            }
          >
            <Form.Item name="item" label={t("test.item")} rules={[{ required: true }]}>
              <Input />
            </Form.Item>
            <Form.Item name="amount" label={t("test.amount")} rules={[{ required: true }]}>
              <InputNumber min={0} step={10000} style={{ width: "100%" }} />
            </Form.Item>
            <Button type="primary" htmlType="submit" loading={emitEvent.isPending}>
              {t("test.send")}
            </Button>
          </Form>
        </Card>
        <Card title={t("test.jobTitle")} style={{ width: 420 }}>
          <Typography.Paragraph type="secondary">{t("test.jobDesc")}</Typography.Paragraph>
          <Button onClick={() => emit("SAMPLE_JOB_DONE", { job: t("test.jobName") })}>
            {t("test.jobSend")}
          </Button>
        </Card>
      </Flex>
    </PageContainer>
  );
}

export default NotifyTestPage;
```

### 확인하기

두 브라우저 탭으로 확인합니다. (Step 12-3 참고)

- [ ] 탭 A: `manager`로 로그인
- [ ] 탭 B: `user`로 로그인 → 샘플 → 알림 발생 테스트 → [결재 요청] → "3명에게 알림을 보냈습니다." (규칙 대상 MANAGER·ADMIN 역할: 박지훈, 정도윤, 김민수)
- [ ] 탭 A에 15초 안에 "결재 요청: 노트북" 토스트가 뜹니다. 🔔에도 숫자가 늘어 있습니다.
- [ ] 탭 B에서 [작업 완료] → 탭 B 자신(이서연)에게 "작업 완료" 토스트가 뜹니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 화면 이벤트로 알림을 보내는 샘플 추가"
```

---

## Step 12-7. 실시간 알림으로 바꾸려면 (읽기만)

> 🎯 **할 일**: 폴링을 SSE·WebSocket으로 바꾸는 방법을 알아 둡니다. 이 Step은 코드를 입력하지 않습니다.

폴링은 단순하지만 "최대 15초 늦게" 도착합니다. 즉시 도착해야 한다면 서버가 **밀어 주는(push)** 방식을 씁니다.

| 방식           | 특징                                                                    | 언제                          |
| -------------- | ----------------------------------------------------------------------- | ----------------------------- |
| 폴링 (현재)    | 단순. 어떤 서버·프록시와도 동작                                         | 몇 초~몇십 초 늦어도 될 때    |
| **SSE**        | 서버 → 브라우저 한 방향. HTTP 그대로라 설정이 쉽다. Spring `SseEmitter` | 알림처럼 받기만 하면 될 때    |
| WebSocket      | 양방향. 채팅 등                                                         | 브라우저도 계속 보내야 할 때  |

바꾸는 방법은 간단합니다. **"새 알림이 있다"는 신호를 받으면 캐시를 무효화**하기만 하면, 나머지(🔔 숫자, 토스트, 알림함)는 지금 코드 그대로 동작합니다.

```tsx
// 예: SSE로 바꿀 때 (헤더 근처에서 한 번만 실행)
useEffect(() => {
  const source = new EventSource(`${import.meta.env.VITE_API_BASE_URL}/me/notifications/stream`);
  source.onmessage = () => queryClient.invalidateQueries({ queryKey: notificationKeys.all });
  return () => source.close(); // 화면이 사라지면 연결을 닫는다
}, [queryClient]);
```

그리고 `useNotificationSummary`의 `refetchInterval`을 빼거나 길게(예: 5분, 연결이 끊겼을 때의 안전망) 바꿉니다.
알림 관련 코드가 `shared/notification/queries.ts` 한 곳에 모여 있어서, 바꿀 곳도 거기뿐입니다.

> ⚠️ **주의** `EventSource`는 `Authorization` 헤더를 붙일 수 없습니다. SSE를 쓸 때는 쿠키 인증(httpOnly)을 쓰거나, 짧은 수명의 토큰을 주소에 붙이는 방식을 서버와 정합니다.

---

## 🏁 Part 12를 마치며

마일스톤입니다. Top 영역에 미확인 알림이 표시되고, 화면 이벤트가 규칙에 따라 정해진 사람에게 전달됩니다.

| 요구사항                                         | 구현                                                  |
| ------------------------------------------------ | ----------------------------------------------------- |
| 읽지 않은 공지사항 알림                          | 🔔 [공지] 탭, 상세를 열면 읽음 처리                   |
| 특정 화면에서 특정 이벤트 → 정해진 대상에게 알림 | 이벤트 + 규칙(역할·사용자) + `useEmitEvent` / 서버 `emitEvent` |
| Top 영역에 미확인 알림 표시                      | `NotificationBell` + 폴링 + 토스트                    |

| 증상                                       | 원인과 해결                                                                      |
| ------------------------------------------ | -------------------------------------------------------------------------------- |
| 처음 들어올 때 기존 알림이 전부 토스트로 뜬다 | 첫 조회에서는 기억만 한다 (`lastSeenId`가 null일 때)                          |
| 다른 탭의 계정에 알림이 안 간다            | 규칙의 대상·사용 여부 확인. 받는 사람의 역할 확인                                |
| 숨은 탭에서도 계속 요청이 나간다           | 기본값이면 브라우저 탭이 안 보일 때 멈춘다. `refetchIntervalInBackground`를 켜지 않았는지 확인 |

다음은 실무에서 자주 쓰는 기능을 골라 먹는 Part입니다. → [Part 13. 실무 기능 모음](part-13-practical-features.md)
