# Generics & Constraints — Tham số kiểu, ràng buộc và suy luận kiểu

> Tags: #typescript #generics #constraints #keyof #variance | Nguồn: `docs/01-javascript-typescript.md` câu 21 | Mức: P0

## 1. Định nghĩa chính xác

**Generics** là tham số kiểu (`<T>`) cho phép viết hàm/class/type tái sử dụng mà vẫn giữ type safety — như template ở compile-time, được thay thế bằng type cụ thể khi sử dụng (instantiation). **Constraints** (`T extends U`) giới hạn `T` phải là subtype của `U`, đảm bảo `T` có shape tối thiểu để thao tác an toàn. **Default type parameter** (`T = string`) cung cấp fallback khi không suy luận được.

## 2. Cơ chế hoạt động

### 2.1 Instantiation và suy luận (inference)

Khi gọi `identity<string>("hi")` hoặc `identity("hi")` (bỏ `<string>`), checker suy luận `T` từ argument. Quy tắc:

1. Explicit: `identity<string>("hi")` → `T = string`.
2. Inference từ argument: `identity(42)` → `T = number`.
3. Inference từ return + context: `const x: string[] = identity([])` → `T = string`.
4. Nhiều vị trí: `function merge<T, U>(a: T, b: U): T & U` — mỗi param suy ra `T`, `U` riêng.

Không suy luận được → dùng default hoặc `unknown`.

### 2.2 Constraints với `extends`

`T extends { length: number }` nghĩa là `T` phải có `length: number`. Bên trong hàm, chỉ được dùng những gì constraint đảm bảo:

```ts
function getLength<T extends { length: number }>(x: T): number { return x.length; }
getLength("hi"); getLength([1,2]); // ok
// getLength(123); // ❌ number không có length
```

Constraint phổ biến:

- `T extends string` — chỉ string subtype (literal).
- `T extends keyof U` — `T` phải là key của `U`.
- `K extends string & keyof T` — key là string và thuộc `T` (lọc symbol).
- `T extends abstract new (...args: any) => any` — constructor.

### 2.3 `keyof` và indexed access `T[K]`

`keyof T` là union các key của `T`. `T[K]` là type của `T[K]`:

```ts
function getProp<T, K extends keyof T>(obj: T, key: K): T[K] { return obj[key]; }
getProp({ name: "An", age: 20 }, "name"); // string, key sai → lỗi
```

Đây là pattern type-safe property access, thay cho `obj[key as string]`.

### 2.4 Generic với class, type alias, default

```ts
class Store<T = string> { private data: T[] = []; add(item: T) {} }
type Nullable<T> = T | null;
type HasId = { id: string };
function merge<T extends HasId, U extends HasId>(a: T, b: U): T & U { return { ...a, ...b }; }
```

### 2.5 Variance (biến thể)

- **Covariant**: `T` ở vị trí output (return) — `Array<T>` covariant theo `T` (nếu `Cat extends Animal` thì `Cat[]` assign cho `Animal[]` trong TS, dù thực ra nên invariant).
- **Contravariant**: `T` ở vị trí input (param) — với `strictFunctionTypes`, `(x: Cat) => void` không assign cho `(x: Animal) => void` (ngược lại mới đúng).
- **Invariant**: vừa in vừa out — phải khớp chính xác.

```ts
type Fn<T> = (x: T) => void; // contravariant khi strictFunctionTypes
type Fn2 = (x: string | number) => void;
const f: Fn2 = (x: string) => {}; // ❌ khi strictFunctionTypes: string không đủ cho string|number
```

### 2.6 Generic constraints với `infer` (liên kết conditional)

`T extends Promise<infer U> ? U : never` — `infer` đặt trong `extends` để bắt type con ra, là cầu nối generics ↔ conditional.

## 3. Ví dụ tối thiểu

```ts
// 3.1 Cơ bản + inference
function identity<T>(x: T): T { return x; }
const s = identity("hello"); // T inferred string, s: string
const n = identity<number>(42); // explicit

// 3.2 Constraint
function getLength<T extends { length: number }>(x: T) { return x.length; }
console.log(getLength("hi")); // 2
console.log(getLength([1,2,3])); // 3
console.log(getLength({ length: 10, foo: 1 })); // 10

// 3.3 keyof constraint — type-safe get
function getProp<T, K extends keyof T>(obj: T, key: K): T[K] { return obj[key]; }
const user = { name: "An", age: 20, active: true };
const name = getProp(user, "name"); // string
// getProp(user, "email"); // ❌

// 3.4 Generic class với default
class ApiResponse<T = unknown> {
  constructor(public data: T, public status: number) {}
  map<U>(fn: (x: T) => U): ApiResponse<U> { return new ApiResponse(fn(this.data), this.status); }
}
const r = new ApiResponse({ id: 1 }, 200); // T = {id:number}
const r2 = r.map(x => x.id); // ApiResponse<number>

// 3.5 Multiple constraints + conditional
type IsArray<T> = T extends any[] ? true : false;
type A = IsArray<string[]>; // true
type B = IsArray<string>; // false

// 3.6 Factory với constructor constraint
function create<T extends abstract new (...args: any) => any>(Ctor: T): InstanceType<T> {
  return new Ctor();
}
class Foo { x = 1; }
const foo = create(Foo); // Foo

// 3.7 Generic với mapped — DeepPartial
type DeepPartial<T> = T extends object ? { [P in keyof T]?: DeepPartial<T[P]> } : T;
type User = { name: string; address: { city: string } };
type PartialUser = DeepPartial<User>; // { name?: string; address?: { city?: string } }

// 3.8 Variance ví dụ
type Handler<T> = (x: T) => void;
let hAnimal: Handler<{ kind: string }> = () => {};
let hDog: Handler<{ kind: string; bark(): void }> = () => {};
// hAnimal = hDog; // ❌ khi strictFunctionTypes — contravariant
hDog = hAnimal; // ok — Animal handler chấp nhận Dog (ít yêu cầu hơn)
```

```ts
// 3.9 Generic + overload vs generic
// Overload cho trường hợp đặc biệt:
function pick<T, K extends keyof T>(obj: T, key: K): T[K];
function pick<T, K extends keyof T>(obj: T, keys: K[]): Pick<T, K>;
function pick(obj: any, keys: any) {
  return Array.isArray(keys) ? Object.fromEntries(keys.map((k: string) => [k, obj[k]])) : obj[keys];
}
```

## 4. So sánh / Phân loại

| Khái niệm | Cú pháp | Ý nghĩa |
|-----------|---------|---------|
| Type param | `<T>` | Tham số kiểu |
| Constraint | `T extends U` | `T` phải là subtype của `U` |
| Default | `T = string` | Fallback khi không infer được |
| `keyof` | `keyof T` | Union các key của `T` |
| `T[K]` | Indexed access | Type của property `K` trên `T` |
| `infer` | `T extends Promise<infer U>` | Bắt type con |
| Variance | `strictFunctionTypes` | Kiểm soát tính tương thích hàm |

| Suy luận | Ví dụ | Kết quả |
|----------|-------|---------|
| Từ argument | `identity(42)` | `T = number` |
| Từ nhiều arg | `merge({id:"1"}, {id:"2", extra:1})` | `T`, `U` riêng |
| Từ context | `const x: string[] = identity([])` | `T = string` |
| Không suy được | `function foo<T>()` | `T = unknown` (hoặc default) |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không generic hóa sớm**: chỉ thêm `<T>` khi có ít nhất 2 use case với type khác nhau. Generic không cần thiết làm signature phức tạp, error message dài (`Type 'X' is not assignable to 'T extends ...'`).
- **Không constraint quá rộng**: `T extends any` vô nghĩa; `T extends object` quá rộng, mất gợi ý. Constraint nên vừa đủ để dùng `T` an toàn.
- **Không dùng `any` trong constraint**: `T extends any` che lỗi, nên `unknown` hoặc shape cụ thể.
- **Cẩn thận variance với function param**: `strictFunctionTypes` bật sẽ bắt lỗi contravariant — đừng tắt để che.
- **Không lạm dụng default che lỗi inference**: `T = any` làm mất an toàn, nên `unknown` hoặc bỏ default để buộc caller specify.
- **Chi phí compile**: generics lồng nhau + conditional sâu làm instantiation chậm, nhất là trong utility đệ quy.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Không suy luận được `T`**
  - Triệu chứng: `T` thành `unknown`, lỗi `Property 'x' does not exist on type 'unknown'`.
  - Fix: thêm constraint hoặc explicit `<T>`, hoặc cung cấp context type.
  - Đo: hover `T` trong VSCode, `tsc --noEmit` xem inferred.

- **Lỗi 2: Constraint thiếu `keyof`**
  - Triệu chứng: `obj[key]` lỗi `Element implicitly has an 'any' type`.
  - Fix: `K extends keyof T`, dùng `T[K]`.
  - Đo: `noImplicitAny: true` bắt.

- **Lỗi 3: Variance với `strictFunctionTypes`**
  - Triệu chứng: `Type '(x: string) => void' is not assignable to '(x: string|number) => void'`.
  - Fix: param phải chấp nhận supertype, không subtype. Sửa signature hoặc dùng `bivariantHack` (hiếm).
  - Đo: `strictFunctionTypes: true` trong `tsconfig`.

- **Lỗi 4: Generic làm error message khó đọc**
  - Triệu chứng: lỗi dài hàng chục dòng với `T extends ...`.
  - Fix: đặt tên `T` có nghĩa (`TItem`), tách type alias, dùng `extends` đơn giản.
  - Đo: `tsc --extendedDiagnostics` xem instantiation count.

- **Lỗi 5: Quên `extends` khi cần `T[K]`**
  - Triệu chứng: `Type 'K' cannot be used to index type 'T'`.
  - Fix: `K extends keyof T`.

- **Công cụ**:
  - `tsconfig`: `strict: true`, `strictFunctionTypes`, `noImplicitAny`.
  - `tsc --extendedDiagnostics`, `typescript-analyze-trace`.
  - VSCode: hover, `Go to Type Definition`, `Peek Type`.

## 7. Câu hỏi tự kiểm tra

1. Phân biệt `T extends string` (constraint) và `T = string` (default). Khi nào dùng cái nào?
2. Viết hàm `getProp<T, K extends keyof T>(obj: T, key: K): T[K]` và giải thích vì sao cần `K extends keyof T` thay vì `K: string`.
3. `strictFunctionTypes` bật thì `(x: string) => void` có assign được cho `(x: string | number) => void` không? Giải thích qua variance.

<details>
<summary>Đáp án 30s</summary>

1. `T extends string` là constraint — `T` phải là subtype của `string` (literal như `"a"|"b"`), đảm bảo `T` có behavior của `string`. `T = string` là default — khi không infer được thì `T` fallback thành `string`, không ràng buộc. Dùng `extends` để giới hạn, `= default` để cung cấp fallback.
2. `K extends keyof T` đảm bảo `K` là key hợp lệ của `T`, nên `T[K]` an toàn và `getProp(user, "email")` báo lỗi. `K: string` thì mọi string đều qua, `T[K]` lỗi `cannot index` và mất type safety.
3. Không. Param là vị trí contravariant: hàm nhận `string|number` phải xử lý cả hai, trong khi `(x: string)=>void` chỉ xử lý `string` — thiếu `number` → không an toàn. Ngược lại `(x: string|number)=>void` assign cho `(x: string)=>void` thì được (ít yêu cầu hơn có thể thay cho nhiều yêu cầu).

</details>

---
*Tham khảo chi tiết: `docs/01-javascript-typescript.md` — Câu 21. Spec: [TS Handbook — Generics](https://www.typescriptlang.org/docs/handbook/2/generics.html), [Variance](https://www.typescriptlang.org/docs/handbook/2/functions.html#variance).*
