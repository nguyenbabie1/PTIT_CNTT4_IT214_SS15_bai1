# BÁO CÁO BÀI TẬP 1

## MÔ PHỎNG GIAO THỨC 2PC CHO GIAO DỊCH ĐẶT VÉ TÀU HỎA


## 1. Mục tiêu

Bài tập mô phỏng giao thức **Two-Phase Commit (2PC)** trong một giao dịch đặt vé tàu hỏa phân tán. Giao dịch có hai dịch vụ tham gia:

- `TrainService`: kiểm tra và tạm giữ chỗ trên chuyến tàu.
- `WalletService`: kiểm tra và tạm giữ số tiền cần thanh toán trong ví điện tử.

`Coordinator` điều phối toàn bộ giao dịch. Giao dịch chỉ được xác nhận khi cả hai dịch vụ đều chuẩn bị thành công. Nếu có ít nhất một dịch vụ thất bại, toàn bộ dịch vụ tham gia phải hoàn tác để dữ liệu không bị sai lệch.

## 2. Dữ liệu đầu vào

```text
bookingData = {
    bookingId: "TRAIN-2024-089",
    trainCode: "SE5",
    customerWalletId: "W-456",
    price: 1200000
}
```

Ý nghĩa:

- Mã giao dịch đặt vé: `TRAIN-2024-089`.
- Chuyến tàu cần đặt: `SE5`.
- Ví thanh toán: `W-456`.
- Giá vé: `1.200.000 đồng`.

## 3. Quy trình 2PC

### 3.1. Pha 1 – Prepare/Voting

Coordinator gửi lệnh `PREPARE` đến tất cả dịch vụ.

1. `TrainService` kiểm tra chuyến tàu còn chỗ hay không. Nếu còn chỗ, dịch vụ tạm khóa ghế và trả về `READY`; nếu hết chỗ, trả về `ABORT`.
2. `WalletService` kiểm tra số dư. Nếu đủ tiền, dịch vụ tạm khóa 1.200.000 đồng và trả về `READY`; nếu không đủ tiền, trả về `ABORT`.
3. Coordinator lưu phản hồi của từng dịch vụ trước khi đưa ra quyết định chung.

Việc tạm giữ ghế và tiền chưa phải là xác nhận cuối cùng. Các tài nguyên này chỉ được ghi nhận vĩnh viễn sau lệnh `COMMIT`.

### 3.2. Pha 2 – Commit hoặc Rollback

- Nếu **tất cả** phản hồi là `READY`, Coordinator gửi `COMMIT` đến cả `TrainService` và `WalletService`.
- Nếu **ít nhất một** phản hồi là `ABORT`, hoặc xảy ra lỗi/timeout, Coordinator gửi `ROLLBACK` đến **tất cả dịch vụ tham gia**, kể cả dịch vụ đã trả về `READY` và dịch vụ báo lỗi.

Gửi rollback đến toàn bộ dịch vụ giúp giải phóng mọi tài nguyên có thể đã được tạm giữ và đưa hệ thống về trạng thái trước giao dịch.

## 4. Mã giả hoàn chỉnh

```text
FUNCTION coordinator(bookingData):
    services  <- [TrainService, WalletService]
    responses <- empty map

    PRINT "Bắt đầu giao dịch: " + bookingData.bookingId
    PRINT "=== PHASE 1: PREPARE ==="

    FOR EACH service IN services:
        TRY:
            PRINT "Coordinator -> " + service.name + ": PREPARE"
            response <- service.prepare(bookingData)
            responses[service.name] <- response
            PRINT service.name + " -> Coordinator: " + response
        CATCH error OR timeout:
            responses[service.name] <- ABORT
            PRINT service.name + " -> Coordinator: ABORT (lỗi/timeout)"
        END TRY
    END FOR

    allReady <- TRUE

    FOR EACH response IN responses.values:
        IF response != READY:
            allReady <- FALSE
            BREAK
        END IF
    END FOR

    PRINT "=== PHASE 2: DECISION ==="

    IF allReady == TRUE:
        PRINT "Quyết định toàn cục: COMMIT"

        FOR EACH service IN services:
            service.commit(bookingData.bookingId)
            PRINT "Coordinator -> " + service.name + ": COMMIT"
        END FOR

        RETURN "BOOKING_SUCCESS"
    ELSE:
        PRINT "Quyết định toàn cục: ROLLBACK"

        // Rollback TẤT CẢ dịch vụ đã tham gia giao dịch.
        FOR EACH service IN services:
            service.rollback(bookingData.bookingId)
            PRINT "Coordinator -> " + service.name + ": ROLLBACK"
        END FOR

        RETURN "BOOKING_FAILED"
    END IF
END FUNCTION
```

## 5. Mô phỏng các dịch vụ

```text
FUNCTION TrainService.prepare(bookingData):
    IF isSeatAvailable(bookingData.trainCode):
        temporarilyHoldSeat(bookingData.bookingId, bookingData.trainCode)
        RETURN READY
    ELSE:
        RETURN ABORT
    END IF
END FUNCTION

FUNCTION WalletService.prepare(bookingData):
    IF getBalance(bookingData.customerWalletId) >= bookingData.price:
        temporarilyHoldMoney(
            bookingData.bookingId,
            bookingData.customerWalletId,
            bookingData.price
        )
        RETURN READY
    ELSE:
        RETURN ABORT
    END IF
END FUNCTION

FUNCTION TrainService.commit(bookingId):
    confirmHeldSeat(bookingId)
END FUNCTION

FUNCTION WalletService.commit(bookingId):
    deductHeldMoney(bookingId)
END FUNCTION

FUNCTION TrainService.rollback(bookingId):
    releaseHeldSeatIfExists(bookingId)
END FUNCTION

FUNCTION WalletService.rollback(bookingId):
    releaseHeldMoneyIfExists(bookingId)
END FUNCTION
```

Các hàm rollback được thiết kế an toàn khi gọi kể cả khi tài nguyên chưa được giữ. Nhờ đó, Coordinator có thể gửi `ROLLBACK` đến mọi participant mà không phải đoán dịch vụ nào đã thay đổi trạng thái.

## 6. Kết quả chạy thử

### 6.1. Trường hợp thành công

Giả sử chuyến `SE5` còn chỗ và ví `W-456` có ít nhất 1.200.000 đồng.

```text
Bắt đầu giao dịch: TRAIN-2024-089
=== PHASE 1: PREPARE ===
Coordinator -> TrainService: PREPARE
TrainService -> Coordinator: READY
Coordinator -> WalletService: PREPARE
WalletService -> Coordinator: READY
=== PHASE 2: DECISION ===
Quyết định toàn cục: COMMIT
Coordinator -> TrainService: COMMIT
Coordinator -> WalletService: COMMIT
Kết quả: BOOKING_SUCCESS
```

Kết quả: ghế được xác nhận và tiền bị trừ chính thức. Hai dịch vụ cùng commit nên dữ liệu nhất quán.

### 6.2. Trường hợp thất bại do ví không đủ tiền

Giả sử `TrainService` đã giữ ghế thành công nhưng `WalletService` phát hiện số dư nhỏ hơn 1.200.000 đồng.

```text
Bắt đầu giao dịch: TRAIN-2024-089
=== PHASE 1: PREPARE ===
Coordinator -> TrainService: PREPARE
TrainService -> Coordinator: READY
Coordinator -> WalletService: PREPARE
WalletService -> Coordinator: ABORT
=== PHASE 2: DECISION ===
Quyết định toàn cục: ROLLBACK
Coordinator -> TrainService: ROLLBACK
Coordinator -> WalletService: ROLLBACK
Kết quả: BOOKING_FAILED
```

Coordinator không chỉ rollback `WalletService` mà gửi rollback đến cả hai dịch vụ. `TrainService` giải phóng ghế đã giữ; `WalletService` giải phóng khoản tiền tạm giữ nếu có. Vì vậy không xảy ra tình trạng giữ ghế nhưng không thanh toán.

## 7. Điều kiện quyết định

```text
TrainService    WalletService    Quyết định
READY           READY            COMMIT tất cả
READY           ABORT            ROLLBACK tất cả
ABORT           READY            ROLLBACK tất cả
ABORT           ABORT            ROLLBACK tất cả
```

Coordinator chỉ commit khi biểu thức `allReady == TRUE`. Chỉ cần một phản hồi khác `READY`, bao gồm `ABORT`, exception hoặc timeout, kết quả phải là rollback toàn cục.

## 8. Kết luận

Mô phỏng đã thực hiện đủ hai pha của 2PC. Trong pha Prepare, các participant kiểm tra điều kiện và tạm khóa tài nguyên. Trong pha quyết định, Coordinator commit khi tất cả đều sẵn sàng; ngược lại, Coordinator rollback toàn bộ dịch vụ tham gia. Cơ chế này duy trì tính nguyên tử: giao dịch đặt vé và thanh toán hoặc cùng thành công, hoặc cùng bị hủy.

