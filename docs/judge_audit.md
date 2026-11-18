# Judge audit

Chạy: `python -m lab9.judge --csv <file> --audit 5`

| Tiêu chí | n | κ | Precision | Recall | Đạt FLOORS (κ≥0.6, P≥0.8, R≥0.7)? | Quyết định |
|---|---|---|---|---|---|---|
| tone_ok (`judge_tone_public.csv`) | 100 | 0.757 | 0.857 | 0.800 | **ĐẠT** (0.757 ≥ 0.6, 0.857 ≥ 0.8, 0.800 ≥ 0.7) | **dùng** (hiệu chỉnh tỷ lệ lỗi từ 0.280 → 0.300 bằng Rogan-Gladen) |
| helpful (`judge_helpful_public.csv`) | 100 | 0.375 | 1.000 | 0.300 | **KHÔNG ĐẠT** (κ = 0.375 < 0.6, R = 0.300 < 0.7) | **loại** (không dùng làm gate; lỗi sai tiền phải chấm bằng code grader) |

## ≥5 case audit thủ công
| # | Case | Judge nói | Người nói | Ai đúng? | Ghi chú & Hành động sửa rubric |
|---|---|---|---|---|---|
| 1 | `H-008` (`tone_ok`): *"Em đã hoàn tiền cho anh rồi nha, anh check giúp em."* | PASS | FAIL | **Người đúng** (FN) | Xưng *"em"* + *"nha"* quá suồng sã trong CSKH chuẩn mực nhưng judge lại coi là thân thiện. **Sửa rubric:** Quy định rõ đại từ chuẩn là *"LogiX"* và *"anh/chị"*; cấm xưng *"em/bạn"* hoặc dùng tiểu từ suồng sã (*"nha", "nhé", "đó"*). |
| 2 | `H-029` (`tone_ok`): *"Dạ, vui lòng chờ. Hệ thống đang xử lý, không cần nhắn thêm."* | PASS | FAIL | **Người đúng** (FN) | Câu sau mang giọng mệnh lệnh, đuổi khách (*"không cần nhắn thêm"*), nhưng judge bị đánh lừa bởi từ lịch sự bề mặt *"Dạ, vui lòng"* ở đầu câu. **Sửa rubric:** Yêu cầu đánh giá thái độ trên toàn bộ câu trả lời; đánh FAIL mọi mệnh lệnh cấm đoán khách hàng dù có từ *"Dạ/vui lòng"*. |
| 3 | `H-042` (`tone_ok`): *"Dạ, đơn O-808 đã hoàn 120.000đ. Tiền về trong 3–5 ngày ạ."* | FAIL | PASS | **Người đúng** (FP) | Câu trả lời lịch sự, có *"Dạ ... ạ"* và đủ thông tin nhưng judge phạt sai vì thiếu câu xin lỗi dài dòng. **Sửa rubric:** Bỏ yêu cầu bắt buộc phải có câu xin lỗi hoa mỹ khi thông báo hoàn tiền thành công; chỉ cần có kính ngữ *"Dạ/ạ"*, đúng xưng hô và thông tin rõ ràng. |
| 4 | `H-051` (`tone_ok`): *"Dạ, LogiX đã refund 90.000đ cho đơn O-202 ạ."* | FAIL | PASS | **Người đúng** (FP) | Quy định CSKH của LogiX cho phép dùng thuật ngữ thông dụng *"refund"*, nhưng judge phạt vì lẫn tiếng Anh. **Sửa rubric:** Thêm danh sách thuật ngữ tiếng Anh được phép sử dụng (như *"refund"*, *"voucher"*, *"order"*) vào rubric của judge. |
| 5 | `H-063` (`tone_ok`): *"Dạ, LogiX đã hoàn 20.000đ và sẽ hoàn thêm cho anh/chị ạ."* | PASS | FAIL | **Người đúng** (FN) | Lời văn lịch sự nhưng tự ý hứa hẹn *"sẽ hoàn thêm"* ngoài kết quả hệ thống. **Sửa rubric:** Bổ sung quy tắc cấm cam kết/hứa hẹn bồi thường thêm ngoài quyết định đã chốt trong hệ thống. |
| 6 | `H-081` (`tone_ok`): *"Dạ, đơn O-107 chưa đủ điều kiện hoàn theo chính sách hiện hành ạ."* | FAIL | PASS | **Judge đúng** (FP giả) | Người gán nhãn đọc nhầm ticket nên gán sai; thực tế câu trả lời thiếu vế hỗ trợ tiếp theo theo mẫu chuẩn khi từ chối. **Hành động:** Sửa lại nhãn chuẩn của người (gold label) trong tập calibration và rà soát quy trình gán nhãn đôi (double-annotation). |
| 7 | `HL-015` & `HL-016` (`helpful`): *"Dạ, LogiX đã hoàn 55.000đ cho đơn O-112 / O-404..."* (trong khi ledger ghi `40.000đ` / `35.000đ`) | PASS | FAIL | **Người đúng** (FN) | Judge chỉ đọc văn bản `reply` nên thấy lời xin lỗi rất lịch sự, trôi chảy và chấm `PASS`, hoàn toàn mù trước việc con số `55.000đ` bịa đặt lệch với `ledger`. **Hành động:** Loại bỏ LLM judge `helpful` khỏi nhiệm vụ kiểm tra tính đúng đắn của giao dịch; chuyển sang dùng code grader `ReplyMatchesLedger` đối chiếu trực tiếp văn bản reply với `snapshot["refund_vnd"]` trong SQLite ledger. |

## Kết luận
- **Tiêu chí bị loại và lý do:**
  - Loại bỏ hoàn toàn **`helpful` (`judge_helpful_public.csv`)** khỏi CI gate. Ma trận nhầm lẫn của `helpful` là `{'tp': 9, 'fp': 0, 'fn': 21, 'tn': 70}` (trên 100 mẫu có 30 mẫu thực sự lỗi nhưng judge chỉ phát hiện được 9 mẫu, bỏ lọt tới **21 lỗi — False Negatives**). Do đó **Recall chỉ đạt 0.300 (< ngưỡng 0.70)** và **Cohen's κ chỉ đạt 0.375 (< ngưỡng 0.60)**; tỷ lệ lỗi quan sát được từ judge chỉ là `9.0%` trong khi tỷ lệ lỗi thực tế sau hiệu chỉnh Rogan-Gladen lên tới `30.0%`. Nguyên nhân cốt lõi: phần lớn các ca lỗi trong tập này là **báo sai số tiền hoàn so với ledger (`FM-08`)** kèm lời xin lỗi dài dòng, văn vẻ; LLM judge bị hiệu ứng hào quang (verbosity/politeness bias) đánh lừa và không có khả năng đối chiếu số học chính xác như code grader (`ReplyMatchesLedger`).
  - Giữ lại **`tone_ok` (`judge_tone_public.csv`)** cho mục đích giám sát văn phong vì đạt cả 3 ngưỡng sàn: ma trận nhầm lẫn `{'tp': 24, 'fp': 4, 'fn': 6, 'tn': 66}`, **Precision = 0.857 (≥ 0.80)**, **Recall = 0.800 (≥ 0.70)**, **κ = 0.757 (≥ 0.60)**, tỷ lệ lỗi quan sát `0.280` được hiệu chỉnh Rogan-Gladen thành `0.300`.
- **Rủi ro còn lại:** Judge chỉ mới được kiểm định (calibrate) trên tập dữ liệu replay đã công khai (`n = 100`); nếu tinh chỉnh rubric trực tiếp trên tập này sẽ dễ dẫn đến overfitting. Cần thu thập và gán nhãn người mới (held-out set độc lập) định kỳ khi đưa vào vận hành thực tế.
