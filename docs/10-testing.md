# 10. Testing - 12 Câu Hỏi Senior

> 12 câu hỏi về testing cho Frontend Senior (Câu 161-172) - từ unit, integration, E2E, testing pyramid đến RTL, MSW, Playwright/Cypress và chiến lược test flow thực tế. Senior không hỏi "viết test thế nào" mà hỏi "test cái gì, vì sao, và trade-off".

## Mục lục

- [Câu 161: Unit vs Integration vs E2E - phân biệt](#câu-161-unit-vs-integration-vs-e2e---phân-biệt)
- [Câu 162: Testing Pyramid là gì? Vì sao không chỉ viết E2E?](#câu-162-testing-pyramid-là-gì-vì-sao-không-chỉ-viết-e2e)
- [Câu 163: Test React component như thế nào?](#câu-163-test-react-component-như-thế-nào)
- [Câu 164: Jest vs Vitest - khác gì, chọn gì?](#câu-164-jest-vs-vitest---khác-gì-chọn-gì)
- [Câu 165: React Testing Library (RTL) - triết lý](#câu-165-react-testing-library-rtl---triết-lý)
- [Câu 166: Vì sao không test implementation detail?](#câu-166-vì-sao-không-test-implementation-detail)
- [Câu 167: Mock API và MSW - khi nào dùng gì?](#câu-167-mock-api-và-msw---khi-nào-dùng-gì)
- [Câu 168: Playwright vs Cypress - chọn gì cho E2E?](#câu-168-playwright-vs-cypress---chọn-gì-cho-e2e)
- [Câu 169: Thiết kế test cho flow Login → Dashboard → Add Product → Checkout](#câu-169-thiết-kế-test-cho-flow-login--dashboard--add-product--checkout)
- [Câu 170: Test custom hook như thế nào?](#câu-170-test-custom-hook-như-thế-nào)
- [Câu 171: Snapshot testing - khi nào hữu ích, khi nào hại?](#câu-171-snapshot-testing---khi-nào-hữu-ích-khi-nào-hại)
- [Câu 172: Code Coverage, Flaky Test và chiến lược test hiệu quả](#câu-172-code-coverage-flaky-test-và-chiến-lược-test-hiệu-quả)


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 161: Unit vs Integration vs E2E - phân biệt

**Trả lời Senior:**
Ba tầng test khác nhau ở **scope, tốc độ, độ tin cậy và chi phí**:

- **Unit test**: test **1 đơn vị nhỏ nhất** cô lập (function, hook, util). Không render DOM thật, mock hết dependency. Chạy **ms**, rẻ, dễ debug nhưng không bắt lỗi tích hợp. Ví dụ: `formatPrice(1000) === "1.000₫"`.
- **Integration test**: test **nhiều unit phối hợp** (component + hook + API mock + store). Render component thật, mock network ở biên. Đây là **tầng giá trị nhất** cho Frontend. Ví dụ: `ProductList` fetch từ MSW rồi render đúng `ProductCard`.
- **E2E test**: test **toàn hệ thống** như user thật trên browser thật, không mock (hoặc mock tối thiểu), đi qua BE, DB, auth. Chậm (giây-phút), flaky, đắt nhưng bắt lỗi hệ thống mà 2 tầng kia không thấy. Ví dụ: Playwright mở `login → thêm giỏ → checkout → thanh toán`.

Bảng:

| | Unit | Integration | E2E |
|---|---|---|---|
| Scope | 1 hàm/hook | 1 feature (nhiều component) | Cả app |
| Môi trường | Node/jsdom | jsdom + MSW | Browser thật |
| Tốc độ | < 50ms | 100-500ms | 5-30s |
| Flaky | Thấp | Trung bình | Cao |
| Khi nào | Logic thuần | Component + data flow | Critical path |

```typescript
// Unit - thuần logic
import { calcTotal } from './cart';
test('calcTotal tính đúng', () => {
  expect(calcTotal([{ price: 100, qty: 2 }])).toBe(200);
});

// Integration - component + API mock
import { render, screen } from '@testing-library/react';
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';
const server = setupServer(http.get('/api/products', () => HttpResponse.json([{ id: 1, name: 'Giày' }])));
beforeAll(() => server.listen()); afterAll(() => server.close());
test('render list từ API', async () => {
  render(<ProductList />);
  expect(await screen.findByText('Giày')).toBeInTheDocument();
});

// E2E - Playwright
// test('checkout', async ({ page }) => {
//   await page.goto('/login'); await page.fill('[name=email]', 'a@a.com'); ...
// });
```

**Trade-off:** Team chỉ viết unit sẽ có coverage cao nhưng vẫn bug khi ghép component. Chỉ viết E2E sẽ chậm, flaky, CI 30 phút. Senior cân bằng: **nhiều integration, vừa unit, ít E2E cho critical path**.

**Câu hỏi đào sâu:** Vì sao Kent C. Dodds nói "càng gần user, càng tự tin"? Unit test cho component có phải lúc nào cũng cần?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 162: Testing Pyramid là gì? Vì sao không chỉ viết E2E?

**Trả lời Senior:**
Testing Pyramid (Mike Cohn) mô tả **tỷ lệ test lý tưởng**: đáy to là **Unit (70%)**, giữa là **Integration (20%)**, đỉnh nhỏ là **E2E (10%)**. Biến thể cho Frontend là **Testing Trophy** (Kent C. Dodds): **Static (TS/ESLint) → Unit → Integration → E2E**, nhấn mạnh Integration là thân trophy - to nhất.

Vì sao không chỉ E2E dù nó giống user nhất?

1.  **Chi phí**: E2E cần browser, BE, DB, data seeding, chạy chậm 50-100x so với unit. 500 E2E = CI 1 giờ, 500 unit = 10s.
2.  **Flaky**: E2E phụ thuộc timing, network, animation, third-party → fail ngẫu nhiên, team mất niềm tin, bỏ test.
3.  **Debug khó**: E2E fail chỉ báo "timeout", phải xem video/trace. Unit fail báo đúng dòng `expect`.
4.  **Feedback loop**: dev cần feedback trong 5s khi code, không thể đợi E2E.

Ngược lại, chỉ unit thì **tin cậy thấp**: mock quá nhiều, không test wiring.

```
        /\
       /E2E\          10% - Critical flow (login, checkout)
      /------\
     /Integr. \     20-50% - Component + API + Store
    /----------\
   /   Unit     \  40-60% - Util, hook, pure logic
  /--------------\
 / Static (TS)    \ - Type, lint bắt lỗi trước khi chạy
```

```typescript
// Trophy áp dụng cho FE
// 1. Static: tsc --noEmit bắt lỗi type
// 2. Unit: test/format.test.ts
// 3. Integration: test/ProductList.test.tsx (RTL + MSW)
// 4. E2E: e2e/checkout.spec.ts (Playwright)

// Chống pyramid ngược (ice cream cone): nhiều E2E, ít unit -> CI chậm, flaky
```

**Trade-off:** Dự án startup có thể chấp nhận pyramid dẹt: ít E2E hơn, tăng integration. App ngân hàng thì E2E nhiều hơn nhưng vẫn phải giữ pyramid, không để thành cone.

**Câu hỏi đào sâu:** Testing Trophy khác Pyramid thế nào? Khi nào nên phá pyramid và tăng E2E?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 163: Test React component như thế nào?

**Trả lời Senior:**
Test component Senior không test `state` hay `props` nội bộ mà test **hành vi người dùng**: render gì, click gì, thấy gì. Stack chuẩn: **Vitest/Jest + RTL + user-event + MSW**.

Quy trình 3 bước: **Render → Interact → Assert** (Arrange-Act-Assert).

1.  **Render** với `render(<Component />)` trong `jsdom`.
2.  **Interact** bằng `userEvent` (click, type, select) - mô phỏng user thật, không `fireEvent` thô.
3.  **Assert** bằng `screen.getByRole` (ưu tiên role), `findBy` cho async, `toBeInTheDocument`.

```typescript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import ProductCard from './ProductCard';

// Component cần test
// function ProductCard({ product, onAdd }: Props) {
//   return <article><h2>{product.name}</h2><button onClick={() => onAdd(product)}>Thêm</button></article>
// }

test('hiển thị tên và giá, click Thêm gọi onAdd', async () => {
  const user = userEvent.setup();
  const onAdd = vi.fn();
  const product = { id: '1', name: 'Giày Nike', price: 1000 };

  render(<ProductCard product={product} onAdd={onAdd} />);

  // Assert render - ưu tiên getByRole
  expect(screen.getByRole('heading', { name: /giày nike/i })).toBeInTheDocument();
  expect(screen.getByRole('button', { name: /thêm/i })).toBeInTheDocument();

  // Interact
  await user.click(screen.getByRole('button', { name: /thêm/i }));

  // Assert hành vi
  expect(onAdd).toHaveBeenCalledWith(product);
  expect(onAdd).toHaveBeenCalledTimes(1);
});

// Async + API
test('hiển thị loading rồi data', async () => {
  render(<ProductList />); // bên trong useQuery fetch /api/products
  expect(screen.getByText(/đang tải/i)).toBeInTheDocument();
  expect(await screen.findByText('Giày Nike')).toBeInTheDocument(); // wait
});

// Cần provider
import { QueryClientProvider } from '@tanstack/react-query';
function renderWithProviders(ui: React.ReactElement) {
  const qc = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  return render(<QueryClientProvider client={qc}>{ui}</QueryClientProvider>);
}
```

**Trade-off:** `userEvent` chậm hơn `fireEvent` nhưng chính xác hơn (gõ từng ký tự, trigger đủ event). Luôn setup `userEvent.setup()` mới mỗi test để isolate.

**Câu hỏi đào sâu:** Vì sao `getByRole` tốt hơn `getByTestId`? `findBy` khác `getBy` + `waitFor` thế nào?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 164: Jest vs Vitest - khác gì, chọn gì?

**Trả lời Senior:**
Cả hai đều là **test runner + assertion + mock + coverage**, API tương thích 90% (`describe`, `test`, `expect`, `vi.fn`/`jest.fn`). Khác ở **engine và tốc độ**.

- **Jest** (2014, Facebook): chạy trên `jsdom`, transform bằng `babel-jest`/`ts-jest`, cache riêng, ecosystem lớn, snapshot, watch mode mạnh. Chậm hơn vì phải transform lại, config `jest.config.js` phức tạp khi dùng ESM/TS.
- **Vitest** (2021, Evan You): chạy trên **Vite** (esbuild + Vite transform), dùng chung `vite.config.ts`, hỗ trợ **ESM, TS, JSX native**, **HMR cho test**, chạy **nhanh 3-10x**, worker threads, snapshot tương thích Jest. Dùng `jsdom` hoặc `happy-dom`.

| | Jest | Vitest |
|---|---|---|
| Transform | Babel/ts-jest | Vite/esbuild (native ESM) |
| Tốc độ | Chậm, cold start lâu | Nhanh, cache Vite |
| Config | `jest.config.js` riêng | Dùng `vite.config.ts` |
| ESM | Cần flag experimental | Native |
| Mock | `jest.mock`, `jest.fn` | `vi.mock`, `vi.fn` (alias `jest` được) |
| Phù hợp | CRA, legacy | Vite, Next, modern FE |

```typescript
// vitest.config.ts - gọn, dùng chung alias với Vite
import { defineConfig } from 'vitest/config';
export default defineConfig({
  test: {
    environment: 'jsdom',
    globals: true, // để không cần import describe/test
    setupFiles: ['./test/setup.ts'],
    coverage: { provider: 'v8' },
  },
});

// setup.ts
import '@testing-library/jest-dom/vitest';

// Test viết y như Jest
import { describe, it, expect, vi } from 'vitest';
test('mock', () => {
  const fn = vi.fn(() => 42);
  expect(fn()).toBe(42);
  expect(vi.isMockFunction(fn)).toBe(true);
});

// Jest tương đương: jest.fn(), jest.mock('./api')
```

**Trade-off:** Jest ổn định, plugin nhiều, nhưng với dự án Vite/Next mới, Vitest là default vì **không config 2 lần** và nhanh. Migration Jest → Vitest thường chỉ đổi `jest.` thành `vi.` và alias.

> Update 2025: Vitest 3.x + Vite 6, coverage v8/browserslist

**Câu hỏi đào sâu:** Vì sao Vitest nhanh hơn Jest? Khi nào vẫn phải dùng Jest (React Native, legacy Babel)?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 165: React Testing Library (RTL) - triết lý

**Trả lời Senior:**
RTL triết lý **"test như user"** (Guiding Principles của Kent C. Dodds): test nên **tương tác với DOM như user**, ưu tiên **accessibility**, không quan tâm implementation. Thay vì test `state` hay `props`, test **user thấy gì và làm gì**.

Nguyên tắc:

1.  **Truy vấn theo cách user tìm**: `getByRole` (button, heading) > `getByLabelText` > `getByPlaceholderText` > `getByText` > `getByTestId` (cuối cùng). Nếu phải dùng `testId`, là UI thiếu accessibility.
2.  **Không test implementation**: không `wrapper.state()`, không `wrapper.instance()`, không shallow render.
3.  **Dùng `userEvent` thay `fireEvent`**: user gõ, click, tab như thật.
4.  **Assert visible behavior**: `toBeInTheDocument`, `toBeDisabled`, `toHaveAccessibleName`.

```typescript
// ❌ Enzyme - test implementation
// expect(wrapper.state().isOpen).toBe(true);
// expect(wrapper.find('Button').prop('onClick')).toBe(fn);

// ✅ RTL - test hành vi
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

test('user mở modal và thấy form', async () => {
  const user = userEvent.setup();
  render(<Header />);

  // User tìm nút theo role + tên, như screen reader
  const openBtn = screen.getByRole('button', { name: /mở giỏ hàng/i });
  await user.click(openBtn);

  // Sau click, dialog xuất hiện
  expect(screen.getByRole('dialog')).toBeInTheDocument();
  expect(screen.getByLabelText(/email/i)).toBeInTheDocument();

  // Không assert state nội bộ
});

// Thứ tự ưu tiên query - từ tốt nhất
// 1. getByRole('button', { name: /submit/i }) - a11y
// 2. getByLabelText(/email/i) - form
// 3. getByPlaceholderText(/nhập email/i)
// 4. getByText(/tiếp tục/i)
// 5. getByDisplayValue, getByAltText
// 6. getByTestId('submit-btn') - cuối cùng
```

**Trade-off:** RTL làm test hơi dài hơn Enzyme nhưng **bền**: refactor từ `useState` sang `useReducer` hay đổi class không làm test fail, chỉ khi UI/user flow đổi mới fail - đúng tín hiệu.

**Câu hỏi đào sâu:** Vì sao RTL khuyến khích `getByRole`? Enzyme shallow vs RTL mount khác triết lý thế nào?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 166: Vì sao không test implementation detail?

**Trả lời Senior:**
Implementation detail là thứ **user không quan tâm** nhưng dev thay đổi thường xuyên: tên state, tên hàm nội bộ, thứ tự gọi `useEffect`, cấu trúc component, CSS class. Test vào chúng làm test **giòn (brittle)**: refactor không đổi hành vi nhưng test fail, team sợ refactor, test bị xóa.

Ví dụ: test `component.state.count === 1` hoặc `expect(fetch).toHaveBeenCalledWith('/api/v2/products')` cứng URL nội bộ. Khi đổi `count` thành `total` hay đổi endpoint, test fail dù user vẫn thấy đúng.

Nên test **contract công khai**: props, DOM output, event, API contract qua MSW.

```typescript
// ❌ Test implementation detail - giòn
test('giòn: check state nội bộ', () => {
  const wrapper = shallow(<Counter />);
  // @ts-ignore
  expect(wrapper.state('count')).toBe(0); // fail khi đổi sang useReducer
  wrapper.instance().increment();
  expect(wrapper.state('count')).toBe(1);
});

// ❌ Test chi tiết gọi hàm
test('giòn: check gọi đúng hàm nội bộ', () => {
  const spy = vi.spyOn(api, 'fetchProductsV2');
  render(<List />);
  expect(spy).toHaveBeenCalled(); // fail khi đổi sang fetchProductsV3
});

// ✅ Test hành vi - bền
test('bền: user thấy count tăng khi click', async () => {
  const user = userEvent.setup();
  render(<Counter />);
  expect(screen.getByText('0')).toBeInTheDocument();
  await user.click(screen.getByRole('button', { name: /tăng/i }));
  expect(screen.getByText('1')).toBeInTheDocument();
  // Dù đổi useState -> useReducer, test vẫn pass
});

// ✅ Test contract API qua MSW, không spy hàm nội bộ
test('bền: hiển thị product từ API', async () => {
  server.use(http.get('/api/products', () => HttpResponse.json([{ name: 'A' }])));
  render(<List />);
  expect(await screen.findByText('A')).toBeInTheDocument();
});

// Ngoại lệ: có thể test implementation nếu là library util, không phải UI
```

**Trade-off:** Đôi khi test implementation nhanh hơn (spy `localStorage.setItem`). Chấp nhận nếu đó là **contract** (persist), nhưng ưu tiên test observable: "reload vẫn thấy cart" thay vì "đã gọi setItem".

**Câu hỏi đào sâu:** Khi nào test implementation detail là chấp nhận được? Làm sao biết mình đang test detail thay vì behavior?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 167: Mock API và MSW - khi nào dùng gì?

**Trả lời Senior:**
Có 3 cách mock API trong test FE:

1.  **`vi.mock` / `jest.mock`**: mock module (`fetch`, `axios`, `lib/api`). Nhanh, đơn giản nhưng **không test integration thật**, mock sai contract vẫn pass, và mock phải khớp import path.
2.  **`fetch` mock thủ công**: `global.fetch = vi.fn(() => Promise.resolve(...))`. Thô, phải tự handle mọi case.
3.  **MSW (Mock Service Worker)**: intercept **network layer** (Service Worker trên browser, `http` server trên Node). App gọi `fetch('/api/products')` thật, MSW chặn và trả mock. **Không mock code, mock network** → test gần thật nhất, dùng chung cho test, dev, Storybook.

Senior ưu tiên **MSW cho integration/E2E**, `vi.mock` chỉ cho unit cô lập hoặc third-party SDK.

```typescript
// ❌ vi.mock - mock module
vi.mock('./api', () => ({ fetchProducts: vi.fn(() => Promise.resolve([{ id: 1 }])) }));
// Vấn đề: nếu đổi ./api sang ./services/api, mock hỏng; không test fetch thật

// ✅ MSW - mock network
// mocks/handlers.ts
import { http, HttpResponse } from 'msw';
export const handlers = [
  http.get('/api/products', ({ request }) => {
    const url = new URL(request.url);
    const category = url.searchParams.get('category');
    if (category === 'shoes') return HttpResponse.json([{ id: 1, name: 'Giày' }]);
    return HttpResponse.json([]);
  }),
  http.post('/api/cart', async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ success: true }, { status: 201 });
  }),
  http.get('/api/products/:id', ({ params }) => {
    return HttpResponse.json({ id: params.id, name: 'Chi tiết' });
  }),
];

// mocks/server.ts (Node cho Vitest)
import { setupServer } from 'msw/node';
import { handlers } from './handlers';
export const server = setupServer(...handlers);

// setup.ts
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

// Trong test - override cho case lỗi
test('hiển thị lỗi khi API 500', async () => {
  server.use(http.get('/api/products', () => HttpResponse.json({ message: 'Lỗi' }, { status: 500 })));
  render(<ProductList />);
  expect(await screen.findByText(/lỗi/i)).toBeInTheDocument();
});
```

| | `vi.mock` | MSW |
|---|---|---|
| Mock tầng | Module | Network |
| Refactor an toàn | Kém (path đổi là hỏng) | Cao (URL đổi mới hỏng) |
| Dùng chung dev/test | Không | Có |
| Setup | Nhanh | Cần server/worker |
| Dùng khi | Unit, SDK | Integration |

**Trade-off:** MSW setup thêm 1-2 file, nhưng cho **độ tin cậy cao** và **tái sử dụng**. Với E2E Playwright, có thể dùng `page.route()` tương tự MSW.

**Câu hỏi đào sâu:** MSW khác `vi.mock('axios')` thế nào về độ tin cậy? Khi nào vẫn cần `vi.mock` dù đã có MSW?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 168: Playwright vs Cypress - chọn gì cho E2E?

**Trả lời Senior:**
Cả hai đều E2E trên browser thật, khác kiến trúc:

- **Cypress**: chạy **trong browser** (in-process), API `cy.*` chainable, time-travel debugger, UI đẹp, dễ học. Hạn chế: **chỉ 1 tab/origin**, không multi-tab, không test file download native tốt, chạy trên Electron/Chromium chính, Firefox/WebKit hỗ trợ muộn, không hỗ trợ mobile native.
- **Playwright** (Microsoft): chạy **ngoài browser** (out-of-process) qua WebSocket điều khiển **Chromium, Firefox, WebKit** thật, hỗ trợ **multi-tab, multi-origin, iframe, file download, api testing, parallel, trace viewer, codegen**. Nhanh hơn, ít flaky hơn nhờ **auto-wait** mạnh.

| | Cypress | Playwright |
|---|---|---|
| Browser | Chromium (chính), FF/WebKit muộn | Chromium + Firefox + WebKit (3 engine thật) |
| Multi-tab/origin | Không | Có |
| Ngôn ngữ | JS/TS chính | JS/TS, Python, Java, C# |
| Parallel | Cần Cypress Cloud (trả phí) | Native `workers` free |
| Auto-wait | Có | Mạnh hơn (actionability checks) |
| Trace | Time travel | Trace Viewer + Video + Screenshot |
| Phù hợp | Team nhỏ, cần DX nhanh | Team cần cross-browser, CI scale |

```typescript
// Cypress
describe('checkout', () => {
  it('login và mua', () => {
    cy.visit('/login');
    cy.get('[data-cy=email]').type('a@a.com');
    cy.get('[data-cy=submit]').click();
    cy.url().should('include', '/dashboard');
    cy.contains('Thêm vào giỏ').click();
  });
});

// Playwright - hiện đại, async/await, parallel
import { test, expect } from '@playwright/test';
test('checkout', async ({ page }) => {
  await page.goto('/login');
  await page.getByRole('textbox', { name: /email/i }).fill('a@a.com');
  await page.getByRole('button', { name: /đăng nhập/i }).click();
  await expect(page).toHaveURL(/dashboard/);
  await page.getByRole('button', { name: /thêm vào giỏ/i }).click();
  await expect(page.getByText(/đã thêm/i)).toBeVisible();
  // Multi-tab
  // const newPage = await context.newPage();
});

// playwright.config.ts
export default defineConfig({
  testDir: './e2e',
  workers: 4,
  use: { trace: 'on-first-retry', screenshot: 'only-on-failure' },
  projects: [{ name: 'chromium' }, { name: 'firefox' }, { name: 'webkit' }],
});
```

**Trade-off:** Cypress DX dễ cho người mới, nhưng Playwright là **chuẩn mới cho 2024-2026**: nhanh, free parallel, cross-browser thật, Microsoft hậu thuẫn. Trừ khi team đã đầu tư Cypress Cloud, nên chọn Playwright.

**Câu hỏi đào sâu:** Vì sao Playwright ít flaky hơn Cypress? Khi nào vẫn chọn Cypress (plugin ecosystem, team quen)?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 169: Thiết kế test cho flow Login → Dashboard → Add Product → Checkout

**Trả lời Senior:**
Đây là **critical path** của e-commerce, phải có **1 E2E hạnh phúc (happy path)** + **n integration** cho từng bước, không dồn hết vào 1 E2E dài.

**Chiến lược phân tầng:**

1.  **Unit**: `validateLoginForm`, `calcTotal`, `formatCurrency`.
2.  **Integration (RTL + MSW)**: test từng page cô lập với API mock - login form submit, dashboard fetch, add product mutation.
3.  **E2E (Playwright)**: 1 flow hạnh phúc duy nhất, dùng **Page Object Model (POM)**, **seed data**, **isolated auth**.

Sơ đồ:

```
Unit (nhanh) -> Integration (mock) -> E2E (1 flow thật)
LoginForm.test.tsx (RTL) -> Dashboard.test.tsx (MSW) -> e2e/purchase.spec.ts (Playwright)
```

```typescript
// Integration - Login
test('login thành công chuyển hướng', async () => {
  const user = userEvent.setup();
  server.use(http.post('/api/login', () => HttpResponse.json({ token: 'abc', user: { id: 1 } })));
  renderWithRouter(<LoginPage />);
  await user.type(screen.getByLabelText(/email/i), 'a@a.com');
  await user.type(screen.getByLabelText(/mật khẩu/i), '123456');
  await user.click(screen.getByRole('button', { name: /đăng nhập/i }));
  expect(await screen.findByText(/dashboard/i)).toBeInTheDocument();
});

// E2E - POM
// pages/LoginPage.ts
export class LoginPage {
  constructor(private page: Page) {}
  async login(email: string, password: string) {
    await this.page.goto('/login');
    await this.page.getByLabel(/email/i).fill(email);
    await this.page.getByLabel(/mật khẩu/i).fill(password);
    await this.page.getByRole('button', { name: /đăng nhập/i }).click();
    await expect(this.page).toHaveURL(/dashboard/);
  }
}
// e2e/purchase.spec.ts
test('flow mua hàng', async ({ page }) => {
  const login = new LoginPage(page);
  await login.login('test@e2e.com', '123456'); // seed user có sẵn

  await page.getByRole('button', { name: /thêm sản phẩm/i }).click();
  await page.getByLabel(/tên sản phẩm/i).fill('Giày Test ' + Date.now());
  await page.getByRole('button', { name: /lưu/i }).click();
  await expect(page.getByText(/đã tạo/i)).toBeVisible();

  await page.getByRole('button', { name: /thêm vào giỏ/i }).first().click();
  await page.getByRole('link', { name: /giỏ hàng/i }).click();
  await page.getByRole('button', { name: /thanh toán/i }).click();
  await expect(page.getByText(/đặt hàng thành công/i)).toBeVisible();

  // Assert API: đơn hàng tạo
  // await expect(page.request ... ) hoặc check DB seeding
});

// Tips Senior: 
// - Auth: dùng storageState để login 1 lần cho nhiều test
// - Data: tạo product với timestamp để không trùng
// - Cleanup: afterAll xóa data test
// - Flaky: dùng getByRole + auto-wait, không dùng waitForTimeout cứng
```

| Tầng | Số test | Ví dụ |
|---|---|---|
| Unit | 10-20 | validate, calc |
| Integration | 5-10 | login form, add product form |
| E2E | 1-2 | happy path + 1 unhappy (checkout fail) |

**Trade-off:** E2E dài dễ flaky, nên tách. Nếu E2E fail, phải có video/trace. Không mock API trong E2E happy path, nhưng có thể mock payment gateway.

**Câu hỏi đào sâu:** Vì sao không nên có nhiều E2E cho cùng flow? Làm sao giảm flaky cho E2E checkout (network, animation)?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 170: Test custom hook như thế nào?

**Trả lời Senior:**
Custom hook là logic có `useState`/`useEffect` nên không test như hàm thuần. Có 2 cách:

1.  **`renderHook` của RTL**: render hook trong component test vô hình, cho `result.current`, `rerender`, `unmount`. Nhanh, cô lập.
2.  **Test qua component thật**: tạo `TestComponent` dùng hook rồi test hành vi. Gần user hơn nhưng dài.

Ưu tiên `renderHook` cho unit hook, và thêm 1 integration test qua component nếu hook liên quan DOM.

Với hook async/timer, dùng `act`, `waitFor`, `vi.useFakeTimers`.

```typescript
// hooks/useCounter.ts
export function useCounter(initial = 0) {
  const [count, setCount] = useState(initial);
  const inc = useCallback(() => setCount(c => c + 1), []);
  const dec = useCallback(() => setCount(c => c - 1), []);
  return { count, inc, dec };
}

// Test bằng renderHook
import { renderHook, act } from '@testing-library/react';
test('useCounter tăng giảm', () => {
  const { result } = renderHook(() => useCounter(0));
  expect(result.current.count).toBe(0);
  act(() => result.current.inc());
  expect(result.current.count).toBe(1);
  act(() => result.current.dec());
  expect(result.current.count).toBe(0);
});

// Hook async - fetch
// hooks/useDebounce.ts
export function useDebounce<T>(value: T, delay = 300) {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const t = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(t);
  }, [value, delay]);
  return debounced;
}
test('useDebounce debounce đúng', () => {
  vi.useFakeTimers();
  const { result, rerender } = renderHook(({ v }) => useDebounce(v, 300), { initialProps: { v: 'a' } });
  expect(result.current).toBe('a');
  rerender({ v: 'ab' });
  expect(result.current).toBe('a'); // chưa qua delay
  act(() => vi.advanceTimersByTime(300));
  expect(result.current).toBe('ab');
  vi.useRealTimers();
});

// Hook cần provider
import { QueryClientProvider } from '@tanstack/react-query';
test('useProducts hook', async () => {
  const wrapper = ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={new QueryClient()}>{children}</QueryClientProvider>
  );
  const { result } = renderHook(() => useProducts(), { wrapper });
  await waitFor(() => expect(result.current.isSuccess).toBe(true));
  expect(result.current.data).toHaveLength(1);
});
```

**Trade-off:** `renderHook` nhanh nhưng không test hook tương tác với DOM thật. Với hook như `useClickOutside`, nên test qua component có DOM.

**Câu hỏi đào sâu:** `act` để làm gì? Vì sao phải `vi.useFakeTimers` + `advanceTimers` cho debounce?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 171: Snapshot testing - khi nào hữu ích, khi nào hại?

**Trả lời Senior:**
Snapshot lưu **output render** (HTML/JSON) vào file `.snap`, lần sau so sánh, nếu khác thì fail. Hữu ích để **phát hiện thay đổi bất ngờ**, nhưng dễ bị lạm dụng.

**Hữu ích khi:**

- Test **UI ít đổi, cấu trúc quan trọng**: email template, markdown renderer, config JSON, AST.
- Test **component thuần presentational** nhỏ: `Badge`, `Alert` với nhiều variant, snapshot như **visual regression rẻ tiền**.
- Kết hợp với **inline snapshot** cho object/serialize: `expect(value).toMatchInlineSnapshot()`.

**Hại khi:**

- Snapshot **quá lớn** (cả page 500 dòng): thay 1 class cũng fail, dev `npm test -u` bừa, mất ý nghĩa.
- Test **logic bằng snapshot**: `expect(calcTotal(cart)).toMatchSnapshot()` - nên dùng `toBe` rõ ràng.
- Team **update snapshot không review**: `toMatchSnapshot` thành `toBe` trá hình.

```typescript
// ✅ Hữu ích - component nhỏ nhiều variant
import { render } from '@testing-library/react';
test('Badge snapshot các variant', () => {
  const { container } = render(<><Badge variant="success">OK</Badge><Badge variant="error">Lỗi</Badge></>);
  expect(container).toMatchSnapshot(); // lưu HTML 2 badge
});

// ✅ Inline snapshot cho object
test('serialize config', () => {
  expect(generateConfig({ theme: 'dark' })).toMatchInlineSnapshot(`
    {
      "theme": "dark",
      "variables": {
        "--bg": "#000",
      },
    }
  `);
});

// ❌ Hại - snapshot cả page lớn
test('ProductPage snapshot', () => {
  const { container } = render(<ProductPage />); // 500 dòng HTML
  expect(container).toMatchSnapshot(); // thay 1 chữ cũng fail, ai cũng -u
});

// Thay bằng assertion cụ thể
test('ProductPage hiển thị đúng', async () => {
  render(<ProductPage />);
  expect(screen.getByRole('heading', { name: /giày/i })).toBeInTheDocument();
  expect(screen.getByRole('button', { name: /mua/i })).toBeEnabled();
});

// Tips Senior:
// - Dùng snapshot cho presentational, không cho container/logic
// - Review .snap như code, không auto -u
// - Dùng toMatchSnapshot cho CSS-in-JS class cũng giòn, nên mock className
```

| Dùng | Không dùng |
|---|---|
| Email template, component nhỏ | Page lớn, logic |
| Inline snapshot object | Thay cho `toBe`/`toEqual` |

**Trade-off:** Snapshot rẻ nhưng **độ tin cậy thấp** nếu không review. Nhiều team senior bỏ snapshot cho UI, chỉ giữ cho serialize/config.

**Câu hỏi đào sâu:** Vì sao snapshot lớn làm team "mù" với update? Khi nào `toMatchInlineSnapshot` tốt hơn `toMatchSnapshot`?


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 172: Code Coverage, Flaky Test và chiến lược test hiệu quả

**Trả lời Senior:**
**Coverage** (`lines/branches/functions/statements`) đo **code nào được chạy qua trong test**, không đo **đúng hay không**. 100% coverage không đảm bảo không bug, 70% với test tốt có thể hơn 100% với test vô nghĩa.

**Flaky test** là test **lúc pass lúc fail** không do code đổi, do timing, async, network, animation, timezone, order dependency. Flaky giết niềm tin: team ignore fail, bỏ test.

**Chiến lược Senior:**

1.  **Đặt target coverage hợp lý**: 70-80% tổng, 90% cho util/core, không ép 100%. Dùng `coverageThreshold` để chặn tụt, nhưng không để coverage thành KPI.
2.  **Diệt flaky**: 
     - Dùng `findBy`/`waitFor` thay `waitForTimeout` cứng.
     - Mock timer với `vi.useFakeTimers` thay `setTimeout` thật.
     - Isolate test: `beforeEach` reset state, không share global.
     - Với E2E, dùng `auto-wait`, `getByRole`, retry 2 lần, fix animation `prefers-reduced-motion`.
3.  **Chiến lược**: theo Trophy - nhiều integration, ít E2E, test **behavior** không test implementation, dùng MSW cho ổn định.

```typescript
// vitest.config.ts coverage
export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      thresholds: {
        lines: 70,
        branches: 70,
        // không ép 100%
      },
      exclude: ['src/mocks/**', '**/*.stories.tsx'],
    },
  },
});

// ❌ Flaky - timeout cứng
test('flaky', async () => {
  render(<Async />);
  await new Promise(r => setTimeout(r, 500)); // 500ms có đủ?
  expect(screen.getByText('Done')).toBeInTheDocument();
});

// ✅ Ổn định - waitFor
test('ổn định', async () => {
  render(<Async />);
  expect(await screen.findByText('Done')).toBeInTheDocument(); // tự wait
});

// ❌ Flaky - order dependency
let count = 0;
test('test 1', () => { count = 1; });
test('test 2', () => { expect(count).toBe(1); }); // fail khi chạy isolate

// ✅ Isolate
beforeEach(() => { count = 0; });

// E2E flaky fix - Playwright
// await page.waitForTimeout(1000); // ❌
// await expect(page.getByRole('button', { name: /lưu/i })).toBeEnabled(); // ✅ auto-wait
```

| Sai lầm | Đúng |
|---|---|
| Ép 100% coverage | 70-80% + review quality |
| `waitForTimeout` | `findBy`/`expect.toBeVisible` |
| Share state giữa test | `beforeEach` reset |

**Trade-off:** Coverage là **tín hiệu, không phải mục tiêu**. Team senior xem coverage report để tìm code chưa test, không để bonus theo coverage.

**Câu hỏi đào sâu:** Vì sao 100% coverage vẫn có bug? Làm sao phát hiện và cách ly flaky test trong CI?

[↑ Quay lại Mục lục](#mục-lục)
---
