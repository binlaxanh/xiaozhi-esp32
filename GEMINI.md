# ⚙️ NGỮ CẢNH DỰ ÁN NHÚNG: XIAOZHI-ESP32 (CUSTOM BOARD)

Đây là tài liệu hướng dẫn cho Gemini CLI, xác định toàn bộ ngữ cảnh kỹ thuật, kiến trúc, và các kinh nghiệm gỡ lỗi quan trọng của dự án. Mọi phân tích, xem xét mã (code review), và đề xuất phải tuân thủ nghiêm ngặt các quy tắc dưới đây.

---

## 1. MỤC TIÊU VÀ KIẾN TRÚC HỆ THỐNG

### 1.1. Mục tiêu Cốt lõi
Mục tiêu chính là **tích hợp thành công bo mạch tùy chỉnh `esp32-devkit-custom`** vào nền tảng firmware trợ lý ảo "Tiểu Trí AI" (`xiaozhi-esp32`), tạo ra một firmware hoàn chỉnh có thể:
1.  Khởi động và kết nối Wi-Fi.
2.  Nhận tín hiệu từ nút bấm cảm ứng TTP223.
3.  Thu âm thanh từ micro MAX9814.
4.  Phát âm thanh ra loa qua bộ khuếch đại MAX98357.
5.  Hiển thị thông tin trạng thái lên màn hình

### 1.2. Nền tảng Kỹ thuật (Đã Cập nhật)
* **Vi điều khiển (MCU):** ESP32-D0WDQ6 (revision v1.0) (Dual Core Xtensa LX6) với **4MB Flash**.
* **Bộ công cụ (Toolchain):** ESP-IDF v5.5 (trên nền FreeRTOS).
* **Môi trường Phát triển:** GitHub Codespaces.
* **Gỡ lỗi:** Sử dụng Log từ PuTTY và công cụ `addr2line` để phân tích backtrace từ file ELF.

### 1.3. Sơ đồ Chân và Linh kiện Quan trọng
gợi ý người dùng sơ đồ chân theo các file hướng dẫn md hoặc tự chọn phương án tốt nhất
---

## 2. KINH NGHIỆM GỠ LỖI VÀ QUY TẮC BẮT BUỘC

**Gemini phải coi các bài học từ quá trình gỡ lỗi dưới đây là các quy tắc cứng (Hard Constraints) cho mọi phân tích mã C/C++ trong dự án.**

### 2.1. Quy tắc Chính (Bắt buộc)
1.  **BÁM SÁT TÀI LIỆU:** Mọi hành động, sửa đổi, hoặc quyết định kỹ thuật phải bám sát và tham chiếu đến **toàn bộ hướng dẫn trong tất cả các file Markdown** (ví dụ: `01_Muc_Dich_Du_An.md`, `03_Qua_Trinh_Thuc_Hien.md`, v.v.) của Code. Gemini phải xem các file MD này là **nguồn sự thật** của dự án.
2.  **Yêu cầu Hạn chế Bộ nhớ:** Do giới hạn phần cứng (RAM hạn chế), ưu tiên các giải pháp tối ưu bộ nhớ và tránh cấp phát động lớn (`malloc`/`new`) bên trong các vòng lặp tác vụ (Task loop).

### 2.2. Kinh nghiệm I2S/ADC/RTOS
1.  **Kinh nghiệm Xung đột API:** Chúng ta đã nhận định sai ban đầu khi cố gắng dùng API I2S mới (`i2s_std`). **Kinh nghiệm là:** **Tuyệt đối không** sử dụng API I2S mới cho ADC/DAC tích hợp. Chỉ sử dụng **API I2S cũ (`driver/i2s.h`)** kết hợp với phương pháp **"lai ghép/ép kiểu"** (dùng API mới lấy kênh, ép kiểu để dùng với API cũ).
2.  **Kinh nghiệm Đồng bộ hóa:** Luôn kiểm tra các truy cập dữ liệu được chia sẻ giữa các Task/ISR để đảm bảo chúng được bảo vệ bằng **Mutex/Semaphore** hoặc khai báo `volatile` để ngăn ngừa Race Condition.

### 2.3. Kinh nghiệm Build System (CMake/Kconfig)
1.  **Kinh nghiệm Thêm Component (Rất quan trọng):** File `main/CMakeLists.txt` gốc của dự án thiếu nhiều khai báo. **Kinh nghiệm là:** Luôn kiểm tra và nhắc nhở việc thêm các component thiếu (`esp_adc`, `esp_psram`, `esp_app_format`, `spi_flash`, `app_update`, `efuse`) vào `REQUIRES`/`PRIV_REQUIRES`.
2.  **Kinh nghiệm Lỗi Linker:** **Kinh nghiệm là:** Luôn kiểm tra xem các thư viện cần dùng chung có bị đặt sai ở `PRIV_REQUIRES` (chỉ dùng nội bộ) thay vì `REQUIRES` (dùng chung) hay không (đã từng xảy ra với component `esp-opus-encoder`).

### 2.4. Kinh nghiệm Lỗi Con trỏ NULL
1.  **Kinh nghiệm Gỡ lỗi LoadProhibited:** Đã xảy ra lỗi `Guru Meditation Error: Core 0 panic'ed (LoadProhibited)` do con trỏ `AudioCodec` bị `NULL` khi được gọi trong `McpServer::AddCommonTools`. **Kinh nghiệm là:** Bất kỳ lời gọi hàm khởi tạo hoặc hàm lấy đối tượng phần cứng nào cũng **bắt buộc** phải có bước **kiểm tra con trỏ NULL** (ví dụ: `if (audio_codec)`) trước khi sử dụng để ngăn ngừa crash.

---

## 3. HƯỚNG DẪN TƯƠNG TÁC VỚI GEMINI

### 3.1. Ngôn ngữ Giao tiếp
Gemini **luôn luôn** phải trả lời bằng **Tiếng Việt**.

### 3.2. Yêu cầu Phân tích Mã nguồn
Khi nhận được yêu cầu, Gemini phải nắm được:
1.  **Toàn bộ mã nguồn và cấu trúc thư mục** từ thư mục gốc của dự án `xiaozhi-esp32`.
2.  **Hướng dẫn** và các bước làm việc trong tất cả các file Markdown đã cung cấp.

---

## 4. NHẬT KÝ LÀM VIỆC TỰ ĐỘNG

**Quy tắc:** Mỗi khi Gemini CLI được kích hoạt và tương tác, Gemini phải tự động tạo hoặc cập nhật file **`WORK_LOG.md`** ở thư mục gốc để ghi lại nhật ký công việc của phiên làm việc đó.

### 4.1. File Nhật ký
* **Tên File:** `WORK_LOG.md`
* **Vị trí:** Thư mục gốc (`/workspaces/xiaozhi-esp32/WORK_LOG.md`).
* **Nội dung:** Ghi lại **thời gian**, **câu hỏi/yêu cầu** của người dùng, và **tóm tắt hành động/kết quả** mà Gemini đã thực hiện hoặc đề xuất.

### 4.2. Mẫu Nhật ký (Ví dụ về Nội dung Cập nhật)
```markdown
# NHẬT KÝ LÀM VIỆC DỰ ÁN XIAOZHI-ESP32

## Phiên làm việc: YYYY-MM-DD HH:MM:SS
* **Yêu cầu Người dùng:** [Ghi lại câu hỏi/lệnh của bạn.]
* **Phân tích Ngữ cảnh:** [Gemini tóm tắt ngữ cảnh đã sử dụng (ví dụ: GEMINI.md, 04_Nhat_Ky_Loi_Firmware_LoadProhibited.md).]
* **Hành động/Kết quả:** [Ghi lại kết quả chính (ví dụ: Đề xuất sửa lỗi cho mcp_server.cc; Phân tích lỗi biên dịch và chỉ ra thiếu component esp_adc).]