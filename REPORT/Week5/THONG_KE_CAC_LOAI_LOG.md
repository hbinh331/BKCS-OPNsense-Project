# Thống kê các log trong OPNsense

## 0. Tóm tắt nhanh luồng tạo log và hiển thị

1. **Event phát sinh và đi vào syslog:** syslog-ng nhận message nội bộ của
   chính nó qua `internal()`, kernel log qua `/dev/klog`, và message từ các
   tiến trình qua socket `/var/run/log` hoặc socket đặc quyền
   `/var/run/logpriv`. Các source trong cấu hình được gom vào `s_all`.
2. **Phân loại:** cấu hình local tạo filter theo `program`, `facility`, đôi
   khi cả `message`. Các filter được ghép vào cấu hình syslog-ng.
3. **Lưu cục bộ:** khi bật `loglocal`, message khớp filter được ghi theo ngày
   vào `/var/log/<nhóm>/<nhóm>_YYYYMMDD.log`. Message chưa khớp filter nào
   khác được đưa vào nhóm `system` nếu severity thuộc `notice..emerg`.
4. **Fetch qua API/backend:** GUI gọi
   `/api/diagnostics/log/{module}/{scope}`. Diagnostics API chuyển tham số
   tìm kiếm, severity, thời gian và phân trang tới action `system diag log`;
   action này chạy `queryLog.py`, dùng `LogMatcher` để tìm và parse log file.
   API cũng hỗ trợ export, live stream và clear.
5. **Hiển thị:** GUI hiển thị ngày, severity, process và nội dung. Facility và
   PID có trong bảng nhưng bị ẩn mặc định. API trả các record cùng metadata
   phân trang.

**Code:**

- Nguồn log: [`001-local.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/sources/001-local.conf#L1-L4)
- Gom source: [`syslog-ng.conf.in`](../../core/src/opnsense/service/templates/OPNsense/Syslog/syslog-ng.conf.in#L16-L28)
- Filter và file đích: [`syslog-ng-local.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/syslog-ng-local.conf#L1-L52)
- API Diagnostics: [`Diagnostics Api LogController.php`](../../core/src/opnsense/mvc/app/controllers/OPNsense/Diagnostics/Api/LogController.php#L39-L101)
- Action backend: [`actions_system.conf`](../../core/src/opnsense/service/conf/actions.d/actions_system.conf#L8-L37)
- Đọc và parse: [`queryLog.py`](../../core/src/opnsense/scripts/syslog/queryLog.py#L43-L76),
  [`log_matcher.py`](../../core/src/opnsense/scripts/syslog/log_matcher.py#L87-L154)
- GUI: [`log.volt`](../../core/src/opnsense/mvc/app/views/OPNsense/Diagnostics/log.volt#L241-L250)

**Tài liệu chính thức cho các lệnh/API bên ngoài:**

- [FreeBSD `logger(1)`](https://man.freebsd.org/cgi/man.cgi?query=logger&sektion=1)
  mô tả cú pháp và các tùy chọn `-p`, `-t`.
- [FreeBSD `syslog(3)`](https://man.freebsd.org/cgi/man.cgi?query=syslog&sektion=3)
  mô tả các hàm `syslog()` và `openlog()` mà code dùng để gửi log.
- [FreeBSD `syslog.conf(5)`](https://man.freebsd.org/cgi/man.cgi?query=syslog.conf&sektion=5)
  giải thích `facility` và mức độ (`severity`) như `user.err`.
- [FreeBSD `syslogd(8)`](https://man.freebsd.org/cgi/man.cgi?query=syslogd&sektion=8)
  mô tả dịch vụ syslog của FreeBSD.
- [FreeBSD `dmesg(8)`](https://man.freebsd.org/cgi/man.cgi?query=dmesg&sektion=8)
  mô tả cách đọc vùng lưu thông báo kernel; lệnh này đọc log đã có, không
  tạo log mới.
- [FreeBSD `tail(1)`](https://man.freebsd.org/cgi/man.cgi?query=tail&sektion=1)
  mô tả cách theo dõi các dòng mới trong file; `streamLog.py` dùng `tail -f`
  cho live log.
- [syslog-ng `syslog-ng-ctl(1)`](https://github.com/syslog-ng/syslog-ng/blob/develop/doc/man/local/syslog-ng-ctl.1)
  là manual upstream cho lệnh `syslog-ng-ctl stats` được gọi trong action
  Statistics.

Các hàm như `LogController::__call()`, `LogMatcher.match_records()` và các
action `configdRun()` xem ở các bản source tương ứng trong repo của OPNsense:
[LogController.php](https://github.com/opnsense/core/blob/master/src/opnsense/mvc/app/controllers/OPNsense/Diagnostics/Api/LogController.php),
[queryLog.py](https://github.com/opnsense/core/blob/master/src/opnsense/scripts/syslog/queryLog.py),
[log_matcher.py](https://github.com/opnsense/core/blob/master/src/opnsense/scripts/syslog/log_matcher.py),
[streamLog.py](https://github.com/opnsense/core/blob/master/src/opnsense/scripts/syslog/streamLog.py),
[AuthenticationFactory.php](https://github.com/opnsense/core/blob/master/src/opnsense/mvc/app/library/OPNsense/Auth/AuthenticationFactory.php),
[Config.php](https://github.com/opnsense/core/blob/master/src/opnsense/mvc/app/library/OPNsense/Core/Config.php),
[ControllerRoot.php](https://github.com/opnsense/core/blob/master/src/opnsense/mvc/app/controllers/OPNsense/Base/ControllerRoot.php)
và [FirmwareController.php](https://github.com/opnsense/core/blob/master/src/opnsense/mvc/app/controllers/OPNsense/Core/Api/FirmwareController.php).

## 1. Log đến từ system

"system" = log kernel, log nội bộ, các log ko match filter nào (system catch-all)

### Proof of Concept   

>Chỉ thử trên OPNsense lab/VM riêng. **Shell** là cửa sổ dòng lệnh của OPNsense (mở bằng console hoặc SSH). Sau thao tác, vào **Diagnostics > Log File**, chọn đúng nhóm log và tìm nội dung được ghi trong hướng dẫn. Cần bật **local logging** (ghi log trên chính máy OPNsense) để xem các file log cục bộ.

>Hai kiểu kiểm tra trong cột PoC có mục đích khác nhau:
>- **PoC sự kiện thật:** thao tác bình thường trên lab khiến dịch vụ/kernel thật phát log, ví dụ client test xin DHCP lease hoặc đăng nhập VPN test. Đây mới là cách kiểm tra event của thành phần đó.
>- **Kiểm tra đường ghi log:** lệnh `logger` gửi một dòng mẫu vào syslog để xem nó có được lưu/hiển thị ở nhóm dự kiến không. Nó không chạy dịch vụ, không tạo lỗi của dịch vụ và không chứng minh dịch vụ thật đã phát log. Hướng dẫn sẽ ghi rõ khi chỉ đang kiểm tra đường ghi log.

#### Cú pháp lệnh `logger` (dùng để tạo PoC ở 1 số loại log)

Syntax:

```sh
logger -p <facility>.<severity> -t <tag> '<message>'
```

Ví dụ kiểm tra nhóm system:

```sh
logger -p user.err -t poc-system 'POC-system-error'
```

- `logger`: gửi dòng nội dung vào syslog. Xem cú pháp đầy đủ trong
  [manual FreeBSD `logger(1)`](https://man.freebsd.org/cgi/man.cgi?query=logger&sektion=1).
- `-p`: chọn thông tin phân loại của dòng. Trong `user.err`, `user` là
  `facility` (nhóm syslog) và `err` là `severity` (mức lỗi). Đây chỉ là nhãn
  mức độ; lệnh không làm dịch vụ gặp lỗi.
- `-t poc-system`: gắn `tag` (nhãn) `poc-system` vào dòng. Đây không phải tên chương
  trình được khởi chạy; logger vẫn là bên gửi dòng.
- `'POC-system-error'`: `message` (nội dung) của dòng log. Chuỗi `POC-...` là **dấu tìm
  riêng** (marker), giúp nhận ra đúng dòng thử trong trang log; nó không phải
  thuật ngữ hay loại log đặc biệt.

Chạy lệnh trên tạo một bản ghi thử có nhãn `poc-system`, không tạo log kernel
hay log lỗi thật. Tìm `POC-system-error` trong nhóm system để kiểm tra đường
ghi log. Không dùng kết quả này làm bằng chứng rằng một daemon/kernel đã phát
log. Với kernel và syslog-ng nội bộ, chỉ quan sát log tự nhiên: `logger`
không thể tạo message có nguồn thật từ kernel hoặc từ chính syslog-ng.

Trong các hàng bên dưới, “nhóm log” nghĩa là mục tương ứng trong giao diện
Diagnostics; “bộ lọc” là quy tắc chọn dòng theo nhãn chương trình/nhóm syslog;
`internal()` nhận log do syslog-ng tự tạo; `/dev/klog` là nơi đọc log kernel.
Quyền truy cập quyết định tài khoản nào được mở trang nào.

### Khôi phục sau khi thử

- Lệnh `logger` chỉ thêm một bản ghi; không đổi cấu hình hoặc trạng thái dịch
  vụ. Không cần rollback và không nên xóa cả file log chỉ để xóa dòng thử.
- Dùng tài khoản thử nghiệm cho đăng nhập sai/kiểm tra quyền. Nếu không rõ
  chính sách khóa tài khoản, không thử sai mật khẩu.
- Khi PoC lưu cấu hình, ghi lại giá trị cũ, chỉ sửa trường mô tả vô hại trên
  VM lab, rồi khôi phục đúng giá trị cũ.
- Nếu cần quay lại snapshot, tắt VM thử nghiệm, dùng chức năng snapshot của
  hypervisor để khôi phục snapshot tạo trước PoC, khởi động lại và kiểm tra
  GUI/mạng/dịch vụ. Snapshot không hoàn tác ảnh hưởng ra ngoài VM như lease
  DHCP hoặc trạng thái peer trên máy khác; đừng dùng nó để biện minh cho thử
  nghiệm gây gián đoạn.
- Nếu không có VM riêng hoặc chưa kiểm tra được cách khôi phục, chỉ dùng
  kiểm tra đường ghi log hoặc quan sát log có sẵn.

| Loại log | Event/điều kiện tạo log | Thông tin xem được khi fetch | Câu log hiển thị cho người dùng | Reference code | Proof of Concept |
|---|---|---|---|---|---|
| Kernel | Kernel ghi thông điệp vào `/dev/klog`; syslog-ng đọc file thiết bị này và gán tên chương trình là `kernel`. Dòng cấu hình chứng minh nguồn log, không liệt kê toàn bộ loại kernel event. | Message kernel nằm trong `line`; timestamp, process, severity và metadata khác phụ thuộc định dạng thông điệp và parser nhận diện được. | Không có literal log message của kernel trong file cấu hình này; không thể xác định câu cụ thể chỉ từ source `/dev/klog`. GUI hiển thị message kernel thực tế ở `Line`. | [`001-local.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/sources/001-local.conf#L2) cấu hình đọc `/dev/klog`; [`log_matcher.py`](../../core/src/opnsense/scripts/syslog/log_matcher.py#L108-L133) parse record. | **Sự kiện kernel thật:** không có lệnh `logger` phù hợp để tạo sự kiện này. Chỉ xem log kernel đã phát sinh tự nhiên trong mục system; có thể dùng [`dmesg(8)`](https://man.freebsd.org/cgi/man.cgi?query=dmesg&sektion=8) để đọc buffer kernel hiện có. Lệnh `logger` gửi log từ shell, nên kết quả của nó không thể chứng minh kernel đã phát log. |
| Syslog-ng nội bộ | Khi syslog-ng tự sinh message nội bộ, source `internal()` nhận message đó. | Message và metadata chung của syslog; không có parser nghiệp vụ riêng trong code này. | Không có literal mẫu ở source `internal()`; `Line` chứa message do syslog-ng thực tế phát ra. | [`001-local.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/sources/001-local.conf#L1) chỉ khai báo source, không định nghĩa câu message. | **Log syslog-ng thật:** chỉ xem dòng syslog-ng tự ghi trong lúc hoạt động bình thường. `logger` là chương trình khác gửi log nên không thể tạo đúng nguồn `internal()`; không cố làm syslog-ng gặp lỗi. |
| System catch-all | Khi message không khớp bất kỳ filter local nào khác và severity từ `notice` đến `emerg`, syslog-ng ghi message vào destination `system`. Đây không phải bản sao toàn bộ mọi log: message mức thấp hơn hoặc đã được filter khác nhận không thuộc nhóm catch-all này. | Record chuẩn hóa như phần cấu trúc chung; nội dung event ở `line`. Trang system mặc định chọn severity `Notice`, có thể thay đổi ở GUI. | Không có một câu cố định: `system` là catch-all nên `Line` giữ nguyên message của event nguồn được chuyển vào nhóm này. | [`syslog-ng-local.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/syslog-ng-local.conf#L35-L52) định nghĩa điều kiện và destination system; [`Diagnostics LogController.php`](../../core/src/opnsense/mvc/app/controllers/OPNsense/Diagnostics/LogController.php#L68-L69) đặt severity mặc định. | **Thử đường ghi, không phải sự kiện system thật:** trên shell OPNsense lab chạy `logger -p user.err -t poc-system 'POC-system-error'`. Mở mục system, tìm `POC-system-error`; thấy dòng này nghĩa là syslog nhận được message thử, không phải một dịch vụ thật gặp lỗi. |
| Configd/API | Khi `configd.py` hoặc `api` phát syslog message, filter đưa vào nhóm `configd`, ngoại trừ message có `facility(auth)`. Filter này phân loại log; bản thân nó không định nghĩa tất cả event hoặc trường của API/configd. | Nội dung nguyên bản trong `line`, cùng metadata chung. | Không có literal câu log chung trong filter. Ví dụ cụ thể phụ thuộc action và emitter configd/API; GUI hiển thị nguyên message emitter gửi. | [`configd.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/local/configd.conf#L4-L5) lọc chương trình và loại trừ facility auth. | **Thử bộ lọc, không phải event API thật:** chạy `logger -p user.err -t configd.py 'POC-configd-test'`. Mở mục configd và tìm chuỗi đó. `-t configd.py` khớp tên chương trình mà bộ lọc cấu hình; logger không gọi configd/API. |

## 2. Log đến từ dịch vụ

Các filter dưới đây chứng minh syslog-ng phân loại message theo tên chương
trình/facility. Trừ nơi có emitter code cụ thể được dẫn, chi tiết event phụ
thuộc message của daemon

| Dịch vụ/nhóm |Event/điều kiện tạo log | Thông tin hiển thị khi fetch | Câu log hiển thị cho người dùng | Reference code | Proof of Concept |
|---|---|---|---|---|---|
| Firewall filter (`filterlog`) | Khi chương trình `filterlog` phát syslog message. Đây là log filter riêng, không nên gộp với nhóm `firewall`. | Message thô ở `line`; metadata chung nếu parser trích xuất được. Code filter ở đây không chứng minh các trường packet cụ thể được tách thành cột riêng. | Không có literal log message trong filter. GUI hiển thị nguyên dòng do `filterlog` gửi; không thể khẳng định format chỉ từ file filter. | [`filter.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/local/filter.conf#L4-L5) chỉ lọc `program("filterlog")`. | **Sự kiện thật trong lab:** dùng một client test riêng nối với VM OPNsense. Tạo tạm rule chặn và bật ghi log cho đúng một gói ping (ICMP) từ client test; gửi đúng một ping rồi kiểm tra mục filter. Xóa rule tạm ngay sau đó. Không áp dụng rule lên WAN, đường quản trị hoặc mạng đang dùng. Nếu chỉ muốn kiểm tra hiển thị: `logger -p user.err -t filterlog 'POC-filterlog-test'`; dòng này không đại diện cho gói bị chặn thật. |
| Firewall (`firewall`) | Khi chương trình `firewall` gửi syslog message. Filter không mô tả event chi tiết nào ngoài nguồn chương trình. | Message thô và metadata chung. | Không có câu mẫu trong filter; `Line` là message nguyên bản của chương trình `firewall`. | [`firewall.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/local/firewall.conf#L4-L5) chỉ nhận diện program. | **Thử bộ lọc:** chạy `logger -p user.err -t firewall 'POC-firewall-test'`, rồi tìm chuỗi trong mục firewall. Đây chỉ là log mẫu mang nhãn `firewall`, không phải một sự kiện firewall thật; không cần đổi rule. |
| DHCP Kea | Khi một trong các tiến trình Kea DHCPv4, DHCPv6, control agent hoặc DDNS gửi syslog. Code watcher/renew của OPNsense còn ghi một số event prefix IPv6: watcher khởi động, thêm route, lỗi tìm next hop/thêm route; renew ghi lỗi xóa lease động, tạo cấu hình hoặc reload cấu hình. | Các giá trị như prefix, interface, địa chỉ hoặc lỗi (nếu có trong message) được hiển thị trong `line`, không phải trường lease được parser chung tách riêng. | Ví dụ literal/template: `startup kea prefix watcher`; `add route <prefix> -> <next-hop>`; `failed adding route <prefix> -> <next-hop>`. Dấu ngoặc nhọn biểu thị giá trị runtime được chèn vào chuỗi. | [`kea.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/local/kea.conf#L4-L5) lọc bốn chương trình Kea; [`kea_prefix_watcher.py`](../../core/src/opnsense/scripts/kea/kea_prefix_watcher.py#L110-L139) tạo các message mẫu; [`kea_prefix_renew.py`](../../core/src/opnsense/scripts/kea/kea_prefix_renew.py#L37-L69) ghi một số lỗi renew/reload. | **Sự kiện thật:** từ client test trên mạng lab, xin hoặc gia hạn một lease DHCP bình thường; tìm lease/event tương ứng trong log Kea. Không sửa cấu hình DHCP để cố tạo lỗi. Nếu chỉ thử đường ghi: `logger -p user.err -t kea-dhcp4 'POC-kea-test'`; đó là log mẫu, không phải Kea đã phát. |
| DHCP relay | Khi tiến trình `dhcrelay` phát syslog. Filter không mô tả các loại bản tin DHCP relay cụ thể. | Message thô và metadata chung. | Không có literal mẫu trong filter; GUI hiển thị message do `dhcrelay` phát. | [`dhcrelay.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/local/dhcrelay.conf#L4-L5) chỉ lọc program. | **Sự kiện thật:** trên lab đã cấu hình DHCP relay, cho client test xin/gia hạn lease một lần và kiểm tra log relay. Không gửi DHCP vào mạng không thuộc quyền quản trị. `logger -p user.err -t dhcrelay 'POC-relay-test'` chỉ thử bộ lọc, không mô phỏng trao đổi DHCP. |
| Unbound / resolver | Khi Unbound hoặc thành phần logger gửi message. Logger OPNsense có các message cụ thể như đóng logger, lỗi export database, lỗi mở pipe và các bước restore. | Message cụ thể ở `line`; parser chung không tách truy vấn DNS thành các cột riêng. | Ví dụ literal: `Closing logger`; `Unable to open pipe. This is likely because Unbound isn't running.`; `Restore needed, but backup locked, exit...` | [`resolver.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/local/resolver.conf#L4-L5) lọc `unbound`; [`logger.py`](../../core/src/opnsense/scripts/unbound/logger.py#L129-L129), [`logger.py`](../../core/src/opnsense/scripts/unbound/logger.py#L223-L223), [`logger.py`](../../core/src/opnsense/scripts/unbound/logger.py#L271-L271) tạo các message mẫu. | **Sự kiện thật:** từ client lab, hỏi resolver một tên miền thử nghiệm đã biết; kiểm tra log nếu cấu hình hiện tại có bật ghi truy vấn. Không dừng Unbound hoặc làm hỏng pipe/cấu hình. Để chỉ thử bộ lọc: `logger -p user.err -t unbound 'POC-unbound-test'`; logger không thực hiện truy vấn DNS. |
| Dnsmasq | Khi tiến trình `dnsmasq` gửi syslog. Filter không chứng minh rằng mọi truy vấn DNS hoặc DHCP đều được log. | Message thô và metadata chung. | Không có literal mẫu trong filter; `Line` hiển thị message từ daemon. | [`dnsmasq.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/local/dnsmasq.conf#L4-L5) chỉ lọc program. | **Sự kiện thật nếu dịch vụ đang được dùng:** từ client lab gửi một truy vấn DNS hoặc xin lease bình thường rồi tìm log dnsmasq. Nếu dnsmasq không hoạt động thì không thay cấu hình để bật chỉ nhằm thử. `logger -p user.err -t dnsmasq 'POC-dnsmasq-test'` chỉ thử bộ lọc. |
| Gateway monitor (`dpinger`) | Khi `dpinger` phát syslog message. Filter chỉ xác định nguồn, không nêu các số liệu gateway được ghi trong từng message. | Message thô; không có trường gateway riêng được parser chung tạo ra. | Không có literal mẫu trong filter; không nên tự suy ra câu có RTT/loss/state khi chưa có emitter cụ thể. | [`gateways.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/local/gateways.conf#L4-L5) chỉ lọc `dpinger`. | **Không chủ động làm hỏng kết nối để tạo lỗi:** xem log dpinger tự phát sinh khi monitor hoạt động bình thường. `logger -p user.err -t dpinger 'POC-dpinger-test'` chỉ kiểm tra bộ lọc và nơi hiển thị; nó không kiểm tra trạng thái gateway. |
| Hostwatch | Khi chương trình `hostwatch` phát syslog. | Message thô và metadata chung. | Không có literal mẫu trong filter; GUI hiển thị message do `hostwatch` gửi. | [`hostwatch.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/local/hostwatch.conf#L4-L5) chỉ lọc program. | **Sự kiện thật:** nếu hostwatch đang bật, kết nối một client test vào interface lab theo cách bình thường rồi tìm event host mới. Nếu chỉ cần thử bộ lọc, chạy `logger -p user.err -t hostwatch 'POC-hostwatch-test'`; không cần đổi interface. |
| IPsec (`charon`) | Khi daemon `charon` gửi syslog. Nội dung IKE/tunnel cụ thể do daemon phát; filter chỉ định tuyến log theo tên chương trình. | Message thô và metadata chung; không có cột SA/IKE riêng từ parser chung. | Không có literal mẫu trong filter; `Line` giữ message do `charon` phát. | [`ipsec.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/local/ipsec.conf#L4-L5) chỉ lọc program. | **Sự kiện thật:** trong lab có peer test đã cấu hình, khởi tạo kết nối tunnel bình thường và tìm log charon. Không làm rớt tunnel production hoặc sửa peer để tạo lỗi. `logger -p user.err -t charon 'POC-charon-test'` chỉ thử bộ lọc. |
| `lighttpd` / Web GUI server | Khi `lighttpd` phát syslog message. Filter không tự chứng minh mọi HTTP request hay mọi thao tác GUI đều được ghi tại đây. | Message thô và metadata chung. | Không có literal mẫu trong filter; không thể kết luận mọi request HTTP đều xuất hiện ở GUI log từ filter này. | [`lighttpd.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/local/lighttpd.conf#L4-L5) chỉ lọc program. | **Sự kiện thật:** mở một trang GUI bình thường trên VM lab rồi kiểm tra log lighttpd (chỉ khi cấu hình gửi log này vào syslog). `logger -p user.err -t lighttpd 'POC-lighttpd-test'` chỉ thử bộ lọc; nó không phải request do web server ghi nhận. |
| Monit | Khi `monit` gửi syslog. Nội dung phụ thuộc cấu hình và message của Monit. | Message thô và metadata chung. | Không có literal mẫu trong filter; `Line` phụ thuộc message Monit thực tế. | [`monit.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/local/monit.conf#L4-L5). | **Sự kiện thật:** chờ lần kiểm tra định kỳ bình thường của Monit và xem log đã có. Không dừng service để tạo cảnh báo. `logger -p user.err -t monit 'POC-monit-test'` chỉ thử bộ lọc, không chứng minh Monit phát hiện lỗi. |
| NTP | Khi `ntp`, `ntpd` hoặc `ntpdate` gửi syslog. | Message thô và metadata chung. | Không có literal mẫu trong filter; `Line` hiển thị message do các daemon NTP gửi. | [`ntpd.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/local/ntpd.conf#L4-L5). | **Sự kiện thật:** để NTP đồng bộ theo lịch bình thường rồi xem log nếu daemon có gửi message vào syslog. Không chỉnh lệch đồng hồ hoặc dừng NTP. `logger -p user.err -t ntpd 'POC-ntp-test'` chỉ thử bộ lọc. |
| OpenVPN | Khi OpenVPN gửi message. Emitter trong OPNsense ghi user xác thực, auth method (khi thành công), thông báo lỗi xác thực; quản lý instance ghi role/ID/PID khi start và cảnh báo timeout khởi động. | Username, auth method, instance/PID hoặc lỗi (nếu có) nằm trong `line`; không được tách thành cột nghiệp vụ. | Ví dụ template: `user '<username>' authenticated using '<authName>'`; `OpenVPN <role> <vpnid> instance started on PID <pid>.`; `OpenVPN <role> <vpnid> instance start timed out.` | [`openvpn.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/local/openvpn.conf#L4-L5) lọc chương trình; [`user_pass_verify.php`](../../core/src/opnsense/scripts/openvpn/user_pass_verify.php#L153-L179) tạo message xác thực; [`ovpn_service_control.php`](../../core/src/opnsense/scripts/openvpn/ovpn_service_control.php#L72-L74) tạo message start/timeout. | **Sự kiện thật:** kết nối bằng client và tài khoản test tới VPN lab; kiểm tra log xác thực thành công. Nếu cần kiểm tra thất bại, chỉ nhập sai mật khẩu một lần cho tài khoản thử nghiệm đã xác nhận không bị khóa sau lần sai. `logger -p user.err -t openvpn 'POC-openvpn-test'` chỉ thử bộ lọc, không phải kết quả xác thực. |
| Package (`pkg`) | Khi `pkg` hoặc `pkg-static` phát syslog. Filter không mô tả chính xác các event cài đặt/nâng cấp nào được ghi. | Message thô và metadata chung. | Không có literal mẫu trong filter; `Line` hiển thị message do `pkg`/`pkg-static` phát. | [`pkg.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/local/pkg.conf#L4-L5) chỉ lọc program. | **Không cần cài/gỡ package để thử:** xem log từ thao tác package đã có. Để chỉ thử bộ lọc, chạy `logger -p user.err -t pkg 'POC-pkg-test'`; thao tác này không gọi pkg và không thay đổi package. |
| Captive portal | Khi tiến trình `captiveportal` gửi syslog. Filter cho biết nguồn, không đặc tả trường phiên portal. | Message thô và metadata chung; thông tin user/session chỉ có nếu được daemon đưa vào message. | Không có literal mẫu trong filter; không thể khẳng định Session ID/IP/MAC có mặt nếu chưa thấy message emitter. | [`portalauth.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/local/portalauth.conf#L4-L5) chỉ lọc program. | **Sự kiện thật:** trên portal lab, đăng nhập rồi đăng xuất bằng tài khoản và client test. Nếu muốn kiểm tra đăng nhập sai, làm một lần duy nhất và chỉ khi biết tài khoản không bị khóa; nếu không, dùng `logger -p user.err -t captiveportal 'POC-portal-test'` để chỉ thử bộ lọc. |
| Routing daemons | Khi `routed`, `olsrd`, `zebra`, `ospfd` hoặc `bgpd` gửi syslog. | Message thô; parser chung không giải mã route/protocol thành cột. | Không có literal mẫu trong filter; `Line` là message do routing daemon phát. | [`routing.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/local/routing.conf#L4-L9) chỉ lọc program names. | **Không chủ động làm thay đổi định tuyến:** xem log do routing daemon tạo trong quá trình hoạt động/hội tụ bình thường ở lab. `logger -p user.err -t zebra 'POC-routing-test'` chỉ thử bộ lọc; không thêm/xóa route hay làm rớt peer. |
| Suricata / IDS | Khi Suricata gửi log. Template EVE khai báo loại event alert và có cấu hình HTTP/TLS; output EVE/syslog và output syslog alert có các tùy chọn bật riêng. Không nên giả định mọi loại event đều đang được bật trên mọi cấu hình đang chạy. | EVE message nằm trong `line`; GUI log chung không dựng cột alert/HTTP/TLS riêng. | Không có câu message literal trong filter hoặc YAML; các file xác nhận loại EVE (`alert`, `http`, `tls`) chứ không cố định nội dung từng alert. `Line` hiển thị message EVE thực tế. | [`suricata.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/local/suricata.conf#L4-L5) lọc chương trình; [`suricata.yaml`](../../core/src/opnsense/service/templates/OPNsense/IDS/suricata.yaml#L50-L70) khai báo EVE types; [`suricata.yaml`](../../core/src/opnsense/service/templates/OPNsense/IDS/suricata.yaml#L86-L118) cấu hình EVE/syslog. | **Sự kiện thật:** chỉ chạy rule/PCAP kiểm thử vô hại đã chuẩn bị trong mạng lab cô lập và kiểm tra alert tương ứng. Không tạo traffic tấn công trên mạng thật. `logger -p user.err -t suricata 'POC-suricata-test'` chỉ thử đường syslog, không chứng minh IDS phát hiện gì. |
| VPN facility `local3` | Khi message có facility `local3`; filter nhóm theo facility, không xác định một daemon VPN cụ thể. | Message thô; chỉ dựa vào filter này không thể xác định dịch vụ phát log. | Không có literal mẫu vì filter chỉ dựa vào facility; `Line` phụ thuộc emitter gửi message. | [`vpn.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/local/vpn.conf#L4-L5). | **Thử phân loại theo nhóm syslog:** chạy `logger -p local3.err -t poc-vpn 'POC-local3-test'`, rồi tìm chuỗi trong mục VPN. `local3.err` chọn nhóm `local3` và mức lỗi; đây là log mẫu, không phải VPN thật phát lỗi. |
| WireGuard | Khi `wireguard` phát message hoặc kernel message khớp tên interface `wg` kèm số. Code quản lý dịch vụ ghi instance/interface start và stop. | Nội dung như tên instance/interface nằm trong `line`. | Ví dụ template: `wireguard instance <name> (<interface>) started`; `wireguard instance <name> (<interface>) stopped`. | [`wireguard.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/local/wireguard.conf#L4-L6) lọc; [`wg-service-control.php`](../../core/src/opnsense/scripts/wireguard/wg-service-control.php#L149-L160) tạo hai message mẫu. | **Sự kiện thật:** kết nối peer WireGuard test đã cấu hình trong lab và tìm log kết nối. Không ngắt peer đang dùng. `logger -p user.err -t wireguard 'POC-wireguard-test'` chỉ thử bộ lọc, không tạo kết nối WireGuard. |

### Dịch vụ được ánh xạ ở trang Diagnostics nhưng cần kiểm tra thêm

Diagnostics controller có ánh xạ scope cho `ipsec`, `resolver`, `suricata`,
`squid` và `dhcpd`. Trong các file local filter được khảo sát có filter Kea và
DHCP relay, nhưng không thấy `squid.conf` hoặc `dhcpd.conf`. Vì vậy việc có
ánh xạ trong controller không đủ để kết luận dịch vụ đó có destination local
riêng trong cấu hình này; cần kiểm tra thêm file legacy hoặc cấu hình package
trên hệ thống đang chạy.

**Code:** [`Diagnostics LogController.php`](../../core/src/opnsense/mvc/app/controllers/OPNsense/Diagnostics/LogController.php#L49-L70)
định nghĩa ánh xạ UI; [`local`](../../core/src/opnsense/service/templates/OPNsense/Syslog/local)
có các filter hiện hành; [`log_matcher.py`](../../core/src/opnsense/scripts/syslog/log_matcher.py#L138-L154)
hỗ trợ cả đường dẫn log legacy.

## 3. Log đến từ người dùng

“Log đến từ người dùng” ở đây nghĩa là **log ghi nhận event do người dùng thực
hiện hoặc liên quan tới danh tính/phiên/quyền của người dùng**. Các event này
thường được phát bởi auth/framework hoặc bởi một dịch vụ như OpenVPN; chúng
không nhất thiết nằm trong một file riêng tên `user`.

| Loại log | Event/điều kiện tạo log | Thông tin khi xem được khi fetch | Câu log hiển thị cho người dùng  | Reference code | Proof of Concept |
|---|---|---|---|---|---|
| Xác thực thành công | Trong `authenticate()`, khi authenticator xác thực được và service constraints đạt, code ghi username, tên service, class của service và authenticator. | Các trường chung của log; username/service/authenticator nằm trong message `line`, không có cột username riêng. | `user <username> authenticated successfully for <service> [using <service-class> + <authenticator-class>]` | [`AuthenticationFactory.php`](../../core/src/opnsense/mvc/app/library/OPNsense/Auth/AuthenticationFactory.php#L187-L200) mở syslog với ident `audit` và tạo câu mẫu; [`audit.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/local/audit.conf#L4-L5) nhận `program("audit")` hoặc `facility(auth)`. | **Sự kiện thật:** đăng nhập GUI một lần bằng tài khoản thử nghiệm hợp lệ của bạn; tìm message xác thực thành công trong mục audit. Không dùng tài khoản của người khác. |
| Xác thực thất bại / constraint từ chối | Khi authenticator thất bại, service constraint không đạt, hoặc hết các authenticator mà không xác thực được. Code ghi message `WARNING` cho constraint/không xác thực được và `DEBUG` cho một nhánh thất bại. | Message có username, service và thông tin authenticator/constraint khi emitter đưa vào. Có thể cần chọn mức `Debug` trên UI để xem log debug. | Ví dụ template: `user <username> could not authenticate for <service>. [using <service-class> + <authenticator-class>]`; nhánh khác ghi `user <username> failed authentication for <service> on <service-class> via <authenticator-class>`. | [`AuthenticationFactory.php`](../../core/src/opnsense/mvc/app/library/OPNsense/Auth/AuthenticationFactory.php#L197-L232) chứa các chuỗi và severity; [`log.volt`](../../core/src/opnsense/mvc/app/views/OPNsense/Diagnostics/log.volt#L88-L100) gửi severity được chọn. | **Sự kiện thật trong lab:** nhập sai mật khẩu đúng một lần bằng tài khoản thử nghiệm đã xác nhận không có chính sách khóa sau lần sai, rồi tìm log audit. Nếu không rõ chính sách khóa, không thử sai mật khẩu; dùng `logger -p auth.err -t audit 'POC-auth-failure-test'` chỉ để thử bộ lọc audit, không phải log xác thực thất bại thật. |
| Không có session, session hết hạn, không được truy cập URI | Trong kiểm tra quyền request, code audit khi không có user/session hoạt động, khi session timeout hoặc URI không được ACL cho phép. Message có thể ghi username, URI và remote address tùy nhánh. | Thông tin chi tiết nằm trong `line`; metadata hiển thị theo bảng Diagnostics. | Ví dụ template: `no active session, user not found (called "<request-uri>" @ <remote-address>)`; `session expired (<username> @ <remote-address>)`; `uri <request-uri> not accessible for user <username>`. | [`ControllerRoot.php`](../../core/src/opnsense/mvc/app/controllers/OPNsense/Base/ControllerRoot.php#L140-L173) chứa các chuỗi audit được hiển thị; placeholder là giá trị runtime. | **Sự kiện thật:** dùng tài khoản thử nghiệm chỉ có quyền hạn chế để mở một trang vô hại mà tài khoản đó không được phép xem; xác nhận trang không mở và tìm log audit. Không dùng URI chứa dữ liệu nhạy cảm. Nếu không có tài khoản thử nghiệm phù hợp, chỉ xem log có sẵn. |
| Thay đổi cấu hình | Khi lưu thay đổi cấu hình, code xác định username revision từ session hoặc user hệ thống; nếu có, gắn remote IP. Audit ghi username, người impersonate (nếu có), tên file backup, URI/script và mô tả revision. | Các dữ liệu trên là nội dung message audit; API không tạo field username/URI riêng trong record chung. | Template: `user <username><impersonation> changed configuration to <backup-filename> in <request-uri-or-script> <description>` | [`Config.php`](../../core/src/opnsense/mvc/app/library/OPNsense/Core/Config.php#L470-L475) tạo username/context; [`Config.php`](../../core/src/opnsense/mvc/app/library/OPNsense/Core/Config.php#L551-L560) tạo câu log; [`Config.php`](../../core/src/opnsense/mvc/app/library/OPNsense/Core/Config.php#L760-L760) gọi audit sau thay đổi. | **Sự kiện thật trong VM lab:** ghi lại giá trị mô tả cũ của một đối tượng test, đổi mô tả đó, lưu, xác nhận log audit rồi khôi phục giá trị cũ và lưu lần nữa. Không đổi firewall, địa chỉ, route hay cấu hình ảnh hưởng kết nối. |
| Thao tác firmware/package có audit | Một số action trong Firmware API ghi username thực hiện reboot, poweroff, firmware update/upgrade, security audit, plugins sync hoặc thao tác package. Đây là các event có emitter cụ thể; không kết luận rằng mọi thao tác admin đều có audit tương tự. | Tên user và hành động nằm trong message audit, cùng metadata syslog chung. | Ví dụ template: `[Firmware] User <username> executed a reboot`; các action khác thay nội dung thành poweroff, firmware update/upgrade, audit hoặc plugins sync. | [`FirmwareController.php`](../../core/src/opnsense/mvc/app/controllers/OPNsense/Core/Api/FirmwareController.php#L426-L487), [`FirmwareController.php`](../../core/src/opnsense/mvc/app/controllers/OPNsense/Core/Api/FirmwareController.php#L556-L599), [`FirmwareController.php`](../../core/src/opnsense/mvc/app/controllers/OPNsense/Core/Api/FirmwareController.php#L640-L723) ghi action và username. | **Sự kiện ít tác động:** trên lab, dùng chức năng Firmware > Audit (kiểm tra bảo mật) và tìm message audit. Không chọn reboot, poweroff, upgrade, install, remove hoặc reinstall. Nếu chỉ thử bộ lọc audit: `logger -p auth.err -t audit 'POC-firmware-test'`; dòng này không chứng minh Firmware API đã chạy action. |
| Người dùng xác thực qua OpenVPN | Script OpenVPN ghi username và auth method khi xác thực; nếu không thành công có thể ghi warning/error. Vì emitter là OpenVPN, đây là log dịch vụ có dữ liệu liên quan người dùng. | Username/auth method xuất hiện trong message `line`, không phải cột riêng. | Ví dụ template: `user '<username>' authenticated using '<authName>'`; khi thất bại: `user '<username>' could not authenticate.` | [`user_pass_verify.php`](../../core/src/opnsense/scripts/openvpn/user_pass_verify.php#L142-L179) tạo các message; filter dịch vụ: [`openvpn.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/local/openvpn.conf#L4-L5). | **Sự kiện thật:** dùng tài khoản test kết nối thành công tới VPN lab để xác minh log có username/auth method. Nếu kiểm tra thất bại, chỉ nhập sai một lần khi đã xác nhận không có chính sách khóa. Nếu không chắc, dùng lệnh thử ở hàng OpenVPN; lệnh đó không phải log xác thực thật. |

## Nguồn/đích khác cần phân biệt

- **Remote syslog là đích gửi, không phải nguồn log:** cấu hình destination có
  thể lọc theo program, level hoặc facility rồi gửi qua UDP/TCP/TLS. Vì vậy đây
  là forwarding ra ngoài luồng local, không phải một loại event mới.
  **Ref:** [`syslog-ng-destinations.conf`](../../core/src/opnsense/service/templates/OPNsense/Syslog/syslog-ng-destinations.conf#L8-L37).
- **Syslog statistics không phải nội dung log event:** service API có action
  gọi `syslog-ng-ctl stats`, và GUI có tab Statistics riêng. Dữ liệu đó là số
  liệu trạng thái/đếm của syslog-ng, không phải record log được parse như
  Diagnostics. Cú pháp lệnh nằm trong manual upstream
  [`syslog-ng-ctl(1)`](https://github.com/syslog-ng/syslog-ng/blob/develop/doc/man/local/syslog-ng-ctl.1).
  **Ref:** [`actions_syslog.conf`](../../core/src/opnsense/service/conf/actions.d/actions_syslog.conf#L41-L43),
  [`Syslog index.volt`](../../core/src/opnsense/mvc/app/views/OPNsense/Syslog/index.volt#L45-L46),
  [`Syslog index.volt`](../../core/src/opnsense/mvc/app/views/OPNsense/Syslog/index.volt#L109-L116).
- **File log riêng của dịch vụ có thể tồn tại ngoài luồng local filter:** cần
  kiểm tra runtime/package hoặc file cấu hình dịch vụ tương ứng trước khi kết
  luận toàn bộ log của dịch vụ chỉ nằm ở các target nói trên. Backend có hỗ
  trợ một số đường dẫn legacy.
  **Ref:** [`log_matcher.py`](../../core/src/opnsense/scripts/syslog/log_matcher.py#L138-L154).
