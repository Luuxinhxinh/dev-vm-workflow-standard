# DECENTRALIZED DEV-VM WORKFLOW STANDARD
> **Chuẩn Quy trình Phát triển & Kiểm thử Dự án Hệ thống Phân tán (Local VM + Git + Remote-SSH)**  
> *Áp dụng tối ưu cho PNet v8, EVE-NG và các dự án ảo hóa / Linux kernel không phụ thuộc server tập trung (Proxmox/Cloud).*

---

## 🎯 1. Bối cảnh & Mục tiêu

Khi phát triển các dự án có can thiệp sâu vào tầng hệ điều hành, kernel KVM, ảo hóa mạng (như **PNET, EVE-NG, Cyber Range**), việc lập trình trực tiếp trên máy Host (Windows/macOS) là **bất khả thi**. 

Tuy nhiên, nếu **không có máy chủ ảo hóa dùng chung (như Proxmox VE)** và các thành viên **làm việc từ xa tại nhà (không chung mạng LAN)**, mô hình này giải quyết triệt để 3 bài toán:
1. **Không code trực tiếp ở máy Host**: Toàn bộ runtime, web server, privilege daemon, KVM module đều chạy trong máy ảo Linux cục bộ của mỗi người.
2. **Không bị lệch nhịp môi trường**: 100% thành viên dùng chung cấu hình Golden Base VM và quy chuẩn đồng bộ Git.
3. **Phát triển mượt mà, test được ngay**: Code qua VS Code Remote-SSH (gõ phím ở Host, lưu và chạy trực tiếp trên file system của VM).

---

## 🏗️ 2. Mô hình Kiến trúc Phân tầng (Architecture)

```text
┌─────────────────────────────────────────────────────────────┐
│                 MÁY HOST (WINDOWS / MACOS)                  │
│                                                             │
│   [ VS Code / Cursor / IDE ]       [ Trình duyệt Web / UI ] │
│        │ (Remote-SSH)                         ▲             │
│        │                                      │ (HTTP:80)   │
└────────┼──────────────────────────────────────┼─────────────┘
         │                                      │
         ▼ (Port 22 SSH)                        │
┌───────────────────────────────────────────────┴─────────────┐
│              MÁY ẢO CỤC BỘ (LOCAL DEV VM)                   │
│         VMware Workstation / VirtualBox (Ubuntu)            │
│                                                             │
│   • Thư mục mã nguồn: /opt/unetlab/                         │
│   • Web Stack: Apache / Nginx + PHP                         │
│   • Daemon đặc quyền: pnetlab-brokerd.py (Python root)      │
│   • Ảo hóa: KVM / QEMU / Docker (Nested Virtualization)     │
│   • Local Git: Track branch, commit, push, pull             │
└─────────────────────────────────────────────────────────────┘
                               ▲
                               │ git push / git pull
                               ▼
        ┌─────────────────────────────────────────────┐
        │       GITHUB / GITLAB CLOUD REPOSITORY      │
        │    https://github.com/Luuxinhxinh/pnet-v8   │
        └─────────────────────────────────────────────────────┘
```

---

## 📋 3. Hướng dẫn Setup Chi tiết Từng Bước

### Bước 1: Khởi tạo Máy ảo Cơ sở (Golden Base VM)
> *Chỉ cần 1 người trong nhóm thực hiện và xuất file `.ova` chia sẻ cho cả nhóm.*

1. **Cài đặt phần mềm ảo hóa**: Cài VMware Workstation Pro / Player hoặc VirtualBox trên máy Host.
2. **Tạo máy ảo Ubuntu Server**:
   * Phiên bản khuyến nghị: **Ubuntu Server 20.04 LTS** hoặc **22.04 LTS** (đối với PNet v8) hoặc bản theo tài liệu đặc tả.
   * Cấu hình tối thiểu: 4 vCPU, 8GB RAM, 50GB Ổ đĩa (SSD).
3. **BẬT NESTED VIRTUALIZATION (Bắt buộc)**:
   * **VMware**: Vào `Virtual Machine Settings` -> `Processors` -> Tích chọn:
     * `Virtualize Intel VT-x/EPT or AMD-V/RVI`
     * `Virtualize IOMMU (IO memory management unit)`
   * **VirtualBox**: Cài đặt -> Hệ thống -> Bộ xử lý -> Bật `Enable Nested VT-x/AMD-V`.
4. **Cấu hình Card Mạng (Network Adapter)**:
   * Chọn chế độ **NAT** (có port forwarding) hoặc **Bridged** (nếu mạng nhà cho phép cấp IP nội bộ trực tiếp).
   * Đặt Port Forwarding trên VMware/VirtualBox (nếu dùng NAT):
     * Host Port `2222` -> Guest Port `22` (SSH)
     * Host Port `8080` -> Guest Port `80` (Web UI PNet/EVE)

---

### Bước 2: Setup Môi trường PNet/EVE bên trong Máy ảo
Mở terminal máy ảo hoặc SSH vào, chạy script bootstrap chuẩn hóa:

```bash
# Cập nhật và cài đặt các công cụ cơ bản
sudo apt update && sudo apt install -y git curl wget rsync net-tools python3-pip

# Clone mã nguồn dự án vào đúng thư mục hệ thống
sudo mkdir -p /opt/unetlab
cd /opt/unetlab
sudo git clone git@github.com:Luuxinhxinh/pnet-v8.git .

# Thiết lập quyền hạn thực thi đúng chuẩn www-data & root
sudo chown -R www-data:www-data /opt/unetlab/html
sudo chmod -R 775 /opt/unetlab/html
```

---

### Bước 3: Cấu hình VS Code Remote - SSH trên Máy Host
Để code trên máy Host mượt mà như file nội bộ:

1. Trên máy Host, cài đặt Extension: **Remote - SSH** (`ms-vscode-remote.remote-ssh`) trong VS Code.
2. Mở file cấu hình SSH trên Host (`~/.ssh/config`):
   ```ssh
   Host pnet-dev-vm
       HostName 127.0.0.1
       Port 2222
       User root
       IdentityFile ~/.ssh/id_rsa
   ```
   *(Nếu dùng IP mạng nội bộ trực tiếp thì thay `HostName` bằng IP máy ảo, Port `22`).*
3. Bấm góc dưới bên trái VS Code -> Chọn **Connect to Host...** -> Chọn `pnet-dev-vm`.
4. Mở thư mục dự án: `/opt/unetlab`.
5. Bạn có thể mở Terminal ngay trong VS Code: terminal này đang chạy trực tiếp trên Linux của máy ảo.

---

## 🔄 4. Quy trình Vận hành Git Hàng Ngày (Không lệch nhịp)

### 4.1. Nguyên tắc cốt lõi
* **Không commit các file nặng**: File lab `.unl`, image QEMU/IOL `.qcow2`, file session, temporary logs tuyệt đối phải nằm trong `.gitignore`.
* **Mỗi tính năng một nhánh riêng**: Nhánh `main` luôn là bản ổn định nhất.

### 4.2. Vòng lặp phát triển chuẩn (Development Loop)

#### Đầu ngày / Trước khi code:
Kéo cập nhật mới nhất từ đồng đội về máy ảo:
```bash
cd /opt/unetlab
sudo ./scripts/sync-env.sh
```

#### Trong quá trình code:
1. Tạo branch tính năng mới:
   ```bash
   git checkout -b feature/toi-uu-broker
   ```
2. Chỉnh sửa code trên VS Code (Auto-save vào file system của máy ảo).
3. Thử nghiệm ngay trên trình duyệt máy Host:
   * Truy cập `http://localhost:8080` (hoặc `http://<IP-máy-ảo>`).
   * Xem log trực tiếp trong terminal máy ảo:
     ```bash
     sudo journalctl -u pnetlab-brokerd -f
     ```

#### Khi hoàn thành tính năng:
```bash
git add .
git commit -m "feat(broker): bo sung verb quan ly node"
git push origin feature/toi-uu-broker
```
Tạo Pull Request trên GitHub để các thành viên khác review trước khi merge vào `main`.

---

## 🌐 5. Phối hợp & Test chéo Từ Xa (Remote Pair-Testing)

Khi làm việc tại nhà khác mạng, muốn đồng đội truy cập máy ảo của mình để test thử:

### Cách 1: Sử dụng Tailscale (Miễn phí, Bảo mật Mesh VPN)
1. Cài Tailscale lên máy ảo của mỗi người:
   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   sudo tailscale up
   ```
2. Mỗi máy ảo sẽ có 1 IP cố định trong mạng riêng (ví dụ `100.80.90.10`).
3. Gửi IP này cho bạn cùng nhóm, bạn đó có thể mở trình duyệt vào xem và thao tác thử lab trực tiếp.

### Cách 2: Chia sẻ nhanh qua Cloudflare Tunnel
1. Cài đặt `cloudflared` trên máy ảo:
   ```bash
   sudo apt install -y cloudflared
   cloudflared tunnel --url http://localhost:80
   ```
2. Terminal sẽ cấp một đường link public tạm thời (dạng `https://xyz.trycloudflare.com`). Gửi link cho team để test tức thì.

---

## 🛠️ 6. Scripts Hỗ trợ Có Sẵn trong Repo này

* `scripts/sync-env.sh`: Đồng bộ code Git, cài dependencies mới, chạy migrations và restart daemons tự động.
* `scripts/setup-dev-vm.sh`: Script 1-click khởi tạo toàn bộ thư viện cần thiết trên một máy ảo Ubuntu mới.
* `.gitignore.template`: Mẫu file loại trừ cho các dự án ảo hóa PNet / EVE-NG.
