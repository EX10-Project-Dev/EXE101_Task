# FIND&GO: luồng tìm địa điểm cho khách chưa đăng nhập

## Mục tiêu

Hoàn thành luồng cho khách chưa đăng nhập: giới thiệu FIND&GO, nhận nhu cầu đi đâu, đề xuất địa điểm F&B phù hợp, cho xem chi tiết và mở đường đi trên Google Maps. Khi khách mở chỉ đường, hệ thống ghi nhận log theo thiết kế backend.

Task này dựa trên tài liệu `Customer user( chưa authentication) prompt 1-3 lệnh → register.docx`. Phạm vi hiện tại không bao gồm tạo tài khoản, hồ sơ cá nhân, theo dõi người dùng hay luồng tài khoản doanh nghiệp (B2B).

## Luồng người dùng

1. **Landing Page** — giới thiệu FIND&GO và cho khách bắt đầu tìm kiếm.
2. **Nhập nhu cầu** — khách nhập hoặc chọn các tiêu chí.
3. **Diễn giải và tìm kiếm** — backend chuẩn hóa yêu cầu; điều kiện bắt buộc dùng để lọc, sở thích dùng để xếp hạng.
4. **Địa điểm gợi ý** — hiển thị kết quả theo thứ tự phù hợp, kèm thông tin tóm tắt và lý do được gợi ý. Số kết quả tùy dữ liệu phù hợp, không mặc định phải có đúng ba địa điểm.
5. **Chi tiết địa điểm** — hiển thị thông tin địa điểm có sẵn.
6. **Chọn địa điểm** — khách chọn một địa điểm.
7. **Mở Google Maps** — mở bản đồ/chỉ đường cho địa điểm được chọn và ghi nhận log theo thiết kế đã thống nhất.

## Tiêu chí tìm kiếm

| Tiêu chí | Ví dụ giá trị |
| --- | --- |
| Purpose | Study / Work, Eat & Drink, Hangout, Date, Group Gathering, Explore |
| Budget | Số tiền hoặc mức ngân sách mỗi người |
| Distance | Near me, bán kính hoặc khoảng cách tối đa |
| Group Size | Solo, 2 people, 3–5 people, 6+ people hoặc số cụ thể |
| Category | Café, Restaurant, Tea & Beverage, Dessert, Fast Food, Other F&B |
| Atmosphere | Quiet, Cozy, Social, Spacious, Outdoor, Aesthetic |
| Amenities | Wi-Fi, Power Outlets, Group Seating, Air Conditioning, Parking, Restroom |

## Phân công

### Nhã — FE UI

- [ ] Dựng Landing Page và nút bắt đầu tìm kiếm.
- [ ] Dựng form nhập/chọn 7 tiêu chí; có thể cho nhập 1–3 prompt/lệnh theo thiết kế sản phẩm.
- [ ] Validate đầu vào cơ bản và hiển thị trạng thái đang tìm, không có kết quả, lỗi.
- [ ] Dựng danh sách địa điểm gợi ý theo thứ tự backend trả về; hiển thị lý do gợi ý.
- [ ] Dựng trang/khung chi tiết địa điểm.
- [ ] Thêm thao tác chọn địa điểm và mở Google Maps.
- [ ] Kết nối UI với API backend sau khi thống nhất contract.

### Thái — BE: API và nghiệp vụ luồng

- [ ] Chốt request/response API với Nhã và Khôi.
- [ ] Tạo endpoint nhận nhu cầu tìm kiếm và trả kết quả gợi ý.
- [ ] Tạo endpoint lấy chi tiết địa điểm.
- [ ] Validate request, xử lý input thiếu/sai và trả lỗi có cấu trúc.
- [ ] Tạo endpoint ghi log khi khách chọn địa điểm/mở chỉ đường (không yêu cầu đăng nhập).
- [ ] Viết hướng dẫn chạy API và cập nhật ví dụ request/response vào README.

### Khôi — BE: dữ liệu địa điểm và tìm kiếm

- [ ] Chốt model địa điểm và mapping các trường với 7 tiêu chí tìm kiếm.
- [ ] Chuẩn bị dữ liệu mẫu địa điểm F&B để FE/BE có thể tích hợp.
- [ ] Cài logic lọc các điều kiện bắt buộc.
- [ ] Cài logic xếp hạng theo các sở thích còn lại.
- [ ] Trả danh sách theo thứ tự điểm phù hợp, kèm lý do gợi ý và không ép số lượng kết quả cố định.
- [ ] Thống nhất với bạn cách xử lý trường hợp không có dữ liệu hoặc không có địa điểm phù hợp.

## API contract đề xuất — cần BE chốt trước khi FE tích hợp

Các đường dẫn dưới đây là gợi ý để chia việc, chưa phải contract đã được xác nhận.

- `POST /api/recommendations` — nhận tiêu chí/prompt và trả danh sách địa điểm đã xếp hạng.
- `GET /api/venues/{id}` — trả chi tiết một địa điểm.
- `POST /api/navigation-logs` — nhận địa điểm được chọn và ghi nhận hành động mở chỉ đường.

FE và BE cần thống nhất cấu trúc request/response, tên field, cách biểu diễn tiêu chí bắt buộc/sở thích và dữ liệu vị trí trước khi tích hợp.

## Hoàn thành khi

- [ ] Khách chưa đăng nhập đi được trọn luồng từ Landing Page đến Google Maps.
- [ ] Form gửi được các tiêu chí đã chọn/nhập và xử lý được lỗi, loading, kết quả rỗng.
- [ ] Kết quả được lọc theo điều kiện bắt buộc và xếp hạng theo sở thích.
- [ ] Khách xem được chi tiết, chọn địa điểm và mở đúng Google Maps.
- [ ] Log điều hướng được ghi theo contract backend.
- [ ] FE, API và dữ liệu mẫu đã tích hợp với nhau.

## Cách cập nhật task

Mỗi người tự tick `- [ ]` thành `- [x]` khi hoàn tất mục của mình. Nếu một mục đang làm dở hoặc bị chặn, ghi chú ngay bên dưới mục đó để cả nhóm biết.
