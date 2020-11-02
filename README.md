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
```
### 1.2 Gauge
Gague là metric biểu thị giá trị có thể tăng ,giảm tùy ý. 

Ví dụ, gauge có thể biểu thị nhiệt độ hiện tại hoặc bộ nhớ hiện tại đang sử dụng, CPU
```bash
# HELP node_cpu_frequency_max_hertz Maximum cpu thread frequency in hertz.
# TYPE node_cpu_frequency_max_hertz gauge
node_cpu_frequency_max_hertz{cpu="0"} 3.3e+09
node_cpu_frequency_max_hertz{cpu="1"} 3.3e+09
node_cpu_frequency_max_hertz{cpu="2"} 3.3e+09
node_cpu_frequency_max_hertz{cpu="3"} 3.3e+09
```

### 1.3 Summary
Summary có chức năng giống với hàm histogram_quantile(), tuy nhiên percentiles được tính toán ở phía client. Summary có nguồn gốc từ count và sum counters(Giống như Histogram) và trả về kết quả là giá trị quantile(phân vị)

```bash
# HELP prometheus_engine_query_duration_seconds Query timings
# TYPE prometheus_engine_query_duration_seconds summary
prometheus_engine_query_duration_seconds{slice="inner_eval",quantile="0.5"} 0.000310732
prometheus_engine_query_duration_seconds{slice="inner_eval",quantile="0.9"} 0.000508063
prometheus_engine_query_duration_seconds{slice="inner_eval",quantile="0.99"} 0.000793674
prometheus_engine_query_duration_seconds_sum{slice="inner_eval"} 6.49343868100001
prometheus_engine_query_duration_seconds_count{slice="inner_eval"} 15106
```


### 1.4 Histogram
Prometheus Histogram là một tần số tích lũy.Histogram có nguồn gốc là counter, cái mà đếm các sự kiện đã xảy ra, một bộ đếm tổng giá trị sự kiện, và bộ đếm khác cho mỗi bucket. Buckets đếm có bao nhiêu lần giá trị sự kiện nhỏ hơn hoặc bằng giá trị bucket(bucket value)
```bash
# HELP prometheus_http_request_duration_seconds Histogram of latencies for HTTP requests.
# TYPE prometheus_http_request_duration_seconds histogram
prometheus_http_request_duration_seconds_bucket{handler="/",le="0.1"} 2
prometheus_http_request_duration_seconds_bucket{handler="/",le="0.2"} 2
prometheus_http_request_duration_seconds_bucket{handler="/",le="0.4"} 2
prometheus_http_request_duration_seconds_bucket{handler="/",le="1"} 2
prometheus_http_request_duration_seconds_bucket{handler="/",le="3"} 2
prometheus_http_request_duration_seconds_bucket{handler="/",le="8"} 2
prometheus_http_request_duration_seconds_bucket{handler="/",le="20"} 2
prometheus_http_request_duration_seconds_bucket{handler="/",le="60"} 2
prometheus_http_request_duration_seconds_bucket{handler="/",le="120"} 2
prometheus_http_request_duration_seconds_bucket{handler="/",le="+Inf"} 2
prometheus_http_request_duration_seconds_sum{handler="/"} 0.016814989
prometheus_http_request_duration_seconds_count{handler="/"} 2
```


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