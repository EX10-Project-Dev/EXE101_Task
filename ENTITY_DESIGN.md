# Đề xuất thiết kế entity cho FIND&GO

Tài liệu này là bản đề xuất để FE, BE và nhóm dữ liệu thống nhất trước khi tạo bảng/entity trong EF Core. Đây chưa phải schema đã được chốt. Tên field dùng PascalCase theo C#; dữ liệu lưu database dùng các entity bên dưới, còn dữ liệu gửi/nhận API dùng DTO riêng.

## Tóm tắt nhanh cho cả nhóm

Phần này là bản đồ tổng quan để mọi người nắm mô hình trước khi đọc từng bảng và field chi tiết. FIND&GO lấy Atly làm hình mẫu cho trải nghiệm khám phá địa điểm trên bản đồ và hoạt động cộng đồng; tài liệu này chỉ chọn các hành vi phù hợp với yêu cầu nhóm và phạm vi đồ án.

| Nhóm dữ liệu | Entity chính | Dùng để làm gì |
| --- | --- | --- |
| Địa điểm | `Venue`, `VenueCategory`, `VenueTag`, `Amenity`, `VenuePhoto`, `OpeningHour`, `OpeningHourException` | Lưu từng quán/chi nhánh cùng thông tin cần để tìm, lọc và hiển thị trên bản đồ. |
| Tài khoản và cộng đồng | `AppUser`, `UserFollow`, `VenueLike`, `Review` | Đại diện cho tài khoản, quan hệ theo dõi, lượt thả tim và review. Các tính năng cộng đồng yêu cầu đăng nhập. |
| Bản đồ cá nhân | `UserMap`, `UserMapVenue` | Lưu các địa điểm riêng tư của một tài khoản, độc lập với việc thả tim. |
| Lịch sử tìm kiếm | `SearchHistory` | Giữ tối đa 6 lượt gần đây cho mỗi `UserId` hoặc `AnonymousSessionId`; lượt cũ nhất bị xóa khi thêm lượt thứ 7. |
| Điều hướng | `NavigationLog` | Ghi nhận thao tác mở chỉ đường nếu nhóm giữ yêu cầu này trong MVP. |
| Địa điểm do người dùng gửi | `Venue.CreatedByUserId` và trạng thái kiểm duyệt | Địa điểm mới chờ duyệt trước khi công khai; không cần tạo một bản sao `Venue` riêng cho trạng thái chờ. |
| B2B mở rộng | `BusinessProfile`, `Promotion` | Hỗ trợ doanh nghiệp xác minh/quản lý địa điểm và tạo khuyến mãi ở giai đoạn sau, ngoài MVP. |

### Các quy tắc nghiệp vụ quan trọng

- Một `Venue` là một địa điểm hoặc chi nhánh cụ thể; hai chi nhánh cùng thương hiệu là hai bản ghi.
- Khách chưa đăng nhập có thể xem bản đồ, tìm địa điểm và mở chỉ đường; không được xem/viết review, xem hồ sơ người khác hoặc react.
- Một tài khoản có tối đa một `Review` cho mỗi `Venue`; review có thể được chỉnh sửa. **Không yêu cầu xác minh người dùng đã ghé quán để được review.** Review không đồng nghĩa với việc đã ghé.
- Hồ sơ mặc định hiển thị công khai trong app. Từ review trên trang quán, người xem có thể mở hồ sơ người viết và follow/kết nối; quyền riêng tư của từng review vẫn được áp dụng.
- Review mặc định `Public` với người dùng đã đăng nhập; người viết có thể chọn `Public`, `FollowersOnly` hoặc `Private`. Guest vẫn không xem được review.
- Thả tim (`VenueLike`) và lưu vào bản đồ cá nhân (`UserMapVenue`) là hai hành động riêng. Bản đồ cá nhân không chia sẻ với người khác.
- Lưu tất cả tiêu chí tìm kiếm hợp lệ, kể cả khi khách chưa đăng nhập. Khi khách đăng nhập/đăng ký, nối lịch sử theo `AnonymousSessionId` vào tài khoản mà không tạo bản ghi trùng.
- Tọa độ GPS chỉ dùng khi người dùng chủ động chọn “Gần tôi”; không lưu vị trí chính xác thường xuyên trong hồ sơ. Nếu cần lưu tọa độ trong lịch sử của lượt đó, coi đây là dữ liệu riêng tư và chỉ lưu tối thiểu.
- Địa điểm do người dùng gửi ở trạng thái chờ duyệt. Doanh nghiệp/B2B và khuyến mãi chưa thuộc MVP.

### Cách đọc mô hình

Các entity là dữ liệu lưu bền vững trong database. DTO là cấu trúc request/response của API. Khoảng cách, điểm phù hợp và lý do gợi ý được tính theo từng lượt tìm kiếm và trả qua DTO; không lưu như thuộc tính cố định của `Venue`. Các bảng dưới đây giải thích cụ thể field, kiểu dữ liệu và ràng buộc; nhóm cần chốt các quyết định trước khi tạo migration.

## Phạm vi và thứ tự triển khai

Theo tài liệu yêu cầu, sản phẩm có ba flow cấp cao:

1. Khách chưa đăng nhập tìm địa điểm, xem gợi ý, xem chi tiết và mở Google Maps.
2. Khách có tài khoản tạo hồ sơ/bản đồ, lưu địa điểm, nhận gợi ý dựa trên địa điểm đã lưu, follow người khác và xem địa điểm/review được chia sẻ.
3. Doanh nghiệp tạo hồ sơ, đăng địa điểm và tạo khuyến mãi.

**MVP nên làm trước:** `Venue`, category/tag/amenity, giờ mở cửa, ảnh và `NavigationLog` nếu nhóm giữ yêu cầu ghi log. Theo định hướng hiện tại, khách chưa đăng nhập được xem bản đồ, tìm quán và mở đường đi; các tính năng xã hội như xem review/profile và react yêu cầu đăng nhập. User/social và B2B entities được mô tả bên dưới để định hướng, nhưng chỉ triển khai khi flow tương ứng đã được thống nhất.

## Quy ước chung

- `Guid` phù hợp cho ID của entity nghiệp vụ. `AppUser.Id` đang theo ASP.NET Identity, thường là `string`; dùng đúng kiểu ID hiện có.
- Field bắt buộc không chấp nhận `null`; field tùy chọn có thể `null`.
- Tiền dùng `decimal`, không dùng `float`/`double`; giá hiện tại tính bằng VND.
- Thời điểm lưu dưới dạng `DateTimeOffset` theo UTC; giao diện đổi sang múi giờ địa phương.
- Các trường trạng thái nên là enum hoặc mã có tập giá trị được kiểm soát, không nhập văn bản tùy ý.
- Không lưu dữ liệu dẫn xuất như khoảng cách, điểm xếp hạng hay lý do đề xuất cố định trong entity `Venue`.

## 1. Entity địa điểm

### `Venue` — một địa điểm/chi nhánh

Mỗi bản ghi là một địa điểm cụ thể. Hai chi nhánh cùng thương hiệu vẫn có hai `Venue` khác nhau.

| Field | Kiểu C# gợi ý | Bắt buộc? | Ý nghĩa |
| --- | --- | --- | --- |
| `Id` | `Guid` | Có | ID nội bộ, khóa chính của địa điểm. |
| `Name` | `string` | Có | Tên hiển thị của quán/chi nhánh. |
| `CategoryId` | `Guid` | Có | Khóa ngoại đến `VenueCategory`, ví dụ Café hoặc Restaurant. |
| `Description` | `string?` | Không | Mô tả ngắn về địa điểm. |
| `Address` | `string` | Có | Địa chỉ đầy đủ để hiển thị và định vị. |
| `Latitude` | `decimal` | Có | Vĩ độ của pin bản đồ; phải trong khoảng -90 đến 90. |
| `Longitude` | `decimal` | Có | Kinh độ của pin bản đồ; phải trong khoảng -180 đến 180. |
| `PriceMin` | `decimal?` | Không | Mức giá thấp tham khảo; giá âm không hợp lệ. |
| `PriceMax` | `decimal?` | Không | Mức giá cao tham khảo; nếu có cả hai thì phải lớn hơn hoặc bằng `PriceMin`. |
| `PriceBasis` | `PriceBasis?` | Không | Giá tính theo gì, ví dụ mỗi người hoặc mỗi món. |
| `CurrencyCode` | `string` | Có | Mã tiền tệ ISO, mặc định `VND`. |
| `GooglePlaceId` | `string?` | Không | ID địa điểm từ Google nếu có; hữu ích để đối chiếu dữ liệu. |
| `WebsiteUrl` | `string?` | Không | Website/fanpage chính thức nếu có. |
| `MenuUrl` | `string?` | Không | Link menu nếu có. |
| `PhoneNumber` | `string?` | Không | Số điện thoại liên hệ nếu có. |
| `Status` | `VenueStatus` | Có | Trạng thái: `Pending`, `Published`, `Closed`, `Rejected`. Địa điểm do user gửi bắt đầu là `Pending`; chỉ hiển thị công khai sau khi được kiểm duyệt. |
| `CreatedAt` | `DateTimeOffset` | Có | Thời điểm tạo bản ghi. |
| `UpdatedAt` | `DateTimeOffset?` | Không | Thời điểm sửa gần nhất. |
| `CreatedByUserId` | `string?` | Không | User gửi/tạo địa điểm; để trống nếu do admin/nhóm nhập hoặc seed. |
| `BusinessProfileId` | `Guid?` | Không | Doanh nghiệp sở hữu/quản lý địa điểm, dùng ở flow B2B. |

**Ghi chú:** `GoogleMapsUrl` có thể dựng từ `GooglePlaceId` hoặc tọa độ khi cần; không nhất thiết lưu trùng nếu URL chỉ phục vụ mở bản đồ. Khoảng cách đến người dùng không phải thuộc tính cố định của quán. Link chia sẻ của FIND&GO cũng không cần lưu thành field; tạo URL ổn định trỏ đến trang chi tiết theo `Venue.Id` (hoặc slug có ID), để khi mở link app/web tải đúng địa điểm.

### `VenueCategory` — danh mục chính

| Field | Kiểu C# gợi ý | Bắt buộc? | Ý nghĩa |
| --- | --- | --- | --- |
| `Id` | `Guid` | Có | Khóa chính danh mục. |
| `Name` | `string` | Có | Tên hiển thị, ví dụ Café, Restaurant. |
| `Code` | `string` | Có | Mã ổn định dùng trong API, ví dụ `cafe`; duy nhất. |
| `IsActive` | `bool` | Có | Cho biết danh mục còn được dùng để tạo/tìm địa điểm không. |

Một `Venue` có một category chính. Nếu nhóm cần nhiều category cho một quán, đổi sang quan hệ nhiều-nhiều thay vì thêm cột category thứ hai.

### `VenueTag` và `VenueTagLink` — tags/mục đích

Dùng tags cho thuộc tính linh hoạt như `Study/Work`, `Date`, `Quiet`, `Outdoor`, `Aesthetic`.

`VenueTag`:

| Field | Kiểu C# gợi ý | Bắt buộc? | Ý nghĩa |
| --- | --- | --- | --- |
| `Id` | `Guid` | Có | Khóa chính của tag. |
| `Name` | `string` | Có | Tên hiển thị. |
| `Code` | `string` | Có | Mã ổn định, duy nhất, dùng trong lọc/API. |
| `TagType` | `VenueTagType` | Có | Nhóm tag, đề xuất `Purpose` hoặc `Atmosphere`. |

`VenueTagLink` là bảng liên kết nhiều-nhiều:

| Field | Kiểu C# gợi ý | Bắt buộc? | Ý nghĩa |
| --- | --- | --- | --- |
| `VenueId` | `Guid` | Có | Địa điểm được gắn tag. |
| `VenueTagId` | `Guid` | Có | Tag được gắn cho địa điểm. |

Cặp (`VenueId`, `VenueTagId`) là khóa duy nhất để không gắn trùng tag cho cùng một quán.

### `Amenity` và `VenueAmenityLink` — tiện ích

Dùng danh mục tiện ích như Wi‑Fi, ổ cắm, điều hòa, chỗ xe máy, chỗ ô tô, nhà vệ sinh, chỗ ngồi nhóm.

`Amenity` gồm `Id: Guid`, `Name: string`, `Code: string`, `IsActive: bool`. Các field này đều bắt buộc; `Code` duy nhất.

`VenueAmenityLink` gồm `VenueId: Guid`, `AmenityId: Guid`, `Availability: AmenityAvailability?`, `Note: string?`. `Availability` đề xuất các giá trị `Available`, `Unavailable`, `Unknown`; dùng `Unknown` khi chưa kiểm chứng, không hiểu dữ liệu thiếu là “không có”. Cặp ID là duy nhất.

### `VenueOpeningHour` — giờ mở cửa thường lệ

| Field | Kiểu C# gợi ý | Bắt buộc? | Ý nghĩa |
| --- | --- | --- | --- |
| `Id` | `Guid` | Có | Khóa chính. |
| `VenueId` | `Guid` | Có | Địa điểm áp dụng giờ này. |
| `DayOfWeek` | `DayOfWeek` | Có | Thứ trong tuần. |
| `OpenTime` | `TimeOnly?` | Tùy trạng thái | Giờ mở cửa; null nếu đóng cửa hoặc chưa rõ. |
| `CloseTime` | `TimeOnly?` | Tùy trạng thái | Giờ đóng cửa; null nếu đóng cửa hoặc chưa rõ. |
| `IsClosed` | `bool` | Có | Đánh dấu đóng cửa thường lệ vào ngày này. |
| `Note` | `string?` | Không | Ghi chú như “mở xuyên đêm”. |

Nếu quán nghỉ giữa ngày hoặc có nhiều khung giờ trong một ngày, cho phép nhiều bản ghi theo `VenueId` + `DayOfWeek`. Thêm ràng buộc: `OpenTime` và `CloseTime` phải có khi `IsClosed` là false.

### `VenueHoursException` — ngày nghỉ/giờ đặc biệt

| Field | Kiểu C# gợi ý | Bắt buộc? | Ý nghĩa |
| --- | --- | --- | --- |
| `Id` | `Guid` | Có | Khóa chính. |
| `VenueId` | `Guid` | Có | Địa điểm áp dụng ngoại lệ. |
| `Date` | `DateOnly` | Có | Ngày có giờ khác thường lệ; duy nhất theo địa điểm/ngày. |
| `IsClosed` | `bool` | Có | Đóng cửa hoàn toàn trong ngày đó. |
| `OpenTime` | `TimeOnly?` | Không | Giờ mở đặc biệt nếu vẫn mở. |
| `CloseTime` | `TimeOnly?` | Không | Giờ đóng đặc biệt nếu vẫn mở. |
| `Note` | `string?` | Không | Lý do/ghi chú, ví dụ nghỉ lễ. |

### `VenuePhoto` — ảnh địa điểm

| Field | Kiểu C# gợi ý | Bắt buộc? | Ý nghĩa |
| --- | --- | --- | --- |
| `Id` | `Guid` | Có | Khóa chính ảnh. |
| `VenueId` | `Guid` | Có | Địa điểm thuộc về ảnh. |
| `ImageUrl` | `string` | Có | URL hoặc key lưu ảnh. |
| `PhotoType` | `VenuePhotoType` | Có | Ví dụ `Exterior`, `Interior`, `Menu`, `Other`. |
| `Caption` | `string?` | Không | Chú thích ảnh. |
| `SortOrder` | `int` | Có | Thứ tự hiển thị, mặc định 0. |
| `Source` | `string?` | Không | Nguồn ảnh hoặc người cung cấp. |
| `UsagePermission` | `string?` | Không | Ghi chú quyền sử dụng/giấy phép; không dùng ảnh không rõ quyền. |
| `CreatedAt` | `DateTimeOffset` | Có | Thời điểm thêm ảnh. |

## 2. Entity người dùng và hoạt động xã hội

### `AppUser` — tài khoản

Dự án BE đã có `AppUser` kế thừa ASP.NET Identity. Không tạo entity user trùng lặp; chỉ bổ sung field hồ sơ nếu cần.

| Field bổ sung | Kiểu C# gợi ý | Bắt buộc? | Ý nghĩa |
| --- | --- | --- | --- |
| `DisplayName` | `string` | Có | Tên hiển thị trên profile/review. |
| `Bio` | `string?` | Không | Giới thiệu ngắn. |
| `AvatarUrl` | `string?` | Không | Ảnh đại diện. |
| `ProfileVisibility` | `ProfileVisibility` | Có | Quyền xem profile: `Public`, `FollowersOnly` hoặc `Private`; mặc định `Public` theo quyết định hiện tại. Quyền riêng tư của từng review/map vẫn có thể khác. |
| `CreatedAt` | `DateTimeOffset` | Có | Thời điểm tạo hồ sơ. |
| `IsActive` | `bool` | Có | Tài khoản còn hoạt động không. |

Email, mật khẩu, khóa chính và trạng thái xác nhận email đã thuộc Identity; không lưu mật khẩu lần nữa trong field riêng.

### `UserMap` — bản đồ/bộ sưu tập địa điểm của user

| Field | Kiểu C# gợi ý | Bắt buộc? | Ý nghĩa |
| --- | --- | --- | --- |
| `Id` | `Guid` | Có | Khóa chính. |
| `OwnerUserId` | `string` | Có | Chủ sở hữu bản đồ; FK đến `AppUser`. |
| `Name` | `string` | Có | Tên bản đồ, ví dụ “Quán muốn thử”. |
| `Description` | `string?` | Không | Mô tả. |
| `CreatedAt` | `DateTimeOffset` | Có | Thời điểm tạo. |
| `UpdatedAt` | `DateTimeOffset?` | Không | Thời điểm sửa gần nhất. |

### `UserMapVenue` — địa điểm được lưu vào bản đồ

| Field | Kiểu C# gợi ý | Bắt buộc? | Ý nghĩa |
| --- | --- | --- | --- |
| `UserMapId` | `Guid` | Có | Bản đồ nhận địa điểm. |
| `VenueId` | `Guid` | Có | Địa điểm được lưu. |
| `AddedAt` | `DateTimeOffset` | Có | Thời điểm lưu. |
| `Note` | `string?` | Không | Ghi chú riêng của chủ bản đồ. |

Cặp (`UserMapId`, `VenueId`) duy nhất để tránh lưu trùng một quán vào cùng bản đồ. Bản đồ lưu là riêng tư theo tài khoản và người khác không xem được; vì vậy không cần field `Visibility` cho `UserMap` ở phiên bản này. Dữ liệu này vẫn còn sau khi phiên đăng nhập hết hạn và có thể dùng để cá nhân hóa gợi ý sau khi user đăng nhập lại. Việc lưu vào map không tự động có nghĩa user đã ghé hoặc review quán.

### `VenueLike` — lượt tim

| Field | Kiểu C# gợi ý | Bắt buộc? | Ý nghĩa |
| --- | --- | --- | --- |
| `UserId` | `string` | Có | User thả/bỏ tim. |
| `VenueId` | `Guid` | Có | Địa điểm được tim. |
| `Visibility` | `ContentVisibility` | Có | Ai nhìn thấy lượt tim: `Public`, `FollowersOnly`, `Private`; đề xuất mặc định `Public` trong phạm vi người đã đăng nhập. |
| `CreatedAt` | `DateTimeOffset` | Có | Thời điểm thả tim. |

Cặp (`UserId`, `VenueId`) duy nhất. Khi bỏ tim, xóa quan hệ này. Khách chưa đăng nhập không xem được lượt tim. Tim không tự động lưu địa điểm vào bản đồ cá nhân.

### `Review` — đánh giá/review

| Field | Kiểu C# gợi ý | Bắt buộc? | Ý nghĩa |
| --- | --- | --- | --- |
| `Id` | `Guid` | Có | Khóa chính review. |
| `UserId` | `string` | Có | Người viết review. |
| `VenueId` | `Guid` | Có | Địa điểm được review. |
| `Rating` | `int?` | Không | Điểm 1–5; null nếu chỉ viết nhận xét. |
| `Content` | `string?` | Không | Nội dung review; nhóm có thể giới hạn độ dài, ví dụ 2.000 ký tự. |
| `Visibility` | `ContentVisibility` | Có | Ai được xem: `Public`, `FollowersOnly` hoặc `Private`; đề xuất review mới mặc định `Public` trong phạm vi người đã đăng nhập. |
| `CreatedAt` | `DateTimeOffset` | Có | Thời điểm đăng. |
| `UpdatedAt` | `DateTimeOffset?` | Không | Thời điểm sửa. |
| `IsHidden` | `bool` | Có | Ẩn review khi bị kiểm duyệt; mặc định false. |

Ràng buộc: `Rating` nằm trong 1–5; phải có ít nhất `Rating` hoặc `Content`. User có thể review nhiều địa điểm, nhưng chỉ được có một review cho mỗi cặp (`UserId`, `VenueId`); cho phép sửa review hiện có, không tạo review thứ hai cho cùng quán. Tạo unique index cho cặp này. **Không yêu cầu check-in/GPS hay bằng chứng ghé quán để viết review**; không gắn huy hiệu “đã xác minh”. Review không tự xác nhận user đã ghé quán. Khách chưa đăng nhập không xem được review; `Public` nghĩa công khai cho người dùng đã đăng nhập trong app, không phải khách truy cập ẩn danh.

### `UserFollow` — theo dõi người dùng

| Field | Kiểu C# gợi ý | Bắt buộc? | Ý nghĩa |
| --- | --- | --- | --- |
| `FollowerUserId` | `string` | Có | User chủ động theo dõi. |
| `FollowedUserId` | `string` | Có | User được theo dõi. |
| `CreatedAt` | `DateTimeOffset` | Có | Thời điểm follow. |

Cặp ID là duy nhất; không cho phép hai ID giống nhau. Nếu cần duyệt yêu cầu follow tài khoản riêng tư, thêm `Status` như `Pending`/`Accepted`/`Rejected`; nếu follow luôn công khai thì chưa cần field này.

## 3. Entity log và tìm kiếm

### `SearchHistory` — lịch sử tìm kiếm gần đây

Mỗi lần gửi một yêu cầu tìm kiếm hợp lệ sẽ tạo một bản ghi. Lịch sử là riêng tư, chỉ chủ tài khoản mới xem/xóa được. Phiên đăng nhập hết hạn không xóa dữ liệu còn trong giới hạn lưu. Lưu tiêu chí tìm kiếm để chạy lại; không lưu nguyên danh sách kết quả vì dữ liệu địa điểm và thứ hạng có thể thay đổi. Giữ tối đa **6 lượt tìm kiếm gần đây** cho mỗi tài khoản hoặc phiên khách; khi gửi lượt thứ 7, xóa lượt cũ nhất để còn 6 lượt mới nhất.

| Field | Kiểu C# gợi ý | Bắt buộc? | Ý nghĩa |
| --- | --- | --- | --- |
| `Id` | `Guid` | Có | Khóa chính của lượt tìm kiếm. |
| `UserId` | `string?` | Không | Tài khoản thực hiện tìm kiếm; FK đến `AppUser`, null khi tìm với vai trò khách. |
| `AnonymousSessionId` | `string?` | Không | Mã phiên ngẫu nhiên của trình duyệt khi khách chưa đăng nhập. Không chứa thông tin định danh trực tiếp. |
| `CriteriaJson` | `string` | Có | Tiêu chí đã gửi, ví dụ mục đích, ngân sách, nhóm người, tags và tiện ích. |
| `CreatedAt` | `DateTimeOffset` | Có | Thời điểm thực hiện tìm kiếm. |
| `DisplayName` | `string?` | Không | Tên gợi nhớ do user đặt, nếu có lưu tìm kiếm yêu thích. |

Ràng buộc: phải có `UserId` hoặc `AnonymousSessionId`. Với tài khoản, tìm kiếm được gắn vào `UserId`; khi xem lại, FE gọi API bằng tiêu chí đã lưu và hiển thị kết quả mới. Với khách, lưu theo mã phiên ẩn danh; khi khách đăng nhập/đăng ký, backend gắn lịch sử khách vào tài khoản rồi chỉ giữ 6 lượt mới nhất tổng cộng, xóa lượt cũ hơn nếu vượt giới hạn. User có thể xóa một lượt hoặc xóa toàn bộ lịch sử. Giới hạn 6 là cấu hình nghiệp vụ, không phải field riêng trên mỗi dòng.

Chỉ ghi yêu cầu hợp lệ đã được gửi đi, không ghi form nháp. Nếu tiêu chí có vị trí GPS chính xác, chỉ lưu khi cần để chạy lại tìm kiếm; lịch sử phải được bảo vệ như dữ liệu riêng tư của user.

### `NavigationLog` — ghi nhận mở đường đi

Chỉ triển khai nếu nhóm giữ yêu cầu ghi log khi khách chọn địa điểm/mở Google Maps.

| Field | Kiểu C# gợi ý | Bắt buộc? | Ý nghĩa |
| --- | --- | --- | --- |
| `Id` | `Guid` | Có | Khóa chính log. |
| `VenueId` | `Guid` | Có | Địa điểm người dùng mở chỉ đường tới. |
| `UserId` | `string?` | Tùy flow | User đã đăng nhập; nullable nếu cho khách ẩn danh tìm kiếm. |
| `AnonymousSessionId` | `string?` | Tùy flow | Mã phiên ẩn danh để nối log, không chứa thông tin định danh trực tiếp. |
| `CreatedAt` | `DateTimeOffset` | Có | Thời điểm mở chỉ đường. |

Không lưu tọa độ chính xác của khách trong log trừ khi có nhu cầu rõ ràng và được nhóm chấp thuận. `UserId` hoặc `AnonymousSessionId` nên có ít nhất một giá trị.

### Search request / recommendation result — DTO, không mặc định là entity

`RecommendationRequestDto` có thể nhận `Prompt`, `Purpose`, `BudgetMin`, `BudgetMax`, `MaxDistanceKm`, `GroupSize`, `FoodPreferences`, `AtmosphereTags`, `AmenityCodes` và `Origin` tùy chọn. Mọi tiêu chí đều tùy chọn; nếu người dùng gửi trống tất cả, API trả gợi ý chung/popular thay vì lỗi. Nếu có `MaxDistanceKm`, cần `Origin` để tính khoảng cách; vị trí gốc mặc định có thể lấy từ GPS sau khi người dùng cấp quyền, hoặc người dùng nhập/chọn điểm bắt đầu trên bản đồ. FE/app chuyển vị trí đã chọn thành tọa độ trước khi gửi; người dùng không phải nhập tọa độ dạng số. Nếu không có `Origin`, bỏ lọc khoảng cách.

`RecommendationResultDto` nên trả `VenueId`, `Name`, `Category`, tọa độ của quán, địa chỉ, khoảng giá, ảnh/tiện ích cần hiển thị, `DistanceKm`, `MatchScore` và `Reasons`. `DistanceKm`, `MatchScore`, `Reasons` là kết quả tính toán, không phải field cố định của `Venue`. `SearchHistory` chỉ lưu tiêu chí để khôi phục tìm kiếm theo tài khoản; mỗi lần mở lại, API tính kết quả hiện tại và trả khoảng cách, điểm phù hợp, lý do mới. Link chia sẻ được dựng từ ID địa điểm và route chi tiết của ứng dụng, không lưu lặp vào entity.

`DirectionsRequestDto` là request riêng, phát sinh khi người dùng chọn quán và yêu cầu chỉ đường. Có thể nhận `Origin` và `Destination`; điểm bắt đầu mặc định lấy từ GPS khi được cấp quyền, nhưng user có thể tự nhập/chọn điểm bắt đầu. Điểm đến mặc định là quán vừa chọn và user có thể đổi thành địa điểm khác. Origin/destination được gửi dưới dạng tọa độ đã phân giải hoặc venue/place ID, không cần tạo entity chỉ để dựng đường đi. FE mở Google Maps bằng cặp điểm này; GPS không phải dữ liệu tọa độ của quán trong response.

## 4. Entity doanh nghiệp và khuyến mãi — flow B2B

### `BusinessProfile` — hồ sơ doanh nghiệp

| Field | Kiểu C# gợi ý | Bắt buộc? | Ý nghĩa |
| --- | --- | --- | --- |
| `Id` | `Guid` | Có | Khóa chính hồ sơ. |
| `OwnerUserId` | `string` | Có | Tài khoản chủ/quản lý; FK đến `AppUser`. |
| `BusinessName` | `string` | Có | Tên doanh nghiệp. |
| `ContactEmail` | `string` | Có | Email liên hệ công việc. |
| `PhoneNumber` | `string?` | Không | Số điện thoại liên hệ. |
| `WebsiteUrl` | `string?` | Không | Website/fanpage. |
| `VerificationStatus` | `BusinessVerificationStatus` | Có | `Pending`, `Verified`, `Rejected`. |
| `CreatedAt` | `DateTimeOffset` | Có | Thời điểm tạo. |
| `UpdatedAt` | `DateTimeOffset?` | Không | Thời điểm sửa. |

Một business có thể quản lý nhiều `Venue`; `Venue.BusinessProfileId` nối địa điểm với hồ sơ doanh nghiệp. Cách cho nhiều nhân viên cùng quản lý một hồ sơ cần chốt sau; khi cần, thêm bảng `BusinessMember`.

### `Promotion` — khuyến mãi

| Field | Kiểu C# gợi ý | Bắt buộc? | Ý nghĩa |
| --- | --- | --- | --- |
| `Id` | `Guid` | Có | Khóa chính khuyến mãi. |
| `BusinessProfileId` | `Guid` | Có | Doanh nghiệp tạo khuyến mãi. |
| `VenueId` | `Guid` | Có | Địa điểm áp dụng. |
| `Title` | `string` | Có | Tiêu đề. |
| `Description` | `string?` | Không | Chi tiết chương trình. |
| `DiscountType` | `DiscountType` | Có | `Percentage`, `FixedAmount`, `SpecialOffer`. |
| `DiscountValue` | `decimal?` | Không | Giá trị giảm nếu có; phần trăm trong khoảng 0–100. |
| `StartsAt` | `DateTimeOffset` | Có | Thời gian bắt đầu. |
| `EndsAt` | `DateTimeOffset` | Có | Thời gian kết thúc, phải sau `StartsAt`. |
| `Terms` | `string?` | Không | Điều kiện áp dụng. |
| `ImageUrl` | `string?` | Không | Ảnh quảng bá. |
| `Status` | `PromotionStatus` | Có | `Draft`, `Published`, `Expired`, `Cancelled`. |
| `CreatedAt` | `DateTimeOffset` | Có | Thời điểm tạo. |

## 5. Thuộc tính không nên gắn vào `Venue`

- `DistanceKm`: tính từ tọa độ người tìm tới địa điểm.
- `DrivingDistanceKm`: phụ thuộc điểm xuất phát, dịch vụ chỉ đường và thời điểm truy vấn.
- `MatchScore`/`RecommendationReasons`: phụ thuộc yêu cầu tìm kiếm và thuật toán hiện tại.
- `IsLiked`, `IsSaved`, `VisitedAt`: thuộc về tương tác của từng user, không phải bản thân quán.
- `ObservedNoiseLevel`: nếu giữ, cần ghi nhận người và thời điểm đo; có thể để trong review/observation thay vì coi là sự thật cố định của quán.

## 6. Quan hệ chính

```text
VenueCategory 1 ─── * Venue
Venue * ─── * VenueTag             (qua VenueTagLink)
Venue * ─── * Amenity              (qua VenueAmenityLink)
Venue 1 ─── * VenueOpeningHour
Venue 1 ─── * VenueHoursException
Venue 1 ─── * VenuePhoto
AppUser 1 ─── * UserMap
UserMap * ─── * Venue               (qua UserMapVenue)
AppUser * ─── * Venue               (qua VenueLike)
AppUser 1 ─── * Review * ─── 1 Venue
AppUser * ─── * AppUser             (qua UserFollow)
BusinessProfile 1 ─── * Venue
BusinessProfile 1 ─── * Promotion * ─── 1 Venue
Venue 1 ─── * NavigationLog
```

## 7. Trả lời và quyết định còn mở

Các ý dưới đây phản ánh đề xuất hiện tại của Thái; cần cả nhóm xác nhận trước khi tạo migration.

| Câu hỏi | Định hướng hiện tại | Ghi chú thiết kế |
| --- | --- | --- |
| 1. Khách chưa đăng nhập được làm gì? | Khách chưa đăng nhập xem map, tìm quán và mở đường đi được; không xem/đăng review, xem profile người khác hoặc react. Các tính năng cộng đồng yêu cầu đăng nhập. | Đây là chế độ khách tương tự cách nhóm muốn dùng app bản đồ: cho dùng chức năng bản đồ cơ bản, yêu cầu tài khoản để tham gia nội dung xã hội. “Premium” hiện được hiểu là tính năng yêu cầu đăng nhập, không phải gói trả phí. |
| 2. Tim và lưu map có khác nhau không? | Có. Tim có thể hiển thị trên hoạt động/profile theo quyền riêng tư; lưu là danh sách cá nhân gắn với tài khoản và private mặc định. | `UserMap`/`UserMapVenue` lưu server-side để đồng bộ theo tài khoản và cá nhân hóa gợi ý. |
| 3. User có thể review nhiều quán không? | Có, nhưng mỗi user chỉ có một review cho mỗi quán. | User có thể sửa review đó; không thể tạo review thứ hai cho cùng quán. Dùng unique index (`UserId`, `VenueId`). |
| 4. Ai xem được nội dung của user? | Profile mặc định `Public` trong app; người dùng có thể chọn quyền hiển thị profile/nội dung như `Public`, `FollowersOnly` hoặc `Private`. | Người xem mở profile từ tên/ảnh người viết review trên trang địa điểm rồi có thể follow. Guest vẫn không được xem profile/review theo chế độ guest đã chốt. |
| 5. Review mặc định hiển thị thế nào? | Khách chưa đăng nhập không xem review; review mới của user đăng nhập mặc định `Public`. | `Public` trong sản phẩm có nghĩa người dùng đăng nhập xem được, không đồng nghĩa khách ẩn danh xem được. User có thể đổi quyền nếu UI hỗ trợ. |
| 6. Review có chứng minh đã ghé quán không? | **Không cần xác minh đã ghé để review.** | Không thêm entity/check-in, không yêu cầu GPS và không gắn huy hiệu “đã xác minh”. Review không đồng nghĩa với một lượt ghé. |
| 7. Lưu lịch sử tìm kiếm bao lâu? | **Tối đa 6 lượt gần đây** cho mỗi tài khoản/phiên khách; lượt thứ 7 xóa lượt cũ nhất. | `SearchHistory` lưu tiêu chí, không lưu snapshot kết quả; user có thể xem/xóa lịch sử của mình. Guest dùng session ID; khi đăng nhập gộp lịch sử rồi chỉ giữ 6 lượt mới nhất tổng cộng. |
| 8. Ai tạo/publish quán? | Có thể cho user gửi địa điểm theo hướng UGC; đề xuất bản ghi user gửi ở trạng thái `Pending` trước khi public. | Admin/nhóm kiểm duyệt dữ liệu trong MVP. B2B không bắt buộc ở MVP; giai đoạn sau doanh nghiệp có thể xác minh hồ sơ, claim/quản lý địa điểm và đăng khuyến mãi. |
| 9. Chia sẻ link địa điểm ra sao? | Mỗi địa điểm có link ổn định; mở link sẽ vào trang chi tiết đúng địa điểm. | Dựng URL từ ID/slug; không cần field `ShareUrl` trong bảng `Venue`. |

Nhóm có thể bắt đầu triển khai các entity và flow đã chốt. Không có entity xác minh lượt ghé trong thiết kế hiện tại; giới hạn 6 lượt tìm kiếm là cấu hình nghiệp vụ, không làm thay đổi cấu trúc bảng. Có thể thêm social/B2B migrations theo từng giai đoạn.
