# Xiaozhi_Alam
Thêm giờ báo thức cho Xiaozhi
1/ copy thư mục main vào project
-------------------------------------------------------------
---- main/application.h ----
2/ Thêm include vào
	- Code:
	#include "alarm_manager.h" 
-------------------------------------------------------------
---- main/application.cc ----
3/ Thêm code vào hàm : void Application::Start() sau dòng audio_service_.Start();
	- Code:
	/* Initialize Alarm Manager */
	AlarmManager::getInstance().init();
	ESP_LOGI(TAG, "Alarm Manager initialized");
4/ Thêm code vào hàm: `Application::MainEventLoop()` tròng hàm điều kiện if (bits & MAIN_EVENT_CLOCK_TICK) 
	sau dòng display->UpdateStatusBar();
	- Code:
	// Kiểm tra alarm mỗi giây
	AlarmManager::getInstance().checkAlarms();  
-------------------------------------------------------------
---- mcp_server.cc ----
5/ Thêm include vào
	- Code:
	#include "alarm_manager.h" 
6/ Thêm code vào trong hàm `McpServer::AddCommonTools() 
	tìm về cuối #endif  // ← Dòng này đã có sẵn (đóng #ifdef HAVE_LVGL)
	- Code:

	// ===== Thêm ALARM/REMINDER TOOLS =====
    AddTool("self.alarm.set",
        "Đặt báo thức hoặc nhắc nhở cho người dùng.\n"
        "Tham số:\n"
        "  `hour`: Giờ (0-23, bắt buộc)\n"
        "  `minute`: Phút (0-59, bắt buộc)\n"
        "  `message`: Nội dung nhắc nhở (tùy chọn)\n"
        "  `repeated`: Lặp lại hàng ngày (true/false, mặc định false)\n"
        "Trả về:\n"
        "  Trạng thái đặt báo thức.",
        PropertyList({
            Property("hour", kPropertyTypeInteger, 0, 23),
            Property("minute", kPropertyTypeInteger, 0, 59),
            Property("message", kPropertyTypeString, "Báo thức"),
            Property("repeated", kPropertyTypeBoolean, false)
        }),
        [](const PropertyList& properties) -> ReturnValue {
            auto& alarm_mgr = AlarmManager::getInstance();
            
            uint8_t hour = static_cast<uint8_t>(properties["hour"].value<int>());
            uint8_t minute = static_cast<uint8_t>(properties["minute"].value<int>());
            bool repeated = properties["repeated"].value<bool>();
            
            AlarmManager::Alarm new_alarm;
            new_alarm.hour = hour;
            new_alarm.minute = minute;
            new_alarm.message = "";
            new_alarm.enabled = true;
            new_alarm.repeated = repeated;
            
            alarm_mgr.addAlarm(new_alarm);

            char buffer[128];
            snprintf(buffer, sizeof(buffer), 
                     "{\"success\": true, \"message\": \"Đã đặt báo thức lúc %02d:%02d\"}",
                     hour, minute);
            
            ESP_LOGI("MCP", "Set alarm: %02d:%02d (repeated: %d)", 
                     hour, minute, repeated);
            
            return std::string(buffer);
        });

    AddTool("self.alarm.list",
        "Xem danh sách tất cả báo thức đã đặt, sắp xếp theo thời gian gần nhất.",
        PropertyList(),
        [](const PropertyList& properties) -> ReturnValue {
            auto& alarm_mgr = AlarmManager::getInstance();
            std::string info = alarm_mgr.getAllAlarmsInfo();
            return "{\"success\": true, \"message\": \"" + info + "\"}";
        });

    AddTool("self.alarm.clear",
        "Xóa tất cả báo thức.",
        PropertyList(),
        [](const PropertyList& properties) -> ReturnValue {
            auto& alarm_mgr = AlarmManager::getInstance();
            alarm_mgr.clearAll();
            return "{\"success\": true, \"message\": \"Đã xóa tất cả báo thức\"}";
        });
    // ← KẾT THÚC Alam	
-------------------------------------------------------------
----  `main/CMakeLists.txt`  ----
7/ Trong phần set(SOURCES kế  "main.cc"
	- Code:
	"alarm_manager.cc"
        "main.cc"
-------------------------------------------------------------
8/ ** Yêu cầu file chông báo ** xem để hiểu ko có sửa gì
   - Format: `.ogg` (Ogg Vorbis)
   - Độ dài: 1-3 giây
   - Kích thước: < 50KB (để tiết kiệm bộ nhớ)
   - 16000Hz
   - main/assets/common/alarm_beep.ogg`

Phần sau đây trong CMake đã tự động thêm tất cả các file vào 
```cmake
file(GLOB LANG_SOUNDS ${CMAKE_CURRENT_SOURCE_DIR}/assets/locales/${LANG_DIR}/*.ogg)
file(GLOB COMMON_SOUNDS ${CMAKE_CURRENT_SOURCE_DIR}/assets/common/*.ogg)

# ... một số dòng khác ...
idf_component_register(SRCS ${SOURCES}
                    EMBED_FILES ${LANG_SOUNDS} ${COMMON_SOUNDS}
                    INCLUDE_DIRS ${INCLUDE_DIRS}
                    WHOLE_ARCHIVE
                    )










