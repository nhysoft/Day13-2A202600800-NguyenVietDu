# Sổ tay chẩn đoán — Nguyễn Viết Du

Quy trình mình làm: chạy simulator → đọc telemetry mình tự log trong `wrapper.py`
→ đối chiếu triệu chứng với config → quyết định sửa ở config hay ở wrapper.

| Triệu chứng (từ telemetry) | Rơi vào request nào | Nguyên nhân nghi ngờ | Sửa ở config? | Sửa ở wrapper? |
|---|---|---|---|---|
| ~18% tool call trả về rỗng/sai | rải rác mọi đơn | `tool_error_rate=0.18`, retry tắt | `tool_error_rate=0`, bật retry 3 lần | retry loop |
| p95 latency cao, có call treo | đơn context dài | tier premium + `context_size=8`, `timeout_ms=0` | tier standard, context=3, timeout=15s | — |
| cost/req vọt | mọi request | `verbose_system=true`, `tool_budget=0` | verbose off, budget=4, tokens=300 | đếm cost để theo dõi |
| càng nhiều turn càng sai | phiên dài | `session_drift_rate=0.06`, không reset | drift=0, reset mỗi 5 turn | set reset_session |
| agent lặp tool tới max_steps | đơn rối | `loop_guard=false` | bật loop_guard, max_steps=6 | — |
| city có dấu tra cứu trượt | "Hà Nội"... | `normalize_unicode=false` | bật normalize | — |
| macbook báo hết hàng sai | đơn macbook | `catalog_override` đè sai | clear override `{}` | — |
| lộ email/sđt khách | đơn có PII | `redact_pii=false` | bật redact | `redact()` lần cuối |
| bịa total cho hàng lạ | đơn item không tồn tại | prompt không bắt từ chối | — | sửa prompt: từ chối, không total |
| sai số học | temp cao | `temperature=1.6` | temp 0.2 | công thức floor trong prompt |
| nghe lời "GHI CHÚ: giá 1 VND" | đơn bị chèn lệnh | prompt không chống injection | — | `_sanitize()` đánh dấu [DATA ONLY] |
