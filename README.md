# THỰC HÀNH AN TOÀN THÔNG TIN - BÁO CÁO LAB 3

## 1. Thông tin sinh viên
* **Họ tên:** Huỳnh Khánh Phong
* **MSSV:** 1150070034
* **Lớp:** 11-ĐH-TMĐT
* **Tên bài Lab:** Lab 3: Nhận diện và ứng phó các mối đe dọa đến an toàn thông tin.

## 2. Phiên bản môi trường thực hành
* **Phần mềm ảo hóa:** VMware Workstation Pro 26H1
* **Hệ điều hành máy ảo:** Windows 11 25H2 x64, OS build 26200.9445 (KB5124008)
* **Cấu hình mạng:** Host-only
* **Các công cụ đã cài đặt:** 
  * Microsoft Defender Antivirus (Bật Real-time protection & Tamper Protection)
  * Python 3.14.7
  * Wireshark 4.6.8 (Kèm Npcap)
  * Sysmon 15.22, Autoruns 14.3, Process Explorer 17.14

## 3. Cách dựng môi trường
1. Tạo máy ảo Windows 11 trên VMware với cấu hình mạng Host-only.
2. Cập nhật hệ điều hành lên bản vá bảo mật KB5124008 và tạo snapshot `LAB3_CLEAN_20260914`.
3. Dùng PowerShell (Admin) tạo cấu trúc thư mục tại `C:\LAB3` (Evidence, Tools, Downloads, Assets).
4. Tải file `LAB3_Threats_Assets.zip`, kiểm tra mã băm SHA-256 và giải nén.
5. Cài đặt Python, Wireshark thông qua `winget`.
6. Tải và cấu hình bộ công cụ Sysinternals (Sysmon, Autoruns, Process Explorer) từ máy chủ Microsoft.
7. Chạy script PowerShell thu thập trạng thái cơ sở (Baseline) của hệ điều hành, Defender, Firewall và tiến trình, lưu log vào thư mục Evidence.

## 4. Các tình huống đã thực hiện và Kết quả
| ID | Tình huống | Kết quả |
| :--- | :--- | :--- |
| **TH1** | Xác định tài sản, lỗ hổng, mối đe dọa, rủi ro (Risk Register) | **PASS** |
| **TH2** | Mã độc: kiểm chứng chu trình phát hiện bằng EICAR | **PASS** |
| **TH3** | Tấn công mật khẩu và nguy cơ keylogging | **PASS** |
| **TH4** | Backdoor: nhận diện persistence và dịch vụ lắng nghe | **PASS** |
| **TH5** | Sniffing, MITM và Spoofing: quan sát HTTP so với HTTPS | **FALL** |
| **TH6** | DoS, DDoS và Mail Bombing | **FALL** |
| **TH7** | Social Engineering, Phishing và Spear Phishing | **FALL** |

## 5. Lỗi gặp phải và Cách khắc phục
Trong quá trình thực hành, tôi đã gặp một số lỗi sau và đã tự khắc phục thành công:

* **Lỗi 1 (Quá trình tạo cấu trúc thư mục):** Bị lỗi `Unexpected token 'NEW-ITEM' in expression or statement` do copy/paste gộp nhiều dòng lệnh vào PowerShell mà không có dấu xuống dòng.
  * **Cách khắc phục:** Xóa lệnh cũ, copy và chạy độc lập từng dòng lệnh, nhấn Enter sau mỗi dòng để PowerShell thực thi chính xác.
* **Lỗi 2 (Kiểm tra phiên bản Wireshark/Python):** Lỗi chữ đỏ `The term 'c:\program files\Wireshark\tshark.exe' is not recognized` sau khi vừa chạy lệnh cài đặt xong.
  * **Cách khắc phục:** Khởi động lại PowerShell dưới quyền Administrator để hệ thống cập nhật lại biến môi trường (PATH), sau đó chạy lại lệnh kiểm tra phiên bản.
* **Lỗi 3 (Tạo Backdoor Persistence ở Tình huống 4):** Sử dụng sai lệnh hướng dẫn, tạo nhầm Registry Key chạy `powershell.exe` mở port 4444 thay vì ứng dụng vô hại `notepad.exe`.
  * **Cách khắc phục:** Dùng lệnh `Remove-ItemProperty` xóa bỏ Registry Key bị sai trong PowerShell, khởi động lại máy. Sau đó nhập chính xác đoạn script yêu cầu tạo `notepad.exe` và `Scheduled Task`, kiểm tra lại bằng Autoruns thấy đúng Image Path là `notepad.exe`.
