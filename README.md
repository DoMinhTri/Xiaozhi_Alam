# Xiaozhi Alarm Manager - Hướng Dẫn Tích Hợp

## Mô Tả

Quản lý báo thức và nhắc nhở, cho phép đặt, xem danh sách, xóa báo thức với hỗ trợ lặp lại hàng ngày.

## 1. Copy Files

```bash

cp main/assets/common/alarm_beep.ogg <project>/main/assets/common/

cp main/tools/alarm_manager.h <project>/main/tools/
cp main/tools/alarm_manager.cc <project>/main/tools/
```

## 2. CMakeLists.txt

```cmake
idf_component_register(
    SRCS 
        # ...
        "tools/alarm_manager.cc"
    # ...
)
```

## 3. Application - Tích Hợp Kiểm Tra Báo Thức

### Header - Thêm vào `main/application.cc`
```cpp
#include "tools/alarm_manager.h"
```

### Vòng Lặp Chính - Thêm vào `MainEventLoop()` xử lý `MAIN_EVENT_CLOCK_TICK`
```cpp
if (bits & MAIN_EVENT_CLOCK_TICK) {
    clock_ticks_++;
    auto display = Board::GetInstance().GetDisplay();
    display->UpdateStatusBar();
    
    // Check alarms every second
    AlarmManager::getInstance().checkAlarms();

    // Print the debug info every 10 seconds
    if (clock_ticks_ % 10 == 0) {
        SystemInfo::PrintHeapStats();
    }
}
```

**Mục đích**: Kiểm tra báo thức mỗi giây, sẽ trigger nếu đúng giờ

---

## 4. MCP Server

### Header - Thêm vào `main/mcp_server.cc`
```cpp
#include "tools/alarm_manager.h"
```

### Đăng Ký Tool - Thêm vào `main/mcp_server.cc` trong hàm khởi tạo
### void McpServer::AddCommonTools()
### Vị trí thêm vào sau hàm #ifdef HAVE_LVGL  ... #endif
```cpp
AlarmManager::getInstance().mcpVoice();
```

---
**Phức Tạp**: ⭐⭐ Medium | **Thời gian**: ~5 mins

--------------------------------------------------
--------------------------------------------------
--------------------------------------------------

## 5. Exception

Ném `std::runtime_error` nếu:
- Thời gian không hợp lệ (giờ > 23, phút > 59)
- Báo thức trùng lặp (cùng thời gian)
- NVS lỗi (lưu trữ không thành công)

## 6. Mô Tả API

Quản lý danh sách báo thức, hỗ trợ đặt báo thức một lần hoặc lặp lại hàng ngày. Dữ liệu được lưu vào NVS Flash.

## 7. MCP Tools

```
self.alarm.set      - Đặt báo thức mới
self.alarm.list     - Xem danh sách báo thức
self.alarm.clear    - Xóa tất cả báo thức
```

## 8. Chi Tiết Hàm

### 8.1. getInstance()
```cpp
static AlarmManager& getInstance();
```
**Return**: Reference đến instance duy nhất của AlarmManager (Singleton)

**Mô tả**: Lấy instance toàn cục

---

### 8.2. init()
```cpp
void init();
```
**Mô tả**: Khởi tạo AlarmManager, mở NVS, tải báo thức từ bộ nhớ

**Exception**: `std::runtime_error` nếu mở NVS thất bại

---

### 8.3. addAlarm()
```cpp
void addAlarm(const Alarm& alarm);
```
**Tham số**: 
- `alarm`: Struct chứa thông tin báo thức (hour, minute, message, enabled, repeated)

**Return**: Không

**Mô tả**: Thêm báo thức mới, tự động kiểm tra trùng lặp và lưu vào NVS

**Exception**: Bỏ qua nếu thời gian không hợp lệ hoặc trùng lặp

---

### 8.4. parseResponse()
```cpp
void parseResponse(const std::string& text);
```
**Tham số**: 
- `text`: Chuỗi text từ LLM (ví dụ: "đặt báo thức 9h45")

**Return**: Không

**Mô tả**: Phân tích text, trích xuất thời gian, tạo báo thức mới

**Hỗ trợ định dạng**:
- "9h45" - Định dạng compact
- "9 giờ 45" - Định dạng đầy đủ
- "09:45" - Định dạng tiêu chuẩn
- "hàng ngày", "mỗi ngày" - Báo thức lặp lại

---

### 8.5. checkAlarms()
```cpp
void checkAlarms();
```
**Return**: Không

**Mô tả**: Kiểm tra báo thức mỗi giây, trigger nếu đúng giờ

**Gọi từ**: Main task mỗi giây

---

### 8.6. getNextAlarmInfo()
```cpp
std::string getNextAlarmInfo();
```
**Return**: Chuỗi thông tin báo thức tiếp theo (VD: "09:45 - Dậy sáng")

**Mô tả**: Lấy thông tin báo thức sắp tới nhất

---

### 8.7. getAllAlarmsInfo()
```cpp
std::string getAllAlarmsInfo();
```
**Return**: Chuỗi chứa danh sách tất cả báo thức, sắp xếp theo thời gian

**Ví dụ**:
```
Danh sách báo thức:
1. 09:45 - Dậy sáng (còn 5h30p)
2. 14:00 - Ăn cơm [Hàng ngày] (còn 10h15p)
3. 22:00 - Đi ngủ [Hàng ngày] (còn 18h45p)
```

---

### 8.8. clearAll()
```cpp
void clearAll();
```
**Return**: Không

**Mô tả**: Xóa tất cả báo thức, lưu vào NVS

---

### 8.9. isDuplicateAlarm()
```cpp
bool isDuplicateAlarm(uint8_t hour, uint8_t minute);
```
**Tham số**: 
- `hour`: Giờ (0-23)
- `minute`: Phút (0-59)

**Return**: `true` nếu tồn tại báo thức cùng thời gian, `false` nếu không

**Mô tả**: Kiểm tra trùng lặp trước khi thêm báo thức mới

---

### 8.10. cleanupExpiredAlarms()
```cpp
void cleanupExpiredAlarms();
```
**Return**: Không

**Mô tả**: Xóa các báo thức đã hết hạn (enabled=false, repeated=false)

---

### 8.11. triggerAlarm()
```cpp
void triggerAlarm(const Alarm& alarm);
```
**Tham số**: 
- `alarm`: Báo thức để trigger

**Return**: Không

**Mô tả**: 
- Hiển thị thông báo trên màn hình
- Phát âm thanh báo thức 5 lần (cách nhau 1 giây)
- Tự động vô hiệu hóa nếu không phải lặp lại

---

### 8.12. getAlarms()
```cpp
const std::vector<Alarm>& getAlarms() const;
```
**Return**: Reference đến vector chứa danh sách báo thức

**Mô tả**: Lấy danh sách báo thức trực tiếp (chỉ đọc)

---

### 8.13. mcpVoice()
```cpp
void mcpVoice();
```
**Mô tả**: Đăng ký 3 MCP tool:
- `self.alarm.set` - Đặt báo thức (tham số: hour, minute, message, repeated)
- `self.alarm.list` - Xem danh sách báo thức
- `self.alarm.clear` - Xóa tất cả báo thức

**MCP Tools**:
```
self.alarm.set(hour, minute, message="", repeated=false)
self.alarm.list()
self.alarm.clear()
```

---

## 9. Struct Alarm

```cpp
struct Alarm {
    uint8_t hour;           // Giờ (0-23)
    uint8_t minute;         // Phút (0-59)
    std::string message;    // Nội dung nhắc nhở
    bool enabled;           // Báo thức có hoạt động
    bool repeated;          // Lặp lại hàng ngày
};
```

**Ví dụ**:
```cpp
AlarmManager::Alarm alarm;
alarm.hour = 9;
alarm.minute = 45;
alarm.message = "Dậy sáng";
alarm.enabled = true;
alarm.repeated = true;

AlarmManager::getInstance().addAlarm(alarm);
```

---

## 10. Lưu Trữ NVS

Báo thức được lưu trữ trong NVS Flash với namespace "alarms":
- `count`: Số lượng báo thức
- `alarm_0`, `alarm_1`, ...: Dữ liệu từng báo thức

**Định dạng lưu trữ**: `HH:MM|message|enabled|repeated`

**Ví dụ**: `09:45|Dậy sáng|1|1`

