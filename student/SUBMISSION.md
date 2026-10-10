# Báo cáo bài nộp — Day 23 Sensor Fusion Lab

Kết quả chạy trên Waymo với `--fusion compare --seed 0`. Cách nộp: [hướng dẫn nộp](../SUBMISSION.md).

## Thông tin học viên

- Họ tên: Nguyễn Xuân Thành
- MSSV: 2A202602666
- Email: nxthanh16112004@gmail.com
- Link repo (fork): https://github.com/NxThnh/K4-L2L3-DAY23-NguyenXuanThanh-2A202602666-SensorFusion
- Commit hash nộp (`git rev-parse HEAD`): Lấy từ commit cuối cùng và gửi trên LMS (không thể ghi hash của chính commit vào file này).

## Tóm tắt kết quả

- `fusion_mode`: `compare`; `frames`: `[0, 198]`; `segment`: `training_segment-10963653239323173269_1924_000_1944_000_with_camera_labels.tfrecord`; `seed`: `0`. Đây là segment của khóa học có sẵn tại máy; segment mặc định trong cấu hình mẫu không có tại máy.
- Detection: `precision = 0.9951515151515151`, `recall = 0.5757363253856943`, `tp/fp/fn = 821/4/605`.
- LiDAR: `rmse = 0.21534259632015257 m`, `matches = 746`, `sum_sq_err = 34.593835607268524 m²`, `ghost_track_frames = 3`, `missed_gt_frames = 680`, `mean_confirmed_tracks = 3.763819095477387`.
- Fused: `rmse = 0.22073963770124871 m`, `matches = 746`, `sum_sq_err = 36.349586788748994 m²`, `ghost_track_frames = 3`, `missed_gt_frames = 680`, `mean_confirmed_tracks = 3.763819095477387`.
- Camera làm RMSE tăng `0.00539704138109614 m`, vẫn trong giới hạn `0.05 m`. Hai mode có cùng 746 cặp ghép, 3 ghost frame và 680 miss frame; camera chỉ thay đổi ước lượng vị trí của các track đã có, không đổi vòng đời track. `precision_track = 746/(746+3) ≈ 0.996`, `coverage = 746/821 ≈ 0.909` ở cả hai mode. Detector bỏ sót 605 nhãn xe, nên không thể đánh giá chỉ từ RMSE; các miss frame cũng phản ánh giới hạn detection và thời gian xác nhận track.

Chạy từ root repo:

```bash
fusion-run-lab --config student/config/paths.yaml --fusion compare --seed 0
```

`rmse = sqrt(sum_sq_err/matches)` trên vị trí 3D của confirmed tracks ghép
một-một với GT xe trong cửa sổ BEV, gate XY **2.0 m**; `null` nếu không có cặp.
Camera dùng tâm hộp 2D ground-truth FRONT có nhiễu seeded, **không** dùng camera
detector. Kết quả này không đo hiệu quả một perception system độc lập với GT.

`grade_run.log` là JSONL, mỗi `(mode,frame)` đúng một record với các trường:
`mode`, `frame`, `det_tp`, `det_fp`, `det_fn`, `valid_gt`, `confirmed`, `matches`,
`sum_sq_err`, `ghosts`, `misses`. Đảm bảo `matches+ghosts==confirmed` và
`matches+misses==valid_gt`; tổng/trung bình record phải khớp `metrics.json`.
File per-mode `metrics_lidar.json`, `metrics_fused.json`, `grade_run_lidar.log`,
`grade_run_fused.log` được giữ để đối chiếu.

## Giải thích ngắn (Parts E–H — tự viết)

1. LiDAR đo tâm hộp trong không gian 3D: `z` có ba tọa độ mét, `R` là ma trận 3×3 với phương sai của từng trục. Camera đo tâm hộp ảnh: `z` có hai tọa độ pixel, `R` là ma trận 2×2 với phương sai theo hai trục ảnh. EKF dùng cùng trạng thái 6D nhưng hàm quan sát và Jacobian khác nhau.
2. Gating Mahalanobis loại các cặp track–đo không phù hợp trước khi gán. Khoảng cách này chuẩn hóa innovation theo `S = HPHᵀ + R`, nên xét cả độ bất định của track lẫn cảm biến. Ngưỡng χ² phụ thuộc số chiều đo; chỉ các cặp trong FOV và trong gate mới được greedy matching.
3. Đây là **track-then-fuse**: mỗi frame dự báo cùng một tập track một lần, ghép và cập nhật LiDAR, rồi ghép và cập nhật camera trên chính tập track đó. `grade_run.log` ghi cùng chỉ số frame cho hai mode; số liệu hai mode ở phần trên cho thấy tác động của bước camera sau LiDAR. Camera dùng tâm hộp GT 2D có nhiễu, vì vậy kết quả này không đánh giá một camera detector độc lập.
4. Calibration lệch làm `h(x)` dự đoán pixel sai có hệ thống, khiến innovation `z − h(x)` liên tục lệch theo một hướng hoặc lớn bất thường. Khoảng cách Mahalanobis tăng; cặp đúng có thể bị gate loại, hoặc nếu vẫn được nhận thì update kéo trạng thái track sai.
5. `sensor` tường minh giúp `associate_and_update` gọi `manage_tracks` đúng cả khi danh sách đo rỗng. Trong lượt LiDAR, track không được ghép nhưng còn trong FOV bị trừ score, có thể bị xóa; đo LiDAR chưa ghép tạo track mới. Lượt camera chỉ tinh chỉnh EKF, không thay score, không tạo hay xóa track.
6. Track mới có score `1/window` và trạng thái `initialized`. Mỗi LiDAR hit cộng `1/window` (tối đa 1), hit chưa xác nhận chuyển sang `tentative`; score **lớn hơn** `confirmed_threshold` thì thành `confirmed`. Miss trong FOV trừ `1/window` nhưng track đã confirmed vẫn giữ trạng thái đó. Xóa nếu `Pxx` hoặc `Pyy` lớn hơn `max_P`; ngoài ra xóa confirmed nếu score nhỏ hơn `delete_threshold`, hoặc chưa confirmed nếu score không còn dương.

## Bonus (không bắt buộc)

Liệt kê phần bonus đã làm, file bằng chứng trong `student/bonus/` và kết quả chính
(xem [RUBRIC.md](../RUBRIC.md) mục 2). Không làm thì ghi "Không".

- Không.

## Khai báo sử dụng AI (bắt buộc)

Ghi rõ, kể cả khi không dùng ("Không dùng AI"). Xem [RULES.md](../RULES.md) mục 2.

- Công cụ đã dùng (ChatGPT, Copilot, Claude, …): ChatGPT Codex.
- Dùng cho phần nào (hàm, câu hỏi, debug): Triển khai và rà soát Part E–H; chạy kiểm thử, Waymo và hỗ trợ soạn báo cáo.
- Cách bạn đã kiểm tra lại (pytest, chạy Waymo, đối chiếu công thức): `pytest student/tests -q` đạt 128 test; đối chiếu công thức EKF, Mahalanobis, chiếu camera và điều kiện vòng đời track với mã nguồn; chạy `fusion-run-lab --fusion compare --seed 0` và đối chiếu `metrics.json` với `grade_run.log`.

## Checklist nộp

- [x] **Part E–H** trong `workspace/` đã implement; `pytest student/tests -q` đạt 128 test, không có `failed`/`xfailed`.
- [x] Part A–D không sửa.
- [x] Lần chạy chấm điểm: `--fusion compare --seed 0`, `frame_start: 0`, `frame_end: 198`.
- [x] `student/artifacts/metrics*.json` và `student/artifacts/grade_run*.log` do runner sinh ra, không sửa tay.
- [x] Đã điền đủ file này, gồm khai báo AI.
- [x] Không commit dữ liệu Waymo, weights, `paths.yaml`, API key.
- [x] `python tools/check_submission.py` báo `KẾT QUẢ: SẴN SÀNG NỘP` sau commit báo cáo.
- [ ] Đã push và nộp link repo + commit hash trên LMS ([hướng dẫn nộp](../SUBMISSION.md))
