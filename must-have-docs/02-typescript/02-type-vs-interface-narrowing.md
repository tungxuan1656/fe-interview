# type vs interface & Type Narrowing — Bí danh kiểu, hợp nhất khai báo và thu hẹp kiểu

> Tags: #typescript #type #interface #narrowing #type-guard #discriminated-union | Nguồn: `docs/01-javascript-typescript.md` câu 20, 28, 26 | Mức: P0

## 1. Định nghĩa chính xác

- **type alias** (`type T = ...`): đặt tên cho bất kỳ type nào — primitive, union, intersection, tuple, conditional, mapped, template literal. Không tạo thực thể runtime, chỉ là bí danh compile-time, **không hỗ trợ declaration merging** (trùng tên báo lỗi).
- **interface** (`interface I {}`): khai báo shape của object/class, hỗ trợ **declaration merging** (nhiều `interface I` cùng tên tự merge), `extends` kế thừa, và `implements` bởi class. Được cache tốt hơn trong hệ type checker với union lớn.
- **Type Narrowing**: quá trình thu hẹp type từ rộng sang hẹp dựa trên control flow (`typeof`, `instanceof`, `in`, `===`, `switch` trên discriminant, truthiness). Type guard (`param is Type`) và assertion function (`asserts param is Type`) là cơ chế tường minh để nói cho checker biết cách narrow.
- **Discriminated union**: union của các object có chung trường discriminant (`kind`/`type`) với literal type khác nhau — cho phép narrowing chính xác nhất qua `switch`.

## 2. Cơ chế hoạt động

### 2.1 type vs interface — khác năng lực

| Năng lực | `type` | `interface` |
|----------|--------|-------------|
| Alias cho mọi type | Có (`string \| number`, `keyof T`, `T extends U ? X : Y`) | Không — chỉ object shape |
| Union/Intersection | `type A = B \| C`, `A & B` | Không trực tiếp (phải `type X = I & J`) |
| Declaration merging | Không (duplicate identifier) | Có — nhiều `interface User` merge thành một |
| `extends` / `implements` | `T & U` (intersection) | `interface B extends A`, `class C implements I` |
| `keyof` / mapped / conditional | Có | Hạn chế |
| Performance với union lớn | Có thể chậm (big intersection) | Được intern/cache tốt hơn |

Khuyến nghị: `interface` cho object shape public/library (tận dụng merging/augmentation), `type` cho union/utility/transform.

### 2.2 Declaration merging

```ts
interface User { name: string; }
interface User { age: number; }
// → User = { name: string; age: number; }

type UserT = { name: string };
// type UserT = { age: number }; // ❌ Duplicate identifier
```

Merging cũng áp dụng cho `namespace` + `class`/`function`/`enum` — nền tảng của module augmentation.

### 2.3 Narrowing — các nhánh

1. **typeof guard**: `typeof x === "string"` → `x: string` trong nhánh true, `x: Exclude<typeof x, string>` trong nhánh else.
2. **instanceof**: `x instanceof Date` → `Date`.
3. **`in` operator**: `"a" in x` → `x` có property `a`.
4. **Equality narrowing**: `x === null`, `x === "a"` → literal.
5. **Discriminant narrowing**: `switch (s.kind)` — chính xác nhất.
6. **Truthiness**: `if (x)` loại `null|undefined|0|""|false`.
7. **Custom type guard**: `function isCat(p: Pet): p is Cat` — return `boolean` nhưng annotate `is`.
8. **Assertion function**: `function assertIsString(x: unknown): asserts x is string` — nếu không throw thì narrow sau lời gọi.

Control flow analysis: checker theo dõi mọi nhánh, gán type hẹp nhất cho biến tại mỗi điểm. Sau `if (isCat(p))`, `p` là `Cat` trong then, `Dog` trong else.

### 2.4 `is` vs `asserts`

- `is`: trả `boolean`, dùng trong `if (isCat(p))`. Không throw.
- `asserts`: `void` nhưng nếu return thì param đã là `Type`, nếu không thỏa thì throw. Dùng khi muốn fail-fast: `assertIsString(x); x.toUpperCase()`.

### 2.5 Enum vs Union vs `as const`

Liên quan trực tiếp tới chọn `type` hay `interface` cho tập giá trị hữu hạn:

| Tiêu chí | `enum` | Union `type` | `as const` object |
|----------|--------|--------------|-------------------|
| Runtime | Có (object + reverse mapping với number enum) | Không (chỉ type) | Có (object readonly) |
| Tree-shakable | Kém | Hoàn toàn | Tốt |
| `const enum` | Inline, mất khi `isolatedModules` | — | — |
| Khuyến nghị | Tránh (trừ khi cần reverse mapping) | Mặc định | Khi cần cả runtime + type |

## 3. Ví dụ tối thiểu

```ts
// 3.1 type vs interface
type StringOrNumber = string | number; // interface không làm được
type Nullable<T> = T | null;
type Keys = keyof { a: 1; b: 2 }; // "a" | "b"

interface A { x: number; }
interface B extends A { y: number; } // extends
type C = A & { y: number }; // intersection tương đương type

// interface merging — augment Window
interface Window { myApp: { version: string }; }
// window.myApp.version ok

// 3.2 Narrowing cơ bản
function padLeft(x: string | number, n: number) {
  if (typeof x === "string") return " ".repeat(n) + x; // x: string
  return " ".repeat(n) + x.toString(); // x: number
}

// 3.3 Discriminated union — pattern mạnh nhất
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rect"; w: number; h: number }
  | { kind: "square"; size: number };

function area(s: Shape) {
  switch (s.kind) {
    case "circle": return Math.PI * s.radius ** 2;
    case "rect": return s.w * s.h;
    case "square": return s.size ** 2;
    default: { const _ex: never = s; throw new Error(`Unhandled ${s}`); }
  }
}

// 3.4 Custom type guard + filter
type Cat = { meow(): void };
type Dog = { bark(): void };
function isCat(p: Cat | Dog): p is Cat { return (p as Cat).meow !== undefined; }
function speak(p: Cat | Dog) { if (isCat(p)) p.meow(); else p.bark(); }

// Guard cho Array.filter — loại null
const mixed: (string | null)[] = ["a", null, "b"];
const filtered: string[] = mixed.filter((x): x is string => x !== null);

// 3.5 Assertion function
function assertIsString(x: unknown): asserts x is string {
  if (typeof x !== "string") throw new Error("Not a string");
}
function foo(x: unknown) {
  assertIsString(x);
  x.toUpperCase(); // ok
}

// 3.6 in operator + truthiness
type AObj = { type: "a"; a: string };
type BObj = { type: "b"; b: number };
function handle(x: AObj | BObj) {
  if ("a" in x) console.log(x.a); // AObj
  else console.log(x.b); // BObj
}

// 3.7 as const vs enum vs union
const StatusConst = { Pending: "pending", Success: "success", Error: "error" } as const;
type Status = typeof StatusConst[keyof typeof StatusConst]; // "pending" | "success" | "error"
// Runtime: StatusConst.Pending, Type: Status — best of both worlds
```

```ts
// 3.8 Narrowing với unknown (liên kết file 01)
function parseJson(s: string): unknown { return JSON.parse(s); }
const data = parseJson('{"id":1}');
if (typeof data === "object" && data !== null && "id" in data) {
  console.log((data as { id: number }).id);
}
```

## 4. So sánh / Phân loại

| Tiêu chí | `type` | `interface` |
|----------|--------|-------------|
| Dùng cho | mọi type, utility, conditional | object shape, class contract, public API |
| Union/Intersection | `\|`, `&` trực tiếp | Phải qua `type X = I & J` |
| Merging | Không | Có |
| `implements` | `type` cũng `implements` được (TS 2.7+) nhưng `interface` rõ hơn | `class implements I` |
| Khả năng transform | Mạnh (mapped, conditional, infer) | Hạn chế |
| Hiệu năng checker | Intersection lớn có thể chậm | Cache tốt hơn |

| Narrowing | Cú pháp | Khi dùng |
|-----------|---------|----------|
| `typeof` | `typeof x === "string"` | primitive union |
| `instanceof` | `x instanceof Date` | class |
| `in` | `"prop" in x` | object union |
| Discriminant | `switch (x.kind)` | union có literal chung |
| Guard `is` | `isCat(x)` | logic phức tạp |
| `asserts` | `assertIsString(x)` | fail-fast, không cần `if` |

| Tập giá trị hữu hạn | Chọn |
|---------------------|------|
| Chỉ cần type | `type Status = "a" \| "b"` |
| Cần cả runtime + type | `as const` object |
| Cần reverse mapping / legacy | `enum` (cân nhắc `const enum`) |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không dùng `interface` cho union/conditional**: `interface` không biểu diễn `string | number` hay `T extends U ? X : Y`. Cố ép sẽ phải tạo `type` trung gian.
- **Không dùng `type` intersection khổng lồ**: `type Big = A & B & C & ...` với hàng chục field làm error message khó đọc và checker chậm. Thay bằng `interface extends`.
- **Không lạm dụng declaration merging**: merging là global, dễ gây conflict khi lib augment cùng tên. Đặt augmentation riêng trong `types/` và document.
- **Không viết guard trả `boolean` thay vì `is`**: `function isCat(p): boolean` không giúp narrow — phải `p is Cat`.
- **Không dùng `as` thay narrow**: `x as string` bỏ qua kiểm tra, mất lợi ích của narrowing. Dùng guard/assertion.
- **Không dùng truthiness khi `0`/`""` là hợp lệ**: `if (x)` loại cả `0` và `""`. Dùng `x != null` hoặc `x !== ""` tường minh.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Guard không narrow**
  - Triệu chứng: `if (isCat(p)) p.meow()` vẫn lỗi `Property 'meow' does not exist on type 'Dog'`.
  - Nguyên nhân: `isCat` khai `boolean` thay vì `p is Cat`.
  - Fix: `function isCat(p: Cat|Dog): p is Cat { ... }`.
  - Đo: `tsc --strict` báo, hover type trong VSCode xem `p` chưa narrow.

- **Lỗi 2: `as` cast che lỗi**
  - Triệu chứng: runtime crash dù compile pass.
  - Fix: thay `as` bằng guard; bật ESLint `@typescript-eslint/consistent-type-assertions` và `no-unsafe-assignment`.
  - Đo: `grep -rn " as " src | wc -l` — số lượng cast nên thấp.

- **Lỗi 3: Discriminated union thiếu `default` exhaustive**
  - Triệu chứng: thêm variant mới không báo lỗi.
  - Fix: `default: assertNever(x)` với `never`.
  - Đo: `tsc --strict` bắt.

- **Lỗi 4: `in` narrowing sai khi prototype**
  - Triệu chứng: `"toString" in x` luôn true do prototype.
  - Fix: dùng discriminant riêng (`kind`) hoặc `Object.hasOwn(x, "prop")` (ES2022).

- **Lỗi 5: Enum numeric cho phép giá trị ngoài tập**
  - Triệu chứng: `enum E { A, B } ; const e: E = 999;` không lỗi.
  - Fix: dùng union `type E = "A" | "B"` hoặc `as const`.
  - Đo: ESLint `no-magic-numbers`, review enum usage.

- **Công cụ**:
  - `tsc --strict --noEmit` — kiểm tra narrowing.
  - ESLint: `@typescript-eslint/no-explicit-any`, `consistent-type-assertions`, `strict-boolean-expressions`.
  - Dev: hover type, `Ctrl+Space` xem discriminant, `Go to Type Definition`.

## 7. Câu hỏi tự kiểm tra

1. Khi nào nên dùng `interface` và khi nào bắt buộc dùng `type`? Cho ví dụ `interface` không làm được.
2. Phân biệt `function isCat(p): boolean` và `function isCat(p): p is Cat`. Vì sao chỉ cái thứ hai mới giúp narrowing trong `if`?
3. So sánh `asserts x is string` với `x is string`. Khi nào dùng assertion function thay vì type guard thông thường?

<details>
<summary>Đáp án 30s</summary>

1. `interface` cho object shape public, cần `extends`/`implements`/merging/augmentation. `type` bắt buộc khi cần union (`string|number`), intersection, tuple, conditional (`T extends U ? X : Y`), mapped, `keyof`, template literal — những thứ `interface` không biểu diễn được.
2. `boolean` chỉ trả true/false, checker không biết true tương ứng với `Cat`. `p is Cat` là type predicate — nói với checker "nếu return true thì `p` là `Cat`", nên trong `if (isCat(p))` thì `p` được narrow thành `Cat` ở then và `Dog` ở else.
3. `is` trả boolean, dùng trong `if` để rẽ nhánh. `asserts` không trả boolean mà throw nếu sai, nên sau `assertIsString(x)` thì `x` là `string` ở mọi dòng tiếp theo (không cần `if`). Dùng `asserts` khi muốn fail-fast thay vì rẽ nhánh.

</details>

---
*Tham khảo chi tiết: `docs/01-javascript-typescript.md` — Câu 20, 28, 26. Spec: [TS Handbook — Objects vs Intersections](https://www.typescriptlang.org/docs/handbook/2/objects.html), [Narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html), [Handling `unknown`](https://www.typescriptlang.org/docs/handbook/2/functions.html#unknown).*
