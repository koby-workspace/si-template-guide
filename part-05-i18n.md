# Part 5. 다국어

> **이 Part의 목표**: 화면 문구를 코드에서 빼서 언어별 JSON 파일에 두고, 헤더에서 언어를 바꾸면 우리 문구·antd·AG Grid·날짜가 함께 바뀌게 합니다.
> 화면이 2개일 때 하면 30분, 30개일 때 하면 며칠이 걸리는 작업입니다. 그래서 지금 합니다.

---

## Step 5-1. i18next 설정과 공통 문구

> 🎯 **할 일**: 다국어 라이브러리를 설치하고, 번역 파일을 넣기만 하면 자동 등록되는 구조를 만듭니다.

### 번역 대상은 네 종류다

| 종류            | 예                                  | 방법                                                   |
| --------------- | ----------------------------------- | ------------------------------------------------------ |
| 화면 고정 문구  | "조회", "사용자 관리"               | JSON 번역 파일 + `t("키")` (이 Part)                   |
| 라이브러리 문구 | antd 달력, AG Grid 필터 메뉴        | 각 라이브러리의 로케일 객체 교체 (Step 5-2)            |
| 데이터 문구     | 메뉴명                              | 데이터에 언어별 값 저장 `{ ko, en }` (Part 6)          |
| 서버 메시지     | "이미 사용 중인 이메일입니다."      | `Accept-Language` 헤더로 서버가 번역 (Step 5-5)        |

### 왜 react-i18next인가요?

React 다국어의 사실상 표준입니다. 네임스페이스(파일 나누기), 보간(`{{count}}`), 복수형을 지원하고 자료가 가장 많습니다.
대안으로는 react-intl(FormatJS), Lingui, Paraglide 등이 있습니다.

### 파일 구조

```
src/shared/i18n/
├─ index.ts          i18next 초기화
├─ locales.ts        지원 언어 목록, 라이브러리 로케일 연결
└─ locales/
   ├─ ko/
   │  ├─ common.json   버튼·공통 메시지 (모든 화면)
   │  ├─ layout.json   헤더·설정 패널
   │  ├─ errors.json   오류 문구
   │  └─ users.json    사용자 화면
   └─ en/              (ko와 같은 파일들)
```

**네임스페이스 = 파일 이름**입니다. 화면 전용 문구는 그 화면의 파일에, 두 곳 이상에서 쓰면 `common`에 둡니다.
번역을 외부에 맡길 때 `locales/` 폴더 하나만 전달하면 됩니다.

### 따라 하기

▶️ **터미널**

```powershell
npm install i18next@26.7.2 react-i18next@17.1.13
```

✅ **커밋**

```powershell
git add .
git commit -m "chore: i18next, react-i18next 의존성 추가"
```

먼저 언어 설정을 저장소에 추가합니다.

📄 **`src/shared/i18n/locales.ts`** (새 파일)

```ts
export type Locale = "ko" | "en";

/** 지원 언어 목록. 언어를 추가하면 여기에 한 줄, locales/ 폴더에 번역 파일을 추가한다. */
export const locales: { value: Locale; label: string }[] = [
  { value: "ko", label: "한국어" },
  { value: "en", label: "English" },
];
```

✏️ **`src/shared/preferences/preferencesStore.ts`** — import를 추가합니다.

```ts
import type { Density, ThemePresetName } from "@/shared/theme/presets";
```

바꿀 코드

```ts
import type { Locale } from "@/shared/i18n/locales";
import type { Density, ThemePresetName } from "@/shared/theme/presets";
```

✏️ **`src/shared/preferences/preferencesStore.ts`** — 타입에 언어를 추가합니다.

```ts
  primaryColor: string | null; // null이면 프리셋의 색을 쓴다
```

바꿀 코드

```ts
  primaryColor: string | null; // null이면 프리셋의 색을 쓴다
  locale: Locale;
  setLocale: (locale: Locale) => void;
```

✏️ **`src/shared/preferences/preferencesStore.ts`** — 기본값을 추가합니다.

```ts
      primaryColor: null,
```

바꿀 코드

```ts
      primaryColor: null,
      locale: "ko",
      setLocale: (locale) => set({ locale }),
```

이제 i18next를 초기화합니다. 번역 파일을 하나하나 import 하지 않고, Vite의 `import.meta.glob`으로 **폴더에 있는 JSON을 전부** 가져옵니다.

📄 **`src/shared/i18n/index.ts`** (새 파일)

```ts
import i18n from "i18next";
import { initReactI18next } from "react-i18next";
import { usePreferences } from "@/shared/preferences/preferencesStore";

// locales/<언어>/<네임스페이스>.json 파일을 빌드할 때 전부 찾아서 가져온다.
// 새 번역 파일을 폴더에 넣기만 하면 자동으로 등록된다.
const files = import.meta.glob<Record<string, unknown>>("./locales/*/*.json", {
  eager: true,
  import: "default",
});

// { "./locales/ko/common.json": {...} } → { ko: { common: {...} } }
const resources: Record<string, Record<string, Record<string, unknown>>> = {};
for (const [path, messages] of Object.entries(files)) {
  const [, lng, ns] = path.match(/\.\/locales\/(\w+)\/(\w+)\.json$/) ?? [];
  if (!lng || !ns) continue;
  resources[lng] ??= {};
  resources[lng][ns] = messages;
}

i18n.use(initReactI18next).init({
  resources,
  lng: usePreferences.getState().locale, // 저장된 언어로 시작한다
  fallbackLng: "ko", // 번역이 빠진 키는 한국어로 보여 준다
  defaultNS: "common",
  initAsync: false, // 번역이 이미 메모리에 있으니 기다리지 않고 바로 초기화한다
  interpolation: { escapeValue: false }, // React가 이미 XSS를 막으므로 이중 이스케이프를 끈다
});

export default i18n;
```

공통 문구 파일입니다. 앞으로 여러 화면에서 쓸 버튼·메시지를 미리 넣어 둡니다.

📄 **`src/shared/i18n/locales/ko/common.json`** (새 파일)

```json
{
  "button": {
    "search": "조회",
    "reset": "초기화",
    "create": "등록",
    "edit": "수정",
    "save": "저장",
    "delete": "삭제",
    "cancel": "취소",
    "close": "닫기",
    "confirm": "확인",
    "refresh": "새로고침",
    "add": "추가",
    "list": "목록",
    "export": "내보내기"
  },
  "label": {
    "all": "전체",
    "use": "사용",
    "unuse": "미사용",
    "createdAt": "등록일",
    "updatedAt": "수정일"
  },
  "message": {
    "saved": "저장되었습니다.",
    "deleted": "삭제되었습니다.",
    "confirmDelete": "{{count}}건을 삭제하시겠습니까?",
    "selectRows": "대상을 선택해 주세요.",
    "unsavedChanges": "저장하지 않은 변경 내용이 있습니다. 닫으시겠습니까?"
  },
  "validation": {
    "required": "{{field}} 입력은 필수입니다.",
    "email": "이메일 형식이 올바르지 않습니다."
  }
}
```

📄 **`src/shared/i18n/locales/en/common.json`** (새 파일)

```json
{
  "button": {
    "search": "Search",
    "reset": "Reset",
    "create": "New",
    "edit": "Edit",
    "save": "Save",
    "delete": "Delete",
    "cancel": "Cancel",
    "close": "Close",
    "confirm": "OK",
    "refresh": "Refresh",
    "add": "Add",
    "list": "List",
    "export": "Export"
  },
  "label": {
    "all": "All",
    "use": "In use",
    "unuse": "Not in use",
    "createdAt": "Created",
    "updatedAt": "Updated"
  },
  "message": {
    "saved": "Saved.",
    "deleted": "Deleted.",
    "confirmDelete": "Delete {{count}} item(s)?",
    "selectRows": "Please select items.",
    "unsavedChanges": "You have unsaved changes. Close anyway?"
  },
  "validation": {
    "required": "{{field}} is required.",
    "email": "Invalid email format."
  }
}
```

i18n 설정은 앱이 시작될 때 **한 번만** 실행되면 됩니다. `main.tsx`에서 import 합니다.

✏️ **`src/main.tsx`** — import를 추가합니다.

```tsx
import "./index.css";
```

바꿀 코드

```tsx
import "./index.css";
import "@/shared/i18n";
```

`import "파일"`처럼 이름 없이 가져오면 "그 파일을 실행만 하라"는 뜻입니다. 초기화 코드가 실행됩니다.

### 키 이름 규칙

| 규칙                                    | 좋은 예                        | 나쁜 예                          |
| --------------------------------------- | ------------------------------ | -------------------------------- |
| 키는 의미로                             | `button.save`                  | `저장버튼`, `btn1`               |
| 문장을 이어 붙이지 않는다 (어순이 다름) | `t("total", { count })`        | `t("total") + count + t("unit")` |
| 한국어 조사 문제는 조사 없는 문장으로   | `"{{field}} 입력은 필수입니다."` | `"{{field}}을(를) 입력하세요."`  |

### 확인하기

- [ ] 화면은 그대로입니다. 오류가 없으면 됩니다. (아직 `t()`를 쓰는 곳이 없습니다)

✅ **커밋**

```powershell
git add .
git commit -m "feat: 다국어 기본 설정 및 공통 문구 추가"
```

---

## Step 5-2. 언어 전환과 라이브러리 언어 맞추기

> 🎯 **할 일**: 헤더에서 언어를 고르면 우리 문구, antd, AG Grid, 날짜, `<html lang>`이 한꺼번에 바뀌게 합니다.

### 원리: 한 곳에서 동기화한다

언어를 바꾸면 함께 바뀌어야 하는 것이 여섯 가지입니다.

```
setLocale("en")  →  preferencesStore.locale = "en"  (저장, 새로고침 유지)
                     ├─ i18n.changeLanguage("en")           우리 문구
                     ├─ <ConfigProvider locale={enUS}>       antd 달력·페이지 이동·"데이터 없음"
                     ├─ AG Grid localeText                  그리드 메뉴·필터
                     ├─ dayjs.locale("en")                  요일·월 이름
                     └─ <html lang="en">                    접근성, 브라우저 번역 기능
```

화면마다 처리하지 않고, **저장소의 `locale` 하나를 원본으로** 두고 나머지가 따라오게 만듭니다.

### 따라 하기

▶️ **터미널**

```powershell
npm install dayjs@1.11.23
```

> 💡 **팁** dayjs는 antd가 내부에서 쓰는 날짜 라이브러리라 이미 깔려 있습니다. 하지만 우리 코드에서 직접 쓰므로 직접 의존성으로 올립니다. (Step 2-2의 주의사항)

✅ **커밋**

```powershell
git add .
git commit -m "chore: dayjs 의존성 추가"
```

언어 코드와 각 라이브러리의 로케일 객체를 연결합니다.

📄 **`src/shared/i18n/locales.ts`** (전체 교체)

```ts
import type { Locale as AntdLocale } from "antd/es/locale";
import enUS from "antd/locale/en_US";
import koKR from "antd/locale/ko_KR";
import { AG_GRID_LOCALE_EN_US, AG_GRID_LOCALE_KO_KR } from "@ag-grid-community/locale";
import "dayjs/locale/ko";

export type Locale = "ko" | "en";

/** 지원 언어 목록. 언어를 추가하면 여기에 한 줄, locales/ 폴더에 번역 파일을 추가한다. */
export const locales: { value: Locale; label: string }[] = [
  { value: "ko", label: "한국어" },
  { value: "en", label: "English" },
];

// 언어 코드 → 각 라이브러리의 로케일. 언어를 추가하면 여기에도 한 줄씩 추가한다.
export const antdLocales: Record<Locale, AntdLocale> = { ko: koKR, en: enUS };
export const gridLocales: Record<Locale, Record<string, string>> = {
  ko: AG_GRID_LOCALE_KO_KR,
  en: AG_GRID_LOCALE_EN_US,
};
```

화면에 아무것도 그리지 않고 동기화만 하는 컴포넌트를 만듭니다.

📄 **`src/shared/i18n/LocaleSync.tsx`** (새 파일)

```tsx
import { useEffect } from "react";
import dayjs from "dayjs";
import i18n from "@/shared/i18n";
import { usePreferences } from "@/shared/preferences/preferencesStore";

// 저장소의 locale이 바뀌면 React 밖의 것들(i18next, dayjs, <html lang>)을 맞춘다.
function LocaleSync() {
  const locale = usePreferences((state) => state.locale);

  useEffect(() => {
    i18n.changeLanguage(locale);
    dayjs.locale(locale);
    document.documentElement.lang = locale;
  }, [locale]);

  return null;
}

export default LocaleSync;
```

✏️ **`src/shared/theme/AppThemeProvider.tsx`** — antd 로케일을 저장소 값으로 바꿉니다.

```tsx
  const primaryColor = usePreferences((state) => state.primaryColor);
```

바꿀 코드

```tsx
  const primaryColor = usePreferences((state) => state.primaryColor);
  const locale = usePreferences((state) => state.locale);
```

✏️ **`src/shared/theme/AppThemeProvider.tsx`** — `ConfigProvider`에 넘깁니다.

```tsx
    <ConfigProvider
      locale={koKR}
```

바꿀 코드

```tsx
    <ConfigProvider
      locale={antdLocales[locale]}
```

✏️ **`src/shared/theme/AppThemeProvider.tsx`** — `LocaleSync`를 넣습니다.

```tsx
        <CssVariableBridge />
```

바꿀 코드

```tsx
        <CssVariableBridge />
        <LocaleSync />
```

✏️ **`src/shared/theme/AppThemeProvider.tsx`** — import를 고칩니다.

```tsx
import koKR from "antd/locale/ko_KR";
import { usePreferences } from "@/shared/preferences/preferencesStore";
```

바꿀 코드

```tsx
import { antdLocales } from "@/shared/i18n/locales";
import LocaleSync from "@/shared/i18n/LocaleSync";
import { usePreferences } from "@/shared/preferences/preferencesStore";
```

AG Grid에도 언어를 넣습니다.

✏️ **`src/shared/ui/DataGrid.tsx`** — 언어를 읽습니다.

```tsx
  const gridTheme = useGridTheme();
```

바꿀 코드

```tsx
  const gridTheme = useGridTheme();
  const locale = usePreferences((state) => state.locale);
```

✏️ **`src/shared/ui/DataGrid.tsx`** — `localeText`와 `key`를 넘깁니다.

```tsx
      <AgGridReact<T> theme={gridTheme} defaultColDef={mergedColDef} {...props} />
```

바꿀 코드

```tsx
      {/* localeText는 처음 만들 때만 적용되는 옵션이다. 언어가 바뀌면 key를 바꿔 그리드를 새로 만든다. */}
      <AgGridReact<T>
        key={locale}
        theme={gridTheme}
        localeText={gridLocales[locale]}
        defaultColDef={mergedColDef}
        {...props}
      />
```

✏️ **`src/shared/ui/DataGrid.tsx`** — import를 추가합니다.

```tsx
import { useGridTheme } from "@/shared/theme/useGridTheme";
```

바꿀 코드

```tsx
import { gridLocales } from "@/shared/i18n/locales";
import { usePreferences } from "@/shared/preferences/preferencesStore";
import { useGridTheme } from "@/shared/theme/useGridTheme";
```

> 💡 **팁** 인터넷 예제에는 `AG_GRID_LOCALE_KR`, `AG_GRID_LOCALE_EN`이 많이 나옵니다. AG Grid 36.3부터 이 이름들은 **deprecated**(곧 없어질 예정)가 되어
> `AG_GRID_LOCALE_KO_KR`, `AG_GRID_LOCALE_EN_US`로 바뀌었습니다. VS Code에서 이름에 **취소선**이 그어져 있으면 deprecated라는 뜻이니, 마우스를 올려 대체 이름을 확인하세요.

### 코드 읽어 보기: `key`로 컴포넌트를 새로 만들기

AG Grid의 `localeText`는 타입 정의에 `@initial`이라고 표시되어 있습니다. **처음 만들 때만 읽고, 그 뒤에 바꿔도 무시한다**는 뜻입니다.
이럴 때 React의 `key`를 씁니다. **`key`가 바뀌면 React는 그 컴포넌트를 버리고 완전히 새로 만듭니다.**
언어를 바꿀 때 그리드의 정렬 상태가 초기화되는 대가가 있지만, 언어 전환은 드문 일이라 괜찮습니다.

> 💡 **팁** 라이브러리 옵션이 "바꿨는데 반영이 안 된다"면 타입 정의(`F12`)에서 `@initial` 같은 표시를 찾아보세요. 이 `key` 기법은 Part 7의 탭 새로고침에도 다시 나옵니다.

이제 헤더에 언어 선택 메뉴를 넣습니다.

✏️ **`src/app/layout/AppHeader.tsx`** — 언어 메뉴를 넣습니다.

```tsx
      <div style={{ flex: 1 }} />
```

바꿀 코드

```tsx
      <div style={{ flex: 1 }} />
      <Dropdown
        menu={{
          items: locales.map(({ value, label }) => ({ key: value, label })),
          selectedKeys: [locale],
          onClick: ({ key }) => setLocale(key as Locale),
        }}
      >
        <Button type="text" icon={<GlobalOutlined />}>
          {locales.find((item) => item.value === locale)?.label}
        </Button>
      </Dropdown>
```

✏️ **`src/app/layout/AppHeader.tsx`** — 값을 읽습니다.

```tsx
  const setMode = usePreferences((state) => state.setMode);
```

바꿀 코드

```tsx
  const setMode = usePreferences((state) => state.setMode);
  const locale = usePreferences((state) => state.locale);
  const setLocale = usePreferences((state) => state.setLocale);
```

✏️ **`src/app/layout/AppHeader.tsx`** — import를 고칩니다.

```tsx
import { Button, Layout, theme } from "antd";
import {
  MenuFoldOutlined,
```

바꿀 코드

```tsx
import { Button, Dropdown, Layout, theme } from "antd";
import {
  GlobalOutlined,
  MenuFoldOutlined,
```

✏️ **`src/app/layout/AppHeader.tsx`** — import를 추가합니다.

```tsx
import SettingsDrawer from "@/app/layout/SettingsDrawer";
```

바꿀 코드

```tsx
import SettingsDrawer from "@/app/layout/SettingsDrawer";
import { locales, type Locale } from "@/shared/i18n/locales";
```

### 확인하기

- [ ] 헤더의 🌐 한국어 → English를 고릅니다. 아직 우리 문구는 한국어지만(다음 Step에서 번역), 아래는 영어로 바뀝니다.
  - 그리드 컬럼 제목의 메뉴(☰)를 열면 `Sort Ascending` 등 영어
  - 설정 패널의 색 선택기 등 antd 내부 문구
- [ ] F12 → Elements에서 `<html lang="en">`
- [ ] 새로고침해도 English가 유지됩니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 언어 전환 및 antd·AG Grid·dayjs 로케일 연동"
```

---

## Step 5-3. 레이아웃 번역

> 🎯 **할 일**: 헤더·설정 패널·404 화면의 문구를 번역 파일로 옮깁니다.

### 따라 하기

📄 **`src/shared/i18n/locales/ko/layout.json`** (새 파일)

```json
{
  "header": {
    "toggleSider": "메뉴 접기/펴기",
    "toggleTheme": "라이트/다크 전환",
    "settings": "화면 설정"
  },
  "settings": {
    "title": "화면 설정",
    "mode": "모드",
    "light": "라이트",
    "dark": "다크",
    "system": "시스템",
    "theme": "테마",
    "primaryColor": "대표 색상",
    "resetColor": "테마 기본값",
    "density": "밀도",
    "densityDefault": "기본",
    "densityCompact": "촘촘하게"
  },
  "preset": {
    "default": "기본",
    "public": "공공",
    "modern": "모던",
    "forest": "포레스트",
    "sunset": "선셋"
  }
}
```

📄 **`src/shared/i18n/locales/en/layout.json`** (새 파일)

```json
{
  "header": {
    "toggleSider": "Toggle menu",
    "toggleTheme": "Toggle light/dark",
    "settings": "Display settings"
  },
  "settings": {
    "title": "Display settings",
    "mode": "Mode",
    "light": "Light",
    "dark": "Dark",
    "system": "System",
    "theme": "Theme",
    "primaryColor": "Primary color",
    "resetColor": "Theme default",
    "density": "Density",
    "densityDefault": "Default",
    "densityCompact": "Compact"
  },
  "preset": {
    "default": "Default",
    "public": "Public",
    "modern": "Modern",
    "forest": "Forest",
    "sunset": "Sunset"
  }
}
```

오류 문구는 따로 모아 둡니다. Part 6~7에서 쓸 화면 오류 문구도 미리 넣습니다.

📄 **`src/shared/i18n/locales/ko/errors.json`** (새 파일)

```json
{
  "api": {
    "network": "서버에 연결할 수 없습니다.",
    "server": "잠시 후 다시 시도해 주세요.",
    "forbidden": "권한이 없습니다."
  },
  "page": {
    "notFound": "요청하신 화면을 찾을 수 없습니다.",
    "notFoundOrForbidden": "화면이 없거나 접근 권한이 없습니다.",
    "notReady": "화면 준비 중입니다.",
    "notReadyDetail": "화면 파일 \"{{component}}\"이(가) 아직 없습니다.",
    "crashed": "화면을 표시하는 중 오류가 발생했습니다.",
    "retry": "다시 시도",
    "goHome": "처음으로"
  }
}
```

📄 **`src/shared/i18n/locales/en/errors.json`** (새 파일)

```json
{
  "api": {
    "network": "Cannot connect to the server.",
    "server": "Please try again later.",
    "forbidden": "You do not have permission."
  },
  "page": {
    "notFound": "The page you requested was not found.",
    "notFoundOrForbidden": "The page does not exist or you do not have access.",
    "notReady": "This page is under construction.",
    "notReadyDetail": "Page file \"{{component}}\" does not exist yet.",
    "crashed": "An error occurred while displaying this page.",
    "retry": "Retry",
    "goHome": "Go to start"
  }
}
```

> 💡 **팁** `"notReadyDetail"`의 `\"`는 JSON 문자열 안에 큰따옴표를 넣는 방법입니다.

이제 코드에서 씁니다. 컴포넌트에서는 `useTranslation` 훅으로 `t` 함수를 받습니다.

✏️ **`src/app/layout/AppHeader.tsx`** — `t`를 받습니다.

```tsx
  const { token } = theme.useToken();
```

바꿀 코드

```tsx
  const { t } = useTranslation("layout");
  const { token } = theme.useToken();
```

✏️ **`src/app/layout/AppHeader.tsx`** — 버튼에 툴팁 대신 `title`을 붙입니다.

```tsx
      <Button
        type="text"
        icon={collapsed ? <MenuUnfoldOutlined /> : <MenuFoldOutlined />}
        onClick={onToggleCollapsed}
      />
```

바꿀 코드

```tsx
      <Button
        type="text"
        title={t("header.toggleSider")}
        icon={collapsed ? <MenuUnfoldOutlined /> : <MenuFoldOutlined />}
        onClick={onToggleCollapsed}
      />
```

✏️ **`src/app/layout/AppHeader.tsx`** — 테마·설정 버튼에도 붙입니다.

```tsx
      <Button
        type="text"
        icon={isDark ? <SunOutlined /> : <MoonOutlined />}
        onClick={() => setMode(isDark ? "light" : "dark")}
      />
      <Button type="text" icon={<SettingOutlined />} onClick={() => setSettingsOpen(true)} />
```

바꿀 코드

```tsx
      <Button
        type="text"
        title={t("header.toggleTheme")}
        icon={isDark ? <SunOutlined /> : <MoonOutlined />}
        onClick={() => setMode(isDark ? "light" : "dark")}
      />
      <Button
        type="text"
        title={t("header.settings")}
        icon={<SettingOutlined />}
        onClick={() => setSettingsOpen(true)}
      />
```

✏️ **`src/app/layout/AppHeader.tsx`** — import를 추가합니다.

```tsx
import { useState } from "react";
```

바꿀 코드

```tsx
import { useState } from "react";
import { useTranslation } from "react-i18next";
```

설정 패널은 문구가 많으니 통째로 바꿉니다.

📄 **`src/app/layout/SettingsDrawer.tsx`** (전체 교체)

```tsx
import { useTranslation } from "react-i18next";
import { Button, ColorPicker, Drawer, Flex, Radio, Segmented, Typography } from "antd";
import { usePreferences, type ThemeMode } from "@/shared/preferences/preferencesStore";
import { presets, type Density, type ThemePresetName } from "@/shared/theme/presets";

interface SettingsDrawerProps {
  open: boolean;
  onClose: () => void;
}

function SettingsDrawer({ open, onClose }: SettingsDrawerProps) {
  const { t } = useTranslation("layout");
  const mode = usePreferences((state) => state.mode);
  const preset = usePreferences((state) => state.preset);
  const density = usePreferences((state) => state.density);
  const setMode = usePreferences((state) => state.setMode);
  const setPreset = usePreferences((state) => state.setPreset);
  const setDensity = usePreferences((state) => state.setDensity);
  const primaryColor = usePreferences((state) => state.primaryColor);
  const setPrimaryColor = usePreferences((state) => state.setPrimaryColor);

  return (
    <Drawer title={t("settings.title")} open={open} onClose={onClose} size={320}>
      <Flex vertical gap={24}>
        <div>
          <Typography.Title level={5}>{t("settings.mode")}</Typography.Title>
          <Segmented<ThemeMode>
            block
            value={mode}
            onChange={setMode}
            options={[
              { value: "light", label: t("settings.light") },
              { value: "dark", label: t("settings.dark") },
              { value: "system", label: t("settings.system") },
            ]}
          />
        </div>
        <div>
          <Typography.Title level={5}>{t("settings.theme")}</Typography.Title>
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
                  {t(`preset.${name}`)}
                </Flex>
              </Radio>
            ))}
          </Radio.Group>
        </div>
        <div>
          <Typography.Title level={5}>{t("settings.primaryColor")}</Typography.Title>
          <Flex gap={8}>
            <ColorPicker
              disabledAlpha
              value={primaryColor ?? presets[preset]?.colorPrimary}
              onChangeComplete={(color) => setPrimaryColor(color.toHexString())}
            />
            <Button disabled={!primaryColor} onClick={() => setPrimaryColor(null)}>
              {t("settings.resetColor")}
            </Button>
          </Flex>
        </div>
        <div>
          <Typography.Title level={5}>{t("settings.density")}</Typography.Title>
          <Segmented<Density>
            block
            value={density}
            onChange={setDensity}
            options={[
              { value: "default", label: t("settings.densityDefault") },
              { value: "compact", label: t("settings.densityCompact") },
            ]}
          />
        </div>
      </Flex>
    </Drawer>
  );
}

export default SettingsDrawer;
```

`presetLabels` 표가 사라지고 `` t(`preset.${name}`) ``로 바뀌었습니다. 프리셋 이름이 곧 번역 키가 됩니다.

📄 **`src/app/errors/NotFoundPage.tsx`** (전체 교체)

```tsx
import { useTranslation } from "react-i18next";
import { Link } from "react-router";
import { Button, Result } from "antd";

function NotFoundPage() {
  const { t } = useTranslation("errors");

  return (
    <Result
      status="404"
      title="404"
      subTitle={t("page.notFound")}
      extra={
        <Link to="/">
          <Button type="primary">{t("page.goHome")}</Button>
        </Link>
      }
    />
  );
}

export default NotFoundPage;
```

### 확인하기

- [ ] English로 바꾸면 설정 패널, 버튼에 마우스를 올렸을 때 나오는 설명, 404 화면이 영어로 바뀝니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 레이아웃 다국어 적용"
```

---

## Step 5-4. 사용자 화면 번역

> 🎯 **할 일**: 사용자 화면을 번역하면서, 컴포넌트 **밖** 상수에 `t()`를 쓰면 안 되는 이유를 배웁니다.

### 함정: 컴포넌트 밖의 상수

지금 `columnDefs`는 컴포넌트 밖에 있습니다. 여기에 `t()`를 쓰면 어떻게 될까요?

```tsx
const columnDefs = [{ headerName: i18n.t("users:column.name") }]; // ❌ 파일을 처음 읽을 때 딱 한 번 실행된다

function UsersPage() { ... }
```

파일이 처음 실행될 때 한 번만 계산되므로, 언어를 바꿔도 컬럼 제목은 그대로입니다.
그래서 **컴포넌트 안으로 옮기고, 언어(`t`)가 바뀔 때만 다시 만들도록** `useMemo`를 씁니다.

### 따라 하기

📄 **`src/shared/i18n/locales/ko/users.json`** (새 파일)

```json
{
  "search": {
    "name": "이름",
    "namePlaceholder": "이름 입력",
    "role": "역할"
  },
  "column": {
    "username": "아이디",
    "name": "이름",
    "email": "이메일",
    "phone": "전화번호",
    "roles": "역할",
    "useYn": "사용"
  },
  "role": {
    "ADMIN": "관리자",
    "MANAGER": "매니저",
    "USER": "사용자"
  }
}
```

📄 **`src/shared/i18n/locales/en/users.json`** (새 파일)

```json
{
  "search": {
    "name": "Name",
    "namePlaceholder": "Enter a name",
    "role": "Role"
  },
  "column": {
    "username": "User ID",
    "name": "Name",
    "email": "Email",
    "phone": "Phone",
    "roles": "Roles",
    "useYn": "Use"
  },
  "role": {
    "ADMIN": "Administrator",
    "MANAGER": "Manager",
    "USER": "User"
  }
}
```

📄 **`src/features/users/pages/UsersPage.tsx`** (전체 교체)

```tsx
import { useMemo, useState } from "react";
import { useTranslation } from "react-i18next";
import { Button, Form, Input, Select, Space } from "antd";
import type { ColDef } from "ag-grid-community";
import { useUserList } from "@/features/users/api/queries";
import type { User, UserSearchCondition } from "@/features/users/types";
import DataGrid from "@/shared/ui/DataGrid";

const ROLE_CODES = ["ADMIN", "MANAGER", "USER"];

function UsersPage() {
  const { t } = useTranslation("users");
  const [form] = Form.useForm<UserSearchCondition>();
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

  return (
    <div style={{ height: "100%", display: "flex", flexDirection: "column" }}>
      <Form form={form} layout="inline" onFinish={handleSearch} style={{ marginBottom: 16 }}>
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
        <Space>
          <Button type="primary" htmlType="submit">
            {t("common:button.search")}
          </Button>
          <Button onClick={handleReset}>{t("common:button.reset")}</Button>
        </Space>
      </Form>
      <DataGrid<User> rowData={data} columnDefs={columnDefs} loading={isFetching} />
    </div>
  );
}

export default UsersPage;
```

### 코드 읽어 보기

- `useTranslation("users")`: 기본 네임스페이스를 `users`로 정합니다. `t("column.name")`은 `users.json`에서 찾습니다.
- `t("common:button.search")`: 다른 네임스페이스는 `네임스페이스:키`로 씁니다.
- `` t(`role.${code}`) ``: 데이터 코드값을 번역 키로 씁니다. 데이터에 `ADMIN`을 둔 이유가 여기서 빛납니다.
- 화면 제목 `<h2>`는 지웠습니다. Part 6에서 **메뉴 데이터의 이름**으로 자동 표시되기 때문입니다.

> ⚠️ **주의** 영어는 한국어보다 깁니다. 고정 폭(`width`)보다 `minWidth`·`flex`를 쓰고, 영어로 한 번씩 화면을 훑어보세요.

### 확인하기

- [ ] English로 바꾸면 조회 조건, 버튼, **그리드 컬럼 제목**, 역할 이름까지 바뀝니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 사용자 화면 다국어 적용"
```

---

## Step 5-5. API 오류 문구와 서버 언어 전달

> 🎯 **할 일**: 오류 알림을 번역하고, 서버에도 현재 언어를 알려 줍니다.

### 원리

컴포넌트 밖(`showApiError`)에서는 훅을 쓸 수 없으니 `i18n.t()`를 직접 부릅니다. 이 함수는 **부를 때마다** 현재 언어로 번역하므로 괜찮습니다.
(Step 5-4의 함정은 "한 번만 실행되는 곳"에서 쓸 때의 문제입니다.)

서버가 보내는 메시지(`"이미 사용 중인 이메일입니다."`)는 서버가 번역해야 합니다. HTTP에는 이를 위한 표준 헤더 `Accept-Language`가 있습니다.
Spring Boot는 이 헤더를 보고 `messages_en.properties`를 고르는 기능이 기본으로 있습니다.

### 따라 하기

📄 **`src/shared/api/showApiError.ts`** (전체 교체)

```ts
import { ApiError } from "@/shared/api/httpClient";
import i18n from "@/shared/i18n";
import { antdStatic } from "@/shared/ui/antdStatic";

/** API 오류를 사용자에게 알린다. 화면마다 처리하지 않아도 되도록 Query 전역 처리기에서 호출한다. */
export function showApiError(error: unknown) {
  if (!(error instanceof ApiError)) {
    // 요청 취소는 오류가 아니다. 그 외에는 네트워크 단절 등이다.
    if (error instanceof DOMException && error.name === "AbortError") return;
    antdStatic.message?.error(i18n.t("errors:api.network"));
    return;
  }
  if (error.status >= 500) {
    antdStatic.message?.error(i18n.t("errors:api.server"));
    return;
  }
  if (error.status === 403) {
    antdStatic.message?.error(i18n.t("errors:api.forbidden"));
    return;
  }
  antdStatic.message?.error(error.message);
}
```

✏️ **`src/shared/api/httpClient.ts`** — 언어 헤더를 붙입니다.

```ts
    headers: { "Content-Type": "application/json" },
```

바꿀 코드

```ts
    headers: { "Content-Type": "application/json", "Accept-Language": i18n.language },
```

✏️ **`src/shared/api/httpClient.ts`** — import를 추가합니다.

```ts
import type { ProblemDetail } from "@/shared/api/types";
```

바꿀 코드

```ts
import type { ProblemDetail } from "@/shared/api/types";
import i18n from "@/shared/i18n";
```

### 확인하기

- [ ] F12 → Network → `users` 요청 → Headers → Request Headers에 `Accept-Language: en` (English 선택 시)
- [ ] Step 3-7처럼 500 오류를 일부러 만들면, English에서는 `Please try again later.`가 뜹니다. 확인 후 되돌립니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: API 오류 메시지 다국어 적용"
```

---

## 🏁 Part 5를 마치며

| 증상                                           | 원인과 해결                                                                    |
| ---------------------------------------------- | ------------------------------------------------------------------------------ |
| 언어를 바꿨는데 컬럼 제목·Select 옵션만 그대로 | 컴포넌트 **밖** 상수에서 번역했다 → 컴포넌트 안 `useMemo(…, [t])`              |
| 화면에 `column.name` 같은 키가 그대로 보인다   | 번역 누락. 파일 이름(네임스페이스)과 키를 확인                                 |
| 그리드 문구만 언어가 안 바뀐다                 | `localeText`는 처음에만 적용된다 → `key={locale}`                              |
| 새 번역 파일을 만들었는데 인식이 안 된다       | `locales/<언어>/<이름>.json` 위치 확인. 개발 서버를 재시작해 본다              |

> 🧩 **확장** 지금 `t("column.name")`이 실제로 무슨 글자인지 보려면 JSON 파일을 열어 봐야 하죠? **i18n Ally** 확장을 설치하면
> 코드의 `t("...")` 옆에 번역문이 바로 표시되고, 빠진 번역도 알려 줍니다. 설치 후 `.vscode/settings.json`에
> `"i18n-ally.localesPaths": ["src/shared/i18n/locales"]`, `"i18n-ally.namespace": true`, `"i18n-ally.pathMatcher": "{locale}/{namespace}.json"`를 넣어 주세요.

다음은 메뉴를 코드가 아닌 **데이터**로 그립니다. 이 템플릿의 심장이 되는 부분입니다. → [Part 6. 데이터로 그리는 메뉴](part-06-menu.md)
