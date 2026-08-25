# 18. Production Scenarios

> Các bài này dùng để chuyển từ **biết kiến thức** sang **ra quyết định trong production**.

## Scenario 01 — Dashboard realtime bị lag

### Context

Một operational dashboard nhận event realtime liên tục. Khi số lượng event tăng, CPU browser cao và thao tác filter/input bị giật.

### Interviewer hỏi

- Bạn sẽ xác định bottleneck ở đâu?
- WebSocket có phải nguyên nhân duy nhất không?
- Bạn xử lý batching, throttling hoặc buffering thế nào?
- Khi nào dùng `startTransition`?
- Bạn đo kết quả bằng metric nào?

### Strong answer nên chạm tới

- Đo trước bằng Performance Profiler và React Profiler.
- Phân biệt network throughput với render throughput.
- Kiểm tra frequency của state updates và số component bị invalidated.
- Batch/coalesce events trước khi cập nhật UI nếu business cho phép.
- Tách data ingestion khỏi rendering.
- Chỉ dùng virtualization khi bottleneck thực sự nằm ở DOM/list rendering.
- Dùng transition cho non-urgent UI work, không dùng nó để che giấu một data pipeline tệ.

---

## Scenario 02 — Production white screen sau deployment

### Context

Sau release mới, một phần user mở web chỉ thấy màn hình trắng.

### Interviewer hỏi

- Bạn triage trong 5 phút đầu thế nào?
- Làm sao phân biệt JS exception, chunk loading error và API failure?
- Rollback hay hotfix?
- Làm sao ngăn incident lặp lại?

### Strong answer nên chạm tới

```text
Impact
  ↓
Reproduce
  ↓
Error monitoring
  ↓
Console / Network
  ↓
Release correlation
  ↓
Rollback or mitigate
  ↓
Root cause
  ↓
Regression protection
```

Cần xem error rate theo release/version/browser, chunk URL, cache behavior và source map. Nếu impact lớn và nguyên nhân chưa rõ, rollback là một mitigation hợp lệ; không nên cố debug production trong khi user vẫn đang bị ảnh hưởng.

---

## Scenario 03 — Search bị race condition

### Context

User gõ `react`, sau đó nhanh chóng đổi thành `react query`. Response của request cũ đôi khi về sau và ghi đè kết quả mới.

### Interviewer hỏi

- Vì sao race condition xảy ra?
- Abort request có đủ không?
- Bạn xử lý ở UI, data layer hay cả hai?

### Strong answer nên chạm tới

- Debounce để giảm request không cần thiết.
- AbortController để cancel request không còn cần.
- Data layer nên bảo vệ khỏi stale response, không chỉ dựa vào UI timing.
- Có thể dùng request key/version để chỉ accept response hiện tại.
- Nếu dùng query library, tận dụng query key và cancellation semantics thay vì tự xây cache riêng.

---

## Scenario 04 — 500 component cần refactor

### Context

Một frontend lớn có component dùng chung nhưng coupling cao. Team muốn chuyển sang feature/domain boundaries.

### Interviewer hỏi

- Bạn có rewrite toàn bộ không?
- Tiêu chí chọn component nào migrate trước?
- Làm sao tránh phá production?

### Strong answer nên chạm tới

Không rewrite big-bang. Trước tiên xác định dependency graph, ownership và change frequency. Chọn một vertical slice có business boundary rõ, tạo target architecture, migrate từng phần và giữ compatibility layer khi cần.

Success metric nên bao gồm defect rate, lead time, build/test time, ownership clarity và tốc độ thay đổi — không chỉ số lượng file đã move.

---

## Scenario 05 — Bundle tăng mạnh

### Context

Sau vài tháng, JavaScript transferred tăng đáng kể và LCP/INP xấu đi.

### Interviewer hỏi

- Bạn tìm nguyên nhân thế nào?
- Có phải cứ code splitting là tốt?
- Khi nào tree-shaking không hiệu quả?

### Strong answer nên chạm tới

- So sánh bundle trước/sau theo release.
- Phân tích dependency graph và source map.
- Kiểm tra duplicate dependencies, barrel exports, dynamic imports và package formats.
- Tách critical path khỏi code ít dùng.
- Không tạo quá nhiều chunk nhỏ nếu request/overhead và caching bị ảnh hưởng.
- Đo real-user metrics sau rollout thay vì chỉ nhìn bundle size.

---

## Scenario 06 — Multi-tenant frontend

### Context

Một sản phẩm được triển khai cho nhiều partner với branding, feature set và cấu hình khác nhau.

### Interviewer hỏi

- Bạn fork codebase cho từng partner hay dùng một codebase?
- Tenant configuration nằm ở đâu?
- Làm sao tránh `if (tenant === ...)` lan khắp codebase?

### Strong answer nên chạm tới

Tách **domain logic** khỏi **tenant configuration**. Dùng configuration/schema rõ ràng cho branding, capabilities và integrations; boundary bằng adapter/strategy thay vì conditional rải rác. Nếu tenant có behavior khác biệt rất lớn, đánh giá lại boundary thay vì cố nhồi tất cả vào một abstraction.

---

## Scenario 07 — CI/CD release failure

### Context

Một release pipeline build nhiều white-label applications. Một thay đổi nhỏ làm một số app build được, số khác fail.

### Interviewer hỏi

- Bạn thiết kế pipeline và dependency isolation thế nào?
- Làm sao retry an toàn?
- Làm sao biết artifact nào đã được release?
- Rollback strategy là gì?

### Strong answer nên chạm tới

- Reproducible build.
- Lockfile và version pinning phù hợp.
- Artifact immutable.
- Tách build/configuration khỏi source logic.
- Build matrix theo tenant/platform.
- Record commit SHA, app version và artifact metadata.
- Promotion cùng một artifact thay vì rebuild khi deploy.

---

## Scenario 08 — API chậm nhưng frontend bị đổ lỗi

### Context

User phàn nàn trang chậm. Backend cho rằng API latency ổn.

### Interviewer hỏi

- Bạn chứng minh bottleneck ở đâu?
- Network time khác frontend processing time thế nào?

### Strong answer nên chạm tới

Phân rã latency thành DNS/connection/TLS/request/server/response parsing/main-thread/rendering. Dùng browser Performance và Network waterfall cùng backend timing. Không kết luận dựa trên một metric duy nhất.

---

## Scenario 09 — Shared state trở thành God Store

### Context

Một Zustand/Redux store chứa gần như mọi state của ứng dụng.

### Interviewer hỏi

- Vấn đề thực sự là library hay state ownership?
- Bạn refactor thế nào?

### Strong answer nên chạm tới

Phân loại lại state theo lifetime và ownership. Server state đưa về data-fetching layer; URL state giữ trong URL; local UI state giữ gần component; shared client state chỉ tồn tại khi nhiều domain thực sự cần. Refactor theo domain boundary thay vì chỉ chia một store thành nhiều file.

---

## Scenario 10 — Security incident

### Context

Một URL parameter được render vào UI và security team phát hiện khả năng XSS.

### Interviewer hỏi

- Bạn xác định attack surface thế nào?
- Encode, sanitize và CSP khác nhau ra sao?
- Nếu đã có production vulnerability thì rollout thế nào?

### Strong answer nên chạm tới

Xác định source → transformation → sink, ưu tiên bỏ unsafe HTML nếu không cần. Dùng contextual escaping/sanitization phù hợp, CSP như defense-in-depth chứ không thay thế secure coding. Với incident production: mitigate, patch, verify, audit các path tương tự và bổ sung regression/security tests.
