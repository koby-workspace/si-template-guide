# Part 10. 메뉴·권한 관리

> **이 Part의 목표**: 관리자가 **화면에서** 메뉴를 추가하고 소스 파일을 매핑하며, 역할별로 메뉴·버튼 권한을 바꿀 수 있게 합니다.
> 저장하면 새로고침 없이 사이드바·탭·버튼이 바로 바뀝니다. 메뉴 깊이는 최대 3단계로 제한합니다.

---

## Step 10-1. 화면 레지스트리를 shared로 옮기기

> 🎯 **할 일**: 메뉴 관리 화면도 화면 파일 목록을 쓸 수 있도록 `pageRegistry`를 `shared`로 옮깁니다.

### 왜 옮기나요?

메뉴 관리 화면(`features/menu-admin`)은 "화면 파일" 선택지를 만들려고 `pageKeys`가 필요합니다. 그런데 `pageRegistry.ts`는 `app/`에 있습니다.
의존 방향 규칙(Part 2)은 `app → features → shared`입니다. **feature가 app을 import 하면 규칙 위반**입니다.

이럴 때의 답도 Part 2에 적어 두었습니다. **여러 곳이 쓰는 것은 shared로 내린다.**

> 💡 **팁** `import.meta.glob`의 패턴이 `features` 폴더를 가리키지만 이것은 import 문이 아니라 **빌드할 때 만드는 파일 목록**입니다.
> `shared`가 특정 기능을 아는 것이 아니므로 규칙을 어기지 않습니다.

### 따라 하기

▶️ **터미널**

```powershell
Move-Item src/app/pageRegistry.ts src/shared/page/pageRegistry.ts
```

✏️ **`src/app/layout/PageRenderer.tsx`** — import 경로를 바꿉니다.

```tsx
import { getPage } from "@/app/pageRegistry";
```

바꿀 코드

```tsx
import { getPage } from "@/shared/page/pageRegistry";
```

> ⌨️ **단축키** 파일을 옮긴 뒤 다른 곳에서 옛 경로를 쓰는지 찾으려면 `Ctrl+Shift+F`로 `app/pageRegistry`를 검색합니다.
> 탐색기에서 드래그로 옮겼다면 VS Code가 import를 자동으로 고쳐 줍니다.

✅ **커밋**

```powershell
git add .
git commit -m "refactor: 화면 레지스트리를 shared로 이동"
```

---

## Step 10-2. 메뉴 관리 API

> 🎯 **할 일**: 메뉴를 등록·수정·삭제하는 가짜 API를 만들고, 서버 쪽 검증(중복, 3단계 제한 등)을 넣습니다.

### 원리: 깊이 계산

3단계 제한을 지키려면 두 가지를 계산할 수 있어야 합니다.

```
깊이(depth): 위로 몇 단계인가        샘플(1) > 화면 패턴(2) > 그리드 편집(3)
높이(height): 아래로 몇 단계인가     "화면 패턴"의 높이는 2 (자기 + 그리드 편집)
```

메뉴를 어떤 폴더 아래로 옮길 때 `상위 폴더의 깊이 + 내 높이 ≤ 3`이어야 합니다. 하위 메뉴까지 함께 옮겨지기 때문입니다.

### 따라 하기

깊이 계산 함수를 메뉴 도우미 파일에 추가합니다. 화면과 가짜 서버가 **같은 함수**를 씁니다.

✏️ **`src/shared/menu/tree.ts`** — 파일 끝에 함수를 추가합니다.

```ts
/** 메뉴의 조상을 위에서부터 차례로 돌려준다. 자기 자신을 포함한다. (브레드크럼, 폴더 펼치기용) */
```

바꿀 코드

```ts
/** 메뉴의 깊이. 최상위 메뉴가 1, 그 아래가 2... id가 null(최상위의 부모)이면 0 */
export function getDepth(menus: Menu[], id: number | null): number {
  let depth = 0;
  let current = menus.find((menu) => menu.id === id);
  while (current) {
    depth += 1;
    const parentId: number | null = current.parentId;
    current = menus.find((menu) => menu.id === parentId);
  }
  return depth;
}

/** 자기 자신을 포함한 하위 메뉴 id 전부 */
export function getDescendantIds(menus: Menu[], id: number): number[] {
  const children = menus.filter((menu) => menu.parentId === id);
  return [id, ...children.flatMap((child) => getDescendantIds(menus, child.id))];
}

/** 자기 자신을 포함한 하위 트리의 높이. 자식이 없으면 1 */
export function getSubtreeHeight(menus: Menu[], id: number): number {
  const children = menus.filter((menu) => menu.parentId === id);
  return 1 + Math.max(0, ...children.map((child) => getSubtreeHeight(menus, child.id)));
}

/** 메뉴의 조상을 위에서부터 차례로 돌려준다. 자기 자신을 포함한다. (브레드크럼, 폴더 펼치기용) */
```

등록·수정에 보내는 타입을 추가합니다. `Menu`가 유니온이라 `Omit<Menu, "id">`를 쓰면 유형별 필드(`path` 등)가 사라집니다.
그래서 유형마다 `Omit`을 하고 다시 유니온으로 묶습니다.

✏️ **`src/shared/menu/types.ts`** — 타입을 추가합니다.

```ts
export type Menu = FolderMenu | PageMenu | LinkMenu;
```

바꿀 코드

```ts
export type Menu = FolderMenu | PageMenu | LinkMenu;

/** 등록·수정할 때 보내는 값. 유형별 필드가 사라지지 않도록 유형마다 Omit 한다. */
export type MenuInput = Omit<FolderMenu, "id"> | Omit<PageMenu, "id"> | Omit<LinkMenu, "id">;
```

가짜 API입니다. 메뉴 관리와 권한 관리 두 화면이 전체 메뉴를 보므로 둘 중 하나의 조회 권한만 있으면 됩니다.

📄 **`src/mocks/handlers/menus.ts`** (전체 교체)

```ts
import { http, HttpResponse } from "msw";
import { actionsOf, authorize } from "@/mocks/auth";
import { menusTable, permissionsTable } from "@/mocks/db";
import { apiUrl, problem } from "@/mocks/utils";
import { getDepth, getDescendantIds, getSubtreeHeight, MAX_MENU_DEPTH } from "@/shared/menu/tree";
import { ACTIONS, type MenuInput } from "@/shared/menu/types";

const MENU = "MENU_MGMT";

/** 서버 검증. 화면에서 막아도 서버는 반드시 다시 확인한다. */
function validate(input: MenuInput, id?: number) {
  const menus = menusTable.all();
  const others = menus.filter((menu) => menu.id !== id);
  const errors: { field: string; message: string }[] = [];
  const parent = menus.find((menu) => menu.id === input.parentId);

  if (!id && others.some((menu) => menu.code === input.code)) {
    errors.push({ field: "code", message: "이미 있는 메뉴 코드입니다." });
  }
  if (
    input.type === "PAGE" &&
    others.some((menu) => menu.type === "PAGE" && menu.path === input.path)
  ) {
    errors.push({ field: "path", message: "이미 사용 중인 주소입니다." });
  }
  if (input.parentId !== null && parent?.type !== "FOLDER") {
    errors.push({ field: "parentId", message: "상위 메뉴는 폴더여야 합니다." });
  } else if (
    id &&
    input.parentId !== null &&
    getDescendantIds(menus, id).includes(input.parentId)
  ) {
    errors.push({ field: "parentId", message: "자기 자신이나 하위 메뉴 아래로 옮길 수 없습니다." });
  } else {
    // 깊이 제한: 상위의 깊이 + 내 하위 트리 높이 ≤ 최대 깊이
    const height = id ? getSubtreeHeight(menus, id) : 1;
    const depth = getDepth(menus, input.parentId);
    if (depth + height > MAX_MENU_DEPTH) {
      errors.push({
        field: "parentId",
        message: `메뉴는 최대 ${MAX_MENU_DEPTH}단계까지 만들 수 있습니다.`,
      });
    } else if (input.type === "FOLDER" && depth + 1 >= MAX_MENU_DEPTH) {
      errors.push({ field: "type", message: `${MAX_MENU_DEPTH}단계에는 폴더를 만들 수 없습니다.` });
    }
  }
  return errors.length > 0 ? problem(400, errors[0].message, { errors }) : null;
}

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

  // ── 메뉴 관리 ──
  // 전체 메뉴(사용 안 함 포함). 메뉴 관리·권한 관리 중 하나라도 조회 권한이 있으면 볼 수 있다.
  http.get(apiUrl("/menus"), ({ request }) => {
    const { user, error } = authorize(request);
    if (error) return error;
    const canRead = ["MENU_MGMT", "ROLE_MGMT"].some((code) =>
      actionsOf(user, code).includes("READ"),
    );
    if (!canRead) return problem(403, "권한이 없습니다.");
    return HttpResponse.json(menusTable.all());
  }),

  http.post(apiUrl("/menus"), async ({ request }) => {
    const { error } = authorize(request, MENU, "CREATE");
    if (error) return error;
    const input = (await request.json()) as MenuInput;
    const invalid = validate(input);
    if (invalid) return invalid;
    const menu = menusTable.insert(input);
    // 편의 기능: 새 메뉴는 관리자(ADMIN) 역할에 모든 권한을 자동으로 준다. 다른 역할은 권한 관리에서 준다.
    if (menu.type !== "FOLDER") {
      permissionsTable.insert({ roleCode: "ADMIN", menuCode: menu.code, actions: ACTIONS });
    }
    return HttpResponse.json(menu, { status: 201 });
  }),

  http.put(apiUrl("/menus/:id"), async ({ request, params }) => {
    const { error } = authorize(request, MENU, "UPDATE");
    if (error) return error;
    const id = Number(params.id);
    const current = menusTable.find(id);
    if (!current) return problem(404, "메뉴를 찾을 수 없습니다.");
    const input = (await request.json()) as MenuInput;
    // 코드와 유형은 바꿀 수 없다. 코드는 권한·탭이 참조하는 키이고, 유형이 바뀌면 필드 구성이 달라진다.
    if (input.type !== current.type) return problem(400, "메뉴 유형은 바꿀 수 없습니다.");
    const invalid = validate({ ...input, code: current.code }, id);
    if (invalid) return invalid;
    return HttpResponse.json(menusTable.update(id, { ...input, code: current.code }));
  }),

  http.delete(apiUrl("/menus/:id"), ({ request, params }) => {
    const { error } = authorize(request, MENU, "DELETE");
    if (error) return error;
    const id = Number(params.id);
    if (menusTable.all().some((menu) => menu.parentId === id)) {
      return problem(
        400,
        "하위 메뉴가 있는 메뉴는 삭제할 수 없습니다. 하위 메뉴를 먼저 삭제하세요.",
      );
    }
    const menu = menusTable.find(id);
    menusTable.remove(id);
    // 그 메뉴의 권한 데이터도 함께 지운다
    permissionsTable
      .all()
      .filter((permission) => permission.menuCode === menu?.code)
      .forEach((permission) => permissionsTable.remove(permission.id));
    return new HttpResponse(null, { status: 204 });
  }),
];
```

앱 쪽입니다. 전체 메뉴 조회는 메뉴 관리·권한 관리 **두 기능**이 쓰므로 `shared`에 둡니다. (feature끼리는 import 하지 않는다 — 규칙 2)

📄 **`src/shared/menu/useAllMenus.ts`** (새 파일)

```ts
import { useQuery } from "@tanstack/react-query";
import { httpClient } from "@/shared/api/httpClient";
import type { Menu } from "@/shared/menu/types";

export const allMenusKey = ["menus"] as const;

/** 전체 메뉴(사용 안 함 포함). 메뉴 관리·권한 관리 화면용. */
export function useAllMenus() {
  return useQuery({
    queryKey: allMenusKey,
    queryFn: () => httpClient.get<Menu[]>("/menus"),
  });
}
```

📄 **`src/features/menu-admin/api/queries.ts`** (새 파일)

```ts
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { httpClient } from "@/shared/api/httpClient";
import type { Menu, MenuInput } from "@/shared/menu/types";
import { allMenusKey } from "@/shared/menu/useAllMenus";
import { myMenusKey } from "@/shared/menu/useMyMenus";

/** 메뉴가 바뀌면 관리 화면의 목록과 "내 메뉴"(사이드바·탭·제목)를 함께 무효화한다. → 새로고침 없이 바로 반영 */
function useInvalidateMenus() {
  const queryClient = useQueryClient();
  return () =>
    Promise.all([
      queryClient.invalidateQueries({ queryKey: allMenusKey }),
      queryClient.invalidateQueries({ queryKey: myMenusKey }),
    ]);
}

export function useSaveMenu() {
  const invalidate = useInvalidateMenus();
  return useMutation({
    mutationFn: ({ id, input }: { id?: number; input: MenuInput }) =>
      id ? httpClient.put<Menu>(`/menus/${id}`, input) : httpClient.post<Menu>("/menus", input),
    meta: { skipGlobalError: true },
    onSuccess: invalidate,
  });
}

export function useDeleteMenu() {
  const invalidate = useInvalidateMenus();
  return useMutation({
    mutationFn: (id: number) => httpClient.delete(`/menus/${id}`),
    onSuccess: invalidate,
  });
}
```

> 💡 **팁** API 함수가 짧으면 `xxxApi.ts`를 따로 두지 않고 훅 안에 바로 써도 됩니다. 파일을 나누는 기준은 "규모가 커졌을 때"입니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 메뉴 관리 API 추가"
```

---

## Step 10-3. 메뉴 관리 화면

> 🎯 **할 일**: 왼쪽 트리에서 메뉴를 고르고, 오른쪽 폼에서 수정·등록합니다. 저장하면 사이드바가 바로 바뀝니다.

### 화면 구성과 입력 항목

| 영역          | 구성                                                     |
| ------------- | -------------------------------------------------------- |
| 왼쪽 (트리)   | 메뉴 트리 (사용 안 함은 흐리게) · [최상위 추가] [하위 추가] |
| 오른쪽 (상세) | 선택한 메뉴의 입력 폼 · [삭제] [저장]                     |

유형을 바꾸면 입력칸이 바뀝니다(`Form.useWatch("type")`).

| 항목            | 폴더 | 화면 | 링크 | 비고                                                       |
| --------------- | :--: | :--: | :--: | ---------------------------------------------------------- |
| 유형            |  ✓   |  ✓   |  ✓   | 저장 후 변경 불가                                          |
| 상위 메뉴       |  ✓   |  ✓   |  ✓   | 폴더만, 3단계 제한에 걸리는 폴더는 고를 수 없음            |
| 메뉴 코드       |  ✓   |  ✓   |  ✓   | 영문 대문자·숫자·`_`, 저장 후 변경 불가                    |
| 메뉴명 (ko/en)  |  ✓   |  ✓   |  ✓   | 한국어 필수                                                |
| 주소            |      |  ✓   |      | `/`로 시작, 중복 불가                                      |
| **화면 파일**   |      |  ✓   |      | **소스 파일 매핑.** 목록에서 고르거나 직접 입력            |
| URL             |      |      |  ✓   | `http://`·`https://`                                       |
| 아이콘·정렬·표시·사용 |  ✓   |  ✓   |  ✓   |                                                            |

### "화면 파일" 매핑의 원리와 한계

"화면 파일" 칸은 `pageKeys`(빌드할 때 찾아 둔 화면 파일 목록)를 선택지로 보여 주고, **직접 입력도** 받습니다.
SI에서는 메뉴를 먼저 등록하고 화면을 나중에 만드는 일이 흔하기 때문입니다. 아직 없는 파일을 적으면 경고만 하고 저장은 허용합니다.

> ⚠️ **주의** 운영 환경에서는 **빌드할 때 있던 화면 파일만** 열 수 있습니다. 새 화면 = 새 배포입니다.
> 반대로 메뉴 **구성**(이름·순서·위치·표시 여부·권한)은 배포 없이 바꿀 수 있습니다. 이것이 일반적인 SI 메뉴 관리의 범위입니다.

### 따라 하기

📄 **`src/shared/i18n/locales/ko/menuAdmin.json`** (새 파일)

```json
{
  "tree": {
    "title": "메뉴",
    "addRoot": "최상위 추가",
    "addChild": "하위 추가",
    "maxDepth": "메뉴는 최대 {{max}}단계까지 만들 수 있습니다."
  },
  "editor": {
    "createTitle": "새 메뉴",
    "editTitle": "메뉴 수정",
    "select": "왼쪽에서 메뉴를 선택하거나 새 메뉴를 추가하세요."
  },
  "field": {
    "type": "유형",
    "parent": "상위 메뉴",
    "root": "(최상위)",
    "code": "메뉴 코드",
    "nameKo": "메뉴명(한국어)",
    "nameEn": "메뉴명(영어)",
    "path": "주소",
    "component": "화면 파일",
    "url": "URL",
    "icon": "아이콘",
    "sortOrder": "정렬",
    "visible": "사이드바 표시"
  },
  "type": {
    "FOLDER": "폴더",
    "PAGE": "화면",
    "LINK": "외부 링크"
  },
  "help": {
    "code": "영문 대문자, 숫자, _ 로 입력합니다. 저장한 뒤에는 바꿀 수 없습니다.",
    "path": "/로 시작합니다. 예: /board/faq",
    "component": "src/features/<기능>/pages/<이름>Page.tsx 파일이라면 \"<기능>/<이름>Page\"",
    "fileExists": "파일이 있습니다: {{file}}",
    "fileMissing": "아직 없는 파일입니다. 저장은 되지만 \"화면 준비 중\"으로 표시됩니다.",
    "usedBy": "다른 메뉴도 이 파일을 씁니다: {{name}}"
  },
  "rule": {
    "code": "영문 대문자, 숫자, _ 만 쓸 수 있습니다.",
    "path": "/로 시작하는 주소를 입력하세요.",
    "url": "http:// 또는 https://로 시작해야 합니다."
  }
}
```

📄 **`src/shared/i18n/locales/en/menuAdmin.json`** (새 파일)

```json
{
  "tree": {
    "title": "Menus",
    "addRoot": "Add top level",
    "addChild": "Add child",
    "maxDepth": "Menus can be nested up to {{max}} levels."
  },
  "editor": {
    "createTitle": "New menu",
    "editTitle": "Edit menu",
    "select": "Select a menu on the left or add a new one."
  },
  "field": {
    "type": "Type",
    "parent": "Parent",
    "root": "(Top level)",
    "code": "Menu code",
    "nameKo": "Name (Korean)",
    "nameEn": "Name (English)",
    "path": "Path",
    "component": "Page file",
    "url": "URL",
    "icon": "Icon",
    "sortOrder": "Order",
    "visible": "Show in sidebar"
  },
  "type": {
    "FOLDER": "Folder",
    "PAGE": "Page",
    "LINK": "External link"
  },
  "help": {
    "code": "Uppercase letters, digits, and underscores. Cannot be changed after saving.",
    "path": "Starts with /. e.g. /board/faq",
    "component": "For src/features/<feature>/pages/<Name>Page.tsx, enter \"<feature>/<Name>Page\"",
    "fileExists": "File found: {{file}}",
    "fileMissing": "File does not exist yet. You can save, but the page will show \"under construction\".",
    "usedBy": "Another menu also uses this file: {{name}}"
  },
  "rule": {
    "code": "Use uppercase letters, digits, and underscores only.",
    "path": "Enter a path starting with /.",
    "url": "Must start with http:// or https://."
  }
}
```

오른쪽 입력 폼입니다. 이 화면의 핵심이라 깁니다. 천천히 읽어 보세요.

📄 **`src/features/menu-admin/components/MenuEditor.tsx`** (새 파일)

```tsx
import { useTranslation } from "react-i18next";
import {
  App,
  AutoComplete,
  Button,
  Flex,
  Form,
  Input,
  InputNumber,
  Radio,
  Select,
  Switch,
  TreeSelect,
  Typography,
} from "antd";
import { useDeleteMenu, useSaveMenu } from "@/features/menu-admin/api/queries";
import type { UseYn } from "@/shared/api/types";
import { showApiError } from "@/shared/api/showApiError";
import { usePermission } from "@/shared/auth/usePermission";
import CodeRadio from "@/shared/codes/CodeRadio";
import type { LocalizedText } from "@/shared/i18n/locales";
import { useLocalizedText } from "@/shared/i18n/useLocalizedText";
import MenuIcon from "@/shared/menu/MenuIcon";
import { menuIcons } from "@/shared/menu/menuIcons";
import {
  buildMenuTree,
  getDepth,
  getDescendantIds,
  getSubtreeHeight,
  MAX_MENU_DEPTH,
} from "@/shared/menu/tree";
import type { Menu, MenuInput, MenuNode, MenuType } from "@/shared/menu/types";
import { pageKeys } from "@/shared/page/pageRegistry";
import { applyServerErrors } from "@/shared/ui/applyServerErrors";

const ROOT = 0; // 상위 메뉴 선택지에서 "최상위"를 나타내는 값 (서버로는 null을 보낸다)

// 폼은 유니온을 다루기 어렵다. 모든 필드를 가진 평평한 모양으로 받고, 저장할 때 유형에 맞게 바꾼다.
interface MenuFormValues {
  type: MenuType;
  parentId: number;
  code: string;
  names: LocalizedText;
  icon?: string;
  sortOrder: number;
  visible: boolean;
  useYn: UseYn;
  path?: string;
  component?: string;
  url?: string;
}

function toInput(values: MenuFormValues): MenuInput {
  const base = {
    parentId: values.parentId === ROOT ? null : values.parentId,
    code: values.code,
    names: values.names,
    icon: values.icon,
    sortOrder: values.sortOrder,
    visible: values.visible,
    useYn: values.useYn,
  };
  if (values.type === "PAGE") {
    return { ...base, type: "PAGE", path: values.path ?? "", component: values.component ?? "" };
  }
  if (values.type === "LINK") return { ...base, type: "LINK", url: values.url ?? "" };
  return { ...base, type: "FOLDER" };
}

interface MenuEditorProps {
  menus: Menu[];
  menu?: Menu; // 수정할 메뉴. 없으면 새 메뉴
  parentId?: number | null; // 새 메뉴를 만들 위치
  onSaved: (menu: Menu) => void;
  onDeleted: () => void;
}

function MenuEditor({ menus, menu, parentId = null, onSaved, onDeleted }: MenuEditorProps) {
  const { t } = useTranslation("menuAdmin");
  const { message, modal } = App.useApp();
  const { can } = usePermission();
  const localize = useLocalizedText();
  const [form] = Form.useForm<MenuFormValues>();
  const saveMenu = useSaveMenu();
  const deleteMenu = useDeleteMenu();
  const isEdit = menu !== undefined;

  const type = Form.useWatch("type", form);
  const component = Form.useWatch("component", form);

  const required = (field: string) => ({
    required: true,
    message: t("common:validation.required", { field }),
  });

  // ── 상위 메뉴 선택지: 폴더만. 3단계를 넘게 되는 폴더와 자기 자신·하위는 고를 수 없다 ──
  const height = menu ? getSubtreeHeight(menus, menu.id) : 1;
  const blockedIds = menu ? getDescendantIds(menus, menu.id) : [];
  const toTreeOption = (node: MenuNode): object => ({
    value: node.id,
    title: localize(node.names),
    disabled: blockedIds.includes(node.id) || getDepth(menus, node.id) + height > MAX_MENU_DEPTH,
    children: node.children.filter((child) => child.type === "FOLDER").map(toTreeOption),
  });
  const parentOptions = [
    {
      value: ROOT,
      title: t("field.root"),
      children: buildMenuTree(menus.filter((item) => item.type === "FOLDER")).map(toTreeOption),
    },
  ];

  // ── 화면 파일: 목록에서 고르거나 직접 입력한다 ──
  const menuUsing = (key?: string) =>
    menus.find((item) => item.type === "PAGE" && item.component === key && item.id !== menu?.id);
  const componentOptions = pageKeys.map((key) => {
    const user = menuUsing(key);
    return {
      value: key,
      label: user ? `${key}  (${localize(user.names)})` : key,
    };
  });
  const componentStatus = () => {
    if (!component) return t("help.component");
    if (!pageKeys.includes(component)) {
      return <Typography.Text type="warning">{t("help.fileMissing")}</Typography.Text>;
    }
    const file = `src/features/${component.replace("/", "/pages/")}.tsx`;
    const user = menuUsing(component);
    return (
      <Typography.Text type="success">
        {t("help.fileExists", { file })}
        {user && ` · ${t("help.usedBy", { name: localize(user.names) })}`}
      </Typography.Text>
    );
  };

  const handleFinish = (values: MenuFormValues) => {
    saveMenu.mutate(
      { id: menu?.id, input: toInput(values) },
      {
        onSuccess: (saved) => {
          message.success(t("common:message.saved"));
          onSaved(saved);
        },
        onError: (error) => {
          if (!applyServerErrors(form, error)) showApiError(error);
        },
      },
    );
  };

  const handleDelete = () => {
    if (!menu) return;
    modal.confirm({
      title: t("common:message.confirmDelete", { count: 1 }),
      okButtonProps: { danger: true },
      onOk: () =>
        deleteMenu
          .mutateAsync(menu.id)
          .then(() => {
            message.success(t("common:message.deleted"));
            onDeleted();
          })
          .catch(() => undefined),
    });
  };

  const initialValues: Partial<MenuFormValues> = menu
    ? { ...menu, parentId: menu.parentId ?? ROOT }
    : {
        type: "PAGE",
        parentId: parentId ?? ROOT,
        names: { ko: "", en: "" },
        sortOrder: menus.filter((item) => item.parentId === parentId).length + 1,
        visible: true,
        useYn: "Y",
      };

  return (
    <Form
      form={form}
      name="menuForm"
      layout="vertical"
      initialValues={initialValues}
      onFinish={handleFinish}
      style={{ maxWidth: 640 }}
    >
      <Form.Item name="type" label={t("field.type")}>
        <Radio.Group
          disabled={isEdit}
          optionType="button"
          options={(["FOLDER", "PAGE", "LINK"] as const).map((value) => ({
            value,
            label: t(`type.${value}`),
          }))}
        />
      </Form.Item>
      <Form.Item name="parentId" label={t("field.parent")}>
        <TreeSelect treeData={parentOptions} treeDefaultExpandAll />
      </Form.Item>
      <Form.Item
        name="code"
        label={t("field.code")}
        extra={t("help.code")}
        rules={[required(t("field.code")), { pattern: /^[A-Z0-9_]+$/, message: t("rule.code") }]}
      >
        <Input disabled={isEdit} />
      </Form.Item>
      <Flex gap={12}>
        <Form.Item
          name={["names", "ko"]}
          label={t("field.nameKo")}
          rules={[required(t("field.nameKo"))]}
          style={{ flex: 1 }}
        >
          <Input />
        </Form.Item>
        <Form.Item name={["names", "en"]} label={t("field.nameEn")} style={{ flex: 1 }}>
          <Input />
        </Form.Item>
      </Flex>

      {type === "PAGE" && (
        <>
          <Form.Item
            name="path"
            label={t("field.path")}
            extra={t("help.path")}
            rules={[required(t("field.path")), { pattern: /^\/\S*$/, message: t("rule.path") }]}
          >
            <Input placeholder="/board/faq" />
          </Form.Item>
          <Form.Item
            name="component"
            label={t("field.component")}
            extra={componentStatus()}
            rules={[required(t("field.component"))]}
          >
            <AutoComplete
              options={componentOptions}
              placeholder="samples/HelloPage"
              showSearch={{
                filterOption: (input, option) =>
                  String(option?.value ?? "")
                    .toLowerCase()
                    .includes(input.toLowerCase()),
              }}
            />
          </Form.Item>
        </>
      )}

      {type === "LINK" && (
        <Form.Item
          name="url"
          label={t("field.url")}
          rules={[required(t("field.url")), { pattern: /^https?:\/\//, message: t("rule.url") }]}
        >
          <Input placeholder="https://" />
        </Form.Item>
      )}

      <Flex gap={12} wrap>
        <Form.Item name="icon" label={t("field.icon")} style={{ width: 240 }}>
          <Select
            allowClear
            showSearch
            options={Object.keys(menuIcons).map((name) => ({
              value: name,
              label: (
                <Flex gap={8}>
                  <MenuIcon name={name} />
                  {name}
                </Flex>
              ),
            }))}
          />
        </Form.Item>
        <Form.Item name="sortOrder" label={t("field.sortOrder")}>
          <InputNumber min={0} />
        </Form.Item>
        <Form.Item name="visible" label={t("field.visible")} valuePropName="checked">
          <Switch />
        </Form.Item>
        <Form.Item name="useYn" label={t("common:label.use")}>
          <CodeRadio group="USE_YN" />
        </Form.Item>
      </Flex>

      <Flex gap={8} justify="end">
        {isEdit && can("DELETE") && (
          <Button danger onClick={handleDelete}>
            {t("common:button.delete")}
          </Button>
        )}
        <Button
          type="primary"
          htmlType="submit"
          loading={saveMenu.isPending}
          disabled={!can(isEdit ? "UPDATE" : "CREATE")}
        >
          {t("common:button.save")}
        </Button>
      </Flex>
    </Form>
  );
}

export default MenuEditor;
```

화면입니다.

📄 **`src/features/menu-admin/pages/MenuAdminPage.tsx`** (새 파일)

```tsx
import { useState } from "react";
import { useTranslation } from "react-i18next";
import { Button, Card, Empty, Flex, Space, Tooltip, Tree, theme } from "antd";
import type { DataNode } from "antd/es/tree";
import MenuEditor from "@/features/menu-admin/components/MenuEditor";
import Can from "@/shared/auth/Can";
import { useLocalizedText } from "@/shared/i18n/useLocalizedText";
import MenuIcon from "@/shared/menu/MenuIcon";
import { buildMenuTree, getDepth, MAX_MENU_DEPTH } from "@/shared/menu/tree";
import type { MenuNode } from "@/shared/menu/types";
import { useAllMenus } from "@/shared/menu/useAllMenus";
import PageContainer from "@/shared/ui/PageContainer";
import PageLoading from "@/shared/ui/PageLoading";

// 오른쪽 편집기 상태. id가 있으면 수정, 없으면 parentId 아래에 새 메뉴
type Selection = { id: number } | { parentId: number | null };

function MenuAdminPage() {
  const { t } = useTranslation("menuAdmin");
  const { token } = theme.useToken();
  const localize = useLocalizedText();
  const { data: menus } = useAllMenus();
  const [selection, setSelection] = useState<Selection | null>(null);

  if (!menus) return <PageLoading />;

  const selectedMenu =
    selection && "id" in selection ? menus.find((menu) => menu.id === selection.id) : undefined;
  const canAddChild =
    selectedMenu?.type === "FOLDER" && getDepth(menus, selectedMenu.id) < MAX_MENU_DEPTH;

  // 메뉴 트리 → antd Tree 형식. 사용 안 하는 메뉴는 흐리게 보여 준다.
  const toTreeNode = (node: MenuNode): DataNode => ({
    key: node.id,
    title: (
      <span style={{ color: node.useYn === "N" ? token.colorTextDisabled : undefined }}>
        <MenuIcon name={node.icon} /> {localize(node.names)}
      </span>
    ),
    children: node.children.map(toTreeNode),
  });

  // 편집기를 메뉴(또는 새 위치)마다 새로 만든다. 다른 메뉴를 고르면 폼이 그 메뉴의 값으로 깨끗하게 바뀐다.
  const editorKey = selection
    ? "id" in selection
      ? `edit-${selection.id}`
      : `new-${selection.parentId}`
    : "none";

  return (
    <PageContainer>
      <Flex gap={12} style={{ flex: 1, minHeight: 0 }}>
        <Card
          size="small"
          title={t("tree.title")}
          style={{ width: 340, display: "flex", flexDirection: "column" }}
          styles={{ body: { flex: 1, overflow: "auto" } }}
          extra={
            <Can action="CREATE">
              <Space size={4}>
                <Button size="small" onClick={() => setSelection({ parentId: null })}>
                  {t("tree.addRoot")}
                </Button>
                <Tooltip
                  title={canAddChild ? undefined : t("tree.maxDepth", { max: MAX_MENU_DEPTH })}
                >
                  <Button
                    size="small"
                    disabled={!canAddChild}
                    onClick={() => selectedMenu && setSelection({ parentId: selectedMenu.id })}
                  >
                    {t("tree.addChild")}
                  </Button>
                </Tooltip>
              </Space>
            </Can>
          }
        >
          <Tree
            blockNode
            defaultExpandAll
            treeData={buildMenuTree(menus).map(toTreeNode)}
            selectedKeys={selectedMenu ? [selectedMenu.id] : []}
            onSelect={([key]) => key !== undefined && setSelection({ id: Number(key) })}
          />
        </Card>

        <Card
          size="small"
          title={selectedMenu ? t("editor.editTitle") : selection ? t("editor.createTitle") : " "}
          style={{ flex: 1, minWidth: 0, display: "flex", flexDirection: "column" }}
          styles={{ body: { flex: 1, overflow: "auto" } }}
        >
          {selection ? (
            <MenuEditor
              key={editorKey}
              menus={menus}
              menu={selectedMenu}
              parentId={"parentId" in selection ? selection.parentId : undefined}
              onSaved={(menu) => setSelection({ id: menu.id })}
              onDeleted={() => setSelection(null)}
            />
          ) : (
            <Empty description={t("editor.select")} />
          )}
        </Card>
      </Flex>
    </PageContainer>
  );
}

export default MenuAdminPage;
```

### 코드 읽어 보기

**`Form.useWatch("type", form)`**: 폼의 특정 값이 바뀔 때마다 화면을 다시 그립니다. 유형에 따라 입력칸을 바꾸는 데 씁니다.

**평평한 폼 값 → 유니온**: 폼 라이브러리는 "유형에 따라 필드가 달라지는 타입"을 다루기 어렵습니다. 그래서 폼은 모든 필드를 가진 `MenuFormValues`로 받고,
저장할 때 `toInput()`에서 유형에 맞는 `MenuInput`으로 바꿉니다. 타입이 어긋나면 `toInput`에서 컴파일 오류가 나므로 안전합니다.

**`key={editorKey}`**: Part 5·7에서 쓴 `key` 기법입니다. 다른 메뉴를 고르면 편집기를 새로 만들어서 `initialValues`가 다시 적용됩니다.

**`type Selection = { id } | { parentId }`와 `"id" in selection`**: `in` 연산자로도 유니온을 좁힐 수 있습니다. `type` 같은 구별 필드가 없을 때 씁니다.

**`valuePropName="checked"`**: `Switch`는 값을 `value`가 아니라 `checked`로 받습니다. 폼에 그 사실을 알려 줍니다.

### 확인하기

- [ ] `admin` → 시스템 관리 → 메뉴 관리. 왼쪽에 전체 메뉴 트리가 보입니다.
- [ ] "사용자 관리"를 고르고 메뉴명(한국어)을 "사용자 관리(테스트)"로 저장 → **사이드바와 열린 탭 제목, 브레드크럼이 바로 바뀝니다.** 확인 후 되돌립니다.
- [ ] "화면 파일" 칸을 비우고 `users/`를 입력하면 목록이 걸러집니다. 없는 이름(`abc/AbcPage`)을 넣으면 주황색 경고가 뜹니다.
- [ ] 유형을 "외부 링크"로 바꾸면 주소·화면 파일 칸이 사라지고 URL 칸이 나타납니다. (새 메뉴일 때)

✅ **커밋**

```powershell
git add .
git commit -m "feat: 메뉴 관리 화면 추가"
```

---

## Step 10-4. 깊이 3단계 제한 확인

> 🎯 **할 일**: 3단계 제한이 화면·서버 양쪽에서 지켜지는지 확인합니다.

이미 Step 10-2(서버)와 10-3(화면)에 들어가 있습니다. 제한이 **세 겹**인 이유를 정리하고 직접 확인해 봅니다.

| 위치                    | 하는 일                                                      | 왜 필요한가                         |
| ----------------------- | ------------------------------------------------------------ | ----------------------------------- |
| 메뉴 관리 화면          | [하위 추가] 비활성, 상위 메뉴 선택지에서 걸리는 폴더 비활성 | 사용자가 실수하기 전에 막는다       |
| 서버 (`validate`)       | 깊이·높이 계산 후 400 응답                                   | 화면을 거치지 않은 요청도 막는다    |
| 사이드바 (`buildMenuTree`) | 3단계 아래는 그리지 않음                                  | 데이터가 잘못 들어와도 화면이 안 깨진다 |

### 확인하기

- [ ] 샘플 → 화면 패턴(2단계 폴더)을 고르면 [하위 추가]가 **활성**, 그 아래 "그리드 편집"(3단계)을 고르면 **비활성**입니다.
- [ ] 샘플(1단계)을 고르고 [하위 추가] → 유형을 "폴더"로 → 상위 메뉴에서 "화면 패턴"을 고르고 저장 → "3단계에는 폴더를 만들 수 없습니다." (상위 메뉴 칸에)
- [ ] "화면 패턴" 폴더를 고르고 상위 메뉴 선택지를 열면, 자기 자신은 비활성입니다. "샘플" 아래 다른 2단계 폴더도 고를 수 없습니다. (하위까지 4단계가 되므로)

> 💡 **팁** 이 Step에는 코드 변경이 없어서 커밋도 없습니다. 확인만 하면 됩니다.

---

## Step 10-5. 권한 관리 API

> 🎯 **할 일**: 역할을 등록·수정·삭제하고, 역할별 메뉴 권한을 저장하는 가짜 API를 만듭니다.

### 따라 하기

📄 **`src/mocks/handlers/roles.ts`** (전체 교체)

```ts
import { http, HttpResponse } from "msw";
import { authorize } from "@/mocks/auth";
import { permissionsTable, rolesTable, usersTable } from "@/mocks/db";
import { apiUrl, problem } from "@/mocks/utils";
import type { Permission, Role } from "@/shared/auth/types";

const MENU = "ROLE_MGMT";

export const roleHandlers = [
  // 역할 목록. 사용자 등록 화면의 선택지 등으로 쓰므로 로그인만 하면 조회할 수 있다.
  http.get(apiUrl("/roles"), ({ request }) => {
    const { error } = authorize(request);
    if (error) return error;
    return HttpResponse.json(rolesTable.all());
  }),

  http.post(apiUrl("/roles"), async ({ request }) => {
    const { error } = authorize(request, MENU, "CREATE");
    if (error) return error;
    const input = (await request.json()) as Omit<Role, "id">;
    if (rolesTable.all().some((role) => role.code === input.code)) {
      const message = "이미 있는 역할 코드입니다.";
      return problem(409, message, { errors: [{ field: "code", message }] });
    }
    return HttpResponse.json(rolesTable.insert(input), { status: 201 });
  }),

  http.put(apiUrl("/roles/:id"), async ({ request, params }) => {
    const { error } = authorize(request, MENU, "UPDATE");
    if (error) return error;
    const input = (await request.json()) as Role;
    const role = rolesTable.update(Number(params.id), {
      names: input.names,
      description: input.description,
      useYn: input.useYn,
    });
    return role ? HttpResponse.json(role) : problem(404, "역할을 찾을 수 없습니다.");
  }),

  http.delete(apiUrl("/roles/:id"), ({ request, params }) => {
    const { error } = authorize(request, MENU, "DELETE");
    if (error) return error;
    const role = rolesTable.find(Number(params.id));
    if (!role) return problem(404, "역할을 찾을 수 없습니다.");
    if (usersTable.all().some((user) => user.roleCodes.includes(role.code))) {
      return problem(400, "사용자에게 지정된 역할은 삭제할 수 없습니다.");
    }
    permissionsTable
      .all()
      .filter((permission) => permission.roleCode === role.code)
      .forEach((permission) => permissionsTable.remove(permission.id));
    rolesTable.remove(role.id);
    return new HttpResponse(null, { status: 204 });
  }),

  // ── 역할별 메뉴 권한 ──
  http.get(apiUrl("/roles/:code/permissions"), ({ request, params }) => {
    const { error } = authorize(request, MENU, "READ");
    if (error) return error;
    return HttpResponse.json(
      permissionsTable.all().filter((permission) => permission.roleCode === params.code),
    );
  }),

  // 역할의 권한을 통째로 바꾼다. 보낸 목록이 곧 그 역할의 전체 권한이다.
  http.put(apiUrl("/roles/:code/permissions"), async ({ request, params }) => {
    const { error } = authorize(request, MENU, "UPDATE");
    if (error) return error;
    const roleCode = String(params.code);
    const input = (await request.json()) as Pick<Permission, "menuCode" | "actions">[];
    permissionsTable
      .all()
      .filter((permission) => permission.roleCode === roleCode)
      .forEach((permission) => permissionsTable.remove(permission.id));
    input
      .filter((item) => item.actions.length > 0)
      .forEach((item) => permissionsTable.insert({ ...item, roleCode }));
    return new HttpResponse(null, { status: 204 });
  }),
];
```

📄 **`src/features/role-admin/api/queries.ts`** (새 파일)

```ts
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import { httpClient } from "@/shared/api/httpClient";
import type { Permission, Role } from "@/shared/auth/types";
import { rolesKey } from "@/shared/auth/useRoles";
import { myMenusKey } from "@/shared/menu/useMyMenus";

export type RoleInput = Omit<Role, "id">;
export type PermissionInput = Pick<Permission, "menuCode" | "actions">;

const permissionsKey = (roleCode: string) => ["role-permissions", roleCode] as const;

export function useRolePermissions(roleCode?: string) {
  return useQuery({
    queryKey: permissionsKey(roleCode ?? ""),
    queryFn: () => httpClient.get<Permission[]>(`/roles/${roleCode}/permissions`),
    enabled: !!roleCode,
  });
}

export function useSaveRole() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: ({ id, input }: { id?: number; input: RoleInput }) =>
      id ? httpClient.put<Role>(`/roles/${id}`, input) : httpClient.post<Role>("/roles", input),
    meta: { skipGlobalError: true },
    onSuccess: () => queryClient.invalidateQueries({ queryKey: rolesKey }),
  });
}

export function useDeleteRole() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (id: number) => httpClient.delete(`/roles/${id}`),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: rolesKey }),
  });
}

/** 권한을 저장하면 "내 메뉴"도 무효화한다. 내 역할의 권한을 바꿨다면 사이드바·버튼이 바로 바뀐다. */
export function useSavePermissions(roleCode: string) {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (input: PermissionInput[]) =>
      httpClient.put(`/roles/${roleCode}/permissions`, input),
    onSuccess: () =>
      Promise.all([
        queryClient.invalidateQueries({ queryKey: permissionsKey(roleCode) }),
        queryClient.invalidateQueries({ queryKey: myMenusKey }),
      ]),
  });
}
```

✅ **커밋**

```powershell
git add .
git commit -m "feat: 권한 관리 API 추가"
```

---

## Step 10-6. 권한 관리 화면

> 🎯 **할 일**: 왼쪽에서 역할을 고르면, 오른쪽에 "메뉴 × 동작" 체크 표가 나오는 화면을 만듭니다.

### 체크 규칙

- **조회(READ) 없이 다른 권한은 줄 수 없습니다.** 조회를 끄면 같은 행의 나머지도 꺼지고, 다른 권한을 켜면 조회도 켜집니다.
- 열 제목의 체크박스로 그 동작을 전체 선택·해제합니다.
- 폴더 메뉴는 표에 없습니다. 하위 메뉴 하나라도 조회 권한이 있으면 서버가 폴더도 내려줍니다.
- 저장하지 않은 변경이 있으면 **탭 제목에 ●** 가 붙습니다. Part 7의 `useTabDirty`를 그대로 씁니다.

> 💡 **팁** 이 표는 AG Grid가 아니라 antd `Table`로 만듭니다. 행이 수십 개이고, 칸마다 체크박스가 들어가는 **폼에 가까운 표**이기 때문입니다.
> 대량 데이터·엑셀 같은 조작은 AG Grid, 적은 행의 입력 표는 antd Table. 도구를 상황에 맞게 고르는 연습입니다.

### 따라 하기

📄 **`src/shared/i18n/locales/ko/roleAdmin.json`** (새 파일)

```json
{
  "role": {
    "title": "역할",
    "code": "역할 코드",
    "nameKo": "이름(한국어)",
    "nameEn": "이름(영어)",
    "description": "설명",
    "createTitle": "역할 등록",
    "editTitle": "역할 수정"
  },
  "permission": {
    "title": "메뉴 권한",
    "menu": "메뉴",
    "hidden": "숨김",
    "select": "왼쪽에서 역할을 선택하세요."
  },
  "action": {
    "READ": "조회",
    "CREATE": "등록",
    "UPDATE": "수정",
    "DELETE": "삭제",
    "EXPORT": "내보내기"
  },
  "codeRule": "영문 대문자, 숫자, _ 만 쓸 수 있습니다."
}
```

📄 **`src/shared/i18n/locales/en/roleAdmin.json`** (새 파일)

```json
{
  "role": {
    "title": "Roles",
    "code": "Role code",
    "nameKo": "Name (Korean)",
    "nameEn": "Name (English)",
    "description": "Description",
    "createTitle": "New role",
    "editTitle": "Edit role"
  },
  "permission": {
    "title": "Menu permissions",
    "menu": "Menu",
    "hidden": "Hidden",
    "select": "Select a role on the left."
  },
  "action": {
    "READ": "Read",
    "CREATE": "Create",
    "UPDATE": "Update",
    "DELETE": "Delete",
    "EXPORT": "Export"
  },
  "codeRule": "Use uppercase letters, digits, and underscores only."
}
```

권한 체크 표입니다. 역할을 바꾸면 부모가 `key`로 새로 만들어 주므로, 처음 받은 권한으로 state를 한 번만 초기화하면 됩니다.

📄 **`src/features/role-admin/components/PermissionMatrix.tsx`** (새 파일)

```tsx
import { useMemo, useState } from "react";
import { useTranslation } from "react-i18next";
import { App, Button, Checkbox, Flex, Table, Tag } from "antd";
import type { ColumnsType } from "antd/es/table";
import { useSavePermissions } from "@/features/role-admin/api/queries";
import { usePermission } from "@/shared/auth/usePermission";
import type { Permission } from "@/shared/auth/types";
import { useLocalizedText } from "@/shared/i18n/useLocalizedText";
import { buildMenuTree, getMenuChain } from "@/shared/menu/tree";
import { ACTIONS, type Action, type Menu, type MenuNode } from "@/shared/menu/types";
import { useTabDirty } from "@/shared/tabs/useTabDirty";

type Draft = Record<string, Action[]>; // 메뉴 코드 → 동작

const toDraft = (permissions: Permission[]): Draft =>
  Object.fromEntries(permissions.map((permission) => [permission.menuCode, permission.actions]));

// 비교용: 빈 권한을 빼고, 키 순서를 맞춘다
const normalize = (draft: Draft) =>
  JSON.stringify(
    Object.entries(draft)
      .filter(([, actions]) => actions.length > 0)
      .sort(([a], [b]) => a.localeCompare(b)),
  );

interface PermissionMatrixProps {
  roleCode: string;
  menus: Menu[];
  initial: Permission[];
}

function PermissionMatrix({ roleCode, menus, initial }: PermissionMatrixProps) {
  const { t } = useTranslation("roleAdmin");
  const { message } = App.useApp();
  const { can } = usePermission();
  const localize = useLocalizedText();
  const savePermissions = useSavePermissions(roleCode);
  const editable = can("UPDATE");

  const [saved, setSaved] = useState(() => toDraft(initial)); // 마지막으로 저장된 상태
  const [draft, setDraft] = useState(saved); // 지금 편집 중인 상태
  const dirty = normalize(draft) !== normalize(saved);
  useTabDirty(dirty);

  // 표의 행: 폴더를 뺀 메뉴를 트리 순서대로. "시스템 관리 › 사용자 관리"처럼 경로로 보여 준다.
  const rows = useMemo(() => {
    const flatten = (nodes: MenuNode[]): MenuNode[] =>
      nodes.flatMap((node) => [node, ...flatten(node.children)]);
    return flatten(buildMenuTree(menus))
      .filter((menu) => menu.type !== "FOLDER")
      .map((menu) => ({
        code: menu.code,
        visible: menu.visible,
        label: getMenuChain(menus, menu.code)
          .map((item) => localize(item.names))
          .join(" › "),
      }));
  }, [menus, localize]);

  const has = (code: string, action: Action) => draft[code]?.includes(action) ?? false;

  // 체크 규칙: 조회를 끄면 전부 끈다. 다른 동작을 켜면 조회도 켠다.
  const apply = (current: Action[], action: Action, checked: boolean): Action[] => {
    if (action === "READ" && !checked) return [];
    const next = new Set(current);
    if (checked) next.add(action).add("READ");
    else next.delete(action);
    return ACTIONS.filter((item) => next.has(item)); // 항상 같은 순서로 저장한다
  };

  const toggle = (codes: string[], action: Action, checked: boolean) =>
    setDraft((prev) => {
      const next = { ...prev };
      codes.forEach((code) => (next[code] = apply(prev[code] ?? [], action, checked)));
      return next;
    });

  const columns: ColumnsType<(typeof rows)[number]> = [
    {
      title: t("permission.menu"),
      dataIndex: "label",
      render: (label: string, row) => (
        <>
          {label} {!row.visible && <Tag>{t("permission.hidden")}</Tag>}
        </>
      ),
    },
    ...ACTIONS.map((action) => {
      const checkedCount = rows.filter((row) => has(row.code, action)).length;
      return {
        key: action,
        width: 100,
        align: "center" as const,
        title: (
          <Checkbox
            disabled={!editable}
            checked={checkedCount === rows.length}
            indeterminate={checkedCount > 0 && checkedCount < rows.length}
            onChange={(event) =>
              toggle(
                rows.map((row) => row.code),
                action,
                event.target.checked,
              )
            }
          >
            {t(`action.${action}`)}
          </Checkbox>
        ),
        render: (_: unknown, row: (typeof rows)[number]) => (
          <Checkbox
            disabled={!editable}
            checked={has(row.code, action)}
            onChange={(event) => toggle([row.code], action, event.target.checked)}
          />
        ),
      };
    }),
  ];

  const handleSave = () => {
    const input = Object.entries(draft).map(([menuCode, actions]) => ({ menuCode, actions }));
    savePermissions.mutate(input, {
      onSuccess: () => {
        setSaved(draft);
        message.success(t("common:message.saved"));
      },
    });
  };

  return (
    <Flex vertical gap={8} style={{ height: "100%" }}>
      <Flex justify="end" gap={8}>
        <Button disabled={!dirty} onClick={() => setDraft(saved)}>
          {t("common:button.reset")}
        </Button>
        <Button
          type="primary"
          disabled={!editable || !dirty}
          loading={savePermissions.isPending}
          onClick={handleSave}
        >
          {t("common:button.save")}
        </Button>
      </Flex>
      <Table
        size="small"
        rowKey="code"
        columns={columns}
        dataSource={rows}
        pagination={false}
        scroll={{ y: "calc(100vh - 360px)" }}
      />
    </Flex>
  );
}

export default PermissionMatrix;
```

역할 등록·수정 모달입니다. Part 9의 모달과 같은 패턴입니다.

📄 **`src/features/role-admin/components/RoleFormModal.tsx`** (새 파일)

```tsx
import { useTranslation } from "react-i18next";
import { App, Form, Input, Modal } from "antd";
import { useSaveRole, type RoleInput } from "@/features/role-admin/api/queries";
import { showApiError } from "@/shared/api/showApiError";
import type { Role } from "@/shared/auth/types";
import CodeRadio from "@/shared/codes/CodeRadio";
import { applyServerErrors } from "@/shared/ui/applyServerErrors";

interface RoleFormModalProps {
  role?: Role;
  onClose: () => void;
}

function RoleFormModal({ role, onClose }: RoleFormModalProps) {
  const { t } = useTranslation("roleAdmin");
  const { message } = App.useApp();
  const [form] = Form.useForm<RoleInput>();
  const saveRole = useSaveRole();
  const required = (field: string) => ({
    required: true,
    message: t("common:validation.required", { field }),
  });

  const handleFinish = (input: RoleInput) => {
    saveRole.mutate(
      { id: role?.id, input },
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
      title={role ? t("role.editTitle") : t("role.createTitle")}
      okText={t("common:button.save")}
      cancelText={t("common:button.cancel")}
      onOk={form.submit}
      onCancel={onClose}
      confirmLoading={saveRole.isPending}
      mask={{ closable: false }}
    >
      <Form
        form={form}
        name="roleForm"
        layout="vertical"
        initialValues={role ?? { useYn: "Y", description: "", names: { ko: "", en: "" } }}
        onFinish={handleFinish}
      >
        <Form.Item
          name="code"
          label={t("role.code")}
          rules={[required(t("role.code")), { pattern: /^[A-Z0-9_]+$/, message: t("codeRule") }]}
        >
          <Input disabled={!!role} />
        </Form.Item>
        <Form.Item
          name={["names", "ko"]}
          label={t("role.nameKo")}
          rules={[required(t("role.nameKo"))]}
        >
          <Input />
        </Form.Item>
        <Form.Item name={["names", "en"]} label={t("role.nameEn")}>
          <Input />
        </Form.Item>
        <Form.Item name="description" label={t("role.description")}>
          <Input.TextArea rows={2} />
        </Form.Item>
        <Form.Item name="useYn" label={t("common:label.use")}>
          <CodeRadio group="USE_YN" />
        </Form.Item>
      </Form>
    </Modal>
  );
}

export default RoleFormModal;
```

화면입니다. 메뉴 데이터에 `role-admin/RoleAdminPage`가 이미 있으므로 파일만 만들면 연결됩니다.

📄 **`src/features/role-admin/pages/RoleAdminPage.tsx`** (새 파일)

```tsx
import { useMemo, useState } from "react";
import { useTranslation } from "react-i18next";
import { App, Button, Card, Empty, Flex, Space } from "antd";
import type { ColDef } from "ag-grid-community";
import { useDeleteRole, useRolePermissions } from "@/features/role-admin/api/queries";
import PermissionMatrix from "@/features/role-admin/components/PermissionMatrix";
import RoleFormModal from "@/features/role-admin/components/RoleFormModal";
import Can from "@/shared/auth/Can";
import type { Role } from "@/shared/auth/types";
import { usePermission } from "@/shared/auth/usePermission";
import { useRoles } from "@/shared/auth/useRoles";
import { useCodeName } from "@/shared/codes/useCodes";
import { useLocalizedText } from "@/shared/i18n/useLocalizedText";
import { useAllMenus } from "@/shared/menu/useAllMenus";
import DataGrid from "@/shared/ui/DataGrid";
import PageContainer from "@/shared/ui/PageContainer";
import PageLoading from "@/shared/ui/PageLoading";

function RoleAdminPage() {
  const { t } = useTranslation("roleAdmin");
  const { message, modal } = App.useApp();
  const { can } = usePermission();
  const localize = useLocalizedText();
  const formatUseYn = useCodeName("USE_YN");
  const roles = useRoles();
  const { data: menus } = useAllMenus();
  const [selected, setSelected] = useState<Role>();
  const permissions = useRolePermissions(selected?.code);
  const deleteRole = useDeleteRole();
  const [editing, setEditing] = useState<{ role?: Role } | null>(null);

  const columns = useMemo<ColDef<Role>[]>(
    () => [
      { headerName: t("role.code"), field: "code" },
      { headerName: t("role.nameKo"), valueGetter: ({ data }) => data && localize(data.names) },
      {
        headerName: t("common:label.use"),
        field: "useYn",
        width: 90,
        flex: 0,
        valueFormatter: ({ value }) => formatUseYn(value),
      },
    ],
    [t, localize, formatUseYn],
  );

  const handleDelete = () => {
    if (!selected) return;
    modal.confirm({
      title: t("common:message.confirmDelete", { count: 1 }),
      okButtonProps: { danger: true },
      onOk: () =>
        deleteRole
          .mutateAsync(selected.id)
          .then(() => {
            message.success(t("common:message.deleted"));
            setSelected(undefined);
          })
          .catch(() => undefined),
    });
  };

  return (
    <PageContainer>
      <Flex gap={12} style={{ flex: 1, minHeight: 0 }}>
        <Card
          size="small"
          title={t("role.title")}
          style={{ width: 380, display: "flex", flexDirection: "column" }}
          styles={{ body: { flex: 1, display: "flex", flexDirection: "column", minHeight: 0 } }}
          extra={
            <Space size={4}>
              <Can action="CREATE">
                <Button size="small" onClick={() => setEditing({})}>
                  {t("common:button.add")}
                </Button>
              </Can>
              <Can action="DELETE">
                <Button size="small" danger disabled={!selected} onClick={handleDelete}>
                  {t("common:button.delete")}
                </Button>
              </Can>
            </Space>
          }
        >
          <DataGrid<Role>
            rowData={roles.data}
            columnDefs={columns}
            loading={roles.isFetching}
            getRowId={({ data }) => data.code}
            rowSelection={{ mode: "singleRow", checkboxes: false, enableClickSelection: true }}
            onSelectionChanged={({ api }) => setSelected(api.getSelectedRows()[0])}
            onRowDoubleClicked={({ data }) => data && can("UPDATE") && setEditing({ role: data })}
          />
        </Card>

        <Card
          size="small"
          title={
            selected
              ? `${t("permission.title")} - ${localize(selected.names)}`
              : t("permission.title")
          }
          style={{ flex: 1, minWidth: 0, display: "flex", flexDirection: "column" }}
          styles={{ body: { flex: 1, minHeight: 0 } }}
        >
          {!selected && <Empty description={t("permission.select")} />}
          {selected && (!permissions.data || !menus) && <PageLoading />}
          {selected && permissions.data && menus && (
            // 역할이 바뀌면 key가 바뀌어 표를 새로 만든다 → 그 역할의 권한으로 깨끗하게 시작한다
            <PermissionMatrix
              key={selected.code}
              roleCode={selected.code}
              menus={menus}
              initial={permissions.data}
            />
          )}
        </Card>
      </Flex>

      {editing && <RoleFormModal role={editing.role} onClose={() => setEditing(null)} />}
    </PageContainer>
  );
}

export default RoleAdminPage;
```

### 코드 읽어 보기

**`useState(() => toDraft(initial))`**: 함수를 넘기면 **처음 한 번만** 실행됩니다. 매 렌더마다 계산하지 않으므로 무거운 초기값에 씁니다.

**`saved`와 `draft` 두 상태**: "마지막으로 저장된 것"과 "편집 중인 것"을 나눠 두면, 변경 여부(dirty) 판단·[초기화]·저장 후 갱신이 모두 쉬워집니다.

**`ColumnsType<(typeof rows)[number]>`**: `rows` 배열 원소의 타입을 그대로 꺼내 씁니다. 타입을 따로 선언하지 않아도 됩니다.

### 확인하기

- [ ] `admin` → 시스템 관리 → 권한 관리. `MANAGER`를 고르면 권한 표가 나옵니다.
- [ ] "사용자 관리" 행의 [등록]을 켜면 [조회]도 함께 켜집니다. [조회]를 끄면 그 행이 모두 꺼집니다.
- [ ] 아무거나 바꾸면 탭 제목 앞에 ●가 붙고, 탭을 닫으려 하면 확인 창이 뜹니다.
- [ ] `MANAGER`에 "사용자 관리 · 등록"을 주고 저장 → 로그아웃 → `manager`로 로그인 → 사용자 관리에 [등록] 버튼이 생겼습니다.
- [ ] `admin`으로 돌아와 `ADMIN` 역할의 "메뉴 관리 · 조회"를 끄고 저장하면 **즉시** 사이드바에서 메뉴 관리가 사라지고 탭도 닫힙니다. (Part 7의 정리 로직)
      확인 후 `admin`으로는 다시 켤 수 없으니, F12 → Application → Local Storage에서 `mockdb:permissions`를 지우고 새로고침하면 시드로 돌아옵니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 역할별 메뉴 권한 관리 화면 추가"
```

---

## Step 10-7. 새 화면을 만들어 연결해 보기 ✋

> 🎯 **할 일**: 화면 파일 하나만 만들고, 나머지는 전부 **관리 화면에서** 연결합니다. 이 템플릿이 의도한 개발 흐름을 처음부터 끝까지 해 봅니다.

### 따라 하기

① **화면 파일을 만듭니다.** 코드는 이것뿐입니다.

📄 **`src/features/samples/pages/HelloPage.tsx`** (새 파일)

```tsx
import { Card, Typography } from "antd";
import { usePage } from "@/shared/page/PageContext";
import PageContainer from "@/shared/ui/PageContainer";

function HelloPage() {
  const { menuCode } = usePage();

  return (
    <PageContainer>
      <Card>
        <Typography.Title level={4}>Hello!</Typography.Title>
        <Typography.Paragraph>
          이 화면은 파일 하나만 만들고 메뉴 관리에서 연결했습니다. 메뉴 코드: {menuCode}
        </Typography.Paragraph>
      </Card>
    </PageContainer>
  );
}

export default HelloPage;
```

② **메뉴를 등록합니다.** `admin` → 메뉴 관리 → "샘플" 선택 → [하위 추가]

| 항목      | 값                       |
| --------- | ------------------------ |
| 유형      | 화면                     |
| 메뉴 코드 | `SAMPLE_HELLO`           |
| 메뉴명    | 헬로 / Hello             |
| 주소      | `/samples/hello`         |
| 화면 파일 | `samples/HelloPage` (목록에서 선택. "파일이 있습니다" 초록 글씨) |
| 아이콘    | `SendOutlined`           |

[저장] → **사이드바에 "헬로"가 바로 나타납니다.** (관리자 역할에는 새 메뉴 권한이 자동으로 붙습니다)

③ **권한을 줍니다.** 권한 관리 → `USER` → "샘플 › 헬로"의 [조회] 켜기 → [저장]

④ 로그아웃 → `user`로 로그인 → 샘플 → 헬로가 보이고 열립니다.

### 스스로 답해 보기

<details>
<summary>Q. 라우트 등록, 메뉴 배열 수정, import 추가를 했나요?</summary>

하나도 하지 않았습니다. 화면 파일만 규칙(`features/*/pages/*Page.tsx`)대로 만들면 화면 레지스트리가 찾아 주고,
메뉴·주소·제목·탭·권한은 전부 **데이터**가 정합니다. 이것이 Part 6부터 쌓아 온 "화면은 데이터가 정한다" 설계의 결과입니다.

</details>

> ⚠️ **주의** 메뉴와 권한은 가짜 DB(localStorage)에 저장되므로 Git 커밋 대상이 아닙니다. 커밋하는 것은 화면 파일 하나뿐입니다.
> 가짜 DB를 지우면 메뉴도 사라지니, 오래 남길 메뉴는 `src/mocks/data/menus.ts` 시드에 추가하세요. (실제 프로젝트에서는 서버 DB에 남습니다)

✅ **커밋**

```powershell
git add .
git commit -m "feat: 샘플 헬로 화면 추가"
```

---

## 🏁 Part 10을 마치며

마일스톤입니다. 관리자가 화면에서 메뉴를 만들고 소스 파일을 연결하며, 역할별로 메뉴·버튼 권한을 줄 수 있습니다.

| 증상                                       | 원인과 해결                                                                 |
| ------------------------------------------ | --------------------------------------------------------------------------- |
| 메뉴를 추가했는데 사이드바에 안 보인다     | 권한이 없다 → 권한 관리에서 [조회]를 준다 (ADMIN은 자동)                    |
| 화면 파일을 골랐는데 "준비 중"이다         | 파일 이름 규칙, default export 확인. 새 파일은 개발 서버가 자동 인식한다   |
| 권한을 바꿨는데 화면이 그대로다            | 저장 후 `["me", "menus"]` 무효화 누락                                       |
| 메뉴 코드를 바꾸고 싶다                    | 코드는 권한·탭이 참조하는 키라 바꿀 수 없다 → 새 메뉴를 만들고 옛 메뉴 삭제 |

다음은 관리자가 글자색·굵게 등 서식을 넣어 쓰는 공지사항입니다. → [Part 11. 공지사항 (서식 에디터)](part-11-notice.md)
