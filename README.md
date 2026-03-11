# 🎮 Survival Shooter - Battle Royale with AI Intelligence System

## 📋 Mô tả dự án

Game **Survival Shooter** là một trò chơi Battle Royale 2D top-down, hỗ trợ cả chế độ **Solo** (đơn) và **Multiplayer** (trực tuyến P2P qua PeerJS). Người chơi chiến đấu với 3 Bot AI và các quái vật (Zombie) trên bản đồ 3000x3000 pixel, với vùng an toàn (Safe Zone) co dần theo thời gian.

**Công nghệ:** HTML5 Canvas, Vanilla JavaScript, PeerJS (WebRTC P2P).

---

## 🧠 HỆ THỐNG THUẬT TOÁN AI

### 1. AI Quái Vật (Monster AI)

#### 1.1. Thuật toán di chuyển — Greedy Chase + Separation

Quái vật sử dụng mô hình **Steering Behavior** đơn giản gồm 2 thành phần:

**a) Truy đuổi mục tiêu gần nhất (Greedy Nearest-Target Chase):**

```
Với mỗi frame:
  1. Duyệt tất cả Player (cả người và bot) còn sống
  2. Tính khoảng cách Euclidean: d = √((mx - px)² + (my - py)²)
  3. Chọn Player có d nhỏ nhất làm mục tiêu (Greedy)
  4. Tính vector hướng: dir = normalize(target.pos - monster.pos)
  5. Di chuyển: monster.pos += dir × speed
```

- **Độ phức tạp:** O(n) với n = số Player, thực thi mỗi frame.
- **Ưu điểm:** Đơn giản, hiệu quả, tạo áp lực liên tục.

**b) Tách đàn (Separation / Flocking Avoidance):**

Khi 2 quái vật ở quá gần nhau (khoảng cách < 25px), chúng đẩy nhau ra để tránh chồng chất:

```
Với mỗi quái vật khác m2:
  Nếu distance(m1, m2) < 25:
    moveVector -= (m2.pos - m1.pos)   // đẩy ngược hướng
```

Kết quả `moveVector` được **normalize** rồi nhân với `speed` để ra hướng di chuyển cuối cùng, tổng hợp cả truy đuổi lẫn tách đàn.

#### 1.2. Tấn công cận chiến

```
Nếu distance(monster, player) < monster.radius + player.radius
  VÀ player không đang Dash
  VÀ cooldown đã hết (1000ms):
    → Gây 10 damage cho player
```

#### 1.3. Spawn quái vật

- Spawn mỗi **1.5 giây**, tối đa **40 con**.
- Vị trí spawn: ngẫu nhiên ở **ngoài rìa camera** của người chơi (tạo cảm giác quái xuất hiện từ bên ngoài tầm nhìn).

---

### 2. AI Player (Bot AI) — Finite State Machine + KNN Intelligence

Bot AI là phần cốt lõi của hệ thống, được thiết kế với kiến trúc phân lớp:

```
┌─────────────────────────────────────────────────┐
│              BotPlayer (Hành vi)                │
│  ┌───────────────────────────────────────────┐  │
│  │     Finite State Machine (FSM)            │  │
│  │  ZONE ← FLEE ← ATTACK → LOOT → WANDER   │  │
│  └────────────────┬──────────────────────────┘  │
│                   │                              │
│  ┌────────────────▼──────────────────────────┐  │
│  │         BotBrain (Bộ não)                 │  │
│  │  • KNN Predictor (dự đoán hành vi)        │  │
│  │  • IQ System (chỉ số thông minh)          │  │
│  │  • Tactical Memory (bộ nhớ chiến thuật)   │  │
│  │  • Combat Parameters (tham số chiến đấu)  │  │
│  └────────────────┬──────────────────────────┘  │
│                   │                              │
│  ┌────────────────▼──────────────────────────┐  │
│  │      Knowledge System (Tri thức)          │  │
│  │  • Global KNN (dataset chung)             │  │
│  │  • Knowledge Pool (sự kiện chia sẻ)       │  │
│  │  • Death Knowledge Transfer               │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

#### 2.1. Máy trạng thái hữu hạn (Finite State Machine - FSM)

Bot có **5 trạng thái**, được đánh giá theo **thứ tự ưu tiên** mỗi frame:

```
                 ┌──────────────┐
          ┌─────►│    ZONE      │  (Ưu tiên cao nhất)
          │      │ Di chuyển    │
          │      │ về vùng an   │
          │      │ toàn         │
          │      └──────────────┘
          │
          │      ┌──────────────┐
          ├─────►│    FLEE      │  (HP thấp / đông kẻ địch)
          │      │ Chạy trốn,   │
          │      │ tìm vật phẩm │
          │      └──────────────┘
          │
    Mỗi   │     ┌──────────────┐
   frame───┤    │   ATTACK     │  (Có kẻ địch trong tầm)
          ├────►│ Bắn + di     │
          │     │ chuyển chiến  │
          │     │ thuật         │
          │     └──────────────┘
          │
          │     ┌──────────────┐
          ├────►│    LOOT      │  (Có vật phẩm gần)
          │     │ Nhặt item    │
          │     └──────────────┘
          │
          │     ┌──────────────┐
          └────►│   WANDER     │  (Mặc định)
                │ Đi lang thang│
                └──────────────┘
```

**Logic quyết định trạng thái (mỗi frame):**

```python
# Pseudocode
distToZone = distance(bot, zone.center)
closestThreat = findClosest(allThreats)    # Player + Monster
nearbyThreats = count(threats trong 400px)

if distToZone > zone.radius - 50:
    state = "ZONE"                          # 1. Ưu tiên sống sót trong bo

elif HP < maxHP × fleeThreshold
   OR not shouldEngage(HP, threatDist, nearbyThreats):
    state = "FLEE"                          # 2. Máu thấp → chạy

elif closestThreat AND closestThreat.dist < smartAttackRange:
    state = "ATTACK"                        # 3. Có mục tiêu → tấn công

elif closestItem gần:
    state = "LOOT"                          # 4. Nhặt vật phẩm

else:
    state = "WANDER"                        # 5. Lang thang
```

Trong đó:
- `smartAttackRange = 400 + (IQ / 100) × 150` — Bot IQ cao phát hiện xa hơn.
- `fleeThreshold` — Ngưỡng HP để chạy trốn, ban đầu ngẫu nhiên, được điều chỉnh qua học tập.

#### 2.2. Chi tiết hành vi từng trạng thái

##### ZONE (Di chuyển về vùng an toàn)
```
moveTo(zone.center.x, zone.center.y)
```
Đơn giản di chuyển thẳng về tâm vùng an toàn.

##### FLEE (Chạy trốn)
```
Nếu có vật phẩm hồi máu < 350px:
    → Di chuyển tới vật phẩm (ưu tiên hồi máu khi chạy)

Nếu có kẻ địch < 300px:
    fleeDir = bot.pos - threat.pos          # Hướng ngược kẻ địch

    Nếu IQ > 40:
        safeDir = zone_direction × 0.3 + fleeDir × 0.7
        → Chạy trốn NHƯNG hướng về phía bo (thông minh hơn)
    Ngược lại:
        → Chạy thẳng khỏi kẻ thù (có thể ra ngoài bo)

    Nếu Dash sẵn sàng:
        → Sử dụng Dash để thoát nhanh

Nếu không có mối đe dọa gần:
    → Wander ngẫu nhiên
```

##### ATTACK (Tấn công)

Đây là trạng thái phức tạp nhất, kết hợp nhiều thuật toán:

**a) Kiểm tra Line of Sight (LoS):**
```
Sử dụng thuật toán Line-Rectangle Intersection:
  Nếu đường thẳng từ bot → target cắt bất kỳ obstacle nào:
    → Không có tầm nhìn → Di chuyển đến gần target
  Ngược lại:
    → Có tầm nhìn → Chiến đấu
```

Thuật toán LoS kiểm tra giao cắt đoạn thẳng với 4 cạnh hình chữ nhật (obstacle) sử dụng **parametric line intersection**:

$$t_a = \frac{(x_4-x_3)(y_1-y_3) - (y_4-y_3)(x_1-x_3)}{(y_4-y_3)(x_2-x_1) - (x_4-x_3)(y_2-y_1)}$$

$$t_b = \frac{(x_2-x_1)(y_1-y_3) - (y_2-y_1)(x_1-x_3)}{(y_4-y_3)(x_2-x_1) - (x_4-x_3)(y_2-y_1)}$$

Giao cắt xảy ra khi $0 \leq t_a \leq 1$ và $0 \leq t_b \leq 1$.

**b) Di chuyển chiến thuật (Tactical Movement):**
```
preferredRange = brain.preferredRange   # Khoảng cách lý tưởng (220-340px)

Nếu dist < preferredRange - 80:
    → Lùi lại (quá gần)
Nếu dist > preferredRange + 70:
    → Tiến lên (quá xa)
Nếu IQ > 35:
    → Strafing: di chuyển theo phương vuông góc (perpendicular)
    perpX = -(target.y - bot.y)
    perpY = target.x - bot.x
    direction = sin(time × 0.003 + bot.x) > 0 ? 1 : -1
    → Tạo chuyển động zig-zag khó bắn trúng
```

**c) Bắn với dự đoán KNN (xem mục 2.3).**

##### LOOT (Nhặt vật phẩm)
```
Nếu có item < 400px (hoặc < 700px nếu IQ > 50):
    → Di chuyển tới item gần nhất
Ngược lại:
    → Wander
```

##### WANDER (Lang thang)
```
Mỗi 2 giây: chọn góc ngẫu nhiên mới (0 → 2π)
Di chuyển theo hướng đó với 50% tốc độ
```

#### 2.3. Thuật toán KNN (K-Nearest Neighbors) — Dự đoán hành vi mục tiêu

Đây là thuật toán AI cốt lõi giúp Bot **dự đoán hướng di chuyển** của mục tiêu để bắn chặn đầu (predictive aiming).

##### Cấu trúc dữ liệu

```
Dataset entry = {
    features: [distance, targetHP, isChased],   // Vector đặc trưng
    label:    { x: dirX, y: dirY },             // Hướng di chuyển thực tế
    time:     timestamp
}
```

- **distance**: Khoảng cách bot → mục tiêu
- **targetHP**: Máu hiện tại của mục tiêu
- **isChased**: Mục tiêu có đang bị quái đuổi không (0 hoặc 1)
- **label**: Hướng di chuyển thực tế tại thời điểm ghi nhận

##### Thuật toán dự đoán

```python
def predict(distance, targetHP, isChased, k=3):
    if len(dataset) < 5:
        return None    # Chưa đủ dữ liệu

    # Tính khoảng cách Weighted Euclidean tới mỗi điểm trong dataset
    for each data_point in dataset:
        d = sqrt(
            (data.distance - distance)² × 0.5     # Trọng số thấp
          + (data.targetHP - targetHP)² × 2.0      # Trọng số cao (HP quan trọng)
          + (data.isChased - isChased)² × 50       # Trọng số rất cao (trạng thái)
        )

    # Chọn k=3 điểm gần nhất
    topK = sort_by_distance(all_distances)[:k]

    # Trung bình hướng di chuyển
    avgDir = average(topK.labels)
    return normalize(avgDir)
```

**Trọng số đặc trưng:**
| Đặc trưng | Trọng số | Lý do |
|---|---|---|
| distance | 0.5 | Ít ảnh hưởng đến hành vi |
| targetHP | 2.0 | HP thấp → hành vi khác biệt (chạy trốn) |
| isChased | 50 | Bị quái đuổi thay đổi hoàn toàn hành vi |

##### Áp dụng dự đoán vào bắn

```python
predictedDir = brain.predictTarget(dist, target.hp, isChased)
if predictedDir:
    timeToReach = dist / bullet_speed        # Thời gian đạn bay tới
    targetSpeed = target.dashSpeed if target.isDashing else target.speed
    aimX = target.x + predictedDir.x × targetSpeed × timeToReach
    aimY = target.y + predictedDir.y × targetSpeed × timeToReach
    # → Bắn vào vị trí DỰ ĐOÁN, không phải vị trí hiện tại
```

##### Hai tầng KNN

```
┌─────────────────────────┐     ┌─────────────────────────┐
│    Local KNN (per Bot)  │     │    Global KNN (chung)   │
│  • Dataset riêng mỗi bot│     │  • Ghi từ Player (người)│
│  • Ưu tiên sử dụng nếu │     │  • Dùng khi local chưa  │
│    dataset ≥ 10 điểm    │     │    đủ dữ liệu           │
│  • Max 200 entries      │     │  • Max 200 entries       │
└─────────────────────────┘     └─────────────────────────┘
         ↕ Transfer Knowledge ↕
```

- Mỗi bot có **Local KNN** riêng, ghi nhận kinh nghiệm chiến đấu cá nhân.
- **Global KNN** ghi nhận hành vi thực của người chơi (khi dash, di chuyển).
- Nếu Local KNN đã có ≥ 10 mẫu → dùng kết quả Local (chính xác hơn).
- Nếu chưa đủ → fallback về Global KNN.

#### 2.4. Hệ thống IQ (Chỉ số thông minh)

Mỗi bot có chỉ số **IQ (0 → 100)** ảnh hưởng trực tiếp đến gameplay:

| Thành phần | Công thức | Ảnh hưởng |
|---|---|---|
| Aim Spread | `max(0.02, 0.15 - IQ/100 × 0.12)` | IQ cao → bắn chính xác hơn |
| Fire Rate | `max(0.7, 1.0 - IQ/100 × 0.3)` | IQ cao → bắn nhanh hơn |
| Attack Range | `400 + IQ/100 × 150` | IQ cao → phát hiện xa hơn |
| Flee IQ > 40 | Chạy hướng về bo thay vì chạy thẳng | Sống sót lâu hơn |
| Strafe IQ > 35 | Di chuyển zig-zag khi chiến đấu | Khó bị bắn trúng |
| Loot IQ > 50 | Phát hiện vật phẩm xa hơn (700px) | Thu thập hiệu quả |

**Cách tăng IQ:**

| Hành động | IQ tăng |
|---|---|
| Bắn trúng mục tiêu | +0.15 |
| Hạ gục mục tiêu | +2.0 |
| Tránh né thành công | +0.1 |
| Combat experience thành công | +0.3 |
| Thời gian sống sót (mỗi 2s) | +0.05 |
| Học từ bot khác (xem 2.5) | +0.05 → +1.5 |
| Kế thừa tri thức bot chết | +variable |

#### 2.5. Hệ thống Học tập (Learning System)

##### a) Học từ Bot sống (Peer Learning)

Mỗi **2 giây**, bot quét các bot khác trong bán kính **600px**:

```python
def learnFromOther(otherBrain):
    if cooldown chưa hết (3s):
        return
    if other.IQ ≤ my.IQ:
        return                            # Chỉ học từ bot giỏi hơn

    iqGap = other.IQ - my.IQ
    learnAmount = min(1.5, iqGap × 0.05)  # Học nhiều hơn nếu gap lớn
    my.IQ += learnAmount

    # Transfer KNN data (lấy mẫu ngẫu nhiên 5 entries)
    for i in range(min(5, len(other.localKNN))):
        sample = random_choice(other.localKNN.dataset)
        my.localKNN.dataset.append(copy(sample))

    # Điều chỉnh tham số chiến đấu theo bot giỏi hơn
    my.preferredRange += (other.preferredRange - my.preferredRange) × 0.1
    my.aggressionBias += (other.aggressionBias - my.aggressionBias) × 0.1
    my.fleeThreshold  += (other.fleeThreshold - my.fleeThreshold) × 0.1

    # Transfer tactical memory (3 vùng nguy hiểm gần nhất)
    for mem in other.tacticalMemory[-3:]:
        if không trùng lặp:
            my.tacticalMemory.append(copy(mem))
```

##### b) Kế thừa tri thức khi Bot chết (Death Knowledge Transfer)

Khi một bot bị hạ, tri thức của nó được **truyền lại cho tất cả bot còn sống** (trừ bot giết nó):

```python
# Bot chết tạo gói tri thức
deathKnowledge = {
    dataset: localKNN.dataset[-20:],       # 20 mẫu KNN gần nhất
    tacticalMemory: tacticalMemory[-10:],   # 10 vùng nguy hiểm
    preferredRange, aggressionBias, iq
}

# Mỗi bot sống hấp thụ
def absorbDeathKnowledge(knowledge):
    # Merge KNN dataset
    for data in knowledge.dataset:
        my.localKNN.dataset.append(copy(data))

    # Merge tactical memory
    for mem in knowledge.tacticalMemory:
        my.tacticalMemory.append(copy(mem))

    # IQ bonus
    bonus = min(3, knowledge.iq × 0.05)
    my.IQ += bonus
```

→ Cơ chế này tạo hiệu ứng **"bot càng lúc càng giỏi"** khi game diễn ra, vì tri thức tích lũy không bị mất.

##### c) Bộ nhớ chiến thuật (Tactical Memory)

Bot ghi nhớ các **vùng nguy hiểm** (nơi bị nhận sát thương):

```python
def recordDangerZone(x, y):
    tacticalMemory.append({x, y, time: now})

def isDangerousArea(x, y, radius=150):
    return any(
        mem for mem in tacticalMemory
        if (now - mem.time < 15s) AND distance(mem, {x,y}) < radius
    )
```

Vùng nguy hiểm hết hiệu lực sau **15 giây**.

##### d) Knowledge Pool (Tri thức tập thể)

Hệ thống sự kiện chia sẻ giữa tất cả bot:

```
KnowledgePool.addEvent('damage_taken', {x, y, bot_name})
KnowledgePool.addEvent('bot_death', {victim, killer})
```

Lưu tối đa **50 sự kiện gần nhất**, giúp bot có cái nhìn tổng quan về trận đấu.

#### 2.6. Tham số cá nhân hóa (Per-Bot Personality)

Mỗi bot được khởi tạo với tham số ngẫu nhiên, tạo phong cách chơi riêng:

| Tham số | Giá trị khởi tạo | Ý nghĩa |
|---|---|---|
| `aggressionBias` | 0.3 → 0.8 | Mức độ hung hãn |
| `preferredRange` | 220 → 340 px | Khoảng cách chiến đấu lý tưởng |
| `fleeThreshold` | 0.3 (mặc định) | Tỉ lệ HP để chạy trốn |
| `IQ` | 10 (mặc định) | Bắt đầu thấp, tăng dần |

Các tham số này được **điều chỉnh** qua quá trình học từ bot khác (xem 2.5a).

#### 2.7. Hàm quyết định tham chiến (Engagement Decision)

```python
def shouldEngage(myHp, maxHp, threatDist, threatCount):
    hpRatio = myHp / maxHp
    dynamicThreshold = fleeThreshold + (threatCount - 1) × 0.1

    if hpRatio < dynamicThreshold:
        return False                          # Máu thấp + nhiều kẻ thù → chạy
    if IQ > 50 AND threatDist < 150 AND threatCount > 2:
        return False                          # Bot thông minh không liều lĩnh
    return hpRatio > 0.4 OR aggressionBias > 0.7
                                              # Máu đủ hoặc tính cách hung hãn
```

---

### 3. Tổng kết các thuật toán AI

| Thuật toán | Đối tượng | Mục đích |
|---|---|---|
| **Greedy Nearest Chase** | Monster | Truy đuổi player gần nhất |
| **Separation (Boids)** | Monster | Tránh chồng chất giữa quái |
| **Finite State Machine** | Bot AI | Quyết định hành vi tổng thể |
| **KNN (K-Nearest Neighbors)** | Bot AI | Dự đoán di chuyển mục tiêu, bắn chặn đầu |
| **Weighted Euclidean Distance** | KNN | Tính khoảng cách có trọng số giữa đặc trưng |
| **Line of Sight (LoS)** | Bot AI | Kiểm tra tầm nhìn qua chướng ngại |
| **Parametric Line Intersection** | LoS | Giao cắt đoạn thẳng - hình chữ nhật |
| **Peer Learning** | Bot AI | Học hỏi từ bot giỏi hơn gần đó |
| **Death Knowledge Transfer** | Bot AI | Kế thừa tri thức bot đã chết |
| **Tactical Memory** | Bot AI | Ghi nhớ vùng nguy hiểm |
| **Dynamic Engagement** | Bot AI | Quyết định tấn công/rút lui theo ngữ cảnh |
| **Predictive Aiming** | Bot AI | Dùng KNN dự đoán vị trí tương lai để ngắm bắn |
| **Strafing Movement** | Bot AI | Di chuyển vuông góc tránh đạn |

---

### 4. Sơ đồ luồng AI tổng quát mỗi frame

```
┌──────────────┐
│  Game Loop   │
│  (mỗi frame) │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────────────┐
│ Monster.update()                             │
│  1. Tìm player gần nhất (Greedy)            │
│  2. Tấn công cận chiến nếu đủ gần           │
│  3. Di chuyển = Chase + Separation           │
│  4. Xử lý va chạm với obstacle              │
└──────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────┐
│ BotPlayer.updateAI()                         │
│  1. Peer Learning (mỗi 2s)                   │
│  2. Ghi nhận damage → Tactical Memory        │
│  3. Đánh giá FSM → chọn State                │
│  4. Thực thi hành vi theo State:             │
│     • ZONE: moveTo(zone.center)              │
│     • FLEE: chạy + dash + tìm item           │
│     • ATTACK:                                │
│       a. Check LoS                           │
│       b. Tactical Movement (strafe)          │
│       c. KNN Predict → Predictive Aim        │
│       d. Shoot với spread theo IQ            │
│     • LOOT: moveTo(nearest item)             │
│     • WANDER: di chuyển ngẫu nhiên           │
│  5. Xử lý va chạm với obstacle              │
└──────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────┐
│ Collision & Damage Resolution                │
│  • Đạn trúng → recordDamageDealt, IQ++      │
│  • Hạ gục → recordKill, Death Knowledge      │
│  • Item pickup → hồi 30 HP                  │
└──────────────────────────────────────────────┘
```

---

## 🎮 Hướng dẫn chơi

| Phím | Hành động |
|---|---|
| **WASD** | Di chuyển |
| **Chuột trái** | Bắn |
| **SPACE** | Dash (lướt nhanh) |
| **R** | Nạp đạn |

---

## 🌐 Multiplayer

- Sử dụng **PeerJS (WebRTC)** cho kết nối P2P.
- **Host** chạy toàn bộ game logic, broadcast state 15Hz.
- **Client** chạy client-side prediction + nhận state reconciliation.
- Hỗ trợ tối đa **4 người chơi**, slot trống được điền bằng Bot AI.
