# AI Support Log — Day20

Dương Thị Hồng Viên · MHV 2A202602385 · Dự án P-115 / ViR-Bot

## AI đã giúp tôi ở đâu?

Tôi dùng AI ở ba chỗ trong phạm vi được phép:

1. **Brainstorm ứng viên core action:** Tôi mô tả persona ADAS và core job, nhờ AI liệt kê 5–6 hành động ứng viên (gửi query, xem evidence, xác nhận relevance, lưu workspace, đóng task) và đặt câu hỏi phản biện từng ứng viên—ví dụ "query gửi đi có đủ chứng minh người dùng nhận được giá trị chưa?" và "lưu workspace mà chưa xem evidence có tính là hoàn tất không?" Tôi dùng các câu hỏi đó để tự lọc, không để AI chọn hộ.

2. **Gợi ý tên event dạng object\_action và acceptance criteria mẫu:** Tôi cung cấp danh sách 6 điểm đo cần ghi, nhờ AI đề xuất tên event theo convention `object_action` và viết một acceptance criteria mẫu để tôi thấy format. AI gợi ý `search_task_closed`, `workspace_item_added`, `evidence_viewed`… và viết mẫu cho `evidence_viewed`. Tôi điều chỉnh trigger, payload và mapping metric của từng event theo hiểu biết của mình về service.

3. **Đóng vai khách hàng khó tính để test core job và cadence:** Tôi nhờ AI đặt câu hỏi nghi vấn từ góc nhìn ADAS—"Nếu tôi chỉ gửi query rồi copy kết quả, task có nên tự đóng không?", "Nếu dữ liệu đổi version sau khi tôi lưu, workspace item đó còn giá trị không?"—để kiểm tra xem completion rule và cadence tôi chọn có giữ được không.

## AI sai, hời hợt hoặc đề xuất metric sai nature ở đâu?

- **Đề xuất D7 retention mặc định:** Khi tôi hỏi về retention, AI ban đầu gợi ý cửa sổ 7 ngày theo kiểu habit app mà không hỏi về nature của workflow. Tôi phải chỉ ra rằng ADAS quay lại theo assignment, không theo lịch cố định, nên D7 sẽ phạt người không có nhu cầu mới chứ không đo churn thật. AI sau đó đồng ý nhưng không tự đưa ra được định nghĩa NTR—tôi phải tự thiết kế 6 thành phần.

- **Đề xuất search count và top-3 làm engagement proxy:** AI gợi ý "số lần search mỗi tuần" và "click-through rate trên top 3" làm leading metric. Tôi bác vì nhiều query cùng task không phải nhiều giá trị; top-3 CTR đo sự thu hút của kết quả chứ không đo người dùng có kiểm tra được evidence chưa. Tôi chọn Evidence Reach Rate thay thế vì nó đo tiền đề thật của việc chọn video.

- **Bỏ sót mẫu số denominator:** Ở Searchability Coverage, AI đề xuất dùng "số video trả về" làm mẫu số. Tôi sửa thành tổng inventory hợp lệ từ snapshot Postgres, vì nếu dùng kết quả tìm kiếm làm mẫu số thì metric đo độ rộng kết quả chứ không đo coverage thật.

## Tôi đã tự sửa hoặc quyết định lại điều gì?

- **Tự chọn core action:** Sau khi đọc `src/search/service.py` và README, tôi quyết định core action là "hoàn tất nhiệm vụ bằng cách xem evidence, xác nhận phù hợp và lưu workspace"—không phải gửi query hay nhận top 3—vì chỉ ở bước đó người dùng mới có đầu vào đã kiểm tra cho công việc tiếp theo.

- **Tự kết luận cadence:** Tôi đọc phần trigger và repeat condition rồi tự viết: nhịp đo là mỗi cơ hội nhiệm vụ, tổng hợp theo tuần để vận hành; tuần là nhịp báo cáo chứ không phải kỳ vọng người dùng quay lại mỗi tuần. AI không đưa ra kết luận này.

- **Tự định nghĩa NTR 6 thành phần:** AI chỉ gợi ý cửa sổ D7/D28 rời rạc, không tổ hợp thành unit × project × assignment đồng thời. Tôi tự viết từng thành phần, đặc biệt quyết định dùng deadline nhiệm vụ làm boundary và ghi no-opportunity riêng thay vì bỏ qua người không có task mới.

- **Tự viết metric hypothesis:** Tôi tự viết giả thuyết cho loop—lưu yêu cầu và video/version vào workspace giúp NTR tăng vì người có nhiệm vụ tiếp theo tái dùng được ngữ cảnh—và ghi rõ cách kiểm chứng bằng baseline trước thay đổi, không đặt target % khi chưa có dữ liệu.

- **Tự quyết định loại counter-metric:** AI không gợi ý Invalid Selection Rate. Tôi tự thêm vì xác nhận relevance là tự đánh giá, cần review độc lập để phát hiện sai sót; không có counter-metric này thì NSM tăng không chứng minh chất lượng thật.
