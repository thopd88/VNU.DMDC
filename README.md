# VNU.DMDC

**DANH MỤC BẢNG MÃ DÙNG CHUNG** phục vụ kết nối, chia sẻ dữ liệu trong và ngoài Đại học Quốc gia Hà Nội (ĐHQGHN).

---

## Quick reference (cho hệ thống khác tham chiếu)

### Cấu trúc thư mục

| Thư mục / File | Mô tả |
|----------------|--------|
| `txt_catalogs/` | Danh mục gốc dạng text: mỗi file `NNN_TEN-DANH-MUC.txt` (UTF-8, dòng đầu = tên danh mục, dòng 2 = header, sau đó `Mã\|Tên` hoặc có cột thêm) |
| `sql_catalogs/structure.sql` | **Định nghĩa bảng** (CREATE TABLE) cho toàn bộ danh mục – dùng để tạo schema. |
| `sql_catalogs/data_combined.sql` | **Toàn bộ dữ liệu** (INSERT) trong một file – chạy sau `structure.sql`. |
| `sql_catalogs/data_sql/*.sql` | Dữ liệu từng bảng riêng lẻ (INSERT theo từng danh mục) – tiện cho migration từng phần. |

### Quy ước tên bảng

- Tên file txt: `NNN_TEN-DANH-MUC.txt` (ví dụ: `001_CAP-KHEN-THUONG.txt`).
- Tên bảng SQL: dạng `TEN_DANH_MUC`, dấu gạch ngang → gạch dưới (ví dụ: `CAP_KHEN_THUONG`).
- Ánh xạ đầy đủ: xem comment `-- From NNN_...` trong `sql_catalogs/structure.sql`.

### Số lượng danh mục

- **153 danh mục** (001–153), gồm cả danh mục đặc thù ĐHQGHN (ví dụ: cây tổ chức, xã/phường cũ).

---

## Mô hình dữ liệu (để map và migrate)

### Chuẩn chung (đa số bảng)

- **Cột:** `Ma` (VARCHAR(50)), `Ten` (VARCHAR(255)).
- **Khóa tham chiếu:** dùng giá trị **Ma** khi hệ thống khác map dữ liệu (FK sang bảng sự nghiệp, đào tạo, v.v.).

### Bảng có cấu trúc đặc biệt

| Bảng | Cột | Ghi chú |
|------|-----|--------|
| `XA_PHUONG` | `Ma`, `Ten`, `Tinh` | Thêm cột tỉnh/thành. |
| `XA_PHUONG_CU` | `Ma`, `Ten`, … | Danh mục xã/phường cũ (theo file nguồn). |
| `CAY_TO_CHUC_DHQGHN` | `Ma`, `Ten`, `Don_vi_cha` | Cây tổ chức ĐHQGHN; `Don_vi_cha` tham chiếu `Ma` (self-reference). |

Chi tiết từng bảng xem trong `sql_catalogs/structure.sql`.

---

## Cách migrate dữ liệu sang catalog này

### Bước 1: Tạo schema

```bash
# Trong DB của bạn (MySQL/MariaDB/PostgreSQL/SQL Server tùy engine)
mysql -u user -p your_db < sql_catalogs/structure.sql
```

- Nếu engine khác PostgreSQL: có thể bỏ hoặc chỉnh lại câu `REFERENCES` trong `CAY_TO_CHUC_DHQGHN` (Don_vi_cha) cho phù hợp.

### Bước 2: Nạp dữ liệu

**Cách A – Một lần (khuyến nghị cho lần đầu):**

```bash
mysql -u user -p your_db < sql_catalogs/data_combined.sql
```

**Cách B – Từng bảng (cho migration chọn lọc):**

```bash
# Ví dụ chỉ một vài danh mục
mysql -u user -p your_db < sql_catalogs/data_sql/TRINH_DO_DAO_TAO_DAU_RA.sql
mysql -u user -p your_db < sql_catalogs/data_sql/GIOI_TINH.sql
# ...
```

### Bước 3: Map mã hệ thống cũ → Ma catalog

1. **Lấy danh sách mã chuẩn:** đọc từ `sql_catalogs/data_combined.sql` hoặc từng file trong `sql_catalogs/data_sql/` (cột đầu trong `INSERT ... VALUES` là `Ma`).
2. **Tạo bảng/bảng map (migration):**  
   `(ma_he_thong_cu, ma_catalog)` với `ma_catalog` trùng với `Ma` trong bảng danh mục tương ứng.
3. **Cập nhật dữ liệu nghiệp vụ:**  
   dùng bảng map để thay thế mã cũ bằng `Ma` của VNU.DMDC (ví dụ: cập nhật cột `Trinh_do_dao_tao`, `Gioi_tinh`, …).

### Bước 4: Dùng bảng txt (nếu không dùng SQL)

- File trong `txt_catalogs/` có dạng: dòng 1 = tên danh mục, dòng 2 = header (vd: `Mã|Tên`), từ dòng 3 = dữ liệu, phân tách `|`.
- Có thể import bằng script (Python, ETL, …) đọc UTF-8 và map cột vào bảng hoặc vào mã tham chiếu của hệ thống bạn.

---

## Lưu ý khi tích hợp

- **Mã (`Ma`)** có thể dạng số thập phân (vd: `001.00000050`) hoặc mã phân cấp (vd: `143.G25.39.000`). Nên lưu đúng chuỗi, không convert sang số.
- **Encoding:** toàn bộ file dùng **UTF-8**.
- **Thứ tự thực thi:** luôn chạy `structure.sql` trước, sau đó mới chạy file dữ liệu (`data_combined.sql` hoặc các file trong `data_sql/`).
- Cập nhật danh mục: khi có phiên bản mới từ ĐHQGHN, so sánh diff của `txt_catalogs/` và `sql_catalogs/` rồi áp dụng lại structure/data tương ứng và chạy lại bước map mã nếu có thay đổi `Ma`.
