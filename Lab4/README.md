# BÁO CÁO THỰC HÀNH LAB 4: KHẢO SÁT VÀ ĐÁNH GIÁ BỀ MẶT MẠNG BẰNG NMAP

## 1. Thông tin sinh viên
* **Họ và tên:** Huỳnh Khánh Phong[cite: 22]
* **MSSV:** 1150070034[cite: 22]
* **Lớp:** 11-ĐH-TMĐT[cite: 22]

## 2. Phiên bản môi trường
* **Hệ điều hành máy quét:** Kali Linux 2026.2 (Nmap version 7.99)[cite: 22].
* **Hệ điều hành máy đích 1:** Metasploitable 2 (Linux)[cite: 22].
* **Hệ điều hành máy đích 2:** Windows 11 Pro (Version 25H2)[cite: 22].
* **Nền tảng ảo hóa:** VMware Workstation[cite: 22].

## 3. Cách dựng môi trường
* **Thiết lập Card mạng:** Tạo và cấu hình mạng VirtualBox/VMware Host-Only (VMnet1) với dải IP `192.168.56.0/24`[cite: 22].
* **Kết nối máy ảo:** Cấu hình Card mạng của cả 3 máy ảo (Kali, Windows 11, Metasploitable 2) sử dụng chung chuẩn Host-Only (VMnet1) để tạo thành một mạng nội bộ khép kín[cite: 22].
* **Danh sách IP cấp phát:**
  * Windows 11 (Máy đích phụ): `192.168.56.129`[cite: 22].
  * Kali Linux (Máy quét): `192.168.56.128`[cite: 22].
  * Metasploitable 2 (Máy đích chính): `192.168.56.130`[cite: 22].

## 4. Các tình huống đã thực hiện
Trong Lab 4, các kỹ thuật dò quét mạng sau đã được thực hiện thành công trên mục tiêu Metasploitable 2:
1. **Khám phá host (Host Discovery):** Sử dụng lệnh `-sn` để quét toàn bộ dải mạng `192.168.56.0/24` tìm các thiết bị đang hoạt động[cite: 22].
2. **Quét cổng TCP cơ bản:** Thực hiện TCP Connect scan (`-sT`) và TCP SYN scan (`-sS`), so sánh thời gian và quyền thực thi[cite: 22].
3. **Quét lén lút (Stealth Scan):** Thực thi các kỹ thuật gửi cờ bất thường gồm FIN scan (`-sF`), Xmas scan (`-sX`), và NULL scan (`-sN`) để quan sát phản hồi `open|filtered`[cite: 22].
4. **Khảo sát tường lửa:** Sử dụng ACK scan (`-sA`) để kiểm tra chính sách lọc mạng[cite: 22].
5. **Quét cổng UDP:** Sử dụng lệnh `-sU --top-ports 20` để dò tìm các dịch vụ UDP phổ biến[cite: 22].
6. **Nhận diện hệ thống:** Dò tìm phiên bản dịch vụ (`-sV`) và dự đoán hệ điều hành (`-O`)[cite: 22].
7. **Quét tổng hợp (Aggressive Scan):** Sử dụng lệnh `-A` để lấy toàn bộ thông tin phiên bản, hệ điều hành, traceroute và chạy script mặc định[cite: 22].
8. **Nmap Scripting Engine (NSE):** Khai thác thông tin SMB bằng script `smb-os-discovery` và kiểm tra lỗi bảo mật bằng script `smb-vuln-ms17-010`[cite: 22].
9. **Xuất báo cáo:** Chuyển đổi và lưu trữ kết quả quét dưới dạng file Text (`-oN`), XML (`-oX`), Grepable (`-oG`) và xuất file HTML bằng `xsltproc`[cite: 22].
10. **Hoàn thiện lý thuyết:** Trả lời chi tiết 10 câu hỏi phân tích sự khác biệt giữa các kỹ thuật quét và cơ chế phòng thủ[cite: 22].

## 5. Kết quả
* **Đánh giá:** **PASS**
* Tất cả các câu lệnh đều được thực thi thành công, kết quả trả về hiển thị chính xác các cổng, dịch vụ, hệ điều hành của máy mục tiêu (Metasploitable 2) và xuất file báo cáo đầy đủ theo đúng yêu cầu[cite: 22].

## 6. Lỗi gặp phải và cách khắc phục
* **Lỗi 1: Lệnh quét bị treo hoặc chạy rất chậm do phân giải DNS.**
  * *Tình trạng:* Khi chạy các lệnh như `-sn`, `-sT` hoặc `-sS`, Nmap liên tục báo trạng thái `Parallel DNS resolution of 2 hosts` và bị kẹt tiến độ ở 0.00%[cite: 22].
  * *Cách khắc phục:* Thêm tham số `-n` vào câu lệnh (ví dụ: `sudo nmap -sn -n 192.168.56.0/24`) để yêu cầu Nmap bỏ qua bước phân giải tên miền ngược, giúp lệnh hoàn thành lập tức[cite: 22].
* **Lỗi 2: Không thể ping từ Kali sang Windows 11 (Mất 100% gói tin).**
  * *Tình trạng:* Kali Linux ping sang Windows 11 trả về kết quả `100% packet loss`, không kết nối được dù cấu hình IP đúng[cite: 22].
  * *Cách khắc phục:* Tắt tính năng Windows Defender Firewall trên máy ảo Windows 11 để cho phép các gói tin ICMP (ping) đi qua[cite: 22].
* **Lỗi 3: Không cài đặt được Zenmap trên Kali Linux.**
  * *Tình trạng:* Khi chạy lệnh `sudo apt install zenmap`, hệ thống báo lỗi `Unable to locate package zenmap` do công cụ này không còn hỗ trợ sẵn trên một số phiên bản Kali mới[cite: 22].
  * *Cách khắc phục:* Bỏ qua Zenmap và sử dụng hoàn toàn giao diện dòng lệnh (CLI) của Nmap để thực thi 100% các yêu cầu của bài Lab[cite: 22].
