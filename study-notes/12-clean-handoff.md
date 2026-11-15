# Bài 12 – Bàn giao sạch cuối mỗi session

> Bài gốc: [EN](../docs/en/lectures/lecture-12-why-every-session-must-leave-a-clean-state/index.md) · [VI](../docs/vi/lectures/lecture-12-why-every-session-must-leave-a-clean-state/index.md)
> Code ví dụ: `.../lecture-12-.../code/` (`cleanup-scanner.ts`, `benchmark-runner.ts`, `cleanup-loop.md`)
> Project thực hành: **P06** (tiếp, có benchmark + ablation)
> Milestone DocVault: **M11: session exit checklist, cleanup scanner, quality document, harness ablation**

---

## 1. Ý chính trong 30 giây

Entropy tăng là mặc định. Agent **copy pattern có sẵn trong repo, kể cả pattern xấu**, nên chỉ một ít bừa bộn sẽ kéo theo bừa bộn nhiều hơn. **Clean state** là điều kiện bắt buộc để coi session là xong: build pass, test pass, progress đã cập nhật, không còn rác, startup path chạy được. Thêm vào đó là **cleanup loop định kỳ** và **đơn giản hoá harness** khi model mạnh lên.

## 2. Khái niệm cốt lõi

| Khái niệm | Nghĩa |
|---|---|
| **Clean state (5 điều kiện)** | build, tests, progress recorded, no stale artifacts, startup path OK |
| **Session integrity** | Như transaction: commit sạch hoặc rollback về trạng thái nhất quán gần nhất |
| **Quality document** | Bảng chấm chất lượng từng module, cập nhật liên tục |
| **Cleanup loop** | Phiên bảo trì định kỳ để giảm entropy |
| **Harness simplification** | Định kỳ tắt thử một component; nếu không tụt thì bỏ |
| **Idempotent cleanup** | Chạy bao nhiêu lần cũng cho cùng kết quả |

**Hai chế độ cleanup** (giống GC): *immediate* (cuối mỗi session, kiểu reference counting) và *periodic* (hàng tuần, quét toàn bộ, kiểu tracing).

**Merge philosophy khi throughput cao:** nếu chi phí sửa bug thấp hơn chi phí chờ review thì merge nhanh và sửa nhanh là hợp lý. Ở môi trường throughput thấp thì làm vậy là thiếu trách nhiệm.

## 3. Ví dụ minh hoạ

**Session exit checklist (`clean-state-checklist.md`):**

```markdown
- [ ] `npm run build` pass
- [ ] `npm run verify` pass (check + lint + architecture + unit)
- [ ] `npm run e2e` pass nếu đã đụng renderer/preload/main
- [ ] feature_list.json cập nhật qua verify-feature.ts
- [ ] claude-progress.md + session-handoff.md đã cập nhật
- [ ] Không còn console.log/debugger/`.only(`/file tạm (`bash scripts/cleanup-scanner.sh`)
- [ ] `npm run dev` mở app và có log `app.ready`
- [ ] git status sạch, đã commit
```

**Cleanup scanner (idempotent):**

```bash
#!/usr/bin/env bash
# scripts/cleanup-scanner.sh
issues=0
report() { echo "STALE: $1"; echo "  FIX: $2"; issues=$((issues+1)); }
grep -rn "console\.log\|debugger" src/ --include=*.ts --include=*.tsx | grep -v "logger.ts" \
  && report "debug statements in src/" "replace with log() from src/services/logger.ts or remove"
grep -rn "\.only(" tests/ && report ".only in tests" "remove .only so the full suite runs"
ls .tmp/ 2>/dev/null | grep -q . && report ".tmp/ not empty" "rm -rf .tmp/* (E2E data dir)"
git status --porcelain | grep -q . && report "uncommitted changes" "commit finished work or git restore abandoned work"
echo "cleanup-scanner: $issues issue(s)"; exit $((issues > 0))
```

## 4. Áp dụng vào DocVault: M11

**① Quality document (`quality-document.md`):**

```markdown
# Quality document – cập nhật 2026-12-20

## DocumentService (A)
- Verify: pass · Agent hiểu được: dễ · Test: ổn định · Boundary: đúng

## IndexingService (B)
- Verify: pass · Test: 1 flaky (embedding timeout giả lập) · Ghi chú: file > 5MB chậm

## QaService (C)
- Verify: partial (NOT_FOUND chưa có E2E) · Agent hiểu được: khó (parse citation rải 3 file)
- Hành động: gom parser vào rag/citation-parser.ts + thêm E2E NOT_FOUND
```

Session mới đọc file này và **ưu tiên module điểm thấp nhất**.

**② Cleanup loop hàng tuần (chạy tay trước, sang bài 13 tự động hoá):**

1. `bash scripts/cleanup-scanner.sh`
2. Chạy `npx tsx scripts/rag-eval.ts` (bài 13) để phát hiện drift chất lượng.
3. Cập nhật quality-document; mở PR refactor nhỏ cho module điểm thấp nhất.

**③ Harness ablation (giống P06):** chọn 3 task cố định (ví dụ: thêm filter theo tag, thêm export câu trả lời ra Markdown, sửa một bug cài sẵn). Chạy:

| Biến thể | Bỏ gì |
|---|---|
| Full | – |
| −feature_list | xoá feature_list.json + rule |
| −progress/handoff | xoá claude-progress.md, session-handoff.md |
| −e2e | bỏ `npm run e2e` khỏi DoD |
| −architecture check | bỏ check-architecture.sh |

Ghi completion rate, số lần bạn phải can thiệp, và số lỗi lọt. Component nào bỏ đi mà kết quả không tụt thì cân nhắc bỏ hẳn (harness simplification). Tham khảo `projects/project-06/solution/scripts/benchmark.sh`.

**.NET tương đương:** cleanup scanner dùng `dotnet format --verify-no-changes`, tìm `Console.WriteLine`/`Debugger.Break()`, `[Fact(Skip=...)]`.

## 5. Checklist tự kiểm tra

- [ ] 5 session liên tiếp đều qua clean-state checklist (ghi lại số vi phạm theo từng chiều).
- [ ] Có quality-document với ít nhất 4 module.
- [ ] Đã chạy ablation ít nhất 2 biến thể.
- [ ] Đã chạy `npx tsx docs/en/lectures/lecture-12-why-every-session-must-leave-a-clean-state/code/cleanup-scanner.ts`.

## 6. Câu hỏi suy ngẫm

1. Pattern "xấu" nào trong DocVault mà agent đang copy đi copy lại?
2. Component harness nào bạn nghĩ là thừa với model hiện tại? Ablation có xác nhận không?
