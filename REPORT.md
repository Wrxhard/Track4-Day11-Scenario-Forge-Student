MSSV: 2A202602552

# Lab 11 · Scenario Forge — báo cáo

Thị trấn **B** (`evidence/meta.json: _run.town`); MSSV gốc: `2A202602552`.

Hoàn thiện 10 TODO; `python -m pytest -q` đạt **41 passed in 70.54s**, gồm parity Python–JavaScript. Môi trường: numpy 2.3.2, pytest 8.4.2, Node v22.18.0. Evidence chạy đủ ngân sách, không dùng `--quick`. Báo cáo được cập nhật và commit theo từng checkpoint.

MSSV có chữ A, khác định dạng chỉ chữ số mà starter đọc từ REPORT. Giữ nguyên mã đúng và dùng biến môi trường có sẵn của grader, không sửa runner hay seed:

```powershell
$env:LAB11_MSSV = '2A202602552'
python run_all.py all
```

## 1. Phát lại log có trung thực không? (bước A)

`evidence/replay.json`: `n = 400`, `dropped_human_crash_logs = 0`, `n_used = 400`, `crash_in_both = 0`.

| Môi trường | Tổng va chạm | front | rear | side |
|---|---:|---:|---:|---:|
| Phát lại (`replay`) | 121 | 0 | 109 | 12 |
| Phản ứng (`reactive`) | 1 | 1 | 0 | 0 |

109/121 va chạm phát lại là từ phía sau. Planner có thể phanh sớm hơn người lái trong log, nhưng xe sau vẫn đi đúng quỹ đạo đã ghi: nó không phản ứng với vị trí và vận tốc mới của ego. Vì vậy va chạm phản ánh sự không tương thích của log với planner, không phải lỗi tương ứng trong thế giới phản ứng.

Ví dụ `replay_only[0]`: `scenario = 2`, `contact = rear`; thế giới phản ứng không va chạm, `min_ttc_reactive = 7.460629498692649 s`. Chạy kiểm tra riêng đúng params của ví dụ cho sự kiện ego–TG tại **17.65 s**, frame **353**, `fake = true`: TG trong log đi vào đuôi ego đang chậm lại. Thời gian/frame này lấy từ lần chạy kiểm tra riêng, không phải trường của `replay.json`; ví dụ sẽ được export thành R-01 để xem lại.

## 2. Kịch bản bất khả thi (bước B)

Liệt kê các luật trong `is_feasible` và căn cứ vật lý của từng luật (ma sát, gia tốc, tốc độ người). Trong `hunt.json`, cuộc săn nào sinh nhiều kịch bản bất khả thi nhất, và vì sao CEM lại bị kéo về phía đó? Điều gì xảy ra với ước lượng nếu bạn **đếm** chúng là va chạm?

## 3. Monte Carlo cho bạn biết gì (bước C1)

`evidence/mc.json`: `crashes = 7`, `n = 3000`, `p = 0.00233333333333`; Wilson 95% **[0.0011307176584, 0.0048088764933]**; `infeasible = 17`. Kịch bản bất khả thi không chạy và không tính là thất bại; mẫu số theo định nghĩa pipeline vẫn là 3.000 lần rút tự nhiên.

| Kiểu (`per_mode`) | k | Wilson lo | Wilson hi |
|---|---:|---:|---:|
| bus_bypass | 7 | 0.001130717658 | 0.004808876493 |
| cut_in | 0 | 0 | 0.001278895665 |
| crosswalk_block | 0 | 0 | 0.001278895665 |
| rear_ended | 0 | 0 | 0.001278895665 |
| other | 0 | 0 | 0.001278895665 |

Xấp xỉ độ rộng tương đối ±20%: n ≈ 1.96²(1-p)/(0.2²p) = **41,064** lần chạy; xấp xỉ hiếm n ≈ 96/p ≈ 41,143. Đây chỉ là kế hoạch ngân sách dựa trên p ước lượng từ 7 sự kiện, chưa phải bảo đảm độ chính xác.

`cut_in`, `crosswalk_block`, `rear_ended`, `other` chưa xuất hiện. Không kết luận chúng không thể xảy ra: báo **p ≤ 0.0012788956648** theo cận trên Wilson của 0/3.000, thay vì xác suất bằng 0. Khoảng chuẩn ở k=0 suy biến thành [0,0] nên không dùng.

## 4. Importance sampling, ESS và độ suy biến (bước C3)

Bảng: với từng kiểu và từng seed trong `estimate.json`, ghi `p`, `hits`, `ess`, `top_share`. Ba seed có đồng ý với nhau không, và có đồng ý với Monte Carlo không? Ở đâu `top_share` lớn (> 0,5), và vì sao khi đó con số thực ra chỉ là trọng số của một mẫu? So ngân sách (`budget`) với số lần chạy Monte Carlo cần để có cùng độ chính xác. Phần `other` trong tổng lấy từ đâu, và vì sao không săn nó?

## 5. Các kiểu thất bại (bước C2 + 3D)

Mỗi kiểu bạn tìm thấy: một ảnh chụp trình xem 3D (chương 9) ở đúng lúc va chạm, và một câu mô tả chuỗi sự kiện (ai thấy gì, ai chờ gì). Kiểu nào cuộc săn **không** tìm thấy, và bạn tin điều đó ở mức nào? Nếu chỉ chạy **một** cuộc săn chung cho mọi kiểu thì điều gì xảy ra (gợi ý: `hunt.json: found` của từng cuộc săn)?

## 6. Sửa planner (bước E + 3D)

`fix.json`: policy được chọn, `fail` và `fail_hi` trên TRAIN, trên TEST (`fail_hi_seen` và `fail_hi`), và của policy gốc trên TEST; vì sao `fail_hi` của bản sửa có thể lớn hơn `fail` của policy gốc dù bản sửa tốt hơn thật? quãng đường và số lần phanh gấp. Một ảnh chụp 3D có **bóng mờ** (policy gốc) cho một kịch bản mà bản sửa tránh được. Có núm chỉnh nào "nghe thì an toàn hơn" nhưng thực tế làm tăng thất bại? Giải thích bằng cách thế giới phản ứng lại.

## 7. Từ mô phỏng ra đường thật

Phân phối tự nhiên trong `space.py` là giả định. Chọn **ba** giả định ảnh hưởng nhiều nhất đến con số của bạn (ví dụ khe chấp nhận của xe chuyển làn, tỉ lệ người đi bộ ngó qua xe buýt, ma sát đường ướt). Với mỗi cái: dữ liệu thật nào sẽ thay nó, lấy từ đâu, và nếu nó sai thì con số của bạn sai theo hướng nào?

## 8. Giới hạn

Ba điều mà xác suất bạn báo **không** nói, ví dụ: thế giới chỉ có một con phố; người lái chỉ là IDM; cảm biến hoàn hảo trừ phần bị xe buýt che. Nếu được thêm một tuần, bạn sẽ đo gì tiếp?

## 9. Rắc rối của bạn (bước G + chương 10)

- **Rắc rối:** bạn dựng gì, và ngoài đời nó giống tình huống nào.
- **Căn cứ:** vì sao bạn nghĩ planner sẽ xử lý kém.
- **Giả định:** `space` và `exposure` lấy từ đâu (`hazard.json: why`).
- **Kết quả**, theo `mine.json`:
  - số va chạm `avoidable` (tránh được bằng phanh hết cỡ, biết trước), `unavoidable`, `unattributed`, và `low_speed`, với policy gốc và với policy bạn chọn; `accepted` (bao nhiêu phần `space` của bạn là khả thi);
  - P(lỗi planner | rắc rối) kèm khoảng Wilson;
  - tần suất mỗi chuyến, đặt cạnh tần suất lách xe buýt của bạn.
- **Hình:** một ảnh chương 9 hoặc 10 ở lúc va chạm.
- **Bạn tự lái thì sao:** nếu bạn đã thử lái, bạn tránh được không, và bạn làm gì khác planner?
- **Cách sửa:** núm chỉnh nào sửa được rắc rối này? Nếu không núm nào sửa được, planner cần thêm **logic gì** (mô tả, không cần code)?
