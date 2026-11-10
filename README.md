# Tài liệu Phân tích và Đảo ngược Mã nguồn Ứng dụng TNVN

Tài liệu này tổng hợp kết quả phân tích kỹ thuật, cấu trúc mã nguồn, hạ tầng backend và các cơ chế bảo mật được trích xuất từ gói cài đặt APK của ứng dụng Thanh Niên Việt Nam (TNVN).

---

## 1. Công nghệ và Ngôn ngữ sử dụng

Biểu đồ kiến trúc công nghệ: `https://files.catbox.moe/mzf8de.svg`

* **Dart (Flutter):** Chiếm khoảng 90% logic nghiệp vụ của ứng dụng. Mã nguồn được biên dịch AOT (Ahead-Of-Time) thành mã máy 64-bit nằm trong thư viện `libapp.so`.
* **Java / Kotlin:** Đóng vai trò lớp vỏ bọc Android gốc (`MainActivity extends FlutterActivity`), chịu trách nhiệm quản lý và điều phối các dịch vụ chạy nền như Firebase Cloud Messaging (FCM), WorkManager và Local Notifications.
* **C / C++ (Native Libraries):** Đảm nhiệm các tác vụ bảo mật cấp thấp và xử lý dữ liệu nặng:
  * `libanti.so`: Thư viện phát hiện can thiệp, chống dịch ngược và kiểm tra môi trường chạy.
  * `libekyc-native-full.so`: Xử lý hình ảnh phục vụ nhận diện sinh trắc học và căn cước công dân.
  * `libwechatqrcode.so`: Thuật toán quét và giải mã mã QR tối ưu.

---

## 2. Kiến trúc Nền tảng và Cập nhật OTA

### 2.1. Nền tảng đa tổ chức (Multi-tenant Platform)
Mã nguồn ứng dụng sử dụng package gốc `package:tcctxh` (Tổ chức Chính trị - Xã hội). Ứng dụng TNVN được xây dựng trên một bộ khung Super-App dùng chung cho nhiều đoàn thể, tổ chức chính trị - xã hội tại Việt Nam, bao gồm:
* TNVN: Đoàn Thanh niên Cộng sản Hồ Chí Minh.
* HND: Hội Nông dân.
* HPN: Hội Phụ nữ.
* DHD: Đại hội Đảng.
* CD4T: Công đoàn.

### 2.2. Cập nhật mã nguồn từ xa (Over-The-Air Update)
* Ứng dụng tích hợp nền tảng Shorebird với định danh `app_id: d55cb3e9-edc8-430c-8a2e-2ffc1255e599`.
* Cơ chế này cho phép đội ngũ phát triển phát hành các bản vá mã nguồn Dart trực tiếp đến thiết bị người dùng mà không cần thông qua quy trình phê duyệt lại gói ứng dụng trên Google Play.

---

## 3. Cơ chế Hoạt động và Luồng Nghiệp vụ Chính

### 3.1. Luồng Xác thực và Quản lý Đoàn viên Điện tử (HCYU Membership)
* **Đăng ký / Đăng nhập:** Hỗ trợ đăng nhập qua số điện thoại và mật khẩu (`/v2/api/app/login`), hoặc thông qua hệ thống định danh tập trung VNPT SSO (`/auth/vdcp_sso`).
* **Xác thực eKYC:** Sử dụng SDK `com.vnptit.idg.sdk` để thu thập ảnh căn cước công dân (mặt trước/sau) và ảnh chân dung. Thư viện `libekyc-native-full.so` thực hiện kiểm tra độ mờ, nhận diện ảnh chụp màn hình giả lập trước khi gửi vector đặc trưng về endpoint `/be/v2/api/app/tnvn/update_ekyc`.
* **Nghiệp vụ công tác Đoàn:**
  * Khai báo thông tin sổ đoàn viên (`/be/api/app/tnvn/declare_member_info`).
  * Phân cấp cây tổ chức Đoàn từ Trung ương đến Chi đoàn cơ sở (`/be/api/cates/children/unions`).
  * Ban chấp hành xét duyệt hoặc từ chối đoàn viên mới (`/be/api/app/tnvn/approve_member`, `/be/api/app/tnvn/decline_member`).

### 3.2. Điểm danh Sự kiện / Đại hội bằng QR Code
* Sử dụng camera tích hợp `libwechatqrcode.so` để giải mã mã QR sự kiện nhanh chóng dưới điều kiện ánh sáng yếu hoặc mã QR in bị mờ.
* Gửi dữ liệu điểm danh qua API `/be/api/app/event/check-in`, bao gồm tọa độ GPS (plugin `geolocator`), ID sự kiện và mã định danh người dùng nhằm xác nhận sự hiện diện thực tế.

### 3.3. Học tập Lý luận và Thi Trực tuyến (LMS Integration)
* Khi đoàn viên tham gia thi trực tuyến các bài học lý luận chính trị hoặc Nghị quyết:
  * Ứng dụng gọi `/be/api/app/tnvn/getTokenForVivas` để lấy SSO Access Token.
  * Mở WebView bảo mật trỏ đến hệ thống VNPT E-Learning:  
    `https://thanhnienvietnam.vnlms.vn/security/appThanhnien?id_token=...`
* Kết quả và chứng chỉ điện tử được đồng bộ ngược lại ứng dụng qua API `/be/api/app/tnvn/certificates`.

### 3.4. Hệ thống Chat và Giao tiếp Nội bộ (`comm_mobile_client`)
* Tích hợp một phân hệ Chat riêng biệt, sử dụng giao thức WebSocket hai chiều:
  * Máy chủ kết nối: `wss://api-daihoi-tnvn.vivas.vn/ws`
* Hỗ trợ quản lý tin nhắn cá nhân, nhóm chi đoàn, tạo khảo sát/bình chọn (poll) và nhận thông báo thời gian thực qua FCM.

### 3.5. Hệ thống Chống Can thiệp và Đảo ngược Mã (`libanti.so`)
Qua trích xuất chuỗi nhị phân của `libanti.so`, hệ thống nhúng bộ phòng vệ sâu:
* **Anti-Frida:** Giám sát cổng mặc định 27042, quét `/proc/self/maps` tìm chuỗi `frida-agent.so`, kiểm tra ngắt luồng (thread inspection).
* **Anti-Xposed / LSPosed:** Kiểm tra các lớp `de.robv.android.xposed.XposedBridge`, phát hiện can thiệp vào `ClassLoader`.
* **Anti-Emulator:** Quét đường ống ảo hóa (`/dev/qemu_pipe`, `/dev/socket/qemud`), kiểm tra thuộc tính phần cứng liên quan đến x86, VirtualBox, Genymotion, Nox.
* **Anti-DualApp:** Ngăn chặn chạy trong các không gian nhân bản ứng dụng (Parallel Space, Dual App) để chống gian lận điểm danh và thi trực tuyến.

---

## 4. Hệ thống Backend và Danh mục API Endpoints

Hạ tầng backend do VIVAS (VNPT Technology) vận hành tại các dải tên miền:
* **Core App Backend:** `https://tnvn.vivas.vn/be/api/app/` (hoặc `https://tnvn-pro.vivas.vn`)
* **News / Portal:** `https://news-tnvn.vivas.vn` và `https://newstnvn.doanthanhnien.vn`
* **Sự kiện & WebSocket:** `https://api-daihoi-tnvn.vivas.vn` (WSS: `wss://api-daihoi-tnvn.vivas.vn/ws`)
* **Hệ sinh thái Mini App & CMS:** `https://hst.vivas.vn/cms/app/v1`
* **Nền tảng Thi học tập LMS:** `https://thanhnienvietnam.vnlms.vn`

### Bảng Tổng hợp API Endpoints

| Nhóm chức năng | Endpoint | Phương thức | Mô tả chi tiết |
| :--- | :--- | :--- | :--- |
| Auth | `/v2/api/app/login` | POST | Đăng nhập tài khoản và nhận JWT token |
| Auth | `/v2/api/app/otp` | POST | Yêu cầu gửi mã OTP qua SMS |
| Auth | `/v2/api/app/verify` | POST | Xác thực mã OTP |
| Auth | `/v2/api/app/validate_token` | POST | Kiểm tra tính hợp lệ của token |
| Đoàn viên | `/be/api/app/tnvn/check_member_login` | POST | Kiểm tra trạng thái hồ sơ đoàn viên |
| Đoàn viên | `/be/api/app/tnvn/declare_member_info` | POST | Khai báo hồ sơ sổ đoàn viên |
| Đoàn viên | `/be/api/app/tnvn/approve_member` | POST | Cán bộ Đoàn phê duyệt hồ sơ |
| Đoàn viên | `/be/api/app/cates` | GET | Lấy danh mục tổ chức Đoàn các cấp |
| Đoàn viên | `/be/api/cates/children/unions` | GET | Lấy danh sách chi đoàn cấp dưới |
| eKYC | `/be/api/app/tnvn/get_ekyc` | GET | Lấy kết quả xác thực CCCD |
| eKYC | `/be/v2/api/app/tnvn/update_ekyc` | POST | Cập nhật thông tin eKYC từ CCCD |
| Điểm danh | `/be/api/app/event/check-in` | POST | Quét QR kèm GPS điểm danh hội nghị, sự kiện |
| Điểm danh | `/be/api/app/event/check-in/history` | GET | Danh sách lịch sử các sự kiện đã tham gia |
| Tin tức | `/cms/app/v1/post/posts` | GET | Danh sách bài viết, tin tức |
| Tin tức | `/cms/app/v1/post/react_post` | POST | Thả tim, tương tác bài viết |
| Khảo sát | `/cms/app/v1/pollvote/pollvote_detail` | GET | Chi tiết cuộc thăm dò ý kiến |
| Khảo sát | `/cms/app/v1/pollvote/post_pollvote` | POST | Gửi phiếu bình chọn |
| Thi cử | `/be/api/app/tnvn/certificates` | GET | Danh sách chứng chỉ thi trực tuyến |
| Hỏi đáp | `/cms/app/v1/my_question/create_my_question` | POST | Gửi câu hỏi tư vấn chính sách, vay vốn |

---

## 5. Cấu trúc Mã nguồn Phân tích (Dart / Flutter)

File apk: `https://files.catbox.moe/0n9av6.zip`

```text
lib/
├── app/
│   ├── config/
│   │   ├── app_binding.dart
│   │   └── app_theme.dart
│   ├── constants/
│   │   ├── app_constants.dart      # URLs, Base URL, AppType (TNVN, HND...)
│   │   ├── app_colors.dart
│   │   └── api_endpoints.dart
│   └── utils/
│       ├── xor_utils.dart          # Hàm mã hóa và giải mã payload
│       └── date_time_utils.dart
├── core/
│   ├── data/
│   │   ├── local/
│   │   │   ├── app_storage.dart    # SharedPreferences & SecureStorage
│   │   │   └── app_data.dart
│   │   ├── models/
│   │   │   ├── request/
│   │   │   │   ├── access_token_request.dart
│   │   │   │   └── checkin_qr_request.dart
│   │   │   └── response/
│   │   │       ├── access_token_response.dart
│   │   │       └── member_info_response.dart
│   │   └── repositories/
│   │       ├── account_repository.dart
│   │       ├── member_repository.dart
│   │       └── checkin_event_repository.dart
│   └── presentation/
│       ├── modules/
│       │   ├── init_splash/
│       │   ├── login/
│       │   ├── home/
│       │   │   ├── tabs/
│       │   │   │   ├── news/
│       │   │   │   ├── work/       # Nghiệp vụ Đoàn, Sổ đoàn viên
│       │   │   │   ├── chat/       # Giao tiếp nội bộ
│       │   │   │   ├── discover/   # Tiện ích, VNPT Pay
│       │   │   │   └── person/     # Quản lý cá nhân, QR Scanner
│       │   │   └── otp/
│       └── routes/
│           └── app_pages.dart      # Khai báo GetPage và Route names
└── services/
    ├── dio_client.dart             # Cấu hình Dio, Auth Token Interceptor
    └── api_result.dart