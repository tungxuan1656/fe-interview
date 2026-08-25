# any, unknown, never, void — Bốn bottom/top type và semantics an toàn kiểu

> Tags: #typescript #any #unknown #never #void #type-system | Nguồn: `docs/01-javascript-typescript.md` câu 19 | Mức: P0

## 1. Định nghĩa chính xác

- **any**: top type không an toàn — tương thích hai chiều với mọi type (`any` assign được cho `T` và `T` assign được cho `any`), tắt toàn bộ type checking tại vị trí sử dụng. Là escape hatch, làm mất soundness.
- **unknown**: top type an toàn — mọi giá trị đều assign được cho `unknown`, nhưng `unknown` không assign được cho type cụ thể nếu chưa **narrowing**. Là `any` an toàn, buộc kiểm tra trước khi dùng.
- **never**: bottom type — tập rỗng, không có giá trị nào thuộc `never`. Dùng cho hàm không bao giờ return (throw/infinite loop) và để biểu diễn nhánh không thể xảy ra trong exhaustive check. `never` assign được cho mọi type, nhưng không type nào (trừ `never` và `any`) assign được cho `never`.
- **void**: type của giá trị không có ý nghĩa — hàm `void` có thể không return hoặc `return undefined`. `undefined` là subtype của `void` (khi `strictNullChecks` bật, `void` vẫn chấp nhận `undefined` nhưng không chấp nhận `null`).

> Phân biệt cốt lõi: `any` bỏ qua kiểm tra, `unknown` buộc kiểm tra, `never` biểu diễn "không bao giờ xảy ra", `void` biểu diễn "không quan tâm giá trị trả về".

## 2. Cơ chế hoạt động

### 2.1 Quan hệ assignability (với `strict: true`)

```
any  <-> T        (hai chiều, unsafe)
unknown  <- T     (mọi T assign vào unknown được)
T  <- unknown     (chỉ khi T là unknown/any, ngược lại lỗi)
never -> T        (never assign cho T được)
T -> never        (chỉ never/any assign vào never được)
void <- undefined (và <- never)
```

- **any**: compiler bỏ qua constraint. `let x: any = 1; x.foo.bar()` không lỗi — property access, call đều cho phép.
- **unknown**: mọi thao tác (property access, call, operator) đều lỗi cho tới khi narrow bằng `typeof`, `instanceof`, `in`, `===`, type guard `is`, hoặc assertion `asserts`.
- **never**: xuất hiện tự nhiên khi:
  - Hàm `throw` hoặc loop vô hạn không có đường return.
  - Giao của các type xung khắc: `string & number` → `never`.
  - Nhánh `else` sau khi đã cover hết union — biến còn lại là `never`.
  - Conditional type filter: `T extends U ? never : T` loại bỏ nhánh.
- **void** vs **undefined**:
  - `(): void` — caller không được dựa vào giá trị trả về. `() => void` cho phép callback `return` bất kỳ giá trị nào mà caller sẽ bỏ qua (ví dụ `arr.forEach((x): void => 1)` hợp lệ).
  - `(): undefined` — hàm phải trả về `undefined` tường minh, nghiêm ngặt hơn.

### 2.2 Exhaustive check với `never`

```ts
type Shape = { kind: "circle"; r: number } | { kind: "square"; size: number };
function assertNever(x: never): never { throw new Error(`Unhandled: ${JSON.stringify(x)}`); }
function area(s: Shape) {
  switch (s.kind) {
    case "circle": return Math.PI * s.r ** 2;
    case "square": return s.size ** 2;
    default: return assertNever(s); // nếu thêm kind mới mà quên case → s không còn là never → lỗi compile
  }
}
```
Compiler suy luận: sau khi loại trừ `circle` và `square`, type còn lại là `never`. Nếu union mở rộng, `s` sẽ không còn `never` → lỗi.

### 2.3 `never` trong conditional type

Khi conditional type distribute trên union, nhánh `never` bị loại khỏi union (do `never` là identity của union): `string | never` → `string`. Đây là cơ chế của `Exclude`/`Extract`.

## 3. Ví dụ tối thiểu

```ts
// 3.1 any — mất an toàn
let a: any = { id: 1 };
a.foo.bar.baz(); // không lỗi compile, crash runtime
a = "string"; a = 123; // đều ok

// 3.2 unknown — buộc narrow
let u: unknown = JSON.parse('{"name":"An"}');
if (typeof u === "object" && u !== null && "name" in u) {
  console.log((u as { name: string }).name); // ok sau khi narrow
}
// u.toString(); // ❌ lỗi: Object is of type 'unknown'

// unknown với type guard tái sử dụng
function isString(x: unknown): x is string { return typeof x === "string"; }
function upper(x: unknown) {
  if (isString(x)) return x.toUpperCase(); // x narrowed thành string
  return String(x);
}

// 3.3 never — hàm không return + exhaustive
function fail(msg: string): never { throw new Error(msg); }
function infinite(): never { while (true) {} }

type Result = "ok" | "error";
function handle(r: Result) {
  if (r === "ok") return 1;
  if (r === "error") return 0;
  // r ở đây là never — nếu thêm giá trị mới cho Result mà quên handle, lỗi ở dòng dưới
  const _exhaustive: never = r;
  return _exhaustive;
}

// 3.4 void vs undefined
function log(msg: string): void { console.log(msg); }
const v1: void = undefined; // ok
// const v2: void = null; // ❌ khi strictNullChecks

type Cb = () => void;
const cb: Cb = () => 42; // ok — caller bỏ qua return value
// Ngược lại với (): undefined thì phải return undefined:
type StrictCb = () => undefined;
const scb: StrictCb = () => undefined; // phải tường minh

// 3.5 unknown trong catch (TS 4.4+ mặc định unknown)
try { JSON.parse("bad"); } catch (e: unknown) {
  if (e instanceof Error) console.log(e.message);
}
```

```ts
// 3.6 Phân biệt void trong generic context
function forEach<T>(arr: T[], fn: (x: T) => void) { arr.forEach(fn); }
forEach([1,2,3], x => x * 2); // không lỗi dù callback return number — void bỏ qua

// 3.7 never từ intersection xung khắc
type Impossible = string & number; // never
type Filtered = "a" | "b" | "c" extends infer T ? T extends "a" ? never : T : never; // "b" | "c"
```

## 4. So sánh / Phân loại

| Tiêu chí | `any` | `unknown` | `never` | `void` |
|----------|-------|-----------|---------|--------|
| Có giá trị runtime | Có (bất kỳ) | Có (bất kỳ, chưa biết) | Không có giá trị nào | `undefined` (không quan tâm giá trị) |
| Assign vào `any/unknown/never/void` | `any` → mọi `T` | `unknown` → chỉ `unknown/any` | `never` → mọi `T` | `void` → `void/undefined/any` |
| Thao tác (`.prop`, `()`) | Cho phép hết | Cấm tới khi narrow | Không thể tạo giá trị để thao tác | Bỏ qua giá trị trả về |
| Dùng khi nào | Escape hatch, migration | Nhận data ngoài (API, JSON) | Exhaustive check, filter type, hàm throw | Callback/hàm không trả gì |
| An toàn | Không | Có | Có (bắt missing case) | Có |

| Tình huống | Chọn gì |
|------------|---------|
| Data từ `fetch`/`JSON.parse`/`catch` | `unknown` + narrow |
| Hàm throw hoặc loop vô hạn | `never` |
| Callback `onClick`, `forEach` | `() => void` |
| Giá trị phải là `undefined` tường minh | `undefined` |
| Tạm thời bypass trong migration | `any` (đánh dấu `// TODO: remove any`) |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không dùng `any`** nếu có thể dùng `unknown` + guard. `any` lan truyền: một biến `any` làm mọi biểu thức liên quan thành `any`, vô hiệu hóa kiểm tra toàn nhánh. Nếu bắt buộc dùng `any` (interop JS), cô lập ở boundary và bọc bằng guard.
- **Không dùng `unknown` như `any` với `as` bừa bãi**: `as string` sau `unknown` mà không narrow thực chất là `any` trá hình. Dùng type guard/assertion function thay vì cast.
- **Không nhầm `void` với `undefined`/`never`**: `void` không có nghĩa "không có giá viên" — nó có nghĩa "caller không dùng giá trị trả về". Dùng `undefined` khi cần giá trị cụ thể, `never` khi hàm không bao giờ trả về.
- **Không return `never` giả**: hàm khai `never` nhưng thực tế có đường `return undefined` sẽ lỗi. Chỉ khai `never` khi chắc chắn throw/loop.
- **Chi phí**: `unknown` yêu cầu viết guard nhiều hơn, nhưng đó là chi phí đúng — đổi lại bắt 30-40% bug `undefined`/`null` và type mismatch.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Lạm dụng `any` làm mất kiểm tra**
  - Triệu chứng: `noImplicitAny` không bắt, nhưng lỗi runtime `Cannot read properties of undefined`.
  - Fix: bật `noImplicitAny: true`, ESLint `@typescript-eslint/no-explicit-any: "warn"`, thay `any` bằng `unknown` + guard.
  - Đo: `npx type-coverage --detail --strict` — tỉ lệ `any` nên < 2%; `tsc --noEmit --strict` + `eslint`.

- **Lỗi 2: Nhầm `void` trong Promise**
  - Triệu chứng: `async function foo(): Promise<void> { return "hi"; }` — với `void` thì không lỗi? Thật ra `Promise<void>` cho phép `return` giá trị nhưng caller bỏ qua, dễ gây hiểu lầm.
  - Fix: nếu muốn cấm return giá trị, dùng `Promise<undefined>` hoặc không `return`.
  - Đo: review signature `Promise<void>` trong API.

- **Lỗi 3: Exhaustive check không hoạt động**
  - Triệu chứng: thêm variant mới cho union nhưng không báo lỗi.
  - Nguyên nhân: quên `default: assertNever(s)` hoặc dùng `if/else` không cover hết mà không có `never` check.
  - Fix: luôn có `default` với `assertNever`, hoặc gán `const _exhaustive: never = s`.
  - Đo: `tsc --strict` sẽ báo `Type 'NewVariant' is not assignable to type 'never'`.

- **Lỗi 4: `unknown` trong `catch` bị `as` sai**
  - Triệu chứng: `catch (e) { (e as Error).message }` crash khi `throw "string"`.
  - Fix: `if (e instanceof Error) e.message else String(e)`.
  - Đo: `useUnknownInCatchVariables: true` (mặc định từ TS 4.4).

- **Công cụ**:
  - `tsconfig`: `strict: true`, `noImplicitAny`, `useUnknownInCatchVariables`.
  - ESLint: `@typescript-eslint/no-explicit-any`, `no-unsafe-assignment/call/member-access/return`.
  - `type-coverage` đo tỉ lệ `any`; `tsc --noEmit` đo lỗi.

## 7. Câu hỏi tự kiểm tra

1. Vì sao `unknown` được gọi là `any` an toàn? Cho ví dụ `unknown` bắt lỗi mà `any` bỏ qua.
2. Phân biệt `void` và `never` trong kiểu trả về của hàm. Khi nào `() => void` lại chấp nhận `return 42` mà `() => never` thì không?
3. `never` được dùng thế nào để làm exhaustive check cho discriminated union? Viết hàm `assertNever` và giải thích vì sao thêm variant mới sẽ gây lỗi compile.

<details>
<summary>Đáp án 30s</summary>

1. `any` hai chiều, cho phép mọi thao tác không kiểm tra (`a.foo()` không lỗi); `unknown` chỉ cho phép sau khi narrow (`if (typeof x==="string") x.toUpperCase()`). Nhận `JSON.parse` vào `any` rồi gọi `.toUpperCase()` sẽ crash runtime không báo trước, còn `unknown` buộc `typeof` trước nên an toàn.
2. `void` = hàm có thể return `undefined` hoặc không return, caller bỏ qua giá trị — nên `(): void => 42` hợp lệ (giá trị bị ignore). `never` = hàm không bao giờ trả về (throw/loop), không có giá trị nào thỏa — nên không thể `return 42`. `Promise<void>` cũng bỏ qua giá trị resolve.
3. Sau `switch` cover hết `kind`, biến còn lại suy ra `never`. Đặt `default: assertNever(s)` với `function assertNever(x: never): never`. Khi thêm `kind` mới, `s` trong `default` không còn `never` mà là variant mới → `Type 'NewKind' is not assignable to 'never'` → bắt thiếu case ngay lúc compile.

</details>

---
*Tham khảo chi tiết: `docs/01-javascript-typescript.md` — Câu 19. Spec: [TS Handbook — Basic Types](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html), [never](https://www.typescriptlang.org/docs/handbook/2/functions.html#never), [unknown](https://www.typescriptlang.org/docs/handbook/2/functions.html#unknown).*
