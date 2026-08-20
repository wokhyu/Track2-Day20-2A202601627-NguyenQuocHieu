# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Quốc Hiệu
**Cohort:** A20-K3
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 Home Single Language (build 10.0.26200)
- **CPU:** AMD Ryzen 5 6600H with Radeon Graphics (Zen 3+)
- **Cores:** 6 physical / 12 logical
- **CPU extensions:** AVX2 (Zen 3+ không có AVX-512)
- **RAM:** 15.2 GB
- **Accelerator:** NVIDIA GeForce RTX 3050 Ti Laptop GPU, 4096 MiB VRAM — backend CUDA (Vulkan có mặt nhưng không dùng)
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-cuda-12.4-x64.zip` (build `b10488`, binary dựng sẵn, không tự compile)
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** `UD-Q4_K_XL` (primary) + `UD-Q2_K_XL` (compare) — từ `models/active.json`

**Chạy ở đâu:** laptop của tôi (local, không dùng cloud fallback)
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Windows không có `make` nên mọi target chạy qua `.\lab.ps1`. Hai chỗ phải sửa, cả hai đều
là encoding: console PowerShell 5.1 dùng cp1252 nên em dash trong `lab.ps1` hiện thành ký
tự rác — thay bằng hyphen; và `labkit.write_report()` ghi file theo locale cp1252, làm phần
nhận xét tiếng Việt không encode được, kéo theo `verify.py` crash khi đọc lại. Sửa cả hai
sang UTF-8 tường minh, kèm reconfigure stdout vì `verify.py` in ký tự `✗` cũng crash trên
console cp1252. Không bước nào fail về phía model hay CUDA.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 5857 | 478 / 736 | 84.1 / 84.7 | 5749 / 6057 / 6057 | 11.9 |
| UD-Q2_K_XL | 2.24 | 5346 | 434 / 797 | 60.6 / 61.6 | 4257 / 4628 / 4628 | 16.5 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

2-bit decode nhanh 1.39x (16.5 vs 11.9 tok/s), E2E P50 giảm 26%, đĩa nhỏ hơn 0.73 GB.
Phần tăng tốc nằm hết ở decode; TTFT gần như không đổi (434 vs 478 ms) và TTFT P95 còn tệ
hơn — đúng hình dạng memory-bandwidth-bound. Đã hỏi 7 prompt giống nhau trên cả hai server
(`temperature=0`): bản 2-bit hỏng ở **khả năng tuân thủ chỉ dẫn** — phá format JSON được
yêu cầu, trả lời dài dòng, không viết xong hàm. Đáng dùng cho chat; không đáng cho RAG.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 2.94 | 2400 | 3600 | 4700 | 7.4 | 0.0% |
| 50 | 2.93 | 15000 | 17000 | 17000 | 40.9 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.00×
- **P95 tăng:** 4.72×
- **Effective concurrency ở 50 users:** 40.9 so với `--parallel` = 4 slots (occupancy 10.23×)

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.98 / 4 slots (99.5%) — kèm `requests_deferred` đỉnh 45

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Bão hoà từ **dưới 10 users**: ở 10 users effective concurrency đã 7.4 > 4 slot. Bằng chứng
thuyết phục nhất là RPS đứng yên 2.94 → 2.93 khi tải chào tăng 5×, trong khi hàng đợi dâng
lên 45. Phần latency thêm là **queue time**, biết bằng cách: slot luôn đầy nên service time
= 4 slot / 2.93 RPS = 1.37 s và không đổi giữa hai run; vậy trong 13.97 s trung bình ở 50
users, 1.37 s là compute còn 12.60 s (90%) là chờ. Knob đổi trước: `--parallel` 4 → 8-12
kèm nâng `--ctx-size` (hiện chỉ 512 token/slot). Vì decode bị chặn bởi băng thông đọc trọng
số — một lần đọc phục vụ 12 sequence gần như cùng chi phí với 4 — và gauge 3.98/4 cho thấy
scheduler đã pack kín, tức đang *thiếu slot* chứ không kém hiệu quả. Trần là 4096 MiB VRAM
nên phải đo lại sau khi tăng.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | localhost, `llama-server` chạy tay | **stub** (không compose/k8s/terraform) |
| N17 Data pipeline | list `TOY_DOCS` in-memory, 6 doc | **stub** (không DAG, không batch job) |
| N18 Lakehouse | dict Python trong source | **stub** (không Delta/Iceberg/SQLite) |
| N19 Vector + features | keyword overlap, không embedding | **stub** (`embed_backend = "keyword overlap"`) |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 2917.1 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

Khớp kỳ vọng nhưng vì lý do tầm thường: embed/retrieve = 0.0 ms không phải vì nhanh mà vì
cả hai đều là stub trên 6 doc nằm sẵn trong RAM. Điều bất ngờ nằm *bên trong* stage llm:
`server_timings` cho prefill 277.8 ms + decode 338.3 ms = 616 ms, so với wall-clock 2917 ms
— **2301 ms (79%) không phải compute**. Muốn giảm 2× thì tấn công đúng khoảng trống đó (giữ
kết nối `httpx.Client`, tách đo mạng khỏi server, bật streaming), không phải đổi model:
decode chỉ 338 ms, có nhanh gấp đôi cũng chỉ cắt 170/2917 ms.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Thí nghiệm chính là thread sweep của `make tune` (`-t` từ 1 đến 24, `ngl=99`,
metric `tg128`). Nó cho một **null result** — và chính null result đó là phát hiện đáng giá
nhất của lab, vì nó xác định nút cổ chai *không* nằm ở đâu. Thay đổi thật sự dịch được số
là đổi quantization, và cơ chế giải thích cả hai là một.

```
make tune  (-t 1 -> -t 6, ngl=99, tg128):
before:  85.5 tok/s   (-t 1)
after:   86.8 tok/s   (-t 6, dinh cua ca sweep 1/3/6/12/24)
speedup: 1.02x        <- null result, nam trong nhieu do

quantization UD-Q4_K_XL -> UD-Q2_K_XL  (make bench):
before:  11.9 tok/s   (TPOT P50 84.1 ms, 2.97 GB)
after:   16.5 tok/s   (TPOT P50 60.6 ms, 2.24 GB)
speedup: 1.39x
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

**Vì sao `-t` không làm gì — và chỗ này khác kỳ vọng từ deck.** Deck dạy đường cong thread
có knee ở số nhân vật lý rồi tụt xuống, vì hai thread SMT tranh cùng một FPU/SIMD unit và
cùng L1/L2 cache, cộng chi phí barrier đồng bộ ở cuối mỗi layer. Sweep của tôi phẳng lì
1.02× từ `-t 1` tới `-t 24`, và riêng `-t 1` vẫn giữ 98% đỉnh. Cơ chế: với `ngl=99` toàn bộ
layer nằm trên GPU, nên **CPU không nằm trên đường tới hạn**. Vòng lặp decode chỉ còn để CPU
dựng compute graph, enqueue kernel, sampling rồi chờ GPU — một chuỗi việc *tuần tự*, không
chia nhỏ cho nhiều thread được. Thêm thread chỉ làm phình một thread pool đang ngồi chờ. Đó
cũng là lý do phần "tụt sau knee" biến mất: SMT contention và cache thrashing chỉ tính tiền
khi các thread thật sự đang tính toán, mà ở đây chúng không có gì để tranh. Tới `-t 24` (4×
số nhân vật lý) mới lộ chi phí oversubscription của scheduler OS, nhưng cũng chỉ ăn 0.2
tok/s — đủ để thấy cơ chế có tồn tại, quá nhỏ để đáng chỉnh.

**Nút cổ chai thật là memory bandwidth, và phép thử quantization chứng minh bằng số.** Trong
decode với batch = 1, mỗi token sinh ra phải đọc lại **toàn bộ** trọng số model từ VRAM, rồi
làm đúng một multiply-add trên mỗi trọng số vừa đọc. Arithmetic intensity vì thế xấp xỉ
2 FLOP/byte — nằm sâu bên trái ridge point của mọi GPU hiện đại, tức điểm làm việc rơi vào
nhánh bandwidth của roofline chứ không phải nhánh compute. Hệ quả kiểm chứng được: thời gian
mỗi token phải tỉ lệ **thuận với số byte trọng số**, và không phụ thuộc số thread CPU.
UD-Q2_K_XL nhỏ hơn UD-Q4_K_XL theo tỉ lệ 2.97/2.24 = 1.33×; nếu mô hình bandwidth đúng thì
decode phải nhanh lên đúng chừng đó. Đo được **1.39×** — lệch 5% so với dự đoán thuần từ tỉ
lệ byte. Hai chi tiết phụ củng cố thêm: (a) TTFT gần như đứng yên (478 → 434 ms) vì prefill
xử lý cả prompt trong một matmul theo batch, tái dùng mỗi trọng số cho nhiều token nên bị
chặn bởi **compute** — ít bit không giúp gì ở đó, và TTFT P95 của bản 2-bit thậm chí *tệ
hơn* (797 vs 736 ms), hợp lý vì dequantization Q2 tốn thêm việc; (b) nếu một chi phí cố định
(client, mạng, sampling) chiếm phần lớn TPOT thì thu nhỏ model sẽ mua được **ít hơn** 1.33×
rất nhiều — đo ra 1.39× nghĩa là gần như *toàn bộ* chi phí mỗi token trong lần chạy đó co
giãn theo kích thước model. Đúng chữ ký của một hệ bandwidth-bound.

**Kết luận rút ra.** Thứ tự knob đáng chỉnh trên máy này được quyết định bởi một câu hỏi duy
nhất: knob đó có giảm số byte phải đọc cho mỗi token không? Quantization có (1.39×), và tăng
batch width qua `--parallel` cũng có — một lần đọc trọng số phục vụ được nhiều sequence, đó
chính là knob tôi chọn cho phần load ở §3. `-t` thì không, `ngl` đã 99 nên cũng hết dư địa.
Một null result được giải thích đúng vẫn tiết kiệm thời gian thật: nó loại nguyên một họ knob
khỏi danh sách phải thử.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** không làm bonus track nào.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

Cùng một model, cùng `ngl=99`, ba cách đo cho ba tốc độ decode lệch nhau tới 7×:
`llama-bench` 86.8 tok/s, `server_timings` trong pipeline 81 tok/s (30 token / 369 ms),
nhưng client streaming ở `make bench` chỉ thấy 11.9 tok/s. Pipeline cũng cho thấy 79%
wall-clock của stage llm không phải prefill hay decode. Tôi **chưa** tách được nguyên nhân
— ứng viên là chi phí mỗi chunk SSE phía client, hoặc lần chạy bench không được offload GPU
như hai lần kia. Phép đo cần làm tiếp rất rẻ: `benchmark.py` đã nhận `timings` từ SSE nhưng
chỉ dùng `predicted_n`; ghi thêm `predicted_ms` là tách được ngay thời gian server khỏi thời
gian client. Bài học: một con số tok/s không kèm ngữ cảnh đo ở đâu thì gần như vô nghĩa.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/` (hiện có 7)
- [x] `make verify` → **exit 0**
- [x] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [x] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
