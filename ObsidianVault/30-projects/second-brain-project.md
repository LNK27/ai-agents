---
id: second-brain-project
title: "Second Brain Project"
type: project
status: active
tags:
  - project
  - second-brain
  - ai-agents
created_at: 2026-05-21T17:10:00+07:00
updated_at: 2026-05-21T17:10:00+07:00
writer: system
---

# 🧠 Dự án Second Brain (Local-first AI Stack)

## 🎯 Mục tiêu dự án
Xây dựng một hệ thống **Agentic Second Brain** tự vận hành hoàn toàn cục bộ trên máy tính Dell Latitude 5421 (Windows, 16GB RAM). Hệ thống đóng vai trò mở rộng trí tuệ cá nhân, tự học và quản lý kho tri thức một cách có hệ thống thông qua Obsidian.

## 🧱 Các thành phần kiến trúc cốt lõi
- **OpenAgentd:** Bộ điều phối trung tâm (orchestrator), cung cấp giao diện Cockpit UI điều phối đa tác tử, cổng phân quyền và giám sát hệ thống.
- **browser-harness:** Runtime duyệt web tự động sử dụng Brave + Playwright kết nối qua MCP stdio server.
- **Hermes Agent:** Mô hình suy luận hỗ trợ lập kế hoạch và soạn thảo kỹ năng (Phase 2).
- **Obsidian Vault:** Nơi lưu trữ tri thức bền vững và có cấu trúc.

---

## 📅 Lộ trình Triển khai (Roadmap v7)

### 🟢 Phase 1: Core Runtime An Toàn (ĐANG THỰC HIỆN)
- [x] Tạo Python venv và cài đặt dependencies cho `browser-harness`
- [x] Sửa `test_browser.py` ép chạy Brave Browser thành công
- [x] Tách provider và Google API Key riêng biệt cho `browser-harness` và `OpenAgentd`
- [x] Tích hợp `browser-use` MCP vào OpenAgentd config (`mcp.json`)
- [x] Xây dựng cơ chế **MCP watchdog** tự phục hồi khi crash
- [x] Thiết lập cổng bảo mật **Shell Permission Gate** (allow/deny/ask)
- [x] Khởi tạo cấu trúc **Obsidian Vault skeleton** v7 AI-readable
- [ ] Tích hợp chạy thực tế toàn bộ hệ thống kiểm tra end-to-end

### 🟡 Phase 2: Memory Workflow & Ingest (KẾ TIẾP)
- [ ] Xây dựng **Vault Gatekeeper** để quản lý hàng đợi ghi tuần tự tránh file lock.
- [ ] Định hình Note Schema với YAML frontmatter đầy đủ.
- [ ] Thực hiện cơ chế **Human Ingest/Reconcile** quét và đánh chỉ mục ghi chú của người dùng tạo thủ công.
- [ ] Thiết lập kết nối **Hermes sidecar API** để hỗ trợ lập luận sâu.

### 🔴 Phase 3: Quan sát & Mở rộng
- [ ] Đo lường hiệu năng, tỷ lệ thành công của tác vụ, độ trễ và dung lượng bộ nhớ.
- [ ] Lên lịch tác vụ tự động (tóm tắt tin tức mỗi đêm, báo cáo tuần).

---
[[MAP_OF_CONTENT|Xem Bản Đồ Tri Thức (MAP_OF_CONTENT)]] | [[30-projects/_index|Quay lại danh sách Projects]]
