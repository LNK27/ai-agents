---
id: ADR-002-hermes-hybrid-integration
title: Hermes Agent Hybrid Integration — Implementation Plan v5
type: note
status: draft
tags: []
created_at: '2026-08-13T04:39:31.131333+00:00'
updated_at: '2026-08-13T04:39:31.131333+00:00'
source_refs: []
relations: []
last_summarized_at: null
writer: human
---
# Hermes Agent Hybrid Integration — Implementation Plan v5

Tích hợp `NousResearch/hermes-agent` vào `OpenAgentd` theo mô hình **Hybrid**: vận hành qua MCP (lifecycle) và HTTP (data logic), bảo toàn Single Writer Policy (ADR-001 D8).

> [!NOTE]
> **Trạng thái sau audit 2026-08-13:** Đây vẫn là kế hoạch, chưa phải tính năng đã triển khai. Codebase hiện có Hermes proposal-only adapter, hàng đợi duyệt thủ công, read-only query và skill drafting; chưa có Bridge, auto-approve, active MCP registration, tests Bridge/auto-approve, hoặc ADR-003 sửa đổi D8. Những phần có sẵn không được coi là bằng chứng triển khai cho các component bên dưới.

> [!WARNING]
> **Changelog v4 → v5 — 3 lỗi kỹ thuật đã sửa qua 5 lượt phản biện:**
>
> | # | Lỗi v4 | Sửa v5 | Nguồn phát hiện |
> | :--- | :--- | :--- | :--- |
> | 1 | Auto-approve mô tả mơ hồ — "Lead Agent kiểm tra" có thể hiểu LLM tự quyết | Hàm `_is_auto_approvable()` thuần Python, 5 điều kiện boolean, chạy ở service layer | Sonnet lượt 1 + Antigravity |
> | 2 | Bridge dùng `asyncio.gather()` + try/except nuốt exception trong hàm con | `asyncio.wait(FIRST_COMPLETED)`, bỏ try/except trong hàm con, cancel pending có timeout 2s, guard `if pending:` | Sonnet lượt 2–4 + Antigravity |
> | 3 | Bỏ cap 2KB cho auto-approve mà không có lý do | Giữ lại cap 2KB (`len(body.encode("utf-8")) > 2048`) | Sonnet lượt 1 |

---

## Quyết định Kiến trúc (9 điểm chốt)

| # | Quyết định | Chi tiết |
| :--- | :--- | :--- |
| 1 | Bảo toàn Single Writer Policy | Loại bỏ hoàn toàn `hermes_direct_execute`. Không mở path ghi bypass queue |
| 2 | Auto-approve bằng hard-coded rules | Hàm `_is_auto_approvable()` trong `hermes_approval.py`, 5 điều kiện boolean. Lead Agent **không tham gia** quyết định |
| 3 | Giữ cap 2KB cho note auto-approve | `len(body.encode("utf-8")) > 2048` → giữ pending |
| 4 | Bridge dùng `asyncio.wait(FIRST_COMPLETED)` | Cover cả exception-based failures và silent-return (uvicorn/fastmcp return `None`) |
| 5 | Bỏ try/except trong `run_http()`/`run_mcp()` | Exception bay tự nhiên lên `asyncio.wait`. Không catch, không nuốt |
| 6 | Cancel pending với timeout + guard | `asyncio.wait(pending, timeout=2.0)` có guard `if pending:` chống `ValueError` khi empty set |
| 7 | Bridge JSON: Regex → Pydantic | Parse bằng regex trích JSON khỏi text rác LLM, validate bằng Pydantic schema |
| 8 | Startup validation | Ping upstream 8642 + token check khi Bridge khởi động |
| 9 | Dynamic ports | `HERMES_BRIDGE_PORT`, `HERMES_API_PORT` qua env vars |

---

## Proposed Changes

### Component 1: Hybrid MCP Bridge

#### [NEW] [hermes_mcp_bridge.py](file:///D:/ai-agents/OpenAgentd/scripts/hermes_mcp_bridge.py)

Script Python chạy song song **stdio MCP server** (lifecycle, được MCPManager quản lý) và **HTTP adapter** (data endpoints, để `HttpHermesClient` gọi tới).

**Kiến trúc runtime:**

```
OpenAgentd (MCPManager)
    ↕ stdio (MCP protocol)
    hermes_mcp_bridge.py
        ├── MCP Server (fastmcp) → lifecycle: health, start, stop
        └── HTTP Server (uvicorn) → data: /v1/write-intents, /v1/query, /v1/skill-drafts
                ↓ HTTP
        hermes-agent gateway (port 8642)
```

**Bridge main loop — `asyncio.wait(FIRST_COMPLETED)`:**

```python
import asyncio
import os

from loguru import logger

async def run_http():
    """Start HTTP adapter. No try/except — exceptions propagate to run_bridge."""
    import uvicorn
    config = uvicorn.Config(app, host="127.0.0.1", port=BRIDGE_PORT)
    await uvicorn.Server(config).serve()

async def run_mcp():
    """Start MCP stdio server. No try/except — exceptions propagate to run_bridge."""
    await mcp.run_async()

async def run_bridge():
    """Run both servers as an atomic unit. Any exit = full process crash."""
    tasks = [
        asyncio.create_task(run_http()),
        asyncio.create_task(run_mcp()),
    ]
    try:
        done, pending = await asyncio.wait(tasks, return_when=asyncio.FIRST_COMPLETED)

        # Any task completing (raise or return) is a failure — both must run indefinitely
        for task in done:
            exc = task.exception()
            if exc:
                logger.error("Bridge component failed with exception: {}", exc)
            else:
                # Both tasks may complete in the same event loop iteration,
                # resulting in a normal return. This is still a failure since
                # servers should never stop on their own.
                logger.error("Bridge component exited unexpectedly without error")

        # Cancel surviving tasks with timeout — cooperative cancellation
        # may hang if the task is in a blocking call that doesn't check
        # CancelledError. Timeout ensures we don't block os._exit().
        for task in pending:
            task.cancel()
        if pending:
            try:
                await asyncio.wait(pending, timeout=2.0)
            except Exception:
                pass

    except Exception as exc:
        logger.error("Bridge startup failed: {}", exc)

    os._exit(1)
```

> [!IMPORTANT]
> **Tại sao `asyncio.wait(FIRST_COMPLETED)` thay vì `asyncio.gather()`:**
>
> Yêu cầu hệ thống: "cả 2 task phải sống vô thời hạn, bất kỳ task nào kết thúc = system failure".
>
> - `gather()` semantics: "đợi tất cả hoàn thành, raise nếu có raise" — **sai khớp** vì nó coi return thành công là OK, nhưng trong domain này return = failure (server không được phép tự kết thúc).
> - `wait(FIRST_COMPLETED)` semantics: "trả về ngay khi bất kỳ task nào hoàn thành" — **đúng khớp** vì mọi completion (raise hay return) đều là tín hiệu cần xử lý.
>
> Ví dụ cụ thể: uvicorn's `Server.serve()` return `None` khi `should_exit` được set (documented API). FastMCP's `run_async()` return khi stdin EOF (pipe đóng). Cả hai là exit paths bình thường, không raise exception — `gather()` không detect được.

**JSON Parser + Pydantic Validation:**

```python
import re
import json
from pydantic import BaseModel, ValidationError

_JSON_BLOCK_RE = re.compile(r"```json\s*\n(.*?)\n\s*```", re.DOTALL)
_JSON_OBJECT_RE = re.compile(r"\{.*\}", re.DOTALL)

def extract_and_validate(raw_text: str, schema: type[BaseModel]) -> BaseModel:
    """Two-layer parse: regex extraction → Pydantic validation."""
    # Layer 1: Extract JSON from LLM text garbage
    match = _JSON_BLOCK_RE.search(raw_text)
    if match:
        json_str = match.group(1)
    else:
        match = _JSON_OBJECT_RE.search(raw_text)
        if not match:
            raise ValueError("No JSON object found in response")
        json_str = match.group(0)

    # Layer 2: Parse + validate
    try:
        data = json.loads(json_str)
    except json.JSONDecodeError as exc:
        raise ValueError(f"Invalid JSON: {exc}") from exc

    try:
        return schema.model_validate(data)
    except ValidationError as exc:
        raise ValueError(f"Schema validation failed: {exc}") from exc
```

**Startup Validation:**

```python
async def validate_upstream():
    """Ping hermes-agent gateway on startup. Fail fast if misconfigured."""
    try:
        async with httpx.AsyncClient(timeout=5.0) as client:
            resp = await client.get(f"{HERMES_API_URL}/v1/health",
                                     headers={"X-API-Key": HERMES_API_KEY})
            resp.raise_for_status()
    except httpx.HTTPStatusError as exc:
        if exc.response.status_code == 401:
            logger.error("HERMES_API_KEY is invalid or missing. Check ~/.hermes/.env")
        raise SystemExit(1)
    except httpx.ConnectError:
        logger.error("Cannot reach hermes-agent at {}. Is 'hermes gateway' running?",
                     HERMES_API_URL)
        raise SystemExit(1)
```

---

### Component 2: MCP Config & Settings

#### [MODIFY] [mcp.json](file:///D:/ai-agents/OpenAgentd/.openagentd/config/mcp.json)

Thêm entry cho hermes-bridge:

```json
{
  "servers": {
    "hermes-bridge": {
      "transport": "stdio",
      "command": "uv",
      "args": ["run", "python", "scripts/hermes_mcp_bridge.py"],
      "env": {
        "HERMES_BRIDGE_PORT": "${HERMES_BRIDGE_PORT}",
        "HERMES_API_URL": "${HERMES_API_URL}",
        "HERMES_API_KEY": "${HERMES_API_KEY}"
      },
      "enabled": true
    }
  }
}
```

> [!CAUTION]
> `OpenAgentd` chỉ resolve `${VAR}` và không hỗ trợ bash-style `${VAR:-default}`. Giá trị mặc định phải được cung cấp bởi code/configuration của Bridge (ví dụ `8090` và `http://localhost:8642`), không được nhúng trong `mcp.json`.

**Startup retry reconciliation:** trước khi Bridge trả lỗi cho watchdog, `validate_upstream()` phải retry `ConnectError` tối đa ba lần, cách nhau hai giây; `401 Unauthorized` phải fail ngay. Việc này khớp với mục tiêu chống crash loop của kế hoạch tích hợp toàn diện. Khi dọn các task bị cancel, bắt `BaseException` quanh bước wait timeout nếu có xử lý exception, vì `asyncio.CancelledError` không kế thừa từ `Exception`.

#### [MODIFY] [config.py](file:///D:/ai-agents/OpenAgentd/app/core/config.py)

Thêm 1 setting:

```python
# Hermes auto-approve kill switch. When False, all proposals stay pending
# regardless of risk level. Does not affect manual approve/reject.
OPENAGENTD_HERMES_AUTO_APPROVE_ENABLED: bool = True
```

Cập nhật `OPENAGENTD_HERMES_BASE_URL` default trỏ về Bridge port:

```python
OPENAGENTD_HERMES_BASE_URL: str = "http://localhost:8090"
```

---

### Component 3: Auto-Approve Policy

#### [MODIFY] [hermes_approval.py](file:///D:/ai-agents/OpenAgentd/app/services/hermes_approval.py)

Thêm hàm `_is_auto_approvable()` và tích hợp vào `enqueue()`:

```python
# --- Constants ---
AUTO_APPROVE_FOLDERS = frozenset({"00-inbox", "10-sources"})
AUTO_APPROVE_MAX_BODY_BYTES = 2048

def _is_auto_approvable(intent: HermesIntentProposal) -> bool:
    """Hard-coded auto-approve rules. No LLM involvement.

    All 5 conditions are code-generated booleans/values from
    _normalize_intent() in hermes.py. None depend on LLM output.
    """
    if intent.folder not in AUTO_APPROVE_FOLDERS:
        return False
    if intent.exists_conflict:
        return False
    if intent.invalid_reason is not None:
        return False
    if intent.body_truncated:
        return False
    if len(intent.body.encode("utf-8")) > AUTO_APPROVE_MAX_BODY_BYTES:
        return False
    return True
```

> [!IMPORTANT]
> **Tại sao 5 điều kiện này là "rule cứng" chứ không phải LLM-decided:**
>
> | Điều kiện | Nguồn giá trị | LLM liên quan? |
> | :--- | :--- | :--- |
> | `intent.folder` | String từ JSON, so sánh `in frozenset` | ❌ |
> | `intent.exists_conflict` | `validate_vault_note_path().exists()` — filesystem check | ❌ |
> | `intent.invalid_reason` | `_normalize_intent()` code validation | ❌ |
> | `intent.body_truncated` | `len(body) > body_limit` — integer comparison | ❌ |
> | `len(body.encode()) > 2048` | Byte-length check | ❌ |

**Tích hợp vào `enqueue()`:**

```python
async def enqueue(
    self,
    session_id: str,
    intents: list[HermesIntentProposal],
    *,
    approver: str = "agent:auto",
) -> HermesEnqueueResult:
    """Add valid Hermes intents to the queue for one session.

    If auto-approve is enabled, intents passing _is_auto_approvable()
    are approved immediately during enqueue. The agent receives both
    auto-approved results and remaining pending_ids.
    """
    from app.core.config import settings

    async with self._lock:
        auto_approved: list[PendingHermesIntent] = []
        pending: list[PendingHermesIntent] = []

        for intent in intents:
            entry = PendingHermesIntent(
                pending_id=str(uuid7()),
                session_id=session_id,
                intent=intent,
            )
            self._entries[entry.pending_id] = entry

            if (
                settings.OPENAGENTD_HERMES_AUTO_APPROVE_ENABLED
                and _is_auto_approvable(intent)
            ):
                # Auto-approve: write through gatekeeper immediately
                try:
                    result = await self._write_intent_locked(entry, approver=approver)
                    entry.status = "approved"
                    entry.result_path = result.path
                    entry.updated_at = datetime.now(UTC)
                    auto_approved.append(entry)
                    logger.info(
                        "hermes_auto_approved pending_id={} path={}",
                        entry.pending_id,
                        result.path,
                    )
                except Exception as exc:
                    # Auto-approve failed — fall back to pending
                    logger.warning(
                        "hermes_auto_approve_failed pending_id={} error={}",
                        entry.pending_id,
                        exc,
                    )
                    pending.append(entry)
            else:
                pending.append(entry)

        evicted_count = self._evict_oldest_pending_locked(session_id)
        return HermesEnqueueResult(
            entries=pending,  # Only pending entries need agent attention
            auto_approved=auto_approved,
            evicted_count=evicted_count,
        )
```

`HermesEnqueueResult` cần thêm field `auto_approved`:

```python
@dataclass(frozen=True)
class HermesEnqueueResult:
    """Result of enqueueing Hermes intents."""
    entries: list[PendingHermesIntent]
    auto_approved: list[PendingHermesIntent] = field(default_factory=list)
    evicted_count: int = 0
```

**Cập nhật `hermes_propose` tool output** để hiển thị auto-approved results:

Trong [hermes_propose.py](file:///D:/ai-agents/OpenAgentd/app/agent/tools/builtin/hermes_propose.py), sửa `_format_proposal()` thêm section `auto_approved` vào JSON output để agent biết note nào đã được ghi tự động.

---

### Component 4: Tests (13 cases)

#### [NEW] [test_hermes_mcp_bridge.py](file:///D:/ai-agents/OpenAgentd/tests/scripts/test_hermes_mcp_bridge.py)

**Bridge Lifecycle Tests (5 cases):**

| # | Test case | Assert |
| :--- | :--- | :--- |
| 1 | `run_http()` raise exception | `os._exit(1)` được gọi |
| 2 | `run_mcp()` raise exception | `os._exit(1)` được gọi |
| 3 | `run_http()` return `None` (silent exit — uvicorn `should_exit`) | `os._exit(1)` được gọi |
| 4 | 1 task raise, 1 task pending | (a) `task.cancel()` gọi trên pending task, (b) `os._exit(1)` gọi — **2 assertion tách biệt** |
| 5 | Cả 2 task complete cùng event loop tick (pending = empty set) | Không `ValueError`, `os._exit(1)` gọi đúng 1 lần |

> [!NOTE]
> **Test case 4** cần 2 assertion tách biệt: nếu chỉ assert `os._exit(1)` ở cuối, test vẫn pass khi logic cancel bị xóa nhầm trong refactor.
>
> **Test case 5** bảo vệ guard `if pending:` — nếu guard bị xóa, `asyncio.wait([], timeout=2.0)` raise `ValueError`.

**Bridge Parser Tests (2 cases):**

| # | Test case | Assert |
| :--- | :--- | :--- |
| 6 | LLM text rác bọc `````json ... ````` kẹp JSON hợp lệ | `extract_and_validate()` trả về đúng Pydantic model |
| 7 | JSON hợp lệ nhưng thiếu required field (`folder`) | `extract_and_validate()` raise `ValueError` với message chứa "validation failed" |

#### [NEW] [test_hermes_auto_approve.py](file:///D:/ai-agents/OpenAgentd/tests/services/test_hermes_auto_approve.py)

**Auto-Approve Policy Tests (6 cases):**

| # | Test case | Nhánh `_is_auto_approvable()` | Assert |
| :--- | :--- | :--- | :--- |
| 8 | `00-inbox`, no conflict, no invalid, no truncated, body < 2KB | Happy path (all 5 pass) | `_is_auto_approvable() == True`, entry `status == "approved"` |
| 9 | `50-decisions` (folder ngoài whitelist) | `folder not in whitelist` | `_is_auto_approvable() == False`, entry `status == "pending"` |
| 10 | `00-inbox` + `exists_conflict=True` | `exists_conflict` | `_is_auto_approvable() == False`, entry `status == "pending"` |
| 11 | `00-inbox` + `invalid_reason="path traversal"` | `invalid_reason is not None` | `_is_auto_approvable() == False`, entry `status == "pending"` |
| 12 | `00-inbox` + `body_truncated=True` | `body_truncated` | `_is_auto_approvable() == False`, entry `status == "pending"` |
| 13 | `00-inbox`, no conflict, no truncated, body = 2049 bytes UTF-8 | `len(body.encode()) > 2048` | `_is_auto_approvable() == False`, entry `status == "pending"` |

> [!IMPORTANT]
> **Cases 8–13 phủ đủ 5/5 nhánh `if` trong `_is_auto_approvable()`:**
> - Happy path: case 8
> - `folder` fail: case 9
> - `exists_conflict` fail: case 10
> - `invalid_reason` fail: case 11
> - `body_truncated` fail: case 12
> - `body size` fail: case 13
>
> Mỗi nhánh có test riêng để refactor sau không xóa nhầm guard mà CI vẫn xanh.

---

## ADR Amendment

> [!CAUTION]
> **ADR-001 D8 Amendment cần thiết trước khi merge:**
>
> Quy tắc D8 gốc: *"Hermes never writes directly"*.
>
> Quy tắc D8 sửa đổi: *"Hermes never writes directly. Auto-approve is a code-enforced fast path within the approval queue — it does not bypass the queue, it accelerates queue processing for intents passing deterministic rules. The rules are defined in `_is_auto_approvable()` and may only be modified via code change + test update."*
>
> Ghi lại bằng ADR mới (ADR-XXX) hoặc amendment trong ADR-001, không patch lặng lẽ.

---

## Verification Plan

### Automated Tests

```bash
# 1. Bridge lifecycle + parser tests
uv run pytest tests/scripts/test_hermes_mcp_bridge.py --no-cov -v

# 2. Auto-approve policy tests (all 6 cases covering 5/5 branches)
uv run pytest tests/services/test_hermes_auto_approve.py --no-cov -v

# 3. Regression — existing Hermes tests must still pass
uv run pytest tests/services/test_hermes.py tests/services/test_hermes_approval.py --no-cov -q

# 4. Lint & type check modified files
uv run ruff check scripts/hermes_mcp_bridge.py app/core/config.py app/services/hermes_approval.py
uv run ruff format --check scripts/hermes_mcp_bridge.py app/core/config.py app/services/hermes_approval.py
```

### Manual Verification

1. Khởi chạy `hermes gateway` trên cổng `8642` với API Key.
2. Khởi chạy OpenAgentd với Bridge được bật. Xác minh log console không có cảnh báo lệch Key.
3. Yêu cầu Lead Agent lưu một bookmark nhanh → Kiểm tra log xem note được auto-approve (`hermes_auto_approved`) và ghi vào vault.
4. Yêu cầu Lead Agent viết tài liệu kiến trúc vào `50-decisions` → Xác minh note giữ lại `pending` trong queue.
5. Kill HTTP server của Bridge (giả lập crash) → Xác minh toàn bộ Bridge process sập và MCPManager watchdog restart nó.
