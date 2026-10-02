# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: làm một mình (solo), không có nhóm; mã phòng: ____ (LC điền)
- Thành viên: Lưu Thị Lan Anh, MSSV 2A202602266 (làm solo nên không dùng `TEAMMATES.md`).
- Trạng thái: `executed-by-group` (một người chạy, tự chạy thật trên máy cá nhân; không dùng `provided-results`).
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Lưu Thị Lan Anh; 2026-10-02, chạy xong khoảng 18:34 giờ máy (smoke 11:32–11:34 UTC); Windows 11 Home, Docker Desktop (Linux containers), linux/amd64 native.
- Image tag và image ID; phiên bản repo: `day13-pointpillars:lc-20261001-amd64`; image ID `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`; repo_revision `0831856d921609312d42c7582c366e5a311bb7b1` (`working_tree_dirty: true` theo smoke.json).
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `input/demo.pcd` (KITTI 000008 đã chuyển đổi, CC BY-NC-SA 3.0, 17238 điểm), frame_id `demo`; chạy trên laptop cá nhân; input_sha256 `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`.
- Checkpoint: PointPillars KITTI `epoch_160.pth` có sẵn trong image; sha256 `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`.
- Phạm vi: front-window (không `--full-scene`); score threshold 0.3; giới hạn container 4 CPU / 4 GB, không mạng.
- Giả định kênh thứ tư/intensity và nguồn z_ground: PCD đã bỏ reflectance thật, RGB=0 làm placeholder, adapter dùng kênh hằng theo lớp (không phải intensity phục hồi). `z_ground` = 0.075 m, ước lượng từ chính scan; `z_pcd = z_kitti + 1.73 m`.

## Ba lượt inference thật

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/boxes-demo-delta-0-voxel-0.16.json`, `side-demo-delta-0-voxel-0.16.png`, `summary.csv` | 1 hộp `vehicles` (x≈13.2, y≈-0.5, score 0.32), đáy hộp xuống tới z≈-0.4 m, thấp hơn đường z=0 trên ảnh Side. |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `side-demo-delta-1.73-voxel-0.16.png`, `summary.csv` | 10 vehicles, 2 pedestrian, 1 two-wheels; trải từ x≈2 đến x≈58 m, đáy hầu hết hộp nằm khoảng z≈0 đến 0.4 m, bám cụm điểm. |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `side-demo-delta-1.73-voxel-0.32.png`, `summary.csv` | Cả 6 hộp đều `pedestrian` (hẹp, cao), tập trung x≈9–20 m, 1 hộp ở x≈33.5; không còn hộp `vehicles` hay `two-wheels`. |

- **A/B: thay input trước model có khác dịch cùng một hằng số cho output không? Vì sao?** Có khác nhiều. Nếu chỉ là dịch z thì A phải giống B với mọi hộp lệch đúng 1.73 m; thực tế A chỉ có 1 hộp còn B có 13 hộp (class và vị trí x/y cũng khác). Lý do: PointPillars chạy lại trên input đã dịch. Checkpoint KITTI huấn luyện với sensor cao khoảng 1.73 m so với mặt đất, nên khi delta=0 mặt đất nằm ở z≈0 thay vì z≈-1.73 như model kỳ vọng, và nhiều hộp không được kích hoạt. Việc dịch z trước inference vì vậy làm thay đổi *đầu vào của mạng* (số hộp, vị trí, lớp), khác với dịch hộp sau inference (chỉ đổi z, giữ nguyên số hộp/class/x/y/yaw). Bằng chứng: `summary.csv` A (n_boxes=1) so với B (n_boxes=13); hai ảnh Side. Chỉ đổi delta nên đây là so sánh sạch.
- **B/C: thấy gì khi đổi pillar? Có đủ bằng chứng để nói cấu hình nào tốt hơn không?** Pillar 0.32 m cho 6 hộp so với 13 hộp, toàn bộ là `pedestrian`; các hộp `vehicles` và `two-wheels` của B biến mất. Một số hộp pedestrian của C có vị trí gần hộp của B (ví dụ x≈10.5, y≈4.9 ở C; B có two-wheels x≈10.3, y≈5.2 nên có thể C đổi class của cùng một đối tượng), nhưng một số hộp C (x≈18.7, y≈0.2; x≈34.0, y≈-5.0) không trùng rõ hộp B. Chưa đủ bằng chứng nói cấu hình nào tốt hơn: không có nhãn tham chiếu, nhiều hộp hơn hay ít hộp hơn đều không tự là đúng hơn; C dùng checkpoint train cho pillar 0.16 nên 0.32 là thay biểu diễn đầu vào chứ không phải mô hình đã tối ưu cho nó.
- **Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?** Chỉ dùng cửa sổ phía trước của checkpoint nên vật thể ngoài ROI (phía sau, hai bên xa) không phải bằng chứng model bỏ sót. Ảnh Side là hình chiếu x-z toàn scene nên các hộp có y khác nhau chồng lên nhau (vùng x≈6–11 m có nhiều hộp chồng), không thể suy ra yaw hay tách đối tượng từ một ảnh Side; cần thêm Top/Front và camera.
- **JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?** Không import JSON nào vào CVAT/Robotaxi: dữ liệu KITTI demo, không cùng frame. Trong ba JSON, A (1 hộp, score 0.32 sát ngưỡng) và C (chỉ pedestrian) là yếu nhất; B có nhiều hộp nhưng vẫn cần kiểm class, yaw, kích thước bằng Top/Front và ảnh camera trước khi coi là điểm khởi đầu.

## Ca QC có kiểm soát — không import CVAT

Nguồn: `run-B/boxes-demo-delta-1.73-voxel-0.16.json` (sha256 `2ffb4e85…fcfc`), `z_ground=0.075`, `height_offset=delta+z_ground=1.805 m`.

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 m | Không đổi (bản sao của B) | Không có lỗi z do helper; không dừng. Vẫn chỉ là output model chưa kiểm, không phải đáp án. | `qc-cases/side-correct.png` giống hệt `run-B` Side; mọi hộp bám cụm điểm. |
| case-batch-z | 13 / 13 | −1.805 m cho mọi hộp | Không; chỉ z đổi | **Dừng batch**, không sửa tay từng hộp; báo LC kiểm phép chuyển z/regenerate prediction. | `qc-cases/side-batch-z.png`: toàn bộ hộp bị chìm dưới đường z=0 cùng một lượng, trong khi điểm cloud giữ nguyên; `manifest.json` ghi mọi hộp bị trừ `delta + z_ground`. |
| case-one-box-z | 1 / 13 | −1.805 m (chỉ hộp đầu tiên, vehicles x≈8.1, y≈1.2) | Không; chỉ z của hộp đó đổi | **Kiểm từng hộp**, không kết luận lỗi pipeline; sửa/loại hộp bị lệch, 12 hộp còn lại bám cụm điểm. | `qc-cases/side-one-box-z.png`: một hộp đỏ (x≈6–10 m) chìm xuống z≈-1.65…-0.1 trong khi 12 hộp còn lại giữ như B. |

Các ca được helper `pipeline-qc-cases.py` tạo từ prediction thật B bằng biến đổi có chủ đích; không phải kết quả inference riêng hay nhãn đúng, không import vào CVAT.

## Nhận xét cá nhân

**Lưu Thị Lan Anh (làm solo; tự chạy thật, không dùng provided-results).**

- *Vai trò:* một mình vận hành lệnh `student-bundle.py run`, kiểm cấu hình/JSON, xem ảnh Side và ghi log; không có nhóm để đổi vai.
- *Quan sát A/B/C:* A có 1 hộp (`run-A/summary.csv`), B có 13 hộp, C có 6 hộp toàn pedestrian (`run-C/side-demo-delta-1.73-voxel-0.32.png`). Chỉ A→B (đổi delta) đã tăng số hộp từ 1 lên 13; B→C (đổi pillar) làm hộp `vehicles` biến mất.
- *Phép z thuận/ngược:* trước inference `z_model = z_source − z_ground − delta` để đưa cloud về hệ mà checkpoint KITTI kỳ vọng; sau inference `z_source = z_model + z_ground + delta` để trả hộp về hệ PCD nguồn. Đổi delta ở đầu vào làm mạng thấy một scene khác nên kết quả đổi; quên phép ngược thì hộp chỉ bị dịch thấp đi một lượng không đổi mà không đổi class/x/y.
- *Quyết định lỗi batch:* ở `case-batch-z` toàn bộ 13 hộp cùng chìm 1.805 m dưới mặt đất nên quyết định là **dừng batch** và báo LC tạo lại prediction từ pipeline đúng, không sửa tay từng hộp; ở `case-one-box-z` chỉ một hộp lệch nên **kiểm từng hộp**.
- *Điều chưa chắc:* không có nhãn tham chiếu nên không biết hộp nào đúng; Side chồng nhiều hộp (x≈6–11 m) nên chưa kết luận được yaw; mean_z và số hộp không phải điểm chất lượng. Chưa kiểm B/C bằng ảnh camera/Top/Front vì gói Student không có ảnh camera. Cũng chưa chắc việc hộp two-wheels ở B đổi thành pedestrian ở C là cùng một đối tượng.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
