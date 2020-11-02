# Các câu hỏi

- Nói rõ lại về 4 loại metrics: Gauge, counter, histogram, summary
- Tìm hiểu về postgreSQL exporter: Cách exporter lấy metrics
- Tìm hiểu về các mode trong CPU

## 1. Nói rõ lại về 4 loại metrics: Gauge, counter, histogram, summary

### 1.1 Counter
Counter biểu thị giá trị chỉ tăng hoặc reset về 0. 

Ví dụ, ta có thể dùng counter để thể hiện số lượng request, các lỗi, các nhiệm vụ đã hoàn thành, tổng thời gian tiêu tốn CPU.

```bash
# HELP node_cpu_seconds_total Seconds the cpus spent in each mode.
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="idle"} 21570.98
node_cpu_seconds_total{cpu="0",mode="iowait"} 679.52
node_cpu_seconds_total{cpu="0",mode="irq"} 0
node_cpu_seconds_total{cpu="0",mode="nice"} 25.28
node_cpu_seconds_total{cpu="0",mode="softirq"} 293.36
# HELP node_disk_discard_time_seconds_total This is the total number of seconds spent by all discards.
# TYPE node_disk_discard_time_seconds_total counter
node_disk_discard_time_seconds_total{device="sda"} 0
# HELP node_disk_discarded_sectors_total The total number of sectors discarded successfully.
# TYPE node_disk_discarded_sectors_total counter
node_disk_discarded_sectors_total{device="sda"} 0
```
### 1.2 Gauge

Gague biểu thị giá trị có thể tăng ,giảm tùy ý. 

Ví dụ, node_filesystem_avail_bytes là một gauge metrics, nó lấy thông tin từ câu lệnh **df**
```bash

# HELP node_filesystem_avail_bytes Filesystem space available to non-root users in bytes.
# TYPE node_filesystem_avail_bytes gauge
node_filesystem_avail_bytes{device="/dev/sda1",fstype="vfat",mountpoint="/boot/efi"} 5.35801856e+08
node_filesystem_avail_bytes{device="/dev/sda5",fstype="ext4",mountpoint="/"} 8.9824536576e+11
node_filesystem_avail_bytes{device="tmpfs",fstype="tmpfs",mountpoint="/run"} 3.99982592e+08
node_filesystem_avail_bytes{device="tmpfs",fstype="tmpfs",mountpoint="/run/lock"} 5.238784e+06
node_filesystem_avail_bytes{device="tmpfs",fstype="tmpfs",mountpoint="/run/snapd/ns"} 3.99982592e+08
node_filesystem_avail_bytes{device="tmpfs",fstype="tmpfs",mountpoint="/run/user/1000"} 4.02546688e+08
```

### 1.3 Histogram
Prometheus Histogram là một tần số tích lũy.Histogram có nguồn gốc là counter, cái mà đếm các sự kiện đã xảy ra, một bộ đếm tổng giá trị sự kiện, và bộ đếm khác cho mỗi bucket. Buckets đếm có bao nhiêu lần giá trị sự kiện nhỏ hơn hoặc bằng giá trị bucket(bucket value)

* Số lượng bucket và các giá trị bucket được quy định trước trong histogram:
  * Format của bucket: < basename >_bucket{le="< bound_value >"}
  * Trong đó < basename > là tên cơ sở của metrics và < bound_value > là ngưỡng quy định của 1 bucket
  * Mỗi histogram có một bucket < basename >_bucket{le="+Inf"} và giá trị đó bằng giá trị < basename >_count
* Histogram sử dụng các counter:
  * < basename >_count là counter đếm số lượng sự kiện xảy ra
  * < basename >_sum là counter tổng giá trị các sự kiện


```bash
# HELP http_request_duration_seconds request duration histogram
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.5"} 0
http_request_duration_seconds_bucket{le="1"} 1
http_request_duration_seconds_bucket{le="2"} 2
http_request_duration_seconds_bucket{le="3"} 3
http_request_duration_seconds_bucket{le="5"} 3
http_request_duration_seconds_bucket{le="+Inf"} 3
http_request_duration_seconds_sum 6
http_request_duration_seconds_count 3
```
### 1.4 Summary
Summary có chức năng giống với hàm histogram_quantile(), tuy nhiên percentiles được tính toán ở phía client. Summary có nguồn gốc từ count và sum counters(Giống như Histogram) và trả về kết quả là giá trị quantile(phân vị)

Summary được tạo ra từ 2 loại metrics là counter và gauge:
* < basename >_count là counter đếm số lượng sự kiện xảy ra
* < basename >_sum là counter tổng giá trị các sự kiện

```bash
# HELP http_request_duration_seconds request duration summary
# TYPE http_request_duration_seconds summary
http_request_duration_seconds{quantile="0.5"} 2
http_request_duration_seconds{quantile="0.9"} 3
http_request_duration_seconds{quantile="0.99"} 3
http_request_duration_seconds_sum 6
http_request_duration_seconds_count 3
```
* http_request_duration_seconds_sum = 1s + 2s + 3s = 6s
* http_request_duration_seconds_count = 3, bởi vì chỉ có 3 request.
* http_request_duration_seconds{quantile="0.5"} = 2, nghĩa là có 50% số request mà thời gian đợi phản hồi dưới 2s.
* http_request_duration_seconds{quantile="0.9"} = 3, nghĩa là có 90% số request mà thời gian đợi phản hồi dưới 3s.
* http_request_duration_seconds{quantile="0.99"} = 3, nghĩa là có 99% số request mà thời gian đợi phản hồi dưới 3s.
## 2. PostgreSQL exporter
PostgreSQL exporter là exporter sử dụng để khai thác thông tin từ database PostgreSQL.

### Setting the Postgres server's data source name

Để expose các metrics, hay discovery thông tin của database, exporter cần có trường DATA_SOURCE_Name chứa địa chỉ để truy cập database:
```bash
DATA_SOURCE_NAME="postgresql://login:password@hostname:port/dbname"
```
Sau đó, expoter discovery database bằng cách sử dụng các câu lệnh truy vấn. Ví dụ các metrics sau được khai báo trong file queries.yaml:

```yaml
pg_replication:
  query: "SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())) as lag"
  master: true
  metrics:
    - lag:
        usage: "GAUGE"
        description: "Replication lag behind master in seconds"

pg_postmaster:
  query: "SELECT pg_postmaster_start_time as start_time_seconds from pg_postmaster_start_time()"
  master: true
  metrics:
    - start_time_seconds:
        usage: "GAUGE"
        description: "Time at which postmaster started"
```
Sẽ được các metrics như sau:

```bash
# HELP pg_postmaster_start_time_seconds Time at which postmaster started
# TYPE pg_postmaster_start_time_seconds gauge
pg_postmaster_start_time_seconds{server="127.0.0.1:5432"} 1.604278992e+09

# HELP pg_replication_lag Replication lag behind master in seconds
# TYPE pg_replication_lag gauge
pg_replication_lag{server="127.0.0.1:5432"} NaN
```

## 3. Tìm hiểu về các mode trong CPU

node_cpu_seconds_total là metric bắt nguồn từ file /proc/stat trong hệ thống, nó thể hiện số lượng thời gian CPU bỏ ra với mỗi chế độ.

Sau đây là các mode trong CPU:

- user: Thời gian bỏ ra trong userland
- system: Thời gian sử dụng trong kernel
- iowait: Thời gian chờ hệ thống I/O
- idle: Thời gian nhàn rỗi của CPU 
- irq&softirq: Thời gian service bị gián đoạn
- guest: Chế độ này được sử dụng khi ta dùng máy ảo VM
- steal: thời gian khác VMs "đánh cắp" từ CPUs của mình
- nice: mode xét độ ưu tiên tiến trình trong CPU, có giá trị từ -20 tới 20, giá trị càng cao thì độ ưu tiên tiến trình trong CPU càng thấp.

## Exceptions

### Percentile

"Số phân vị Pth là một giá trị mà tại đó nhiều nhất có P% số trường hợp quan sát trong tập dữ liệu có giá trị thấp hơn giá trị này và nhiều nhất là (100 – P)% của trường hợp có giá trị lớn hơn giá trị này".

Hay nói cách khác k<sup>th</sup> percentile là giá trị nằm trong một data set mà chia data thành 2 phần:

1. Phần bé hơn chứa k% 
2. Phần lớn hơn(phần còn lại của data) chứa (100 - k)%

Để tính giá trị k<sup>th</sup> percentile ta làm như sau:

1. Chọn dãy số từ nhỏ tới lớn
2. Nhân k% với số lượng dãy số(trong trường hợp này là **n**): index = k * n / 100
3. Nếu index là số không nguyên, thì làm tròn lên số nguyên gần nhất rồi sang bước 4.Nếu index là số nguyên thì sang bước 5
4. Tìm thứ tự thứ **index** trong dãy giá trị đã cho. Giá trị tương ứng trong tập dữ liệu chính là k<sup>th</sup> percentile
5. TÌm thứ tự thứ **index** trong dãy giá trị đã cho. Gía trị tương ứng trong tập dữ liệu là giá trị trung bình của số đó với số tiếp theo liền sau nó trong dãy.


Ví dụ, giả sử ta có một dãy các điểm từ bé tới lớn: 

`43, 54, 56, 61, 62, 66, 68, 69, 69, 70, 71, 72, 77, 78, 79, 85, 87, 88, 89, 93, 95, 96, 98, 99, 99.`

Để tìm ra 90<sup>th</sup> percentile trong dãy(có 25 giá trị), ta nhân 90% * 25 = 22.5(index). Làm tròn số gần nhất với nó là 23.

Đếm từ trái sang phải, ta thấy giá trị thứ 23 là 98. Vậy 98 là 90<sup>th</sup> percentile trong dãy này.

Nếu tìm 20<sup>th</sup> percentile trong dãy(có 25 giá trị), ta nhân 20% * 25 = 5(index). Vậy ta có số thứ 5 trong dãy có giá trị: 62. Vì index = 5 là số nguyên, nên 20<sup>th</sup> percentile là giá trị trung bình của (62 + 66)/2 = 64.

Source: 
- [www.dummies.com/education/math/statistics/how-to-calculate-percentiles-in-statistics](https://www.dummies.com/education/math/statistics/how-to-calculate-percentiles-in-statistics/)