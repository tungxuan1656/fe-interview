# 20. Frontend Platform Engineering

> Phần dành cho Senior có ownership vượt khỏi một application: build, release, shared infrastructure, multi-tenant và developer experience.

## 1. Monorepo

### Interview questions

- Khi nào monorepo đáng dùng?
- Package boundary nên được enforce thế nào?
- Làm sao tránh shared package trở thành dumping ground?
- Turborepo/Nx giải quyết vấn đề gì và không giải quyết vấn đề gì?
- CI chỉ chạy package bị ảnh hưởng thế nào?

### Senior signals

Không chọn monorepo chỉ vì "nhiều app". Hãy xem team ownership, dependency graph, release coupling, shared code và developer workflow.

---

## 2. Build system

Các chủ đề nên hiểu:

- Vite và bundling pipeline
- ESM/CJS
- tree-shaking
- code splitting
- source maps
- dependency duplication
- environment configuration
- build caching
- artifact reproducibility

### Trap

> Bundle nhỏ hơn có luôn đồng nghĩa ứng dụng nhanh hơn không?

Không. Cần xem network, cache hit, parsing/compilation, execution, main-thread work và user-centric metrics.

---

## 3. CI/CD

Một frontend pipeline tốt nên có:

```text
Commit
 ↓
Lint / Typecheck
 ↓
Unit / Integration tests
 ↓
Build immutable artifact
 ↓
Preview / Staging
 ↓
Smoke test
 ↓
Promote artifact
 ↓
Production
 ↓
Observe
```

### Interview questions

- Build lại artifact khi promote production hay dùng lại artifact staging?
- Rollback frontend thế nào?
- Feature flag khác deployment rollback ra sao?
- Làm sao trace production artifact về commit?

---

## 4. Multi-tenant / White-label

### Problem

Nhiều tenant có:

- branding khác nhau
- feature capability khác nhau
- API/integration khác nhau
- deployment/release cadence khác nhau

### Thiết kế

Tách:

```text
Core domain logic
        +
Tenant configuration
        +
Capability flags
        +
Integration adapters
        +
Brand/theme layer
```

Tránh conditional kiểu:

```ts
if (tenant === 'A') {}
if (tenant === 'B') {}
if (tenant === 'C') {}
```

ở mọi tầng của application.

### Follow-up

- Khi nào fork codebase hợp lý?
- Tenant có custom feature riêng thì boundary ở đâu?
- Config validation làm thế nào?
- Làm sao test hàng chục tenant mà CI không tăng tuyến tính?

---

## 5. Release engineering

Senior nên biết cách thiết kế:

- versioning
- release channels
- staged rollout
- canary
- rollback
- feature flags
- artifact metadata
- changelog
- release audit trail

### Failure mode

> Deployment thành công nhưng browser vẫn chạy chunk cũ.

Cần nghĩ đến CDN/cache headers, asset naming, immutable assets, HTML caching và compatibility giữa HTML version với JS chunks.

---

## 6. Observability

Frontend observability không chỉ là console.log.

Nên phân biệt:

- logs
- errors
- traces
- performance metrics
- product metrics
- release health

Các metric nên gắn với release/version và segment phù hợp: browser, OS, route, tenant hoặc feature.

---

## 7. Developer Experience

Platform engineering tốt phải giảm cognitive load cho team.

Ví dụ:

- một command để tạo feature chuẩn
- shared lint/type/test config
- reusable CI workflow
- preview environment
- standard error handling
- standard observability
- documented release process

### Senior question

> Làm sao chứng minh platform team tạo ra giá trị?

Không chỉ bằng số package. Hãy đo lead time, CI duration, failure rate, onboarding time, deployment frequency và developer friction.

---

## 8. Staff-level discussion

Khi hệ thống lớn, câu hỏi không còn là:

> "Tool nào tốt nhất?"

mà là:

> "Boundary nào giúp tổ chức thay đổi hệ thống nhanh và an toàn nhất?"

Đó là lý do platform architecture cần được đánh giá cùng **team topology, ownership, release process và business constraints**.
