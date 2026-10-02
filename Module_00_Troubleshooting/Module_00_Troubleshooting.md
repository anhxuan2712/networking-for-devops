# MODULE 0: TƯ DUY TROUBLESHOOTING & HỘP CÔNG CỤ THỰC CHIẾN

---

## 1. MỤC TIÊU HỌC TẬP
Sau khi hoàn thành chương này, kỹ sư có thể:
1. Áp dụng chuẩn xác phương pháp **Divide-and-Conquer (bắt đầu từ L4)** để cô lập nguyên nhân sự cố mạng trong vòng dưới 5 phút, tránh việc phỏng đoán mò mẫm.
2. Sử dụng thành thạo và phân biệt chính xác ý nghĩa các trường dữ liệu của bộ công cụ L2–L7 trên Linux: `ip`, `ss`, `dig`, `curl -w`, `mtr`, `openssl s_client`, `nc`, `nmap`.
3. Đọc hiểu sự khác biệt cốt tử của `Recv-Q` và `Send-Q` trên `ss` giữa trạng thái `LISTEN` và trạng thái `ESTABLISHED`.
4. Viết các bộ lọc BPF (Berkeley Packet Filter) nâng cao cho `tcpdump`, biết cách bắt gói an toàn không gây sập (OOM/I-O stall) máy chủ production chịu tải cao.
5. Giải mã và dựng lại luồng sự cố từ file `.pcap` bằng Wireshark qua các chỉ báo: TCP Retransmission, Duplicate ACK, ZeroWindow, Window Probe.

---

## 2. VẤN ĐỀ THỰC TẾ TRÊN PRODUCTION
Trong môi trường production (Kubernetes, Hybrid Cloud, Microservices), triệu chứng phổ biến nhất được ghi nhận từ ứng dụng là:
> *"API timeout 504"*, *"Connection reset by peer"*, *"Service unavailable 503"*, hoặc *"Không gọi được Database"*.

Phản xạ sai lầm phổ biến của kỹ sư thiếu nền tảng mạng:
* **Chạy `ping <IP>`:** Thấy gói tin phản hồi 0% packet loss => Kết luận vội vàng *"Mạng bình thường, lỗi do code ứng dụng"*. Thực tế `ping` dùng ICMP (L3), không chứng minh được Web Server (L7) có đang lắng nghe trên TCP Port 443 (L4) hay không, hoặc Firewall có đang chặn TCP SYN hay không.
* **Khởi động lại (Restart) container/service một cách mù quáng:** Làm mất toàn bộ dấu vết trạng thái kernel socket, hàng đợi kết nối và bằng chứng sự cố (bảng conntrack thuộc kernel/network namespace không bị xóa khi restart service, nhưng các kết nối ứng dụng đang theo dõi sẽ bị gián đoạn và reset), khiến sự cố tái diễn ngay khi tải tăng trở lại.
* **Đọc log ứng dụng thụ động:** Log ứng dụng chỉ ghi nhận kết quả cuối cùng (ví dụ: `i/o timeout` sau 30 giây). Log không thể trả lời gói tin bị drop tại card mạng cục bộ, tại VPC Route Table, tại Security Group, hay bị Reset bởi đối tác phía xa.

**Bộ công cụ và tư duy ở Module 0 là "bộ đồ nghề sống còn"** giúp kỹ sư đưa ra các giả thuyết khoa học, kiểm chứng bằng số liệu nhị phân/packet thực tế, và khắc phục sự cố có phương pháp.

---

## 3. NGUYÊN LÝ HOẠT ĐỘNG & PHƯƠNG PHÁP LUẬN

### 3.1. Ba mô hình Troubleshooting kinh điển

```
                 [ TẦNG 7: APPLICATION ]   <-- Top-Down bắt đầu từ đây (curl, log app)
                 [ TẦNG 6: PRESENTATION ]
                 [ TẦNG 5: SESSION      ]
---------------------------------------------------------------------------------------
 [ ĐIỂM BẮT ĐẦU ] --> [ TẦNG 4: TRANSPORT    ]   <-- Divide-and-Conquer: Kiểm tra Socket/Port (nc, ss, tcpdump)
---------------------------------------------------------------------------------------
                 [ TẦNG 3: NETWORK      ]   <-- Ping, Route, IP
                 [ TẦNG 2: DATA LINK    ]   <-- ARP, MAC, Interface
                 [ TẦNG 1: PHYSICAL     ]   <-- Bottom-Up bắt đầu từ đây (Link state, Cáp)
```

1. **Bottom-Up (Từ dưới lên - L1 → L7):**
   * *Luồng đi:* Kiểm tra cáp/link card mạng → Bảng MAC/ARP → Định tuyến IP → Port L4 → TLS → HTTP.
   * *Ưu/Nhược điểm:* Rất chắc chắn nhưng tốn nhiều thời gian. Thích hợp cho việc triển khai Data Center mới hoặc cắm cụm Bare-metal mới.
2. **Top-Down (Từ trên xuống - L7 → L1):**
   * *Luồng đi:* Kiểm tra log ứng dụng → Request HTTP bằng `curl` → Kiểm tra bắt tay TLS → Kiểm tra TCP socket.
   * *Ưu/Nhược điểm:* Phù hợp khi nhà phát triển (Developer) báo lỗi ứng dụng, nhưng dễ bị lạc vào biển log mà không phát hiện được lỗi hạ tầng tầng thấp.
3. **Divide-and-Conquer (Chia để trị - Bắt đầu từ L4): Khuyến nghị cho SRE/DevOps**
   * *Nguyên lý:* Đứng tại Tầng 4 (Transport) để kiểm tra socket bằng `nc -zv <IP> <Port>` hoặc `ss -tulpn`.
   * *Nếu L4 THÔNG:* Sự cố **thường nằm ở L5–L7** (TLS Certificate, HTTP Gateway, Header, Code App, DNS). Không cần tốn thời gian kiểm tra L1–L3 (Dây cáp, ARP, IP Route).
   * *Nếu L4 TẮC:* Sự cố nằm ở L1–L4 (Firewall/Security Group drop packet, sai Routing, Server chưa bind listen port, tràn SYN/Accept queue). Lúc này thu hẹp phạm vi kiểm tra xuống L3 (`ip route`, `mtr`) và L2 (`ip neigh`).

> [!WARNING]
> **Khi nào phải quay lại kiểm tra L3 dù TCP 3-way handshake (L4) thành công?**
> * **Hiện tượng MTU / PMTUD Black Hole:** Gói tin bắt tay TCP 3-way handshake (SYN, SYN-ACK, ACK) có kích thước rất nhỏ (~40–60 bytes) nên đi qua trơn tru. Tuy nhiên, khi truyền dữ liệu payload lớn (như TLS Certificate Chain hoặc HTTP POST body), kích thước gói vượt quá MTU của hop trung gian và bị drop âm thầm nếu bản tin ICMP Type 3 Code 4 (Fragmentation Needed) bị chặn (xem chi tiết tại Case 1 và Module 1).
> * **Mất gói ngắt quãng (Intermittent Packet Loss):** Bắt tay TCP có thể may mắn thành công ngẫu nhiên, nhưng khi truyền luồng dữ liệu liên tục sẽ bị drop, timeout và retransmission kéo dài do suy hao đường truyền L3.

---

### 3.2. Vòng lặp 6 bước xử lý sự cố chuẩn SRE

```
 [1. Triệu chứng] ---> [2. Giả thuyết] ---> [3. Kiểm chứng]
 (Metric, Alert)       (Do DNS / MTU / L4)   (tcpdump, ss, dig)
                                                     |
                                            +--------+--------+
                                            |                 |
                                      [Khớp dữ liệu]   [Sai giả thuyết]
                                            |                 |
                                            v                 v
                                    [4. Giảm thiểu]    (Quay lại Bước 2)
                                    (Mitigation/Rollback)
                                            |
                                            v
                                    [5. Root Cause]
                                    (Sửa tận gốc)
                                            |
                                            v
                                    [6. Postmortem]
```

---

## 4. CHI TIẾT KỸ THUẬT & CÁC THAM SỐ CỐT LÕI

### 4.1. Giải mã bí mật của `ss`: Recv-Q và Send-Q

Lệnh `ss` (Socket Statistics) lấy trực tiếp thông tin từ kernel socket qua giao tiếp `netlink`, nhanh hơn đáng kể khi có nhiều socket so với việc quét chuỗi văn bản `/proc/net/tcp` của lệnh cũ `netstat`.

Ý nghĩa của `Recv-Q` và `Send-Q` bị đảo ngược hoàn toàn tùy thuộc vào trạng thái kết nối:

| Trạng thái Socket | Ý nghĩa của `Recv-Q` | Ý nghĩa của `Send-Q` |
| :--- | :--- | :--- |
| **LISTEN** *(Socket đang chờ kết nối mới)* | **Số lượng kết nối đã hoàn thành 3-way handshake** nhưng ứng dụng chưa gọi hàm `accept()` để lấy ra xử lý. | **Giới hạn tối đa của Accept Queue** (Backlog limit), tính bằng min(somaxconn, backlog trong code). |
| **ESTABLISHED** *(Kết nối đang truyền dữ liệu)* | **Số byte dữ liệu đã nhận được từ mạng**, nằm trong Receive Buffer của Kernel nhưng ứng dụng chưa `read()`. | **Số byte dữ liệu đã gửi đi** nằm trong Send Buffer của Kernel nhưng **chưa nhận được ACK** từ phía đối tác. |

> [!IMPORTANT]
> **Quy tắc vàng chẩn đoán quá tải bằng `ss`:**
> * Nếu ở trạng thái `LISTEN` mà `Recv-Q > 0` và vượt quá hoặc tiến sát `Send-Q` => **Ứng dụng bị nghẽn (CPU/Thread lock, Event Loop blocked), không kịp gọi `accept()`.**
>   * *Hành vi kernel mặc định (`tcp_abort_on_overflow = 0`):* Khi Accept Queue đã đầy, kernel sẽ âm thầm drop gói SYN mới (client bị timeout / retransmit SYN), và bỏ qua gói ACK cuối cùng của bắt tay 3 bước (server gửi lại SYN-ACK).
>   * *Nếu cấu hình `tcp_abort_on_overflow = 1`:* Kernel sẽ gửi gói `RST` phản hồi khi nhận gói ACK cuối của bắt tay 3 bước để đóng kết nối ngay lập tức.
> * Nếu ở trạng thái `ESTABLISHED` mà `Send-Q` tăng cao liên tục => **Đường truyền mạng bị nghẽn (packet loss/chậm RTT)** hoặc **Bên nhận bị treo (ZeroWindow), không kịp đọc dữ liệu.**

---

### 4.2. Bắt gói an toàn trên Production với `tcpdump`

Khi chạy `tcpdump` trên hệ thống chịu tải 50.000 req/s, nếu không có tham số bảo vệ, terminal sẽ bị tràn I/O hoặc ghi đầy ổ đĩa gây sập server.

Các cờ (flags) bắt buộc phải nắm:
* `-n`: Không resolve IP thành hostname (tránh sinh thêm hàng nghìn truy vấn DNS ngược gây quá tải DNS server).
* `-nn`: Không resolve cả IP lẫn Port (ví dụ giữ nguyên số port `443` thay vì đổi thành chữ `https`).
* `-s <snaplen>`: Giới hạn số byte bắt trên mỗi packet. Nếu chỉ cần debug bắt tay TCP Header / cờ TCP / Sequence Number, đặt `-s 96` hoặc `-s 128` (giúp giảm đáng kể dung lượng RAM/Disk ghi xuống, tùy thuộc vào kích thước gói tin thực tế). Nếu cần phân tích TLS ClientHello / SNI, cần snaplen lớn hơn nhiều (~1500 bytes) hoặc full packet (`-s 0`) vì TLS ClientHello hiện đại chứa nhiều extension và cipher suites.
* `-c <count>`: Dừng lại sau khi bắt đúng N gói.
* `-C <file_size_MB> -W <file_count> -w <filename>`: Cơ chế ghi xoay vòng (Ring Buffer). Lưu ý tham số `-C` trong `tcpdump` tính theo đơn vị triệu byte (10^6 = 1.000.000 bytes). Ví dụ `-C 100 -W 5 -w /var/log/trace.pcap` sẽ ghi tối đa 5 file, mỗi file 100.000.000 bytes rồi ghi đè file cũ nhất, không bao giờ làm tràn ổ đĩa.
* `-B <buffer_size_KB>`: Tăng buffer nhận gói của kernel (ví dụ `-B 4096` = 4096 KB = 4 MB) để tránh hiện tượng *packet dropped by kernel* khi bắt gói trong điều kiện lưu lượng tải tăng đột biến.

---

## 5. LỆNH THỰC HÀNH, CONFIG & CÁCH ĐỌC OUTPUT MẪU

*Môi trường thực thi chuẩn: Linux (Ubuntu 22.04 LTS kernel 5.15 / Debian 12 kernel 6.1 / RHEL 9 kernel 5.14)*

### 5.1. Khảo sát socket và hàng đợi bằng `ss`

```bash
# Liệt kê tất cả socket TCP đang LISTEN kèm process ID
ss -ltnp
```
*(Ví dụ minh họa, chưa chạy thử)*
```text
State    Recv-Q   Send-Q     Local Address:Port      Peer Address:Port   Process                                          
LISTEN   0        511              0.0.0.0:80             0.0.0.0:*       users:(("nginx",pid=1234,fd=6))                  
LISTEN   129      128            127.0.0.1:8080           0.0.0.0:*       users:(("gunicorn",pid=5678,fd=5))               
```
*Cách đọc:*
* Dòng `nginx:80`: `Send-Q` là 511 (giới hạn queue cấu hình), `Recv-Q` là 0 => Bình thường, Nginx gọi `accept()` lấy kết nối ra xử lý tức thì.
* Dòng `gunicorn:8080`: `Recv-Q` là 129, vượt quá `Send-Q` là 128 => **Accept Queue đã bị tràn (Overflow).** Các client gửi kết nối mới tới port 8080 lúc này đang bị drop gói SYN hoặc treo chờ.

```bash
# Đếm tổng quan socket trên toàn bộ hệ thống
ss -s
```
*(Ví dụ minh họa, chưa chạy thử)*
```text
Total: 1250
TCP:   4050 (estab 320, closed 3500, orphaned 0, timewait 3200)
Transport Total     IP        IPv6
RAW       0         0         0        
UDP       12        8         4        
TCP       550       400       150      
```
*Cách đọc:* Có 3.200 kết nối ở trạng thái `timewait`. Trạng thái `TIME_WAIT` là cơ chế bình thường của giao thức TCP để đảm bảo các gói tin trễ tiêu tán hết trên đường truyền. Nó chỉ đáng lo ngại khi số lượng chiếm phần lớn dải ephemeral port khả dụng (kiểm tra qua `/proc/sys/net/ipv4/ip_local_port_range`) và hệ thống bắt đầu xuất hiện lỗi cấp phát port (`cannot assign requested address`).

```bash
# Xem thông tin timer của các socket TCP (bao gồm cả ESTABLISHED và TIME-WAIT)
ss -tnoa
```
*(Ví dụ minh họa, chưa chạy thử)*
```text
State        Recv-Q  Send-Q    Local Address:Port      Peer Address:Port   Timer
ESTAB        0       0         192.168.1.10:443       10.0.0.5:54321      timer:(keepalive,118min,0)
ESTAB        0       1460      192.168.1.10:443       10.0.0.6:54322      timer:(on,852ms,2)
TIME-WAIT    0       0         127.0.0.1:8080         127.0.0.1:45678     timer:(timewait,58sec,0)
```
*Cách đọc timer:*
* `keepalive`: Đang chạy timer TCP Keepalive (còn 118 phút trước đợt probe tiếp theo).
* `on`: Đang chạy timer truyền lại (Retransmission Timer / RTO). `852ms` là thời gian còn lại trước lần thử lại tiếp theo, `2` là số lần đã retransmit.
* `timewait`: Socket đang đếm ngược thời gian 2MSL (còn 58 giây) trước khi giải phóng hoàn toàn.

> [!WARNING]
> **Bẫy thường gặp:** Nhầm lẫn giữa `Recv-Q` của socket `LISTEN` và `ESTABLISHED`. Thấy `Recv-Q` cao ở socket `LISTEN` lại tưởng là ứng dụng nhận nhiều payload, trong khi bản chất là ứng dụng đang nghẽn CPU/lock thread không kịp gọi `accept()`.

---

### 5.2. Đo đạc chi tiết Timeline kết nối bằng `curl -w`

Tạo file định dạng metrics `curl-format.txt`:
```text
    time_namelookup:  %{time_namelookup}s\n
       time_connect:  %{time_connect}s\n
    time_appconnect:  %{time_appconnect}s\n
   time_pretransfer:  %{time_pretransfer}s\n
      time_redirect:  %{time_redirect}s\n
 time_starttransfer:  %{time_starttransfer}s (TTFB)\n
                    ----------\n
         time_total:  %{time_total}s\n
```

Chạy lệnh kiểm tra tới endpoint mục tiêu (không dùng `-v` để tránh làm rối output timing):
```bash
curl -w "@curl-format.txt" -o /dev/null -s "https://api.example.com/healthz"
```
*(Ví dụ minh họa, chưa chạy thử)*
```text
    time_namelookup:  0.004123s
       time_connect:  0.035412s
    time_appconnect:  0.066820s
   time_pretransfer:  0.066950s
      time_redirect:  0.000000s
 time_starttransfer:  0.852104s (TTFB)
                    ----------
         time_total:  0.854210s
```
*Phương pháp phân tích thời gian:*
1. **Phân giải DNS:** `0.004s` (4.1ms) => DNS rất nhanh.
2. **Bắt tay TCP Handshake (L4):** time_connect - time_namelookup = 0.0354 - 0.0041 = 0.0313s (~31ms, xấp xỉ 1 RTT mạng).
3. **Bắt tay TLS Handshake (L6):** time_appconnect - time_connect = 0.0668 - 0.0354 = 0.0314s (~31ms, xấp xỉ 1 RTT trong TLS 1.3).
4. **Thời gian máy chủ backend xử lý code (TTFB - L7):** time_starttransfer - time_pretransfer = 0.8521 - 0.0669 = 0.7852s (785ms).
5. **KẾT LUẬN:** Toàn bộ giai đoạn mạng L3/L4/L6 chỉ tốn ~67ms, nhưng Backend App mất tới 785ms mới trả về byte dữ liệu đầu tiên => **Nghi mạnh backend/phía sau Load Balancer xử lý chậm (Database slow query, lock thread, external dependency); cần đối chiếu log ứng dụng và đo trực tiếp vào backend bỏ qua Load Balancer/Proxy để xác nhận.**

> [!WARNING]
> **Bẫy thường gặp:** Dùng cờ `-v` chung với `curl -w` khiến các dòng debug TLS/HTTP header in xen lẫn với khối format metrics, gây khó khăn cho việc parse log tự động hoặc đọc nhầm giá trị.

---

### 5.3. Kiểm tra TLS Certificate Chain bằng `openssl s_client`

```bash
# Debug kết nối TLS, kiểm tra SNI, đàm phán ALPN và hiển thị Certificate Chain
openssl s_client -connect api.example.com:443 -servername api.example.com -alpn h2,http/1.1 -showcerts < /dev/null
```
*(Ví dụ minh họa trích đoạn, chưa chạy thử)*
```text
CONNECTED(00000003)
---
Certificate chain
 0 s:CN = api.example.com
   i:C = US, O = Let's Encrypt, CN = R10
 1 s:C = US, O = Let's Encrypt, CN = R10
   i:C = US, O = Internet Security Research Group, CN = ISRG Root X1
---
SSL handshake has read 3210 bytes and written 410 bytes
Verification: OK
---
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 256 bit
Secure Renegotiation IS NOT supported
ALPN protocol: h2
```
*Cách đọc:*
* `Certificate chain`: Hiển thị chuỗi chứng chỉ đầy đủ từ Server Cert (0) tới Intermediate Cert (1).
* `Verification: OK`: Hệ thống tin cậy chuỗi chứng chỉ này theo Trust Store cục bộ.
* `ALPN protocol: h2`: Đàm phán thành công giao thức HTTP/2 qua ALPN extension.

> [!WARNING]
> **Bẫy thường gặp:** Quên tham số `-servername` khi debug máy chủ dùng SNI (Server Name Indication). Khi thiếu `-servername`, web server trả về default certificate (thường là self-signed hoặc domain mặc định), dẫn đến kết luận sai rằng chứng chỉ SSL của website bị lỗi.

---

### 5.4. Bắt gói tin chọn lọc với `tcpdump` BPF Filter

```bash
# Bắt gói tin TCP SYN thuần túy (SYN=1, ACK=0) để phát hiện connection mới hoặc SYN scan
tcpdump -nn -i eth0 'tcp[tcpflags] & (tcp-syn) != 0 and tcp[tcpflags] & (tcp-ack) == 0'

# Bắt các gói tin RST (kết nối bị reset cưỡng bức) trên port 3306 (MySQL)
tcpdump -nn -i any 'tcp port 3306 and tcp[tcpflags] & tcp-rst != 0' -c 50

# Bắt gói tin UDP DNS có tổng độ dài gói tin trên dây (wire length bao gồm cả L2 Ethernet header) lớn hơn 512 bytes
tcpdump -nn -i eth0 'udp port 53 and greater 512'

# Xuất file an toàn: Bắt trên interface ens5, file xoay vòng 50.000.000 bytes, tối đa 3 file, chụp 96 bytes header
tcpdump -nn -i ens5 -s 96 -C 50 -W 3 -w /tmp/capture_prod.pcap 'port 443'
```

> [!WARNING]
> **Bẫy thường gặp:** Bộ lọc `tcp[tcpflags] & tcp-syn != 0` sẽ bắt cả gói **SYN** (khởi tạo kết nối) lẫn gói **SYN-ACK** (phản hồi từ server). Nếu chỉ muốn bắt duy nhất gói mở kết nối từ client, bắt buộc phải lọc thêm `tcp[tcpflags] & tcp-ack == 0`.

---

### 5.5. Khảo sát hạ tầng L2/L3 bằng `ip` (`iproute2`)

Lệnh `ip` là công cụ chuẩn thay thế hoàn toàn cho `ifconfig`, `route`, và `arp` cũ.

```bash
# 1. Xem vắn tắt địa chỉ IP và trạng thái card mạng
ip -br addr
ip -br link

# 2. Kiểm tra bảng định tuyến cho một IP đích cụ thể
ip route get 1.1.1.1

# 3. Xem bảng ARP / Neighbor cache (L2)
ip neigh

# 4. Kiểm tra thống kê lỗi và drop gói tin trên interface
ip -s link show dev eth0
```
*(Ví dụ minh họa, chưa chạy thử)*
```text
$ ip -br addr
lo               UNKNOWN        127.0.0.1/8 ::1/128 
eth0             UP             192.168.1.50/24 fe80::a00:27ff:fe4e:66a1/64 

$ ip route get 1.1.1.1
1.1.1.1 via 192.168.1.1 dev eth0 src 192.168.1.50 uid 1000 
    cache 

$ ip neigh
192.168.1.1 dev eth0 lladdr 52:54:00:12:34:56 REACHABLE
192.168.1.20 dev eth0 lladdr 52:54:00:98:76:54 STALE
192.168.1.99 dev eth0  FAILED

$ ip -s link show dev eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 08:00:27:4e:66:a1 brd ff:ff:ff:ff:ff:ff
    RX:  bytes packets errors dropped missed mcast   
     154201994  124500      0       0      0     0 
    TX:  bytes packets errors dropped carrier collsns 
      12890450   89200      0       0       0       0 
```
*Cách đọc:*
* `ip route get`: Chỉ ra chính xác gateway (`192.168.1.1`), interface output (`eth0`) và source IP được chọn theo bảng định tuyến kernel.
* `ip neigh`:
  * `REACHABLE`: Địa chỉ MAC hợp lệ và đã được xác thực gần đây.
  * `STALE`: Bản ghi ARP hợp lệ nhưng đã quá hạn kiểm tra liveness; kernel sẽ probe lại khi có traffic mới.
  * `FAILED`: Không nhận được phản hồi ARP Reply => Thiết bị đích không tồn tại hoặc bị cô lập ở tầng L2.
* `ip -s link`: Cột `dropped`, `errors`, `missed` ở nhánh RX/TX tăng cao là dấu hiệu nghẽn ring buffer card mạng hoặc lỗi phần cứng/driver L1–L2.

> [!WARNING]
> **Bẫy thường gặp:** Trạng thái card mạng hiển thị `NO-CARRIER` hoặc `DOWN` trong `ip link` nhưng kỹ sư chỉ kiểm tra `ip addr`, dẫn đến không phát hiện ra việc đứt link vật lý hoặc switch port bị disable.

---

### 5.6. Truy vấn và phân tích DNS chuyên sâu bằng `dig`

```bash
# 1. Truy vấn nhanh chỉ lấy giá trị IP
dig +short api.example.com

# 2. Truy vấn chi tiết, chỉ hiển thị phần ANSWER và Header status
dig +noall +answer +comments api.example.com

# 3. Truy vấn lần vết từ Root DNS Servers (+trace)
dig +trace api.example.com

# 4. Truy vấn chỉ định DNS Server cụ thể qua giao thức TCP (+tcp)
dig @8.8.8.8 api.example.com +tcp
```
*(Ví dụ minh họa, chưa chạy thử)*
```text
$ dig +noall +answer +comments api.example.com
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 41205
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; ANSWER SECTION:
api.example.com.	300	IN	A	203.0.113.10
```
*Cách đọc:*
* `status`:
  * `NOERROR`: Phân giải thành công, có bản ghi trả về.
  * `NXDOMAIN`: Tên miền không tồn tại trên hệ thống DNS.
  * `SERVFAIL`: DNS Server gặp lỗi nội bộ (hết hạn DNSSEC, timeout khi query upstream).
  * `REFUSED`: DNS Server từ chối phục vụ (chặn IP, policy restriction).
* `flags`:
  * `aa` (Authoritative Answer): Bản ghi được trả về trực tiếp từ DNS Server quản lý chính thức của tên miền (Authoritative Nameserver).
  * `ra` (Recursion Available) và `rd` (Recursion Desired): Truy vấn đệ quy từ Resolver/Cache.

> [!WARNING]
> **Bẫy thường gặp:** Không phân biệt giữa Resolver Cache và Authoritative Nameserver. Khi vừa đổi bản ghi DNS, query mặc định tới local resolver có thể vẫn trả về IP cũ do TTL cache, cần dùng `dig @<auth_ns> <domain>` để xác nhận bản ghi gốc đã cập nhật hay chưa.

---

### 5.7. Phân tích chất lượng đường truyền với `mtr`

`mtr` (My Traceroute) kết hợp tính năng của `traceroute` và `ping`, gửi gói tin liên tục để đo tỷ lệ mất gói và độ trễ trên từng hop.

```bash
# Chạy chế độ báo cáo (report mode) gửi 100 gói ICMP
mtr -r -c 100 203.0.113.10

# Chạy báo cáo bằng TCP SYN qua cổng 443 (vượt firewall chặn ICMP)
mtr -r -c 100 -T -P 443 203.0.113.10
```
*(Ví dụ minh họa, chưa chạy thử)*
```text
HOST: devops-box                  Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- 192.168.1.1                0.0%   100    0.4   0.5   0.3   1.2   0.2
  2.|-- 10.50.0.1                  0.0%   100    2.1   2.3   1.9   5.4   0.6
  3.|-- 203.0.113.254             60.0%   100   15.2  16.1  14.8  45.0   4.2
  4.|-- 198.51.100.1               0.0%   100   15.4  15.5  14.9  18.2   0.5
  5.|-- 203.0.113.10               0.0%   100   15.6  15.7  15.1  19.0   0.6
```
*Cách đọc:*
* **Quy tắc phân biệt Router Rate-limit ICMP và Mất gói thực sự:**
  * Tại **Hop 3** (`203.0.113.254`): Hiển thị `Loss% = 60.0%`. Tuy nhiên tại **Hop 4** và **Hop 5**, `Loss%` quay trở lại `0.0%`. => **Đây là hiện tượng Router tại Hop 3 giới hạn tốc độ (Rate-limit) xử lý gói ICMP tạo bởi CPU điều khiển (Control Plane), không phải sự cố mất gói trên đường truyền dữ liệu.**
  * Nếu mất gói xảy ra thực sự do đứt cáp/nghẽn mạng, tỷ lệ mất gói sẽ xuất hiện tại một hop và **tiếp tục duy trì hoặc tăng dần trên tất cả các hop phía sau cho tới đích**.

> [!WARNING]
> **Bẫy thường gặp:** Hoảng loạn khi thấy một hop ở giữa đường có packet loss 50%–90% trong khi hop đích 0% loss. Cần nhìn vào hop cuối cùng để xác định chất lượng đường truyền thực tế.

---

### 5.8. Kiểm tra và quét cổng với `nc` (Netcat) và `nmap`

```bash
# 1. Netcat: Kiểm tra nhanh cổng TCP 443 với timeout 3s
nc -zv -w 3 203.0.113.10 443

# 2. Netcat: Mở một TCP listener tạm thời trên port 9090 để test kết nối đến
# Cú pháp OpenBSD netcat (Ubuntu/Debian): nc -l 9090 (hoặc nc -l -p 9090 trên traditional/ncat)
nc -l -p 9090

# 3. Nmap: Quét TCP SYN Stealth Scan (-sS) vào các cổng chỉ định (yêu cầu quyền root)
sudo nmap -sS -p 80,443,3306 203.0.113.10

# 4. Nmap: Quét cổng UDP (-sU)
sudo nmap -sU -p 53,123 203.0.113.10
```
*(Ví dụ minh họa, chưa chạy thử)*
```text
$ sudo nmap -sS -p 80,443,3306 203.0.113.10
Starting Nmap 7.80 ( https://nmap.org )
Nmap scan report for 203.0.113.10
PORT     STATE    SERVICE
80/tcp   open     http
443/tcp  open     https
3306/tcp filtered mysql
```
*Cách đọc trạng thái cổng:*
* `open`: Dịch vụ đang hoạt động, gửi lại phản hồi TCP SYN-ACK hoặc UDP response.
* `closed`: Server nhận được gói tin nhưng không có ứng dụng nào lắng nghe trên cổng, kernel gửi trả gói `RST` (hoặc ICMP Port Unreachable đối với UDP).
* `filtered`: Nmap không nhận được bất kỳ phản hồi nào hoặc nhận được bản tin ICMP Filtered => Gói tin bị drop bởi Firewall / Security Group / WAF.
* `open|filtered` (đặc thù khi quét UDP `-sU`): Không có phản hồi trả về từ cổng UDP. Do UDP là giao thức không hướng kết nối, Nmap không thể phân biệt được giữa việc cổng đang mở (ứng dụng nhận payload nhưng không gửi phản hồi) hay gói tin bị firewall drop âm thầm.

> [!CAUTION]
> **Cảnh báo an toàn và pháp lý:** Chỉ thực hiện quét mạng bằng `nmap` trên các máy chủ và dải IP thuộc quyền quản trị của bạn hoặc đã được sự đồng ý bằng văn bản. Việc quét không ủy quyền có thể kích hoạt hệ thống IDS/IPS và vi phạm chính sách an ninh thông tin.

---

### 5.9. Phân tích gói tin chuyên sâu bằng Wireshark

Wireshark là công cụ trực quan hóa file capture `.pcap`.

#### Các Display Filters cốt tử trong SRE/DevOps:
* `tcp.analysis.retransmission`: Lọc các gói TCP phải truyền lại do timeout (RTO).
* `tcp.analysis.fast_retransmission`: Lọc các gói truyền lại nhanh được kích hoạt bởi 3 Duplicate ACK liên tiếp.
* `tcp.analysis.duplicate_ack`: Lọc các gói ACK trùng lặp (bên nhận báo hiệu bị thiếu một packet ở giữa luồng).
* `tcp.analysis.zero_window`: Lọc các gói tin mà bên nhận thông báo `Window Size = 0` (bộ đệm Receive Buffer đã đầy hoàn toàn, yêu cầu bên gửi ngừng phát dữ liệu).
* `tcp.analysis.window_full`: Lọc các gói tin mà bên gửi đã truyền hết kích thước Receive Window cho phép của bên nhận.
* `tcp.flags.reset == 1`: Lọc tất cả gói tin TCP Reset (ngắt kết nối cưỡng bức).
* `tls.handshake.type == 1`: Lọc các gói TLS Client Hello (chứa danh sách ciphers, SNI).

#### Công cụ phân tích chuyên sâu:
1. **Expert Information (`Analyze → Expert Information`):** Tự động gom nhóm các vấn đề bất thường trong luồng mạng thành 4 cấp độ: *Errors*, *Warnings*, *Notes*, *Chats*. Giúp định vị ngay hiện tượng Out-of-Order, ZeroWindow, hoặc Handshake Failure.
2. **TCP Stream Graphs (`Statistics → TCP Stream Graphs → Time-Sequence (tcptrace)`):**
   * Đồ thị biểu diễn tiến trình gửi byte sequence theo thời gian.
   * *Đường dốc đều đặn:* Băng thông mượt mà, không nghẽn.
   * *Đường nằm ngang (Flatline):* Luồng truyền bị tắc nghẽn (TCP Stall do ZeroWindow, retransmission timeout hoặc mất gói nghiêm trọng).

```
   Sequence
      ^
      |                 / (Truyền mượt mà)
      |                /
      |          +----+   (Flatline: Stall / Retransmission / ZeroWindow)
      |         /
      |        /
      +-------------------------> Time
```

* **Ý nghĩa bộ ba ZeroWindow - Window Probe:**
  * Khi ứng dụng nhận xử lý quá chậm, Receive Buffer đầy, nó gửi gói `TCP ZeroWindow`.
  * Bên gửi dừng truyền dữ liệu và định kỳ gửi gói `TCP ZeroWindowProbe` để thăm dò xem Receive Buffer của bên nhận đã giải phóng chưa.
  * *Hành vi probe:* Tùy hệ điều hành; Linux thường gửi probe với sequence number cũ (seq - 1) và độ dài 0 byte; một số hệ điều hành khác gửi probe với 1 byte payload. Wireshark gắn nhãn `[TCP ZeroWindowProbe]` cho cả hai dạng [CẦN XÁC MINH].
  * Khi bên nhận giải phóng được bộ đệm, nó phản hồi bằng gói `TCP WindowUpdate` để kích hoạt việc truyền dữ liệu trở lại.

> [!WARNING]
> **Bẫy thường gặp:** Nhầm lẫn giữa *Capture Filter* (dùng cú pháp BPF khi đang chạy bắt gói) và *Display Filter* (dùng cú pháp riêng của Wireshark khi phân tích file `.pcap`).

---

## 6. BÀI LAB THỰC HÀNH TỪNG BƯỚC (STEP-BY-STEP LAB)

### Tên bài Lab: Cô lập và truy tìm nguyên nhân "Service Unavailable" do tràn Accept Queue trên Linux

*Mục tiêu:* Tự tạo môi trường giả lập sự cố Accept Queue bị nghẽn, sử dụng bộ công cụ `ss`, `curl`, `nstat`, `tcpdump` và `openssl` để quan sát hành vi của Linux Kernel và đo đạc chính xác hiện tượng drop kết nối.

```
+-------------------------------------------------------------+
|                      LINUX HOST / VM                        |
|                                                             |
|  [ Client: curl / openssl / nc ]                            |
|              |                                              |
|              v (TCP / HTTP Request tới 127.0.0.1:9090)      |
|              |                                              |
|  [ Linux Kernel TCP Stack: Accept Queue (Backlog = 2) ]     |
|              |                                              |
|              v                                              |
|  [ Backend Server: Python Hang Server (Không accept) ]      |
+-------------------------------------------------------------+
```

#### Bước 1: Dựng môi trường giả lập sự cố (Server treo không gọi `accept()`)
Mở Terminal 1 và khởi tạo script Python socket server với backlog cực nhỏ:
```bash
cat << 'EOF' > /tmp/hang_server.py
import socket
import time

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.bind(('127.0.0.1', 9090))
# Thiết lập listen backlog = 2
s.listen(2)
print("[*] Server dang lang nghe tren 127.0.0.1:9090 (Backlog=2)...")

# Treo tien trinh, co tinh khong goi s.accept()
while True:
    time.sleep(10)
EOF

python3 -u /tmp/hang_server.py &
```

#### Bước 2: Khảo sát trạng thái socket ban đầu
Kiểm tra trạng thái socket khi chưa có traffic:
```bash
ss -lnt '( sport = :9090 )'
```
*(Ví dụ minh họa, chưa chạy thử)*
```text
State      Recv-Q Send-Q Local Address:Port  Peer Address:Port
LISTEN     0      2          127.0.0.1:9090       0.0.0.0:*
```
*Nhận xét:* `Send-Q` = 2 (kích thước hàng đợi tối đa), `Recv-Q` = 0 (chưa có kết nối nào hoàn thành 3-way handshake đang chờ).

#### Bước 3: Kiểm tra request đơn lẻ và lấp đầy Accept Queue
1. **Gửi 1 request đầu tiên:**
   ```bash
   curl -m 2 http://127.0.0.1:9090/
   ```
   *Hiện tượng:* Với `listen(2)`, kernel vẫn hoàn tất 3-way handshake cho các kết nối đầu tiên (khoảng backlog + 1 kết nối). Do đó `curl` **kết nối thành công ở tầng TCP**, sau đó bị treo chờ response từ ứng dụng và timeout ở tầng ứng dụng sau 2 giây (`curl: (28) Operation timed out after 2000 milliseconds with 0 bytes received`), **hoàn toàn không báo lỗi connection timed out**.

2. **Lấp đầy hoàn toàn Accept Queue bằng cách mở đồng thời 5 kết nối nền:**
   ```bash
   for i in 1 2 3 4 5; do nc -w 60 127.0.0.1 9090 & done
   ```

3. **Kiểm tra lại trạng thái socket sau khi lấp đầy hàng đợi:**
   ```bash
   ss -lnt '( sport = :9090 )'
   ```
   *(Ví dụ minh họa, chưa chạy thử)*
   ```text
   State      Recv-Q Send-Q Local Address:Port  Peer Address:Port
   LISTEN     3      2          127.0.0.1:9090       0.0.0.0:*
   ```
   *Phân tích:* `Recv-Q` (3) đã vượt quá `Send-Q` (2). Accept Queue đã chính thức bị **tràn (Overflow)**.

#### Bước 4: Kiểm chứng phản ứng Kernel khi Accept Queue bị tràn
1. **Chụp bộ đếm kernel ngay trước khi gửi request kiểm tra:**
   ```bash
   nstat -az TcpExtListenOverflows TcpExtListenDrops
   ```
   *(Ví dụ minh họa, chưa chạy thử)*
   ```text
   TcpExtListenOverflows         5    0.0
   TcpExtListenDrops             5    0.0
   ```
   *(Lưu ý: Bộ đếm trước đó đã khác 0 do các tiến trình `nc` nền ở Bước 3 bị drop và retransmit).*

2. **Bắt gói tin trên loopback interface:**
   ```bash
   sudo tcpdump -nn -i lo 'port 9090' -c 10 &
   ```

3. **Gửi kết nối mới khi hàng đợi đã đầy:**
   ```bash
   curl --connect-timeout 2 --max-time 3 http://127.0.0.1:9090/
   ```
   *Kết quả:* Lúc này `curl` báo lỗi ngay ở bước kết nối TCP:
   `curl: (28) Connection timed out after 2000 milliseconds`

4. **Kiểm tra lại bộ đếm kernel ngay sau lệnh curl:**
   ```bash
   nstat -az TcpExtListenOverflows TcpExtListenDrops
   ```
   *(Ví dụ minh họa, chưa chạy thử)*
   ```text
   TcpExtListenOverflows         7    0.0
   TcpExtListenDrops             7    0.0
   ```
   *Nhận định delta:* Cả hai chỉ số tăng thêm delta >= 2 (gồm gói SYN ban đầu tại t=0 và gói SYN truyền lại tại t ≈ 1s trong khoảng thời gian 2 giây của `--connect-timeout`).

5. **Quan sát log `tcpdump`:**
   *(Ví dụ minh họa, chưa chạy thử)*
   ```text
   127.0.0.1.48210 > 127.0.0.1.9090: Flags [S], seq 10001, win 65495, length 0
   127.0.0.1.48210 > 127.0.0.1.9090: Flags [S], seq 10001, win 65495, length 0
   ```
   *Giải thích bản chất:* Khi Accept Queue đã đầy, theo mã nguồn Linux Kernel (`tcp_conn_request`), gói tin SYN mới bị drop âm thầm mà không có SYN-ACK phản hồi. Client liên tục gửi lại gói SYN cho tới khi chạm `--connect-timeout 2`.
   *(Ghi chú: Lệnh `tcpdump` trên cổng 9090 cũng có thể bắt lẫn các gói SYN retransmit của các tiến trình `nc` nền; để quan sát riêng gói tin của `curl`, có thể lọc theo source port của curl).*

#### Bước 5: Kiểm tra cổng non-TLS bằng `openssl s_client`
Để thấy rõ phản ứng bắt tay TLS vào cổng non-TLS (khác với server bị treo không accept ở port 9090), khởi chạy một Web Server HTTP thô trên port 9091:
```bash
python3 -m http.server 9091 &
openssl s_client -connect 127.0.0.1:9091 < /dev/null
```
*(Ví dụ minh họa, chưa chạy thử)*
```text
CONNECTED(00000003)
# Trên OpenSSL 3.0 (Ubuntu 22.04 / Debian 12):
40477D80B57F0000:error:0A00010B:SSL routines:ssl3_get_record:wrong version number:../ssl/record/ssl3_record.c:354:
# Trên OpenSSL 1.1.1:
140123456:error:1408F10B:SSL routines:ssl3_get_record:wrong version number:../ssl/record/ssl3_record.c:332:
```
*Ý nghĩa:* Giúp kỹ sư phân biệt giữa lỗi mạng L4 (không kết nối được) và lỗi giao thức L6/L7 (gọi nhầm TLS vào cổng HTTP thuần).

#### Bước 6: Dọn dẹp môi trường Lab
```bash
pkill -f hang_server.py
pkill -f "http.server 9091"
pkill -x nc
rm -f /tmp/hang_server.py
```

---

## 7. CÁC CASE STUDY SỰ CỐ PRODUCTION THỰC TẾ

### Case 1: Ứng dụng timeout ngắt quãng; `ping` thông suốt nhưng `curl` treo ở TLS Handshake (MTU / PMTUD Black Hole)

* **Triệu chứng:** Client gọi API HTTPS tới một cụm backend trên Cloud thỉnh thoảng bị timeout (30s). `ping` tới IP Gateway thì 100% phản hồi với RTT 5ms. Lệnh `curl -v` hiển thị:
  ```text
  * Connected to api.example.com (203.0.113.10) port 443 (#0)
  * ALPN, offering h2,http/1.1
  * TLSv1.3 (OUT), TLS handshake, Client hello (1):
  * [Treo mãi tại đây cho đến khi timeout]
  ```
* **Bối cảnh mạng:** Client sử dụng card mạng `eth0` với MTU chuẩn 1500. Giữa đường truyền có một tunnel định tuyến (WireGuard / IPsec / GRE) với MTU hiệu dụng bị giảm xuống 1420 bytes do phần overhead đóng gói header tunnel.
* **Giả thuyết:**
  1. *Giả thuyết A:* Tường lửa/WAF chặn IP của client.
  2. *Giả thuyết B (MTU Black Hole):* Gói tin nhỏ (TCP SYN handshake và TLS Client Hello nhỏ) đi qua bình thường. Nhưng khi Server gửi chuỗi dữ liệu phản hồi đầu tiên (gồm `ServerHello` + `Certificate Chain` lớn chứa nhiều chứng chỉ, tổng kích thước vài KB được chia thành nhiều segment TCP đầy MSS 1460 bytes), các segment này vượt quá MTU 1420 của tunnel. Router trung gian drop gói và gửi bản tin `ICMP Type 3 Code 4 (Fragmentation Needed)` / `ICMPv6 Type 2 (Packet Too Big)` ngược về phía Server. Tuy nhiên, Firewall / Security Group phía **Server** đã chặn toàn bộ lưu lượng ICMP chiều vào. Server không nhận được thông báo PMTUD nên không thể hạ MSS, liên tục retransmit các segment lớn mà Client không bao giờ nhận được, dẫn đến Client treo vĩnh viễn ở bước chờ ServerHello.
* **Lệnh kiểm tra và chẩn đoán:**
  1. **Từ phía Client:** Chạy `tracepath` để thăm dò Path MTU (lệnh này nhận được ICMP từ router trung gian gửi về client):
     ```bash
     tracepath 203.0.113.10
     ```
     *(Ví dụ minh họa, chưa chạy thử)*
     ```text
      1?: [LOCALHOST]     pmtu 1500
      1:  192.168.1.1                                           0.512ms 
      2:  10.100.0.1                                           12.310ms pmtu 1420
      3:  203.0.113.10                                         15.420ms reached
          Resume: pmtu 1420
     ```
  2. **Từ phía Server:** Chạy phép thử Black Hole hướng về Client với cờ Don't Fragment (`-M do`):
     ```bash
     # Thử kích thước chuẩn 1472 bytes (1472 + 8 ICMP + 20 IP = 1500 MTU)
     ping -c 2 -M do -s 1472 <CLIENT_IP>
     # Thử kích thước nhỏ hơn 1392 bytes (1392 + 8 + 20 = 1420 MTU)
     ping -c 2 -M do -s 1392 <CLIENT_IP>
     # Kiểm tra tracepath từ Server về Client
     tracepath <CLIENT_IP>
     ```
     *Cách đọc kết quả:*
     * Gói 1472 bytes từ Server: Không có phản hồi (im lặng hoàn toàn / 100% loss) do router tunnel drop gói và bản tin ICMP Frag Needed gửi ngược về server bị chặn bởi firewall/SG phía Server (Black Hole).
     * Gói 1392 bytes từ Server: Phản hồi thành công 0% loss => Xác nhận Path MTU bị giới hạn ở 1420 bytes.
     * `tracepath <CLIENT_IP>` chạy từ Server: Không thấy pmtu giảm (vẫn hiển thị 1500) do ICMP phản hồi bị drop tại firewall server.
  3. **Bắt gói tin trên máy chủ server bằng `tcpdump`:**
     ```bash
     sudo tcpdump -nn -i any 'host <CLIENT_IP> and port 443'
     ```
     *Hiện tượng:* Thấy Server liên tục gửi lại các TCP segment kích thước `length 1460` (retransmission) nhưng client không gửi bất kỳ ACK nào cho các segment này.
* **Giải thích nguyên nhân:** Server tính toán TCP MSS dựa trên MSS mà Client quảng bá trong gói SYN ban đầu (theo MTU 1500 của Client => MSS = 1500 - 40 = 1460). Do đó Server không hề biết đường truyền trung gian có MTU nhỏ hơn nếu bản tin ICMP Fragmentation Needed bị chặn phía Server.
* **Lý do sự cố xảy ra "ngắt quãng":** Chỉ các request đi qua luồng định tuyến có tunnel VPN hoặc chỉ các domain có chuỗi TLS Certificate Chain lớn vượt quá MTU mới bị drop; các request HTTP payload nhỏ hoặc kết nối không qua tunnel vẫn hoạt động bình thường.
* **Cách xử lý:**
  1. Cấu hình **TCP MSS Clamping** trên Gateway/Router biên của tunnel:
     ```bash
     iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
     ```
  2. Mở cho phép các bản tin ICMP Type 3 Code 4 (ICMPv4) và ICMPv6 Type 2 qua Firewall/Security Group phía Server để cơ chế Path MTU Discovery (PMTUD - RFC 1191) hoạt động chính xác.
  3. Đặt MTU chính xác trên interface của tunnel hoặc card mạng client (`ip link set dev eth0 mtu 1420`).

> [!NOTE]
> *Ghi chú mở rộng:* Với các bộ cipher suites hậu lượng tử (Hybrid Post-Quantum TLS như X25519Kyber768 / ML-KEM), kích thước của bản tin `ClientHello` từ phía Client có thể vượt quá 1 packet (> 1 MSS) và cũng có thể bị drop ngay từ chiều Client → Server nếu gặp MTU mismatch qua các thiết bị middlebox [CẦN XÁC MINH].

---

### Case 2: Server từ chối kết nối dù CPU/RAM rảnh do cấu hình giới hạn Accept Queue thấp

* **Triệu chứng:** Vào đợt cao điểm, hệ thống API Gateway báo lỗi `502 Bad Gateway` ngẫu nhiên cho khoảng 15% request. CPU máy chủ chỉ sử dụng 25%, RAM dư 60%, băng thông mạng 100 Mbps / 1 Gbps.
* **Giả thuyết:**
  1. *Giả thuyết 1:* Cạn kiệt Ephemeral Port (lưu ý: hiện tượng này thường xảy ra ở phía client/API Gateway khi gọi outbound tới backend do thiếu connection pooling, hiếm khi xảy ra trên server chỉ nhận kết nối inbound).
  2. *Giả thuyết 2:* Số lượng kết nối vượt quá giới hạn File Descriptor (`ulimit -n`).
  3. *Giả thuyết 3:* Hàng đợi lắng nghe Accept Queue bị nghẽn do cấu hình backlog trong code/config ứng dụng bị đặt thấp.
* **Lệnh kiểm tra và loại trừ:**
  1. **Loại trừ Giả thuyết 1 (Ephemeral Port):**
     ```bash
     # Kiểm tra tổng số kết nối đang hoạt động và TIME_WAIT
     ss -s
     # Kiểm tra dải ephemeral port của hệ thống
     cat /proc/sys/net/ipv4/ip_local_port_range
     ```
     *Kết quả:* Dải port từ `32768 60999` (hơn 28.000 port), tổng số socket `ESTABLISHED` + `TIME_WAIT` chỉ là 2.500 => Không thiếu ephemeral port.
  2. **Loại trừ Giả thuyết 2 (File Descriptor):**
     ```bash
     # Kiểm tra giới hạn Max Open Files của tiến trình server
     cat /proc/<PID>/limits | grep "Max open files"
     # Đếm số FD thực tế đang mở
     ls /proc/<PID>/fd | wc -l
     ```
     *Kết quả:* Giới hạn `Max open files` là 65535, số FD đang sử dụng chỉ là 1.200 => Không cạn kiệt File Descriptor.
  3. **Kiểm tra và xác nhận Giả thuyết 3 (Accept Queue saturation):**
     ```bash
     # Kiểm tra hàng đợi socket
     ss -lnt '( sport = :8000 )'
     # Kiểm tra bộ đếm tràn queue của kernel
     nstat -az TcpExtListenOverflows TcpExtListenDrops
     ```
     *(Ví dụ minh họa, chưa chạy thử)*
     ```text
     $ ss -lnt '( sport = :8000 )'
     State    Recv-Q   Send-Q   Local Address:Port   Peer Address:Port
     LISTEN   129      128             0.0.0.0:8000          0.0.0.0:*

     $ nstat -az TcpExtListenOverflows TcpExtListenDrops
     TcpExtListenOverflows         45210    0.0
     TcpExtListenDrops             45210    0.0
     ```
* **Phân tích nguyên nhân:**
  * Trên các hệ điều hành Linux hiện đại (kernel 5.4+ như Ubuntu 22.04, Debian 12, RHEL 9), giá trị `net.core.somaxconn` mặc định của kernel đã là `4096` (hoặc cao hơn).
  * Tuy nhiên, trong mã nguồn hoặc file cấu hình của ứng dụng (ví dụ trong mã nguồn Python socket `s.listen(128)` hoặc Node.js server), backlog bị gán cứng giá trị cũ là `128`.
  * Do công thức Max Queue = min(somaxconn, backlog), kích thước Accept Queue bị bóp xuống chỉ còn 128 slot. Khi lưu lượng kết nối bùng nổ trong giây cao điểm (Burst connection), hàng đợi đầy lập tức khiến kernel drop các gói kết nối tiếp theo.
  * *Lưu ý kỹ thuật:* Tham số `net.ipv4.tcp_max_syn_backlog` chỉ quản lý hàng đợi **SYN Queue** (kết nối nửa mở `SYN_RECV`), không liên quan trực tiếp đến dung lượng Accept Queue.
* **Cách xử lý:**
  1. Cấu hình bền vững tham số `net.core.somaxconn` trong file `/etc/sysctl.d/99-network.conf`:
     ```bash
     echo "net.core.somaxconn = 65535" | sudo tee -a /etc/sysctl.d/99-network.conf
     sudo sysctl --system
     ```
     *(Cảnh báo: Lệnh `sysctl -w` chỉ có tác dụng tạm thời trong bộ nhớ và sẽ bị mất sau khi reboot máy chủ; tham khảo chi tiết tại Mục 8.5).*
  2. Nâng tham số backlog trong cấu hình hoặc mã nguồn ứng dụng (ví dụ Gunicorn: `--backlog 4096`, Nginx: `listen 80 backlog=4096;`).
  3. **Bước bắt buộc - Restart/Reload ứng dụng:** Do tham số backlog của socket được chốt tại thời điểm socket gọi hàm `listen()`, việc đổi `somaxconn` hoặc sửa file cấu hình không có tác dụng với tiến trình đang chạy. Bắt buộc phải restart/reload ứng dụng, sau đó kiểm tra lại bằng `ss -lnt` để đảm bảo cột `Send-Q` đã tăng lên giá trị mới.
  4. **Lưu ý quan trọng từ SRE:** Tăng kích thước Accept Queue chỉ là giải pháp đệm giúp chịu được lưu lượng tăng đột biến trong vài giây (Burst absorption). Nếu ứng dụng bị nghẽn CPU hoặc xử lý đồng bộ quá chậm, hàng đợi dù đặt 65535 cũng sẽ nhanh chóng bị lấp đầy. Giải pháp căn cơ là tối ưu tốc độ gọi hàm `accept()`, tăng số lượng worker process/threads hoặc chuyển sang mô hình xử lý non-blocking bất đồng bộ (`epoll`/`io_uring`).

---

## 8. HIỂU LẦM PHỔ BIẾN VÀ CÁC BẪY KỸ THUẬT (PITFALLS)

1. **Bẫy `ping` thành công nghĩa là mạng thông suốt:**
   * *Thực tế:* `ping` chỉ kiểm tra được tầng 3 (ICMP). Rất nhiều trường hợp ICMP đi thông nhưng Firewall/Security Group chặn port TCP 443 (L4), hoặc dịch vụ Web Server bị crash/deadlock (L7).
2. **Hiểu lầm `tcpdump` làm rớt gói tin trên mạng:**
   * *Thực tế:* `tcpdump` sử dụng cơ chế BPF gắn vào socket clone ở tầng kernel. Nó không bao giờ chặn hay drop gói tin thực tế đang đi tới ứng dụng. Tuy nhiên, nếu CPU quá tải hoặc buffer quá nhỏ, chính tiến trình `tcpdump` sẽ bị mất (drop) gói tin hiển thị (`dropped by kernel`), chứ không làm mất gói tin của ứng dụng.
3. **Nhầm lẫn giữa `Recv-Q` của trạng thái LISTEN và ESTABLISHED trên `ss`:**
   * *Hậu quả:* Thấy `Recv-Q` = 100 ở trạng thái `LISTEN` lại tưởng là có 100 bytes dữ liệu đang chờ đọc, trong khi thực tế đó là **100 kết nối người dùng đang bị nghẽn ở cửa sổ bắt tay**.
4. **Lạm dụng `traceroute` truyền thống:**
   * *Thực tế:* `traceroute` mặc định trên Linux dùng UDP, trên Windows dùng ICMP. Rất nhiều router biên trên Internet chặn UDP/ICMP nhưng cho phép TCP đi qua. Để debug web service, luôn ưu tiên dùng TCP SYN Traceroute: `sudo traceroute -T -p 443 <domain>`.

---

## 9. CÂU HỎI TỰ KIỂM TRA & PHỎNG VẤN SENIOR

### 9.1. Trắc nghiệm & Tự kiểm tra (Có đáp án)

1. **Lệnh `ss -lnt` hiển thị giá trị tại cột `Send-Q` là 512 cho một tiến trình HTTP. Con số 512 này đại diện cho điều gì?**
   * *Đáp án:* Đại diện cho kích thước tối đa của Accept Queue (Backlog limit) được thiết lập cho socket này, tính bằng min(somaxconn, backlog).

2. **Khi dùng `curl -w` đo kết nối HTTPS, chỉ số nào phản ánh độ trễ mạng (RTT) của tầng Transport (L4)?**
   * *Đáp án:* Hiệu số giữa `time_connect` và `time_namelookup` (xấp xỉ 1 RTT mạng: RTT xấp xỉ time_connect - time_namelookup).

3. **Tại sao khi chạy `tcpdump` trên hệ thống chịu tải cao, kỹ sư luôn phải kèm theo cờ `-n` hoặc `-nn`?**
   * *Đáp án:* Để tắt tính năng phân giải ngược IP/Port thành Hostname/Service name qua DNS, tránh gây quá tải DNS resolver và tránh làm nghẽn tiến trình bắt gói.

4. **Khi một server Linux nhận được gói tin TCP SYN gửi vào một port chưa có tiến trình nào `LISTEN`, kernel sẽ phản hồi gói tin gì?**
   * *Đáp án:* Kernel sẽ phản hồi gói tin mang cờ `RST, ACK` (TCP Reset).

5. **Sự khác biệt giữa bộ đếm `TcpExtListenOverflows` và `TcpExtListenDrops` trong lệnh `nstat` là gì?**
   * *Đáp án:* `TcpExtListenOverflows` đếm số lần kết nối bị từ chối cụ thể do Accept Queue bị tràn. `TcpExtListenDrops` là tập hợp rộng hơn, bao gồm tất cả các trường hợp kết nối bị drop tại socket lắng nghe (gồm cả tràn Accept Queue, không đủ bộ nhớ cấp phát socket, hoặc tràn SYN Queue khi không bật syncookies).

---

### 9.2. Câu hỏi Phỏng vấn Senior Platform / SRE

> **Câu hỏi 1 (Scenario-based):**
> *"Một dịch vụ backend trên production thỉnh thoảng ghi nhận lỗi `Connection reset by peer` từ client. Bạn sẽ thiết lập quy trình bắt gói `tcpdump` như thế nào để khoanh vùng sự cố an toàn trên hệ thống tải cao, và các bước phân tích tiếp theo trên Wireshark là gì?"*
>
> **Gợi ý trả lời mức Senior:**
> 1. *Quy trình bắt gói 2 giai đoạn an toàn:*
>    * **Giai đoạn 1 (Bắt gói RST để xác định 4-tuple và thời điểm):** Dùng snaplen nhỏ (`-s 128`), ghi xoay vòng an toàn để chỉ bắt các gói mang cờ RST:
>      ```bash
>      sudo tcpdump -nn -i any -s 128 -C 100 -W 3 -w /var/log/tcp_rst_only.pcap 'tcp port <PORT> and (tcp[tcpflags] & tcp-rst != 0)'
>      ```
>      Từ file này, xác định bên phát ra RST (Server hay Client) và lấy đúng cặp IP:Port (4-tuple) xảy ra lỗi.
>    * **Giai đoạn 2 (Bắt trọn vẹn luồng dữ liệu của kết nối mục tiêu):** Khi đã cô lập được IP nguồn/đích hoặc tái hiện được vấn đề, bắt luồng đầy đủ (đủ payload hoặc snaplen phù hợp) có ring buffer cho riêng luồng đó:
>      ```bash
>      sudo tcpdump -nn -i any -s 0 -C 200 -W 5 -w /var/log/tcp_full_stream.pcap 'host <CLIENT_IP> and port <PORT>'
>      ```
> 2. *Phân tích trên Wireshark:*
>    * Mở file `.pcap` của **Giai đoạn 2** (file chứa trọn vẹn luồng dữ liệu), tìm gói tin RST và chọn **Follow → TCP Stream** để quan sát toàn bộ chuỗi gói tin diễn ra ngay trước cờ RST.
>    * *Trường hợp 1:* RST do Server gửi ngay sau khi nhận dữ liệu từ Client => Thường do ứng dụng đóng socket (`close()`) khi trong Receive Buffer vẫn còn dữ liệu chưa đọc (`unread data`), hoặc tiến trình ứng dụng bị crash / OOM-killed đột ngột.
>    * *Trường hợp 2:* RST xuất hiện sau một khoảng thời gian dài không có gói tin nào truyền qua => Do **Idle Timeout** của Firewall / NAT Gateway / Load Balancer / conntrack nằm ở giữa tự động hủy bảng trạng thái và gửi RST khi có gói tin mới đến.

---

> **Câu hỏi 2 (System Internals):**
> *"Sự khác biệt căn bản giữa SYN Queue và Accept Queue trong Linux kernel là gì? Khi một trong hai hàng đợi này bị tràn, hành vi mặc định của kernel sẽ xử lý gói tin TCP tiếp theo như thế nào?"*
>
> **Gợi ý trả lời mức Senior:**
> 1. *SYN Queue (Incomplete Connection Queue):* Chứa các socket đang ở trạng thái nửa mở `SYN_RECV` (chờ gói ACK cuối cùng từ client để hoàn thành bắt tay 3 bước). Giới hạn quản lý bởi `net.ipv4.tcp_max_syn_backlog`.
> 2. *Accept Queue (Completed Connection Queue):* Chứa các socket đã hoàn tất 3-way handshake (`ESTABLISHED`) nhưng ứng dụng tầng user-space chưa gọi hàm `accept()` để lấy ra xử lý. Giới hạn quản lý bởi min(net.core.somaxconn, backlog trong code).
> 3. *Hành vi khi tràn hàng đợi:*
>    * *Khi tràn SYN Queue:* Nếu bật `net.ipv4.tcp_syncookies = 1`, kernel sẽ không lưu trạng thái vào SYN Queue mà mã hóa thông tin kết nối vào Initial Sequence Number (ISN) trong gói SYN-ACK gửi lại client. Nếu tắt SYN Cookies (`0`), gói SYN mới sẽ bị drop âm thầm.
>    * *Khi tràn Accept Queue:*
>      * **Với gói SYN mới:** Kernel drop gói SYN âm thầm; client không nhận được SYN-ACK và sẽ gửi lại SYN (SYN Retransmission) với backoff (~1s, 2s, 4s...).
>      * **Với gói ACK cuối cùng của 3-way handshake:** Mặc định (`tcp_abort_on_overflow = 0`), kernel sẽ bỏ qua gói ACK này (không chuyển socket sang trạng thái sẵn sàng cho app); server sẽ gửi lại gói SYN-ACK để cho app thêm thời gian giải phóng Accept Queue. Nếu cấu hình `tcp_abort_on_overflow = 1`, kernel sẽ lập tức gửi gói `RST` cho client để hủy kết nối.

---

## 10. TÓM TẮT & CHEAT SHEET LỆNH

### Tóm tắt cốt lõi trong 5 dòng:
1. Luôn dùng phương pháp **Divide-and-Conquer**, bắt đầu kiểm tra từ **Tầng 4 (Transport/Port)** trước khi đi sâu vào DNS/L7 hoặc L1–L3 (trừ trường hợp nghi vấn MTU Black Hole hoặc mất gói L3).
2. `ping` chỉ chứng minh tầng 3 (IP) thông suốt, không chứng minh được dịch vụ L4/L7 đang sẵn sàng hoạt động.
3. Trên lệnh `ss`, `Recv-Q` ở trạng thái `LISTEN` lớn hơn 0 là dấu hiệu **ứng dụng bị nghẽn không kịp gọi `accept()` (Accept Queue saturation)**.
4. Đo lường thời gian bằng `curl -w` giúp bóc tách chính xác độ trễ do DNS, do TCP Handshake, do TLS, hay do Backend xử lý code (TTFB).
5. Khi bắt gói bằng `tcpdump` trên production, bắt buộc phải dùng `-nn`, `-s <snaplen>`, và cơ chế xoay vòng `-C -W -w` để bảo vệ tài nguyên hệ thống.

---

### Cheat Sheet Lệnh Module 0:

```bash
# -----------------------------------------------------------------------------
# 1. KHẢO SÁT SOCKET VÀ HÀNG ĐỢI (L4)
# -----------------------------------------------------------------------------
ss -ltnp                             # Xem tất cả TCP port đang LISTEN kèm Process/PID
ss -tupn                             # Xem các socket TCP đã ESTABLISHED và UDP đã bind/kết nối
ss -s                                # Thống kê tổng số lượng socket theo trạng thái (estab, timewait)
ss -tnoa                             # Xem các socket TCP kèm thông tin Timer (keepalive, on, timewait)
nstat -az TcpExtListen*              # Xem bộ đếm tràn Accept Queue / SYN Drops

# -----------------------------------------------------------------------------
# 2. KHẢO SÁT HẠ TẦNG VÀ ĐỊNH TUYẾN (L2 - L3)
# -----------------------------------------------------------------------------
ip -br addr                          # Xem nhanh IP trên các interface
ip -br link                          # Xem nhanh trạng thái link L2 (UP/DOWN/NO-CARRIER)
ip route get <IP>                    # Kiểm tra route, gateway và interface được chọn tới IP đích
ip neigh                             # Xem bảng ARP/Neighbor (REACHABLE/STALE/FAILED)
ip -s link show dev <IFACE>          # Thống kê gói tin drop/error/missed trên card mạng

# -----------------------------------------------------------------------------
# 3. ĐO ĐẠC VÀ DEBUG DỊCH VỤ (L3 - L7)
# -----------------------------------------------------------------------------
nc -zvw3 <IP> <PORT>                 # Quét kiểm tra mở port TCP với timeout 3s
nc -zvuw3 <IP> <PORT>                # Quét kiểm tra port UDP (Lưu ý: UDP không đáng tin để kết luận 'mở')
sudo traceroute -T -p 443 <DOMAIN>   # Traceroute bằng TCP SYN qua port 443 (yêu cầu root)
mtr -r -c 100 -T -P 443 <IP>         # Báo cáo MTR 100 gói TCP SYN qua port 443
dig +short <DOMAIN>                  # Truy vấn nhanh địa chỉ IP từ DNS
dig +noall +answer <DOMAIN>          # Truy vấn DNS chỉ lấy phần trả lời và TTL
dig +trace <DOMAIN>                  # Lần vết truy vấn DNS từ 13 cụm Root Servers
dig @<DNS_IP> <DOMAIN> +tcp          # Truy vấn DNS chỉ định Server qua TCP
openssl s_client -connect <H>:<P> -servername <H> -alpn h2,http/1.1 -showcerts < /dev/null # Debug TLS/Cert Chain

# -----------------------------------------------------------------------------
# 4. BẮT GÓI TIN AN TOÀN TRÊN PRODUCTION (TCPDUMP)
# -----------------------------------------------------------------------------
# Bắt gói tin TCP SYN thuần túy (kết nối mới, SYN=1, ACK=0)
tcpdump -nn -i any 'tcp[tcpflags] & tcp-syn != 0 and tcp[tcpflags] & tcp-ack == 0'

# Bắt các gói tin RST trên cổng chỉ định
tcpdump -nn -i any 'tcp port 443 and tcp[tcpflags] & tcp-rst != 0'

# Bắt và lưu xoay vòng 3 file x 50.000.000 bytes, giới hạn 96 bytes/packet
tcpdump -nn -i eth0 -s 96 -C 50 -W 3 -w /tmp/dump.pcap 'tcp port 80 or tcp port 443'

# Bắt các gói tin DNS response trả về mã lỗi (rcode != 0) qua UDP
# Lưu ý: Timeout là sự vắng mặt của response nên không thể lọc bằng BPF; cần so sánh số lượng query/response hoặc log DNS resolver
tcpdump -nn -i any 'udp src port 53 and udp[10] & 0x80 != 0 and udp[11] & 0x0f != 0' -v
```

---

## 🔍 TỰ RÀ SOÁT KỸ THUẬT

* **Trạng thái thực thi và kiểm chứng lệnh:**
  1. *Thực thi:* Chưa chạy thử trên Linux; học viên cần tự chạy các lab và đối chiếu output.
  2. *Đánh dấu output mẫu:* Tất cả các khối lệnh thực hành và output mẫu trong Module 0 đã được gắn nhãn minh họa kỹ thuật `(Ví dụ minh họa, chưa chạy thử)` để đảm bảo tính trung thực tuyệt đối.
* **Các điểm phụ thuộc phiên bản / Kernel / Ứng dụng:**
  1. *Mặc định `net.core.somaxconn`:* Trên Linux kernel từ 5.4 trở đi (bao gồm Ubuntu 20.04/22.04, Debian 11/12, RHEL 9), giá trị mặc định là `4096`. Trên các kernel cũ trước 5.4, giá trị mặc định là `128`.
  2. *Mặc định backlog của ứng dụng:* Node.js (thông qua libuv fallback) thường mặc định là `511`, Nginx trên Linux mặc định là `511`, Gunicorn mặc định là `2048`. Backlog thực tế của socket luôn bị giới hạn bởi min(somaxconn, backlog).
  3. *Hành vi khi tràn Accept Queue (`tcp_abort_on_overflow`):* Mặc định là `0` (drop gói SYN mới, bỏ qua gói ACK cuối, đợi retransmit). Khi đặt thành `1`, kernel sẽ gửi gói `RST` phản hồi khi nhận gói ACK cuối.
  4. *Tham số snaplen `-s` của `tcpdump`:* Đặt `-s 96` hoặc `-s 128` chỉ đủ để chụp IP/TCP header. Để phân tích bản tin TLS ClientHello / SNI hoặc HTTP Header, cần snaplen lớn hơn nhiều (~1500 bytes hoặc `-s 0`).
  5. *Đơn vị cờ `-C` trong `tcpdump`:* Theo manpage chuẩn của `tcpdump`, cờ `-C` nhận đối số tính theo đơn vị triệu byte (10^6 = 1.000.000 bytes), không phải MiB (1.048.576 bytes).
  6. *Lệnh `ss` trên container tối giản (Alpine Linux):* [CẦN XÁC MINH] Trên Alpine Linux sử dụng BusyBox, lệnh `ss` tích hợp sẵn có thể thiếu một số tùy chọn nâng cao như `-p` (xem PID/Process); cần cài gói `iproute2` (`apk add iproute2`) để có đầy đủ tính năng.
  7. *Hybrid Post-Quantum TLS ClientHello:* [CẦN XÁC MINH] Các extension hậu lượng tử (như Kyber / ML-KEM) làm tăng đáng kể kích thước ClientHello, có thể khiến gói tin bị phân mảnh hoặc drop trên các thiết bị mạng cũ không hỗ trợ gói lớn.
