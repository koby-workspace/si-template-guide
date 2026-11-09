# Part 13. 실무 기능 모음

> **이 Part의 목표**: SI 프로젝트라면 거의 반드시 나오는 기능과 화면을 채웁니다.
> 메뉴 시드에 미리 넣어 둔 "준비 중" 화면들(입력 폼, 그리드 편집, 파일 업로드, 내 정보 수정, 대시보드)이 여기서 완성됩니다.

### 이 Part를 진행하는 법

- **Step 13-1은 먼저 하세요.** 뒤 Step들이 쓰는 도우미 함수가 들어 있습니다.
- 나머지는 **관심 있는 것부터 골라서** 해도 됩니다. 각 Step은 되도록 새 파일만 만들고, 기존 파일은 Part 12까지의 모양을 기준으로 고칩니다.
  다만 같은 파일을 고치는 Step이 몇 개 있으므로, 건너뛴 Step이 있으면 ✏️의 "찾을 코드"가 조금 다를 수 있습니다. 그럴 때는 위치만 맞춰 넣으면 됩니다.
- 이 가이드의 검증은 **위에서부터 순서대로** 했습니다. 막히면 순서대로 하는 것이 가장 확실합니다.

| Step  | 기능                         | 쓰는 곳                   |
| ----- | ---------------------------- | ------------------------- |
| 13-1  | 포맷·마스킹·검증 유틸        | (모든 곳)                 |
| 13-2  | 입력 폼 샘플                 | 샘플 › 화면 패턴 › 입력 폼 |
| 13-3  | 그리드 편집 샘플 (일괄 저장) | 샘플 › 화면 패턴 › 그리드 편집 |
| 13-4  | 파일 업로드·다운로드         | 샘플 › 파일 업로드        |
| 13-5  | 엑셀 다운로드                | 사용자 관리               |
| 13-6  | 내 정보 수정·비밀번호 변경   | 내 정보 › 내 정보 수정    |
| 13-7  | 메뉴 검색 (Ctrl+K, 초성)     | 헤더                      |
| 13-8  | 즐겨찾기·최근 메뉴           | 화면 제목, 시작 안내      |
| 13-9  | 대시보드                     | 대시보드                  |
| 13-10 | 세션 만료 안내               | 헤더                      |
| 13-11 | 전역 로딩 표시줄             | 화면 맨 위                |
| 13-12 | 개발자 패널                  | 오른쪽 아래 (개발 모드)   |

---

## Step 13-1. 포맷·마스킹·검증 유틸

> 🎯 **할 일**: 금액·전화번호·사업자번호 표시, 개인정보 가리기(마스킹), 사업자번호 검증, 파일 저장 도우미를 만듭니다.

### 원리: 순수 함수

이 Step의 함수들은 모두 **순수 함수**입니다. 같은 입력이면 항상 같은 결과를 내고, 바깥(화면, 서버, 전역 변수)에 아무 영향도 주지 않습니다.
그래서 만들기 쉽고, 어디서든 안심하고 쓸 수 있고, 나중에 테스트를 배울 때 **첫 테스트 대상**으로 딱 좋습니다.

### 따라 하기

✏️ **`src/shared/lib/format.ts`** — 파일 끝에 함수를 추가합니다.

```ts
/** "2026-11-07T09:30:00Z" → "2026-11-07 18:30" (내 PC 시간대로) */
export function formatDateTime(value?: string | null) {
  return value ? dayjs(value).format("YYYY-MM-DD HH:mm") : "";
}
```

바꿀 코드

```ts
/** "2026-11-07T09:30:00Z" → "2026-11-07 18:30" (내 PC 시간대로) */
export function formatDateTime(value?: string | null) {
  return value ? dayjs(value).format("YYYY-MM-DD HH:mm") : "";
}

/** 1234567 → "1,234,567" */
export function formatNumber(value?: number | null) {
  return value === undefined || value === null ? "" : value.toLocaleString();
}

/** 숫자만 남긴다. "010-1234 5678" → "01012345678" */
export const onlyDigits = (value = "") => value.replace(/\D/g, "");

/** 입력 중인 전화번호에 하이픈을 넣는다. "0212345678" → "02-1234-5678", "01012345678" → "010-1234-5678" */
export function formatPhone(value = "") {
  const digits = onlyDigits(value).slice(0, 11);
  if (digits.startsWith("02")) {
    // 서울 지역번호는 두 자리다
    return digits.replace(/^(\d{2})(\d{0,4})(\d{0,4}).*$/, (_, a, b, c) =>
      [a, b, c].filter(Boolean).join("-"),
    );
  }
  const middle = digits.length > 10 ? 4 : 3;
  return digits.replace(new RegExp(`^(\\d{3})(\\d{0,${middle}})(\\d{0,4}).*$`), (_, a, b, c) =>
    [a, b, c].filter(Boolean).join("-"),
  );
}

/** "1234567890" → "123-45-67890" (사업자등록번호) */
export function formatBizNo(value = "") {
  const digits = onlyDigits(value).slice(0, 10);
  return digits.replace(/^(\d{3})(\d{0,2})(\d{0,5}).*$/, (_, a, b, c) =>
    [a, b, c].filter(Boolean).join("-"),
  );
}

/** 이름 가운데를 가린다. "홍길동" → "홍*동", "남궁민수" → "남**수", "홍길" → "홍*" */
export function maskName(name = "") {
  if (name.length <= 1) return name;
  if (name.length === 2) return `${name[0]}*`;
  return `${name[0]}${"*".repeat(name.length - 2)}${name.at(-1)}`;
}

/** "010-1234-5678" → "010-****-5678" */
export function maskPhone(phone = "") {
  return phone.replace(
    /^(\d{2,3})-(\d{3,4})-(\d{4})$/,
    (_, a, b, c) => `${a}-${"*".repeat(b.length)}-${c}`,
  );
}

/** "minsu.kim@example.com" → "mi*******@example.com" */
export function maskEmail(email = "") {
  const [id, domain] = email.split("@");
  if (!domain) return email;
  return `${id.slice(0, 2)}${"*".repeat(Math.max(id.length - 2, 0))}@${domain}`;
}
```

📄 **`src/shared/lib/validators.ts`** (새 파일)

```ts
import { onlyDigits } from "@/shared/lib/format";

/** 입력 형식 정규식. antd Form의 rules에 { pattern } 으로 쓴다. */
export const patterns = {
  phone: /^0\d{1,2}-\d{3,4}-\d{4}$/,
  bizNo: /^\d{3}-\d{2}-\d{5}$/,
};

/**
 * 사업자등록번호 검증. 형식뿐 아니라 마지막 자리(검증 번호)까지 계산해서 확인한다.
 * 규칙: 앞 9자리에 1,3,7,1,3,7,1,3,5를 곱해 더하고, 9번째 자리×5의 십의 자리를 더한 뒤,
 *       (10 - 합계 % 10) % 10 이 마지막 자리와 같아야 한다.
 */
export function isValidBizNo(value = "") {
  const digits = onlyDigits(value).split("").map(Number);
  if (digits.length !== 10) return false;
  const weights = [1, 3, 7, 1, 3, 7, 1, 3, 5];
  let sum = weights.reduce((acc, weight, index) => acc + digits[index] * weight, 0);
  sum += Math.floor((digits[8] * 5) / 10);
  return (10 - (sum % 10)) % 10 === digits[9];
}
```

파일 저장 도우미입니다. 엑셀·첨부파일 다운로드가 함께 씁니다.

📄 **`src/shared/lib/saveBlob.ts`** (새 파일)

```ts
/**
 * 브라우저 메모리의 파일(Blob)을 사용자 PC에 저장한다.
 * 임시 주소(object URL)를 만들어 <a download>를 누른 것처럼 하고, 바로 주소를 해제한다.
 */
export function saveBlob(blob: Blob, fileName: string) {
  const url = URL.createObjectURL(blob);
  const link = document.createElement("a");
  link.href = url;
  link.download = fileName;
  link.click();
  URL.revokeObjectURL(url); // 해제하지 않으면 다운로드할 때마다 메모리가 쌓인다
}
```

### 코드 읽어 보기

**정규식 `replace`의 콜백**: `(_, a, b, c) => ...`에서 `_`는 전체 일치, `a`·`b`·`c`는 괄호로 묶은 부분입니다. 입력 중에는 뒤쪽이 비어 있으므로 `filter(Boolean)`으로 빈 조각을 빼고 하이픈으로 잇습니다.
그래서 "0101" 까지만 쳐도 "010-1"처럼 자연스럽게 보입니다.

> 💡 **팁** 직접 실험해 보세요. F12 → Console은 모듈을 직접 import 할 수 없으니, 개발 서버가 켜진 상태에서
> `const f = await import("/src/shared/lib/format.ts"); f.formatPhone("01012345678")` 처럼 Vite 경로로 불러오면 됩니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 포맷·마스킹·검증 유틸 추가"
```

---

## Step 13-2. 입력 폼 샘플

> 🎯 **할 일**: SI 입력 화면에 자주 나오는 입력 요소를 한 화면에 모은 샘플을 만듭니다. 새 입력 화면을 만들 때 복사해 쓰는 용도입니다.

### 들어가는 것

| 입력          | 기법                                                         |
| ------------- | ------------------------------------------------------------ |
| 전화번호      | 입력하는 동안 하이픈 자동 (`normalize`)                      |
| 사업자번호    | 자동 하이픈 + 검증 공식 (`validator`)                        |
| 금액          | 천 단위 쉼표 (`formatter` / `parser`)                        |
| 부서·성별     | 공통코드 (`CodeSelect`, `CodeRadio`)                         |
| 날짜·기간     | `DatePicker`, `RangePicker` (dayjs 객체)                     |
| 비밀번호 확인 | 다른 칸과 비교 (`dependencies` + 함수형 규칙)                |
| 동의          | 체크해야만 저장 (`valuePropName="checked"` + 규칙)           |

### 따라 하기

샘플 화면들의 문구를 한 파일에 모읍니다. 이 Part의 샘플(입력 폼, 그리드 편집, 파일)이 함께 씁니다.

📄 **`src/shared/i18n/locales/ko/samples.json`** (새 파일)

```json
{
  "form": {
    "basic": "기본 정보",
    "detail": "상세 정보",
    "name": "이름",
    "gender": "성별",
    "dept": "부서",
    "phone": "전화번호",
    "bizNo": "사업자등록번호",
    "bizNoInvalid": "올바른 사업자등록번호가 아닙니다.",
    "email": "이메일",
    "birthDate": "생년월일",
    "period": "계약 기간",
    "salary": "연봉",
    "password": "비밀번호",
    "passwordConfirm": "비밀번호 확인",
    "passwordMismatch": "비밀번호가 일치하지 않습니다.",
    "memo": "메모",
    "agree": "개인정보 수집에 동의합니다.",
    "agreeRequired": "동의가 필요합니다.",
    "result": "입력 결과 (마스킹 적용)"
  },
  "grid": {
    "help": "셀을 더블클릭해 수정하고 [저장]으로 한꺼번에 반영합니다. 초록=추가, 주황=수정, 취소선=삭제",
    "name": "상품명",
    "category": "분류",
    "price": "가격",
    "stock": "재고",
    "addRow": "행 추가",
    "deleteRow": "행 삭제",
    "undo": "변경 취소",
    "noChanges": "변경된 내용이 없습니다.",
    "saved": "추가 {{created}}건, 수정 {{updated}}건, 삭제 {{deleted}}건을 저장했습니다."
  },
  "file": {
    "drag": "파일을 끌어다 놓거나 클릭해서 고르세요.",
    "hint": "10MB 이하, 이미지·PDF·엑셀·워드·텍스트 파일만 올릴 수 있습니다.",
    "tooLarge": "10MB를 넘는 파일은 올릴 수 없습니다.",
    "badType": "올릴 수 없는 파일 형식입니다.",
    "uploaded": "{{name}} 업로드 완료",
    "name": "파일명",
    "size": "크기",
    "uploadedAt": "올린 시각",
    "download": "다운로드",
    "memoryOnly": "가짜 서버는 파일을 메모리에만 들고 있어서, 새로고침하면 목록이 사라집니다."
  }
}
```

📄 **`src/shared/i18n/locales/en/samples.json`** (새 파일)

```json
{
  "form": {
    "basic": "Basic information",
    "detail": "Details",
    "name": "Name",
    "gender": "Gender",
    "dept": "Department",
    "phone": "Phone",
    "bizNo": "Business registration no.",
    "bizNoInvalid": "Invalid business registration number.",
    "email": "Email",
    "birthDate": "Date of birth",
    "period": "Contract period",
    "salary": "Annual salary",
    "password": "Password",
    "passwordConfirm": "Confirm password",
    "passwordMismatch": "Passwords do not match.",
    "memo": "Memo",
    "agree": "I agree to the collection of personal information.",
    "agreeRequired": "You must agree.",
    "result": "Result (masked)"
  },
  "grid": {
    "help": "Double-click a cell to edit, then press Save to apply all changes. Green=new, orange=edited, strikethrough=deleted",
    "name": "Product",
    "category": "Category",
    "price": "Price",
    "stock": "Stock",
    "addRow": "Add row",
    "deleteRow": "Delete row",
    "undo": "Discard changes",
    "noChanges": "No changes to save.",
    "saved": "Saved: {{created}} added, {{updated}} updated, {{deleted}} deleted."
  },
  "file": {
    "drag": "Drag files here or click to choose.",
    "hint": "Up to 10MB. Images, PDF, Excel, Word, and text files only.",
    "tooLarge": "Files larger than 10MB cannot be uploaded.",
    "badType": "This file type is not allowed.",
    "uploaded": "{{name}} uploaded",
    "name": "File name",
    "size": "Size",
    "uploadedAt": "Uploaded",
    "download": "Download",
    "memoryOnly": "The mock server keeps files in memory only, so the list disappears on reload."
  }
}
```

📄 **`src/features/samples/pages/FormSamplePage.tsx`** (새 파일)

```tsx
import { useState } from "react";
import { useTranslation } from "react-i18next";
import {
  Button,
  Card,
  Checkbox,
  Col,
  DatePicker,
  Descriptions,
  Flex,
  Form,
  Input,
  InputNumber,
  Modal,
  Row,
} from "antd";
import type { Dayjs } from "dayjs";
import CodeRadio from "@/shared/codes/CodeRadio";
import CodeSelect from "@/shared/codes/CodeSelect";
import { formatBizNo, formatPhone, maskEmail, maskName, maskPhone } from "@/shared/lib/format";
import { isValidBizNo, patterns } from "@/shared/lib/validators";
import { useTabDirty } from "@/shared/tabs/useTabDirty";
import PageContainer from "@/shared/ui/PageContainer";

interface SampleForm {
  name: string;
  gender: string;
  dept?: string;
  phone?: string;
  bizNo?: string;
  email?: string;
  birthDate?: Dayjs;
  period?: [Dayjs, Dayjs];
  salary?: number;
  password: string;
  passwordConfirm: string;
  memo?: string;
  agree: boolean;
}

function FormSamplePage() {
  const { t } = useTranslation("samples");
  const [form] = Form.useForm<SampleForm>();
  const [dirty, setDirty] = useState(false);
  const [result, setResult] = useState<SampleForm | null>(null);
  useTabDirty(dirty);

  const required = (field: string) => ({
    required: true,
    message: t("common:validation.required", { field }),
  });

  return (
    <PageContainer>
      <Form
        form={form}
        name="formSample"
        layout="vertical"
        initialValues={{ gender: "M", agree: false }}
        onValuesChange={() => setDirty(true)}
        onFinish={(values) => {
          setResult(values);
          setDirty(false);
        }}
      >
        <Card title={t("form.basic")} size="small" style={{ marginBottom: 12 }}>
          {/* Row·Col: 24칸 격자. 넓은 화면(md 이상)에서는 2칸씩(12/24), 좁으면 1칸(24/24) */}
          <Row gutter={16}>
            <Col xs={24} md={12}>
              <Form.Item name="name" label={t("form.name")} rules={[required(t("form.name"))]}>
                <Input />
              </Form.Item>
            </Col>
            <Col xs={24} md={12}>
              <Form.Item name="gender" label={t("form.gender")}>
                <CodeRadio group="GENDER" />
              </Form.Item>
            </Col>
            <Col xs={24} md={12}>
              <Form.Item name="dept" label={t("form.dept")}>
                <CodeSelect group="DEPT" allowClear />
              </Form.Item>
            </Col>
            <Col xs={24} md={12}>
              <Form.Item
                name="phone"
                label={t("form.phone")}
                // normalize: 입력값이 폼에 저장되기 전에 바꾼다. 타이핑하는 동안 하이픈이 붙는다.
                normalize={(value?: string) => formatPhone(value)}
                rules={[{ pattern: patterns.phone, message: t("users:form.phoneRule") }]}
              >
                <Input placeholder="010-1234-5678" />
              </Form.Item>
            </Col>
            <Col xs={24} md={12}>
              <Form.Item
                name="bizNo"
                label={t("form.bizNo")}
                normalize={(value?: string) => formatBizNo(value)}
                rules={[
                  {
                    // 함수형 검증: 비어 있으면 통과, 값이 있으면 검증 공식으로 확인
                    validator: (_, value?: string) =>
                      !value || isValidBizNo(value)
                        ? Promise.resolve()
                        : Promise.reject(new Error(t("form.bizNoInvalid"))),
                  },
                ]}
              >
                <Input placeholder="123-45-67890" />
              </Form.Item>
            </Col>
            <Col xs={24} md={12}>
              <Form.Item
                name="email"
                label={t("form.email")}
                rules={[{ type: "email", message: t("common:validation.email") }]}
              >
                <Input />
              </Form.Item>
            </Col>
          </Row>
        </Card>

        <Card title={t("form.detail")} size="small" style={{ marginBottom: 12 }}>
          <Row gutter={16}>
            <Col xs={24} md={12}>
              <Form.Item name="birthDate" label={t("form.birthDate")}>
                <DatePicker style={{ width: "100%" }} />
              </Form.Item>
            </Col>
            <Col xs={24} md={12}>
              <Form.Item name="period" label={t("form.period")}>
                <DatePicker.RangePicker style={{ width: "100%" }} />
              </Form.Item>
            </Col>
            <Col xs={24} md={12}>
              <Form.Item name="salary" label={t("form.salary")}>
                <InputNumber<number>
                  style={{ width: "100%" }}
                  min={0}
                  step={1000000}
                  // 보여 줄 때는 쉼표를 넣고, 값으로 바꿀 때는 쉼표를 뺀다
                  formatter={(value) => (value ? Number(value).toLocaleString() : "")}
                  parser={(value) => Number(value?.replace(/,/g, "") || 0)}
                />
              </Form.Item>
            </Col>
            <Col xs={24} md={12} />
            <Col xs={24} md={12}>
              <Form.Item
                name="password"
                label={t("form.password")}
                rules={[required(t("form.password")), { min: 8 }]}
              >
                <Input.Password autoComplete="new-password" />
              </Form.Item>
            </Col>
            <Col xs={24} md={12}>
              <Form.Item
                name="passwordConfirm"
                label={t("form.passwordConfirm")}
                dependencies={["password"]} // 비밀번호가 바뀌면 이 칸도 다시 검사한다
                rules={[
                  required(t("form.passwordConfirm")),
                  ({ getFieldValue }) => ({
                    validator: (_, value?: string) =>
                      !value || value === getFieldValue("password")
                        ? Promise.resolve()
                        : Promise.reject(new Error(t("form.passwordMismatch"))),
                  }),
                ]}
              >
                <Input.Password autoComplete="new-password" />
              </Form.Item>
            </Col>
            <Col span={24}>
              <Form.Item name="memo" label={t("form.memo")}>
                <Input.TextArea rows={3} maxLength={500} showCount />
              </Form.Item>
            </Col>
            <Col span={24}>
              <Form.Item
                name="agree"
                valuePropName="checked"
                rules={[
                  {
                    validator: (_, value: boolean) =>
                      value
                        ? Promise.resolve()
                        : Promise.reject(new Error(t("form.agreeRequired"))),
                  },
                ]}
              >
                <Checkbox>{t("form.agree")}</Checkbox>
              </Form.Item>
            </Col>
          </Row>
        </Card>

        <Flex justify="end" gap={8}>
          <Button
            onClick={() => {
              form.resetFields();
              setDirty(false);
            }}
          >
            {t("common:button.reset")}
          </Button>
          <Button type="primary" htmlType="submit">
            {t("common:button.save")}
          </Button>
        </Flex>
      </Form>

      <Modal
        open={result !== null}
        title={t("form.result")}
        footer={null}
        onCancel={() => setResult(null)}
      >
        {result && (
          <Descriptions column={1} size="small" bordered>
            <Descriptions.Item label={t("form.name")}>{maskName(result.name)}</Descriptions.Item>
            <Descriptions.Item label={t("form.phone")}>{maskPhone(result.phone)}</Descriptions.Item>
            <Descriptions.Item label={t("form.email")}>{maskEmail(result.email)}</Descriptions.Item>
            <Descriptions.Item label={t("form.bizNo")}>{result.bizNo}</Descriptions.Item>
            <Descriptions.Item label={t("form.birthDate")}>
              {result.birthDate?.format("YYYY-MM-DD")}
            </Descriptions.Item>
            <Descriptions.Item label={t("form.period")}>
              {result.period?.map((date) => date.format("YYYY-MM-DD")).join(" ~ ")}
            </Descriptions.Item>
            <Descriptions.Item label={t("form.salary")}>
              {result.salary?.toLocaleString()}
            </Descriptions.Item>
          </Descriptions>
        )}
      </Modal>
    </PageContainer>
  );
}

export default FormSamplePage;
```

### 코드 읽어 보기

**`normalize` vs `formatter`**: `normalize`는 **폼에 저장되는 값 자체**를 바꿉니다(`"010-1234-5678"`이 저장됨). `InputNumber`의 `formatter`는 **보이는 모양만** 바꾸고 값은 숫자 그대로입니다.
금액은 숫자로 계산해야 하니 `formatter`, 전화번호는 하이픈이 있는 문자열이 곧 값이니 `normalize`를 씁니다.

**날짜는 dayjs 객체**: antd 날짜 입력은 문자열이 아니라 dayjs 객체를 값으로 씁니다. 서버로 보낼 때 `.format("YYYY-MM-DD")`나 `.toISOString()`으로 바꿉니다.

**`dependencies`**: 비밀번호를 고치면 "비밀번호 확인" 칸도 다시 검사합니다. 서로 관계 있는 칸을 연결할 때 씁니다.

> ⚠️ **주의** 마스킹은 **보여 줄 때** 하는 것입니다. 화면에서 가려도 데이터는 원래 값입니다. 진짜 개인정보 보호는 서버가 권한에 따라 마스킹된 값만 내려주는 것입니다.

### 확인하기

- [ ] 샘플 → 화면 패턴 → 입력 폼. 전화번호에 `01012345678`을 치면 `010-1234-5678`이 됩니다.
- [ ] 사업자번호 `1234567890` → `123-45-67890`이 되고 "올바른 사업자등록번호가 아닙니다." `2208162517`은 통과합니다.
- [ ] 연봉에 `50000000` → `50,000,000`
- [ ] 비밀번호 확인이 다르면 안내, 동의를 안 하면 저장되지 않습니다.
- [ ] 저장하면 이름·전화·이메일이 가려진 결과가 나옵니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 입력 폼 샘플 화면 추가"
```

---

## Step 13-3. 그리드 편집 샘플 (일괄 저장)

> 🎯 **할 일**: 엑셀처럼 그리드에서 바로 수정·추가·삭제하고, [저장]으로 한꺼번에 반영하는 화면을 만듭니다. SI에서 가장 자주 요구되는 화면 중 하나입니다.

### 원리: 행마다 상태를 붙인다

서버에 "무엇이 바뀌었는지"만 보내려면 행마다 상태를 기억해야 합니다.

| 상태 | 뜻   | 표시     | 저장할 때            |
| ---- | ---- | -------- | -------------------- |
| `C`  | 추가 | 초록 배경 | `POST` (또는 묶어서) |
| `U`  | 수정 | 주황 배경 | `PUT`                |
| `D`  | 삭제 | 취소선   | `DELETE`             |

- 추가한 행(`C`)을 삭제하면 그냥 목록에서 뺍니다. 서버에 없는 행이니까요.
- 추가한 행을 수정해도 상태는 `C`로 둡니다.

> 💡 **팁** 이 샘플은 서버 없이 화면 안에서만 동작합니다. 실제로는 [저장]에서 `{ created, updated, deleted }`를 한 번에 보내는
> "일괄 저장 API"를 서버와 맞추는 경우가 많습니다. 그 모양이 이 샘플의 `changes` 객체입니다.

### 따라 하기

📄 **`src/features/samples/pages/GridEditPage.tsx`** (새 파일)

```tsx
import { useMemo, useState } from "react";
import { useTranslation } from "react-i18next";
import { App, Button, Space, Typography, theme } from "antd";
import type { ColDef, RowStyle } from "ag-grid-community";
import { formatNumber } from "@/shared/lib/format";
import { useTabDirty } from "@/shared/tabs/useTabDirty";
import DataGrid from "@/shared/ui/DataGrid";
import PageContainer from "@/shared/ui/PageContainer";

type RowStatus = "C" | "U" | "D";

interface Product {
  id: number;
  name: string;
  category: string;
  price: number;
  stock: number;
  status?: RowStatus; // 화면에서만 쓰는 변경 상태
}

const CATEGORIES = ["노트북", "모니터", "주변기기"];

const initialRows: Product[] = [
  { id: 1, name: "업무용 노트북 A", category: "노트북", price: 1500000, stock: 12 },
  { id: 2, name: "업무용 노트북 B", category: "노트북", price: 1800000, stock: 5 },
  { id: 3, name: "27인치 모니터", category: "모니터", price: 320000, stock: 30 },
  { id: 4, name: "무선 키보드", category: "주변기기", price: 59000, stock: 120 },
  { id: 5, name: "무선 마우스", category: "주변기기", price: 39000, stock: 200 },
];

function GridEditPage() {
  const { t } = useTranslation("samples");
  const { message } = App.useApp();
  const { token } = theme.useToken();
  const [rows, setRows] = useState<Product[]>(initialRows);
  const [selected, setSelected] = useState<Product[]>([]);

  const changed = rows.filter((row) => row.status);
  useTabDirty(changed.length > 0);

  const columnDefs = useMemo<ColDef<Product>[]>(
    () => [
      { headerName: t("grid.name"), field: "name", flex: 2, editable: true },
      {
        headerName: t("grid.category"),
        field: "category",
        editable: true,
        cellEditor: "agSelectCellEditor", // 셀 편집을 선택 상자로
        cellEditorParams: { values: CATEGORIES },
      },
      {
        headerName: t("grid.price"),
        field: "price",
        editable: true,
        cellEditor: "agNumberCellEditor",
        cellEditorParams: { min: 0 },
        valueFormatter: ({ value }) => formatNumber(value),
        cellStyle: { textAlign: "right" },
      },
      {
        headerName: t("grid.stock"),
        field: "stock",
        editable: true,
        cellEditor: "agNumberCellEditor",
        cellEditorParams: { min: 0, precision: 0 },
        cellStyle: { textAlign: "right" },
      },
    ],
    [t],
  );

  // 셀을 고치면 그 행을 "수정"으로 표시한다. (추가한 행은 계속 "추가")
  const handleCellChanged = (data: Product) =>
    setRows((prev) =>
      prev.map((row) =>
        row.id === data.id ? { ...data, status: row.status === "C" ? "C" : "U" } : row,
      ),
    );

  const addRow = () =>
    setRows((prev) => [
      // 아직 서버에 없는 행은 음수 id로 구별한다
      { id: -Date.now(), name: "", category: CATEGORIES[0], price: 0, stock: 0, status: "C" },
      ...prev,
    ]);

  const deleteRows = () => {
    const ids = selected.map((row) => row.id);
    setRows((prev) =>
      prev
        .filter((row) => !(ids.includes(row.id) && row.status === "C")) // 추가했던 행은 그냥 뺀다
        .map((row) => (ids.includes(row.id) ? { ...row, status: "D" as const } : row)),
    );
  };

  const save = () => {
    if (changed.length === 0) {
      message.info(t("grid.noChanges"));
      return;
    }
    // 서버에 보낼 변경 묶음. 실제 프로젝트에서는 이 객체를 일괄 저장 API로 보낸다.
    const changes = {
      created: changed.filter((row) => row.status === "C"),
      updated: changed.filter((row) => row.status === "U"),
      deleted: changed.filter((row) => row.status === "D").map((row) => row.id),
    };
    message.success(
      t("grid.saved", {
        created: changes.created.length,
        updated: changes.updated.length,
        deleted: changes.deleted.length,
      }),
    );
    // 저장이 끝났다고 보고 상태를 지운다. 삭제한 행은 목록에서 뺀다.
    setRows((prev) =>
      prev.filter((row) => row.status !== "D").map((row) => ({ ...row, status: undefined })),
    );
  };

  return (
    <PageContainer
      extra={
        <Space>
          <Button onClick={addRow}>{t("grid.addRow")}</Button>
          <Button danger disabled={selected.length === 0} onClick={deleteRows}>
            {t("grid.deleteRow")}
          </Button>
          <Button disabled={changed.length === 0} onClick={() => setRows(initialRows)}>
            {t("grid.undo")}
          </Button>
          <Button type="primary" onClick={save}>
            {t("common:button.save")}
          </Button>
        </Space>
      }
    >
      <Typography.Text type="secondary">{t("grid.help")}</Typography.Text>
      <DataGrid<Product>
        rowData={rows}
        columnDefs={columnDefs}
        getRowId={({ data }) => String(data.id)}
        rowSelection={{ mode: "multiRow" }}
        onSelectionChanged={({ api }) => setSelected(api.getSelectedRows())}
        onCellValueChanged={({ data }) => data && handleCellChanged(data)}
        stopEditingWhenCellsLoseFocus // 다른 곳을 누르면 편집을 끝낸다 (입력이 사라지는 사고 방지)
        // 반환 타입을 RowStyle로 적어 준다. 안 적으면 TypeScript가 경우마다 다른 모양을 합쳐서 추론해 타입 오류가 난다.
        getRowStyle={({ data }): RowStyle | undefined => {
          if (data?.status === "C") return { background: token.colorSuccessBg };
          if (data?.status === "U") return { background: token.colorWarningBg };
          if (data?.status === "D") return { textDecoration: "line-through", opacity: 0.5 };
          return undefined;
        }}
      />
    </PageContainer>
  );
}

export default GridEditPage;
```

### 코드 읽어 보기

**`editable: true`와 셀 편집기**: AG Grid 무료판도 셀 편집을 지원합니다. `agSelectCellEditor`(선택), `agNumberCellEditor`(숫자), `agLargeTextCellEditor`(긴 글) 등이 기본으로 들어 있습니다.

**`onCellValueChanged`**: 셀 편집이 끝나면 불립니다. AG Grid는 행 데이터 객체를 **직접 고친** 뒤 알려 줍니다. 우리는 그 값을 React state에도 반영하고 상태를 붙입니다.

**`{ ...row, status: undefined }`**: 행을 복사하면서 상태만 지웁니다. 원래 객체를 고치지 않는 불변 업데이트입니다.

**`getRowStyle`의 토큰 색**: `colorSuccessBg`, `colorWarningBg`는 다크 모드에서 자동으로 어두운 색이 됩니다.

### 확인하기

- [ ] 샘플 → 화면 패턴 → 그리드 편집. 가격 셀을 더블클릭해 바꾸면 행이 주황색, 탭 제목에 ●
- [ ] [행 추가] → 맨 위에 초록 행. 분류 셀을 더블클릭하면 선택 상자가 나옵니다.
- [ ] 행을 체크하고 [행 삭제] → 취소선. 방금 추가한 행을 삭제하면 그냥 사라집니다.
- [ ] [저장] → "추가 1건, 수정 1건, 삭제 1건을 저장했습니다." → 색이 사라지고 ●도 사라집니다.

> 💡 **팁** `({ status, ...rest }) => rest`처럼 구조 분해로 필드를 빼는 방법도 있지만, 쓰지 않는 `status` 변수 때문에 ESLint가 오류를 냅니다.
> (`@typescript-eslint/no-unused-vars`) 팀에서 이 패턴을 자주 쓴다면 `eslint.config.js`에 `ignoreRestSiblings: true` 옵션을 켜기도 합니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 그리드 편집 샘플 화면 추가"
```

---

## Step 13-4. 파일 업로드·다운로드

> 🎯 **할 일**: 파일을 서버에 올리고, 목록에서 내려받고, 지우는 샘플을 만듭니다. HTTP 클라이언트에 업로드·다운로드 기능을 추가합니다.

### 원리와 함정

**업로드**는 JSON이 아니라 `FormData`(multipart/form-data)로 보냅니다.

> ⚠️ **주의** `FormData`를 보낼 때 `Content-Type`을 **직접 지정하면 안 됩니다.** 브라우저가 `multipart/form-data; boundary=----xxx`처럼
> 경계(boundary) 값을 붙여서 자동으로 정해야 서버가 파일을 나눌 수 있습니다. 지금 `httpClient`는 항상 `application/json`을 붙이므로 고쳐야 합니다.

**다운로드**는 응답을 JSON이 아니라 `Blob`(파일 덩어리)으로 받습니다. 파일 이름은 응답 헤더 `Content-Disposition`에 들어 있는데,
한글 이름은 `filename*=UTF-8''%ED%95%9C...`처럼 인코딩되어 오므로 풀어야 합니다.

> 💡 **팁** `fetch`는 업로드 **진행률**을 알려 주지 않습니다. 큰 파일의 진행 막대가 꼭 필요하면 `XMLHttpRequest`나 axios를 그 부분에만 씁니다.

### 따라 하기

HTTP 클라이언트를 정리합니다. "요청을 보내고 오류를 처리하는 부분"을 `send`로 빼서, JSON 요청·업로드·다운로드가 함께 씁니다.

📄 **`src/shared/api/httpClient.ts`** (전체 교체)

```ts
import type { ProblemDetail } from "@/shared/api/types";
import { useAuthStore } from "@/shared/auth/authStore";
import i18n from "@/shared/i18n";

const BASE_URL = import.meta.env.VITE_API_BASE_URL;

// tsconfig의 erasableSyntaxOnly 때문에 constructor(public status) 같은 축약 문법은 쓸 수 없다. 필드를 직접 선언한다.
export class ApiError extends Error {
  readonly status: number;
  readonly problem: ProblemDetail;

  constructor(status: number, problem: ProblemDetail) {
    super(problem.detail ?? problem.title ?? `HTTP ${status}`);
    this.status = status;
    this.problem = problem;
  }
}

interface RequestOptions {
  params?: object;
  body?: unknown;
  signal?: AbortSignal;
}

// 값이 비어 있는 조건은 빼고 쿼리스트링을 만든다. { name: "김", role: undefined } → "?name=김"
function toQueryString(params: object = {}) {
  const search = new URLSearchParams();
  for (const [key, value] of Object.entries(params)) {
    if (value !== undefined && value !== null && value !== "") search.append(key, String(value));
  }
  const text = search.toString();
  return text ? `?${text}` : "";
}

/** 요청을 보내고 오류를 처리한다. 성공하면 Response를 그대로 돌려준다. (JSON·파일 공통) */
async function send(method: string, url: string, options: RequestOptions = {}) {
  const token = useAuthStore.getState().accessToken;
  const isForm = options.body instanceof FormData;
  const response = await fetch(BASE_URL + url + toQueryString(options.params), {
    method,
    headers: {
      // FormData는 브라우저가 boundary를 붙여 Content-Type을 정해야 한다. 직접 넣지 않는다.
      ...(isForm ? {} : { "Content-Type": "application/json" }),
      "Accept-Language": i18n.language,
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
    },
    body:
      options.body === undefined
        ? undefined
        : isForm
          ? (options.body as FormData)
          : JSON.stringify(options.body),
    signal: options.signal,
  });

  // 로그인이 만료되면 토큰을 지운다. 화면 이동은 RequireAuth가 상태를 보고 알아서 한다.
  if (response.status === 401) useAuthStore.getState().logout();

  if (!response.ok) {
    // 서버가 JSON이 아닌 오류를 줄 수도 있다 (프록시의 502 등). 그래도 앱이 죽지 않게 상태코드만 담는다.
    const problem = await response.json().catch(() => ({ status: response.status }));
    throw new ApiError(response.status, problem);
  }
  return response;
}

async function request<T>(method: string, url: string, options: RequestOptions = {}): Promise<T> {
  const response = await send(method, url, options);
  if (response.status === 204) return undefined as T; // 204는 본문이 없다
  return response.json();
}

/** Content-Disposition 헤더에서 파일 이름을 꺼낸다. 한글 이름(filename*=UTF-8''...)을 먼저 본다. */
function fileNameOf(response: Response, fallback: string) {
  const header = response.headers.get("Content-Disposition") ?? "";
  const encoded = header.match(/filename\*=UTF-8''([^;]+)/i)?.[1];
  if (encoded) return decodeURIComponent(encoded);
  return header.match(/filename="?([^";]+)"?/i)?.[1] ?? fallback;
}

export const httpClient = {
  get: <T>(url: string, options?: RequestOptions) => request<T>("GET", url, options),
  post: <T>(url: string, body?: unknown) => request<T>("POST", url, { body }),
  put: <T>(url: string, body?: unknown) => request<T>("PUT", url, { body }),
  delete: <T = void>(url: string) => request<T>("DELETE", url),
  /** 파일 업로드 (multipart/form-data) */
  upload: <T>(url: string, formData: FormData) => request<T>("POST", url, { body: formData }),
  /** 파일 다운로드. 파일 내용(Blob)과 서버가 알려 준 파일 이름을 돌려준다. */
  download: async (url: string, fallbackName = "download") => {
    const response = await send("GET", url);
    return { blob: await response.blob(), fileName: fileNameOf(response, fallbackName) };
  },
};
```

가짜 서버의 파일 API입니다. 파일은 localStorage에 넣기엔 크므로 **메모리**에만 둡니다. 새로고침하면 사라집니다.

📄 **`src/mocks/handlers/files.ts`** (새 파일)

```ts
import { http, HttpResponse } from "msw";
import { authorize } from "@/mocks/auth";
import { apiUrl, problem } from "@/mocks/utils";

interface StoredFile {
  id: number;
  name: string;
  size: number;
  contentType: string;
  uploadedAt: string;
  blob: Blob;
}

// 메모리 저장소. 새로고침하면 사라진다. (파일은 크기가 커서 localStorage에 넣지 않는다)
const files = new Map<number, StoredFile>();
let nextId = 1;
const MENU = "SAMPLE_FILE";

// 목록·응답에는 파일 내용(blob)을 빼고 정보만 담는다
const toInfo = (stored: StoredFile) => ({
  id: stored.id,
  name: stored.name,
  size: stored.size,
  contentType: stored.contentType,
  uploadedAt: stored.uploadedAt,
});

export const fileHandlers = [
  http.get(apiUrl("/files"), ({ request }) => {
    const { error } = authorize(request, MENU, "READ");
    if (error) return error;
    return HttpResponse.json([...files.values()].map(toInfo));
  }),

  http.post(apiUrl("/files"), async ({ request }) => {
    const { error } = authorize(request, MENU, "CREATE");
    if (error) return error;
    const form = await request.formData();
    const file = form.get("file");
    if (!(file instanceof File)) return problem(400, "파일이 없습니다.");
    const stored: StoredFile = {
      id: nextId++,
      name: file.name,
      size: file.size,
      contentType: file.type || "application/octet-stream",
      uploadedAt: new Date().toISOString(),
      blob: file,
    };
    files.set(stored.id, stored);
    return HttpResponse.json(toInfo(stored), { status: 201 });
  }),

  http.get(apiUrl("/files/:id"), ({ request, params }) => {
    const { error } = authorize(request, MENU, "READ");
    if (error) return error;
    const stored = files.get(Number(params.id));
    if (!stored) return problem(404, "파일을 찾을 수 없습니다.");
    return new HttpResponse(stored.blob, {
      headers: {
        "Content-Type": stored.contentType,
        // 한글 파일 이름은 RFC 5987 형식(filename*=UTF-8''...)으로 인코딩해서 보낸다
        "Content-Disposition": `attachment; filename*=UTF-8''${encodeURIComponent(stored.name)}`,
      },
    });
  }),

  http.delete(apiUrl("/files/:id"), ({ request, params }) => {
    const { error } = authorize(request, MENU, "DELETE");
    if (error) return error;
    files.delete(Number(params.id));
    return new HttpResponse(null, { status: 204 });
  }),
];
```

✏️ **`src/mocks/handlers.ts`** — 등록합니다.

```ts
import { codeHandlers } from "@/mocks/handlers/codes";
```

바꿀 코드

```ts
import { codeHandlers } from "@/mocks/handlers/codes";
import { fileHandlers } from "@/mocks/handlers/files";
```

✏️ **`src/mocks/handlers.ts`** — 배열에 추가합니다.

```ts
  ...notificationHandlers,
];
```

바꿀 코드

```ts
  ...notificationHandlers,
  ...fileHandlers,
];
```

화면입니다. antd `Upload`의 기본 업로드 대신 `customRequest`로 **우리 `httpClient`** 를 쓰게 합니다. 그래야 토큰·오류 처리가 똑같이 적용됩니다.

📄 **`src/features/samples/pages/FileSamplePage.tsx`** (새 파일)

```tsx
import { useMemo } from "react";
import { useTranslation } from "react-i18next";
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import { Alert, App, Button, Space, Upload } from "antd";
import { DeleteOutlined, DownloadOutlined, InboxOutlined } from "@ant-design/icons";
import type { ColDef } from "ag-grid-community";
import { httpClient } from "@/shared/api/httpClient";
import { showApiError } from "@/shared/api/showApiError";
import { usePermission } from "@/shared/auth/usePermission";
import { formatDateTime } from "@/shared/lib/format";
import { saveBlob } from "@/shared/lib/saveBlob";
import DataGrid from "@/shared/ui/DataGrid";
import PageContainer from "@/shared/ui/PageContainer";

interface FileInfo {
  id: number;
  name: string;
  size: number;
  contentType: string;
  uploadedAt: string;
}

const MAX_SIZE = 10 * 1024 * 1024; // 10MB
// 허용 확장자. 화면에서 막아도 서버는 반드시 다시 검사한다 (실행 파일 업로드 방지)
const ALLOWED = [".png", ".jpg", ".jpeg", ".gif", ".pdf", ".xlsx", ".docx", ".txt"];

const formatSize = (size: number) =>
  size < 1024 * 1024 ? `${(size / 1024).toFixed(1)} KB` : `${(size / 1024 / 1024).toFixed(1)} MB`;

function FileSamplePage() {
  const { t } = useTranslation("samples");
  const { message } = App.useApp();
  const { can } = usePermission();
  const queryClient = useQueryClient();
  const filesKey = ["sample-files"];
  const { data: files, isFetching } = useQuery({
    queryKey: filesKey,
    queryFn: () => httpClient.get<FileInfo[]>("/files"),
  });
  const removeFile = useMutation({
    mutationFn: (id: number) => httpClient.delete(`/files/${id}`),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: filesKey }),
  });

  const download = (file: FileInfo) =>
    httpClient
      .download(`/files/${file.id}`, file.name)
      .then(({ blob, fileName }) => saveBlob(blob, fileName))
      .catch(showApiError);

  const columnDefs = useMemo<ColDef<FileInfo>[]>(
    () => [
      { headerName: t("file.name"), field: "name", flex: 3 },
      {
        headerName: t("file.size"),
        field: "size",
        valueFormatter: ({ value }) => formatSize(value),
      },
      {
        headerName: t("file.uploadedAt"),
        field: "uploadedAt",
        valueFormatter: ({ value }) => formatDateTime(value),
      },
      {
        headerName: "",
        width: 120,
        flex: 0,
        sortable: false,
        cellRenderer: ({ data: file }: { data?: FileInfo }) =>
          file && (
            <Space size={4}>
              <Button
                size="small"
                type="text"
                icon={<DownloadOutlined />}
                title={t("file.download")}
                onClick={() => download(file)}
              />
              {can("DELETE") && (
                <Button
                  size="small"
                  type="text"
                  danger
                  icon={<DeleteOutlined />}
                  onClick={() => removeFile.mutate(file.id)}
                />
              )}
            </Space>
          ),
      },
    ],
    // eslint-disable-next-line react-hooks/exhaustive-deps -- download, removeFile.mutate는 매번 같은 일을 한다
    [t, can],
  );

  return (
    <PageContainer>
      <Alert type="info" showIcon title={t("file.memoryOnly")} />
      {can("CREATE") && (
        <Upload.Dragger
          multiple
          showUploadList={false}
          accept={ALLOWED.join(",")}
          // 올리기 전에 검사한다. false를 돌려주면 올리지 않는다.
          beforeUpload={(file) => {
            const extension = file.name.slice(file.name.lastIndexOf(".")).toLowerCase();
            if (!ALLOWED.includes(extension)) {
              message.error(t("file.badType"));
              return Upload.LIST_IGNORE;
            }
            if (file.size > MAX_SIZE) {
              message.error(t("file.tooLarge"));
              return Upload.LIST_IGNORE;
            }
            return true;
          }}
          // antd의 기본 업로드 대신 우리 httpClient로 보낸다 (토큰·오류 처리가 똑같이 적용된다)
          customRequest={({ file, onSuccess, onError }) => {
            const formData = new FormData();
            formData.append("file", file as File);
            httpClient
              .upload<FileInfo>("/files", formData)
              .then((info) => {
                message.success(t("file.uploaded", { name: info.name }));
                queryClient.invalidateQueries({ queryKey: filesKey });
                onSuccess?.(info);
              })
              .catch((error: Error) => {
                showApiError(error);
                onError?.(error);
              });
          }}
        >
          <p className="ant-upload-drag-icon">
            <InboxOutlined />
          </p>
          <p className="ant-upload-text">{t("file.drag")}</p>
          <p className="ant-upload-hint">{t("file.hint")}</p>
        </Upload.Dragger>
      )}
      <DataGrid<FileInfo>
        rowData={files}
        columnDefs={columnDefs}
        loading={isFetching}
        getRowId={({ data: file }) => String(file.id)}
      />
    </PageContainer>
  );
}

export default FileSamplePage;
```

### 확인하기

- [ ] 샘플 → 파일 업로드. 한글 이름의 PDF나 이미지를 끌어다 놓으면 목록에 나옵니다.
- [ ] F12 → Network → `files` 업로드 요청의 Request Headers에 `Content-Type: multipart/form-data; boundary=...`가 **자동으로** 붙어 있습니다.
- [ ] 다운로드 아이콘 → 원래 한글 이름 그대로 저장됩니다.
- [ ] `.exe` 파일이나 10MB 넘는 파일은 거절됩니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 파일 업로드·다운로드 추가"
```

---

## Step 13-5. 엑셀 다운로드

> 🎯 **할 일**: 사용자 목록 조회 결과를 엑셀(.xlsx) 파일로 내려받습니다. `EXPORT` 권한이 있을 때만 버튼이 보입니다.

### 어떻게 만들까?

| 방법                          | 장점                               | 단점                                          |
| ----------------------------- | ---------------------------------- | --------------------------------------------- |
| **서버가 만든다** (정석)      | 수십만 건도 OK. 서식·권한을 서버가 통제 | 서버 개발 필요                                |
| AG Grid Enterprise            | 그리드 모양 그대로                 | **유료**                                      |
| **프론트에서 exceljs** (채택) | 서버 없이 바로. 수천 건까지 충분   | 라이브러리가 크다(약 1MB) → 누를 때 내려받는다 |
| CSV                           | 가장 간단 (AG Grid 무료판도 지원)  | 엑셀에서 열면 한글이 깨지기 쉽다(BOM 필요), 서식 없음 |

서버 페이징 화면이라 그리드에는 한 페이지만 있습니다. 그래서 **같은 조건으로 전체를 다시 받아서** 파일을 만듭니다.

> ⚠️ **주의** npm의 `xlsx`(SheetJS) 패키지는 npm 버전이 오래 전에 멈춰 있고 보안 이슈가 알려져 있습니다. 쓰려면 공식 사이트 배포판을 씁니다.
> 이 가이드는 npm에서 관리되는 `exceljs`를 씁니다.

### 따라 하기

▶️ **터미널**

```powershell
npm install exceljs@4.4.0
```

✅ **커밋**

```powershell
git add .
git commit -m "chore: exceljs 의존성 추가"
```

📄 **`src/shared/lib/exportExcel.ts`** (새 파일)

```ts
import { saveBlob } from "@/shared/lib/saveBlob";

export interface ExcelColumn<T> {
  header: string;
  value: (row: T) => string | number | null | undefined;
  width?: number;
}

/**
 * 표 데이터를 .xlsx 파일로 저장한다.
 * exceljs는 크기가 커서, 맨 위 import가 아니라 버튼을 누를 때 내려받는다(동적 import).
 */
export async function exportExcel<T>(fileName: string, columns: ExcelColumn<T>[], rows: T[]) {
  const { default: ExcelJS } = await import("exceljs");
  const workbook = new ExcelJS.Workbook();
  const sheet = workbook.addWorksheet("Sheet1");

  sheet.columns = columns.map((column) => ({ header: column.header, width: column.width ?? 18 }));
  rows.forEach((row) => sheet.addRow(columns.map((column) => column.value(row) ?? "")));
  sheet.getRow(1).font = { bold: true }; // 제목 행
  sheet.views = [{ state: "frozen", ySplit: 1 }]; // 스크롤해도 제목 행 고정

  const buffer = await workbook.xlsx.writeBuffer();
  saveBlob(
    new Blob([buffer], {
      type: "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
    }),
    `${fileName}.xlsx`,
  );
}
```

사용자 화면에 버튼을 붙입니다.

✏️ **`src/features/users/pages/UsersPage.tsx`** — 내보내기 함수를 추가합니다.

```tsx
  const roleOptions = useMemo(
    () => roles.map((role) => ({ value: role.code, label: localize(role.names) })),
    [roles, localize],
  );
```

바꿀 코드

```tsx
  const roleOptions = useMemo(
    () => roles.map((role) => ({ value: role.code, label: localize(role.names) })),
    [roles, localize],
  );

  // 엑셀: 그리드에는 한 페이지만 있으므로 같은 조건으로 전체(최대 1만 건)를 다시 받아서 만든다.
  const [exporting, setExporting] = useState(false);
  const handleExport = async () => {
    setExporting(true);
    try {
      const all = await usersApi.list({ ...condition, page: 0, size: 10000, sort: paging.sort });
      const roleName = (code: string) =>
        roleOptions.find((option) => option.value === code)?.label ?? code;
      await exportExcel(
        `users-${dayjs().format("YYYYMMDD-HHmm")}`,
        [
          { header: t("column.username"), value: (user: User) => user.username },
          { header: t("column.name"), value: (user) => user.name },
          { header: t("column.email"), value: (user) => user.email, width: 28 },
          { header: t("column.phone"), value: (user) => user.phone },
          { header: t("column.roles"), value: (user) => user.roleCodes.map(roleName).join(", ") },
          { header: t("column.useYn"), value: (user) => formatUseYn(user.useYn) },
          { header: t("common:label.createdAt"), value: (user) => user.createdAt },
        ],
        all.content,
      );
    } catch (error) {
      showApiError(error);
    } finally {
      setExporting(false);
    }
  };
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — 버튼을 넣습니다.

```tsx
          <Can action="DELETE">
            <Button danger onClick={handleDelete}>
              {t("common:button.delete")}
            </Button>
          </Can>
        </Space>
```

바꿀 코드

```tsx
          <Can action="DELETE">
            <Button danger onClick={handleDelete}>
              {t("common:button.delete")}
            </Button>
          </Can>
          <Can action="EXPORT">
            <Button icon={<FileExcelOutlined />} loading={exporting} onClick={handleExport}>
              {t("common:button.export")}
            </Button>
          </Can>
        </Space>
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — import를 고칩니다.

```tsx
import { App, Button, Form, Input, Pagination, Select, Space } from "antd";
import type { ColDef } from "ag-grid-community";
import { useDeleteUsers, useUserPage } from "@/features/users/api/queries";
```

바꿀 코드

```tsx
import { App, Button, Form, Input, Pagination, Select, Space } from "antd";
import { FileExcelOutlined } from "@ant-design/icons";
import type { ColDef } from "ag-grid-community";
import dayjs from "dayjs";
import { useDeleteUsers, useUserPage } from "@/features/users/api/queries";
import { usersApi } from "@/features/users/api/usersApi";
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — import를 추가합니다.

```tsx
import type { PageRequest } from "@/shared/api/types";
```

바꿀 코드

```tsx
import { showApiError } from "@/shared/api/showApiError";
import type { PageRequest } from "@/shared/api/types";
```

✏️ **`src/features/users/pages/UsersPage.tsx`** — import를 추가합니다.

```tsx
import { useLocalizedText } from "@/shared/i18n/useLocalizedText";
```

바꿀 코드

```tsx
import { useLocalizedText } from "@/shared/i18n/useLocalizedText";
import { exportExcel } from "@/shared/lib/exportExcel";
```

### 코드 읽어 보기

**`await import("exceljs")`**: Part 3의 MSW처럼 **동적 import**입니다. 앱을 처음 열 때는 exceljs를 받지 않고, [내보내기]를 처음 누를 때 받습니다.
`npm run build`를 해 보면 exceljs가 별도의 큰 JS 파일로 나뉘어 있습니다.

**`{ default: ExcelJS }`**: exceljs는 예전 방식(CommonJS) 라이브러리라서 내보낸 것이 `default` 안에 들어 있습니다.

### 확인하기

- [ ] `admin` → 사용자 관리 → [내보내기] → `users-20261107-1830.xlsx` 저장. 엑셀에서 한글과 역할 이름이 잘 보이고, 제목 행이 굵게 고정되어 있습니다.
- [ ] 이름에 `김`으로 조회한 뒤 내보내면 그 조건의 결과만 들어갑니다.
- [ ] F12 → Network → JS 필터. [내보내기]를 **처음** 누를 때 exceljs 파일을 받습니다.
- [ ] `user`에게는 사용자 관리 메뉴가 없고, `manager`는 `EXPORT` 권한이 있어서 버튼이 보입니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 사용자 목록 엑셀 다운로드 추가"
```

---

## Step 13-6. 내 정보 수정·비밀번호 변경

> 🎯 **할 일**: 로그인한 사람이 자기 이름·이메일·전화번호와 비밀번호를 바꾸는 화면을 만듭니다. 헤더의 사용자 메뉴 → "내 정보 수정"으로 열립니다.

### 따라 하기

가짜 서버에 API를 추가합니다. 비밀번호 변경은 **현재 비밀번호를 확인**한 뒤에만 허용합니다.

✏️ **`src/mocks/handlers/auth.ts`** — 배열 끝에 추가합니다.

```ts
  http.get(apiUrl("/me"), ({ request }) => {
    const { user, error } = authorize(request);
    if (error) return error;
    return HttpResponse.json(user);
  }),
];
```

바꿀 코드

```ts
  http.get(apiUrl("/me"), ({ request }) => {
    const { user, error } = authorize(request);
    if (error) return error;
    return HttpResponse.json(user);
  }),

  // 내 정보 수정. 역할·사용 여부처럼 관리자가 정하는 값은 바꿀 수 없다.
  http.put(apiUrl("/me"), async ({ request }) => {
    const { user, error } = authorize(request);
    if (error) return error;
    const input = (await request.json()) as { name: string; email: string; phone: string };
    if (usersTable.all().some((item) => item.id !== user.id && item.email === input.email)) {
      const message = "이미 사용 중인 이메일입니다.";
      return problem(409, message, { errors: [{ field: "email", message }] });
    }
    return HttpResponse.json(
      usersTable.update(user.id, { name: input.name, email: input.email, phone: input.phone }),
    );
  }),

  http.put(apiUrl("/me/password"), async ({ request }) => {
    const { user, error } = authorize(request);
    if (error) return error;
    const { currentPassword, newPassword } = (await request.json()) as {
      currentPassword: string;
      newPassword: string;
    };
    const credential = credentialsTable.all().find((item) => item.username === user.username);
    if (!credential || credential.password !== currentPassword) {
      const message = "현재 비밀번호가 올바르지 않습니다.";
      return problem(400, message, { errors: [{ field: "currentPassword", message }] });
    }
    credentialsTable.update(credential.id, { password: newPassword });
    return new HttpResponse(null, { status: 204 });
  }),
];
```

문구 파일입니다.

📄 **`src/shared/i18n/locales/ko/profile.json`** (새 파일)

```json
{
  "info": "기본 정보",
  "password": "비밀번호 변경",
  "username": "아이디",
  "name": "이름",
  "email": "이메일",
  "phone": "전화번호",
  "currentPassword": "현재 비밀번호",
  "newPassword": "새 비밀번호",
  "confirmPassword": "새 비밀번호 확인",
  "passwordRule": "8자 이상, 영문과 숫자를 섞어 주세요.",
  "mismatch": "새 비밀번호가 일치하지 않습니다.",
  "changePassword": "비밀번호 변경",
  "passwordChanged": "비밀번호를 바꿨습니다."
}
```

📄 **`src/shared/i18n/locales/en/profile.json`** (새 파일)

```json
{
  "info": "Basic information",
  "password": "Change password",
  "username": "User ID",
  "name": "Name",
  "email": "Email",
  "phone": "Phone",
  "currentPassword": "Current password",
  "newPassword": "New password",
  "confirmPassword": "Confirm new password",
  "passwordRule": "At least 8 characters with letters and digits.",
  "mismatch": "New passwords do not match.",
  "changePassword": "Change password",
  "passwordChanged": "Your password has been changed."
}
```

📄 **`src/features/profile/pages/ProfilePage.tsx`** (새 파일)

```tsx
import { useTranslation } from "react-i18next";
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { App, Button, Card, Flex, Form, Input } from "antd";
import { httpClient } from "@/shared/api/httpClient";
import { showApiError } from "@/shared/api/showApiError";
import { useMe } from "@/shared/auth/useMe";
import { formatPhone } from "@/shared/lib/format";
import { patterns } from "@/shared/lib/validators";
import { applyServerErrors } from "@/shared/ui/applyServerErrors";
import PageContainer from "@/shared/ui/PageContainer";
import PageLoading from "@/shared/ui/PageLoading";

interface ProfileInput {
  name: string;
  email: string;
  phone: string;
}

interface PasswordInput {
  currentPassword: string;
  newPassword: string;
  confirmPassword: string;
}

function ProfilePage() {
  const { t } = useTranslation("profile");
  const { message } = App.useApp();
  const queryClient = useQueryClient();
  const { data: me } = useMe();
  const [profileForm] = Form.useForm<ProfileInput>();
  const [passwordForm] = Form.useForm<PasswordInput>();

  const saveProfile = useMutation({
    mutationFn: (input: ProfileInput) => httpClient.put("/me", input),
    meta: { skipGlobalError: true },
    // 헤더의 이름도 바뀌도록 내 정보 캐시를 무효화한다
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ["me", "profile"] }),
  });
  const changePassword = useMutation({
    mutationFn: ({ currentPassword, newPassword }: PasswordInput) =>
      httpClient.put("/me/password", { currentPassword, newPassword }),
    meta: { skipGlobalError: true },
  });

  if (!me) return <PageLoading />;

  const required = (field: string) => ({
    required: true,
    message: t("common:validation.required", { field }),
  });

  return (
    <PageContainer>
      <Flex gap={16} wrap align="start">
        <Card title={t("info")} style={{ width: 420 }}>
          <Form
            form={profileForm}
            name="profileForm"
            layout="vertical"
            initialValues={me}
            onFinish={(input) =>
              saveProfile.mutate(input, {
                onSuccess: () => message.success(t("common:message.saved")),
                onError: (error) => {
                  if (!applyServerErrors(profileForm, error)) showApiError(error);
                },
              })
            }
          >
            <Form.Item label={t("username")}>
              <Input value={me.username} disabled />
            </Form.Item>
            <Form.Item name="name" label={t("name")} rules={[required(t("name"))]}>
              <Input />
            </Form.Item>
            <Form.Item
              name="email"
              label={t("email")}
              rules={[
                required(t("email")),
                { type: "email", message: t("common:validation.email") },
              ]}
            >
              <Input />
            </Form.Item>
            <Form.Item
              name="phone"
              label={t("phone")}
              normalize={(value?: string) => formatPhone(value)}
              rules={[{ pattern: patterns.phone, message: t("users:form.phoneRule") }]}
            >
              <Input />
            </Form.Item>
            <Button type="primary" htmlType="submit" loading={saveProfile.isPending}>
              {t("common:button.save")}
            </Button>
          </Form>
        </Card>

        <Card title={t("password")} style={{ width: 420 }}>
          <Form
            form={passwordForm}
            name="passwordForm"
            layout="vertical"
            onFinish={(input) =>
              changePassword.mutate(input, {
                onSuccess: () => {
                  message.success(t("passwordChanged"));
                  passwordForm.resetFields();
                },
                onError: (error) => {
                  if (!applyServerErrors(passwordForm, error)) showApiError(error);
                },
              })
            }
          >
            <Form.Item
              name="currentPassword"
              label={t("currentPassword")}
              rules={[required(t("currentPassword"))]}
            >
              <Input.Password autoComplete="current-password" />
            </Form.Item>
            <Form.Item
              name="newPassword"
              label={t("newPassword")}
              rules={[
                required(t("newPassword")),
                { pattern: /^(?=.*[A-Za-z])(?=.*\d).{8,}$/, message: t("passwordRule") },
              ]}
            >
              <Input.Password autoComplete="new-password" />
            </Form.Item>
            <Form.Item
              name="confirmPassword"
              label={t("confirmPassword")}
              dependencies={["newPassword"]}
              rules={[
                required(t("confirmPassword")),
                ({ getFieldValue }) => ({
                  validator: (_, value?: string) =>
                    !value || value === getFieldValue("newPassword")
                      ? Promise.resolve()
                      : Promise.reject(new Error(t("mismatch"))),
                }),
              ]}
            >
              <Input.Password autoComplete="new-password" />
            </Form.Item>
            <Button type="primary" htmlType="submit" loading={changePassword.isPending}>
              {t("changePassword")}
            </Button>
          </Form>
        </Card>
      </Flex>
    </PageContainer>
  );
}

export default ProfilePage;
```

### 확인하기

- [ ] 헤더의 이름 → 내 정보 수정. 이름을 바꿔 저장하면 **헤더의 이름도 바로** 바뀝니다. (`["me", "profile"]` 무효화)
- [ ] 다른 사람의 이메일로 바꾸면 이메일 칸 아래에 안내가 나옵니다.
- [ ] 현재 비밀번호를 틀리면 "현재 비밀번호가 올바르지 않습니다." (현재 비밀번호 칸 아래)
- [ ] 비밀번호를 바꾸고 로그아웃 → 새 비밀번호로 로그인됩니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 내 정보 수정 및 비밀번호 변경 화면 추가"
```

---

## Step 13-7. 메뉴 검색 (Ctrl+K, 초성 검색)

> 🎯 **할 일**: 어디서든 `Ctrl+K`를 누르면 메뉴 검색 창이 열리고, "ㅅㅇㅈ"처럼 초성만 쳐도 "사용자 관리"를 찾게 합니다.

### 원리: 한글 초성 계산

한글 음절은 유니코드에서 **규칙적으로** 배치되어 있습니다. "가"(0xAC00)부터 시작해서 초성 하나당 588개(중성 21 × 종성 28)씩 묶여 있습니다.
그래서 계산만으로 초성을 뽑을 수 있습니다.

```
"사" = 0xC0AC → (0xC0AC - 0xAC00) / 588 = 9.0... → 초성 목록의 9번째 = "ㅅ"
```

> 💡 **팁** 직접 만들어 보면 재미있는 주제라 이 가이드는 직접 구현합니다. 라이브러리로는 토스의 `es-hangul`이 있습니다.

### 따라 하기

📄 **`src/shared/lib/hangul.ts`** (새 파일)

```ts
const CHOSUNG = "ㄱㄲㄴㄷㄸㄹㅁㅂㅃㅅㅆㅇㅈㅉㅊㅋㅌㅍㅎ";
const FIRST = 0xac00; // "가"
const LAST = 0xd7a3; // "힣"

/** 한글의 초성만 뽑는다. "사용자 관리" → "ㅅㅇㅈ ㄱㄹ". 한글이 아닌 글자는 그대로 둔다. */
export function getChosung(text: string) {
  return [...text]
    .map((char) => {
      const code = char.charCodeAt(0);
      if (code < FIRST || code > LAST) return char;
      return CHOSUNG[Math.floor((code - FIRST) / 588)];
    })
    .join("");
}

/** 검색어가 글자에 들어 있는가. 공백·대소문자를 무시하고, 검색어가 초성뿐이면 초성으로 비교한다. */
export function matchKeyword(text: string, keyword: string) {
  const target = text.toLowerCase().replace(/\s/g, "");
  const query = keyword.toLowerCase().replace(/\s/g, "");
  if (!query) return true;
  if (target.includes(query)) return true;
  return /^[ㄱ-ㅎ]+$/.test(query) && getChosung(target).includes(query);
}
```

📄 **`src/shared/i18n/locales/ko/search.json`** (새 파일)

```json
{
  "button": "메뉴 검색",
  "placeholder": "메뉴 이름 또는 초성 (예: ㅅㅇㅈ)",
  "empty": "찾는 메뉴가 없습니다.",
  "help": "↑↓ 이동 · Enter 열기 · Esc 닫기"
}
```

📄 **`src/shared/i18n/locales/en/search.json`** (새 파일)

```json
{
  "button": "Search menus",
  "placeholder": "Menu name or Korean initials",
  "empty": "No matching menus.",
  "help": "↑↓ move · Enter open · Esc close"
}
```

📄 **`src/app/layout/MenuSearch.tsx`** (새 파일)

```tsx
import { useEffect, useMemo, useState } from "react";
import { useTranslation } from "react-i18next";
import { Button, Empty, Input, Modal, Typography, theme } from "antd";
import { SearchOutlined } from "@ant-design/icons";
import { matchKeyword } from "@/shared/lib/hangul";
import { useLocalizedText } from "@/shared/i18n/useLocalizedText";
import MenuIcon from "@/shared/menu/MenuIcon";
import { getMenuChain } from "@/shared/menu/tree";
import { useMyMenus } from "@/shared/menu/useMyMenus";
import { useOpenPage } from "@/shared/page/useOpenPage";

/** 헤더의 메뉴 검색. Ctrl+K로 열고, 초성으로도 찾는다. */
function MenuSearch() {
  const { t } = useTranslation("search");
  const { token } = theme.useToken();
  const localize = useLocalizedText();
  const openPage = useOpenPage();
  const { data: menus = [] } = useMyMenus();
  const [open, setOpen] = useState(false);
  const [keyword, setKeyword] = useState("");
  const [cursor, setCursor] = useState(0);

  // 어디서든 Ctrl+K (맥은 Cmd+K). 헤더는 탭 밖이라 화면 하나에만 있으므로 isActive 확인이 필요 없다.
  useEffect(() => {
    const handleKeyDown = (event: KeyboardEvent) => {
      if ((event.ctrlKey || event.metaKey) && event.key.toLowerCase() === "k") {
        event.preventDefault(); // 브라우저 기본 동작(주소창 검색 등)을 막는다
        setOpen(true);
      }
    };
    window.addEventListener("keydown", handleKeyDown);
    return () => window.removeEventListener("keydown", handleKeyDown);
  }, []);

  // 열 수 있는 화면 메뉴만, "상위 › 메뉴" 경로와 함께. 한국어·영어 이름 모두로 찾는다.
  const results = useMemo(
    () =>
      menus
        .filter((menu) => menu.type === "PAGE" && menu.visible)
        .filter((menu) => matchKeyword(`${menu.names.ko} ${menu.names.en}`, keyword))
        .map((menu) => ({
          menu,
          path: getMenuChain(menus, menu.code)
            .map((item) => localize(item.names))
            .join(" › "),
        }))
        .slice(0, 20),
    [menus, keyword, localize],
  );

  const close = () => {
    setOpen(false);
    setKeyword("");
    setCursor(0);
  };

  const choose = (index: number) => {
    const result = results[index];
    if (!result) return;
    openPage(result.menu.code);
    close();
  };

  const handleKeyDown = (event: React.KeyboardEvent) => {
    if (event.key === "ArrowDown") setCursor((value) => Math.min(value + 1, results.length - 1));
    if (event.key === "ArrowUp") setCursor((value) => Math.max(value - 1, 0));
    if (event.key === "Enter") choose(cursor);
  };

  return (
    <>
      <Button
        icon={<SearchOutlined />}
        onClick={() => setOpen(true)}
        style={{ color: token.colorTextSecondary, minWidth: 200, justifyContent: "flex-start" }}
      >
        {t("button")}
        <Typography.Text keyboard style={{ marginInlineStart: "auto" }}>
          Ctrl K
        </Typography.Text>
      </Button>
      <Modal
        open={open}
        onCancel={close}
        footer={null}
        closable={false}
        width={520}
        destroyOnHidden
      >
        <Input
          autoFocus
          size="large"
          prefix={<SearchOutlined />}
          placeholder={t("placeholder")}
          value={keyword}
          onChange={(event) => {
            setKeyword(event.target.value);
            setCursor(0);
          }}
          onKeyDown={handleKeyDown}
        />
        <div style={{ marginTop: 8, maxHeight: 360, overflow: "auto" }}>
          {results.length === 0 && (
            <Empty image={Empty.PRESENTED_IMAGE_SIMPLE} description={t("empty")} />
          )}
          {results.map(({ menu, path }, index) => (
            <div
              key={menu.code}
              role="option"
              aria-selected={index === cursor}
              onMouseEnter={() => setCursor(index)}
              onClick={() => choose(index)}
              style={{
                display: "flex",
                gap: 8,
                alignItems: "center",
                padding: "8px 12px",
                cursor: "pointer",
                borderRadius: token.borderRadius,
                background: index === cursor ? token.colorPrimaryBg : undefined,
              }}
            >
              <MenuIcon name={menu.icon} />
              <span>{path}</span>
            </div>
          ))}
        </div>
        <Typography.Text type="secondary" style={{ fontSize: 12 }}>
          {t("help")}
        </Typography.Text>
      </Modal>
    </>
  );
}

export default MenuSearch;
```

✏️ **`src/app/layout/AppHeader.tsx`** — 제목 오른쪽 빈 공간 앞에 검색 버튼을 넣습니다.

```tsx
      <div style={{ flex: 1 }} />
```

바꿀 코드

```tsx
      <div style={{ flex: 1 }} />
      <MenuSearch />
```

✏️ **`src/app/layout/AppHeader.tsx`** — import를 추가합니다.

```tsx
import NotificationBell from "@/app/layout/NotificationBell";
```

바꿀 코드

```tsx
import MenuSearch from "@/app/layout/MenuSearch";
import NotificationBell from "@/app/layout/NotificationBell";
```

### 코드 읽어 보기

**`event.preventDefault()`**: 브라우저는 `Ctrl+K`에 자기 기능(주소창 검색)을 붙여 두었습니다. 막지 않으면 둘 다 실행됩니다.

**`destroyOnHidden`**: 모달을 닫으면 안의 내용을 없앱니다. 다시 열 때 입력칸에 `autoFocus`가 다시 적용되어 바로 타이핑할 수 있습니다.

**키보드 조작**: 마우스 없이 `Ctrl+K` → 입력 → `↓` → `Enter`로 화면을 엽니다. 업무 시스템을 하루 종일 쓰는 사용자에게 큰 차이를 만듭니다.

### 확인하기

- [ ] 아무 화면에서나 `Ctrl+K` → 검색 창이 열리고 바로 입력할 수 있습니다.
- [ ] `ㅅㅇㅈ` → "시스템 관리 › 사용자 관리". `Enter`로 탭이 열립니다.
- [ ] `user` → 영어 이름으로도 찾습니다. 숨김 메뉴(공지사항 상세)는 나오지 않습니다.
- [ ] `user` 계정으로 로그인하면 권한 없는 메뉴는 검색되지 않습니다. (내 메뉴만 검색하므로)

✅ **커밋**

```powershell
git add .
git commit -m "feat: 메뉴 검색 기능 추가"
```

---

## Step 13-8. 즐겨찾기·최근 메뉴

> 🎯 **할 일**: 화면 제목 옆 ☆로 즐겨찾기를 하고, 최근 연 화면을 기억해서, 열린 탭이 없을 때 바로가기로 보여 줍니다.

### 원리

즐겨찾기·최근 메뉴는 **메뉴 코드 목록**만 저장하면 됩니다. 이름·아이콘은 메뉴 데이터에서 찾으므로, 언어를 바꾸거나 메뉴 이름이 바뀌어도 그대로 맞습니다.
권한이 사라진 메뉴는 내 메뉴에 없으니 저절로 안 보입니다.

> 💡 **팁** 이 템플릿은 간단히 브라우저(localStorage)에 저장합니다. 그래서 PC를 바꾸면 사라지고, 같은 PC의 다른 계정과 공유됩니다.
> 실제 프로젝트에서는 `/api/me/favorites`처럼 **서버에 사람별로** 저장하는 것이 일반적입니다. 저장소 파일 하나만 Query로 바꾸면 됩니다.

### 따라 하기

📄 **`src/shared/menu/favoritesStore.ts`** (새 파일)

```ts
import { create } from "zustand";
import { persist } from "zustand/middleware";

const MAX_RECENT = 8;

interface FavoritesState {
  favorites: string[]; // 메뉴 코드
  recent: string[]; // 메뉴 코드, 최근 것이 앞
  toggleFavorite: (code: string) => void;
  addRecent: (code: string) => void;
}

export const useFavoritesStore = create<FavoritesState>()(
  persist(
    (set) => ({
      favorites: [],
      recent: [],
      toggleFavorite: (code) =>
        set((state) => ({
          favorites: state.favorites.includes(code)
            ? state.favorites.filter((item) => item !== code)
            : [...state.favorites, code],
        })),
      // 같은 메뉴는 맨 앞으로 옮기고, 최대 개수를 넘으면 오래된 것부터 버린다
      addRecent: (code) =>
        set((state) => ({
          recent: [code, ...state.recent.filter((item) => item !== code)].slice(0, MAX_RECENT),
        })),
    }),
    { name: "favorites" },
  ),
);
```

화면 제목 옆에 ☆를 붙입니다. 모든 화면이 `PageContainer`를 쓰므로 한 곳만 고치면 됩니다.

✏️ **`src/shared/ui/PageContainer.tsx`** — 제목 옆에 별을 넣습니다.

```tsx
          <Typography.Title level={4} style={{ margin: 0 }}>
            {title ?? (current ? localize(current.names) : "")}
          </Typography.Title>
```

바꿀 코드

```tsx
          <Flex align="center" gap={4}>
            <Typography.Title level={4} style={{ margin: 0 }}>
              {title ?? (current ? localize(current.names) : "")}
            </Typography.Title>
            <Button
              type="text"
              size="small"
              icon={
                isFavorite ? <StarFilled style={{ color: token.colorWarning }} /> : <StarOutlined />
              }
              onClick={() => toggleFavorite(menuCode)}
            />
          </Flex>
```

✏️ **`src/shared/ui/PageContainer.tsx`** — 즐겨찾기 상태를 읽습니다.

```tsx
  const chain = getMenuChain(menus, menuCode);
```

바꿀 코드

```tsx
  const { token } = theme.useToken();
  const isFavorite = useFavoritesStore((state) => state.favorites.includes(menuCode));
  const toggleFavorite = useFavoritesStore((state) => state.toggleFavorite);

  const chain = getMenuChain(menus, menuCode);
```

✏️ **`src/shared/ui/PageContainer.tsx`** — import를 고칩니다.

```tsx
import { Breadcrumb, Flex, Typography } from "antd";
import { useLocalizedText } from "@/shared/i18n/useLocalizedText";
```

바꿀 코드

```tsx
import { Breadcrumb, Button, Flex, Typography, theme } from "antd";
import { StarFilled, StarOutlined } from "@ant-design/icons";
import { useLocalizedText } from "@/shared/i18n/useLocalizedText";
import { useFavoritesStore } from "@/shared/menu/favoritesStore";
```

탭이 열릴 때 최근 메뉴에 기록합니다. 탭을 여는 곳은 `useTabUrlSync` 한 곳뿐입니다.

✏️ **`src/app/layout/useTabUrlSync.ts`** — 최근 메뉴를 기록합니다.

```ts
    lastPath.current = resolved.path;
```

바꿀 코드

```ts
    lastPath.current = resolved.path;
    useFavoritesStore.getState().addRecent(resolved.menu.code);
```

✏️ **`src/app/layout/useTabUrlSync.ts`** — import를 추가합니다.

```ts
import { useMyMenus } from "@/shared/menu/useMyMenus";
```

바꿀 코드

```ts
import { useFavoritesStore } from "@/shared/menu/favoritesStore";
import { useMyMenus } from "@/shared/menu/useMyMenus";
```

시작 안내 화면에 바로가기를 보여 줍니다.

📄 **`src/shared/i18n/locales/ko/workspace.json`** (새 파일)

```json
{
  "favorites": "즐겨찾기",
  "recent": "최근 사용",
  "noFavorites": "화면 제목 옆 ☆를 누르면 여기에 나타납니다."
}
```

📄 **`src/shared/i18n/locales/en/workspace.json`** (새 파일)

```json
{
  "favorites": "Favorites",
  "recent": "Recent",
  "noFavorites": "Click ☆ next to a page title to add it here."
}
```

📄 **`src/app/layout/WorkspaceEmpty.tsx`** (전체 교체)

```tsx
import { useTranslation } from "react-i18next";
import { Button, Card, Empty, Flex, Space, Typography } from "antd";
import { useLocalizedText } from "@/shared/i18n/useLocalizedText";
import { useFavoritesStore } from "@/shared/menu/favoritesStore";
import MenuIcon from "@/shared/menu/MenuIcon";
import { useMyMenus } from "@/shared/menu/useMyMenus";
import { useOpenPage } from "@/shared/page/useOpenPage";
import { START_MENU_CODE } from "@/shared/tabs/tabStore";

/** 열린 탭이 없을 때(주소 "/") 보여 주는 안내. 시작 화면은 고정 탭이 아니라 바로가기로 제공한다. */
function WorkspaceEmpty() {
  const { t } = useTranslation("tabs");
  const localize = useLocalizedText();
  const openPage = useOpenPage();
  const { data: menus = [] } = useMyMenus();
  const favorites = useFavoritesStore((state) => state.favorites);
  const recent = useFavoritesStore((state) => state.recent);
  const start = menus.find((menu) => menu.code === START_MENU_CODE);

  // 코드 목록 → 내 메뉴. 권한이 사라졌거나 삭제된 메뉴는 저절로 빠진다.
  const toMenus = (codes: string[]) =>
    codes.flatMap((code) => menus.filter((menu) => menu.code === code && menu.type === "PAGE"));

  const shortcuts = (codes: string[]) => (
    <Space wrap>
      {toMenus(codes).map((menu) => (
        <Button
          key={menu.code}
          icon={<MenuIcon name={menu.icon} />}
          onClick={() => openPage(menu.code)}
        >
          {localize(menu.names)}
        </Button>
      ))}
    </Space>
  );

  return (
    <Flex vertical align="center" justify="center" gap={24} style={{ height: "100%", padding: 24 }}>
      <Empty description={t("empty.description")}>
        {start && (
          <Button type="primary" onClick={() => openPage(start.code)}>
            {t("empty.open", { name: localize(start.names) })}
          </Button>
        )}
      </Empty>
      <Card size="small" title={t("workspace:favorites")} style={{ width: "100%", maxWidth: 720 }}>
        {favorites.length > 0 ? (
          shortcuts(favorites)
        ) : (
          <Typography.Text type="secondary">{t("workspace:noFavorites")}</Typography.Text>
        )}
      </Card>
      {recent.length > 0 && (
        <Card size="small" title={t("workspace:recent")} style={{ width: "100%", maxWidth: 720 }}>
          {shortcuts(recent)}
        </Card>
      )}
    </Flex>
  );
}

export default WorkspaceEmpty;
```

### 확인하기

- [ ] 사용자 관리 제목 옆 ☆ → ★ (노란색). 새로고침해도 유지됩니다.
- [ ] 화면 몇 개를 연 뒤 모든 탭을 닫으면, 시작 안내 아래에 **즐겨찾기**와 **최근 사용** 바로가기가 보입니다.
- [ ] `user`로 로그인하면 권한 없는 메뉴(사용자 관리)는 즐겨찾기에 있어도 보이지 않습니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 즐겨찾기 및 최근 메뉴 추가"
```

---

## Step 13-9. 대시보드

> 🎯 **할 일**: 요약 숫자와 차트가 있는 대시보드를 만듭니다. 차트는 테마·다크 모드를 따릅니다.

### 무엇을 어떻게 보여 줄까?

대시보드는 "예뻐 보이는 차트 모음"이 아니라 **한눈에 상황을 알려 주는 화면**입니다.

| 보여 줄 것        | 형태                    | 이유                                             |
| ----------------- | ----------------------- | ------------------------------------------------ |
| 전체 사용자 수 등 | **숫자 카드** (차트 아님) | 값 하나는 숫자가 가장 빨리 읽힌다                |
| 월별 가입자 수    | 세로 막대               | 시간에 따른 양의 변화                            |
| 역할별 사용자 수  | 가로 막대               | 항목별 양 비교. 이름이 길어도 읽기 쉽다          |

차트 규칙 몇 가지를 지킵니다.

- **색은 테마 토큰에서.** 시리즈가 하나뿐이라 대표색 하나만 쓰고, 범례는 두지 않습니다. 카드 제목이 무엇인지 말해 줍니다.
- **격자·축은 흐리게**, 막대는 끝만 살짝 둥글게(4px). 데이터가 주인공입니다.
- **마우스를 올리면 값**(툴팁). 막대마다 숫자를 다 쓰지 않습니다.
- 두 가지 단위를 한 차트에 넣는 **이중 축 차트는 쓰지 않습니다.** 필요하면 차트를 나눕니다.

### 왜 recharts인가요?

React 컴포넌트를 조립하듯 차트를 그려서 배우기 쉽습니다. 수만 개 점이나 복잡한 차트가 필요하면 ECharts를 검토합니다.

### 따라 하기

▶️ **터미널**

```powershell
npm install recharts@3.10.1
```

✅ **커밋**

```powershell
git add .
git commit -m "chore: recharts 의존성 추가"
```

가짜 서버에 대시보드 API를 만듭니다. 실제 서버에서는 집계 쿼리(SQL `GROUP BY`)로 만드는 데이터입니다.

📄 **`src/mocks/handlers/dashboard.ts`** (새 파일)

```ts
import dayjs from "dayjs";
import { http, HttpResponse } from "msw";
import { authorize } from "@/mocks/auth";
import { noticesTable, rolesTable, usersTable } from "@/mocks/db";
import { apiUrl } from "@/mocks/utils";

// 차트가 비어 보이지 않게 지난달까지는 예시 숫자를 더한다
const SAMPLE_SIGNUPS = [8, 12, 7, 15, 11, 0];

export const dashboardHandlers = [
  http.get(apiUrl("/dashboard"), ({ request }) => {
    const { error } = authorize(request, "DASHBOARD", "READ");
    if (error) return error;

    const users = usersTable.all();
    const monthlySignups = SAMPLE_SIGNUPS.map((sample, index) => {
      const month = dayjs()
        .subtract(5 - index, "month")
        .format("YYYY-MM");
      const count = users.filter((user) => user.createdAt.startsWith(month)).length;
      return { month, count: sample + count };
    });
    const usersByRole = rolesTable.all().map((role) => ({
      roleCode: role.code,
      count: users.filter((user) => user.roleCodes.includes(role.code)).length,
    }));

    return HttpResponse.json({
      userCount: users.length,
      activeUserCount: users.filter((user) => user.useYn === "Y").length,
      noticeCount: noticesTable.all().length,
      monthlySignups,
      usersByRole,
    });
  }),
];
```

✏️ **`src/mocks/handlers.ts`** — 등록합니다.

```ts
import { codeHandlers } from "@/mocks/handlers/codes";
```

바꿀 코드

```ts
import { codeHandlers } from "@/mocks/handlers/codes";
import { dashboardHandlers } from "@/mocks/handlers/dashboard";
```

✏️ **`src/mocks/handlers.ts`** — 배열에 추가합니다.

```ts
  ...notificationHandlers,
```

바꿀 코드

```ts
  ...notificationHandlers,
  ...dashboardHandlers,
```

📄 **`src/shared/i18n/locales/ko/dashboard.json`** (새 파일)

```json
{
  "welcome": "{{name}} 님, 안녕하세요.",
  "userCount": "전체 사용자",
  "activeUserCount": "사용 중인 사용자",
  "noticeCount": "공지사항",
  "unread": "내 미확인 알림",
  "monthlySignups": "월별 가입자 (최근 6개월)",
  "usersByRole": "역할별 사용자",
  "count": "명"
}
```

📄 **`src/shared/i18n/locales/en/dashboard.json`** (새 파일)

```json
{
  "welcome": "Hello, {{name}}.",
  "userCount": "Total users",
  "activeUserCount": "Active users",
  "noticeCount": "Notices",
  "unread": "My unread",
  "monthlySignups": "Sign-ups per month (last 6 months)",
  "usersByRole": "Users by role",
  "count": "users"
}
```

대시보드 화면입니다. Part 6에서 만든 간단한 화면을 바꿉니다.

📄 **`src/features/dashboard/pages/DashboardPage.tsx`** (전체 교체)

```tsx
import { useTranslation } from "react-i18next";
import { useQuery } from "@tanstack/react-query";
import { Card, Col, Row, Statistic, theme } from "antd";
import { BellOutlined, NotificationOutlined, TeamOutlined, UserOutlined } from "@ant-design/icons";
import { Bar, BarChart, CartesianGrid, ResponsiveContainer, Tooltip, XAxis, YAxis } from "recharts";
import { httpClient } from "@/shared/api/httpClient";
import { useMe } from "@/shared/auth/useMe";
import { useRoles } from "@/shared/auth/useRoles";
import { useLocalizedText } from "@/shared/i18n/useLocalizedText";
import { useNotificationSummary } from "@/shared/notification/queries";
import PageContainer from "@/shared/ui/PageContainer";
import PageLoading from "@/shared/ui/PageLoading";

interface DashboardData {
  userCount: number;
  activeUserCount: number;
  noticeCount: number;
  monthlySignups: { month: string; count: number }[];
  usersByRole: { roleCode: string; count: number }[];
}

const CHART_HEIGHT = 260;

function DashboardPage() {
  const { t } = useTranslation("dashboard");
  const { token } = theme.useToken();
  const localize = useLocalizedText();
  const { data: me } = useMe();
  const { data: roles = [] } = useRoles();
  const { data: summary } = useNotificationSummary();
  const { data } = useQuery({
    queryKey: ["dashboard"],
    queryFn: () => httpClient.get<DashboardData>("/dashboard"),
  });

  if (!data) return <PageLoading />;

  // 차트 공통 모양: 축·격자는 흐리게, 글자는 보조 글자색. 전부 테마 토큰이라 다크 모드에서도 맞는다.
  const axisProps = {
    tick: { fill: token.colorTextSecondary, fontSize: 12 },
    axisLine: false,
    tickLine: false,
  };
  const tooltipProps = {
    cursor: { fill: token.colorFillTertiary },
    contentStyle: {
      background: token.colorBgElevated,
      border: `1px solid ${token.colorBorderSecondary}`,
      borderRadius: token.borderRadius,
      color: token.colorText,
    },
    formatter: (value: unknown) => [`${value} ${t("count")}`, ""],
  };
  const roleName = (code: string) => {
    const role = roles.find((item) => item.code === code);
    return role ? localize(role.names) : code;
  };

  const stats = [
    { title: t("userCount"), value: data.userCount, icon: <TeamOutlined /> },
    { title: t("activeUserCount"), value: data.activeUserCount, icon: <UserOutlined /> },
    { title: t("noticeCount"), value: data.noticeCount, icon: <NotificationOutlined /> },
    {
      title: t("unread"),
      value: (summary?.notificationCount ?? 0) + (summary?.noticeCount ?? 0),
      icon: <BellOutlined />,
    },
  ];

  return (
    <PageContainer title={me ? t("welcome", { name: me.name }) : undefined}>
      <Row gutter={[12, 12]}>
        {stats.map((stat) => (
          <Col key={stat.title} xs={12} lg={6}>
            <Card size="small">
              <Statistic title={stat.title} value={stat.value} prefix={stat.icon} />
            </Card>
          </Col>
        ))}
        <Col xs={24} lg={14}>
          <Card size="small" title={t("monthlySignups")}>
            <ResponsiveContainer width="100%" height={CHART_HEIGHT}>
              <BarChart data={data.monthlySignups} margin={{ top: 8, right: 8, left: -16 }}>
                <CartesianGrid vertical={false} stroke={token.colorBorderSecondary} />
                <XAxis dataKey="month" {...axisProps} />
                <YAxis allowDecimals={false} {...axisProps} />
                <Tooltip {...tooltipProps} />
                <Bar
                  dataKey="count"
                  fill={token.colorPrimary}
                  radius={[4, 4, 0, 0]}
                  maxBarSize={40}
                />
              </BarChart>
            </ResponsiveContainer>
          </Card>
        </Col>
        <Col xs={24} lg={10}>
          <Card size="small" title={t("usersByRole")}>
            <ResponsiveContainer width="100%" height={CHART_HEIGHT}>
              <BarChart
                layout="vertical"
                data={data.usersByRole.map((item) => ({ ...item, name: roleName(item.roleCode) }))}
                margin={{ top: 8, right: 16 }}
              >
                <CartesianGrid horizontal={false} stroke={token.colorBorderSecondary} />
                <XAxis type="number" allowDecimals={false} {...axisProps} />
                <YAxis type="category" dataKey="name" width={96} {...axisProps} />
                <Tooltip {...tooltipProps} />
                <Bar dataKey="count" fill={token.colorPrimary} radius={[0, 4, 4, 0]} barSize={20} />
              </BarChart>
            </ResponsiveContainer>
          </Card>
        </Col>
      </Row>
    </PageContainer>
  );
}

export default DashboardPage;
```

### 코드 읽어 보기

**`ResponsiveContainer`**: 부모의 폭에 맞춰 차트 크기를 정합니다. 높이는 숫자로 정해 주어야 합니다.

**`{...axisProps}`**: 두 차트의 축에 같은 설정을 펼쳐 넣습니다. 모양 규칙을 한 곳에서 관리합니다.

**`layout="vertical"`**: 가로 막대 차트입니다. 이때는 X축이 숫자(`type="number"`), Y축이 항목(`type="category"`)이 됩니다.

### 확인하기

- [ ] 대시보드에 인사말, 숫자 카드 4개, 막대 차트 2개가 보입니다. 막대에 마우스를 올리면 값이 나옵니다.
- [ ] 다크 모드·다른 테마로 바꾸면 막대 색과 축 글자색이 함께 바뀝니다.
- [ ] 사용자를 등록한 뒤 대시보드 탭을 우클릭 → 새로고침하면 이번 달 막대와 숫자가 늘어납니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 대시보드 화면 추가"
```

---

## Step 13-10. 세션 만료 안내

> 🎯 **할 일**: 일정 시간 아무 조작이 없으면 자동 로그아웃하고, 1분 전에 경고 창으로 연장할 기회를 줍니다. 공공·금융 SI에서 자주 요구됩니다.

### 원리

```
마지막 조작(클릭·키 입력) 시각을 기억 → 1초마다 남은 시간 계산
  헤더에 남은 시간 표시 (29:41)
  남은 시간 ≤ 1분 → "곧 로그아웃됩니다" [연장]
  남은 시간 0 → 로그아웃 (만료로 처리 → 다시 로그인하면 보던 화면으로)
```

> ⚠️ **주의** 이 기능은 **화면 쪽 안내**입니다. 진짜 세션 만료는 서버가 토큰·세션 유효 시간으로 결정합니다.
> 실제 프로젝트에서 [연장]은 서버에 토큰 갱신(refresh) 요청을 보내는 것으로 바꿉니다.

### 따라 하기

만료 시간을 환경 변수로 뺍니다.

✏️ **`.env`** — 변수를 추가합니다.

```
VITE_MOCK_DELAY=300
```

바꿀 코드

```
VITE_MOCK_DELAY=300
# 아무 조작이 없을 때 자동 로그아웃까지의 시간(분)
VITE_SESSION_TIMEOUT_MINUTES=30
```

✏️ **`src/env.d.ts`** — 타입을 추가합니다.

```ts
  readonly VITE_MOCK_DELAY: string;
```

바꿀 코드

```ts
  readonly VITE_MOCK_DELAY: string;
  readonly VITE_SESSION_TIMEOUT_MINUTES: string;
```

✏️ **`src/shared/i18n/locales/ko/auth.json`** — 문구를 추가합니다.

```json
  "logout": "로그아웃"
}
```

바꿀 코드

```json
  "logout": "로그아웃",
  "session": {
    "remaining": "자동 로그아웃까지 남은 시간 (누르면 연장)",
    "warningTitle": "곧 자동 로그아웃됩니다",
    "warning": "{{seconds}}초 동안 아무 조작이 없으면 로그아웃됩니다.",
    "extend": "연장"
  }
}
```

✏️ **`src/shared/i18n/locales/en/auth.json`** — 문구를 추가합니다.

```json
  "logout": "Sign out"
}
```

바꿀 코드

```json
  "logout": "Sign out",
  "session": {
    "remaining": "Time until automatic sign-out (click to extend)",
    "warningTitle": "You will be signed out soon",
    "warning": "You will be signed out in {{seconds}} seconds of inactivity.",
    "extend": "Stay signed in"
  }
}
```

📄 **`src/app/layout/SessionTimer.tsx`** (새 파일)

```tsx
import { useEffect, useRef, useState } from "react";
import { useTranslation } from "react-i18next";
import { Modal, Tag, Tooltip } from "antd";
import { ClockCircleOutlined } from "@ant-design/icons";
import { useAuthStore } from "@/shared/auth/authStore";

const TIMEOUT_MS = Number(import.meta.env.VITE_SESSION_TIMEOUT_MINUTES) * 60_000;
const WARN_MS = 60_000; // 1분 전에 경고

/** 헤더의 세션 남은 시간. 조작이 없으면 경고 후 자동 로그아웃한다. */
function SessionTimer() {
  const { t } = useTranslation("auth");
  const lastActivity = useRef(0); // 화면에 보여 줄 값이 아니라서 ref. 시각은 Effect에서 넣는다
  const [remaining, setRemaining] = useState(TIMEOUT_MS);

  useEffect(() => {
    lastActivity.current = Date.now(); // 화면이 나타난 시각부터 센다
    const touch = () => {
      lastActivity.current = Date.now();
    };
    // 마우스 이동은 너무 잦고, 자리를 비운 사람의 마우스가 흔들려도 연장되면 안 되므로 넣지 않는다
    const events = ["mousedown", "keydown", "wheel", "touchstart"] as const;
    events.forEach((name) => window.addEventListener(name, touch, { passive: true }));

    const timer = window.setInterval(() => {
      const left = TIMEOUT_MS - (Date.now() - lastActivity.current);
      if (left <= 0) useAuthStore.getState().logout(); // 만료로 로그아웃 → 다시 로그인하면 보던 화면으로
      setRemaining(Math.max(0, left));
    }, 1000);

    return () => {
      events.forEach((name) => window.removeEventListener(name, touch));
      window.clearInterval(timer);
    };
  }, []);

  const extend = () => {
    lastActivity.current = Date.now();
    setRemaining(TIMEOUT_MS);
  };

  const seconds = Math.ceil(remaining / 1000);
  const text = `${String(Math.floor(seconds / 60)).padStart(2, "0")}:${String(seconds % 60).padStart(2, "0")}`;
  const warning = remaining > 0 && remaining <= WARN_MS;

  return (
    <>
      <Tooltip title={t("session.remaining")}>
        <Tag
          icon={<ClockCircleOutlined />}
          color={warning ? "error" : undefined}
          onClick={extend}
          style={{ cursor: "pointer", marginInlineEnd: 0, fontVariantNumeric: "tabular-nums" }}
        >
          {text}
        </Tag>
      </Tooltip>
      <Modal
        open={warning}
        title={t("session.warningTitle")}
        okText={t("session.extend")}
        onOk={extend}
        closable={false}
        mask={{ closable: false }}
        cancelButtonProps={{ style: { display: "none" } }}
      >
        {t("session.warning", { seconds })}
      </Modal>
    </>
  );
}

export default SessionTimer;
```

✏️ **`src/app/layout/AppHeader.tsx`** — 🔔 앞에 넣습니다.

```tsx
      <NotificationBell />
      <UserMenu />
```

바꿀 코드

```tsx
      <SessionTimer />
      <NotificationBell />
      <UserMenu />
```

✏️ **`src/app/layout/AppHeader.tsx`** — import를 추가합니다.

```tsx
import SettingsDrawer from "@/app/layout/SettingsDrawer";
```

바꿀 코드

```tsx
import SessionTimer from "@/app/layout/SessionTimer";
import SettingsDrawer from "@/app/layout/SettingsDrawer";
```

### 코드 읽어 보기

**`setInterval`과 정리**: Effect 안에서 만든 타이머는 정리 함수에서 반드시 `clearInterval` 합니다. 안 그러면 화면이 사라져도 타이머가 계속 돌고, StrictMode에서는 두 개가 돕니다.

**`useRef(Date.now())`를 쓰지 않은 이유**: 컴포넌트 함수는 **순수**해야 합니다. 같은 props·state면 몇 번을 다시 그려도 같은 결과가 나와야 합니다.
`Date.now()`는 부를 때마다 값이 달라서 렌더 중에 부르면 ESLint가 `react-hooks/purity` 오류를 냅니다. 시각처럼 "그때그때 달라지는 값"은 Effect나 이벤트 처리 안에서 읽습니다.

**`fontVariantNumeric: "tabular-nums"`**: 숫자 폭을 같게 해서 1초마다 글자가 좌우로 흔들리지 않게 합니다.

### 확인하기

시험할 때는 시간을 줄입니다. 프로젝트 폴더에 `.env.local` 파일을 만들고 `VITE_SESSION_TIMEOUT_MINUTES=2`를 넣은 뒤 개발 서버를 다시 켭니다. (`.env.local`은 Git에 올라가지 않습니다)

- [ ] 헤더에 `02:00`부터 줄어드는 시계가 보입니다. 클릭하거나 키를 누르면 다시 `02:00`
- [ ] 1분 동안 가만히 두면 경고 창. [연장]을 누르면 사라집니다.
- [ ] 2분 동안 가만히 두면 로그인 화면. 다시 로그인하면 보던 화면으로 돌아갑니다.
- [ ] 확인이 끝나면 `.env.local`을 지웁니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 세션 만료 안내 및 연장 기능 추가"
```

---

## Step 13-11. 전역 로딩 표시줄

> 🎯 **할 일**: 서버 요청이 진행 중일 때 화면 맨 위에 얇은 막대를 보여 줍니다. 저장 버튼 하나하나에 로딩을 다는 것과 별개로, "지금 뭔가 하고 있다"를 앱 전체에서 알려 줍니다.

### 원리

TanStack Query는 지금 진행 중인 조회·저장 개수를 알려 주는 훅(`useIsFetching`, `useIsMutating`)을 제공합니다. 0보다 크면 막대를 보여 주면 됩니다.

그런데 🔔는 15초마다 조용히 조회합니다. 그때마다 막대가 깜빡이면 거슬리므로, 조회에 **`meta: { silent: true }`** 표시를 붙여서 세지 않게 합니다.

### 따라 하기

`meta`에 `silent`를 쓸 수 있다고 TypeScript에 알려 줍니다. (Part 3에서 `skipGlobalError`를 등록한 곳)

✏️ **`src/app/queryClient.ts`** — 조회 meta 타입을 추가합니다.

```ts
    mutationMeta: { skipGlobalError?: boolean };
```

바꿀 코드

```ts
    mutationMeta: { skipGlobalError?: boolean };
    queryMeta: { silent?: boolean }; // true면 전역 로딩 표시줄에서 세지 않는다
```

✏️ **`src/shared/notification/queries.ts`** — 🔔 폴링을 조용하게 만듭니다.

```ts
    refetchInterval: NOTIFICATION_POLL_MS,
```

바꿀 코드

```ts
    refetchInterval: NOTIFICATION_POLL_MS,
    meta: { silent: true }, // 주기적인 조회로 전역 로딩 막대가 깜빡이지 않게 한다
```

막대 모양은 CSS 애니메이션으로 만듭니다.

📄 **`src/app/layout/GlobalLoadingBar.module.css`** (새 파일)

```css
.bar {
  position: fixed;
  inset: 0 0 auto 0;
  height: 2px;
  z-index: 2000;
  overflow: hidden;
  pointer-events: none; /* 막대가 클릭을 가로채지 않게 */
}

.bar::before {
  content: "";
  position: absolute;
  inset: 0;
  width: 40%;
  background: var(--app-color-primary);
  animation: slide 1.2s ease-in-out infinite;
}

@keyframes slide {
  from {
    transform: translateX(-100%);
  }
  to {
    transform: translateX(250%);
  }
}
```

📄 **`src/app/layout/GlobalLoadingBar.tsx`** (새 파일)

```tsx
import { useIsFetching, useIsMutating } from "@tanstack/react-query";
import styles from "@/app/layout/GlobalLoadingBar.module.css";

/** 서버 요청이 진행 중이면 화면 맨 위에 얇은 막대를 보여 준다. silent 조회는 세지 않는다. */
function GlobalLoadingBar() {
  const fetching = useIsFetching({ predicate: (query) => !query.meta?.silent });
  const mutating = useIsMutating();
  if (fetching + mutating === 0) return null;
  return <div className={styles.bar} role="progressbar" aria-label="loading" />;
}

export default GlobalLoadingBar;
```

✏️ **`src/app/layout/MainLayout.tsx`** — 맨 위에 넣습니다.

```tsx
    <Layout style={{ height: "100%" }}>
      <AppHeader
```

바꿀 코드

```tsx
    <Layout style={{ height: "100%" }}>
      <GlobalLoadingBar />
      <AppHeader
```

✏️ **`src/app/layout/MainLayout.tsx`** — import를 추가합니다.

```tsx
import AppHeader from "@/app/layout/AppHeader";
```

바꿀 코드

```tsx
import AppHeader from "@/app/layout/AppHeader";
import GlobalLoadingBar from "@/app/layout/GlobalLoadingBar";
```

### 확인하기

- [ ] 사용자 관리에서 [조회]를 누르면 화면 맨 위에 파란 막대가 잠깐 흐릅니다. (가짜 서버 지연 300ms 동안)
- [ ] 아무것도 안 하고 기다리면, 🔔의 15초 조회 때는 막대가 나타나지 않습니다.
- [ ] 테마를 바꾸면 막대 색도 바뀝니다. (`--app-color-primary`)

✅ **커밋**

```powershell
git add .
git commit -m "feat: 전역 로딩 표시줄 추가"
```

---

## Step 13-12. 개발자 패널

> 🎯 **할 일**: 개발 모드에서만 보이는 패널을 만듭니다. 계정 전환, 응답 지연 조절, 다음 요청 오류 만들기, 가짜 DB 초기화, 현재 화면 권한 보기를 클릭 몇 번으로 합니다.

### 왜 필요한가요?

로딩 화면, 오류 처리, 권한별 화면을 확인하려고 매번 코드를 고치거나 로그아웃·로그인을 반복하면 지칩니다.
패널 하나로 이런 상황을 바로 만들 수 있으면 **화면 품질을 확인하는 일이 귀찮지 않게** 됩니다.

### 원리: 앱과 가짜 서버가 대화하는 법

규칙 3번 때문에 앱 코드는 `mocks`를 import 할 수 없습니다. 그래서 **localStorage의 약속된 키**로 대화합니다.
패널이 `mock:control`에 `{ delay: 2000 }`을 쓰면, 가짜 서버가 요청을 받을 때마다 그 값을 읽습니다. 키와 모양은 `shared`에 두고 양쪽이 같이 씁니다.

### 따라 하기

📄 **`src/shared/dev/mockControl.ts`** (새 파일)

```ts
/** 개발자 패널 ↔ 가짜 서버 사이의 약속. localStorage로 주고받는다. */
export interface MockControl {
  delay?: number; // 응답 지연(ms). 없으면 .env의 VITE_MOCK_DELAY
  failNext?: number; // 다음 요청 하나를 이 상태코드로 실패시킨다
}

const KEY = "mock:control";

export function readMockControl(): MockControl {
  try {
    return JSON.parse(localStorage.getItem(KEY) ?? "{}");
  } catch {
    return {};
  }
}

export function writeMockControl(control: MockControl) {
  localStorage.setItem(KEY, JSON.stringify(control));
}
```

가짜 서버의 맨 앞 핸들러(모든 요청이 지나가는 곳)가 이 값을 읽게 합니다.

✏️ **`src/mocks/handlers.ts`** — 공통 핸들러를 바꿉니다.

```ts
  http.all(apiUrl("/*"), async () => {
    await delay(MOCK_DELAY);
  }),
```

바꿀 코드

```ts
  http.all(apiUrl("/*"), async () => {
    // 개발자 패널의 설정: 지연 시간, "다음 요청 실패"
    const control = readMockControl();
    await delay(control.delay ?? MOCK_DELAY);
    if (control.failNext) {
      writeMockControl({ ...control, failNext: undefined }); // 한 번만 실패시킨다
      return problem(control.failNext, `개발자 패널에서 만든 ${control.failNext} 오류`);
    }
  }),
```

✏️ **`src/mocks/handlers.ts`** — import를 고칩니다.

```ts
import { apiUrl, MOCK_DELAY } from "@/mocks/utils";
```

바꿀 코드

```ts
import { apiUrl, MOCK_DELAY, problem } from "@/mocks/utils";
import { readMockControl, writeMockControl } from "@/shared/dev/mockControl";
```

문구 파일입니다. 개발자용이지만 다른 화면과 같은 방식으로 둡니다.

📄 **`src/shared/i18n/locales/ko/dev.json`** (새 파일)

```json
{
  "title": "개발자 패널",
  "account": "계정 전환",
  "delay": "응답 지연",
  "failNext": "다음 요청 하나를 실패시키기",
  "failArmed": "다음 요청이 {{status}}로 실패합니다.",
  "resetDb": "가짜 DB 초기화",
  "resetConfirm": "가짜 DB를 시드 데이터로 되돌리고 새로고침합니다.",
  "permission": "현재 화면 권한",
  "noPage": "열린 화면이 없습니다."
}
```

📄 **`src/shared/i18n/locales/en/dev.json`** (새 파일)

```json
{
  "title": "Developer panel",
  "account": "Switch account",
  "delay": "Response delay",
  "failNext": "Fail the next request",
  "failArmed": "The next request will fail with {{status}}.",
  "resetDb": "Reset mock DB",
  "resetConfirm": "Reset the mock DB to seed data and reload.",
  "permission": "Current page permissions",
  "noPage": "No page is open."
}
```

📄 **`src/app/dev/DevPanel.tsx`** (새 파일)

```tsx
import { useState } from "react";
import { useTranslation } from "react-i18next";
import { useNavigate } from "react-router";
import { useQueryClient } from "@tanstack/react-query";
import {
  App,
  Button,
  Descriptions,
  Drawer,
  Flex,
  FloatButton,
  Segmented,
  Space,
  Tag,
  Typography,
} from "antd";
import { BugOutlined } from "@ant-design/icons";
import { httpClient } from "@/shared/api/httpClient";
import { useAuthStore } from "@/shared/auth/authStore";
import { readMockControl, writeMockControl } from "@/shared/dev/mockControl";
import { useMyMenus } from "@/shared/menu/useMyMenus";
import { useActiveTabKey } from "@/shared/tabs/useActiveTabKey";
import { useTabStore } from "@/shared/tabs/tabStore";

const ACCOUNTS = ["admin", "manager", "user"];
const DELAYS = [0, 300, 2000];
const FAILURES = [401, 403, 500];

/** 개발 모드 전용 도구. 운영 빌드에는 포함되지 않는다. (MainLayout에서 import.meta.env.DEV로 막는다) */
function DevPanel() {
  const { t } = useTranslation("dev");
  const { message, modal } = App.useApp();
  const navigate = useNavigate();
  const queryClient = useQueryClient();
  const [open, setOpen] = useState(false);
  const [control, setControl] = useState(readMockControl);
  const activeKey = useActiveTabKey();
  const { data: menus = [] } = useMyMenus();
  const activeTab = useTabStore((state) => state.tabs.find((tab) => tab.key === activeKey));
  const activeMenu = menus.find((menu) => menu.code === activeTab?.menuCode);

  const update = (next: typeof control) => {
    writeMockControl(next);
    setControl(next);
  };

  // 다른 계정으로 바로 다시 로그인한다. 이전 사람의 캐시·탭은 지운다.
  const switchAccount = async (username: string) => {
    const { accessToken } = await httpClient.post<{ accessToken: string }>("/auth/login", {
      username,
      password: `${username}1234`,
    });
    queryClient.clear();
    const { tabs, close } = useTabStore.getState();
    close(tabs.map((tab) => tab.key));
    useAuthStore.getState().login(accessToken);
    navigate("/");
    message.success(username);
    setOpen(false);
  };

  const resetDb = () =>
    modal.confirm({
      title: t("resetConfirm"),
      onOk: () => {
        Object.keys(localStorage)
          .filter((key) => key.startsWith("mockdb:"))
          .forEach((key) => localStorage.removeItem(key));
        window.location.reload();
      },
    });

  return (
    <>
      <FloatButton
        icon={<BugOutlined />}
        tooltip={t("title")}
        onClick={() => setOpen(true)}
        // 오른쪽 아래는 화면의 [저장] 버튼 자리라서 왼쪽 아래(Query Devtools 위)에 둔다
        style={{ insetInlineStart: 16, insetInlineEnd: "auto", insetBlockEnd: 72 }}
      />
      <Drawer
        title={t("title")}
        placement="left"
        open={open}
        onClose={() => setOpen(false)}
        size={340}
      >
        <Flex vertical gap={24}>
          <div>
            <Typography.Title level={5}>{t("account")}</Typography.Title>
            <Space wrap>
              {ACCOUNTS.map((username) => (
                <Button key={username} onClick={() => switchAccount(username)}>
                  {username}
                </Button>
              ))}
            </Space>
          </div>
          <div>
            <Typography.Title level={5}>{t("delay")}</Typography.Title>
            <Segmented<number>
              value={control.delay ?? Number(import.meta.env.VITE_MOCK_DELAY)}
              options={DELAYS.map((value) => ({ value, label: `${value}ms` }))}
              onChange={(delay) => update({ ...control, delay })}
            />
          </div>
          <div>
            <Typography.Title level={5}>{t("failNext")}</Typography.Title>
            <Space wrap>
              {FAILURES.map((status) => (
                <Button
                  key={status}
                  danger
                  onClick={() => {
                    update({ ...readMockControl(), failNext: status });
                    message.warning(t("failArmed", { status }));
                  }}
                >
                  {status}
                </Button>
              ))}
            </Space>
          </div>
          <div>
            <Typography.Title level={5}>{t("permission")}</Typography.Title>
            {activeMenu && "actions" in activeMenu ? (
              <Descriptions column={1} size="small">
                <Descriptions.Item label="menuCode">{activeMenu.code}</Descriptions.Item>
                <Descriptions.Item label="actions">
                  {activeMenu.actions.map((action) => (
                    <Tag key={action}>{action}</Tag>
                  ))}
                </Descriptions.Item>
              </Descriptions>
            ) : (
              <Typography.Text type="secondary">{t("noPage")}</Typography.Text>
            )}
          </div>
          <Button danger onClick={resetDb}>
            {t("resetDb")}
          </Button>
        </Flex>
      </Drawer>
    </>
  );
}

export default DevPanel;
```

레이아웃에 붙입니다. **운영 빌드에 들어가지 않도록** 동적 import로 불러옵니다.

✏️ **`src/app/layout/MainLayout.tsx`** — 개발 모드에서만 불러오는 컴포넌트를 만듭니다.

```tsx
const { Sider, Content } = Layout;
```

바꿀 코드

```tsx
const { Sider, Content } = Layout;

// 개발 모드에서만 개발자 패널을 불러온다. 운영 빌드에서는 import.meta.env.DEV가 false라서 아예 포함되지 않는다.
const DevPanel = import.meta.env.DEV ? lazy(() => import("@/app/dev/DevPanel")) : () => null;
```

✏️ **`src/app/layout/MainLayout.tsx`** — 맨 끝에 넣습니다.

```tsx
        </Content>
      </Layout>
    </Layout>
  );
}
```

바꿀 코드

```tsx
        </Content>
      </Layout>
      <Suspense>
        <DevPanel />
      </Suspense>
    </Layout>
  );
}
```

✏️ **`src/app/layout/MainLayout.tsx`** — import를 고칩니다.

```tsx
import { useState } from "react";
```

바꿀 코드

```tsx
import { lazy, Suspense, useState } from "react";
```

### 코드 읽어 보기

**`useState(readMockControl)`**: 함수를 **호출하지 않고** 넘기면 처음 한 번만 실행됩니다. (`useState(readMockControl())`는 렌더할 때마다 localStorage를 읽습니다)

**`"actions" in activeMenu`**: 내 메뉴(`MyMenu`)에는 `actions`가 있습니다. `in`으로 확인하면 TypeScript가 타입을 좁혀 줍니다.

**운영 빌드에서 빠지는지 확인**: `npm run build` 후 `dist/assets`에서 `DevPanel`이라는 이름의 파일이 없으면 성공입니다.

### 확인하기

- [ ] 왼쪽 아래 🐞 버튼 → 패널. [user] → 바로 user로 바뀌고 메뉴가 줄어듭니다.
- [ ] 응답 지연 2000ms → 조회할 때 로딩이 2초 동안 보입니다. (로딩 화면 확인용)
- [ ] [500] → 다음 조회에서 "잠시 후 다시 시도해 주세요." 알림. 그다음 조회는 정상입니다.
- [ ] [401] → 다음 요청에서 로그인 화면으로. 다시 로그인하면 보던 화면으로 돌아갑니다.
- [ ] 사용자 관리 화면에서 열면 "현재 화면 권한"에 `USER_MGMT`와 동작들이 보입니다.
- [ ] [가짜 DB 초기화] → 등록했던 사용자·공지가 사라지고 시드로 돌아갑니다.

✅ **커밋**

```powershell
git add .
git commit -m "feat: 개발자 패널 추가"
```

---

## 🏁 Part 13을 마치며

메뉴 시드의 모든 "준비 중" 화면이 채워졌습니다. 템플릿의 기능이 완성되었습니다. 화면을 캡처해 두세요.

| 기능                | 핵심 기법                                                    |
| ------------------- | ------------------------------------------------------------ |
| 포맷·마스킹·검증    | 순수 함수, 정규식                                            |
| 입력 폼             | `normalize`, `formatter`/`parser`, `dependencies`, `validator` |
| 그리드 편집         | 셀 편집기, 행 상태(C/U/D), 일괄 저장                         |
| 파일                | `FormData`(Content-Type 직접 지정 금지), `Blob`, `Content-Disposition` |
| 엑셀                | exceljs 동적 import, 전체 재조회                             |
| 메뉴 검색           | 전역 단축키, 한글 초성 계산                                  |
| 즐겨찾기·최근       | 코드만 저장, 표시는 메뉴 데이터                              |
| 대시보드            | 숫자 카드 + 막대, 테마 토큰 색                               |
| 세션 만료           | 활동 시각 ref, `setInterval` 정리                            |
| 로딩 표시줄         | `useIsFetching`, 조회 `meta`                                 |
| 개발자 패널         | localStorage 약속, 개발 전용 동적 import                     |

마지막 Part에서는 진짜 서버를 연결하고 배포할 준비를 합니다. → [Part 14. 백엔드 연결과 배포](part-14-backend-deploy.md)
