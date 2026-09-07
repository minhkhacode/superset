# Cơ chế phân quyền RBAC trong Superset

Superset dùng RBAC có sẵn của **Flask-AppBuilder (FAB)** — không tự viết hệ thống phân quyền riêng. Xem kiến trúc tổng quan tại [Superset-Architecture.md](Superset-Architecture.md).

## Mô hình Permission — View — Role — User

- **Permission** (verb): hành động, vd `can_edit`, `can_show`, `can_delete`, `can_list`, `can_add`, `database access`, `datasource access`, `schema access`, `all_datasource_access`...
- **View/Menu** (resource): đối tượng bị tác động, vd model (`Dashboard`, `Chart`, `User`...), trang menu (`SQL Lab`, `Explore`...), hoặc **một resource cụ thể** (vd tên database `[My DB]`, schema `[My DB].[public]`, dataset `[sales_table]`).
- **Permission-View pair**: ghép Permission + View thành 1 quyền cụ thể, ví dụ `database access on [My DB]`, `can_read on Dashboard`.
- **Role**: tập hợp nhiều Permission-View pair.
- **User**: được gán 1 hoặc nhiều Role.

Toàn bộ mapping này (roles, permissions, permission_view, user-role) được lưu trong **Metadata Database** (các bảng `ab_role`, `ab_permission`, `ab_view_menu`, `ab_permission_view`, `ab_permission_view_role`, `ab_user`, `ab_user_role`) — không phải config file.

## 5 Role dựng sẵn (built-in)

| Role | Phạm vi | Ghi chú |
|---|---|---|
| **Admin** | Toàn quyền — quản lý user/role/permission, sửa data source, sửa/xóa chart-dashboard của người khác, truy cập mọi database | Superset auto tạo user admin đầu tiên qua `init.adminUser` trong values-superset.yaml |
| **Alpha** | Truy cập **mọi** data source đã kết nối, được thêm/sửa data source, nhưng **không** quản lý được user/role của người khác | Vẫn cần được cấp "database access" cho từng DB như Gamma (không tự động có mọi DB) |
| **Gamma** | Chỉ xem được data source **đã được cấp quyền tường minh** (qua permission-view hoặc do sở hữu chart/dashboard); không được thêm/sửa data source | Role mặc định phù hợp cho end-user/viewer |
| **sql_lab** | Bật menu/tính năng SQL Lab | Thường gán kèm với Alpha/Gamma; 2 quyền phụ (`can_estimate_query_cost`, `can_format_sql`) phải cấp thêm nếu cần |
| **Public** | Role hạn chế nhất, dùng cho user **chưa đăng nhập** xem dashboard công khai (nếu bật `PUBLIC_ROLE_LIKE`) | Không bật theo mặc định |

Lưu ý: **không nên sửa trực tiếp** 5 role này — Superset tự đồng bộ lại về giá trị gốc mỗi khi chạy `superset init` (chính là job `init` trong deployment này, chạy sau mỗi lần Helm install/upgrade — post-install/post-upgrade hook). Sửa trực tiếp sẽ bị ghi đè sau lần upgrade kế tiếp.

## Custom Role (khuyến nghị cho phân quyền thực tế)

Thay vì sửa role dựng sẵn, tạo role riêng ở **Settings → List Roles → +** rồi gán các Permission-View pair cụ thể, ví dụ:
- `database access on [Data Warehouse A]` — cho phép dùng toàn bộ dataset trong DB đó.
- `schema access on [Data Warehouse A].[public]` — hẹp hơn, chỉ 1 schema.
- `datasource access on [tên dataset]` — hẹp nhất, chỉ 1 dataset/table cụ thể.
- Cộng thêm các quyền UI cần thiết (`can_read on Dashboard`, `can_read on Chart`...).

Một user có thể được gán **nhiều role cùng lúc** (role cộng dồn quyền, không role nào loại trừ role nào).

## Row Level Security (RLS) — phân quyền tới từng dòng dữ liệu

RLS hẹp hơn "database/schema/datasource access" — thay vì chặn cả dataset, nó **lọc bớt dòng dữ liệu** trả về khi query, áp dụng tại tầng SQLAlchemy trước khi kết quả rời khỏi Superset:

- Cấu hình ở **Settings → Row Level Security → +**: chọn dataset(s) bị áp filter, gán cho role/user cụ thể, viết một `WHERE clause` (SQL) sẽ được nối vào mọi query trên dataset đó.
- Hỗ trợ Jinja template để filter động theo user đang đăng nhập, vd: `tenant_id = '{{ current_username() }}'` hoặc dựa vào thuộc tính user.
- 2 loại filter:
  - **Regular**: chỉ áp dụng cho user thuộc role được gán.
  - **Base**: áp dụng cho **mọi user trừ** role được liệt kê (deny-by-default).
- Nhiều filter cùng `group_key` → nối bằng `OR`; khác `group_key` → nối bằng `AND`.

## Cách áp dụng quyền cho User / nhóm user

1. **Từng user riêng lẻ**: Settings → List Users → Edit → chọn 1+ Role ở field "Role". Đây là cách gán phổ biến nhất, quản lý thủ công qua UI hoặc `/api/v1/security/...`.
2. **Qua Auth provider (LDAP/OAuth/OIDC)**: nếu Superset cấu hình `AUTH_TYPE` LDAP/OAuth và set `AUTH_ROLES_MAPPING`, Superset tự map group/claim từ IdP sang Role tương ứng **mỗi lần user login** — không cần gán tay từng người. Đây là cách "theo nhóm" phổ biến nhất trong thực tế, nhưng **deployment hiện tại chưa cấu hình** (không thấy `AUTH_TYPE`/`AUTH_ROLES_MAPPING` trong `configOverrides`/`config` của `values-superset.yaml` → mặc định chỉ dùng DB auth nội bộ, phân role thủ công qua UI).
3. **Group (FAB Group)**: bản Superset mới hơn có khái niệm Group tách biệt với Role (gán quyền cho cả nhóm user cùng lúc thay vì role) — cần kiểm tra version image cụ thể đang dùng (`image.tag` trong values đang để `~`, tức lấy theo `Chart.AppVersion`) có hỗ trợ hay chưa trước khi áp dụng.

## Ghi chú riêng cho deployment này

- `init.createAdmin: true` + `init.adminUser` (username `admin`, password mẫu `admin`) — tài khoản này có role **Admin**, full quyền. **Phải đổi password trước production** (đã note ở [Superset-Architecture.md](Superset-Architecture.md)).
- Vì không cấu hình LDAP/OAuth, mọi user khác admin phải được tạo + gán role thủ công qua UI (hoặc API) — muốn phân quyền theo phòng ban/nhóm tự động thì cần bổ sung `AUTH_TYPE`/`AUTH_ROLES_MAPPING` vào `config`/`configOverrides`.
- Role/permission được re-sync mỗi lần Job `init` chạy (post-install/post-upgrade hook) — custom role tự tạo không bị mất, chỉ 5 role dựng sẵn bị đồng bộ lại.
