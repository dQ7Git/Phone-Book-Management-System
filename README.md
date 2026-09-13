# Phone Book Management System (Python + Tkinter + MySQL)

Ứng dụng quản lý sổ danh bạ, xây dựng bám sát 2 file được giao:

- `Requirement_Specification_Design_Document.pdf` — tài liệu đặc tả yêu cầu,
  Use Case chi tiết (UC1–UC7), Functional/Non-Functional Requirements, ERD,
  Class Diagram, mock-up giao diện.
- `database.sql` — cấu trúc 3 bảng `Users`, `Categories`, `Contacts`.

## 1. Cài đặt & cấu hình MySQL

**Bước 1 — Cài MySQL Server** (nếu máy chưa có): cài MySQL Community
Server, hoặc dùng XAMPP/MAMP (đều có kèm MySQL) — nhớ **khởi động** service
MySQL trước khi chạy chương trình.

**Bước 2 — Cài thư viện Python:**

```bash
pip install -r requirements.txt
```

**Bước 3 — Sửa thông tin kết nối** trong `database.py`, đầu file (đổi
`password` cho đúng mật khẩu MySQL của bạn; đổi `host`/`port` nếu MySQL
không chạy trên máy local mặc định):

```python
DB_CONFIG = {
    "host": "localhost",
    "port": 3306,
    "user": "root",
    "password": "",   # <-- đổi thành mật khẩu MySQL của bạn
}
DB_NAME = "phonebook_db"
```

**Bước 4 — Chạy chương trình:**

```bash
python main.py
```

Lần chạy đầu tiên, hàm `init_db()` sẽ **tự động** tạo database
`phonebook_db` và 3 bảng (Users, Categories, Contacts) nếu chưa có — bạn
**không bắt buộc** phải tự chạy `database_mysql_fixed.sql` bằng tay trước
(dù chạy tay cũng không sao, vì code dùng `CREATE TABLE IF NOT EXISTS`).

Nếu MySQL chưa được bật hoặc sai mật khẩu, chương trình sẽ hiện 1 hộp
thoại lỗi rõ ràng ("Database Connection Error: ...") thay vì bị crash,
kèm gợi ý kiểm tra lại `DB_CONFIG`.

## 2. Cấu trúc project

```
phonebook_app/
├── main.py         # Điểm khởi chạy (python main.py)
├── app.py          # Toàn bộ giao diện Tkinter (View + Controller)
├── database.py     # Kết nối MySQL + toàn bộ hàm CRUD (Model)
├── validators.py   # Các hàm kiểm tra dữ liệu bằng regex
├── requirements.txt
└── README.md
```

Cách chia file đi theo đúng kiến trúc MVC nói ở mục IV báo cáo, nhưng được
đơn giản hoá cho phù hợp với một project môn học: `database.py` là Model,
`app.py` gộp chung View + Controller (thường gặp trong các project Tkinter
cỡ nhỏ).

## 3. Bảng đối chiếu Use Case ↔ Code

| Use Case (báo cáo)        | Hàm trong `database.py`            | Nơi gọi trong `app.py`     |
|---------------------------|-------------------------------------|-----------------------------|
| UC1 – Register Account    | `register_user`                     | `RegisterPage`              |
| UC2 – Login                | `login_user`                        | `LoginPage`                  |
| UC3 – Add Contact          | `add_contact`, `phone_exists`, `name_exists` | `ContactDialog` (thêm mới) |
| UC4 – Update Contact       | `update_contact`, `get_contact_by_id` | `ContactDialog` (sửa)     |
| UC5 – Delete Contact       | `delete_contact`                    | `DashboardPage.handle_delete` |
| UC6 – Manage Groups        | `add_category`, `rename_category`, `delete_category` (có transaction) | `GroupManagerDialog` |
| UC7 – Search & Filter      | `search_contacts`                   | `DashboardPage` (thanh tìm kiếm + debounce) |
| Update Profile *(sơ đồ Use Case)* | `update_full_name`           | `ProfileDialog`              |
| Change Password *(sơ đồ Use Case)*| `change_password`             | `ChangePasswordDialog`       |

Mọi thông báo lỗi/thành công trong code (ví dụ: *"Invalid email or
password"*, *"This contact no longer exists. Refreshing list."*, *"Group
already exists."*...) được lấy **nguyên văn** từ mục "Detailed Use Case
Specifications" của báo cáo.

## 4. Các NFR đã được hiện thực

- **NFR1 (Usability)**: nút Save/Register bị khoá cho đến khi các trường
  bắt buộc hợp lệ; lỗi hiện ngay dưới ô nhập (màu đỏ) khi người dùng gõ.
- **NFR2 (Reliability)**: xoá nhóm dùng transaction SQLite thật
  (`BEGIN` / `COMMIT` / `ROLLBACK`) để không tạo ra "orphan records"; phiên
  đăng nhập tự hết hạn sau 2 giờ không thao tác.
- **NFR3 (Performance)**: có tạo index trên `fullName` và `phoneNumber`; ô
  tìm kiếm áp dụng debounce 300ms trước khi truy vấn CSDL.
- **NFR4 (Security)**: mật khẩu băm bằng bcrypt (cost factor 10); mọi câu
  truy vấn Contacts đều có điều kiện `WHERE user_email = ?` để 1 tài khoản
  không bao giờ đọc được dữ liệu của tài khoản khác.

## 5. Một vài điểm đơn giản hoá có chủ đích

Vì đây là ứng dụng **desktop, chạy 1 máy, 1 người dùng tại một thời điểm**
(không phải hệ thống client-server nhiều người dùng thật), một vài điểm
trong báo cáo được đơn giản hoá cho phù hợp:

- "Session token / JWT" trong báo cáo được thay bằng 1 biến
  `self.current_user` lưu trong bộ nhớ khi ứng dụng đang chạy.
- Tình huống "liên hệ bị xoá ở tab khác" được mô phỏng bằng cách kiểm tra
  lại record trong MySQL trước khi Update/Delete.
- Chương trình chỉ kiểm tra kết nối MySQL và hiện lỗi rõ ràng **ngay lúc
  khởi động** (`init_db()`). Nếu MySQL bị tắt/rớt mạng *giữa lúc đang
  dùng* app, thao tác đó sẽ báo lỗi Python thông thường thay vì thông báo
  đẹp — đây là giới hạn hợp lý cho một project SQLite/MySQL desktop cỡ
  môn học, không phải một hệ thống production.

Đây là các đơn giản hoá hợp lý cho một project Tkinter/MySQL chạy độc lập;
toàn bộ **logic nghiệp vụ** (validate, chống trùng, transaction, phân
quyền theo `user_email`...) vẫn được cài đặt đầy đủ và đã được kiểm thử.
