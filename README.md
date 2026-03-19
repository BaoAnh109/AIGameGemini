# 🎮 AIGameGemini – Survival Shooter với Hệ thống AI

Dự án gồm một **game bắn súng sinh tồn** (battle royale) chạy trực tiếp trên trình duyệt, kết hợp **tài liệu giáo dục** giải thích các thuật toán AI được dùng trong game.

---

## 📁 Cấu trúc dự án

| File | Mô tả |
|------|-------|
| `v4.html` | Game Survival Shooter – chế độ Solo và Multiplayer |
| `teaching_algorithms.html` | AI Teaching Lab – tài liệu giải thích thuật toán AI |

---

## 🕹️ v4.html – Survival Shooter

### Giới thiệu

**Survival Shooter** là game bắn súng theo góc nhìn từ trên xuống (top-down), phong cách battle royale. Người chơi chiến đấu với các bot AI trên bản đồ 3000×3000, trong khi vùng an toàn (safe zone) liên tục thu hẹp.

### Tính năng

- 🤖 **AI thông minh** với hệ thống 4 lớp: State Tree + Decision Tree + A* + KNN
- 🌐 **Multiplayer P2P** (dùng PeerJS – không cần server): tạo phòng / vào phòng qua mã 6 ký tự
- 🗺️ Bản đồ **3000×3000** với vật cản (chướng ngại vật bê tông)
- 💉 Cơ chế **loot** – quái rơi đồ, người chơi nhặt để hồi máu
- 🔵 **Safe Zone** co dần, ép buộc di chuyển như battle royale thực thụ
- 📊 Bảng thống kê thời gian thực (HP, dash, ammo, trạng thái AI)

### Cách chơi

| Phím / Thao tác | Hành động |
|-----------------|-----------|
| `W A S D` | Di chuyển |
| Chuột trái | Bắn |
| `Space` | Lướt (Dash) |
| `R` | Nạp đạn |
| `O` | Bật/tắt hiển thị đường đi A* của quái |

### Chế độ chơi

- **Solo** – Chơi một mình đấu với các bot AI
- **Multiplayer** – Chơi online với bạn bè qua kết nối P2P:
  1. Một người chọn **Tạo Phòng**, chia sẻ mã phòng
  2. Người kia chọn **Vào Phòng**, nhập mã và kết nối
  3. Host bấm **Bắt đầu trận đấu**

### Hệ thống AI

AI bot hoạt động theo vòng lặp liên tục:

```
Nhìn trạng thái → Chọn nhánh hành vi → Học từ kết quả
```

**Các macro-state của AI:**

| State | Điều kiện kích hoạt |
|-------|---------------------|
| `SAFEZONE` | Ngoài vùng an toàn → ưu tiên số 1 |
| `SURVIVAL` | Máu thấp hoặc bị bao vây |
| `COMBAT` | Phát hiện mục tiêu trong tầm nhìn |
| `RESOURCE` | Cần nhặt loot |
| `EXPLORE` | Không có mối đe dọa, đi khám phá |

**Các tactic chiến đấu (Decision Tree trong COMBAT):**
`aggressive attack`, `kite`, `strafe`, `ambush`, `retreat fire`, `dash escape`, `zone reposition`

---

## 📚 teaching_algorithms.html – AI Teaching Lab

### Giới thiệu

Trang tài liệu trình bày theo phong cách **báo cáo mô phỏng**, giải thích chi tiết 3 thuật toán AI cốt lõi trong game, phù hợp cho:

- Báo cáo môn học / đồ án
- Thuyết trình về AI trong game
- Tự học về game AI

### Các thuật toán được giải thích

#### 1. 🌳 State Tree
- Vai trò: Quyết định **chiến lược tổng quát** của AI
- Luồng: Thu thập context → Chấm điểm từng nhánh → So sánh → Ánh xạ sang hành vi → Xuất đầu ra
- Đầu ra: macro-state → giao nhiệm vụ cho A* và lớp hành động

#### 2. 📏 KNN (K-Nearest Neighbors)
- Vai trò: **Dự đoán** hướng di chuyển / hành vi tiếp theo của mục tiêu để tính toán lead shot
- Dữ liệu huấn luyện: `[distance, targetHp, targetSpeed, isDashing, isChased, relativeAngle]`
- Đầu ra: vector hướng bắn dự đoán `[dirX, dirY]`

#### 3. 🗺️ A* Pathfinding
- Vai trò: **Tìm đường vòng** qua vật cản khi không có đường thẳng đến mục tiêu
- Sử dụng khi: bot về safe zone, farm quái, đuổi target mất line-of-sight

---

## 🚀 Cách chạy

Không cần cài đặt gì cả – chỉ cần mở file bằng trình duyệt:

```bash
# Mở game
open v4.html

# Mở tài liệu AI
open teaching_algorithms.html
```

Hoặc kéo thả file `.html` vào cửa sổ trình duyệt bất kỳ (Chrome, Firefox, Edge...).

> ⚠️ Để chơi **Multiplayer**, cả hai người cần mở file qua HTTP server (không phải `file://`) hoặc dùng dịch vụ như [Live Server](https://marketplace.visualstudio.com/items?itemName=ritwickdey.LiveServer) trong VS Code.

---

## 🛠️ Công nghệ sử dụng

- **Vanilla JavaScript** – không dùng framework
- **HTML5 Canvas** – render game
- **[PeerJS v1.5.4](https://peerjs.com/)** – kết nối P2P cho multiplayer
- Thuật toán AI tự xây dựng: State Tree, Decision Tree, KNN, A*

---

## 📝 Ghi chú

Dự án được xây dựng như một bài thực hành minh họa cách các thuật toán AI kinh điển (State Tree, KNN, A*) có thể được kết hợp để tạo ra hành vi bot thông minh trong game. Trang `teaching_algorithms.html` đi kèm giúp người xem hiểu rõ nguyên lý hoạt động của từng thành phần.
