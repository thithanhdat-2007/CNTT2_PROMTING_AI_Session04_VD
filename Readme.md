# BÁO CÁO THIẾT KẾ GIẢI PHÁP PROMPTING ỨNG DỤNG THỰC TẾ
**Chủ đề:** Kỹ thuật Prompting Nâng cao (Logic & Chuỗi - Chain-of-Thought, Prompt Chaining & Xử lý đứt gãy)

---

## PHẦN 1: BỐI CẢNH NGHIỆP VỤ & THIẾT KẾ GIẢI PHÁP (I/O)

### 1.1. Bối cảnh & Vấn đề thực tế
Một cửa hàng bán lẻ thời trang/điện máy đa kênh (Facebook, Zalo, Website) mỗi ngày nhận hàng trăm tin nhắn đặt hàng và khiếu nại từ khách hàng với văn phong tự nhiên, lộn xộn, viết tắt hoặc thiếu thông tin. 
* **Vấn đề:** Nhân viên CSKH tốn nhiều thời gian đọc hiểu, lọc thông tin, kiểm tra tồn kho và tính tiền thủ công, dễ dẫn đến sai sót khi chốt đơn hoặc bỏ sót các đơn hàng thiếu thông tin (SĐT, địa chỉ, size/màu).
* **Giải pháp:** Ứng dụng **Prompt Chaining (Chuỗi Prompt)** kết hợp **Chain-of-Thought (Tư duy từng bước)** để chia nhỏ quy trình thành 2 mắt xích độc lập:
  * **Mắt xích 1 (Prompt 1):** Trích xuất dữ liệu thô, làm sạch, chuẩn hóa thành định dạng JSON và bắt lỗi thiếu/sai dữ liệu (xử lý đứt gãy).
  * **Mắt xích 2 (Prompt 2):** Tiếp nhận JSON từ Bước 1, đối chiếu với Bảng giá/Tồn kho, suy luận logic (CoT) để tính tổng tiền và tự động sinh phản hồi phù hợp cho khách hàng.

### 1.2. Thiết kế luồng Input / Output (I/O)

#### A. Mắt xích 1: Trích xuất & Kiểm tra tính toàn vẹn dữ liệu (Data Extraction & Validation)
* **Input 1:** Đoạn hội thoại/tin nhắn thô của khách hàng (Text tự do).
* **Output 1 (Cấu trúc JSON chuẩn):**
  ```json
  {
    "trang_thai_du_lieu": "HOP_LE | THIEU_THONG_TIN | KHONG_XAC_DINH",
    "khach_hang": {
      "ten": "String hoặc null",
      "sdt": "String (10 số) hoặc null",
      "dia_chi": "String hoặc null"
    },
    "don_hang": [
      {
        "ten_san_pham": "String",
        "phan_loai": "String (Màu/Size) hoặc null",
        "so_luong": "Integer (mặc định 1 nếu khách không nói rõ)"
      }
    ],
    "danh_sach_loi_thieu": ["Liệt kê cụ thể các trường bị thiếu hoặc sai định dạng"]
  }
  ```

#### B. Mắt xích 2: Đối chiếu kho, Tính toán Logic & Phản hồi (Reasoning & Response)
* **Input 2:** 
  1. Dữ liệu `Output 1` (JSON).
  2. Dữ liệu tham chiếu: Bảng giá & Tồn kho hiện tại của cửa hàng.
* **Output 2 (Cấu trúc Markdown):**
  * **Khối `<thinking>` (Chain-of-Thought):** Trình bày từng bước suy luận kiểm tra lỗi, đối chiếu tồn kho, áp dụng công thức tính tiền và phí ship.
  * **Khối `<customer_reply>`:** Tin nhắn phản hồi lịch sự, đúng trọng tâm (xin bổ sung thông tin nếu đứt gãy, báo hết hàng nếu thiếu kho, hoặc gửi bill chốt đơn nếu hợp lệ).

### 1.3. Bẫy dữ liệu lường trước & Cơ chế xử lý đứt gãy (Edge Cases)
1. **Khách quên để lại SĐT hoặc SĐT sai độ dài (không đủ 10 số / chứa chữ):** AI không được tự bịa số, phải gán `null` và đưa vào `danh_sach_loi_thieu`.
2. **Đặt hàng thiếu phân loại (Size/Màu):** Ví dụ chỉ nói *"lấy 1 áo thun"* mà không nói size/màu $\rightarrow$ Gán `phan_loai: null`, kích hoạt luồng hỏi lại khách ở Mắt xích 2 thay vì chốt đơn bừa.
3. **Tin nhắn rác / Hỏi dạo không có ý định mua:** Gán `trang_thai_du_lieu: "KHONG_XAC_DINH"`, ngắt chuỗi tính tiền để tiết kiệm tài nguyên và chuyển sang kịch bản tư vấn chung.
4. **Sản phẩm khách đặt không có trong danh mục hoặc hết hàng:** Mắt xích 2 phải đối chiếu chính xác tên sản phẩm, nếu `ton_kho == 0` hoặc `so_luong_dat > ton_kho` thì dừng tính bill và báo cáo tình trạng kho.

---

## PHẦN 2: THIẾT KẾ PROMPT THỰC THI (PROMPT CHAINING)

### Prompt 1: Bộ lọc Trích xuất & Bắt lỗi Dữ liệu
```markdown
# ROLE
Bạn là một Chuyên viên Xử lý Dữ liệu Đơn hàng (Data Parsing Specialist) cực kỳ cẩn trọng và chính xác.

# TASK
Phân tích tin nhắn thô của khách hàng, trích xuất thông tin đặt hàng và kiểm tra tính đầy đủ, hợp lệ của dữ liệu.

# RULES & ERROR HANDLING (XỬ LÝ BẪY DỮ LIỆU)
1. TUYỆT ĐỐI KHÔNG tự suy diễn hoặc bịa đặt thông tin khách không cung cấp. Nếu thiếu, bắt buộc điền giá trị `null`.
2. Kiểm tra Số điện thoại (`sdt`): Phải bắt đầu bằng số `0` và có đúng `10 chữ số`. Nếu thiếu hoặc thừa số, gán `sdt: null` và ghi chú lỗi `"SĐT không hợp lệ"`.
3. Kiểm tra Phân loại (`phan_loai`): Sản phẩm thời trang bắt buộc phải có Size hoặc Màu sắc. Nếu khách không ghi, gán `null` và ghi chú lỗi `"Thiếu phân loại Size/Màu"`.
4. Kiểm tra Số lượng (`so_luong`): Nếu khách nói "lấy cái áo..." (số ít) mà không ghi số, mặc định là `1`. Nếu số lượng <= 0 hoặc vô lý, ghi chú lỗi.
5. Phân loại `trang_thai_du_lieu`:
   - `"HOP_LE"`: Có đủ Tên, SĐT hợp lệ, Địa chỉ, và tất cả sản phẩm đều có Phân loại.
   - `"THIEU_THONG_TIN"`: Khách có ý định mua hàng nhưng thiếu ít nhất 1 thông tin trên.
   - `"KHONG_XAC_DINH"`: Tin nhắn spam, hỏi dạo, không có sản phẩm nào được đặt.

# OUTPUT FORMAT
Chỉ trả về duy nhất một khối mã JSON hợp lệ theo cấu trúc sau, không giải thích thêm:
{
  "trang_thai_du_lieu": "",
  "khach_hang": {
    "ten": null,
    "sdt": null,
    "dia_chi": null
  },
  "don_hang": [
    {
      "ten_san_pham": "",
      "phan_loai": null,
      "so_luong": 1
    }
  ],
  "danh_sach_loi_thieu": []
}

# INPUT DATA
Tin nhắn khách hàng:
"""
{{TIN_NHAN_KHACH_HANG}}
"""
```

---

### Prompt 2: Bộ xử lý Logic (Chain-of-Thought) & Sinh Phản hồi
```markdown
# ROLE
Bạn là Trợ lý Vận hành Đơn hàng & CSKH thông minh của cửa hàng thời trang GenZ Store.

# CONTEXT (DỮ LIỆU CỬA HÀNG)
Bảng giá & Tồn kho hiện tại:
1. Áo Thun Basic Cotton | Giá: 150.000đ | Tồn kho: Đen (Size M: 5, Size L: 0), Trắng (Size M: 10, Size L: 3)
2. Quần Jeans Ống Rộng | Giá: 320.000đ | Tồn kho: Xanh nhạt (Size 29: 2, Size 30: 4)
3. Áo Khoác Hoodie Nỉ | Giá: 280.000đ | Tồn kho: Xám (Size L: 1, Size XL: 0)

Chính sách vận chuyển:
- Phí ship đồng giá: 30.000đ.
- Miễn phí vận chuyển (Freeship) cho đơn hàng có tổng giá trị sản phẩm từ 500.000đ trở lên.

# INSTRUCTION (CHAIN-OF-THOUGHT)
Hãy xử lý dữ liệu JSON đầu vào theo đúng trình tự suy luận từng bước trong thẻ `<thinking>` trước khi viết câu trả lời cho khách trong thẻ `<customer_reply>`:

- **Bước 1 (Kiểm tra trạng thái đầu vào):** Xem `trang_thai_du_lieu`. 
  - Nếu là `"KHONG_XAC_DINH"`, bỏ qua bước tính tiền, chuyển sang viết lời chào hỏi và mời khách xem mẫu.
  - Nếu là `"THIEU_THONG_TIN"`, liệt kê các mục trong `danh_sach_loi_thieu` để chuẩn bị hỏi xin bổ sung. (Vẫn tiếp tục Bước 2 để kiểm tra xem món khách định mua còn hàng không).
- **Bước 2 (Đối chiếu Danh mục & Tồn kho):** Với từng sản phẩm trong `don_hang`, kiểm tra xem có khớp tên trong Bảng giá không và `so_luong` đặt có `<= Tồn kho` của đúng phân loại đó không. Ghi nhận các món hết hàng hoặc không tồn tại.
- **Bước 3 (Tính toán tài chính - Chỉ thực hiện khi đơn hợp lệ và đủ tồn kho):**
  - Tính `Tạm tính = Tổng (Đơn giá * Số lượng)`.
  - Xét điều kiện Freeship: Nếu `Tạm tính >= 500.000đ` -> `Phí ship = 0đ`, ngược lại `Phí ship = 30.000đ`.
  - Tính `Tổng thanh toán = Tạm tính + Phí ship`.
- **Bước 4 (Tổng hợp kịch bản phản hồi):** Viết tin nhắn gửi khách hàng thân thiện, xưng "Shop" - gọi "bạn/anh/chị", trình bày rõ ràng.

# OUTPUT FORMAT
<thinking>
[Ghi chi tiết quá trình suy luận 4 bước tại đây]
</thinking>

<customer_reply>
[Nội dung tin nhắn gửi cho khách hàng]
</customer_reply>

# INPUT DATA
Dữ liệu JSON từ Bước 1:
"""
{{OUTPUT_JSON_TU_PROMPT_1}}
"""
```

---

## PHẦN 3: KẾT QUẢ CHẠY THỬ NGHIỆM THỰC TẾ

Để kiểm chứng khả năng xử lý logic và bẫy lỗi đứt gãy, hệ thống được chạy thử với **2 kịch bản thực tế**:
* **Kịch bản 1 (Bẫy dữ liệu đứt gãy + Hết tồn kho):** Khách viết tắt, nhập SĐT bị thiếu số, thiếu size quần và đặt trúng mẫu áo đã hết size L.
* **Kịch bản 2 (Đơn hàng hợp lệ + Đạt điều kiện Freeship):** Khách cung cấp đủ thông tin, hệ thống tự động tính tiền và miễn phí vận chuyển.

---

### KỊCH BẢN 1: KIỂM THỬ BẪY LỖI & XỬ LÝ ĐỨT GÃY

#### 1. Input thô từ khách hàng:
> *"Shop ơi gửi cho mình 1 áo thun basic màu đen size L với 1 quần jeans ống rộng xanh nhạt nhé. Mình là Minh Anh, sđt 098765432, địa chỉ 12 Nguyễn Trãi, Thanh Xuân, Hà Nội."*

#### 2. Kết quả chạy thử Prompt 1 (Trích xuất & Bắt lỗi):
```json
{
  "trang_thai_du_lieu": "THIEU_THONG_TIN",
  "khach_hang": {
    "ten": "Minh Anh",
    "sdt": null,
    "dia_chi": "12 Nguyễn Trãi, Thanh Xuân, Hà Nội"
  },
  "don_hang": [
    {
      "ten_san_pham": "Áo Thun Basic Cotton",
      "phan_loai": "Đen - Size L",
      "so_luong": 1
    },
    {
      "ten_san_pham": "Quần Jeans Ống Rộng",
      "phan_loai": null,
      "so_luong": 1
    }
  ],
  "danh_sach_loi_thieu": [
    "SĐT không hợp lệ (chỉ có 9 chữ số: 098765432)",
    "Thiếu phân loại Size cho sản phẩm: Quần Jeans Ống Rộng (mới có màu Xanh nhạt)"
  ]
}
```

#### 3. Kết quả chạy thử Prompt 2 (Suy luận CoT & Phản hồi):
```markdown
<thinking>
- Bước 1 (Kiểm tra trạng thái đầu vào): Trạng thái là `THIEU_THONG_TIN`. Có 2 lỗi cần báo khách bổ sung: (1) SĐT `098765432` bị thiếu 1 số, (2) Quần Jeans Ống Rộng màu Xanh nhạt chưa chọn Size (hiện có size 29 và 30).
- Bước 2 (Đối chiếu Danh mục & Tồn kho):
  1. Áo Thun Basic Cotton (Đen - Size L): Đối chiếu kho -> Tồn kho = 0. -> BÁO HẾT HÀNG (gợi ý sang Đen Size M hoặc Trắng Size L còn 3 cái).
  2. Quần Jeans Ống Rộng (Xanh nhạt - chưa rõ size): Đối chiếu kho -> Còn hàng ở Size 29 (2 cái) và Size 30 (4 cái).
- Bước 3 (Tính toán tài chính): Tạm dừng tính bill chốt đơn vì đơn hàng đang bị đứt gãy thông tin (thiếu SĐT chuẩn, thiếu size quần) và có sản phẩm hết kho.
- Bước 4 (Tổng hợp kịch bản phản hồi): Cảm ơn bạn Minh Anh, báo khéo việc Áo thun đen size L đang hết (gợi ý đổi mẫu), đồng thời xin bổ sung số điện thoại đủ 10 số và size quần jeans để shop lên đơn.
</thinking>

<customer_reply>
Chào bạn Minh Anh, cảm ơn bạn đã quan tâm và đặt hàng tại GenZ Store! 

Shop đã nhận được thông tin đơn hàng của bạn, tuy nhiên hiện đang có một chút vấn đề nhỏ cần bạn xác nhận lại giúp Shop nhé:

1. **Về tình trạng sản phẩm:** Mẫu **Áo Thun Basic Cotton màu Đen (Size L)** hiện tại Shop vừa tạm hết hàng ạ. Bạn có muốn đổi sang **màu Trắng (Size L)** hoặc **màu Đen (Size M)** không ạ?
2. **Về thông tin đơn hàng:**
   - Mẫu **Quần Jeans Ống Rộng (Xanh nhạt)** bạn chưa chọn size, hiện Shop đang sẵn **Size 29 và Size 30**, bạn mặc size nào để Shop giữ hàng nhé?
   - Số điện thoại `098765432` bạn gửi đang bị thiếu 1 chữ số, bạn nhắn lại giúp Shop SĐT chính xác (đủ 10 số) để bên vận chuyển liên hệ giao hàng ạ.

Bạn phản hồi lại giúp Shop để Shop lên đơn gửi đi sớm nhất cho mình nhé!
</customer_reply>
```

---

### KỊCH BẢN 2: KIỂM THỬ LUỒNG CHỐT ĐƠN HỢP LỆ (HAPPY PATH)

#### 1. Input thô từ khách hàng:
> *"Chốt cho anh 2 cái áo thun basic trắng size L với 1 cái áo khoác hoodie nỉ xám size L nha. Tên Hoàng Nam, 0912345678, giao về 45 Lê Lợi, Quận 1, TP.HCM."*

#### 2. Kết quả chạy thử Prompt 1:
```json
{
  "trang_thai_du_lieu": "HOP_LE",
  "khach_hang": {
    "ten": "Hoàng Nam",
    "sdt": "0912345678",
    "dia_chi": "45 Lê Lợi, Quận 1, TP.HCM"
  },
  "don_hang": [
    {
      "ten_san_pham": "Áo Thun Basic Cotton",
      "phan_loai": "Trắng - Size L",
      "so_luong": 2
    },
    {
      "ten_san_pham": "Áo Khoác Hoodie Nỉ",
      "phan_loai": "Xám - Size L",
      "so_luong": 1
    }
  ],
  "danh_sach_loi_thieu": []
}
```

#### 3. Kết quả chạy thử Prompt 2:
```markdown
<thinking>
- Bước 1 (Kiểm tra trạng thái đầu vào): Trạng thái là `HOP_LE`. Đầy đủ tên, SĐT 10 số, địa chỉ và phân loại sản phẩm.
- Bước 2 (Đối chiếu Danh mục & Tồn kho):
  1. Áo Thun Basic Cotton (Trắng - Size L): Khách đặt 2. Tồn kho = 3 -> ĐỦ HÀNG. Đơn giá: 150.000đ.
  2. Áo Khoác Hoodie Nỉ (Xám - Size L): Khách đặt 1. Tồn kho = 1 -> ĐỦ HÀNG. Đơn giá: 280.000đ.
- Bước 3 (Tính toán tài chính):
  - Tiền Áo Thun: $2 \times 150.000 = 300.000\text{đ}$
  - Tiền Áo Khoác Hoodie: $1 \times 280.000 = 280.000\text{đ}$
  - Tạm tính: $300.000 + 280.000 = 580.000\text{đ}$
  - Xét Freeship: Vì $580.000\text{đ} \ge 500.000\text{đ}$ -> Phí ship = $0\text{đ}$ (Miễn phí vận chuyển).
  - Tổng thanh toán: $580.000 + 0 = 580.000\text{đ}$.
- Bước 4 (Tổng hợp kịch bản phản hồi): Xác nhận chốt đơn thành công cho anh Hoàng Nam, liệt kê chi tiết hóa đơn, thông báo được Freeship và cảm ơn.
</thinking>

<customer_reply>
Dạ GenZ Store xin chào anh Hoàng Nam! Shop xác nhận đã lên đơn thành công cho mình với thông tin chi tiết như sau ạ:

**THÔNG TIN NHẬN HÀNG:**
- **Người nhận:** Anh Hoàng Nam
- **Số điện thoại:** 0912345678
- **Địa chỉ:** 45 Lê Lợi, Quận 1, TP.HCM

**CHI TIẾT ĐƠN HÀNG:**
1. Áo Thun Basic Cotton (Trắng - Size L): 2 x 150.000đ = 300.000đ
2. Áo Khoác Hoodie Nỉ (Xám - Size L): 1 x 280.000đ = 280.000đ
- **Tạm tính:** 580.000đ
- **Phí vận chuyển:** 0đ *(Đơn hàng trên 500k được Freeship ạ)*
- **TỔNG THANH TOÁN (COD):** **580.000đ**

Shop sẽ đóng gói và bàn giao cho đơn vị vận chuyển ngay hôm nay. Cảm ơn anh Nam đã tin tưởng ủng hộ GenZ Store ạ!
</customer_reply>
```