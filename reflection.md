# Individual Reflection — Đào Hồng Sơn (2A202600462)

## 1. Role (Vai trò)
Head of engineering, solution architecture + investors
Phát triển toàn bộ logic backend, sửa lỗi nghiêm trọng khiến app bị treo/crash.

## 2. Phần phụ trách cụ thể
- Thiết kế và triển khai toàn bộ hệ thống backend: Settings app, quản lý ngày giờ, âm thanh alarm, alarm service, các màn hình chính (Home, CrossCheck, Settings, Navigation).
- Xây dựng shared store giữa các màn hình để CrossCheck ghi, Home và AlarmService đọc chung dữ liệu lịch thuốc.


## 3. SPEC mạnh/yếu
- **Mạnh nhất:** Thiết kế luồng AI scan → CrossCheck → Home có kiến trúc rõ ràng, chia rõ mock và thực. AlarmService tách biệt, có demo mode để test nhanh.
- **Yếu nhất:** Không có cơ chế lưu trữ dữ liệu bền (file JSON / SQLite) — mỗi lần khởi động lại app, lịch thuốc luôn quay về `SAMPLE_SCHEDULE` gốc. Không tương thích thực tế.

## 4. Đóng góp cụ thể

### 4.1. Sửa lỗi nghiêm trọng
- Test và phát hiện lỗi `dismiss(force=True)` không tồn tại trong KivyMD 1.2.0 → sửa chỉ dùng `dismiss()`.
- Xác minh tất cả các import module mới bằng `python -c` sau mỗi lần sửa để đảm bảo không break app.

### 4.2. Hệ thống màu sắc và Theme
- **Tái cấu trúc bảng màu tập trung** trong `main.py`: Chuyển toàn bộ màu sắc thành `ListProperty` (hơn 50 màu) gắn trực tiếp vào `MDApp`, giúp KV Language bind tự động khi đổi theme mà không cần reload widget. Mỗi nhóm màu có ý nghĩa rõ: nền, card thuốc (taken/normal/unlocked), chữ, trạng thái, nút, alarm popup, chatbot bubble, crosscheck form, settings, header home, search bar, điểm thưởng/streak.
- **Tách `ThemeManager`** thành module riêng (`theme_manager.py`) — singleton `EventDispatcher` quản lý trạng thái Light/Dark. Khi theme đổi → dispatch event → `_on_theme_changed` trong `MDApp` cập nhật cả `theme_cls.theme_style` lẫn toàn bộ `ListProperty` màu sắc.
- **`AppColors` class** với ~30 method tĩnh trả về màu theo chế độ Light/Dark, bao phủ mọi nhóm UI (background, surface, card, text, status, button, alarm, chatbot, crosscheck).
- Sửa lỗi **app đơ sau khi quét đơn thuốc**: nguyên nhân là `MDDialog` button lambda capture biến `self._dialog` trước khi được gán → `None.dismiss()` crash. Đã viết lại `_show_dialog` và `_dialog_dismiss_then` đúng thứ tự khởi tạo → gắn callback.

### 4.3. Lưu trữ cài đặt
- **`app_settings.py`** — lưu trữ JSON vào file `.app_settings.json` cùng thư mục `main.py`, có `fsync` để Windows đảm bảo ghi đĩa. Hỗ trợ: Dark Mode, âm thanh báo thức, rung, chọn file chuông. Mỗi lần chạy app đều đọc lại → cài đặt không bị mất khi tắt/mở app.

### 4.4. Quản lý ngày giờ cho test
- **`app_datetime.py`** — hệ thống mốc ảo có tính năng tự động cộng thời gian thực trôi qua (`anchor_virtual + monotonic`). Admin có thể set `.app_datetime` để test alarm ở bất kỳ mốc nào mà đồng hồ vẫn "chạy" bình thường. Hỗ trợ nhiều format ngày (`dd/mm/yyyy`, `yyyy-mm-dd`, có/không giây).

### 4.5. Hệ thống âm thanh báo thức
- **`sound_manager.py`** — quản lý phát nhạc qua `pygame.mixer`, fallback sang `winsound.Beep` nếu file không tồn tại. Hỗ trợ: phát preview ngắn, phát lặp alarm đến khi `stop_alarm()`, rung lặp qua `Clock.schedule_interval`, vibration platform-aware (Windows winsound / Android jnius). Danh sách hơn 40 file `.mp3` được scan tự động từ thư mục `assets/sounds/` và gán nhãn tiếng Việt.
- Thiết kế an toàn trạng thái: `_pygame_ready` flag tránh gọi `pygame.mixer.init()` nhiều lần gây crash trên Windows.

### 4.6. Alarm Service nhắc nhở uống thuốc
- **`services/alarm_service.py`** — kiểm tra giờ uống thuốc mỗi 10 giây (demo) hoặc 30 giây (production), so sánh với `get_app_now()`. Hỗ trợ 5 lần nhắc với khoảng cách tăng dần: 0s → 1p → 5p → 10p → 15p. Cơ chế `snooze` tính delay cho lần nhắc tiếp theo. Grace period 60 phút tránh nhắc những mốc quá xa. Dùng `Clock.schedule_interval` — tự động hủy khi `stop()`.

### 4.7. Màn hình Home và MedCard
- **`screens/home_screen.py`** — đồng hồ sống (`StringProperty` + `Clock.schedule_interval`), `MedCard` widget cho từng thuốc, `AlarmPopup` hiển thị thông tin khi alarm trigger. Nút "Đã uống" bật/tắt dựa trên `BooleanProperty` (taken/unlocked). Tích hợp `MDSnackbar` để phản hồi hành động người dùng.
- Navigation timing: dùng `Clock.schedule_once` cho `_start_alarm` và `load_schedule` tránh xung đột cùng frame với đóng dialog/animation.

### 4.8. Màn hình CrossCheck y khoa
- **`screens/crosscheck_screen.py`** — kiến trúc phân tầng widget: `MedEditCard` → `MedScheduleSlot` → `TimeSlotPopup`. Mỗi khung giờ là mốc chuẩn hóa y khoa (buổi cố định, theo bữa ăn, PRN). Form hoàn toàn chỉnh sửa được, checkbox bắt buộc xác nhận trước khi lưu. Kết hợp `MOCK_RESULT_SUCCESS` để demo.

### 4.9. Màn hình Settings
- **`screens/settings_screen.py`** — ba switch (Dark Mode, âm thanh, rung) kết nối trực tiếp `app_settings` và `ThemeManager`. Popup chọn âm thanh tự động scan thư mục `sounds/` và hiển thị nhãn tiếng Việt. Nút nghe thử preview 2.5 giây. Màu popup tự thích ứng Light/Dark theo bảng màu app.

### 4.10. Màn hình AI Scan (giao diện)
- **`screens/ai_scan_screen.py`** — chạy AI trên background thread (`threading.Thread`, daemon), kết quả schedule về main thread qua `Clock.schedule_once` để tránh crash. Tự động fallback về mock data khi `ai_core` lỗi.

### 4.11. Dữ liệu mock và ảnh đơn thuốc
- **`data/mock_data.py`** — hai bộ mock: `MOCK_RESULT_SUCCESS` (95% confidence, 3 loại thuốc với schedule chi tiết) và `MOCK_RESULT_FAILURE` (23% confidence). `SAMPLE_SCHEDULE` cho HomeScreen với 6 mốc thuốc trong ngày.
- **`data/prescription_image.py`** — tạo ảnh đơn thuốc giả lập 800×1050 px, 300 DPI bằng PIL. Font Times New Roman Bold, nội dung đầy đủ: header phòng khám, thông tin bệnh nhân, danh sách thuốc, lưu ý, chữ ký bác sĩ, đóng dấu "ĐÃ KÊ ĐƠN". Cache file PNG để không tạo lại mỗi lần chạy.

### 4.12. Navigation và ScreenManager
- 10 màn hình được đăng ký trong `main.py`: home, ai_scan, crosscheck, chatbot, settings, reminders, profile, schedule, notification. Navigation qua `self.manager.current = "screen_name"` ở mọi màn hình.

### 4.13. Shared store giữa các màn hình
- Tạo **`data/schedule_store.py`** — singleton `EventDispatcher` làm shared store giữa `CrossCheckScreen` (ghi) và `HomeScreen` / `AlarmService` (đọc), thay vì mỗi màn hình đọc `SAMPLE_SCHEDULE` tĩnh.
- Tạo **`data/med_icons.py`** — hàm `resolve_med_icon()` tự gán icon PNG theo tên thuốc, kết hợp `_normalize_schedule_row()` trong store để mọi dòng lịch lúc nào cũng có icon hiển thị.
- Cập nhật **`screens/crosscheck_screen.py`** → `save_schedule()` ghi vào store rồi mới `manager.current = "home"`.
- Cập nhật **`screens/home_screen.py`** và **`services/alarm_service.py`** → đọc từ `schedule_store.load()` thay vì `SAMPLE_SCHEDULE`.
- Sửa navigation timing: `_navigate_to_crosscheck` dùng `Clock.schedule_once` tránh xung đột cùng frame với đóng dialog/animation.

## 5. Điều học được
Callback trong Kivy/KivyMD cần cực kỳ cẩn thận về thứ tự tham chiếu biến — đặc biệt khi lambda capture biến instance trước khi nó được gán (closure capture lúc **định nghĩa**, không phải lúc **gọi**), dẫn đến lỗi `None.xxx()` rất khó debug.

## 6. Nếu làm lại, đổi gì?
Thiết kế data layer từ đầu với **Repository pattern** (lớp trung gian giữa UI và mock data), có hàm `save()` / `load()` đọc/ghi file JSON thực sự ngay từ đầu, thay vì để `SAMPLE_SCHEDULE` là biến tĩnh toàn cục rồi phải refactor lại sau.

## 7. AI giúp gì / AI sai (mislead) gì?
- **Giúp ích:** Nhận diện nhanh nguyên nhân gốc của hai lỗi: (1) `self._dialog` capture trước khi gán trong list comprehension của button, (2) CrossCheck lưu không đúng nơi nên Home luôn thấy data cũ. Đề xuất architecture singleton EventDispatcher cho shared store là pattern phù hợp.
- **Sai / Mislead:** Lần đầu đề xuất dùng `dlg.dismiss(force=True)` — tham số `force` không tồn tại trong KivyMD 1.2.0, phải thử mới biết và sửa lại. Cũng từng gợi ý tham số `note` cho `MedScheduleSlot` nhưng thực tế KV truyền theo positional, không phải keyword — phải đọc lại file mới thấy.
