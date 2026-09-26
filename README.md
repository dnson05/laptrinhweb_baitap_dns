
# Bài tập 1: Triển khai hệ thống Docker Compose với Nginx, Node-RED, MariaDB, phpMyAdmin và Cloudflare Tunnel

## Họ và tên: Đàm Ngọc Sơn
## Lớp: K59.KMT.K01
## MSSV: K235480106061

## Mục tiêu

- Giả lập môi trường Linux OS trên máy Windows.
- Cài đặt Docker và Docker Compose.
- Triển khai 5 dịch vụ trong Docker Compose: **Nginx**, **Node-RED**, **MariaDB**, **phpMyAdmin**, **Cloudflared**.
- Cấu hình Nginx chạy đồng thời 2 website với 2 domain (subdomain) khác nhau.
- Public 2 website ra Internet thông qua Cloudflare Tunnel với domain thật.

## Thông tin môi trường

| Thành phần | Giá trị |
|---|---|
| Hệ điều hành giả lập | WSL2 – Ubuntu 22.04 |
| Domain sử dụng | `damngocson.id.vn` |
| Subdomain 1 | `site1.damngocson.id.vn` → trang HTML tĩnh + phpMyAdmin |
| Subdomain 2 | `site2.damngocson.id.vn` → Node-RED |
| Tên tunnel Cloudflare | `sonlab` |

---

## 1. Giả lập Linux OS bằng WSL2

### 1.1. Cài đặt WSL2 + Ubuntu

Mở **PowerShell (Run as Administrator)** trên Windows:

```powershell
wsl --install -d Ubuntu-22.04
```

### 1.2. Xử lý lỗi thường gặp: `WslRegisterDistribution failed with error: 0x80370114`

Lỗi này do thiếu tính năng ảo hóa. Khắc phục:

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

Kiểm tra ảo hóa đã bật trong BIOS/UEFI:

```powershell
systeminfo
```

→ Tìm dòng **"Virtualization Enabled In Firmware"** phải là **Yes**. Nếu **No**, vào BIOS bật **Intel VT-x** hoặc **AMD-V / SVM Mode**.

Sau đó khởi động lại máy và cài lại:

```powershell
wsl --update
wsl --set-default-version 2
wsl --install -d Ubuntu-22.04
```

### 1.3. Khởi tạo user Linux lần đầu

Sau khi cài xong, mở app **Ubuntu** từ Start Menu, nhập:

```
Enter new UNIX username: dnson05
New password: ********
Retype new password: ********
```

> <img width="1487" height="760" alt="image" src="https://github.com/user-attachments/assets/1c536a45-4bbc-4aae-bb24-5c1bc8ab2d12" />

---

## 2. Cài đặt Docker và Docker Compose

Chạy trong terminal Ubuntu (WSL):

```bash
sudo apt update && sudo apt upgrade -y

sudo apt remove docker docker-engine docker.io containerd runc -y

sudo apt install -y ca-certificates curl gnupg lsb-release

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker $USER
newgrp docker

docker --version
docker compose version
```

> <img width="1483" height="757" alt="image" src="https://github.com/user-attachments/assets/d9b96583-7e1e-4b51-8432-a2870fa3fe64" />

---

## 3. Cài đặt cloudflared và thiết lập Cloudflare Tunnel

### 3.1. Cài cloudflared

```bash
cd ~
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb -o cloudflared.deb
sudo dpkg -i cloudflared.deb
cloudflared --version
```
### 3.2. Đăng nhập Cloudflare

```bash
cloudflared tunnel login
```

> <img width="1917" height="1028" alt="image" src="https://github.com/user-attachments/assets/39a509b6-e7f3-4735-8eb6-e848565b0839" />


### 3.3. Tạo tunnel

```bash
cloudflared tunnel create sonlab
```

Kết quả trả về UUID của tunnel

```
Created tunnel sonlab with id 62a949f4-1857-411d-beda-ad5706d3c62a
```

Kiểm tra danh sách tunnel:

```bash
cloudflared tunnel list
```
> <img width="1480" height="762" alt="image" src="https://github.com/user-attachments/assets/47311b86-9b69-4105-8c43-0f99f0b07ce1" />

### 3.4. Trỏ DNS cho 2 subdomain về tunnel

```bash
cloudflared tunnel route dns sonlab site1.damngocson.id.vn
cloudflared tunnel route dns sonlab site2.damngocson.id.vn
```


> <img width="1917" height="1032" alt="image" src="https://github.com/user-attachments/assets/6e3047bc-a1e9-4f73-a716-b08708e5db32" />

---

## 4. Chuẩn bị cấu trúc thư mục project

```bash
mkdir -p ~/lab1/{nginx/conf.d,nginx/site1,nginx/site2,cloudflared}
cd ~/lab1


cp ~/.cloudflared/62a949f4-1857-411d-beda-ad5706d3c62a.json ~/lab1/cloudflared/
cp ~/.cloudflared/config.yml ~/lab1/cloudflared/
cp ~/.cloudflared/cert.pem ~/lab1/cloudflared/


chmod 644 ~/lab1/cloudflared/*
```
---

## 5. File cấu hình Cloudflare Tunnel — `cloudflared/config.yml`

```yaml
tunnel: sonlab
credentials-file: /etc/cloudflared/62a949f4-1857-411d-beda-ad5706d3c62a.json

ingress:
  - hostname: site1.damngocson.id.vn
    service: http://nginx:80
  - hostname: site2.damngocson.id.vn
    service: http://nginx:80
  - service: http_status:404
```

---

## 6. File `docker-compose.yml`

```yaml
services:
  nginx:
    image: nginx:latest
    container_name: nginx
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/site1:/usr/share/nginx/site1:ro
      - ./nginx/site2:/usr/share/nginx/site2:ro
    depends_on:
      - nodered
      - phpmyadmin
    networks:
      - labnet

  nodered:
    image: nodered/node-red:latest
    container_name: nodered
    restart: unless-stopped
    ports:
      - "1880:1880"
    volumes:
      - nodered_data:/data
    networks:
      - labnet

  mariadb:
    image: mariadb:10.11
    container_name: mariadb
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: rootpass123
      MYSQL_DATABASE: labdb
      MYSQL_USER: labuser
      MYSQL_PASSWORD: labpass123
    volumes:
      - mariadb_data:/var/lib/mysql
    networks:
      - labnet

  phpmyadmin:
    image: phpmyadmin:latest
    container_name: phpmyadmin
    restart: unless-stopped
    environment:
      PMA_HOST: mariadb
      PMA_USER: root
      PMA_PASSWORD: rootpass123
    depends_on:
      - mariadb
    networks:
      - labnet

  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: unless-stopped
    command: tunnel --config /etc/cloudflared/config.yml run
    volumes:
      - ./cloudflared:/etc/cloudflared
    depends_on:
      - nginx
    networks:
      - labnet

networks:
  labnet:
    driver: bridge

volumes:
  nodered_data:
  mariadb_data:
```

---

## 7. Cấu hình Nginx cho 2 website với 2 domain khác nhau

### 7.1. `nginx/conf.d/site1.conf` — trang tĩnh + phpMyAdmin

```nginx
server {
    listen 80;
    server_name site1.damngocson.id.vn;

    location / {
        root /usr/share/nginx/site1;
        index index.html;
        try_files $uri $uri/ =404;
    }

    location /phpmyadmin/ {
        proxy_pass http://phpmyadmin:80/;
        proxy_set_header Host $host;
    }
}
```

### 7.2. `nginx/conf.d/site2.conf` — reverse proxy sang Node-RED

```nginx
server {
    listen 80;
    server_name site2.damngocson.id.vn;

    location / {
        proxy_pass http://nodered:1880;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### 7.3. Trang test cho site1

```bash
echo "<h1>Site 1 - damngocson.id.vn</h1>" > ~/lab1/nginx/site1/index.html
```

---

## 8. Khởi động toàn bộ hệ thống

```bash
cd ~/lab1
docker compose up -d
```

Kiểm tra trạng thái các container:

```bash
docker compose ps
```

> <img width="1487" height="761" alt="image" src="https://github.com/user-attachments/assets/db7229ea-c9a6-4a08-9720-549a5d6a80a8" />


Kiểm tra log cloudflared (xác nhận tunnel kết nối thành công):

```bash
docker compose logs cloudflared
```


> <img width="1917" height="1078" alt="image" src="https://github.com/user-attachments/assets/5e771eb8-8319-40a7-b8b8-ad8f08ce8dfb" />


Kiểm tra log nginx (đảm bảo không lỗi cú pháp):

```bash
docker compose logs nginx
```

---

## 9. Xử lý các lỗi đã gặp trong quá trình làm

| Lỗi | Nguyên nhân | Cách khắc phục |
|---|---|---|
| `WslRegisterDistribution failed 0x80370114` | Thiếu tính năng ảo hóa Windows / BIOS chưa bật VT-x | Bật `Windows Subsystem for Linux`, `Virtual Machine Platform` qua DISM/Optional Features, bật Virtualization trong BIOS |
| `cloudflared: command not found` | Chưa cài cloudflared trong WSL Ubuntu | Tải file `.deb` và cài bằng `dpkg -i` |
| `cannot access archive 'cloudflared.deb': No such file or directory` | Đang đứng ở thư mục `/mnt/c/WINDOWS/system32` thay vì thư mục Linux | `cd ~` trước khi tải và cài |
| `cp: cannot create regular file ... Permission denied` | Thư mục đích bị lỗi quyền do lệnh trước bị ngắt giữa chừng | `sudo chown -R $USER:$USER ~/lab1` rồi copy lại |
| `Cannot determine default origin certificate path` | Chưa mount `cert.pem` vào container cloudflared | Copy thêm `cert.pem` từ `~/.cloudflared/` vào `~/lab1/cloudflared/` |
| `Can't read origin cert from /etc/cloudflared/cert.pem` | File `cert.pem`, `config.yml`, file `.json` bị quyền đọc quá chặt | `chmod 644` cho các file cấu hình cloudflared |

---

## 10. Kết quả kiểm thử

- [ ] `docker compose ps` — cả 5 container đều **Up**
- [ ] Truy cập `https://site1.damngocson.id.vn` → hiển thị trang tĩnh
- [ ] Truy cập `https://site2.damngocson.id.vn` → hiển thị giao diện Node-RED
- [ ] Truy cập `https://site1.damngocson.id.vn/phpmyadmin/` → đăng nhập thành công bằng `root` / `rootpass123`, thấy database `labdb`

Hiển thị trang tĩnh
> <img width="1917" height="1030" alt="image" src="https://github.com/user-attachments/assets/2650a520-705e-42d7-be9f-c0d82c23554b" />
hiển thị giao diện Node-RED
> <img width="1917" height="1027" alt="image" src="https://github.com/user-attachments/assets/d67da9fb-f7e3-4d92-8788-798e0090e353" />
> <img width="1917" height="1031" alt="image" src="https://github.com/user-attachments/assets/dc5ec292-b58c-4b41-923a-f5a0fee6f5c3" />
> 

---

## 11. Sơ đồ luồng dữ liệu tổng quan

```
Người dùng
   │
   ▼
Cloudflare (DNS + Tunnel)
   │
   ▼
Container: cloudflared
   │
   ▼
Container: nginx  ──── phân biệt theo server_name ────┐
   │                                                    │
   ▼ (site1)                                            ▼ (site2)
Trang HTML tĩnh + proxy /phpmyadmin/ → phpMyAdmin      Node-RED
                                             │
                                             ▼
                                          MariaDB
```

---

## 12. Kết luận

Bài tập đã hoàn thành đầy đủ các yêu cầu:

1. ✅ Giả lập Linux OS bằng WSL2.
2. ✅ Cài Docker + Docker Compose.
3. ✅ Triển khai 5 dịch vụ trong Docker Compose: Nginx, Node-RED, MariaDB, phpMyAdmin, Cloudflared.
4. ✅ Cấu hình Nginx chạy 2 website với 2 domain khác nhau (`site1.damngocson.id.vn` và `site2.damngocson.id.vn`), publish ra Internet qua Cloudflare Tunnel với domain thật `damngocson.id.vn`.
