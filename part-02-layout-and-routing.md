# Part 2. 화면 뼈대 만들기

> **이 Part의 목표**: 헤더·사이드바·본문으로 나뉜 관리자 화면 뼈대를 만들고, 메뉴를 누르면 화면이 바뀌게 합니다. 그리드도 처음 띄워 봅니다.
> 이 Part가 끝나면 "관리자 화면처럼 보이는" 앱이 됩니다.

---

## Step 2-1. 폴더 구조 잡기

> 🎯 **할 일**: 앞으로 파일을 어디에 둘지 규칙을 정하고, `App.tsx`를 첫 위치로 옮깁니다.

### 왜 지금 정하나요?

파일이 몇 개 없을 때는 어디에 둬도 상관없습니다. 하지만 화면이 30개가 되면 폴더 규칙이 없는 프로젝트는 금방 뒤죽박죽이 됩니다.
종류별로 폴더를 나누는 방식(`pages/`, `api/`, `components/`)은 단순해 보이지만, 기능 하나를 고치려면 폴더 여러 곳을 오가야 합니다.
그래서 이 템플릿은 **기능별**로 모읍니다.

```
src/
├─ main.tsx      시작점
├─ app/          조립: Provider, 라우트, 레이아웃
├─ features/     업무 기능: 화면 + 그 기능만 쓰는 API·타입·부품
│  ├─ users/
│  └─ notice/
├─ shared/       공통 기반: API 통신, 테마, 다국어, 공통 UI, 유틸
└─ mocks/        가짜 서버 (Part 3)
```

규칙은 세 개입니다.

1. **위에서 아래로만 import 합니다.** `app` → `features` → `shared` 순서입니다. `shared`는 `features`를 모릅니다.
   그래서 `shared`는 다른 프로젝트에 통째로 복사해도 동작합니다.
2. **feature끼리 직접 import 하지 않습니다.** 둘 다 필요한 것은 `shared`로 올립니다.
3. **앱 코드는 `mocks`를 import 하지 않습니다.** 가짜 서버를 끄면 흔적 없이 빠져야 하기 때문입니다.

> 💡 **팁** 이 구조는 요즘 많이 쓰는 FSD(Feature-Sliced Design)를 단순화한 것입니다. 빈 폴더를 미리 만들어 두지 않습니다.
> **필요해지는 Step에서 만듭니다.**

### 따라 하기

▶️ **터미널**

```powershell
New-Item -ItemType Directory src/app
Move-Item src/App.tsx src/app/App.tsx
```

✏️ **`src/main.tsx`** — 옮긴 경로로 바꿉니다.

```tsx
import App from "@/App";
```

바꿀 코드

```tsx
import App from "@/app/App";
```

> 💡 **팁** VS Code 탐색기에서 파일을 드래그로 옮기면 "import를 업데이트할까요?"라고 묻습니다. **예**를 고르면 경로를 자동으로 고쳐 줍니다.
> `@/`로 쓴 import도 잘 따라옵니다.

✅ **커밋**

```powershell
git add .
git commit -m "refactor: App을 src/app 폴더로 이동"
```

---

## Step 2-2. Ant Design 설치

> 🎯 **할 일**: UI 컴포넌트 라이브러리를 설치하고 첫 컴포넌트를 써 봅니다.

### 왜 Ant Design인가요?

SI 화면은 폼, 표, 모달, 트리가 대부분입니다. 이걸 직접 만들면 끝이 없습니다. Ant Design(antd)은 **관리자 화면에 필요한 부품이 거의 다 있고**,
자료가 많으며, 디자인 토큰(Part 4)으로 테마를 바꾸기 쉽습니다. 대안으로는 MUI, Mantine, shadcn/ui 등이 있습니다.

### 따라 하기

▶️ **터미널**

```powershell
npm install antd@6.7.0 @ant-design/icons@6.3.4
```

> ⚠️ **주의** `@ant-design/icons`는 antd를 설치하면 함께 깔리기 때문에 설치하지 않아도 import가 됩니다. 하지만 이렇게 **간접 의존성**에 기대면
> antd 업데이트 때 갑자기 깨질 수 있습니다. **코드에서 직접 import 하는 패키지는 반드시 직접 설치합니다.**

✅ **커밋**

```powershell
git add .
git commit -m "chore: antd 의존성 추가"
```

> 💡 **팁** 라이브러리 추가는 `chore` 커밋으로 **먼저 분리**하고, 실제 사용은 다음 커밋에서 합니다.
> `package-lock.json`의 큰 변경이 기능 변경과 섞이지 않아서 나중에 읽기 쉽습니다. 앞으로도 이 순서를 지킵니다.

---

## Step 2-3. 메인 레이아웃

> 🎯 **할 일**: 헤더·사이드바·본문 세 영역을 만듭니다.

### 원리: 컴포넌트를 조립한다

React 화면은 컴포넌트를 레고처럼 끼워 맞춘 것입니다. antd `Layout`은 영역을 나누는 부품입니다.

```
<Layout>                 전체 (세로 방향)
  <Header />             위쪽 고정 영역
  <Layout>               나머지 (가로 방향)
    <Sider />            왼쪽 메뉴
    <Content />          본문
  </Layout>
</Layout>
```

### 따라 하기

먼저 화면 전체 높이를 쓸 수 있게 전역 스타일을 고칩니다.

📄 **`src/index.css`** (전체 교체)

```css
/* 브라우저가 body에 기본으로 주는 여백(8px)을 없앤다. 레이아웃이 화면에 꽉 차야 하기 때문이다. */
body {
  margin: 0;
}

/* 앱이 화면 높이를 꽉 채우도록 뿌리부터 높이를 물려준다. */
html,
body,
#root {
  height: 100%;
}
```

📄 **`src/app/layout/MainLayout.tsx`** (새 파일)

```tsx
import { useState } from "react";
import { Button, Layout, theme } from "antd";
import { MenuFoldOutlined, MenuUnfoldOutlined } from "@ant-design/icons";

const { Header, Sider, Content } = Layout;

function MainLayout() {
  const [collapsed, setCollapsed] = useState(false);
  // 색을 직접 쓰지 않고 테마 값(토큰)을 읽는다. 그래야 다크 모드에서 자동으로 바뀐다. (Part 4)
  const { token } = theme.useToken();

  return (
    <Layout style={{ height: "100%" }}>
      <Header
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
          onClick={() => setCollapsed(!collapsed)}
        />
        <strong>{import.meta.env.VITE_APP_TITLE}</strong>
      </Header>
      <Layout>
        <Sider
          theme="light"
          width={220}
          collapsedWidth={56}
          collapsed={collapsed}
          style={{ borderInlineEnd: `1px solid ${token.colorBorderSecondary}` }}
        >
          메뉴 자리
        </Sider>
        <Content style={{ padding: 16, overflow: "auto" }}>본문 자리</Content>
      </Layout>
    </Layout>
  );
}

export default MainLayout;
```

📄 **`src/app/App.tsx`** (전체 교체)

```tsx
import MainLayout from "@/app/layout/MainLayout";

function App() {
  return <MainLayout />;
}

export default App;
```

### 코드 읽어 보기: `useState`

이 Step의 핵심은 이 한 줄입니다.

```tsx
const [collapsed, setCollapsed] = useState(false);
```

- `collapsed`: 현재 값 (처음은 `false`)
- `setCollapsed`: 값을 바꾸는 함수

React의 규칙은 단순합니다. **`set` 함수로 값을 바꾸면, React가 컴포넌트 함수를 다시 실행해서 화면을 새로 그립니다.**
그래서 "버튼을 눌렀을 때 사이드바를 접어라"라고 명령하지 않습니다. "`collapsed`가 `true`면 이렇게 보여라"라고 **선언**만 해 둡니다.

```tsx
icon={collapsed ? <MenuUnfoldOutlined /> : <MenuFoldOutlined />}   // 값에 따라 다른 아이콘
onClick={() => setCollapsed(!collapsed)}                            // 클릭하면 값을 뒤집는다
```

> ⚠️ **주의** 일반 변수(`let collapsed = false`)를 바꿔도 화면은 다시 그려지지 않습니다. 화면에 영향을 주는 값은 반드시 state로 둡니다.

> ⌨️ **단축키** `Ctrl+Space`: 자동완성 띄우기. `<Button `까지 치고 누르면 쓸 수 있는 속성 목록이 나옵니다.
> `F12`: 정의로 이동. `Button` 위에서 누르면 타입 정의로 갑니다. `Alt+←`: 이전 위치로 돌아오기.

### 확인하기

- [ ] 헤더, 왼쪽 메뉴 자리, 본문 자리가 보입니다.
- [ ] 접기 버튼을 누르면 사이드바가 좁아지고 아이콘이 바뀝니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 헤더, 사이드바, 본문 레이아웃 구성"
```

---

## Step 2-4. 화면 전환 (라우팅)

> 🎯 **할 일**: 주소에 따라 다른 화면이 나오게 합니다.

### 원리: SPA와 라우터

전통적인 웹은 주소가 바뀌면 서버가 새 HTML을 줍니다. React 앱은 **HTML이 하나뿐**인 SPA(Single Page Application)입니다.
주소가 바뀌어도 새로고침하지 않고, 자바스크립트가 주소를 보고 다른 컴포넌트를 그립니다. 이 일을 해 주는 것이 **라우터**입니다.

### 따라 하기

▶️ **터미널**

```powershell
npm install react-router@8.5.1
```

✅ **커밋**

```powershell
git add .
git commit -m "chore: react-router 의존성 추가"
```

화면 세 개를 만듭니다. 업무 화면은 `features/<기능>/pages/` 아래에 둡니다. 이 위치 규칙은 Part 6에서 아주 중요해집니다.

📄 **`src/features/home/pages/HomePage.tsx`** (새 파일)

```tsx
function HomePage() {
  return <h2>홈</h2>;
}

export default HomePage;
```

📄 **`src/features/users/pages/UsersPage.tsx`** (새 파일)

```tsx
function UsersPage() {
  return <h2>사용자 관리</h2>;
}

export default UsersPage;
```

없는 주소로 들어왔을 때 보여 줄 화면도 만듭니다. antd `Result`는 결과 안내 화면을 만드는 부품입니다.

📄 **`src/app/errors/NotFoundPage.tsx`** (새 파일)

```tsx
import { Link } from "react-router";
import { Button, Result } from "antd";

function NotFoundPage() {
  return (
    <Result
      status="404"
      title="404"
      subTitle="요청하신 페이지를 찾을 수 없습니다."
      extra={
        <Link to="/">
          <Button type="primary">홈으로</Button>
        </Link>
      }
    />
  );
}

export default NotFoundPage;
```

이제 주소와 화면을 연결합니다.

📄 **`src/app/App.tsx`** (전체 교체)

```tsx
import { Route, Routes } from "react-router";
import NotFoundPage from "@/app/errors/NotFoundPage";
import MainLayout from "@/app/layout/MainLayout";
import HomePage from "@/features/home/pages/HomePage";
import UsersPage from "@/features/users/pages/UsersPage";

function App() {
  return (
    <Routes>
      {/* MainLayout 안의 <Outlet /> 자리에 아래 화면들이 그려진다 */}
      <Route element={<MainLayout />}>
        <Route index element={<HomePage />} />
        <Route path="users" element={<UsersPage />} />
        <Route path="*" element={<NotFoundPage />} />
      </Route>
    </Routes>
  );
}

export default App;
```

✏️ **`src/app/layout/MainLayout.tsx`** — 본문 자리에 `Outlet`을 넣습니다.

```tsx
        <Content style={{ padding: 16, overflow: "auto" }}>본문 자리</Content>
```

바꿀 코드

```tsx
        <Content style={{ padding: 16, overflow: "auto" }}>
          <Outlet />
        </Content>
```

✏️ **`src/app/layout/MainLayout.tsx`** — import를 추가합니다.

```tsx
import { useState } from "react";
```

바꿀 코드

```tsx
import { useState } from "react";
import { Outlet } from "react-router";
```

마지막으로 앱 전체를 라우터로 감쌉니다. 라우터 기능(`useNavigate` 등)은 `BrowserRouter` 안쪽에서만 쓸 수 있습니다.

✏️ **`src/main.tsx`** — import를 추가합니다.

```tsx
import { createRoot } from "react-dom/client";
```

바꿀 코드

```tsx
import { createRoot } from "react-dom/client";
import { BrowserRouter } from "react-router";
```

✏️ **`src/main.tsx`** — `App`을 감쌉니다.

```tsx
    <App />
```

바꿀 코드

```tsx
    <BrowserRouter>
      <App />
    </BrowserRouter>
```

> 💡 **팁** `path="*"`는 "위의 어떤 경로에도 안 맞으면"이라는 뜻입니다. 오타가 난 주소가 여기로 옵니다.

### 확인하기

- [ ] `http://localhost:5173/` → "홈"
- [ ] 주소창에 `/users` 입력 → "사용자 관리"
- [ ] 주소창에 `/abc` 입력 → 404 화면, [홈으로] 버튼 동작

✅ **커밋**

```powershell
git add .
git commit -m "feat: 라우팅 구조 구성 및 404 화면 추가"
```

---

## Step 2-5. 사이드 메뉴

> 🎯 **할 일**: 사이드바 메뉴를 클릭해서 화면을 이동합니다.

### 따라 하기

📄 **`src/app/layout/SideMenu.tsx`** (새 파일)

```tsx
import { useLocation, useNavigate } from "react-router";
import { Menu } from "antd";
import { HomeOutlined, TeamOutlined } from "@ant-design/icons";

// 지금은 코드에 직접 적는다. Part 6에서 서버 데이터로 바꾼다.
const items = [
  { key: "/", icon: <HomeOutlined />, label: "홈" },
  { key: "/users", icon: <TeamOutlined />, label: "사용자 관리" },
];

function SideMenu() {
  const { pathname } = useLocation(); // 현재 주소
  const navigate = useNavigate(); // 코드로 주소를 바꾸는 함수

  return (
    <Menu
      mode="inline"
      items={items}
      selectedKeys={[pathname]}
      onClick={({ key }) => navigate(key)}
      style={{ borderInlineEnd: "none" }}
    />
  );
}

export default SideMenu;
```

✏️ **`src/app/layout/MainLayout.tsx`** — 메뉴 자리에 `SideMenu`를 넣습니다.

```tsx
          메뉴 자리
```

바꿀 코드

```tsx
          <SideMenu />
```

✏️ **`src/app/layout/MainLayout.tsx`** — import를 추가합니다.

```tsx
import { MenuFoldOutlined, MenuUnfoldOutlined } from "@ant-design/icons";
```

바꿀 코드

```tsx
import { MenuFoldOutlined, MenuUnfoldOutlined } from "@ant-design/icons";
import SideMenu from "@/app/layout/SideMenu";
```

### 코드 읽어 보기: 훅(Hook)

`useLocation`, `useNavigate`, `useState`처럼 `use`로 시작하는 함수를 **훅**이라고 합니다. 컴포넌트에 기능을 끼워 넣는 함수입니다.
규칙은 두 가지입니다.

1. **컴포넌트(또는 다른 훅)의 맨 위에서만** 호출합니다.
2. **`if`나 `for` 안에서 호출하지 않습니다.** React가 호출 순서로 훅을 구분하기 때문입니다.

ESLint의 `react-hooks` 규칙이 위반을 잡아 주니, 빨간 줄이 생기면 메시지를 읽어 보세요.

`selectedKeys={[pathname]}`는 "지금 주소와 같은 메뉴를 강조하라"는 뜻입니다. 주소가 **단일 진실 공급원**이라서, 메뉴를 클릭하든
주소를 직접 입력하든 강조 표시가 항상 맞습니다.

> ⌨️ **단축키** `Ctrl+.`: 빠른 수정. 빨간 줄이 있는 이름 위에서 누르면 "import 추가" 같은 제안이 나옵니다.
> 위에서 `<SideMenu />`만 먼저 입력하고 `Ctrl+.`로 import를 자동으로 넣어 보세요.

### 확인하기

- [ ] 메뉴를 클릭하면 본문이 바뀌고, 주소창 주소도 바뀝니다.
- [ ] 브라우저 뒤로가기를 누르면 이전 화면으로 돌아가고 메뉴 강조도 따라옵니다.
- [ ] 사이드바를 접으면 아이콘만 남습니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 사이드 메뉴 추가"
```

---

## Step 2-6. Provider 모으기와 한국어 적용

> 🎯 **할 일**: 앱 전체를 감싸는 설정을 한 파일로 모으고, antd 문구를 한국어로 바꿉니다.

### Provider가 뭔가요?

`BrowserRouter`처럼 앱 전체를 감싸서 **안쪽 컴포넌트에 기능을 제공**하는 컴포넌트를 Provider라고 합니다.
앞으로 테마, 서버 상태 등 Provider가 계속 늘어나므로 `main.tsx`가 지저분해지기 전에 한 파일로 모읍니다.

antd도 `ConfigProvider`로 언어·테마를 받습니다. 기본 언어가 영어라서 날짜 선택창, 페이지 이동 문구 등이 영어로 나옵니다.

`App`(antd의 컴포넌트)은 알림·확인창이 테마를 제대로 따르게 해 주는 Provider입니다. 이름이 우리 `App`과 겹치므로 `AntdApp`으로 바꿔서 가져옵니다.

### 따라 하기

📄 **`src/app/AppProviders.tsx`** (새 파일)

```tsx
import type { ReactNode } from "react";
import { BrowserRouter } from "react-router";
import { App as AntdApp, ConfigProvider } from "antd";
import koKR from "antd/locale/ko_KR";

interface AppProvidersProps {
  children: ReactNode;
}

// 앱 전체를 감싸는 Provider를 한 곳에 모은다.
function AppProviders({ children }: AppProvidersProps) {
  return (
    <ConfigProvider locale={koKR}>
      <AntdApp style={{ height: "100%" }}>
        <BrowserRouter>{children}</BrowserRouter>
      </AntdApp>
    </ConfigProvider>
  );
}

export default AppProviders;
```

📄 **`src/main.tsx`** (전체 교체)

```tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import "./index.css";
import App from "@/app/App";
import AppProviders from "@/app/AppProviders";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <AppProviders>
      <App />
    </AppProviders>
  </StrictMode>,
);
```

### 코드 읽어 보기: props와 children

컴포넌트는 함수이고, **props는 그 함수의 매개변수**입니다. `children`은 컴포넌트 태그 사이에 넣은 내용을 받는 특별한 prop입니다.

```tsx
<AppProviders>    {/* 여기 사이에 넣은 것이 */}
  <App />         {/* children으로 들어간다 */}
</AppProviders>
```

TypeScript에서는 props의 모양을 `interface`로 적습니다. 이 프로젝트의 규칙은 이렇습니다.

- **객체 모양**(데이터 모델, props)은 `interface`
- **그 외**(유니온 등)는 `type`

```ts
interface User { id: number; name: string }   // 객체 모양
type Role = "ADMIN" | "USER";                  // 유니온
```

### 확인하기

- [ ] 화면이 이전과 똑같이 나옵니다. (Provider는 눈에 보이는 변화가 없습니다)

✅ **커밋**

```powershell
git add .
git commit -m "feat: AppProviders 추가 및 antd 한국어 로케일 적용"
```

---

## Step 2-7. AG Grid 첫 만남

> 🎯 **할 일**: 사용자 화면에 데이터 그리드를 띄웁니다.

### 왜 antd Table이 아니라 AG Grid인가요?

antd `Table`은 단순한 표에 좋습니다. 하지만 SI 화면은 **엑셀처럼 쓰는 표**를 요구합니다. 컬럼 크기 조절·이동·고정, 셀 편집, 수만 건 렌더링 같은 것들입니다.
AG Grid는 이런 기능을 갖춘 대표적인 그리드입니다.

> ⚠️ **주의** 우리는 무료판(Community)을 씁니다. 아래 기능은 **유료(Enterprise) 전용**입니다.
> 엑셀(.xlsx) 내보내기, 행 그룹, 마스터-디테일, 범위 선택·복사, 피벗.
> 고객이 "엑셀처럼 되는 그리드"를 요구하면 라이선스가 필요한지 먼저 확인하세요. 무료판으로 되는 것과 안 되는 것을 구분하는 게 중요합니다.

### 따라 하기

▶️ **터미널**

```powershell
npm install ag-grid-community@36.3.0 ag-grid-react@36.3.0 @ag-grid-community/locale@36.3.0
```

✅ **커밋**

```powershell
git add .
git commit -m "chore: ag-grid 의존성 추가"
```

AG Grid는 쓸 기능(모듈)을 앱 시작 시 한 번 등록해야 동작합니다.

✏️ **`src/main.tsx`** — 모듈을 등록합니다.

```tsx
import { createRoot } from "react-dom/client";
import "./index.css";
```

바꿀 코드

```tsx
import { createRoot } from "react-dom/client";
import { AllCommunityModule, ModuleRegistry } from "ag-grid-community";
import "./index.css";
```

✏️ **`src/main.tsx`** — 등록 코드를 넣습니다.

```tsx
createRoot(document.getElementById("root")!).render(
```

바꿀 코드

```tsx
// AG Grid는 쓸 기능을 앱 시작 시 한 번 등록해야 한다. 여기서는 무료 기능 전체를 등록한다.
ModuleRegistry.registerModules([AllCommunityModule]);

createRoot(document.getElementById("root")!).render(
```

> 💡 **팁** 번들 크기가 중요해지면 `AllCommunityModule` 대신 필요한 모듈만 골라 등록할 수 있습니다. 지금은 편의를 택합니다.

📄 **`src/features/users/pages/UsersPage.tsx`** (전체 교체)

```tsx
import { AgGridReact } from "ag-grid-react";
import type { ColDef } from "ag-grid-community";

interface UserRow {
  id: number;
  name: string;
  email: string;
  roleCode: string;
}

const rows: UserRow[] = [
  { id: 1, name: "김민수", email: "minsu.kim@example.com", roleCode: "ADMIN" },
  { id: 2, name: "이서연", email: "seoyeon.lee@example.com", roleCode: "USER" },
  { id: 3, name: "박지훈", email: "jihoon.park@example.com", roleCode: "MANAGER" },
];

// 컬럼 정의. field는 데이터의 어떤 속성을 보여 줄지 정한다.
const columnDefs: ColDef<UserRow>[] = [
  { headerName: "번호", field: "id", width: 90, flex: 0 },
  { headerName: "이름", field: "name" },
  { headerName: "이메일", field: "email", flex: 2 },
  { headerName: "권한", field: "roleCode" },
];

// 모든 컬럼에 공통으로 적용된다. 컬럼에 같은 옵션을 직접 쓰면 그쪽이 우선한다.
const defaultColDef: ColDef<UserRow> = { sortable: true, flex: 1 };

function UsersPage() {
  return (
    <div style={{ height: "100%", display: "flex", flexDirection: "column" }}>
      <h2 style={{ marginTop: 0 }}>사용자 관리</h2>
      {/* 그리드는 자기 높이를 스스로 정하지 않는다. 부모에게 높이를 받아야 보인다. */}
      <div style={{ flex: 1, minHeight: 0 }}>
        <AgGridReact<UserRow>
          rowData={rows}
          columnDefs={columnDefs}
          defaultColDef={defaultColDef}
        />
      </div>
    </div>
  );
}

export default UsersPage;
```

### 코드 읽어 보기

**`ColDef<UserRow>[]`의 꺾쇠**: 제네릭입니다. "이 컬럼 정의는 `UserRow` 데이터를 다룬다"고 알려 줍니다.
덕분에 `field: "nmae"` 같은 오타가 바로 빨간 줄로 잡힙니다. 직접 해 보세요.

**`flex`**: 남는 폭을 비율로 나눕니다. 이름(1)과 권한(1)이 같고 이메일(2)이 두 배로 넓습니다.
번호는 `width: 90`으로 고정했는데, `defaultColDef`의 `flex: 1`을 끄려고 `flex: 0`을 함께 적었습니다.

**컴포넌트 밖의 상수**: `columnDefs`를 함수 밖에 둔 이유는 다시 그릴 때마다 새로 만들지 않기 위해서입니다.
(Part 5에서 다국어를 넣으면 컴포넌트 안으로 들어갑니다. 왜 그런지는 그때 설명합니다.)

> ⚠️ **주의** AG Grid가 안 보이고 화면이 비어 있다면 십중팔구 **높이 문제**입니다. 그리드를 감싼 부모에 높이가 있어야 합니다.
> 그래서 위 코드는 바깥 `div`를 세로 flex로 만들고, 그리드를 감싼 `div`에 `flex: 1`을 줘서 남은 높이를 전부 받게 했습니다.
> `minHeight: 0`은 flex 자식이 내용 크기 이하로 줄어들 수 있게 해 주는 필수 옵션입니다.

마지막으로, 본문 영역이 탄력적으로 높이를 전달하도록 `Content`의 스크롤은 그대로 둡니다.
브라우저 창 크기를 줄여도 그리드가 함께 줄어드는지 확인해 보세요.

### 확인하기

- [ ] `/users`에서 3행짜리 그리드가 보입니다.
- [ ] 컬럼 제목을 클릭하면 정렬됩니다. 경계선을 드래그하면 폭이 바뀝니다. 제목을 끌면 순서가 바뀝니다.
- [ ] 다른 메뉴에 갔다가 돌아오면 정렬이 초기화됩니다. (화면이 없어졌다가 다시 만들어지기 때문입니다. Part 7의 탭이 이 문제를 해결합니다.)

### 스스로 답해 보기

<details>
<summary>Q. 왜 <code>rows</code>에 권한을 "관리자"가 아니라 "ADMIN"으로 적었을까요?</summary>

화면에 보여 줄 글자를 데이터 값으로 쓰면 다국어가 불가능하고, 이름을 바꾸면 데이터가 깨집니다.
**데이터에는 코드값**을 두고, **화면 표시는 번역·공통코드로** 변환합니다. Part 9에서 이 코드를 공통코드와 연결합니다.

</details>

✅ **커밋**

```powershell
git add .
git commit -m "feat: 사용자 화면에 AG Grid 표시"
```

---

## 🏁 Part 2를 마치며

여기까지 오면 관리자 화면의 뼈대가 완성됩니다. 화면을 캡처해 두세요.

| 배운 것             | 어디에 썼나                     |
| ------------------- | ------------------------------- |
| 컴포넌트, JSX       | 모든 화면                       |
| props, children     | `AppProviders`                  |
| state (`useState`)  | 사이드바 접기                   |
| 훅의 규칙           | `useLocation`, `useNavigate`    |
| 라우팅              | `Routes`, `Route`, `Outlet`     |
| 제네릭              | `ColDef<UserRow>`               |

지금 사용자 데이터는 코드에 직접 적혀 있습니다. 다음 Part에서는 서버가 없어도 진짜 서버처럼 동작하는 **가짜 서버**를 만들어, 데이터를 코드 밖으로 뺍니다.
→ [Part 3. 가짜 서버와 API 계층](part-03-mock-api.md)
