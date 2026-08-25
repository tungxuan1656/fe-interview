# Strict Mode, Decorator, Module Augmentation & Project References — Cấu hình nghiêm ngặt, siêu lập trình và mở rộng kiểu

> Tags: #typescript #strict-mode #decorator #metadata #declaration-merging #module-augmentation #project-references | Nguồn: `docs/01-javascript-typescript.md` câu 25, 27, 29, 30 | Mức: P1

## 1. Định nghĩa chính xác

- **Strict Mode (`strict: true`)**: bật toàn bộ strict flags (`strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`, `strictBindCallApply`, `noImplicitThis`, `alwaysStrict`, `useUnknownInCatchVariables`...), biến TS thành sound hơn — null/undefined phải explicit, `any` ngầm bị cấm, function variance đúng.
- **Decorator**: hàm intercept khai báo `class`/`method`/`property`/`parameter` qua cú pháp `@decorator`, nhận `target`/`key`/`descriptor` (legacy) hoặc `value`/`context` (TC39 Stage 3). Cần `experimentalDecorators` + `emitDecoratorMetadata` để emit metadata.
- **Declaration Merging**: TS merge nhiều declaration cùng tên (interface+interface, class+namespace, function+namespace). Nền tảng của augmentation.
- **Module Augmentation**: dùng `declare module "lib" { interface X {} }` để mở rộng type của module bên ngoài mà không sửa `node_modules`.
- **Project References** (`composite` + `references`): chia monorepo thành nhiều `tsconfig` con, `tsc -b` build incremental chỉ check file đổi, cache `.tsbuildinfo`.

## 2. Cơ chế hoạt động

### 2.1 Strict flags — quan trọng nhất `strictNullChecks`

| Flag | Tác dụng |
|------|----------|
| `strictNullChecks` | `null`/`undefined` không còn subtype của mọi type; `string` ≠ `string\|null`. Bắt 30-40% bug. |
| `noImplicitAny` | Cấm `any` suy ra ngầm (param không type, `noImplicitAny` bắt). |
| `strictFunctionTypes` | Param của hàm check **contravariant** đúng (thay vì bivariant). |
| `strictBindCallApply` | `fn.call`/`apply`/`bind` check args đúng. |
| `noImplicitThis` | `this` không type → lỗi. |
| `alwaysStrict` | Emit `"use strict"` trong JS. |
| `useUnknownInCatchVariables` | `catch (e)` mặc định `unknown` thay vì `any`. |

Khi tắt `strictNullChecks`:

```ts
let s: string = null; // không lỗi — bug runtime
s.toUpperCase(); // crash
```

Khi bật:

```ts
let s: string | null = null;
s.toUpperCase(); // ❌ Object is possibly 'null' — phải narrow: s?.toUpperCase() hoặc if (s)
```

### 2.2 Decorator — 2 thế hệ

**Legacy (TS `experimentalDecorators: true`)** — NestJS/Angular hiện tại:

```ts
function Log(target: any, key: string, descriptor: PropertyDescriptor) {
  const orig = descriptor.value;
  descriptor.value = function(...args: any[]) { console.log(key, args); return orig.apply(this, args); };
}
class S { @Log save(d: string) {} } // descriptor.value bị wrap
```

Thứ tự: `@f @g class C` → `f(g(C))`. Class decorator nhận `constructor`, method decorator nhận `target, key, descriptor`, property/param decorator nhận `target, key, index`.

**TC39 new (TS 5+ với `experimentalDecorators: false`)**:

```ts
function logged(value: any, context: ClassMethodDecoratorContext) {
  if (context.kind === "method") return function(...args: any[]) { /* ... */ return value.apply(this, args); };
}
```

`reflect-metadata` + `emitDecoratorMetadata` emit `design:type`, `design:paramtypes`, `design:returntype` để DI container (NestJS) đọc type runtime:

```ts
import "reflect-metadata";
function Required(target: any, key: string) {
  const type = Reflect.getMetadata("design:type", target, key); // String, Number...
}
```

### 2.3 Declaration Merging

Các cặp merge được:

- `interface` + `interface` → hợp property.
- `class` + `namespace` / `function` + `namespace` / `enum` + `namespace` → thêm static.
- Không merge `type` + `type` (lỗi duplicate).

### 2.4 Module Augmentation

```ts
// types/express.d.ts — phải là module (có import/export)
import "express";
declare module "express-serve-static-core" {
  interface Request { user?: { id: string; role: string }; }
}
// global augmentation
declare global { interface Window { myApp: { version: string }; } }
// ambient cho JS lib
declare module "my-js-lib" { export function doThing(x: string): number; }
```

Yêu cầu: `declare module "xxx"` khớp specifier, file chứa `import`/`export` để thành module (nếu không sẽ là script global và không augment).

### 2.5 Project References & Monorepo perf

```
tsconfig.json (root) → references: [{ path: "./packages/app" }, { path: "./packages/ui" }]
packages/app/tsconfig.json → composite: true, incremental: true, declaration: true
packages/ui/tsconfig.json  → composite: true
```

- `composite: true` bắt buộc cho referenced project — bật `incremental`, `declaration`, `strict`... và chỉ cho phép `outDir`/`rootDir`.
- `tsc -b` build theo DAG, cache `.tsbuildinfo`, chỉ rebuild project đổi.
- `skipLibCheck: true` bỏ check `.d.ts` trong `node_modules` — nhanh hơn, an toàn vì lib đã check.
- `isolatedModules: true` để tương thích `esbuild`/`swc` transpile đơn file.

## 3. Ví dụ tối thiểu

```ts
// 3.1 strictNullChecks
let s: string | null = null;
// s.toUpperCase(); // ❌
if (s !== null) s.toUpperCase(); // ok
s?.toUpperCase(); // ok
const v = s ?? "default"; // ok

// 3.2 noImplicitAny + strictFunctionTypes
// function foo(x) {} // ❌ Parameter 'x' implicitly has an 'any' type
type Fn = (x: string | number) => void;
const f: Fn = (x: string) => {}; // ❌ khi strictFunctionTypes — contravariant

// 3.3 Legacy decorator
function Log(target: any, key: string, descriptor: PropertyDescriptor) {
  const orig = descriptor.value;
  descriptor.value = function(...args: any[]) {
    console.log(`Call ${key}`, args);
    return orig.apply(this, args);
  };
}
function Injectable(target: Function) {
  Reflect.defineMetadata("injectable", true, target);
}
class Service {
  @Log
  save(data: string) { console.log("saving", data); }
}

// 3.4 Metadata với reflect-metadata
import "reflect-metadata";
function RequiredMeta(target: any, key: string) {
  const type = Reflect.getMetadata("design:type", target, key);
  console.log(`Property ${key} type:`, type?.name);
}
class User {
  @RequiredMeta
  name!: string;
}

// 3.5 Declaration merging
interface UserM { name: string; }
interface UserM { age: number; }
const u: UserM = { name: "An", age: 20 }; // merge

class Album { label: string = ""; }
namespace Album { export class Label { title = ""; } }
const lab = new Album.Label();

// 3.6 Module augmentation — Express Request
// File: types/express.d.ts
// import "express";
// declare module "express-serve-static-core" {
//   interface Request { user?: { id: string; role: string }; }
// }
// Sử dụng: app.get("/", (req, res) => { req.user?.id; });

// Augment Window
// File: types/global.d.ts
// export {};
// declare global { interface Window { myApp: { version: string }; } }
// window.myApp.version

// Augment MUI Theme
// declare module "@mui/material/styles" {
//   interface Palette { neutral: Palette["primary"]; }
// }
```

```json
// 3.7 Project References — tsconfig
// tsconfig.json (root)
{
  "compilerOptions": { "strict": true, "skipLibCheck": true, "incremental": true, "composite": true, "declaration": true, "declarationMap": true, "isolatedModules": true },
  "references": [{ "path": "./packages/app" }, { "path": "./packages/ui" }]
}
// packages/app/tsconfig.json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": { "composite": true, "outDir": "dist", "tsBuildInfoFile": "./.tsbuildinfo" },
  "references": [{ "path": "../ui" }]
}
```

```bash
# 3.8 Lệnh build & chẩn đoán
tsc -b --verbose          # build references theo DAG
tsc --noEmit --incremental # check only, cache
tsc --extendedDiagnostics  # xem time per phase
tsc --generateTrace trace && npx analyze-trace trace # trace file chậm
npx ts-prune               # tìm export không dùng
```

## 4. So sánh / Phân loại

| Strict flag | Tắt | Bật (`strict: true`) |
|-------------|-----|----------------------|
| `strictNullChecks` | `string` nhận `null` | `string \| null` mới nhận `null`, `?.`/`??`/`if` để narrow |
| `noImplicitAny` | `function foo(x)` ok | `function foo(x: any)` mới ok |
| `strictFunctionTypes` | bivariant (lỏng) | contravariant đúng |
| `strictBindCallApply` | `fn.call(null, 123)` không check | check args |

| Decorator thế hệ | Flag | API | Dùng |
|------------------|------|-----|------|
| Legacy | `experimentalDecorators: true` | `target, key, descriptor` | NestJS, Angular, TypeORM hiện tại |
| TC39 Stage 3 | `experimentalDecorators: false` (TS 5+) | `value, context` | Tương lai, ES native |

| Merging | Hợp lệ | Không hợp lệ |
|---------|--------|--------------|
| `interface + interface` | Có | — |
| `class + namespace` | Có | — |
| `type + type` | — | Duplicate identifier |
| `enum + namespace` | Có | — |

| Augmentation | Cú pháp | Phạm vi |
|--------------|---------|---------|
| Global | `declare global { interface Window {} }` | Toàn project |
| Module | `declare module "lib" { interface X {} }` | Module `"lib"` |
| Ambient | `declare module "js-lib" { export function f() }` | Cho JS không có type |

| Tool monorepo perf | Tác dụng |
|--------------------|----------|
| `composite` + `references` | Incremental theo project |
| `skipLibCheck` | Bỏ check `.d.ts` lib |
| `isolatedModules` + `esbuild/swc` | Transpile nhanh, `tsc --noEmit` chỉ check |
| Barrel `index.ts` re-export | Tránh — làm check chậm, thay bằng import trực tiếp |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không tắt `strict` để "cho nhanh"**: tắt `strictNullChecks` che 30-40% bug null/undefined. Chấp nhận thêm `| null`, `!` có kiểm soát thay vì tắt.
- **Không dùng `!` (non-null assertion) vô tội vạ**: `s!.toUpperCase()` silence lỗi nhưng crash nếu `s` thực sự `null`. Chỉ dùng khi đã chứng minh không null (ví dụ sau `if` hoặc `Map.get` với `has` trước).
- **Không lạm dụng decorator cho business logic**: decorator là magic, khó trace, thứ tự `@f @g` dễ nhầm (`f(g(C))`). Chỉ dùng khi framework yêu cầu (DI, validation).
- **Không augment bừa để che bug lib**: augmentation là global, ảnh hưởng toàn project. Nếu lib thiếu type, ưu tiên `@types/` hoặc PR, chỉ augment khi cần patch tạm.
- **Không dùng `const enum` với `isolatedModules`**: `const enum` bị inline và xóa, cross-file với `isolatedModules`/`esbuild` sẽ lỗi. Dùng union `type` + `as const`.
- **Không bật `composite` sai**: referenced project phải `composite: true` và có `outDir`/`rootDir`; quên sẽ lỗi `File is not under 'rootDir'`.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: `!` assertion che null**
  - Triệu chứng: `user!.name` crash khi `user` là `null`.
  - Fix: `if (user) user.name` hoặc `user?.name`; bật ESLint `@typescript-eslint/no-non-null-assertion: "warn"`.
  - Đo: `grep -rn "!" src | grep -v "!="` đếm `!`; `tsc --strict` bắt.

- **Lỗi 2: File augmentation không hoạt động**
  - Triệu chứng: `req.user` vẫn lỗi `Property 'user' does not exist`.
  - Nguyên nhân: file không có `import`/`export` nên thành script, không phải module; `declare module` không khớp specifier.
  - Fix: thêm `import "express"` hoặc `export {}` đầu file; kiểm tra module name (`express-serve-static-core` vs `express`).
  - Đo: `tsc --traceResolution` xem file có được include.

- **Lỗi 3: Decorator không chạy**
  - Triệu chứng: `@Log` không log.
  - Nguyên nhân: quên `experimentalDecorators: true` + `emitDecoratorMetadata: true`, hoặc dùng decorator trên arrow property (không có descriptor).
  - Fix: bật flags trong `tsconfig`, đặt decorator trên method/getter, không trên `prop = () => {}`.
  - Đo: `tsc --listFiles` xem emit, DevTools breakpoint trong decorator.

- **Lỗi 4: `tsc -b` không incremental**
  - Triệu chứng: mỗi lần `tsc -b` vẫn check hết.
  - Nguyên nhân: thiếu `composite: true` hoặc `incremental: true`, hoặc import barrel `index.ts`.
  - Fix: bật `composite`, `tsBuildInfoFile`, tránh barrel, dùng `nx`/`turborepo` cache.
  - Đo: `tsc -b --verbose`, `tsc --extendedDiagnostics` xem `Files`/`I/O`/`Check time`.

- **Lỗi 5: Legacy vs new decorator nhầm**
  - Triệu chứng: `context.kind` undefined hoặc `descriptor` undefined.
  - Fix: thống nhất `experimentalDecorators` cho cả project; legacy dùng `target,key,descriptor`, new dùng `value,context`.
  - Đo: đọc `tsconfig` và TS version (`tsc --version`).

- **Công cụ**:
  - `tsc --noEmit --strict --extendedDiagnostics`, `tsc --generateTrace`.
  - ESLint: `@typescript-eslint/no-explicit-any`, `no-non-null-assertion`, `strict-boolean-expressions`.
  - `type-coverage`, `ts-prune`, `madge` (circular).

## 7. Câu hỏi tự kiểm tra

1. Vì sao `strictNullChecks` được coi là flag quan trọng nhất trong `strict`? Cho ví dụ tắt/bật khác nhau thế nào với `let s: string = null`.
2. Phân biệt legacy decorator (`experimentalDecorators: true`) và TC39 decorator (TS 5+). `emitDecoratorMetadata` và `reflect-metadata` có vai trò gì?
3. File `types/express.d.ts` augment `Request` nhưng không hoạt động. Liệt kê 2 nguyên nhân phổ biến và cách fix. `declare global` khác `declare module` thế nào?

<details>
<summary>Đáp án 30s</summary>

1. Vì nó loại `null`/`undefined` khỏi mọi type, buộc `string` phải `string| null` mới nhận `null` và mọi dùng phải narrow (`?.`, `??`, `if`). Tắt thì `let s: string = null` không lỗi và `s.toUpperCase()` crash runtime; bật thì báo `Type 'null' is not assignable to 'string'` và `Object is possibly 'null'` ngay lúc compile, bắt ~30-40% bug.
2. Legacy: decorator là hàm nhận `(target, key, descriptor)` (method) hoặc `(target)` (class), thứ tự `@f @g C` → `f(g(C))`, cần `experimentalDecorators`. TC39: nhận `(value, context)` với `context.kind`. `emitDecoratorMetadata` emit `design:type/paramtypes/returntype`, `reflect-metadata` đọc qua `Reflect.getMetadata` để DI (NestJS) biết type runtime.
3. (1) File không có `import`/`export` nên thành script global, `declare module` không được coi là augmentation — fix: thêm `import "express"` hoặc `export {}`. (2) Sai module name — phải `declare module "express-serve-static-core"` (nơi `Request` được định nghĩa), không phải `"express"`. `declare global` mở rộng global scope (`Window`, `globalThis`), `declare module "x"` mở rộng module `"x"`.

</details>

---
*Tham khảo chi tiết: `docs/01-javascript-typescript.md` — Câu 25, 27, 29, 30. Spec: [TS Handbook — Strict](https://www.typescriptlang.org/docs/handbook/2/basic-types.html#strictness), [Decorators](https://www.typescriptlang.org/docs/handbook/decorators.html), [Declaration Merging](https://www.typescriptlang.org/docs/handbook/declaration-merging.html), [Module Augmentation](https://www.typescriptlang.org/docs/handbook/declaration-merging.html#module-augmentation), [Project References](https://www.typescriptlang.org/docs/handbook/project-references.html).*
