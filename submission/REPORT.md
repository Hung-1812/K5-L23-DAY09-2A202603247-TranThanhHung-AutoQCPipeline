# Báo cáo Day 09 — <Trần Thành Hùng>, <2A202603247>

## 1. Kết luận trước
Pipeline kết luận **Hold**. Intake quét 224 ảnh, loại 31 và giữ 193 ảnh. Sau khi mô phỏng duyệt 80 ảnh, không còn lỗi bắt buộc nhưng 34/193 ảnh còn cảnh báo (17,62%), vượt ngưỡng Hold 10%. Dataset đã xuất tại `submission/export/clean_v1/` chỉ là bản nháp, chưa được phát hành.

## 2. Lỗi và phân việc
Intake loại 31/224 ảnh: `LEAK_TEST` 11, `DUP_IMAGE_EXACT` 8, `DUP_IMAGE_NEAR` 7, `BAD_SIZE` 3 và `CORRUPT_IMAGE` 2. Ở QC vòng 1 trên 193 ảnh còn lại, báo cáo ghi 36 lỗi bắt buộc và 145 cảnh báo (đếm issue/dòng, không phải số ảnh duy nhất).

| Mã lỗi bắt buộc (v1) | Số lỗi | Xử lý |
|---|---:|---|
| `BAD_CLASS_ID` | 6 | Máy xóa dòng nhãn có ID ngoài danh sách; cần người kiểm tra ảnh nếu việc xóa có thể làm mất vật thể. |
| `BAD_CLASS_NAME` | 2 | Máy chuẩn hóa tên lớp trong `data.yaml` (1 thay đổi). |
| `BAD_FORMAT` | 3 | Máy xóa dòng không hợp lệ (gồm dòng thiếu cột hoặc có `nan`). |
| `BOX_OUT_OF_IMAGE` | 8 | Máy cắt box về biên ảnh (`clip_box`). |
| `BOX_ZERO` | 4 | Máy xóa dòng box rỗng. |
| `DUP_BOX` | 7 | Máy xóa dòng box lặp. |
| `MISSING_LABEL_FILE` | 6 | Máy không tự bịa nhãn; chuyển người gán. |

`03_fixes.csv` ghi 35 thay đổi: 20 `drop_line`, 8 `clip_box`, 6 `needs_label` và 1 `rename_classes`. Cảnh báo v1 gồm `POSSIBLE_MISSING` 53, `POSSIBLE_LOOSE_BOX` 27, `POSSIBLE_FALSE_POS` 24, `POSSIBLE_WRONG_CLASS` 17, `EMPTY_LABEL` 11, `BOX_TINY` 6, `BOX_HUGE` 3 và `CLASS_RARE` 4. Đây là tín hiệu cần kiểm tra, không phải xác nhận lỗi ngữ nghĩa.

Hàng đợi có 121 ảnh, vượt ngân sách 80; người duyệt mô phỏng đã xử lý 80, còn 41 ảnh ngoài ngân sách. Các ảnh có `NEEDS_LABEL` và nhiều cờ ngữ nghĩa được ưu tiên cao: ví dụ `img_0020` đứng đầu (priority 11.524) với `FIXED_DROP_LINE`, `POSSIBLE_MISSING` và `POSSIBLE_WRONG_CLASS`; sáu ảnh thiếu file nhãn có priority 10.

## 3. Calibration
| Tham số | Khởi đầu | Đã chọn | Precision / Recall (trước → sau) | Lý do |
|---|---:|---:|---|---|
| `duplicates.phash_dist` | 2 | 16 | `DUP_IMAGE_NEAR`: 1.00 / 0.50 → 1.00 / 0.83 | Recall trên calib tăng, precision không giảm; chấp nhận bắt thêm near-duplicate nhưng kiểm tra thủ công vì ngưỡng cao có thể gộp nhầm ảnh khác. |
| `second_opinion.conf` | 0.25 | 0.6 | `WRONG_CLASS`: 0.56 / 0.83 → 0.62 / 0.83; `MISSING`: 0.50 / 1.00 → 0.57 / 1.00; `LOOSE_BOX`: 0.29 / 0.40 → 0.20 / 0.20; `FALSE_POS`: 0.36 / 1.00 → 0.36 / 1.00 | Cải thiện precision của sai lớp/thiếu nhãn mà giữ recall hai nhóm này; cảnh báo box lỏng bị bỏ sót nhiều hơn. Chọn 0.6 không nhằm làm hàng đợi vừa ngân sách. |

Một điểm calibration dễ đánh lừa là cỡ mẫu nhỏ (39 ảnh) và không đại diện hết dữ liệu thật. Dù `phash_dist=16` cho precision 1.00 trên calib, khi xem ảnh raw, `img_0011` (tennis) bị ghép với `img_0008` (trượt tuyết), còn `img_0177` (lướt sóng) bị ghép với ảnh người ngồi ăn `img_0113`; đây là các nội dung khác nhau. Trên raw, pipeline đưa ra 7 cờ `DUP_IMAGE_NEAR`, nên cần rà từng cặp thay vì tin precision calib tuyệt đối. Với `conf=0.6`, hàng đợi thực tế vẫn là 121 ảnh; nâng conf chỉ để ép queue xuống 80 sẽ làm giảm recall, không giải quyết nhu cầu duyệt còn thiếu.

## 4. Cửa chặn và điểm mù
Đã chạy kiểm tra riêng với ngân sách duyệt 0: `python -m pipeline.run --budget 0 --reports work/report_gate_verify --work work/run_gate_verify --export-dir work/export_gate_verify`. Kết quả **Rework**, mã thoát **20**; còn 6 lỗi `MISSING_LABEL_FILE` và không tồn tại `work/export_gate_verify/clean_v1`. Gate đã chặn xuất dataset khi còn lỗi bắt buộc.

Điểm mù/hạn chế:
- Hash cảm nhận có thể coi ảnh khác nội dung là trùng, như hai cặp `img_0011`/`img_0008` và `img_0177`/`img_0113` đã nêu; ngược lại biến đổi/cắt ảnh mạnh cũng có thể lọt qua.
- `second_opinion` chỉ so với dự đoán của một model, không xác định model hay nhãn nào đúng. Các `POSSIBLE_*` còn lại sau duyệt (trong đó có 3 `POSSIBLE_WRONG_CLASS`) vẫn cần người xem; cảnh báo không đồng nghĩa lỗi đã xác nhận.
- Box rất nhỏ có thể bị model bỏ qua; `min_area_px` và cỡ mẫu calibration giới hạn khả năng phát hiện vật nhỏ/hiếm.

## 5. Train: raw vs clean
Đã train cùng model **yolo26n**, 20 epoch, hai seed (0, 1), `imgsz=640`, batch 16:

| Biến thể | Ảnh train | mAP50 (TB ± SD) | mAP50-95 (TB ± SD) | Precision (TB ± SD) | Recall (TB ± SD) |
|---|---:|---:|---:|---:|---:|
| raw | 224 | 0.7258 ± 0.0002 | 0.5117 ± 0.0038 | 0.7635 ± 0.0612 | 0.6490 ± 0.0281 |
| clean | 193 | 0.7301 ± 0.0014 | 0.5007 ± 0.0013 | 0.8001 ± 0.0194 | 0.6243 ± 0.0119 |

Chênh mAP50 (clean − raw) là **+0.0043**; lớn hơn SD giữa hai seed của mỗi biến thể, nhưng chỉ có hai seed và tập test 100 ảnh nên chưa đủ cơ sở kết luận clean tốt hơn tổng thể. mAP50-95 của clean thấp hơn 0.0110, recall thấp hơn 0.0247 trong khi precision cao hơn 0.0366. Các giả thuyết cần kiểm tiếp: (1) raw có ảnh trùng/rò rỉ với test/val làm sai lệch đánh giá; (2) clean vẫn còn nhãn sai/thiếu hoặc box lỏng, như các cảnh báo v3; (3) lớp mất cân bằng (class stats: person 550 box, truck 41, bus 31). Cần rà các ảnh/nhãn tương ứng và lặp lại train với thêm seed trước khi kết luận.

## 6. Quyết định và ghi chú AI
Quyết định là **Hold**; để xem xét Release cần người xử lý cảnh báo chưa được duyệt, kiểm tra lại các cặp gần trùng đáng ngờ, rồi chạy lại pipeline và xác nhận tỷ lệ cảnh báo dưới 10% cùng không còn lỗi bắt buộc.

- **Dùng AI:** Có, hỗ trợ đọc/đối chiếu báo cáo và soạn phần diễn giải; không dùng AI để viết bốn module kiểm tra.
- **Prompt chính:** yêu cầu chọn hai ngưỡng từ calibration, kiểm tra intake/hàng đợi và tổng hợp các báo cáo pipeline để hoàn thiện báo cáo.
- **Cách kiểm:** đối chiếu nhận định với `calibration_report.md`, CSV/summary, chạy pipeline với cấu hình hiện tại và mở xem các cặp ảnh bị báo near-duplicate.
- **Một lần AI sai:** nhận xét ban đầu rằng `img_0004` có thể thiếu box ô tô chỉ dựa trên ảnh trực quan; chưa xác nhận bằng nhãn gốc/báo cáo nên không dùng làm kết luận lỗi.
