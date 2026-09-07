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

## Chi tiết bước 5: "dùng driver + URI đã lưu để query trực tiếp vào data warehouse"

Đây là bước xảy ra **mỗi lần** user chạy SQL Lab hoặc load một chart/dashboard dùng DB connection đó — không phải một bước một lần như "+ Database". Cụ thể:

1. **Lấy connection**: Superset load record `Database` từ bảng `dbs` (Metadata DB) theo `database_id` mà chart/dataset/SQL Lab tab đang tham chiếu.
2. **Giải mã + build engine** (hàm `Database.get_sqla_engine()` trong code Superset):
   - Giải mã `sqlalchemy_uri`/`encrypted_extra` bằng `SECRET_KEY` để lấy lại password/credential gốc.
   - Áp thêm `extra` JSON đã lưu (connect args, engine params, timeout...) và ghép thành SQLAlchemy engine hoàn chỉnh, dùng đúng driver đã cài (`psycopg2` cho Postgres, `clickhouse-connect`, `google-cloud-bigquery`, `elasticsearch-dbapi`...).
   - Engine này được **cache trong process** (theo `database_id` + schema + user, để hỗ trợ per-user impersonation) — không tạo lại từ đầu mỗi request, connection pool (SQLAlchemy `QueuePool`) được tái sử dụng.
3. **Build câu SQL** — 2 trường hợp khác nhau:
   - **SQL Lab**: dùng nguyên văn SQL user gõ, chỉ validate bằng `sqlparse` (chặn statement nguy hiểm nếu DB không cho phép DML/DDL) và tự thêm `LIMIT` theo cấu hình `ROW_LIMIT` nếu user chưa set.
   - **Chart/Dashboard**: Superset lấy định nghĩa Dataset (bảng vật lý hoặc virtual SQL) + config của chart (metrics, group by, filter, time range, order) rồi **tự sinh câu SQL** qua lớp `SqlaTable`/query object (SQLAlchemy Core) — có thể chèn thêm điều kiện Row-Level Security (RLS) dựa theo role của user đang xem.
4. **Thực thi**:
   - **Đồng bộ**: `supersetNode` tự mở connection từ pool, chạy câu SQL, đợi kết quả (chặn request cho tới khi xong).
   - **Bất đồng bộ** (SQL Lab async / `GLOBAL_ASYNC_QUERIES`): task được đẩy qua Celery/Redis, `supersetWorker` là process thực sự mở connection + chạy SQL — logic build engine/query giống hệt, chỉ khác process nào gọi.
   - Giao thức mạng thực tế phụ thuộc driver: Postgres/ClickHouse thường qua TCP tới port DB; BigQuery/Elasticsearch thực chất là gọi REST API qua HTTPS ra ngoài internet/GCP, không phải TCP thuần.
5. **Trả kết quả**: rows trả về được nạp vào `pandas.DataFrame`, format lại (pivot, đổi tên cột, áp post-processing của chart) rồi serialize JSON trả cho frontend; nếu là query đồng bộ không lỗi thì kết quả này mới được ghi vào **cache Redis** (theo `defaultTimeout`) để lần load sau khỏi phải lặp lại toàn bộ bước 1-4.

### Ý nghĩa hạ tầng cho deployment này

- Cả `supersetNode` **và** `supersetWorker` đều cần cài driver (qua `bootstrapScript` chạy ở cả 2 command) và cần network egress ra data warehouse đích — không chỉ riêng `supersetNode`.
- Nếu data warehouse là BigQuery/Elasticsearch (REST API), egress cần cho phép HTTPS ra ngoài (internet/GCP), khác với Postgres/ClickHouse nội bộ cluster chỉ cần mở TCP tới đúng port.
- Đây là lý do `supersetWorker` cũng có init container `wait-for-postgres-redis` — nó cần cả Metadata DB (đọc connection info) lẫn Redis (nhận task/ghi kết quả), rồi mới tự mở kết nối riêng tới data warehouse đích khi thực thi.

## Lưu ý riêng cho deployment này

- Muốn thêm engine mới (MySQL, Snowflake, Trino...) phải sửa `bootstrapScript` để cài thêm driver rồi rolling-restart pod — không cấu hình được chỉ qua UI.
- `SECRET_KEY` là chìa khoá mã hoá toàn bộ password/credential đã lưu trong Metadata DB — nếu đổi `SECRET_KEY` sau khi đã có connection, các connection cũ **không giải mã được nữa** (phải nhập lại password/credential).
- Test connection và query thật đều đi thẳng từ pod `supersetNode`/`supersetWorker` ra ngoài cluster tới data warehouse đích — cần đảm bảo NetworkPolicy/Security Group cho phép egress phù hợp.
