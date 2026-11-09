# Part 1. 프로젝트 만들기

> **이 Part의 목표**: 빈 폴더에서 시작해, 저장만 하면 코드가 자동으로 정돈되고 실수가 바로 표시되는 개발 환경을 만듭니다.
> 이 Part는 화면 변화가 거의 없습니다. 그 대신 뒤의 모든 작업이 편해집니다. 꼼꼼할 때 해 두기 좋은 구간입니다.

---

## Step 1-1. Vite로 프로젝트 생성

> 🎯 **할 일**: 명령 몇 줄로 React + TypeScript 프로젝트를 만들고 브라우저에서 띄웁니다.

### 왜 Vite인가요?

브라우저는 `.tsx` 파일을 읽지 못합니다. TypeScript를 JavaScript로 바꾸고, 수백 개의 파일을 묶어서 브라우저가 이해할 수 있는 형태로
만들어 주는 도구가 필요합니다. 이것을 **빌드 도구(번들러)** 라고 하고, Vite는 그중 가장 빠르고 설정이 적은 도구입니다.

- **개발 중**: 파일을 저장하면 바뀐 부분만 브라우저에 즉시 반영합니다. (HMR, Hot Module Replacement)
- **배포할 때**: `dist` 폴더에 최적화된 정적 파일을 만듭니다. 이 파일만 웹 서버에 올리면 됩니다.

### 따라 하기

VS Code를 열고 `` Ctrl+` ``로 터미널을 연 뒤 실행합니다. (이 가이드는 프로젝트 부모 폴더를 `C:\dev\workspace`로 가정합니다. 다른 폴더를 써도 됩니다.)

▶️ **터미널**

```powershell
cd C:\dev\workspace
npm create vite@9.3.4 si-template -- --template react-ts --eslint --no-interactive
cd si-template
npm install
code .
```

한 줄씩 뜯어 보면 이렇습니다.

| 부분                   | 뜻                                                                                  |
| ---------------------- | ----------------------------------------------------------------------------------- |
| `npm create vite@9.3.4` | Vite 프로젝트 생성기를 실행합니다. 이 가이드와 같은 결과를 얻도록 버전을 고정했습니다 |
| `si-template`          | 만들 폴더 이름                                                                      |
| `--`                   | "여기부터는 npm이 아니라 생성기에게 주는 옵션"이라는 구분선                         |
| `--template react-ts`  | React + TypeScript 템플릿                                                           |
| `--eslint`             | 코드 검사기로 ESLint를 씁니다. (안 쓰면 기본값은 Oxlint)                            |
| `--no-interactive`     | 질문하지 말고 바로 만듭니다                                                         |
| `npm install`          | `package.json`에 적힌 라이브러리를 `node_modules` 폴더에 내려받습니다               |
| `code .`               | 현재 폴더를 VS Code로 엽니다                                                        |

> 💡 **팁** `--eslint`를 쓰는 이유: Oxlint는 빠르지만, 현장에서 쓰이는 규칙과 플러그인이 가장 많은 쪽은 아직 ESLint입니다.
> 그리고 이 옵션을 쓰면 생성기가 TypeScript를 6.0으로 맞춰 줍니다. 최신은 7.0이지만 `typescript-eslint`가 아직 6.0까지만 지원하기 때문입니다.

새로 열린 VS Code 창에서 다시 `` Ctrl+` ``로 터미널을 열고 개발 서버를 켭니다.

▶️ **터미널**

```powershell
npm run dev
```

터미널에 나온 `http://localhost:5173/`을 `Ctrl+클릭`하면 브라우저가 열립니다. 끌 때는 터미널에서 `Ctrl+C`를 누릅니다.

### 만들어진 파일 둘러보기

| 파일·폴더            | 하는 일                                                                       |
| -------------------- | ----------------------------------------------------------------------------- |
| `index.html`         | 앱의 입구. 브라우저가 가장 먼저 읽습니다. `<div id="root">` 안에 앱이 그려집니다 |
| `src/main.tsx`       | 자바스크립트 시작점. `App` 컴포넌트를 `#root`에 그립니다                      |
| `src/App.tsx`        | 첫 번째 화면 컴포넌트                                                         |
| `public/`            | 가공 없이 그대로 배포되는 파일 (파비콘 등)                                    |
| `package.json`       | 프로젝트 이름, 실행 명령(`scripts`), 사용 라이브러리 목록                     |
| `package-lock.json`  | 설치된 라이브러리의 **정확한 버전** 기록. 직접 수정하지 않습니다              |
| `node_modules/`      | 내려받은 라이브러리. Git에 올리지 않습니다 (언제든 `npm install`로 복구)      |
| `tsconfig*.json`     | TypeScript 설정. `app`은 브라우저 코드용, `node`는 `vite.config.ts`용         |
| `vite.config.ts`     | Vite 설정                                                                     |
| `eslint.config.js`   | ESLint 설정                                                                   |

### 줄바꿈 문자 통일하기

Git 저장소를 만들기 전에 한 가지를 먼저 해 둡니다.

Windows는 줄바꿈으로 `CRLF`(`\r\n`)를, 맥·리눅스·서버는 `LF`(`\n`)를 씁니다. 이게 섞이면 내용은 그대로인데 Git이 파일 전체가 바뀌었다고 표시합니다.
협업 시 아주 흔한 사고라서, **프로젝트 시작할 때 `LF`로 고정**합니다.

📄 **`.gitattributes`** (새 파일)

```
* text=auto eol=lf
*.png binary
*.jpg binary
*.ico binary
```

> ⌨️ **단축키** VS Code에서 새 파일 만들기: 탐색기에서 프로젝트 이름 위에 마우스를 올리면 나오는 📄+ 아이콘.
> 또는 `Ctrl+N`으로 빈 파일을 만들고 `Ctrl+S`로 경로를 지정합니다.

### 첫 커밋

`Ctrl+C`로 개발 서버를 끄고 Git 저장소를 만듭니다.

✅ **커밋**

```powershell
git init
git add .
git commit -m "chore: 프로젝트 초기 생성 (Vite)"
```

### 커밋 메시지 규칙

이 가이드의 커밋 메시지는 **Conventional Commits** 형식을 따릅니다. `<type>: <subject>` 형태이고, type은 아래 8개만 씁니다.

| type       | 언제                                                                           |
| ---------- | ------------------------------------------------------------------------------ |
| `feat`     | 새로운 기능 추가                                                               |
| `fix`      | 버그 수정                                                                      |
| `docs`     | 문서 수정 (코드 변경 없음)                                                     |
| `style`    | 코드 포맷팅, 세미콜론 등 (논리 변경 없음). **화면 CSS 작업이 아닙니다**        |
| `refactor` | 리팩터링 (기능 변화 없음)                                                      |
| `perf`     | 성능 개선                                                                      |
| `test`     | 테스트 코드                                                                    |
| `chore`    | 빌드, 패키지, 설정 등 기타 (의존성 추가 포함)                                  |

- type은 영어, subject는 한글로 씁니다. 50자 이내, 끝에 마침표 없음.
- **한 커밋에는 한 가지 목적만** 담습니다. 나중에 되돌리거나 찾을 때 훨씬 쉽습니다.

### 스스로 답해 보기

<details>
<summary>Q. <code>node_modules</code>를 지워도 괜찮은 이유는?</summary>

`package.json`과 `package-lock.json`에 정확한 버전이 기록되어 있어서, `npm install`이면 똑같이 복구됩니다.
그래서 Git에도 올리지 않습니다. 이상한 오류가 날 때 `node_modules`를 지우고 다시 설치하는 것은 흔한 해결법입니다.

</details>

---

## Step 1-2. 샘플 코드 정리

> 🎯 **할 일**: 템플릿이 넣어 준 샘플을 지우고 깨끗한 출발점을 만듭니다.

▶️ **터미널**

```powershell
Remove-Item src/App.css
Remove-Item src/assets -Recurse
Remove-Item public/icons.svg
```

📄 **`src/App.tsx`** (전체 교체)

```tsx
function App() {
  return <h1>SI Template</h1>;
}

export default App;
```

📄 **`src/index.css`** (전체 교체)

```css
/* 브라우저가 body에 기본으로 주는 여백(8px)을 없앤다. 레이아웃이 화면에 꽉 차야 하기 때문이다. */
body {
  margin: 0;
}
```

✏️ **`index.html`** — 언어를 한국어로 바꿉니다.

```html
<html lang="en">
```

바꿀 코드

```html
<html lang="ko">
```

### 코드 읽어 보기

`src/App.tsx`는 이 프로젝트의 모든 화면이 따르는 가장 작은 형태입니다.

```tsx
function App() {                   // 컴포넌트 = 화면 조각을 돌려주는 함수. 이름은 대문자로 시작
  return <h1>SI Template</h1>;     // HTML처럼 보이는 이 문법이 JSX
}

export default App;                // 다른 파일에서 import 할 수 있게 내보낸다
```

`src/main.tsx`를 열어 보면 이 컴포넌트가 화면에 그려지는 과정을 볼 수 있습니다.

```tsx
createRoot(document.getElementById('root')!).render(  // index.html의 <div id="root">를 찾아서
  <StrictMode>                                         // (개발 중 실수를 찾아 주는 검사 모드)
    <App />                                            // 그 안에 App을 그린다
  </StrictMode>,
)
```

> 💡 **팁** `getElementById('root')!`의 `!`는 TypeScript에게 "이 값은 절대 null이 아니야"라고 알려 주는 표시입니다.
> 틀리면 실행 중 오류가 나므로, 확실할 때만 씁니다.

> ⚠️ **주의** `<StrictMode>`는 개발 모드에서 컴포넌트를 **일부러 두 번 실행**합니다. 부수 효과 실수를 찾기 위해서입니다.
> 나중에 `console.log`가 두 번 찍혀도 놀라지 마세요. 운영 빌드에서는 한 번만 실행됩니다.

### 확인하기

- [ ] `npm run dev` → 브라우저에 `SI Template` 글자만 보입니다.
- [ ] `src/App.tsx`의 글자를 고치고 저장하면 새로고침 없이 바뀝니다. 이것이 HMR입니다.

✅ **커밋**

```powershell
git add .
git commit -m "chore: vite 샘플 소스 및 리소스 제거"
```

---

## Step 1-3. Prettier로 코드 모양 통일

> 🎯 **할 일**: 저장할 때 코드 모양이 자동으로 정돈되게 합니다.

### 왜 필요한가요?

따옴표를 `'`로 쓸지 `"`로 쓸지, 세미콜론을 붙일지 같은 문제로 고민하는 시간은 아깝습니다.
Prettier는 이런 **모양 문제를 도구에 맡기는** 포매터입니다. 팀 전체 코드 모양이 같아져서, Git 변경 내역에는 진짜 바뀐 내용만 남습니다.

### 따라 하기

▶️ **터미널**

```powershell
npm install -D prettier@3.9.10
```

> 💡 **팁** `-D`는 `--save-dev`의 줄임입니다. **개발할 때만 필요한** 도구라는 뜻이고, `package.json`의 `devDependencies`에 들어갑니다.
> 이 가이드는 재현성을 위해 설치 명령에 버전을 고정합니다. 실제 프로젝트에서는 `npm outdated`로 새 버전을 확인하고 올리면 됩니다.

📄 **`.prettierrc.json`** (새 파일)

```json
{
  "printWidth": 100
}
```

기본값은 한 줄 80자인데, 요즘 모니터와 JSX 코드에는 좁아서 100으로 넓힙니다. 나머지는 Prettier 기본값을 그대로 씁니다.
(규칙을 많이 바꿀수록 논쟁 거리가 늘어납니다. 기본값이 곧 "모두가 익숙한 모양"입니다.)

📄 **`.prettierignore`** (새 파일)

```
dist
node_modules
package-lock.json
public/mockServiceWorker.js
```

마지막 줄의 파일은 Part 3에서 자동 생성되는 파일입니다. 미리 제외해 둡니다.

✏️ **`package.json`** — 실행 명령을 추가합니다.

```json
    "preview": "vite preview"
```

바꿀 코드

```json
    "preview": "vite preview",
    "format": "prettier --write .",
    "format:check": "prettier --check ."
```

✅ **커밋**

```powershell
git add .
git commit -m "chore: Prettier 설치 및 설정 추가"
```

이제 한 번 실행합니다. 템플릿 코드의 작은따옴표가 큰따옴표로, 세미콜론이 붙는 등 전체 모양이 바뀝니다.

▶️ **터미널**

```powershell
npm run format
```

> 💡 **팁** 이렇게 **모양만 바꾼 변경**은 기능 변경과 섞지 말고 따로 커밋합니다. 그래야 나중에 진짜 변경을 찾기 쉽습니다.
> 그래서 type도 `style`입니다.

✅ **커밋**

```powershell
git add .
git commit -m "style: Prettier로 기존 파일 포맷 정리"
```

### 저장할 때 자동으로

매번 `npm run format`을 치는 건 번거롭습니다. VS Code가 저장할 때마다 해 주도록 합니다.

> 🧩 **확장** 지금 터미널에서 명령을 쳐야 포맷이 됩니다. `Ctrl+Shift+X`로 확장 탭을 열고 **Prettier - Code formatter**를 설치하면,
> 아래 설정과 합쳐져서 `Ctrl+S`만 눌러도 정돈됩니다.

📄 **`.vscode/settings.json`** (새 파일)

```json
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true,
  "files.eol": "\n"
}
```

📄 **`.vscode/extensions.json`** (새 파일)

```json
{
  "recommendations": ["esbenp.prettier-vscode"]
}
```

`extensions.json`은 "이 프로젝트에 권장하는 확장" 목록입니다. 팀원이 프로젝트를 열면 VS Code가 설치를 제안합니다.

✏️ **`.gitignore`** — VS Code 설정 파일을 Git에 포함시킵니다. (다른 개발자도 같은 설정을 쓰도록)

```
!.vscode/extensions.json
```

바꿀 코드

```
!.vscode/extensions.json
!.vscode/settings.json
```

### 확인하기

- [ ] `src/App.tsx`에서 `return` 앞 공백을 마구 넣고 `Ctrl+S` → 원래대로 돌아옵니다.
- [ ] VS Code 오른쪽 아래 상태 표시줄에 `LF`가 보입니다. (`CRLF`로 보이는 파일은 그 글자를 클릭해서 `LF`로 바꿉니다)

✅ **커밋**

```powershell
git add .
git commit -m "chore: VS Code 저장 시 자동 포맷 설정"
```

---

## Step 1-4. ESLint로 실수 잡기

> 🎯 **할 일**: 코드의 실수를 편집기에서 바로 보이게 합니다.

### Prettier와 뭐가 다른가요?

| 도구     | 보는 것                                                          |
| -------- | ---------------------------------------------------------------- |
| Prettier | **모양**. 들여쓰기, 따옴표, 줄바꿈                               |
| ESLint   | **의미**. 안 쓰는 변수, 훅 규칙 위반, 잘못된 의존성 배열 등 실수 |

ESLint는 프로젝트를 만들 때 이미 설치됐습니다. 직접 느껴 봅시다.

### 불편함을 먼저 겪어 보기

`src/App.tsx`를 잠깐 아래처럼 고칩니다. (이 코드는 커밋하지 않습니다.)

```tsx
function App() {
  const unused = 1;
  return <h1>SI Template</h1>;
}
```

저장해도 VS Code에서는 아무 표시가 없습니다. 터미널에서 `npm run lint`를 쳐야 비로소 오류가 나옵니다.

```
  2:9  error  'unused' is assigned a value but never used  @typescript-eslint/no-unused-vars
```

명령을 칠 때까지 실수를 모른다는 게 불편합니다.

> 🧩 **확장** `Ctrl+Shift+X` → **ESLint** (Microsoft) 설치. 설치 후 위 코드를 다시 보면 `unused` 아래에 **빨간 물결**이 생기고,
> 마우스를 올리면 이유가 나옵니다. 터미널을 볼 필요가 없어집니다.

방금 쓴 `const unused = 1;` 줄은 지웁니다.

> ⌨️ **단축키** `Ctrl+Shift+K`: 현재 줄 삭제. `Ctrl+Z`: 되돌리기. `Ctrl+Shift+M`: 문제(오류) 목록 패널 열기. `F8`: 다음 오류로 이동.

### 저장할 때 고칠 수 있는 건 고치기

✏️ **`.vscode/settings.json`** — 저장할 때 ESLint가 자동 수정 가능한 항목을 고치게 합니다.

```json
  "editor.formatOnSave": true,
```

바꿀 코드

```json
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
```

✏️ **`.vscode/extensions.json`** — 권장 확장에 ESLint를 추가합니다.

```json
  "recommendations": ["esbenp.prettier-vscode"]
```

바꿀 코드

```json
  "recommendations": ["esbenp.prettier-vscode", "dbaeumer.vscode-eslint"]
```

### 확인하기

- [ ] `npm run lint`가 아무 메시지 없이 끝납니다.

✅ **커밋**

```powershell
git add .
git commit -m "chore: VS Code ESLint 연동 설정"
```

---

## Step 1-5. `@/` 경로 별칭

> 🎯 **할 일**: `../../../shared/api`처럼 긴 상대 경로 대신 `@/shared/api`로 쓸 수 있게 합니다.

### 왜 필요한가요?

폴더가 깊어지면 import가 이렇게 됩니다.

```ts
import { httpClient } from "../../../shared/api/httpClient"; // 몇 단계 올라가는지 세야 한다
import { httpClient } from "@/shared/api/httpClient"; // src 기준. 파일을 옮겨도 그대로
```

### 원리: 경로를 해석하는 주체가 둘이다

이 프로젝트에서 import 경로를 읽는 프로그램은 둘입니다.

| 프로그램             | 언제                | 하는 일                           |
| -------------------- | ------------------- | --------------------------------- |
| **TypeScript**       | 코드 작성·타입 검사 | 경로를 따라가서 타입을 확인합니다 |
| **Vite (번들러)**    | 실행·빌드           | 경로를 따라가서 코드를 묶습니다   |

그래서 두 곳 모두 별칭을 알아야 합니다. 예전에는 양쪽에 따로 적었지만, Vite 8은 `resolve.tsconfigPaths` 옵션으로 **tsconfig에 적힌 경로를 그대로 읽어 줍니다.**
설정을 한 곳에만 둘 수 있으니 "바뀔 수 있는 것은 한 곳에서만" 원칙에도 맞습니다.

### 따라 하기

✏️ **`tsconfig.app.json`** — 별칭을 정의합니다.

```json
    "jsx": "react-jsx",
```

바꿀 코드

```json
    "jsx": "react-jsx",

    /* 경로 별칭: "@/"는 src 폴더를 뜻한다 */
    "paths": {
      "@/*": ["./src/*"]
    },
```

✏️ **`vite.config.ts`** — Vite가 tsconfig의 경로를 읽게 합니다.

```ts
  plugins: [react()],
```

바꿀 코드

```ts
  plugins: [react()],
  resolve: {
    tsconfigPaths: true,
  },
```

바로 써 봅니다.

✏️ **`src/main.tsx`** — `App` import를 별칭으로 바꿉니다.

```tsx
import App from "./App.tsx";
```

바꿀 코드

```tsx
import App from "@/App";
```

> 💡 **팁** `vite.config.ts`나 `tsconfig` 파일을 고친 뒤에는 개발 서버를 껐다 켜야 합니다. (Vite 설정은 자동 재시작되지만 tsconfig는 아닐 수 있습니다.)
> VS Code가 `@/`를 못 찾는다고 빨간 줄을 띄우면 `Ctrl+Shift+P` → **TypeScript: Restart TS Server**를 실행합니다.

### 확인하기

- [ ] `npm run build`가 성공합니다.
- [ ] `npm run dev` → 화면이 그대로 보입니다.

### 스스로 답해 보기

<details>
<summary>Q. <code>tsconfig</code>에만 설정하고 <code>resolve.tsconfigPaths</code>를 안 켜면 어떤 일이 생길까요?</summary>

편집기에서는 빨간 줄이 없고 타입 검사도 통과하지만, 실행하면 Vite가 `@/App`을 찾지 못해 오류가 납니다.
TypeScript는 타입 검사만 하고, 실제로 파일을 찾아서 묶는 것은 Vite이기 때문입니다.

</details>

✅ **커밋**

```powershell
git add .
git commit -m "chore: @ 경로 별칭 설정"
```

---

## Step 1-6. 환경 변수

> 🎯 **할 일**: 개발·운영에 따라 달라지는 값을 코드 밖으로 뺍니다.

### 왜 필요한가요?

API 주소, 가짜 서버 사용 여부처럼 **환경마다 달라지는 값**을 코드에 박아 두면 배포할 때마다 코드를 고쳐야 합니다.
이런 값은 `.env` 파일에 두고 코드에서는 이름으로만 읽습니다.

| 파일               | 언제 읽히나                          |
| ------------------ | ------------------------------------ |
| `.env`             | 항상 (공통 기본값)                   |
| `.env.development` | `npm run dev`                        |
| `.env.production`  | `npm run build`                      |
| `.env.local`       | 항상, 내 PC 전용 (Git에서 제외됨)    |

뒤에 있는 파일이 앞의 값을 덮어씁니다.

### 따라 하기

📄 **`.env`** (새 파일)

```
VITE_APP_TITLE=SI Template
VITE_API_BASE_URL=/api
VITE_USE_MOCK=false
VITE_MOCK_DELAY=300
```

📄 **`.env.development`** (새 파일)

```
# 개발 중에는 가짜 서버를 켠다 (Part 3)
VITE_USE_MOCK=true
```

코드에서 `import.meta.env.VITE_APP_TITLE`처럼 읽는데, 타입을 알려 주면 자동완성이 되고 오타도 잡힙니다.

📄 **`src/env.d.ts`** (새 파일)

```ts
// .env 파일의 값에 타입을 붙인다. 새 변수를 추가하면 여기에도 추가한다.
interface ImportMetaEnv {
  readonly VITE_APP_TITLE: string;
  readonly VITE_API_BASE_URL: string;
  readonly VITE_USE_MOCK: string;
  readonly VITE_MOCK_DELAY: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

> ⚠️ **주의** 환경 변수의 값은 **전부 문자열**입니다. `VITE_USE_MOCK=false`는 불리언이 아니라 글자 `"false"`입니다.
> `if (import.meta.env.VITE_USE_MOCK)`처럼 쓰면 `"false"`도 참이 되어 버립니다. 반드시 `=== "true"`로 비교합니다.

> ⚠️ **주의** `VITE_`로 시작하는 변수는 빌드할 때 JS 파일에 **그대로 박힙니다.** 브라우저에서 누구나 볼 수 있습니다.
> 비밀번호나 API 키를 넣으면 안 됩니다. `VITE_` 접두사는 "공개해도 되는 값만 넘긴다"는 안전장치입니다.

브라우저 탭 제목에도 써 봅니다. Vite는 `index.html` 안의 `%변수명%`을 값으로 바꿔 줍니다.

✏️ **`index.html`** — 제목을 환경 변수로 바꿉니다.

```html
    <title>si-template</title>
```

바꿀 코드

```html
    <title>%VITE_APP_TITLE%</title>
```

✏️ **`src/App.tsx`** — 화면에도 환경 변수 값을 표시합니다.

```tsx
  return <h1>SI Template</h1>;
```

바꿀 코드

```tsx
  return <h1>{import.meta.env.VITE_APP_TITLE}</h1>;
```

`{ }`는 JSX 안에서 "여기부터는 JavaScript"라는 표시입니다.

### 확인하기

- [ ] `npm run dev`를 다시 시작합니다. (`.env`를 고치면 재시작이 필요합니다)
- [ ] `.env`의 `VITE_APP_TITLE`을 바꾸면 화면 글자와 브라우저 탭 제목이 함께 바뀝니다. 확인했으면 원래대로 돌립니다.

### 스스로 답해 보기

<details>
<summary>Q. 프론트엔드에는 왜 비밀값을 숨길 방법이 없을까요?</summary>

프론트엔드 코드는 결국 사용자의 브라우저로 내려받아져 실행됩니다. 내려받은 파일은 누구나 열어 볼 수 있습니다.
비밀값이 필요한 작업은 반드시 서버가 해야 합니다.

</details>

✅ **커밋**

```powershell
git add .
git commit -m "chore: 환경 변수 파일 및 타입 정의 추가"
```

---

## Part 1을 마치며

지금까지 만든 것을 돌아봅시다.

- 저장하면 코드 모양이 자동으로 정돈됩니다. (Prettier)
- 실수는 편집기에 빨간 줄로 바로 보입니다. (ESLint)
- `@/`로 import 할 수 있습니다.
- 환경별 설정이 코드 밖에 있습니다.

`git log --oneline`을 쳐 보세요. 작고 목적이 분명한 커밋이 쌓여 있을 겁니다.

> 💡 **팁** 이 프로젝트에서 Claude Code를 함께 쓴다면, 프로젝트 폴더에 `CLAUDE.md`를 두어 커밋 규칙과 코드 규칙을 적어 두세요.
> 기존 `si-react-template`의 `CLAUDE.md`를 복사해서 프로젝트 이름만 바꿔도 됩니다.

다음은 드디어 화면을 만듭니다. → [Part 2. 화면 뼈대](part-02-layout-and-routing.md)
