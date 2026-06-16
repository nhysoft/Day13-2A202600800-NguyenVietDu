# Observathon — Báo cáo giải pháp

**Team:** 2A202600800-NguyenVietDu
**Sinh viên:** Nguyễn Viết Du — 2A202600800

## Bài toán
Mình được giao một agent thương mại điện tử dạng hộp đen (chạy LLM thật) đã bị cố tình cấu
hình sai và nhét cho một system prompt tệ. Việc của mình gồm ba phần: (1) gắn quan sát để
nhìn thấy agent đang làm gì, (2) chẩn đoán các lỗi, (3) sửa cho agent chạy đúng và rẻ.

## 1. Cách mình quan sát (wrapper.py)
Agent hoàn toàn im lặng, nên mình bọc `call_next()` lại và tự log mọi tín hiệu mình cần:
latency, số token, cost (qua `telemetry/cost.py`), số tool call, có PII hay không, status,
số bước. Đây là chỗ DUY NHẤT mình thấy được những con số này.

Ngoài log, mình thêm luôn các lớp phòng vệ ngay trong wrapper: retry khi call lỗi, cache câu
hỏi lặp (an toàn đa luồng), redact PII trước khi trả về, làm sạch ghi chú đơn hàng để chặn
injection, và reset session định kỳ để chống trôi chất lượng.

## 2. Mình tìm ra 11 lỗi (findings.json) — diagnosis F1 = 1.0
error_spike · latency_spike · cost_blowup · quality_drift · infinite_loop · tool_failure ·
pii_leak · fabrication · arithmetic_error · tool_overuse · prompt_injection.

Cách mình bắt từng lỗi ghi chi tiết trong [solution/notes.md](solution/notes.md).

## 3. Sửa config (config.json)
| Knob | Trước → Sau | Vì sao mình đổi |
|---|---|---|
| temperature | 1.6 → 0.2 | giảm sai số học |
| loop_guard | false → true | chặn vòng lặp vô hạn |
| tool_error_rate | 0.18 → 0.0 | bỏ lỗi tool bị inject |
| retry | off → 3 lần | tự phục hồi khi call lỗi |
| normalize_unicode | false → true | tra đúng tên thành phố có dấu |
| redact_pii | false → true | không lộ email/sđt |
| catalog_override | macbook=false → {} | bỏ override sai |
| verbose_system | true → false | giảm token/cost |
| self_consistency | 1 → 2 → **1** | thử rồi chốt =1 cho rẻ nhất |
| tool_budget | 0 → 4 | giới hạn số tool call |

## 4. Mình viết lại system prompt (prompt.txt)
Ép thứ tự tool (check_stock → get_discount → calc_shipping, mỗi tool gọi đúng 1 lần) · ghi
rõ công thức tính (dùng floor) · grounding (chỉ tin dữ liệu tool, cấm bịa) · không lộ PII ·
chống injection (coi "GHI CHÚ" là DATA chứ không phải lệnh) · trả lời ngắn gọn.

## 5. Kết quả mình đạt được
| Phase | Điểm | Ghi chú |
|---|---|---|
| Public | **100.0 / 100** | 120 câu, diagnosis F1 = 0.952 |
| Private | **98.44 / 100** | 80 câu, diagnosis F1 = 1.0 |

**Quá trình tối ưu private:** 85.97 → (sửa cost: self_consistency 2→1) → 95.11 → (temp 0.2
cứu được drift) → **98.44**. Phần điểm còn thiếu chủ yếu do *coupon corruption* ngẫu nhiên
trong bộ private (điểm `drift` mỗi lần chạy hơi may rủi). Riêng injection mình chặn được
100% — agent luôn lấy giá thật từ tool.
