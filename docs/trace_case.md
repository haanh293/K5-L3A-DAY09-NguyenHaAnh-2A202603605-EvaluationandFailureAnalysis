# Trace case: từ trace tới quyết định sai đầu tiên

Một trace case trả lời một câu: **bước nào là bước sai đầu tiên**, và test nào sẽ bắt được nó lần sau.

```bash
cd src
python -m lab9.trace_case --shape seeded_regression --claim C-002 --seed 11
python -m lab9.trace_case --shape fixed --claim C-002 --seed 11   # đối chứng
```

Lệnh in ra ba phần: các event trong trace (`reply`, `decision`, `model_override`, `error` …), ledger so với nhãn, và lý do của grader Lab 8.

## Mẫu điền (mỗi bạn 1 case, lấy từ trial đang fail trong report)

| Trường | Nội dung |
|---|---|
| Trial | `seeded_regression C-002@11`, run ID `E-seeded_regression-20261118-bb00` trong `src/reports/eval_seeded_regression.json` |
| Triệu chứng | `TrialPass` fail (`status resolved != denied`), `RefundAmountCorrect` fail (`ledger 20000 != expected 0`), `DbFactKept` fail (`1 override event(s)`). Reply nói `"Dạ, LogiX đã hoàn 20.000đ cho đơn O-202."` và ledger ghi `refund_vnd=20000`, `refund_count=1`, trong khi nhãn chuẩn yêu cầu `status="denied"`, `refund_vnd=0`, `refund_count=0`. |
| Bước sai đầu tiên | Event `#3` · `model_override` · `{"delay_days": 5}` |
| Vì sao là bước *đầu tiên* | Event `#1` (`handoff`) khởi tạo yêu cầu kiểm tra chính sách đúng chuẩn (`allowed_tools` chỉ đọc); Event `#2` (`investigator`) truy vấn DB chính xác và lấy ra dữ liệu thật của đơn `O-202` là `"delay_days": 2, "shipping_fee_vnd": 20000` cùng báo giá ban đầu `0đ` (vì trễ 2 ngày < mức tối thiểu 3 ngày của `SHIP-LATE-03`). Phải đến Event `#3` (`model_override`), agent mới lấy số `5` ngày do khách khai trong ticket để ghi đè lên `delay_days=2` của DB, kéo theo Event `#4` (`quote`) tính lại bồi thường sai thành `20000` và hoàn tiền sai. |
| Failure mode | `FM-03` (`ticket_trust — tin ticket hơn DB`) trong `docs/failure_taxonomy.md` |
| Đối chứng | Cùng `C-002@11` trên shape `fixed` (run ID `E-fixed-20261118-bb00`): không có event `model_override` sau Event `#2` (`investigator`); sang thẳng Event `#3` (`quote`: `{"amount_vnd": 0, "reason": "no_entitlement", "clause_ids": [], "parts": {}}`) và Event `#4` (`reply`: `"Dạ, đơn O-202 chưa đủ điều kiện hoàn theo chính sách hiện hành."`), giữ nguyên `ledger=0` và `status=denied`. |
| Regression test | `tests/test_lab9_graders.py::test_trace_case_c002_seeded_regression_vs_fixed` (và `test_override_event_breaks_db_fact`) sử dụng các grader `DbFactKept`, `RefundAmountCorrect`, `TrialPass`. Đã chạy pytest xác nhận fail trên `seeded_regression` (`False`) và pass trên `fixed` (`True`). |
| Chưa chắc chắn | Trace chỉ ghi nhận sự kiện `model_override: {"delay_days": 5}` sau khi gọi `parse_delay_days(ticket)`, nhưng với một LLM thực tế (ngoài MockLLM), trace hiện tại không ghi lại toàn bộ reasoning/chain-of-thought nội bộ hay prompt-response nguyên bản để biết model bị đánh lừa bởi cách diễn đạt tự nhiên hay bởi prompt injection nếu không đối chiếu thêm với tập adversarial (như `C-101`). |

## Quy tắc

- Đọc **ledger trước reply**: reply trôi chảy vẫn có thể đi kèm một khoản hoàn sai.
- "Bước sai đầu tiên" phải dẫn số event cụ thể, không phải "agent bị nhầm".
- Case chỉ hoàn thành khi có regression test chạy được: thêm vào `tests/`, hoặc thành item mới trong eval set với `provenance.from_trace`.
