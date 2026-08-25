# Scope, Hoisting và TDZ — Phạm vi từ vựng, cơ chế nâng khai báo và vùng chết tạm thời

> Tags: #scope #hoisting #tdz #lexical-environment | Nguồn: `docs/01-javascript-typescript.md` câu 1 | Mức: P0

## 1. Định nghĩa chính xác

**Scope** là tập hợp các quy tắc xác định khả năng truy cập biến tại một vị trí trong mã nguồn, được quyết định bởi cấu trúc LexicalEnvironment tại thời điểm biên dịch/phân tích. JavaScript sử dụng **Lexical Scope** (static scope): scope của biến được xác định bởi vị trí khai báo trong mã nguồn, không phải nơi hàm được gọi.

**Hoisting** là hiện tượng engine di chuyển việc tạo binding của khai báo lên đầu scope trong **Creation phase** trước khi thực thi code. Kết quả: có thể tham chiếu tới khai báo trước dòng khai báo trong mã nguồn, nhưng hành vi khác nhau giữa `var` và `let/const`.

**Temporal Dead Zone (TDZ)** là khoảng thời gian từ khi bước vào scope cho tới khi binding của `let/const` được khởi tạo (Initialization), trong đó mọi truy cập tới biến sẽ ném `ReferenceError`.

## 2. Cơ chế hoạt động

### 2.1 Execution Context và LexicalEnvironment

Mỗi Execution Context chứa `LexicalEnvironment` và `VariableEnvironment`, mỗi Environment gồm `EnvironmentRecord` + `outer` reference.

Quy trình cho mỗi scope (global, function, block):

1. **Creation phase** (khi entering scope):
   - Với `var`: `CreateAndInitializeBinding(name, undefined)` — tạo và khởi tạo ngay với `undefined`, sau đó `CreateGlobalVarBinding` gắn lên Global Object nếu ở global scope.
   - Với `let/const`: `CreateMutableBinding(name)` hoặc `CreateImmutableBinding(name)` — tạo binding nhưng **chưa khởi tạo** (uninitialized). Trạng thái này chính là TDZ.
   - Với `function` declaration: `CreateAndInitializeBinding` với function object — hoist toàn bộ, có thể gọi trước khai báo.
   - Với `class`: như `let` — nằm trong TDZ, không thể `new` trước khai báo.
2. **Execution phase**: thực thi tuần tự. Khi gặp dòng `let x = 1`:
   - `InitializeBinding(x, 1)` — thoát TDZ, gán giá trị.
   - Với `var x = 1`: thực chất là `SetMutableBinding` (vì đã khởi tạo ở creation phase).

```
Creation phase:  scan declarations → create bindings → var/function initialized, let/const uninitialized
Execution phase: chạy code → InitializeBinding khi gặp let/const → SetMutableBinding khi gán
TDZ: [entering scope] ──────────── [let/const declaration + initialization] ──→ accessible
```

### 2.2 Block Scope vs Function Scope

- `var`: **function-scoped** hoặc global-scoped. Khai báo trong `if`/`for`/`{}` không tạo scope mới — vẫn thuộc function bao ngoài.
- `let/const`: **block-scoped** — mỗi `{}` tạo một LexicalEnvironment mới. `for (let i...)` còn đặc biệt hơn: tạo binding mới cho mỗi iteration (per-iteration Environment).

### 2.3 Hoisting thực chất

Hoisting không phải engine "di chuyển code". Đó là hệ quả của việc Creation phase xử lý khai báo trước Execution phase. Tất cả declaration đều được "hoist" (tạo binding sớm), nhưng chỉ `var`/`function` được khởi tạo sớm nên truy cập được.

## 3. Ví dụ tối thiểu

```js
// 3.1 var hoisting → undefined, let/const → ReferenceError
console.log(a); // undefined
var a = 1;

console.log(typeof b); // ReferenceError: Cannot access 'b' before initialization
let b = 2;

// typeof với biến chưa từng khai báo thì không lỗi (trả 'undefined')
// nhưng với biến trong TDZ thì vẫn lỗi — điểm phân biệt interview hay hỏi
console.log(typeof undeclaredVar); // 'undefined'
try { console.log(typeof tdzVar); let tdzVar = 1; } catch (e) { console.log(e.name); } // ReferenceError

// 3.2 function declaration hoist toàn bộ
foo(); // 'foo'
function foo() { return 'foo'; }

// function expression / arrow không hoist như declaration
try { bar(); } catch (e) { console.log(e.name); } // TypeError hoặc ReferenceError
var bar = function() {};

// class trong TDZ
try { new MyClass(); } catch (e) { console.log(e.name); } // ReferenceError
class MyClass {}

// 3.3 var trong block không tạo scope
function testVar() {
  if (false) var x = 1;
  console.log(x); // undefined — declaration đã hoist, nhưng không gán
}
testVar();

// 3.4 let per-iteration binding — closure đúng
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log('var:', i), 0);
}
// 3 3 3 — cùng một binding i

for (let j = 0; j < 3; j++) {
  setTimeout(() => console.log('let:', j), 0);
}
// 0 1 2 — mỗi iteration một LexicalEnvironment mới

// 3.5 const không phải immutable tuyệt đối
const obj = { x: 1 };
obj.x = 2; // hợp lệ — mutate property
// obj = {}; // TypeError: Assignment to constant variable
Object.freeze(obj); // mới thực sự shallow immutable
```

```js
// 3.6 TDZ và default parameter
let x = 1;
function tdExample(a = x, b = y) {
  let y = 2;
  return [a, b];
}
console.log(tdExample()); // ReferenceError: Cannot access 'y' before initialization
// param initializer cũng có TDZ riêng
```

## 4. So sánh / Phân loại

| Tiêu chí | `var` | `let` | `const` |
|----------|-------|-------|---------|
| Scope | function / global | block | block |
| Hoisting | Có, khởi tạo `undefined` | Có, không khởi tạo (TDZ) | Có, không khởi tạo (TDZ) |
| Re-declare cùng scope | Cho phép | `SyntaxError` | `SyntaxError` |
| Re-assign | Cho phép | Cho phép | Không cho phép (immutable binding) |
| Gắn lên Global Object (`window`) | Có (`window.varName`) | Không | Không |
| Khởi tạo bắt buộc | Không (`var x;` → `undefined`) | Không (`let x;` → `undefined` sau khi qua TDZ) | Có (`const x;` → `SyntaxError`) |
| Per-iteration binding trong for | Không (1 binding) | Có (mỗi vòng 1 binding) | Có (nhưng không re-assign được, chỉ dùng cho `for...of/in`) |

| Khai báo | Truy cập trước dòng khai báo |
|----------|-------------------------------|
| `function foo(){}` | Thành công — trả function object |
| `var x` | `undefined` |
| `let/const/class` | `ReferenceError` (TDZ) |
| Chưa khai báo bao giờ | `ReferenceError` (hoặc `typeof` → `'undefined'`) |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Luôn dùng `const` mặc định**: signal rằng binding không bị re-assign, giúp engine tối ưu và người đọc suy luận dễ hơn.
- **Dùng `let` khi cần re-assign**: accumulator, flag, biến đếm.
- **Không dùng `var`**:
  - Gây ô nhiễm Global Object, khó 추론 scope.
  - Hoisting với `undefined` che giấu lỗi TDZ đáng lẽ phải fail-fast.
  - Không có per-iteration binding, dễ gây bug closure trong loop.
  - Cho phép re-declare che lỗi typo.
- **Ngoại lệ duy nhất cho `var`**: code phải chạy trên engine không hỗ trợ ES2015, hoặc polyfill yêu cầu function-scoped hoisting có chủ ý (rất hiếm).
- **Không lạm dụng block `{}` chỉ để tạo scope** nếu không cần thiết — làm code khó đọc; thay bằng function/module scope.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: Truy cập `let/const` trong TDZ**
  - Triệu chứng: `ReferenceError: Cannot access 'X' before initialization`.
  - Nguyên nhân: tham chiếu trước dòng khai báo, hoặc circular dependency giữa module ESM (live binding trong TDZ).
  - Fix: di chuyển khai báo lên trước, hoặc tách module.
  - Đo: ESLint rule `no-use-before-define`, TypeScript `noUncheckedIndexedAccess` gián tiếp bắt.

- **Lỗi 2: Nghĩ `const` là deep immutable**
  - Triệu chứng: mutate `const obj = {a:1}; obj.a=2` không lỗi, gây bug state trong Redux/React.
  - Fix: `Object.freeze` (shallow) hoặc `immer`/`structuredClone` cho deep immutability. Dùng `as const` trong TS.
  - Đo: ESLint `no-param-reassign`, `prefer-const`, freeze trong test.

- **Lỗi 3: `var` trong `if`/`for` rò rỉ ra ngoài**
  - Triệu chứng: biến `var` tồn tại ngoài block, ghi đè biến cùng tên.
  - Fix: thay bằng `let/const`, bật ESLint `block-scoped-var`, `no-var`.
  - Đo: `grep -n "var "`, coverage cho thấy biến tồn tại ngoài mong đợi.

- **Lỗi 4: Closure trong loop với `var`**
  - Triệu chứng: `for(var i...) setTimeout(()=>i)` luôn ra giá trị cuối.
  - Fix: đổi `var`→`let`, hoặc IIFE `(function(i){...})(i)`.
  - Đo: unit test với fake timers (`vi.useFakeTimers()`), kiểm tra giá trị per iteration.

- **Lỗi 5: `typeof` trong TDZ vẫn ném**
  - Triệu chứng: `typeof x` tưởng an toàn nhưng với `let x` phía dưới thì ném `ReferenceError`.
  - Fix: không dùng `typeof` để dò biến block-scoped trước khai báo; kiểm tra scope rõ ràng.
  - Debug: Chrome DevTools → Sources → Scope panel hiển thị `<value unavailable>` trong TDZ.

- **Công cụ**:
  - ESLint: `no-var`, `prefer-const`, `no-use-before-define: ["error", { "classes": true }]`
  - TypeScript: `useDefineForClassFields`, strict mode.
  - DevTools: Breakpoint tại dòng đầu block → Scope → xem binding `uninitialized`.

## 7. Câu hỏi tự kiểm tra

1. Vì sao `var` truy cập trước khai báo trả `undefined` còn `let` lại ném `ReferenceError`, dù cả hai đều được hoisting?
2. `for (let i=0; i<3; i++) setTimeout(()=>console.log(i))` in ra `0 1 2`, còn `var` thì `3 3 3`. Cơ chế per-iteration Environment hoạt động thế nào?
3. `typeof undeclaredVar` trả `'undefined'` nhưng `typeof tdzVar` (với `let tdzVar` ở dòng sau) lại ném `ReferenceError`. Giải thích khác biệt?

<details>
<summary>Đáp án 30s</summary>

1. Cả hai đều tạo binding ở Creation phase, nhưng `var` dùng `CreateAndInitializeBinding(undefined)` nên đã khởi tạo; `let/const` chỉ `CreateMutableBinding` chưa khởi tạo → nằm trong TDZ, mọi `GetBindingValue` đều ném `ReferenceError` cho tới `InitializeBinding`.
2. Spec `ForLoopEvaluation` với `let` tạo LexicalEnvironment mới mỗi iteration, copy giá trị `i` trước đó sang environment mới. Mỗi closure `setTimeout` giữ `[[Environment]]` khác nhau. `var` chỉ có một binding duy nhất trong function/global environment.
3. `typeof undeclaredVar` không tìm thấy binding nào → trả `'undefined'` (toán tử `typeof` có special case cho unresolvable reference). `typeof tdzVar` tìm thấy binding nhưng binding đang uninitialized (trong TDZ) → spec yêu cầu ném `ReferenceError` trước khi kiểm tra `typeof`.

</details>

---
*Tham khảo chi tiết: `docs/01-javascript-typescript.md` — Câu 1. Spec: [ECMA-262 §13.3.1 Let and Const Declarations](https://tc39.es/ecma262/#sec-let-and-const-declarations), [§8.1 Lexical Environments](https://tc39.es/ecma262/#sec-lexical-environments).*
