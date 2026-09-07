# [Data] Ticket: SRE-1491 Nghiên cứu cài đặt và cấu hình Superset

## Tiến độ công việc

| # | Đầu việc | Trạng thái | Tài liệu chi tiết |
|---|---|---|---|
| 1 | Kiến trúc hệ thống (Application, Metadata DB, Redis cache) | ✅ Xong | [Superset-Architecture.md](Superset-Architecture.md) |
| 2 | Cài đặt service qua ArgoCD | ✅ Xong | [Argocd-Apps/superset-app.yaml](Argocd-Apps/superset-app.yaml), [Kubernetes-Apps/values-superset.yaml](Kubernetes-Apps/values-superset.yaml) |
| 3 | Sử dụng SQL Lab để viết query | ✅ Xong | Mục 3 dưới đây (chưa có file riêng) |
| 4 | Kết nối PostgreSQL/ClickHouse + cấu hình Dataset | ✅ Xong | [Superset-Add-Database-Flow.md](Superset-Add-Database-Flow.md) |
| 5 | Cơ chế phân quyền RBAC | ✅ Xong | [Superset-RBAC.md](Superset-RBAC.md) |
| 6 | Cấu hình cache Redis cho query nặng | ✅ Xong | Mục 6 dưới đây (dựa trên `Superset-Architecture.md` + `values-superset.yaml`) |

Hướng phát triển tiếp theo: Thực hiện benchmark tải thực tế, test failover Redis/Postgres, cấu hình LDAP/OAuth.

---

## 1. Kiến trúc hệ thống

Xem đầy đủ tại [Superset-Architecture.md](Superset-Architecture.md). Tóm tắt:

- **Superset Application (Flask + React)**: 4 pod trong deployment — `supersetNode` (web/API, gunicorn, port 8088), `supersetWorker` (Celery worker, autoscale 2-6 replica), `supersetCeleryBeat` (scheduler, `enabled: false` - đang tắt), `supersetWebsockets` (Node.js, chỉ chạy khi `GLOBAL_ASYNC_QUERIES: true` — đang bật).
- **Metadata Database (PostgreSQL 14.17, subchart bitnamilegacy)**: lưu user/role/permission, database connection, dataset/chart/dashboard, query history — **không** lưu dữ liệu phân tích thực.
- **Caching Layer (Redis 7.0.10, standalone)**: kiêm 3 vai trò — query/result cache, Celery broker + results backend, Redis Stream cho `GLOBAL_ASYNC_QUERIES`.
- **Data Warehouse (bên ngoài Superset, không do Helm chart này quản lý)**: nơi lưu dữ liệu phân tích thực (PostgreSQL, ClickHouse trong scope ticket này) — Superset chỉ kết nối tới qua SQLAlchemy, cả `supersetNode` và `supersetWorker` đều mở connection trực tiếp tới đây khi chạy query thật.

Luồng cơ bản: User → `supersetNode` → check Redis cache → cache miss thì query đồng bộ (tự chạy) hoặc đẩy Celery (bất đồng bộ) → `supersetWorker` chạy → ghi kết quả vào Redis → `supersetWebsockets` đẩy real-time về browser qua WebSocket.

---

## 2. Cài đặt Superset qua ArgoCD

File: [Argocd-Apps/superset-app.yaml](Argocd-Apps/superset-app.yaml).

- ArgoCD `Application` dùng **multi-source**:
  - Nguồn 1: Helm chart gốc `apache/superset` từ `https://apache.github.io/superset`, `targetRevision: 0.22.4`.
  - Nguồn 2: Git repo `https://github.com/minhkhacode/superset.git` (nhánh `main`), chỉ dùng để cung cấp file `values-superset.yaml` (`ref: git-repo`) cho nguồn 1 qua `helm.valueFiles`.
- `destination`: cluster hiện tại (`https://kubernetes.default.svc`), namespace `superset`.
- `syncPolicy.automated`: `prune: true`, `selfHeal: true` — ArgoCD tự đồng bộ và tự sửa drift; `CreateNamespace=true` nên không cần tạo namespace tay.
- Toàn bộ tuỳ biến (feature flag, cache, driver DB, resource, RBAC ban đầu...) nằm trong 1 file duy nhất `Kubernetes-Apps/values-superset.yaml`, không sửa trực tiếp trên ArgoCD/cluster.
- Điểm cần lưu ý khi cấu hình:
  - `bootstrapScript` cài driver DB lúc container khởi động (`postgres`, `bigquery`, `clickhouse-connect`, `elasticsearch`) — muốn thêm engine mới (MySQL, Trino...) phải sửa script này rồi rolling-restart, **không** cấu hình được qua UI.
  - Job `init` (Helm hook `post-install,post-upgrade`) tạo admin user + đồng bộ lại 5 role dựng sẵn mỗi lần deploy — có fix riêng cho bug Istio sidecar khiến Job không bao giờ `Completed`.
  - `GLOBAL_ASYNC_QUERIES_WEBSOCKET_URL` phải set cứng dạng string trong `config` để né một bug parse kiểu dữ liệu (int → float64) của chart 0.22.4 khi build port qua `printf "%d"`.

---

## 3. Sử dụng SQL Lab để viết truy vấn

- Vào qua menu **SQL → SQL Lab** (cần role có gắn quyền `sql_lab`, xem mục 5).
- Giao diện gồm: panel trái chọn **Database → Schema → Table** (browser tự liệt kê bảng/cột nếu connection hỗ trợ introspection), editor SQL ở giữa, panel kết quả ở dưới.
- Chạy query: nút **Run** hoặc `Ctrl/Cmd+Enter`; có thể chọn **Run Async** (checkbox) để đẩy qua Celery/Redis thay vì chờ đồng bộ — nên bật cho query nặng để không giữ connection HTTP quá lâu (xem mục 6).
- Superset tự thêm `LIMIT` theo cấu hình `ROW_LIMIT` nếu câu query chưa có `LIMIT` — tránh kéo về quá nhiều dòng làm treo trình duyệt.
- Cho phép DDL/DML (`INSERT/UPDATE/DELETE`, tạo bảng...) phụ thuộc cờ **Allow DDL/DML** khi khai báo database connection — mặc định các driver hiện có (Postgres/BigQuery/ClickHouse/Elasticsearch) đang **tắt** cờ này nên SQL Lab chỉ chạy được `SELECT`.
- Tiện ích trong editor: **Format SQL** (cần permission `can_format_sql`), **Estimate cost** trước khi chạy (cần `can_estimate_query_cost` — chỉ khả dụng với driver hỗ trợ, ví dụ BigQuery), lưu **Query History**, lưu thành **Saved Query** để tái dùng.
- Từ kết quả query có thể: tải CSV/xuất, hoặc bấm **"Create Chart"/"Explore"** để biến trực tiếp câu SQL đang chạy thành một **virtual Dataset** rồi vẽ chart (xem mục 4).
- Quyền `sql_lab` chỉ bật *menu*; quyền chạy được trên database/schema nào vẫn do RBAC (`database access`/`schema access`) quyết định — user thấy menu SQL Lab nhưng không thấy DB nếu chưa được cấp quyền.

---

## 4. Kết nối PostgreSQL, ClickHouse và cấu hình Dataset

### 4.1. Kết nối database

Luồng đầy đủ (test connection, mã hoá credential, driver nào dùng khi nào...) đã viết chi tiết tại [Superset-Add-Database-Flow.md](Superset-Add-Database-Flow.md). Tóm tắt riêng cho 2 engine trong scope ticket:

| | PostgreSQL | ClickHouse |
|---|---|---|
| Driver cài qua `bootstrapScript` | `.[postgres]` (psycopg2) | `clickhouse-connect` |
| SQLAlchemy URI mẫu | `postgresql://user:pass@host:5432/dbname` | `clickhousedb://user:pass@host:8123/dbname` (HTTP interface) |
| Giao thức thật khi query | TCP thuần tới port 5432 | HTTP(S) tới port 8123 (hoặc 9440 nếu native+TLS) |
| Egress cần mở | TCP tới port Postgres | TCP/HTTP tới port ClickHouse |

Các bước qua UI: **Settings → Data → Databases → + DATABASE** → chọn engine → nhập host/port/user/password/dbname (hoặc dán thẳng SQLAlchemy URI) → **Test Connection** → **Connect**. Cả 2 pod `supersetNode` và `supersetWorker` đều cần network egress tới data warehouse đích vì cả 2 đều có thể chạy query thật (đồng bộ vs qua Celery).

### 4.2. Cấu hình Dataset

Sau khi có database connection, vào **Settings → Data → Datasets → + DATASET**, có 2 cách tạo:

- **Physical dataset**: chọn Database → Schema → Table có sẵn — Superset map trực tiếp 1 bảng vật lý, tự đọc cột/kiểu dữ liệu qua introspection của driver.
- **Virtual dataset**: tại SQL Lab, chạy 1 câu SQL bất kỳ (join, aggregate, subquery...) rồi bấm **"Save as Dataset"** — Superset lưu nguyên câu SQL đó, mỗi lần chart dùng dataset này sẽ chạy lại câu SQL làm subquery bên trong.

Trong cả 2 loại, sau khi tạo có thể chỉnh ở tab **Edit dataset**:
- **Columns**: đánh dấu cột nào là `Temporal` (dùng cho time range filter), thêm **Calculated columns** (biểu thức SQL tính từ cột khác).
- **Metrics**: định nghĩa sẵn công thức aggregate (`SUM(revenue)`, `COUNT(DISTINCT user_id)`...) để tái dùng khi vẽ chart, tránh mỗi chart phải tự viết lại.
- **Settings**: đổi tên hiển thị, gán **Owners**, set **Cache timeout** riêng cho dataset này (override `defaultTimeout` chung của Redis cache ở mục 6) — hữu ích khi 1 vài bảng dữ liệu update chậm/nhanh khác nhau và không muốn dùng chung 1 TTL.
- Quyền truy cập dataset kiểm soát qua RBAC (`datasource access on [tên dataset]`, xem mục 5) — độc lập với quyền lên cả database.

---

## 5. Cơ chế phân quyền RBAC

Xem đầy đủ tại [Superset-RBAC.md](Superset-RBAC.md). Tóm tắt:

- Dùng RBAC có sẵn của Flask-AppBuilder: mô hình **Permission (verb) + View/Menu (resource) → Permission-View pair → Role → User**, lưu trong Metadata DB (bảng `ab_*`), không phải config file.
- 5 role dựng sẵn: **Admin** (toàn quyền), **Alpha** (mọi data source nhưng cần cấp `database access` từng DB), **Gamma** (chỉ xem data source được cấp quyền tường minh — mặc định cho viewer), **sql_lab** (bật menu SQL Lab), **Public** (user chưa đăng nhập, tắt mặc định). 5 role này bị Job `init` đồng bộ lại về giá trị gốc mỗi lần deploy → không sửa trực tiếp.
- Khuyến nghị tạo **Custom Role** riêng, gán permission-view cụ thể (`database access on [DB]`, `schema access`, `datasource access on [dataset]`) thay vì sửa role dựng sẵn.
- **Row Level Security (RLS)**: lọc theo dòng dữ liệu (không phải chặn cả bảng), viết `WHERE` clause có Jinja template theo user đăng nhập, hỗ trợ 2 loại filter `Regular`/`Base` (deny-by-default).
- Gán quyền hiện tại: thủ công qua **Settings → List Users → Edit → Role** (deployment **chưa** cấu hình `AUTH_TYPE`/`AUTH_ROLES_MAPPING` cho LDAP/OAuth nên không tự map role theo nhóm AD/IdP).

---

## 6. Cấu hình cache Redis cho query nặng

Redis (subchart `bitnamilegacy/redis` 7.0.10, standalone) đảm nhiệm đồng thời 3 việc, cấu hình trong `values-superset.yaml` mục `cache:` và `supersetWebsockets.config.redis`:

| Cấu hình | Giá trị hiện tại | Ý nghĩa |
|---|---|---|
| `cache.keyPrefix` | `superset_` | Prefix key cache kết quả query của chart/dashboard |
| `cache.defaultTimeout` | `86400s` (24h) | TTL mặc định — có thể override riêng theo từng Dataset (mục 4.2) |
| `cache.resultsBackendKeyPrefix` | `superset_results` | Prefix cho kết quả trả về từ Celery task |
| `cache.asyncQueries.keyPrefix` / `.timeout` | `qc-` / `86400s` | Prefix/TTL riêng cho cơ chế `GLOBAL_ASYNC_QUERIES` |
| `featureFlags.GLOBAL_ASYNC_QUERIES` | `true` | Bật đẩy query nặng qua Celery + trả kết quả qua WebSocket thay vì polling |
| `supersetWebsockets.config.redis` | trỏ về `<release>-redis-headless:6379` | Websocket service đọc Redis Stream (`redisStreamPrefix: async-events-`) do `supersetWorker` ghi, đẩy real-time về browser |

Cơ chế cho query nặng: khi user chạy SQL Lab bật **Run Async** (mục 3) hoặc load chart/dashboard lúc `GLOBAL_ASYNC_QUERIES` bật, `supersetNode` không tự chạy mà đẩy task vào Redis (Celery broker) → `supersetWorker` (autoscale 2-6 replica theo CPU) nhận và chạy query thật → ghi kết quả vào Redis (results backend) + ghi event vào Redis Stream → `supersetWebsockets` đọc stream, đẩy kết quả về đúng browser đang chờ qua WebSocket, xác thực bằng JWT (`jwtSecret` + cookie `async-token`) — tránh phải polling liên tục và tránh giữ 1 connection HTTP đồng bộ trong lúc query chạy lâu.

### Rủi ro/điểm cần xử lý trước production (ghi nhận trong lúc nghiên cứu, chưa fix)

- `SECRET_KEY` (mã hoá credential DB trong Metadata DB), `supersetWebsockets.config.jwtSecret`, và mật khẩu admin (`init.adminUser.password: admin`) đều đang là **giá trị mẫu** trong `values-superset.yaml` — bắt buộc đổi trước khi lên production.
- Redis subchart có `auth.password: superset` nhưng `auth.enabled: false` — auth thực tế **chưa bật**, cần rà lại nếu muốn bảo vệ Redis.
- `GLOBAL_ASYNC_QUERIES_WEBSOCKET_URL` đang trỏ DNS nội bộ cluster (`ws://apache-superset-ws.superset.svc.cluster.local:8080/ws`) — nếu Superset được truy cập từ ngoài cluster, WebSocket sẽ không hoạt động trừ khi bật ingress/httproute route thêm `/ws` hoặc đổi thành domain public (`wss://`).
- Chưa cấu hình `AUTH_TYPE`/`AUTH_ROLES_MAPPING` (LDAP/OAuth) — phân role hiện tại hoàn toàn thủ công qua UI, chưa tự động theo nhóm phòng ban.
- Chưa benchmark tải thực tế cho autoscaling `supersetWorker` (min 2 - max 6 replica) — ngưỡng `targetCPUUtilizationPercentage: 80` là giá trị khởi điểm, cần tinh chỉnh sau khi có traffic thật.
