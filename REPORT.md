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

`evidence/estimate.json: seeds[].per` (mỗi hàng n=600, beta=0.2):

| Kiểu | seed | p / cận khi chưa thấy | hits | ESS | top_share |
|---|---:|---:|---:|---:|---:|
| bus_bypass | 0 | 0.0007417118054 | 42 | 136.09531018 | 0.11196421 |
| cut_in | 0 | chưa thấy; ≤ 0.025 | 0 | 133.68510771 | 0.00000000 |
| crosswalk_block | 0 | 1.928884788e-06 | 63 | 121.35499583 | 0.64968251 |
| rear_ended | 0 | chưa thấy; ≤ 0.025 | 0 | 122.19581281 | 0.00000000 |
| bus_bypass | 1 | 0.001112942203 | 54 | 140.49957035 | 0.24315900 |
| cut_in | 1 | chưa thấy; ≤ 0.025 | 0 | 105.30566683 | 0.00000000 |
| crosswalk_block | 1 | 4.187815365e-06 | 63 | 121.90355441 | 0.73867728 |
| rear_ended | 1 | chưa thấy; ≤ 0.025 | 0 | 110.50852268 | 0.00000000 |
| bus_bypass | 2 | 0.0008876014581 | 49 | 129.49278747 | 0.15385034 |
| cut_in | 2 | chưa thấy; ≤ 0.025 | 0 | 113.72639507 | 0.00000000 |
| crosswalk_block | 2 | 8.197776373e-07 | 73 | 130.12033034 | 0.37327580 |
| rear_ended | 2 | chưa thấy; ≤ 0.025 | 0 | 106.13591015 | 0.00000000 |

Các hàng chưa thấy có `top_share = 0` do không có đóng góp, không phải bằng chứng ước lượng ổn định. Cận đúng là `upper_zero_hits = 3/(0.2×600) = 0.025`; không dùng 3/ESS và không nhận [0,0] trong trường CI xấp xỉ là khoảng xác suất thật.

| seed | `total` (ước lượng điểm) | `lo` | `hi` |
|---|---:|---:|---:|
| 0 | 0.000743640690175 | 0.000390865002166 | 0.00237531204299 |
| 1 | 0.00111713001847 | 0.000389770509916 | 0.00312338519182 |
| 2 | 0.000888421235761 | 0.000425395425363 | 0.00263034271096 |

Tổng qua seed lệch tối đa khoảng 1.502 lần; `bus_bypass` khoảng 1.501 lần; `crosswalk_block` khoảng 5.108 lần. Trung vị tổng 0.000888421235761, thấp hơn MC 2.626 lần. Khoảng tổng IS vẫn giao với Wilson của MC, nhưng ba ước lượng điểm đều thấp hơn cận Wilson dưới của MC; không kết luận hai phương pháp đã đồng ý chính xác.

`crosswalk_block` seed 0 (`top_share = 0.64968251`) và seed 1 (`0.73867728`) vượt 0.5: một mẫu quyết định hơn nửa tổng w×y. Nhiều hits dưới q vẫn có thể có trọng số p/q rất nhỏ; số hits không bảo đảm thông tin dưới p. CI ±1.96se khi đó có thể hẹp sai. Seed 2 top_share thấp hơn cũng không loại trừ một miền va chạm hoàn toàn chưa được q thăm dò. Dùng proposal từ lần elite đầu tiên đủ va chạm giúp hạn chế co q nhưng không chứng minh độ phủ.

`budget = 16800` = 4×(8×300 + 3×600), gồm 9.600 lần săn và 7.200 lần ước lượng. So với khoảng 41.064 lần MC cho mục tiêu ±20% ở phần 3, ngân sách này nhỏ hơn 2.444 lần, nhưng IS hiện **chưa đạt mục tiêu ±20%**: riêng bus-bypass có 1.96se/p khoảng 47.6%, 65.4%, 52.2%.

So cùng sai số chuẩn quan sát, số MC tương đương được suy ra bằng n_MC ≈ p(1-p)/se_IS² (tính từ các trường p/se của từng hàng, không phải lần chạy thật):

| Kiểu | seed | n_MC xấp xỉ |
|---|---:|---:|
| bus_bypass | 0 | 22,880 |
| crosswalk_block | 0 | 1,160,118 |
| bus_bypass | 1 | 8,073 |
| crosswalk_block | 1 | 430,910 |
| bus_bypass | 2 | 15,891 |
| crosswalk_block | 2 | 7,051,769 |

Đây là so sánh chi phí có điều kiện trên ước lượng IS hiện có, không chứng nhận hiệu quả khi proposal bỏ sót vùng thất bại.

`estimate.json: other` lấy từ `mc.json`, `k=0`, `n=3000`, `hi=0.0012788956648`. Báo other chưa quan sát, p ≤ cận này. Không săn other vì không có một cơ chế/điểm nguy hiểm thống nhất; bỏ nó khỏi tổng sẽ làm phân hoạch thất bại không đầy đủ. `seeds[].hi` của runner cộng cận other khi k=0, nhưng không cộng các cận 0.025 của cut-in/rear-ended chưa thấy. Vì vậy bảng khoảng tổng trên là đầu ra xấp xỉ của runner, không phải chứng nhận mọi kiểu hiếm đều đã được khống chế.

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

`evidence/fix.json: chosen.policy = {'cutinLat': 3.0}`. Các núm khác giữ mặc định; `cutinLat` từ 2.15 lên 3.0 m làm planner nhận C2 lấn ngang là xe dẫn sớm hơn. Chọn bằng TRAIN: lọc `progress ≥ 0.95×210.746632755 = 200.209301117 m`, rồi xếp `fail_hi` nhỏ nhất, hòa thì `progress` lớn nhất. `cutinLat = 3.3` có cùng `fail_hi` nhưng progress TRAIN thấp hơn một chút nên không thắng 3.0. Không dùng TEST để đổi lựa chọn và không thêm ứng viên sau khi thấy TEST.

| Nguồn trong fix.json | fail | fail_hi_seen | fail_hi | progress (m) | harsh (frame) | unseen |
|---|---:|---:|---:|---:|---:|---|
| Gốc TRAIN | 0.00114763194093 | 0.00230265907484 | 0.00230265907484 | 210.746632755 | 8.714843750 | không |
| Chọn TRAIN | 5.55067427644e-07 | 1.00312589003e-06 | 1.00312589003e-06 | 210.830298543 | 8.269531250 | bus_bypass |
| Gốc TEST | 0.000697100207626 | 0.00141137377381 | 0.00269026943861 | 214.165453369 | 8.746000000 | không |
| Chọn TEST | 3.92669882226e-07 | 7.15784224511e-07 | 0.038779611449 | 214.087490560 | 8.076333333 | bus_bypass |

TRAIN chỉ sàng lọc vùng policy gốc thất bại. TEST dùng seed mới, săn lại theo từng policy và holdout tự nhiên 3.000 mẫu; vì thế không lấy TRAIN gần như không có va chạm làm chứng nhận. Ở TEST, `chosen_test.crosswalk_block.hits = 60`, `n = 400`, p=3.92669882226e-07; `chosen_test.bus_bypass.hits = 0`, cận `upper_zero_hits = 0.0375`. `other_crashes` của bản sửa có k=0/3.000, Wilson hi=0.0012788956648.

Cận bản sửa là 0.0375 + 7.15784224511e-07 + 0.0012788956648 = 0.038779611449, lớn hơn ước lượng điểm policy gốc dù dữ liệu quan sát cho thấy mức giảm lớn. Phần lớn độ rộng do ngân sách không đủ để chứng nhận một kiểu đã không còn được thấy, không phải bằng chứng policy sửa có thêm 3.75% va chạm. `fail_hi_seen` chỉ mô tả các kiểu có hits, không phải cận đầy đủ cho cả policy.

TEST mất quãng đường 0.036403%, nhỏ hơn 5%. `harsh` đếm frame gia tốc < -3 m/s² theo TODO 9, không đếm số đợt phanh độc lập; trung bình giảm từ 8.746000000 xuống 8.076333333 frame. Progress chỉ tính tới lần va chạm đầu, không thưởng phần chạy sau tai nạn.

Một chỉnh sửa nghe an toàn hơn không luôn giảm thất bại: trên cùng TRAIN, giảm `cautionV` từ 6 xuống 5 m/s làm `table[2].fail = 0.000291481249734` tăng thành `table[0].fail = 0.000490849574377` (~1.684 lần), trong khi progress giảm từ 188.361876 xuống 171.427227 m. Đây là so ước lượng điểm TRAIN, các cận còn giao nhau nên chưa chứng minh mức tăng về mặt thống kê. Cơ chế có thể giải thích là đổi tốc độ ego cũng đổi lúc nó đến khe xe C2, lúc C2 nhập làn và phản ứng của xe sau; thế giới phản ứng không giữ nguyên lịch giao thông khi ta chạy chậm hơn. Không suy ra quan hệ đơn điệu “càng chậm càng an toàn”.

**So sánh 3D:** file bổ sung `evidence/fix_comparison.json` (định dạng avsim-viewer/1) có run **FIX-B-04**, xe chính dùng policy sửa, bóng mờ dùng policy gốc. Chạy kiểm tra riêng cùng params: B-04 gốc chạm C2 ở 12.6 s/frame 252, bản sửa không có sự kiện ego; ngược lại C-04 vẫn block ở 22.8 s/frame 456 dưới cả hai policy (`inspections.json: chosen_events`). Nhập file so sánh riêng ở chương 9, bật bóng mờ, dừng tại 12.6 s và lưu `shots/fix_ghost.png`. Các run hunt trong export chuẩn ghim policy gốc nên bóng mờ là bản sửa; file bổ sung đảo đúng chiều yêu cầu câu này. Đây là kiểm tra bổ sung từ params của cloud, không thay đổi TEST hay lựa chọn policy. Ảnh thật chưa có vì công cụ chặn mở file offline; không dùng ảnh minh họa thay thế.

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

**Kết quả (`evidence/mine.json`).** `world = town`, `conditional_on = your town's traffic`; rút 1.500 lần, `infeasible = 3`, `accepted = 0.998` = 99.8%, n giữ lại 1.497 ở cả hai policy; `overlap_at_start = 0`. `exact.planner = [['ego', 'H0', 6.8]]` khớp cảnh cụ thể trong hazard.json.

| Policy | n | crashes | avoidable | unavoidable | unattributed | low_speed | avoidable_foreknown | P(lỗi planner \| hazard) | Wilson 95% |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---|
| Gốc (`baseline`) | 1497 | 990 | 990 | 0 | 0 | 0 | 990 | 0.661322645291 | [0.636962615345, 0.684856822496] |
| Sửa (`chosen`) | 1497 | 862 | 861 | 1 | 0 | 0 | 861 | 0.575150300601 | [0.549948166502, 0.599967720691] |

`avoidable` là phanh tối đa ngay khi hazard đã bắt đầu **và nhìn thấy**; `unavoidable` là phanh như vậy vẫn va chạm; `unattributed` là va chạm không quy cho hazard. `low_speed` là cờ chồng lên các nhóm, không cộng như nhóm thứ tư. Xác suất lỗi chỉ dùng `avoidable/n`, không dùng tất cả crashes/n.

Số lỗi tránh được giảm 129 (13.030303% tương đối), xác suất giảm 8.617234 điểm phần trăm. Nhưng bản sửa vẫn còn 861 lỗi tránh được và 1 va chạm không tránh được bằng oracle: mở rộng cutinLat giúp một phần, chưa giải quyết dự đoán hành lang lạng của xe đạp. `avoidable_foreknown - avoidable = 0` ở cả hai; trong sweep này oracle biết từ lúc bắt đầu không cứu thêm trường hợp nào so với oracle đợi nhìn thấy.

Ví dụ `baseline.examples[0]`, `chosen.examples[0]` cùng params, sẽ export thành **GB-01/GC-01**: xe đạp bắt đầu và oracle phanh tại 3.65 s; cả hai planner vẫn chạm H0 ở **5.7 s/frame 114**. Tốc độ ego tại tiếp xúc giảm từ 7.667248249 xuống 3.672028303 m/s. Ảnh cần chụp `shots/hazard_bike.png`: chọn GB-01 hoặc GC-01 trong chương 9, dừng ở 5.7 s; hoặc mở cảnh cụ thể trong chương 10 và dừng 6.8 s/frame 136.

**So tần suất mỗi chuyến.** Vì exposure chưa biết, chỉ có f_gốc(e) = 0.661322645291×e và f_sửa(e) = 0.575150300601×e; chưa có f đo ngoài đời. `bus_bypass` có trung vị IS 0.000887601458123 và MC 7/3.000 = 0.00233333333333 trong định nghĩa chuyến mô phỏng của lab. Điểm hòa theo trung vị IS là e ≈ 0.00134216099274 (gốc) hoặc 0.00154325131569 (sửa); theo MC là e ≈ 0.00352828282828 và 0.00405691056911. Đây là **ngưỡng tính toán nếu biết e**, không phải số exposure được ước lượng hay chọn để làm hazard nguy hiểm hơn. So sánh còn phụ thuộc miền uniform thử nghiệm có đại diện cho hazard ngoài đời hay không.

**Giới hạn oracle.** `avoidable` là chuẩn lạc quan vì biết tác nhân nguy hiểm ngay khi nhìn thấy và phanh hết lực, nên có thể đếm nhiều hơn số lỗi planner thực tế có thể tránh với perception và phản ứng hữu hạn. Ngược lại chỉ thử phanh, không thử đánh lái, làm số tránh được nhỏ hơn nếu còn một đường thoát bằng lái. `avoidable_foreknown` còn phanh từ lúc hazard bắt đầu dù bị che; chênh lệch với `avoidable` đo ảnh hưởng che khuất.

**Tự lái và ảnh.** Chưa thực hiện tự lái; không nhận kết quả oracle là khả năng lái của mình. Ảnh 3D phải chụp từ viewer khi nhập evidence, ở frame va chạm; chưa có ảnh vì giới hạn công cụ mở file offline.

**Hướng sửa.** Tăng ngưỡng nhận xe lấn làn (`cutinLat`) có thể giúp nhận xe đạp sớm, nhưng phải đánh giá TRAIN/TEST và ảnh hưởng giao thông. Logic cần bổ sung là dự đoán hành lang chuyển động ngang của xe đạp, kiểm tra giao nhau với hành lang ego theo thời gian, phanh sớm nếu khoảng dừng an toàn sắp bị mất; thêm dữ liệu ý định và độ bất định thay vì chỉ nới một ngưỡng khoảng cách.
