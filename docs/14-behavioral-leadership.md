# 14. Behavioral & Leadership - 13 Câu Hỏi Senior

> 13 câu hỏi hành vi và leadership (Câu 187-199) dành cho Senior Frontend. Nhà tuyển dụng không chỉ hỏi code mà hỏi **cách bạn làm việc với con người, ra quyết định và chịu trách nhiệm**. Mỗi câu dùng framework **STAR (Situation - Task - Action - Result)** + trade-off + ví dụ thực tế.

## Mục lục

- [Câu 187: Review code Junior - làm sao vừa nghiêm vừa không làm nản?](#câu-187-review-code-junior---làm-sao-vừa-nghiêm-vừa-không-làm-nản)
- [Câu 188: Junior không đồng ý với review của bạn](#câu-188-junior-không-đồng-ý-với-review-của-bạn)
- [Câu 189: Hai Senior có giải pháp khác nhau - bạn xử lý thế nào?](#câu-189-hai-senior-có-giải-pháp-khác-nhau---bạn-xử-lý-thế-nào)
- [Câu 190: Khi nào chấp nhận Technical Debt?](#câu-190-khi-nào-chấp-nhận-technical-debt)
- [Câu 191: Deadline gấp mà code chưa tốt - ship hay delay?](#câu-191-deadline-gấp-mà-code-chưa-tốt---ship-hay-delay)
- [Câu 192: Kể về một quyết định sai của bạn](#câu-192-kể-về-một-quyết-định-sai-của-bạn)
- [Câu 193: Thuyết phục team đổi architecture / tech stack](#câu-193-thuyết-phục-team-đổi-architecture--tech-stack)
- [Câu 194: PM yêu cầu feature bạn cho là không nên làm](#câu-194-pm-yêu-cầu-feature-bạn-cho-là-không-nên-làm)
- [Câu 195: Backend không đồng ý với API design bạn đề xuất](#câu-195-backend-không-đồng-ý-với-api-design-bạn-đề-xuất)
- [Câu 196: Production incident - bạn làm gì trong 30 phút đầu?](#câu-196-production-incident---bạn-làm-gì-trong-30-phút-đầu)
- [Câu 197: Mentor Junior - cách bạn giúp họ grow?](#câu-197-mentor-junior---cách-bạn-giúp-họ-grow)
- [Câu 198: Phân chia task trong team - làm sao công bằng và hiệu quả?](#câu-198-phân-chia-task-trong-team---làm-sao-công-bằng-và-hiệu-quả)
- [Câu 199: Khi nào tự code vs delegate cho người khác?](#câu-199-khi-nào-tự-code-vs-delegate-cho-người-khác)


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 187: Review code Junior - làm sao vừa nghiêm vừa không làm nản?

**Gợi ý trả lời Senior:**

> **STAR:** *Situation* - Junior gửi PR 800 dòng, logic đúng nhưng đặt tên kém, không test, copy-paste. *Task* - Mình cần đảm bảo chất lượng mà vẫn giữ động lực. *Action* - Mình chia review thành 3 tầng: **Must fix** (bug, security), **Should fix** (naming, complexity), **Nit** (style). Mỗi comment đều kèm **vì sao** và **gợi ý code**, không chỉ "sai rồi". Khen phần làm tốt trước, hỏi thay vì ra lệnh ("Mình nghĩ tách hàm này sẽ dễ test hơn, em thấy sao?"). Nếu quá nhiều vấn đề thì call 15 phút thay vì 30 comment. *Result* - Junior fix nhanh, hiểu lý do, PR sau tốt hơn 50%.

**Trade-off:** Review quá nghiêm → Junior nản, review quá dễ → nợ kỹ thuật. Cân bằng bằng **bar rõ ràng** (checklist team) và **tone tôn trọng**.

**Ví dụ thực tế:**

```markdown
❌ "Code này tệ, viết lại đi"
✅ "Hàm này đang làm 2 việc (fetch + format). Mình tách thành `fetchUser` và `formatUser` thì test dễ hơn, em thử nhé? Ví dụ: ..."
```

**Tips:** Dùng Conventional Comments (`nit:`, `suggestion:`, `issue:`), bật `CODEOWNERS`, auto-format/lint để không tranh cãi style.


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 188: Junior không đồng ý với review của bạn

**Gợi ý trả lời Senior:**

> **STAR:** *Situation* - Mình request change về việc dùng `useMemo` không cần thiết, Junior cho rằng cần để tối ưu. *Task* - Giải quyết bất đồng không thành cãi nhau. *Action* - Mình 1) Lắng nghe lý do của bạn trước, 2) Đưa **data** (profiler cho thấy không cải thiện), 3) Đặt **tiêu chí chung** (team convention, performance budget), 4) Nếu vẫn bất đồng → đề xuất **thử 2 cách, đo rồi quyết** hoặc nhờ **third opinion** từ Senior khác/tech lead. Không dùng quyền lực để ép. *Result* - Thống nhất bỏ `useMemo` thừa, Junior hiểu khi nào cần memo, team có thêm quy tắc rõ.

**Trade-off:** Ép theo ý mình → mất trust. Nhường vô nguyên tắc → codebase loạn. Senior chọn **đo + quy tắc chung**.

**Câu nói mẫu:** "Mình hiểu góc của em. Mình cùng đo profiler 1 phút nhé, nếu không khác thì mình giữ code đơn giản hơn, em thấy ok không?"


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 189: Hai Senior có giải pháp khác nhau - bạn xử lý thế nào?

**Gợi ý trả lời Senior:**

> **STAR:** *Situation* - Mình và một Senior khác tranh luận: mình muốn `Zustand + TanStack Query`, bạn kia muốn `Redux Toolkit`. *Task* - Chọn giải pháp tốt nhất cho team, không phải cho ego. *Action* - Mình tổ chức **ADR (Architecture Decision Record)**: liệt kê **context, options, trade-off, decision, consequence**. Mỗi option chấm theo tiêu chí: độ phức tạp, learning curve, bundle size, DX, khả năng hire. Viết RFC ngắn, cho team comment 2 ngày, rồi quyết theo **consensus hoặc lead quyết**. Dù kết quả là gì, mình **commit 100%**. *Result* - Team chọn Zustand+Query, có ADR lưu lại để 6 tháng sau không tranh lại.

**Trade-off:** Không có giải pháp hoàn hảo, chỉ có phù hợp context. Senior không thắng tranh luận mà **giúp team ra quyết định có cơ sở**.

**Template ADR:**

```markdown
# ADR-05: State Management
- Context: App 50k DAU, team 5 FE, cần cache server state
- Options: Redux Toolkit vs Zustand+Query
- Decision: Zustand+Query (ít boilerplate, đúng loại state)
- Consequence: 2 lib, cần guideline khi nào dùng store vs query
```


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 190: Khi nào chấp nhận Technical Debt?

**Gợi ý trả lời Senior:**

> **Trả lời:** Technical debt không phải lúc nào cũng xấu — nó là **vay để ship nhanh**, miễn là **có kế hoạch trả**. Mình chấp nhận debt khi: 1) **Deadline business** cứng (ra mắt trước Tết), 2) **Chưa rõ yêu cầu** (làm MVP validate, không over-engineer), 3) **Debt cô lập** (chỉ 1 module, không lan). Mình **không chấp nhận** debt ở **core** (checkout, auth, payment) hoặc debt làm **tăng bug production**.

**Cách quản lý debt:**

```typescript
// 1. Đánh dấu rõ
// TODO(debt): hardcode fee, refactor khi có API - ticket JIRA-123 - deadline 2026-03
const fee = 30000;

// 2. Track trong backlog, gắn label `tech-debt`, ưu tiên mỗi sprint 20%
// 3. Viết ADR ghi vì sao vay, khi nào trả
```

**STAR ví dụ:** "Dự án promo Tết, BE chưa có API tính phí. Mình hardcode phí 30k để kịp deadline, tạo ticket refactor, và 2 tuần sau khi BE xong thì fix. Nếu không vay thì lỡ mùa cao điểm."


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 191: Deadline gấp mà code chưa tốt - ship hay delay?

**Gợi ý trả lời Senior:**

> **Framework:** Mình đánh giá theo **ma trận Impact vs Risk**:
> - **High impact + Low risk** (debt cô lập, có test) → **Ship**, tạo ticket trả nợ ngay sprint sau
> - **High risk** (payment, security, data loss) → **Không ship**, thương lượng scope hoặc deadline
> - Luôn **nói rõ với PM/Stakeholder** trade-off: "Ship kịp nhưng sẽ tốn 2 ngày refactor sau, hoặc delay 1 ngày để làm đúng. Anh chọn?"

**STAR:** "Feature coupon deadline Black Friday, code còn thiếu test e2e. Mình ship với feature flag 10% user, monitor 24h, và viết test ngay sau. Nếu bug thì tắt flag, không ảnh hưởng 90% còn lại."

**Tips:** Đừng im lặng ship code xấu — **communicate**, **flag**, **monitor**.


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 192: Kể về một quyết định sai của bạn

**Gợi ý trả lời Senior:**

> **STAR:** *Situation* - Mình từng chọn `Redux + redux-saga` cho app nhỏ 10 trang vì "chuẩn enterprise". *Task* - Cần state đơn giản. *Action* - Mình over-engineer, boilerplate nhiều, Junior mất 2 tuần onboarding. *Result* - Velocity chậm, bug saga khó debug. *Lesson* - Mình migrate sang Zustand+Query trong 1 sprint, velocity tăng 30%. Bài học: **chọn boring solution**, **phù hợp scale hiện tại**, không phải scale tưởng tượng. Giờ mình luôn hỏi "Nếu app chỉ 10 trang, cần Redux không?" trước khi quyết.

**Cách kể hay:** Thành thật, nói **mình đã sửa thế nào** và **quy tắc mới** để không lặp lại. Nhà tuyển dụng đánh giá **self-awareness** hơn là không sai bao giờ.


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 193: Thuyết phục team đổi architecture / tech stack

**Gợi ý trả lời Senior:**

> **Framework:** 1) **Đo pain hiện tại** (bundle 1.5MB, build 3 phút, bug 5/tuần) → 2) **PoC nhỏ** (migrate 1 feature, đo trước/sau) → 3) **RFC + số liệu** (cost, lợi, risk, rollback) → 4) **Pilot** (1 team/1 route) → 5) **Retro** rồi nhân rộng. Không "đập đi xây lại".

**STAR:** "Mình muốn đổi từ CRA sang Vite. Mình đo: build CRA 180s, Vite 25s. Làm PoC migrate 1 feature, demo cho team, viết RFC, team đồng ý pilot 1 route, sau 2 tuần build nhanh 7x, team vote migrate toàn bộ."

**Trade-off:** Đổi stack tốn thời gian, risk regression. Phải chứng minh **ROI rõ ràng** (ví dụ tiết kiệm 10h build/tháng).


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 194: PM yêu cầu feature bạn cho là không nên làm

**Gợi ý trả lời Senior:**

> **Cách xử lý:** 1) **Hiểu WHY** trước khi nói NO — PM có data user/business mình chưa thấy. 2) **Đưa alternative** dựa trên data/tech: "Feature X tốn 2 tuần, nhưng dùng Y có 80% giá trị mà chỉ 3 ngày". 3) **Nói bằng ngôn ngữ business**: cost, risk, opportunity cost, chứ không phải "code xấu". 4) Nếu vẫn phải làm → **làm MVP + đo**, không làm perfect.

**Ví dụ:** PM muốn "animation 3D cho homepage". Mình: "3D tốn 2 tuần, tăng LCP 1s, chỉ 5% user thấy. Mình đề xuất parallax nhẹ 2 ngày, A/B test, nếu CTR tăng mới đầu tư 3D. PM đồng ý."

**Mẫu câu:** "Mình hiểu mục tiêu là tăng conversion. Mình lo là cách này sẽ làm chậm trang và tốn 2 tuần. Mình đề xuất cách B nhanh hơn, mình cùng thử nhé?"


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 195: Backend không đồng ý với API design bạn đề xuất

**Gợi ý trả lời Senior:**

> **Nguyên tắc:** FE/BE là **cùng team**, không phải đối đầu. Mình 1) **Chuẩn bị trước**: viết **API proposal** (endpoint, request/response example, pagination, error format) dựa trên **use case FE** (cần gì, render thế nào). 2) **Lắng nghe constraint BE**: DB, performance, breaking change. 3) **Tìm win-win**: BE muốn REST thuần, FE muốn `include` để tránh waterfall → thỏa hiệp `expand` param hoặc BFF. 4) **Dùng spec chung**: OpenAPI, thống nhất trước khi code.

**STAR:** "Mình cần `GET /products?include=inventory,price` để tránh 3 request. BE lo query nặng. Mình đề xuất `GET /api/bff/products` (BFF gom) hoặc `?expand=inventory` optional, BE ok vì không ảnh hưởng client khác. Viết OpenAPI, generate type, cả 2 cùng happy."

**Trade-off:** Đừng đòi API "chuẩn REST" khi FE cần performance — hãy nói về **user impact**.


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 196: Production incident - bạn làm gì trong 30 phút đầu?

**Gợi ý trả lời Senior:**

> **Quy trình 30 phút:**
> 1. **0-5'**: Xác nhận incident (Sentry alert, user report), đánh giá **severity** (bao nhiêu % user, có mất tiền không), **call** on-call/BE/PM
> 2. **5-10'**: **Mitigate** trước, **fix** sau — tắt feature flag, rollback deploy, scale, chứ không debug sâu
> 3. **10-20'**: Thu thập **log, release, diff**, tìm root cause sơ bộ, cập nhật **status page** cho stakeholder
> 4. **20-30'**: Fix minimal hoặc rollback, deploy hotfix, monitor

**STAR:** "Checkout 500 sau deploy, 20% user không đặt được. Mình tắt flag `new-checkout` trong 2 phút → traffic về flow cũ, error giảm. Sau đó `git diff` thấy BE đổi field `total` thành `totalAmount`, FE chưa update. Fix 1 dòng, deploy, monitor 1h. Hôm sau viết post-mortem."

**Post-mortem template:** Timeline, Root cause, Impact, Action items (test, contract, flag).

**Câu nói phỏng vấn:** "Mình ưu tiên **giảm thiệt hại** trước, **hiểu nguyên nhân** sau. Không fix lớn khi đang cháy."


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 197: Mentor Junior - cách bạn giúp họ grow?

**Gợi ý trả lời Senior:**

> **Framework:** 1) **Đánh giá level** (mới ra trường vs 1 năm), 2) **Giao task vừa sức + stretch** (70% làm được, 30% thử thách), 3) **Pair + Review** (pair 1h đầu, review chi tiết, giải thích why), 4) **Cho ownership** (1 feature nhỏ end-to-end), 5) **Feedback thường xuyên** (weekly 1-1, khen/chê cụ thể), 6) **Dạy cách học** (đọc doc, debug, hỏi đúng cách).

**Ví dụ:** "Junior mới vào, mình cho làm `ProductCard` → `ProductList` → `Search` (tăng dần). Mỗi PR mình review kèm link doc, gợi ý 1 cuốn sách. Sau 3 tháng bạn tự lead feature nhỏ."

**Anti-pattern:** Giao task quá khó → nản, giao quá dễ → chán, chỉ chê không khen → mất động lực.


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 198: Phân chia task trong team - làm sao công bằng và hiệu quả?

**Gợi ý trả lời Senior:**

> **Nguyên tắc:** 1) **Skill + Interest + Growth** — ai giỏi gì làm nấy, nhưng luân phiên để ai cũng học. 2) **Độ phức tạp** — task khó cho Senior + Junior pair, task dễ cho Junior solo. 3) **Minh bạch** — planning poker, mọi người tự pick sau khi thảo luận. 4) **Buffer** — chừa 20% cho bug, review, học.

**Cách làm:** Sprint planning: list task, estimate story point, phân theo **capacity** không phải **số task**. Dùng **RACI** cho task lớn. Daily check blocker, không micro-manage.

**Ví dụ:** "Sprint có 5 task: 2 khó (search, payment), 3 vừa. Mình nhận payment (risk cao), Junior A làm search pair với mình, Junior B làm 2 task vừa + 1 task học (viết test)."


[↑ Quay lại Mục lục](#mục-lục)

---

### Câu 199: Khi nào tự code vs delegate cho người khác?

**Gợi ý trả lời Senior:**

> **Ma trận quyết định:**

| Tình huống | Tự code | Delegate |
|---|---|---|
| Task **critical, risk cao**, cần kinh nghiệm | ✅ |  |
| Task **cần làm nhanh** để unblock team | ✅ |  |
| Task **vừa sức Junior** + cơ hội học |  | ✅ |
| Task **lặp lại**, có thể scale |  | ✅ (kèm guideline) |
| Bạn đang **quá tải**, team rảnh |  | ✅ |
| Cần **xây trust / growth** cho member |  | ✅ |

**Nguyên tắc Senior:** **Delegate không phải vứt việc** — phải **rõ outcome, deadline, support**. Check-in giữa chừng, không đợi fail mới hỏi. Và **delegate cả ownership**, không chỉ task.

**STAR:** "Mình từng ôm hết feature checkout vì 'làm nhanh hơn'. Kết quả mình burnout, Junior không grow. Sau đó mình delegate phần UI cho Junior, mình làm logic payment + review, velocity team tăng, mình có thời gian làm architecture."

**Câu kết phỏng vấn:** "Senior giỏi không phải code nhiều nhất, mà là **giúp team code nhiều và tốt hơn**."

