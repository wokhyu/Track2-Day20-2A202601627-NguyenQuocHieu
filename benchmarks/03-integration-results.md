# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 3131.2 | 3131.3 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 2813.9 | 2813.9 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 2806.1 | 2806.2 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **2917.1** · total **2917.1**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Khai báo N16-N19 và đọc số

**Cả bốn mảnh đều là stub.** Lần chạy này dùng nguyên `pipeline.py` mặc định, không nối vào
hạ tầng thật nào:

| Mảnh | Trạng thái | Cái gì đứng thay | Bằng chứng |
|:--|:--|:--|:--|
| N16 Cloud/IaC | **stub** | localhost, `llama-server` chạy tay | không có compose/k8s/terraform trong repo |
| N17 Data pipeline | **stub** | list `TOY_DOCS` in-memory, 6 doc | không có DAG hay batch job |
| N18 Lakehouse | **stub** | dict Python trong source | không có bảng Delta/Iceberg/SQLite |
| N19 Vector + features | **stub** | keyword overlap, không có embedding | `embed_backend = "keyword overlap"`, embed = 0.0 ms |

**Stage chiếm ưu thế: `llm`, 100% tổng thời gian - đúng như dự đoán, nhưng vì lý do tầm
thường.** embed và retrieve đo 0.0 ms không phải vì chúng nhanh, mà vì chúng gần như không
tồn tại: retrieve chỉ là một vòng đếm từ trùng trên 6 doc nằm sẵn trong RAM, còn embed bị
bỏ qua hoàn toàn khi không có embedding server. Với một N19 thật (index vector, gọi mạng,
top-k trên hàng triệu vector), tỷ lệ này sẽ đổi hẳn. Nói cách khác: kết quả 100% ở đây không
chứng minh gì về pipeline production, nó chỉ xác nhận đường serving hoạt động.

**Điều bất ngờ nằm bên trong stage `llm`.** Server tự báo thời gian tính toán qua
`server_timings`, và nó nhỏ hơn wall-clock rất nhiều:

| | Trung bình 3 query |
|:--|--:|
| prefill (`prompt_ms`, 376 token) | 277.8 ms |
| decode (`predicted_ms`, 77 token) | 338.3 ms |
| **Tổng compute server báo** | **616.1 ms** |
| Wall-clock stage `llm` | 2917.1 ms |
| **Chênh lệch không giải thích được** | **2301 ms (78.9%)** |

Tốc độ thô rất hợp lý: prefill ~451 tok/s, decode ~76 tok/s (khớp với 86.8 tok/s của
`llama-bench` ở `01-tuning-tg128.md`). Nghĩa là **GPU chỉ bận 21% thời gian của một request**;
gần 4/5 độ trễ nằm ngoài prefill và decode - HTTP round-trip qua loopback, tokenize/detokenize,
điều phối slot, và phần khởi động cho request đầu (query 1 chậm nhất: 3131 ms). Ba query là
quá ít để tách bạch các nguyên nhân này.

**Nếu phải giảm một nửa latency, tấn công đúng khoảng 2301 ms đó trước.** Lý do: nó là mục
lớn nhất theo bất kỳ cách chia nào, và nó *không* phải chi phí tính toán - tức có thể cắt mà
không đụng tới chất lượng model. Việc cần làm theo thứ tự: instrument client để tách rõ thời
gian mạng khỏi thời gian server, dùng một `httpx.Client` giữ kết nối thay vì mở lại mỗi
request, và bật streaming để người dùng thấy token đầu sau ~300 ms thay vì chờ trọn 2.9 giây.

Chỉ khi khoảng trống đó đã đóng thì việc chỉnh model mới đáng: giảm số token sinh ra (câu trả
lời hiện chỉ 23-30 token nên dư địa nhỏ), hoặc cắt context xuống dưới 3 doc để giảm prefill.
Đổi sang quantization thấp hơn là lựa chọn tệ nhất ở đây - decode chỉ chiếm 338 ms, có nhanh
gấp đôi cũng chỉ tiết kiệm 170 ms trên 2917 ms, đổi lại rủi ro sai format đã ghi nhận ở
`01-quickstart-results.md`.
