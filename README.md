#  LAN File Transfer Application & Network Traffic Analysis via Wireshark

Đồ án kết thúc môn **Truyền thông dữ liệu** — Khoa Điện tử - Viễn thông, Trường Đại học Khoa học Tự nhiên (ĐHQG-HCM). 

Dự án triển khai thành công một hệ thống Client-Server chạy trên nền tảng **MATLAB** để truyền tải các tệp tin đa định dạng trong cùng mạng cục bộ (LAN) thông qua giao thức **TCP/IP**. Đồng thời, sử dụng công cụ **Wireshark** để giám sát, bắt và phân tích chuyên sâu cấu trúc gói tin mạng ở Transport Layer và Network Layer.

---

##  Thành viên thực hiện (Nhóm 8 - Lớp 23DTV_CLC1)
* **Trần Thành Thịnh** (Nhóm trưởng) — *Trách nhiệm chính:* Thiết kế kiến trúc, lập trình Socket (MATLAB), cấu hình và phân tích gói tin với Wireshark, tổng hợp báo cáo kỹ thuật.
* **Nguyễn Phước Đăng Minh** — *Trách nhiệm chính:* Biên tập nội dung (PowerPoint + Word) và chạy thử nghiệm kiểm thử Wireshark.
* **Nguyễn Văn Đạt** — *Trách nhiệm chính:* Thuyết trình, hỗ trợ kiểm thử Wireshark và thiết kế giao diện báo cáo PowerPoint.

---

##  1. Thiết kế & Kiến trúc Hệ thống (System Architecture)

Ứng dụng hoạt động dựa trên mô hình **Client-Server** tuần tự nhằm quản lý tập trung và đảm bảo tính dễ mở rộng:

* **Client (Máy gửi):** Đóng vai trò khởi tạo kết nối TCP đến Socket của Server. Tiến hành đọc file dữ liệu gốc, thực hiện phân mảnh và truyền tải luồng byte nhị phân qua đường truyền.
* **Server (Máy nhận):** Lắng nghe kết nối liên tục tại cổng chỉ định. Tiếp nhận luồng byte, bóc tách dữ liệu theo giao thức tự định nghĩa để thu thập thông tin cấu trúc tệp tin và tiến hành ghi/tái tạo lại file hoàn chỉnh xuống ổ cứng.

###  Giao thức Tầng Ứng dụng tự định nghĩa (Custom Application Header)
Để giải quyết bài toán nhận diện ranh giới dữ liệu trong luồng stream liên tục của TCP (tránh hiện tượng dính gói/vỡ gói), nhóm đã thiết kế cấu trúc bản tin gồm 3 thành phần chính:
* **Header (1 byte):** Lưu trữ giá trị số nguyên đại diện cho độ dài ký tự của tên file.
* **Filename (n bytes):** Chuỗi ký tự định danh tên tệp tin được đặt ngay sau Header.
* **Payload:** Toàn bộ dữ liệu nhị phân thô (Raw binary) của tệp tin cần truyền tải.

---

##  2. Triển khai Mã nguồn cốt lõi (Core Implementation)

### Phía Client (Gửi dữ liệu)
```matlab
% 1. Khởi tạo kết nối TCP đến Server và thực hiện 3-way handshake
IP_SERVER = '172.16.0.56'; 
PORT = 5001;
t = tcpclient(IP_SERVER, PORT); 

% 2. Gửi Header chứa thông tin cấu trúc file (Giao thức tự định nghĩa)
write(t, uint8(length(filename))); % Byte 1: Gửi độ dài tên file
write(t, uint8(filename));         % Các Byte tiếp theo: Gửi chuỗi tên file

% 3. Thuật toán băm nhỏ dữ liệu (Chunking) để truyền Payload tránh nghẽn buffer
chunkSize = 5242880; % Giới hạn kích thước mỗi chunk là 5MB
for i = 1:chunkSize:totalBytes
    endIdx = min(i + chunkSize - 1, totalBytes);
    write(t, FileData(i:endIdx)); % Bơm luồng byte nhị phân vào đường truyền
end
```
### Phía Client (Gửi dữ liệu)
```matlab
% 1. Khởi tạo Server lắng nghe kết nối tại Port chỉ định
server = tcpserver("0.0.0.0", 5001);

% 2. Bóc tách Header của giao thức tự định nghĩa
while server.NumBytesAvailable < 1, pause(0.1); end
name_length = read(server, 1, "uint8"); % Đọc 1 Byte đầu lấy độ dài tên file

while server.NumBytesAvailable < name_length, pause(0.1); end
received_filename = char(read(server, name_length, "uint8")); % Đọc chuỗi ký tự tên file

% 3. Nhận luồng dữ liệu Payload và ghi tuần tự xuống ổ cứng
fid = fopen(['copy_of_', received_filename], 'w');
while true
    if server.NumBytesAvailable > 0
        data = read(server, server.NumBytesAvailable, "uint8"); % Đọc bộ đệm mạng
        fwrite(fid, data, 'uint8'); % Ghi trực tiếp luồng byte nhị phân ra file
    end
    
    % Điều kiện thoát: Client đã ngắt kết nối VÀ bộ đệm mạng đã được đọc sạch
    if ~server.Connected && server.NumBytesAvailable == 0
        break;
    end
end
fclose(fid);
```
---
## 3. Giám sát & Phân tích Gói tin với Wireshark
Hệ thống sử dụng phần mềm Wireshark để trực quan hóa hoạt động thực tế của giao thức TCP, đối chiếu trực tiếp giữa lý thuyết mạng và thực nghiệm:

### Quá trình Thiết lập Kết nối (TCP 3-Way Handshake)
Wireshark bắt được trọn vẹn tiến trình bắt tay 3 bước diễn ra nghiêm ngặt:

* **Gói tin SYN được Client phát đi để đề nghị thiết lập kết nối.**

* **Gói tin SYN-ACK phản hồi từ Server nhằm chấp nhận kết nối.**

* **Gói tin ACK cuối cùng từ Client gửi sang để xác nhận hoàn tất kết nối.**
<img width="2106" height="875" alt="z7853135842318_ae7bd71530d76d00b558b2e6fce3d116" src="https://github.com/user-attachments/assets/b0e71643-6e0e-4274-b4a9-6896cf0451d9" />

### Quá trình Truyền dữ liệu & Cơ chế Tự sửa lỗi (Error Control)
* **Trong điều kiện mạng lý tưởng, chỉ số Sequence number (Số thứ tự gói) tăng tuần tự và các gói ACK (Xác nhận) phản hồi chính xác giúp đảm bảo dữ liệu không bị sai thứ tự.**

* **Khi thực hiện kiểm thử trong môi trường thực tế có biến động vật lý, Wireshark ghi nhận các gói tin đặc biệt như TCP Dup ACK và TCP Retransmission (Truyền lại). Đây là minh chứng rõ ràng cho cơ chế kiểm soát lỗi ưu việt của TCP, chủ động phát hiện mất gói và truyền lại tự động nhằm bảo vệ tính toàn vẹn tuyệt đối của dữ liệu tại điểm đích.**
<img width="2105" height="877" alt="z7853137894084_57e557b966fdb4e3915c480899f3db0c" src="https://github.com/user-attachments/assets/3c6a98b5-22e2-4771-aa22-87ea815bd507" />

---

## 4. Kết quả & Đánh giá Thử nghiệm

###  Kết quả đạt được

| Tiêu chí đánh giá | Kết quả thực tế | Giải thích kỹ thuật |
| :--- | :--- | :--- |
| **Tỷ lệ truyền tải** | **Đạt 100%** trong mạng LAN | Kết nối qua Switch/Router ổn định, dữ liệu trùng khớp hoàn toàn, không mất mát hay sai lệch so với file gốc. |
| **Độ đa dạng định dạng** | **Hỗ trợ tuyệt đối** | Nhờ cơ chế băm byte nhị phân tại Application Layer, hệ thống xử lý mượt mà từ file văn bản (.txt, .doc), file đa phương tiện (.jpg, .ppt) cho đến mã nguồn (.m, .py). |


### Đánh giá Ưu điểm & Hạn chế
* **Ưu điểm: Độ tin cậy cực kỳ cao nhờ tận dụng tối đa lợi thế của TCP (chống mất gói, sắp xếp đúng thứ tự). Cấu trúc mã nguồn tinh gọn, trực quan và dễ dàng triển khai trong môi trường LAN.**

* **Hạn chế kỹ thuật: Chịu ảnh hưởng bởi năng lực xử lý đơn luồng (Single-thread), Server chỉ phục vụ tuần tự 1-1 nên dễ gây nghẽn khi có nhiều yêu cầu đồng thời. Dữ liệu truyền tải ở dạng bản rõ (Plain text), chưa áp dụng các tiêu chuẩn mã hóa bảo mật mạng (SSL/TLS) và thiếu giao diện đồ họa (GUI) cho người dùng cuối.**
---
## 5. Hướng phát triển Đề xuất
*  **Nâng cấp kiến trúc đa nhiệm:** Cải tiến Server sang mô hình **Đa luồng (Multi-threaded Server)** hoặc xử lý bất đồng bộ (Asynchronous), tích hợp cấu trúc hàng đợi (Queue) để tiếp nhận và xử lý nhiều Client đồng thời.
*  **Tối ưu hóa băng thông:** Tích hợp các thuật toán nén dữ liệu hiệu năng cao (như GZIP hoặc LZ4) trực tiếp tại tầng ứng dụng trước khi đẩy dữ liệu vào TCP Socket nhằm giảm dung lượng tải.
*  **Tăng cường bảo mật bảo mật:** Triển khai lớp mã hóa bảo mật đầu cuối, mã hóa toàn bộ dữ liệu Payload bằng thuật toán đối xứng **AES** hoặc đóng gói qua tiêu chuẩn bảo mật **SSL/TLS**.
* **Hoàn thiện trải nghiệm người dùng:** Thiết kế giao diện đồ họa (GUI) thân thiện bằng MATLAB App Designer, bổ sung tính năng tự động kết nối lại (Retry) và tính năng **Resume** (truyền tiếp khi mạng bị gián đoạn giữa chừng).

