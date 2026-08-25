# URL & Form State — URL as source of truth (?category/page), useSearchParams, share/back/forward & RHF

> Tags: #URL-State #Form-State #useSearchParams #React-Hook-Form #SearchParams #Nuqs | Nguồn: `docs/09-state-management.md` câu 159, 160 + `docs/04-frontend-architecture.md` câu 75-76 | Mức: P0

## 1. Định nghĩa chính xác

**URL State** là state được **serialize vào URL** (`pathname` + `searchParams` `?category=shoes&page=2&sort=price&q=nike`), là **single source of truth** cho shareable state: **share link**, **bookmark**, **back/forward**, **SSR prefetch** và **queryKey** của Server State. **Form State** là giá trị/validation/dirty/touched của input; gồm **local `useState` (controlled)** và **React Hook Form (RHF)** (`uncontrolled + register + ref` — input tự quản DOM value, chỉ `onSubmit`/`onBlur` sync, giảm re-render). Hai loại này **không thuộc global store** (Zustand/Redux) khi cần share/url, và form draft chưa Apply không đẩy lên URL.

## 2. Cơ chế hoạt động

- **URL as state source — `useSearchParams` / `nuqs`**:
  - `const [searchParams, setSearchParams] = useSearchParams()` (react-router) đọc `searchParams.get('category')`, ghi `setSearchParams({ category: 'shoes', page: '1' }, { replace: false })` → `history.pushState` → URL đổi → component remount/read lại → `useQuery(['products', category, page])` fetch.
  - `nuqs` (`useQueryState`) typed hơn: `const [page, setPage] = useQueryState('page', parseAsInteger.withDefault(1))` + `shallow: false` để batch.
  - Đặc tính: **serializable string-only**, nên `priceRange: [0,100]` → `?price=0-100`, `brand: string[]` → `?brand=nike,adidas` hoặc `?brand=nike&brand=adidas` (`getAll`). Không lưu object phức tạp.
  - Back/forward: browser `popstate` → `searchParams` đổi → UI khôi phục → không cần store.
  - SSR: server đọc `req.url` → prefetch `queryClient.prefetchQuery(['products', params])` → hydrate.

- **URL vs Zustand vs local draft**:
  - **URL**: filter/sort/page/tab/search `q` cần share, SEO. **Zustand**: `isCartOpen` nhiều route nhưng không share. **Local `useState` draft**: slider `price` đang kéo chưa Apply → `const [draft, setDraft] = useState(priceParam)` → `onBlur`/`onClick Apply` mới `setSearchParams`.
  - Quy tắc: **draft local → committed URL → Server State queryKey**.

- **Form State — controlled `useState` vs RHF**:
  - Controlled: `<input value={value} onChange={e=>setValue(e.target.value)} />` — mỗi keystroke `setState` → re-render cả form (OK với 1-2 fields).
  - RHF: `<input {...register('email', { required: '...', pattern: ... })} />` — **uncontrolled**, `ref` đăng ký DOM, RHF quản `formState: { errors, isDirty, isValid, touchedFields }` via `Proxy`, chỉ field lỗi mới re-render (không re-render cả form). `resolver: zodResolver(schema)` cho validation typed (`zod`/`yup`).
  - RHF API: `handleSubmit(onValid)`, `watch`, `setValue`, `reset`, `control` cho `Controller` (custom Select/DatePicker controlled).

- **Form ↔ URL ↔ Server**:
  - Search: `input` (Form local) → `debounce 300ms` → `setSearchParams({ q: debounced })` hoặc `useQuery(['search', debounced], { enabled: debounced.length>=2 })`.
  - Filter form: `RHF` `defaultValues` từ `searchParams`, `onSubmit` → `setSearchParams(values)`.

```
Search input "nik" → useState local → debounce 300ms → "nike" → setSearchParams({ q: "nike" }) hoặc queryKey ['search','nike'] → TanStack Query fetch
Filter sidebar: RHF draft { brand, price } → Apply → setSearchParams({ brand: 'nike', price: '0-100', page: '1' }) → URL → ProductList reads → queryKey đổi → SWR fetch
Back button: popstate → searchParams "brand=adidas" → UI khôi phục không cần Zustand
```

## 3. Ví dụ tối thiểu

```tsx
// 3.1 URL State — useSearchParams, share link / back-forward / pagination
import { useSearchParams } from 'react-router-dom';
import { useQuery } from '@tanstack/react-query';
import { useState } from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

// Filter + pagination là URL — copy link là share
function ProductList() {
  const [searchParams, setSearchParams] = useSearchParams();
  const category = searchParams.get('category') ?? 'all';
  const page = Number(searchParams.get('page') ?? '1');
  const sort = searchParams.get('sort') ?? 'price';

  const { data, isPending } = useQuery({
    queryKey: ['products', { category, page, sort }], // URL → queryKey
    queryFn: ({ signal }) => fetch(`/api/products?category=${category}&page=${page}&sort=${sort}`, { signal }).then(r=>r.json()),
    staleTime: 60_000,
  });

  const setCategory = (c: string) => setSearchParams({ category: c, page: '1', sort });
  const setPage = (p: number) => setSearchParams({ category, page: String(p), sort });

  if (isPending) return <Skeleton />;
  return <>
    <select value={category} onChange={e=>setCategory(e.target.value)}>
      <option value="all">All</option><option value="shoes">Shoes</option>
    </select>
    {data.map((p: Product)=><Card key={p.id} product={p} />)}
    <button onClick={()=>setPage(page-1)} disabled={page<=1}>Prev</button>
    <button onClick={()=>setPage(page+1)}>Next — page {page}</button>
    {/* Copy URL ?category=shoes&page=2 — share là thấy đúng filter */}
  </>;
}

// nuqs typed (optional) — thay useSearchParams khi cần type/parse
// import { useQueryState, parseAsInteger } from 'nuqs';
// const [page, setPage] = useQueryState('page', parseAsInteger.withDefault(1));
// const [category, setCategory] = useQueryState('category', { defaultValue: 'all' });

// Draft local chưa Apply — không đẩy URL mỗi keystroke
function PriceFilter() {
  const [searchParams, setSearchParams] = useSearchParams();
  const committed = searchParams.get('price') ?? '0-100';
  const [draft, setDraft] = useState(committed); // local draft
  return <div>
    <input value={draft} onChange={e=>setDraft(e.target.value)} placeholder="0-100" />
    <button onClick={()=>setSearchParams({ ...Object.fromEntries(searchParams), price: draft, page: '1' })}>Apply</button>
  </div>;
}
```

```tsx
// 3.2 Form State — useState (nhỏ) vs RHF (nhiều field, validation, không re-render thừa)
import { useForm, Controller } from 'react-hook-form';

// Nhỏ 1-2 fields — useState đủ
function SearchInput({ onSearch }: { onSearch: (q: string)=>void }) {
  const [draft, setDraft] = useState('');
  return <input value={draft} onChange={e=>setDraft(e.target.value)} onBlur={()=>onSearch(draft)} placeholder="Search" />;
}

// Lớn hơn — RHF uncontrolled + zodResolver, chỉ field lỗi re-render
const schema = z.object({
  email: z.string().email('Email không hợp lệ'),
  password: z.string().min(8, 'Tối thiểu 8 ký tự'),
});
type FormValues = z.infer<typeof schema>;

function LoginForm() {
  const { register, handleSubmit, control, formState: { errors, isDirty, isValid, isSubmitting } } = useForm<FormValues>({
    resolver: zodResolver(schema), // typed validation
    mode: 'onBlur', // validate khi blur, không mỗi keystroke
    defaultValues: { email: '', password: '' },
  });

  const onSubmit = async (values: FormValues) => {
    await fetch('/api/login', { method: 'POST', body: JSON.stringify(values) });
  };

  return <form onSubmit={handleSubmit(onSubmit)}>
    <input {...register('email')} placeholder="Email" />
    {errors.email && <span>{errors.email.message}</span>}

    <input type="password" {...register('password')} />
    {errors.password && <span>{errors.password.message}</span>}

    {/* Controller cho custom component controlled (Select, DatePicker) */}
    <Controller name="email" control={control} render={({ field }) => <CustomInput {...field} />} />

    <button disabled={!isDirty || !isValid || isSubmitting}>Login</button>
  </form>;
}

// Form ↔ URL: defaultValues từ URL, submit đẩy URL
function FilterForm() {
  const [searchParams, setSearchParams] = useSearchParams();
  const { register, handleSubmit } = useForm<{ brand: string }>({
    defaultValues: { brand: searchParams.get('brand') ?? '' },
  });
  return <form onSubmit={handleSubmit(values => setSearchParams({ brand: values.brand, page: '1' }))}>
    <input {...register('brand')} />
    <button>Apply</button>
  </form>;
}
```

## 4. So sánh / Phân loại

| Tiêu chí | **URL State** (`useSearchParams`) | **Client Global** (Zustand) | **Form `useState`** | **RHF** | **Ephemeral `useState`** |
|----------|------------------------------------|------------------------------|---------------------|---------|---------------------------|
| **Persist** | URL (share, bookmark, back/forward, SSR) | Memory / `persist` | Không | Không | Không |
| **Re-render** | Chỉ component đọc `searchParams` | Per-selector | Mỗi keystroke re-render cả form | Uncontrolled — chỉ field đổi | Per-component |
| **Validation** | Không (chỉ parse) | Không | Tự viết | `resolver` (zod/yup) | Không |
| **Serialize** | String only | Bất kỳ | Bất kỳ | Bất kỳ | Bất kỳ |
| **Dùng cho** | `?category&page&sort&q&tab` | `isCartOpen` (nhiều route, không share) | 1-2 inputs đơn giản | Form nhiều field, validation | `isHovered` |

| URL State | Zustand/Redux | Khi nào chọn |
|-----------|---------------|--------------|
| `?brand=nike&page=2` (share, SEO, back) | `brand/page` trong store (không share) | URL khi cần share/back/SSR; Zustand chỉ khi internal wizard không cần link |
| `?q=nike` search term (bookmark) | `searchDraft` local | `q` cho results, draft input là Form local |
| `/products/shoes` (pathname) | `selectedCategory` in store | Pathname cho routing, không store |

| Form | Khi dùng | Khi KHÔNG dùng |
|------|----------|----------------|
| **`useState` controlled** | 1-2 fields, không validation phức tạp (search input, draft price) | Form 10 fields + validation → lag, logic rối |
| **RHF** | Nhiều field, `errors/isDirty/touched`, `zodResolver`, `Controller` cho custom input | 1 input đơn giản → thừa ceremony |

| URL `setSearchParams` option | Hành vi |
|------------------------------|---------|
| `setSearchParams({ page: '2' })` | `push` — back được |
| `setSearchParams({ page: '2' }, { replace: true })` | `replace` — không thêm history (dùng cho `q` khi gõ nhiều, tránh history spam) |
| `nuqs` `useQueryState('page', parseAsInteger)` | Typed + default, batch, shallow |


## 5. Trade-off / Khi nào KHÔNG dùng

- **Không lưu filter vào Zustand nếu cần share**: gửi link `/products` mà không có `?brand=nike` → người nhận thấy khác bạn, QA không reproduce, SEO không index filtered page. URL là source of truth cho shareable.
- **Không đẩy mọi keystroke lên URL**: `onChange={e=>setSearchParams({ q: e.target.value })}` mỗi ký tự → history spam (50 entries cho "nike"), back 50 lần mới thoát. Dùng **debounce 300ms** + `{ replace: true }` cho `q`, hoặc draft local → Apply mới push.
- **Không lưu object/complex vào URL**: `?filter={price:[0,100]}` JSON xấu, dài, không bookmark-friendly. Serialize phẳng `?price=0-100&brand=nike,adidas`.
- **Không dùng `useState` controlled cho form lớn**: 20 fields controlled → mỗi keystroke re-render 20 inputs, INP cao. Dùng RHF `register` uncontrolled.
- **Không dùng RHF cho 1 input đơn**: `SearchInput` 1 field mà `useForm` là overkill (thêm `resolver`, `handleSubmit`). `useState` đủ.
- **Không dùng URL cho `cart`, `isModalOpen`**: cart trước login là client state, không share; URL `/cart?items=...` lộ data, dài, không typed. Dùng Zustand `persist`.
- **Không quên `Controller` cho custom component**: `Select`/`DatePicker` controlled không `register` được → dùng `Controller` + `control`.
- **Chi phí**: URL State thêm parse/stringify (`get`/`getAll`, `Number()`), cần `zod` cho validate URL; RHF thêm bundle ~9KB nhưng tiết kiệm re-render 90%.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Mỗi keystroke `setSearchParams` → history spam, back 50 lần**
  - Triệu chứng: gõ "nike" → history 4 entries (`n`, `ni`, `nik`, `nike`), back không về trang trước.
  - Fix: `debounce 300ms` + `setSearchParams({ q: debounced }, { replace: true })` (replace, không push) hoặc `onBlur`/`Apply` mới push.
  - Đo: DevTools → Application → History stack; click back thủ công.

- **Lỗi 2: Filter trong Zustand → mất khi share/back/refresh**
  - Triệu chứng: copy URL `/products` gửi đồng nghiệp → họ thấy `all` dù bạn đang `shoes`.
  - Fix: chuyển `category/page/sort/q` sang `useSearchParams`; `useQuery` key đọc từ URL.
  - Đo: mở URL trong incognito, check filter; back/forward.

- **Lỗi 3: Form controlled 20 fields → lag khi gõ**
  - Triệu chứng: gõ 1 ký tự, cả form re-render, INP 400ms.
  - Fix: chuyển sang RHF `register` (uncontrolled) + `mode: 'onBlur'`, `Controller` cho custom.
  - Đo: React DevTools Profiler → flamegraph, render count/keystroke; WDYR; Lighthouse INP.

- **Lỗi 4: RHF quên `resolver` → validation rải rác, không typed**
  - Triệu chứng: `if (!email.includes('@')) setError(...)` thủ công, không đồng bộ `zod` schema với `TypeScript`.
  - Fix: `zodResolver(schema)` + `z.infer<typeof schema>` cho `FormValues`.
  - Đo: type errors, test validation.

- **Lỗi 5: `searchParams.get('page')` không parse → `"2" + 1 = "21"`**
  - Triệu chứng: `page = searchParams.get('page') ?? 1` là string, `page + 1` sai.
  - Fix: `Number(searchParams.get('page') ?? '1')` hoặc `nuqs parseAsInteger`.
  - Đo: console.log typeof, pagination sai.

- **Lỗi 6: Draft và URL không sync → Apply không cập nhật UI**
  - Triệu chứng: `PriceFilter` `draft` local nhưng `searchParams` đổi từ back → draft vẫn cũ.
  - Fix: `useEffect(() => setDraft(searchParams.get('price') ?? '0-100'), [searchParams])` hoặc derive thẳng từ URL không draft nếu không cần.
  - Đo: click back, check input value.

- **Công cụ**:
  - **URL bar + History** — share, back/forward, bookmark.
  - **React DevTools Profiler + WDYR** — form re-render, RHF vs controlled.
  - **TanStack Query DevTools** — `queryKey` từ URL, stale/fresh.
  - **RHF DevTools** (`@hookform/devtools`) — `formState`, `errors`, `isDirty`.
  - **Network** — `?category&page` query string đúng không.

## 7. Câu hỏi tự kiểm tra

1. Vì sao filter/sort/page/q nên là URL State (`useSearchParams`) thay vì Zustand, và `setSearchParams` khi nào `push` vs `replace` + vì sao cần debounce/draft trước khi đẩy URL?
2. Phân biệt Form `useState` controlled vs RHF `register` (uncontrolled) — khi nào dùng cái nào, `resolver: zodResolver` và `Controller` dùng cho gì, vì sao RHF giảm re-render?
3. Luồng URL → `queryKey` → Server State và Form ↔ URL phối hợp thế nào (search debounce, filter Apply), và URL không nên lưu gì (`cart`, object phức tạp)?

<details>
<summary>Đáp án 30s</summary>

1. **URL State** `?category&page&sort&q` **share link**, **back/forward** (`popstate`), **bookmark**, **SSR prefetch**, là `queryKey` cho `useQuery`; Zustand không share, không survive refresh, không SEO. **`push`** (default) thêm history — back được, dùng cho pagination/category; **`replace`** không thêm history — dùng cho `q` khi gõ nhiều (debounce 300ms + `replace: true`) tránh history spam (gõ "nike" không tạo 4 entries). **Draft local** (`useState` cho slider đang kéo) trước khi Apply → `setSearchParams` để không đẩy mỗi keystroke, chỉ commit mới push.
2. **`useState` controlled** `value`/`onChange` mỗi keystroke `setState` → re-render cả form, OK cho 1-2 fields. **RHF** `register` là **uncontrolled** + `ref`, input tự quản DOM, chỉ `onSubmit`/`onBlur` sync, `formState` (`errors/isDirty/isValid`) qua `Proxy` nên chỉ field lỗi mới re-render — giảm 90%. **`zodResolver(schema)`** validate typed (zod ↔ `FormValues`), **`Controller`** cho custom controlled component (`Select`/`DatePicker`) không `register` được. Dùng RHF cho form nhiều field/validation, `useState` cho 1 input đơn.
3. **URL → queryKey**: `searchParams.get('category')` → `useQuery(['products', { category, page, sort }])` → URL đổi → key đổi → SWR fetch. **Form ↔ URL**: search `input` local (Form) → debounce → `setSearchParams({ q }, { replace:true })` hoặc `queryKey ['search', debounced]`; filter **RHF** `defaultValues` từ `searchParams`, `onSubmit` → `setSearchParams(values)`. **Không lưu** `cart` (client persist, không share), object phức tạp JSON (`?filter={price:[0,100]}`) → serialize phẳng `?price=0-100&brand=nike`, `isModalOpen` ephemeral.

</details>

---
*Tham khảo chi tiết: `docs/09-state-management.md` — Câu 159, 160, `docs/04-frontend-architecture.md` — Câu 75-76. Liên quan: `01-state-classification.md` (5 loại), `02-tanstack-query.md` (queryKey từ URL), `03-zustand-redux-jotai.md` (khi nào không Zustand). Spec: [React Router — useSearchParams](https://reactrouter.com/en/main/hooks/use-search-params), [React Hook Form](https://react-hook-form.com/), [nuqs](https://nuqs.47ng.com/).*
