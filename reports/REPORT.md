# Lab 25B — Report

**Họ tên / nhóm:** Phùng Quốc Việt  **MSV:** 2A202602456  **Commit:** `2543dd6`  **Điểm tự động (`make grade`):** 84.0 / 88 (+ 4 điểm dữ liệu ẩn của giảng viên)

---

## 1. Kết quả Game Day (naive vs của tôi)

Dữ liệu thực nghiệm được trích xuất từ `reports/gameday/naive.json` và `reports/gameday/resilient.json`:

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | resilient (của tôi) | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | **PASS** |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | resilient (của tôi) | 1.0000 | 10.920 | 1.041 | 0.007 | 2.119 | 0.0 | **PASS** |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | resilient (của tôi) | 1.0000 | 4.660 | 1.060 | 0.034 | 2.145 | 0.0 | **PASS** |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | resilient (của tôi) | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0.0 | **PASS** |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | None | FAIL |
| G5 | resilient (của tôi) | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | None | **PASS** |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | resilient (của tôi) | 1.0000 | 5.961 | 1.055 | 0.019 | 2.189 | 0.0 | **PASS** |

---

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| `attempt_timeout_s` | `8.0` | Dựa trên **G4** (slow region) và **G6** (provider outage). Ở G4, provider alpha phản hồi chậm thêm 5s (khoảng 6.5s). Timeout 8s vừa đủ cho request thành công mà không bị timeout nhầm, đồng thời ở G6 khi alpha-eu treo, client cắt lỗ sau 8s để failover sang beta trước khi user hết 30s deadline. |
| `max_attempts_per_provider` | `2` | Dựa trên **G2** và **G3**. Giới hạn tối đa 1 lần retry (tổng 2 attempts) trên cùng một provider giúp triệt tiêu nguy cơ retry loop. Ở G3, thông số này giữ `retry_ratio = 0.034 <= 0.10`. |
| `backoff_base_s` / `cap_s` | `0.25` / `4.0` | Dựa trên **G1** và **G2**. Full jitter với base 0.25s giải tán xung đột ban đầu và cap 4.0s giới hạn thời gian chờ tối đa, đảm bảo request vẫn hoàn tất trong vòng 30s deadline của user. |
| `retry_budget_ratio` | `0.1` | Dựa trên **G3** (`retry_ratio = 0.034 <= 0.10`). Giới hạn tỷ lệ retry toàn hệ thống không vượt quá ~10% tổng request ban đầu, ngăn chặn tình trạng khuếch đại tải. |
| `breaker.window_size` / `min_calls` | `20` / `10` | Dựa trên **G2** và **G4**. Window 20 với min 10 mẫu giúp circuit breaker có đủ dữ liệu thống kê để ra quyết định chính xác, tránh trip nhầm vào ban đêm hoặc khi tải thưa. |
| `breaker.failure_rate_threshold` | `0.5` | Dựa trên **G2** và **G3**. Khi tỷ lệ lỗi đạt $\ge 50\%$, provider rõ ràng đang gặp sự cố nghiêm trọng, cần ngắt mạch ngay lập tức sang OPEN để dồn tải sang fallback. |
| `breaker.slow_call_s` | `5.0` | Dựa trên **G4**. Ở G4, alpha không trả lỗi HTTP nhưng latency tăng lên > 6s. Ngưỡng 5.0s giúp phát hiện sự suy thoái hiệu năng và trip breaker, giúp `latency_p95_s` giảm từ 13.764s (naive) xuống 5.562s (resilient). |
| `breaker.open_duration_s` | `10.0` | Dựa trên **G2**. Thời gian mở mạch 10s cho phép provider alpha xả sạch hàng đợi quá tải trước khi client gửi request thăm dò ở HALF-OPEN. Nhờ đó `recovery_s = 0.0s <= 30s`. |
| `breaker.half_open_max_probes` | `2` | Dựa trên **G2**. Chỉ cho phép tối đa 2 request thăm dò đồng thời tại trạng thái HALF-OPEN, tránh hiện tượng thundering herd dội ngược tải làm sập lại provider đang hồi phục. |

### Sơ đồ quyết định của `on_error`:

```
                           [on_error(attempt, err)]
                                      │
                         classify(err) -> ErrorKind
                                      │
         ┌────────────────────────────┼────────────────────────────┐
         ▼                            ▼                            ▼
   CLIENT_ERROR                RATE_LIMITED /             OVERLOADED /
  (400, 401, 422)              QUOTA_EXHAUSTED         SERVER_ERROR / TIMEOUT
         │                            │                            │
   release permit               release permit            breaker.on_failure()
         │                            │                            │
   GiveUp("client_error")      Đặt cooldown:                       │
   (0 retry, 0 failover)       - 429: now + retry_after            │
                               - Quota: now + 300s                 │
                                      │                            │
                               Failover ngay lập tức               │
                               (idx += 1, tries = 0)               │
                                      │                            │
                                      │              tries + 1 < max_attempts
                                      │              AND budget.try_acquire()
                                      │              AND fits deadline?
                                      │                     │            │
                                      │                   (Yes)         (No)
                                      │                     │            │
                                      │               Retry cùng       Failover
                                      │               provider với     (idx += 1,
                                      │               Full Jitter      tries = 0)
                                      │                     │            │
                                      └─────────────────────┴────────────┘
                                                            │
                                                     _route(call, now)
```

---

## 3. Ba thí nghiệm

1. **Retry budget là lớp phòng thủ cuối:**
   Khi cố tình vô hiệu hóa circuit breaker (`breaker.min_calls: 1000000000`) và tăng `max_attempts_per_provider: 5`:
   - *(a) Có Retry Budget (`ratio: 0.1`):* Trên G2, `amplification` được kìm hãm ở mức 1.09, `retry_ratio` chỉ 0.05, hệ thống phục hồi trong 0.0s.
   - *(b) Tắt Retry Budget (`ratio: 100`, max tokens $10^9$):* Trên G2, `amplification` tăng vọt lên 4.82, `retry_ratio` tăng lên 3.82, `recovery_s` bị trễ hơn 45s do lượng retry ồ ạt dội vào provider đang nghẽn.
   *Nhận định:* Kể cả khi Circuit Breaker hoạt động bình thường, việc duy trì cả hai lớp phòng thủ là bắt buộc: Breaker bảo vệ theo từng provider cục bộ, trong khi Retry Budget là "chốt chặn cuối cùng" bảo vệ toàn bộ hệ thống khỏi tình trạng khuếch đại lưu lượng ở cấp độ vĩ mô khi nhiều provider cùng suy thoái.

2. **Thử nghiệm `attempt_timeout_s: 30`:**
   Khi đặt timeout lên 30s và chạy G6 (provider A sập, alpha-eu treo):
   - `availability` tụt từ 1.0000 xuống 0.5120 do request bị kẹt cứng ở alpha-eu trong suốt 30s deadline của user, dẫn đến `give_up_reasons` ghi nhận hàng loạt lỗi deadline.
   - Trái lại, ở G4 (slow region), p95 gần như không đổi (~13s) và availability vẫn 1.0 vì alpha vẫn trả lời trong 6–7s.
   *Lưu ý về Survivorship bias:* Latency p95 chỉ tính trên các lượt *thành công*. Khi hàng loạt lượt bị treo 30s rồi hết hạn bị loại bỏ, p95 có thể trông "đẹp giả tạo" trong khi availability thực tế đã sụp đổ. Timeout đúng phải dựa trên **p99 thời gian phản hồi lúc bình thường cộng dung sai chuyển vùng (failover)**, tuyệt đối không được gán bằng deadline của user.

3. **Bỏ lọc tool calling khỏi `ladder()`:**
   Khi chạy G6 mà không lọc bỏ model thiếu tính năng tool calling, số lượng `incompatible_routes` tăng vọt lên **446 lượt vi phạm**.
   *Góc nhìn Product Manager:* 446 lượt tương đương 446 khách hàng yêu cầu hoàn tiền nhưng hệ thống lại điều phối đến một mô hình không có khả năng kích hoạt nghiệp vụ hoàn tiền. Người dùng nhận được câu trả lời vô nghĩa hoặc báo lỗi hệ thống, gây giảm sút nghiêm trọng chỉ số hài lòng khách hàng (CSAT) và vi phạm cam kết chất lượng dịch vụ (SLA) của doanh nghiệp.

---

## 4. Workflow hoàn tiền

Dữ liệu thực nghiệm trích xuất từ `reports/workflows.json`:

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0 | 81 | 128 | 81 | 0 | 14.967,00 $ | 1.123 |
| yours (của tôi) | 1.0 | 66 | 0 | 0 | 0 | 0,00 $ | 0.303 |

**Phân tích cơ chế:**
- **Vì sao chỉ Idempotency Key là chưa đủ?** Nếu chỉ có Idempotency Key mà không có Checkpoint, mỗi khi worker crash và khởi động lại, worker mới phải chạy lại từ đầu toàn bộ các bước LLM (`classify_request`), dẫn đến `wasted_llm_per_crash` tăng vọt (1.123 ở bản naive) làm bùng nổ chi phí token.
- **Vì sao chỉ Checkpoint là chưa đủ?** Nếu chỉ có Checkpoint mà không có Idempotency Key, trong trường hợp thanh toán thành công nhưng gói tin mạng trả về bị timeout (`ServiceTimeout`), worker sẽ retry bước thanh toán và cổng thanh toán sẽ xử lý như một giao dịch mới, gây trừ tiền kép (`duplicate_refunds`).
- **Hậu quả nếu ghi checkpoint TRƯỚC khi gọi payment:** Nếu checkpoint được ghi vào store trước khi gọi cổng thanh toán, khi worker crash ngay giữa lúc giao dịch đang xử lý hoặc chưa kịp gửi đi, worker mới thức dậy đọc checkpoint thấy bước payment "đã xong" nên bỏ qua $\rightarrow$ **Mất hoàn tiền của khách hàng (`lost refunds`)**. Do đó, bắt buộc phải thực thi step thành công rồi mới được ghi checkpoint.

---

## 5. Alerting & error budget

Dữ liệu thực nghiệm trích xuất từ `reports/slo.json`:

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter (của tôi) | Độ trễ phát hiện |
|---|---|---|---|---:|
| `checkout_bug` | page | page | page | 27 phút |
| `provider_outage` | page | page | page | 4 phút |
| `deploy_blip` | none | **page (False)** | **none** | Không page |
| `slow_leak` | ticket | none (Bỏ sót) | **ticket** | 1337 phút |
| `degraded_evening` | page | page | page | 291 phút |
| `night_spike` | none | **page (False)** | **none** | Không page |
| `small_bump` | none | **page (False)** | **none** | Không page |

**Báo cáo Error Budget 30 ngày:**
- Tổng requests: 25.901.755
- Số lỗi tối đa cho phép (SLO 99.5%): 129.508,8 lỗi
- Số lỗi thực tế ghi nhận: 55.618 lỗi
- Availability đạt được: **99.7853%**
- Ngân sách lỗi đã tiêu thụ: **42.95%**
- Ngân sách lỗi còn lại: **57.05%**
- **Quyết định:** **Không đóng băng release** (`freeze_releases: False`) vì ngân sách lỗi còn hơn 57%.

*Lý giải:* Sự cố `small_bump` (kéo dài 20 phút ở mức 12% lỗi) chỉ làm tiêu hao khoảng ~0.5% ngân sách lỗi của cả tháng. Nó không đe dọa đến mục tiêu SLO 30 ngày, do đó theo nguyên lý SRE, hệ thống không được kích hoạt page để tránh gây mệt mỏi cảnh báo (alert fatigue) cho kỹ sư trực on-call vào lúc 11 giờ trưa.

---

## 6. Postmortem — Băng G2 với client naive (blameless)

*Biểu đồ chuỗi thời gian: `reports/gameday/G2_overload_storm.png` và dữ liệu CSV `reports/gameday/naive_G2_overload_storm_timeseries.csv`.*

![G2 Overload Storm](/reports/gameday/G2_overload_storm.png)

### Tóm tắt sự cố (Executive Summary)
Vào lúc $t=120\text{s}$, provider chính (`alpha`) bị mất 75% năng lực tính toán GPU trong vòng 180 giây. Do client phiên bản v1 (`NaiveClient`) sử dụng chiến lược retry 3 tầng chồng chéo không jitter, không circuit breaker và không kiểm soát retry budget, lưu lượng gửi đến provider đã bị khuếch đại gấp gần 10 lần (lên tới 254 attempts/s), làm nghẽn hoàn toàn hàng đợi dịch vụ. Hệ thống rơi vào trạng thái lỗi siêu bền vững (metastable failure) và không thể tự phục hồi cho đến $t=455\text{s}$, tức 155 giây sau khi provider đã khôi phục 100% tài nguyên phần cứng.

### Ảnh hưởng (Impact)
- **Số lượt user thất bại:** 1.285 / 2.362 lượt người dùng thất bại (availability chỉ đạt **45.60%**).
- **Thời gian gián đoạn:** 335 giây (từ $t=120$ đến $t=455$).
- **Chi phí phát sinh:** Chi phí phục vụ mỗi lượt thành công tăng từ 2.0 lên 4.627 đơn vị (+131%).
- **Tiêu hao Error Budget:** Sự cố làm phát sinh hơn 2.737 lỗi timeout và 3.194 lỗi quá tải, đốt cháy hơn **180%** ngân sách lỗi cho phép của dịch vụ trong tháng nếu đây là tháng vận hành thực tế.

### Timeline sự cố (Giây ảo — trích xuất từ file CSV)

| $t$ (giây) | Sự kiện ghi nhận | Bằng chứng định lượng từ CSV |
|---:|---|---|
| **120** | Provider alpha bị mất 75% GPU. Bắt đầu xuất hiện nghẽn hàng đợi. | `calls_ok` tụt giảm từ 7 req/s xuống 2 req/s; `attempts_alpha` ghi nhận 5. |
| **143** | Bão retry bùng nổ đỉnh điểm. NaiveClient liên tục retry dồn dập vào alpha. | `attempts_alpha` tăng vọt lên **254 attempts/s**; `calls_ok` tụt về **0**. |
| **200** | Hệ thống sụp đổ hoàn toàn. Requests bị timeout và loại bỏ hàng loạt. | `calls_ok = 0`, hàng đợi server kẹt cứng, `attempts_alpha` duy trì 140–180 req/s. |
| **300** | Provider alpha phục hồi 100% GPU, nhưng hệ thống vẫn tê liệt hoàn toàn. | Phần cứng phục hồi nhưng `calls_ok` vẫn bằng **0**; `attempts_alpha` là 115 req/s. |
| **360** | Vòng lặp Metastable failure duy trì trạng thái chết lâm sàng của hệ thống. | 60 giây sau khi có lại GPU, `calls_ok` vẫn bằng **0**; hàng đợi chứa đầy request rác. |
| **455** | Hàng đợi server xả hết request cũ; hệ thống bắt đầu phục hồi phục vụ. | `calls_ok` tăng trở lại lên 4 req/s (tại $t=455$) và đạt 12 req/s (tại $t=461$). |

### Trigger vs Root Cause
- **Trigger (Ngòi nổ kích hoạt):** Provider alpha bị sự cố cơ sở hạ tầng, sụt giảm đột ngột 75% dung lượng GPU xử lý trong 3 phút ($t=120 \rightarrow 300$).
- **Root Cause (Nguyên nhân gốc rễ):** Kiến trúc client v1 thiếu khả năng tự vệ: 3 tầng retry lồng nhau (app $\times$ gateway $\times$ SDK = tối đa 36 lần thử/call), delay cố định không có ngẫu nhiên hóa (Full Jitter), không có Circuit Breaker ngắt mạch và không có Retry Budget giới hạn tỷ lệ retry.
- **Cơ chế Metastable Failure (Vì sao không tự hồi phục tại $t=300$):** Khi client timeout sau 8s, server của provider vẫn tiếp tục xử lý request cũ trong hàng đợi FIFO (công sức xử lý vô ích). Trong khi đó, client lại liên tục nhồi thêm các request retry mới. Đến $t=300$, mặc dù GPU đã khôi phục, hàng đợi của server vẫn chứa đầy hàng nghìn request cũ mà người dùng đã bỏ cuộc từ lâu. Tải retry duy trì liên tục ở mức $\sim 150\text{ req/s}$ vượt xa năng lực xử lý danh định (18 slot $\approx 15\text{ req/s}$), tạo thành một vòng lặp tự duy trì trạng thái quá tải kéo dài thêm 155 giây.

### Điều gì giúp / Điều gì làm tệ hơn
- **Làm tệ hơn:** Delay cố định khiến các retry ùa vào server cùng một thời điểm; client tiếp tục retry vào một endpoint đang trả về mã lỗi quá tải 529; thiếu cơ chế failover sang `alpha-eu` hay `beta`.
- **Giúp ích:** Cơ chế queue timeout (10s) của simulator giúp server tự drop bớt một phần request quá hạn, nếu không hệ thống sẽ bị treo vĩnh viễn.

### Action Items (Biện pháp khắc phục hệ thống)

| # | Hành động cải tiến | Phân loại | Người phụ trách (Owner) | Cách kiểm chứng định lượng |
|---|---|---|---|---|
| 1 | Thay thế `NaiveClient` bằng `ResilientClient`: tích hợp Circuit Breaker trượt và Fallback Ladder | **Mitigate** | Nhóm Core Client (Việt) | Chạy mô phỏng G2: `amplification <= 1.2`, `recovery_s <= 5s` trong CI |
| 2 | Áp dụng Full Jitter Exponential Backoff và Retry Budget (tối đa 10% request) | **Prevent** | Nhóm Reliability (Việt) | Unit test `test_budget_caps_retries` pass và `retry_ratio <= 0.05` trên G2 |
| 3 | Thiết lập cảnh báo Multi-Window Multi-Burn-Rate (Page 14.4x và Page 6x) | **Detect** | Nhóm SRE On-call (Việt) | Kiểm thử trên `data/sli_30d.csv`: 0 false page, phát hiện sự cố $\le 30$ phút |

---

## 7. Một điểm yếu còn lại

**Tình huống:** Khi cả 4 provider triển khai đều đồng loạt bị suy thoái hoặc tăng độ trễ (ví dụ: đứt cáp quang quốc tế hoặc sự cố diện rộng trên toàn bộ các datacenter), Fallback Ladder sẽ duyệt qua toàn bộ các provider trong danh sách (`alpha` $\rightarrow$ `alpha-eu` $\rightarrow$ `beta` $\rightarrow$ `gamma-mini`). Mỗi provider sẽ giữ request tối đa `attempt_timeout_s = 8.0s`. Sau 3 lần failover, tổng thời gian chờ có thể lên tới $8 \times 3 = 24\text{s}$, tiến sát đến deadline 30s của user khiến độ trễ p95 tăng cao và user phải chờ đợi rất lâu trước khi nhận kết quả.

**Giải pháp khắc phục:** 
Áp dụng cơ chế **Hedging Requests (gọi dự phòng song song có điều kiện)** kết hợp **Adaptive Timeout**:
- Thay vì timeout cố định 8s cho mọi bước, timeout của lần thử kế tiếp sẽ được co lại theo công thức $\min(\text{timeout}, \text{deadline} - \text{now} - \text{buffer})$.
- Theo dõi latency EWMA (Exponential Weighted Moving Average) của từng provider trong thời gian thực để chủ động hạ bậc ưu tiên của các provider đang chậm mà không cần chờ circuit breaker trip.

---

## 8. Cache (Part G)

Số liệu thực nghiệm trích xuất từ `reports/cache/resilient.json` và `reports/cache/cached.json` (`make cache`):

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của tôi) | **0.9120** | **0.9645** | **0.635** | **174** | **0** | **6** | **0.927** | **PASS** |

| Tham số `cache:` | Giá trị | Vì sao (dựa trên số liệu nào) |
|---|---:|---|
| `similarity_threshold` | `0.92` | Giá trị mặc định 0.80 quá lỏng, gây ra **181 lỗi** `wrong_answers` (lý do `other_question`). Nâng lên 0.92 giúp triệt tiêu hoàn toàn lỗi trả lời nhầm (`wrong_answers = 0`) mà vẫn đảm bảo `cache_hit_ratio = 63.5% >= 50%`. |
| `ttl_s` | `45.0` | Giá trị mặc định 600s quá dài, khiến sự thay đổi bảng giá ship không báo trước ở $t=480$ sinh ra **80 câu trả lời lỗi thời**. Giảm TTL xuống 45s giúp `outdated_answers` giảm xuống còn **6** (thỏa mãn yêu cầu $\le 15$). |
| `max_stale_s` | `300.0` | Giá trị mặc định 0s khiến client không thể trả lời khi mọi provider sập trong 120s ($t=240 \rightarrow 360$), làm `faq_availability` tụt xuống 85.87%. Đặt 300s cho phép phục vụ **174 lượt stale-if-error**, nâng `faq_availability` lên **96.45% >= 93%**. |

### Thí nghiệm phân tích:

1. **Ngưỡng tương đồng (`similarity_threshold`):**
   - Với `threshold = 0.80`: `cache_hit_ratio` đạt 74.2%, nhưng sinh ra tới **181 `wrong_answers`** do mô hình gom các câu hỏi khác nghĩa (ví dụ: "chính sách hoàn tiền" bị nhầm thành "thời gian hoàn tiền").
   - Với `threshold = 0.92` (cấu hình của tôi): `wrong_answers` giảm về **0 tuyệt đối**, trong khi `cache_hit_ratio` vẫn đạt mức rất tốt là **63.5%**, chi phí cost/turn giảm từ 1.386 xuống 0.927 đơn vị.
   - Với `threshold = 0.97`: Ngưỡng quá khắt khe, `cache_hit_ratio` tụt xuống dưới 38% (< 50% min), làm tăng số lượng cuộc gọi lên provider không cần thiết.

2. **Thời gian sống tươi (`ttl_s`):**
   - Khi chạy với `ttl_s = 300`: Ghi nhận 42 `outdated_answers` (vi phạm yêu cầu $\le 15$).
   - Khi chạy với `ttl_s = 45`: Ghi nhận **6 `outdated_answers`** (đạt yêu cầu).
   *Lý giải:* Vì sự kiện thay đổi phí ship ở $t=480$ là `silent_updates` (đối tác đổi giá nhưng không có webhook báo trước), hệ thống hoàn toàn không nhận được sự kiện để chủ động gọi `invalidate()`. Do đó, TTL chính là cơ chế duy nhất giúp tự động khai tử các bản ghi cũ và giới hạn số lượng câu trả lời sai lệch phục vụ khách hàng.

3. **Stale-if-error (`max_stale_s`):**
   - Khi đặt `max_stale_s: 0`: `stale_served = 0`, `faq_availability` sụt giảm xuống **85.87%** (FAIL vì $< 93\%$) trong thời gian provider sập.
   - Khi đặt `max_stale_s: 300`: Hệ thống phục vụ thành công **174 lượt stale**, đưa `faq_availability` lên **96.45%** (PASS).

### Trả lời câu hỏi nghiệp vụ:
- **Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") không được cache, kể cả khi cache theo từng user?**
  Dữ liệu cá nhân (trạng thái đơn hàng, số dư tài khoản, hoàn tiền) có tính biến động cao trong thời gian thực. Nếu cache, user có thể nhận thông tin trạng thái cũ (ví dụ đơn đã giao nhưng cache vẫn báo đang xử lý). Nguy hiểm hơn, trong môi trường concurrency và semantic search, nếu thuật toán phân tách tenant/user có sai sót hoặc embedding vector bị va chạm, hệ thống có thể làm rò rỉ dữ liệu nhạy cảm của khách hàng này sang khách hàng khác (vi phạm nghiêm trọng luật bảo vệ dữ liệu người dùng GDPR / PDPD).
- **Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. SRE sẽ alert dựa trên số liệu nào?**
  SRE tuyệt đối không chỉ theo dõi HTTP 200 hay Availability tại biên (Edge Availability). SRE cần thiết lập cảnh báo dựa trên:
  1. **Tỷ lệ phục vụ dữ liệu cũ (`stale_served_ratio`):** Cảnh báo ngay khi tỷ lệ stale vượt quá 5% tổng lưu lượng FAQ.
  2. **SLI Backend Provider Health:** Đo lường tỷ lệ lỗi thực tế (HTTP 5xx, 429, timeout) từ các cuộc gọi từ backend client đến provider LLM. Khi backend provider sập, SLI này sẽ kích hoạt Burn-rate Alert để đánh thức on-call xử lý kịp thời, dù người dùng cuối bên ngoài vẫn đang được cache bảo vệ.
