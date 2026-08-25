# Forms: Controlled vs Uncontrolled — defaultValue, RHF, Controller và watch

> Tags: #forms #controlled #uncontrolled #react-hook-form #ref #Controller #watch | Nguồn: `docs/02-react-co-ban.md` câu 34, 46 | Mức: P0

## 1. Định nghĩa chính xác

- **Controlled Component**: form element mà **value** được React state quản lý hoàn toàn: `<input value={state} onChange={e => setState(e.target.value)} />`. Single source of truth là React, mỗi keystroke qua `setState` ⇒ re-render.
- **Uncontrolled Component**: value nằm trong **DOM** (`HTMLInputElement.value`), React không quản lý mỗi keystroke. Đọc qua `ref` hoặc `defaultValue` khi cần (submit). Khai báo bằng `defaultValue`/`defaultChecked` thay vì `value`/`checked`.
- **defaultValue / defaultChecked**: giá trị **khởi tạo** cho uncontrolled input, chỉ có tác dụng **mount lần đầu**. Sau mount, DOM tự quản lý, React không can thiệp. Đổi `defaultValue` sau mount không update DOM (trừ khi `key` đổi để remount).
- **React Hook Form (RHF)**: thư viện form **uncontrolled + ref** — đăng ký input bằng `register` (gắn `ref`), đọc value từ DOM trực tiếp, chỉ re-render khi cần (error, isDirty). Kết hợp `resolver` (zod/yup) cho validation schema, giảm re-render 10-100x so với controlled cho form lớn.
- **Controller**: wrapper của RHF cho **UI lib controlled** (MUI, AntD, custom `Select`) vốn nhận `value`/`onChange` — RHF vẫn quản lý qua `field` props nhưng field đó là controlled từ góc nhìn UI lib.
- **watch vs useWatch**: `watch()` subscribe toàn form hoặc field và **trigger re-render** ở component gọi; `useWatch()` isolate subscription để re-render chỉ ở component con nhỏ (tránh re-render toàn form).

## 2. Cơ chế hoạt động

### 2.1 Controlled flow

```
user type → onChange → setState(newValue) → re-render → input value={newValue} → paint
```

- Ogni ký tự đều qua React state ⇒ dễ **live validation**, **format** (mask, uppercase), **disable submit** theo `isValid`, nhưng **mỗi keystroke re-render** toàn component (và có thể subtree). Với 20 field, mỗi gõ re-render 20 input.

### 2.2 Uncontrolled flow

```
user type → DOM update (không qua React) → submit → ref.current.value
register('name') → RHF gắn ref vào input, lưu value trong internal map (không state)
```

- RHF dùng `ref` để đọc `input.value` khi cần, không `useState` cho mỗi field ⇒ **không re-render** khi gõ. Chỉ khi `errors` đổi hoặc `isDirty`/`watch` thì re-render. Validation chạy qua `resolver` khi submit hoặc theo `mode`.

### 2.3 defaultValue vs value

- `value` = controlled — React ghi đè DOM mỗi render.
- `defaultValue` = uncontrolled — chỉ set `input.defaultValue` lúc mount (`initialValue`), sau đó bỏ.
- Chuyển uncontrolled → controlled (hoặc ngược) giữa render gây warning React: `A component is changing an uncontrolled input to be controlled`. Fix: luôn cung cấp `value` hoặc `defaultValue` nhất quán, và `defaultValues` cho RHF.

### 2.4 RHF internals

- `useForm({ defaultValues, resolver })` trả `{ register, handleSubmit, control, formState }`.
- `register(name)` trả `{ ref, onChange, onBlur, name }` — RHF subscribe `onChange` để track `dirty`/`errors` mà không `setState` cho value.
- `Controller` nhận `control` và `render={({ field }) => <MUISelect {...field} />}` — RHF giữ `field.value` trong internal state và đồng bộ với UI controlled.
- `watch` đọc toàn bộ `formValues` và re-render caller; `useWatch({ control, name })` tạo isolated subscription — chỉ component dùng `useWatch` re-render.

### 2.5 Khi nào dùng Controller?

Khi UI component không expose `ref` hoặc là controlled-only (`<Select value={v} onChange={setV} />`). Khi input là native (`<input>`, `<textarea>`, `<select>`) thì dùng `register` thuần (nhanh hơn).

## 3. Ví dụ tối thiểu

```tsx
// 3.1 Controlled — mỗi gõ re-render
import { useState } from "react";

function Controlled() {
  const [value, setValue] = useState("");
  const [error, setError] = useState("");
  return (
    <>
      <input
        value={value}
        onChange={e => {
          const v = e.target.value;
          setValue(v); // re-render mỗi ký tự
          setError(v.length < 2 ? "Quá ngắn" : "");
        }}
      />
      {error && <p>{error}</p>}
    </>
  );
}

// 3.2 Uncontrolled — đọc khi submit, ít re-render
import { useRef } from "react";

function Uncontrolled() {
  const ref = useRef<HTMLInputElement>(null);
  const onSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    console.log(ref.current?.value); // đọc khi cần
  };
  return (
    <form onSubmit={onSubmit}>
      <input defaultValue="hello" ref={ref} />
      <button>Submit</button>
    </form>
  );
}

// 3.3 Hybrid — defaultValue + controlled state cục bộ
function Hybrid({ defaultValue }: { defaultValue: string }) {
  const [value, setValue] = useState(defaultValue); // khởi từ prop, sau đó controlled nội bộ
  return <input value={value} onChange={e => setValue(e.target.value)} />;
}

// 3.4 File input — bắt buộc uncontrolled
function FileInput() {
  const fileRef = useRef<HTMLInputElement>(null);
  return <input type="file" ref={fileRef} />;
}
```

```tsx
// 3.5 React Hook Form — uncontrolled + ref, tối ưu cho form lớn
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

const schema = z.object({
  name: z.string().min(2, "Tên quá ngắn"),
  email: z.string().email("Email sai"),
  age: z.coerce.number().min(18, "Phải >=18"),
});
type FormValues = z.infer<typeof schema>;

function RHFForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting, isDirty },
  } = useForm<FormValues>({
    resolver: zodResolver(schema),
    defaultValues: { name: "", email: "", age: 18 },
    mode: "onBlur", // validate khi blur, không mỗi keystroke
  });

  const onSubmit = async (data: FormValues) => {
    await fetch("/api/submit", { method: "POST", body: JSON.stringify(data) });
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* register gắn ref — không re-render khi gõ */}
      <input {...register("name")} placeholder="Tên" />
      {errors.name && <p>{errors.name.message}</p>}

      <input {...register("email")} placeholder="Email" />
      {errors.email && <p>{errors.email.message}</p>}

      <input type="number" {...register("age")} />
      {errors.age && <p>{errors.age.message}</p>}

      <button disabled={isSubmitting || !isDirty}>Submit</button>
    </form>
  );
}

// 3.6 Controller — cho UI controlled (MUI/AntD)
import { Controller, useForm } from "react-hook-form";

function RHFWithController() {
  const { control, handleSubmit } = useForm<{ city: string }>({
    defaultValues: { city: "" },
  });
  return (
    <form onSubmit={handleSubmit(data => console.log(data))}>
      <Controller
        name="city"
        control={control}
        render={({ field }) => (
          // field = { value, onChange, onBlur, ref, name }
          // Giả lập MUI Select controlled
          <select value={field.value} onChange={e => field.onChange(e.target.value)}>
            <option value="">Chọn</option>
            <option value="hcm">HCM</option>
            <option value="hn">Hà Nội</option>
          </select>
        )}
      />
      <button>Submit</button>
    </form>
  );
}

// 3.7 watch vs useWatch — isolate re-render
import { useForm, useWatch } from "react-hook-form";

function WatchDemo() {
  const { register, control, watch } = useForm<{ a: string; b: string }>({
    defaultValues: { a: "", b: "" },
  });
  // ❌ watch toàn form — component này re-render mỗi khi a HOẶC b đổi
  const all = watch(); // hoặc watch("a")

  return (
    <>
      <input {...register("a")} placeholder="a" />
      <input {...register("b")} placeholder="b" />
      <IsolatedWatch control={control} />
    </>
  );
}
function IsolatedWatch({ control }: { control: any }) {
  // ✅ chỉ component này re-render khi a đổi, không re-render parent
  const a = useWatch({ control, name: "a" });
  return <div>a = {a}</div>;
}

// 3.8 useFieldArray — dynamic list tối ưu
import { useFieldArray } from "react-hook-form";

function FieldArrayDemo() {
  const { control, register, handleSubmit } = useForm<{ tags: { value: string }[] }>({
    defaultValues: { tags: [{ value: "" }] },
  });
  const { fields, append, remove } = useFieldArray({ control, name: "tags" });
  return (
    <form onSubmit={handleSubmit(console.log)}>
      {fields.map((f, i) => (
        <div key={f.id}>
          <input {...register(`tags.${i}.value` as const)} />
          <button type="button" onClick={() => remove(i)}>
            Xóa
          </button>
        </div>
      ))}
      <button type="button" onClick={() => append({ value: "" })}>
        Thêm
      </button>
    </form>
  );
}
```

## 4. So sánh / Phân loại

| Tiêu chí | Controlled | Uncontrolled (`ref`/`defaultValue`) | RHF (uncontrolled + ref) |
|----------|------------|-------------------------------------|--------------------------|
| Source of truth | React state (`value` + `onChange`) | DOM (`ref.current.value`) | DOM (RHF internal map via `ref`) + `defaultValues` |
| Re-render khi gõ | Có — mỗi keystroke | Không | Không (chỉ khi error/dirty/watch) |
| Khai báo | `value` / `checked` | `defaultValue` / `defaultChecked` | `register` + `defaultValues` |
| Validation | Thủ công trong `onChange` | Thủ công khi submit | `resolver` (zod/yup) + `mode` |
| Phù hợp | Form nhỏ 1-5 field cần live validate/format | Form đơn giản, file input | Form lớn 5+ field, cần performance + schema |
| Chi phí | Re-render nhiều | Khó dynamic UI (hiện lỗi ngay khi gõ cần watch) | Thêm dependency, học API |

| `watch` | `useWatch` | `getValues` |
|---------|------------|-------------|
| Re-render component gọi khi field đổi | Chỉ re-render component con dùng `useWatch` | Không re-render, đọc snapshot khi cần |
| Dùng cho hiển thị live toàn form | Dùng cho hiển thị live isolate | Dùng trong `onSubmit`/`onClick` |

| Khi nào `Controller` |
|----------------------|
| UI lib controlled (`MUI Select`, `AntD DatePicker`, `react-select`) không expose `ref` → phải dùng `Controller` để RHF điều khiển qua `field.value`/`field.onChange`. Input native thì `register` đủ. |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không controlled cho form lớn**: 20 field controlled ⇒ 20 `useState` + re-render toàn form mỗi keystroke ⇒ lag, nhất là khi có validation/format nặng. Dùng RHF.
- **Không uncontrolled thuần nếu cần live UI**: hiện lỗi ngay khi gõ, disable submit theo `isValid`, format mask theo input — uncontrolled cần `watch`/`useWatch` bù lại, phức tạp hơn controlled nhỏ.
- **Không quên `defaultValues`/`defaultValue`**: thiếu ⇒ React warning uncontrolled↔controlled, RHF không khởi tạo `formState`, `reset` không về đúng.
- **Không dùng `watch` bừa bãi**: `watch()` trong component lớn ⇒ re-render toàn form mỗi keystroke (mất lợi ích RHF). Dùng `useWatch` isolate hoặc `getValues` khi không cần render.
- **Không `Controller` cho input native**: `Controller` thêm layer controlled, chậm hơn `register` ref. Chỉ dùng khi UI lib yêu cầu.
- **Không `value` + `defaultValue` cùng lúc**: chọn một. `value` là controlled, `defaultValue` là uncontrolled — trộn gây warning.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Warning “changing uncontrolled to controlled”**
  - Triệu chứng: `value={undefined}` lúc đầu rồi có string sau.
  - Nguyên nhân: `value` ban đầu `undefined` (uncontrolled) sau thành string (controlled).
  - Fix: `value={value ?? ""}` hoặc `defaultValues` đầy đủ cho RHF; khởi `useState("")` không `undefined`.
  - Đo: console warning, React DevTools props.

- **Lỗi 2: `defaultValue` đổi không update UI**
  - Triệu chứng: prop `defaultValue` đổi nhưng input không đổi.
  - Nguyên nhân: `defaultValue` chỉ mount, không sync.
  - Fix: `key={defaultValue}` để remount, hoặc chuyển controlled `value`, hoặc RHF `reset({ ... })`.
  - Đo: inspect DOM, log `defaultValue`.

- **Lỗi 3: RHF `watch` làm re-render toàn form**
  - Triệu chứng: gõ 1 field, toàn form render, Profiler commit dài.
  - Fix: thay `watch()` bằng `useWatch({ control, name })` trong component con nhỏ, hoặc `getValues()` nếu không cần hiển thị.
  - Đo: React DevTools Profiler → highlight, `why-did-you-render` log.

- **Lỗi 4: Quên `defaultValues` trong RHF**
  - Triệu chứng: `register` uncontrolled warning, `reset` không hoạt động, `isDirty` sai.
  - Fix: luôn `useForm({ defaultValues: { ... } })` khớp schema.
  - Đo: RHF `formState.defaultValues`, console.

- **Lỗi 5: Controlled field trong RHF không qua `Controller`**
  - Triệu chứng: MUI `Select` không update, ref không gắn.
  - Fix: dùng `Controller` cho controlled lib.
  - Đo: UI không phản hồi, `field.value` log.

- **Lỗi 6: Validation chạy mỗi keystroke gây lag**
  - Triệu chứng: `mode: "onChange"` + `zodResolver` nặng ⇒ mỗi gõ validate toàn schema.
  - Fix: `mode: "onBlur"` hoặc `onSubmit`, hoặc `reValidateMode`.
  - Đo: Profiler, `performance.now()` trong resolver.

- **Công cụ**:
  - React DevTools Profiler — đo re-render khi typing.
  - `why-did-you-render` cho form.
  - RHF DevTools (`@hookform/devtools`): `<DevTool control={control} />` inspect form state.
  - ESLint `react-hooks/exhaustive-deps` cho `useWatch`.

## 7. Câu hỏi tự kiểm tra

1. Phân biệt `value` vs `defaultValue` và controlled vs uncontrolled. Khi nào đổi `defaultValue` không update DOM?
2. Vì sao React Hook Form nhanh hơn controlled cho form lớn? `register` + `ref` hoạt động thế nào và khi nào cần `Controller`?
3. `watch` vs `useWatch` vs `getValues` khác gì? Vì sao `watch()` trong component lớn làm mất lợi ích performance của RHF?

<details>
<summary>Đáp án 30s</summary>

1. `value` + `onChange` = controlled, React là source of truth, mỗi render ghi đè DOM. `defaultValue` = uncontrolled, chỉ set lúc mount, sau đó DOM tự quản lý. Đổi `defaultValue` sau mount không update vì React không sync — phải `key` để remount, hoặc chuyển controlled, hoặc RHF `reset()`. Trộn `value`/`defaultValue` gây warning.
2. Controlled mỗi keystroke `setState` ⇒ re-render. RHF dùng **uncontrolled + ref**: `register` gắn `ref` vào input, lưu value trong internal map (không state), chỉ re-render khi `errors`/`isDirty` đổi ⇒ gõ không render. `Controller` cần khi UI lib là controlled-only (MUI/AntD) không expose `ref` — RHF phải điều khiển qua `field.value`/`onChange`.
3. `watch()` subscribe và re-render component gọi khi field đổi; `watch()` toàn form ⇒ re-render rộng. `useWatch()` tạo subscription isolate — chỉ component con dùng `useWatch` re-render. `getValues()` đọc snapshot không subscribe, không re-render. Dùng `watch` toàn form trong component lớn làm mọi keystroke re-render toàn cây, mất tối ưu uncontrolled của RHF.

</details>

---
*Tham khảo chi tiết: `docs/02-react-co-ban.md` — Câu 34, 46. Spec: [React Docs — Controlled vs Uncontrolled](https://react.dev/reference/react-dom/components/input#controlling-an-input-with-a-state-variable), [RHF — get started](https://react-hook-form.com/get-started), [Controller](https://react-hook-form.com/docs/usecontroller/controller), [watch/useWatch](https://react-hook-form.com/docs/usewatch).*
