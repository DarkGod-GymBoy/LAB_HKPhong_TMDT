# THỰC HÀNH AN TOÀN HỆ THỐNG THÔNG TIN - BÁO CÁO LAB 5

## 1. Thông tin sinh viên
* **Họ tên:** Huỳnh Khánh Phong
* **MSSV:** 1150070034
* **Tên bài Lab:** Lab 5: Thiết lập mô hình tường lửa pfSense.

## 2. Môi trường thực hành & Quá trình chuẩn bị
* **Hệ thống mạng ảo hóa (VMware):** 
  * Cấu hình 3 Network Adapters cho tường lửa pfSense bao gồm: Bridged (WAN), Host-only VMnet2 (LAN - `10.0.0.0/8`), và LAN Segment `dmz-net` (DMZ - `172.16.0.0/16`).
  * Máy tính Host (Máy thật) có IP LAN là `10.0.0.100` và `10.0.0.2`.
* **Cấu hình cơ bản pfSense:**
  * Truy cập menu Console gán IP `10.0.0.1` cho interface LAN.
  * Truy cập WebGUI qua `https://10.0.0.1` (tài khoản mặc định admin/pfsense), hoàn thành Setup Wizard và đổi mật khẩu quản trị.
  * Bật interface OPT1, đổi tên thành DMZ, cấu hình Static IPv4 `172.16.0.1/16`.
  * Chuyển chế độ Outbound NAT sang Hybrid Outbound NAT.
  * Vô hiệu hóa hai rule "Default allow LAN to any" mặc định trên cổng LAN và khởi tạo Rule nền tảng (Pass LAN subnets to Any). Xóa lịch sử kết nối bằng công cụ Reset States.

## 3. Các tình huống cấu hình Tường lửa (Firewall Rules)

### Tình huống 1: Chặn ICMP nhưng vẫn cho phép Web/DNS
* **Thực hiện:** Tạo 3 rule trên tab LAN theo thứ tự: (1) Block ICMP từ LAN subnets đến Any, (2) Pass TCP/UDP cổng 53 (DNS), và (3) Pass TCP cổng 80/443 (HTTP/HTTPS).
* **Kết quả:** Từ máy ảo Kali (`10.0.0.3`), lệnh ping `8.8.8.8` bị chặn (Request timed out). Lệnh `nslookup` (phân giải tên miền) và `curl -4 https://example.com` (truy cập Web) thành công.

### Tình huống 2: Chỉ cho một host cụ thể ra Internet
* **Thực hiện:** Xóa rule cũ, thiết lập rule Pass cho IP `10.0.0.2` (máy thật) lên trên cùng, sau đó đặt rule Block toàn bộ `LAN subnets` ở bên dưới.
* **Kết quả:** Máy thật `10.0.0.2` ping `8.8.8.8` thành công. Máy ảo Kali `10.0.0.3` ping `8.8.8.8` thất bại với tỷ lệ 100% packet loss.

### Tình huống 3: Cô lập vùng DMZ khỏi mạng LAN
* **Thực hiện:** 
  * Cấu hình máy ảo Kali chuyển sang vùng DMZ với IP tĩnh `172.16.0.2/16`, Gateway `172.16.0.1`.
  * Tạo rule trên tab DMZ: Rule Block `DMZ subnets` đi `LAN subnets` đặt phía trên Rule Pass `DMZ subnets` đi Any (Internet).
* **Khắc phục lỗi định tuyến máy Host:** Khi máy thật `10.0.0.2` ping vào DMZ bị timeout do Windows không nhận diện được đường đi. Vấn đề này được khắc phục bằng lệnh CMD quyền Admin: `route add 172.16.0.0 mask 255.255.0.0 10.0.0.1`.
* **Kết quả:** Ping từ máy LAN `10.0.0.2` đến DMZ `172.16.0.2` thành công. Ping ngược từ DMZ `172.16.0.2` vào LAN `10.0.0.2` bị pfSense chặn lại (100% packet loss).

### Tình huống 4: Port Forward WAN → DMZ
* **Thực hiện:** 
  * Khởi động dịch vụ Apache2 trên máy Kali `172.16.0.2` và xác nhận web chạy thành công ở `localhost`.
  * Trong cài đặt WAN của pfSense, bỏ chọn `Block private networks` và `Block bogon networks`.
  * pfSense nhận IP WAN là `192.168.1.23`. Cấu hình NAT Port Forwarding: Interface WAN, TCP, Destination Port `8080`, trỏ về Target IP `172.16.0.2` Port `80` (HTTP).
* **Kết quả:** Từ máy host thật, gõ `http://192.168.1.23:8080` truy cập thành công trang mặc định "Apache2 Debian Default Page" nằm trong vùng DMZ.

### Tình huống 5: Bật logging và đọc Firewall Log
* **Thực hiện:** Tạo lại rule Block ICMP từ LAN đi Any và đánh dấu tick chọn `Log packets that are handled by this rule`. Đóng vai người dùng vi phạm bằng cách dùng CMD máy thật gõ `ping 10.0.0.1`.
* **Kết quả:** Lệnh ping bị chặn. Trong Status > System Logs > Firewall, hệ thống lưu lại bản ghi có dấu X đỏ, hiển thị IP nguồn `10.0.0.2`, đích `10.0.0.1`, giao thức ICMP, kèm theo tên rule chặn là "Test Log Block Ping".

---

## 4. Trả lời câu hỏi ôn tập (Phần D)

**1. Phân biệt vai trò của NAT và firewall rule trong mô hình.**
*   **NAT (Network Address Translation):** Đóng vai trò chuyển đổi địa chỉ IP và Port. Outbound NAT giúp các dải IP nội bộ ẩn mình sau IP public của WAN để ra ngoài Internet. Port Forward (Destination NAT) giúp hướng các luồng truy cập hợp lệ từ Internet vào một máy chủ nội bộ cụ thể.
*   **Firewall rule:** Đóng vai trò là công cụ kiểm soát, quyết định việc cho phép (Pass) hoặc từ chối (Block/Reject) luồng dữ liệu đi ngang qua các cổng mạng dựa trên giao thức, địa chỉ IP, và port.

**2. Vì sao nên tách máy chủ Web/Mail/FTP vào DMZ thay vì đặt trong LAN?**
*   Các máy chủ Web/Mail/FTP cần phải mở cổng giao tiếp ra Internet để người dùng bên ngoài truy cập, do đó chúng có rủi ro bị tấn công và chiếm quyền điều khiển rất cao. Việc đặt chúng ở vùng DMZ cô lập giúp bảo vệ mạng LAN; kể cả khi máy chủ trong DMZ bị tin tặc xâm nhập, chúng vẫn bị tường lửa chặn lại không thể lan truyền tấn công sâu vào các máy trạm và máy chủ dữ liệu trong LAN.

**3. Nếu rule Block nằm dưới một rule Pass tổng quát thì kết quả có thể như thế nào?**
*   Do pfSense quét các quy tắc tường lửa theo thứ tự từ trên xuống dưới và áp dụng nguyên tắc "first match" (khớp lệnh nào thực thi ngay lệnh đó), nếu một rule Block nằm bên dưới một rule Pass bao hàm nó, gói tin đã được rule Pass cho phép đi qua trước khi kịp chạy đến rule Block. Điều này khiến rule Block hoàn toàn vô tác dụng.

**4. Muốn chặn ping nhưng vẫn cho truy cập web, cần cấu hình các rule nào?**
*   Cần cấu hình các rule theo thứ tự ưu tiên từ trên xuống:
    1.  `Block` giao thức `ICMP` từ `LAN net` đi `Any` (Chặn ping).
    2.  `Pass` giao thức `TCP/UDP` port `53` từ `LAN net` đi `Any` (Cho phép phân giải tên miền DNS).
    3.  `Pass` giao thức `TCP` port `80` và `443` từ `LAN net` đi `Any` (Cho phép tải trang Web HTTP và HTTPS).

**5. Logging của firewall giúp ích gì trong xử lý sự cố?**
*   Logging cung cấp lịch sử và bằng chứng lưu lượng mạng bị tường lửa tác động. Giúp quản trị viên xác định nguyên nhân thiết bị không thể kết nối do rule nào gây ra (Troubleshooting), cũng như theo dõi, phát hiện kịp thời các hành vi tấn công, quét cổng bất thường hướng vào hệ thống.

**6. Nêu ít nhất ba biện pháp hardening cho pfSense trong mô hình này.**
*   (1) Thay đổi mật khẩu tài khoản quản trị `admin` mặc định thành mật khẩu mạnh và không chia sẻ.
*   (2) Không cho phép truy cập giao diện quản trị WebGUI từ ngoài vùng WAN; chỉ giới hạn quyền truy cập từ một IP hoặc dải mạng quản trị an toàn trong LAN bằng Anti-Lockout Rule.
*   (3) Tuân thủ chặt chẽ nguyên tắc "Default Deny": Chặn toàn bộ mọi luồng dữ liệu theo mặc định, sau đó mới xét duyệt và tạo rule Pass cho từng dịch vụ cụ thể được cấp phép.
