# Using lsof for tracing and analyzing performance on a Linux system

<details>
<summary>**1. `lsof` là gì?** </summary>

**`lsof`** (Liệt kê các tệp đang mở - **L**i**s**t **O**pen **F**iles) là công cụ dòng lệnh trên Linux để hiển thị **tất cả tệp, socket, pipe, hoặc thiết bị** đang được mở bởi các tiến trình, lsof rất hữu ích cho việc:

- Gỡ lỗi ứng dụng.
- Kiểm tra cổng mạng.
- Phát hiện file bị khóa, v.v.

Vì `lsof` làm việc chặt chẽ với **kernel**, nên cần **phải được cấu hình đúng với hiệu điều hành muốn sử dụng**. Mọi thao tác trong Linux (đọc/ghi tệp, kết nối mạng, thậm chí thư mục) đều được xem là "tệp mở" và **`lsof`** giúp theo dõi chúng.

</details>

## **2. Các tùy chọn phổ biến của `lsof`**

| **Tùy chọn** | **Ý nghĩa** |
| --- | --- |
| **`-i`** | Liệt kê các kết nối mạng (socket) đang mở (VD: **`lsof -i :80`**). |
| **`-p`** | Hiển thị tệp mở bởi tiến trình có PID chỉ định (VD: **`lsof -p 1234`**). |
| **`-u`** | Liệt kê tệp mở bởi người dùng cụ thể (VD **`lsof -u root`**). |
| **`-c`** | Lọc các tiến trình theo tên command (process name) (VD: **`lsof -c firefox)`**. |
| **`+D`** | Hiển thị tệp mở trong thư mục cụ thể (VD: **`lsof +D /var/log`**). |
| **`-t`** | Chỉ xuất PID của tiến trình. |
| **`-n`** | Không phân giải hostname. |
| **`-P`** | Không phân giải service name. |

## 3. Cách cài đặt **`lsof`**

1. **Cài đặt từ trình quản lý gói (package manager)**
    
    
    | Hệ điều hành | Lệnh cài |
    | --- | --- |
    | Debian / Ubuntu | `sudo apt install lsof` |
    | RHEL / CentOS | `sudo yum install lsof` |
    | Arch Linux | `sudo pacman -Syu lsof` |
    | NixOS | `nix-env -i lsof` |
2. **Tự build `lsof` từ mã nguồn** – nếu bản cài đặt sẵn bị lỗi hoặc thiếu tính năng
    - **Legacy build system.**
    - **Autotools-based build system.**

## 4. Triển khai/thực hiện: Build from source (Legacy Build System trên Ubuntu)

Bước 1: Cài đặt các gói cần thiết

```bash
sudo apt update
sudo apt install gcc make wget tar
```

![image.png](image.png)

Bước 2: Tải mã nguồn `lsof`

```bash
mkdir ~/lsof-build
cd ~/lsof-build

wget https://github.com/lsof-org/lsof/archive/refs/tags/4.99.4.tar.gz 

tar -xzf 4.99.4.tar.gz
cd lsof-4.99.4
```

![image.png](image%201.png)

Bước 3. Biên dịch, cài đặt và kiểm tra phiên bản

```bash
./Configure linux
make
sudo make install
```

![image.png](image%202.png)

```bash
./lsof -v

#hoặc

sudo cp ./lsof /usr/bin/
sudo chmod 755 /usr/bin/lsof
lsof -v
```

![image.png](image%203.png)

## **5. Sử dụng `lsof` để truy vết & phân tích hiệu năng Ubuntu - Linux**

Tóm tắt ý nghĩa các cột kết quả khi thực hiện các ví dụ thực tế:

| **Cột** | **Ý nghĩa** |
| --- | --- |
| **`COMMAND`** | Tên tiến trình đang mở file. |
| **`PID`** | Process ID của tiến trình. |
| **`USER`** | User sở hữu tiến trình. |
| **`FD`** | File descriptor: số FD, tiến trình đang làm gì. |
| **`TYPE`** | Loại tài nguyên mà FD đang tham chiếu. |
| **`DEVICE`** | Major/minor number của thiết bị chứa file (ổ đĩa). |
| **`SIZE/OFF`** | Kích thước file hoặc offset (0 nghĩa là file trống). |
| **`NODE`** | Inode của file trên filesystem. |
| **`NAME`** | Đường dẫn tuyệt đối của file. |
1. Trường hợp 1: phân tích file đang được mở/ghi
    - Tạo file và mở
    
    ```bash
    touch /tmp/test.log #tạo file trống 
    tail -f /tmp/test.log &  #tail: theo dõi thời gian thực, &: chạy ngầm
    ```
    
    - Dùng lsof tìm process đang giữ file
    
    ```bash
    sudo lsof /tmp/test.log
    ```
    
    ![image.png](e6823036-ab9c-40ea-ae1e-d697fcdf1841.png)
    
    Ta thấy:
    
    - COMMAND: tiến trình tail đang mở file test.log.
    - PID: mã định danh tiến trình là 22519.
    - USER: truongvo đang thực thi.
    - FD: đại diện cho tài nguyên mở (file, socket, device) mà process có thể truy cập. Mỗi FD được kernel quản lý và ánh xạ một tài nguyên cụ thể. File Descriptor thông thường số thứ tự + quyền truy cập hoặc FD đặc biệt (ví dụ như cwd, txt, mem,… Trong ví dụ FD có số thứ tự là **`3`**, **`r`** (read-only) - tiến trình đang đọc file. Theo thiết kế Unix (1970s) thì mọi tiến trình cần 3 luồng I/O cơ bản để process có thể giao tiếp với user (terminal):
        - 0u (stdin): nhận input (bàn phím,…), u = read + write.
        - 1u (stdout): xuất output (màn hình,….).
        - 2u (stderr): xuất thông báo lỗi.
        
        Do đó bắt đầu từ 3 trở đi mới dành cho tài nguyên do người dùng mở. Ví dụ như hình bên dưới (sudo lsof -p $$ là để xem PID của shell terminal hiện tại), một process sẽ luôn có 3 FD cơ bản và những FD mà process đó cần sử dụng:
        
        - cwd: thư mục làm việc hiện tại /home/truongvo
        - rtd: thư mục gốc là /
        - txt: file đang thực thi từ /usr/bin/bash
        - mem: file được nạp vào các bộ nhớ
        
        ![image.png](55e9784e-9524-4d80-acbc-b2f98db7b9ef.png)
        
    - TYPE: REG là regular file thường. Ngoài ra còn có các loại như:
        - DIR: biết process đang ở thư mục nào.
        - CHR: biết process đang dùng terminal.
        - IPv4/IPv6: biết process đang nghe port gì.
        - …
    - DEVICE: 8, 2 là Major/Minor của thiết bị chứa file đó, số driver ổ cứng là 8 và phân vùng thứ 2 trên ổ cứng.
    - SIZE/OFF: nếu TYPE là REG thì hiển thị kích thước file, còn TYPE là loại khác thì hiển thị Offset - vị trí đang đọc/ghi file. Trên ví dụ thì size là 0 vì file .log trống.
    - NODE: Inode number duy nhất cho mỗi file, 524361 đại diện cho ‘địa chỉ” của metadata (thông tin mô tả của file như là: quyền truy cập, chủ sở hữu, kích thước file, thời gian tạo/sửa đổi,…):
    
    ![image.png](2dab4938-5e41-44bb-a5ea-bd422e207f65.png)
    
    Có thể sử dụng Inode để tìm khi không nhớ tên file:
    
    ![image.png](6de13eda-75e9-4ac9-9627-3717c12f0f27.png)
    
    - NAME: đường dẫn tuyệt đối của file là **`/tmp/test.log`**
2. Trường hợp 2: file đã bị gỡ nhưng vẫn đang được mở

Tình huống thông thường khi tạo rồi xóa 1 file log thì file hoàn toàn mất:

```bash
echo "init log" > app.log
```

![image.png](fe6efe23-09bd-466d-bc1a-36b76bb571dd.png)

Nhưng vẫn có trường hợp dù đã xóa file nhưng file vẫn đang được mở như sau:

Tạo log và script python để giữ file .log:

```bash
nano app.py
```

```python
import time

log = open("app.log", "a")

try:
	while True:
		log.write("Logging...\n")
		log.flush()
		time.sleep(5)
		
except KeyboardInterrupt:
		log.close()
```

```bash
python3 app.py & #Chạy ngầm

jobs #Kiểm tra python có chạy không

rm app.log #Remove log file

sudo lsof | grep app.log 
```

![image.png](e4fe043b-a506-4a7a-8896-dcd4077fc080.png)

Dù đã remove file .log nhưng khi kiểm tra lại bằng lsof thì log file vẫn còn được giữ bởi process:

- `5760` là PID của script.
- FD: 3w write đang ghi vào. TYPE là REG nên là SIZE và size đang tăng dần do đang được ghi
- `(deleted)` nghĩa là file đã xóa, nhưng process vẫn giữ file .log không những tốn tài nguyên CPU mà còn đang chiếm thêm bộ nhớ do size tăng.

Nếu muốn thật sự remove log file thì thực hiện kill <PID> process và kiểm tra lại lsof thì process không còn giữ file nữa:

![image.png](9ce03d93-2d18-40df-b0d3-2d8635b42eda.png)

https://youtu.be/IUWe5Zvnt88

1. Trường hợp 3: kiểm tra cổng mạng đang bị chiếm dụng

Cú pháp lsof giúp lọc chi tiết các kết nối mạng (TCP/UDP), IP, cổng, hostname, phiên bản.

```bash
sudo lsof -i[version][protocol][@hostname|address][service:port]
```

```bash
sudo lsof -i #Liệt kê các kết nối mạng
```

![image.png](image%204.png)

- COMMAND: systemd-r là dịch vụ truy vấn DNS nội bộ (port 53), avahi-daemon multicast DNS (tự động phát hiện thiết bị cùng mạng LAN).
- FD: firefox mở ra nhiều FD để kết nối HTTPS, truy cập file cache, pipe để giữa các tiến trình con.
- TYPE: IPv4/IPv6.
- DEVICE: trường hợp này không còn là device driver (major/minor) mà là socket ID (kernel cấp để phân biệt socket).
- SIZE/OFF: luôn là 0t0 (0: size; textual 0: offset = 0) vì socket không đọc/ghi giống như file.
- NODE: loại kết nối socket TCP/UDP.
- NAME: (ESTABLISH) là trạng thái bắt tay 3 bước TCP  thành công và sẵn sàng truyền dữ liệu giữ 2 bên, (LISTEN) là server chờ kết nối từ client.

```bash
sudo lsof -i4 #Liệt kê kết nối IPv4 (hoặc IPv6)
```

![image.png](4ea9b8b5-a480-4387-b08a-de8e37116404.png)

```bash
sudo lsof -iTCP #Liệt kê kết nối TCP (hoặc UDP)
```

![image.png](951f0b3b-76a2-45bd-aff9-5fbc7c7b6ab6.png)

```bash
sudo lsof -i :443 #Liệt kê cổng 443 (HTTPS)
```

![Ubuntu 64-bit-2025-06-23-21-56-38.png](189fffa8-f1d9-4b62-894a-0f5de332ea34.png)

Một số kết hợp lọc chi tiết hơn:

```bash
sudo lsof -i4 -sTCP:LISTEN #Liệt kê kết nối IPv4 dùng TCP và đang dùng socket LISTEN
```

![image.png](image%205.png)

```bash
sudo lsof -i4 -nP #Liệt kê kết nối IPv4 và không resolve hostname (n), port service (P)
```

![image.png](6015bbbb-209f-4720-98a3-e0653ebb73d7.png)

Tình huống dùng lsof để khắc phục sự cố:

1. Ngăn chặn kết nối không đáng tin:

Không phân giải hostname, port service để xem chính xác các địa chỉ IP thật, ví dụ như hệ thống bị nhiễm malware và thực hiện 1 kết nối ngược từ nạn nhân đến máy chủ hacker nhằm vượt qua tường lửa qua đó hacker có thể điều khiển được hệ thống bị malware, để khắc phục có thể dùng:

```bash
sudo lsof -i nP | grep ESTABLISHED
```

Để kiểm tra các kết nối đến các địa chi IP nếu thấy IP bất thường, không đáng tin thì lập tức kill <PID> để ngắt kết nối, ngăn chặn kết nối xấu.

1. Giả lập tình huống chiếm dụng port 5000 làm cho không thể tạo 1 kết nối khác với port 5000:

![image.png](bf9ed2c9-9619-481e-a76b-ce20c3abf561.png)

Tạo 1 python chiếm port 5000 trên 1 terminal, mở 1 terminal khác để giả lập 1 chương trình khác muốn dùng port 5000 nhưng  “OSError: [Errno 98] Address already in use” port 5000 đã bị chiếm dụng nên không thể tạo kết nối. Để khắc phục cần liệt kê sudo lsof -i :500 để xem PID nào chiếm port 5000, sau đó kill <PID> chiếm port 5000 để có thể dùng port 5000.

https://www.youtube.com/watch?v=qxUoR0ewB6I

## 6. Tài liệu tham khảo

https://lsof.readthedocs.io/en/latest/

https://www.geeksforgeeks.org/lsof-command-in-linux-with-examples/

[How to List Open Files in Linux | lsof Command - GeeksforGeeks](https://www.geeksforgeeks.org/lsof-command-in-linux-with-examples/)

https://www.youtube.com/watch?v=dUQV9WS-A3Y