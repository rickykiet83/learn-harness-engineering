# Bài 01 – Model mạnh không có nghĩa là thực thi đáng tin

> Bài gốc: [EN](../docs/en/lectures/lecture-01-why-capable-agents-still-fail/index.md) · [VI](../docs/vi/lectures/lecture-01-why-capable-agents-still-fail/index.md)
> Code ví dụ: `docs/en/lectures/lecture-01-why-capable-agents-still-fail/code/`
> Project thực hành: **P01: Prompt-only vs. Rules-first** (`projects/project-01/`)
> Milestone DocVault: **M0: chạy thí nghiệm baseline, viết Definition of Done đầu tiên**

---

## 1. Ý chính trong 30 giây

Cùng một model, cùng một prompt, nhưng khác môi trường thì kết quả khác hẳn nhau. Khi agent làm hỏng việc, phản xạ "đổi sang model đắt hơn" thường sai. **Hãy kiểm tra harness trước.** Phần lớn lỗi là *harness-induced failure*: model đủ năng lực nhưng môi trường có lỗ hổng cấu trúc.

## 2. Năm kiểu agent hay "kẹt"

| # | Triệu chứng | Ví dụ | Layer phòng thủ |
|---|---|---|---|
| 1 | Yêu cầu mơ hồ, agent phải đoán | "Thêm chức năng search" (search cái gì? phân trang không?) | **Task specification** |
| 2 | Quy ước ngầm không được ghi lại | Team dùng EF Core 8 + `AsNoTracking()` nhưng agent viết kiểu cũ | **Context provision** |
| 3 | Môi trường chưa sẵn sàng | Thiếu dependency, sai version Node, agent tốn context để sửa `npm install` | **Execution environment** |
| 4 | Không có cách verify | Không có test/lint, agent "cảm thấy xong" là báo xong | **Verification feedback** |
| 5 | Mất state giữa các session | Session mới phải khám phá lại từ đầu | **State management** |

Năm layer này là khung **chẩn đoán**: mỗi lỗi phải quy được về một layer cụ thể.

## 3. Thuật ngữ cốt lõi

- **Harness**: mọi thứ ngoài model weights (instructions, tools, environment, state, feedback).
- **Harness-induced failure**: model đủ giỏi, môi trường có lỗi.
- **Verification gap**: khoảng cách giữa độ tự tin của agent và độ đúng thực tế. Đây là kiểu lỗi phổ biến nhất.
- **Context anxiety**: khi gần hết context window, agent vội kết thúc, bỏ qua bước verify.
- **Diagnostic loop**: chạy → quan sát lỗi → quy về layer → sửa layer đó → chạy lại.
- **Definition of Done (DoD)**: tập điều kiện kiểm chứng được bằng lệnh (test pass, lint sạch, typecheck pass).

## 4. Ví dụ minh hoạ

**Prompt tồi:**

```
Thêm API search tài liệu.
```

**Prompt có DoD (viết cho agent):**

```
Mục tiêu: endpoint GET /api/documents/search?q=...
Completion criteria:
- Trả về tối đa 20 item/trang, hỗ trợ ?page=
- Mỗi kết quả có snippet đã highlight từ khoá
- Unit test mới nằm trong tests/search.test.ts, `npm test` pass
- `npm run check` (tsc --noEmit) pass
- Không sửa file ngoài src/services/search/ và src/main/ipc-handlers.ts
```

Liên hệ với kinh nghiệm .NET: đây giống "Acceptance Criteria" trong một user story tốt, nhưng **phải chạy được bằng lệnh**. "Code clean" không phải DoD, còn `dotnet test` pass thì là DoD.

**Nhật ký chẩn đoán đơn giản** (giữ trong `docs/learning-log.md`):

```markdown
| Ngày | Task | Kết quả | Layer gây lỗi | Đã sửa harness thế nào |
|---|---|---|---|---|
| 11/15 | App shell | Fail: không launch | Environment | Thêm .nvmrc + script `npm run dev` vào AGENTS.md |
| 11/16 | Doc list | Fail: báo xong nhưng list rỗng | Verification | Thêm test render list với 2 file mẫu |
```

Sau vài vòng, bạn sẽ thấy layer nào là bottleneck.

## 5. Áp dụng vào DocVault: M0

**Mục tiêu:** tự cảm nhận khác biệt "có harness" so với "không harness" ngay trên app của mình.

1. Tạo 2 thư mục tách biệt `docvault-baseline/` và `docvault-harness/`. Chạy lần lượt từng cái, không để cái này nhìn thấy cái kia.
2. **Lần 1 (baseline):** chỉ đưa prompt *"Build an Electron app that shows a list of documents and lets me ask questions about them."* Không AGENTS.md, không DoD. Đo thời gian, ghi lại agent làm gì, app có chạy không.
3. **Lần 2 (có harness tối thiểu):** đặt sẵn một `AGENTS.md` ngắn và DoD cho M0:

   ```markdown
   # DocVault – Milestone M0
   Stack: Electron 33, TypeScript 5, React 18, Vite 6, Vitest 2. Node 20.
   ## Definition of Done
   - `npm run dev` mở cửa sổ 1200x800, contextIsolation=true, nodeIntegration=false
   - Sidebar trái hiển thị "No documents yet" khi thư mục data trống
   - Panel phải có ô nhập câu hỏi + nút Ask (chưa cần trả lời thật)
   - App tạo thư mục `~/.docvault/data` khi khởi động lần đầu
   - `npm run check` và `npm test` pass
   ```

4. So sánh 2 lần chạy, quy từng lỗi về 1 trong 5 layer.

**Tham chiếu code:** `projects/project-01/starter/task-prompt.md` (prompt yếu) và `projects/project-01/solution/` (AGENTS.md, feature_list.json, init.sh). Hãy đọc kỹ `solution/feature_list.json` để xem 4 feature của P01 được mô tả thế nào.

## 6. Checklist tự kiểm tra

- [ ] Kể được 5 layer chẩn đoán và cho mỗi layer một ví dụ trong dự án thật của bạn.
- [ ] Viết được DoD cho một task bất kỳ chỉ gồm các điều kiện chạy được bằng lệnh.
- [ ] Đã chạy thử `npx tsx docs/en/lectures/lecture-01-why-capable-agents-still-fail/code/failure-pattern-demo.ts`.
- [ ] Có file `learning-log.md` với ít nhất 2 dòng chẩn đoán.

## 7. Câu hỏi suy ngẫm

1. Lần gần nhất bạn nghĩ "model này dở", lỗi đó thực ra thuộc layer nào?
2. Trong team .NET hiện tại, có bao nhiêu quy ước chỉ nằm trong đầu người hoặc trong Slack? Agent có thấy chúng không?
