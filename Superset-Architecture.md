# Superset

Repo triển khai Apache Superset lên Kubernetes qua Helm chart (`Kubernetes-Apps/values-superset.yaml`) + ArgoCD (`Argocd-Apps/`).

## Kiến trúc hệ thống

Superset gồm 3 thành phần cốt lõi do Helm chart này quản lý/deploy, cộng thêm Celery worker/beat, (tuỳ chọn) một websocket service cho async query, và **Data Warehouse** — hệ thống nằm ngoài phạm vi deploy của chart này nhưng là đích đến của mọi query.

### 1. Superset Application (Flask + React)

- **Backend**: Python/Flask + Flask-AppBuilder (auth, RBAC/permission, REST API) + SQLAlchemy (kết nối tới các data source: Postgres, BigQuery, ClickHouse, Elasticsearch...).
- **Frontend**: React SPA (build bằng Webpack), gọi backend qua REST API.
- **Vai trò**: nhận request từ user (xem dashboard, chart, chạy SQL Lab), dịch thành SQL query gửi tới data warehouse của user, render kết quả.
- **Trong deployment này**:
  - `supersetNode` — Deployment chạy gunicorn, phục vụ web/API, port 8088.
  - `supersetWorker` — Celery worker, chạy task bất đồng bộ (autoscale 2-6 replicas theo CPU).
  - `supersetCeleryBeat` — scheduler cho Celery (đang `enabled: false`, chỉ cần khi dùng Alerts & Reports).
  - `supersetWebsockets` — service Node.js riêng, chỉ hoạt động khi feature flag `GLOBAL_ASYNC_QUERIES: true` (đang bật).

### 2. Metadata Database (PostgreSQL)

- Lưu: user/role/permission, database connections, định nghĩa datetaset/chart/dashboard, query history, log, lịch alert/report.
- **Không** lưu dữ liệu phân tích thực — dữ liệu business nằm ở data warehouse riêng mà Superset chỉ kết nối tới qua SQLAlchemy.
- Trong deployment: subchart `postgresql` (bitnamilegacy/postgresql 14.17), database/user `superset`, có PVC persistence.

### 3. Caching Layer (Redis)

Một Redis instance nhưng đảm nhiệm 3 vai trò khác nhau (phân tách bằng `cacheDb`/`celeryDb`):

- **Query/result cache**: cache kết quả query của chart/dashboard (`defaultTimeout: 86400`s, key prefix `superset_`) — load lại chart lần 2 sẽ lấy từ cache thay vì query lại data warehouse.
- **Celery broker + results backend**: hàng đợi task bất đồng bộ giữa `supersetNode` (producer) và `supersetWorker` (consumer) — dùng cho SQL Lab async query, report snapshot/email.
- **Redis Streams cho `GLOBAL_ASYNC_QUERIES`**: khi query nặng chạy qua Celery, `supersetWorker` ghi event kết quả vào một Redis Stream (prefix `async-events-`); `supersetWebsockets` đọc stream này và đẩy trực tiếp về browser qua WebSocket (thay vì browser phải polling), xác thực bằng JWT (`jwtSecret` + cookie `async-token`).

Trong deployment: subchart `redis` (bitnamilegacy/redis 7.0.10, standalone, `auth.enabled: false`, không bật persistence).

### 4. Data Warehouse (bên ngoài Superset)

- Là nơi lưu **dữ liệu phân tích thực** (business data) — hoàn toàn tách biệt với Metadata Database. Superset không sở hữu/quản lý hạ tầng này, chỉ kết nối tới qua SQLAlchemy (xem [Superset-Add-Database-Flow.md](Superset-Add-Database-Flow.md)).
- Trong scope ticket này: **PostgreSQL** và **ClickHouse**, khai báo qua **Settings → Data → Databases**, driver tương ứng cài bằng `bootstrapScript` (`.[postgres]`, `clickhouse-connect`).
- Cả `supersetNode` (query đồng bộ) và `supersetWorker` (query bất đồng bộ qua Celery) đều mở connection **trực tiếp** tới đây khi thực thi SQL Lab hoặc load chart/dashboard — không đi qua Redis/Metadata DB ở bước này, Redis/Metadata DB chỉ tham gia trước (đọc connection info, permission) và sau (ghi cache) query thật.
- Không nằm trong `values-superset.yaml` — không do chart này deploy/quản lý, cần đảm bảo NetworkPolicy/Security Group cho phép egress từ namespace `superset` tới đây.

## Luồng tương tác cơ bản

1. User → browser → `supersetNode` (qua Service/Ingress).
2. `supersetNode` xác thực/authorize qua Flask-AppBuilder, đọc metadata (dataset, chart, permission) từ **PostgreSQL**.
3. `supersetNode` kiểm tra **cache** (Redis) trước:
   - **Cache hit** → trả kết quả ngay.
   - **Cache miss, query đồng bộ** → `supersetNode` tự chạy SQL tới data warehouse, ghi kết quả vào cache, trả về browser.
   - **Cache miss, query bất đồng bộ** (SQL Lab async hoặc `GLOBAL_ASYNC_QUERIES`) → `supersetNode` đẩy task vào Celery queue (Redis broker) → `supersetWorker` nhận task, chạy query, ghi kết quả vào Redis (results backend) + ghi event vào Redis Stream → `supersetWebsockets` đọc stream, đẩy kết quả real-time về browser qua WebSocket.
4. `supersetCeleryBeat` (nếu bật) định kỳ đẩy task snapshot/report (Alerts & Reports) vào Celery queue để `supersetWorker` xử lý và gửi email.

## Cơ chế Celery Worker / Celery Beat chạy task cụ thể như thế nào

### Celery Worker — thực thi task

1. **Đẩy task (producer, phía `supersetNode`)**: khi user tick "Run Async" ở SQL Lab hoặc load chart/dashboard lúc `GLOBAL_ASYNC_QUERIES` bật — `supersetNode` ghi 1 row `Query` vào Metadata DB (status `PENDING`), rồi gọi `task.delay(...)` (Celery API). Hàm này serialize tên task + tham số thành message, `LPUSH` vào Redis dưới dạng 1 **list** đóng vai trò hàng đợi (queue mặc định tên `celery`) — đây là ý nghĩa "Redis = broker". `supersetNode` trả response ngay (query_id), không đợi kết quả.
2. **Lấy task (consumer, phía `supersetWorker`)**: lệnh chạy là `celery --app=superset.tasks.celery_app:app worker` (xem `values-superset.yaml`). Khi start, worker mở kết nối Redis và liên tục **`BRPOP`** (blocking pop) trên list đó — "ngồi chờ" chứ không polling tốn CPU. Có message tới → tra tên task trong registry (các hàm đăng ký bằng `@celery_app.task`, vd `get_sql_results` cho SQL Lab) → giao cho 1 process con trong pool (Celery mặc định dùng pool *prefork* — nhiều tiến trình OS riêng, số lượng = số CPU container thấy được, vì values file không set `--concurrency` tường minh).
3. Vì nhiều pod worker (autoscale 2-6) cùng `BRPOP` chung 1 list Redis → mô hình **competing consumers**: `BRPOP` atomic nên không có 2 worker cùng nhận trùng 1 task.
4. **Bên trong 1 task** (ví dụ SQL Lab async):
   - Load lại row `Query` từ Metadata DB theo `query_id`.
   - Gọi `Database.get_sqla_engine()` → build/tái sử dụng SQLAlchemy engine, **mở connection TCP thật** từ chính pod `supersetWorker` tới Data Warehouse (Postgres/ClickHouse).
   - Chạy SQL, load kết quả vào `pandas.DataFrame`.
   - Ghi kết quả vào **results backend** — namespace Redis riêng (`resultsBackendKeyPrefix: superset_results`), khác với cache chart thường (`keyPrefix: superset_`).
   - Update row `Query` trong Metadata DB → status `SUCCESS`/`FAILED`.
   - Nếu `GLOBAL_ASYNC_QUERIES` bật: `XADD` thêm 1 event vào Redis Stream (`async-events-...`) để `supersetWebsockets` đọc và đẩy real-time về browser.

### Celery Beat — chỉ lên lịch, không tự thực thi

- Lệnh: `celery ... beat --pidfile /tmp/celerybeat.pid --schedule /tmp/celerybeat-schedule` — 2 file local chỉ để Beat nhớ lần chạy gần nhất của từng lịch (tránh chạy trùng khi pod restart).
- Beat không tự biết "Alert A chạy 8h sáng" — nó chỉ giữ 1 lịch cố định trong code Superset (vd "mỗi 60s bắn 1 task quét report"). Chính task đó (khi chạy — bởi **Worker**, không phải Beat) mới query Metadata DB để xem Alert/Report nào tới giờ (cron field lưu trong DB), rồi mới đẩy tiếp task thực thi thật (chụp ảnh, gửi mail) vào queue.
- Vì vậy chỉ nên chạy **đúng 1 replica** Beat (không autoscale) — 2 Beat cùng chạy sẽ bắn trùng lịch. Deployment hiện tại `enabled: false` nên không có tiến trình nào giữ lịch.

### Ý nghĩa hạ tầng

- `supersetNode` (producer) và `supersetWorker` (consumer) phải cùng trỏ về 1 Redis (`cache.celeryUrl`) — lệch cấu hình thì task "biến mất" (không lỗi, không log, không ai xử lý).
- Số worker replica (2-6, autoscale theo CPU) quyết định thông lượng xử lý song song — query nặng dồn dập mà CPU chưa kịp scale, task xếp hàng chờ trong Redis list, browser thấy `PENDING` lâu dù chưa hề mở connection tới data warehouse.

## Lưu ý riêng cho deployment này

Xem thêm comment chi tiết trong `Kubernetes-Apps/values-superset.yaml`:

- `GLOBAL_ASYNC_QUERIES_WEBSOCKET_URL` đang set cứng bằng DNS nội bộ cluster — nếu Superset được truy cập từ ngoài cluster, cần bật ingress/httproute route thêm path `/ws` hoặc đổi URL này thành domain public (wss://).
- `SECRET_KEY`, `supersetWebsockets.config.jwtSecret`, và mật khẩu admin (`init.adminUser.password`) đều đang là giá trị mẫu — **phải đổi trước khi lên production**.
- Redis hiện `auth.enabled: false` dù có set `password` — cần rà lại nếu muốn bật auth.
