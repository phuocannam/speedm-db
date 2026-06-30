# Cấu hình tải road_speed.db qua GitHub Release

Tài liệu ghi lại cách app SpeedM tải file database `road_speed.db` từ GitHub Release (private repo) thay vì đóng gói trong `assets/`, nhằm giảm size APK khi build/publish.

## Tổng quan luồng hoạt động

1. App khởi động → gọi GitHub API lấy thông tin **release mới nhất** của repo chứa DB.
2. So sánh `tag_name` của release với tag đã lưu trong `SharedPreferences` (lần tải trước).
3. Nếu giống nhau và file DB đã có trên máy → dùng luôn, không tải lại (`UpToDate`).
4. Nếu khác (có bản DB mới) hoặc chưa có file → tải file `road_speed.db` đính kèm trong release về `filesDir`, có báo % tiến trình.
5. Sau khi tải xong → verify (đúng kích thước + mở được SQLite + đọc được bảng `meta`) trước khi ghi đè file chính thức, tránh hỏng DB đang dùng nếu mất mạng giữa chừng.
6. Lưu `tag_name` mới vào `SharedPreferences` để lần sau so sánh.

Code chính nằm ở: `DatabaseHelper.kt` (package `com.example.speedm.mapmatcher`).

## Thông tin repo lưu trữ DB

| Thông tin | Giá trị |
|---|---|
| Repo | `phuocannam/speedm-db` |
| Trạng thái | Private |
| File asset | `road_speed.db` |
| Tag mẫu đã tạo | `db-v2026.06.30-1512` |

## Quy ước đặt tag cho release mới

GitHub không chấp nhận tag có nhiều dấu chấm liên tiếp gây lỗi *"tag name is not well-formed"*. Dùng định dạng:

```
db-v{YYYY}.{MM}.{DD}-{HHmm}
```

Ví dụ: `db-v2026.06.30-1512` (build lúc 15:12 ngày 30/06/2026).

> Lưu ý: khi gõ tag trong form tạo release, phải **bấm chọn dòng gợi ý "Create new tag: ... on publish"** hiện ra bên dưới ô nhập — nếu chỉ gõ chữ mà không chọn, GitHub sẽ báo lỗi tag không hợp lệ.

## Quy trình publish 1 bản DB mới

1. Build DB mới theo `DB_BUILD_GUIDE.md` (`export_sqlite.py` / `osm2pgsql_speedm.lua`).
2. Vào `https://github.com/phuocannam/speedm-db/releases/new`.
3. **Choose a tag** → gõ tag theo quy ước ở trên → chọn dòng "Create new tag" hiện ra.
4. **Release title** → đặt tên tuỳ ý (ví dụ trùng với tag).
5. **Release notes** (không bắt buộc nội dung gì đặc biệt, gợi ý ghi để dễ tra cứu sau):
   ```
   Database build {ngày}

   - Nguồn: OSM export theo Thông tư 38/2024/TT-BGTVT
   - Build bằng export_sqlite.py / osm2pgsql_speedm.lua
   - created_at (meta table): {giá trị created_at trong DB}
   ```
6. Kéo thả file `road_speed.db` vào khung **"Attach binaries by dropping them here or selecting them"**.
7. Đợi upload xong (566MB tuỳ mạng có thể mất vài phút) → bấm **Publish release**.

Sau khi publish, app sẽ tự động phát hiện tag mới ở lần mở tiếp theo và tải về — **không cần sửa code hay build lại app** mỗi lần có DB mới.

## Personal Access Token (PAT)

Vì repo private, app cần token để gọi GitHub API và tải asset.

| Thiết lập | Giá trị đã chọn |
|---|---|
| Loại token | Fine-grained PAT |
| Repository access | Only select repositories → `speedm-db` |
| Permissions | Contents: **Read-only** (không cấp quyền nào khác) |
| Token name | `speedm-db-readonly` |
| Expiration | 90 ngày (giai đoạn dev/test — xem ghi chú bên dưới) |

Tạo/quản lý token tại: `https://github.com/settings/personal-access-tokens`

### ⚠️ Cảnh báo bảo mật

Token được nhúng cứng (hardcode) trong `DatabaseHelper.kt` → nằm trong APK đã build → **có thể bị trích xuất qua decompile**. Vì token chỉ có quyền đọc (Read-only) trên đúng 1 repo chỉ chứa file DB, rủi ro nếu lộ là thấp (chỉ tải được file DB, không sửa/xoá được gì). Tuy vậy:

- Không bao giờ dùng token classic có quyền rộng (toàn tài khoản) cho việc này.
- Nếu cần an toàn hơn về sau, nên đổi sang kiến trúc proxy: app gọi 1 serverless function (vd Cloudflare Worker) giữ token phía server, thay vì gọi GitHub trực tiếp từ app.
- Vì token đã từng được dán trực tiếp trong một đoạn chat để nhờ điền vào code, nên cân nhắc **revoke token cũ và tạo token mới** trước khi release chính thức ra Play Store, để chắc chắn không có bản sao nào của token nằm ngoài tầm kiểm soát.

### Khi token hết hạn (Expiration)

GitHub sẽ gửi email nhắc trước khi token hết hạn. Khi đó:

1. Tạo token mới (cùng thiết lập Contents: Read-only trên repo `speedm-db`).
2. Cập nhật hằng số `GITHUB_TOKEN` trong `DatabaseHelper.kt`.
3. Build lại app và release bản cập nhật.

Nếu app đã có nhiều người dùng phụ thuộc vào tính năng tự tải DB, nên cân nhắc chọn **No expiration** cho token (nếu tổ chức GitHub cho phép) để tránh gián đoạn, kèm tự nhắc bản thân theo dõi định kỳ — token không hết hạn vẫn có thể tự thu hồi (revoke) bất kỳ lúc nào nếu nghi ngờ bị lộ.

## Các hằng số cần cập nhật trong `DatabaseHelper.kt`

```kotlin
private const val GITHUB_OWNER = "phuocannam"
private const val GITHUB_REPO  = "speedm-db"
private const val GITHUB_TOKEN = "github_pat_..."   // PAT Contents:Read-only
```

## Checklist khi setup trên máy/app mới

- [ ] Repo `speedm-db` (hoặc repo tương đương) đã tạo, private, có ít nhất 1 release với asset `road_speed.db`.
- [ ] Fine-grained PAT đã tạo, scope đúng repo, quyền Contents: Read-only.
- [ ] `DatabaseHelper.kt` đã điền đúng `GITHUB_REPO` và `GITHUB_TOKEN`.
- [ ] `AndroidManifest.xml` đã có quyền `INTERNET` và `ACCESS_NETWORK_STATE`.
- [ ] File `road_speed.db` đã **xoá khỏi** `app/src/main/assets/` (không còn bundle trong APK).
- [ ] Không còn đoạn code nào tham chiếu `context.assets.open("road_speed.db")` hoặc tương tự (kiểm tra bằng search `assets.open` / `AssetManager` toàn project).

## File liên quan

- `DatabaseHelper.kt` — logic tải/verify/mở DB.
- `DbMeta.kt` — đọc `created_at` từ bảng `meta` để hiển thị version DB lên UI debug.
- `AndroidManifest.xml` — khai báo quyền `INTERNET`, `ACCESS_NETWORK_STATE`.
- `DB_BUILD_GUIDE.md` — hướng dẫn build file `road_speed.db` từ đầu (OSM data).
