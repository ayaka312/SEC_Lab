# SEC_Lab
## 1. Kiến trúc Tổng thể (Architecture Overview)
**Hệ thống được thiết kế theo mô hình Defense-in-Depth, tách biệt giám sát thành 2 tầng lớp rõ ràng:**
 ``` 
1.	Tầng Host/Node Security (Wazuh Agent): Chịu trách nhiệm bảo vệ hệ điều hành nền tảng (OS), giám sát các tiến trình hệ thống, đăng nhập SSH và toàn vẹn file hệ thống (FIM).
2.	Tầng Workload/Container Security (Falco): Chịu trách nhiệm giám sát runtime của các container, phát hiện các hành vi bất thường bên trong Pod và audit cấu hình Kubernetes.
 ```
### Architecture – Current Implementation
<ASCII sơ đồ hiện tại>

                         +-----------------------------+
                         |     Kubernetes Cluster      |
                         |                             |
                         |   +---------------------+   |
                         |   |   Falco DaemonSet   |   |
                         |   |  (privileged, eBPF) |   |
                         |   +----------+----------+   |
                         |              |              |
                         |              v              |
                         |        Falcosidekick        |
                         |              |              |
                         |           Email Alert       |
                         |                             |
                         +-----------------------------+
                                       ^
                                       |
                     +-------------------------------------+
                     | (Container Syscalls: exec, net)     |
                     +-------------------------------------+
                                       ^
                                       |
          +-----------------------------------------------------------+
          |                       Node VM (Host)                      |
          |                                                           |
          |   +---------+                                             |
          |   | Auditd  |  (kernel syscall audit)                     |
          |   +----+----+                                             |
          |        |                                                  |
          |        v                                                  |
          |   /var/log/audit/audit.log                                |
          |        |                                                  |
          |        v                                                  |
          |   +----------------+        TCP             +------------+|
          |   |  Wazuh Agent   | -------------------->  | Wazuh Mgr  ||
          |   | (root, systemd)|                        |  + Index   ||
          |   +----------------+                        +------------+|
          |                                                           |
          +-----------------------------------------------------------+


### Sơ đồ vị trí Agent:
**Wazuh Agent:**
 ``` 
  •	Vị trí: Cài trực tiếp trên Kubernetes Node (VM).
  •	Loại hình: Systemd Service (Process nền của Linux).
  •	Quyền hạn: Chạy với quyền root để có thể đọc auditd logs và quét toàn bộ filesystem.
  •	Vai trò chính: Thu thập logs từ auditd (syscalls), Syslog, và quét FIM (File Integrity).
```
**Falco Agent:**
```
  •	Vị trí: Triển khai dạng DaemonSet trên Kubernetes Cluster (mỗi Node một Pod).  
  •	Loại hình: Pod/Container.  
  •	Quyền hạn: Chạy privileged: true để load kernel module/eBPF probe nhằm bắt syscalls của các container khác.  
  •	Vai trò chính: Bắt syscalls container qua kernel module/eBPF, tích hợp K8s Metadata.
```
## 2. Chi tiết Triển khai & Cấu hình (Implementation Details)
### A. Giám sát Node VM (Wazuh + Auditd)
**Để đáp ứng yêu cầu giám sát runtime ở mức OS,  cấu hình tích hợp sâu giữa Wazuh và Linux Auditd (auditd).**
#### 1. Process Execution Monitoring:
```
Sử dụng auditd để bắt syscall execve và execveat. Cấu hình phân loại rõ ràng:
•	Root Activity: Giám sát mọi lệnh chạy bởi root (euid=0) -> Key: wazuh_exec_root.
•	User Activity: Giám sát lệnh chạy bởi user thực (auid>=1000) -> Key: wazuh_exec_user.
•	Mục tiêu: Phát hiện hacker leo thang đặc quyền hoặc chạy mã độc trực tiếp trên Host.
```
#### 2. Network Activity Monitoring:
```
Giám sát các kết nối mạng chiều đi (Outbound) thông qua syscall connect.
•	Scope: Giám sát cả Root và User Interactive.
•	Mục tiêu: Phát hiện C2 Beaconing hoặc hành vi tải malware từ internet về node (curl, wget).
```
#### 3. File Integrity & Sensitive Access:
```
Sử dụng mô hình lai (Hybrid):
•	Real-time Alert (Auditd): Đặt watch (-w) lên các file nhạy cảm như /etc/shadow, /etc/passwd, /etc/ssh/sshd_config. Bất kỳ hành vi đọc/ghi nào đều sinh log ngay lập tức.
•	Content Change (Wazuh FIM): Cấu hình syscheck quét realtime các thư mục quan trọng (/etc/containerd, /var/lib/kubelet/pki) để phát hiện file bị thay đổi nội dung hoặc file lạ mới sinh ra.
```
### B. Giám sát Workload Container (Falco)
Falco được cấu hình sử dụng kết hợp 2 bộ rule:
- Default Rules: Bộ rule chuẩn của cộng đồng (phát hiện shell, crypto mining, container escape cơ bản).
- Custom Rules (tragtt-runtime-rules.yaml): Các rule tự phát triển để giảm nhiễu (False Positive) và tập trung vào các kịch bản cụ thể như (một vài case demo và có thể phát triển them ở rule custom này):

  •	Spawn interactive shell trong namespace Production.
  
  •	Truy cập Cloud Metadata IP (169.254.169.254).
  
  •	Sử dụng công cụ mạng (netcat, nmap) trong container.

## 3. Luồng dữ liệu End-to-End (Data Flow)

**Hệ thống xử lý hai luồng dữ liệu song song:**
##### Luồng 1: Node VM Monitoring (Wazuh)
```
1.	Thu thập: Wazuh Agent đọc dữ liệu từ audit.log (Kernel Audit), syslog, và quét file thay đổi (FIM)…
2.	Chuyển tiếp: Agent mã hóa dữ liệu và gửi qua TCP port 1514 về Wazuh Manager (Control Plane).
3.	Phân tích: Wazuh Manager giải mã (Decoder) và đối chiếu với Rule.
4.	Lưu trữ & Hiển thị: Cảnh báo được lưu vào Indexer (Elasticsearch/OpenSearch) và hiển thị trên Wazuh Dashboard.
```
##### Luồng 2: Workload & Runtime Security (Falco)
```
1.	Thu thập:
  •	Syscall: Falco Driver (eBPF) bắt các lệnh gọi hệ thống từ Container.
  •	K8s Audit: k8s-metacollector thu thập metadata và logs từ K8s API Server.
2.	Phân tích: Falco Engine so sánh sự kiện với file Rule (default + custom).
3.	Xử lý: Nếu vi phạm, Falco đẩy log JSON sang Falcosidekick.
4.	Báo cáo: Falcosidekick format lại bản tin và gửi cảnh báo qua SMTP (Gmail) tới quản trị viên.
```
## 4. Quyết định thiết kế chính (Design Decisions)
```
•	Tại sao dùng Falco cho Container? Falco thấu hiểu ngữ cảnh Kubernetes (Namespace, Pod Name, Image) tốt hơn Wazuh. Wazuh chỉ nhìn thấy process ID trên host mà không biết process đó thuộc Pod nào.
•	Tại sao dùng Wazuh cho Node? Falco tập trung vào syscall stream, không mạnh về File Integrity Monitoring (FIM) hay log analysis của OS (như SSH login failure) bằng Wazuh. Kết hợp cả hai lấp đầy các điểm mù.
•	Kênh cảnh báo (Alerting): Sử dụng Email cho các cảnh báo runtime từ Falco để đảm bảo tính tức thời (Real-time).
```
## 5. Phân tích mở rộng & Điểm nghẽn (Scalability & Bottlenecks)
**Nếu hệ thống mở rộng lên hàng trăm nodes hoặc triển khai Multi-tenant, các vấn đề sau sẽ phát sinh:**
```
1.	Nút thắt cổ chai tại Control Plane (Bottleneck):
  •	Wazuh Manager: Là điểm tập trung duy nhất. Nếu quá nhiều Agent gửi log cùng lúc, Manager sẽ bị quá tải (CPU/Network). -> Giải pháp: Cần triển khai Wazuh Cluster (Master/Worker).
  •	K8s Metacollector: Khi cluster quá lớn, lượng event từ K8s API Server rất khủng khiếp, Pod này có thể bị OOM (Out Of Memory).
2.	Vấn đề Multi-tenant (Nhiều team dùng chung):
  •	Hiện tại Falco Rules áp dụng cho toàn bộ Cluster. Khó để team A có bộ luật riêng khác team B.
  •	Cảnh báo Email đang gửi chung vào một hộp thư. Cần định tuyến (Routing) alert dựa trên Namespace để gửi đúng cho team phụ trách.
3.	Hiệu năng Node:
  •	Falco chạy eBPF/Kernel module có thể chiếm 1-3% CPU. Nếu lưu lượng syscall quá lớn (ví dụ Database High-load), Falco có thể làm chậm ứng dụng (drop syscalls).
```
## 6. Future Upgrade

### Data Flow Explanation (Current)
Falco giám sát container runtime và sinh alert độc lập

Wazuh giám sát node VM runtime và có control plane riêng

❌ Hai pipeline song song, không correlation node ↔ container

### Architecture – Target Design
Falco = Runtime Security Control Plane duy nhất
Wazuh = Node-level security sensor

<ASCII sơ đồ nâng cấp>

                    +--------------------------------------+
                    |        Kubernetes Cluster            |
                    |                                      |
                    |   +------------------------------+   |
                    |   |     Falco Control Plane      |   |
                    |   |                              |   |
                    |   |  - Falco Engine              |   |
                    |   |  - Runtime Rules             |   |
                    |   |  - K8s Audit / Config Rules  |   |
                    |   +--------------+---------------+   |
                    |                  |                   |
                    |           Falcosidekick              |
                    |                  |                   |
                    |        Alert / Report / SIEM         |
                    +--------------------------------------+
          
                        ^                               ^
                        |                               |
             (Container Runtime Events)       (Node Runtime Events)
          
              +--------------------------+      +--------------------------+
              |     Falco DaemonSet      |      |      Wazuh Agent         |
              |  (per-node, privileged)  |      |   (systemd, root)        |
              +------------+-------------+      +------------+-------------+
                       |                                   |
                       v                                   v
                 +----------------------+              +---------+
                 |  Kernel Syscalls     |              | Auditd  |
                 | (exec, file, net)    |              | (kernel)|
                 +----------------------+              +---------+
                                                       

### Data Flow Explanation
#### Luồng A – Container Runtime
```
Container

  → Kernel syscall
  
    → Falco DaemonSet
    
      → Falco Control Plane
      
        → Falcosidekick
        
          → Alert / Report
```

- Phát hiện: shell spawn, network bất thường, file access,...
- Có đầy đủ metadata: Pod, Namespace, Image


#### Luồng B – Node VM Runtime

```
OS Process / File / Network

  → Auditd
  
    → Wazuh Agent (chuẩn hoá JSON)
    
      → Falco Control Plane
      
        → Falcosidekick
        
          → Alert / Report
```
- Wazuh không còn là control plane
- Chỉ đóng vai trò node security sensor

