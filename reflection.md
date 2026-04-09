# Individual Reflection — Đào Hồng Sơn (2A202600462)

## 1. Role (Vai trò)
Phát triển toàn bộ logic backend & kết nối dữ liệu giữa các màn hình; sửa lỗi nghiêm trọng khiến app bị treo/crash.

## 2. Phần phụ trách cụ thể
- Sửa lỗi **app đơ sau khi quét đơn thuốc**: nguyên nhân là `MDDialog` button lambda capture biến `self._dialog` trước khi được gán → `None.dismiss()` crash. Đã viết lại `_show_dialog` và `_dialog_dismiss_then` đúng thứ tự khởi tạo → gắn callback.
- Tạo **`data/schedule_store.py`** — singleton `EventDispatcher` làm shared store giữa `CrossCheckScreen` (ghi) và `HomeScreen` / `AlarmService` (đọc), thay vì mỗi màn hình đọc `SAMPLE_SCHEDULE` tĩnh.
- Tạo **`data/med_icons.py`** — hàm `resolve_med_icon()` tự gán icon PNG theo tên thuốc, kết hợp `_normalize_schedule_row()` trong store để mọi dòng lịch lúc nào cũng có icon hiển thị.
- Cập nhật **`screens/crosscheck_screen.py`** → `save_schedule()` ghi vào store rồi mới `manager.current = "home"`.
- Cập nhật **`screens/home_screen.py`** và **`services/alarm_service.py`** → đọc từ `schedule_store.load()` thay vì `SAMPLE_SCHEDULE`.
- Sửa navigation timing: `_navigate_to_crosscheck` dùng `Clock.schedule_once` tránh xung đột cùng frame với đóng dialog/animation.
- Cập nhật toàn bộ chuỗi tiếng Việt có dấu trong `kv/ai_scan_screen.kv` (tiêu đề, trạng thái, cảnh báo).

## 3. SPEC mạnh/yếu
- **Mạnh nhất:** Thiết kế luồng AI scan → CrossCheck → Home có kiến trúc rõ ràng, chia rõ mock và thực. AlarmService tách biệt, có demo mode để test nhanh.
- **Yếu nhất:** Không có cơ chế lưu trữ dữ liệu bền (file JSON / SQLite) — mỗi lần khởi động lại app, lịch thuốc luôn quay về `SAMPLE_SCHEDULE` gốc. Không tương thích thực tế.

## 4. Đóng góp cụ thể khác (ngoài mục 2)
- Test và phát hiện lỗi `dismiss(force=True)` không tồn tại trong KivyMD 1.2.0 → sửa chỉ dùng `dismiss()`.
- Xác minh tất cả các import module mới bằng `python -c` sau mỗi lần sửa để đảm bảo không break app.

## 5. Điều học được
Callback trong Kivy/KivyMD cần cực kỳ cẩn thận về thứ tự tham chiếu biến — đặc biệt khi lambda capture biến instance trước khi nó được gán (closure capture lúc **định nghĩa**, không phải lúc **gọi**), dẫn đến lỗi `None.xxx()` rất khó debug.

## 6. Nếu làm lại, đổi gì?
Thiết kế data layer từ đầu với **Repository pattern** (lớp trung gian giữa UI và mock data), có hàm `save()` / `load()` đọc/ghi file JSON thực sự ngay từ đầu, thay vì để `SAMPLE_SCHEDULE` là biến tĩnh toàn cục rồi phải refactor lại sau.

## 7. AI giúp gì / AI sai (mislead) gì?
- **Giúp ích:** Nhận diện nhanh nguyên nhân gốc của hai lỗi: (1) `self._dialog` capture trước khi gán trong list comprehension của button, (2) CrossCheck lưu không đúng nơi nên Home luôn thấy data cũ. Đề xuất architecture singleton EventDispatcher cho shared store là pattern phù hợp.
- **Sai / Mislead:** Lần đầu đề xuất dùng `dlg.dismiss(force=True)` — tham số `force` không tồn tại trong KivyMD 1.2.0, phải thử mới biết và sửa lại. Cũng từng gợi ý tham số `note` cho `MedScheduleSlot` nhưng thực tế KV truyền theo positional, không phải keyword — phải đọc lại file mới thấy.
