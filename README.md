Câu 1: 
+ code tạo `vktSinhvien`
```sql
USE thuchanh;
GO

CREATE OR ALTER VIEW vktSinhvien
AS
SELECT 
    k.MaKhoa,
    k.TenKhoa,
    l.MaLop,
    l.TenLop,
    n.MaNganh,
    n.TenNganh,
    sv.MaSV,
    sv.HoTen,
    sv.Ngaysinh,
    sv.GioiTinh,
    sv.Dantoc
FROM Sinhvien sv
JOIN Lop l ON sv.MaLop = l.MaLop
JOIN Nganh n ON l.MaNganh = n.MaNganh
JOIN Khoa k ON n.MaKhoa = k.MaKhoa;
GO
```

+ code tạo `vktDangkyhoc`
```sql
CREATE OR ALTER VIEW vktDangkyhoc
AS
SELECT
    dk.HocKyDangKy,
    dk.MaSV,
    dk.MaMonhoc,
    mh.TenMonhoc,
    mh.Sotinchi,
    dk.PhantramKT,
    dk.DiemKT,
    dk.DiemTH,
    dk.DiemTB10,
    dk.DiemTB4,
    dk.DiemTBC
FROM Dangkyhoc dk
JOIN Monhoc mh ON dk.MaMonhoc = mh.MaMonhoc;
GO
```

+ code tạo `vktDangkyhoc1`
```sql
USE thuchanh;
GO

CREATE OR ALTER VIEW vktDangkyhoc1
AS
SELECT
    sv.TenKhoa,
    sv.TenLop,
    sv.TenNganh,
    sv.MaSV,
    sv.HoTen,
    sv.Ngaysinh,
    COUNT(dk.MaMonhoc) AS soHPDK,
    SUM(dk.Sotinchi) AS soTCDK
FROM vktSinhvien sv
JOIN vktDangkyhoc dk 
    ON sv.MaSV = dk.MaSV
GROUP BY
    sv.TenKhoa,
    sv.TenLop,
    sv.TenNganh,
    sv.MaSV,
    sv.HoTen,
    sv.Ngaysinh;
GO
```

Câu 2:
+ tạo thủ tục `spktTinhdiem`
```sql
USE thuchanh;
GO

CREATE OR ALTER PROC spktTinhdiem
    @hocky NVARCHAR(20),
    @malop NVARCHAR(20)
AS
BEGIN
    SET NOCOUNT ON;

    -- a) Cập nhật phần trăm điểm kiểm tra = 35%
    UPDATE dk
    SET dk.PhantramKT = 35
    FROM Dangkyhoc dk
    JOIN Sinhvien sv ON dk.MaSV = sv.MaSV
    WHERE sv.MaLop = @malop
      AND dk.HocKyDangKy = @hocky;

    -- b) Tính điểm trung bình hệ 10
    -- Công thức: DiemTB10 = DiemKT * 35% + DiemTH * 65%
    UPDATE dk
    SET dk.DiemTB10 = ROUND(
        dk.DiemKT * dk.PhantramKT / 100.0
        + dk.DiemTH * (100 - dk.PhantramKT) / 100.0,
        2
    )
    FROM Dangkyhoc dk
    JOIN Sinhvien sv ON dk.MaSV = sv.MaSV
    WHERE sv.MaLop = @malop
      AND dk.HocKyDangKy = @hocky;

    -- c, d) Quy đổi sang điểm hệ 4 và điểm chữ
    UPDATE dk
    SET 
        dk.DiemTB4 = td.diemso4,
        dk.DiemTBC = td.diemchu
    FROM Dangkyhoc dk
    JOIN Sinhvien sv ON dk.MaSV = sv.MaSV
    JOIN Thangdiem td
        ON dk.DiemTB10 >= td.tudiem
       AND dk.DiemTB10 <= td.dendiem
    WHERE sv.MaLop = @malop
      AND dk.HocKyDangKy = @hocky;
END;
GO
```

a. cập nhật phần trăm 
```sql
UPDATE dk
SET dk.PhantramKT = 35
FROM Dangkyhoc dk
JOIN Sinhvien sv ON dk.MaSV = sv.MaSV
WHERE sv.MaLop = N'111121531'
  AND dk.HocKyDangKy = N'3-2023-2024';
```

b. tính điểm tb
```sql
UPDATE dk
SET dk.DiemTB10 = ROUND(
    dk.DiemKT * dk.PhantramKT / 100.0
    + dk.DiemTH * (100 - dk.PhantramKT) / 100.0,
    2
)
FROM Dangkyhoc dk
JOIN Sinhvien sv ON dk.MaSV = sv.MaSV
WHERE sv.MaLop = N'111121531'
  AND dk.HocKyDangKy = N'3-2023-2024';
```

c. chuyển sang thang điểm 4
```sql
UPDATE dk
SET dk.DiemTB4 = td.diemso4
FROM Dangkyhoc dk
JOIN Sinhvien sv ON dk.MaSV = sv.MaSV
JOIN Thangdiem td
    ON dk.DiemTB10 >= td.tudiem
   AND dk.DiemTB10 <= td.dendiem
WHERE sv.MaLop = N'111121531'
  AND dk.HocKyDangKy = N'3-2023-2024';
```

d. chuyển sang thang điểm chữ
```sql
UPDATE dk
SET dk.DiemTBC = td.diemchu
FROM Dangkyhoc dk
JOIN Sinhvien sv ON dk.MaSV = sv.MaSV
JOIN Thangdiem td
    ON dk.DiemTB10 >= td.tudiem
   AND dk.DiemTB10 <= td.dendiem
WHERE sv.MaLop = N'111121531'
  AND dk.HocKyDangKy = N'3-2023-2024';
```

kiểm tra
```sql
USE thuchanh;
GO

DECLARE @return_value INT;

EXEC @return_value = dbo.spktTinhdiem
    @hocky = N'3-2023-2024',
    @malop = N'111121531';

SELECT 'Return Value' = @return_value;
GO
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/8b094322-3284-487f-8d2a-8b38fc9c09ee" />

Câu 4:
+ tạo bảng Diemtrungbinh
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/af5979d0-5234-4c4b-8628-6c688b78792c" />

+ tạo thủ tục `pktTinhdiemtb`
```sql
USE thuchanh;
GO

CREATE OR ALTER PROC spktTinhdiemtb
    @hocky NVARCHAR(20),
    @malop NVARCHAR(20)
AS
BEGIN
    SET NOCOUNT ON;

    -- Xóa dữ liệu cũ để chạy lại không bị trùng
    DELETE dtb
    FROM Diemtrungbinh dtb
    JOIN Sinhvien sv ON dtb.MaSV = sv.MaSV
    WHERE sv.MaLop = @malop
      AND dtb.tenhocky = @hocky;

    -- Thêm MaSV, tenhocky, DiemTB10
    INSERT INTO Diemtrungbinh(MaSV, tenhocky, DiemTB10)
    SELECT
        dk.MaSV,
        @hocky AS tenhocky,
        ROUND(
            SUM(dk.DiemTB10 * mh.Sotinchi) / NULLIF(SUM(mh.Sotinchi), 0),
            2
        ) AS DiemTB10
    FROM Dangkyhoc dk
    JOIN Sinhvien sv ON dk.MaSV = sv.MaSV
    JOIN Monhoc mh ON dk.MaMonhoc = mh.MaMonhoc
    WHERE sv.MaLop = @malop
      AND dk.HocKyDangKy = @hocky
    GROUP BY dk.MaSV;

    -- Cập nhật DiemTB4
    UPDATE dtb
    SET dtb.DiemTB4 = td.diemso4
    FROM Diemtrungbinh dtb
    JOIN Thangdiem td
        ON dtb.DiemTB10 >= td.tudiem
       AND dtb.DiemTB10 <= td.dendiem
    WHERE dtb.tenhocky = @hocky;

    -- Cập nhật DiemTBC
    UPDATE dtb
    SET dtb.DiemTBC = td.diemchu
    FROM Diemtrungbinh dtb
    JOIN Thangdiem td
        ON dtb.DiemTB10 >= td.tudiem
       AND dtb.DiemTB10 <= td.dendiem
    WHERE dtb.tenhocky = @hocky;

    -- Hiện kết quả
    SELECT
        MaSV,
        tenhocky,
        DiemTB10,
        DiemTB4,
        DiemTBC
    FROM Diemtrungbinh
    WHERE tenhocky = @hocky
    ORDER BY MaSV;
END;
GO
```

