---
id: map-of-content
title: Map of Content
type: moc
status: active
tags:
  - moc
  - index
created_at: 2026-05-21T17:10:00+07:00
updated_at: 2026-05-21T17:10:00+07:00
writer: system
---

# 🧠 Second Brain - Map of Content

Chào mừng đến với **Second Brain** của tôi. Đây là cổng thông tin trung tâm, đóng vai trò bản đồ chỉ dẫn (MOC - Map of Content) điều hướng qua toàn bộ kho tri thức cá nhân. Hệ thống được thiết kế theo cấu trúc v7 AI-readable để hỗ trợ tối đa cho cả người dùng và các AI Agent (như OpenAgentd và Hermes) tương tác một cách có hệ thống.

## 🗂️ Cấu trúc Kho lưu trữ (Vault Structure)

Kho tri thức này được chia thành các khu vực chức năng rõ rệt theo mô hình PARA mở rộng:

| Mã số | Thư mục | Mô tả | Index |
|---|---|---|---|
| **00** | [[00-inbox/_index\|00-inbox]] | Hộp thư đến: Nơi tiếp nhận ghi chú thô, ý tưởng chưa phân loại, dữ liệu ingest thô. | [[00-inbox/_index\|Inbox Index]] |
| **10** | [[10-sources/_index\|10-sources]] | Nguồn thông tin: Tài liệu đọc, bài báo, clip, podcast, bookmark từ web. | [[10-sources/_index\|Sources Index]] |
| **20** | [[20-topics/_index\|20-topics]] | Chủ đề & Khái niệm: Các mảng kiến thức dài hạn, khái niệm cốt lõi. | [[20-topics/_index\|Topics Index]] |
| **30** | [[30-projects/_index\|30-projects]] | Dự án hiện tại: Các dự án đang chạy có mục tiêu cụ thể và deadline. | [[30-projects/_index\|Projects Index]] |
| **40** | [[40-people/_index\|40-people]] | Danh bạ & Mối quan hệ: Thông tin người liên hệ, cuộc họp, tương tác xã hội. | [[40-people/_index\|People Index]] |
| **50** | [[50-decisions/_index\|50-decisions]] | Nhật ký Quyết định: Nhật ký lưu trữ các quyết định kiến trúc, công việc (ADR). | [[50-decisions/_index\|Decisions Index]] |
| **90** | [[90-archive/_index\|90-archive]] | Kho lưu trữ: Dự án đã hoàn thành, tài liệu không còn sử dụng nhưng cần giữ lại. | [[90-archive/_index\|Archive Index]] |

---

## 🤖 Nguyên tắc Tương tác dành cho AI Agent

1. **Single Writer Policy:** Chỉ ghi vào Vault thông qua `OpenAgentd Vault Gatekeeper`. Tuyệt đối không để nhiều Agent ghi đồng thời.
2. **Metadata Standard:** Mọi ghi chú mới phải tuân thủ nghiêm ngặt chuẩn YAML Frontmatter (có `id`, `title`, `type`, `status`, `tags`, `created_at`, `updated_at`, `writer`).
3. **Link Integrity:** Ưu tiên sử dụng WikiLinks `[[Tên ghi chú]]` để kết nối các khái niệm trong Vault.

---
*Last updated: 2026-05-21 by Antigravity*
