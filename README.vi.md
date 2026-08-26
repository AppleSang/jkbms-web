# JK-BMS Monitor (bản tiếng Việt)

Ứng dụng web **không cần cài đặt** để giám sát và điều khiển mạch quản lý pin **JK-BMS** qua **Bluetooth Low Energy**, dùng **Web Bluetooth API** ngay trên trình duyệt.

## Tính năng

- **Số liệu pin thời gian thực**: dung lượng pin SOC (chính xác tới số lẻ trên fw ≥ 11), dung lượng còn lại / cấu hình / đã lão hóa (Ah), SOH, số chu kỳ, điện áp tổng, dòng điện, công suất
- **Điện áp từng cell** làm nổi bật cell thấp nhất / cao nhất và chênh lệch (mV)
- **Biểu đồ mini 30 giây** trên mỗi ô số liệu (điện áp, dòng, công suất, cell min/max, chênh lệch, nhiệt độ, dòng cân bằng)
- **Nhiệt độ**: cảm biến pin 1 & 2, nhiệt độ MOS (giá trị cảm biến lỗi hiển thị `--`)
- **Điều khiển BMS**: bật/tắt MOS sạc, MOS xả, cân bằng, và **chế độ Emergency** (chỉ fw ≥ 11, có hộp thoại cảnh báo nguy hiểm)
- **Tự nhận diện phiên bản firmware** (SW ≥ 11.x → layout mới dịch 32 byte, cũ hơn → layout 24S), kèm menu chọn tay
- **Xử lý khung BLE bền bỉ**: tự dò lại header giữa stream, chịu được byte rác xen giữa (như flood `AT\r\n` của JK-PB), ghép đúng khung khi gói notify bị cắt tùy ý
- **Giao diện 2 ngôn ngữ (Việt / Anh)** — tự nhận theo địa chỉ IP (GeoJS, có phương án dự phòng), chuyển bằng 1 nút
- **Chủ đề tối / sáng**, lưu trong `localStorage`
- Máy tính: log ghi ra console DevTools; điện thoại: bảng log hiện luôn trên trang kèm hex dump

## Yêu cầu

| Yêu cầu | Chi tiết |
|---|---|
| Trình duyệt | Chrome hoặc Edge (máy tính hoặc Android). **Không** hỗ trợ iOS Safari / Firefox |
| Kết nối | Trang phải chạy qua **HTTPS** hoặc **http://localhost** (yêu cầu của Web Bluetooth) |
| Mạch BMS | JK-BMS dùng module **BLE** (giao thức JK02). Module chỉ có Bluetooth cổ điển (SPP) sẽ không hiện |
| Firmware | Hỗ trợ cả firmware cũ (< 11) lẫn mới (≥ 11) |

## Cách sử dụng

1. Chạy server tĩnh cho thư mục (hoặc dùng bất kỳ host HTTPS nào):

   ```bash
   python -m http.server 8080
   # mở http://localhost:8080
   ```

2. Popup kết nối hiện ngay — nhấn **Kết nối Bluetooth** rồi chọn BMS trong danh sách. Trang hiển thị **tất cả** thiết bị BLE lân cận (nhiều module JK quảng cáo tên không bắt đầu bằng "JK" nên không lọc theo tên); sau khi chọn, trang tự kiểm tra service JK (FFE0).
3. Xem số liệu: vòng tròn SOC, các ô kèm biểu đồ 30 giây, lưới điện áp cell — cập nhật liên tục (BMS tự đẩy khung trạng thái ~1 giây/lần sau lần hỏi đầu).
4. Dùng mục **Điều khiển BMS** để bật/tắt sạc / xả / cân bằng, hoặc bật Emergency (có hộp thoại cảnh báo; trạng thái nút tự đồng bộ lại từ BMS sau ~0.4 giây).
5. Thanh công cụ: chọn phiên bản khung (khuyến nghị để **Tự động**), nút chuyển **EN/VI**, nút đổi giao diện tối/sáng.
6. Xử lý sự cố:
   - *"Không có service JK-BMS (FFE0)"* — chọn nhầm thiết bị, hoặc BMS đang bị ứng dụng chính hãng chiếm (hãy ngắt kết nối app trên điện thoại trước).
   - Danh sách trống — kiểm tra module BLE của BMS đã lên nguồn, chưa nối nơi khác; Bluetooth máy đã bật.
   - Số liệu sai lệch — thử chọn tay phiên bản khung (`fw < 11` hoặc `fw ≥ 11`).
   - Trang không đổi sau khi cập nhật file — nhấn `Ctrl+F5` để phá cache.

> ⚠️ **Cảnh báo an toàn**: các nút điều khiển ghi trực tiếp vào BMS. Chế độ Emergency vô hiệu hóa bảo vệ cell, có thể làm hỏng pin hoặc gây nguy hiểm. Sử dụng trên rủi ro của bạn.

## Giải thích kỹ thuật

### Tầng BLE

- Service GATT `0xFFE0`, characteristic `0xFFE1` (một số module có 2 characteristic cùng UUID FFE1: một chỉ ghi, một notify — trang chọn theo thuộc tính khai báo).
- `requestDevice({ acceptAllDevices: true, optionalServices: [0xFFE0] })` — không lọc theo tên vì nhiều module JK quảng cáo tên khác/rỗng.
- Characteristic notify là **cầu UART thô**: gói tin không tuân theo ranh giới khung, nên trang tự tích lũy bộ đệm, dò lại header `55 AA EB 90` và tiêu thụ khung đúng 300 byte.

### Khung lệnh gửi đi (20 byte)

```
AA 55 90 EB | ADDR | LEN | VALUE(4, LE) | zero-pad tới 19 | CHECKSUM
```

- `CHECKSUM` = tổng 8-bit của byte 0..18.
- Lệnh dùng: `0x96` (hỏi dữ liệu cell/trạng thái), `0x97` (hỏi thông tin thiết bị).
- Ghi công tắc: `ADDR` = `0x1D` sạc, `0x1E` xả, `0x1F` cân bằng, `0x6B` emergency (chỉ fw ≥ 11); `LEN` = 4, giá trị `0x00000001` / `0x00000000`.
- Hỏi vòng lặp: `0x97` một lần sau khi nối, `0x96` mỗi 2 giây tới khi có dữ liệu, sau đó chỉ giữ kết nối nếu im lặng > 9 giây (BMS tự đẩy dữ liệu).

### Khung phản hồi (300 byte)

```
55 AA EB 90 | LOẠI | BỘ ĐẾM | payload... | CHECKSUM(byte cuối = tổng byte 0..298)
```

Chấp nhận loại: `0x01` cài đặt, `0x02` cell info, `0x03` device info. Phải đúng checksum **và** loại đã biết (tổng 8-bit có ~1/256 xác suất khớp nhầm với rác giống header).

### Tự nhận diện phiên bản

Khung device info (`0x03`) chứa phiên bản firmware tại offset 30. `major ≥ 11` → layout "mới" (mọi trường sau khối cell/điện trở bị dịch **+32** byte); ngược lại là layout 24S cũ. Cách này giống batmon-ha và tránh việc đoán sai qua bitmask cell.

### Bảng offset khung cell info (`0x02`)

| Offset | Trường | Hệ số |
|---|---|---|
| 6 + 2·i | điện áp cell i (24 hoặc 32 ô) | mV |
| 54..57 | bitmask cell đang bật | — |
| 118+off | điện áp tổng | 0.001 V |
| 126+off | dòng điện (có dấu; **dương = đang sạc**) | 0.001 A |
| 130+off / 132+off | nhiệt độ cảm biến 1 / 2 (−2000 = không có) | 0.1 °C |
| 112+off (mới) / 134+off (cũ) | nhiệt độ MOS | 0.1 °C |
| 134+off (mới) / 136+off (cũ) | bitmask lỗi | — |
| 138+off | dòng cân bằng | 0.001 A |
| 140+off | trạng thái cân bằng (0 nghỉ / 1 sạc / 2 xả) | — |
| 141+off | byte SOC (độ phân giải 1%) | % |
| 142+off | dung lượng còn lại | 0.001 Ah |
| 146+off | dung lượng lão hóa (fw ≥ 11) / danh định (cũ) | 0.001 Ah |
| 150+off | số chu kỳ | — |
| 158+off | SOH (fw ≥ 11) | % |
| 162+off | thời gian chạy | giây |
| 166+off / 167+off | MOS sạc / xả đang bật | bool |
| 186+off | bộ đếm thời gian emergency | giây |

`off` = 32 với layout mới, 0 với layout cũ.

### Khung cài đặt (`0x01`)

Số cell @114, công tắc sạc/xả/cân bằng @118/122/126, dung lượng do người dùng cấu hình @130 (0.001 Ah). **Dung lượng cấu hình lấy từ khung này** — trên fw ≥ 11, trường dung lượng trong khung cell info là giá trị lão hóa nội bộ của BMS, lệch dần so với mức người dùng đặt.

### Độ chính xác SOC

Byte SOC chỉ có độ phân giải nguyên %. Trên fw ≥ 11, trang tính lại `còn lại / lão hóa × 100` (đều là giá trị nội bộ của BMS), khôi phục độ chính xác số lẻ giống app chính hãng (VD `65.87 %` thay vì `66 %`).

## Nguồn tham khảo (Credits)

Dự án này dựa trên những dự án mã nguồn mở tuyệt vời sau:

- **[syssi/esphome-jk-bms](https://github.com/syssi/esphome-jk-bms)** — tài liệu giao thức JK02/JK04 chú thích từng byte, địa chỉ thanh ghi công tắc (0x1D/0x1E/0x1F/0x6B) và định dạng khung ghi.
- **[fl4p/batmon-ha](https://github.com/fl4p/batmon-ha)** (và thư viện `bmslib` kèm theo) — chiến thuật dò lại khung `feed_frames` bền bỉ, nhận diện layout theo phiên bản firmware (SW ≥ 11 → offset 32), bản sửa lấy dung lượng từ khung cài đặt (issue #365), tính lại SOC số lẻ (issue #369), cùng các khung dữ liệu thật dùng làm fixture kiểm thử.
- **[sstallion/go-jk-bms](https://github.com/sstallion/go-jk-bms)** và **[jblance/mpp-solar](https://github.com/jblance/mpp-solar)** — tài liệu giao thức tham khảo thêm được hai dự án trên dẫn tới.
- **[Web Bluetooth API](https://developer.mozilla.org/docs/Web/API/Web_Bluetooth_API)** — tài liệu MDN và ghi chú triển khai của Chrome.
- Biến theme trong `style.css` theo quy ước biến CSS của shadcn/ui.

Phục vụ mục đích cá nhân và học tập; không kèm bảo hành. Làm việc với pin tiềm ẩn nguy hiểm — hãy tự kiểm chứng mọi thứ liên quan đến an toàn.
