# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=6` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 174 | 2.94 | 2400 | 3600 | 4700 | 7.4 | 0.0% |
| 50 | 173 | 2.93 | 15000 | 17000 | 17000 | 40.9 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.00x** (20% of linear) |
| P95 latency | **4.72x** |
| Effective concurrency at 50 users | 40.9 vs `--parallel 4` slots (occupancy/slot ratio 10.23) |

**Saturated.** Throughput delivered only 1.00x for 5x the offered load, and effective concurrency (40.9) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.00x while P95 moved 4.72x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Nhận định

**RPS plateau phẳng:** 2.94 -> 2.93 khi tải chào tăng 5x (10 lên 50 users). Throughput đứng
yên tuyệt đối.

**P95 phồng 4.72x:** 3600 -> 17000 ms. Ở 50 users, P50/P95/P99 dồn cụm 15000/17000/17000 ms -
mọi request đều chờ cùng một hàng đợi dài.

**Effective concurrency (Little's Law) so với 4 slot:**

| | 10 users | 50 users |
|:--|--:|--:|
| N = RPS x avg latency | 2.94 x 2.51 s = 7.4 | 2.93 x 13.97 s = **40.9** |
| Occupancy / `--parallel 4` | 1.85x | **10.23x** |

Chỉ 4 request đang decode, ~37 cái còn lại xếp hàng. Gauge server xác nhận:
`requests_processing` = 4 kịch trần, `requests_deferred` đỉnh 45, `n_busy_slots_per_decode`
3.98/4.

**Phần dôi ra là queue, không phải compute.** Slot luôn đầy nên service time = 4 slot /
2.93 RPS = 1.37 s, không đổi giữa hai run. Vậy queue = 1.14 s (45% của 2.51 s) ở 10 users và
**12.60 s (90% của 13.97 s)** ở 50 users. Toàn bộ 11.5 s tăng thêm là thời gian chờ, 0 ms là
tính toán.

**Bão hòa từ dưới 10 users:** ở 10 users N đã là 7.4 > 4 slot. Knee nằm ở N = 4, khoảng 4
concurrent users. Con số thuyết phục nhất là RPS đứng yên trong khi hàng đợi dâng lên 45 -
latency có thể đổ cho nhiễu, throughput bằng phẳng thì không.

**Goodput với SLO P95 <= 5 s:** 10 users đạt (P95 3.6 s) -> 2.94 RPS; 50 users trượt
(P95 17 s) -> **0 RPS**, dù throughput thô không hề giảm.

**Knob chỉnh đầu tiên: tăng `--parallel` 4 lên 8-12, kèm nâng `--ctx-size`.** Decode bị chặn
bởi băng thông đọc trọng số GPU; một lần đọc phục vụ 12 sequence gần như cùng chi phí với 4,
nên mở batch width là cách amortize đúng chỗ. Scheduler đã pack 99.5% - hệ đang *thiếu slot*
chứ không kém hiệu quả. Ràng buộc: `ctx=2048` chia cho 4 slot mới được 512 token/slot, tăng
parallel phải tăng ctx theo, trần là 4096 MiB VRAM.

Loại các knob khác: `-t` cho đường cong phẳng 1.02x ở `01-tuning-tg128.md` (CPU không phải
nút cổ chai khi `ngl=99`); `ngl` đã 99, hết layer để offload; hạ quantization xuống Q2 có
tấn công đúng băng thông nhưng trả giá chất lượng - để bước hai.

**Việc thứ hai: admission control.** Giới hạn độ sâu hàng đợi, từ chối sớm thay vì để request
chờ 12.6 s rồi trả lời quá hạn. Không tăng RPS thô nhưng tăng goodput tại SLO.
