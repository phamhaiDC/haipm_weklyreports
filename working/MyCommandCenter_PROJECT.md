# MyCommandCenter — Project Reference

> Công cụ cá nhân của HaiPM (Core Technology, Dcorp): lưu nhanh **File Drive**, **Task**, **Khách hàng**.
> Nạp file này vào context lần sau để tiếp tục phát triển mà không phải dò lại.
> **Phiên bản frontend hiện tại: v1.5.**

---

## 1. Kiến trúc

```
Frontend (1 file HTML tĩnh, GitHub Pages)
   │  POST JSON { action, token, ...fields }
   ▼
n8n webhook  →  Switch "Route Action"  →  node Postgres theo action  →  "Format Response"  →  "Respond"
   │                                                                         (bọc {ok,data})
   ▼
PostgreSQL (schema command_center)
```

- **Không có backend riêng.** n8n đóng vai lớp API giữa frontend tĩnh và Postgres. Quyết định giữ hướng này (Phương án A) vì tool cá nhân, ít entity, ít traffic. Ngưỡng cân nhắc chuyển sang Express API: khi số entity/action vượt ~5 loại, cần quan hệ/join nhiều bảng, validation nghiêm, hoặc workflow n8n khó sửa. Khi đó chỉ đổi `cfg.url`, frontend giữ nguyên.

---

## 2. Cấu hình kết nối

| Mục | Giá trị |
|---|---|
| Webhook | `https://n8n-edu.dcorp.com.vn/webhook/command-center` |
| Token | `HCM-HAI-1982` (nhúng client-side JS — **lộ**, xem §7) |
| n8n workflow ID | `yYkqlgEkXoAUUHI2` |
| Postgres credential | `n8n_postgres` (`AQjZ8SxH1YCQcO9f`) |
| DB schema | `command_center` |
| Request | `POST` JSON: `{ action, token, ...fields }` |
| Response | `{ ok: true, data: <object | array> }` (node "Format Response" bọc) |

---

## 3. Schema DB (`command_center`)

**drive_files**
| Cột | Kiểu | Ghi chú |
|---|---|---|
| id | BIGSERIAL PK | |
| name | TEXT | |
| url | TEXT | link Google Drive |
| tag | TEXT | **legacy** — không dùng ở UI từ v1.4, giữ để không mất dữ liệu cũ |
| note | TEXT | |
| customer_id | BIGINT | FK logic → customers.id (nullable) |
| created_at | TIMESTAMPTZ | |

**tasks**
| Cột | Kiểu | Ghi chú |
|---|---|---|
| id | BIGSERIAL PK | |
| name | TEXT | |
| ref_code | TEXT | mã TECH / link ClickUp |
| priority | TEXT | high / mid / low |
| status | TEXT | open / done |
| customer_id | BIGINT | nullable |
| created_at | TIMESTAMPTZ | |

**customers**
| Cột | Kiểu | Ghi chú |
|---|---|---|
| id | BIGSERIAL PK | |
| name | TEXT NOT NULL | |
| code | TEXT | mã KH (VTI, 4PS...) |
| segment | TEXT | chain / restaurant / cafe / other |
| contact | TEXT | đầu mối |
| status | TEXT | active / paused |
| note | TEXT | |
| created_at | TIMESTAMPTZ | |

---

## 4. API actions

| Action | Fields gửi kèm | Node Postgres |
|---|---|---|
| file_list | — | SELECT drive_files |
| file_add | name, url, note, customer_id | INSERT RETURNING * |
| file_update | id, name, url, note, customer_id | UPDATE RETURNING * |
| file_delete | id | DELETE ($1) |
| task_list | — | SELECT tasks |
| task_add | name, ref_code, priority, customer_id | INSERT (status='open') RETURNING * |
| task_toggle | id | UPDATE status ($1) |
| task_update | id, name, ref_code, priority, customer_id | UPDATE RETURNING * |
| task_delete | id | DELETE ($1) |
| customer_list | — | SELECT customers |
| customer_add | name, code, segment, contact, status, note | INSERT RETURNING * |
| customer_update | id, name, code, segment, contact, status, note | UPDATE RETURNING * |
| customer_delete | id | DELETE ($1) |

**customer_id rỗng → lưu NULL** (xử lý bằng ternary trong query template).

---

## 5. Quy ước node n8n (2 kiểu query)

**Kiểu 1 — SELECT / DELETE / TOGGLE:** query thường + `$1`, dùng `options.queryReplacement`.
```
DELETE FROM command_center.<table> WHERE id = $1 RETURNING id
Query Replacement: ={{ $json.body.id }}
```

**Kiểu 2 — INSERT / UPDATE:** template inline (KHÔNG có `$1`, KHÔNG queryReplacement), escape `'` và ép số cho id/customer_id.
```
{{ `UPDATE command_center.tasks SET
   name='${$json.body.name.replace(/'/g, "''")}',
   customer_id=${$json.body.customer_id ? parseInt($json.body.customer_id,10) : 'NULL'}
   WHERE id=${parseInt($json.body.id,10)} RETURNING ...` }}
```

- List trả `added` bằng `TO_CHAR(created_at AT TIME ZONE 'Asia/Ho_Chi_Minh', 'DD/MM/YYYY')`.
- INSERT/UPDATE luôn `RETURNING ...` (frontend cần id + field mới để cập nhật local, khỏi reload).
- Switch "Route Action": 13 output, điều kiện mỗi rule `={{ $json.body.action }}` **equals** outputKey. Mọi nhánh → "Format Response".

---

## 6. ⚠️ Bẫy đã gặp trong quá trình build (đọc kỹ để khỏi lặp)

1. **Switch condition phải là Expression:** ô Value 1 của rule phải là `={{ $json.body.action }}` (có `=`). Sửa tại chỗ hay bị "nuốt" dấu `=` khi ô đang ở Fixed. **Fix chắc ăn:** xóa rule lỗi → copy (⧉) một rule đã đúng → chỉ đổi Value 2 (rightValue) + Rename output, **không chạm Value 1**.
2. **save ≠ publish:** phải **Publish** mới có hiệu lực. Autosave không đủ.
3. **queryReplacement chỉ cho node có `$1`** (delete/toggle). Node template inline phải để **trống** queryReplacement, nếu không lỗi *"more parameters than placeholders"*.
4. **Đừng paste nhãn vào field:** từng dán nhầm cả chữ `Query:` / `Query Replacement:` vào ô Query → SQL hỏng.
5. **`=` đầu query template:** version-history/export của n8n hiển thị query template dạng `{{ ... }}` (không `=`) kể cả node đang chạy tốt. Đây là cách export hiển thị, không hẳn là lỗi — **verify bằng cách test**, đừng kết luận qua bản export. Nếu node trả về nguyên chuỗi text thay vì chạy → bật ô Query sang Expression.
6. **"Import from File" có thể nhân đôi node** trên workflow đang có sẵn. Không dùng để sửa nhỏ. Sửa 1 field → sửa tại chỗ; thay 1 node → copy-paste node đơn rồi nối lại dây.
7. **n8n MCP trong phiên Claude là READ-ONLY** (get history/version, validate_node_config, restore). Không ghi được workflow. Claude chỉ đọc/validate/sinh JSON copy-paste; **user phải tự áp + Publish**.
8. Frontend đọc `res.data` (mảng hoặc object đơn); luôn cần `RETURNING` để có id.

---

## 7. Bảo mật

- Token nằm trong JS tĩnh → **ai View Source cũng thấy**. Chấp nhận được cho tool cá nhân. Nếu tab KH chứa dữ liệu khách hàng thật nhạy cảm → thêm **Cloudflare Access** (đã có Zero Trust) trước, hoặc hạ quyền webhook.

---

## 8. Frontend

- 1 file HTML tự chứa, host **GitHub Pages**. Font **Montserrat + DM Mono**. Brand đỏ Dcorp `#e30613`, header tối `#1a1f2e`.
- 3 tab: **File Drive** · **Task** · **Khách hàng**.
- Add ở toolbar; Sửa qua modal chung (file/task/customer); dropdown KH động (lấy từ `allCustomers`).
- File lọc theo Khách hàng; Task lọc theo trạng thái.

**Màu client (v1.5):** mỗi KH 1 màu tông nhẹ, gán ổn định `id % 12` từ `CLIENT_PALETTE` (12 tông). Áp cho card File, badge KH trên Task, và card tab Khách hàng (làm legend). File chưa gắn KH → nền trung tính.

**Lịch sử phiên bản**
| Ver | Nội dung |
|---|---|
| v1.3 | Thêm tab Khách hàng (CRUD); nút Sửa (update) cho File & Task qua modal |
| v1.4 | File: bỏ "Nhóm" → thay bằng "Khách hàng"; Task thêm Khách hàng; cả hai tùy chọn. Thêm cột `customer_id` |
| v1.5 | Màu nền tông nhẹ theo từng client (auto, ổn định theo id) |

---

## 9. Backlog / ý tưởng mở rộng

- **Màu client tự thiết lập** (thay vì auto): thêm cột `customers.color`, ô chọn màu trong form KH, sửa customer_add/update + list. (Hiện đang auto theo id.)
- Lọc Task theo Khách hàng (hiện chỉ lọc trạng thái).
- Khi >12 KH có thể trùng màu (id % 12) — mở rộng palette hoặc chuyển sang màu thiết lập tay.
- Cân nhắc Express API nếu độ phức tạp tăng (xem §1).

---

*Nội bộ Dcorp · Core Technology · tài liệu tham chiếu để tiếp tục project MyCommandCenter.*
