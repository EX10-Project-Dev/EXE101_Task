# FIND&GO — task chung

Tài liệu này là bảng công việc chung của FE và BE. Mỗi người cập nhật checklist của mình khi hoàn tất; các mục chưa được bắt đầu vẫn để trống. Thiết kế entity và các quyết định nghiệp vụ được giải thích trong [ENTITY_DESIGN.md](ENTITY_DESIGN.md).

## Mục tiêu sản phẩm

FIND&GO lấy Atly làm hình mẫu trải nghiệm: khám phá địa điểm theo nhu cầu trên bản đồ, xem thông tin/review, rồi tham gia cộng đồng bằng hồ sơ và các hoạt động quanh địa điểm. Đây là tham chiếu sản phẩm, không phải yêu cầu sao chép mọi tính năng Atly; các quyết định và giới hạn đồ án bên dưới là phạm vi của FIND&GO.

## Luồng sản phẩm

1. **Khách chưa đăng nhập:** xem bản đồ, tìm địa điểm, xem thông tin quán và mở Google Maps. Khách không xem/viết review, không xem hồ sơ người khác và không react; các thao tác này yêu cầu đăng nhập.
2. **Tìm kiếm:** người dùng nhập tiêu chí hoặc prompt; hệ thống lọc điều kiện bắt buộc, xếp hạng theo sở thích và giữ tối đa **6 lượt tìm kiếm gần đây**. Khi thực hiện lượt thứ 7, hệ thống xóa lượt cũ nhất. Khách dùng `AnonymousSessionId`; sau đăng nhập/đăng ký, lịch sử được gắn vào tài khoản và vẫn giữ tối đa 6 lượt mới nhất. Chủ tài khoản xem và xóa lịch sử của mình.
3. **Người dùng đã đăng nhập:** lưu địa điểm vào bản đồ cá nhân riêng tư (người khác không xem được), thả tim địa điểm, viết review, xem hồ sơ/review được chia sẻ và follow người khác. Từ review trên trang quán, người xem có thể mở hồ sơ người viết để kết nối/follow. Profile mặc định công khai trong app; quyền riêng tư từng review vẫn được áp dụng. Việc review có cần xác nhận đã ghé hay không để bàn sau.
4. **Chia sẻ địa điểm:** mỗi quán có link ổn định; người nhận mở link sẽ vào đúng trang chi tiết địa điểm trong FIND&GO.
5. **Đăng địa điểm cộng đồng:** người dùng gửi địa điểm mới; địa điểm ở trạng thái chờ duyệt trước khi xuất hiện công khai.
6. **B2B:** hồ sơ doanh nghiệp, xác minh/nhận quản lý địa điểm và khuyến mãi là hướng mở rộng, chưa thuộc MVP.

## Quyết định nghiệp vụ hiện tại

- Một tài khoản chỉ có **một review cho mỗi địa điểm**; có thể sửa review đó.
- Profile mặc định `Public` trong app; người dùng có thể follow người viết review từ trang địa điểm. Review mặc định `Public` với người dùng đã đăng nhập; chủ review có thể chọn `Public`, `FollowersOnly` hoặc `Private`. Khách không đăng nhập vẫn không được xem review/profile.
- Việc review có cần xác minh đã ghé hay không chưa chốt; không đưa check-in hoặc “đã xác minh” vào schema cho đến khi nhóm bàn xong.
- Lịch sử tìm kiếm lưu tiêu chí đã gửi, không lưu bản chụp kết quả. Giữ tối đa 6 lượt gần đây trên mỗi tài khoản hoặc phiên khách; khi thêm lượt thứ 7, xóa lượt cũ nhất. Lịch sử riêng tư theo chủ sở hữu và có thể xóa.
- Link chia sẻ địa điểm được tạo từ ID/slug và route trang chi tiết; không cần lưu URL trùng trong bảng địa điểm.
- Vị trí GPS chỉ lấy khi người dùng chủ động dùng tính năng “Gần tôi”; không lưu tọa độ chính xác vào hồ sơ mặc định.
- Quãng đường, điểm phù hợp và lý do gợi ý là dữ liệu tính cho kết quả/API, không phải thuộc tính cố định của địa điểm.

## Giai đoạn 1 — MVP khám phá địa điểm và lịch sử tìm kiếm

### Nhã — FE UI

- [ ] Dựng màn hình bản đồ và luồng nhập nhu cầu (tiêu chí/prompt).
- [ ] Thêm lựa chọn “Gần tôi”; giải thích và xử lý trường hợp người dùng từ chối quyền vị trí.
- [ ] Hiển thị pin, danh sách gợi ý, lý do phù hợp và trạng thái loading/rỗng/lỗi.
- [ ] Dựng chi tiết địa điểm và thao tác mở chỉ đường bằng Google Maps.
- [ ] Thêm thao tác chia sẻ link ổn định của địa điểm; link mở đúng trang chi tiết trong ứng dụng.
- [ ] Dựng đăng ký/đăng nhập tối thiểu để gắn lịch sử vào tài khoản.
- [ ] Dựng trang lịch sử tìm kiếm của tài khoản: xem và xóa lượt tìm kiếm.
- [ ] Khóa các thao tác xem/viết review, xem hồ sơ người khác và react đối với khách; hiển thị hướng đăng nhập.
- [ ] Tích hợp API sau khi FE/BE thống nhất request/response.

### Thái — BE: API và nghiệp vụ

- [ ] Chốt API contract với Nhã và Khôi: kiểu dữ liệu, validation và lỗi.
- [ ] Tạo endpoint tìm kiếm/gợi ý và endpoint chi tiết địa điểm.
- [ ] Lưu mọi yêu cầu tìm kiếm hợp lệ vào `SearchHistory`, gồm tiêu chí tìm kiếm.
- [ ] Giới hạn lịch sử ở tối đa 6 lượt cho mỗi tài khoản hoặc phiên khách; lượt thứ 7 xóa lượt cũ nhất.
- [ ] Gắn lượt tìm kiếm vào `UserId` khi đăng nhập hoặc `AnonymousSessionId` khi là khách.
- [ ] Hỗ trợ nối lịch sử khách vào tài khoản sau đăng nhập/đăng ký; tránh lưu trùng khi nối.
- [ ] Tạo API để tài khoản xem và xóa lịch sử tìm kiếm của chính mình.
- [ ] Tạo endpoint ghi navigation log nếu nhóm giữ yêu cầu ghi nhận thao tác mở chỉ đường.
- [ ] Cập nhật README hướng dẫn chạy API và ví dụ request/response.

### Khôi — BE: dữ liệu địa điểm và tìm kiếm

- [ ] Chốt model địa điểm, category, tag, tiện ích, giờ mở cửa và ảnh theo [ENTITY_DESIGN.md](ENTITY_DESIGN.md).
- [ ] Chuẩn bị dữ liệu mẫu địa điểm có tọa độ hợp lệ để FE/BE tích hợp.
- [ ] Cài lọc các điều kiện bắt buộc như ngân sách, khu vực/khoảng cách, nhóm khách và category.
- [ ] Cài xếp hạng theo sở thích; trả lý do gợi ý và không ép số lượng kết quả cố định.
- [ ] Hỗ trợ “Gần tôi” khi có tọa độ được người dùng cho phép; xử lý hợp lý khi không có tọa độ.

## Tiêu chí tìm kiếm gợi ý

| Tiêu chí | Ví dụ |
| --- | --- |
| Mục đích | Học/làm việc, ăn uống, tụ tập, hẹn hò, đi nhóm, khám phá |
| Ngân sách | Khoảng giá mỗi người (VND) |
| Khoảng cách | Gần tôi hoặc khoảng cách tối đa |
| Số người | Một người, hai người, 3–5, 6+ hoặc số cụ thể |
| Loại địa điểm | Café, nhà hàng, trà/nước, tráng miệng, thức ăn nhanh, khác |
| Không khí | Yên tĩnh, ấm cúng, sôi động, rộng rãi, ngoài trời |
| Tiện ích | Wi-Fi, ổ cắm, chỗ ngồi nhóm, điều hòa, bãi xe, nhà vệ sinh |

## Giai đoạn 2 — Tính năng cộng đồng (sau MVP)

- [ ] Hoàn thiện hồ sơ người dùng và quyền hiển thị hồ sơ.
- [ ] Tạo bản đồ cá nhân riêng tư, không chia sẻ danh sách lưu với người khác.
- [ ] Thêm thả tim địa điểm, review (mỗi người một review/địa điểm, được chỉnh sửa) và quyền riêng tư review.
- [ ] Profile mặc định công khai; mở profile của người viết từ review trên trang quán.
- [ ] Thêm follow/kết nối và xem nội dung được chia sẻ theo quyền.
- [ ] Thống nhất sau về việc review có cần chứng minh đã ghé hay không trước khi thiết kế phần check-in/visited.
- [ ] Cho người dùng gửi địa điểm mới; hỗ trợ trạng thái chờ duyệt và quy trình quản trị.

## Giai đoạn 3 — B2B (ngoài MVP)

- [ ] Chốt nhu cầu chủ doanh nghiệp, xác minh/nhận quản lý địa điểm và quyền chỉnh sửa.
- [ ] Thiết kế và triển khai chương trình khuyến mãi sau khi luồng B2B được duyệt.

## API contract cần thống nhất

Đây là nhóm endpoint gợi ý để FE/BE cùng chốt, chưa phải contract cuối cùng:

- `POST /api/search` — tìm kiếm và lưu lịch sử; nhận tiêu chí/prompt cùng vị trí nếu người dùng đã cho phép.
- `GET /api/venues/{id}` — lấy chi tiết địa điểm.
- `GET /venues/{id}` — route trang địa điểm có thể chia sẻ; mở đúng detail trong web/app.
- `GET /api/search-history` — lấy lịch sử của tài khoản hiện tại.
- `DELETE /api/search-history/{id}` — xóa một lượt tìm kiếm thuộc tài khoản hiện tại.
- `POST /api/search-history/attach-anonymous` — nối lịch sử session khách sau khi xác thực (có thể gộp vào luồng đăng nhập thay vì endpoint riêng).
- `POST /api/navigation-logs` — ghi thao tác mở chỉ đường nếu được giữ trong phạm vi.

BE cần thống nhất cách cấp/duy trì `AnonymousSessionId`, cấu trúc tiêu chí và chính sách lưu tọa độ trước khi FE tích hợp. Giới hạn lịch sử là 6 lượt trên mỗi tài khoản hoặc phiên khách; khi nối lịch sử khách vào tài khoản, chỉ giữ 6 lượt mới nhất tổng cộng. Không trả lịch sử của người khác qua API.

## Hoàn thành Giai đoạn 1 khi

- [ ] Khách có thể tìm địa điểm, xem chi tiết và mở Google Maps.
- [ ] API lọc điều kiện bắt buộc, xếp hạng theo sở thích và xử lý kết quả rỗng/lỗi.
- [ ] Mỗi tài khoản/phiên khách giữ tối đa 6 lượt tìm kiếm hợp lệ; lượt thứ 7 xóa lượt cũ nhất. Khi nối lịch sử khách, chỉ giữ 6 lượt mới nhất tổng cộng.
- [ ] Link chia sẻ địa điểm mở đúng trang chi tiết quán.
- [ ] Sau đăng nhập/đăng ký, lịch sử khách được nối vào tài khoản theo chính sách đã thống nhất.
- [ ] Tài khoản xem và xóa được lịch sử của chính mình.
- [ ] FE, API và dữ liệu mẫu đã tích hợp; checklist chỉ được tick khi mục tương ứng thật sự hoàn tất.

## Cách cập nhật

Mỗi người tự đổi `- [ ]` thành `- [x]` khi hoàn thành phần mình và ghi chú ngắn nếu mục đang bị chặn. Các giai đoạn sau MVP vẫn để mở cho đến khi nhóm xác nhận triển khai.
