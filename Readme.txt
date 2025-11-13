1/ copy thư mục main vào project
-------------------------------------------------------------
---- main\application.cc ----
-------------------------------------------------------------
2/ Thêm include vào
	- Code:
	#include "alarm_manager.h" 
-------------------------------------------------------------
3/ Thêm code vào hàm : void Application::Start() sau dòng audio_service_.Start();
	- Code:
	/* Initialize Alarm Manager */
	AlarmManager::getInstance().init();
	ESP_LOGI(TAG, "Alarm Manager initialized");
-------------------------------------------------------------
4/ Thêm code vào hàm: `Application::MainEventLoop()` 
	trong hàm điều kiện if (bits & MAIN_EVENT_CLOCK_TICK) 
	sau dòng : display->UpdateStatusBar();
	- Code:
	// Kiểm tra alarm mỗi giây
	AlarmManager::getInstance().checkAlarms();  
-------------------------------------------------------------
---- main\mcp_server.cc ----
-------------------------------------------------------------
5/ Thêm include vào
	- Code:
	#include "alarm_manager.h" 
-------------------------------------------------------------
6/ Thêm code vào trong hàm `McpServer::AddCommonTools() 
	Tìm dòng  #ifdef HAVE_LVGL
	Tìm về cuối #endif  
	- Code:
	// Thêm vào MCP
	AlarmManager::getInstance().mcpVoice();
-------------------------------------------------------------
----  main\CMakeLists.txt  ----
-------------------------------------------------------------
7/ Trong phần set(SOURCES kế  "main.cc"
	- Code:
	"alarm_manager.cc"
        "main.cc"
-------------------------------------------------------------
-------------------------------------------------------------
-------------------------------------------------------------
-------------------------------------------------------------
** Yêu cầu file chông báo ** xem để hiểu ko có sửa gì
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










