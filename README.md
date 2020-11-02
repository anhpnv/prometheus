# Các câu hỏi
- Nói rõ lại về 4 loại metrics: Gauge, counter, histogram, summary
- Tìm hiểu về postgreSQL exporter: Cách exporter lấy metrics
- Tìm hiểu về các mode trong CPU

## 1. Nói rõ lại về 4 loại metrics: Gauge, counter, histogram, summary
### 1.1 Counter
Counter biểu thị giá trị chỉ tăng hoặc reset về 0. Ví dụ, ta có thể dùng counter để thể hiện số lượng request, các lỗi, các nhiệm vụ đã hoàn thành. Counter không được sử dụng để thể hiện giá trị có thể giảm.
### 1.2 Gauge
Gague là metric biểu thị giá trị có thể tăng ,giảm tùy ý. Ví dụ, gauge có thể biểu thị nhiệt độ hiện tại hoặc bộ nhớ hiện tại đang sử dụng
### 1.3 Summary
Summary là một metric tổng hợp từ các loại metrics khác. Summary metrics bao gồm 2 counters, và tùy tình huống có thể sử dụng gauges. Summary metrics được sử dụng để theo dõi kích cỡ của các sự kiện, như khoảng thời gian chúng sử dụng, thông qua phương thức **observe**.

Dưới đây là ví dụ về metrics summary:

```bash
# HELP prometheus_rule_evaluation_duration_seconds The duration for a rule to execute.
# TYPE prometheus_rule_evaluation_duration_seconds summary
prometheus_rule_evaluation_duration_seconds{quantile="0.5"} 6.4853e-05
prometheus_rule_evaluation_duration_seconds{quantile="0.9"} 0.00010102
prometheus_rule_evaluation_duration_seconds{quantile="0.99"} 0.000177367
prometheus_rule_evaluation_duration_seconds_sum 1.623860968846092e+06
prometheus_rule_evaluation_duration_seconds_count 1.112293682e+09
```

metric có đuôi là **_sum** và **_count** là counter. **_count** tăng lên mỗi khi sử dụng **observe**, và **_sum** tăng lên bởi giá trị của observation.


### 1.4 Histogram
Histogram 
Histogram có nhiều điểm giống với metrics summary.Histogram là sự pha trộn của nhiều counters khác nhau. Ví dụ về 1 loại histogram metrics:

```bash
# HELP prometheus_http_request_duration_seconds Histogram of latencies for HTTP requests.
# TYPE prometheus_http_request_duration_seconds histogram
prometheus_http_request_duration_seconds_bucket{handler="/",le="0.1"} 25547
prometheus_http_request_duration_seconds_bucket{handler="/",le="0.2"} 26688
prometheus_http_request_duration_seconds_bucket{handler="/",le="0.4"} 27760
prometheus_http_request_duration_seconds_bucket{handler="/",le="1"} 28641
prometheus_http_request_duration_seconds_bucket{handler="/",le="3"} 28782
prometheus_http_request_duration_seconds_bucket{handler="/",le="8"} 28844
prometheus_http_request_duration_seconds_bucket{handler="/",le="20"} 28855
prometheus_http_request_duration_seconds_bucket{handler="/",le="60"} 28860
prometheus_http_request_duration_seconds_bucket{handler="/",le="120"} 28860
prometheus_http_request_duration_seconds_bucket{handler="/",le="+Inf"} 28860
prometheus_http_request_duration_seconds_sum{handler="/"} 1863.80491025699
prometheus_http_request_duration_seconds_count{handler="/"} 28860
```
**_sum** và **_count** cơ chế giống với metrics summary
Điểm nổi bật của loại metrics này chính là các metrics có đuôi "_bucket", nó thể hiện phần histogram thực tế của histogram.Cụ thể hơn, chúng là counter, **le** được gọi là less than(nhỏ hơn hoặc bằng). Vì vậy 26688 requests trong 200ms, 27760 requests trong 400ms và tổng là 28860
### 1.5 Làm thế nào để biết được metrics thuộc loại nào?


Thông thường, các thông tin về cách sử dụng cũng như loại metrics được thể hiện qua url của nó: 127.0.0.1:9090


```bash
# HELP net_conntrack_dialer_conn_attempted_total Total number of connections attempted by the given dialer a given name.
# TYPE net_conntrack_dialer_conn_attempted_total counter
net_conntrack_dialer_conn_attempted_total{dialer_name="alertmanager"} 1
net_conntrack_dialer_conn_attempted_total{dialer_name="cadvisor"} 1
net_conntrack_dialer_conn_attempted_total{dialer_name="default"} 0
net_conntrack_dialer_conn_attempted_total{dialer_name="nodeexporter"} 1
net_conntrack_dialer_conn_attempted_total{dialer_name="prometheus"} 1
net_conntrack_dialer_conn_attempted_total{dialer_name="pushgateway"} 1
# HELP net_conntrack_dialer_conn_closed_total Total number of connections closed which originated from the dialer of a given name.
# TYPE net_conntrack_dialer_conn_closed_total counter
net_conntrack_dialer_conn_closed_total{dialer_name="alertmanager"} 0
net_conntrack_dialer_conn_closed_total{dialer_name="cadvisor"} 0
net_conntrack_dialer_conn_closed_total{dialer_name="default"} 0
net_conntrack_dialer_conn_closed_total{dialer_name="nodeexporter"} 0
net_conntrack_dialer_conn_closed_total{dialer_name="prometheus"} 0
net_conntrack_dialer_conn_closed_total{dialer_name="pushgateway"} 0
```

Trong đó: 
- #HELP sẽ dùng để biết thông tin metrics đang làm gì, - 
- #TYPE sẽ chỉ loại metrics của metrics đó, ví dụ metrics net_conntrack_dialer_conn_attempted_total là counter

Ngoài ra, có một số cách phân biệt cơ bản để rút ra dưới đây:
- Các metrics có đuôi là total, count, sum thường là counter
- Các metrics có đuôi là bucket sẽ là histogram


## 2. PostgreSQL exporter
PostgreSQL exporter là exporter sử dụng để khai thác thông tin từ database PostgreSQL.

### Cách cài đặt
Cách cài đặt postgres_exporter:

```bash
$ go get github.com/wrouesnel/postgres_exporter
$ cd ${GOPATH-$HOME/go}/src/github.com/wrouesnel/postgres_exporter
$ go run mage.go binary
$ export DATA_SOURCE_NAME="postgresql://login:password@hostname:port/dbname"
$ ./postgres_exporter <flags>
```
Với <flags> có thể là:

- help: dùng để xem các tùy chọn flag khác
- web.listen-address: port địa chỉ của exporter, mặc định là ```:9187```
- web.telemetry-path: Đường dẫn expose metrics, mặc định là ```metrics```
- auto-discover-databases: Tự động discovery databases trên server
- log.level: Cài đặt các mức độ log: debug, info, warn, error, fatal 

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