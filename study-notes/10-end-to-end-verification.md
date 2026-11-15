# Bài 10 – Chỉ chạy full pipeline mới là verify thật

> Bài gốc: [EN](../docs/en/lectures/lecture-10-why-end-to-end-testing-changes-results/index.md) · [VI](../docs/vi/lectures/lecture-10-why-end-to-end-testing-changes-results/index.md)
> Code ví dụ: `.../lecture-10-.../code/` (`e2e-runner.ts`, `architecture-rules.md`, `review-feedback-to-rule.md`)
> Project thực hành: **P05** (tiếp)
> Milestone DocVault: **M9: Playwright E2E cho luồng import → index → ask, architecture check có FIX message**

---

## 1. Ý chính trong 30 giây

Unit test cô lập và mock dependency, nên **mù với lỗi ở ranh giới component**. Trong Electron, renderer gửi path tương đối trong khi preload cần path tuyệt đối; cả hai unit test đều xanh nhưng app hỏng. Chỉ E2E mới chứng minh được không có lỗi cấp hệ thống. Thêm nữa, khi agent biết sẽ bị E2E kiểm, **nó code khác đi**: để ý interface, error path, boundary.

## 2. Khái niệm cốt lõi

| Khái niệm | Nghĩa |
|---|---|
| **Component boundary defect** | A và B đều pass unit test nhưng tương tác thì sai |
| **Testing adequacy gradient** | unit ≤ integration ≤ E2E về khả năng phát hiện lỗi |
| **Architectural boundary enforcement** | Biến rule kiến trúc thành check tự động trong CI |
| **Review feedback promotion** | Comment review lặp lại thì chuyển thành test/lint |
| **Agent-oriented error message** | Lỗi phải nói *what, why, how to fix* |
| **Enforce invariants, don't micromanage** | Bắt buộc "parse data ở boundary", không bắt buộc dùng thư viện nào |

**4 loại lỗi unit test bỏ sót:** interface mismatch, state propagation, resource lifecycle (handle, connection), environment dependency (packaged app khác dev).

**Layered architecture (OpenAI):** Types → Config → Repo → Service → Runtime → UI, phụ thuộc chỉ đi một chiều, enforce bằng lint. Bạn sẽ thấy quen vì nó giống Clean Architecture/Onion trong .NET.

## 3. Ví dụ minh hoạ: architecture check

```bash
#!/usr/bin/env bash
# scripts/check-architecture.sh
fail=0
if grep -rnE "from ['\"](fs|node:fs|path|electron)['\"]" src/renderer/; then
  echo "ERROR: Node/Electron import in renderer"
  echo "WHY:   renderer runs sandboxed (contextIsolation); direct I/O breaks security model"
  echo "FIX:   add an IPC channel in src/shared/types.ts, handler in src/main/ipc-handlers.ts, expose via src/preload/preload.ts, call window.api.<method>()"
  fail=1
fi
if grep -rn "ANTHROPIC_API_KEY\|process.env" src/renderer/ src/preload/; then
  echo "ERROR: secret/env access outside main process"
  echo "FIX:   read config in src/main/config.ts; pass only non-secret values to renderer"
  fail=1
fi
if grep -rnE "ipcRenderer\.(invoke|send)\(['\"]" src/preload/ | grep -v IPC_CHANNELS; then
  echo "ERROR: hard-coded IPC channel string"
  echo "FIX:   use IPC_CHANNELS.<NAME> from src/shared/types.ts"
  fail=1
fi
exit $fail
```

Tham chiếu bản gốc: `projects/project-04/solution/scripts/check-architecture.sh`.

## 4. Áp dụng vào DocVault: M9

**① E2E bằng Playwright cho Electron** (`tests/e2e/rag-flow.spec.ts`):

```ts
import { _electron as electron, test, expect } from "@playwright/test";
import path from "node:path";

test("import → index → ask → citation", async () => {
  const app = await electron.launch({
    args: ["dist/main/main.js"],
    env: { ...process.env, DOCVAULT_LLM: "mock", DOCVAULT_EMBEDDING: "fake", DOCVAULT_DATA: ".tmp/e2e-data" },
  });
  const win = await app.firstWindow();

  await win.getByTestId("import-input").setInputFiles(path.resolve("tests/fixtures/rag-guide.md"));
  await expect(win.getByTestId("doc-item")).toHaveCount(1);

  await win.getByTestId("index-all").click();
  await expect(win.getByTestId("index-status")).toHaveText(/Indexed/, { timeout: 15_000 });

  await win.getByTestId("question-input").fill("Chunk size mặc định là bao nhiêu?");
  await win.getByTestId("ask-button").click();
  const cite = win.getByTestId("citation").first();
  await expect(cite).toBeVisible();
  await cite.click();
  await expect(win.getByTestId("doc-detail-title")).toHaveText("rag-guide.md");

  await app.close();
});
```

Script trong `package.json`:

```json
"scripts": {
  "e2e": "npm run build && playwright test tests/e2e",
  "verify": "npm run check && npm run lint && bash scripts/check-architecture.sh && npm test",
  "verify:full": "npm run verify && npm run e2e"
}
```

**② Validation hierarchy trong AGENTS.md:**

```markdown
## Validation hierarchy
- L1 unit (npm test): luôn bắt buộc
- L2 integration (tests/integration): bắt buộc khi đụng service/DB
- L3 E2E (npm run e2e): bắt buộc khi thay đổi đi qua renderer ↔ preload ↔ main
- Bỏ qua một level bắt buộc = CHƯA XONG
```

**③ Review feedback promotion:** mỗi lần bạn review thấy agent lặp lại lỗi, ví dụ "quên `data-testid`" hoặc "không handle NOT_FOUND ở UI", hãy biến nó thành một check hoặc test và ghi vào `docs/review-rules.md` (nguồn, ngày, check tương ứng).

**Tương đương .NET (Avalonia):** E2E dùng `Avalonia.Headless` + xUnit; architecture test dùng **NetArchTest** hoặc **ArchUnitNET** (`Types().That().ResideInNamespace("UI").ShouldNot().HaveDependencyOn("Infrastructure")`).

## 5. Checklist tự kiểm tra

- [ ] `npm run e2e` chạy được luồng RAG đầy đủ với mock LLM + fake embedding.
- [ ] `check-architecture.sh` có ≥3 rule, mỗi rule có WHY/FIX.
- [ ] Đã cố ý cài một lỗi interface mismatch (ví dụ path tương đối) và xác nhận unit test không bắt được còn E2E thì bắt được.
- [ ] Đã đọc `docs/en/lectures/lecture-10-why-end-to-end-testing-changes-results/code/review-feedback-to-rule.md`.

## 6. Câu hỏi suy ngẫm

1. Bug nào ở dự án thật gần đây là component boundary defect?
2. E2E chậm hơn (2 giây so với 15 giây). Khi nào nên chạy E2E trong vòng làm việc của agent?
