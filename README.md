# MÔN HỌC LẬP TRÌNH WEB
## Họ và tên: Đàm Ngọc Sơn
## Lớp: K59.KMT.K01
## MSSV: K235480106061



# Bài tập 1: Triển khai hệ thống Docker Compose với Nginx, Node-RED, MariaDB, phpMyAdmin và Cloudflare Tunnel

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

# Bài tập 2: Xây dựng API bằng Node-RED và gọi API bằng JavaScript qua Nginx
 
## Mục tiêu
 
1. Sử dụng Node-RED (node `http in` + `http response`) để tạo một API đơn giản trả về dữ liệu dạng JSON.
2. Cấu hình Nginx để website (dùng JavaScript) gọi được API trên qua domain thật, kèm thuật toán xử lý dữ liệu tự nghĩ.
3. Viết JavaScript trong trang HTML để gọi API và hiển thị dữ liệu.
## Thông tin API
 
| Thành phần | Giá trị |
|---|---|
| Domain sử dụng | `damngocson.id.vn` |
| Endpoint API | `https://site1.damngocson.id.vn/api/dssv` |
| Phương thức | `GET` |
| Định dạng trả về | JSON |
---
 
## 1. Tạo API bằng Node-RED
 
### 1.1. Sơ đồ flow
 
```
[http in]  --->  [function]  --->  [http response]
GET /api/dssv    Xử lý dữ liệu     Trả về JSON
```
> <img width="1917" height="1028" alt="image" src="https://github.com/user-attachments/assets/148ce6b2-9e10-46b0-b33c-5f7ff1d26689" />

### 1.2. Cấu hình node `http in`
 
- **Method**: `GET`
- **URL**: `/api/dssv`
### 1.3. Cấu hình node `function` — thuật toán tự nghĩ
 
Thuật toán áp dụng:
- **Sắp xếp giảm dần** danh sách sinh viên theo số tiền (`money`).
- **Phân loại trạng thái**: nếu `money >= 300000` → `"Du dieu kien"`, ngược lại → `"Chua du"`.
```javascript
// Du lieu mau danh sach sinh vien
const dssv = [
    { name: "Dam Ngoc Son", money: 500000 },
    { name: "Truong Van Hai", money: 550000 },
    { name: "Pham Thanh Son", money: 700000 },
    { name: "Nguyen Van An", money: 350000 },
    { name: "Hoang Dinh Diep", money: 250000 }
];
 
// Thuat toan tu nghi: sap xep giam dan theo money
dssv.sort((a, b) => b.money - a.money);
 
// Thuat toan tu nghi: danh dau trang thai theo dieu kien money >= 300000
const dssv_final = dssv.map(sv => ({
    ...sv,
    status: sv.money >= 300000 ? "Du dieu kien" : "Chua du"
}));
 
msg.payload = {
    ok: 1,
    msg: "thanh cong",
    dssv: dssv_final
};
 
// Bat buoc set Content-Type de trinh duyet hieu day la JSON
msg.headers = { "Content-Type": "application/json" };
 
return msg;
```
 
### 1.4. Cấu hình node `http response`
 
Giữ mặc định (Status code: 200), không cần chỉnh thêm.
 
### 1.5. Deploy
 
Bấm nút đỏ **Deploy** ở góc trên bên phải giao diện Node-RED.
 
> <img width="1917" height="1028" alt="image" src="https://github.com/user-attachments/assets/1de40f89-4ee3-4445-a25c-0085b5e77bd8" />
 
### 1.6. Test API nội bộ (trong container)
 
```bash
curl http://localhost:1880/api/dssv
```
 
Kết quả:
 
```json
{"ok":1,"msg":"thanh cong","dssv":[
  {"name":"Pham Thanh Son","money":700000,"status":"Du dieu kien"},
  {"name":"Truong Van Hai","money":550000,"status":"Du dieu kien"},
  {"name":"Dam Ngoc Son","money":500000,"status":"Du dieu kien"},
  {"name":"Nguyen Van An","money":350000,"status":"Du dieu kien"},
  {"name":"Hoang Dinh Diep","money":250000,"status":"Chua du"}
]}
```
 
> <img width="1917" height="1078" alt="image" src="https://github.com/user-attachments/assets/5aed0f3e-a7d3-4904-9ae9-96ffca2bdc91" />
 
---
 
## 2. Cấu hình Nginx để gọi API qua domain thật
 
Vì Node-RED chạy nội bộ trong Docker network (`nodered:1880`), cần Nginx làm reverse proxy để expose API ra domain công khai.
 
### 2.1. Thêm block `location /api/` vào file `nginx/conf.d/site1.conf`
 
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
 
    # === Reverse proxy API sang Node-RED ===
    location /api/ {
        proxy_pass http://nodered:1880/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
 
        # Cho phép JS gọi API cross-origin (CORS)
        add_header 'Access-Control-Allow-Origin' '*' always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS' always;
        add_header 'Access-Control-Allow-Headers' 'Content-Type' always;
    }
}
```
 
### 2.2. Khởi động lại Nginx để áp dụng cấu hình
 
```bash
cd ~/lab1
docker compose restart nginx
```
 
### 2.3. Test API qua domain thật
 
```bash
curl https://site1.damngocson.id.vn/api/dssv
```
 
Kết quả trả về giống hệt khi test nội bộ ở mục 1.6, xác nhận Nginx đã reverse proxy đúng qua Cloudflare Tunnel.
 
> <img width="1485" height="760" alt="image" src="https://github.com/user-attachments/assets/fb99eb69-8e43-4743-8d39-a477377ecf96" />

---
 
## 3. Viết JavaScript trong HTML để gọi API
 
### 3.1. Nội dung file `nginx/site1/index.html`
 
```html
<!DOCTYPE html>
<html lang="vi">
<head>
<meta charset="UTF-8">
<title>Danh sách sinh viên</title>
<style>
    body { font-family: Arial, sans-serif; margin: 40px; background: #f4f6f8; }
    h1 { color: #2c3e50; }
    table { border-collapse: collapse; width: 100%; max-width: 600px; background: white; }
    th, td { border: 1px solid #ddd; padding: 10px; text-align: left; }
    th { background: #2c3e50; color: white; }
    .du { color: green; font-weight: bold; }
    .chua { color: #c0392b; }
    #status { margin-bottom: 15px; font-style: italic; }
</style>
</head>
<body>
 
<h1>Danh sách sinh viên (gọi từ API Node-RED)</h1>
<p id="status">Đang tải dữ liệu...</p>
 
<table id="bang-sv" style="display:none;">
    <thead>
        <tr><th>Họ tên</th><th>Số tiền</th><th>Trạng thái</th></tr>
    </thead>
    <tbody id="body-sv"></tbody>
</table>
 
<script>
    // Gọi API bằng fetch()
    fetch('/api/dssv')
        .then(response => {
            if (!response.ok) {
                throw new Error('Lỗi HTTP: ' + response.status);
            }
            return response.json();
        })
        .then(data => {
            const statusEl = document.getElementById('status');
            const tableEl = document.getElementById('bang-sv');
            const bodyEl = document.getElementById('body-sv');
 
            if (data.ok === 1) {
                statusEl.textContent = 'Kết quả: ' + data.msg;
                tableEl.style.display = 'table';
 
                data.dssv.forEach(sv => {
                    const row = document.createElement('tr');
                    const cssClass = sv.status === 'Du dieu kien' ? 'du' : 'chua';
 
                    row.innerHTML = `
                        <td>${sv.name}</td>
                        <td>${sv.money.toLocaleString('vi-VN')} đ</td>
                        <td class="${cssClass}">${sv.status}</td>
                    `;
                    bodyEl.appendChild(row);
                });
            } else {
                statusEl.textContent = 'API trả về lỗi!';
            }
        })
        .catch(error => {
            document.getElementById('status').textContent = 'Lỗi khi gọi API: ' + error.message;
            console.error('Fetch error:', error);
        });
</script>
 
</body>
</html>
```
 
### 3.2. Giải thích logic JS
 
- `fetch('/api/dssv')`: gửi request GET tới API (đường dẫn tương đối, tự động dùng domain hiện tại đang mở trang).
- `.then(response => response.json())`: chuyển response thành object JavaScript.
- Nếu `data.ok === 1`: duyệt qua mảng `dssv`, tạo từng dòng `<tr>` và chèn vào bảng, gán màu class `du`/`chua` theo `status`.
- `.catch()`: bắt lỗi nếu API lỗi hoặc mất kết nối, hiển thị thông báo lỗi thay vì để trang trắng.
> <img width="1917" height="1030" alt="image" src="https://github.com/user-attachments/assets/2bceed41-3a32-467a-bad5-e5ddfacf2946" />

---
 
## 4. Sơ đồ luồng dữ liệu tổng quan
 
```
Trình duyệt (JS fetch)
      │
      ▼
https://site1.damngocson.id.vn/api/dssv
      │
      ▼
Cloudflare Tunnel → Nginx (location /api/)
      │
      ▼
proxy_pass → Node-RED container :1880/api/dssv
      │
      ▼
[http in] → [function: thuật toán sắp xếp + phân loại] → [http response]
      │
      ▼
Trả JSON: {"ok":1,"msg":"thanh cong","dssv":[...]}
      │
      ▼
JS nhận JSON → render thành bảng HTML
```
 
---
 
## 5. Các lỗi đã gặp và cách khắc phục
 
| Lỗi | Nguyên nhân | Cách khắc phục |
|---|---|---|
| `curl: Cannot GET /api/dssv` | Flow Node-RED chưa Import/Deploy thành công, hoặc sai URL trong node `http in` | Kiểm tra lại tab flow đã import đủ 3 node và nối dây đúng, bấm lại Deploy |
| Sửa dữ liệu sinh viên nhưng gọi API không thấy thay đổi | Chỉnh code trong node `function` nhưng quên bấm Deploy | Double-click vào node `function` → sửa mảng `dssv` → bấm `Done` → bấm **Deploy** ở góc trên bên phải |
 
---
 
## 6. Kết quả kiểm thử
 
- [x] `curl http://localhost:1880/api/dssv` (nội bộ) → trả JSON đúng
- [x] `curl https://site1.damngocson.id.vn/api/dssv` (qua domain thật) → trả JSON đúng
- [x] Mở `https://site1.damngocson.id.vn` trên trình duyệt → bảng dữ liệu hiển thị đúng, sắp xếp giảm dần, phân màu theo trạng thái
> <img width="1480" height="757" alt="image" src="https://github.com/user-attachments/assets/0d7c1025-09c9-4345-bb20-1f1a2e8b8faa" />
> <img width="1917" height="1030" alt="image" src="https://github.com/user-attachments/assets/e5cd20e9-fda0-46cb-bfc9-27ddebff254e" />

---
 
## 7. Kết luận
 
Bài tập đã hoàn thành đầy đủ 3 yêu cầu:
 
1. ✅ Tạo API bằng Node-RED với `http in` + `function` (thuật toán tự nghĩ: sắp xếp + phân loại) + `http response`.
2. ✅ Cấu hình Nginx `location /api/` reverse proxy sang Node-RED, cho phép gọi API qua domain thật `https://site1.damngocson.id.vn/api/dssv`.
3. ✅ Viết JavaScript (`fetch API`) trong trang HTML để gọi API và hiển thị dữ liệu dưới dạng bảng có định dạng trực quan.
 
