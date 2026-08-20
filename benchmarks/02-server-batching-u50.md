# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 14 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.98 of 4 slots (100%) |
| `requests_processing` | 4 |
| `requests_deferred` | 45 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 18912 |

Highest sampled value was **3.98 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Quan sát

**Peak batch width: 3.98 / 4 slot (99.5%).** Đây là giá trị `n_busy_slots_per_decode` cao
nhất trong 14 mẫu lấy suốt 60 s. Continuous batching hoạt động đúng: scheduler pack gần như
kín mọi decode step, không có slot nào bỏ trống.

**So với effective concurrency ở `02-server-results.md` (40.9): hai số không mâu thuẫn, vì
chúng đo hai thứ khác nhau.**

- 3.98 là *batch width* - bao nhiêu request đang thực sự decode cùng một step. Trần cứng là
  `--parallel 4`, không bao giờ vượt được.
- 40.9 là *occupancy* theo Little's Law - bao nhiêu request đang nằm trong hệ, tính cả những
  request mới chỉ xếp hàng. Số này không có trần.

Chênh lệch 40.9 − 4 ≈ 37 chính là chiều dài hàng đợi, và `requests_deferred` đỉnh 45 xác
nhận độc lập con số đó từ phía server.

**Tin cả hai, mỗi số cho một kết luận riêng.** 3.98/4 nói rằng server không lãng phí tài
nguyên - nút cổ chai không nằm ở scheduler mà ở chỗ chỉ có 4 slot. 40.9 nói rằng tải chào
gấp 10 lần năng lực phục vụ, nên phần dôi ra biến thành 12.6 s queue time trong mỗi request.
Nếu buộc phải chọn một số để hành động, chọn 3.98/4: nó là gauge server tự báo, đo trực tiếp,
không phải suy ra từ latency phía client. Và nó chỉ thẳng knob cần chỉnh - tăng `--parallel`.
