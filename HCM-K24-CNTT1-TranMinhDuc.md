BÀI THI KT-DGNL
Họ và tên: Trần Minh Đức - HCM-K24-CNTT1

CÂU 1: CHỈ RA VÀ GIẢI THÍCH CÁC LỖ HỔNG / LỖI NGHIỆP VỤ

Đoạn mã xử lý createOrder trong bài tồn tại 04 lỗ hổng / lỗi nghiệp vụ nghiêm trọng như sau:

1. Lỗ hổng Tin tưởng dữ liệu đơn giá từ Client (Price Tampering / Client-Side Price Manipulation)
- Chi tiết lỗ hổng: Đoạn mã tính tổng tiền đơn hàng dựa trực tiếp vào đơn giá do Client gửi lên trong DTO:
  double totalPrice = request.getUnitPrice() * request.getQuantity();
- Tác động / Hậu quả trong thực tế:
  + Kẻ tấn công (hacker) hoặc người dùng cố tình có thể sử dụng các công cụ như Burp Suite, Postman, Fiddler hoặc DevTools để can thiệp HTTP Request, sửa unitPrice thành 0, 0.01 hoặc số âm (ví dụ -100000).
  + Hậu quả: Thất thoát doanh thu nghiêm trọng cho cửa hàng/doanh nghiệp (người mua có thể sở hữu sản phẩm đắt tiền với giá 0 đồng hoặc khiến hệ thống hoàn tiền lại cho họ), làm sai lệch hoàn toàn dữ liệu kế toán tài chính.

2. Lỗi Tranh chấp dữ liệu và Bán vượt tồn kho (Race Condition / Concurrency Issue & Over-selling)
- Chi tiết lỗ hổng: Thao tác đọc tồn kho (product.getStock()) và trừ tồn kho (product.setStock(...)) diễn ra theo cơ chế Đọc-Sửa-Ghi (Read-Modify-Write) thông thường, không được bảo vệ bởi bất kỳ cơ chế khóa (Locking) nào (như Pessimistic Locking, Optimistic Locking hay Atomic DB Update).
- Tác động / Hậu quả trong thực tế:
  + Khi có nhiều yêu cầu (requests) mua cùng 1 mặt hàng gửi lên hệ thống tại cùng một thời điểm (ví dụ: sự kiện Flash Sale, sản phẩm chỉ còn 1 cái tồn kho nhưng có 100 người bấm nút "Đặt hàng" cùng lúc).
  + Các luồng (threads) xử lý đồng thời sẽ cùng đọc được stock = 1 từ DB -> Tất cả đều vượt qua điều kiện stock >= quantity -> Tất cả đều thực hiện trừ kho và tạo đơn hàng thành công.
  + Hậu quả: Xảy ra hiện tượng bán vượt tồn kho (Over-selling), làm số lượng tồn kho sản phẩm bị âm trong CSDL. Doanh nghiệp không đủ hàng để giao, buộc phải hủy đơn, đền bù hợp đồng và tổn hại nghiêm trọng đến uy tín thương hiệu.

3. Thiếu quản lý giao dịch (Missing Transactional Management)
- Chi tiết lỗ hổng: Phương thức createOrder thực hiện 2 thao tác ghi vào cơ sở dữ liệu (productRepository.save(product) và orderRepository.save(order)), nhưng phương thức không có annotation @Transactional.
- Tác động / Hậu quả trong thực tế:
  + Nếu bước trừ tồn kho và lưu sản phẩm (productRepository.save(product)) thành công, nhưng bước lưu đơn hàng (orderRepository.save(order)) bị lỗi (ví dụ: đứt kết nối CSDL, vi phạm ràng buộc dữ liệu constraint, lỗi bộ nhớ...), hệ thống sẽ không tự động Rollback bước 1.
  + Hậu quả: Dữ liệu bị bất nhất (Inconsistent Data): Sản phẩm bị trừ tồn kho trong DB nhưng không hề có đơn hàng nào được tạo ra cho khách.

4. Không kiểm tra (Validation) tính hợp lệ của số lượng (quantity)
- Chi tiết lỗ hổng: Đoạn mã không kiểm tra request.getQuantity() có phải là số nguyên dương (> 0) hay không.
- Tác động / Hậu quả trong thực tế:
  + Kẻ tấn công có thể truyền quantity <= 0 (ví dụ quantity = -5).
  + Điều kiện product.getStock() >= -5 vẫn đúng (ví dụ stock = 10 >= -5). Khi thực hiện product.setStock(10 - (-5)), tồn kho của sản phẩm sẽ bị tăng ngược lên thành 15 (hack cộng tồn kho ảo), đồng thời tổng tiền đơn hàng bị tính sai.

(Lưu ý bổ sung: Việc dùng kiểu dữ liệu double cho giá tiền dễ gây ra lỗi sai số làm tròn số thực dấu phẩy động trong tính toán tài chính).


CÂU 2: ĐỀ XUẤT GIẢI PHÁP KHẮC PHỤC CỤ THỂ

Dưới đây là giải pháp khắc phục triệt để cho từng lỗ hổng đã chỉ ra ở Câu 1:

1. Khắc phục Lỗi Tin tưởng giá từ Client
- Giải pháp: Loại bỏ thuộc tính unitPrice khỏi OrderRequestDTO. Đơn giá sản phẩm phải được truy vấn trực tiếp từ CSDL dựa vào productId.
- Sửa mã:
  BigDecimal unitPrice = product.getPrice(); 
  BigDecimal totalPrice = unitPrice.multiply(BigDecimal.valueOf(request.getQuantity()));

2. Khắc phục Lỗi Tranh chấp dữ liệu (Race Condition) & Quản lý giao dịch (@Transactional)
- Giải pháp 1 (Thêm @Transactional): Đánh dấu phương thức createOrder với @Transactional của Spring (org.springframework.transaction.annotation.Transactional) để đảm bảo tính nguyên tố ACID. Nếu có lỗi xảy ra ở bất cứ đâu trong hàm, toàn bộ thao tác CSDL sẽ được Rollback.
- Giải pháp 2 (Khóa đồng thời - Concurrency Control):
  + Pessimistic Locking (Khóa bi quan - Khuyên dùng cho Flash Sale/Đơn hàng): Trong ProductRepository, khai báo phương thức tìm kiếm sản phẩm với @Lock(LockModeType.PESSIMISTIC_WRITE) (SELECT ... FOR UPDATE). Luồng đầu tiên giữ khóa sẽ xử lý xong transaction rồi mới nhả cho luồng sau.
  + Hoặc Update Query nguyên tố (Atomic Update): UPDATE Product p SET p.stock = p.stock - :qty WHERE p.id = :id AND p.stock >= :qty

3. Khắc phục Lỗi Validate dữ liệu đầu vào (quantity)
- Giải pháp: Sử dụng Hibernate Validator / Spring Validation (@Min(1), @NotNull) trên OrderRequestDTO kết hợp kiểm tra chặt chẽ ở tầng Service.

4. Khắc phục Lỗi Kiểu dữ liệu tiền tệ
- Giải pháp: Đổi kiểu dữ liệu của unitPrice và totalPrice từ double sang BigDecimal (hoặc Long lưu theo đơn vị nhỏ nhất như xu/đồng) để tránh sai số dấu phẩy động.


MÃ NGUỒN HOÀN CHỈNH SAU KHI REFACTOR (REFACTORED CODE)

1. OrderRequestDTO.java:

package com.shop.order.dto;

import javax.validation.constraints.Min;
import javax.validation.constraints.NotNull;

public class OrderRequestDTO { 
    @NotNull(message = "Product ID khong duoc de trong")
    private Long productId; 
    
    @Min(value = 1, message = "So luong dat hang phai lon hon hoac bang 1")
    private int quantity;

    // Loai bo unitPrice khoi DTO de tranh Price Tampering

    public Long getProductId() { return productId; }
    public void setProductId(Long productId) { this.productId = productId; }

    public int getQuantity() { return quantity; }
    public void setQuantity(int quantity) { this.quantity = quantity; }
}


2. ProductRepository.java:

package com.shop.order.repository;

import com.shop.order.entity.Product;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Lock;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import javax.persistence.LockModeType;
import java.util.Optional;

public interface ProductRepository extends JpaRepository<Product, Long> {
    
    // Su dung Pessimistic Lock de ngan chan Race Condition & Over-selling
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdWithLock(@Param("id") Long id);
}


3. OrderService.java:

package com.shop.order.service;

import com.shop.order.dto.OrderRequestDTO;
import com.shop.order.entity.Order;
import com.shop.order.entity.Product;
import com.shop.order.repository.OrderRepository;
import com.shop.order.repository.ProductRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;

@Service
public class OrderService {

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private OrderRepository orderRepository;

    @Transactional // Quan ly Transaction: Rollback tu dong neu gap loi
    public Order createOrder(OrderRequestDTO request) {

        // 1. Validate so luong dau vao
        if (request.getQuantity() <= 0) {
            throw new IllegalArgumentException("So luong phai lon hon 0");
        }

        // 2. Lay san pham tu DB co kem Pessimistic Lock (SELECT FOR UPDATE)
        Product product = productRepository.findByIdWithLock(request.getProductId())
                .orElseThrow(() -> new RuntimeException("San pham khong ton tai"));

        // 3. Kiem tra ton kho
        if (product.getStock() < request.getQuantity()) {
            throw new RuntimeException("San pham khong du so luong ton kho");
        }

        // 4. Lay gia CHUAN tu CSDL (truy van DB, khong tin client) & dung BigDecimal
        BigDecimal unitPrice = product.getPrice(); 
        BigDecimal totalPrice = unitPrice.multiply(BigDecimal.valueOf(request.getQuantity()));

        // 5. Tru ton kho
        product.setStock(product.getStock() - request.getQuantity());
        productRepository.save(product);

        // 6. Tao va luu don hang
        Order order = new Order();
        order.setProductId(product.getId());
        order.setQuantity(request.getQuantity());
        order.setTotalPrice(totalPrice);
        order.setStatus("CONFIRMED");

        return orderRepository.save(order);
    }
}


CÂU 3: GIẢI PHÁP ƯU TIÊN SỬA TRƯỚC TIÊN VÀ LÝ DO

Nếu phải chọn MỘT giải pháp ưu tiên sửa trước tiên, tôi sẽ chọn:
-> Khắc phục lỗi "Tin tưởng giá từ Client" (Lấy giá sản phẩm trực tiếp từ CSDL và loại bỏ unitPrice khỏi Request DTO).

Lý do lựa chọn:

1. Mức độ thiệt hại tài chính tức thì (Immediate & Guaranteed Financial Damage):
   - Lỗi thao túng giá (Price Tampering) trực tiếp dẫn đến thất thoát tiền mặt của doanh nghiệp. Kẻ xấu có thể mua mặt hàng trị giá hàng chục triệu đồng với giá 0đ hoặc 1đ.
   - Thiệt hại này xảy ra ngay lập tức và chắc chắn 100% trên từng đơn hàng bị can thiệp.

2. Khả năng bị khai thác cực kỳ dễ dàng (High Exploitability & Low Barrier to Entry):
   - Lỗi này không cần điều kiện môi trường đặc biệt. Bất kỳ người dùng nào chỉ với kiến thức cơ bản (biết bấm F12 DevTools, Postman hoặc dùng proxy tool) đều có thể khai thác thành công.
   - Trong khi đó, lỗi Race Condition đòi hỏi truy cập đồng thời cao (Flash Sale), còn lỗi thiếu @Transactional chỉ phát sinh khi hệ thống gặp sự cố hiếm hoi ở tầng DB.

3. Tỷ lệ hiệu quả trên công sức bỏ ra tối ưu nhất (Lowest Effort, Highest Return):
   - Việc loại bỏ unitPrice khỏi DTO và lấy product.getPrice() từ DB chỉ mất 1 - 2 phút sửa code, hoàn toàn không nguy cơ làm sập hệ thống hay đòi hỏi nâng cấp hạ tầng phức tạp, nhưng mang lại hiệu quả bảo vệ tài sản doanh nghiệp 100%.
