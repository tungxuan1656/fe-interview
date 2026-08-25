# Mapped Types & Template Literal Types — Biến đổi khóa và chuỗi ở tầng kiểu

> Tags: #typescript #mapped-types #template-literal #key-remapping #intrinsic | Nguồn: `docs/01-javascript-typescript.md` câu 24 | Mức: P1

## 1. Định nghĩa chính xác

**Mapped type** `{ [P in keyof T]: ... }` là toán tử biến đổi từng property của `T` bằng cách iterate qua `keyof T`, cho phép thêm/bỏ `readonly`/`?` và remap key qua `as`. **Template Literal type** `` `prefix${T}` `` là type-level string interpolation: khi `T` là union string, nó distributive tạo tập hợp chuỗi mới; kết hợp với intrinsic `Uppercase/Lowercase/Capitalize/Uncapitalize` và `infer` để parse/biến đổi chuỗi.

## 2. Cơ chế hoạt động

### 2.1 Mapped type cơ bản

```ts
type ReadonlyT<T> = { readonly [P in keyof T]: T[P] };
type PartialT<T> = { [P in keyof T]?: T[P] };
```

Cấu trúc: `[P in keyof T]` — `P` là biến kiểu duyệt từng key, `T[P]` là indexed access. Modifier:

- `+?` thêm optional, `-?` bỏ optional, `?` giữ nguyên.
- `+readonly` thêm readonly, `-readonly` bỏ readonly.
- Không ghi `?`/`readonly` → giữ modifier gốc; ghi `?` → thêm; `-?` → bỏ.

### 2.2 Key remapping với `as` (TS 4.1+)

```ts
type Getters<T> = { [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K] };
type G = Getters<{ name: string; age: number }>; // { getName: ()=>string; getAge: ()=>number }
```

`as NewKey` remap key. Nếu `as` ra `never` thì key bị loại (filter):

```ts
type FilterStringKeys<T> = { [K in keyof T as T[K] extends string ? K : never]: T[K] };
```

`string & K` cần vì `keyof T` có thể chứa `symbol | number`, mà `Capitalize` chỉ nhận `string`.

### 2.3 Template Literal type

```ts
type EventName<T extends string> = `on${Capitalize<T>}`;
type E = EventName<"click" | "change">; // "onClick" | "onChange" — distributive
```

Khi `T` là union, template literal distributive — tương tự conditional. Kết hợp `infer` để parse:

```ts
type Params<T extends string> =
  T extends `${string}:${infer P}/${infer Rest}` ? P | Params<Rest> :
  T extends `${string}:${infer P}` ? P : never;
```

### 2.4 Intrinsic string manipulation

`Uppercase<S>`, `Lowercase<S>`, `Capitalize<S>`, `Uncapitalize<S>` — compile-time, không runtime:

```ts
type U = Uppercase<"hello">; // "HELLO"
type C = Capitalize<"hello">; // "Hello"
```

Dùng trong mapped remapping và event/style DSL.

### 2.5 Kết hợp — DSL type-safe

```ts
// Route → params
type Route = "/user/:id" | "/post/:postId/comment/:commentId";
type RouteParams<T extends string> = T extends `${string}:${infer P}/${infer R}` ? P | RouteParams<`/${R}`> : T extends `${string}:${infer P}` ? P : never;

// CSS → style object
type StyleKey = "color" | "background";
type Style = { [K in StyleKey as `style${Capitalize<K>}`]: string | number };
```

## 3. Ví dụ tối thiểu

```ts
// 3.1 Mapped cơ bản + modifier
type Mutable<T> = { -readonly [P in keyof T]: T[P] };
type RequiredT<T> = { [P in keyof T]-?: T[P] };
type Optional<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;

type User = { readonly name: string; age?: number };
type M = Mutable<User>; // { name: string; age?: number }
type R = RequiredT<User>; // { readonly name: string; age: number }

// 3.2 Key remapping — Getters / OptionsFlags
type OptionsFlags<T> = { [P in keyof T as `is${Capitalize<string & P>}`]: boolean };
type Flags = OptionsFlags<{ darkMode: string; collapsed: boolean }>; // { isDarkMode: boolean; isCollapsed: boolean }

type Getters<T> = { [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K] };
type Setters<T> = { [K in keyof T as `set${Capitalize<string & K>}`]: (v: T[K]) => void };
type Person = { name: string; age: number };
type PersonGetters = Getters<Person>; // { getName: ()=>string; getAge: ()=>number }

// 3.3 Filter key bằng never
type StringKeys<T> = { [K in keyof T as T[K] extends string ? K : never]: T[K] };
type T1 = StringKeys<{ a: string; b: number; c: string }>; // { a: string; c: string }

// 3.4 Template literal — event/style
type EventName<T extends string> = `on${Capitalize<T>}`;
type E1 = EventName<"click" | "change" | "submit">; // "onClick" | "onChange" | "onSubmit"

type Style = { [K in "color" | "background" as `style${Capitalize<K>}`]: string | number };
// { styleColor: string|number; styleBackground: string|number }

// 3.5 Parse route params với infer + template literal
type Params<T extends string> =
  T extends `${string}:${infer P}/${infer Rest}` ? P | Params<Rest> :
  T extends `${string}:${infer P}` ? P : never;
type P1 = Params<"/post/:postId/comment/:commentId">; // "postId" | "commentId"
type P2 = Params<"/user/:id">; // "id"

// 3.6 Snake → Camel (đệ quy)
type SnakeToCamel<S extends string> =
  S extends `${infer Head}_${infer Tail}` ? `${Head}${Capitalize<SnakeToCamel<Tail>>}` : S;
type SCT1 = SnakeToCamel<"hello_world_foo">; // "helloWorldFoo"
type SCT2 = SnakeToCamel<"snake_case">; // "snakeCase"

// 3.7 Deep utility kết hợp
type DeepPartial<T> = T extends object ? { [P in keyof T]?: DeepPartial<T[P]> } : T;
type DeepReadonly<T> = T extends (...args: any) => any ? T : T extends object ? { readonly [P in keyof T]: DeepReadonly<T[P]> } : T;

// 3.8 Intrinsic
type UC = Uppercase<"hello">; // "HELLO"
type LC = Lowercase<"HELLO">; // "hello"
type Cap = Capitalize<"hello">; // "Hello"
type Uncap = Uncapitalize<"Hello">; // "hello"
```

```ts
// 3.9 Thực tế React: props → data-attr
type DataAttrs<T extends string> = { [K in T as `data-${K}`]: string };
type DA = DataAttrs<"id" | "testid">; // { "data-id": string; "data-testid": string }

// 3.10 Mapped với union string
type Perms = "read" | "write" | "delete";
type PermMap = { [K in Perms as `can${Capitalize<K>}`]: boolean };
// { canRead: boolean; canWrite: boolean; canDelete: boolean }
```

## 4. So sánh / Phân loại

| Khái niệm | Cú pháp | Ví dụ |
|-----------|---------|-------|
| Mapped | `{ [P in keyof T]: T[P] }` | `Partial<T>` |
| `+?`/`-?` | `{ [P in keyof T]-?: T[P] }` | `Required<T>` |
| `±readonly` | `{ -readonly [P in keyof T]: T[P] }` | `Mutable<T>` |
| Key remapping | `{ [P in keyof T as NewKey]: ... }` | `Getters<T>` |
| Filter key | `as T[P] extends string ? P : never` | `StringKeys<T>` |
| Template literal | `` `on${Capitalize<T>}` `` | `EventName<"click">` → `"onClick"` |
| Intrinsic | `Uppercase<S>` | `Uppercase<"hi">` → `"HI"` |
| Parse với infer | `` T extends `${infer A}:${infer B}` `` | `Params<"/:id">` |

| Modifier | Ghi `?` | Ghi `-?` | Không ghi |
|----------|---------|----------|-----------|
| `?` | Thêm optional cho mọi key | Bỏ optional (Required) | Giữ nguyên |

| Template literal union | Kết quả |
|------------------------|---------|
| `` `on${"a"\|"b"}` `` | `"ona" \| "onb"` (distributive) |
| `` `${Uppercase<"hi">}` `` | `"HI"` |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không dùng mapped quá rộng trên union lớn**: `{ [P in keyof T]: ... }` với `T` là union hàng trăm key làm autocomplete chậm, tooltip dài. Hạn chế phạm vi hoặc tách nhỏ.
- **Không quên `string & K`**: `K` có thể là `symbol`, `Capitalize<symbol>` lỗi. Luôn `string & K` khi dùng intrinsic.
- **Không đệ quy template literal quá sâu**: `SnakeToCamel` với chuỗi dài dễ hit recursion limit (50 levels, TS 5.0+ giới hạn). Thêm base case và cân nhắc độ dài.
- **Không dùng template literal để thay runtime logic**: template literal chỉ ở tầng type, không sinh code runtime. Cần hàm `snakeToCamel()` runtime riêng.
- **Chi phí**: mapped + template literal lồng nhau làm type instantiation nặng, nhất là `DeepPartial` trên object sâu.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Quên `string & K`**
  - Triệu chứng: `Type 'K' does not satisfy 'string'`.
  - Fix: `` `get${Capitalize<string & K>}` ``.
  - Đo: `tsc --strict`.

- **Lỗi 2: Key remapping ra `never` không filter như ý**
  - Triệu chứng: key vẫn tồn tại với `never` value.
  - Nguyên nhân: `as never` mới loại key, còn value `never` thì key vẫn tồn tại.
  - Fix: `as T[K] extends string ? K : never` (remap key), không phải `T[K] extends string ? T[K] : never` (value).

- **Lỗi 3: Template literal không distributive như mong đợi**
  - Triệu chứng: `` `on${T}` `` với `T = string` ra `on${string}` (wide) thay vì union cụ thể.
  - Fix: đảm bảo `T extends string` là union literal, không phải `string` wide.

- **Lỗi 4: Đệ quy `SnakeToCamel` hit limit**
  - Triệu chứng: `Type instantiation is excessively deep`.
  - Fix: dùng tail recursion, hoặc giới hạn độ sâu, hoặc dùng lib `type-fest` đã tối ưu.

- **Lỗi 5: Mapped làm mất modifier gốc**
  - Triệu chứng: `Partial` làm mất `readonly` không mong muốn.
  - Fix: chỉ định `+?`/`-readonly` tường minh, hoặc dùng `+?` để giữ.

- **Công cụ**:
  - `tsc --strict --noEmit`, hover type, `Ctrl+Space`.
  - `expect-type`, `tsd` test type-level transform.
  - `tsc --extendedDiagnostics` đo instantiation.

## 7. Câu hỏi tự kiểm tra

1. Phân biệt `{ [P in keyof T]?: T[P] }` và `{ [P in keyof T]-?: T[P] }`. Khi nào cần `-?`?
2. Vì sao trong `Getters<T>` phải viết `` `get${Capitalize<string & K>}` `` thay vì `` `get${Capitalize<K>}` ``? `string & K` có tác dụng gì?
3. Viết `SnakeToCamel<S>` bằng template literal + `infer` + đệ quy. Giải thích base case và bước đệ quy.

<details>
<summary>Đáp án 30s</summary>

1. `?:` thêm optional cho mọi key (`Partial`), `-?` bỏ optional (`Required`). Cần `-?` khi muốn biến `Partial`/`optional` thành bắt buộc, ví dụ `Required<User>` bỏ `?` khỏi `age?: number`.
2. `keyof T` có thể là `string | number | symbol`, còn `Capitalize` chỉ nhận `string`. `string & K` intersection lọc ra chỉ `string` (do `string & symbol = never`), nên `Capitalize<string & K>` an toàn. Không có `& string` sẽ lỗi `Type 'K' does not satisfy 'string'`.
3. `type SnakeToCamel<S extends string> = S extends \`\${infer H}_\${infer Tail}\` ? \`\${H}\${Capitalize<SnakeToCamel<Tail>>}\` : S;` Base case: không còn `_` thì trả `S`. Đệ quy: tách `Head` trước `_` và `Tail` sau, recurse `Tail` rồi `Capitalize` và nối lại.

</details>

---
*Tham khảo chi tiết: `docs/01-javascript-typescript.md` — Câu 24. Spec: [TS Handbook — Mapped Types](https://www.typescriptlang.org/docs/handbook/2/mapped-types.html), [Template Literal Types](https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html).*
