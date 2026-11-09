# Đề 5 – Mạch phát hiện té ngã (ESP32 + MPU6050 + MQTT) – Bản FreeRTOS

## 1. Kiến trúc hệ thống

```
                    Core 1                                        Core 0
┌──────────────────────────────────────┐            ┌────────────────────────────┐
│ TaskDocData (ưu tiên 2)              │  dataQueue │ TaskTruyenData (ưu tiên 1) │
│  - đọc MPU6050 mỗi 100 ms            │ ─────────► │  - kết nối WiFi + MQTT     │──► MQTT Broker
│  - tính gia tốc, góc nghiêng         │            │  - WiFi OK → đọc dataQueue │       │
│  - thuật toán té ngã                 │            │    và publish JSON         │       ▼
│  - in Serial                         │            │  - nhận lệnh từ Web        │     Web/App
└───────────────┬──────────────────────┘            └──────────────┬─────────────┘   (index.html)
                │ alertQueue                                         │ lệnh LED/Loa
                ▼                                                    │ (alertQueue)
┌──────────────────────────────────────┐ ◄──────────────────────────┘
│ TaskCanhBao (ưu tiên 3)              │
│  - đọc alertQueue                    │
│  - điều khiển Còi + Đèn (GPIO25/26)  │
└──────────────────────────────────────┘
```

## 2. Các Task

| Task | Core | Ưu tiên | Stack | Nhiệm vụ |
|---|---|---|---|---|
| `TaskTruyenData` | **0** | 1 | 8192 | Kết nối WiFi; khi WiFi và MQTT đã thành công mới đọc `dataQueue` và gửi lên broker |
| `TaskDocData` | **1** | 2 | 4096 | Đọc MPU6050, hiển thị Serial, đưa dữ liệu đầy đủ vào `dataQueue` và trạng thái cảnh báo vào `alertQueue` |
| `TaskCanhBao` | **1** | 3 | 2048 | Đọc `alertQueue`, điều khiển Còi/Đèn |

Ưu tiên của `TaskCanhBao` cao nhất vì task này chủ yếu ở trạng thái chờ (blocked) trên hàng đợi, khi có tin sẽ được xử lý ngay.

## 3. Các hàng đợi (Queue)

| Queue | Độ dài | Phần tử | Bên gửi | Bên nhận |
|---|---|---|---|---|
| `dataQueue` ("data Queue") | 10 | `SensorData` (ax, ay, az, aTotal, tilt, state) | TaskDocData | TaskTruyenData |
| `alertQueue` ("data Alert") | 10 | `AlertMsg` (loại + giá trị) | TaskDocData, callback MQTT | TaskCanhBao |

`AlertMsg` có 3 loại: `ALERT_STATE` (trạng thái từ thuật toán, `FALL` thì bật Đèn + Còi), `CMD_LED` và `CMD_BUZZER` (lệnh thủ công từ nút trên Web).

Lý do thiết kế:
- **Chỉ `TaskCanhBao` được ghi chân LED/Buzzer**, nên không có xung đột giữa chế độ tự động và thủ công.
- `xQueueSend(..., 0)` không chờ: nếu mạng chậm làm `dataQueue` đầy thì bỏ gói mới, `TaskDocData` không bao giờ bị chặn, thuật toán té ngã vẫn chạy đúng thời gian.
- Lệnh **Reset** từ Web dùng cờ `resetRequest`, vì máy trạng thái nằm trong `TaskDocData`.

## 4. Thuật toán cảnh báo té ngã (máy trạng thái)

| Trạng thái | Điều kiện vào |
|---|---|
| NORMAL | Mặc định, `aTotal` ≈ 1 g |
| FREEFALL | `aTotal` < 0,5 g (hết sau 1 s nếu không có va chạm) |
| IMPACT | `aTotal` > 2 g **hoặc** góc nghiêng đổi > 50° trong 0,5 s |
| FALL | Sau IMPACT 1,5 s: `tilt` > 60° **và** `aTotal` trong 0,7–1,3 g (nằm yên) |

FALL tự bật Đèn + Còi, giữ cho đến khi bấm **Reset** trên Web. Nếu sau va chạm người vẫn đứng (nhảy, ngồi phịch) thì quay về NORMAL, coi là báo động giả.

## 5. Chạy thử trên Wokwi
1. Dán `sketch.ino`, `diagram.json`, `libraries.txt` vào project ESP32 mới, bấm Start.
2. Đổi `PREFIX` (trong `sketch.ino` và `index.html`) thành cùng một chuỗi riêng.
3. Mở `index.html` bằng trình duyệt.
4. Bấm vào chip MPU6050 và chỉnh gia tốc (m/s²) theo kịch bản:

| Bước | X, Y, Z | Giữ | Trạng thái |
|---|---|---|---|
| 1 | 0, 0, 9,8 | tùy ý | NORMAL |
| 2 | 0, 0, 0 | < 1 s | FREEFALL |
| 3 | 0, 0, 30 | ngắn | IMPACT |
| 4 | 9,8, 0, 0 | > 1,5 s | FALL |

## 6. Kết quả kiểm tra đã làm
- Biên dịch thử mã nguồn với bộ header giả lập (stub) của Arduino/FreeRTOS: không có lỗi cú pháp.
- Chạy thử máy trạng thái trên máy tính với các kịch bản trên: ra đúng NORMAL → FREEFALL → IMPACT → FALL, báo động giả quay về NORMAL, rơi tự do ngắn quay về NORMAL.
- **Chưa** chạy trên Wokwi hoặc ESP32 thật, nên cần chạy thử trước khi nộp.
