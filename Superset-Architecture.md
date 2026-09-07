# Superset

Repo triển khai Apache Superset lên Kubernetes qua Helm chart (`Kubernetes-Apps/values-superset.yaml`) + ArgoCD (`Argocd-Apps/`).

## Kiến trúc hệ thống

Superset gồm 3 thành phần cốt lõi, cộng thêm Celery worker/beat và (tuỳ chọn) một websocket service cho async query.

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

- Lưu: user/role/permission, database connections, định nghĩa dataset/chart/dashboard, query history, log, lịch alert/report.
- **Không** lưu dữ liệu phân tích thực — dữ liệu business nằm ở data warehouse riêng mà Superset chỉ kết nối tới qua SQLAlchemy.
- Trong deployment: subchart `postgresql` (bitnamilegacy/postgresql 14.17), database/user `superset`, có PVC persistence.

### 3. Caching Layer (Redis)

Một Redis instance nhưng đảm nhiệm 3 vai trò khác nhau (phân tách bằng `cacheDb`/`celeryDb`):

- **Query/result cache**: cache kết quả query của chart/dashboard (`defaultTimeout: 86400`s, key prefix `superset_`) — load lại chart lần 2 sẽ lấy từ cache thay vì query lại data warehouse.
- **Celery broker + results backend**: hàng đợi task bất đồng bộ giữa `supersetNode` (producer) và `supersetWorker` (consumer) — dùng cho SQL Lab async query, report snapshot/email.
- **Redis Streams cho `GLOBAL_ASYNC_QUERIES`**: khi query nặng chạy qua Celery, `supersetWorker` ghi event kết quả vào một Redis Stream (prefix `async-events-`); `supersetWebsockets` đọc stream này và đẩy trực tiếp về browser qua WebSocket (thay vì browser phải polling), xác thực bằng JWT (`jwtSecret` + cookie `async-token`).

Trong deployment: subchart `redis` (bitnamilegacy/redis 7.0.10, standalone, `auth.enabled: false`, không bật persistence).

## Luồng tương tác cơ bản

1. User → browser → `supersetNode` (qua Service/Ingress).
2. `supersetNode` xác thực/authorize qua Flask-AppBuilder, đọc metadata (dataset, chart, permission) từ **PostgreSQL**.
3. `supersetNode` kiểm tra **cache** (Redis) trước:
   - **Cache hit** → trả kết quả ngay.
   - **Cache miss, query đồng bộ** → `supersetNode` tự chạy SQL tới data warehouse, ghi kết quả vào cache, trả về browser.
   - **Cache miss, query bất đồng bộ** (SQL Lab async hoặc `GLOBAL_ASYNC_QUERIES`) → `supersetNode` đẩy task vào Celery queue (Redis broker) → `supersetWorker` nhận task, chạy query, ghi kết quả vào Redis (results backend) + ghi event vào Redis Stream → `supersetWebsockets` đọc stream, đẩy kết quả real-time về browser qua WebSocket.
4. `supersetCeleryBeat` (nếu bật) định kỳ đẩy task snapshot/report (Alerts & Reports) vào Celery queue để `supersetWorker` xử lý và gửi email.

## Lưu ý riêng cho deployment này

Xem thêm comment chi tiết trong `Kubernetes-Apps/values-superset.yaml`:

- `GLOBAL_ASYNC_QUERIES_WEBSOCKET_URL` đang set cứng bằng DNS nội bộ cluster — nếu Superset được truy cập từ ngoài cluster, cần bật ingress/httproute route thêm path `/ws` hoặc đổi URL này thành domain public (wss://).
- `SECRET_KEY`, `supersetWebsockets.config.jwtSecret`, và mật khẩu admin (`init.adminUser.password`) đều đang là giá trị mẫu — **phải đổi trước khi lên production**.
- Redis hiện `auth.enabled: false` dù có set `password` — cần rà lại nếu muốn bật auth.
