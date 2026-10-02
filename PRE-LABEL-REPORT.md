# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Cá nhân - Lê Hữu Nghĩa (Tài khoản: 2A202602285)
- Thành viên: Lê Hữu Nghĩa - MSSV: 2A202602285 (chi tiết vai trò tại TEAMMATES.md).
- Trạng thái: `executed-on-room-LC-machine` (kết hợp phân tích từ native runner bundle).
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Lê Hữu Nghĩa; 02/10/2026; Linux x86_64 / Windows amd64.
- Image tag và image ID; phiên bản repo: `day13-pointpillars:lab` (Image Checkpoint SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`); repo revision student Day 13.
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `demo.pcd` (KITTI frame `000008`, 17.238 điểm, SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`); chạy trong môi trường lab được cấp phép.
- Checkpoint: `/opt/PointPillars/pretrained/epoch_160.pth` (PointPillars pretrained KITTI có sẵn trong image).
- Phạm vi: front-window ROI; score threshold: `0.3`.
- Giả định kênh thứ tư/intensity và nguồn z_ground: Kênh thứ 4 gán giá trị hằng số (RGB=0/adapter) do bản demo đã lược bỏ reflectance gốc; `z_ground` được ước lượng tự động từ phân bố điểm thấp nhất mặt đường của PCD.

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | -0.19 m | `boxes-demo-delta-0-voxel-0.16.json`, `side-demo-delta-0-voxel-0.16.png`, `summary.csv` | Chỉ detect được 1 hộp duy nhất ở cự ly gần; điểm chưa dịch delta khiến phân bố độ cao lệch hẳn dải cảm biến KITTI. |
| B | 1.73 | 0.16 | 13 | 0.52 m | `boxes-demo-delta-1.73-voxel-0.16.json`, `side-demo-delta-1.73-voxel-0.16.png`, `summary.csv` | Phát hiện 13 hộp (10 vehicles, 2 pedestrian, 1 two-wheels) bám tốt cụm điểm ở khoảng cách 10m - 50m. |
| C | 1.73 | 0.32 | 6 | 0.38 m | `boxes-demo-delta-1.73-voxel-0.32.json`, `side-demo-delta-1.73-voxel-0.32.png`, `summary.csv` | Chỉ phát hiện 6 pedestrian, mất toàn bộ class vehicles; ô pillar tăng gấp đôi (32cm) làm suy giảm độ phân giải hình học. |

- A/B — chỉ đổi delta: A có 1 hộp; B có 13 hộp. Ảnh Side và file JSON cho thấy vùng cự ly trung bình $x \in [15\text{m}, 45\text{m}]$ ở lượt A bị sót hầu hết xe, trong khi lượt B bắt được 10 xe. Đây là chạy lại model trên input có tọa độ z được dịch tương ứng với chiều cao đặt LiDAR của KITTI (~1.73m), làm thay đổi cách điểm phân bố vào các pillar, không chỉ dịch hộp cũ; điều em còn chưa chắc là hướng quay (yaw) của các đối tượng bị che khuất ở xa.
- B/C — chỉ đổi pillar: B có 13 hộp; C có 6 hộp. Khi tăng kích thước pillar từ 0.16m lên 0.32m, số lượng hộp giảm mạnh (từ 13 xuống 6 hộp), mất toàn bộ các hộp vehicle và chỉ còn lại 6 pedestrian. Độ phân giải lưới pillar thô hơn làm giảm đặc trưng chi tiết của hình học cụm điểm. Chưa đủ bằng chứng để kết luận C tốt hơn (ngược lại, C bỏ sót nhiều đối tượng xe).
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào? Ảnh Side là hình chiếu trực giao $x-z$ nên các vật thể ở cùng khoảng cách $x$ nhưng lệch làn đường $y$ sẽ bị chiếu đè lên nhau, dễ gây ngộ nhận là trùng hộp hoặc miss. Hơn nữa, góc Side không thể hiện được góc xoay ngang (yaw trên mặt phẳng $x-y$), nên bắt buộc phải kết hợp góc nhìn Trên (Top/BEV) và ảnh camera.
- JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp? Cả 3 file JSON A/B/C đều là kết quả chạy trên KITTI demo, không áp dụng trực tiếp cho schema và tọa độ của Robotaxi. Cần kiểm tra kỹ pipeline chuyển đổi tọa độ, frame ID và chỉ lấy pre-label từ nguồn Robotaxi chính thức do LC nạp.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 m | Không đổi | Kiểm từng hộp / Giữ nguyên | Tọa độ z của tất cả 13 hộp giữ nguyên đúng theo phép biến đổi gốc từ prediction B. |
| case-batch-z | 13 / 13 | -(delta + z_ground) (lệch ~1.73m) | Không đổi | **Dừng batch, không sửa tay** | Toàn bộ 13 hộp đều bị chìm sâu xuống lòng đất cùng một khoảng cách; đây là lỗi pipeline quên phép cộng ngược z_ground + delta, cần báo LC/kỹ thuật fix code. |
| case-one-box-z | 1 / 13 | -(delta + z_ground) | Không đổi | **Kiểm từng hộp** | Chỉ duy nhất hộp đầu tiên bị tụt z, 12 hộp còn lại nằm đúng vị trí; đây là lỗi đơn lẻ của đối tượng, cần kiểm tra nhiều góc nhìn để hiệu chỉnh riêng hộp đó. |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

- **Lê Hữu Nghĩa (MSSV: 2A202602285):**
  - *Vai trò thực hiện:* Độc lập thực hiện toàn bộ quy trình: vận hành runner A/B/C, đọc và kiểm tra các file JSON/CSV/Side, phân tích 3 ca QC pipeline và hoàn thiện báo cáo thí nghiệm.
  - *Quan sát thực nghiệm A/B/C:* Ở lượt A (delta=0) model chỉ tìm được 1 hộp do input không khớp chiều cao cảm biến; khi bù delta=1.73m ở lượt B, số hộp tăng lên 13 bám rất tốt vào cụm điểm xe. Ở lượt C, việc tăng kích cỡ ô pillar lên 0.32m làm mất toàn bộ 10 hộp xe (chỉ còn 6 pedestrian), chứng tỏ pillar size ảnh hưởng trực tiếp đến khả năng phân tách cụm điểm lớn.
  - *Diễn giải phép biến đổi z:* Điểm trước khi vào model được đưa về hệ tọa độ chuẩn theo công thức $z_{\text{model}} = z_{\text{source}} - z_{\text{ground}} - \text{delta}$. Sau khi mạng dự đoán xong hộp, script bắt buộc phải cộng ngược lại $z_{\text{source}} = z_{\text{model}} + z_{\text{ground}} + \text{delta}$ để đưa hộp về đúng cao độ mặt đất thực tế.
  - *Quyết định khi gặp lỗi pipeline:* Nếu phát hiện tất cả các hộp trong batch đều bị chìm/nổi cùng một lượng z (như trong `case-batch-z`), hành động duy nhất đúng là **dừng sửa thủ công ngay lập tức**, báo LC và đội ngũ kỹ thuật sửa pipeline biến đổi. Việc cố gắng kéo từng hộp bằng tay trong trường hợp này là sai nguyên tắc và gây lãng phí công sức.
  - *Điều chưa chắc chắn:* Hình chiếu Side phẳng $x-z$ dễ gây nhầm lẫn khi các xe đỗ so le nhau bị chồng lấn hình chiếu; việc xác định hướng đầu xe (yaw) của các đối tượng ở xa (>40m) bắt buộc phải đối chiếu thêm với ảnh camera độ phân giải cao.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:

