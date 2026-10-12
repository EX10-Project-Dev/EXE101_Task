# Đề xuất thiết kế entity cho FIND&GO

Tài liệu này là bản đề xuất để FE, BE và nhóm dữ liệu thống nhất trước khi tạo bảng/entity trong EF Core. Đây chưa phải schema đã được chốt. Tên field dùng PascalCase theo C#; dữ liệu lưu database dùng các entity bên dưới, còn dữ liệu gửi/nhận API dùng DTO riêng.

## Phạm vi và thứ tự triển khai

Theo tài liệu yêu cầu, sản phẩm có ba flow cấp cao:

1. Khách chưa đăng nhập tìm địa điểm, xem gợi ý, xem chi tiết và mở Google Maps.
2. Khách có tài khoản tạo hồ sơ/bản đồ, lưu địa điểm, nhận gợi ý dựa trên địa điểm đã lưu, follow người khác và xem địa điểm/review được chia sẻ.
3. Doanh nghiệp tạo hồ sơ, đăng địa điểm và tạo khuyến mãi.

**MVP nên làm trước:** `Venue`, category/tag/amenity, giờ mở cửa, ảnh và `NavigationLog` nếu nhóm giữ yêu cầu ghi log. Các entity user/social và B2B được mô tả bên dưới để định hướng, nhưng chỉ triển khai khi flow tương ứng đã được thống nhất.

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
| `Status` | `VenueStatus` | Có | Trạng thái: đề xuất dùng `Pending`, `Published`, `Closed`, `Rejected`. |
| `CreatedAt` | `DateTimeOffset` | Có | Thời điểm tạo bản ghi. |
| `UpdatedAt` | `DateTimeOffset?` | Không | Thời điểm sửa gần nhất. |
| `CreatedByUserId` | `string?` | Không | User tạo dữ liệu; để trống nếu do nhóm nhập/seed. |
| `BusinessProfileId` | `Guid?` | Không | Doanh nghiệp sở hữu/quản lý địa điểm, dùng ở flow B2B. |

**Ghi chú:** `GoogleMapsUrl` có thể dựng từ `GooglePlaceId` hoặc tọa độ khi cần; không nhất thiết lưu trùng nếu URL chỉ phục vụ mở bản đồ. Khoảng cách đến người dùng không phải thuộc tính cố định của quán.

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
| `ProfileVisibility` | `ProfileVisibility` | Có | `Public`, `FollowersOnly` hoặc `Private`; mặc định theo quyết định của nhóm. |
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
| `Visibility` | `ContentVisibility` | Có | `Public`, `FollowersOnly`, `Private`. |
| `CreatedAt` | `DateTimeOffset` | Có | Thời điểm tạo. |
| `UpdatedAt` | `DateTimeOffset?` | Không | Thời điểm sửa gần nhất. |

### `UserMapVenue` — địa điểm được lưu vào bản đồ

| Field | Kiểu C# gợi ý | Bắt buộc? | Ý nghĩa |
| --- | --- | --- | --- |
| `UserMapId` | `Guid` | Có | Bản đồ nhận địa điểm. |
| `VenueId` | `Guid` | Có | Địa điểm được lưu. |
| `AddedAt` | `DateTimeOffset` | Có | Thời điểm lưu. |
| `Note` | `string?` | Không | Ghi chú riêng của chủ bản đồ. |

Cặp (`UserMapId`, `VenueId`) duy nhất để tránh lưu trùng một quán vào cùng bản đồ. Việc lưu vào map không tự động có nghĩa user đã ghé hoặc review quán.

### `VenueLike` — lượt tim

| Field | Kiểu C# gợi ý | Bắt buộc? | Ý nghĩa |
| --- | --- | --- | --- |
| `UserId` | `string` | Có | User thả/bỏ tim. |
| `VenueId` | `Guid` | Có | Địa điểm được tim. |
| `CreatedAt` | `DateTimeOffset` | Có | Thời điểm thả tim. |

Cặp (`UserId`, `VenueId`) duy nhất. Khi bỏ tim, xóa quan hệ này. Nếu nhóm quyết định tim chính là lưu vào map, có thể bỏ entity này và dùng một hành động thống nhất.

### `Review` — đánh giá/review

| Field | Kiểu C# gợi ý | Bắt buộc? | Ý nghĩa |
| --- | --- | --- | --- |
| `Id` | `Guid` | Có | Khóa chính review. |
| `UserId` | `string` | Có | Người viết review. |
| `VenueId` | `Guid` | Có | Địa điểm được review. |
| `Rating` | `int?` | Không | Điểm 1–5; null nếu chỉ viết nhận xét. |
| `Content` | `string?` | Không | Nội dung review; nhóm có thể giới hạn độ dài, ví dụ 2.000 ký tự. |
| `Visibility` | `ContentVisibility` | Có | Ai được xem: public, followers hoặc private. |
| `CreatedAt` | `DateTimeOffset` | Có | Thời điểm đăng. |
| `UpdatedAt` | `DateTimeOffset?` | Không | Thời điểm sửa. |
| `IsHidden` | `bool` | Có | Ẩn review khi bị kiểm duyệt; mặc định false. |

Ràng buộc: `Rating` nằm trong 1–5; phải có ít nhất `Rating` hoặc `Content`. Nếu mỗi user chỉ được có một review hiện hành cho một địa điểm, tạo unique index (`UserId`, `VenueId`); nếu muốn lưu nhiều lần ghé/review thì bỏ ràng buộc đó.

### `UserFollow` — theo dõi người dùng

| Field | Kiểu C# gợi ý | Bắt buộc? | Ý nghĩa |
| --- | --- | --- | --- |
| `FollowerUserId` | `string` | Có | User chủ động theo dõi. |
| `FollowedUserId` | `string` | Có | User được theo dõi. |
| `CreatedAt` | `DateTimeOffset` | Có | Thời điểm follow. |

Cặp ID là duy nhất; không cho phép hai ID giống nhau. Nếu cần duyệt yêu cầu follow tài khoản riêng tư, thêm `Status` như `Pending`/`Accepted`/`Rejected`; nếu follow luôn công khai thì chưa cần field này.

## 3. Entity log và tìm kiếm

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

`RecommendationRequestDto` có thể nhận `Prompt`, `Purpose`, `BudgetMin`, `BudgetMax`, `MaxDistanceKm`, `GroupSize`, `CategoryCodes`, `AtmosphereTags`, `AmenityCodes`, `Latitude`, `Longitude`. Các field tùy chọn theo giao diện; tọa độ cần thiết khi dùng lọc theo khoảng cách/Near me.

`RecommendationResultDto` nên trả `VenueId`, `Name`, `Category`, tọa độ, địa chỉ, khoảng giá, ảnh/tiện ích cần hiển thị, `DistanceKm`, `MatchScore` và `Reasons`. `DistanceKm`, `MatchScore`, `Reasons` là kết quả tính toán, không phải field cố định của `Venue`.

Nếu cần khôi phục tìm kiếm sau khi refresh/đăng nhập trên thiết bị khác, có thể lưu thêm `SearchSession` gồm `Id`, `UserId?`, `AnonymousSessionId?`, `CriteriaJson`, `CreatedAt`, `ExpiresAt`. Nếu chỉ cần giữ kết quả trong cùng thiết bị, FE có thể giữ trạng thái tìm kiếm tạm thời; chưa cần entity này.

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

## 7. Những điểm nhóm cần xác nhận trước khi migrate database

- [ ] Khách có thể tìm và xem địa điểm trước khi đăng nhập không, hay phải đăng nhập ngay từ đầu?
- [ ] “Thả tim” và “lưu vào bản đồ” là một hành động hay hai hành động?
- [ ] Một user được có một hay nhiều review cho cùng một địa điểm?
- [ ] Bản đồ của user có thể public/followers/private không?
- [ ] Review được xem công khai, chỉ follower, hay theo quyền của profile?
- [ ] Có cần lưu lượt ghé (`VenueVisit`) riêng với review không?
- [ ] Có cần khôi phục tìm kiếm từ server hay chỉ giữ tạm trên thiết bị?
- [ ] Ai được tạo/publish `Venue`, và B2B cần xác minh doanh nghiệp ở mức nào?

Không tạo migration trước khi nhóm chốt các quyết định ảnh hưởng đến quan hệ và quyền riêng tư. Có thể triển khai MVP địa điểm trước, rồi thêm social/B2B migrations theo từng flow.
