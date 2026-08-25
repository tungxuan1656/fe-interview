# RTL + MSW + Playwright/Cypress + Vitest/Jest — getByRole, user-event, network vs module mock, out-of-process vs in-process

> Tags: #RTL #React-Testing-Library #user-event #getByRole #MSW #vi.mock #Playwright #Cypress #Vitest #Jest #jsdom | Nguồn: `docs/10-testing.md` câu 163-168, 170 | Mức: P0

## 1. Định nghĩa chính xác

**React Testing Library (RTL)** triết lý **"test như user"** (Kent C. Dodds): truy vấn DOM qua **accessibility role** (`getByRole`) như screen reader, tương tác qua **`user-event`** (mô phỏng `click`/`type`/`tab` đầy đủ event sequence), không test implementation (`state`, `props` nội bộ, `wrapper.instance()`). **MSW (Mock Service Worker)** mock ở **network layer** (Service Worker trên browser, `msw/node` trên Node) — intercept `fetch`/`XHR` thật, app gọi `fetch('/api/products')` thật, MSW chặn và trả `HttpResponse.json()`. Ngược lại **`vi.mock`/`jest.mock`** mock ở **module layer** — thay `import * as api from './api'` bằng `vi.fn()`. **Playwright** là E2E **out-of-process** (điều khiển browser qua WebSocket/CDP, hỗ trợ Chromium/Firefox/WebKit thật, multi-tab/multi-origin, trace viewer). **Cypress** là E2E **in-process** (chạy trong browser, `cy.*` chainable, time-travel). **Vitest** là test runner **Vite-native** (esbuild + Vite transform, ESM/TS/JSX native, HMR, worker threads, 3-10× nhanh Jest). **Jest** (babel-jest/ts-jest, `jest.config.js` riêng, jsdom) là runner legacy.

## 2. Cơ chế hoạt động

- **RTL — Render → Interact (user-event) → Assert (getByRole)**:
  1. `render(<Component />)` mount vào `jsdom` (Vitest `environment: 'jsdom'`).
  2. `userEvent.setup()` tạo instance cô lập mỗi test; `await user.click(button)` dispatch chuỗi `pointerDown → mouseDown → focus → click → pointerUp` như user thật — khác `fireEvent.click` chỉ bắn 1 event thô.
  3. Query ưu tiên **a11y**: `getByRole('button', { name: /thêm/i })` > `getByLabelText` > `getByPlaceholderText` > `getByText` > `getByDisplayValue` > `getByAltText` > `getByTestId` (cuối cùng). Nếu phải dùng `testId`, UI thiếu a11y.
  4. Assert: `toBeInTheDocument()`, `toBeDisabled()`, `toHaveAccessibleName()`. Async: `findByRole` (tự `waitFor` polling 50ms, timeout 1000ms) cho data sau fetch.

- **MSW vs `vi.mock` — network vs module**:
  - `vi.mock('./api', () => ({ fetchProducts: vi.fn(() => Promise.resolve([])) }))`: thay module tại import. Nhanh nhưng **giòn**: đổi path ` ./api → ./services/api` mock hỏng, không test `fetch` thật, không dùng chung cho dev/Storybook, mock sai contract vẫn pass.
  - MSW: `http.get('/api/products', () => HttpResponse.json([...]))` + `setupServer(...handlers)` (Node) hoặc `setupWorker` (browser). App **không biết bị mock**, request đi thật tới network layer rồi bị chặn. Ưu: refactor an toàn (chỉ URL đổi mới hỏng), tái sử dụng cho test/dev/Storybook, phát hiện contract drift sớm. Với Playwright E2E, tương đương `page.route('/api/**', route => route.fulfill(...))`.

- **Playwright (out-of-process) vs Cypress (in-process)**:
  - **Playwright**: Node process điều khiển **3 browser engine thật** (Chromium, Firefox, WebKit) qua WebSocket. Hỗ trợ **multi-tab** (`context.newPage()`), **multi-origin**, `iframe`, `file download`, **parallel workers free**, **auto-wait** (actionability: visible, stable, enabled, not covered), **trace viewer** (DOM snapshot + network + timeline + video/screenshot). `playwright.config.ts`: `workers: 4`, `projects: [{ name: 'chromium' }, { name: 'firefox' }, { name: 'webkit' }]`, `use: { trace: 'on-first-retry' }`.
  - **Cypress**: chạy **trong browser** (iframe), `cy.get('[data-cy=email]').type('a@a.com')` chainable, time-travel debugger, DX dễ. Hạn chế: **không multi-tab/origin** (workaround `cy.origin` hạn chế), chỉ Chromium chính (FF/WebKit muộn), parallel cần **Cypress Cloud trả phí**.

- **Vitest (Vite-native) vs Jest (Babel)**:
  - **Vitest**: dùng **Vite pipeline** (esbuild transform, `vite.config.ts` alias/plugin share), **native ESM**, **HMR cho test**, `happy-dom`/`jsdom`, `vi.fn`/`vi.mock` (alias `jest` tương thích), `coverage: { provider: 'v8' }`. Config gọn: `defineConfig({ test: { globals: true, environment: 'jsdom', setupFiles: ['./test/setup.ts'] } })`.
  - **Jest**: `babel-jest`/`ts-jest` transform riêng, `jest.config.js` duplicate alias, ESM cần `experimental-vm-modules`, cold start chậm. Ổn định, ecosystem lớn (React Native).
  - Migration Jest → Vitest: đổi `jest.fn` → `vi.fn`, `jest.mock` → `vi.mock`, giữ `describe/test/expect` 90%.

- **Custom hook test**:
  - `renderHook(() => useCounter(0))` render hook trong component vô hình, trả `result.current`, `rerender`, `unmount`. Cần `act(() => result.current.inc())` và `vi.useFakeTimers` + `advanceTimersByTime` cho debounce.

## 3. Ví dụ tối thiểu

```tsx
// 3.1 RTL — getByRole > getByTestId, user-event, findBy cho async
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import ProductCard from './ProductCard'; // ({ product, onAdd }) => <article><h2>{p.name}</h2><button onClick={() => onAdd(p)}>Thêm</button></article>

test('RTL: user thấy heading + click gọi onAdd', async () => {
  const user = userEvent.setup(); // isolate mỗi test
  const onAdd = vi.fn();
  const product = { id: '1', name: 'Giày Nike', price: 1000 };

  render(<ProductCard product={product} onAdd={onAdd} />);

  // Ưu tiên getByRole — như user + screen reader tìm
  expect(screen.getByRole('heading', { name: /giày nike/i })).toBeInTheDocument();
  const btn = screen.getByRole('button', { name: /thêm/i });
  expect(btn).toBeInTheDocument();
  expect(btn).toBeEnabled();

  await user.click(btn);
  expect(onAdd).toHaveBeenCalledWith(product);
  expect(onAdd).toHaveBeenCalledTimes(1);
});

// ❌ Enzyme / implementation detail — giòn
// expect(wrapper.state().isOpen).toBe(true); // fail khi đổi useState → useReducer
// expect(screen.getByTestId('add-btn')).toBeInTheDocument(); // cuối cùng, nếu phải dùng là UI thiếu a11y

// Async + provider
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
function renderWithProviders(ui: React.ReactElement) {
  const qc = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  return render(<QueryClientProvider client={qc}>{ui}</QueryClientProvider>);
}

test('RTL async: loading rồi data', async () => {
  renderWithProviders(<ProductList />); // bên trong useQuery fetch /api/products
  expect(screen.getByText(/đang tải/i)).toBeInTheDocument();
  expect(await screen.findByText('Giày Nike')).toBeInTheDocument(); // findBy = getBy + waitFor
});

// Custom hook — renderHook + act + fake timers
import { renderHook, act, waitFor } from '@testing-library/react';
import { useCounter } from './useCounter';
import { useDebounce } from './useDebounce';

test('useCounter tăng giảm (renderHook)', () => {
  const { result } = renderHook(() => useCounter(0));
  expect(result.current.count).toBe(0);
  act(() => result.current.inc());
  expect(result.current.count).toBe(1);
});

test('useDebounce debounce đúng (fake timers)', () => {
  vi.useFakeTimers();
  const { result, rerender } = renderHook(({ v }) => useDebounce(v, 300), { initialProps: { v: 'a' } });
  expect(result.current).toBe('a');
  rerender({ v: 'ab' });
  expect(result.current).toBe('a'); // chưa qua delay
  act(() => vi.advanceTimersByTime(300));
  expect(result.current).toBe('ab');
  vi.useRealTimers();
});
```

```ts
// 3.2 MSW (network) vs vi.mock (module)
// ❌ vi.mock — mock module, giòn
vi.mock('./api', () => ({ fetchProducts: vi.fn(() => Promise.resolve([{ id: 1 }])) }));
// Vấn đề: đổi ./api → ./services/api mock hỏng; không test fetch thật

// ✅ MSW — mock network, bền, dùng chung dev/test
// mocks/handlers.ts
import { http, HttpResponse } from 'msw';
export const handlers = [
  http.get('/api/products', ({ request }) => {
    const url = new URL(request.url);
    const cat = url.searchParams.get('category');
    if (cat === 'shoes') return HttpResponse.json([{ id: 1, name: 'Giày' }]);
    return HttpResponse.json([]);
  }),
  http.post('/api/cart', async ({ request }) => {
    const body = await request.json() as any;
    return HttpResponse.json({ success: true, id: body.id }, { status: 201 });
  }),
  http.get('/api/products/:id', ({ params }) => HttpResponse.json({ id: params.id, name: 'Chi tiết' })),
];

// mocks/server.ts (Node cho Vitest)
import { setupServer } from 'msw/node';
import { handlers } from './handlers';
export const server = setupServer(...handlers);

// test/setup.ts
import '@testing-library/jest-dom/vitest';
beforeAll(() => server.listen({ onUnhandledRequest: 'error' })); // error nếu request không mock
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

// Trong test — override cho case lỗi
test('MSW override: hiển thị lỗi khi API 500', async () => {
  const { http, HttpResponse } = await import('msw');
  server.use(http.get('/api/products', () => HttpResponse.json({ message: 'Lỗi' }, { status: 500 })));
  renderWithProviders(<ProductList />);
  expect(await screen.findByText(/lỗi/i)).toBeInTheDocument();
});
```

```ts
// 3.3 Playwright (out-of-process) vs Cypress (in-process)
// Cypress — in-process, chainable, 1 tab
// cypress/e2e/checkout.cy.ts
describe('checkout (Cypress)', () => {
  it('login và mua', () => {
    cy.visit('/login');
    cy.get('[data-cy=email]').type('a@a.com'); // data-cy vì không a11y-first
    cy.get('[data-cy=submit]').click();
    cy.url().should('include', '/dashboard');
    cy.contains('Thêm vào giỏ').click();
  });
});

// Playwright — out-of-process, async/await, multi-tab, trace
// e2e/checkout.spec.ts
import { test, expect } from '@playwright/test';
test('checkout (Playwright) — a11y + multi-tab', async ({ page, context }) => {
  await page.goto('/login');
  await page.getByRole('textbox', { name: /email/i }).fill('a@a.com');
  await page.getByRole('button', { name: /đăng nhập/i }).click();
  await expect(page).toHaveURL(/dashboard/); // auto-wait
  await page.getByRole('button', { name: /thêm vào giỏ/i }).click();
  await expect(page.getByText(/đã thêm/i)).toBeVisible();

  // Multi-tab — Cypress không làm được
  const newPage = await context.newPage();
  await newPage.goto('/cart');
  await expect(newPage.getByRole('heading', { name: /giỏ hàng/i })).toBeVisible();
});

// playwright.config.ts
import { defineConfig } from '@playwright/test';
export default defineConfig({
  testDir: './e2e',
  workers: 4, // parallel free
  use: { trace: 'on-first-retry', screenshot: 'only-on-failure', video: 'retain-on-failure' },
  projects: [{ name: 'chromium' }, { name: 'firefox' }, { name: 'webkit' }], // 3 engine thật
});

// 3.4 Vitest (Vite-native) vs Jest
// vitest.config.ts — dùng chung vite.config alias
import { defineConfig } from 'vitest/config';
export default defineConfig({
  test: {
    environment: 'jsdom',
    globals: true, // không cần import describe/test
    setupFiles: ['./test/setup.ts'],
    coverage: { provider: 'v8', thresholds: { lines: 70, branches: 70 }, exclude: ['src/mocks/**', '**/*.stories.tsx'] },
  },
});
// setup.ts
import '@testing-library/jest-dom/vitest';

// Test viết y như Jest — chỉ đổi vi. vs jest.
import { describe, it, expect, vi } from 'vitest';
test('vi.fn mock', () => {
  const fn = vi.fn(() => 42);
  expect(fn()).toBe(42);
  expect(vi.isMockFunction(fn)).toBe(true);
});
// Jest tương đương: jest.fn(), jest.mock('./api'), jest.config.js riêng + babel-jest
```

## 4. So sánh / Phân loại

| Tiêu chí | `getByRole` (a11y) | `getByLabelText` | `getByPlaceholderText` | `getByText` | `getByTestId` |
|----------|-------------------|------------------|------------------------|-------------|---------------|
| **Ưu tiên** | **1 — cao nhất** | 2 | 3 | 4 | **6 — cuối cùng** |
| **Mô phỏng** | User + screen reader tìm button/heading | User tìm label của input | User thấy placeholder | User thấy text | Dev — user không thấy |
| **Bền** | Cao (đổi CSS không fail) | Cao | Trung bình | Cao | Thấp (testId đổi là fail) |
| **Khi dùng** | Mặc định | Form `label` | Khi không có label | Text generic | Chỉ khi không có role/label/text (canvas, icon thuần) |

| Tiêu chí | `userEvent` | `fireEvent` |
|----------|-------------|-------------|
| **Event sequence** | `pointerDown → mouseDown → focus → click` đầy đủ | 1 event thô |
| **Độ chính xác** | Như user thật (gõ từng ký tự, trigger đủ) | Nhanh nhưng thiếu event |
| **Tốc độ** | Chậm hơn | Nhanh hơn |
| **Khuyến nghị** | **Mặc định** (`userEvent.setup()` mỗi test) | Chỉ khi cần test low-level |

| Tiêu chí | MSW (network) | `vi.mock` / `jest.mock` (module) |
|----------|---------------|----------------------------------|
| **Mock tầng** | **Network** — intercept `fetch`/`XHR` | **Module** — thay `import` |
| **Refactor an toàn** | Cao (URL đổi mới hỏng) | Kém (path đổi là hỏng) |
| **Dùng chung dev/test/Storybook** | Có (`setupWorker` + `setupServer`) | Không |
| **Setup** | Cần `handlers` + `server` 1-2 file | Nhanh, 1 dòng |
| **Dùng khi** | **Integration** (mặc định), E2E `page.route` | **Unit cô lập**, third-party SDK (`next/router`, `firebase`) |

| Tiêu chí | Playwright (out-of-process) | Cypress (in-process) |
|----------|-----------------------------|----------------------|
| **Kiến trúc** | Node điều khiển browser qua **WebSocket/CDP**, ngoài browser | Chạy **trong browser** (iframe) |
| **Browser** | **Chromium + Firefox + WebKit** (3 engine thật) | Chromium chính, FF/WebKit muộn |
| **Multi-tab / multi-origin / iframe** | **Có** (`context.newPage()`, `page.frameLocator`) | Không (hạn chế `cy.origin`) |
| **Ngôn ngữ** | JS/TS, Python, Java, C# | JS/TS chính |
| **Parallel** | **Native `workers` free** | Cần Cypress Cloud trả phí |
| **Auto-wait / Trace** | **actionability checks** + **Trace Viewer** (DOM + network + timeline) | `cy.*` auto-retry + Time-Travel |
| **Phù hợp** | Team cần cross-browser, CI scale, multi-tab | Team nhỏ, cần DX nhanh, đã đầu tư Cypress Cloud |

| Tiêu chí | Vitest (Vite-native) | Jest (Babel) |
|----------|----------------------|--------------|
| **Transform** | **Vite/esbuild**, native ESM, HMR | `babel-jest`/`ts-jest`, cache riêng |
| **Tốc độ** | **Nhanh 3-10×**, cold start thấp | Chậm hơn, transform lại |
| **Config** | Dùng **`vite.config.ts`** (alias/plugin share) | `jest.config.js` riêng, duplicate |
| **ESM/TS/JSX** | Native | Cần flag `experimental-vm-modules` |
| **Mock** | `vi.fn`/`vi.mock` (alias `jest` được) | `jest.fn`/`jest.mock` |
| **Coverage** | `v8` (nhanh) | `v8`/`istanbul` |
| **Phù hợp** | **Vite, Next, modern FE** (default 2024-2026) | CRA, React Native, legacy Babel |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không `getByTestId` làm mặc định**: `getByTestId('submit-btn')` không đảm bảo a11y (button có thể không có accessible name). Chỉ dùng khi element không có role (canvas, `div` icon thuần). Nếu phải thêm `data-testid` để test pass, hãy thêm `aria-label`/`role` thật thay.
- **Không `fireEvent` cho user flow**: `fireEvent.change(input, { target: { value: 'a@a.com' } })` không bắn `focus`/`keydown` → component dùng `onFocus`/`onKeyDown` không được test. Dùng `userEvent.type` dù chậm hơn.
- **Không `vi.mock` cho Integration**: mock module làm test không phát hiện `fetch URL sai`, `header thiếu`. `vi.mock` chỉ cho **Unit** (`utils` gọi `localStorage`) hoặc **third-party SDK** (`vi.mock('next/navigation', () => ({ useRouter: vi.fn() }))`) không thể MSW.
- **Không MSW cho mọi E2E**: E2E happy path **không mock** để test BE thật; chỉ mock **payment gateway**/third-party. Với Playwright, dùng `page.route()` thay MSW worker.
- **Không Cypress khi cần multi-tab/cross-browser scale**: Cypress trade-off DX dễ vs scale kém (không multi-tab, parallel trả phí). Chọn Playwright khi cần `webkit` thật (iOS bug), `newPage()` (chat + dashboard), `workers: 4` free trong CI.
- **Không Jest cho dự án Vite mới**: Jest duplicate `tsconfig` alias, ESM chậm. Trừ **React Native** (Jest preset) hoặc legacy CRA, Vitest là default vì **không config 2 lần** và 3-10× nhanh.
- **Không `renderHook` cho hook gắn DOM**: `useClickOutside` cần DOM thật → test qua `TestComponent` có `ref`, không `renderHook` cô lập. `renderHook` chỉ cho hook logic thuần (`useDebounce`, `useCounter`).
- **Chi phí**: MSW + RTL + Playwright + Vitest thêm ~4 config file (`handlers.ts`, `server.ts`, `setup.ts`, `playwright.config.ts`) nhưng cho suite bền 70-80% coverage; không có thì suite giòn và flaky.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: `getByTestId` che a11y, refactor fail**
  - Triệu chứng: `screen.getByTestId('price')` pass nhưng user screen reader không tìm thấy giá (thiếu `role`/`aria`).
  - Fix: đổi `expect(screen.getByRole('heading', { name: /giá/i })).toBeInTheDocument()`; thêm `aria-label` cho icon button.
  - Đo: **eslint `testing-library/prefer-screen-queries` + `prefer-find-by`**; `axe` (`axe-playwright`) chạy a11y audit; `screen.logTestingPlaygroundURL()` gợi ý query tốt nhất.

- **Lỗi 2: `vi.mock` path sai, mock không có tác dụng**
  - Triệu chứng: `vi.mock('./api', ...)` nhưng component import `../services/api` → mock không chạm, test vẫn gọi thật.
  - Fix: chuyển sang MSW `http.get('/api/products', ...)` không phụ thuộc path; hoặc `vi.mock` đúng relative path.
  - Đo: `msw log` không thấy request bị chặn; `vitest --reporter=verbose` log `vi.mock` hoisted; `server.listen({ onUnhandledRequest: 'error' })` báo request không mock → phát hiện mock module sai.

- **Lỗi 3: MSW handler thiếu → `onUnhandledRequest` error / E2E flaky**
  - Triệu chứng: `Error: [MSW] Warning: intercepted a request without a matching handler`.
  - Fix: thêm handler cho `queryParams`/`method` đúng; `server.resetHandlers()` sau mỗi test để không rò rỉ override.
  - Đo: `msw/node` console warn; Playwright `trace viewer` → Network tab xem request 404; `page.route` log trong `playwright.config.ts` `use: { trace: 'on-first-retry' }` + `npx playwright show-trace trace.zip`.

- **Lỗi 4: Cypress multi-tab fail, Playwright pass**
  - Triệu chứng: `cy.visit('/cart')` mở tab mới fail `CypressError: cy.visit() failed trying to load`.
  - Fix: chuyển sang Playwright `const newPage = await context.newPage(); await newPage.goto('/cart')`.
  - Đo: Cypress error `cross origin`; Playwright `trace viewer` → `pages` pane thấy 2 page; `npx playwright test --project=webkit` phát hiện iOS-only bug.

- **Lỗi 5: Vitest ESM lỗi, Jest pass (hoặc ngược)**
  - Triệu chứng: `Error: require() of ES Module` với Jest, hoặc `Cannot use import statement` với ts-jest.
  - Fix: Vitest native ESM không lỗi; Jest cần `transformIgnorePatterns` + `extensionsToTreatAsEsm`.
  - Đo: `vitest run --coverage` vs `jest --coverage` timing (`vitest 3s vs jest 12s`); `vite --debug` log transform; `es-check` kiểm tra output.

- **Lỗi 6: `userEvent` thiếu `await` → assertion chạy trước click**
  - Triệu chứng: `user.click(btn); expect(onAdd).toHaveBeenCalledTimes(1)` fail vì click chưa resolve.
  - Fix: `await user.click(btn)`; luôn `userEvent.setup()` mới mỗi test để isolate.
  - Đo: Vitest log `Warning: An update to Component inside a test was not wrapped in act(...)`; `screen.debug()` sau `await`.

- **Công cụ đo**:
  - **`vitest --coverage --provider=v8` + `vitest --reporter=verbose`** — unit/integration speed, mock hit.
  - **`msw log` (`onUnhandledRequest: 'error'` + `server.printHandlers()`)** — request nào không mock, handler nào được dùng.
  - **`playwright trace viewer` (`trace: 'on-first-retry'`, `npx playwright show-trace`) + `screenshot: 'only-on-failure'`** — E2E debug visual, network, DOM snapshot per step.
  - **`@testing-library/jest-dom` + `eslint-plugin-testing-library`** — query priority, `findBy` vs `waitFor`.

## 7. Câu hỏi tự kiểm tra

1. Vì sao RTL ưu tiên `getByRole` + `userEvent` hơn `getByTestId` + `fireEvent`, và `findBy` khác `getBy` + `waitFor` thế nào?
2. MSW (network) vs `vi.mock` (module) khác tầng mock nào, refactor nào giòn hơn, và khi nào vẫn cần `vi.mock` dù đã có MSW?
3. Playwright out-of-process vs Cypress in-process khác kiến trúc và capability gì (multi-tab, 3 engine, parallel, trace), và Vitest Vite-native vs Jest khác transform/ESM/config thế nào?

<details>
<summary>Đáp án 30s</summary>

1. **`getByRole('button', { name })`** mô phỏng **user + screen reader** tìm element qua **a11y role/name** — nếu query này pass thì a11y cũng đúng; `getByTestId` user không thấy, đổi `data-testid` là fail dù UI đúng, che bug a11y. **`userEvent.click/type`** dispatch chuỗi đầy đủ `pointerDown → focus → click → keydown` như browser thật; `fireEvent.click` chỉ 1 event thô → miss `onFocus`/`onKeyDown`. **`findBy`** = `getBy` + **`waitFor` polling 50ms** tới 1000ms, dùng cho **async** (`fetch` xong mới có text); `getBy` throw ngay nếu không thấy, `findBy` await.
2. **MSW mock `network layer`** (intercept `fetch`/`XHR` thật qua Service Worker/`http` server) — app gọi `fetch('/api/products')` thật, MSW trả mock; **không mock code**. **`vi.mock` mock `module layer`** (thay `import * as api`). **MSW bền hơn**: đổi `import path` (`./api → ./services/api`) MSW vẫn pass (chỉ URL đổi mới hỏng), `vi.mock` hỏng ngay. MSW **dùng chung dev/test/Storybook**, `vi.mock` không. Vẫn cần `vi.mock` cho **Unit cô lập** hoặc **third-party SDK** không qua network (`next/router`, `firebase auth`, `localStorage`) hoặc mock `Date.now`/`crypto.randomUUID`.
3. **Playwright out-of-process**: Node process điều khiển **Chromium/Firefox/WebKit thật** qua **WebSocket/CDP**, hỗ trợ **multi-tab (`context.newPage()`), multi-origin, iframe, file download**, **parallel `workers:4` free**, **auto-wait actionability** (visible/stable/enabled), **Trace Viewer** (DOM snapshot + network + timeline). **Cypress in-process**: chạy **trong browser iframe**, `cy.*` chainable + time-travel DX dễ, nhưng **không multi-tab**, chủ yếu **Chromium** (FF/WebKit muộn), **parallel cần Cloud trả phí**. **Vitest Vite-native**: dùng **Vite/esbuild transform**, **native ESM/TS/JSX**, **share `vite.config.ts` alias**, **HMR**, nhanh **3-10×** Jest; **Jest** dùng **babel-jest/ts-jest**, `jest.config.js` riêng, ESM cần flag, cold start chậm — chỉ giữ cho **React Native/legacy CRA**.

</details>

---
*Tham khảo chi tiết: `docs/10-testing.md` — Câu 163 (RTL), 164 (Jest/Vitest), 165-166 (implementation detail), 167 (MSW), 168 (Playwright/Cypress), 170 (renderHook). Spec: [RTL — Queries Priority](https://testing-library.com/docs/queries/about/#priority), [MSW — Network vs Mock](https://mswjs.io/docs/comparison), [Playwright — Trace Viewer](https://playwright.dev/docs/trace-viewer), [Vitest — Why Vitest](https://vitest.dev/guide/why).*
