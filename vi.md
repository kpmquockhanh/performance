# MySQL 5.7 Hiệu chỉnh ngay sau khi cài đặt

Bài blog này cập nhật [Blog của Stephane Combaudon về điều chỉnh hiệu suất MySQL][1], và bao gồm cả hiệu suất của MySQL 5.7 ngay sau khi cài đặt.

Một vài năm trước, Stephane Combaudon đã viết một bài blog đăng về [10 cài đặt điều chỉnh hiệu suất của MySQL sau khi cài đặt][1] mà bao gồm các phiên bản cũ hơn của MySQL: 5.1, 5.5 và 5.6. Trong bài viết này, tôi sẽ chú tâm vào những gì cần điều chỉnh trong MySQL 5.7 (cụ thể là trong InnoDB).

Tin tốt là MySQL 5.7 có những giá trị mặc định cho việc hiệu chỉnh tốt hơn. Morgan Tocker đã tạo ra một [trang với tập hợp các tính năng hoàn thiện trong MySQL 5.7][2], và là một điểm tham khảo tuyệt vời. Ví dụ, những biến dưới đây được cài đặt _mặc định_:

Trong MySQL 5.7, có 4 biến thực sự quan trọng mà cần phải được thay đổi. Tuy nhiên, cũng có những biến khác của InnoDB và của MySQL toàn cục mà cần được điều chỉnh theo khối lượng công việc và ổ cứng.

Để bắt đầu, thêm các cài đặt sau vào my.cnf dưới phần [mysqld]. Bạn sẽ cần khởi động lại MySQL:

[mysqld]

# Các biến khác ở đây

innodb_buffer_pool_size = 1G # (adjust value here, 50%-70% of total RAM)

innodb_log_file_size = 256M

innodb_flush_log_at_trx_commit = 1 # may change to 2 or 0

innodb_flush_method = O_DIRECT

 | 

Mô tả:

| ----- |
| **Biến** |  **Giá trị** |  
| innodb_buffer_pool_size |  Bắt đầu với khoảng từ 50 - 70% dung lượng RAM. Không cần phải lớn hơn kích cỡ CSDL |  
| innodb_flush_log_at_trx_commit | 

* 1   (Mặc định)
* 0/2 (hiệu suất cao hơn, độ tin cậy thấp hơn)
 |  
| innodb_log_file_size |  128M – 2G (không cần phải lớn hơn vùng đệm) |  
| innodb_flush_method |  O_DIRECT (tránh trùng vùng nhớ đệm) | 

 
_**Tiếp theo là gì?**_

Tất cả những thứ trên đều tối cho việc khởi đầu cho bất cứ cài đặt mới nào. Có rất nhiều các biến khác mà có thể tăng hiệu suất của MySQL cho một vài công việc. Thông thường, tôi sẽ thiết lập một công cụ theo dõi / vẽ đồ thị MySQL (Ví dụ, ) [Nền tảng quản lý và giám sát Percona][3] và sau đó kiểm tra bảng điều khiển MySQL để xem những điều chỉnh khác.

_**Chúng ta có thể chỉnh những gì dựa vào cơ sở đồ thị?**_

_InnoDB buffer pool size_. Hãy xem các đồ thị sau:

![MySQL 5.7 Performance Tuning][4]

![MySQL 5.7 Performance Tuning][5]

Như ta có thể thấy, chúng ta hoàn toàn có thể kiếm lợi từ việc tăng bộ nhớ đệm một chú sấp sỉ 10G, do chúng ta có sẵn RAM và rất nhiều các trang khác miễn phí nhỏ so với tổng số vùng đệm.


_InnoDB log file size._ Hãy xem các đồ thị sau:

![MySQL 5.7 Performance Tuning][6]

Như chúng ta có thể thấy ở đây, InnoDB thương viết khoảng 2.26GB dữ liệu mỗi giờ, vượt quá tổng kích thước được ghi vào file(2G). Chúng ta có thể tăng biến innodb_log_file_size lên và khởi động lại MySQL. Ngoài ra, hãy sử dụng "hiển thị trạng thái InnoDB" để [tính toán một file log tốt][7]

_**Các biến khác**_

Có  nhiều các biến khác của InnoDB cần được điều chỉnh:

_innodb_autoinc_lock_mode_

Thiết lập [innodb_autoinc_lock_mode][8] =2 (chế độ xen kẽ) có thể nhu cầu cho khóa AUTO-INC cấp bảng (và có thể tăng hiệu suất khi nhiều dòng lệnh insert được sử dụng để ghi giá trị vào các bảng với các khóa chính auto_increment). Điều này yêu cầu binlog_format=ROW hoặc MIXED (và ROW là mặc định trong MySQL 5.7).

_innodb_io_capacity _and_ innodb_io_capacity_max_

Đây là một điều chỉnh nâng cao hơn và chỉ có ý nghĩa khi bạn đang thực hiện rất nhiều công việc ghi trong mọi thời điểm (Nó không áp dụng cho việc đọc, i.e SELECTs). Nếu bạn thực sự cần điều chỉnh nó, các tốt nhất là hiểu về số lượng IOPS mà hệ thống có thể thực hiện. Ví dụ, nếu server có một ổ SSD, chúng ta có thể thiết lập innodb_io_capacity_max=6000 và innodb_io_capacity=3000 (50% trong số lớn nhất). Đây là một ý tưởng tốt để chạy Đó là một ý tưởng tốt để chạy sysbench hoặc bất kỳ công cụ benchmark khác nào để chuẩn hóa thông lượng đĩa.

Nhưng liệu rằng chúng ta có cần lo lắng quá nhiều về việc cài đặt? Nhìn vào đồ thị vùng đệm của "[dirty pages][9]":

![screen-shot-2016-10-03-at-7-19-47-pm][10]

Trong trường hợp này, tổng số lượng các trang "dirty" là rất cao, và trông có vẻ là InnoDB không thể theo kịp với việc xả chúng. Nếu chúng ta có hệ thống ổ đĩa con nhanh (i.e., SSD), chúng ta có thể hưởng lợi từ việc tăng innodb_io_capacity và innodb_io_capacity_max.

_**Kết luận hoặc TL;DR phiên bản**_

Măc định của MySQL 5.7 mới tốt hơn đối với khối lượng mục đích công việc chung. Trong cùng một thời điểm, chúng tôi vẫn cần cấu hình các biến của InnoDB để tận dụng số lượng RAM trong đó. Sau khi cài đặt, hãy làm theo các bước sau:

1. Thêm các biến InnoDB vào my.cnf (như mô tả ở trên) và khởi động lại MySQL
2. Cài đặt hệ thống giám sát, (ví dụ: nền tảng giám sát và quản lý Percona)
3. Nhìn vào biểu đồ và xác định xem liệu MySQL có cần được điều chỉnh thêm hay không.



