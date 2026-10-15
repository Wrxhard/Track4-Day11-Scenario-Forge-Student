# Ảnh 3D cần bổ sung

Chưa có screenshot thật trong thư mục này. Công cụ Browser Use từ chối mở URL `file://` của viewer offline; không dùng ảnh mẫu trong repo làm ảnh kết quả của MSSV này.

Mở `viewer/index.html` trực tiếp bằng trình duyệt trên máy; sang chương 9 · Phòng lab, nhập file JSON trong bảng, chọn run, dừng đúng thời gian và bật bóng mờ khi cần. Chụp cảnh cùng phần đọc sự kiện; lưu PNG vào thư mục này.

| Ảnh | File nhập | Run | Thời gian | Frame |
|---|---|---|---:|---:|
| bus_bypass.png | evidence/viewer.json | B-04 | 12.6 s | 252 |
| crosswalk_block.png | evidence/viewer.json | C-04 | 22.8 s | 456 |
| fix_ghost.png | evidence/fix_comparison.json | FIX-B-04 | 12.6 s | 252 |
| hazard_bike.png | evidence/viewer.json | GB-01 hoặc GC-01 | 5.7 s | 114 |

FIX-B-04 dùng bản sửa làm xe chính, policy gốc làm bóng mờ. Các run hunt B/C của viewer.json chuẩn có chiều ngược lại: xe chính gốc, bóng mờ bản sửa.

Nếu chọn chụp hazard ở chương 10: nhập hazard.json, dừng 6.8 s/frame 136, lúc ego chạm H0 trong cảnh cụ thể. Cảnh này khác run sweep GB-01 nên phải ghi đúng chú thích ảnh.

Ảnh sẽ được thêm trong một commit/push riêng khi có screenshot thật. Không nhận ảnh khác thời điểm hoặc ảnh của người khác làm evidence.
