# Bài 04 – Tách instructions ra nhiều file

> Bài gốc: [EN](../docs/en/lectures/lecture-04-why-one-giant-instruction-file-fails/index.md) · [VI](../docs/vi/lectures/lecture-04-why-one-giant-instruction-file-fails/index.md)
> Code ví dụ: `.../lecture-04-.../code/` (`AGENTS-short.md`, `anti-patterns.md`, `split-vs-monolithic.ts`)
> Project thực hành: **P02** (tiếp)
> Milestone DocVault: **M3: AGENTS.md thành router + topic docs (IPC, RAG, testing)**

---

## 1. Ý chính trong 30 giây

Cứ mỗi lần agent sai lại "thêm một rule" thì `AGENTS.md` phình lên 600 dòng. Khi đó context bị ăn mòn, rule quan trọng nằm giữa file bị bỏ qua (**lost in the middle**), các rule mâu thuẫn nhau. Cách sửa: **entry file ngắn (50–200 dòng) đóng vai router**, chi tiết tách ra topic docs và chỉ đọc khi cần (**progressive disclosure**).

## 2. Năm vấn đề của file instruction khổng lồ

1. **Context budget bị ăn**: 600 dòng ≈ 10–20k tokens, chiếm chỗ của việc đọc code.
2. **Lost in the middle**: LLM dùng thông tin ở giữa văn bản dài kém hơn so với ở đầu và cuối (Liu et al., 2023).
3. **Không phân biệt được ưu tiên**: "không dùng eval()" và "nên dùng functional style" trông giống hệt nhau.
4. **Maintenance decay**: thêm thì dễ, xoá thì sợ, nên file chỉ phình ra.
5. **Mâu thuẫn tích tụ**: "strict mode" và "file legacy được dùng any" cùng tồn tại.

## 3. Khái niệm cốt lõi

- **Instruction bloat**: instruction chiếm hơn 10–15% context window.
- **Signal-to-Noise Ratio (SNR)**: tỉ lệ instruction liên quan đến task hiện tại.
- **Entry file / router**: file ngắn dẫn đường đến docs chi tiết.
- **Reveal on demand**: tổng quan trước, chi tiết khi cần.
- **Mỗi rule có metadata**: *nguồn gốc* (vì sao thêm), *điều kiện áp dụng*, *điều kiện hết hạn*.

> Lưu ý: bài gốc cũng dẫn nghiên cứu của ETH Zurich cho thấy context file **không phải lúc nào cũng** tăng success rate và có thể tăng chi phí inference. Ngắn hơn không tự động tốt hơn. Phải test instruction trên chính các task nó phục vụ.

## 4. Ví dụ minh hoạ: so sánh trước và sau

**Trước (một khối):**

```markdown
# AGENTS.md (612 dòng)
... 80 dòng setup ... 120 dòng deploy ... dòng 300: "Renderer không được import fs" ...
... 90 dòng lịch sử bug ... 60 dòng style ...
```

**Sau (router):**

```markdown
# AGENTS.md

## Project
DocVault: Electron 33 + React 18 + TS 5 desktop app, quản lý tài liệu + RAG Q&A local.

## Quick start
- `./init.sh`        cài + verify môi trường
- `npm run dev`      chạy app
- `npm run verify`   check + lint + test (BẮT BUỘC trước khi báo xong)

## Hard constraints (vi phạm = fail)
1. Renderer không import fs/path/electron, chỉ dùng window.api
2. IPC channel chỉ khai báo trong src/shared/types.ts
3. API key chỉ ở main process
4. Một feature mỗi lần (xem feature_list.json)

## Topic docs: đọc khi task liên quan
- docs/ipc-rules.md: khi thêm/sửa IPC channel hoặc preload
- docs/rag-pipeline.md: khi đụng chunking, embedding, retrieval, prompt
- docs/testing.md: khi viết test hoặc E2E
- docs/ARCHITECTURE.md: khi thêm service/module mới
- docs/DECISIONS.md: trước khi đổi thư viện hay storage

## Before you finish
Chạy `npm run verify`, cập nhật claude-progress.md, commit.
```

Để ý: hard constraints ở **đầu**, checklist "Before you finish" ở **cuối**. Đây là tận dụng hiệu ứng lost in the middle theo chiều có lợi.

## 5. Áp dụng vào DocVault: M3

**Mục tiêu:** refactor harness của DocVault thành router + 3–5 topic docs.

| File | Nội dung | Độ dài |
|---|---|---|
| `AGENTS.md` | Như ví dụ trên | ≤ 100 dòng |
| `docs/ipc-rules.md` | Cách thêm channel: khai báo trong `IPC_CHANNELS` → handler trong `ipc-handlers.ts` → expose trong `preload.ts` → type trong `renderer/types.d.ts` | 50–80 dòng |
| `docs/rag-pipeline.md` | Kích thước chunk, overlap, embedding provider interface, top-k, format citation | 80–120 dòng |
| `docs/testing.md` | Unit với Vitest, mock `LlmProvider`, fixture docs trong `tests/fixtures/`, E2E (bổ sung ở bài 10) | 50–80 dòng |

**Mỗi rule có metadata (ví dụ trong `rag-pipeline.md`):**

```markdown
- Chunk size 500 ký tự, cắt theo paragraph, overlap 50.
  - Nguồn: ADR-004, P03 chunking.
  - Áp dụng: IndexingService.chunk().
  - Hết hạn: khi chuyển sang semantic chunking (ADR mới).
```

**Thực nghiệm nhỏ:** đặt rule "Không gửi API key xuống renderer" ở giữa một file dài, chạy 3 task liên quan; sau đó chuyển rule lên đầu và chạy lại. Ghi kết quả vào learning-log.

**.NET tương đương:** `AGENTS.md` trỏ tới `docs/efcore-rules.md`, `docs/api-conventions.md`. Rule nào enforce được thì dùng Roslyn analyzer hoặc `.editorconfig` thay vì viết bằng chữ.

## 6. Checklist tự kiểm tra

- [ ] `AGENTS.md` của DocVault ≤ 100 dòng, hard constraints ≤ 10 dòng ở đầu file.
- [ ] Mỗi topic doc có dòng "đọc khi nào".
- [ ] Đã tính SNR của AGENTS.md cũ cho 3 loại task (fix bug UI, thêm IPC, sửa RAG).
- [ ] Đã chạy `npx tsx docs/en/lectures/lecture-04-why-one-giant-instruction-file-fails/code/split-vs-monolithic.ts`.

## 7. Câu hỏi suy ngẫm

1. Rule nào trong AGENTS.md có thể biến thành lint/test (tức là enforce bằng code thay vì bằng chữ)?
2. Có rule nào hai file đang nói ngược nhau không?
