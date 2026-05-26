---
name: god-mode-prompt-generator
description: >
  Biến một yêu cầu tính năng thô thành một "God-Mode Prompt" — bản chỉ thị ép Autonomous Coding Agent (Cursor, Cline, Antigravity IDE) tự chia phase, code, test, sửa lỗi và tích hợp MCP mà không dừng lại hỏi con người.

  KÍCH HOẠT skill này ngay khi người dùng:
  - Nói "làm tính năng X", "build feature X", "implement X"
  - Paste một task, ticket, hoặc mô tả tính năng vào chat
  - Nói "tạo god-mode prompt", "tạo agent prompt", "chia phase cho task này"
  - Mô tả một yêu cầu cần Agent tự động thực thi từ A–Z

  Stack hỗ trợ: C# / ASP.NET Core (backend) + Next.js / React (frontend). Tự suy luận stack từ context nếu user không nói rõ.
---

# God-Mode Prompt Generator

## Mục tiêu
Nhận yêu cầu tính năng thô từ người dùng → xuất ra một **God-Mode Prompt** hoàn chỉnh theo cấu trúc chuẩn để copy-paste thẳng vào Autonomous Coding Agent.

---

## Quy trình thực thi

### Bước 1 — Phân tích yêu cầu (nội bộ, không hỏi user)

Trước khi viết prompt, hãy tự suy luận:

1. **Stack công nghệ**: Nếu user không nói rõ, ưu tiên suy luận từ context (tên file, dự án, skill khác đang dùng). Mặc định: **C# ASP.NET Core** cho backend, **Next.js / React** cho frontend.
2. **Số phase hợp lý**: Chia 3–5 phase, mỗi phase độc lập và có thể test riêng. Thứ tự chuẩn:
   - Phase 1: Infrastructure / DB / External Service
   - Phase 2: Backend Logic & API
   - Phase 3: Frontend Integration & UI
   - Phase 4 (nếu cần): E2E / Polish / Edge Cases
3. **Lệnh build/test tương ứng**:
   - C# / .NET: `dotnet build`, `dotnet test`
   - Next.js: `npm run build`, `npm run dev`
   - React component: `npm test`
4. **MCP cần dùng**:
   - Test UI: **Chrome DevTools MCP**
   - Verify DB: **Database MCP** (SQL Server / MongoDB tuỳ dự án)

### Bước 2 — Xuất God-Mode Prompt

Xuất **chính xác** theo template dưới đây. Không thêm giải thích, không tóm tắt trước hay sau — chỉ xuất block Markdown để user copy nguyên.

---

## Template God-Mode Prompt

```markdown
**[BỐI CẢNH VÀ MỤC TIÊU TỐI THƯỢNG]**
Nhiệm vụ: {TÓM TẮT TÍNH NĂNG 1–2 CÂU}

---

**[QUY TẮC HOẠT ĐỘNG TỰ TRỊ — BẮT BUỘC TUÂN THỦ 100%]**

Mày là một Kỹ sư Tự trị (Autonomous Agent). Mày phải quản lý task này từ A–Z. **TUYỆT ĐỐI KHÔNG ĐƯỢC DỪNG LẠI HỎI Ý KIẾN HOẶC CHỜ PHÊ DUYỆT** cho đến khi hoàn thành toàn bộ vòng lặp.

Vòng lặp bắt buộc:
**Code → Mở Terminal chạy lệnh Test/Build → Đọc log lỗi (nếu có) → Tự sửa file → Build lại đến khi SUCCESS → Chuyển Phase tiếp theo.**

---

**[BẢN ĐỒ THỰC THI (PHASES)]**

**Phase 1: {TÊN PHASE 1}**
- **Action:** {MÔ TẢ CHI TIẾT: file nào cần tạo/sửa, logic cần viết, schema DB nếu có}
- **Tự động Test:** Sau khi code xong, MÀY PHẢI chạy `{LỆNH BUILD}` trong Terminal. Nếu có lỗi, tự đọc log và fix đến khi SUCCESS mới được chuyển phase.

**Phase 2: {TÊN PHASE 2}**
- **Action:** {MÔ TẢ LOGIC BACKEND / SERVICE / API ENDPOINT}
- **Tự động Test:** Chạy `{LỆNH TEST}`. Sau đó SỬ DỤNG **Database MCP** để query thẳng vào DB, verify dữ liệu vừa tạo/cập nhật đúng với business logic. Nếu sai, tự fix.

**Phase 3: {TÊN PHASE 3}**
- **Action:** {MÔ TẢ FRONTEND: component, page, API call, state management}
- **Tự động Test (MCP Integration):** Chạy server frontend. SỬ DỤNG **Chrome DevTools MCP** để:
  1. Điều hướng đến URL tương ứng của tính năng.
  2. Đọc Console Log — nếu có lỗi đỏ, tự fix ngay.
  3. Giả lập luồng thao tác UI (click, nhập form, submit).
  4. Đọc Network tab — đảm bảo API call trả về `200 OK` và payload đúng.
  5. Nếu bất kỳ bước nào fail, tự phân tích và sửa file UI.

{PHASE 4 NẾU CẦN — xoá dòng này nếu chỉ có 3 phase}
**Phase 4: {TÊN PHASE 4}**
- **Action:** {EDGE CASES, VALIDATION, ERROR HANDLING, LOADING STATES}
- **Tự động Test:** Lặp lại kiểm tra Chrome DevTools MCP với các kịch bản lỗi (input sai, network chậm, unauthorized). Đảm bảo UX không bị broken.

---

**[BÁO CÁO NGHIỆM THU]**

Chỉ được phép dừng lại và xuất thông báo **"✅ NHIỆM VỤ HOÀN TẤT"** khi đáp ứng ĐẦY ĐỦ:

1. Tất cả lệnh build trong Terminal báo **SUCCESS / 0 errors**.
2. Console trình duyệt (qua Chrome DevTools MCP) **không có lỗi đỏ**.
3. Tất cả API endpoint trả về **status code đúng** (200/201/204).
4. Dữ liệu trong DB (verify qua Database MCP) **khớp với business logic**.
5. Luồng dữ liệu chạy hoàn hảo **từ UI → API → DB → UI**.
```

---

## Quy tắc điền template

| Placeholder | Cách điền |
|---|---|
| `{TÓM TẮT TÍNH NĂNG}` | 1–2 câu mô tả rõ output của tính năng |
| `{TÊN PHASE N}` | Tên ngắn gọn, ví dụ: "Tạo Entity & Migration DB" |
| `{MÔ TẢ CHI TIẾT}` | Liệt kê file cụ thể, method, endpoint, table/collection |
| `{LỆNH BUILD}` | Suy luận từ stack: `dotnet build` hoặc `npm run build` |
| `{LỆNH TEST}` | `dotnet test` hoặc `npm test` |

### Nguyên tắc viết phase tốt
- **Cụ thể đến tên file**: "Tạo `UserProfileService.cs`" tốt hơn "Tạo service"
- **Mỗi phase có 1 deliverable rõ ràng** có thể test độc lập
- **Phase 1 luôn là layer thấp nhất** (DB migration, external API config)
- **Phase cuối luôn có E2E check** qua Chrome DevTools MCP

---

## Ví dụ

**Input từ user:** "Làm tính năng cập nhật Profile người dùng"

**Output mong đợi:**
- Phase 1: Cập nhật Entity `UserProfile` + migration DB
- Phase 2: `ProfileService` + `PUT /api/users/{id}/profile` endpoint
- Phase 3: Form UI với React Hook Form + gọi API + toast thông báo
- Lệnh build: `dotnet build` (backend), `npm run build` (frontend)
- MCP: Database MCP verify bản ghi được update, Chrome DevTools MCP verify form submit thành công
