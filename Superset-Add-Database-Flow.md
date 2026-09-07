# Luồng "+ Database" (thêm kết nối tới một data source mới)

Đây là action user vào **Settings → Data → Databases → "+ DATABASE"** để khai báo một nguồn dữ liệu Superset sẽ query (khác hoàn toàn với Metadata Database — DB này chỉ dùng để Superset chạy SQL Lab/chart tới). Xem kiến trúc tổng quan tại [Superset-Architecture.md](Superset-Architecture.md).

## Thành phần tham gia

| Thành phần | Vai trò trong action này |
|---|---|
| React frontend | Render modal chọn engine + form nhập connection |
| `supersetNode` (Flask/API) | Xử lý toàn bộ request — **duy nhất** pod tham gia, không đụng `supersetWorker`/Redis |
| SQLAlchemy + DB driver | Driver phải có sẵn trong image (cài bởi `bootstrapScript`: `postgres`, `bigquery`, `clickhouse-connect`, `elasticsearch`) |
| Metadata Database (PostgreSQL) | Lưu connection mới vào bảng `dbs` + permission mới vào `ab_permission`/`ab_view_menu` |
| Data warehouse đích | Chỉ bị chạm tới khi "Test Connection" hoặc khi query thật sự sau này |
| Flask-AppBuilder Security Manager | Tự sinh permission `database access on [tên DB]` để gán quyền qua RBAC |

## Luồng chi tiết

1. Frontend gọi `GET /api/v1/database/available/` → liệt kê engine mà driver đã cài hỗ trợ (đúng 4 loại theo `bootstrapScript` hiện tại: Postgres, BigQuery, ClickHouse, Elasticsearch) → render form tương ứng (mỗi engine có field khác nhau, vd BigQuery cần upload service-account JSON).
2. User nhập host/port/user/password/dbname (hoặc paste thẳng SQLAlchemy URI) → bấm **Test Connection**.
3. Frontend `POST /api/v1/database/test_connection/` → `supersetNode` dựng SQLAlchemy engine bằng driver tương ứng, mở kết nối TCP **trực tiếp và đồng bộ** tới data warehouse đích (không qua Celery/Redis) → trả OK/lỗi ngay. → Về hạ tầng: pod `supersetNode` phải có network egress tới được DB đích (NetworkPolicy/security group phải mở).
4. User bấm **Connect/Finish** → frontend `POST /api/v1/database/` → backend:
   - Validate lại connection.
   - Mã hoá password/credential bằng `SECRET_KEY` (chính là giá trị ở `configOverrides.secret` trong `values-superset.yaml`) trước khi lưu.
   - Ghi 1 row mới vào bảng `dbs` (Metadata DB): `sqlalchemy_uri` (password đã obfuscate), `extra` (JSON: engine params...), `encrypted_extra` (credential dạng JSON, vd BigQuery service account).
   - Security Manager tạo permission mới gắn với DB này trong Metadata DB để có thể gán role/user truy cập.
5. DB connection giờ xuất hiện để tạo Dataset → Chart → Dashboard. Khi user thật sự chạy query dùng DB này (SQL Lab, load chart), `supersetNode`/`supersetWorker` mới dùng driver + URI đã lưu để query trực tiếp vào data warehouse — **Redis (cache/Celery) chỉ tham gia từ bước này trở đi, không tham gia lúc "+ Database"**.

## Lưu ý riêng cho deployment này

- Muốn thêm engine mới (MySQL, Snowflake, Trino...) phải sửa `bootstrapScript` để cài thêm driver rồi rolling-restart pod — không cấu hình được chỉ qua UI.
- `SECRET_KEY` là chìa khoá mã hoá toàn bộ password/credential đã lưu trong Metadata DB — nếu đổi `SECRET_KEY` sau khi đã có connection, các connection cũ **không giải mã được nữa** (phải nhập lại password/credential).
- Test connection và query thật đều đi thẳng từ pod `supersetNode`/`supersetWorker` ra ngoài cluster tới data warehouse đích — cần đảm bảo NetworkPolicy/Security Group cho phép egress phù hợp.
