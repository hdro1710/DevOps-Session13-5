# Phản biện kiến trúc: Pull-based (Prometheus) vs Push-based (DataDog, Telegraf, AWS CloudWatch)

> **Bối cảnh:** Trong cuộc họp kiến trúc, một chuyên gia đặt câu hỏi: *"Tại sao dùng Prometheus? Prometheus đi gõ cửa (Pull) từng Service để xin dữ liệu. Với 1000 Microservices liên tục bật tắt (Auto-scaling), Prometheus làm sao biết ai sinh ra mà gõ cửa? Sao không để các service tự Push dữ liệu lên một Server trung tâm?"*
>
> Tài liệu này phân tích hai mô hình và bảo vệ lựa chọn Prometheus một cách có căn cứ, đồng thời thừa nhận rõ những trường hợp Push mới là lựa chọn đúng.

---

## 1. Tóm tắt nhanh (Executive Summary)

| Câu hỏi của chuyên gia | Trả lời ngắn gọn |
|---|---|
| Pull có làm Backend quá tải không? | Ngược lại: **Prometheus tự kiểm soát tốc độ thu thập**, nên không bao giờ bị service "bắn dồn" dữ liệu vào. |
| Auto-scaling thì Prometheus biết service mới ở đâu? | Nhờ **Service Discovery** (Kubernetes API, Consul, Eureka...). Instance mới đăng ký với registry, Prometheus tự phát hiện và tự bắt đầu scrape. |
| Có cần Push không? | **Có**, với job ngắn hạn (batch/cronjob) sống ít hơn một chu kỳ scrape. Giải pháp: **Pushgateway**. |

**Kết luận:** Pull là mô hình mặc định phù hợp cho các service chạy lâu dài (long-running). Push chỉ là ngoại lệ có chủ đích, không phải đối thủ thay thế hoàn toàn.

---

## 2. Hai mô hình hoạt động như thế nào

### 2.1. Pull-based (Prometheus)

```
 ┌──────────────┐   1. Discover targets   ┌─────────────────────┐
 │  Service     │ ◄────────────────────── │  Service Discovery  │
 │  Registry    │                         │ (K8s API / Consul)  │
 └──────────────┘                         └─────────┬───────────┘
                                                    │
 ┌──────────────┐   2. HTTP GET /metrics  ┌─────────▼───────────┐
 │ Service A    │ ◄────────────────────── │                     │
 │ Service B    │ ◄────────────────────── │    Prometheus       │
 │ Service ...N │ ◄────────────────────── │   (scrape loop)     │
 └──────────────┘                         └─────────────────────┘
```

- Mỗi service chỉ cần **expose** một endpoint `/metrics` (dạng text).
- Prometheus **chủ động** gọi HTTP GET theo chu kỳ (ví dụ 15s/lần).
- Danh sách target đến từ Service Discovery, không phải cấu hình tay.

### 2.2. Push-based (DataDog, Telegraf, AWS CloudWatch)

```
 ┌──────────────┐   HTTP/UDP push    ┌─────────────────────┐
 │ Service A    │ ─────────────────► │                     │
 │ Service B    │ ─────────────────► │  Server trung tâm   │
 │ Service ...N │ ─────────────────► │ (DataDog / CW / DB) │
 └──────────────┘                    └─────────────────────┘
   (agent/SDK nhúng trong service)
```

- Mỗi service (hoặc agent đi kèm) **chủ động gửi** metric lên server trung tâm.
- Server **bị động** tiếp nhận, nên tốc độ dữ liệu đến do phía client quyết định.

---

## 3. Phản biện 1: Vì sao Pull giúp Backend ít bị áp lực?

### 3.1. Kiểm soát lưu lượng (Backpressure tự nhiên)

- **Push:** Khi 1000 service cùng push, số lượng ghi/giây do **client quyết định**. Nếu có sự cố (ví dụ deploy lỗi, vòng lặp log metric dày đặc, retry storm), toàn bộ client có thể **đồng loạt bắn dữ liệu** làm server trung tâm quá tải đúng lúc hệ thống đang cần quan sát nhất.
- **Pull:** Prometheus quyết định **khi nào, bao nhiêu, và với tần suất nào** nó scrape. Tải lên Prometheus là **có thể dự đoán**: `số target × số series × tần suất scrape`. Service có bất thường cũng không thể "ép" dữ liệu vào Prometheus.

### 3.2. Server trung tâm chết không kéo theo service chết

- **Push:** Nếu server nhận bị chậm/chết, client phải xử lý: retry, buffer, timeout, drop dữ liệu. Nếu cấu hình không khéo, **thư viện push có thể gây chặn luồng nghiệp vụ** (thread block, memory phình do buffer).
- **Pull:** Service chỉ phục vụ `/metrics` như một endpoint thông thường. Prometheus chết thì service **hoàn toàn không bị ảnh hưởng**, không có buffer, không có retry, không có logic gửi nào trong service.

### 3.3. Phát hiện service chết (Health check miễn phí)

- Mỗi lần scrape sinh ra metric tổng hợp `up{job="...", instance="..."}`:
  - `up == 1`: scrape thành công.
  - `up == 0`: scrape thất bại, service chết hoặc không với tới được.
- **Push:** Khi một service ngừng gửi dữ liệu, server chỉ thấy **sự im lặng**. Không phân biệt được "service chết", "mạng đứt" hay "service không có gì để báo". Phải tự thiết kế cơ chế heartbeat/"absent data alert".

### 3.4. Đơn giản hóa phía service và dễ debug

- Muốn kiểm tra metric của một service? Chỉ cần `curl http://service:8080/metrics` hoặc mở trình duyệt. Không cần dựng server giả lập để nhận push.
- Service không cần biết địa chỉ server monitoring, credentials hay API key. **Giảm coupling** giữa ứng dụng và hệ thống giám sát.
- Dev chạy thử trên máy cá nhân dễ dàng, không cần cấu hình đích đến.

### 3.5. Tránh khuếch đại sự cố (Retry storm)

Khi server push bị chậm, hàng nghìn client đồng loạt retry tạo ra **thundering herd**, làm tình trạng tệ hơn. Pull không có hiện tượng này, vì chỉ có **một** thực thể (Prometheus) quyết định nhịp gọi.

---

## 4. Phản biện 2: Bài toán Auto-scaling, Prometheus biết ai sinh ra mà "gõ cửa"?

Đây là câu hỏi cốt lõi của chuyên gia, và là điểm hiểu lầm phổ biến: **Prometheus không cần ai báo, vì nó hỏi "sổ danh bạ" (Service Discovery) chứ không dò từng địa chỉ.**

### 4.1. Cơ chế

```
 Service mới khởi động
        │
        ▼
 Đăng ký với registry
 (K8s tự tạo Pod / Eureka / Consul register)
        │
        ▼
 Prometheus watch/poll registry  ──►  Cập nhật danh sách target tự động
        │
        ▼
 Scrape /metrics của target mới ở chu kỳ kế tiếp
```

Khi service **tắt**, registry xóa instance, Prometheus **tự gỡ target** khỏi danh sách. Hoàn toàn không cần sửa file cấu hình hay restart Prometheus.

### 4.2. Các nguồn Service Discovery phổ biến

| Hạ tầng | Cấu hình Prometheus | Ghi chú |
|---|---|---|
| **Kubernetes** | `kubernetes_sd_configs` (role: `pod`, `service`, `endpoints`, `node`) | Watch trực tiếp **Kubernetes API Server**; Pod mới sinh ra là có mặt trong danh sách gần như tức thì. Dùng annotation/label để chọn Pod cần scrape. |
| **Consul** | `consul_sd_configs` | Service tự register với Consul; Prometheus truy vấn catalog. |
| **Eureka** (Spring Cloud) | `eureka_sd_configs` | Phù hợp hệ sinh thái Spring Boot/Netflix OSS. |
| **AWS EC2** | `ec2_sd_configs` | Dò theo tag/region của EC2 instance. |
| **DNS** | `dns_sd_configs` | Dùng bản ghi SRV/A. |
| **File-based** | `file_sd_configs` | Công cụ bên ngoài ghi file JSON/YAML, Prometheus tự reload. |

### 4.3. Ví dụ cấu hình (Kubernetes)

```yaml
scrape_configs:
  - job_name: 'microservices'
    scrape_interval: 15s
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Chỉ scrape Pod có annotation prometheus.io/scrape: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      # Cho phép Pod tự khai báo path & port của /metrics
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      # Gắn nhãn hữu ích để truy vấn về sau
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
```

### 4.4. Ví dụ cho Spring Boot (khớp với stack Java)

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,prometheus
  metrics:
    tags:
      application: ${spring.application.name}
```

```xml
<!-- pom.xml -->
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Service chỉ cần expose `/actuator/prometheus`, còn việc "được tìm thấy" do Service Discovery lo.

### 4.5. Hệ quả khi scale lên 1000 service

- **Không có thao tác thủ công**: scale up/down không cần ai cập nhật Prometheus.
- **Label tự động** (`namespace`, `pod`, `app`, `version`) giúp query tổng hợp theo nhóm dù instance sinh/tử liên tục.
- **Quy mô:** một Prometheus instance xử lý hàng triệu series; khi vượt ngưỡng, áp dụng **sharding** (phân vùng theo `hashmod`), **federation**, hoặc các giải pháp mở rộng như **Thanos / Cortex / Mimir / VictoriaMetrics**.

### 4.6. Lưu ý trung thực: Điểm cần cẩn trọng khi auto-scaling

| Vấn đề | Giải thích | Cách giảm thiểu |
|---|---|---|
| **Churn (instance sống rất ngắn)** | Pod sống < 1 chu kỳ scrape có thể không được scrape kịp. | Giảm `scrape_interval` cho nhóm này, hoặc xem xét Push cho trường hợp thực sự ngắn hạn. |
| **Cardinality bùng nổ** | Mỗi Pod mới tạo ra nhãn `pod` mới, series mới làm tăng bộ nhớ. | Hạn chế label có giá trị không giới hạn; tổng hợp theo `deployment`/`app` bằng recording rules. |
| **Độ trễ phát hiện** | Có độ trễ nhỏ giữa lúc Pod sinh ra và lúc được scrape lần đầu. | Chấp nhận được với mục đích giám sát; không dùng metrics để làm logic nghiệp vụ thời gian thực. |

---

## 5. Phản biện 3: Khi nào BẮT BUỘC dùng Push (Pushgateway)?

### 5.1. Vấn đề của Batch Job / CronJob

Một cronjob chạy 5 giây rồi thoát. Prometheus scrape mỗi 15 giây:

```
Thời gian:    0s    5s    10s   15s   30s
Job:         [====]  tắt
Scrape:                          ▲ (scrape lúc 15s)
                                 └─ Target đã biến mất, không ai trả lời!
```

Dữ liệu của job (số bản ghi xử lý, thời gian chạy, thành công/thất bại) **mất trắng** vì không còn tiến trình nào để scrape.

### 5.2. Giải pháp: Pushgateway

```
 ┌────────────┐  push khi kết thúc   ┌──────────────┐   scrape bình thường   ┌────────────┐
 │ Batch Job  │ ───────────────────► │ Pushgateway  │ ◄───────────────────── │ Prometheus │
 │ (sống ngắn)│                      │ (giữ metric) │                        └────────────┘
 └────────────┘                      └──────────────┘
```

- Job **push** metric cuối cùng lên Pushgateway **trước khi thoát**.
- Pushgateway **lưu giữ** các metric đó.
- Prometheus **vẫn Pull** từ Pushgateway như bình thường. **Mô hình tổng thể vẫn là Pull.** Pushgateway chỉ đóng vai trò "hộp thư" trung gian.

### 5.3. Ví dụ push từ shell script

```bash
# Gửi metric lúc kết thúc job
cat <<EOF | curl --data-binary @- http://pushgateway:9091/metrics/job/nightly_backup/instance/db01
# TYPE backup_duration_seconds gauge
backup_duration_seconds 42.7
# TYPE backup_last_success_timestamp gauge
backup_last_success_timestamp $(date +%s)
# TYPE backup_records_processed gauge
backup_records_processed 150000
EOF
```

Cảnh báo "job không chạy" có thể viết bằng PromQL:

```promql
time() - backup_last_success_timestamp > 86400   # quá 24h chưa chạy thành công
```

### 5.4. Các trường hợp hợp lệ khác cho Push

| Trường hợp | Lý do |
|---|---|
| **Batch job / CronJob ngắn hạn** | Sống ngắn hơn chu kỳ scrape (trường hợp chính đáng nhất). |
| **Service sau NAT/Firewall** | Prometheus không thể kết nối vào target; target chủ động đi ra ngoài thì được. |
| **Môi trường Serverless (Lambda, Cloud Functions)** | Không có endpoint `/metrics` ổn định, vòng đời cực ngắn. |
| **Edge / IoT, mạng chập chờn** | Thiết bị gửi khi có kết nối. |

### 5.5. Cảnh báo: Pushgateway KHÔNG phải để thay thế Pull

Tài liệu chính thức của Prometheus khuyến cáo chỉ dùng Pushgateway cho **service-level batch jobs**. Lý do:

1. **Single point of failure & bottleneck**: tất cả metric đi qua một điểm.
2. **Mất khả năng `up`**: Prometheus chỉ thấy Pushgateway "sống", không biết từng job còn sống hay không.
3. **Metric không tự hết hạn**: dữ liệu push lên sẽ **nằm đó mãi** (stale) cho đến khi bị xóa thủ công/qua API, dễ gây báo cáo sai (thấy giá trị cũ tưởng là mới). Nên luôn kèm metric `*_last_success_timestamp`.
4. **Không scale tốt** nếu dùng để gom metric của hàng nghìn instance long-running.

> **Nguyên tắc:** Dùng Pushgateway cho *job*, không dùng cho *instance*.

---

## 6. Bảng so sánh Điểm mạnh và Điểm yếu

### 6.1. Pull-based (Prometheus)

| Điểm mạnh | Điểm yếu |
|---|---|
| Server **kiểm soát tải và tần suất**, không bị "bắn dồn" | Khó với **job sống ngắn** (phải dùng Pushgateway) |
| Service **không phụ thuộc** vào hệ thống monitoring (monitoring chết, service vẫn ổn) | Cần **kết nối mạng một chiều** từ Prometheus tới target (khó với NAT/firewall/đa mạng) |
| Phát hiện service chết **tự động** qua metric `up` | Cần **Service Discovery** được cấu hình đúng |
| **Dễ debug**: `curl /metrics` là thấy ngay dữ liệu | Độ phân giải thời gian bị giới hạn bởi `scrape_interval` (không bắt được spike cực ngắn giữa hai lần scrape) |
| Không cần cấu hình đích đến/credentials trong từng service | Scale lớn cần kiến trúc bổ sung (sharding, federation, Thanos/Mimir) |
| Chuẩn hóa định dạng (OpenMetrics), hệ sinh thái exporter phong phú | Mô hình lưu trữ cục bộ (local TSDB) không phải giải pháp lưu dài hạn mặc định |
| Hạ tầng đơn giản, dễ chạy cục bộ để thử nghiệm | Cardinality cao (nhiều label giá trị biến động) gây tốn RAM |

### 6.2. Push-based (DataDog, Telegraf, AWS CloudWatch)

| Điểm mạnh | Điểm yếu |
|---|---|
| Phù hợp **job ngắn hạn**, serverless, thiết bị edge | Server dễ **quá tải** khi nhiều client đồng loạt gửi (đặc biệt lúc sự cố) |
| Hoạt động tốt qua **NAT/firewall** (client chủ động đi ra) | Client phải xử lý **retry/buffer/timeout**; thư viện push có thể ảnh hưởng hiệu năng service |
| Không cần Service Discovery để *tìm* target | **Không phân biệt** được "service chết" và "service không có gì báo" nếu không thiết kế heartbeat |
| Giữ được **mọi sự kiện** nếu client buffer tốt (không bị mất giữa hai lần scrape) | Mỗi service phải **biết địa chỉ và credentials** của server monitoring (coupling cao hơn) |
| Dễ tích hợp SaaS (DataDog/CloudWatch quản lý hộ hạ tầng) | Khó debug cục bộ: phải có endpoint nhận hoặc công cụ giả lập |
| Telegraf/agent thu thập nhiều loại nguồn (host, DB, logs...) | Dễ phát sinh **chi phí** theo khối lượng dữ liệu (đặc biệt SaaS), **vendor lock-in** |

### 6.3. So sánh theo tiêu chí kiến trúc

| Tiêu chí | Pull (Prometheus) | Push (DataDog / Telegraf / CloudWatch) |
|---|---|---|
| Ai quyết định tần suất? | Server | Client |
| Kiểm soát tải ghi | **Tốt** | Kém hơn khi có burst |
| Khi server monitoring chết | Service **không bị ảnh hưởng** | Client phải xử lý buffer/retry |
| Phát hiện target chết | **Tự động (`up`)** | Phải tự thiết kế |
| Job ngắn hạn | Cần Pushgateway | **Phù hợp tự nhiên** |
| Xuyên NAT/Firewall | Khó | **Dễ** |
| Auto-scaling | Cần Service Discovery (đã có sẵn trên K8s/Consul/Eureka) | Không cần discovery |
| Debug thủ công | **Rất dễ** (`curl /metrics`) | Khó hơn |
| Coupling service ↔ monitoring | **Thấp** | Cao hơn |
| Chi phí vận hành | Tự vận hành (hạ tầng + nhân lực) | SaaS: trả theo lượng dữ liệu |

---

## 7. Phản hồi gợi ý cho chuyên gia trong cuộc họp

> *"Câu hỏi rất đúng trọng tâm. Prometheus không biết service mới ở đâu bằng cách đoán, mà **hỏi thẳng nơi nắm danh sách thật**: Kubernetes API (hoặc Eureka/Consul). Pod nào sinh ra, hệ thống đã biết trước cả Prometheus, nên Prometheus chỉ việc đọc danh sách đó và scrape. Pod tắt thì tự bị gỡ khỏi danh sách. Không ai phải sửa cấu hình.*
>
> *Về lý do không để service tự Push: với 1000 service, nếu push thì chính chúng ta trao quyền cho 1000 client quyết định tải đổ vào server monitoring, và đúng lúc hệ thống có sự cố thì monitoring cũng dễ bị quá tải theo. Với Pull, Prometheus nắm nhịp độ, service không hề biết đến sự tồn tại của hệ thống giám sát, và mỗi lần scrape còn cho ta tín hiệu `up` để biết service còn sống hay không.*
>
> *Chúng tôi không phủ nhận Push. Với cronjob chạy vài giây, Prometheus scrape không kịp, nên chúng tôi dùng **Pushgateway** đúng cho nhóm này. Đó là ngoại lệ có chủ đích, và vẫn nằm trong luồng Pull của Prometheus."*

---

## 8. Khuyến nghị kiến trúc cho hệ thống 1000 Microservices

1. **Lõi giám sát: Prometheus (Pull)** cho toàn bộ service chạy lâu dài, kết hợp **Kubernetes Service Discovery**.
2. **Pushgateway** chỉ cho batch job/cronjob; luôn kèm metric `*_last_success_timestamp` và có chính sách dọn dẹp metric cũ.
3. **Mở rộng quy mô:**
   - Sharding Prometheus theo namespace/team (hoặc `hashmod`).
   - Dùng **Thanos / Mimir / VictoriaMetrics** cho lưu trữ dài hạn và truy vấn toàn cục.
   - Dùng `remote_write` khi cần đẩy dữ liệu sang hệ thống tập trung.
4. **Kiểm soát cardinality:** tránh label có giá trị không giới hạn (user id, request id); dùng recording rules để tổng hợp.
5. **Alerting:** Alertmanager với rule trên `up`, độ trễ, tỷ lệ lỗi; thêm alert "job không chạy" dựa trên timestamp.
6. **Cách tiếp cận lai (hybrid) khi cần:** nếu tổ chức đã dùng SaaS như DataDog/CloudWatch cho một phần (ví dụ hạ tầng AWS managed service), hoàn toàn có thể chạy song song thay vì ép dùng một công cụ duy nhất.

---

## 9. Kết luận

- **Pull không "thua" Push**, mà **phù hợp hơn** với đặc thù microservices chạy lâu dài trên nền tảng có Service Discovery: kiểm soát tải tốt, giảm coupling, phát hiện lỗi tự nhiên, dễ debug.
- **Auto-scaling không phải điểm yếu** của Pull, vì Service Discovery giải quyết triệt để việc "ai sinh ra, ai đã tắt".
- **Push vẫn có chỗ đứng chính đáng** với job ngắn hạn, serverless, và môi trường có NAT/firewall. Pushgateway cho phép Prometheus bao phủ các trường hợp này mà không phá vỡ nguyên tắc Pull.
- Quyết định đúng không phải "Pull hay Push", mà là **dùng Pull làm mặc định và Push làm ngoại lệ có chủ đích**.

---

## Phụ lục: Thuật ngữ

| Thuật ngữ | Giải thích |
|---|---|
| **Scrape** | Hành động Prometheus gửi HTTP GET tới `/metrics` của target để thu thập dữ liệu. |
| **Target** | Một endpoint được Prometheus theo dõi. |
| **Service Discovery** | Cơ chế tự động tìm danh sách target từ nguồn như Kubernetes, Consul, Eureka. |
| **Pushgateway** | Thành phần trung gian lưu metric do job ngắn hạn push lên, để Prometheus scrape sau. |
| **`up`** | Metric tự sinh: 1 nếu scrape thành công, 0 nếu thất bại. |
| **Cardinality** | Số lượng chuỗi thời gian duy nhất (tổ hợp tên metric và nhãn). |
| **Backpressure** | Cơ chế để bên nhận điều tiết tốc độ dữ liệu từ bên gửi. |
| **Thundering herd** | Nhiều client đồng loạt hành động (ví dụ retry) cùng lúc gây quá tải. |
