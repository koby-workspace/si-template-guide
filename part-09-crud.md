# Part 9. 업무 화면 기본기 (CRUD)

> **이 Part의 목표**: 조회·등록·수정·삭제(CRUD)가 되는 진짜 같은 관리 화면을 만듭니다.
> 공통 조회 영역, 등록 모달, 서버 검증 오류 표시, 공통코드, 마스터-디테일, 서버 페이징까지 SI 화면에서 매일 쓰는 패턴을 모두 익힙니다.
> 이 Part가 끝난 사용자 관리 화면이 **새 화면을 만들 때 복사할 표준 예제**가 됩니다.

---

## Step 9-1. 공통 조회 조건 영역

> 🎯 **할 일**: 화면마다 반복되는 "조회 조건 + [조회] [초기화]" 영역을 컴포넌트로 만듭니다.

### 왜 지금 공통화하나요?

설계 원칙 5번: "같은 코드가 2~3번 나오면 그때 뽑아낸다." 이제 화면이 계속 늘어날 예정이고, 모든 목록 화면에 같은 패턴이 들어갑니다.
조회 버튼 위치·Enter 키·같은 조건 재조회 같은 규칙을 **한 곳에서** 정하면 모든 화면이 똑같이 동작합니다.

### 따라 하기

📄 **`src/shared/ui/SearchForm.tsx`** (새 파일)

```tsx
import { useId, useRef, type ReactNode } from "react";
import { useTranslation } from "react-i18next";
import { Button, Form, Space, theme } from "antd";
import { ReloadOutlined, SearchOutlined } from "@ant-design/icons";

interface SearchFormProps<T> {
  onSearch: (values: T) => void; // 조건이 바뀌었을 때
  onSameSearch?: () => void; // 같은 조건으로 다시 조회할 때 (보통 refetch)
  initialValues?: Partial<T>;
  children: ReactNode; // 조회 조건 입력칸들 (Form.Item)
}

/** 조회 조건 영역. 입력칸에서 Enter 또는 [조회]로 조회하고, [초기화]는 조건을 비우고 바로 조회한다. */
function SearchForm<T extends object>({
  onSearch,
  onSameSearch,
  initialValues,
  children,
}: SearchFormProps<T>) {
  const { t } = useTranslation();
  const { token } = theme.useToken();
  const [form] = Form.useForm<T>();
  const formName = useId(); // 입력칸 id가 다른 탭의 폼과 겹치지 않게 폼마다 고유한 이름을 준다
  const lastValues = useRef(JSON.stringify(initialValues ?? {}));

  const handleFinish = (values: T) => {
    const text = JSON.stringify(values);
    // 조건이 같으면 queryKey도 같아서 캐시만 보고 끝난다. SI 사용자는 [조회] = "서버에서 다시 가져오기"로 기대한다.
    if (text === lastValues.current) {
      onSameSearch?.();
      return;
    }
    lastValues.current = text;
    onSearch(values);
  };

  const handleReset = () => {
    form.resetFields();
    form.submit();
  };

  return (
    <Form
      form={form}
      name={formName}
      layout="inline"
      initialValues={initialValues}
      onFinish={handleFinish}
      style={{
        padding: 12,
        rowGap: 8,
        background: token.colorFillAlter,
        border: `1px solid ${token.colorBorderSecondary}`,
        borderRadius: token.borderRadiusLG,
      }}
    >
      {children}
      <Space style={{ marginInlineStart: "auto" }}>
        <Button type="primary" htmlType="submit" icon={<SearchOutlined />}>
          {t("button.search")}
        </Button>
        <Button icon={<ReloadOutlined />} onClick={handleReset}>
          {t("button.reset")}
        </Button>
      </Space>
    </Form>
  );
}

export default SearchForm;
```

📄 **`src/features/users/pages/UsersPage.tsx`** (전체 교체)

```tsx
import { useMemo, useState } from "react";
import { useTranslation } from "react-i18next";
import { Button, Form, Input, Select } from "antd";
import type { ColDef } from "ag-grid-community";
import { useUserList } from "@/features/users/api/queries";
import type { User, UserSearchCondition } from "@/features/users/types";
import Can from "@/shared/auth/Can";
import DataGrid from "@/shared/ui/DataGrid";
import PageContainer from "@/shared/ui/PageContainer";
import SearchForm from "@/shared/ui/SearchForm";

const ROLE_CODES = ["ADMIN", "MANAGER", "USER"];

function UsersPage() {
  const { t } = useTranslation("users");
  const [condition, setCondition] = useState<UserSearchCondition>({});
  const { data, isFetching, refetch } = useUserList(condition);

  // 언어가 바뀌면 t가 바뀐다. 그때만 컬럼 정의를 다시 만든다.
  const columnDefs = useMemo<ColDef<User>[]>(
    () => [
      { headerName: t("column.username"), field: "username" },
      { headerName: t("column.name"), field: "name" },
      { headerName: t("column.email"), field: "email", flex: 2 },
      { headerName: t("column.phone"), field: "phone" },
      {
        headerName: t("column.roles"),
        field: "roleCodes",
        // 값이 배열이면 그리드는 어떻게 보여 줄지 모른다. 글자로 바꾸는 방법을 알려 준다.
        valueFormatter: ({ value }) => value?.map((code: string) => t(`role.${code}`)).join(", "),
      },
      { headerName: t("column.useYn"), field: "useYn", width: 90, flex: 0 },
      { headerName: t("common:label.createdAt"), field: "createdAt" },
    ],
    [t],
  );

  return (
    <PageContainer
      extra={
        <Can action="CREATE">
          <Button type="primary">{t("common:button.create")}</Button>
        </Can>
      }
    >
      <SearchForm<UserSearchCondition> onSearch={setCondition} onSameSearch={refetch}>
        <Form.Item name="name" label={t("search.name")}>
          <Input placeholder={t("search.namePlaceholder")} allowClear />
        </Form.Item>
        <Form.Item name="roleCode" label={t("search.role")}>
          <Select
            placeholder={t("common:label.all")}
            allowClear
            style={{ width: 140 }}
            options={ROLE_CODES.map((code) => ({ value: code, label: t(`role.${code}`) }))}
          />
        </Form.Item>
      </SearchForm>
      <DataGrid<User> rowData={data} columnDefs={columnDefs} loading={isFetching} />
    </PageContainer>
  );
}

export default UsersPage;
```

### 코드 읽어 보기

**조립(합성)**: `SearchForm`은 어떤 입력칸이 들어올지 모릅니다. `children`으로 받아서 그대로 그릴 뿐입니다. 이렇게 하면 화면마다 다른 조건을 넣어도 버튼·배치·동작은 같습니다.

**제네릭 컴포넌트 `<SearchForm<UserSearchCondition>>`**: `onSearch`가 받는 값의 타입을 정해 줍니다. 덕분에 `setCondition`을 그대로 넘길 수 있습니다.

**`form`과 state가 사라졌다**: 화면은 이제 "조건이 바뀌면 어떻게 할지"만 정합니다. 폼 관리는 `SearchForm`의 일입니다.

**`name={useId()}`**: antd는 입력칸에 `name` 값을 그대로 HTML `id`로 붙입니다(`id="name"`). 그런데 MDI에서는 여러 탭의 폼이 **동시에** 살아 있어서
`id="name"`이 화면에 여러 개 생깁니다. 그러면 라벨을 눌렀을 때 엉뚱한 탭의 입력칸에 커서가 가고, 화면 낭독기도 헷갈립니다.
`Form`에 `name`을 주면 id가 `폼이름_name`이 되어 겹치지 않습니다. `useId()`는 React가 컴포넌트마다 고유한 값을 만들어 주는 훅입니다.

### 확인하기

- [ ] 조회 영역이 회색 상자로 바뀌고, 버튼이 오른쪽 끝으로 갑니다.
- [ ] Enter·[조회]·[초기화]·같은 조건 재조회가 이전과 똑같이 동작합니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 공통 조회 조건 영역 컴포넌트 추가"
```

---

## Step 9-2. 사용자 등록

> 🎯 **할 일**: [등록] 버튼으로 모달을 열어 사용자를 등록하고, 저장하면 목록이 자동으로 갱신되게 합니다.

### 원리: mutation과 캐시 무효화

조회(`useQuery`)와 달리 **서버 데이터를 바꾸는** 요청은 `useMutation`을 씁니다.

```
[저장] → useMutation → POST /api/users
   → 저장 중: 버튼 loading (중복 클릭 방지)
   → 성공: "저장되었습니다." → 모달 닫기
           → invalidateQueries(["users"]) → "사용자 캐시는 낡았다"
           → 사용자 목록을 보고 있는 화면이 알아서 다시 조회
```

저장한 화면이 목록 화면에게 "새로고침해"라고 부르지 않습니다. 둘 다 같은 캐시를 보고 있을 뿐입니다.
Part 7의 탭과도 궁합이 좋습니다. 다른 탭에 숨어 있는 목록도 살아 있으므로 **알아서 최신화**됩니다.

### 따라 하기

역할 목록이 필요합니다. 역할은 여러 화면이 쓰므로 `shared/auth`에 둡니다.

📄 **`src/mocks/handlers/roles.ts`** (새 파일)

```ts
import { http, HttpResponse } from "msw";
import { authorize } from "@/mocks/auth";
import { rolesTable } from "@/mocks/db";
import { apiUrl } from "@/mocks/utils";

export const roleHandlers = [
  // 역할 목록. 사용자 등록 화면의 선택지 등으로 쓰므로 로그인만 하면 조회할 수 있다.
  http.get(apiUrl("/roles"), ({ request }) => {
    const { error } = authorize(request);
    if (error) return error;
    return HttpResponse.json(rolesTable.all());
  }),
];
```

✏️ **`src/mocks/handlers.ts`** — 등록합니다.

```ts
import { menuHandlers } from "@/mocks/handlers/menus";
```

바꿀 코드

```ts
import { menuHandlers } from "@/mocks/handlers/menus";
import { roleHandlers } from "@/mocks/handlers/roles";
```

✏️ **`src/mocks/handlers.ts`** — 배열에 추가합니다.

```ts
  ...menuHandlers,
];
```

바꿀 코드

```ts
  ...menuHandlers,
  ...roleHandlers,
];
```

📄 **`src/shared/auth/useRoles.ts`** (새 파일)

```ts
import { useQuery } from "@tanstack/react-query";
import { httpClient } from "@/shared/api/httpClient";
import type { Role } from "@/shared/auth/types";

export const rolesKey = ["roles"] as const;

/** 역할 목록. 거의 바뀌지 않으므로 권한 관리에서 저장할 때만 다시 받는다. */
export function useRoles() {
  return useQuery({
    queryKey: rolesKey,
    queryFn: () => httpClient.get<Role[]>("/roles"),
    staleTime: Infinity,
  });
}
```

가짜 서버에 등록·수정·삭제 API를 추가합니다.

📄 **`src/mocks/handlers/users.ts`** (전체 교체)

```ts
import { http, HttpResponse } from "msw";
import type { UserInput } from "@/features/users/types";
import { authorize } from "@/mocks/auth";
import { credentialsTable, usersTable } from "@/mocks/db";
import { apiUrl, problem } from "@/mocks/utils";

const today = () => new Date().toISOString().slice(0, 10);

export const userHandlers = [
  http.get(apiUrl("/users"), ({ request }) => {
    const { error } = authorize(request, "USER_MGMT", "READ");
    if (error) return error;

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

  http.post(apiUrl("/users"), async ({ request }) => {
    const { error } = authorize(request, "USER_MGMT", "CREATE");
    if (error) return error;

    const input = (await request.json()) as UserInput;
    const user = usersTable.insert({ ...input, createdAt: today() });
    // 새 사용자의 초기 비밀번호는 "아이디1234"
    credentialsTable.insert({ username: user.username, password: `${user.username}1234` });
    return HttpResponse.json(user, { status: 201 });
  }),

  http.put(apiUrl("/users/:id"), async ({ request, params }) => {
    const { error } = authorize(request, "USER_MGMT", "UPDATE");
    if (error) return error;

    const input = (await request.json()) as UserInput;
    // 아이디(username)는 바꿀 수 없다. 바꿀 수 있는 항목만 골라서 저장한다.
    const user = usersTable.update(Number(params.id), {
      name: input.name,
      email: input.email,
      phone: input.phone,
      roleCodes: input.roleCodes,
      useYn: input.useYn,
    });
    if (!user) return problem(404, "사용자를 찾을 수 없습니다.");
    return HttpResponse.json(user);
  }),
];
```

앱 쪽입니다. 등록·수정에 쓰는 입력 타입을 추가합니다.

✏️ **`src/features/users/types.ts`** — 파일 끝에 추가합니다.

```ts
export interface UserSearchCondition {
```

바꿀 코드

```ts
/** 등록·수정할 때 보내는 값. id와 등록일은 서버가 정한다. */
export type UserInput = Omit<User, "id" | "createdAt">;

export interface UserSearchCondition {
```

✏️ **`src/features/users/api/usersApi.ts`** — 등록·수정 함수를 추가합니다.

```ts
  list: (condition: UserSearchCondition, signal?: AbortSignal) =>
    httpClient.get<User[]>("/users", { params: condition, signal }),
};
```

바꿀 코드

```ts
  list: (condition: UserSearchCondition, signal?: AbortSignal) =>
    httpClient.get<User[]>("/users", { params: condition, signal }),
  create: (input: UserInput) => httpClient.post<User>("/users", input),
  update: (id: number, input: UserInput) => httpClient.put<User>(`/users/${id}`, input),
};
```

✏️ **`src/features/users/api/usersApi.ts`** — import를 고칩니다.

```ts
import type { User, UserSearchCondition } from "@/features/users/types";
```

바꿀 코드

```ts
import type { User, UserInput, UserSearchCondition } from "@/features/users/types";
```

✏️ **`src/features/users/api/queries.ts`** — 저장 훅을 추가합니다.

```ts
export function useUserList(condition: UserSearchCondition) {
```

바꿀 코드

```ts
/** 등록(id 없음)과 수정(id 있음)을 하나로 처리한다. 성공하면 사용자 캐시를 모두 무효화한다. */
export function useSaveUser() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: ({ id, input }: { id?: number; input: UserInput }) =>
      id ? usersApi.update(id, input) : usersApi.create(input),
    meta: { skipGlobalError: true }, // 오류는 폼이 직접 보여 준다 (Step 9-4)
    onSuccess: () => queryClient.invalidateQueries({ queryKey: userKeys.all }),
  });
}

export function useUserList(condition: UserSearchCondition) {
```

✏️ **`src/features/users/api/queries.ts`** — import를 고칩니다.

```ts
import { useQuery } from "@tanstack/react-query";
import { usersApi } from "@/features/users/api/usersApi";
import type { UserSearchCondition } from "@/features/users/types";
```

바꿀 코드

```ts
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import { usersApi } from "@/features/users/api/usersApi";
import type { UserInput, UserSearchCondition } from "@/features/users/types";
```

화면 문구를 추가합니다. 역할 이름은 이제 서버의 역할 데이터에서 가져오므로 `role` 항목은 뺍니다.

📄 **`src/shared/i18n/locales/ko/users.json`** (전체 교체)

```json
{
  "search": {
    "name": "이름",
    "namePlaceholder": "이름 입력",
    "role": "역할",
    "useYn": "사용 여부"
  },
  "column": {
    "username": "아이디",
    "name": "이름",
    "email": "이메일",
    "phone": "전화번호",
    "roles": "역할",
    "useYn": "사용"
  },
  "form": {
    "createTitle": "사용자 등록",
    "editTitle": "사용자 수정",
    "usernameRule": "영문 소문자, 숫자, _ 로 3~20자를 입력하세요.",
    "phoneRule": "010-1234-5678 형식으로 입력하세요.",
    "initialPassword": "초기 비밀번호는 \"아이디1234\"입니다."
  }
}
```

📄 **`src/shared/i18n/locales/en/users.json`** (전체 교체)

```json
{
  "search": {
    "name": "Name",
    "namePlaceholder": "Enter a name",
    "role": "Role",
    "useYn": "Status"
  },
  "column": {
    "username": "User ID",
    "name": "Name",
    "email": "Email",
    "phone": "Phone",
    "roles": "Roles",
    "useYn": "Use"
  },
  "form": {
    "createTitle": "New user",
    "editTitle": "Edit user",
    "usernameRule": "Use 3-20 lowercase letters, digits, or underscores.",
    "phoneRule": "Use the format 010-1234-5678.",
    "initialPassword": "The initial password is \"<user ID>1234\"."
  }
}
```

등록·수정 모달입니다. 이 기능에서만 쓰는 부품이라 `features/users/components/`에 둡니다.

📄 **`src/features/users/components/UserFormModal.tsx`** (새 파일)

```tsx
import { useTranslation } from "react-i18next";
import { App, Form, Input, Modal, Radio, Select } from "antd";
import { useSaveUser } from "@/features/users/api/queries";
import type { User, UserInput } from "@/features/users/types";
import { showApiError } from "@/shared/api/showApiError";

interface UserFormModalProps {
  user?: User; // 없으면 등록, 있으면 수정
  roleOptions: { value: string; label: string }[];
  onClose: () => void;
}

function UserFormModal({ user, roleOptions, onClose }: UserFormModalProps) {
  const { t } = useTranslation("users");
  const { message } = App.useApp();
  const [form] = Form.useForm<UserInput>();
  const saveUser = useSaveUser();
  const isEdit = user !== undefined;

  // 필수 입력 규칙. 메시지는 "<항목> 입력은 필수입니다."
  const required = (field: string) => ({
    required: true,
    message: t("common:validation.required", { field }),
  });

  const handleFinish = (input: UserInput) => {
    saveUser.mutate(
      { id: user?.id, input },
      {
        onSuccess: () => {
          message.success(t("common:message.saved"));
          onClose();
        },
        onError: showApiError,
      },
    );
  };

  return (
    <Modal
      open
      title={isEdit ? t("form.editTitle") : t("form.createTitle")}
      okText={t("common:button.save")}
      cancelText={t("common:button.cancel")}
      onOk={form.submit}
      onCancel={onClose}
      confirmLoading={saveUser.isPending}
      mask={{ closable: false }} // 바깥을 실수로 눌러 입력 중인 내용이 날아가지 않게 한다
    >
      <Form
        form={form}
        name="userForm" // 입력칸 id가 조회 영역의 "name"과 겹치지 않게 한다 (Step 9-1 참고)
        layout="vertical"
        initialValues={user ?? { roleCodes: ["USER"], useYn: "Y" }}
        onFinish={handleFinish}
      >
        <Form.Item
          name="username"
          label={t("column.username")}
          extra={isEdit ? undefined : t("form.initialPassword")}
          rules={[
            required(t("column.username")),
            { pattern: /^[a-z0-9_]{3,20}$/, message: t("form.usernameRule") },
          ]}
        >
          <Input disabled={isEdit} />
        </Form.Item>
        <Form.Item name="name" label={t("column.name")} rules={[required(t("column.name"))]}>
          <Input />
        </Form.Item>
        <Form.Item
          name="email"
          label={t("column.email")}
          rules={[
            required(t("column.email")),
            { type: "email", message: t("common:validation.email") },
          ]}
        >
          <Input />
        </Form.Item>
        <Form.Item
          name="phone"
          label={t("column.phone")}
          rules={[{ pattern: /^01\d-\d{3,4}-\d{4}$/, message: t("form.phoneRule") }]}
        >
          <Input placeholder="010-1234-5678" />
        </Form.Item>
        <Form.Item name="roleCodes" label={t("column.roles")} rules={[required(t("column.roles"))]}>
          <Select mode="multiple" options={roleOptions} />
        </Form.Item>
        <Form.Item name="useYn" label={t("column.useYn")}>
          <Radio.Group
            options={[
              { value: "Y", label: t("common:label.use") },
              { value: "N", label: t("common:label.unuse") },
            ]}
          />
        </Form.Item>
      </Form>
    </Modal>
  );
}

export default UserFormModal;
```

화면에 연결합니다. 역할 이름도 서버의 역할 데이터로 바꿉니다.

📄 **`src/features/users/pages/UsersPage.tsx`** (전체 교체)

```tsx
import { useMemo, useState } from "react";
import { useTranslation } from "react-i18next";
import { Button, Form, Input, Select } from "antd";
import type { ColDef } from "ag-grid-community";
import { useUserList } from "@/features/users/api/queries";
import UserFormModal from "@/features/users/components/UserFormModal";
import type { User, UserSearchCondition } from "@/features/users/types";
import Can from "@/shared/auth/Can";
import { useRoles } from "@/shared/auth/useRoles";
import { useLocalizedText } from "@/shared/i18n/useLocalizedText";
import DataGrid from "@/shared/ui/DataGrid";
import PageContainer from "@/shared/ui/PageContainer";
import SearchForm from "@/shared/ui/SearchForm";

function UsersPage() {
  const { t } = useTranslation("users");
  const localize = useLocalizedText();
  const { data: roles = [] } = useRoles();
  const [condition, setCondition] = useState<UserSearchCondition>({});
  const { data, isFetching, refetch } = useUserList(condition);
  // 등록·수정 모달. null이면 닫힘, user가 없으면 등록, 있으면 수정
  const [editing, setEditing] = useState<{ user?: User } | null>(null);

  const roleOptions = useMemo(
    () => roles.map((role) => ({ value: role.code, label: localize(role.names) })),
    [roles, localize],
  );

  // 언어나 역할 목록이 바뀔 때만 컬럼 정의를 다시 만든다.
  const columnDefs = useMemo<ColDef<User>[]>(() => {
    const roleName = (code: string) =>
      roleOptions.find((option) => option.value === code)?.label ?? code;
    return [
      { headerName: t("column.username"), field: "username" },
      { headerName: t("column.name"), field: "name" },
      { headerName: t("column.email"), field: "email", flex: 2 },
      { headerName: t("column.phone"), field: "phone" },
      {
        headerName: t("column.roles"),
        field: "roleCodes",
        // 값이 배열이면 그리드는 어떻게 보여 줄지 모른다. 글자로 바꾸는 방법을 알려 준다.
        valueFormatter: ({ value }) => value?.map(roleName).join(", "),
      },
      { headerName: t("column.useYn"), field: "useYn", width: 90, flex: 0 },
      { headerName: t("common:label.createdAt"), field: "createdAt" },
    ];
  }, [t, roleOptions]);

  return (
    <PageContainer
      extra={
        <Can action="CREATE">
          <Button type="primary" onClick={() => setEditing({})}>
            {t("common:button.create")}
          </Button>
        </Can>
      }
    >
      <SearchForm<UserSearchCondition> onSearch={setCondition} onSameSearch={refetch}>
        <Form.Item name="name" label={t("search.name")}>
          <Input placeholder={t("search.namePlaceholder")} allowClear />
        </Form.Item>
        <Form.Item name="roleCode" label={t("search.role")}>
          <Select
            placeholder={t("common:label.all")}
            allowClear
            style={{ width: 140 }}
            options={roleOptions}
          />
        </Form.Item>
      </SearchForm>
      <DataGrid<User> rowData={data} columnDefs={columnDefs} loading={isFetching} />
      {/* 열 때마다 새로 만든다. 이전에 입력하던 값이 다음에 열 때 남아 있는 버그를 원천 차단한다. */}
      {editing && (
        <UserFormModal
          user={editing.user}
          roleOptions={roleOptions}
          onClose={() => setEditing(null)}
        />
      )}
    </PageContainer>
  );
}

export default UsersPage;
```

### 코드 읽어 보기

**`{editing && <UserFormModal ... />}`**: 모달을 항상 그려 두고 `open`만 바꾸는 방식도 있지만, 그러면 폼 입력값이 남아서
"등록을 누르면 방금 수정하던 사람이 들어 있는" 버그가 생기기 쉽습니다. **열 때 만들고 닫을 때 없애면** 언제나 깨끗한 폼으로 시작합니다.

**`mutate(값, { onSuccess, onError })`**: 이 호출에만 적용되는 처리입니다. 모달 닫기·알림처럼 화면에 관한 일은 여기서,
캐시 무효화처럼 데이터에 관한 일은 훅(`useSaveUser`)의 `onSuccess`에서 합니다. 어느 화면에서 저장하든 캐시는 반드시 갱신됩니다.

**`rules`**: antd 폼의 입력 검증입니다. 서버에 보내기 전에 화면에서 먼저 막아 사용자 경험을 좋게 합니다. (서버도 다시 검사해야 합니다. Step 9-4)

**`App.useApp()`의 `message`**: Part 3에서 배운 대로, 정적 `message.success()` 대신 이것을 씁니다. 다크 모드·언어가 적용됩니다.

### 확인하기

- [ ] `admin`으로 [등록] → 모달 → 아무것도 안 넣고 [저장] → 입력칸 아래 빨간 안내
- [ ] 아이디에 `Hong`(대문자) → 형식 안내. `hong`, 이름, 이메일을 넣고 [저장] → "저장되었습니다." → 목록에 바로 추가됩니다.
- [ ] 새로고침해도 남아 있습니다. (가짜 DB)
- [ ] 로그아웃 후 `hong` / `hong1234`로 로그인됩니다. (역할이 USER라 메뉴가 적습니다)

✅ **커밋**

```powershell
git add .
git commit -m "feat: 사용자 등록 기능 추가"
```

---

## Step 9-3. 사용자 수정·삭제

> 🎯 **할 일**: 행을 더블클릭하면 수정, 체크한 행은 확인 후 삭제되게 합니다.

### 따라 하기

가짜 서버에 삭제 API를 추가합니다. 자기 자신은 지울 수 없게 막습니다.

✏️ **`src/mocks/handlers/users.ts`** — 배열 끝에 추가합니다.

```ts
    if (!user) return problem(404, "사용자를 찾을 수 없습니다.");
    return HttpResponse.json(user);
  }),
];
```

바꿀 코드

```ts
    if (!user) return problem(404, "사용자를 찾을 수 없습니다.");
    return HttpResponse.json(user);
  }),

  http.delete(apiUrl("/users/:id"), ({ request, params }) => {
    const { user, error } = authorize(request, "USER_MGMT", "DELETE");
    if (error) return error;
    if (user.id === Number(params.id)) return problem(400, "자기 자신은 삭제할 수 없습니다.");
    usersTable.remove(Number(params.id));
    return new HttpResponse(null, { status: 204 });
  }),
];
```

✏️ **`src/features/users/api/usersApi.ts`** — 삭제 함수를 추가합니다.

```ts
  update: (id: number, input: UserInput) => httpClient.put<User>(`/users/${id}`, input),
```

바꿀 코드

```ts
  update: (id: number, input: UserInput) => httpClient.put<User>(`/users/${id}`, input),
  remove: (id: number) => httpClient.delete(`/users/${id}`),
```

✏️ **`src/features/users/api/queries.ts`** — 삭제 훅을 추가합니다.

```ts
export function useUserList(condition: UserSearchCondition) {
```

바꿀 코드

```ts
/** 여러 명을 한꺼번에 삭제한다. 하나씩 DELETE 요청을 보내고 모두 끝날 때까지 기다린다. */
export function useDeleteUsers() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (ids: number[]) => Promise.all(ids.map((id) => usersApi.remove(id))),
    onSettled: () => queryClient.invalidateQueries({ queryKey: userKeys.all }),
  });
}

export function useUserList(condition: UserSearchCondition) {
```

> 💡 **팁** `onSettled`는 성공·실패 상관없이 끝나면 불립니다. 3명 중 2명만 지워지고 1명이 실패해도 목록은 다시 받아야 하므로 `onSuccess`가 아니라 `onSettled`를 씁니다.

화면입니다.

✏️ **`src/features/users/pages/UsersPage.tsx`** — 상태와 삭제 처리를 추가합니다.

```tsx
  // 등록·수정 모달. null이면 닫힘, user가 없으면 등록, 있으면 수정
  const [editing, setEditing] = useState<{ user?: User } | null>(null);
```

바꿀 코드

```tsx
  // 등록·수정 모달. null이면 닫힘, user가 없으면 등록, 있으면 수정
  const [editing, setEditing] = useState<{ user?: User } | null>(null);
  const [selected, setSelected] = useState<User[]>([]);
  const deleteUsers = useDeleteUsers();
  const { message, modal } = App.useApp();
  const { can } = usePermission();

  const handleDelete = () => {
    if (selected.length === 0) {
      message.warning(t("common:message.selectRows"));
      return;
    }
    modal.confirm({
      title: t("common:message.confirmDelete", { count: selected.length }),
      okButtonProps: { danger: true },
      // Promise를 돌려주면 끝날 때까지 [확인] 버튼이 로딩 상태로 기다린다.
      onOk: () =>
        deleteUsers
          .mutateAsync(selected.map((user) => user.id))
          .then(() => message.success(t("common:message.deleted")))
          // 실패 알림은 전역 처리기가 이미 띄웠다. 실패를 여기서 받아야 확인 창이 닫힌다.
          .catch(() => undefined),
    });
  };
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — 삭제 버튼을 추가합니다.

```tsx
      extra={
        <Can action="CREATE">
          <Button type="primary" onClick={() => setEditing({})}>
            {t("common:button.create")}
          </Button>
        </Can>
      }
```

바꿀 코드

```tsx
      extra={
        <Space>
          <Can action="CREATE">
            <Button type="primary" onClick={() => setEditing({})}>
              {t("common:button.create")}
            </Button>
          </Can>
          <Can action="DELETE">
            <Button danger onClick={handleDelete}>
              {t("common:button.delete")}
            </Button>
          </Can>
        </Space>
      }
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — 그리드에 선택·더블클릭을 연결합니다.

```tsx
      <DataGrid<User> rowData={data} columnDefs={columnDefs} loading={isFetching} />
```

바꿀 코드

```tsx
      <DataGrid<User>
        rowData={data}
        columnDefs={columnDefs}
        loading={isFetching}
        // 행을 id로 구별한다. 다시 조회해도 같은 행으로 인식해서 선택 상태가 유지된다.
        getRowId={({ data: user }) => String(user.id)}
        rowSelection={{ mode: "multiRow" }}
        onSelectionChanged={({ api }) => setSelected(api.getSelectedRows())}
        onRowDoubleClicked={({ data: user }) => {
          if (user && can("UPDATE")) setEditing({ user });
        }}
      />
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — import를 고칩니다.

```tsx
import { Button, Form, Input, Select } from "antd";
import type { ColDef } from "ag-grid-community";
import { useUserList } from "@/features/users/api/queries";
```

바꿀 코드

```tsx
import { App, Button, Form, Input, Select, Space } from "antd";
import type { ColDef } from "ag-grid-community";
import { useDeleteUsers, useUserList } from "@/features/users/api/queries";
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — import를 추가합니다.

```tsx
import Can from "@/shared/auth/Can";
```

바꿀 코드

```tsx
import Can from "@/shared/auth/Can";
import { usePermission } from "@/shared/auth/usePermission";
```

### 코드 읽어 보기

**`getRowId`**: 이게 없으면 그리드는 다시 조회할 때마다 모든 행을 새것으로 봅니다. 선택이 풀리고 스크롤도 튑니다.
업무 그리드에는 거의 항상 넣습니다.

**`modal.confirm`의 `onOk`에서 Promise 돌려주기**: 삭제가 끝날 때까지 확인 창이 [확인] 버튼을 로딩 상태로 유지합니다. 중복 클릭이 저절로 막힙니다.

**`mutateAsync` vs `mutate`**: `mutateAsync`는 Promise를 돌려줘서 `.then`·`await`를 쓸 수 있습니다. 대신 실패하면 예외가 나므로 받는 쪽에서 처리해야 합니다.
antd 확인 창은 `onOk`의 Promise가 **실패하면 닫히지 않고 그대로 남습니다.** 실패 알림은 전역 처리기가 이미 띄웠으니,
`.catch(() => undefined)`로 실패를 받아서 확인 창이 닫히게 합니다.

### 확인하기

- [ ] 행을 더블클릭 → 수정 모달. 아이디는 바꿀 수 없습니다. 이름을 바꿔 저장하면 목록에 바로 반영됩니다.
- [ ] 체크 2개 → [삭제] → "2건을 삭제하시겠습니까?" → [확인] → 목록에서 사라집니다.
- [ ] 김민수(admin 자신)를 체크하고 삭제 → "자기 자신은 삭제할 수 없습니다." 알림
- [ ] `manager`로 로그인 → [등록]·[삭제] 버튼이 없고, 더블클릭해도 모달이 열리지 않습니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 사용자 수정·삭제 기능 추가"
```

---

## Step 9-4. 서버 검증 오류를 입력칸에 표시

> 🎯 **할 일**: 이미 있는 이메일로 등록하면, 알림 대신 **이메일 입력칸 아래에** 빨간 글씨가 나오게 합니다.

### 원리

화면 검증(`rules`)은 형식만 볼 수 있습니다. "이미 사용 중인가"는 서버만 압니다.
Part 3에서 정한 오류 형식(Problem Details)의 `errors` 배열에 어느 칸이 왜 틀렸는지 들어 있으므로, 그대로 폼에 붙이면 됩니다.

```json
{ "status": 409, "detail": "이미 사용 중인 이메일입니다.", "errors": [{ "field": "email", "message": "이미 사용 중인 이메일입니다." }] }
```

### 따라 하기

📄 **`src/shared/ui/applyServerErrors.ts`** (새 파일)

```ts
import type { FormInstance } from "antd";
import { ApiError } from "@/shared/api/httpClient";

/**
 * 서버 검증 오류(Problem Details의 errors)를 antd 폼 입력칸에 붙인다.
 * 붙였으면 true를 돌려준다. 이때는 전역 알림을 띄우지 않는다.
 */
export function applyServerErrors(form: FormInstance, error: unknown) {
  if (!(error instanceof ApiError) || !error.problem.errors?.length) return false;
  form.setFields(
    error.problem.errors.map(({ field, message }) => ({ name: field, errors: [message] })),
  );
  return true;
}
```

✏️ **`src/features/users/components/UserFormModal.tsx`** — 오류 처리를 바꿉니다.

```tsx
        onError: showApiError,
```

바꿀 코드

```tsx
        // 입력칸에 붙일 수 있는 오류면 붙이고, 아니면 전역 알림으로 보여 준다.
        onError: (error) => {
          if (!applyServerErrors(form, error)) showApiError(error);
        },
```

✏️ **`src/features/users/components/UserFormModal.tsx`** — import를 추가합니다.

```tsx
import { showApiError } from "@/shared/api/showApiError";
```

바꿀 코드

```tsx
import { showApiError } from "@/shared/api/showApiError";
import { applyServerErrors } from "@/shared/ui/applyServerErrors";
```

가짜 서버가 중복을 검사하게 합니다. **화면에서 막아도 서버는 반드시 다시 검사합니다.**

✏️ **`src/mocks/handlers/users.ts`** — 검증 함수를 추가합니다.

```ts
const today = () => new Date().toISOString().slice(0, 10);
```

바꿀 코드

```ts
const today = () => new Date().toISOString().slice(0, 10);

/** 서버 검증: 중복 확인. 문제가 있으면 409 응답을, 없으면 null을 돌려준다. */
function validate(input: UserInput, id?: number) {
  const others = usersTable.all().filter((user) => user.id !== id);
  const errors: { field: string; message: string }[] = [];
  if (!id && others.some((user) => user.username === input.username)) {
    errors.push({ field: "username", message: "이미 사용 중인 아이디입니다." });
  }
  if (others.some((user) => user.email === input.email)) {
    errors.push({ field: "email", message: "이미 사용 중인 이메일입니다." });
  }
  return errors.length > 0 ? problem(409, errors[0].message, { errors }) : null;
}
```

✏️ **`src/mocks/handlers/users.ts`** — 등록할 때 검사합니다.

```ts
    const input = (await request.json()) as UserInput;
    const user = usersTable.insert({ ...input, createdAt: today() });
```

바꿀 코드

```ts
    const input = (await request.json()) as UserInput;
    const invalid = validate(input);
    if (invalid) return invalid;
    const user = usersTable.insert({ ...input, createdAt: today() });
```

✏️ **`src/mocks/handlers/users.ts`** — 수정할 때도 검사합니다.

```ts
    const input = (await request.json()) as UserInput;
    // 아이디(username)는 바꿀 수 없다. 바꿀 수 있는 항목만 골라서 저장한다.
```

바꿀 코드

```ts
    const input = (await request.json()) as UserInput;
    const invalid = validate(input, Number(params.id));
    if (invalid) return invalid;
    // 아이디(username)는 바꿀 수 없다. 바꿀 수 있는 항목만 골라서 저장한다.
```

### 확인하기

- [ ] [등록] → 아이디 `admin` → [저장] → **아이디 칸 아래에** "이미 사용 중인 아이디입니다."
- [ ] 이메일 `minsu.kim@example.com` → 이메일 칸 아래에 안내
- [ ] 칸을 고치면 빨간 글씨가 사라집니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 서버 검증 오류를 폼 필드에 표시"
```

---

## Step 9-5. 공통코드 연동

> 🎯 **할 일**: "사용/미사용", "공지 유형" 같은 선택지를 코드에 박지 않고 **공통코드** 데이터로 관리합니다.

### 원리

SI 시스템에는 화면마다 선택지가 있습니다. 성별, 부서, 공지 유형, 사용 여부... 이걸 코드에 박으면 선택지를 바꿀 때마다 배포해야 합니다.
**공통코드**는 이런 선택지를 `그룹 - 코드 - 이름` 형태로 DB에 두고, 관리 화면(Step 9-6)에서 바꿉니다.

| 그룹코드      | 코드      | 이름 (ko / en)   |
| ------------- | --------- | ---------------- |
| `USE_YN`      | `Y`       | 사용 / In use    |
| `NOTICE_TYPE` | `URGENT`  | 긴급 / Urgent    |
| `DEPT`        | `DEV`     | 개발팀 / Development |

공통코드는 거의 바뀌지 않으므로 **한 번 받고 계속 재사용**합니다(`staleTime: Infinity`). 관리 화면에서 저장할 때만 무효화합니다.

| 부품                              | 용도                                  |
| --------------------------------- | ------------------------------------- |
| `<CodeSelect group="DEPT" />`     | 조회 조건·입력 폼의 Select            |
| `<CodeRadio group="USE_YN" />`    | 항목이 적을 때 라디오                 |
| `useCodeName("USE_YN")`           | 그리드 등에서 코드 → 이름으로 바꾸기  |

### 따라 하기

📄 **`src/shared/codes/types.ts`** (새 파일)

```ts
import type { UseYn } from "@/shared/api/types";
import type { LocalizedText } from "@/shared/i18n/locales";

export interface CodeGroup {
  id: number;
  groupCode: string; // 예: "NOTICE_TYPE"
  groupName: string; // 관리자용 이름. 예: "공지 유형"
  description: string;
  useYn: UseYn;
}

export interface Code {
  id: number;
  groupCode: string;
  code: string; // 예: "URGENT"
  names: LocalizedText; // 화면에 보이는 이름. 예: { ko: "긴급", en: "Urgent" }
  sortOrder: number;
  useYn: UseYn;
  remark: string;
}
```

📄 **`src/mocks/data/codes.ts`** (새 파일)

```ts
import type { Code, CodeGroup } from "@/shared/codes/types";

export const codeGroupSeed: CodeGroup[] = [
  { id: 1, groupCode: "USE_YN", groupName: "사용 여부", description: "Y / N", useYn: "Y" },
  {
    id: 2,
    groupCode: "NOTICE_TYPE",
    groupName: "공지 유형",
    description: "공지사항 구분",
    useYn: "Y",
  },
  { id: 3, groupCode: "GENDER", groupName: "성별", description: "", useYn: "Y" },
  { id: 4, groupCode: "DEPT", groupName: "부서", description: "샘플 부서", useYn: "Y" },
];

function code(
  id: number,
  groupCode: string,
  value: string,
  ko: string,
  en: string,
  remark = "",
): Code {
  return { id, groupCode, code: value, names: { ko, en }, sortOrder: id, useYn: "Y", remark };
}

export const codeSeed: Code[] = [
  code(1, "USE_YN", "Y", "사용", "In use"),
  code(2, "USE_YN", "N", "미사용", "Not in use"),
  code(3, "NOTICE_TYPE", "GENERAL", "일반", "General"),
  code(4, "NOTICE_TYPE", "URGENT", "긴급", "Urgent", "목록 상단에 강조 표시된다."),
  { ...code(5, "NOTICE_TYPE", "EVENT", "이벤트", "Event", "이벤트 종료로 미사용"), useYn: "N" },
  code(6, "GENDER", "M", "남성", "Male"),
  code(7, "GENDER", "F", "여성", "Female"),
  code(8, "DEPT", "DEV", "개발팀", "Development"),
  code(9, "DEPT", "SALES", "영업팀", "Sales"),
  code(10, "DEPT", "HR", "인사팀", "Human resources"),
];
```

✏️ **`src/mocks/db.ts`** — import를 추가합니다.

```ts
import { menuSeed } from "@/mocks/data/menus";
```

바꿀 코드

```ts
import { codeGroupSeed, codeSeed } from "@/mocks/data/codes";
import { menuSeed } from "@/mocks/data/menus";
```

✏️ **`src/mocks/db.ts`** — 테이블을 추가합니다.

```ts
export const rolesTable = createTable("roles", roleSeed);
```

바꿀 코드

```ts
export const rolesTable = createTable("roles", roleSeed);
export const codeGroupsTable = createTable("codeGroups", codeGroupSeed);
export const codesTable = createTable("codes", codeSeed);
```

📄 **`src/mocks/handlers/codes.ts`** (새 파일)

```ts
import { http, HttpResponse } from "msw";
import { authorize } from "@/mocks/auth";
import { codesTable } from "@/mocks/db";
import { apiUrl } from "@/mocks/utils";

export const codeHandlers = [
  // 화면 선택지용 공통코드. 사용 중인 코드만, 정렬 순서대로 준다. 예: /codes?groups=USE_YN,DEPT
  http.get(apiUrl("/codes"), ({ request }) => {
    const { error } = authorize(request);
    if (error) return error;
    const groups = (new URL(request.url).searchParams.get("groups") ?? "").split(",");
    const rows = codesTable
      .all()
      .filter((code) => groups.includes(code.groupCode) && code.useYn === "Y")
      .sort((a, b) => a.sortOrder - b.sortOrder);
    return HttpResponse.json(rows);
  }),
];
```

✏️ **`src/mocks/handlers.ts`** — 등록합니다.

```ts
import { authHandlers } from "@/mocks/handlers/auth";
```

바꿀 코드

```ts
import { authHandlers } from "@/mocks/handlers/auth";
import { codeHandlers } from "@/mocks/handlers/codes";
```

✏️ **`src/mocks/handlers.ts`** — 배열에 추가합니다.

```ts
  ...roleHandlers,
];
```

바꿀 코드

```ts
  ...roleHandlers,
  ...codeHandlers,
];
```

앱 쪽 훅과 부품입니다.

📄 **`src/shared/codes/useCodes.ts`** (새 파일)

```ts
import { useCallback } from "react";
import { useQuery } from "@tanstack/react-query";
import { httpClient } from "@/shared/api/httpClient";
import type { Code } from "@/shared/codes/types";
import { useLocalizedText } from "@/shared/i18n/useLocalizedText";

export const codesKey = ["codes"] as const;

/** 공통코드 그룹 하나의 코드 목록. 거의 바뀌지 않으므로 한 번 받고 계속 쓴다. */
export function useCodes(group: string) {
  return useQuery({
    queryKey: [...codesKey, group],
    queryFn: () => httpClient.get<Code[]>("/codes", { params: { groups: group } }),
    staleTime: Infinity, // 공통코드 관리에서 저장할 때만 무효화한다
  });
}

/** 코드값 → 현재 언어의 이름으로 바꾸는 함수. 그리드의 valueFormatter 등에서 쓴다. */
export function useCodeName(group: string) {
  const { data: codes = [] } = useCodes(group);
  const localize = useLocalizedText();
  return useCallback(
    (value?: string) => {
      const found = codes.find((code) => code.code === value);
      return found ? localize(found.names) : (value ?? "");
    },
    [codes, localize],
  );
}
```

📄 **`src/shared/codes/CodeSelect.tsx`** (새 파일)

```tsx
import { Select, type SelectProps } from "antd";
import { useCodes } from "@/shared/codes/useCodes";
import { useLocalizedText } from "@/shared/i18n/useLocalizedText";

interface CodeSelectProps extends Omit<SelectProps, "options"> {
  group: string;
}

/** 공통코드 그룹으로 선택지를 만드는 Select. 나머지 속성은 antd Select와 같다. */
function CodeSelect({ group, ...props }: CodeSelectProps) {
  const { data: codes = [], isLoading } = useCodes(group);
  const localize = useLocalizedText();

  return (
    <Select
      loading={isLoading}
      options={codes.map((code) => ({ value: code.code, label: localize(code.names) }))}
      {...props}
    />
  );
}

export default CodeSelect;
```

📄 **`src/shared/codes/CodeRadio.tsx`** (새 파일)

```tsx
import { Radio, type RadioGroupProps } from "antd";
import { useCodes } from "@/shared/codes/useCodes";
import { useLocalizedText } from "@/shared/i18n/useLocalizedText";

interface CodeRadioProps extends Omit<RadioGroupProps, "options"> {
  group: string;
}

/** 공통코드 그룹으로 만드는 라디오 버튼 묶음. 항목이 2~4개일 때 Select보다 빠르게 고를 수 있다. */
function CodeRadio({ group, ...props }: CodeRadioProps) {
  const { data: codes = [] } = useCodes(group);
  const localize = useLocalizedText();

  return (
    <Radio.Group
      options={codes.map((code) => ({ value: code.code, label: localize(code.names) }))}
      {...props}
    />
  );
}

export default CodeRadio;
```

사용자 화면에 적용합니다. 사용 여부를 조회 조건·그리드·입력 폼에서 공통코드로 바꿉니다.

✏️ **`src/features/users/pages/UsersPage.tsx`** — 조회 조건을 추가합니다.

```tsx
            options={roleOptions}
          />
        </Form.Item>
      </SearchForm>
```

바꿀 코드

```tsx
            options={roleOptions}
          />
        </Form.Item>
        <Form.Item name="useYn" label={t("search.useYn")}>
          <CodeSelect
            group="USE_YN"
            placeholder={t("common:label.all")}
            allowClear
            style={{ width: 120 }}
          />
        </Form.Item>
      </SearchForm>
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — 그리드에서 코드를 이름으로 보여 줍니다.

```tsx
      { headerName: t("column.useYn"), field: "useYn", width: 90, flex: 0 },
      { headerName: t("common:label.createdAt"), field: "createdAt" },
    ];
  }, [t, roleOptions]);
```

바꿀 코드

```tsx
      {
        headerName: t("column.useYn"),
        field: "useYn",
        width: 100,
        flex: 0,
        valueFormatter: ({ value }) => formatUseYn(value),
      },
      { headerName: t("common:label.createdAt"), field: "createdAt" },
    ];
  }, [t, roleOptions, formatUseYn]);
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — 훅을 부릅니다.

```tsx
  const { data: roles = [] } = useRoles();
```

바꿀 코드

```tsx
  const { data: roles = [] } = useRoles();
  const formatUseYn = useCodeName("USE_YN");
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — import를 추가합니다.

```tsx
import { useRoles } from "@/shared/auth/useRoles";
```

바꿀 코드

```tsx
import { useRoles } from "@/shared/auth/useRoles";
import CodeSelect from "@/shared/codes/CodeSelect";
import { useCodeName } from "@/shared/codes/useCodes";
```

✏️ **`src/features/users/components/UserFormModal.tsx`** — 사용 여부 라디오를 공통코드로 바꿉니다.

```tsx
          <Radio.Group
            options={[
              { value: "Y", label: t("common:label.use") },
              { value: "N", label: t("common:label.unuse") },
            ]}
          />
```

바꿀 코드

```tsx
          <CodeRadio group="USE_YN" />
```

✏️ **`src/features/users/components/UserFormModal.tsx`** — import를 고칩니다.

```tsx
import { App, Form, Input, Modal, Radio, Select } from "antd";
```

바꿀 코드

```tsx
import { App, Form, Input, Modal, Select } from "antd";
```

✏️ **`src/features/users/components/UserFormModal.tsx`** — import를 추가합니다.

```tsx
import { showApiError } from "@/shared/api/showApiError";
```

바꿀 코드

```tsx
import { showApiError } from "@/shared/api/showApiError";
import CodeRadio from "@/shared/codes/CodeRadio";
```

> ⚠️ **주의** 훅이 돌려준 **함수**를 담는 변수 이름을 `formatUseYn`("use_yn의 이름")처럼 지으면 ESLint가 오류를 냅니다.
> `use` + 대문자로 시작하는 이름은 **훅으로 간주**되는데, 그걸 `valueFormatter` 안에서 부르니 "훅은 컴포넌트 맨 위에서만 부를 수 있다"는 규칙 위반이 되기 때문입니다.
>
> ```
> React Hook "formatUseYn" is called in function "valueFormatter" ...   react-hooks/rules-of-hooks
> ```
>
> 그래서 `formatUseYn`처럼 동사로 시작하게 짓습니다. 실무 데이터에는 `useYn`처럼 `use`가 들어간 단어가 흔하니 이름 지을 때 조심하세요.

### 확인하기

- [ ] 사용 여부 조회 조건에 "사용/미사용"이 나오고, 그리드의 사용 컬럼이 `Y` 대신 "사용"으로 보입니다.
- [ ] English로 바꾸면 "In use / Not in use"로 바뀝니다.
- [ ] F12 → Network → `codes?groups=USE_YN` 요청이 **한 번만** 나갑니다. 화면을 오가도 다시 받지 않습니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 공통코드 선택 컴포넌트 추가"
```

---

## Step 9-6. 공통코드 관리 (마스터-디테일)

> 🎯 **할 일**: 왼쪽에서 코드 그룹을 고르면 오른쪽에 그 그룹의 코드가 나오는 관리 화면을 만듭니다.

### 원리: 마스터-디테일

SI에서 아주 흔한 화면 형태입니다. **왼쪽(마스터)에서 선택한 값이 오른쪽(디테일) 조회의 조건**이 됩니다.

```
선택한 그룹 state ─→ useCodesOfGroup(선택한 그룹)  ← queryKey에 그룹이 들어가 있으니 선택만 바꾸면 자동 조회
```

> 💡 **팁** AG Grid의 "마스터-디테일" 기능(행을 펼쳐 안쪽에 표 보여 주기)은 유료(Enterprise)입니다. 이 화면처럼 **표 두 개를 나란히** 두는 방식은 무료판으로 충분합니다.

### 따라 하기

가짜 서버에 관리용 API를 추가합니다.

📄 **`src/mocks/handlers/codes.ts`** (전체 교체)

```ts
import { http, HttpResponse } from "msw";
import { authorize } from "@/mocks/auth";
import { codeGroupsTable, codesTable } from "@/mocks/db";
import { apiUrl, problem } from "@/mocks/utils";
import type { Code, CodeGroup } from "@/shared/codes/types";

const MENU = "CODE_MGMT";

export const codeHandlers = [
  // 화면 선택지용 공통코드. 사용 중인 코드만, 정렬 순서대로 준다. 예: /codes?groups=USE_YN,DEPT
  http.get(apiUrl("/codes"), ({ request }) => {
    const { error } = authorize(request);
    if (error) return error;
    const groups = (new URL(request.url).searchParams.get("groups") ?? "").split(",");
    const rows = codesTable
      .all()
      .filter((code) => groups.includes(code.groupCode) && code.useYn === "Y")
      .sort((a, b) => a.sortOrder - b.sortOrder);
    return HttpResponse.json(rows);
  }),

  // ── 코드 그룹 관리 ──
  http.get(apiUrl("/code-groups"), ({ request }) => {
    const { error } = authorize(request, MENU, "READ");
    if (error) return error;
    return HttpResponse.json(codeGroupsTable.all());
  }),

  http.post(apiUrl("/code-groups"), async ({ request }) => {
    const { error } = authorize(request, MENU, "CREATE");
    if (error) return error;
    const input = (await request.json()) as Omit<CodeGroup, "id">;
    if (codeGroupsTable.all().some((group) => group.groupCode === input.groupCode)) {
      const message = "이미 있는 그룹코드입니다.";
      return problem(409, message, { errors: [{ field: "groupCode", message }] });
    }
    return HttpResponse.json(codeGroupsTable.insert(input), { status: 201 });
  }),

  http.put(apiUrl("/code-groups/:id"), async ({ request, params }) => {
    const { error } = authorize(request, MENU, "UPDATE");
    if (error) return error;
    const input = (await request.json()) as CodeGroup;
    // 그룹코드는 다른 데이터가 참조하는 키라서 바꾸지 않는다.
    const group = codeGroupsTable.update(Number(params.id), {
      groupName: input.groupName,
      description: input.description,
      useYn: input.useYn,
    });
    return group ? HttpResponse.json(group) : problem(404, "그룹을 찾을 수 없습니다.");
  }),

  http.delete(apiUrl("/code-groups/:id"), ({ request, params }) => {
    const { error } = authorize(request, MENU, "DELETE");
    if (error) return error;
    const group = codeGroupsTable.find(Number(params.id));
    if (codesTable.all().some((code) => code.groupCode === group?.groupCode)) {
      return problem(400, "코드가 있는 그룹은 삭제할 수 없습니다. 코드를 먼저 삭제하세요.");
    }
    codeGroupsTable.remove(Number(params.id));
    return new HttpResponse(null, { status: 204 });
  }),

  // ── 코드 관리 ──
  http.get(apiUrl("/code-groups/:groupCode/codes"), ({ request, params }) => {
    const { error } = authorize(request, MENU, "READ");
    if (error) return error;
    const rows = codesTable
      .all()
      .filter((code) => code.groupCode === params.groupCode)
      .sort((a, b) => a.sortOrder - b.sortOrder);
    return HttpResponse.json(rows);
  }),

  http.post(apiUrl("/codes"), async ({ request }) => {
    const { error } = authorize(request, MENU, "CREATE");
    if (error) return error;
    const input = (await request.json()) as Omit<Code, "id">;
    const exists = codesTable
      .all()
      .some((code) => code.groupCode === input.groupCode && code.code === input.code);
    if (exists) {
      const message = "그룹 안에 이미 있는 코드입니다.";
      return problem(409, message, { errors: [{ field: "code", message }] });
    }
    return HttpResponse.json(codesTable.insert(input), { status: 201 });
  }),

  http.put(apiUrl("/codes/:id"), async ({ request, params }) => {
    const { error } = authorize(request, MENU, "UPDATE");
    if (error) return error;
    const input = (await request.json()) as Code;
    const code = codesTable.update(Number(params.id), {
      names: input.names,
      sortOrder: input.sortOrder,
      useYn: input.useYn,
      remark: input.remark,
    });
    return code ? HttpResponse.json(code) : problem(404, "코드를 찾을 수 없습니다.");
  }),

  http.delete(apiUrl("/codes/:id"), ({ request, params }) => {
    const { error } = authorize(request, MENU, "DELETE");
    if (error) return error;
    codesTable.remove(Number(params.id));
    return new HttpResponse(null, { status: 204 });
  }),
];
```

앱 쪽 API와 Query 훅입니다.

📄 **`src/features/common-codes/api/codeAdminApi.ts`** (새 파일)

```ts
import { httpClient } from "@/shared/api/httpClient";
import type { Code, CodeGroup } from "@/shared/codes/types";

export type CodeGroupInput = Omit<CodeGroup, "id">;
export type CodeInput = Omit<Code, "id">;

export const codeAdminApi = {
  groups: () => httpClient.get<CodeGroup[]>("/code-groups"),
  createGroup: (input: CodeGroupInput) => httpClient.post<CodeGroup>("/code-groups", input),
  updateGroup: (id: number, input: CodeGroupInput) =>
    httpClient.put<CodeGroup>(`/code-groups/${id}`, input),
  removeGroup: (id: number) => httpClient.delete(`/code-groups/${id}`),

  codes: (groupCode: string) => httpClient.get<Code[]>(`/code-groups/${groupCode}/codes`),
  createCode: (input: CodeInput) => httpClient.post<Code>("/codes", input),
  updateCode: (id: number, input: CodeInput) => httpClient.put<Code>(`/codes/${id}`, input),
  removeCode: (id: number) => httpClient.delete(`/codes/${id}`),
};
```

📄 **`src/features/common-codes/api/queries.ts`** (새 파일)

```ts
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import {
  codeAdminApi,
  type CodeGroupInput,
  type CodeInput,
} from "@/features/common-codes/api/codeAdminApi";
import { codesKey } from "@/shared/codes/useCodes";

export const codeAdminKeys = {
  all: ["code-admin"] as const,
  groups: () => [...codeAdminKeys.all, "groups"] as const,
  codes: (groupCode: string) => [...codeAdminKeys.all, "codes", groupCode] as const,
};

export function useCodeGroups() {
  return useQuery({ queryKey: codeAdminKeys.groups(), queryFn: codeAdminApi.groups });
}

export function useCodesOfGroup(groupCode?: string) {
  return useQuery({
    queryKey: codeAdminKeys.codes(groupCode ?? ""),
    queryFn: () => codeAdminApi.codes(groupCode!),
    enabled: !!groupCode, // 그룹을 고르기 전에는 조회하지 않는다
  });
}

/** 관리 화면에서 무언가 바뀌면, 관리 화면 캐시와 함께 모든 화면의 공통코드 캐시(useCodes)도 무효화한다. */
function useInvalidateCodes() {
  const queryClient = useQueryClient();
  return () =>
    Promise.all([
      queryClient.invalidateQueries({ queryKey: codeAdminKeys.all }),
      queryClient.invalidateQueries({ queryKey: codesKey }),
    ]);
}

export function useSaveCodeGroup() {
  const invalidate = useInvalidateCodes();
  return useMutation({
    mutationFn: ({ id, input }: { id?: number; input: CodeGroupInput }) =>
      id ? codeAdminApi.updateGroup(id, input) : codeAdminApi.createGroup(input),
    meta: { skipGlobalError: true },
    onSuccess: invalidate,
  });
}

export function useDeleteCodeGroup() {
  const invalidate = useInvalidateCodes();
  return useMutation({ mutationFn: codeAdminApi.removeGroup, onSuccess: invalidate });
}

export function useSaveCode() {
  const invalidate = useInvalidateCodes();
  return useMutation({
    mutationFn: ({ id, input }: { id?: number; input: CodeInput }) =>
      id ? codeAdminApi.updateCode(id, input) : codeAdminApi.createCode(input),
    meta: { skipGlobalError: true },
    onSuccess: invalidate,
  });
}

export function useDeleteCode() {
  const invalidate = useInvalidateCodes();
  return useMutation({ mutationFn: codeAdminApi.removeCode, onSuccess: invalidate });
}
```

> 💡 **팁** `queryFn: () => codeAdminApi.codes(groupCode!)`의 `!`: `enabled: !!groupCode` 덕분에 `groupCode`가 있을 때만 실행되는 것을 우리는 알지만 TypeScript는 모릅니다.
> 그래서 "여기서는 확실히 있다"고 알려 줍니다. Step 1-2에서 본 그 `!`입니다.

문구 파일입니다.

📄 **`src/shared/i18n/locales/ko/codes.json`** (새 파일)

```json
{
  "group": {
    "title": "코드 그룹",
    "groupCode": "그룹코드",
    "groupName": "그룹명",
    "description": "설명",
    "createTitle": "코드 그룹 등록",
    "editTitle": "코드 그룹 수정"
  },
  "code": {
    "title": "코드",
    "code": "코드",
    "nameKo": "이름(한국어)",
    "nameEn": "이름(영어)",
    "sortOrder": "정렬",
    "remark": "비고",
    "createTitle": "코드 등록",
    "editTitle": "코드 수정",
    "selectGroup": "왼쪽에서 코드 그룹을 선택하세요."
  },
  "codeRule": "영문 대문자, 숫자, _ 만 쓸 수 있습니다."
}
```

📄 **`src/shared/i18n/locales/en/codes.json`** (새 파일)

```json
{
  "group": {
    "title": "Code groups",
    "groupCode": "Group code",
    "groupName": "Group name",
    "description": "Description",
    "createTitle": "New code group",
    "editTitle": "Edit code group"
  },
  "code": {
    "title": "Codes",
    "code": "Code",
    "nameKo": "Name (Korean)",
    "nameEn": "Name (English)",
    "sortOrder": "Order",
    "remark": "Remark",
    "createTitle": "New code",
    "editTitle": "Edit code",
    "selectGroup": "Select a code group on the left."
  },
  "codeRule": "Use uppercase letters, digits, and underscores only."
}
```

그룹 등록·수정 모달입니다. Step 9-2의 사용자 모달과 같은 패턴입니다.

📄 **`src/features/common-codes/components/CodeGroupFormModal.tsx`** (새 파일)

```tsx
import { useTranslation } from "react-i18next";
import { App, Form, Input, Modal } from "antd";
import type { CodeGroupInput } from "@/features/common-codes/api/codeAdminApi";
import { useSaveCodeGroup } from "@/features/common-codes/api/queries";
import { showApiError } from "@/shared/api/showApiError";
import CodeRadio from "@/shared/codes/CodeRadio";
import type { CodeGroup } from "@/shared/codes/types";
import { applyServerErrors } from "@/shared/ui/applyServerErrors";

interface CodeGroupFormModalProps {
  group?: CodeGroup;
  onClose: () => void;
}

function CodeGroupFormModal({ group, onClose }: CodeGroupFormModalProps) {
  const { t } = useTranslation("codes");
  const { message } = App.useApp();
  const [form] = Form.useForm<CodeGroupInput>();
  const saveGroup = useSaveCodeGroup();
  const required = (field: string) => ({
    required: true,
    message: t("common:validation.required", { field }),
  });

  const handleFinish = (input: CodeGroupInput) => {
    saveGroup.mutate(
      { id: group?.id, input },
      {
        onSuccess: () => {
          message.success(t("common:message.saved"));
          onClose();
        },
        onError: (error) => {
          if (!applyServerErrors(form, error)) showApiError(error);
        },
      },
    );
  };

  return (
    <Modal
      open
      title={group ? t("group.editTitle") : t("group.createTitle")}
      okText={t("common:button.save")}
      cancelText={t("common:button.cancel")}
      onOk={form.submit}
      onCancel={onClose}
      confirmLoading={saveGroup.isPending}
      mask={{ closable: false }}
    >
      <Form
        form={form}
        name="codeGroupForm"
        layout="vertical"
        initialValues={group ?? { useYn: "Y", description: "" }}
        onFinish={handleFinish}
      >
        <Form.Item
          name="groupCode"
          label={t("group.groupCode")}
          rules={[
            required(t("group.groupCode")),
            { pattern: /^[A-Z0-9_]+$/, message: t("codeRule") },
          ]}
        >
          <Input disabled={!!group} />
        </Form.Item>
        <Form.Item
          name="groupName"
          label={t("group.groupName")}
          rules={[required(t("group.groupName"))]}
        >
          <Input />
        </Form.Item>
        <Form.Item name="description" label={t("group.description")}>
          <Input.TextArea rows={2} />
        </Form.Item>
        <Form.Item name="useYn" label={t("common:label.use")}>
          <CodeRadio group="USE_YN" />
        </Form.Item>
      </Form>
    </Modal>
  );
}

export default CodeGroupFormModal;
```

코드 등록·수정 모달입니다. 이름을 언어별로 입력받습니다. 폼 필드 이름에 `["names", "ko"]`처럼 **배열**을 쓰면 중첩 객체 `{ names: { ko } }`로 모입니다.

📄 **`src/features/common-codes/components/CodeFormModal.tsx`** (새 파일)

```tsx
import { useTranslation } from "react-i18next";
import { App, Form, Input, InputNumber, Modal } from "antd";
import type { CodeInput } from "@/features/common-codes/api/codeAdminApi";
import { useSaveCode } from "@/features/common-codes/api/queries";
import { showApiError } from "@/shared/api/showApiError";
import CodeRadio from "@/shared/codes/CodeRadio";
import type { Code } from "@/shared/codes/types";
import { applyServerErrors } from "@/shared/ui/applyServerErrors";

interface CodeFormModalProps {
  groupCode: string;
  code?: Code;
  nextSortOrder: number; // 새 코드의 기본 정렬 순서
  onClose: () => void;
}

function CodeFormModal({ groupCode, code, nextSortOrder, onClose }: CodeFormModalProps) {
  const { t } = useTranslation("codes");
  const { message } = App.useApp();
  const [form] = Form.useForm<CodeInput>();
  const saveCode = useSaveCode();
  const required = (field: string) => ({
    required: true,
    message: t("common:validation.required", { field }),
  });

  const handleFinish = (values: CodeInput) => {
    saveCode.mutate(
      { id: code?.id, input: { ...values, groupCode } },
      {
        onSuccess: () => {
          message.success(t("common:message.saved"));
          onClose();
        },
        onError: (error) => {
          if (!applyServerErrors(form, error)) showApiError(error);
        },
      },
    );
  };

  return (
    <Modal
      open
      title={code ? t("code.editTitle") : t("code.createTitle")}
      okText={t("common:button.save")}
      cancelText={t("common:button.cancel")}
      onOk={form.submit}
      onCancel={onClose}
      confirmLoading={saveCode.isPending}
      mask={{ closable: false }}
    >
      <Form
        form={form}
        name="codeForm"
        layout="vertical"
        initialValues={
          code ?? { sortOrder: nextSortOrder, useYn: "Y", remark: "", names: { ko: "", en: "" } }
        }
        onFinish={handleFinish}
      >
        <Form.Item
          name="code"
          label={t("code.code")}
          rules={[required(t("code.code")), { pattern: /^[A-Z0-9_]+$/, message: t("codeRule") }]}
        >
          <Input disabled={!!code} />
        </Form.Item>
        <Form.Item
          name={["names", "ko"]}
          label={t("code.nameKo")}
          rules={[required(t("code.nameKo"))]}
        >
          <Input />
        </Form.Item>
        <Form.Item name={["names", "en"]} label={t("code.nameEn")}>
          <Input />
        </Form.Item>
        <Form.Item name="sortOrder" label={t("code.sortOrder")}>
          <InputNumber min={0} />
        </Form.Item>
        <Form.Item name="remark" label={t("code.remark")}>
          <Input.TextArea rows={2} />
        </Form.Item>
        <Form.Item name="useYn" label={t("common:label.use")}>
          <CodeRadio group="USE_YN" />
        </Form.Item>
      </Form>
    </Modal>
  );
}

export default CodeFormModal;
```

화면입니다. 메뉴 데이터에 `common-codes/CommonCodesPage`가 이미 등록되어 있으니, **파일만 만들면** "준비 중"이 화면으로 바뀝니다.

📄 **`src/features/common-codes/pages/CommonCodesPage.tsx`** (새 파일)

```tsx
import { useMemo, useState } from "react";
import { useTranslation } from "react-i18next";
import { App, Button, Card, Empty, Flex, Space } from "antd";
import type { ColDef } from "ag-grid-community";
import {
  useCodeGroups,
  useCodesOfGroup,
  useDeleteCode,
  useDeleteCodeGroup,
} from "@/features/common-codes/api/queries";
import CodeFormModal from "@/features/common-codes/components/CodeFormModal";
import CodeGroupFormModal from "@/features/common-codes/components/CodeGroupFormModal";
import Can from "@/shared/auth/Can";
import { usePermission } from "@/shared/auth/usePermission";
import type { Code, CodeGroup } from "@/shared/codes/types";
import { useCodeName } from "@/shared/codes/useCodes";
import { useLocalizedText } from "@/shared/i18n/useLocalizedText";
import DataGrid from "@/shared/ui/DataGrid";
import PageContainer from "@/shared/ui/PageContainer";

function CommonCodesPage() {
  const { t } = useTranslation("codes");
  const { message, modal } = App.useApp();
  const { can } = usePermission();
  const localize = useLocalizedText();
  const formatUseYn = useCodeName("USE_YN");

  // 마스터: 선택한 그룹 / 디테일: 그 그룹의 코드
  const [selectedGroup, setSelectedGroup] = useState<CodeGroup>();
  const groups = useCodeGroups();
  const codes = useCodesOfGroup(selectedGroup?.groupCode);
  const [selectedCode, setSelectedCode] = useState<Code>();
  const deleteGroup = useDeleteCodeGroup();
  const deleteCode = useDeleteCode();

  // 모달 상태. null이면 닫힘
  const [editingGroup, setEditingGroup] = useState<{ group?: CodeGroup } | null>(null);
  const [editingCode, setEditingCode] = useState<{ code?: Code } | null>(null);

  const groupColumns = useMemo<ColDef<CodeGroup>[]>(
    () => [
      { headerName: t("group.groupCode"), field: "groupCode" },
      { headerName: t("group.groupName"), field: "groupName" },
      {
        headerName: t("common:label.use"),
        field: "useYn",
        width: 90,
        flex: 0,
        valueFormatter: ({ value }) => formatUseYn(value),
      },
    ],
    [t, formatUseYn],
  );

  const codeColumns = useMemo<ColDef<Code>[]>(
    () => [
      { headerName: t("code.code"), field: "code" },
      { headerName: t("code.nameKo"), valueGetter: ({ data }) => data && localize(data.names) },
      { headerName: t("code.sortOrder"), field: "sortOrder", width: 80, flex: 0 },
      {
        headerName: t("common:label.use"),
        field: "useYn",
        width: 90,
        flex: 0,
        valueFormatter: ({ value }) => formatUseYn(value),
      },
      // tooltip: true — 셀에 보이는 값을 마우스를 올렸을 때 툴팁으로도 보여 준다 (긴 비고용)
      { headerName: t("code.remark"), field: "remark", flex: 2, tooltip: true },
    ],
    [t, localize, formatUseYn],
  );

  const confirmDelete = (onOk: () => Promise<unknown>) =>
    modal.confirm({
      title: t("common:message.confirmDelete", { count: 1 }),
      okButtonProps: { danger: true },
      onOk: () =>
        onOk()
          .then(() => message.success(t("common:message.deleted")))
          .catch(() => undefined), // 실패 알림은 전역 처리기가 띄운다. 확인 창은 닫는다.
    });

  return (
    <PageContainer>
      <Flex gap={12} style={{ flex: 1, minHeight: 0 }}>
        {/* 마스터: 코드 그룹 */}
        <Card
          size="small"
          title={t("group.title")}
          style={{ width: 420, display: "flex", flexDirection: "column" }}
          styles={{ body: { flex: 1, display: "flex", flexDirection: "column", minHeight: 0 } }}
          extra={
            <Space size={4}>
              <Can action="CREATE">
                <Button size="small" onClick={() => setEditingGroup({})}>
                  {t("common:button.add")}
                </Button>
              </Can>
              <Can action="DELETE">
                <Button
                  size="small"
                  danger
                  disabled={!selectedGroup}
                  onClick={() =>
                    selectedGroup &&
                    confirmDelete(() =>
                      deleteGroup
                        .mutateAsync(selectedGroup.id)
                        .then(() => setSelectedGroup(undefined)),
                    )
                  }
                >
                  {t("common:button.delete")}
                </Button>
              </Can>
            </Space>
          }
        >
          <DataGrid<CodeGroup>
            rowData={groups.data}
            columnDefs={groupColumns}
            loading={groups.isFetching}
            getRowId={({ data }) => String(data.id)}
            rowSelection={{ mode: "singleRow", checkboxes: false, enableClickSelection: true }}
            onSelectionChanged={({ api }) => {
              setSelectedGroup(api.getSelectedRows()[0]);
              setSelectedCode(undefined);
            }}
            onRowDoubleClicked={({ data }) =>
              data && can("UPDATE") && setEditingGroup({ group: data })
            }
          />
        </Card>

        {/* 디테일: 선택한 그룹의 코드 */}
        <Card
          size="small"
          title={
            selectedGroup ? `${t("code.title")} - ${selectedGroup.groupName}` : t("code.title")
          }
          style={{ flex: 1, display: "flex", flexDirection: "column", minWidth: 0 }}
          styles={{ body: { flex: 1, display: "flex", flexDirection: "column", minHeight: 0 } }}
          extra={
            selectedGroup && (
              <Space size={4}>
                <Can action="CREATE">
                  <Button size="small" onClick={() => setEditingCode({})}>
                    {t("common:button.add")}
                  </Button>
                </Can>
                <Can action="DELETE">
                  <Button
                    size="small"
                    danger
                    disabled={!selectedCode}
                    onClick={() =>
                      selectedCode && confirmDelete(() => deleteCode.mutateAsync(selectedCode.id))
                    }
                  >
                    {t("common:button.delete")}
                  </Button>
                </Can>
              </Space>
            )
          }
        >
          {selectedGroup ? (
            <DataGrid<Code>
              rowData={codes.data}
              columnDefs={codeColumns}
              loading={codes.isFetching}
              getRowId={({ data }) => String(data.id)}
              rowSelection={{ mode: "singleRow", checkboxes: false, enableClickSelection: true }}
              onSelectionChanged={({ api }) => setSelectedCode(api.getSelectedRows()[0])}
              onRowDoubleClicked={({ data }) =>
                data && can("UPDATE") && setEditingCode({ code: data })
              }
            />
          ) : (
            <Empty description={t("code.selectGroup")} style={{ margin: "auto" }} />
          )}
        </Card>
      </Flex>

      {editingGroup && (
        <CodeGroupFormModal group={editingGroup.group} onClose={() => setEditingGroup(null)} />
      )}
      {editingCode && selectedGroup && (
        <CodeFormModal
          groupCode={selectedGroup.groupCode}
          code={editingCode.code}
          nextSortOrder={(codes.data?.length ?? 0) + 1}
          onClose={() => setEditingCode(null)}
        />
      )}
    </PageContainer>
  );
}

export default CommonCodesPage;
```

### 코드 읽어 보기

**`enabled: !!groupCode`**: 그룹을 고르기 전에는 디테일 조회를 하지 않습니다. 선택이 바뀌면 queryKey가 바뀌어 자동으로 조회합니다.

**`rowSelection={{ mode: "singleRow", checkboxes: false, enableClickSelection: true }}`**: 체크박스 없이 **행을 클릭하면 선택**됩니다. 마스터 목록에 어울리는 설정입니다.

**`valueGetter` vs `valueFormatter`**: `field`로 꺼낼 수 없는 값(여기서는 현재 언어의 이름)은 `valueGetter`로 **값 자체를 만듭니다**.
`valueFormatter`는 값은 그대로 두고 **보이는 모양만** 바꿉니다. 정렬·필터는 값 기준으로 동작하므로 둘을 구별해서 씁니다.

**`useInvalidateCodes`**: 코드를 저장하면 관리 화면 캐시뿐 아니라 **다른 화면의 선택지 캐시(`["codes"]`)도** 무효화합니다.
그래서 사용자 관리 탭이 열려 있어도 사용 여부 선택지가 바로 바뀝니다.

### 확인하기

- [ ] 시스템 관리 → 공통코드 관리가 "준비 중" 대신 화면으로 열립니다. **메뉴 데이터는 손대지 않았습니다.**
- [ ] 그룹을 클릭하면 오른쪽에 코드가 나옵니다. 다른 그룹을 누르면 바뀝니다.
- [ ] `USE_YN` 그룹 → `Y` 코드 더블클릭 → 이름(한국어)을 "사용중"으로 저장 → 사용자 관리 탭으로 가면 그리드에 "사용중"
- [ ] 코드가 있는 그룹을 삭제하면 "코드가 있는 그룹은 삭제할 수 없습니다."
- [ ] `manager`로 로그인하면 공통코드 관리에 모든 버튼이 보입니다. (`CODE_MGMT` 권한 전체)

✅ **커밋**

```powershell
git add .
git commit -m "feat: 공통코드 관리 화면 추가"
```

---

## Step 9-7. 서버 페이징과 정렬

> 🎯 **할 일**: 사용자 목록을 서버에서 페이지 단위로 받고, 컬럼 정렬도 서버에 맡깁니다.

### 왜 필요한가요?

지금은 사용자 **전체**를 받아서 그리드가 정렬합니다. 데이터가 10만 건이면 브라우저가 멈춥니다.
SI 업무 데이터는 대부분 크기 때문에, **서버가 한 페이지만** 잘라서 주는 게 정석입니다.

```
GET /api/users?name=김&page=0&size=20&sort=name,asc
→ { content: [...20건], page: { number: 0, size: 20, totalElements: 1234, totalPages: 62 } }
```

Part 3에서 미리 정해 둔 `PageResponse<T>`(Spring 형식) 그대로입니다. `page`는 **0부터** 셉니다. 화면에 보여 줄 때만 +1 합니다.

> 💡 **팁** 정렬을 서버에 맡기면 그리드는 정렬하지 말아야 합니다. 그리드가 받은 20건을 또 정렬하면 "이 페이지 안에서만" 정렬된 엉뚱한 결과가 됩니다.
> 그래서 그리드의 비교 함수를 "항상 같다"(`() => 0`)로 바꿔 순서를 건드리지 않게 하고, 정렬 아이콘 클릭만 받아서 서버에 전달합니다.

### 따라 하기

가짜 서버에 페이징 도우미를 만듭니다. 다른 목록 API도 함께 씁니다.

✏️ **`src/mocks/utils.ts`** — 파일 끝에 추가합니다.

```ts
/** Problem Details 형식의 오류 응답을 만든다. */
export function problem(status: number, detail: string, extra: object = {}) {
  return HttpResponse.json({ status, detail, ...extra }, { status });
}
```

바꿀 코드

```ts
/** Problem Details 형식의 오류 응답을 만든다. */
export function problem(status: number, detail: string, extra: object = {}) {
  return HttpResponse.json({ status, detail, ...extra }, { status });
}

/** 목록을 정렬하고 한 페이지만 잘라 PageResponse 형식으로 돌려준다. sort 예: "name,asc" */
export function paginate<T extends object>(rows: T[], params: URLSearchParams) {
  const page = Number(params.get("page") ?? 0);
  const size = Number(params.get("size") ?? 20);
  const [field, direction] = (params.get("sort") ?? "").split(",");
  const sorted = field
    ? [...rows].sort((a, b) => {
        const x = String(a[field as keyof T] ?? "");
        const y = String(b[field as keyof T] ?? "");
        return direction === "desc" ? y.localeCompare(x) : x.localeCompare(y);
      })
    : rows;
  return {
    content: sorted.slice(page * size, page * size + size),
    page: {
      size,
      number: page,
      totalElements: rows.length,
      totalPages: Math.ceil(rows.length / size),
    },
  };
}
```

✏️ **`src/mocks/handlers/users.ts`** — 목록을 페이지로 돌려줍니다.

```ts
    return HttpResponse.json(rows);
  }),

  http.post(apiUrl("/users"), async ({ request }) => {
```

바꿀 코드

```ts
    return HttpResponse.json(paginate(rows, params));
  }),

  http.post(apiUrl("/users"), async ({ request }) => {
```

✏️ **`src/mocks/handlers/users.ts`** — import를 고칩니다.

```ts
import { apiUrl, problem } from "@/mocks/utils";
```

바꿀 코드

```ts
import { apiUrl, paginate, problem } from "@/mocks/utils";
```

앱 쪽입니다. 페이지 조건 타입을 공통으로 둡니다.

✏️ **`src/shared/api/types.ts`** — 파일 끝에 추가합니다.

```ts
/** 페이징 응답 (Spring Data PagedModel 형식). page.number는 0부터 시작한다. */
export interface PageResponse<T> {
  content: T[];
  page: { size: number; number: number; totalElements: number; totalPages: number };
}
```

바꿀 코드

```ts
/** 페이징 응답 (Spring Data PagedModel 형식). page.number는 0부터 시작한다. */
export interface PageResponse<T> {
  content: T[];
  page: { size: number; number: number; totalElements: number; totalPages: number };
}

/** 페이징 요청 조건. sort 예: "name,asc" (빈 문자열이면 서버 기본 정렬) */
export interface PageRequest {
  page: number;
  size: number;
  sort: string;
}
```

✏️ **`src/features/users/api/usersApi.ts`** — 목록이 페이지를 돌려줍니다.

```ts
  list: (condition: UserSearchCondition, signal?: AbortSignal) =>
    httpClient.get<User[]>("/users", { params: condition, signal }),
```

바꿀 코드

```ts
  list: (condition: UserSearchCondition & PageRequest, signal?: AbortSignal) =>
    httpClient.get<PageResponse<User>>("/users", { params: condition, signal }),
```

✏️ **`src/features/users/api/usersApi.ts`** — import를 추가합니다.

```ts
import { httpClient } from "@/shared/api/httpClient";
```

바꿀 코드

```ts
import { httpClient } from "@/shared/api/httpClient";
import type { PageRequest, PageResponse } from "@/shared/api/types";
```

✏️ **`src/features/users/api/queries.ts`** — 목록 훅을 페이지 조회로 바꿉니다.

```ts
export function useUserList(condition: UserSearchCondition) {
  return useQuery({
    queryKey: userKeys.list(condition),
    // signal: 화면을 떠나면 진행 중인 요청을 취소한다.
    queryFn: ({ signal }) => usersApi.list(condition, signal),
  });
}
```

바꿀 코드

```ts
export function useUserPage(condition: UserSearchCondition & PageRequest) {
  return useQuery({
    queryKey: userKeys.list(condition),
    // signal: 화면을 떠나면 진행 중인 요청을 취소한다.
    queryFn: ({ signal }) => usersApi.list(condition, signal),
    // 다음 페이지를 받는 동안 이전 페이지를 그대로 보여 준다. 그리드가 비었다 찼다 깜빡이지 않는다.
    placeholderData: keepPreviousData,
  });
}
```

✏️ **`src/features/users/api/queries.ts`** — import를 고칩니다.

```ts
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
```

바꿀 코드

```ts
import { keepPreviousData, useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
```

✏️ **`src/features/users/api/queries.ts`** — import를 추가합니다.

```ts
import type { UserInput, UserSearchCondition } from "@/features/users/types";
```

바꿀 코드

```ts
import type { UserInput, UserSearchCondition } from "@/features/users/types";
import type { PageRequest } from "@/shared/api/types";
```

✏️ **`src/features/users/api/queries.ts`** — 캐시 키의 조건 타입을 넓힙니다.

```ts
  list: (condition: UserSearchCondition) => [...userKeys.all, "list", condition] as const,
```

바꿀 코드

```ts
  list: (condition: UserSearchCondition & PageRequest) =>
    [...userKeys.all, "list", condition] as const,
```

화면입니다. 페이지 상태를 두고, 그리드 아래에 antd `Pagination`을 붙입니다.

✏️ **`src/features/users/pages/UsersPage.tsx`** — 서버 정렬용 컬럼 기본값을 컴포넌트 **밖**에 상수로 둡니다.

```tsx
function UsersPage() {
```

바꿀 코드

```tsx
// 정렬은 서버가 한다. 그리드는 받은 순서를 건드리지 않고(항상 "같다"고 비교), 정렬 클릭만 서버 조건으로 바꾼다.
// 컴포넌트 밖 상수로 둔다. 안에서 {{ ... }}로 쓰면 그릴 때마다 새 객체가 되어 그리드가 설정을 다시 적용한다.
const serverSortColDef: ColDef<User> = { comparator: () => 0 };

function UsersPage() {
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — 조회 상태를 바꿉니다.

```tsx
  const [condition, setCondition] = useState<UserSearchCondition>({});
  const { data, isFetching, refetch } = useUserList(condition);
```

바꿀 코드

```tsx
  const [condition, setCondition] = useState<UserSearchCondition>({});
  const [paging, setPaging] = useState<PageRequest>({ page: 0, size: 20, sort: "" });
  const { data, isFetching, refetch } = useUserPage({ ...condition, ...paging });

  // 조건이 바뀌면 첫 페이지부터 다시 본다.
  const handleSearch = (values: UserSearchCondition) => {
    setCondition(values);
    setPaging((prev) => ({ ...prev, page: 0 }));
  };
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — 조회 영역이 새 함수를 쓰게 합니다.

```tsx
      <SearchForm<UserSearchCondition> onSearch={setCondition} onSameSearch={refetch}>
```

바꿀 코드

```tsx
      <SearchForm<UserSearchCondition> onSearch={handleSearch} onSameSearch={refetch}>
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — 그리드를 페이지 데이터와 서버 정렬로 바꾸고, 페이지 이동을 붙입니다.

```tsx
      <DataGrid<User>
        rowData={data}
        columnDefs={columnDefs}
        loading={isFetching}
```

바꿀 코드

```tsx
      <DataGrid<User>
        rowData={data?.content}
        columnDefs={columnDefs}
        loading={isFetching}
        defaultColDef={serverSortColDef}
        onSortChanged={({ api }) => {
          const sorted = api.getColumnState().find((column) => column.sort);
          const sort = sorted ? `${sorted.colId},${sorted.sort}` : "";
          setPaging((prev) => ({ ...prev, page: 0, sort }));
        }}
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — 그리드 아래에 페이지 이동을 넣습니다.

```tsx
      {/* 열 때마다 새로 만든다. 이전에 입력하던 값이 다음에 열 때 남아 있는 버그를 원천 차단한다. */}
```

바꿀 코드

```tsx
      <Pagination
        align="end"
        size="small"
        current={paging.page + 1} // 서버는 0부터, 화면은 1부터
        pageSize={paging.size}
        total={data?.page.totalElements ?? 0}
        showSizeChanger
        pageSizeOptions={[20, 50, 100]}
        showTotal={(total) => t("common:label.total", { count: total })}
        onChange={(page, size) => setPaging((prev) => ({ ...prev, page: page - 1, size }))}
      />
      {/* 열 때마다 새로 만든다. 이전에 입력하던 값이 다음에 열 때 남아 있는 버그를 원천 차단한다. */}
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — import를 고칩니다.

```tsx
import { App, Button, Form, Input, Select, Space } from "antd";
import type { ColDef } from "ag-grid-community";
import { useDeleteUsers, useUserList } from "@/features/users/api/queries";
```

바꿀 코드

```tsx
import { App, Button, Form, Input, Pagination, Select, Space } from "antd";
import type { ColDef } from "ag-grid-community";
import { useDeleteUsers, useUserPage } from "@/features/users/api/queries";
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — import를 추가합니다.

```tsx
import type { User, UserSearchCondition } from "@/features/users/types";
```

바꿀 코드

```tsx
import type { User, UserSearchCondition } from "@/features/users/types";
import type { PageRequest } from "@/shared/api/types";
```

전체 건수 문구를 추가합니다.

✏️ **`src/shared/i18n/locales/ko/common.json`** — 문구를 추가합니다.

```json
    "all": "전체",
```

바꿀 코드

```json
    "all": "전체",
    "total": "총 {{count}}건",
```

✏️ **`src/shared/i18n/locales/en/common.json`** — 문구를 추가합니다.

```json
    "all": "All",
```

바꿀 코드

```json
    "all": "All",
    "total": "Total {{count}}",
```

### 확인하기

- [ ] 그리드 아래에 "총 6건"과 페이지 이동이 보입니다.
- [ ] 페이지 크기를 20 → 아래처럼 바꿔 시험해 봅니다. 사용자를 몇 명 더 등록하거나, `paging`의 기본 `size`를 잠시 `2`로 바꾸면 페이지가 여러 개가 됩니다.
- [ ] 이름 컬럼 제목을 클릭 → Network 탭 요청이 `sort=name,asc` → 다시 클릭하면 `desc`
- [ ] 페이지를 넘길 때 그리드가 비었다가 차지 않고, 이전 페이지가 보이다가 바뀝니다. (`keepPreviousData`)

✅ **커밋**

```powershell
git add .
git commit -m "feat: 사용자 목록 서버 페이징 및 정렬 적용"
```

---

## 🏁 Part 9를 마치며

마일스톤입니다. 등록·수정·삭제가 되는 진짜 같은 관리 화면이 생겼습니다.
**`UsersPage`가 앞으로 새 목록 화면을 만들 때 복사할 표준 예제**입니다.

| 패턴                  | 쓴 곳                                                    |
| --------------------- | -------------------------------------------------------- |
| 공통 조회 영역        | `SearchForm`                                             |
| 등록·수정 모달        | `{editing && <XxxFormModal />}` + `useSaveXxx`           |
| 삭제 확인             | `modal.confirm` + `mutateAsync`                          |
| 서버 검증 오류        | `applyServerErrors(form, error)`                         |
| 공통코드              | `CodeSelect`, `CodeRadio`, `useCodeName`                 |
| 마스터-디테일         | 선택 state → 디테일 queryKey                             |
| 서버 페이징·정렬      | `PageRequest`, `keepPreviousData`, `comparator: () => 0` |

| 증상                                  | 원인과 해결                                                           |
| ------------------------------------- | --------------------------------------------------------------------- |
| 등록 모달을 열면 이전 입력값이 남아 있다 | 모달을 항상 그려 두었다 → 열 때 만들고 닫을 때 없앤다                 |
| 저장했는데 목록이 그대로다            | `invalidateQueries` 누락, 또는 queryKey 앞부분이 다르다               |
| 다시 조회하면 선택이 풀린다           | `getRowId` 누락                                                       |
| 서버 정렬인데 이상하게 섞인다         | 그리드도 정렬하고 있다 → `comparator: () => 0`                        |
| 정렬·컬럼 폭이 자꾸 초기화된다        | 그리드 옵션 객체를 렌더 중에 `{{ }}`로 새로 만든다 → 밖의 상수나 `useMemo` |
| 라벨을 누르면 다른 입력칸에 커서가 간다 | 폼끼리 입력칸 id가 겹친다 → `Form`에 `name`을 준다                    |

다음은 관리자가 화면에서 메뉴와 권한을 직접 바꾸는 기능입니다. → [Part 10. 메뉴·권한 관리](part-10-admin-screens.md)
