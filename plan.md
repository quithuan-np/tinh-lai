# Plan: Kiểm thử và sửa lỗi trang tính lãi (`index.html`)

Trạng thái: **CHỜ DUYỆT**. Chưa sửa file nào ngoài plan.md.

## Inputs (đã điền)

| Trường | Giá trị |
|---|---|
| TASK | Kiểm thử và sửa lỗi trang tính lãi `index.html`: viết test trước, sửa lỗi tìm được, giữ trang chỉ một file và không cần build |
| OUTPUT | `/home/user/tinh-lai`, branch `claude/harness-tinh-lai` |
| CHECKS | Các dòng C1–C14 ở dưới. Chạy bằng `scripts/check.sh` |
| SOURCES | `index.html`, `README.md` (chưa có CLAUDE.md, test hay hook) |
| SCOPE | Chỉ trong repo này. Được sửa `index.html` và tạo `tests/`, `scripts/`, `.claude/settings.json`, `harness/`, `CLAUDE.md`, `plan.md`, `checks.md`, `progress.md`, `package.json`. Không push hay mở PR khi chưa hỏi |
| LIMIT | Mỗi bước sửa tối đa 3 lần, cả run tối đa 1 phiên. Vượt giới hạn thì trả trạng thái dở dang |

Công cụ có sẵn: Node 22, Playwright 1.56.1 (global, Chromium ở `/opt/pw-browsers`), Python 3.13. Không cần cài thêm gì.

## Lỗi đã thấy khi đọc code (chưa chạy)

| # | Lỗi | Vị trí |
|---|---|---|
| B1 | "Lãi mỗi ngày" được làm tròn khi hiển thị, nhưng "Tiền lãi" lại tính từ giá trị chưa làm tròn. Ví dụ 50.000.000 đ, 2%/tháng, 31 ngày: 33.333 × 31 = 1.033.323, trong khi trang hiện 1.033.333 (lệch 10 đ) | `calc()` |
| B2 | Vay và trả cùng ngày (0 ngày) vẫn được chấp nhận, nhưng thông báo lỗi lại ghi "Ngày trả phải **sau** ngày vay" | `calc()` |
| B3 | Số tiền quá 15 chữ số vượt `MAX_SAFE_INTEGER`, bị đổi giá trị mà không báo | listener `amt` |
| B4 | Dán "50,000,000.50" thì phần thập phân bị nối vào thành 5.000.000.050 | listener `amt` |
| B5 | Nhấn Enter không tính được vì ô nhập không nằm trong `<form>` | HTML |
| B6 | Lãi suất không có giới hạn trên (ví dụ nhập 5000%) | `calc()` |

## Cần bạn chốt trước khi chạy

- **D1 (B1):** Hiện "Lãi mỗi ngày" với 2 số lẻ (33.333,33 đ) để phép nhân khớp với tổng. *Đề xuất: đồng ý.*
- **D2 (B2):** Cho phép 0 ngày (lãi = 0) và đổi thông báo thành "Ngày trả không được trước ngày vay". *Đề xuất: đồng ý.*
- **D3 (B3/B4):** Giới hạn 15 chữ số. Khi dán có dấu thập phân thì bỏ phần lẻ sau dấu thập phân cuối. *Đề xuất: đồng ý.*
- **D4 (B5):** Bọc các ô nhập trong `<form>` để nhấn Enter là tính. *Đề xuất: đồng ý.*
- **D5 (B6):** Báo lỗi khi lãi suất trên 100%/tháng. *Đề xuất: đồng ý.* Không thêm cảnh báo pháp lý vì ngoài phạm vi.

## Các bước

Ký hiệu: ∥ là bước chạy song song được với bước khác.

### S0. Dựng harness
- Đọc: `index.html`
- Ghi: `CLAUDE.md`, `scripts/check.sh`, `scripts/syntax.sh`, `.claude/settings.json`, `package.json`
- Việc làm:
  - Tạo tag `pre-harness` để có bản phục hồi.
  - `scripts/syntax.sh`: tách `<script>` ra rồi chạy `node --check`.
  - `scripts/check.sh`: chạy syntax rồi `node --test tests/`.
  - Hook PostToolUse (Edit|Write trên `index.html`) gọi `scripts/syntax.sh`.
- Output: harness chạy được.
- Check: `scripts/check.sh` exit 0 trên `index.html` hiện tại (chưa có test). Cố ý làm hỏng cú pháp một lần thì hook phải chặn, sau đó hoàn tác.

### S1. Viết test trước (kỳ vọng ĐỎ đúng chỗ)
- Đọc: `index.html`, `CLAUDE.md`
- Ghi: `tests/calc.test.mjs` ∥ `tests/ui.test.mjs` (hai file độc lập)
- `calc`: test công thức qua UI thật (Playwright điền ô, đọc kết quả), gồm C1–C8.
- `ui`: test giao diện, gồm C9–C13.
- Output: hai file test.
- Check: `node --test tests/` chạy được, và chỉ các test của B1–B6 fail. Test nào khác fail nghĩa là test sai, phải sửa test.

### S2. Sửa lỗi theo D1–D5
- Đọc: `index.html`, kết quả fail của S1
- Ghi: chỉ `index.html` (một phiên duy nhất được sửa file này)
- Output: `index.html` đã sửa.
- Check: `scripts/check.sh` xanh toàn bộ.

### S3. Kiểm tra trực quan ∥ S4
- Ghi: `artifacts/screenshots/{light,dark}-{form,result,error}.png`
- Check: đủ 6 file, mở được. Tự xem ảnh, không có chữ bị cắt hay tràn ngang.

### S4. Review độc lập ∥ S3
- Subagent chỉ đọc, xem `git diff pre-harness` với mắt mới và trả về danh sách lỗi.
- Ghi: không ghi gì. Kết quả gộp vào `checks.md` trong phiên chính.
- Check: mỗi lỗi subagent tìm ra đều được sửa (quay lại S2, tối đa 3 lần) hoặc ghi vào checks.md là unresolved kèm lý do.

### S5. Hoàn tất
- Ghi: `checks.md`, `progress.md`, rule mới trong `CLAUDE.md`, `harness/TEMPLATE.md` (bản prompt tái sử dụng), cập nhật `README.md` (cách chạy test).
- Check:
  - Chạy lại `scripts/check.sh` trên commit cuối và được exit 0.
  - Mỗi dòng trong checks.md có verdict và evidence.
  - Commit lên branch. **Hỏi trước khi push.**

## Acceptance checks (viết trước khi làm)

| ID | Yêu cầu | Cách kiểm |
|---|---|---|
| C1 | 50.000.000 đ, 2%/tháng, 01/01/2026 → 31/01/2026 = 30 ngày, lãi 1.000.000 đ, gốc + lãi 51.000.000 đ | e2e |
| C2 | Năm nhuận: 28/02/2024 → 01/03/2024 = 2 ngày | e2e |
| C3 | Qua mốc đổi giờ (DST) vẫn ra số nguyên ngày; chạy với `TZ=America/New_York` | e2e |
| C4 | Lãi suất "1,5" và "1.5" cho cùng kết quả | e2e |
| C5 | "Lãi mỗi ngày" × số ngày khớp "Tiền lãi" sai số dưới 1 đ (B1) | e2e |
| C6 | Cùng ngày: ra 0 ngày, lãi 0 đ, không báo lỗi. Ngày trả trước ngày vay thì báo "không được trước" (B2) | e2e |
| C7 | Nhập quá 15 chữ số bị chặn. Dán "50,000,000.50" ra 50.000.000 (B3/B4) | e2e |
| C8 | Lãi suất trên 100 hoặc âm báo lỗi (B6) | e2e |
| C9 | Thiếu ngày, thiếu tiền hoặc thiếu lãi suất thì báo đúng thông báo, không hiện kết quả | e2e |
| C10 | Nhấn Enter trong ô lãi suất là tính (B5) | e2e |
| C11 | "Nhập lại" xóa hết ô, thông báo lỗi và kết quả | e2e |
| C12 | Ở 390×844, cả sáng lẫn tối: `scrollWidth <= clientWidth`, không cuộn ngang | e2e + ảnh |
| C13 | Không có lỗi console khi tải trang và khi tính | e2e |
| C14 | Script trong trang qua `node --check` | `scripts/syntax.sh` |

## Không làm (ngoài phạm vi)
Thêm tính năng mới (lãi kép, trả nhiều lần), đổi thiết kế, deploy hay publish, cảnh báo pháp lý về trần lãi suất.
