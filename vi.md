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


Setting [innodb_autoinc_lock_mode][8] =2 (interleaved mode) can remove the need for table-level AUTO-INC lock (and can increase performance when multi-row insert statements are used to insert values into tables with auto_increment primary key). This requires binlog_format=ROW  or MIXED  (and ROW is the default in MySQL 5.7).

_innodb_io_capacity _and_ innodb_io_capacity_max_

This is a more advanced tuning, and only make sense when you are performing a lot of writes all the time (it does not apply to reads, i.e. SELECTs). If you really need to tune it, the best method is knowing how many IOPS the system can do. For example, if the server has one SSD drive, we can set innodb_io_capacity_max=6000 and innodb_io_capacity=3000 (50% of the max). It is a good idea to run the sysbench or any other benchmark tool to benchmark the disk throughput.

But do we need to worry about this setting? Look at the graph of buffer pool's "[dirty pages][9]":

![screen-shot-2016-10-03-at-7-19-47-pm][10]

In this case, the total amount of dirty pages is high, and it looks like InnoDB can't keep up with flushing them. If we have a fast disk subsystem (i.e., SSD), we might benefit from increasing innodb_io_capacity and innodb_io_capacity_max.

_**Conclusion or TL;DR version**_

The new MySQL 5.7 defaults are much better for general purpose workloads. At the same time, we still need to configure InnoDB variables to take advantages of the amount of RAM on the box. After installation, follow these steps:

1. Add InnoDB variables to my.cnf (as described above) and restart MySQL
2. Install a monitoring system, (e.g., Percona Monitoring and Management platform)
3. Look at the graphs and determine if MySQL needs to be tuned further




