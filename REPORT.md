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

`lab11/feasible.py: is_feasible` loại kịch bản nếu vi phạm một trong các điều kiện:

| Luật | Căn cứ |
|---|---|
| (pi/cutDur)² × 3.5/2 ≤ mu × 9.81; cutDur phải dương | Chuyển làn cosine rộng 3.5 m có gia tốc ngang cực đại như vế trái. Gia tốc không được vượt lực bám ngang quy đổi trên một đơn vị khối lượng. |
| cutBrake ≤ mu × 9.81 | Phanh sau khi nhập làn không được vượt giới hạn bám lốp trong mô hình. |
| pedV ≤ 2.5 m/s | Giới hạn người đi bộ của bài; tốc độ cao hơn không thuộc phạm vi người đi bộ mà lab mô hình hóa. |
| 0 < cutV ≤ 22 m/s | Loại xe chuyển làn đứng yên/chạy ngược và tốc độ vượt giới hạn miền kịch bản (~79.2 km/h). Đây là giới hạn mô hình, không phải định luật vật lý phổ quát. |

Mô hình kiểm tra bám ngang và dọc riêng, chưa kiểm tra vòng tròn ma sát khi đồng thời chuyển làn/phanh. CEM tối đa hóa nguy hiểm nên có xu hướng chọn chuyển làn nhanh, ma sát thấp, phanh mạnh. Bộ lọc phải chạy trước mô phỏng và elite; nếu tính cả chuyển làn/phanh không thể thực hiện là va chạm, xác suất sẽ bị thổi lên và proposal sẽ tập trung sai miền. `evidence/hunt.json` (mỗi cuộc săn 8×300 = 2.400 lần rút):

| Cuộc săn | Bất khả thi bị loại (`infeasible`) | Kiểu thực tế tìm thấy (`found`) |
|---|---:|---|
| bus_bypass | 6 | {'bus_bypass': 193} |
| cut_in | 3 | {'bus_bypass': 20} |
| crosswalk_block | 11 | {'bus_bypass': 4, 'crosswalk_block': 39} |
| rear_ended | 8 | {'bus_bypass': 12, 'crosswalk_block': 11} |

`crosswalk_block` sinh nhiều bất khả thi nhất: **11/2.400**. Điểm stopping-room kéo proposal về biên nguy hiểm; đường ướt rút ngắn giới hạn bám, thời điểm/tốc độ chuyển làn và người đi bộ vẫn được lấy mẫu đồng thời. Không phải mọi mẫu có điểm nguy hiểm cao đều hợp vật lý. Các mẫu bị loại không được đưa vào elite, không được tính là va chạm.

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

Hai kiểu tìm được: `bus_bypass` và `crosswalk_block`. Cột `found` ở phần 2 là tổng qua từng cuộc săn; một kịch bản có thể trùng miền giữa các cuộc săn, không cộng số đó thành xác suất.

| Ví dụ sẽ được export | Sự kiện từ kiểm tra riêng `inspections.json` | Trạng thái ở thời điểm sự kiện | Ảnh cần chụp |
|---|---|---|---|
| B-04 | ego–C2, contact, t=12.6 s, frame 252 | ego x=91.438 m, v=4.287 m/s; C2 x=95.852 m, v=0 | `shots/bus_bypass.png` |
| C-04 | ego–P1, block, t=22.8 s, frame 456 | ego x=121.602 m, v=0; P1 x=122 m, z=3.464 m, v=0 | `shots/crosswalk_block.png` |

B-04: C2 muốn vượt xe buýt còn đỗ (`busGo = 16.0152664629 s`, sự kiện ở 12.6 s), nhận một khe và lấn từ E2 sang E1; C2 bị chậm/dừng ngay trong hành lang ego. Ego nhận xe dẫn muộn và đã phanh đến giới hạn đường ướt 4.905 m/s² nhưng vẫn chạm C2.

C-04: P1 tới mép đường và bắt đầu qua khi chấp nhận khe ngắn (`pedGap = 1.7140166217 s`, `pedLook = 0`). Ego phanh nhưng dừng với thân xe tại vạch x=122 m, làm P1 phải dừng trước thân xe. Đây là **block**, không phải người đi bộ đi vào xe. Trong ví dụ này bus đã được lệnh đi từ 12.016 s, trước lúc P1 tới; không quy mọi crosswalk block cho việc bước ra sau một bus còn đỗ.

Không cuộc săn nào tìm được `cut_in` hoặc `rear_ended` đúng kiểu. Chỉ kết luận **chưa tìm được trong ngân sách/điểm nguy hiểm này**, không kết luận chúng không tồn tại. Một cuộc săn chung dễ hội tụ vào `bus_bypass`: chính cuộc săn `cut_in` chỉ tìm 20 bus-bypass; cuộc săn `rear_ended` tìm 12 bus-bypass và 11 crosswalk-block. Săn riêng cho phép tiếp tục thăm dò miền P1 dù kiểu C2 dễ tìm hơn.

**Ảnh chưa có:** công cụ trình duyệt chặn mở viewer offline `file://`. Các số và mô tả trên được kiểm tra từ Python; chưa được đối chiếu bằng ảnh chụp giao diện. Cần mở viewer chương 9, nhập `evidence/viewer.json` khi export xong, chọn B-04/C-04 và chụp đúng frame trong bảng; hướng dẫn ở `evidence/shots/README.md`. Không dùng ảnh mẫu của repo làm ảnh của lần chạy này.

## 6. Sửa planner (bước E + 3D)

`fix.json`: policy được chọn, `fail` và `fail_hi` trên TRAIN, trên TEST (`fail_hi_seen` và `fail_hi`), và của policy gốc trên TEST; vì sao `fail_hi` của bản sửa có thể lớn hơn `fail` của policy gốc dù bản sửa tốt hơn thật? quãng đường và số lần phanh gấp. Một ảnh chụp 3D có **bóng mờ** (policy gốc) cho một kịch bản mà bản sửa tránh được. Có núm chỉnh nào "nghe thì an toàn hơn" nhưng thực tế làm tăng thất bại? Giải thích bằng cách thế giới phản ứng lại.

## 7. Từ mô phỏng ra đường thật

Các phân phối trong `avsim/space.py` là giả định dạy học. Với thị trấn B, `p_bin = [0.30, 0.85, 0.15]`, `cut_gap_median = 12 m` (`space.VARIANTS`, không phải thống kê giao thông thật).

| Giả định | Dữ liệu thay thế và nơi thu thập | Hướng sai lệch nếu giả định sai |
|---|---|---|
| Khe chấp nhận `cutGap` (lognormal, median 12 m, sigma 0.5) | Camera cố định tại điểm xe vượt xe buýt/chuyển làn; theo dõi quỹ đạo, tốc độ tương đối và khe tại lúc bắt đầu chuyển làn, phân tầng theo loại xe và trời mưa | Nếu người thật nhận khe nhỏ thường xuyên hơn, mô hình có thể đánh giá thấp `bus_bypass`/`cut_in`; khe lớn hơn thường làm giảm xung đột. Cần đo đồng thời thời gian chuyển làn, không chỉ một median. |
| 85% người đi bộ ngó qua xe buýt (`pedLook`) và `pedGap` median 4.5 s | Video có góc nhìn vỉa hè tại bến xe buýt; gán nhãn hành động ngó, mức che khuất và thời gian tới xe gần nhất lúc bắt đầu băng qua | Nếu ít người ngó hơn 85%, bước ra khi xe đang gần sẽ phổ biến hơn và nguy cơ `crosswalk_block` có thể bị đánh giá thấp; tương tác đứng chờ/chắn lối khiến hướng tác động tổng thể cần đo lại. |
| Đường ướt 30%, mu = 0.5 khi ướt / 0.8 khi khô | Nhật ký mưa và tình trạng mặt đường trên tuyến, kết hợp thử phanh có kiểm soát với đúng lốp/tải trọng để suy ra phân phối ma sát | Nếu đường thật trơn hơn hoặc tỷ lệ ướt cao hơn, quãng phanh dài hơn và mô hình thường đánh giá thấp nguy cơ. Cũng phải lọc lại tính khả thi: thấp mu làm một số chuyển làn không thể xảy ra. |

Không chuyển xác suất của thị trấn mô phỏng trực tiếp thành mức an toàn ngoài đường trước khi hiệu chỉnh các phân phối và kiểm tra phụ thuộc giữa chúng.

## 8. Giới hạn

1. Mô phỏng chỉ có một con phố, bến xe buýt và một nút giao trong 24 s; xác suất không đại diện cho mọi tuyến đường, mật độ, thời tiết hay hành trình dài.
2. Xe khác dùng IDM và các luật khe chấp nhận cố định; chưa mô hình đầy đủ người lái mất tập trung, phản ứng trễ, nhường nhau linh hoạt hoặc thay đổi ý định.
3. Cảm biến nhận trạng thái gần như hoàn hảo ngoài quy tắc che khuất hình học; chưa có nhiễu định vị, phát hiện hụt hay độ trễ perception. Hazard dùng oracle phanh cũng không đại diện cho perception thật.

Nếu có thêm một tuần: tăng số seed và ngân sách cho các kiểu IS suy biến, kiểm tra độ phủ các vùng thất bại bằng proposal rộng hơn; thu dữ liệu khe/che khuất/ma sát để phân tích độ nhạy; thử TEST độc lập với nhiễu và độ trễ cảm biến. Với kiểu chưa quan sát, muốn cận 3/(0.2n) < 10^-3 cần hơn 15.000 lần rút cho đúng kiểu, không thể chứng nhận bằng 400 mẫu.

## 9. Rắc rối của bạn (bước G + chương 10)

**Thiết kế.** Xe đạp H0 đang đi sát mép phải làn E2, lạng sang E1 để tránh chướng ngại khi ego bắt kịp. Cảnh cụ thể `hazard.json`: x=10 m, v=4 m/s, d=0.5 m, swerve=3.2 m, swDur=2.3 s, trig=12 m. Đây là cảnh do tôi dựng từ schema của studio và kiểm tra bằng Python; chưa thao tác dựng trong giao diện chương 10 vì công cụ trình duyệt không mở được file offline. File giữ định dạng `avsim-studio/1`, đầy đủ params/policy/hazards để nhập vào viewer.

**Giả thuyết.** Planner coi một tác nhân là xe dẫn dựa trên khoảng lệch ngang hiện tại, nhưng chưa dự đoán toàn bộ hành lang quét của xe đạp đang lạng. Ego có thể giữ tốc độ đến khi xe đạp đã đi gần vào làn, trong khi phanh hết cỡ từ lúc bắt đầu nhìn thấy chuyển động vẫn tránh được.

**Miền thử (`hazard.json: space`, `why`).** x từ -10 đến 30 m, v từ 3.5 đến 4.5 m/s, swerve từ 2.8 đến 3.2 m, swDur từ 2.3 đến 2.8 s, trig từ 10 đến 18 m. Đoạn này nằm trước nút giao x=80 m. Tốc độ tương ứng 12.6–16.2 km/h; gia tốc ngang cực đại của cả miền là pi²×3.2/(2×2.3²) = 2.9851355466 m/s², dưới ngưỡng 3. Uniform là thiết kế phép thử nhạy cảm, chưa phải phân phối đo trên đường.

**Exposure.** Người dùng xác nhận chưa có dữ liệu tần suất. `hazard.json: exposure = null` và `why` nói rõ điều này. Không tạo số ngoài đời từ giả định không có căn cứ. Khi có e lần rắc rối/chuyến, tần suất lỗi là e×P(lỗi | rắc rối); hiện chưa thể so một tần suất mỗi chuyến với `bus_bypass` trên cùng đơn vị.

**Đánh giá.** Chờ `mine.json` đủ 1.500 lần rút để điền bảng baseline/chosen, accepted và khoảng Wilson. Phân loại sẽ tách `avoidable` (phanh tối đa ngay khi nguy hiểm đã bắt đầu và nhìn thấy), `unavoidable` (phanh như vậy vẫn va chạm), `unattributed` (va chạm không quy được cho hazard); `low_speed` là cờ chồng lên các nhóm, không phải nhóm thứ tư để cộng tổng.

**Giới hạn oracle.** `avoidable` là chuẩn lạc quan vì biết tác nhân nguy hiểm ngay khi nhìn thấy và phanh hết lực, nên có thể đếm nhiều hơn số lỗi planner thực tế có thể tránh với perception và phản ứng hữu hạn. Ngược lại chỉ thử phanh, không thử đánh lái, làm số tránh được nhỏ hơn nếu còn một đường thoát bằng lái. `avoidable_foreknown` còn phanh từ lúc hazard bắt đầu dù bị che; chênh lệch với `avoidable` đo ảnh hưởng che khuất.

**Tự lái và ảnh.** Chưa thực hiện tự lái; không nhận kết quả oracle là khả năng lái của mình. Ảnh 3D phải chụp từ viewer khi nhập evidence, ở frame va chạm; chưa có ảnh vì giới hạn công cụ mở file offline.

**Hướng sửa.** Tăng ngưỡng nhận xe lấn làn (`cutinLat`) có thể giúp nhận xe đạp sớm, nhưng phải đánh giá TRAIN/TEST và ảnh hưởng giao thông. Logic cần bổ sung là dự đoán hành lang chuyển động ngang của xe đạp, kiểm tra giao nhau với hành lang ego theo thời gian, phanh sớm nếu khoảng dừng an toàn sắp bị mất; thêm dữ liệu ý định và độ bất định thay vì chỉ nới một ngưỡng khoảng cách.
