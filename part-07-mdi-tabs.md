# Part 7. MDI 탭

> **이 Part의 목표**: 메뉴를 누르면 탭으로 열리고, 여러 화면을 동시에 열어 오가도 입력값·그리드 정렬·스크롤이 그대로 남게 합니다.
> 홈이나 대시보드도 **닫을 수 있습니다.** 고정 탭은 없습니다. 이 템플릿의 얼굴이 되는 기능입니다.

---

## 먼저 큰 그림

MDI(Multiple Document Interface)는 한 화면 안에서 여러 업무 화면을 탭으로 동시에 여는 방식입니다. SI 업무 시스템의 표준 UI입니다.
설계의 핵심 아이디어는 세 가지입니다.

**① 열린 탭 목록은 저장소, 활성 탭은 주소(URL)**

같은 정보를 두 곳에 두면 언젠가 어긋납니다. 그래서 **활성 탭은 저장하지 않고 주소에서 계산합니다.** (단일 진실 공급원)

- 탭 클릭 → 주소 변경 → 그 탭이 활성
- 메뉴 클릭·주소 직접 입력·뒤로가기 → 주소 변경 → (없으면 열고) 그 탭이 활성

**② 열린 탭은 모두 그려 두고, 비활성 탭은 숨긴다 (keep-alive)**

화면을 없앴다 다시 만들면 state가 초기화됩니다. **숨기기만 하면** state가 남습니다.

**③ 탭 안의 화면은 자기 정보를 `usePage()`로 받는다**

숨겨진 탭도 살아 있으므로, 화면이 `useLocation()`을 보면 **다른 탭의 주소**를 읽게 됩니다. Part 6에서 미리 `usePage()`를 만들어 둔 이유입니다.

```
MainLayout
├─ SideMenu        클릭 → navigate(menu.path)
├─ TabBar          antd Tabs (탭 바만 그림)
│                  클릭 → navigate(tab.path) / 닫기 → closeTabs
└─ TabWorkspace    주소가 바뀌면: 메뉴 찾기 → 탭 없으면 열기  (useTabUrlSync)
   └─ TabPanels    열린 탭을 전부 그리고 활성 탭만 보이게
      └─ 탭마다 <div hidden?> → PageContext → (오류 경계) → PageRenderer
```

---

## Step 7-1. 탭 저장소

> 🎯 **할 일**: 열린 탭 목록과 탭 조작 함수를 담는 저장소를 만듭니다.

### 원리: 탭을 무엇으로 구별하나

같은 메뉴는 탭 하나만 엽니다. 그런데 공지사항 **상세**처럼 같은 메뉴라도 대상(`id`)이 다르면 탭을 따로 열고 싶습니다.
그래서 탭의 키는 **메뉴 코드 + 파라미터**입니다.

```
/system/users                 → "USER_MGMT"
/board/notices/view?id=3      → "NOTICE_VIEW?id=3"
/board/notices/view?id=7      → "NOTICE_VIEW?id=7"   (다른 탭)
```

**탭 제목은 저장하지 않습니다.** 언어를 바꾸거나 메뉴 이름을 고치면 탭 제목도 바뀌어야 하기 때문입니다.
제목은 그릴 때마다 `menuCode`로 메뉴 데이터에서 찾습니다. (파생 상태)

**sessionStorage에 저장합니다.** 새로고침에는 남고, 브라우저 탭을 닫으면 지워집니다. localStorage와의 차이입니다.

### 따라 하기

📄 **`src/shared/tabs/resolveTab.ts`** (새 파일)

```ts
import { findMenuByPath } from "@/shared/menu/tree";
import type { Menu } from "@/shared/menu/types";

/** 탭 키를 만든다. 같은 메뉴라도 파라미터가 다르면 다른 탭이다. 예: "NOTICE_VIEW?id=3" */
export function getTabKey(menuCode: string, params: Record<string, string>) {
  // 파라미터 순서가 달라도 같은 탭이 되도록 정렬한다. (?a=1&b=2 와 ?b=2&a=1)
  const entries = Object.entries(params).sort(([a], [b]) => a.localeCompare(b));
  const query = new URLSearchParams(entries).toString();
  return query ? `${menuCode}?${query}` : menuCode;
}

/** 주소를 탭 정보로 바꾼다. 화면 메뉴가 없는 주소면 undefined. */
export function resolveTab(menus: Menu[], pathname: string, search: string) {
  const menu = findMenuByPath(menus, pathname);
  if (menu?.type !== "PAGE") return undefined;
  const params = Object.fromEntries(new URLSearchParams(search));
  return { menu, params, key: getTabKey(menu.code, params), path: pathname + search };
}
```

📄 **`src/shared/tabs/tabStore.ts`** (새 파일)

```ts
import { create } from "zustand";
import { createJSONStorage, persist } from "zustand/middleware";

/** 동시에 열 수 있는 탭 수. 너무 많으면 숨은 화면들 때문에 느려진다. */
export const MAX_TABS = 10;

/** 로그인 직후, 그리고 열린 탭이 없을 때 안내할 시작 화면 */
export const START_MENU_CODE = "DASHBOARD";

export interface Tab {
  key: string; // 메뉴 코드 + 파라미터. 예: "NOTICE_VIEW?id=3"
  menuCode: string; // 제목·권한·화면 파일을 메뉴 데이터에서 찾는다
  path: string; // 탭을 클릭하면 이동할 주소. 예: "/board/notices/view?id=3"
  params: Record<string, string>; // 화면에 넘기는 값. 예: { id: "3" }
  customTitle?: string; // 화면이 정한 제목. 예: "공지 - 서버 점검 안내"
  refreshKey: number; // 이 값을 바꾸면 화면을 처음부터 다시 그린다 (탭 새로고침)
}

interface TabState {
  tabs: Tab[];
  dirtyKeys: string[]; // 저장하지 않은 변경이 있는 탭
  open: (tab: Omit<Tab, "refreshKey">) => void;
  close: (keys: string[]) => void;
  refresh: (key: string) => void;
  setTitle: (key: string, title: string) => void;
  setDirty: (key: string, dirty: boolean) => void;
}

export const useTabStore = create<TabState>()(
  persist(
    (set) => ({
      tabs: [],
      dirtyKeys: [],
      open: (tab) =>
        set((state) =>
          state.tabs.some((item) => item.key === tab.key)
            ? state // 이미 열려 있으면 그대로 둔다
            : { tabs: [...state.tabs, { ...tab, refreshKey: 0 }] },
        ),
      close: (keys) =>
        set((state) => ({
          tabs: state.tabs.filter((item) => !keys.includes(item.key)),
          dirtyKeys: state.dirtyKeys.filter((key) => !keys.includes(key)),
        })),
      refresh: (key) =>
        set((state) => ({
          tabs: state.tabs.map((item) =>
            item.key === key ? { ...item, refreshKey: item.refreshKey + 1 } : item,
          ),
          dirtyKeys: state.dirtyKeys.filter((dirtyKey) => dirtyKey !== key),
        })),
      setTitle: (key, customTitle) =>
        set((state) => ({
          tabs: state.tabs.map((item) => (item.key === key ? { ...item, customTitle } : item)),
        })),
      setDirty: (key, dirty) =>
        set((state) => {
          if (state.dirtyKeys.includes(key) === dirty) return state; // 바뀐 게 없으면 다시 그리지 않는다
          return {
            dirtyKeys: dirty
              ? [...state.dirtyKeys, key]
              : state.dirtyKeys.filter((dirtyKey) => dirtyKey !== key),
          };
        }),
    }),
    {
      name: "tabs",
      storage: createJSONStorage(() => sessionStorage), // 새로고침에는 남고, 브라우저 탭을 닫으면 지워진다
      partialize: (state) => ({ tabs: state.tabs }), // 탭 목록만 저장한다
    },
  ),
);
```

### 코드 읽어 보기

**`set((state) => ...)`**: 이전 상태를 받아서 새 상태를 만드는 방식입니다. 이전 값을 바탕으로 바꿀 때는 항상 이 형태를 씁니다.

**`return state`**: 바뀐 게 없으면 원래 객체를 그대로 돌려줍니다. 새 객체를 만들면 "바뀌었다"고 판단해서 쓸데없이 다시 그리기 때문입니다.

**`partialize`**: 저장할 부분만 고릅니다. `dirtyKeys`는 새로고침하면 의미가 없으므로 저장하지 않습니다.

**`refreshKey`**: Part 5에서 배운 `key` 기법입니다. 이 값을 화면 컴포넌트의 `key`로 넘기면, 값이 바뀔 때 화면이 처음부터 다시 만들어집니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 탭 저장소 추가"
```

---

## Step 7-2. 탭 열기·닫기·전환

> 🎯 **할 일**: 메뉴를 누르면 탭이 생기고, 탭을 오가도 화면 상태가 유지되게 합니다.

### 따라 하기

탭 안의 화면이 자기 탭 정보를 알 수 있도록 `PageContext`를 넓힙니다.

✏️ **`src/shared/page/PageContext.ts`** — 탭 정보를 추가합니다.

```ts
  params: Record<string, string>; // 화면에 넘긴 값. 예: 상세 화면의 { id: "3" }
}
```

바꿀 코드

```ts
  params: Record<string, string>; // 화면에 넘긴 값. 예: 상세 화면의 { id: "3" }
  tabKey: string; // 이 화면이 들어 있는 탭
  isActive: boolean; // 지금 보이는 탭인가. 숨은 탭에서 단축키·주기 조회를 막을 때 쓴다
}
```

현재 주소로 활성 탭 키를 계산하는 훅입니다.

📄 **`src/shared/tabs/useActiveTabKey.ts`** (새 파일)

```ts
import { useLocation } from "react-router";
import { useMyMenus } from "@/shared/menu/useMyMenus";
import { resolveTab } from "@/shared/tabs/resolveTab";

/** 활성 탭은 저장하지 않고 주소에서 계산한다. */
export function useActiveTabKey() {
  const { pathname, search } = useLocation();
  const { data: menus } = useMyMenus();
  return menus ? resolveTab(menus, pathname, search)?.key : undefined;
}
```

탭을 닫는 훅입니다. 보고 있던 탭을 닫으면 옆 탭으로 이동해야 하므로 단순히 저장소에서 빼는 것보다 할 일이 많습니다.

📄 **`src/shared/tabs/useCloseTabs.ts`** (새 파일)

```ts
import { useNavigate } from "react-router";
import { useActiveTabKey } from "@/shared/tabs/useActiveTabKey";
import { useTabStore, type Tab } from "@/shared/tabs/tabStore";

/** 탭을 닫는다. 보고 있던 탭을 닫으면 오른쪽 탭 → 왼쪽 탭 → 시작 화면 순서로 이동한다. */
export function useCloseTabs() {
  const navigate = useNavigate();
  const activeKey = useActiveTabKey();

  return (keys: string[]) => {
    const { tabs, close } = useTabStore.getState();
    close(keys);
    if (!activeKey || !keys.includes(activeKey)) return;

    const index = tabs.findIndex((tab) => tab.key === activeKey);
    const isOpen = (tab: Tab) => !keys.includes(tab.key);
    const next = tabs.slice(index + 1).find(isOpen) ?? tabs.slice(0, index).reverse().find(isOpen);
    navigate(next ? next.path : "/");
  };
}
```

> 💡 **팁** `useTabStore.getState()`는 훅이 아닙니다. 구독하지 않고 **지금 값만 한 번** 읽습니다. 클릭 처리처럼 "그 순간의 값"만 필요할 때 씁니다.

탭 바를 만듭니다. antd `Tabs`에는 화면을 넣지 않고 **탭 머리만** 그립니다. 화면은 `TabPanels`가 따로 그립니다.
그래야 탭마다 오류 경계·새로고침을 직접 제어할 수 있습니다.

📄 **`src/app/layout/TabBar.tsx`** (새 파일)

```tsx
import { useNavigate } from "react-router";
import { Tabs, theme } from "antd";
import { useLocalizedText } from "@/shared/i18n/useLocalizedText";
import { useMyMenus } from "@/shared/menu/useMyMenus";
import { useActiveTabKey } from "@/shared/tabs/useActiveTabKey";
import { useCloseTabs } from "@/shared/tabs/useCloseTabs";
import { useTabStore, type Tab } from "@/shared/tabs/tabStore";

function TabBar() {
  const navigate = useNavigate();
  const { token } = theme.useToken();
  const localize = useLocalizedText();
  const { data: menus = [] } = useMyMenus();
  const tabs = useTabStore((state) => state.tabs);
  const activeKey = useActiveTabKey();
  const closeTabs = useCloseTabs();

  if (tabs.length === 0) return null;

  // 제목은 저장하지 않고 그릴 때마다 메뉴 데이터에서 찾는다. 언어·메뉴 이름이 바뀌면 바로 반영된다.
  const titleOf = (tab: Tab) => {
    const menu = menus.find((item) => item.code === tab.menuCode);
    return tab.customTitle ?? (menu ? localize(menu.names) : tab.menuCode);
  };

  return (
    <Tabs
      type="editable-card"
      hideAdd
      size="small"
      activeKey={activeKey ?? ""}
      items={tabs.map((tab) => ({ key: tab.key, label: titleOf(tab) }))}
      onChange={(key) => {
        const tab = tabs.find((item) => item.key === key);
        if (tab) navigate(tab.path);
      }}
      onEdit={(key, action) => {
        if (action === "remove" && typeof key === "string") closeTabs([key]);
      }}
      tabBarStyle={{ margin: 0 }}
      style={{
        paddingInline: 8,
        paddingTop: 6,
        background: token.colorBgContainer,
        borderBottom: `1px solid ${token.colorBorderSecondary}`,
      }}
    />
  );
}

export default TabBar;
```

열린 탭을 전부 그리고 활성 탭만 보이게 하는 컴포넌트입니다.

📄 **`src/app/layout/TabPanels.tsx`** (새 파일)

```tsx
import PageRenderer from "@/app/layout/PageRenderer";
import { useMyMenus } from "@/shared/menu/useMyMenus";
import { PageContext } from "@/shared/page/PageContext";
import { useActiveTabKey } from "@/shared/tabs/useActiveTabKey";
import { useTabStore } from "@/shared/tabs/tabStore";

/** 열린 탭의 화면을 모두 그려 두고, 활성 탭만 보이게 한다. 그래서 탭을 오가도 화면 상태가 남는다. */
function TabPanels() {
  const tabs = useTabStore((state) => state.tabs);
  const activeKey = useActiveTabKey();
  const { data: menus = [] } = useMyMenus();

  return tabs.map((tab) => {
    const menu = menus.find((item) => item.code === tab.menuCode);
    if (menu?.type !== "PAGE") return null;
    const isActive = tab.key === activeKey;

    return (
      <div
        key={tab.key}
        // 탭마다 스크롤을 따로 가진다. 숨기는 것은 display: none
        style={{
          display: isActive ? "block" : "none",
          height: "100%",
          overflow: "auto",
          padding: 16,
          boxSizing: "border-box",
        }}
      >
        <PageContext
          value={{ menuCode: tab.menuCode, params: tab.params, tabKey: tab.key, isActive }}
        >
          <PageRenderer key={tab.refreshKey} menu={menu} />
        </PageContext>
      </div>
    );
  });
}

export default TabPanels;
```

주소가 바뀌면 탭을 여는 훅입니다.

📄 **`src/app/layout/useTabUrlSync.ts`** (새 파일)

```ts
import { useEffect } from "react";
import { useLocation } from "react-router";
import { useMyMenus } from "@/shared/menu/useMyMenus";
import { resolveTab } from "@/shared/tabs/resolveTab";
import { useTabStore } from "@/shared/tabs/tabStore";

/** 주소가 바뀌면 그 주소의 탭을 연다. 메뉴 클릭·주소 입력·뒤로가기가 모두 여기를 지난다. */
export function useTabUrlSync() {
  const { pathname, search } = useLocation();
  const { data: menus } = useMyMenus();

  useEffect(() => {
    if (!menus) return;
    const resolved = resolveTab(menus, pathname, search);
    if (!resolved) return; // "/"(시작 화면)이나 없는 주소
    useTabStore.getState().open({
      key: resolved.key,
      menuCode: resolved.menu.code,
      path: resolved.path,
      params: resolved.params,
    });
  }, [menus, pathname, search]);
}
```

탭 영역 전체입니다. 없는 주소면 안내를 보여 줍니다.

📄 **`src/app/layout/TabWorkspace.tsx`** (새 파일)

```tsx
import { useLocation } from "react-router";
import NotFoundPage from "@/app/errors/NotFoundPage";
import TabPanels from "@/app/layout/TabPanels";
import { useTabUrlSync } from "@/app/layout/useTabUrlSync";
import { useMyMenus } from "@/shared/menu/useMyMenus";
import { resolveTab } from "@/shared/tabs/resolveTab";
import PageLoading from "@/shared/ui/PageLoading";

function TabWorkspace() {
  const { pathname, search } = useLocation();
  const { data: menus } = useMyMenus();
  useTabUrlSync();

  if (!menus) return <PageLoading />;
  const isUnknown = pathname !== "/" && !resolveTab(menus, pathname, search);

  return (
    <>
      {isUnknown && <NotFoundPage />}
      <TabPanels />
    </>
  );
}

export default TabWorkspace;
```

레이아웃에 붙입니다. 이제 `Content`는 스크롤하지 않고, 탭마다 스크롤합니다.

📄 **`src/app/layout/MainLayout.tsx`** (전체 교체)

```tsx
import { useState } from "react";
import { Layout, theme } from "antd";
import AppHeader from "@/app/layout/AppHeader";
import SideMenu from "@/app/layout/SideMenu";
import TabBar from "@/app/layout/TabBar";
import TabWorkspace from "@/app/layout/TabWorkspace";

const { Sider, Content } = Layout;

function MainLayout() {
  const [collapsed, setCollapsed] = useState(false);
  const { token } = theme.useToken();

  return (
    <Layout style={{ height: "100%" }}>
      <AppHeader collapsed={collapsed} onToggleCollapsed={() => setCollapsed(!collapsed)} />
      <Layout>
        <Sider
          theme="light"
          width={220}
          collapsedWidth={56}
          collapsed={collapsed}
          style={{
            borderInlineEnd: `1px solid ${token.colorBorderSecondary}`,
            overflow: "auto",
          }}
        >
          <SideMenu />
        </Sider>
        {/* minWidth: 0 — 넓은 그리드가 레이아웃을 밀어내지 않게 한다 */}
        <Content style={{ display: "flex", flexDirection: "column", minWidth: 0 }}>
          <TabBar />
          <div style={{ flex: 1, minHeight: 0 }}>
            <TabWorkspace />
          </div>
        </Content>
      </Layout>
    </Layout>
  );
}

export default MainLayout;
```

라우트도 정리합니다. 모든 주소를 `MainLayout`이 받습니다. Part 6의 `RoutedPage`는 이제 필요 없습니다.

📄 **`src/app/App.tsx`** (전체 교체)

```tsx
import { Route, Routes } from "react-router";
import MainLayout from "@/app/layout/MainLayout";

function App() {
  return (
    <Routes>
      {/* 모든 주소를 MainLayout이 받는다. 어떤 화면을 보여 줄지는 메뉴 데이터와 탭이 정한다. */}
      <Route path="*" element={<MainLayout />} />
    </Routes>
  );
}

export default App;
```

▶️ **터미널**

```powershell
Remove-Item src/app/layout/RoutedPage.tsx
```

> ⚠️ **주의** 이전에는 `/`에 들어가면 대시보드로 이동했습니다. 이제 `/`는 **열린 탭이 없는 상태**입니다.
> `/`가 대시보드로 이동하면 대시보드 탭을 닫는 순간 다시 열려서, 사실상 닫을 수 없는 탭이 되기 때문입니다. (요구사항: 홈을 고정하지 말 것)
> 지금은 `/`가 빈 화면이고, Step 7-4에서 안내 화면을 만듭니다.

### 코드 읽어 보기: 숨긴 탭에서도 화면은 "살아 있다"

`display: none`은 **보이지만 않을 뿐** 컴포넌트는 그대로 있습니다. state도, `useEffect`도, 그리드 인스턴스도 살아 있습니다.
그래서 탭 안의 화면은 몇 가지 규칙을 지켜야 합니다.

| 규칙                                                        | 이유                                                                 |
| ----------------------------------------------------------- | -------------------------------------------------------------------- |
| `useLocation`, `useSearchParams`를 쓰지 않는다 → `usePage()` | 숨은 탭도 현재 주소(다른 탭의 주소)를 읽게 된다                      |
| 조회 조건은 화면 state에 둔다 (주소 쿼리 X)                 | 주소창은 하나뿐이다. 탭이 살아 있으므로 state가 유지된다             |
| 전역 단축키는 `isActive`일 때만 처리한다                    | 확인하지 않으면 단축키가 **모든 탭에서 동시에** 실행된다             |
| 주기적 조회(polling)는 `isActive`일 때만                    | 안 보이는 탭이 계속 서버를 부르지 않게 한다                          |

> 💡 **팁** React 19.2에는 숨기되 state를 유지하는 공식 기능 `<Activity mode="hidden">`이 있습니다. 숨길 때 Effect를 정리해 줘서
> 탭이 많을 때 유리하지만, 그리드·에디터처럼 Effect로 만든 외부 인스턴스가 다시 만들어질 수 있습니다. 이 템플릿은 동작이 가장 예측 가능한
> `display: none`으로 시작합니다. 나중에 직접 바꿔 보고 비교해 보는 것도 좋은 공부입니다.

### 확인하기

- [ ] 대시보드, 사용자 관리, 권한 관리를 차례로 누르면 탭이 3개 생깁니다.
- [ ] 사용자 관리에서 이름에 `김`을 넣고 조회하고, 그리드를 이메일로 정렬합니다. 다른 탭에 갔다 돌아오면 **그대로**입니다.
- [ ] 탭을 클릭하면 주소가 바뀌고, 사이드바 강조도 따라옵니다. 브라우저 뒤로가기를 누르면 이전 탭으로 갑니다.
- [ ] 보고 있는 탭의 ×를 누르면 오른쪽 탭(없으면 왼쪽)으로 이동합니다. 마지막 탭을 닫으면 빈 화면(`/`)이 됩니다.
- [ ] 주소창에 `/abc`를 입력하면 404 안내가 나옵니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: MDI 탭 열기·닫기·전환 추가"
```

---

## Step 7-3. 새로고침해도 탭 복원

> 🎯 **할 일**: 새로고침하면 탭이 그대로 복원되게 하고, 그사이 사라진 메뉴의 탭은 정리합니다.

### 원리

탭 목록은 이미 sessionStorage에 저장됩니다(`persist`). 그래서 새로고침해도 목록은 남습니다.
그런데 그사이 **메뉴가 삭제되거나 권한이 사라졌다면** 그 탭은 남아 있으면 안 됩니다. 메뉴 데이터를 받으면 내 메뉴에 없는 탭을 닫습니다.

### 따라 하기

✏️ **`src/app/layout/useTabUrlSync.ts`** — 정리하는 Effect를 추가합니다.

```ts
  }, [menus, pathname, search]);
}
```

바꿀 코드

```ts
  }, [menus, pathname, search]);

  // 메뉴가 바뀌면(새로고침 직후, 메뉴 삭제, 권한 변경) 더 이상 쓸 수 없는 메뉴의 탭을 닫는다.
  useEffect(() => {
    if (!menus) return;
    const pageCodes = new Set(
      menus.filter((menu) => menu.type === "PAGE").map((menu) => menu.code),
    );
    const { tabs, close } = useTabStore.getState();
    const stale = tabs.filter((tab) => !pageCodes.has(tab.menuCode)).map((tab) => tab.key);
    if (stale.length > 0) close(stale);
  }, [menus]);
}
```

### 확인하기

- [ ] 탭 3개를 열고 `F5` → 탭 3개가 그대로이고, 보던 탭이 활성입니다.
- [ ] F12 → Application → **Session Storage** → `tabs`에 목록이 보입니다.
- [ ] Session Storage의 `tabs` 값에서 `menuCode` 하나를 `"NO_SUCH_MENU"`로 고치고 새로고침하면 그 탭이 사라집니다.
- [ ] 브라우저 탭을 닫았다가 새로 열면 탭이 비어 있습니다. (sessionStorage의 특징)

✅ **커밋**

```powershell
git add .
git commit -m "feat: 새로고침 시 열린 탭 복원"
```

---

## Step 7-4. 열린 탭이 없을 때 시작 화면

> 🎯 **할 일**: 모든 탭을 닫았을 때 빈 화면 대신 안내를 보여 줍니다.

### 따라 하기

탭 관련 문구 파일을 만듭니다. 이 Part에서 쓸 문구를 한꺼번에 넣어 둡니다.

📄 **`src/shared/i18n/locales/ko/tabs.json`** (새 파일)

```json
{
  "empty": {
    "description": "열린 화면이 없습니다. 왼쪽 메뉴에서 화면을 선택하세요.",
    "open": "{{name}} 열기"
  },
  "menu": {
    "refresh": "새로고침",
    "close": "닫기",
    "closeOthers": "다른 탭 모두 닫기",
    "closeRight": "오른쪽 탭 모두 닫기",
    "closeAll": "모든 탭 닫기"
  },
  "maxReached": "탭은 최대 {{max}}개까지 열 수 있습니다. 사용하지 않는 탭을 닫아 주세요."
}
```

📄 **`src/shared/i18n/locales/en/tabs.json`** (새 파일)

```json
{
  "empty": {
    "description": "No pages are open. Select a page from the menu on the left.",
    "open": "Open {{name}}"
  },
  "menu": {
    "refresh": "Refresh",
    "close": "Close",
    "closeOthers": "Close others",
    "closeRight": "Close tabs to the right",
    "closeAll": "Close all"
  },
  "maxReached": "You can open up to {{max}} tabs. Please close unused tabs."
}
```

📄 **`src/app/layout/WorkspaceEmpty.tsx`** (새 파일)

```tsx
import { useTranslation } from "react-i18next";
import { useNavigate } from "react-router";
import { Button, Empty, Flex } from "antd";
import { useLocalizedText } from "@/shared/i18n/useLocalizedText";
import { useMyMenus } from "@/shared/menu/useMyMenus";
import { START_MENU_CODE } from "@/shared/tabs/tabStore";

/** 열린 탭이 없을 때(주소 "/") 보여 주는 안내. 시작 화면은 고정 탭이 아니라 바로가기 버튼으로 제공한다. */
function WorkspaceEmpty() {
  const { t } = useTranslation("tabs");
  const navigate = useNavigate();
  const localize = useLocalizedText();
  const { data: menus = [] } = useMyMenus();
  const start = menus.find((menu) => menu.code === START_MENU_CODE);

  return (
    <Flex align="center" justify="center" style={{ height: "100%" }}>
      <Empty description={t("empty.description")}>
        {start?.type === "PAGE" && (
          <Button type="primary" onClick={() => navigate(start.path)}>
            {t("empty.open", { name: localize(start.names) })}
          </Button>
        )}
      </Empty>
    </Flex>
  );
}

export default WorkspaceEmpty;
```

✏️ **`src/app/layout/TabWorkspace.tsx`** — 시작 화면을 넣습니다.

```tsx
      {isUnknown && <NotFoundPage />}
```

바꿀 코드

```tsx
      {pathname === "/" && <WorkspaceEmpty />}
      {isUnknown && <NotFoundPage />}
```

✏️ **`src/app/layout/TabWorkspace.tsx`** — import를 추가합니다.

```tsx
import { useTabUrlSync } from "@/app/layout/useTabUrlSync";
```

바꿀 코드

```tsx
import { useTabUrlSync } from "@/app/layout/useTabUrlSync";
import WorkspaceEmpty from "@/app/layout/WorkspaceEmpty";
```

### 확인하기

- [ ] 모든 탭을 닫으면 "열린 화면이 없습니다" 안내와 [대시보드 열기] 버튼이 보입니다.
- [ ] 대시보드 탭도 ×로 닫힙니다. 고정 탭이 없습니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 열린 탭이 없을 때 시작 안내 표시"
```

---

## Step 7-5. 탭 우클릭 메뉴와 최대 개수

> 🎯 **할 일**: 탭을 우클릭하면 새로고침·닫기 메뉴가 나오고, 휠 클릭으로 닫히며, 최대 10개까지만 열리게 합니다.

### 따라 하기

📄 **`src/app/layout/TabBar.tsx`** (전체 교체)

```tsx
import { useTranslation } from "react-i18next";
import { useNavigate } from "react-router";
import { Dropdown, Tabs, theme, type MenuProps } from "antd";
import { useLocalizedText } from "@/shared/i18n/useLocalizedText";
import { useMyMenus } from "@/shared/menu/useMyMenus";
import { useActiveTabKey } from "@/shared/tabs/useActiveTabKey";
import { useCloseTabs } from "@/shared/tabs/useCloseTabs";
import { useTabStore, type Tab } from "@/shared/tabs/tabStore";

function TabBar() {
  const { t } = useTranslation("tabs");
  const navigate = useNavigate();
  const { token } = theme.useToken();
  const localize = useLocalizedText();
  const { data: menus = [] } = useMyMenus();
  const tabs = useTabStore((state) => state.tabs);
  const refresh = useTabStore((state) => state.refresh);
  const activeKey = useActiveTabKey();
  const closeTabs = useCloseTabs();

  if (tabs.length === 0) return null;

  // 제목은 저장하지 않고 그릴 때마다 메뉴 데이터에서 찾는다. 언어·메뉴 이름이 바뀌면 바로 반영된다.
  const titleOf = (tab: Tab) => {
    const menu = menus.find((item) => item.code === tab.menuCode);
    return tab.customTitle ?? (menu ? localize(menu.names) : tab.menuCode);
  };

  const contextMenu = (tab: Tab, index: number): MenuProps => ({
    items: [
      { key: "refresh", label: t("menu.refresh") },
      { type: "divider" },
      { key: "close", label: t("menu.close") },
      { key: "closeOthers", label: t("menu.closeOthers"), disabled: tabs.length === 1 },
      { key: "closeRight", label: t("menu.closeRight"), disabled: index === tabs.length - 1 },
      { key: "closeAll", label: t("menu.closeAll") },
    ],
    onClick: ({ key, domEvent }) => {
      // 메뉴는 화면에서는 탭 밖(포털)에 그려지지만, React 이벤트는 컴포넌트 트리를 따라 탭까지 올라간다.
      // 막지 않으면 "탭을 클릭했다"로도 처리되어, 방금 닫은 탭으로 이동하면서 다시 열린다.
      domEvent.stopPropagation();
      const keysOf = (list: Tab[]) => list.map((item) => item.key);
      if (key === "refresh") refresh(tab.key);
      if (key === "close") closeTabs([tab.key]);
      if (key === "closeOthers") closeTabs(keysOf(tabs.filter((item) => item.key !== tab.key)));
      if (key === "closeRight") closeTabs(keysOf(tabs.slice(index + 1)));
      if (key === "closeAll") closeTabs(keysOf(tabs));
    },
  });

  return (
    <Tabs
      type="editable-card"
      hideAdd
      size="small"
      activeKey={activeKey ?? ""}
      items={tabs.map((tab, index) => ({
        key: tab.key,
        label: (
          <Dropdown trigger={["contextMenu"]} menu={contextMenu(tab, index)}>
            <span
              // 휠 클릭(가운데 버튼)으로 닫기. 기본 동작(자동 스크롤)은 막는다.
              onMouseDown={(event) => event.button === 1 && event.preventDefault()}
              onAuxClick={(event) => event.button === 1 && closeTabs([tab.key])}
            >
              {titleOf(tab)}
            </span>
          </Dropdown>
        ),
      }))}
      onChange={(key) => {
        const tab = tabs.find((item) => item.key === key);
        if (tab) navigate(tab.path);
      }}
      onEdit={(key, action) => {
        if (action === "remove" && typeof key === "string") closeTabs([key]);
      }}
      tabBarStyle={{ margin: 0 }}
      style={{
        paddingInline: 8,
        paddingTop: 6,
        background: token.colorBgContainer,
        borderBottom: `1px solid ${token.colorBorderSecondary}`,
      }}
    />
  );
}

export default TabBar;
```

이제 최대 개수입니다. 탭을 여는 곳은 `useTabUrlSync` 한 곳뿐이므로 거기서만 막으면 됩니다.
넘치면 알림을 띄우고 **주소를 원래 보던 탭으로 되돌립니다.**

📄 **`src/app/layout/useTabUrlSync.ts`** (전체 교체)

```ts
import { useEffect, useRef } from "react";
import { useTranslation } from "react-i18next";
import { useLocation, useNavigate } from "react-router";
import { App } from "antd";
import { useMyMenus } from "@/shared/menu/useMyMenus";
import { resolveTab } from "@/shared/tabs/resolveTab";
import { MAX_TABS, useTabStore } from "@/shared/tabs/tabStore";

/** 주소가 바뀌면 그 주소의 탭을 연다. 메뉴 클릭·주소 입력·뒤로가기가 모두 여기를 지난다. */
export function useTabUrlSync() {
  const { t } = useTranslation("tabs");
  const { pathname, search } = useLocation();
  const navigate = useNavigate();
  const { message } = App.useApp();
  const { data: menus } = useMyMenus();
  const lastPath = useRef("/"); // 마지막으로 정상 표시된 주소. 탭을 못 열 때 여기로 되돌린다.

  useEffect(() => {
    if (!menus) return;
    const resolved = resolveTab(menus, pathname, search);
    if (!resolved) {
      if (pathname === "/") lastPath.current = "/";
      return; // "/"(시작 화면)이나 없는 주소
    }
    const { tabs, open } = useTabStore.getState();
    const isNew = !tabs.some((tab) => tab.key === resolved.key);
    if (isNew && tabs.length >= MAX_TABS) {
      message.warning(t("maxReached", { max: MAX_TABS }));
      navigate(lastPath.current, { replace: true });
      return;
    }
    open({
      key: resolved.key,
      menuCode: resolved.menu.code,
      path: resolved.path,
      params: resolved.params,
    });
    lastPath.current = resolved.path;
  }, [menus, pathname, search, message, navigate, t]);

  // 메뉴가 바뀌면(새로고침 직후, 메뉴 삭제, 권한 변경) 더 이상 쓸 수 없는 메뉴의 탭을 닫는다.
  useEffect(() => {
    if (!menus) return;
    const pageCodes = new Set(
      menus.filter((menu) => menu.type === "PAGE").map((menu) => menu.code),
    );
    const { tabs, close } = useTabStore.getState();
    const stale = tabs.filter((tab) => !pageCodes.has(tab.menuCode)).map((tab) => tab.key);
    if (stale.length > 0) close(stale);
  }, [menus]);
}
```

### 코드 읽어 보기

**`domEvent.stopPropagation()`**: 우클릭 메뉴(Dropdown)는 화면 맨 위층(포털)에 따로 그려지지만, React의 이벤트는 **DOM 위치가 아니라 컴포넌트 트리**를 따라 부모로 올라갑니다.
메뉴가 탭 제목 안에 선언되어 있으므로, 메뉴 클릭이 탭 클릭으로도 전달됩니다. 그러면 다른 탭을 우클릭해 [닫기]를 눌렀을 때 탭이 닫히자마자 그 탭으로 이동해서 **다시 열립니다.**
그래서 메뉴 클릭에서 전파를 멈춥니다. 포털로 그려지는 모달·드롭다운 안의 클릭이 바깥 부모에 영향을 주면 이것부터 의심하세요.

**`useRef`**: 값을 기억하지만, 바꿔도 다시 그리지 않는 상자입니다. "마지막 정상 주소"는 화면에 보여 줄 값이 아니라 기억만 하면 되므로 state가 아니라 ref입니다.

**`navigate(path, { replace: true })`**: 기록을 쌓지 않고 현재 기록을 바꿉니다. 그래야 뒤로가기를 눌렀을 때 실패한 주소로 다시 가지 않습니다.

**새로고침(`refresh`)**: 저장소의 `refreshKey`를 1 올립니다. `TabPanels`가 이 값을 `PageRenderer`의 `key`로 넘기므로, 화면이 처음부터 다시 만들어집니다.

### 확인하기

- [ ] 탭 우클릭 → 메뉴가 나옵니다. "다른 탭 모두 닫기", "오른쪽 탭 모두 닫기"가 동작합니다.
- [ ] **보고 있지 않은** 탭을 우클릭해 [닫기] → 그 탭만 닫히고 다시 열리지 않습니다. [모든 탭 닫기] → 시작 안내 화면이 됩니다.
- [ ] 사용자 관리에서 조회 조건을 넣고 탭 우클릭 → 새로고침 → 조건이 초기화됩니다.
- [ ] 탭을 휠 클릭하면 닫힙니다.
- [ ] Step 7-3처럼 Session Storage를 고쳐 탭 10개를 만드는 대신, `tabStore.ts`의 `MAX_TABS`를 잠시 `3`으로 바꿔 보세요.
      4번째 메뉴를 누르면 알림이 뜨고 주소가 원래대로 돌아옵니다. 확인했으면 `10`으로 되돌립니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 탭 우클릭 메뉴 및 최대 개수 제한 추가"
```

---

## Step 7-6. 탭별 오류 화면

> 🎯 **할 일**: 한 탭의 화면에서 오류가 나도 그 탭만 오류 화면이 되고, 다른 탭과 앱 전체는 멀쩡하게 합니다.

### 원리: Error Boundary

React에서 렌더링 중 오류가 나면 **앱 전체가 하얗게** 사라집니다. Error Boundary(오류 경계)는 그 아래에서 난 오류를 잡아 대체 화면을 보여 줍니다.
탭마다 하나씩 두면 오류가 그 탭 안에 갇힙니다.

| 잡는 오류                    | 못 잡는 오류                                          |
| ---------------------------- | ----------------------------------------------------- |
| 렌더링 중 오류 (`throw`)     | 이벤트 핸들러(클릭 등) 안의 오류                      |
| `lazy` 화면 파일 로딩 실패   | `setTimeout`, `fetch` 같은 비동기 코드 안의 오류      |

못 잡는 오류는 Part 3의 전역 오류 알림이나 `try/catch`로 처리합니다.

React는 오류 경계를 아직 클래스 컴포넌트로만 제공합니다. `react-error-boundary`는 이를 함수형으로 편하게 쓰게 해 주는 작은 라이브러리입니다.

### 따라 하기

▶️ **터미널**

```powershell
npm install react-error-boundary@6.1.6
```

✅ **커밋**

```powershell
git add .
git commit -m "chore: react-error-boundary 의존성 추가"
```

📄 **`src/app/errors/TabErrorFallback.tsx`** (새 파일)

```tsx
import { useTranslation } from "react-i18next";
import { Button, Result } from "antd";
import type { FallbackProps } from "react-error-boundary";

/** 탭 안의 화면에서 렌더링 오류가 났을 때 그 탭 자리에 보여 준다. */
function TabErrorFallback({ error, resetErrorBoundary }: FallbackProps) {
  const { t } = useTranslation("errors");

  return (
    <Result
      status="error"
      title={t("page.crashed")}
      // 개발 중에만 오류 내용을 보여 준다. 운영에서는 사용자에게 내부 정보를 노출하지 않는다.
      subTitle={import.meta.env.DEV && error instanceof Error ? error.message : undefined}
      extra={<Button onClick={resetErrorBoundary}>{t("page.retry")}</Button>}
    />
  );
}

export default TabErrorFallback;
```

✏️ **`src/app/layout/TabPanels.tsx`** — 화면을 오류 경계로 감쌉니다.

```tsx
          <PageRenderer key={tab.refreshKey} menu={menu} />
```

바꿀 코드

```tsx
          {/* 탭 새로고침(refreshKey 변경) 시 오류 상태도 초기화한다 */}
          <ErrorBoundary FallbackComponent={TabErrorFallback} resetKeys={[tab.refreshKey]}>
            <PageRenderer key={tab.refreshKey} menu={menu} />
          </ErrorBoundary>
```

✏️ **`src/app/layout/TabPanels.tsx`** — import를 추가합니다.

```tsx
import PageRenderer from "@/app/layout/PageRenderer";
```

바꿀 코드

```tsx
import { ErrorBoundary } from "react-error-boundary";
import TabErrorFallback from "@/app/errors/TabErrorFallback";
import PageRenderer from "@/app/layout/PageRenderer";
```

### 확인하기

일부러 오류를 내 봅니다. `DashboardPage.tsx`의 `function DashboardPage() {` 바로 아래에 잠깐 이 줄을 넣습니다.

```tsx
  throw new Error("일부러 낸 오류");
```

- [ ] 대시보드 탭은 오류 화면, 다른 탭은 정상입니다. 헤더·사이드바도 멀쩡합니다.
- [ ] 넣었던 줄을 지우고 저장 → [다시 시도]를 누르면 대시보드가 정상으로 돌아옵니다.

> 💡 **팁** 개발 모드에서는 오류 경계가 잡은 오류도 콘솔에 빨갛게 찍힙니다. 정상입니다. 운영 빌드에서는 조용합니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 탭별 오류 화면 분리"
```

---

## Step 7-7. 화면에서 다른 화면 열기·제목 바꾸기

> 🎯 **할 일**: 화면 코드가 탭을 다룰 때 쓸 도구 훅을 만듭니다.

### 왜 필요한가요?

공지 목록에서 행을 더블클릭하면 상세 화면을 **새 탭으로** 열어야 합니다. 주소를 직접 조립하면(`navigate("/board/notices/view?id=3")`)
메뉴 관리에서 주소를 바꿨을 때 깨집니다. **메뉴 코드로 열면** 주소가 바뀌어도 안전합니다.

```ts
const openPage = useOpenPage();
openPage("NOTICE_VIEW", { id: "3" }); // 탭 "NOTICE_VIEW?id=3"이 열린다
```

상세 탭의 제목도 "공지사항 상세" 대신 "공지 - 서버 점검 안내"처럼 바꾸고 싶습니다. 저장 후에는 자기 탭을 닫고 싶기도 합니다.

### 따라 하기

📄 **`src/shared/page/useOpenPage.ts`** (새 파일)

```ts
import { useCallback } from "react";
import { useNavigate } from "react-router";
import { useMyMenus } from "@/shared/menu/useMyMenus";

/** 메뉴 코드로 화면을 연다. 주소를 직접 조립하지 않으므로 메뉴의 주소가 바뀌어도 안전하다. */
export function useOpenPage() {
  const navigate = useNavigate();
  const { data: menus = [] } = useMyMenus();

  return useCallback(
    (menuCode: string, params: Record<string, string> = {}) => {
      const menu = menus.find((item) => item.code === menuCode);
      if (menu?.type !== "PAGE") return false; // 메뉴가 없거나 권한이 없다
      const query = new URLSearchParams(params).toString();
      navigate(query ? `${menu.path}?${query}` : menu.path);
      return true;
    },
    [menus, navigate],
  );
}
```

📄 **`src/shared/page/usePageTitle.ts`** (새 파일)

```ts
import { useEffect } from "react";
import { usePage } from "@/shared/page/PageContext";
import { useTabStore } from "@/shared/tabs/tabStore";

/** 이 화면이 들어 있는 탭의 제목을 바꾼다. title이 비어 있으면 메뉴 이름을 그대로 쓴다. */
export function usePageTitle(title?: string) {
  const { tabKey } = usePage();
  const setTitle = useTabStore((state) => state.setTitle);

  useEffect(() => {
    if (title) setTitle(tabKey, title);
  }, [tabKey, title, setTitle]);
}
```

📄 **`src/shared/page/useClosePage.ts`** (새 파일)

```ts
import { usePage } from "@/shared/page/PageContext";
import { useCloseTabs } from "@/shared/tabs/useCloseTabs";

/** 이 화면이 들어 있는 탭을 닫는 함수를 돌려준다. (저장 후 닫기 등) */
export function useClosePage() {
  const { tabKey } = usePage();
  const closeTabs = useCloseTabs();
  return () => closeTabs([tabKey]);
}
```

시작 화면도 새 훅을 쓰도록 고칩니다. 같은 일을 하는 코드가 한 곳으로 모입니다.

📄 **`src/app/layout/WorkspaceEmpty.tsx`** (전체 교체)

```tsx
import { useTranslation } from "react-i18next";
import { Button, Empty, Flex } from "antd";
import { useLocalizedText } from "@/shared/i18n/useLocalizedText";
import { useMyMenus } from "@/shared/menu/useMyMenus";
import { useOpenPage } from "@/shared/page/useOpenPage";
import { START_MENU_CODE } from "@/shared/tabs/tabStore";

/** 열린 탭이 없을 때(주소 "/") 보여 주는 안내. 시작 화면은 고정 탭이 아니라 바로가기 버튼으로 제공한다. */
function WorkspaceEmpty() {
  const { t } = useTranslation("tabs");
  const localize = useLocalizedText();
  const openPage = useOpenPage();
  const { data: menus = [] } = useMyMenus();
  const start = menus.find((menu) => menu.code === START_MENU_CODE);

  return (
    <Flex align="center" justify="center" style={{ height: "100%" }}>
      <Empty description={t("empty.description")}>
        {start && (
          <Button type="primary" onClick={() => openPage(start.code)}>
            {t("empty.open", { name: localize(start.names) })}
          </Button>
        )}
      </Empty>
    </Flex>
  );
}

export default WorkspaceEmpty;
```

### 확인하기

- [ ] 시작 화면의 [대시보드 열기]가 이전처럼 동작합니다.
- [ ] `useOpenPage`, `usePageTitle`, `useClosePage`는 Part 11 공지사항에서 본격적으로 씁니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 화면에서 탭을 다루는 훅 추가"
```

---

## Step 7-8. 저장하지 않은 변경이 있으면 닫기 전에 확인

> 🎯 **할 일**: 작성 중인 화면의 탭을 닫거나 브라우저를 닫으려 하면 확인을 받습니다.

### 따라 하기

화면이 "나 지금 수정 중이야"라고 탭에 알리는 훅입니다.

📄 **`src/shared/tabs/useTabDirty.ts`** (새 파일)

```ts
import { useEffect } from "react";
import { usePage } from "@/shared/page/PageContext";
import { useTabStore } from "@/shared/tabs/tabStore";

/** 저장하지 않은 변경이 있음을 탭에 알린다. 탭 제목 앞에 ●가 붙고, 닫을 때 확인을 받는다. */
export function useTabDirty(dirty: boolean) {
  const { tabKey } = usePage();
  const setDirty = useTabStore((state) => state.setDirty);

  useEffect(() => {
    setDirty(tabKey, dirty);
    return () => setDirty(tabKey, false); // 화면이 사라지면 표시도 지운다
  }, [tabKey, dirty, setDirty]);
}
```

닫을 때 확인 창을 띄웁니다. 저장 직후 닫을 때처럼 확인 없이 닫아야 할 때를 위해 `force` 옵션도 둡니다.

📄 **`src/shared/tabs/useCloseTabs.ts`** (전체 교체)

```ts
import { useTranslation } from "react-i18next";
import { useNavigate } from "react-router";
import { App } from "antd";
import { useActiveTabKey } from "@/shared/tabs/useActiveTabKey";
import { useTabStore, type Tab } from "@/shared/tabs/tabStore";

/** 탭을 닫는다. 보고 있던 탭을 닫으면 오른쪽 탭 → 왼쪽 탭 → 시작 화면 순서로 이동한다. */
export function useCloseTabs() {
  const { t } = useTranslation();
  const navigate = useNavigate();
  const { modal } = App.useApp();
  const activeKey = useActiveTabKey();

  return (keys: string[], options: { force?: boolean } = {}) => {
    const { tabs, dirtyKeys, close } = useTabStore.getState();

    const run = () => {
      close(keys);
      if (!activeKey || !keys.includes(activeKey)) return;
      const index = tabs.findIndex((tab) => tab.key === activeKey);
      const isOpen = (tab: Tab) => !keys.includes(tab.key);
      const next =
        tabs.slice(index + 1).find(isOpen) ?? tabs.slice(0, index).reverse().find(isOpen);
      navigate(next ? next.path : "/");
    };

    // 닫을 탭 중 하나라도 저장 안 한 변경이 있으면 확인을 받는다.
    if (!options.force && keys.some((key) => dirtyKeys.includes(key))) {
      modal.confirm({ title: t("message.unsavedChanges"), onOk: run });
      return;
    }
    run();
  };
}
```

✏️ **`src/shared/page/useClosePage.ts`** — `force` 옵션을 넘길 수 있게 합니다.

```ts
  return () => closeTabs([tabKey]);
```

바꿀 코드

```ts
  return (options?: { force?: boolean }) => closeTabs([tabKey], options);
```

탭 제목 앞에 ●를 붙입니다.

✏️ **`src/app/layout/TabBar.tsx`** — 변경 표시를 넣습니다.

```tsx
              {titleOf(tab)}
```

바꿀 코드

```tsx
              {dirtyKeys.includes(tab.key) && (
                <span style={{ color: token.colorWarning, marginInlineEnd: 4 }}>●</span>
              )}
              {titleOf(tab)}
```

✏️ **`src/app/layout/TabBar.tsx`** — 값을 읽습니다.

```tsx
  const refresh = useTabStore((state) => state.refresh);
```

바꿀 코드

```tsx
  const refresh = useTabStore((state) => state.refresh);
  const dirtyKeys = useTabStore((state) => state.dirtyKeys);
```

마지막으로 브라우저 탭을 닫거나 새로고침할 때도 경고합니다. 이것은 브라우저 기본 확인 창이라 문구를 바꿀 수 없습니다.

✏️ **`src/app/layout/TabWorkspace.tsx`** — 브라우저 닫기 경고를 추가합니다.

```tsx
  useTabUrlSync();
```

바꿀 코드

```tsx
  useTabUrlSync();

  // 저장 안 한 탭이 있으면 브라우저 탭 닫기·새로고침 전에 경고한다.
  const hasDirty = useTabStore((state) => state.dirtyKeys.length > 0);
  useEffect(() => {
    if (!hasDirty) return;
    const warn = (event: BeforeUnloadEvent) => event.preventDefault();
    window.addEventListener("beforeunload", warn);
    return () => window.removeEventListener("beforeunload", warn);
  }, [hasDirty]);
```

✏️ **`src/app/layout/TabWorkspace.tsx`** — import를 추가합니다.

```tsx
import { useLocation } from "react-router";
```

바꿀 코드

```tsx
import { useEffect } from "react";
import { useLocation } from "react-router";
```

✏️ **`src/app/layout/TabWorkspace.tsx`** — import를 추가합니다.

```tsx
import { resolveTab } from "@/shared/tabs/resolveTab";
```

바꿀 코드

```tsx
import { resolveTab } from "@/shared/tabs/resolveTab";
import { useTabStore } from "@/shared/tabs/tabStore";
```

### 코드 읽어 보기: Effect의 정리 함수

`useEffect`가 돌려주는 함수는 **정리(cleanup) 함수**입니다. 컴포넌트가 사라지거나 의존성이 바뀌어 Effect가 다시 실행되기 직전에 불립니다.
이벤트 리스너를 등록했다면 정리 함수에서 반드시 해제합니다. 안 그러면 리스너가 계속 쌓입니다.

### 확인하기

실제 사용은 Part 11 공지 작성 화면입니다. 지금은 잠깐 시험해 봅니다. `UsersPage.tsx`의 `const { t } = useTranslation("users");` 아래에 넣습니다.

```tsx
  useTabDirty(true); // 맨 위 import에 import { useTabDirty } from "@/shared/tabs/useTabDirty"; 추가
```

- [ ] 사용자 관리 탭 제목 앞에 주황색 ●가 붙습니다.
- [ ] ×를 누르면 확인 창이 뜨고, [취소]하면 그대로, [확인]하면 닫힙니다.
- [ ] 사용자 관리 탭이 열린 상태에서 `F5` → 브라우저가 "사이트에서 나가시겠습니까?"를 묻습니다.
- [ ] 확인했으면 넣었던 두 줄을 지웁니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 저장하지 않은 변경이 있는 탭 닫기 확인"
```

---

## 🏁 Part 7을 마치며

마일스톤입니다. 여러 화면을 탭으로 동시에 열고, 오가도 입력값이 그대로입니다. 화면을 캡처해 두세요.

| 증상                                               | 원인과 해결                                                                    |
| -------------------------------------------------- | ------------------------------------------------------------------------------ |
| 탭을 오가면 화면이 초기화된다                      | 숨은 탭을 언마운트했다 / 렌더 중에 `lazy()`를 새로 만든다                      |
| 숨은 탭의 화면이 엉뚱한 파라미터를 읽는다          | `useSearchParams`를 썼다 → `usePage().params`                                  |
| 단축키를 누르면 여러 탭이 동시에 반응한다          | 전역 단축키에서 `usePage().isActive` 확인 누락                                 |
| 탭 새로고침이 안 된다                              | `refreshKey`를 화면 컴포넌트의 `key`로 안 넘겼다                               |
| 대시보드 탭이 닫히지 않는다                        | `/`가 대시보드로 이동하게 되어 있다 → `/`는 시작 안내 화면                     |

다음은 로그인입니다. 사람마다 다른 메뉴와 버튼이 보이게 합니다. → [Part 8. 로그인과 권한](part-08-auth-permission.md)
