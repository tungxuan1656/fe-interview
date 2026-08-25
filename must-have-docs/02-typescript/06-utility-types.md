# Utility Types — Tự triển khai Partial/Required/Pick/Omit/Record/Exclude/Extract/ReturnType/Awaited

> Tags: #typescript #utility-types #mapped-types #conditional-types #infer | Nguồn: `docs/01-javascript-typescript.md` câu 22 | Mức: P0

## 1. Định nghĩa chính xác

**Utility Types** là các type alias có sẵn trong `lib.d.ts` dùng mapped type + conditional + `infer` để biến đổi type. Nắm triển khai giúp hiểu `keyof`, `in`, `extends`, `never` distributive, `infer`, và viết utility riêng cho domain (`DeepPartial`, `StrictOmit`). Tám utility lõi: `Partial`/`Required`/`Readonly` (modifier), `Pick`/`Omit`/`Record` (chọn/bỏ/map key), `Exclude`/`Extract` (filter union), `ReturnType`/`Awaited` (unwrap hàm/Promise).

## 2. Cơ chế hoạt động

### 2.1 Modifier utilities — `Partial`, `Required`, `Readonly`

Dựa trên mapped type với modifier `?`/`-?`/`readonly`/`-readonly`:

```ts
type Partial<T>  = { [P in keyof T]?: T[P] };
type Required<T> = { [P in keyof T]-?: T[P] };
type Readonly<T> = { readonly [P in keyof T]: T[P] };
```

`?` thêm optional, `-?` bỏ optional. Không ghi modifier → giữ nguyên. Chỉ shallow — nested object không ảnh hưởng (cần `DeepPartial` đệ quy).

### 2.2 Key selection — `Pick`, `Omit`, `Record`

- **Pick**: chọn subset key, `K extends keyof T`.
- **Omit**: bỏ key, triển khai qua `Pick` + `Exclude`.
- **Record**: map `K` → `T`, `K extends keyof any` (`string | number | symbol`).

```ts
type Pick<T, K extends keyof T>   = { [P in K]: T[P] };
type Record<K extends keyof any, T> = { [P in K]: T };
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;
```

`Omit` dùng `Exclude` để loại `K` khỏi `keyof T`, rồi `Pick` phần còn lại — distributive nên hoạt động đúng với union.

### 2.3 Union filtering — `Exclude`, `Extract`, `NonNullable`

Dựa trên distributive conditional + `never`:

```ts
type Exclude<T, U> = T extends U ? never : T;
type Extract<T, U> = T extends U ? T : never;
type NonNullable<T> = T extends null | undefined ? never : T; // hoặc T & {}
```

`never` là identity của union nên nhánh `never` biến mất: `Exclude<"a"|"b", "a">` → `"b"`.

### 2.4 Function/Promise unwrapping — `ReturnType`, `Parameters`, `Awaited`

Dùng `infer` trong conditional:

```ts
type ReturnType<T extends (...args: any) => any> = T extends (...args: any) => infer R ? R : any;
type Parameters<T extends (...args: any) => any> = T extends (...args: infer P) => any ? P : never;
type Awaited<T> = T extends null | undefined ? T : T extends object & { then(onfulfilled: infer F): any }
  ? F extends (v: infer V) => any ? Awaited<V> : never : T; // lib.d.ts dùng PromiseLike
// Đơn giản:
type SimpleAwaited<T> = T extends PromiseLike<infer U> ? SimpleAwaited<U> : T;
```

`Awaited` đệ quy để unwrap `Promise<Promise<string>>` → `string`.

## 3. Ví dụ tối thiểu

```ts
// 3.1 Triển khai đầy đủ 8 utility + test
type MyPartial<T>    = { [P in keyof T]?: T[P] };
type MyRequired<T>   = { [P in keyof T]-?: T[P] };
type MyReadonly<T>   = { readonly [P in keyof T]: T[P] };
type MyPick<T, K extends keyof T> = { [P in K]: T[P] };
type MyRecord<K extends keyof any, T> = { [P in K]: T };
type MyExclude<T, U> = T extends U ? never : T;
type MyExtract<T, U> = T extends U ? T : never;
type MyNonNullable<T> = T extends null | undefined ? never : T;
type MyOmit<T, K extends keyof any> = MyPick<T, MyExclude<keyof T, K>>;
type MyReturnType<T extends (...args: any) => any> = T extends (...args: any) => infer R ? R : any;
type MyAwaited<T> = T extends PromiseLike<infer U> ? MyAwaited<U> : T;
type MyParameters<T extends (...args: any) => any> = T extends (...args: infer P) => any ? P : never;

// --- Test ---
type User = { id: string; name: string; age?: number; active: boolean };

type TPartial = MyPartial<User>;   // { id?: string; name?: string; age?: number; active?: boolean }
type TRequired = MyRequired<User>; // { id: string; name: string; age: number; active: boolean }
type TPick = MyPick<User, "id" | "name">; // { id: string; name: string }
type TOmit = MyOmit<User, "active">; // { id: string; name: string; age?: number }
type TRecord = MyRecord<"a" | "b", number>; // { a: number; b: number }
type TExclude = MyExclude<"a" | "b" | "c", "a">; // "b" | "c"
type TExtract = MyExtract<"a" | "b" | "c", "a" | "b">; // "a" | "b"
type TNonNull = MyNonNullable<string | null | undefined>; // string
type TRet = MyReturnType<() => string>; // string
type TAwaited = MyAwaited<Promise<Promise<number>>>; // number
type TParams = MyParameters<(a: string, b: number) => void>; // [string, number]

// 3.2 So với lib — NonNullable dùng T & {} (shorthand)
type NonNullableShorthand<T> = T & {}; // loại null/undefined vì {} không chứa null/undefined
// Nhưng T extends null|undefined ? never : T rõ nghĩa hơn, nên dùng

// 3.3 Deep utility cho form/domain
type DeepPartial<T> = T extends object ? { [P in keyof T]?: DeepPartial<T[P]> } : T;
type DeepReadonly<T> = T extends (...args: any) => any ? T : T extends object ? { readonly [P in keyof T]: DeepReadonly<T[P]> } : T;
type DeepRequired<T> = { [P in keyof T]-?: DeepRequired<T[P]> };

type Form = { user: { name: string; address: { city: string } }; tags: string[] };
type FormPatch = DeepPartial<Form>; // mọi field optional đệ quy

// 3.4 StrictOmit — bắt lỗi khi K không thuộc T
type StrictOmit<T, K extends keyof T> = MyPick<T, MyExclude<keyof T, K>>;
// Omit thường cho phép K extends keyof any nên không bắt K sai; StrictOmit bắt

// 3.5 Thực tế: API response transform
type ApiUser = { id: string; name: string; password: string; createdAt: string };
type PublicUser = MyOmit<ApiUser, "password">; // bỏ field nhạy cảm
type UserUpdate = MyPartial<MyPick<ApiUser, "name">> & { id: string }; // PATCH

// 3.6 Awaited trong thực tế
async function fetchUser(): Promise<User> { return null as any; }
type FetchedUser = MyAwaited<ReturnType<typeof fetchUser>>; // User (unwrap Promise<User>)
async function fetchWithRetry(): Promise<Promise<User>> { return null as any; }
type Unwrapped = MyAwaited<ReturnType<typeof fetchWithRetry>>; // User
```

```ts
// 3.7 InstanceType / ConstructorParameters
type MyConstructorParameters<T extends abstract new (...args: any) => any> =
  T extends abstract new (...args: infer P) => any ? P : never;
type MyInstanceType<T extends abstract new (...args: any) => any> =
  T extends abstract new (...args: any) => infer R ? R : any;
class Foo { constructor(public x: number) {} }
type FI = MyInstanceType<typeof Foo>; // Foo
type FP = MyConstructorParameters<typeof Foo>; // [number]
```

## 4. So sánh / Phân loại

| Utility | Triển khai lõi | Dùng khi |
|---------|----------------|----------|
| `Partial<T>` | `{ [P in keyof T]?: T[P] }` | PATCH, form, options optional |
| `Required<T>` | `{ [P in keyof T]-?: T[P] }` | Bắt buộc hết sau khi merge default |
| `Readonly<T>` | `{ readonly [P in keyof T]: T[P] }` | Props/state không mutate |
| `Pick<T,K>` | `{ [P in K]: T[P] }` | Chọn subset |
| `Omit<T,K>` | `Pick<T, Exclude<keyof T, K>>` | Bỏ key (an toàn hơn `Pick` ngược) |
| `Record<K,T>` | `{ [P in K]: T }` | Map key → value, enum map |
| `Exclude<T,U>` | `T extends U ? never : T` | Loại khỏi union |
| `Extract<T,U>` | `T extends U ? T : never` | Giữ lại trong union |
| `NonNullable<T>` | `T extends null\|undefined ? never : T` | Loại null/undefined |
| `ReturnType<T>` | `T extends (...args:any)=>infer R ? R : never` | Lấy return của hàm |
| `Awaited<T>` | `T extends PromiseLike<infer U> ? Awaited<U> : T` | Unwrap Promise đệ quy |
| `Parameters<T>` | `T extends (...args:infer P)=>any ? P : never` | Tuple args |

| Shallow vs Deep | Ví dụ | Ghi chú |
|-----------------|-------|---------|
| Shallow (`Partial`) | `{ a?: T }` | Chỉ 1 cấp |
| Deep (`DeepPartial`) | `{ a?: DeepPartial<T> }` | Đệ quy, cần base case `T extends object` |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không lồng `Omit`/`Pick` quá nhiều trên union lớn**: `Omit<Union, K>` distributive trên từng thành viên, với union hàng chục variant sẽ sinh type khổng lồ, chậm. Thay bằng mapped một lần hoặc `interface extends`.
- **Không nhầm `Partial` với `DeepPartial`**: `Partial` chỉ optional 1 cấp, nested vẫn required. Form lồng nhau cần `DeepPartial`.
- **Không dùng `Omit` với `K` không thuộc `T` mà mong báo lỗi**: `Omit<T, "wrongKey">` không lỗi vì `K extends keyof any`. Dùng `StrictOmit<T, K extends keyof T>` để bắt.
- **Không dùng `Record<string, T>` khi cần key cụ thể**: `Record<string, T>` cho phép mọi string, mất an toàn. Dùng `Record<"a"|"b", T>` hoặc `Map`.
- **Không dùng `ReturnType` trên overload sai**: với hàm overload, `ReturnType` lấy return của **implementation signature** (thường là union), không phải overload cụ thể. Cần conditional riêng.
- **Chi phí**: utility đệ quy (`DeepPartial`, `Awaited`) trên object sâu dễ hit `excessively deep`. Giới hạn độ sâu hoặc dùng iterative.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: `Omit` trên union không distributive như ý**
  - Triệu chứng: `Omit<A|B, "x">` ra `never` hoặc sai.
  - Nguyên nhân: `Omit` hiện đại trong lib đã distributive, nhưng `MyOmit` tự viết thiếu `T extends any ? ... : never` wrapper.
  - Fix: `type DistOmit<T, K extends keyof any> = T extends any ? Pick<T, Exclude<keyof T, K>> : never;`
  - Đo: `tsc --strict` + hover.

- **Lỗi 2: `Partial` làm mất `readonly`**
  - Triệu chứng: `Partial<Readonly<T>>` mất `readonly` nếu tự viết thiếu.
  - Fix: giữ modifier gốc: `{ [P in keyof T]?: T[P] }` giữ `readonly` mặc định; chỉ `?` thay đổi.

- **Lỗi 3: `Awaited` không unwrap `PromiseLike` custom**
  - Triệu chứng: `Awaited<Thenable<string>>` không ra `string`.
  - Fix: dùng `PromiseLike` thay vì `Promise` trong constraint.

- **Lỗi 4: `NonNullable` shorthand `T & {}` che nghĩa**
  - Triệu chứng: `T & {}` hoạt động nhưng khó hiểu và không loại `void` đúng.
  - Fix: dùng `T extends null|undefined ? never : T` tường minh.

- **Lỗi 5: Utility đệ quy OOM**
  - Triệu chứng: `Type instantiation is excessively deep`.
  - Fix: thêm base `T extends object ? ... : T`, giới hạn recursion.

- **Công cụ**:
  - `tsc --strict --noEmit`, `tsc --extendedDiagnostics`.
  - `expect-type` / `tsd` test utility: `expectType<MyPartial<User>>({})`.
  - `type-coverage` đo độ phủ.

## 7. Câu hỏi tự kiểm tra

1. Tự triển khai `Pick`, `Omit`, `Exclude`, `ReturnType`, `Awaited`. Giải thích vai trò của `infer` và `never` trong từng cái.
2. Vì sao `Omit<T, K>` được triển khai qua `Pick<T, Exclude<keyof T, K>>` thay vì mapped trực tiếp? Lợi ích của `Exclude` distributive là gì?
3. Phân biệt `Partial<T>` và `DeepPartial<T>`. Khi nào `Partial` không đủ và cần đệ quy? Viết `DeepPartial` và nêu base case.

<details>
<summary>Đáp án 30s</summary>

1. `Pick<T,K> = { [P in K]: T[P] }` (iterate `K`); `Exclude<T,U> = T extends U ? never : T` (distributive loại `U`); `Omit = Pick<T, Exclude<keyof T, K>>`; `ReturnType<T> = T extends (...args:any)=>infer R ? R : never` (`infer R` bắt return); `Awaited<T> = T extends PromiseLike<infer U> ? Awaited<U> : T` (infer + đệ quy unwrap). `never` loại nhánh, `infer` bắt type con.
2. Vì `keyof T` là union, `Exclude<keyof T, K>` distributive loại từng key thuộc `K`, rồi `Pick` chọn phần còn lại — tận dụng distributive để hoạt động đúng với union `T`. Mapped trực tiếp `{ [P in keyof T as P extends K ? never : P]: T[P] }` cũng được (TS 4.1+) nhưng `Pick+Exclude` là cách lib chuẩn và rõ hơn.
3. `Partial` chỉ `?:` 1 cấp, nested object vẫn required. `DeepPartial<T> = T extends object ? { [P in keyof T]?: DeepPartial<T[P]> } : T` — đệ quy cho tới khi `T` không còn object thì trả `T` (base case). Cần cho form/API patch lồng nhau.

</details>

---
*Tham khảo chi tiết: `docs/01-javascript-typescript.md` — Câu 22. Spec: [TS Handbook — Utility Types](https://www.typescriptlang.org/docs/handbook/utility-types.html), [Mapped Types](https://www.typescriptlang.org/docs/handbook/2/mapped-types.html).*
