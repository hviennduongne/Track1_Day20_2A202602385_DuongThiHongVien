# Metrics Pack — P-115 / ViR-Bot

**Day20 · Dương Thị Hồng Viên · 2A202602385**

> Đây là thiết kế đo lường đề xuất, dựa trên README và `src/search/service.py` của P-115 được đọc trong phiên làm bài. Chưa triển khai các event, chưa có baseline retention hoặc bằng chứng cải thiện. Tất cả cửa sổ thời gian và giả thuyết bên dưới cần được kiểm chứng bằng pilot.

## 00 — Dự án, persona, core job

**Dự án:** 3D Data Hub / ViR-Bot hỗ trợ tìm video trong dữ liệu đa cảm biến. Trong MVP, DE quản lý dữ liệu và phiên bản; ADAS tìm video bằng nhãn/mô tả, xem evidence frame và chọn vào workspace cá nhân. Dữ liệu ví dụ là nuScenes mini. Tìm kiếm ngữ nghĩa dùng SigLIP2; không suy diễn rằng MVP có LLM/agent.

**Persona duy nhất:** kỹ sư ADAS được giao tìm video phù hợp một tình huống để chuẩn bị tập dữ liệu cho công việc tiếp theo. ADAS chỉ đọc dữ liệu nguồn, có quyền thao tác workspace của mình.

**Core job bằng ngôn ngữ người dùng:** “Khi nhận yêu cầu tìm dữ liệu cho một tình huống, tôi cần tìm và chọn được video phù hợp, có evidence để kiểm tra, để có đầu vào cho bước làm việc tiếp theo.”

**Phạm vi:** tìm → kiểm tra evidence → xác nhận phù hợp → lưu vào workspace. Không mở rộng thành clip review/release, huấn luyện mô hình, MCAP hay 3D/4D review ngoài MVP.

**Căn cứ và giới hạn:** service lọc theo project/team và video hiện hành, lọc nhãn ở keyframe, rồi xếp hạng ngữ nghĩa trong phạm vi đã lọc; mặc định top 3 video. Service kiểm tra frame với Postgres để loại dữ liệu cũ/xóa/ngoài scope. Điểm tương đồng không phải xác suất clip phù hợp. Evidence trong tìm kiếm chỉ bằng nhãn có thể là camera sample gần keyframe nhãn; không tự chứng minh hình ảnh chứa đủ tình huống cần tìm. Vì vậy kết quả tìm thấy chưa được coi là giá trị hoàn tất.

**Giả định cần xác nhận:** có nguồn nhiệm vụ/assignment độc lập với việc mở app; người dùng có định danh cá nhân; nhóm có thể thêm thao tác xác nhận relevance và liên kết task với workspace. MVP dùng tài khoản demo dùng chung nên chưa đủ đo retention từng người.

## 01 — Core Action Card (+ tự kiểm 5 tiêu chí)

| Thành phần | Quyết định |
|---|---|
| Actor | Một kỹ sư ADAS có quyền trong project |
| Object | Một nhiệm vụ tìm dữ liệu (`task_id`) và ít nhất một video trong scope |
| Core action | Hoàn tất nhiệm vụ bằng cách kiểm tra evidence, xác nhận video phù hợp yêu cầu và lưu video vào workspace của mình |
| Preconditions | Task có yêu cầu rõ; video/frame còn hợp lệ ở phiên bản được chọn; actor có quyền; kết quả có evidence xem được |
| Completion rule | Task được đóng thành công sau khi có ≥1 video đã xem evidence, được actor xác nhận phù hợp và được backend lưu workspace thành công; liên kết task–video–workspace tồn tại và scope hợp lệ |
| Core value | Có một tập video khởi đầu đã được kiểm tra để dùng cho bước tiếp theo của nhiệm vụ |
| Value evidence | Workspace item đã persist + dấu vết evidence view + relevance confirmation + task outcome; review mẫu độc lập kiểm tra chất lượng |
| Candidate value event | `search_task_closed` với `outcome=success` và completion rule được backend xác thực |

Core job là vấn đề cần giải quyết; core action là hành động hoàn tất nêu trên; core value là đầu vào phù hợp cho công việc; value event chỉ là bản ghi chứng minh. Một câu query, một lượt click hay trả về top 3 không thay thế completion rule.

| Tự kiểm | Kết quả | Lý do / điều kiện |
|---|---|---|
| Gần giá trị thật | Đạt | Clip đã kiểm tra và lưu được; không chỉ nhận kết quả tìm kiếm |
| Có thể lặp lại | Đạt | Lặp lại khi có nhiệm vụ tìm dữ liệu mới |
| Quan sát được | Đạt trong thiết kế | Backend xác thực completion; cần bổ sung instrumentation và task linkage |
| Có ý nghĩa với người dùng | Đạt theo giả định persona | Chọn được đầu vào cho yêu cầu ADAS; cần phỏng vấn/pilot để xác nhận |
| Sản phẩm có thể tác động | Đạt | Cải thiện scope, coverage, evidence và thao tác lưu ảnh hưởng trực tiếp |

**Kết luận:** 5/5 về thiết kế, chưa khẳng định hệ thống hiện tại đã ghi nhận đủ bằng chứng.

## 02 — Action Nature Card + kết luận cadence

| Thành phần | Phân tích |
|---|---|
| Actor | Kỹ sư ADAS trong một project đang hoạt động |
| Intent | Đáp ứng yêu cầu tìm dữ liệu cụ thể |
| Trigger tự nhiên | Nhận nhiệm vụ/tình huống mới; yêu cầu thay đổi; dữ liệu/nhãn được cập nhật khiến tập đã chọn cần kiểm tra lại |
| Effort | Viết query/lọc nhãn, đọc evidence, đối chiếu yêu cầu, chọn và lưu |
| Value timing | Sau khi tập video được kiểm tra và persist, không phải lúc mở app |
| State | Yêu cầu task, query và evidence đã xem, video đã chọn, version và workspace |
| Dependency | Assignment, dữ liệu trong project, chất lượng nhãn/index, quyền truy cập |
| Repeat condition | Có nhu cầu dữ liệu tiếp theo hoặc cần đánh giá lại do thay đổi dữ liệu |

**Nature:** workflow theo nhiệm vụ trong project, có yếu tố phản ứng với thay đổi dữ liệu. Không có căn cứ áp nhịp habit hàng ngày.

**Kết luận theo template:** “Đối với kỹ sư ADAS, core action hoàn tất nhiệm vụ tìm và chọn video có evidence thường xuất hiện theo nhiệm vụ hoặc thay đổi dữ liệu vì nhu cầu phụ thuộc yêu cầu dự án và dữ liệu hiện hành. Do đó, nhịp đo phù hợp là mỗi cơ hội nhiệm vụ ở cấp người dùng–project; tổng hợp theo tuần để theo dõi vận hành.”

Tuần là nhịp báo cáo, không phải lời khẳng định người dùng phải quay lại mỗi tuần. Retention chính đo lần nhiệm vụ tiếp theo; người không có nhu cầu mới không bị mặc định là churn. Cửa sổ tối đa 28 ngày ở mục 04 là giả thuyết pilot, cần so với thời gian assignment thực tế và ghi revision nếu đổi.

## 03 — Metric System

**Đơn vị chung:** `person_id × project_id × task_id`. Task là một yêu cầu tìm dữ liệu, không phải mỗi lần đổi query. Loại tài khoản demo/test/bot, DE/admin và truy cập không hợp lệ. Thời gian lưu UTC; báo cáo tuần theo Asia/Ho_Chi_Minh, thứ Hai 00:00 đến thứ Hai kế tiếp.

### Activation

- **Start:** lần đầu người dùng ADAS thật gửi một search attempt hợp lệ, ghi bằng `search_attempt_completed.request_started_at` (kể cả request hợp lệ gặp lỗi backend).
- **Activation event:** lần đầu `search_task_closed(outcome=success)` đáp ứng completion rule.
- **Window:** 7 × 24 giờ từ start; giả thuyết pilot về thời gian đủ để hoàn tất task đầu.
- **Activation rate:** số người có success trong window / số người bắt đầu lần đầu và đã có đủ 7 ngày quan sát. Người chưa đủ window nằm ở cohort pending.
- **TTV:** thời gian start → success đầu, báo median/P90 trên người activated cùng activation rate; không bỏ người thất bại khỏi rate rồi tuyên bố onboarding tốt.

### Engagement — hai góc

| Metric | Công thức | Ý nghĩa |
|---|---|---|
| Frequency | Số task success trong tuần của nhóm eligible / số người ADAS có ≥1 task chưa đóng ở đầu tuần hoặc được giao trong tuần | Mức lặp lại giá trị trên nhóm có nhu cầu, bao gồm task giao từ tuần trước; báo kèm số người/task eligible |
| Depth: Task Success Rate | Số task trong cohort deadline của tuần đạt success trước hoặc đúng deadline / số task không withdrawn có deadline kết thúc trong tuần | Tử và mẫu cùng cohort task; unfinished quá hạn tính không thành công, withdrawn do hủy nhu cầu báo riêng |

### Retention — metric nối sang loop

**Next-Task Retention (NTR):** tỷ lệ người–project có nhiệm vụ tiếp theo đã kết thúc window và hoàn tất chính nhiệm vụ đó đúng hạn. Công thức: số unit eligible có next-task success / tổng unit eligible đã hết window. Sáu thành phần, cách chọn assignment, pending/withdrawn/no-opportunity và cửa sổ được định nghĩa đầy đủ ở mục 04. Đây là metric Phase 3 mà hypothesis ở mục 05 trỏ tới.

### North Star Metric

**NSM = số nhiệm vụ tìm dữ liệu được hoàn tất có chất lượng mỗi tuần.**

- Unit of value: distinct `task_id` thuộc người dùng ADAS thật trong project.
- Quality threshold: ≥1 video đã xem evidence, được xác nhận phù hợp, lưu workspace thành công và hợp lệ trong scope/version tại lúc chọn.
- Frequency: theo tuần hoàn tất, một task tối đa một lần.
- Công thức: `COUNT(DISTINCT task_id)` của `search_task_closed(outcome=success)` đã vượt toàn bộ quality checks trong tuần.

Đây là proxy giá trị, không chứng minh clip đủ cho huấn luyện hoặc hiệu quả mô hình. Tự xác nhận relevance vẫn có thể sai, nên cần counter-metric review độc lập. Không cộng search count, video count hoặc score similarity vào NSM để tạo “chất lượng”.

### Leading metrics — tối đa ba

| Tên | Công thức | Tại sao có thể dẫn tới NSM |
|---|---|---|
| Evidence Reach Rate | Task có ≥1 `evidence_viewed` / task có search attempt đã kết thúc | Có evidence để đánh giá là tiền đề chọn video; lỗi/no result vẫn ở mẫu số |
| Time to First Evidence | Median/P90 từ request start đầu của task đến evidence view đầu; báo kèm tỷ lệ chưa xem | Giảm thời gian đến bước kiểm tra có thể giảm bỏ dở |
| Searchability Coverage | Số video trong scope đủ điều kiện tìm ngữ nghĩa / số video trong scope tại snapshot | Thiếu camera samples/index làm bỏ sót đầu vào; không dùng Recall thay thế coverage |

Coverage cần snapshot có mẫu số đầy đủ từ inventory Postgres và đối soát index. Không suy ra từ riêng top 3 hay `scope_complete`: service có thể đánh dấu semantic scope chưa đầy đủ do giới hạn candidates, khác với tỷ lệ index coverage.

### Counter-metrics

| Tên | Công thức / cách đọc |
|---|---|
| Invalid Selection Rate | Workspace item bị reviewer độc lập kết luận không đáp ứng yêu cầu / item đã được review; chọn mẫu ngẫu nhiên, báo sample size và review coverage; không coi chưa review là đúng |
| Scope Violation Count | Số lần backend phát hiện actor/project/video ngoài scope ở search/save/close; mục tiêu 0; rejected attempts được báo riêng với actual persisted violations |
| Search Cost per Success | Tổng chi phí search phục vụ người dùng trong tuần / NSM tuần; includes failed attempts; nếu NSM=0 báo N/A và tổng chi phí. Chi phí indexing chia sẻ báo riêng, chưa gán tùy tiện cho task |

Không có dữ liệu baseline để điền tỷ lệ hoặc tuyên bố uplift. Pilot trước–sau/các cohort phải cùng persona, project, loại nhiệm vụ và window; báo cỡ mẫu. Benchmark bên ngoài chỉ dùng khi cùng định nghĩa.

## 04 — Retention Definition (6 thành phần)

**Tên metric:** Next-Task Retention (NTR), retention theo cơ hội nhiệm vụ tiếp theo.

| Thành phần bắt buộc | Định nghĩa có thể tính |
|---|---|
| 1. Unit | Một `person_id × project_id` ADAS thật; không dùng shared demo account |
| 2. Cohort entry | Success đầu tiên theo core action; cohort theo tuần success đầu |
| 3. Return event | Success hợp lệ của task khác, chính là nhiệm vụ được giao tiếp theo sau cohort entry |
| 4. Window | Lấy assignment tiếp theo được ghi nhận từ nguồn nhu cầu độc lập trong 28 ngày sau entry; theo dõi từ assignment đến deadline đã chốt lúc giao, tối đa 28 ngày từ assignment. Chỉ chốt kết quả khi deadline/window kết thúc |
| 5. Threshold | ≥1 lần success của chính task tiếp theo trong window; đổi query/lưu lại cùng task không phải return |
| 6. Segment | ADAS có nhu cầu mới trong cùng project; tách loại nhiệm vụ, project và trạng thái dữ liệu. Không có assignment tiếp theo: no-opportunity, báo riêng |

**Công thức:** số unit eligible hoàn tất next task đúng window / số unit eligible có next assignment và window đã kết thúc. Task quá hạn không hoàn tất là không retained. Task bị hủy do nhu cầu bên ngoài được tách withdrawn cùng lý do, không âm thầm loại bỏ để tăng tỷ lệ. Báo thêm no-opportunity, pending, withdrawn và tỷ lệ có cơ hội trong từng cohort.

**Ví dụ giả định, không phải dữ liệu dự án:** 10 người activated; 6 có next assignment, 1 assignment bị hủy, 1 chưa hết window; còn 4 eligible đã hết window, 3 success ⇒ NTR=3/4. Báo đồng thời 4 no-opportunity, 1 withdrawn và 1 pending; không gọi 3/10 là retention của định nghĩa này.

Ghi assignment từ hệ thống/yêu cầu dự án ngay khi giao, kể cả người dùng không mở app. Nếu chỉ ghi khi họ vào app, denominator sẽ loại người không quay lại và làm NTR tăng giả. Khi chưa có nguồn assignment độc lập, metric này **chưa tính đáng tin cậy**; không thay bằng D7 trơ trọi. Window 28 ngày là đề xuất, cần kiểm chứng với lịch nhiệm vụ. Người có nhiệm vụ ở project khác không tính vào return của project hiện tại.

## 05 — Product Loop (2 chu kỳ + metric hypothesis)

**Loại loop:** workflow/progress loop; state đã tích lũy giúp nhiệm vụ kế tiếp. Workspace hiện hữu là investment; task linkage, request metadata và truy vết lựa chọn là bổ sung đề xuất.

| Bước | Chu kỳ 1 | Chu kỳ 2 |
|---|---|---|
| Natural trigger | Nhận yêu cầu tìm video tình huống A | Nhận yêu cầu B liên quan hoặc yêu cầu A thay đổi do dữ liệu mới |
| Core action | Xem evidence, xác nhận phù hợp, lưu workspace và đóng task A thành công | Dùng state cũ làm điểm bắt đầu nhưng kiểm tra evidence/scope hiện hành, chọn cho task B và đóng thành công |
| Immediate value | Có đầu vào đã được kiểm tra cho A | Có đầu vào phù hợp yêu cầu B |
| Saved state / investment | Workspace item, yêu cầu A, video/version/evidence đã chọn | Workspace được cập nhật và liên kết task B; giữ lịch sử quyết định |
| Next natural trigger | Nhu cầu B phát sinh hoặc dữ liệu/nhãn của tập đang dùng thay đổi | Nhu cầu C hoặc thay đổi tiếp theo khiến cần tìm/kiểm tra lại |
| Repeat value | Hoàn tất một task mới nhờ tận dụng ngữ cảnh đã có | Tiếp tục tích lũy đầu vào và ngữ cảnh có thể dùng lại |

Notification chỉ báo có thay đổi, không tự tạo reason to return. Việc mở lại workspace hay đọc notice chưa đạt core action; nếu không có nhu cầu tìm mới, không ghi success mới. Nếu video cũ đã bị xóa/đổi version, không tái sử dụng mù quáng.

**Metric hypothesis:** “Trong pilot 6 tuần, việc lưu yêu cầu và video/version đã chọn cùng workspace sẽ làm **Next-Task Retention tăng** so với baseline có cùng loại task và project, vì người có nhiệm vụ tiếp theo có thể dùng lại ngữ cảnh và tìm được đầu vào nhanh hơn.” Chỉ phân tích các window đã kết thúc; tiếp tục theo dõi sau pilot cho cohort còn pending.

**Cách kiểm chứng:** baseline trước thay đổi hoặc nhóm đối chứng phù hợp; giữ định nghĩa NTR ở mục 04. Xem Time to First Evidence giảm để kiểm tra cơ chế; Invalid Selection Rate không được xấu đi. Không đặt target % khi chưa có baseline. Nếu NTR không tăng hoặc chất lượng giảm, xem lại usefulness của saved state, phân bố nhiệm vụ và độ đầy đủ dữ liệu; không thêm streak/badge để ép nhịp.

## 06 — Tracking nhanh (8 events + acceptance criteria)

Các event sau là **đề xuất instrumentation**, không phải tuyên bố code hiện tại đã có. `latency_ms`, scope counters và trace trong service là nguồn kỹ thuật hỗ trợ, không tự thành product analytics. Search request thất bại vẫn cần được ghi ở lớp API/application, không chỉ nhánh return thành công.

| Event | Ý nghĩa và trigger chính xác | Metric được map |
|---|---|---|
| `search_task_created` | Assignment từ nguồn nhu cầu độc lập được persist; gồm actor, project, request, assigned_at, deadline, source và loại task; không chờ user mở app | NTR opportunity/denominator; engagement denominator |
| `search_attempt_completed` | Một request search kết thúc ở API layer, thành công/no result/error; ghi request_started_at, completed_at, task_id, status, latency, result count, scope checks và measured search cost | Activation start, TTV start, Evidence Reach denominator, Time to First Evidence start, Search Cost per Success, Scope Violation Count |
| `evidence_viewed` | Evidence frame hợp lệ đã tải và hiển thị trong panel người dùng; ghi task/video/frame/version; không bắn vì thumbnail nằm trong response | Quality threshold, Evidence Reach Rate, Time to First Evidence |
| `result_relevance_confirmed` | Actor chủ động xác nhận video đáp ứng yêu cầu task sau khi xem evidence; backend persist confirmation | Quality threshold của NSM/activation/NTR |
| `workspace_item_added` | Backend đã commit workspace item với task/video/version, không phải click save hoặc optimistic UI | Quality threshold của NSM/activation/NTR; Scope Violation Count ở save |
| `search_task_closed` | Task chuyển lần đầu sang terminal success/unsuccessful/withdrawn; success chỉ sau backend validation; quá deadline job chốt unsuccessful nếu chưa hoàn tất; hủy có lý do | NSM, activation, TTV end, Task Success Rate, Frequency, NTR và trạng thái withdrawn |
| `selection_reviewed` | Reviewer độc lập persist verdict cho workspace item, theo rubric yêu cầu task; review_id/version được ghi | Invalid Selection Rate và review coverage |
| `project_searchability_snapshotted` | Job đối soát inventory/video current và camera samples/index hoàn tất, ví dụ mỗi ngày và sau commit; lưu numerator, denominator, version, timestamp | Searchability Coverage |

**Contract:** mỗi event có `event_id`, event/schema version, UTC timestamp, `person_id`, role, `project_id`, source và environment. Event task có `task_id`; search có `request_id`; selection có `video_id`, dataset/index/label versions và `workspace_item_id` khi đã tồn tại. Dùng outbox hoặc cơ chế delivery có idempotency. Analytics phân biệt event time và ingestion time, không double count do delivery retry. Không đưa token, credentials, raw private query/video vào analytics nếu không cần; dùng request reference và thuộc tính phân loại.

Task identity theo yêu cầu; nhiều query/reformulations nằm trong một task. Không tạo task mới để mỗi save làm tăng NSM. Không cho chuyển success→success để đếm lại; nếu cần sửa quyết định, lưu revision/correction và tính lại aggregate có version. Mỗi người–project chỉ có một cohort entry và một next assignment đầu tiên của định nghĩa NTR.

### Acceptance criteria

1. **Completion đúng:** Given search trả top 3 nhưng chưa xem evidence/xác nhận/lưu; when người dùng đóng tab hoặc click “hoàn tất”; then không phát `search_task_closed(success)` và NSM/activation/NTR không tăng. Chỉ sau đủ evidence, confirmation và workspace commit hợp lệ mới cho success.
2. **Chống đếm trùng:** Given task đã success; when reload, retry save hoặc delivery event lặp; then cùng idempotency key không tạo thêm workspace item/event logic, và `COUNT(DISTINCT task_id)` vẫn tăng đúng 1. Autosave không tạo success.
3. **Scope/version:** Given video đã bị xóa/không còn quyền sau lúc search; when save/close; then backend kiểm tra lại, từ chối selection không hợp lệ, ghi rejected scope check và không tăng NSM. Một lựa chọn hợp lệ ở thời điểm trước được giữ lịch sử, không giả vờ là phiên bản mới.
4. **Không quay lại vẫn có denominator:** Given next task đã được giao từ nguồn ngoài app, deadline đã hết và người dùng không mở app; then assignment vẫn được ghi, task được chốt unsuccessful và unit tính không retained. Chưa hết window thì pending; không có assignment thì no-opportunity.
5. **Lỗi và no result:** Given request hợp lệ nhưng Qdrant lỗi hoặc trả 0 kết quả; then attempt được ghi với status/timing/cost, không có evidence view giả hoặc success giả; mẫu số leading/cost vẫn bao gồm attempt đó.

### Revision và kiểm tra Phase 5

| Revision | Lý do |
|---|---|
| Đổi đơn vị giá trị từ search/video sang task hoàn tất có quality checks | Search count và top 3 chưa chứng minh người dùng có đầu vào phù hợp; nhiều query không phải nhiều giá trị |
| Dùng NTR theo assignment; tuần chỉ để aggregate | Nhịp đến từ nature của workflow, tránh ép daily/weekly và coi tuần không nhu cầu là churn |
| Đưa nguồn assignment độc lập và deadline vào contract | Chặn denominator chỉ chứa người đã quay lại, giúp retention tính được |
| Bổ sung inventory snapshot; cost nằm trong attempt event | Coverage cần mẫu số inventory, cost cần failed attempts; mọi metric phải có nguồn đo |
| Ghi tên NTR trong hypothesis và nối tới mục 04 | Loop cần giả thuyết kiểm chứng bằng metric đã định nghĩa |
| Đưa NTR vào Metric System Phase 3; làm rõ cohort của engagement | Đáp ứng yêu cầu hypothesis trỏ về Phase 3; tránh tử/mẫu thuộc nhóm task hoặc người khác nhau |
| Phân biệt code hiện hữu với tracking đề xuất và quality proxy | Không biến thiết kế lab thành tuyên bố hệ thống đã triển khai hoặc chất lượng đã được chứng minh |

**Bảy tự kiểm Phase 5:** core action là hành động giá trị (đạt); activation không phải login/tutorial (đạt); tần suất không đếm query trùng (đạt); reason to return là nhiệm vụ/thay đổi dữ liệu (đạt); retention theo cadence nhiệm vụ (đạt về thiết kế); 8/8 event map metric (đạt); mọi metric có nguồn event/contract rõ (đạt về thiết kế, cần triển khai).

**Nguồn:** [README dự án](https://github.com/AI20K-Build-Phase-Cohort-4/P-115), [search service](https://github.com/AI20K-Build-Phase-Cohort-4/P-115/blob/main/src/search/service.py), bài học/lab Day20 tại [VLearn](https://vlearn.dev/course/k04-l34-p2-t1/reader?day=D08&part=lab-961eda13-s12-doc). Không có số liệu retention, chi phí hay uplift thực tế được cung cấp trong phiên này.
