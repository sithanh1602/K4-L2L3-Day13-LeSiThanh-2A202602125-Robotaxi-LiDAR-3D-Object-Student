# Báo cáo thực hành PointPillars — Day 13 (solo)

## Thông tin thực hiện

- Hình thức: cá nhân (solo). Họ tên: **Lê Sĩ Thanh**. MSSV: **2A202602125**.
- Trạng thái: đã chạy inference thật trên máy cá nhân; không dùng `provided-results`.
- Thời gian theo log: 02/10/2026, 15:18:21–15:18:53 (UTC+7).
- Môi trường: Docker Linux amd64, CPU; giới hạn mỗi container 4 CPU và 4 GB RAM, không phải RAM đo thực tế.
- Đầu vào: KITTI demo `000008` chuyển thành `input/demo.pcd`, 17.238 điểm; frame_id `demo`.
- SHA256 đầu vào: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`.
- Image tag: `day13-pointpillars:lc-20261001-amd64`.
- Image ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`.
- Checkpoint: `/opt/PointPillars/pretrained/epoch_160.pth`, SHA256 `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`.
- Revision ghi trong manifest gói: `0831856d921609312d42c7582c366e5a311bb7b1`; `working_tree_dirty: true`. Đây là provenance của gói, không khẳng định checkout hiện tại sạch hay cùng revision.
- Giữ checkpoint, score threshold 0.3 và front ROI ở cả ba lượt; không train lại model.
- Front ROI trong hệ model: x=(0,69.12), y=(-39.68,39.68), z=(-3,1) m; không chạy `--full-scene`.
- PCD giữ x/y, dịch z nguồn KITTI +1.73 m, bỏ reflectance thật và thêm RGB=0. Adapter dùng kênh hằng 0 cho vehicles và 0.7 cho pedestrian/two-wheels; không phục hồi intensity.
- `z_ground = 0.075 m`, ước lượng từ đỉnh histogram z; không phải mặt đường cục bộ chính xác ở mọi vị trí.
- Kiểm tra chạy: [smoke.json](student-bundles/ket-qua-solo-01/smoke.json) có `status: passed`; đủ ba lượt và ba ca lỗi z.

## Kết quả A/B/C

Số hộp và mean_z lấy trực tiếp từ `summary.csv`; class lấy từ JSON tương ứng. mean_z là trung bình cao độ tâm hộp, không phải điểm chất lượng.

| Lượt | Delta (m) | Pillar XY (m) | Số hộp | Class | mean_z (m) | Output |
| --- | ---: | ---: | ---: | --- | ---: | --- |
| A | 0 | 0.16 | 1 | 1 vehicles | 0.330 | [CSV](student-bundles/ket-qua-solo-01/run-A/summary.csv), [JSON](student-bundles/ket-qua-solo-01/run-A/boxes-demo-delta-0-voxel-0.16.json), [Side](student-bundles/ket-qua-solo-01/run-A/side-demo-delta-0-voxel-0.16.png) |
| B | 1.73 | 0.16 | 13 | 10 vehicles, 2 pedestrian, 1 two-wheels | 1.034 | [CSV](student-bundles/ket-qua-solo-01/run-B/summary.csv), [JSON](student-bundles/ket-qua-solo-01/run-B/boxes-demo-delta-1.73-voxel-0.16.json), [Side](student-bundles/ket-qua-solo-01/run-B/side-demo-delta-1.73-voxel-0.16.png) |
| C | 1.73 | 0.32 | 6 | 6 pedestrian | 1.091 | [CSV](student-bundles/ket-qua-solo-01/run-C/summary.csv), [JSON](student-bundles/ket-qua-solo-01/run-C/boxes-demo-delta-1.73-voxel-0.32.json), [Side](student-bundles/ket-qua-solo-01/run-C/side-demo-delta-1.73-voxel-0.32.png) |

### So A/B — đổi riêng delta

A chỉ có một hộp vehicles tại tâm x≈13.154 m, z≈0.330 m. Ảnh B có các hộp ở nhiều vùng x≈3.7–55.6 m, trong khi A chỉ có hộp quanh x≈13 m. B xuất hiện thêm pedestrian và two-wheels. JSON B ghi vehicles tại x≈14.766 m, z≈0.900 m; chưa đủ cơ sở xác nhận đây là cùng đối tượng với hộp A.

Đổi delta trước inference làm thay đổi input, tập điểm lọt ROI và output của model. Đây không phải dịch nguyên các hộp A thêm 1.73 m. Nhiều hộp hơn chưa chứng minh B chính xác hơn; không có ground truth/camera để xác nhận tất cả đối tượng.

### So B/C — đổi riêng pillar

Giữ delta=1.73 m, tăng cạnh pillar từ 0.16 lên 0.32 m. Số hộp giảm từ 13 xuống 6; C chỉ có pedestrian, không còn vehicles hoặc two-wheels trong output. Trên ảnh Side, B có các hộp vehicles rộng quanh x≈41 và 56 m; C không có các hộp này. C có hộp pedestrian hẹp quanh x≈9–19 và 33.5 m.

Checkpoint không được train lại cho pillar 0.32 m. Kết quả cho thấy output nhạy với biểu diễn đầu vào, nhưng chưa đủ chứng cứ để kết luận cấu hình nào tốt hơn. Không coi các hộp C là phiên bản đổi class của cùng hộp B khi chưa đối chiếu đối tượng.

## Phép đổi tọa độ z

```text
z_model  = z_source - z_ground - delta
z_source = z_model + z_ground + delta
```

Trong lượt B/C, lượng dịch thuận/ngược là 0.075+1.73=1.805 m. Lượt A vẫn trừ/cộng lại z_ground dù delta=0. Script còn đổi bottom-z thành center-z và chuyển yaw về quy ước lab; JSON đã ở hệ PCD nguồn, không đổi thêm lần nữa.

## Ca lỗi z có kiểm soát

Các ca được helper tạo từ prediction B, không phải inference mới hay QC chéo trên portal. Đã đối chiếu JSON và ảnh Side; source hash trong manifest trỏ đúng B.

| Ca | Số hộp lệch z | Chênh z so với B | Class/x/y/kích thước/yaw/score | Quyết định |
| --- | ---: | ---: | --- | --- |
| case-correct | 0/13 | 0 m | Giữ nguyên | Giữ chuyển đổi nguồn; vẫn cần kiểm chất lượng từng hộp, không xem là nhãn đúng |
| case-batch-z | 13/13 | -1.805 m | Giữ nguyên | Dừng sửa tay cả batch; kiểm phép đổi tọa độ và tạo lại prediction |
| case-one-box-z | 1/13 | -1.805 m ở hộp đầu tiên | Giữ nguyên | Kiểm riêng đối tượng bằng nhiều view; không quy kết cả pipeline từ một hộp |

Bằng chứng: [manifest ca lỗi](student-bundles/ket-qua-solo-01/qc-cases/manifest.json), `qc-cases/case-*.json` và `qc-cases/side-*.png`. Ảnh batch-z cho thấy tất cả hộp dịch xuống; ảnh one-box-z chỉ có hộp vehicles đầu tiên tại x≈8.094 m bị dịch xuống. Tâm z của hộp đó đổi từ 0.921498 m thành -0.883502 m. Các hộp còn lại giữ nguyên.

Nếu gặp hiện tượng tương tự trong dữ liệu thật, cần kiểm transform, frame, cấu hình và bằng chứng hình học trước khi chọn hành động. Các ca ở đây đã được tạo lỗi có chủ đích nên không tự chứng minh nguyên nhân của một sự cố thật.

## Giới hạn và điều chưa chắc

- Side là hình chiếu x-z, chồng các đối tượng có y khác nhau; không đủ để duyệt yaw, chiều rộng, class hoặc hộp thiếu/thừa.
- Đường z=0 trên plot chỉ là tham chiếu. Đáy từng hộp phải kiểm với mặt đường cục bộ.
- Điểm ngoài front ROI không phải bằng chứng detector bỏ sót trong phạm vi thí nghiệm này.
- Mẫu không có camera hoặc ground truth; reflectance thật đã bị loại bỏ. Không tính accuracy, precision/recall hay tuyên bố B là đáp án chuẩn.
- Không import JSON KITTI hoặc `training_only` vào job Robotaxi.

## Nhận xét cá nhân — bản nháp để học viên đọc và xác nhận

Tôi thực hiện bài theo hình thức solo, chạy lệnh tạo A/B/C trên cùng mẫu KITTI. Output A/B thay đổi từ 1 thành 13 hộp khi chỉ đổi delta; B/C thay đổi từ 13 thành 6 hộp khi chỉ đổi pillar. Phép z thuận/ngược trả output về hệ nguồn; thay delta trước inference khác với dịch hộp sau inference. Với ca mọi hộp lệch -1.805 m, hành động phù hợp là dừng chỉnh từng hộp và kiểm pipeline. Với một hộp lệch, cần kiểm riêng đối tượng. Tôi chưa có đủ bằng chứng để xác nhận class, yaw và chất lượng mọi hộp chỉ qua ảnh Side.

Đoạn này được trợ lý soạn từ output đã kiểm; học viên cần đọc, chỉnh theo hiểu biết và xác nhận trước khi nộp. Không thay cho xác nhận đã tự quan sát/phân tích.

## Trạng thái phần portal và nộp bài

- Học viên cho biết đã sửa label trên portal; báo cáo này chưa kiểm chứng Save, nộp v1 hay v2.
- QC chéo/QA trên portal: chưa xác nhận hoàn tất. Ba ca lỗi z không thay thế bước này.
- Báo cáo nằm ở `PRE-LABEL-REPORT.md` tại gốc repo; output nằm trong `student-bundles/ket-qua-solo-01/` được Git ignore. Nộp báo cáo và output qua kênh private LC chỉ định.
- LC ghi nhận: chưa xác nhận; không tự điền đã được duyệt.
