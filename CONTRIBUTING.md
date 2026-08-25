# Đóng góp — Senior Frontend Interview

Cảm ơn bạn muốn đóng góp cho repo 199 câu hỏi này!

## Cách thêm câu hỏi / đáp án

1. Fork repo → tạo branch `feat/cau-xxx` hoặc `fix/...`
2. Xác định chương phù hợp trong `docs/01-16` (ví dụ performance → `docs/05-performance.md`)
3. Thêm câu theo **format chuẩn** của chương:

```markdown
### Câu 105: Tiêu đề câu hỏi?

**Trả lời Senior:**
Nội dung trả lời: cơ chế → trade-off → ví dụ thực tế.

\`\`\`typescript
// code ví dụ
\`\`\`

**Trade-off:** ...

**Câu hỏi đào sâu:** ...
```

- Với chương `13-debugging` dùng format `### Tình huống X: ...` + Triệu chứng → Checklist → Tool → Fix → Phòng ngừa
- Với `14-behavioral` dùng format `### Câu 187: ...` + **Gợi ý trả lời Senior:** STAR
- Với `15-senior-trap` dùng format câu hỏi bẫy → Junior sai → Senior đúng

4. Cập nhật **Mục lục** đầu file và `README.md` (bảng tổng quan) nếu thêm chương mới
5. Đảm bảo tiếng Việt, giọng Senior, không dịch máy, code chạy được

## Format Markdown

- H1 cho tiêu đề chương, H2 cho Mục lục, H3 cho từng câu/tình huống
- Dùng `> blockquote` cho mô tả chương
- Bảng markdown chuẩn `| A | B |`
- Code block ghi rõ ngôn ngữ `typescript` / `html` / `bash`
- Anchor TOC khớp với heading (lowercase, bỏ dấu)

## PR Flow

1. `git commit -m "feat(docs): thêm câu 200 về ..."` (conventional commits)
2. Push và tạo Pull Request — mô tả: chương nào, câu nào, nguồn tham khảo (nếu có)
3. Maintainer review trong 2-3 ngày — có thể yêu cầu bổ sung trade-off / ví dụ
4. Merge sau khi pass review — squash merge

## Quy tắc

- Không copy nguyên văn từ blog/cuộc phỏng vấn có bản quyền — hãy diễn đạt lại theo góc Senior
- Ưu tiên ví dụ thực tế React/Next.js/TypeScript
- Mỗi PR nên tập trung 1 chương hoặc 1 chủ đề

> Mọi đóng góp dù nhỏ (sửa typo, thêm ví dụ, cải thiện bảng) đều được trân trọng!
