# Bài 13 – Từ prompt thủ công đến autonomous loop

> Bài gốc: [EN](../docs/en/lectures/lecture-13-loop-engineering/index.md) · [VI](../docs/vi/lectures/lecture-13-loop-engineering/index.md)
> Code ví dụ: `.../lecture-13-loop-engineering/code/` (`goal-template.md`, `loop-state-template.md`, `maker-prompt.md`, `checker-prompt.md`)
> Project thực hành: **P07: Your first automated loop** (`docs/en/projects/project-07-loop-engineering-first-loop/`, chỉ có docs)
> Milestone DocVault: **M12: RAG eval loop (`/goal`), nightly check (`/loop`/cron), maker–checker**

---

## 1. Ý chính trong 30 giây

Bài 01–12 giả định **bạn ngồi gõ lệnh**. Loop engineering đưa nút "start" cho hệ thống: *"Bạn không prompt agent nữa, bạn thiết kế hệ thống prompt agent."* Loop đơn giản nhất là `/goal`, gồm 3 phần: **goal + cách verify + điều kiện dừng**. Loop không thay harness. Nó là **tầng xây trên harness**: harness làm một lần chạy đáng tin, loop làm việc chạy liên tục thành tự động.

## 2. Khái niệm cốt lõi

**Bốn loại loop:**

| Loại | Trigger | Dừng khi | Claude Code | Dùng cho |
|---|---|---|---|---|
| Turn-based | Bạn gõ | Agent nghĩ xong / bạn ngắt | chat thường | Việc nhỏ, khám phá |
| Goal-based | Bạn đưa goal | Evaluator độc lập xác nhận / hết budget | `/goal` | Việc lớn có tiêu chí xong rõ |
| Time-based | Chu kỳ | Bạn dừng | `/loop`, scheduled tasks, Routines | Polling, kiểm tra định kỳ |
| Event-driven | PR, CI fail, issue | Xử lý xong / hết retry | Routines + GitHub webhook, GitHub Actions | Phản ứng với CI/CD |

**Test một câu:** việc này **có điểm kết thúc** không? Có thì dùng `/goal`. Không có, chỉ cần canh chừng, thì dùng `/loop`. Sai lầm hay gặp: `/loop 10m "tiếp tục implement..."`. Mỗi lần `/loop` chạy độc lập, không nhớ lần trước.

**Sáu primitive của loop** (Addy Osmani):

| Primitive | Vai trò |
|---|---|
| **Automations** | Nhịp tim, tức là trigger theo lịch hoặc event |
| **Worktrees** | Cô lập khi chạy song song (`git worktree`) |
| **Skills** | Tri thức dự án đóng gói (`SKILL.md`), không phải giải thích lại mỗi lần |
| **Connectors (MCP)** | Chạm vào tool thật: GitHub, issue tracker, Slack |
| **Sub-agents** | Tách maker và checker |
| **External state** | Trí nhớ trên đĩa (markdown, board), là xương sống của mọi thứ |

**Generator/Evaluator separation:** *"Một model là luật sư bào chữa giỏi nhất cho output của chính nó."* Trong crew phải có người **không tin** bạn.

**Ratchet loop (Karpathy autoresearch):** sửa code → commit → chạy với budget cố định → metric tốt hơn thì giữ, kém hơn thì `git reset`. Chỉ tiến, không lùi. Người chỉ sửa `program.md` (phương pháp), agent sửa code.

**Bốn chi phí âm thầm:** *verification debt* (điều kiện dừng phải machine-checkable), *comprehension rot* (không còn hiểu code loop sinh ra), *cognitive surrender* (dùng loop để khỏi phải nghĩ), *token blowout* (context phình theo số vòng).

**Thang trưởng thành:** L1 Goal runner → L2 scheduled single-task → L3 multi-agent maker/checker + worktree → L4 self-feeding → L5 fleet.

## 3. Ví dụ minh hoạ

**`loops/goal.md` (dựa trên `goal-template.md`):**

```markdown
# Goal: RAG eval đạt ngưỡng
## Done khi
- `npx tsx scripts/rag-eval.ts` báo hit@5 ≥ 0.90 và groundedness ≥ 0.95 trên eval/questions.jsonl
- `npm run verify:full` xanh
## Verify
- Chạy 2 lệnh trên; checker độc lập đọc report eval/report.json
## Stop
- Đạt Done, HOẶC 8 vòng, HOẶC 2 giờ, HOẶC cần đổi embedding model (cần người quyết định → dừng, ghi lý do)
## Không được đụng
- eval/questions.jsonl (anchor, cấm sửa để "pass"), DECISIONS.md, feature_list.json state
## Được thử
- chunk size/overlap, top-k, prompt template, rerank
```

Ghi chú quan trọng: `eval/questions.jsonl` bị **đóng băng**. Nếu không, loop sẽ "học cách" sửa đề cho dễ pass. Đây chính là Goodhart, sẽ học ở bài 14.

**`loops/loop-state.md`** (external state, mỗi vòng ghi một dòng):

```markdown
| Vòng | Thay đổi | hit@5 | grounded | Quyết định | Commit |
|---|---|---|---|---|---|
| 0 | baseline | 0.78 | 0.92 | – | a1b2c3 |
| 1 | chunk 500→800, overlap 100 | 0.84 | 0.93 | keep | d4e5f6 |
| 2 | top-k 5→8 | 0.83 | 0.90 | revert | – |
```

## 4. Áp dụng vào DocVault: M12

**① Eval set + script** (điều kiện tiên quyết của mọi loop RAG):

```jsonl
{"id":"q01","question":"Chunk size mặc định là bao nhiêu?","expected_doc":"rag-guide.md","expected_contains":["500"]}
{"id":"q02","question":"Thủ đô của Úc là gì?","expected":"NOT_FOUND"}
```

`scripts/rag-eval.ts`: import `tests/fixtures/` vào một data dir tạm, index, hỏi từng câu, tính `hit@5` (expected_doc nằm trong retrieved) + `groundedness` (dùng `assertGrounded` ở bài 09) + `not_found_accuracy`, rồi ghi `eval/report.json`. Chạy được với mock (CI) và với LLM thật (`DOCVAULT_LLM=claude`).

**② Ba thí nghiệm của P07, áp vào DocVault:**

| Thí nghiệm | Làm gì |
|---|---|
| Goal loop | Chạy tay một lần (đếm số lần bạn can thiệp), sau đó chạy `/goal` với `loops/goal.md`, rồi so sánh |
| Timer loop | `/loop 1h Run npm run verify:full and npx tsx scripts/rag-eval.ts; if any metric drops vs loops/loop-state.md, write a short report to loops/alerts.md`. Bản lâu dài: GitHub Actions `schedule: cron` chạy eval mỗi đêm |
| Maker–checker | Maker (sub-agent) đổi một tham số RAG trong worktree riêng. Checker (session mới, prompt khác, nếu được thì model khác) chạy eval + review diff, **bắt buộc dẫn chứng**. Chỉ checker được quyết keep/revert |

**③ Skill cho DocVault** (`.claude/skills/docvault-rag/SKILL.md`): đóng gói "cách thêm một document type mới" hoặc "cách tune RAG" để mỗi vòng loop không phải đọc lại toàn bộ docs.

**④ Kiểm soát chi phí:** đặt budget (số vòng, thời gian), log token usage của mỗi vòng từ trace ở bài 11, và đọc lại code loop sinh ra mỗi tuần (chống comprehension rot).

## 5. Checklist tự kiểm tra

- [ ] Phân biệt được `/goal` và `/loop` bằng test "có điểm kết thúc không".
- [ ] Có `eval/questions.jsonl` ≥ 20 câu (≥ 3 câu NOT_FOUND) và `scripts/rag-eval.ts`.
- [ ] Đã chạy ít nhất một `/goal` và một `/loop` (hoặc cron) trên DocVault.
- [ ] Maker và checker tách context; checker có prompt riêng (xem `code/checker-prompt.md`).
- [ ] Đã tự audit 4 chi phí âm thầm sau một giờ chạy loop.

## 6. Câu hỏi suy ngẫm

1. Việc nào bạn làm tay ≥ 2 lần/tuần ở công ty có thể biến thành loop L2?
2. Nếu loop tune RAG làm hit@5 tăng nhưng người dùng thật thấy câu trả lời tệ hơn, anchor của bạn đang thiếu gì?
