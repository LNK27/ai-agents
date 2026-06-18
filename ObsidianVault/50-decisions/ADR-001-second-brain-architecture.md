---
id: adr-001-second-brain-architecture
title: "ADR 001: Kiến trúc cốt lõi Second Brain v1"
type: decision
status: approved
tags:
  - decision
  - adr
  - architecture
created_at: 2026-05-21T17:10:00+07:00
updated_at: 2026-05-21T17:10:00+07:00
writer: system
---

# ADR 001: Định hình Kiến trúc cốt lõi Second Brain v1 (Roadmap v7)

## Bối cảnh (Context)
Cần xây dựng một hệ thống AI cá nhân tự vận hành (Agentic Second Brain) cục bộ trên máy tính Dell Latitude 5421 (16GB RAM, không GPU rời). Hệ thống cần kết nối linh hoạt giữa khả năng thu thập thông tin tự động, quản lý tri thức cá nhân và khả năng thực thi tác vụ lập trình/tự động hóa an toàn.

## Các Quyết định Kiến trúc chính (Approved Decisions)

Hệ thống đã trải qua 7 vòng phản biện chéo và đạt trạng thái **Design Freeze** với các quyết định sau:

### 1. Thành phần & Phân vai
- **D1 (Orchestrator trung tâm):** Chọn `OpenAgentd` làm bộ điều phối trung tâm thay vì Hermes hay OpenHuman nhờ có Web Cockpit UI, MCP hub và license thân thiện (Apache 2.0).
- **D2 (Giao tiếp):** Giao tiếp giữa các thành phần thông qua giao thức MCP (Model Context Protocol), không trộn mã nguồn để tránh xung đột giấy phép.
- **D3 (Trình duyệt):** Chọn Brave Browser thay cho Chrome mặc định theo sở thích người dùng và tối ưu tài nguyên.
- **D4 (API & Proxy):** Sử dụng Gemini Free Tier kết hợp 9Router local proxy để tối ưu chi phí.
- **D11 (Hệ điều hành):** Chạy native 100% trên Windows/PowerShell. Không sử dụng WSL ở v1 để tránh ma sát dịch đường dẫn (`/mnt/d/` vs `D:\`).

### 2. Cô lập Tài nguyên & Quản lý Hạn ngạch (Quota & Resources)
- **D5 (Cô lập Key):** Cô lập hoàn toàn API key giữa `OpenAgentd` và `browser-harness` (sử dụng 2 Google API Key riêng biệt) nhằm tránh đụng trần hạn ngạch (15 RPM) khi chạy song song.
- **D10 (MCP Watchdog):** Triển khai cơ chế auto-recovery/watchdog cho các MCP server ngay từ Phase 1 để tự phục hồi kết nối trình duyệt khi bị crash trên Windows.

### 3. An toàn Hệ thống (System Security)
- **D6 (Shell Gate):** Thiết lập cổng phê duyệt lệnh shell (**Shell Permission Gate**): tự động cho phép các tác vụ đọc/test an toàn, chặn hoàn toàn các lệnh nguy hiểm (delete root, v.v.), và tạm dừng để hỏi ý kiến người dùng đối với các lệnh thay đổi hệ thống.

### 4. Quản lý Tri thức (Vault Policies)
- **D7 (Obsidian Vault):** Chọn Obsidian Vault làm nguồn sự thật duy nhất (Single Source of Truth), lưu trữ dưới dạng Markdown cục bộ.
- **D8 (Single Writer Policy):** Chỉ cho phép `OpenAgentd` đại diện cho các AI Agent ghi dữ liệu vào Obsidian Vault thông qua một Gatekeeper tuần tự để tránh tranh chấp ghi file (file lock).
- **D9 (Human Ingest):** Người dùng có thể chỉnh sửa trực tiếp qua Obsidian Desktop; hệ thống sẽ có luồng quét định kỳ (reconcile) để nạp các thay đổi từ con người mà không ghi đè.

## Hệ quả (Consequences)
- Hệ thống chạy cực kỳ mượt mà trên môi trường Windows native với lượng RAM 16GB.
- Đảm bảo an toàn tuyệt đối cho dữ liệu của người dùng nhờ có Shell Gate và cơ chế ghi Vault tập trung.
- Tránh được lỗi quá hạn ngạch (rate limiting) từ nhà cung cấp API.

---
[[MAP_OF_CONTENT|Xem Bản Đồ Tri Thức (MAP_OF_CONTENT)]] | [[50-decisions/_index|Quay lại danh sách Quyết định]]
