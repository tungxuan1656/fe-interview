# Conditional Types, infer & Distributive — Logic rẽ nhánh ở tầng kiểu

> Tags: #typescript #conditional-types #infer #distributive #type-level | Nguồn: `docs/01-javascript-typescript.md` câu 23 | Mức: P1

## 1. Định nghĩa chính xác

**Conditional type** `T extends U ? X : Y` là biểu thức rẽ nhánh ở tầng type: nếu `T` assignable cho `U` thì kết quả là `X`, ngược lại `Y`. **Distributive conditional type** xảy ra khi `T` là **naked type parameter** (không bọc trong `[]`, `Promise<>`, `&`, `|`) — khi đó conditional tự **phân phối** trên từng thành viên của union `T` rồi hợp lại. **`infer`** là từ khóa đặt trong nhánh `extends` để "bắt" type con ra và đặt tên biến kiểu (`infer R`), dùng được trong `X`.

## 2. Cơ chế hoạt động

### 2.1 Conditional cơ bản

```ts
type IsString<T> = T extends string ? true : false;
type A = IsString<string>; // true
type B = IsString<number>; // false
type C = IsString<string | number>; // boolean? → xem distributive
```

Checker kiểm tra `T extends U` bằng **assignability**, không phải equality. `string extends string | number` → true.

### 2.2 Distributive — split union

Khi `T` là naked param và `T` là union, conditional phân phối:

```ts
type ToArray<T> = T extends any ? T[] : never;
type X = ToArray<string | number>; // string[] | number[]  (từng thành viên → array rồi hợp)
```

Tương đương:

```ts
type X = (string extends any ? string[] : never) | (number extends any ? number[] : never)
     = string[] | number[]
```

**Tắt distributive**: bọc `T` trong tuple hoặc thêm `& {}`:

```ts
type NoDist<T> = [T] extends [any] ? T[] : never;
type Y = NoDist<string | number>; // (string | number)[]  — không phân phối
```

`Exclude`, `Extract` dựa hoàn toàn vào distributive:

```ts
type Exclude<T, U> = T extends U ? never : T; // phân phối, loại nhánh never
type Extract<T, U> = T extends U ? T : never;
type E = Exclude<"a" | "b" | "c", "a">; // "b" | "c" — "a" → never, bị loại
```

Vì `never` là identity của union (`string | never` = `string`), nhánh `never` biến mất.

### 2.3 `infer` — pattern matching ở tầng type

`infer R` đặt trong `extends` để bắt type:

```ts
type ReturnType<T> = T extends (...args: any) => infer R ? R : never;
type ElementType<T> = T extends (infer E)[] ? E : never;
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;
type UnwrapPromiseDeep<T> = T extends Promise<infer U> ? UnwrapPromiseDeep<U> : T;
```

`infer` có thể đặt nhiều vị trí, kể cả với `...`:

```ts
type Last<T extends any[]> = T extends [...infer _, infer L] ? L : never;
type Head<T extends any[]> = T extends [infer H, ...any[]] ? H : never;
type Tail<T extends any[]> = T extends [any, ...infer Rest] ? Rest : never;

// Route params
type Params<T extends string> =
  T extends `${string}:${infer P}/${infer Rest}` ? P | Params<Rest> :
  T extends `${string}:${infer P}` ? P : never;
type P = Params<"/post/:postId/comment/:commentId">; // "postId" | "commentId"
```

`infer` chỉ hoạt động trong nhánh `extends` của conditional. Có thể `infer` cùng lúc nhiều biến.

### 2.4 Đệ quy conditional

```ts
type Flatten<T> = T extends (infer U)[] ? Flatten<U> : T;
type F = Flatten<number[][][]>; // number

type DeepReadonly<T> = T extends (...args: any) => any ? T : T extends object ? { readonly [P in keyof T]: DeepReadonly<T[P]> } : T;
```

Đệ quy + distributive + `infer` là bộ ba tạo utility `Awaited`, `ReturnType`, `Flatten`.

### 2.5 So với overload/function

Conditional cho phép viết một type duy nhất xử lý nhiều trường hợp, thay vì overload. Nhưng error message khó đọc hơn.

## 3. Ví dụ tối thiểu

```ts
// 3.1 Conditional + distributive
type ToArray<T> = T extends any ? T[] : never;
type A = ToArray<string | number>; // string[] | number[]
type B = ToArray<string | number | boolean>; // string[] | number[] | boolean[]

// Tắt distributive
type NoDist<T> = [T] extends [any] ? T[] : never;
type C = NoDist<string | number>; // (string | number)[]
type D = NoDist<string | number | boolean>; // (string | number | boolean)[]

// 3.2 Exclude / Extract triển khai
type MyExclude<T, U> = T extends U ? never : T;
type MyExtract<T, U> = T extends U ? T : never;
type E1 = MyExclude<"a" | "b" | "c", "a" | "b">; // "c"
type E2 = MyExtract<"a" | "b" | "c", "a" | "b">; // "a" | "b"
type NonNullable<T> = T extends null | undefined ? never : T;

// 3.3 infer — ReturnType, Parameters, Awaited
type MyReturnType<T extends (...args: any) => any> = T extends (...args: any) => infer R ? R : never;
type R1 = MyReturnType<() => string>; // string
type R2 = MyReturnType<(x: number) => boolean>; // boolean

type MyParameters<T extends (...args: any) => any> = T extends (...args: infer P) => any ? P : never;
type P1 = MyParameters<(a: string, b: number) => void>; // [string, number]

type MyAwaited<T> = T extends PromiseLike<infer U> ? MyAwaited<U> : T;
type AW1 = MyAwaited<Promise<string>>; // string
type AW2 = MyAwaited<Promise<Promise<number>>>; // number (đệ quy)

// 3.4 infer với tuple
type First<T extends any[]> = T extends [infer H, ...any[]] ? H : never;
type Last<T extends any[]> = T extends [...infer _, infer L] ? L : never;
type F1 = First<[1,2,3]>; // 1
type L1 = Last<[1,2,3]>; // 3

// 3.5 infer kết hợp template literal
type Getters<T> = { [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K] };
type G = Getters<{ name: string; age: number }>; // { getName: ()=>string; getAge: ()=>number }

type RouteParams<T extends string> =
  T extends `${string}:${infer Param}/${infer Rest}` ? Param | RouteParams<Rest> :
  T extends `${string}:${infer Param}` ? Param : never;
type RP = RouteParams<"/user/:id/post/:postId">; // "id" | "postId"

// 3.6 Flatten đệ quy
type Flatten<T> = T extends (infer U)[] ? Flatten<U> : T;
type F2 = Flatten<number[][]>; // number
type F3 = Flatten<string[]>; // string

// 3.7 Distributive trap — so sánh
type IsNever<T> = T extends never ? true : false;
type IN1 = IsNever<never>; // never (distributive trên never → không nhánh nào chạy → never)
type IsNeverFixed<T> = [T] extends [never] ? true : false;
type IN2 = IsNeverFixed<never>; // true — bọc tuple tắt distributive nên check đúng
```

```ts
// 3.8 Thực tế: type-safe event emitter
type Events = { click: { x: number }; hover: { id: string } };
type Emit<K extends keyof Events> = (event: K, payload: Events[K]) => void;
type HandlerMap = { [K in keyof Events]: (p: Events[K]) => void };
// Conditional giúp infer payload từ event name
```

## 4. So sánh / Phân loại

| Khái niệm | Cú pháp | Tác dụng |
|-----------|---------|----------|
| Conditional | `T extends U ? X : Y` | Rẽ nhánh |
| Distributive | `T` naked param + union | Split union, hợp lại |
| Tắt distributive | `[T] extends [U] ? X : Y` | Giữ union nguyên |
| `infer` | `T extends Promise<infer U>` | Bắt type con |
| Đệ quy | `type F<T> = T extends ... ? F<...> : T` | Unwrap nhiều lớp |

| Utility | Triển khai | Dựa trên |
|---------|------------|----------|
| `Exclude<T,U>` | `T extends U ? never : T` | distributive + never |
| `Extract<T,U>` | `T extends U ? T : never` | distributive |
| `NonNullable<T>` | `T extends null \| undefined ? never : T` | distributive |
| `ReturnType<T>` | `T extends (...args:any)=>infer R ? R : never` | infer |
| `Awaited<T>` | `T extends PromiseLike<infer U> ? Awaited<U> : T` | infer + đệ quy |
| `InstanceType<T>` | `T extends abstract new (...args:any)=>infer R ? R : any` | infer |

| Bật/Tắt distributive | Khi dùng |
|----------------------|----------|
| Bật (naked `T`) | Muốn filter từng thành viên (`Exclude`) |
| Tắt (`[T]`) | Muốn xử lý union như một khối (`IsNever`, `ToArray` giữ nguyên) |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không lạm dụng conditional lồng nhau quá sâu**: `T extends A ? X : T extends B ? Y : Z` làm error message khó đọc, compile chậm. Tách thành type alias trung gian.
- **Không quên distributive khi không muốn split**: `T extends any ? T[] : never` với `T = string|number` ra `string[]|number[]` chứ không phải `(string|number)[]`. Nếu muốn giữ nguyên, bọc `[T]`.
- **Không dùng `infer` ngoài `extends`**: `infer` chỉ hợp lệ trong `T extends ... infer R ? ...`. Đặt sai chỗ báo `Cannot find name 'infer'`.
- **Cẩn thận `never` distributive**: `T extends never ? true : false` với `T = never` ra `never` (không phải `true`) do distributive trên `never` không chạy nhánh nào. Phải `[T] extends [never]`.
- **Chi phí**: conditional đệ quy + distributive trên union lớn có thể làm `type instantiation is excessively deep`.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Distributive không mong muốn**
  - Triệu chứng: `ToArray<string|number>` ra `string[]|number[]` khác ý định `(string|number)[]`.
  - Fix: `type ToArray<T> = [T] extends [any] ? T[] : never` hoặc `T extends any ? ...` → `[T] extends ...`.
  - Đo: hover type trong VSCode, `tsc --noEmit` kiểm tra.

- **Lỗi 2: `infer` không bắt được**
  - Triệu chứng: `T extends Promise<infer U> ? U : T` luôn ra `T` dù `T` là `Promise<string>`.
  - Nguyên nhân: `T` không naked hoặc `T` là union với nhánh không phải Promise.
  - Fix: đảm bảo `T` là naked param, hoặc thêm distributive handling.

- **Lỗi 3: `IsNever` sai**
  - Triệu chứng: `IsNever<never>` ra `never` thay vì `true`.
  - Fix: `type IsNever<T> = [T] extends [never] ? true : false`.
  - Đo: unit test type với `Expect<Equal<...>>`.

- **Lỗi 4: Đệ quy quá sâu**
  - Triệu chứng: `Type instantiation is excessively deep and possibly infinite`.
  - Fix: giới hạn độ sâu, thêm base case `T extends any[] ? ... : T`, tránh `Flatten` trên type quá lồng.
  - Đo: `tsc --extendedDiagnostics` xem instantiation count.

- **Lỗi 5: `infer` nhiều biến sai vị trí**
  - Triệu chứng: `T extends [infer A, infer B] ? ...` lỗi khi `T` không phải tuple.
  - Fix: constraint `T extends any[]` trước, hoặc dùng `...infer Rest`.

- **Công cụ**:
  - `tsc --strict --noEmit`, hover type, `type-fest` `Expect`, `tsd` test type.
  - `tsc --extendedDiagnostics` đo instantiation.

## 7. Câu hỏi tự kiểm tra

1. Vì sao `type ToArray<T> = T extends any ? T[] : never` với `T = string | number` lại ra `string[] | number[]` chứ không phải `(string | number)[]`? Làm sao để ra `(string | number)[]`?
2. Giải thích `infer` trong `type ReturnType<T> = T extends (...args:any)=>infer R ? R : never`. `infer R` được gán khi nào và dùng ở đâu?
3. `Exclude<T,U>` được triển khai `T extends U ? never : T`. Vì sao nó loại được `"a"` khỏi `"a"|"b"|"c"` khi `U="a"`? Vai trò của distributive và `never` là gì?

<details>
<summary>Đáp án 30s</summary>

1. Vì `T` là naked param nên conditional distributive: split `string|number` thành `string extends any ? string[]` và `number extends any ? number[]` rồi hợp `|`. Muốn giữ nguyên, tắt distributive bằng `[T] extends [any] ? T[] : never` — bọc trong tuple nên không phân phối, cả union được check như một khối.
2. Khi `T` khớp `(...args:any)=>infer R` (tức `T` là hàm), checker suy ra `R` chính là return type của `T` và gán cho `R`. `R` sau đó dùng ở nhánh `? R`. Nếu `T` không phải hàm thì rơi vào `: never`.
3. Distributive split `"a"|"b"|"c"` thành ba lần check: `"a" extends "a" ? never : "a"` → `never`, `"b"` → `"b"`, `"c"` → `"c"`. Hợp lại `never|"b"|"c"` = `"b"|"c"` vì `never` là identity của union nên biến mất — kết quả loại `"a"`.

</details>

---
*Tham khảo chi tiết: `docs/01-javascript-typescript.md` — Câu 23. Spec: [TS Handbook — Conditional Types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html), [infer](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html#inferring-within-conditional-types).*
