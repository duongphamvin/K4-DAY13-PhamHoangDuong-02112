# Báo cáo thực hành PointPillars — Day 13

Bản nộp theo quy định hiện tại: đặt cùng `TEAMMATES.md` trong repo cá nhân dùng cho VLearn. Không đưa output KITTI hoặc dữ liệu Robotaxi vào repo.

## Nhóm và provenance

- Mã nhóm/phòng: Nhóm 07 — Nà Ná Na Na (NaNaNaNa) / Phòng Lab Day 13
- Thành viên: xem `TEAMMATES.md` (Lê Đức Tú - 02045, Phạm Anh Huy - 02132, Phạm Hoàng Dương - 02112, Nguyễn Lê Thế Anh - 02164; vai trò đọc kết quả xoay vòng từng lượt).
- Trạng thái: `executed-by-group`
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Lê Đức Tú (chạy runner trên laptop, một lệnh cho cả A/B/C); 2026-10-01, 07:57:41–07:58:31 UTC (14:57–14:58 giờ VN); host Windows 11, Docker runtime linux/amd64, giới hạn 4 CPU / 4g.
- Image tag và image ID; phiên bản repo:
  - Image tag: `day13-pointpillars:lc-20261001-amd64`
  - Image ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`
  - Repo revision trong gói: `0831856d921609312d42c7582c366e5a311bb7b1` (`working_tree_dirty: true` theo `smoke.json`, chép nguyên văn)
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp:
  - Frame: `demo.pcd`, frame_id `demo` (mẫu KITTI 000008 chuyển đổi, CC BY-NC-SA 3.0)
  - Input SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`
  - Chạy local trên laptop nhóm bằng runner `student-bundle.py`, container `--network none`.
- Checkpoint: PointPillars KITTI có sẵn trong image:
  - Path: `/opt/PointPillars/pretrained/epoch_160.pth`
  - Checkpoint SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`
- Phạm vi: front-window, không `--full-scene` (range của checkpoint `0.0, -39.68, -3.0, 69.12, 39.68, 1.0`, tính trong hệ tọa độ model); `--from KITTI`; score threshold `0.3`.
- Giả định kênh thứ tư/intensity và nguồn z_ground:
  - PCD KITTI Student đã bỏ reflectance thật; RGB=0 là placeholder, script dùng kênh hằng của adapter. Đây không phải intensity được phục hồi.
  - $z_{ground} = 0.075\text{ m}$, do script ước lượng từ PCD, giống nhau ở cả ba lượt.

---

## Ba lượt inference thật

Số hộp lấy từ `n_boxes`, mean_z từ `mean_z` trong `run-A/B/C/summary.csv`. `mean_z` là trung bình cao độ tâm hộp, không phải điểm chất lượng.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| **A** | 0 m | 0.16 m | 1 | 0.330 m | `boxes-demo-delta-0-voxel-0.16.json`<br>`side-demo-delta-0-voxel-0.16.png`<br>`summary.csv` | Chỉ **1 hộp** `vehicles`, score **0.322** (sát ngưỡng 0.3), tại $x \approx 13.15, y \approx -0.45, z \approx 0.33$, cao 1.46 m. Đáy hộp ≈ **-0.40 m**, nằm dưới đường $z=0$ và dưới cụm điểm mặt đất trên ảnh Side. |
| **B** | 1.73 m | 0.16 m | 13 | 1.034 m | `boxes-demo-delta-1.73-voxel-0.16.json`<br>`side-demo-delta-1.73-voxel-0.16.png`<br>`summary.csv` | **13 hộp**: 10 `vehicles`, 1 `two-wheels`, 2 `pedestrian`; score 0.318–0.933. Hai xe score cao nhất ở $x \approx 8.09$ (0.933) và $x \approx 14.77$ (0.928) có đáy ≈ 0.15 m, sát cụm điểm mặt đất trên Side. Các hộp ở $x$ 8–11 m chồng nhau trên Side (có thể do khác $y$). Đây là **mốc so sánh**, chưa phải đáp án. |
| **C** | 1.73 m | 0.32 m | 6 | 1.091 m | `boxes-demo-delta-1.73-voxel-0.32.json`<br>`side-demo-delta-1.73-voxel-0.32.png`<br>`summary.csv` | **6 hộp, cả 6 đều `pedestrian`**, không còn `vehicles`/`two-wheels`. Một số nằm trong vùng B có xe: C (9.11, 0.40) gần xe B (8.09, 1.21); C (13.24, -0.95) gần xe B (14.77, -1.08); C (10.46, 4.93) gần two-wheels B (10.32, 5.25). |

### Trả lời câu hỏi phân tích:

* **A/B: Thay input trước model có khác dịch cùng một hằng số cho output không? Vì sao?**
  * **Khác.** Đổi delta làm đổi tọa độ z của toàn bộ điểm trước khi gom pillar, nên model chạy lại trên một input khác. Kết quả là số hộp (1 → 13), lớp, vị trí và score đều đổi; hộp A (13.15, -0.45) cũng không trùng tâm với hộp B gần nhất (14.77, -1.08). Nếu chỉ dịch output sau inference thì số hộp, lớp, x/y, yaw và score giữ nguyên, chỉ z đổi đều một lượng. Ví dụ dịch output của A thì vẫn chỉ có 1 hộp.
  * Kết quả A cho thấy input không bù chiều cao sensor ra ít hộp hơn hẳn. Nhưng chưa thể nói A "bỏ sót 12 đối tượng", vì chưa xác minh 13 hộp của B đều là đối tượng thật.

* **B/C: Thấy gì khi đổi pillar? Có đủ bằng chứng để nói cấu hình nào tốt hơn không?**
  * Số hộp giảm từ 13 xuống 6, và phân bố lớp đổi hẳn: `vehicles` 10 → 0, `two-wheels` 1 → 0, `pedestrian` 2 → 6. Ở vùng $x$ 8–15 m, B có hộp xe dài ≈3.6–3.7 m, còn C có các hộp hẹp dài ≈0.6–1.1 m gán lớp `pedestrian`. Như vậy không phải chỉ mất đối tượng nhỏ: cả lớp xe biến mất và số pedestrian tăng.
  * Checkpoint được train với pillar 0.16 m. Đưa biểu diễn 0.32 m vào cùng checkpoint là thay biểu diễn đầu vào, không phải lỗi z, vì delta vẫn là 1.73.
  * **Chưa đủ bằng chứng để chấm cấu hình nào đúng hơn từng hộp.** Nhiều hộp hơn ở B không chứng minh cả 13 hộp đều đúng; cần Top/Front/camera. Có thể ghi nhận là C gán lớp không nhất quán với hình dạng cụm điểm mà B gọi là xe.

* **Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?**
  * Runner chỉ dùng cửa sổ phía trước của checkpoint, nên vật ngoài ROI không xuất hiện trong output. Đó là giới hạn phạm vi, không phải bằng chứng model bỏ sót.
  * Side là hình chiếu $x$–$z$ toàn scene: các vật ở $y$ khác nhau bị chồng lên nhau (ví dụ $x$ 8–11 m trong B), và ảnh không thể hiện yaw hay đầu/đuôi xe. Side chỉ dùng để xem cao độ/đáy hộp so với cụm điểm; đường $z=0$ là tham chiếu của plot, không phải mặt đường đo được ở mọi vị trí (cụm điểm mặt đất ở $x$ 30–40 m nằm khoảng $z$ 0.3–0.5).

* **JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?**
  * Cả ba JSON A, B, C đều là prediction thô của model pretrained KITTI trên frame demo, chưa được phân xử, và **không được import vào job Robotaxi** (khác frame/dataset).
  * Ngay trong bài KITTI: A chỉ có 1 hộp score sát ngưỡng, đáy chìm; C đổi toàn bộ xe thành người; B là mốc so sánh nhưng vẫn cần kiểm Top/Front/camera, kích thước và yaw từng hộp, đặc biệt các hộp chồng nhau ở $x$ 8–11 m và các hộp xa ($x > 30$ m) ít điểm.

---

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | :---: | :---: | :---: | :--- | :--- |
| `case-correct` | **0 / 13** | 0 m | Không đổi | **Không có lỗi z được tiêm** | Bản sao prediction B (sha256 `16f30b08…`, theo `manifest.json`), ảnh `side-correct.png`. Giữ phép chuyển z của prediction gốc, **không phải** cuboid đúng. |
| `case-batch-z` | **13 / 13** | **-1.805 m** | Không đổi, chỉ trường `z` | **DỪNG BATCH, báo LC kiểm pipeline** | Mọi hộp bị trừ cùng $z_{ground} + \Delta z = 0.075 + 1.73 = 1.805\text{ m}$ (`height_offset_m: 1.805`); `side-batch-z.png`. Cùng một lượng ở toàn batch → nghi quên phép chuyển ngược; yêu cầu tạo lại prediction từ pipeline đúng, không sửa tay từng hộp. |
| `case-one-box-z` | **1 / 13** | **-1.805 m** ở 1 hộp, 0 m ở 12 hộp | Không đổi, chỉ trường `z` của hộp đó | **Kiểm từng hộp** | Chỉ hộp đầu tiên (`vehicles`, $x \approx 8.09, y \approx 1.21$) lệch; `side-one-box-z.png`. Lỗi cục bộ → kiểm đối tượng đó bằng Top/Side/Front và camera, không chốt lỗi pipeline chỉ vì một hộp chìm. |

*Lưu ý*: Helper `pipeline-qc-cases.py` tạo biến đổi có chủ đích từ prediction lượt B (`boxes-demo-delta-1.73-voxel-0.16.json`). Đây không phải kết quả inference riêng, không phải nhãn đúng; `manifest.json` ghi `training_only: true`, không import CVAT.

---

## Nhận xét cá nhân

Mỗi thành viên tự viết mục của mình. Chỉ đọc kết quả chuẩn bị trước thì ghi rõ chưa tự chạy.

### 1. Lê Đức Tú (MSSV: 02045)
* **Vai trò**: Vận hành runner `student-bundle.py` trên laptop (cả A/B/C); kiểm `smoke.json` sau khi chạy.
* **Quan sát A/B/C**: `smoke.json` báo `status: passed` cho cả 5 bước (docker-load, run-A/B/C, qc-cases), mỗi lượt inference mất khoảng 3.5–5.3 giây trên CPU. Lượt A (`boxes-demo-delta-0-voxel-0.16.json`) chỉ có 1 hộp `vehicles` score 0.322, sát ngưỡng 0.3; trên `side-demo-delta-0-voxel-0.16.png` đáy hộp ≈ -0.40 m, chìm dưới cụm điểm mặt đất. Lượt B cùng PCD, cùng checkpoint, chỉ đổi delta lên 1.73 thì ra 13 hộp.
* **Diễn giải phép biến đổi z**: Delta được trừ vào z của điểm trước khi gom pillar ($z_{model} = z_{source} - z_{ground} - \Delta z$), nên A và B là hai lần model nhìn hai input khác nhau. Vì vậy số hộp đổi từ 1 lên 13, không phải hộp cũ bị dịch lên 1.73 m. Script cộng ngược $z_{ground} + \Delta z$ khi xuất JSON nên hộp của cả hai lượt đều ở hệ nguồn.
* **Quyết định lỗi batch/từng hộp và hành động**: Với `case-batch-z`, cả 13 hộp lệch cùng -1.805 m → dừng sửa tay, báo LC kiểm phép chuyển ngược và tạo lại prediction.
* **Điều chưa chắc**: Chỉ có một frame và chưa xem Top/camera, nên chưa biết hộp duy nhất của A có trùng một chiếc xe thật trong 13 hộp B hay không (tâm A 13.15, -0.45 so với xe B gần nhất 14.77, -1.08).

### 2. Phạm Anh Huy (MSSV: 02132)
* **Vai trò**: Kiểm cấu hình/JSON (Lượt A), Xem hình học Side (Lượt B), Ghi log/Thư ký (Lượt C).
* **Quan sát A/B/C**: So B với C (cùng delta 1.73, chỉ đổi pillar 0.16 → 0.32 m): số hộp 13 → 6, nhưng điểm đáng chú ý là lớp: `vehicles` 10 → 0, `pedestrian` 2 → 6. Trong `boxes-demo-delta-1.73-voxel-0.32.json` có pedestrian tại (9.11, 0.40) và (13.24, -0.95), gần hai xe score cao của B tại (8.09, 1.21) và (14.77, -1.08). Trên `side-demo-delta-1.73-voxel-0.32.png`, vùng x 8–15 m chỉ còn các hộp hẹp dài ≈0.6–1.1 m thay cho hộp xe dài ≈3.7 m.
* **Diễn giải phép biến đổi z**: B và C dùng cùng $z_{ground} = 0.075$ và cùng $\Delta z = 1.73$, nên khác biệt B/C không đến từ phép z thuận/ngược. Nó đến từ việc checkpoint được train với pillar 0.16 m nhưng nhận biểu diễn 0.32 m.
* **Quyết định lỗi batch/từng hộp và hành động**: Với `case-one-box-z`, chỉ 1/13 hộp lệch → không dừng batch, kiểm riêng hộp đó bằng nhiều góc nhìn.
* **Điều chưa chắc**: Chưa biết các hộp pedestrian của C là người thật hay là một phần thân xe bị gán nhầm lớp; cần Top view và ảnh camera để phân biệt.

### 3. Phạm Hoàng Dương (MSSV: 02112)

* **Vai trò**: Xem hình học Side ở lượt A, ghi log ở lượt B và kiểm cấu hình/JSON ở lượt C. Tôi cũng đối chiếu `case-batch-z.json` với `case-correct.json` để kiểm phạm vi lỗi z.
* **Quan sát A/B/C**: Trong `qc-cases/case-batch-z.json`, cả 13/13 hộp đều có tâm z giảm đúng 1.805 m so với `case-correct.json`; class, x, y, kích thước, yaw và score giữ nguyên. Mức lệch này khớp `height_offset_m` trong `manifest.json` và bằng $z_{ground} + \Delta z = 0.075 + 1.73$. Ảnh `side-batch-z.png` cũng cho thấy toàn bộ hộp cùng tụt xuống dưới cụm điểm mặt đất.
* **Diễn giải phép biến đổi z**: Model trả hộp trong hệ $z_{model}$. Pipeline phải cộng lại $z_{ground} + \Delta z$ khi chuyển hộp về hệ nguồn. Nếu bỏ bước này, mọi hộp trong frame cùng thiếu đúng 1.805 m; đây là dấu hiệu của lỗi chuyển đổi chung, không phải lỗi hình học riêng lẻ.
* **Quyết định lỗi batch/từng hộp và hành động**: Dừng sửa tay cả batch, báo LC kiểm transform và tạo lại prediction bằng pipeline đúng. Sửa lần lượt 13 hộp chỉ che triệu chứng và lỗi sẽ tiếp tục xuất hiện ở frame khác.
* **Điều chưa chắc**: $z_{ground}$ được ước lượng riêng theo từng frame nên độ lệch có thể không giống nhau giữa các frame. Trong một frame, dấu hiệu đáng tin hơn là nhiều hộp cùng lệch một lượng trong khi các trường khác không đổi.

### 4. Nguyễn Lê Thế Anh (MSSV: 02164)
* **Vai trò**: Ghi log/Thư ký (Lượt A), Kiểm cấu hình/JSON (Lượt B), Xem hình học Side (Lượt C).
* **Quan sát A/B/C**: Trong `qc-cases/case-one-box-z.json` chỉ hộp đầu tiên (`vehicles`, x≈8.09, y≈1.21) có z giảm 1.805 m, 12 hộp còn lại giữ nguyên so với `case-correct.json`. Trên `side-demo-delta-1.73-voxel-0.16.png` của B, vùng x 8–11 m có nhiều hộp chồng lên nhau vì Side gộp mọi y vào một mặt phẳng.
* **Diễn giải phép biến đổi z**: Phép chuyển z áp dụng cùng một công thức cho mọi hộp trong frame, nên nếu chỉ một hộp lệch thì khó giải thích bằng lỗi pipeline; nhiều khả năng là lỗi riêng của đối tượng đó.
* **Quyết định lỗi batch/từng hộp và hành động**: Kiểm từng hộp: xem hộp lệch bằng Top/Side/Front và camera, giữ nguyên các hộp khác; không chốt lỗi pipeline chỉ vì một hộp chìm.
* **Điều chưa chắc**: Side không thể hiện yaw, và đường z=0 chỉ là tham chiếu của plot (cụm điểm mặt đất ở x 30–40 m nằm khoảng z 0.3–0.5), nên chỉ dùng Side để xem cao độ, không đủ để duyệt hình học từng hộp.

---

## LC ghi nhận riêng

*(Phần này do LC điền, nhóm để trống.)*

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
