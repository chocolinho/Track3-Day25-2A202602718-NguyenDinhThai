# Lab 25B — Report

**Họ tên / nhóm:** Nguyễn Đình Thái (2A202602718)  **Commit:** a2da4e7  **Điểm tự động (`make grade`):** 84 / 88 *(100% public benchmark, 4 điểm ẩn giảng viên chấm)*

> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | **PASS** |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0000 | 11.911 | 1.059 | 0.008 | 2.161 | 0.0 | **PASS** |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0000 | 4.738 | 1.058 | 0.034 | 2.141 | 0.0 | **PASS** |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0.0 | **PASS** |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | N/A | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | N/A | **PASS** |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.961 | 1.055 | 0.018 | 2.196 | 0.0 | **PASS** |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0 | Băng G6: Khi alpha-eu bị treo hoàn toàn (`outage: hang`), timeout 8.0s giúp cắt lỗ sớm và còn đủ thời gian trước deadline 30s để failover sang beta; nếu để 30s availability G6 rớt xuống 94.88% (< 95%). |
| max_attempts_per_provider | 2 | Băng G2 & G3: Cho phép 1 lần retry trên cùng provider (tổng 2 attempts), tránh ném tải dồn dập vào provider đang ngộp nhưng vẫn cứu được các lỗi thoáng qua (transient blip). |
| backoff_base_s / cap_s | 0.25 / 4.0 | Băng G1 & G2: Full jitter với base 0.25s và cap 4.0s giúp phân tán các đợt retry không cùng ùa vào 1 tích tắc, cap 4s đủ ngắn để không làm cạn kiệt deadline của lượt gọi (30s). |
| retry_budget_ratio | 0.1 | Băng G3 & G2: Giới hạn lượng retry không vượt quá 10% first attempts; giữ retry_ratio ở G3 là 0.034 (tiêu chuẩn $\le 0.10$), ngăn chặn triệt để retry storm khi sự cố kéo dài. |
| breaker.window_size / min_calls | 20 / 10 | Băng G2 & G4: Cửa sổ 20 request gần nhất phản ánh nhanh chóng sự thay đổi sức khỏe provider; min_calls 10 ngăn ngừa ngắt mạch nhầm khi lưu lượng thấp lúc khởi động hoặc ban đêm. |
| breaker.failure_rate_threshold | 0.5 | Băng G2: Khi alpha mất 75% GPU và tỷ lệ lỗi vượt quá 50%, breaker lập tức trip sang OPEN để chuyển hướng traffic sang alpha-eu, giảm amplification về 1.059. |
| breaker.slow_call_s | 5.0 | Băng G4: Alpha phản hồi chậm thêm 5s không có lỗi (median ~6.2s); đặt ngưỡng 5.0s giúp breaker phát hiện chậm và trip `slow_call_rate`, đưa p95 về 5.562s ($\le 10$s). |
| breaker.open_duration_s | 10.0 | Băng G2 & G4: Cách ly provider trong 10s cho phép hàng đợi phía server xả sạch các request cũ trước khi thử probe lại ở HALF-OPEN. |
| breaker.half_open_max_probes | 2 | Băng G2: Chỉ cho phép tối đa 2 request đồng thời thăm dò ở HALF-OPEN, triệt tiêu hoàn toàn hiện tượng thundering herd đè bẹp provider vừa chớm hồi phục. |

Sơ đồ quyết định của `on_error` (Bảng quyết định):

```
+-------------------+----------------------+--------------------------------------------------------------+
| ErrorKind         | Tác động Breaker     | Hành vi tiếp theo                                            |
+-------------------+----------------------+--------------------------------------------------------------+
| CLIENT_ERROR      | release permit       | GiveUp("client_error") ngay lập tức (không retry, ko failover)|
| QUOTA_EXHAUSTED   | release permit       | cooldown provider = 300s; failover ngay sang provider khác   |
| RATE_LIMITED      | release permit       | cooldown provider = now + retry_after; failover ngay         |
| OVERLOADED /      | ghi nhận on_failure  | NẾU (tries < 2 AND budget còn AND đủ deadline):             |
| SERVER_ERROR /    |                      |     retry cùng provider sau FullJitterBackoff.delay(tries)   |
| TIMEOUT           |                      | NẾU KHÔNG: failover sang provider kế tiếp trong ladder       |
+-------------------+----------------------+--------------------------------------------------------------+
```

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối.** Giả lập breaker cấu hình sai (`breaker.min_calls: 1000000000`) và đặt `max_attempts_per_provider: 5`. Chạy G2: Khi có budget, availability đạt **99.83%**, amplification = **1.378**, retry_ratio = **0.058**, recovery = **0.0s**. Khi tắt budget, amplification vọt lên **1.675** (vi phạm ngưỡng 1.6), retry_ratio vọt lên **0.495** (gấp 8.5 lần), recovery mất tới **25.0s** mới phục hồi. Trên G3: Tắt budget khiến retry_ratio tăng vọt lên **0.159** (vi phạm tiêu chí $\le 0.10$). Khi breaker hoạt động đúng, budget vẫn cực kỳ quan trọng vì breaker bảo vệ ở cấp độ từng provider riêng lẻ, trong khi retry budget bảo vệ toàn hệ thống khỏi cạn kiệt tài nguyên; việc có cả hai tạo nên mô hình phòng thủ theo chiều sâu (defense-in-depth).

2. **`attempt_timeout_s: 30`.** Chạy G4 và G6: Trên G4, breaker bắt slow-call nên vẫn chuyển hướng tốt (p95 = 5.608s, availability 100%). Tuy nhiên trên G6, khi alpha-eu bị treo không hồi đáp, attempt_timeout_s 30s khiến client giữ kết nối chờ đợi quá lâu, làm cạn kiệt deadline 30s của lượt user và không còn cơ hội failover sang beta $\to$ availability của G6 rớt xuống **94.88%** (vi phạm tiêu chuẩn $\ge 95\%$). Timeout "đúng" phải được thiết lập dựa trên p99 latency thực tế khi provider khỏe mạnh cộng thêm một lượng sai số hợp lý (latency budget), và phải nhỏ hơn nhiều so với turn deadline tổng (30s) để đảm bảo client còn đủ thời gian failover sang provider dự phòng.

3. **Bỏ lọc tool calling khỏi `ladder()`.** Chạy G6 khi cả provider A sập: Hệ thống ghi nhận tới **269** lượt `incompatible_routes` và availability tổng thể tụt dốc từ 100% xuống **88.72%**. Bằng ngôn ngữ của một Product Manager: "Khi hệ thống gặp sự cố mất 2 nhà cung cấp chính, nếu không có bộ lọc tương thích, hệ thống đã gửi nhầm 269 yêu cầu hoàn tiền (vốn đòi hỏi AI phải gọi tool kiểm tra đơn hàng và tài khoản ngân hàng) sang model giá rẻ gamma-mini vốn không có khả năng gọi tool. Kết quả là 269 khách hàng nhận câu trả lời ảo (hallucination) thay vì được hoàn tiền, trực tiếp làm hỏng trải nghiệm khách hàng và gây thất thoát niềm tin vào tính năng CSKH tự động."

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0 | 81 | 128 | 81 | 0 | 15.742,00 $ | 1.123 |
| yours | 1.0 | 66 | **0** | **0** | **0** | **0,00 $** | **0.303** |

**Trả lời:** Chỉ idempotency key là **chưa đủ** vì nếu worker crash, worker mới không biết workflow đang dừng ở bước nào và sẽ phải chạy lại từ đầu (lãng phí chi phí gọi LLM và thời gian xử lý). Ngược lại, chỉ checkpoint cũng **chưa đủ** vì nếu mạng chập chờn khi gọi cổng thanh toán (ServiceTimeout), checkpoint chưa được ghi, worker retry sẽ gửi một request mới; nếu thiếu idempotency key, cổng thanh toán sẽ trừ tiền lần thứ hai.
Nếu ghi checkpoint *trước* khi gọi payment: Lỡ worker bị crash ngay sau khi ghi checkpoint nhưng chưa kịp gọi payment (hoặc đang gọi thì crash), worker tiếp theo phục hồi sẽ thấy checkpoint đã có và tưởng rằng tiền đã được hoàn, dẫn đến việc bỏ qua bước thanh toán $\to$ khách hàng không nhận được tiền (**lost refunds**).

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page | page | 27 min |
| provider_outage | page | page | page | 4 min |
| deploy_blip | none | page (false alarm) | none | N/A |
| slow_leak | ticket | none (missed) | ticket | 1337 min |
| degraded_evening | page | page | page | 291 min |
| night_spike | none | page (false alarm) | none | N/A |
| small_bump | none | page (false alarm) | none | N/A |

Error budget 30 ngày: đã tiêu **42.95%**, còn **57.05%** (availability đạt 99.785% so với cam kết 99.500%). **Không** đóng băng release (`freeze_releases: False`) vì budget còn hơn 57%.
Sự cố `small_bump` (20 phút ở 12% lỗi) **không** nên đánh thức ai lúc 11 giờ trưa vì tổng lượng lỗi mà nó gây ra chỉ tiêu tốn một phần cực nhỏ của ngân sách lỗi tháng; nó không kéo dài đủ lâu để đe dọa cam kết SLO 30 ngày. Đánh thức on-call vì một đợt tăng lỗi ngắn sẽ gây ra hội chứng mệt mỏi vì cảnh báo (alert fatigue), làm giảm năng suất và phân tâm kỹ sư khỏi các sự cố thực sự nghiêm trọng.

## 6. Postmortem — Băng G2 với client naive (blameless)

![G2 Overload Storm](gameday/G2_overload_storm.png)

**Tóm tắt (3 câu):** Vào giây ảo t=120, provider chính alpha gặp sự cố sụt giảm 75% năng lực xử lý trong vòng 3 phút do mất GPU. Bản client v1 (naive) với cơ chế retry 3 tầng không backoff/jitter đã phản ứng bằng cách nã dồn dập các lượt retry, làm khuếch đại lưu lượng lên gấp 9.9 lần và khiến tỷ lệ cuộc gọi thành công sập về 0. Ngay cả khi provider alpha đã hồi phục 100% năng lực ở t=300, hệ thống vẫn tiếp tục tê liệt thêm 155 giây do rơi vào trạng thái lỗi siêu bền (metastable failure), chỉ thực sự hồi phục ở t=455 khi các đợt retry cạn kiệt.

**Ảnh hưởng:** 1.285 lượt user thất bại (tổng availability rớt xuống 45.60%), sự cố sập hoàn toàn kéo dài 335 giây (t=120 đến t=455). Giả sử tháng chỉ có sự cố này, nó tiêu tốn vượt quá **108.8%** toàn bộ error budget tháng của hệ thống (làm âm ngân sách lỗi tháng).

**Timeline** (giây ảo — lấy từ CSV `reports/gameday/naive_G2_overload_storm_timeseries.csv`):

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 0–119 | Hệ thống hoạt động bình thường | Lưu lượng ~7–10 attempts/s, calls_ok = 7–15/s, availability = 100%. |
| 120 | alpha mất 75% năng lực GPU | Dung lượng phục vụ giảm đột ngột còn 0.25; hàng đợi bắt đầu ứ đọng; calls_ok giảm về 2/s ở t=121. |
| 136 | Bùng nổ retry storm tầng thứ nhất | Các cuộc gọi bắt đầu timeout; client retry không backoff; attempts_alpha vọt từ 24 lên 53 req/s. |
| 143 | Hệ thống sụp đổ hoàn toàn | attempts_alpha đạt đỉnh 254 req/s; calls_ok rơi về đúng 0; 100% người dùng bị từ chối phục vụ. |
| 150–299 | Giai đoạn duy trì quá tải | attempts_alpha duy trì liên tục ở mức 100–180 req/s, calls_ok = 0; server chìm trong 529 và timeout. |
| 300 | alpha hồi phục 100% năng lực phần cứng | GPU của alpha đã hồi phục, nhưng attempts_alpha vẫn nã vào 115 req/s; calls_ok vẫn là 0/s. |
| 300–452 | Hệ thống vẫn sập — metastable failure! | Hàng đợi server bị nghẽn bởi hàng ngàn request cũ mà client đã timeout/bỏ cuộc (wasted work); server phí công xử lý request chết. |
| 455 | Hệ thống bắt đầu phục hồi | Các lượt retry của các call cũ hết số lần attempt; attempts_alpha giảm về 13 req/s; calls_ok phục hồi đạt 4 req/s (t=455), đạt lại 95% sau 155s kể từ t=300. |

**Trigger vs root cause:**
- **Trigger:** Provider alpha bị mất 75% GPU trong 3 phút (t=120 đến t=300).
- **Root cause:** Kiến trúc client v1 có 3 tầng retry lồng nhau (App $\times$ Gateway $\times$ SDK = 36 attempts/call) với fixed delay 0.2s, hoàn toàn không có jitter, không có retry budget, không có circuit breaker và không có cơ chế failover. Khi trigger xảy ra, lượng attempt tăng vọt gấp 10 lần (amplification 9.904, 34.475 attempts cho 3.481 calls), biến một sự cố suy giảm năng lực tạm thời thành một thảm họa sập toàn diện. Hệ thống rơi vào vòng lặp **Metastable Failure**: hàng đợi provider nghẹt cứng các request mà user đã từ bỏ (wasted work), trong khi client liên tục gửi request mới và retry chồng chéo, tự duy trì tải vượt xa năng lực của server kể cả sau khi server đã phục hồi.

**Điều gì giúp / điều gì làm tệ hơn:**
- *Điều làm tệ hơn:* Delay cố định 0.2s làm các request retry đồng bộ hóa với nhau; việc tiếp tục xử lý các request đã timeout ở server làm lãng phí 100% năng lực GPU còn lại; thiếu circuit breaker khiến 100% traffic dồn vào alpha mà không chuyển tải sang alpha-eu/beta.
- *Điều giúp ích:* Giới hạn thời gian turn_deadline_s (30s) và MAX_ATTEMPTS cuối cùng đã giúp các đợt retry cạn kiệt sau t=452, cho phép hàng đợi xả hết và hệ thống tự thoát khỏi metastable state sau 155 giây.

**Action items** (cụ thể, có owner, đo được):

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Triển khai ResilientClient tích hợp Circuit Breaker và Retry Budget (Part B, C) | Prevent | Tech Lead (Thái) | Amplification $\le 1.10$, retry_ratio $\le 0.01$ trên băng G2 trong pipeline CI. |
| 2 | Giảm số tầng retry trong gateway và SDK xuống đúng 1 tầng duy nhất có jitter | Prevent | Platform Team | Code review & audit cấu hình SDK; 0 tầng retry chồng lấn trong test suite. |
| 3 | Triển khai cảnh báo burn-rate đa cửa sổ (Part E) để phát hiện sớm quá tải | Detect | SRE On-call | Phát hiện sự cố quá tải lớn trong vòng $< 5$ phút, 0 false page trên blip. |
| 4 | Cấu hình tự động failover sang alpha-eu và beta khi breaker của alpha mở | Mitigate | SRE Team | Availability đạt $\ge 99\%$ trên kịch bản G2 với recovery time $\le 5$s. |

## 7. Một điểm yếu còn lại

**Điểm yếu:** Khi provider primary (`alpha`) bị sập hoặc nghẽn, client failover toàn bộ lưu lượng sang `beta`. Tuy nhiên, chi phí của `beta` cao hơn 40% (cost 1.4 vs 1.0) và năng lực xử lý giới hạn hơn (rate limit 16 rps vs 20 rps của alpha). Nếu sự cố diễn ra trên diện rộng và kéo dài, việc toàn bộ tenant đổ dồn sang beta sẽ nhanh chóng làm beta bị nghẽn rate limit (như thấy ở G6), đồng thời làm chi phí vận hành tăng vọt. Ngoài ra, độ trễ p95 ở G2 vẫn chạm mức 11.91s do các attempt ban đầu phải chờ đợi trước khi failover.
**Cách khắc phục:** 
1. Áp dụng cơ chế **Dynamic Cost-Aware Shedding**: Khi chạy trên beta trong sự cố, tạm thời hạ bớt độ dài prompt hoặc chỉ ưu tiên phục vụ các luồng hoàn tiền giá trị cao, các tác vụ phụ chuyển sang xử lý bất đồng bộ.
2. Áp dụng **Hedging Requests**: Với các request quan trọng, gửi song song request sang alpha-eu sau khi request ở alpha vượt quá ngưỡng p75 latency mà chưa có phản hồi, lấy kết quả về đầu tiên và hủy request còn lại, giúp kéo p95 latency xuống dưới 5 giây.

## 8. Cache (Part G)

Số từ `reports/cache/*.json` (`python scripts/run_gameday.py --scenarios configs/scenarios_cache.yaml --client resilient,cached --out reports/cache`).

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | **0.9124** | **0.9645** | **0.652** | **157** | **0** | **3** | **0.915** | **PASS** |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.92 | Ở ngưỡng 0.80, hệ thống trả lời nhầm 157 câu hỏi (`other_question`). Nâng lên 0.92 giúp triệt tiêu hoàn toàn câu trả lời sai (`wrong_answers = 0`) trong khi vẫn duy trì hit ratio cao (65.2%). |
| ttl_s | 60 | Ở TTL 600s, khi bảng phí ship đổi lúc t=480 mà không có webhook, hệ thống phục vụ 80 câu trả lời cũ. Đặt TTL 60s giới hạn số câu trả lời cũ xuống chỉ còn 3 câu ($\le 15$). |
| max_stale_s | 600 | Khi toàn bộ provider sập trong 120s (t=240–360), cấu hình starter (max_stale_s=0) khiến faq_availability tụt xuống 87.33%. Cấu hình 600s cho phép phục vụ 157 câu stale an toàn, nâng faq_availability lên 96.45% ($\ge 93\%$). |

Thí nghiệm (mỗi cái 2–4 câu + số):

1. **Ngưỡng.** Chạy C1 với `similarity_threshold` = 0.80 / 0.92 / 0.97: 
   - Với 0.80: hit ratio đạt 66.6%, chi phí 0.965, nhưng gây ra **157** câu `wrong_answers` (toàn bộ do `other_question`). Các cặp câu bị trả nhầm là những câu có embedding gần nhau nhưng bản chất khác nhau, ví dụ: *"Chính sách hoàn tiền thế nào?"* bị nhầm với *"Thời gian nhận tiền hoàn là bao lâu?"*, hoặc *"Phí ship đi tỉnh là bao nhiêu?"* bị nhầm với *"Chính sách freeship đơn từ 500k"*.
   - Với 0.92: hit ratio đạt **65.2%**, **0** wrong answers, chi phí tối ưu nhất đạt **0.915/turn**.
   - Với 0.97: threshold quá chặt làm hit ratio giảm xuống **58.1%**, làm tăng số lượt phải gọi provider và đẩy chi phí lên 0.964/turn.

2. **TTL.** Chạy với `ttl_s` = 30s / 60s / 300s:
   - TTL 30s: `outdated_answers` = 2, hit ratio = 59.2%.
   - TTL 60s: `outdated_answers` = 3, hit ratio = 65.2%.
   - TTL 300s: `outdated_answers` = 95 (vi phạm nghiêm trọng tiêu chuẩn $\le 15$).
   Webhook CMS không giúp gì được cho `outdated_answers` ở sự cố t=480 vì đây là thay đổi từ đối tác vận chuyển mà **không có thông báo trước** (silent update, CMS không phát sinh event webhook nào). Do đó, TTL chính là cơ chế phòng vệ duy nhất để giới hạn "thời gian sống" tối đa của một thông tin cũ.

3. **Stale-if-error.** Chạy với `max_stale_s: 0`:
   `faq_availability` sụt giảm từ **96.45%** xuống **87.33%** (vi phạm tiêu chuẩn $\ge 93\%$), và `stale_served` = **0**. Khi toàn bộ provider ngừng hoạt động trong 120s (t=240–360), việc không cho phép dùng stale answer khiến mọi câu hỏi FAQ đều thất bại, làm mất đi tầng phòng thủ dự phòng quan trọng nhất trong fallback ladder.

**Trả lời câu hỏi:**
- **Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") không được cache, kể cả khi cache theo từng user?**
  Bởi vì câu hỏi cá nhân phụ thuộc vào trạng thái thời gian thực (*real-time transactional state*) của đơn hàng (ví dụ: trạng thái giao hàng, số dư hoàn tiền, điểm tích lũy thay đổi liên tục). Nếu cache, người dùng sẽ nhận được thông tin cũ (ví dụ: đơn hàng đã giao nhưng cache vẫn báo đang đóng gói), dẫn đến hiểu lầm nghiêm trọng; ngoài ra dữ liệu cá nhân có rủi ro rò rỉ bảo mật cao nếu có bất kỳ lỗi nào trong việc quản lý cache key.
- **Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?**
  On-call không thể chỉ dựa vào `availability` ở tầng người dùng (vì cache đang che giấu sự cố). Cần phải thiết lập alert dựa trên:
  1. **`stale_served_ratio` / `stale_served_count`**: Tỷ lệ hoặc tần suất các câu trả lời đang phải phục vụ dưới dạng stale-if-error tăng vọt.
  2. **Upstream Provider Error Rate / Circuit Breaker State**: Giám sát trực tiếp tỷ lệ lỗi ở tầng gọi LLM provider (`overloaded_error`, `503`, `client_timeout`) và số lượng Circuit Breaker đang ở trạng thái `OPEN`. Khi breaker mở hoặc tỷ lệ stale tăng, hệ thống phải kích hoạt page on-call ngay lập tức.
