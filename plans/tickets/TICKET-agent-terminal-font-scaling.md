# [TICKET / RFC] Dynamic Font Scaling & Zoom for Agent PTY Terminal (HiDPI / 2K / 4K Adaptation)

- **Ticket ID**: `TICKET-AGENT-TERM-FONT-01`
- **Component**: `crates/designos-app` (`terminal.rs`, `theme.rs`, `shell.rs`)
- **Type**: Improvement / UX Enhancement
- **Priority**: P1 (Chất lượng trải nghiệm đọc & làm việc với Agent CLI)
- **Reporter**: QA / User Feedback
- **Target Release**: v0.6.x

---

## 1. Bối cảnh & Vấn đề hiện tại (Problem Statement)

### 1.1 Hiện tượng
Trên các màn hình độ phân giải cao (2K 2560×1440, 4K 3840×2160) chạy Windows với Display Scale 100% (`pixels_per_point = 1.0`):
- Nội dung văn bản trong khung Agent (chạy Antigravity CLI, Claude Code, Codex, v.v.) hiển thị **rất bé** (~2.6 mm chiều cao trên màn hình 27"), gây mỏi mắt và khó đọc các khối mã lệnh, kế hoạch suy luận (thoughts) hoặc bảng so sánh.
- Kéo giãn độ rộng của bảng điều khiển Agent (Side Panel width từ 260px lên 700px+) chỉ làm tăng số cột (`cols` lên 80–120 ký tự/dòng) chứ **không làm to cỡ chữ**.
- Các phím tắt thu phóng thông thường (`Ctrl + =` / `Ctrl + -`) chỉ điều khiển zoom của Canvas vẽ, hoàn toàn không hỗ trợ zoom khung Terminal hay UI Chrome.

### 1.2 Nguyên nhân kỹ thuật trong mã nguồn
Trong [`crates/designos-app/src/terminal.rs`](file:///D:/Github/design-os-studio/crates/designos-app/src/terminal.rs):
1. **Kích thước ô ký tự bị gán cứng (Hardcoded Constants)**:
   ```rust
   // terminal.rs:58-59
   const CHAR_W: f32 = 7.8;
   const ROW_H: f32 = 16.0;
   ```
2. **Cỡ font Monospace cố định**:
   ```rust
   // terminal.rs:5905
   let font_id = FontId::new(11.5, FontFamily::Monospace);
   ```
3. **Tính toán lưới cột/dòng không có tham số cỡ chữ**:
   ```rust
   // terminal.rs:5673-5674
   let cols = ((grid_w / CHAR_W).floor() as u16).clamp(20, 320);
   let rows = ((grid_h / ROW_H).floor() as u16).clamp(4, 120);
   ```
   Do `CHAR_W` và `ROW_H` là hằng số tĩnh, font size 11.5pt không thể thích ứng linh hoạt theo độ phân giải màn hình hoặc nhu cầu của người dùng.

---

## 2. Giải pháp kiến trúc đề xuất (Proposed Technical Design)

### 2.1 Phương án A (Ưu tiên): Dynamic Terminal Font Size & Zoom
Cho phép khung Terminal tự chủ động điều chỉnh cỡ font và tính toán lại hình học ô (`CHAR_W`, `ROW_H`) theo font metrics thực tế:

1. **Thêm thuộc tính cấu hình cỡ chữ vào `TerminalState` / `TerminalTab`**:
   ```rust
   pub struct TerminalState {
       // ...
       pub font_size: f32, // Mặc định: 13.0 (hoặc tự động 14.0 nếu native_pixels_per_point == 1.0 && width >= 2560)
   }
   ```
   * Thang cỡ chữ hỗ trợ: `10.0pt` đến `24.0pt` (bước nhảy `1.0pt`).
   * Lưu giá trị vào `settings.json` (thông qua `designos_library`) để duy trì qua các lần mở app.

2. **Động hóa `CHAR_W` và `ROW_H` theo font metrics**:
   Thay vì dùng hằng số cố định, kích thước ô ký tự được đo trực tiếp từ `egui::Painter` hoặc font layout galley của ký tự mẫu `'M'`:
   ```rust
   let font_id = FontId::new(state.font_size, FontFamily::Monospace);
   let char_w = widgets::text_width(&p, "M", &font_id).max(6.0);
   let row_h = (state.font_size * 1.38).round(); // Đảm bảo line-height cân đối không bị đè dòng
   ```

3. **Phím tắt tương tác thu phóng (Keyboard & Wheel Zoom)**:
   * Khi con trỏ chuột hover hoặc focus bên trong `viewport_rect` của Terminal:
     - `Ctrl + Cuộn chuột lên`: Tăng `font_size += 1.0` (tối đa `24.0`).
     - `Ctrl + Cuộn chuột xuống`: Giảm `font_size -= 1.0` (tối thiểu `10.0`).
     - `Ctrl + Plus` / `Ctrl + Minus` (hoặc `Cmd` trên macOS): Phóng to / Thu nhỏ cỡ chữ terminal.
     - `Ctrl + 0`: Reset về kích thước mặc định (`13.0pt`).
   * Khi thay đổi cỡ chữ, hàm `tab.resize(new_rows, new_cols)` tự động phát tín hiệu `SIGWINCH` đến tiến trình CLI (PTY) để PTY tự wrap lại nội dung vừa vặn với kích thước mới mà không bị tràn viền.

### 2.2 Phương án B (Bổ trợ): Hỗ trợ UI Scale Factor cho toàn bộ Chrome
- Cho phép người dùng chỉnh hệ số zoom toàn giao diện Studio (`ctx.set_zoom_factor(scale)`) với các mốc: `100%`, `125%`, `150%`.

---

## 3. Tiêu chí nghiệm thu (Acceptance Criteria - AC)

| Mã AC | Tiêu chí nghiệm thu | Phương thức kiểm tra |
|---|---|---|
| **AC-01** | Cỡ chữ mặc định của Terminal nâng lên tối thiểu **13.0pt** (hoặc tự động tăng lên 14.0pt trên màn hình 2K/4K) để đọc rõ ràng không mỏi mắt. | Visual inspection trên màn hình 2560×1440. |
| **AC-02** | Hỗ trợ phím tắt `Ctrl + Cuộn chuột` hoặc `Ctrl + =` / `Ctrl + -` khi focus vào Terminal để tăng/giảm font size từ 10pt đến 24pt. | Thao tác tương tác trực tiếp & Unit test sự kiện phím/chuột. |
| **AC-03** | Khi cỡ font thay đổi, lưới ký tự (`cols`, `rows`) được tính toán lại ngay lập tức và đồng bộ vào PTY qua `tab.resize()`, không làm vỡ giao diện TUI/ANSI. | Kiểm thử hiển thị Antigravity CLI / Claude Code khi zoom. |
| **AC-04** | Cỡ chữ đã chọn được ghi nhớ trong cấu hình người dùng (`settings.json`) và giữ nguyên khi mở lại ứng dụng. | Khởi động lại Studio và xác nhận cỡ font không bị reset. |
| **AC-05** | Đảm bảo tuân thủ nguyên tắc hiệu năng: Không gây giật lag, không bump `doc.rev`, duy trì mức CPU nghỉ `0.00%` khi không tương tác. | Đo đạc CPU delta (`ps -o time=`) trong 20s không đổi. |

---

## 4. Tệp cần chỉnh sửa & Đánh giá rủi ro

* **Tệp tác động chính**:
  - [`crates/designos-app/src/terminal.rs`](file:///D:/Github/design-os-studio/crates/designos-app/src/terminal.rs): Xóa bỏ hằng số `CHAR_W` / `ROW_H` cứng, đưa `font_size` vào `TerminalState`, lắng nghe phím tắt zoom và gắn `tab.resize()`.
  - [`crates/designos-app/src/theme.rs`](file:///D:/Github/design-os-studio/crates/designos-app/src/theme.rs): Thêm hằng số ngưỡng `DEFAULT_TERM_FONT_SIZE = 13.0;`.
* **Rủi ro hồi quy (Regression Risks)**:
  - Một số test case golden layout của terminal có thể cần cập nhật nếu test dựa trên số hàng/cột cứng với `CHAR_W = 7.8`. Giải pháp: Giữ test harness sử dụng font size mặc định xác định.
