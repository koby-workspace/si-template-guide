# Part 4. 테마와 라이트/다크

> **이 Part의 목표**: 버튼 하나로 앱 전체가 어두워지고, 테마를 고르면 색·모서리·밀도가 통째로 바뀌게 합니다.
> antd뿐 아니라 AG Grid와 우리가 직접 만든 영역까지 함께 바뀝니다. 눈에 보이는 변화가 커서 재미있는 구간입니다.

---

## Step 4-1. Zustand와 사용자 설정 저장소

> 🎯 **할 일**: 여러 화면이 함께 쓰고, 새로고침해도 남는 값을 저장할 곳을 만듭니다.

### 왜 Zustand인가요?

테마 설정은 헤더, 설정 패널, 그리드 등 **여러 곳에서** 읽습니다. 컴포넌트끼리 props로 넘기면 끝이 없습니다.
이런 값을 두는 곳이 **전역 상태 저장소**입니다.

| 도구              | 특징                                                     |
| ----------------- | -------------------------------------------------------- |
| **Zustand (채택)** | 코드가 아주 적고 Provider가 필요 없습니다. `persist`로 저장까지 한 줄 |
| Context           | 내장 기능. 값이 바뀌면 쓰는 곳이 전부 다시 그려져서 잦은 변경에 약합니다 |
| Redux Toolkit     | 규모가 크고 SI 현장에 많지만, 코드가 많습니다            |

### 따라 하기

▶️ **터미널**

```powershell
npm install zustand@5.1.0
```

✅ **커밋**

```powershell
git add .
git commit -m "chore: zustand 의존성 추가"
```

📄 **`src/shared/preferences/preferencesStore.ts`** (새 파일)

```ts
import { create } from "zustand";
import { persist } from "zustand/middleware";

export type ThemeMode = "light" | "dark" | "system";

interface PreferencesState {
  mode: ThemeMode;
  setMode: (mode: ThemeMode) => void;
}

export const usePreferences = create<PreferencesState>()(
  // persist: 값이 바뀔 때마다 localStorage에 저장하고, 시작할 때 복원한다.
  persist(
    (set) => ({
      mode: "light",
      setMode: (mode) => set({ mode }),
    }),
    { name: "preferences" }, // localStorage 키 이름
  ),
);
```

### 코드 읽어 보기

`create`는 **훅을 만들어 주는 함수**입니다. 쓰는 쪽은 이렇게 됩니다.

```tsx
const mode = usePreferences((state) => state.mode); // 필요한 값만 골라서 구독
```

괄호 안의 함수를 **selector**라고 합니다. `mode`가 바뀔 때만 이 컴포넌트가 다시 그려지고, 다른 값이 바뀌어도 영향이 없습니다.

> ⚠️ **주의** selector 없이 `usePreferences()`로 통째로 받으면 저장소의 어떤 값이 바뀌어도 다시 그려집니다. 항상 필요한 값만 고릅니다.

> 💡 **팁** `set({ mode })`는 `set({ mode: mode })`의 줄임입니다. 변수 이름과 키 이름이 같으면 한 번만 씁니다.

기본값을 `"light"`로 둔 이유: SI 고객사의 매뉴얼·캡처는 대부분 라이트 기준입니다. 프로젝트에 따라 `"system"`으로 바꿔도 됩니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 사용자 환경설정 저장소 추가"
```

---

## Step 4-2. 라이트/다크 전환

> 🎯 **할 일**: 헤더 버튼으로 앱 전체를 어둡게 바꿉니다.

### 원리: 디자인 토큰과 알고리즘

**토큰**은 디자인 값에 붙인 이름입니다. `colorPrimary`(대표 색), `borderRadius`(모서리) 같은 것입니다.
antd는 몇 개의 토큰만 받으면 나머지 수백 가지 색·간격을 **자동으로 계산**합니다.

| 단계      | 예                                         | 누가 정하나      |
| --------- | ------------------------------------------ | ---------------- |
| 시드 토큰 | `colorPrimary: "#1677ff"`, `borderRadius: 6` | 테마 프리셋      |
| 알고리즘  | `darkAlgorithm`, `compactAlgorithm`        | 모드, 밀도 설정  |
| 파생 토큰 | `colorBgContainer`, `colorText`, 버튼 hover 색 | antd가 자동 계산 |

Part 2에서 헤더 색을 `#fff` 대신 `token.colorBgContainer`로 쓴 이유가 여기 있습니다. 토큰을 쓰면 다크 모드에서 저절로 어두워집니다.

### 따라 하기

테마 관련 Provider를 `AppProviders`에서 분리합니다.

📄 **`src/shared/theme/AppThemeProvider.tsx`** (새 파일)

```tsx
import { useEffect, type ReactNode } from "react";
import { App as AntdApp, ConfigProvider, theme } from "antd";
import koKR from "antd/locale/ko_KR";
import { usePreferences } from "@/shared/preferences/preferencesStore";
import AntdStaticBridge from "@/shared/ui/AntdStaticBridge";

interface AppThemeProviderProps {
  children: ReactNode;
}

function AppThemeProvider({ children }: AppThemeProviderProps) {
  const mode = usePreferences((state) => state.mode);

  // 스크롤바 같은 브라우저 기본 요소도 모드를 따르게 한다.
  useEffect(() => {
    document.documentElement.dataset.theme = mode;
    document.documentElement.style.colorScheme = mode;
  }, [mode]);

  return (
    <ConfigProvider
      locale={koKR}
      theme={{ algorithm: mode === "dark" ? theme.darkAlgorithm : theme.defaultAlgorithm }}
    >
      <AntdApp style={{ height: "100%" }}>
        <AntdStaticBridge />
        {children}
      </AntdApp>
    </ConfigProvider>
  );
}

export default AppThemeProvider;
```

📄 **`src/app/AppProviders.tsx`** (전체 교체)

```tsx
import type { ReactNode } from "react";
import { BrowserRouter } from "react-router";
import { QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";
import { queryClient } from "@/app/queryClient";
import AppThemeProvider from "@/shared/theme/AppThemeProvider";

interface AppProvidersProps {
  children: ReactNode;
}

// 앱 전체를 감싸는 Provider를 한 곳에 모은다.
function AppProviders({ children }: AppProvidersProps) {
  return (
    <QueryClientProvider client={queryClient}>
      <AppThemeProvider>
        <BrowserRouter>{children}</BrowserRouter>
      </AppThemeProvider>
      <ReactQueryDevtools buttonPosition="bottom-left" />
    </QueryClientProvider>
  );
}

export default AppProviders;
```

> ⚠️ **주의** Step 3-8(Devtools)을 건너뛰었다면 위 파일에서 `ReactQueryDevtools` 두 줄을 빼세요.

헤더가 점점 커질 예정이니 파일로 분리하면서 전환 버튼을 넣습니다.

📄 **`src/app/layout/AppHeader.tsx`** (새 파일)

```tsx
import { Button, Layout, theme } from "antd";
import {
  MenuFoldOutlined,
  MenuUnfoldOutlined,
  MoonOutlined,
  SunOutlined,
} from "@ant-design/icons";
import { usePreferences } from "@/shared/preferences/preferencesStore";

interface AppHeaderProps {
  collapsed: boolean;
  onToggleCollapsed: () => void;
}

function AppHeader({ collapsed, onToggleCollapsed }: AppHeaderProps) {
  const { token } = theme.useToken();
  const mode = usePreferences((state) => state.mode);
  const setMode = usePreferences((state) => state.setMode);
  const isDark = mode === "dark";

  return (
    <Layout.Header
      style={{
        display: "flex",
        alignItems: "center",
        gap: 8,
        height: 48,
        lineHeight: "48px",
        paddingInline: 12,
        background: token.colorBgContainer,
        borderBottom: `1px solid ${token.colorBorderSecondary}`,
      }}
    >
      <Button
        type="text"
        icon={collapsed ? <MenuUnfoldOutlined /> : <MenuFoldOutlined />}
        onClick={onToggleCollapsed}
      />
      <strong style={{ color: token.colorText }}>{import.meta.env.VITE_APP_TITLE}</strong>
      <div style={{ flex: 1 }} />
      <Button
        type="text"
        icon={isDark ? <SunOutlined /> : <MoonOutlined />}
        onClick={() => setMode(isDark ? "light" : "dark")}
      />
    </Layout.Header>
  );
}

export default AppHeader;
```

📄 **`src/app/layout/MainLayout.tsx`** (전체 교체)

```tsx
import { useState } from "react";
import { Outlet } from "react-router";
import { Layout, theme } from "antd";
import AppHeader from "@/app/layout/AppHeader";
import SideMenu from "@/app/layout/SideMenu";

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
          style={{ borderInlineEnd: `1px solid ${token.colorBorderSecondary}` }}
        >
          <SideMenu />
        </Sider>
        <Content style={{ padding: 16, overflow: "auto" }}>
          <Outlet />
        </Content>
      </Layout>
    </Layout>
  );
}

export default MainLayout;
```

### 코드 읽어 보기

**`useEffect`**: React 바깥 세상(여기서는 `<html>` 태그)과 동기화할 때 씁니다. 두 번째 인자 `[mode]`는 **의존성 배열**로, 이 값이 바뀔 때만 다시 실행합니다.

**상태 끌어올리기**: `collapsed`는 헤더(버튼)와 사이드바 둘 다 필요합니다. 그래서 공통 부모인 `MainLayout`이 갖고, 헤더에는 값과 함수를 props로 내려 줍니다.

> ⌨️ **단축키** `F2`: 이름 바꾸기. 변수 위에서 누르면 사용하는 모든 곳이 한 번에 바뀝니다. 찾아 바꾸기보다 안전합니다.
> `Shift+F12`: 이 이름을 쓰는 곳 모두 찾기.

### 확인하기

- [ ] 달 아이콘을 누르면 antd 영역이 어두워집니다. 새로고침해도 유지됩니다.
- [ ] F12 → Application → Local Storage → `preferences`에 `{"state":{"mode":"dark"},"version":0}`가 보입니다.
- [ ] **그리드만 하얗습니다.** AG Grid는 antd를 모르기 때문입니다. Step 4-4에서 해결합니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 라이트/다크 모드 전환 추가"
```

---

## Step 4-3. 시스템 설정 따르기

> 🎯 **할 일**: `"system"` 모드를 고르면 Windows 설정을 실시간으로 따라가게 합니다.

### 원리: React 밖의 값을 구독하기

OS의 다크 모드 설정은 React 밖에 있습니다. 브라우저는 `matchMedia`로 이 값을 알려 주고, 바뀌면 이벤트를 쏩니다.
이런 **외부 값을 구독**할 때 쓰는 공식 훅이 `useSyncExternalStore`입니다.

- `subscribe`: 값이 바뀌면 알려 달라고 등록하는 함수
- `getSnapshot`: 지금 값을 읽는 함수

### 따라 하기

📄 **`src/shared/theme/useResolvedMode.ts`** (새 파일)

```ts
import { useSyncExternalStore } from "react";
import { usePreferences } from "@/shared/preferences/preferencesStore";

const darkQuery = window.matchMedia("(prefers-color-scheme: dark)");

function subscribe(onChange: () => void) {
  darkQuery.addEventListener("change", onChange);
  return () => darkQuery.removeEventListener("change", onChange); // 구독 해제 함수를 돌려준다
}

/** 설정이 system이면 OS 설정을 보고 light/dark로 확정한다. */
export function useResolvedMode(): "light" | "dark" {
  const mode = usePreferences((state) => state.mode);
  const systemDark = useSyncExternalStore(subscribe, () => darkQuery.matches);
  if (mode === "system") return systemDark ? "dark" : "light";
  return mode;
}
```

✏️ **`src/shared/theme/AppThemeProvider.tsx`** — 모드를 확정된 값으로 바꿉니다.

```tsx
  const mode = usePreferences((state) => state.mode);
```

바꿀 코드

```tsx
  const mode = useResolvedMode();
```

✏️ **`src/shared/theme/AppThemeProvider.tsx`** — import를 바꿉니다.

```tsx
import { usePreferences } from "@/shared/preferences/preferencesStore";
import AntdStaticBridge from "@/shared/ui/AntdStaticBridge";
```

바꿀 코드

```tsx
import { useResolvedMode } from "@/shared/theme/useResolvedMode";
import AntdStaticBridge from "@/shared/ui/AntdStaticBridge";
```

✏️ **`src/app/layout/AppHeader.tsx`** — 헤더 버튼도 확정된 값을 봅니다.

```tsx
  const mode = usePreferences((state) => state.mode);
  const setMode = usePreferences((state) => state.setMode);
  const isDark = mode === "dark";
```

바꿀 코드

```tsx
  const setMode = usePreferences((state) => state.setMode);
  const isDark = useResolvedMode() === "dark";
```

✏️ **`src/app/layout/AppHeader.tsx`** — import를 추가합니다.

```tsx
import { usePreferences } from "@/shared/preferences/preferencesStore";
```

바꿀 코드

```tsx
import { usePreferences } from "@/shared/preferences/preferencesStore";
import { useResolvedMode } from "@/shared/theme/useResolvedMode";
```

### 확인하기

아직 `system`을 고르는 화면이 없으니 콘솔로 시험합니다. Step 4-6에서 화면이 생깁니다.

- [ ] F12 → Application → Local Storage → `preferences`의 `mode`를 `system`으로 고치고 새로고침합니다.
- [ ] Windows 설정 → 개인 설정 → 색 → "기본 앱 모드 선택"을 바꾸면 **새로고침 없이** 앱이 따라 바뀝니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 시스템 다크 모드 설정 연동"
```

---

## Step 4-4. 테마를 따르는 공통 DataGrid

> 🎯 **할 일**: AG Grid를 감싼 공통 컴포넌트를 만들어 다크 모드를 적용합니다.

### 원리: 라이브러리를 감싼다

화면에서 `AgGridReact`를 직접 쓰면, "그리드 기본 행 높이를 전부 줄여 주세요" 같은 요청에 화면 수십 개를 고쳐야 합니다.
**감싼 컴포넌트 하나**를 두면 파일 하나만 고칩니다. 이것이 설계 원칙 1번입니다.

AG Grid는 antd를 모르므로, antd 토큰을 읽어서 AG Grid 테마로 **번역**해 줍니다.

### 따라 하기

📄 **`src/shared/theme/useGridTheme.ts`** (새 파일)

```ts
import { useMemo } from "react";
import { theme } from "antd";
import { themeQuartz } from "ag-grid-community";
import { useResolvedMode } from "@/shared/theme/useResolvedMode";

/** antd 토큰을 읽어 AG Grid 테마를 만든다. */
export function useGridTheme() {
  const { token } = theme.useToken();
  const mode = useResolvedMode();

  return useMemo(
    () =>
      themeQuartz.withParams({
        accentColor: token.colorPrimary,
        backgroundColor: token.colorBgContainer,
        foregroundColor: token.colorText,
        borderColor: token.colorBorderSecondary,
        headerBackgroundColor: token.colorFillAlter,
        fontFamily: token.fontFamily,
        fontSize: token.fontSize,
        browserColorScheme: mode,
      }),
    [token, mode],
  );
}
```

📄 **`src/shared/ui/DataGrid.tsx`** (새 파일)

```tsx
import { useMemo } from "react";
import { AgGridReact, type AgGridReactProps } from "ag-grid-react";
import type { ColDef } from "ag-grid-community";
import { useGridTheme } from "@/shared/theme/useGridTheme";

/** 모든 화면이 쓰는 그리드. 그리드 기본 옵션을 바꾸고 싶으면 이 파일만 고친다. */
function DataGrid<T>({ defaultColDef, ...props }: AgGridReactProps<T>) {
  const gridTheme = useGridTheme();

  const mergedColDef = useMemo<ColDef<T>>(
    () => ({ sortable: true, resizable: true, flex: 1, minWidth: 80, ...defaultColDef }),
    [defaultColDef],
  );

  return (
    // 그리드는 스스로 높이를 정하지 않는다. 부모의 남은 높이를 받도록 flex: 1을 준다.
    <div style={{ flex: 1, minHeight: 240 }}>
      <AgGridReact<T> theme={gridTheme} defaultColDef={mergedColDef} {...props} />
    </div>
  );
}

export default DataGrid;
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — 그리드를 교체합니다.

```tsx
      <div style={{ flex: 1, minHeight: 0 }}>
        <AgGridReact<User>
          rowData={data}
          columnDefs={columnDefs}
          defaultColDef={defaultColDef}
          loading={isFetching}
        />
      </div>
```

바꿀 코드

```tsx
      <DataGrid<User> rowData={data} columnDefs={columnDefs} loading={isFetching} />
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — 이제 필요 없는 상수 `defaultColDef`를 지웁니다.

```tsx
const defaultColDef: ColDef<User> = { sortable: true, flex: 1 };

function UsersPage() {
```

바꿀 코드

```tsx
function UsersPage() {
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — import를 고칩니다.

```tsx
import { AgGridReact } from "ag-grid-react";
import type { ColDef } from "ag-grid-community";
import { useUserList } from "@/features/users/api/queries";
import type { User, UserSearchCondition } from "@/features/users/types";
```

바꿀 코드

```tsx
import type { ColDef } from "ag-grid-community";
import { useUserList } from "@/features/users/api/queries";
import type { User, UserSearchCondition } from "@/features/users/types";
import DataGrid from "@/shared/ui/DataGrid";
```

### 코드 읽어 보기

**제네릭 컴포넌트**: `function DataGrid<T>`는 어떤 데이터든 받는 컴포넌트입니다. 쓰는 쪽이 `<DataGrid<User> ... />`로 타입을 정해 줍니다.

**`{ defaultColDef, ...props }`**: 구조 분해입니다. `defaultColDef`만 따로 꺼내고 나머지는 `props`에 모읍니다.
`{...props}`로 나머지를 그대로 넘기므로, `AgGridReact`의 모든 옵션을 그대로 쓸 수 있습니다.

**`useMemo`**: 계산 결과를 기억해 두는 훅입니다. 그리드는 옵션 객체가 **새 객체로 바뀌면** "옵션이 바뀌었다"고 판단합니다.
렌더할 때마다 `{ sortable: true }`를 새로 만들면 쓸데없이 내부 처리가 일어나므로, 의존성이 바뀔 때만 새로 만듭니다.

> ⚠️ **주의** 이제부터 화면에서 `AgGridReact`를 직접 쓰지 않습니다. 항상 `DataGrid`를 씁니다.

### 확인하기

- [ ] 다크 모드에서 그리드도 어두워집니다.
- [ ] 정렬, 조회 등 기존 기능이 그대로 동작합니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 테마를 따르는 공통 DataGrid 추가"
```

---

## Step 4-5. 테마 프리셋과 화면 밀도

> 🎯 **할 일**: 여러 테마를 고를 수 있게 하고, 화면을 촘촘하게 만드는 옵션을 추가합니다.

### 원리: 설정은 서로 독립이다

모드(3) × 테마(5) × 밀도(2) = 30가지 조합이 **코드 추가 없이** 동작합니다. 각 설정이 독립적이기 때문입니다.
새 테마는 프리셋 객체 하나만 추가하면 됩니다. 템플릿을 새 프로젝트에 가져갈 때 첫 작업이 고객사 프리셋 하나 추가하기입니다.

### 따라 하기

📄 **`src/shared/theme/presets.ts`** (새 파일)

```ts
import type { ThemeConfig } from "antd";

export type ThemePresetName = "default" | "public" | "modern" | "forest" | "sunset";
export type Density = "default" | "compact";

/** 새 테마를 추가하려면 여기에 한 줄 추가하고 ThemePresetName에 이름을 넣는다. */
export const presets: Record<ThemePresetName, ThemeConfig["token"]> = {
  default: { colorPrimary: "#1677ff", borderRadius: 6 }, // antd 기본에 가까운 무난한 테마
  public: { colorPrimary: "#1d4ed8", borderRadius: 2, fontSize: 13 }, // 공공기관풍: 각진 모서리, 작은 글자
  modern: { colorPrimary: "#7c3aed", borderRadius: 10 }, // 보라색, 둥근 모서리
  forest: { colorPrimary: "#15803d", borderRadius: 6 }, // 초록색
  sunset: { colorPrimary: "#ea580c", borderRadius: 8 }, // 주황색
};
```

📄 **`src/shared/preferences/preferencesStore.ts`** (전체 교체)

```ts
import { create } from "zustand";
import { persist } from "zustand/middleware";
import type { Density, ThemePresetName } from "@/shared/theme/presets";

export type ThemeMode = "light" | "dark" | "system";

interface PreferencesState {
  mode: ThemeMode;
  preset: ThemePresetName;
  density: Density;
  setMode: (mode: ThemeMode) => void;
  setPreset: (preset: ThemePresetName) => void;
  setDensity: (density: Density) => void;
}

export const usePreferences = create<PreferencesState>()(
  // persist: 값이 바뀔 때마다 localStorage에 저장하고, 시작할 때 복원한다.
  persist(
    (set) => ({
      mode: "light",
      preset: "default",
      density: "default",
      setMode: (mode) => set({ mode }),
      setPreset: (preset) => set({ preset }),
      setDensity: (density) => set({ density }),
    }),
    { name: "preferences" }, // localStorage 키 이름
  ),
);
```

✏️ **`src/shared/theme/AppThemeProvider.tsx`** — 프리셋과 밀도를 읽습니다.

```tsx
  const mode = useResolvedMode();

```

바꿀 코드

```tsx
  const mode = useResolvedMode();
  const preset = usePreferences((state) => state.preset);
  const density = usePreferences((state) => state.density);

  // 알고리즘은 여러 개를 겹쳐 쓸 수 있다. 다크 + 촘촘하게 같은 조합이 된다.
  const algorithm = [
    mode === "dark" ? theme.darkAlgorithm : theme.defaultAlgorithm,
    ...(density === "compact" ? [theme.compactAlgorithm] : []),
  ];

```

✏️ **`src/shared/theme/AppThemeProvider.tsx`** — `ConfigProvider`에 넘깁니다.

```tsx
    <ConfigProvider
      locale={koKR}
      theme={{ algorithm: mode === "dark" ? theme.darkAlgorithm : theme.defaultAlgorithm }}
    >
```

바꿀 코드

```tsx
    <ConfigProvider locale={koKR} theme={{ algorithm, token: presets[preset] }}>
```

✏️ **`src/shared/theme/AppThemeProvider.tsx`** — import를 추가합니다.

```tsx
import { useResolvedMode } from "@/shared/theme/useResolvedMode";
```

바꿀 코드

```tsx
import { usePreferences } from "@/shared/preferences/preferencesStore";
import { presets } from "@/shared/theme/presets";
import { useResolvedMode } from "@/shared/theme/useResolvedMode";
```

그리드도 밀도를 따르게 합니다. `spacing` 값 하나로 행 높이와 여백이 함께 줄어듭니다.

✏️ **`src/shared/theme/useGridTheme.ts`** — 밀도를 읽습니다.

```ts
  const mode = useResolvedMode();

```

바꿀 코드

```ts
  const mode = useResolvedMode();
  const density = usePreferences((state) => state.density);

```

✏️ **`src/shared/theme/useGridTheme.ts`** — `spacing`을 넣습니다.

```ts
        browserColorScheme: mode,
      }),
    [token, mode],
```

바꿀 코드

```ts
        browserColorScheme: mode,
        spacing: density === "compact" ? 4 : 7,
      }),
    [token, mode, density],
```

✏️ **`src/shared/theme/useGridTheme.ts`** — import를 추가합니다.

```ts
import { useResolvedMode } from "@/shared/theme/useResolvedMode";
```

바꿀 코드

```ts
import { usePreferences } from "@/shared/preferences/preferencesStore";
import { useResolvedMode } from "@/shared/theme/useResolvedMode";
```

### 확인하기

- [ ] F12 → Application → Local Storage의 `preferences`에서 `preset`을 `modern`으로, `density`를 `compact`로 고치고 새로고침합니다.
- [ ] 버튼 색이 보라색, 모서리가 둥글어지고, 그리드 행이 낮아집니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 테마 프리셋 및 화면 밀도 추가"
```

---

## Step 4-6. 화면 설정 패널

> 🎯 **할 일**: 헤더 버튼을 누르면 오른쪽에서 설정 패널이 열리게 합니다.

### 따라 하기

📄 **`src/app/layout/SettingsDrawer.tsx`** (새 파일)

```tsx
import { Drawer, Flex, Radio, Segmented, Typography } from "antd";
import { usePreferences, type ThemeMode } from "@/shared/preferences/preferencesStore";
import { presets, type Density, type ThemePresetName } from "@/shared/theme/presets";

const presetLabels: Record<ThemePresetName, string> = {
  default: "기본",
  public: "공공",
  modern: "모던",
  forest: "포레스트",
  sunset: "선셋",
};

interface SettingsDrawerProps {
  open: boolean;
  onClose: () => void;
}

function SettingsDrawer({ open, onClose }: SettingsDrawerProps) {
  const mode = usePreferences((state) => state.mode);
  const preset = usePreferences((state) => state.preset);
  const density = usePreferences((state) => state.density);
  const setMode = usePreferences((state) => state.setMode);
  const setPreset = usePreferences((state) => state.setPreset);
  const setDensity = usePreferences((state) => state.setDensity);

  return (
    <Drawer title="화면 설정" open={open} onClose={onClose} size={320}>
      <Flex vertical gap={24}>
        <div>
          <Typography.Title level={5}>모드</Typography.Title>
          <Segmented<ThemeMode>
            block
            value={mode}
            onChange={setMode}
            options={[
              { value: "light", label: "라이트" },
              { value: "dark", label: "다크" },
              { value: "system", label: "시스템" },
            ]}
          />
        </div>
        <div>
          <Typography.Title level={5}>테마</Typography.Title>
          <Radio.Group
            value={preset}
            onChange={(event) => setPreset(event.target.value)}
            style={{ display: "flex", flexDirection: "column", gap: 8 }}
          >
            {(Object.keys(presets) as ThemePresetName[]).map((name) => (
              <Radio key={name} value={name}>
                <Flex align="center" gap={8}>
                  <span
                    style={{
                      width: 14,
                      height: 14,
                      borderRadius: "50%",
                      background: presets[name]?.colorPrimary,
                    }}
                  />
                  {presetLabels[name]}
                </Flex>
              </Radio>
            ))}
          </Radio.Group>
        </div>
        <div>
          <Typography.Title level={5}>밀도</Typography.Title>
          <Segmented<Density>
            block
            value={density}
            onChange={setDensity}
            options={[
              { value: "default", label: "기본" },
              { value: "compact", label: "촘촘하게" },
            ]}
          />
        </div>
      </Flex>
    </Drawer>
  );
}

export default SettingsDrawer;
```

✏️ **`src/app/layout/AppHeader.tsx`** — 설정 버튼을 추가합니다.

```tsx
        onClick={() => setMode(isDark ? "light" : "dark")}
      />
    </Layout.Header>
```

바꿀 코드

```tsx
        onClick={() => setMode(isDark ? "light" : "dark")}
      />
      <Button type="text" icon={<SettingOutlined />} onClick={() => setSettingsOpen(true)} />
      <SettingsDrawer open={settingsOpen} onClose={() => setSettingsOpen(false)} />
    </Layout.Header>
```

✏️ **`src/app/layout/AppHeader.tsx`** — state를 추가합니다.

```tsx
  const isDark = useResolvedMode() === "dark";
```

바꿀 코드

```tsx
  const isDark = useResolvedMode() === "dark";
  const [settingsOpen, setSettingsOpen] = useState(false);
```

✏️ **`src/app/layout/AppHeader.tsx`** — import를 고칩니다.

```tsx
import { Button, Layout, theme } from "antd";
import {
  MenuFoldOutlined,
  MenuUnfoldOutlined,
  MoonOutlined,
  SunOutlined,
} from "@ant-design/icons";
import { usePreferences } from "@/shared/preferences/preferencesStore";
```

바꿀 코드

```tsx
import { useState } from "react";
import { Button, Layout, theme } from "antd";
import {
  MenuFoldOutlined,
  MenuUnfoldOutlined,
  MoonOutlined,
  SettingOutlined,
  SunOutlined,
} from "@ant-design/icons";
import SettingsDrawer from "@/app/layout/SettingsDrawer";
import { usePreferences } from "@/shared/preferences/preferencesStore";
```

### 코드 읽어 보기

`Object.keys(presets)`로 프리셋 목록을 만들기 때문에, `presets.ts`에 한 줄 추가하면 이 화면에도 자동으로 나타납니다. **화면은 데이터가 정합니다.**

`.map()` 안의 `key={name}`은 React가 목록의 항목을 구분하는 표시입니다. 빠뜨리면 콘솔에 경고가 납니다.

`presetLabels`가 `Record<ThemePresetName, string>`이라서, 프리셋을 추가하고 라벨을 빠뜨리면 **타입 오류**가 납니다. 실수를 컴파일러가 잡아 줍니다.

### 확인하기

- [ ] 톱니바퀴 버튼 → 패널이 열립니다.
- [ ] 모드·테마·밀도를 바꾸면 즉시 전체가 바뀝니다. 새로고침해도 유지됩니다.
- [ ] 패널 안의 컴포넌트까지 다크 모드가 적용됩니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 화면 설정 패널 추가"
```

---

## Step 4-7. 직접 만든 CSS도 테마를 따르게

> 🎯 **할 일**: 자주 쓰는 토큰을 CSS 변수로 내보냅니다.

### 왜 필요한가요?

컴포넌트 안에서는 `theme.useToken()`을 쓰면 됩니다. 그런데 `.css` 파일에서는 토큰을 읽을 수 없습니다.
Part 11의 에디터처럼 CSS로 꾸미는 영역을 위해, 토큰 몇 개를 `:root`의 CSS 변수로 내보냅니다.

### 따라 하기

📄 **`src/shared/theme/CssVariableBridge.tsx`** (새 파일)

```tsx
import { useEffect } from "react";
import { theme } from "antd";

// 화면에는 아무것도 그리지 않고 CSS 변수만 기록한다.
function CssVariableBridge() {
  const { token } = theme.useToken();

  useEffect(() => {
    const style = document.documentElement.style;
    style.setProperty("--app-color-primary", token.colorPrimary);
    style.setProperty("--app-color-bg", token.colorBgContainer);
    style.setProperty("--app-color-bg-layout", token.colorBgLayout);
    style.setProperty("--app-color-fill", token.colorFillTertiary);
    style.setProperty("--app-color-text", token.colorText);
    style.setProperty("--app-color-text-secondary", token.colorTextSecondary);
    style.setProperty("--app-color-border", token.colorBorder);
    style.setProperty("--app-radius", `${token.borderRadius}px`);
  }, [token]);

  return null;
}

export default CssVariableBridge;
```

✏️ **`src/shared/theme/AppThemeProvider.tsx`** — `ConfigProvider` 안쪽에 넣습니다.

```tsx
        <AntdStaticBridge />
```

바꿀 코드

```tsx
        <AntdStaticBridge />
        <CssVariableBridge />
```

✏️ **`src/shared/theme/AppThemeProvider.tsx`** — import를 추가합니다.

```tsx
import { usePreferences } from "@/shared/preferences/preferencesStore";
```

바꿀 코드

```tsx
import { usePreferences } from "@/shared/preferences/preferencesStore";
import CssVariableBridge from "@/shared/theme/CssVariableBridge";
```

> ⚠️ **주의** `theme.useToken()`은 `ConfigProvider` **안쪽**에서 호출해야 합니다. 바깥에서 부르면 기본값만 나옵니다.
> `AppThemeProvider` 함수 본문이 아니라 자식 컴포넌트를 따로 만든 이유입니다.

### 확인하기

- [ ] F12 → Elements에서 `<html>` 태그를 선택하고 Styles 패널을 보면 `--app-color-primary` 등이 있습니다.
- [ ] 테마를 바꾸면 값이 따라 바뀝니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 테마 토큰을 CSS 변수로 노출"
```

---

## Step 4-8. 새로고침 깜빡임 방지 ○

> 🎯 **할 일**: 다크 모드에서 새로고침할 때 흰 화면이 번쩍이는 것을 막습니다.

### 원리: FOUC

브라우저는 HTML을 받자마자 그립니다. 그런데 antd 스타일은 React가 실행되고 나서야 만들어집니다.
그 사이 아주 잠깐 흰 바탕이 보입니다. 이것을 **FOUC**(Flash of Unstyled Content)라고 합니다. 다크 모드 사용자에겐 눈이 부십니다.

해결법은 `index.html`에서 React보다 **먼저** 저장된 설정을 읽어 배경색만 칠해 두는 것입니다.

### 따라 하기

✏️ **`index.html`** — `<head>` 안, `<title>` 아래에 넣습니다.

```html
    <title>%VITE_APP_TITLE%</title>
```

바꿀 코드

```html
    <title>%VITE_APP_TITLE%</title>
    <script>
      // React가 뜨기 전에 다크 배경을 칠해서 흰 화면이 번쩍이지 않게 한다.
      try {
        const saved = JSON.parse(localStorage.getItem("preferences") || "{}").state || {};
        const dark =
          saved.mode === "dark" ||
          (saved.mode === "system" && matchMedia("(prefers-color-scheme: dark)").matches);
        if (dark) document.documentElement.style.background = "#141414";
      } catch {
        // 저장값이 깨져 있어도 앱 시작을 막지 않는다.
      }
    </script>
```

> 💡 **팁** Zustand의 `persist`는 `{ state: {...}, version: 0 }` 형태로 저장합니다. 그래서 `.state`를 꺼냅니다.

### 확인하기

- [ ] 다크 모드에서 `F5`를 눌러도 흰 화면이 번쩍이지 않습니다.

✅ **커밋**

```powershell
git add .
git commit -m "fix: 다크 모드 새로고침 시 흰 화면 깜빡임 수정"
```

---

## Step 4-9. 대표 색상 직접 고르기 ○

> 🎯 **할 일**: 프리셋 위에 사용자가 고른 색을 덮어씁니다.

✏️ **`src/shared/preferences/preferencesStore.ts`** — 타입에 필드를 추가합니다.

```ts
  density: Density;
  setMode: (mode: ThemeMode) => void;
```

바꿀 코드

```ts
  density: Density;
  primaryColor: string | null; // null이면 프리셋의 색을 쓴다
  setPrimaryColor: (color: string | null) => void;
  setMode: (mode: ThemeMode) => void;
```

✏️ **`src/shared/preferences/preferencesStore.ts`** — 기본값을 추가합니다.

```ts
      density: "default",
      setMode: (mode) => set({ mode }),
```

바꿀 코드

```ts
      density: "default",
      primaryColor: null,
      setPrimaryColor: (primaryColor) => set({ primaryColor }),
      setMode: (mode) => set({ mode }),
```

✏️ **`src/shared/theme/AppThemeProvider.tsx`** — 색을 읽습니다.

```tsx
  const density = usePreferences((state) => state.density);

```

바꿀 코드

```tsx
  const density = usePreferences((state) => state.density);
  const primaryColor = usePreferences((state) => state.primaryColor);

```

✏️ **`src/shared/theme/AppThemeProvider.tsx`** — 토큰에 덮어씁니다.

```tsx
    <ConfigProvider locale={koKR} theme={{ algorithm, token: presets[preset] }}>
```

바꿀 코드

```tsx
    <ConfigProvider
      locale={koKR}
      theme={{
        algorithm,
        token: { ...presets[preset], ...(primaryColor ? { colorPrimary: primaryColor } : {}) },
      }}
    >
```

✏️ **`src/app/layout/SettingsDrawer.tsx`** — 색 선택기를 추가합니다.

```tsx
        <div>
          <Typography.Title level={5}>밀도</Typography.Title>
```

바꿀 코드

```tsx
        <div>
          <Typography.Title level={5}>대표 색상</Typography.Title>
          <Flex gap={8}>
            <ColorPicker
              disabledAlpha
              value={primaryColor ?? presets[preset]?.colorPrimary}
              onChangeComplete={(color) => setPrimaryColor(color.toHexString())}
            />
            <Button disabled={!primaryColor} onClick={() => setPrimaryColor(null)}>
              테마 기본값
            </Button>
          </Flex>
        </div>
        <div>
          <Typography.Title level={5}>밀도</Typography.Title>
```

✏️ **`src/app/layout/SettingsDrawer.tsx`** — 값을 읽습니다.

```tsx
  const setDensity = usePreferences((state) => state.setDensity);
```

바꿀 코드

```tsx
  const setDensity = usePreferences((state) => state.setDensity);
  const primaryColor = usePreferences((state) => state.primaryColor);
  const setPrimaryColor = usePreferences((state) => state.setPrimaryColor);
```

✏️ **`src/app/layout/SettingsDrawer.tsx`** — import를 고칩니다.

```tsx
import { Drawer, Flex, Radio, Segmented, Typography } from "antd";
```

바꿀 코드

```tsx
import { Button, ColorPicker, Drawer, Flex, Radio, Segmented, Typography } from "antd";
```

### 확인하기

- [ ] 대표 색상을 고르면 버튼·메뉴 강조·그리드 강조색이 함께 바뀝니다.
- [ ] [테마 기본값]을 누르면 프리셋 색으로 돌아옵니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 대표 색상 직접 선택 기능 추가"
```

---

## 🏁 Part 4를 마치며

| 증상                                     | 원인과 해결                                                                     |
| ---------------------------------------- | ------------------------------------------------------------------------------- |
| 다크 모드에서 특정 영역만 하얗다         | 색을 하드코딩했다 → 토큰 또는 `--app-*` 변수로. 코드에서 `#`을 검색해 본다      |
| 그리드만 테마를 안 따른다                | `AgGridReact`를 직접 썼다 → `DataGrid` 사용                                     |
| `message.success()` 알림이 라이트로 뜬다 | antd **정적 메서드**는 설정을 못 읽는다 → `App.useApp()`의 `message`를 쓴다     |
| `theme.useToken()` 값이 기본값이다       | `ConfigProvider` 바깥에서 호출했다                                              |

> ⌨️ **단축키** `Ctrl+Shift+F`: 프로젝트 전체에서 찾기. 하드코딩된 색을 찾을 때 `#[0-9a-fA-F]{3,6}`를 정규식 모드(`Alt+R`)로 검색해 보세요.

> 🧩 **확장** 지금 CSS 변수나 색 코드가 어떤 색인지 머릿속으로 상상해야 하죠? 색 코드 옆에 미리보기가 뜨는 기능은 VS Code에 내장되어 있습니다.
> `.css` 파일에서는 바로 보입니다. `.ts` 파일에서도 보고 싶다면 **Color Highlight** 확장을 설치합니다. 설치 후 `presets.ts`를 열면 색 코드에 배경색이 칠해집니다.

다음은 언어를 바꿉니다. 화면이 몇 개 없는 지금이 가장 쉬울 때입니다. → [Part 5. 다국어](part-05-i18n.md)
