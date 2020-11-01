# Các câu hỏi
- Nói rõ lại về 4 loại metrics: Gauge, counter, histogram, summary
- Tìm hiểu về postgreSQL exporter: Cách exporter lấy metrics
- Tìm hiểu về các mode trong CPU

## 1. Nói rõ lại về 4 loại metrics: Gauge, counter, histogram, summary
### 1.1 Counter
Counter là metric biểu thị giá trị chỉ tăng hoặc reset về 0
### 1.2 Gauge
Gague là metric biểu thị giá trị có thể tăng ,giảm, hoặc 0
### 1.3 Summary
Summary là một metric tổng hợp từ các loại metrics khác. Summary metrics bao gồm 2 counters, và tùy tình huống có thể sử dụng gauges. Summary metrics được sử dụng để theo dõi kích cỡ của các sự kiện, như khoảng thời gian chúng sử dụng.

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

metric có đuôi là **_sum** và **_count** là metrics counter.
### 1.4 Histogram
Histogram có nhiều điểm giống với metrics summary.Histogram là sự pha trộn giữa 

## 2. Node exporter

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
- 

