# THỰC HÀNH AN TOÀN HỆ THỐNG THÔNG TIN - BÁO CÁO LAB 6

## 1. Thông tin sinh viên
- **Họ tên:** Huỳnh Khánh Phong
- **MSSV:** 1150070034
- **Tên bài Lab:** Lab 6: Xác thực Active Directory và Chính sách truy cập trên pfSense.

## 2. Môi trường thực hành & Quá trình chuẩn bị
- **Môi trường ảo hóa:** VMware Workstation Pro.
- **Danh sách các máy ảo thành phần:**
  1. **HKPhong_FreeBSD (pfSense 2.7.x):** Máy ảo chạy hệ điều hành FreeBSD / pfSense đóng vai trò làm Router & Firewall chính. Cấu hình 4 card mạng:
     - **Adapter 1 (WAN):** NAT (`192.168.250.0/24`) - Kết nối Internet.
     - **Adapter 2 (MGMT):** Host-Only (`192.168.56.0/24`) - Quản trị hệ thống an toàn.
     - **Adapter 3 (LAN):** LAN Segment `lab6-lan` (`10.6.0.0/24`) - Mạng nội bộ.
     - **Adapter 4 (DMZ):** LAN Segment `lab6-dmz` (`172.16.6.0/24`) - Vùng chứa máy chủ dịch vụ.
  2. **W11_HKPhong (Windows 11):** Máy trạm Client trong mạng LAN (`10.6.0.21/24` hoặc `10.6.0.22/24`), kết nối vào LAN Segment `lab6-lan`, Preferred DNS trỏ về máy chủ Domain Controller `10.6.0.10`.
  3. **KALI_HKPhong (Kali Linux):** Máy trạm thử nghiệm an ninh mạng, sử dụng để thực hiện kiểm thử kết nối, truy vấn DNS, gửi gói tin và phân tích dịch vụ.
  4. **Metasploitable2-Linux (Linux):** Máy chủ Web Server đặt trong vùng DMZ, gán LAN Segment `lab6-dmz`, địa chỉ IP chính `172.16.6.10/24` và gán thêm các IP phụ (IP Aliases) `.20`, `.30`, `.40` trên Nginx/Apache2 để mô phỏng các trang web khác nhau (`JOB_SITE`, `SOCIAL_SITE`, `RESTRICTED_SITE`).

- **Cấu hình cơ bản & Tích hợp Active Directory:**
  * Thiết lập IP các interface trên pfSense: MGMT `192.168.56.254`, LAN `10.6.0.1`, DMZ `172.16.6.1`.
  * Đăng nhập WebGUI qua máy quản trị (`https://192.168.56.254`).
  * Tích hợp LDAP Server trong **System > User Manager > Authentication Servers**: Kết nối tới Active Directory Domain Controller (`10.6.0.10:389`), Base DN `DC=lab6,DC=test`, Bind qua tài khoản dịch vụ `svc_pfsense_ldap`. Kiểm tra xác thực thành công các tài khoản domain bằng công cụ **Diagnostics > Authentication**.
  * Bật Captive Portal trên interface LAN sử dụng authentication backend là `LAB6-AD`. Kiểm thử đăng nhập thành công từ trình duyệt máy client `W11_HKPhong`.
  * Thiết lập các Alias trong **Firewall > Aliases**: `CLIENT_A` (`10.6.0.21`), `CLIENT_B` (`10.6.0.22`), `POLICY_CLIENTS` (`10.6.0.21, 10.6.0.22`), `GENERAL_SITE` (`172.16.6.10`), `JOB_SITE` (`172.16.6.20`), `SOCIAL_SITE` (`172.16.6.30`), `RESTRICTED_SITE` (`172.16.6.40`), và `WEB_PORTS` (80, 443).
  * Khởi tạo Schedule trong **Firewall > Schedules**: `LUNCH_WINDOW` (`11:30 - 13:00`), `WORK_HOURS` (`08:00 - 17:00`).

## 3. Các kịch bản thực hành chính sách (Firewall Policy Scenarios)

### Kịch bản 1: Client-A bị chặn JOB_SITE
- **Thực hiện:** Tạo rule trên tab LAN theo thứ tự: (1) Rule Block TCP/80 từ `CLIENT_A` đến `JOB_SITE` (`172.16.6.20`), có bật logging; (2) Rule Pass TCP/80 từ `CLIENT_A` đến `GENERAL_SITE` (`172.16.6.10`).
- **Kết quả:** Từ máy `W11_HKPhong` (Client-A), truy cập `http://172.16.6.10` thành công hiển thị trang web. Truy cập `http://172.16.6.20` bị chặn hoàn toàn (timeout).

### Kịch bản 2: Cấm RESTRICTED_SITE cho cả Client-A và Client-B
- **Thực hiện:** Tạo rule Block TCP/80 từ Alias nhóm `POLICY_CLIENTS` đến `RESTRICTED_SITE` (`172.16.6.40`), bật log chi tiết.
- **Kết quả:** Các truy cập từ `W11_HKPhong` và `KALI_HKPhong` khi truy cập vào `http://172.16.6.40` đều bị tường lửa chặn lại. Các truy cập tới `GENERAL_SITE` (`172.16.6.10`) vẫn diễn ra bình thường.

### Kịch bản 3: Client-B chỉ được truy cập SOCIAL_SITE trong giờ nghỉ
- **Thực hiện:** Đặt rule Pass TCP/80 từ `CLIENT_B` đến `SOCIAL_SITE` (`172.16.6.30`) gán Schedule `LUNCH_WINDOW` nằm TRÊN rule Block TCP/80 từ `CLIENT_B` đến `SOCIAL_SITE` (Schedule: none/always).
- **Kết quả:** Trong khung giờ nghỉ trưa (`11:30 - 13:00`), truy cập `http://172.16.6.30` thành công. Ngoài khung giờ này, quy tắc Pass bị vô hiệu hóa, quy tắc Block phía dưới bắt kịp lưu lượng và chặn kết nối.

### Kịch bản 4: Chỉ cho phép HTTP/HTTPS trong giờ hành chính
- **Thực hiện:** Tạo rule Pass TCP cổng `WEB_PORTS` (80, 443) từ `POLICY_CLIENTS` đi Any với Schedule `WORK_HOURS` (`08:00 - 17:00`). Đặt một rule Block toàn bộ lưu lượng `POLICY_CLIENTS` đi Any ở phía dưới.
- **Khắc phục sự cố kết nối:** Đảm bảo máy client `W11_HKPhong` cấu hình Preferred DNS trỏ đúng về `10.6.0.10` thay vì `10.6.0.1` để tránh bị rơi vào rule Default Deny đối với lưu lượng UDP/53. Mở rộng lịch trình `WORK_HOURS` bao gồm ngày kiểm thử thực tế và thực hiện **Reset States** trên pfSense.
- **Kết quả:** Trong giờ hành chính, truy cập Web HTTP/HTTPS (`http://172.16.6.10`) thành công. Lệnh `ping 8.8.8.8` (ICMP) bị chặn hoàn toàn theo đúng thiết kế của policy.

### Kịch bản 5: Cô lập DMZ khỏi LAN nhưng cho phép LAN truy cập DMZ
- **Thực hiện:** Tạo rule trên tab DMZ (OPT2): Rule Block từ `DMZ net` (`OPT2 subnets`) đến `LAN net` (`10.6.0.0/24`), bật log. Chiều ngược lại trên tab LAN duy trì các rule Pass cho phép client truy cập web DMZ.
- **Kết quả:** Từ các máy LAN (`W11_HKPhong`) truy cập vào dịch vụ web `172.16.6.10` thành công. Từ máy chủ `Metasploitable2-Linux` (DMZ) ping hoặc khởi tạo kết nối về các máy trong LAN (`10.6.0.10`, `10.6.0.21`) bị tường lửa chặn lại hoàn toàn (100% packet loss).

### Kịch bản 6 & 7: So sánh Block/Reject, Đọc Firewall Log và State Table
- **Thực hiện:** Đổi thử nghiệm hành động của một rule chặn từ `Block` sang `Reject`. Đọc và lọc nhật ký trong **Status > System Logs > Firewall** và kiểm tra các phiên kết nối trong **Diagnostics > States**.
- **Kết quả:** Khi dùng `Reject`, client nhận phản hồi từ chối tức thì (TCP RST). Khi dùng `Block`, client phải chờ hết khoảng thời gian timeout (`ERR_CONNECTION_TIMED_OUT`). Hệ thống log ghi nhận chính xác các bản ghi chặn/cho phép kèm theo IP nguồn, IP đích, cổng dịch vụ và biểu tượng trực quan.

---

## 4. Trả lời câu hỏi phân tích (Phần J)

**1. Vì sao LAB 6 phải dùng topology riêng thay vì tiếp tục sử dụng trực tiếp topology LAB 5?**
* LAB 6 mở rộng bài thực hành với việc tích hợp Active Directory (LDAP), Captive Portal và kiểm soát phân vùng DMZ theo địa chỉ IP/Group mô phỏng nâng cao. Việc dùng topology mới giúp phân tách rõ ràng 4 vùng mạng (WAN, MGMT, LAN, DMZ), đảm bảo môi trường thử nghiệm sạch sẽ, không bị ảnh hưởng bởi các rule NAT hay cài đặt cũ của LAB 5.

**2. LDAP được dùng cho chức năng nào trên pfSense? Firewall rule chuẩn có thể dùng username AD làm Source trực tiếp không?**
* LDAP được dùng làm Authentication Backend để pfSense xác thực thông tin người dùng Active Directory trong Captive Portal hoặc WebGUI. Firewall rule chuẩn của pfSense hoạt động ở Tầng 3/Tầng 4 (IP/Port/Schedule), **không thể** sử dụng trực tiếp AD username làm điều kiện Source trong rule mà phải ánh xạ người dùng thông qua địa chỉ IP tĩnh (hoặc IP do Captive Portal / DHCP gán).

**3. Vì sao Captive Portal được đặt trên LAN nhưng WebGUI quản trị lại tách sang MGMT?**
* Việc tách giao diện WebGUI quản trị sang mạng riêng MGMT giúp đảm bảo nguyên tắc an toàn thông tin: người dùng thông thường trong mạng LAN khi chưa/đã xác thực Captive Portal đều không thể tiếp cận cổng quản trị của tường lửa, ngăn ngừa rủi ro bị tấn công dò quét mật khẩu hoặc khai thác lỗ hổng WebGUI.

**4. Search Scope Entire Subtree có ý nghĩa gì khi pfSense truy vấn Active Directory?**
* Cấu hình `Entire Subtree` cho phép pfSense tìm kiếm đối tượng người dùng/nhóm đệ quy trên toàn bộ cây thư mục Active Directory (bao gồm tất cả các Organizational Units - OU và Container con nằm bên dưới Base DN) thay vì chỉ tìm ở cấp thư mục gốc.

**5. Tại sao nên dùng service account thay vì Domain Administrator làm LDAP bind?**
* Sử dụng service account (`svc_pfsense_ldap`) tuân thủ nguyên tắc đặc quyền tối thiểu (Least Privilege). Tài khoản Bind LDAP chỉ cần quyền đọc (Read) thông tin directory. Nếu dùng tài khoản Domain Administrator, trong trường hợp pfSense bị chiếm quyền điều khiển, thông tin đăng nhập tối cao của Active Directory sẽ bị lộ, dẫn đến nguy cơ mất an toàn cho toàn bộ hệ thống Domain.

**6. Trong Kịch bản 3, vì sao cần một rule Block SOCIAL_SITE nằm dưới rule Pass có Schedule?**
* Trong pfSense, khi một Schedule hết hiệu lực (Inactive), rule Pass gắn Schedule đó sẽ bị coi như không tồn tại. Để đảm bảo lượng lưu lượng bị chặn ngoài khung giờ chỉ định, cần có một rule Block phủ định nằm ngay bên dưới để bắt lại các gói tin này. Nếu không có rule Block bên dưới, gói tin sẽ tiếp tục đi xuống và có thể khớp với một rule Default Allow rộng hơn phía dưới.

**7. Vì sao Client-A/B dùng DNS 10.6.0.10 nhưng rule Internet của client không cần cho DNS tới 10.6.0.10?**
* Do các máy Client (`10.6.0.21`, `.22`) và máy chủ DNS Domain Controller (`10.6.0.10`) cùng nằm trong cùng một dải mạng con LAN (`10.6.0.0/24`). Lưu lượng truy vấn DNS giữa Client và DNS Server được chuyển mạch trực tiếp qua LAN Switch ở Tầng 2, không cần đi qua Gateway pfSense nên không chịu sự kiểm soát của LAN Firewall Rules.

**8. Giải thích first-match bằng một ví dụ cụ thể trong ruleset tổng hợp.**
* Cơ chế First-Match có nghĩa là pfSense quét quy tắc từ trên xuống dưới, gói tin khớp quy tắc nào đầu tiên sẽ lập tức thực thi hành động (Pass/Block) của quy tắc đó và dừng kiểm tra các rule còn lại. 
  * *Ví dụ:* Trong bảng tổng hợp, Rule 1 chặn `CLIENT_A` đi `JOB_SITE`. Khi `Client-A` truy cập `172.16.6.20`, nó khớp ngay Rule 1 và bị Block. Dù Rule 5 bên dưới cho phép `POLICY_CLIENTS` đi `GENERAL_SITE` hoặc Rule 6 cho phép đi Web, gói tin đã bị dừng ngay tại Rule 1.

**9. Vì sao sau khi đổi rule cần tạo kết nối mới hoặc Reset States?**
* pfSense là tường lửa theo dõi trạng thái (Stateful Firewall). Khi một kết nối được chấp nhận hoặc bị chặn, thông tin phiên sẽ được lưu vào State Table. Nếu quản trị viên thay đổi quy tắc, các kết nối đang tồn tại vẫn tiếp tục sử dụng thông tin trong State Table cũ mà không bị áp quy tắc mới. Việc tạo kết nối mới hoặc thực hiện **Reset States** giúp xóa bỏ các phiên cũ, buộc tường lửa đánh giá lại luồng dữ liệu theo đúng ruleset vừa cập nhật.

**10. Mô hình JOB_SITE/SOCIAL_SITE/RESTRICTED_SITE trong DMZ khác Web Category Filtering thật như thế nào?**
* Mô hình trong Lab sử dụng các địa chỉ IP phụ tĩnh (`172.16.6.20`, `.30`, `.40`) để đại diện cho các nhóm website ở Layer 3/Layer 4. Trong khi đó, hệ thống lọc Web Category thực tế (L7 / Web Filter / UTM) phân loại website dựa trên tên miền FQDN, đường dẫn URL, chứng thư SSL/TLS SNI, và cơ sở dữ liệu danh mục được cập nhật liên tục từ mây bảo mật.

**11. Block và Reject khác nhau ra sao về hành vi phía client?**
* **Block:** Tường lửa âm thầm thả (drop) gói tin mà không gửi bất kỳ phản hồi nào về. Client phải chờ cho đến khi hết thời gian chờ kết nối (Timeout), dẫn đến trình duyệt báo lỗi `ERR_CONNECTION_TIMED_OUT`.
* **Reject:** Tường lửa lập tức phản hồi một gói tin từ chối (gói TCP RST đối với TCP hoặc ICMP Port Unreachable đối với UDP). Client nhận được phản hồi tức thì và hiển thị lỗi kết nối ngay lập tức mà không phải chờ đợi.

**12. Nếu một rule Block có log nhưng Firewall Log không xuất hiện bản ghi, cần kiểm tra theo thứ tự nào?**
* Thứ tự kiểm tra troubleshooting:
  1. Kiểm tra xem quy tắc đó đã được tích chọn ô **Log packets handled by this rule** hay chưa.
  2. Kiểm tra vị trí Rule: Xem gói tin có bị một rule Pass/Block khác nằm phía trên khớp trước hay không (First-match rule).
  3. Kiểm tra bộ nhớ trạng thái: Xem kết nối có đang dùng lại một State cũ trong State Table hay không (cần Reset States).
  4. Kiểm tra thông số cấu hình của Rule: Đảm bảo đúng Interface, Protocol, Source IP/Alias, Destination IP/Alias, Port.
  5. Kiểm tra bộ lọc (Filter) trong giao diện Firewall Log xem có đang đặt điều kiện lọc sai làm ẩn bản ghi hay không.
