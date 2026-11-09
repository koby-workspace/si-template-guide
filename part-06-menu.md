# Part 6. 데이터로 그리는 메뉴

> **이 Part의 목표**: 사이드바 메뉴를 코드가 아니라 **서버 데이터**로 그리고, 메뉴 데이터의 `component` 값(예: `"users/UsersPage"`)으로 화면 파일을 찾아 엽니다.
> 이 구조 덕분에 Part 10에서 관리자가 화면에서 메뉴를 추가하고 소스 파일을 매핑할 수 있게 됩니다. 이 템플릿의 심장입니다.

---

## Step 6-1. 메뉴 데이터 모델과 API

> 🎯 **할 일**: 메뉴의 모양을 타입으로 정하고, 가짜 서버가 메뉴 목록을 내려주게 합니다.

### 원리: 메뉴는 세 종류다

| 유형     | 하는 일                          | 필요한 정보                          |
| -------- | -------------------------------- | ------------------------------------ |
| `FOLDER` | 하위 메뉴를 묶는다. 펼치기만 한다 | 이름, 아이콘                         |
| `PAGE`   | 화면을 연다                      | 이름, 주소(`path`), 화면 파일(`component`) |
| `LINK`   | 외부 사이트를 새 창으로 연다     | 이름, `url`                          |

유형마다 필요한 정보가 다릅니다. 이런 데이터는 **구별된 유니온**(discriminated union)으로 표현합니다.

```ts
type Menu = FolderMenu | PageMenu | LinkMenu;

if (menu.type === "PAGE") {
  menu.component; // ✅ 이 안에서는 TypeScript가 PageMenu라고 확신한다 (타입 좁히기)
}
menu.component; // ❌ 오류. FOLDER에는 component가 없으니까
```

"화면 메뉴인데 화면 파일이 없음" 같은 잘못된 데이터를 **타입 단계에서** 막을 수 있습니다.

**왜 `id`가 아니라 `code`로 참조하나요?** 개발 DB와 운영 DB의 `id`는 달라질 수 있지만 `code`(예: `USER_MGMT`)는 같습니다.
권한·탭·즐겨찾기가 `code`를 참조하면 환경이 바뀌어도 깨지지 않습니다.

**메뉴 이름은 왜 `{ ko, en }`인가요?** 관리자가 화면에서 메뉴를 추가하므로, 메뉴 이름은 JSON 번역 파일이 아니라 **데이터**에 언어별로 저장합니다.
SI DB는 보통 `MENU_NM_KO`, `MENU_NM_EN`처럼 컬럼을 나누는데, 백엔드가 `names` 객체로 묶어 주면 됩니다.

### 따라 하기

데이터에 들어가는 다국어 문구의 타입과, 현재 언어로 꺼내는 훅을 만듭니다.

✏️ **`src/shared/i18n/locales.ts`** — 타입을 추가합니다.

```ts
export type Locale = "ko" | "en";
```

바꿀 코드

```ts
export type Locale = "ko" | "en";

/** 데이터에 저장되는 다국어 문구. 예: 메뉴 이름 { ko: "사용자 관리", en: "Users" } */
export type LocalizedText = Record<Locale, string>;
```

📄 **`src/shared/i18n/useLocalizedText.ts`** (새 파일)

```ts
import { useCallback } from "react";
import type { LocalizedText } from "@/shared/i18n/locales";
import { usePreferences } from "@/shared/preferences/preferencesStore";

/** 현재 언어의 문구를 고르는 함수를 돌려준다. 번역이 비어 있으면 한국어로 대신한다. */
export function useLocalizedText() {
  const locale = usePreferences((state) => state.locale);
  return useCallback((text: LocalizedText) => text[locale] || text.ko, [locale]);
}
```

메뉴 타입입니다.

📄 **`src/shared/menu/types.ts`** (새 파일)

```ts
import type { UseYn } from "@/shared/api/types";
import type { LocalizedText } from "@/shared/i18n/locales";

export type MenuType = "FOLDER" | "PAGE" | "LINK";

interface MenuBase {
  id: number;
  parentId: number | null; // 최상위면 null
  code: string; // 바뀌지 않는 식별자. 권한·탭·즐겨찾기가 이 값을 참조한다
  names: LocalizedText;
  icon?: string; // 아이콘 이름 (예: "TeamOutlined")
  sortOrder: number;
  visible: boolean; // false면 사이드바에 숨긴다 (상세 화면처럼 다른 화면에서만 여는 메뉴)
  useYn: UseYn; // "N"이면 아예 사용하지 않는다
}

export interface FolderMenu extends MenuBase {
  type: "FOLDER";
}

export interface PageMenu extends MenuBase {
  type: "PAGE";
  path: string; // 주소. 예: "/system/users"
  component: string; // 화면 파일 키. 예: "users/UsersPage"
}

export interface LinkMenu extends MenuBase {
  type: "LINK";
  url: string; // 외부 주소. 새 창으로 연다
}

export type Menu = FolderMenu | PageMenu | LinkMenu;

// 트리로 바꾼 메뉴. Menu가 유니온이라 interface extends를 쓸 수 없어서 type으로 만든다.
export type MenuNode = Menu & { children: MenuNode[] };
```

메뉴 시드입니다. **앞으로 만들 화면까지 전부 미리 넣습니다.** 아직 파일이 없는 화면은 "준비 중"으로 보입니다.
SI에서는 메뉴 구성을 먼저 확정하고 화면을 하나씩 채워 가는 경우가 많은데, 그 흐름을 그대로 체험할 수 있습니다.

📄 **`src/mocks/data/menus.ts`** (새 파일)

```ts
import type { FolderMenu, LinkMenu, Menu, PageMenu } from "@/shared/menu/types";

// 표처럼 읽히도록 만든 도우미. sortOrder는 id 순서를 그대로 쓴다.
function base(id: number, parentId: number | null, code: string, ko: string, en: string) {
  return {
    id,
    parentId,
    code,
    names: { ko, en },
    sortOrder: id,
    visible: true,
    useYn: "Y" as const,
  };
}

function folder(
  id: number,
  parentId: number | null,
  code: string,
  ko: string,
  en: string,
  icon: string,
): FolderMenu {
  return { ...base(id, parentId, code, ko, en), type: "FOLDER", icon };
}

function page(
  id: number,
  parentId: number | null,
  code: string,
  ko: string,
  en: string,
  path: string,
  component: string,
  icon?: string,
): PageMenu {
  return { ...base(id, parentId, code, ko, en), type: "PAGE", path, component, icon };
}

function link(
  id: number,
  parentId: number | null,
  code: string,
  ko: string,
  en: string,
  url: string,
  icon: string,
): LinkMenu {
  return { ...base(id, parentId, code, ko, en), type: "LINK", url, icon };
}

export const menuSeed: Menu[] = [
  page(
    1,
    null,
    "DASHBOARD",
    "대시보드",
    "Dashboard",
    "/dashboard",
    "dashboard/DashboardPage",
    "DashboardOutlined",
  ),

  folder(10, null, "SYSTEM", "시스템 관리", "System", "SettingOutlined"),
  page(
    11,
    10,
    "USER_MGMT",
    "사용자 관리",
    "Users",
    "/system/users",
    "users/UsersPage",
    "TeamOutlined",
  ),
  page(
    12,
    10,
    "ROLE_MGMT",
    "권한 관리",
    "Roles",
    "/system/roles",
    "role-admin/RoleAdminPage",
    "SafetyOutlined",
  ),
  page(
    13,
    10,
    "MENU_MGMT",
    "메뉴 관리",
    "Menus",
    "/system/menus",
    "menu-admin/MenuAdminPage",
    "MenuOutlined",
  ),
  page(
    14,
    10,
    "CODE_MGMT",
    "공통코드 관리",
    "Common codes",
    "/system/codes",
    "common-codes/CommonCodesPage",
    "TagsOutlined",
  ),
  page(
    15,
    10,
    "NOTI_RULE",
    "알림 규칙",
    "Notification rules",
    "/system/notification-rules",
    "notification/NotificationRulePage",
    "BellOutlined",
  ),

  folder(20, null, "BOARD", "게시판", "Board", "ReadOutlined"),
  page(
    21,
    20,
    "NOTICE",
    "공지사항",
    "Notices",
    "/board/notices",
    "notice/NoticeListPage",
    "NotificationOutlined",
  ),
  {
    ...page(
      22,
      20,
      "NOTICE_VIEW",
      "공지사항 상세",
      "Notice",
      "/board/notices/view",
      "notice/NoticeViewPage",
    ),
    visible: false,
  },
  {
    ...page(
      23,
      20,
      "NOTICE_EDIT",
      "공지사항 작성",
      "Write notice",
      "/board/notices/edit",
      "notice/NoticeEditPage",
    ),
    visible: false,
  },

  // 3단계 예시: 샘플 > 화면 패턴 > 그리드 편집
  folder(30, null, "SAMPLE", "샘플", "Samples", "AppstoreOutlined"),
  folder(31, 30, "SAMPLE_PATTERN", "화면 패턴", "Patterns", "FolderOutlined"),
  page(
    32,
    31,
    "SAMPLE_GRID_EDIT",
    "그리드 편집",
    "Editable grid",
    "/samples/patterns/grid-edit",
    "samples/GridEditPage",
    "TableOutlined",
  ),
  page(
    33,
    31,
    "SAMPLE_FORM",
    "입력 폼",
    "Form",
    "/samples/patterns/form",
    "samples/FormSamplePage",
    "FormOutlined",
  ),
  page(
    34,
    30,
    "SAMPLE_NOTIFY",
    "알림 발생 테스트",
    "Notification test",
    "/samples/notify-test",
    "samples/NotifyTestPage",
    "SendOutlined",
  ),
  page(
    35,
    30,
    "SAMPLE_FILE",
    "파일 업로드",
    "File upload",
    "/samples/file",
    "samples/FileSamplePage",
    "CloudUploadOutlined",
  ),

  folder(40, null, "MY", "내 정보", "My page", "UserOutlined"),
  page(
    41,
    40,
    "MY_NOTIFICATIONS",
    "알림함",
    "Notifications",
    "/my/notifications",
    "notification/NotificationCenterPage",
    "BellOutlined",
  ),
  page(
    42,
    40,
    "MY_PROFILE",
    "내 정보 수정",
    "Profile",
    "/my/profile",
    "profile/ProfilePage",
    "ProfileOutlined",
  ),

  link(
    90,
    null,
    "MANUAL",
    "사용자 매뉴얼",
    "User manual",
    "https://ant.design/components/overview",
    "QuestionCircleOutlined",
  ),
];
```

> 💡 **팁** 줄이 긴 호출은 Prettier가 여러 줄로 나눠 둔 모양입니다. 직접 입력할 때는 한 줄로 써도 저장(`Ctrl+S`)하면 똑같이 정리됩니다.

✏️ **`src/mocks/db.ts`** — import를 추가합니다.

```ts
import { userSeed } from "@/mocks/data/users";
```

바꿀 코드

```ts
import { menuSeed } from "@/mocks/data/menus";
import { userSeed } from "@/mocks/data/users";
```

✏️ **`src/mocks/db.ts`** — 테이블을 추가합니다.

```ts
export const usersTable = createTable("users", userSeed);
```

바꿀 코드

```ts
export const usersTable = createTable("users", userSeed);
export const menusTable = createTable("menus", menuSeed);
```

📄 **`src/mocks/handlers/menus.ts`** (새 파일)

```ts
import { http, HttpResponse } from "msw";
import { menusTable } from "@/mocks/db";
import { apiUrl } from "@/mocks/utils";

export const menuHandlers = [
  // 내가 쓸 수 있는 메뉴. 지금은 사용 중인 메뉴를 전부 준다. (Part 8에서 권한에 따라 거른다)
  http.get(apiUrl("/me/menus"), () => {
    return HttpResponse.json(menusTable.all().filter((menu) => menu.useYn === "Y"));
  }),
];
```

✏️ **`src/mocks/handlers.ts`** — 핸들러를 등록합니다.

```ts
import { userHandlers } from "@/mocks/handlers/users";
```

바꿀 코드

```ts
import { menuHandlers } from "@/mocks/handlers/menus";
import { userHandlers } from "@/mocks/handlers/users";
```

✏️ **`src/mocks/handlers.ts`** — 배열에 추가합니다.

```ts
  ...userHandlers,
];
```

바꿀 코드

```ts
  ...userHandlers,
  ...menuHandlers,
];
```

앱 쪽 API와 Query 훅입니다. 메뉴는 여러 기능이 함께 쓰므로 `shared`에 둡니다.

📄 **`src/shared/menu/useMyMenus.ts`** (새 파일)

```ts
import { useQuery } from "@tanstack/react-query";
import { httpClient } from "@/shared/api/httpClient";
import type { Menu } from "@/shared/menu/types";

export const myMenusKey = ["me", "menus"] as const;

/** 내가 쓸 수 있는 메뉴 목록. 사이드바·탭·화면 제목·권한이 모두 이 데이터 하나를 본다. */
export function useMyMenus() {
  return useQuery({
    queryKey: myMenusKey,
    queryFn: () => httpClient.get<Menu[]>("/me/menus"),
    // 메뉴는 메뉴 관리에서 저장할 때만 바뀐다. 그때 직접 무효화하므로 저절로 낡지 않게 한다.
    staleTime: Infinity,
  });
}
```

> 💡 **팁** `staleTime: Infinity`를 빼면 어떻게 될까요? 이 훅은 화면마다 쓰이므로, 화면이 나타날 때마다 메뉴를 다시 조회하게 됩니다.
> "언제 낡는가"를 데이터 성격에 맞게 정하는 것이 Query를 잘 쓰는 핵심입니다.

### 확인하기

- [ ] F12 → Console에서 `await (await fetch("/api/me/menus")).json()` → 메뉴 배열이 나옵니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 메뉴 데이터 API 추가"
```

---

## Step 6-2. 사이드바를 메뉴 데이터로

> 🎯 **할 일**: 평평한 메뉴 목록을 트리로 바꿔 사이드바에 그립니다. 깊이는 최대 3단계입니다.

### 원리: 평평한 목록 → 트리

서버는 메뉴를 **평평한 목록**(각 메뉴가 `parentId`만 가짐)으로 줍니다. DB 테이블 모양 그대로입니다.
화면은 **트리**가 필요합니다. 그래서 변환합니다.

```
[ {id:10, parentId:null}, {id:11, parentId:10}, {id:31, parentId:30}, ... ]
                              ↓ buildMenuTree
[ {id:10, children:[ {id:11, children:[]} ]}, {id:30, children:[ {id:31, children:[...]} ]} ]
```

자기 자신을 부르는 함수(**재귀**)로 만듭니다. "내 자식들을 찾아서, 각 자식에 대해 또 그 자식들을 찾는다."

**깊이 3단계 제한**: 메뉴가 너무 깊으면 찾기 어렵습니다. 그래서 이 템플릿은 최대 3단계로 정합니다.
이 제한은 세 군데에서 지킵니다. 메뉴 관리 화면(Part 10)은 4단계를 만들 수 없게 막고, 가짜 서버도 저장을 거부합니다.
여기 트리 변환에서도 3단계 아래는 그리지 않습니다. 데이터가 잘못 들어와도 화면이 깨지지 않게 하는 마지막 안전장치입니다.

### 따라 하기

📄 **`src/shared/menu/tree.ts`** (새 파일)

```ts
import type { Menu, MenuNode } from "@/shared/menu/types";

/** 메뉴 깊이 제한. 메뉴 관리 화면·가짜 서버·사이드바가 모두 이 값을 따른다. */
export const MAX_MENU_DEPTH = 3;

/** 평평한 목록을 트리로 바꾼다. 메뉴는 많아야 수백 개라 단순한 재귀로 충분하다. */
export function buildMenuTree(menus: Menu[]): MenuNode[] {
  const sorted = [...menus].sort((a, b) => a.sortOrder - b.sortOrder);
  const build = (parentId: number | null, depth: number): MenuNode[] =>
    sorted
      .filter((menu) => menu.parentId === parentId)
      .map((menu) => ({
        ...menu,
        children: depth < MAX_MENU_DEPTH ? build(menu.id, depth + 1) : [],
      }));
  return build(null, 1);
}

/** 주소로 화면 메뉴를 찾는다. */
export function findMenuByPath(menus: Menu[], path: string) {
  return menus.find((menu) => menu.type === "PAGE" && menu.path === path);
}

/** 메뉴의 조상을 위에서부터 차례로 돌려준다. 자기 자신을 포함한다. (브레드크럼, 폴더 펼치기용) */
export function getMenuChain(menus: Menu[], code: string): Menu[] {
  const chain: Menu[] = [];
  let current = menus.find((menu) => menu.code === code);
  while (current) {
    chain.unshift(current);
    const parentId: number | null = current.parentId;
    current = menus.find((menu) => menu.id === parentId);
  }
  return chain;
}
```

메뉴 데이터에는 아이콘 **이름**(문자열)만 있습니다. 이름으로 실제 아이콘을 찾는 표를 만듭니다.

📄 **`src/shared/menu/menuIcons.ts`** (새 파일)

```ts
import {
  AppstoreOutlined,
  BellOutlined,
  CloudUploadOutlined,
  DashboardOutlined,
  FileOutlined,
  FolderOutlined,
  FormOutlined,
  HomeOutlined,
  MenuOutlined,
  NotificationOutlined,
  ProfileOutlined,
  QuestionCircleOutlined,
  ReadOutlined,
  SafetyOutlined,
  SendOutlined,
  SettingOutlined,
  TableOutlined,
  TagsOutlined,
  TeamOutlined,
  UserOutlined,
} from "@ant-design/icons";

// 아이콘 전체(import * as Icons)를 가져오면 수백 개가 번들에 들어간다. 쓸 것만 등록한다.
// 메뉴 관리 화면의 아이콘 선택 목록도 이 표를 쓴다. 아이콘을 추가하려면 여기에 한 줄 추가한다.
export const menuIcons = {
  AppstoreOutlined,
  BellOutlined,
  CloudUploadOutlined,
  DashboardOutlined,
  FileOutlined,
  FolderOutlined,
  FormOutlined,
  HomeOutlined,
  MenuOutlined,
  NotificationOutlined,
  ProfileOutlined,
  QuestionCircleOutlined,
  ReadOutlined,
  SafetyOutlined,
  SendOutlined,
  SettingOutlined,
  TableOutlined,
  TagsOutlined,
  TeamOutlined,
  UserOutlined,
};

export type MenuIconName = keyof typeof menuIcons;
```

📄 **`src/shared/menu/MenuIcon.tsx`** (새 파일)

```tsx
import { menuIcons, type MenuIconName } from "@/shared/menu/menuIcons";

interface MenuIconProps {
  name?: string;
}

/** 아이콘 이름으로 아이콘을 그린다. 등록되지 않은 이름이면 아무것도 그리지 않는다. */
function MenuIcon({ name }: MenuIconProps) {
  const Icon = name ? menuIcons[name as MenuIconName] : undefined;
  return Icon ? <Icon /> : null;
}

export default MenuIcon;
```

> ⚠️ **주의** 아이콘 표(`menuIcons.ts`)와 컴포넌트(`MenuIcon.tsx`)를 한 파일에 두면 ESLint가 경고합니다(`react-refresh/only-export-components`).
> 컴포넌트 파일에서 컴포넌트 외의 것을 함께 내보내면, 저장할 때 HMR이 화면 상태를 유지하지 못하기 때문입니다. **컴포넌트 파일은 컴포넌트만 내보냅니다.**

이제 사이드바를 다시 씁니다.

📄 **`src/app/layout/SideMenu.tsx`** (전체 교체)

```tsx
import { useMemo, useState } from "react";
import { useLocation, useNavigate } from "react-router";
import { Menu, Skeleton } from "antd";
import type { ItemType } from "antd/es/menu/interface";
import { ExportOutlined } from "@ant-design/icons";
import { useLocalizedText } from "@/shared/i18n/useLocalizedText";
import MenuIcon from "@/shared/menu/MenuIcon";
import { buildMenuTree, findMenuByPath, getMenuChain } from "@/shared/menu/tree";
import type { MenuNode } from "@/shared/menu/types";
import { useMyMenus } from "@/shared/menu/useMyMenus";

function SideMenu() {
  const { pathname } = useLocation();
  const navigate = useNavigate();
  const localize = useLocalizedText();
  const { data: menus } = useMyMenus();
  const [openKeys, setOpenKeys] = useState<string[]>([]);

  const activeCode = menus ? findMenuByPath(menus, pathname)?.code : undefined;

  // 다른 화면으로 이동하면 그 화면의 상위 폴더를 펼친다.
  // "이전 값과 다를 때만 state를 조정"하는 패턴이다. useEffect보다 한 번 덜 그린다.
  const [syncedCode, setSyncedCode] = useState<string>();
  if (menus && activeCode && activeCode !== syncedCode) {
    setSyncedCode(activeCode);
    const folderCodes = getMenuChain(menus, activeCode).map((menu) => menu.code);
    setOpenKeys((keys) => [...new Set([...keys, ...folderCodes])]);
  }

  // 메뉴 트리 → antd Menu의 items 형식
  const items = useMemo(() => {
    const toItem = (node: MenuNode): ItemType | null => {
      if (!node.visible) return null;
      const label = localize(node.names);
      const icon = <MenuIcon name={node.icon} />;
      if (node.type === "FOLDER") {
        const children = node.children.map(toItem).filter((item) => item !== null);
        return { key: node.code, icon, label, children };
      }
      if (node.type === "LINK") {
        return {
          key: node.code,
          icon,
          label: (
            <>
              {label} <ExportOutlined style={{ fontSize: 11 }} />
            </>
          ),
        };
      }
      return { key: node.code, icon, label };
    };
    return buildMenuTree(menus ?? [])
      .map(toItem)
      .filter((item) => item !== null);
  }, [menus, localize]);

  if (!menus) return <Skeleton active style={{ padding: 16 }} />;

  const handleClick = ({ key }: { key: string }) => {
    const menu = menus.find((item) => item.code === key);
    if (menu?.type === "PAGE") navigate(menu.path);
    if (menu?.type === "LINK") window.open(menu.url, "_blank", "noopener");
  };

  return (
    <Menu
      mode="inline"
      items={items}
      selectedKeys={activeCode ? [activeCode] : []}
      openKeys={openKeys}
      onOpenChange={setOpenKeys}
      onClick={handleClick}
      style={{ borderInlineEnd: "none" }}
    />
  );
}

export default SideMenu;
```

메뉴가 길어지면 사이드바 안에서 스크롤되도록 합니다.

✏️ **`src/app/layout/MainLayout.tsx`** — 사이드바에 스크롤을 줍니다.

```tsx
          style={{ borderInlineEnd: `1px solid ${token.colorBorderSecondary}` }}
```

바꿀 코드

```tsx
          style={{
            borderInlineEnd: `1px solid ${token.colorBorderSecondary}`,
            overflow: "auto",
          }}
```

### 코드 읽어 보기

**`node.type === "FOLDER"` 분기**: 유니온의 타입 좁히기가 실제로 쓰이는 곳입니다. `LINK` 분기 안에서만 `node.url`을 쓸 수 있습니다.

**`.filter((item) => item !== null)`**: 숨김 메뉴(`visible: false`)를 빼는 처리입니다. TypeScript 5.5부터는 이렇게만 써도 결과 타입에서 `null`이 빠집니다.

**렌더 중 state 조정**: `useEffect` 안에서 `setState`를 부르면 "그렸다 → 다시 그린다"가 되어 한 번 깜빡입니다.
"이전에 맞춘 값(`syncedCode`)과 다를 때만" 렌더 중에 조정하면 React가 화면에 그리기 전에 바로 다시 계산합니다.
조건이 없으면 무한 반복되므로, 반드시 **"이전 값과 다를 때만"** 이라는 조건을 붙입니다.

> 💡 **팁** `window.open(url, "_blank", "noopener")`의 `noopener`는 새 창이 원래 창을 조작하지 못하게 하는 보안 옵션입니다. 외부 링크에는 항상 붙입니다.

### 확인하기

- [ ] 사이드바에 대시보드, 시스템 관리, 게시판, 샘플, 내 정보, 사용자 매뉴얼이 보입니다.
- [ ] 샘플 → 화면 패턴 → 그리드 편집: **3단계**까지 펼쳐집니다.
- [ ] "공지사항 상세"·"공지사항 작성"은 `visible: false`라 보이지 않습니다.
- [ ] English로 바꾸면 메뉴 이름이 영어로 바뀝니다. (데이터의 `names.en`)
- [ ] 사용자 매뉴얼을 누르면 새 창이 열립니다.
- [ ] 다른 메뉴는 아직 눌러도 404입니다. 다음 Step에서 연결합니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 메뉴 데이터로 사이드바 표시"
```

---

## Step 6-3. 메뉴와 화면 파일 자동 연결

> 🎯 **할 일**: 메뉴의 `component` 문자열로 화면 파일을 찾아서 그립니다. 라우트를 하나하나 등록하지 않습니다.

### 원리: 왜 `import(menu.component)`를 바로 못 하나요?

Vite는 **빌드할 때** 어떤 파일을 묶을지 정해야 합니다. 실행 중에 들어오는 임의의 문자열로는 파일을 찾을 수 없습니다.
대신 `import.meta.glob`으로 "이 패턴에 맞는 파일 전부"를 **빌드 시점에 목록으로** 만들어 두고, 실행 중에는 그 목록에서 고릅니다.

```
빌드할 때:  src/features/*/pages/*Page.tsx 를 전부 찾아 표를 만든다
            "users/UsersPage"         → () => import("/src/features/users/pages/UsersPage.tsx")
            "dashboard/DashboardPage" → () => import(".../DashboardPage.tsx")
실행할 때:  메뉴의 component 값 "users/UsersPage"로 표를 찾아 불러온다
```

보안상으로도 좋습니다. 목록에 없는 파일은 메뉴 데이터를 아무리 조작해도 열 수 없습니다.

**코드 분할**: `import()`로 불러오는 화면은 **별도의 JS 파일**로 빌드됩니다. 처음 화면이 뜰 때 모든 화면 코드를 받지 않고,
그 화면을 처음 열 때 받습니다. 화면이 100개인 SI 시스템에서 첫 로딩 속도를 지키는 핵심입니다.
받는 동안 보여 줄 로딩 화면은 React의 `lazy`와 `Suspense`가 처리합니다.

### 규칙

- 메뉴에 연결할 화면은 `src/features/<기능>/pages/<이름>Page.tsx`에 두고 **default export** 합니다.
- 화면 파일 키는 `<기능>/<이름>Page`입니다. 예: `users/UsersPage`
- 메뉴에 연결되면 안 되는 화면(로그인 등)은 `pages/` 폴더 **밖**에 둡니다.

### 따라 하기

📄 **`src/app/pageRegistry.ts`** (새 파일)

```ts
import { lazy, type ComponentType, type LazyExoticComponent } from "react";

// features/*/pages/*Page.tsx 파일을 빌드할 때 전부 찾는다. 파일 내용은 처음 열 때 내려받는다 (코드 분할).
// 패턴은 반드시 문자열 그대로 써야 한다. 변수로 만들면 Vite가 분석하지 못한다.
const modules = import.meta.glob<{ default: ComponentType }>("/src/features/*/pages/*Page.tsx");

// "/src/features/users/pages/UsersPage.tsx" → "users/UsersPage"
function toPageKey(filePath: string) {
  return filePath.replace("/src/features/", "").replace("/pages/", "/").replace(".tsx", "");
}

const loaders = new Map(Object.entries(modules).map(([path, load]) => [toPageKey(path), load]));

// lazy()는 화면마다 딱 한 번만 만든다. 렌더할 때마다 만들면 매번 "새 컴포넌트"가 되어 화면 상태가 날아간다.
const cache = new Map<string, LazyExoticComponent<ComponentType>>();

/** 존재하는 화면 파일 키 목록. 메뉴 관리 화면의 "화면 파일" 선택지로 쓴다. */
export const pageKeys = [...loaders.keys()].sort();

/** 화면 파일 키로 화면 컴포넌트를 찾는다. 파일이 없으면 undefined. */
export function getPage(pageKey: string) {
  const load = loaders.get(pageKey);
  if (!load) return undefined;
  if (!cache.has(pageKey)) cache.set(pageKey, lazy(load));
  return cache.get(pageKey);
}
```

화면을 불러오는 동안과, 화면 파일이 아직 없을 때 보여 줄 화면입니다.

📄 **`src/shared/ui/PageLoading.tsx`** (새 파일)

```tsx
import { Flex, Spin } from "antd";

function PageLoading() {
  return (
    <Flex align="center" justify="center" style={{ height: "100%", minHeight: 200 }}>
      <Spin />
    </Flex>
  );
}

export default PageLoading;
```

📄 **`src/app/errors/NotReadyPage.tsx`** (새 파일)

```tsx
import { useTranslation } from "react-i18next";
import { Result } from "antd";
import { ToolOutlined } from "@ant-design/icons";

interface NotReadyPageProps {
  component: string;
}

function NotReadyPage({ component }: NotReadyPageProps) {
  const { t } = useTranslation("errors");

  return (
    <Result
      icon={<ToolOutlined />}
      title={t("page.notReady")}
      subTitle={t("page.notReadyDetail", { component })}
    />
  );
}

export default NotReadyPage;
```

메뉴 하나를 받아 화면을 그리는 컴포넌트입니다. Part 7의 탭도 이 컴포넌트를 그대로 씁니다.

📄 **`src/app/layout/PageRenderer.tsx`** (새 파일)

```tsx
import { Suspense } from "react";
import NotReadyPage from "@/app/errors/NotReadyPage";
import { getPage } from "@/app/pageRegistry";
import type { PageMenu } from "@/shared/menu/types";
import PageLoading from "@/shared/ui/PageLoading";

interface PageRendererProps {
  menu: PageMenu;
}

/** 메뉴의 component 값으로 화면 파일을 찾아 그린다. 파일이 없으면 "준비 중"을 보여 준다. */
function PageRenderer({ menu }: PageRendererProps) {
  const Page = getPage(menu.component);
  if (!Page) return <NotReadyPage component={menu.component} />;

  return (
    // 화면 JS 파일을 처음 내려받는 동안 로딩을 보여 준다.
    <Suspense fallback={<PageLoading />}>
      {/* eslint-disable-next-line react-hooks/static-components -- getPage는 캐시된 같은 컴포넌트를 돌려준다 */}
      <Page />
    </Suspense>
  );
}

export default PageRenderer;
```

### 코드 읽어 보기: 린트 규칙을 끄는 법

`eslint-disable-next-line` 주석을 빼면 ESLint가 이렇게 경고합니다.

```
This component is created during render   react-hooks/static-components
```

이 규칙은 "렌더할 때마다 새 컴포넌트를 만드는 실수"를 막아 줍니다. 바로 위 `cache`가 없을 때 생기는 버그를 잡아 주는 고마운 규칙입니다.
그런데 ESLint는 `getPage` 안에 캐시가 있다는 사실까지는 모릅니다. 이렇게 **규칙이 알 수 없는 사실을 우리가 보장할 때만**, 그 한 줄에서 규칙을 끕니다.

- 파일 전체가 아니라 **그 줄만** 끕니다. (`eslint-disable-next-line`)
- `--` 뒤에 **이유**를 적습니다. 나중에 읽는 사람이 "왜 껐지?"를 고민하지 않게 합니다.
- JSX 안에서는 주석을 `{/* */}`로 감쌉니다.

현재 주소에 맞는 메뉴를 찾아 `PageRenderer`에 넘기는 컴포넌트입니다. (Part 7에서 탭으로 바뀌면서 이 파일은 지웁니다.)

📄 **`src/app/layout/RoutedPage.tsx`** (새 파일)

```tsx
import { useLocation } from "react-router";
import NotFoundPage from "@/app/errors/NotFoundPage";
import PageRenderer from "@/app/layout/PageRenderer";
import { findMenuByPath } from "@/shared/menu/tree";
import { useMyMenus } from "@/shared/menu/useMyMenus";
import PageLoading from "@/shared/ui/PageLoading";

/** 현재 주소의 메뉴를 찾아 화면을 그린다. */
function RoutedPage() {
  const { pathname } = useLocation();
  const { data: menus } = useMyMenus();

  if (!menus) return <PageLoading />;
  const menu = findMenuByPath(menus, pathname);
  if (menu?.type !== "PAGE") return <NotFoundPage />;
  return <PageRenderer menu={menu} />;
}

export default RoutedPage;
```

> 💡 **팁** `findMenuByPath`의 결과 타입은 `Menu | undefined`입니다. `menu?.type !== "PAGE"`로 걸러 낸 다음 줄에서는
> TypeScript가 `menu`를 `PageMenu`로 좁혀 줍니다. `PageRenderer`가 `PageMenu`만 받으니 딱 맞습니다.

라우트를 정리합니다. 이제 화면별로 `<Route>`를 쓰지 않고 **모든 주소를 `RoutedPage`가** 받습니다.

📄 **`src/app/App.tsx`** (전체 교체)

```tsx
import { Navigate, Route, Routes } from "react-router";
import MainLayout from "@/app/layout/MainLayout";
import RoutedPage from "@/app/layout/RoutedPage";

function App() {
  return (
    <Routes>
      <Route element={<MainLayout />}>
        <Route index element={<Navigate to="/dashboard" replace />} />
        {/* 화면별 라우트를 등록하지 않는다. 어떤 화면을 보여 줄지는 메뉴 데이터가 정한다. */}
        <Route path="*" element={<RoutedPage />} />
      </Route>
    </Routes>
  );
}

export default App;
```

홈 화면은 대시보드로 바뀝니다. 기존 `home` 폴더를 지우고 대시보드 화면을 만듭니다. (Part 13에서 차트를 넣습니다.)

▶️ **터미널**

```powershell
Remove-Item src/features/home -Recurse
```

📄 **`src/features/dashboard/pages/DashboardPage.tsx`** (새 파일)

```tsx
import { Card, Typography } from "antd";

function DashboardPage() {
  return (
    <Card>
      <Typography.Title level={4}>{import.meta.env.VITE_APP_TITLE}</Typography.Title>
      <Typography.Paragraph type="secondary">
        왼쪽 메뉴에서 화면을 선택하세요. (Part 13에서 대시보드를 완성합니다)
      </Typography.Paragraph>
    </Card>
  );
}

export default DashboardPage;
```

### 확인하기

- [ ] 대시보드, 시스템 관리 → 사용자 관리가 열립니다. 주소는 `/system/users`입니다. (메뉴 데이터의 `path`)
- [ ] 권한 관리, 메뉴 관리 등은 **"화면 준비 중"** 이 보입니다. 메뉴는 있지만 화면 파일이 아직 없기 때문입니다.
- [ ] F12 → Network → JS 필터를 켜고 새로고침한 뒤 사용자 관리를 **처음** 열면 `UsersPage.tsx` 파일을 따로 받습니다. (코드 분할)
- [ ] `npm run build` → `dist/assets`에 `UsersPage-xxxx.js`처럼 화면별 파일이 생깁니다.

### 스스로 답해 보기

<details>
<summary>Q. <code>getPage</code> 안의 <code>cache</code>가 없으면 어떤 일이 생길까요?</summary>

`lazy()`가 부를 때마다 **새 컴포넌트**를 만듭니다. React는 컴포넌트 종류가 바뀌면 이전 것을 버리고 새로 만들기 때문에,
화면이 다시 그려질 때마다 입력값·그리드 상태가 날아갑니다. Part 7의 탭에서는 치명적인 버그가 됩니다.

</details>

✅ **커밋**

```powershell
git add .
git commit -m "feat: 메뉴 데이터와 화면 파일 자동 연결"
```

---

## Step 6-4. 화면 공통 헤더 (제목·경로)

> 🎯 **할 일**: 화면 제목과 경로(브레드크럼)를 화면마다 쓰지 않고, 메뉴 데이터로 자동 표시합니다.

### 원리: 화면은 자기 메뉴를 "받는다"

화면 컴포넌트는 자기가 어느 메뉴에서 열렸는지 알아야 제목을 표시할 수 있습니다. 그런데 화면이 주소(`useLocation`)를 직접 보면 안 됩니다.
Part 7에서 탭이 생기면 **숨겨진 탭의 화면도 살아 있어서**, 주소를 보면 다른 탭의 주소를 읽게 되기 때문입니다.

그래서 화면을 그리는 쪽(`RoutedPage`, 나중에는 탭)이 **Context**로 메뉴 정보를 내려 줍니다.
Context는 "props를 일일이 전달하지 않고, 아래쪽 어디서든 꺼내 쓰는 값"입니다.

```
<PageContext value={{ menuCode: "USER_MGMT", params: {} }}>
  <UsersPage>
    <PageContainer>   ← usePage()로 menuCode를 꺼내 메뉴 이름을 찾는다
```

### 따라 하기

📄 **`src/shared/page/PageContext.ts`** (새 파일)

```ts
import { createContext, use } from "react";

export interface PageContextValue {
  menuCode: string; // 이 화면을 연 메뉴
  params: Record<string, string>; // 화면에 넘긴 값. 예: 상세 화면의 { id: "3" }
}

export const PageContext = createContext<PageContextValue | null>(null);

/** 화면이 자기 정보를 받는 훅. 주소(useLocation) 대신 이것을 쓴다. */
export function usePage() {
  const value = use(PageContext); // React 19: use()로 Context를 읽는다
  if (!value) throw new Error("usePage는 메뉴로 열린 화면 안에서만 쓸 수 있습니다.");
  return value;
}
```

✏️ **`src/app/layout/RoutedPage.tsx`** — 화면에 메뉴 정보를 내려 줍니다.

```tsx
  return <PageRenderer menu={menu} />;
```

바꿀 코드

```tsx
  const params = Object.fromEntries(new URLSearchParams(search));
  return (
    <PageContext value={{ menuCode: menu.code, params }}>
      <PageRenderer menu={menu} />
    </PageContext>
  );
```

✏️ **`src/app/layout/RoutedPage.tsx`** — 쿼리스트링도 읽습니다.

```tsx
  const { pathname } = useLocation();
```

바꿀 코드

```tsx
  const { pathname, search } = useLocation();
```

✏️ **`src/app/layout/RoutedPage.tsx`** — import를 추가합니다.

```tsx
import { useMyMenus } from "@/shared/menu/useMyMenus";
```

바꿀 코드

```tsx
import { useMyMenus } from "@/shared/menu/useMyMenus";
import { PageContext } from "@/shared/page/PageContext";
```

> 💡 **팁** React 19부터 `<PageContext.Provider value={...}>` 대신 `<PageContext value={...}>`로 바로 쓸 수 있습니다.

이제 공통 헤더 컴포넌트입니다.

📄 **`src/shared/ui/PageContainer.tsx`** (새 파일)

```tsx
import type { ReactNode } from "react";
import { Breadcrumb, Flex, Typography } from "antd";
import { useLocalizedText } from "@/shared/i18n/useLocalizedText";
import { getMenuChain } from "@/shared/menu/tree";
import { useMyMenus } from "@/shared/menu/useMyMenus";
import { usePage } from "@/shared/page/PageContext";

interface PageContainerProps {
  title?: string; // 메뉴 이름 대신 쓸 제목 (상세 화면 등)
  extra?: ReactNode; // 제목 오른쪽 버튼 영역
  children: ReactNode;
}

/** 모든 화면의 바깥 틀. 제목과 경로는 메뉴 데이터에서 자동으로 만든다. */
function PageContainer({ title, extra, children }: PageContainerProps) {
  const { menuCode } = usePage();
  const { data: menus = [] } = useMyMenus();
  const localize = useLocalizedText();

  const chain = getMenuChain(menus, menuCode);
  const current = chain.at(-1);

  return (
    <Flex vertical gap={12} style={{ height: "100%" }}>
      <div>
        <Breadcrumb items={chain.map((menu) => ({ title: localize(menu.names) }))} />
        <Flex align="center" justify="space-between" gap={8} style={{ marginTop: 4 }}>
          <Typography.Title level={4} style={{ margin: 0 }}>
            {title ?? (current ? localize(current.names) : "")}
          </Typography.Title>
          {extra}
        </Flex>
      </div>
      {children}
    </Flex>
  );
}

export default PageContainer;
```

사용자 화면에 적용합니다.

✏️ **`src/features/users/pages/UsersPage.tsx`** — 바깥 `div`를 `PageContainer`로 바꿉니다.

```tsx
    <div style={{ height: "100%", display: "flex", flexDirection: "column" }}>
      <Form form={form} layout="inline" onFinish={handleSearch} style={{ marginBottom: 16 }}>
```

바꿀 코드

```tsx
    <PageContainer>
      <Form form={form} layout="inline" onFinish={handleSearch}>
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — 닫는 태그도 바꿉니다.

```tsx
      <DataGrid<User> rowData={data} columnDefs={columnDefs} loading={isFetching} />
    </div>
```

바꿀 코드

```tsx
      <DataGrid<User> rowData={data} columnDefs={columnDefs} loading={isFetching} />
    </PageContainer>
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — import를 추가합니다.

```tsx
import DataGrid from "@/shared/ui/DataGrid";
```

바꿀 코드

```tsx
import DataGrid from "@/shared/ui/DataGrid";
import PageContainer from "@/shared/ui/PageContainer";
```

### 확인하기

- [ ] 사용자 관리 화면 위에 `시스템 관리 / 사용자 관리` 경로와 `사용자 관리` 제목이 보입니다.
- [ ] English로 바꾸면 `System / Users`로 바뀝니다. 화면 코드에는 제목 문구가 **한 글자도 없습니다.**

### 스스로 답해 보기

<details>
<summary>Q. 메뉴 이름을 바꾸려면 어디를 고쳐야 하나요?</summary>

코드가 아니라 **메뉴 데이터**입니다. 지금은 `mocks/data/menus.ts`를 고치면 되고, Part 10 이후에는 메뉴 관리 화면에서 바꾸면 됩니다.
사이드바·제목·경로·(Part 7의) 탭 이름이 전부 같은 데이터를 보기 때문에 한 번에 바뀝니다.

</details>

✅ **커밋**

```powershell
git add .
git commit -m "feat: 화면 공통 헤더 및 경로 표시 추가"
```

---

## 🏁 Part 6을 마치며

지금 새 화면을 추가하는 방법은 이렇습니다.

```
① src/features/<기능>/pages/<이름>Page.tsx 를 만든다 (default export)
② 메뉴 데이터에 component: "<기능>/<이름>Page" 를 넣는다
→ 사이드바·주소·제목·경로·코드 분할이 전부 자동
```

| 증상                                       | 원인과 해결                                                                    |
| ------------------------------------------ | ------------------------------------------------------------------------------ |
| 새로 만든 화면이 "준비 중"으로 나온다      | 폴더·이름 규칙(`features/*/pages/*Page.tsx`), default export, 메뉴의 `component` 철자 |
| 화면 파일 이름을 바꿨더니 "준비 중"        | 메뉴의 `component`는 문자열이라 자동으로 안 바뀐다 → 메뉴 데이터도 고친다       |
| 메뉴가 4단계 이상인데 안 보인다            | 의도된 동작. 최대 3단계 (`MAX_MENU_DEPTH`)                                     |

다음은 이 템플릿의 얼굴, 여러 화면을 탭으로 동시에 여는 MDI입니다. → [Part 7. MDI 탭](part-07-mdi-tabs.md)
