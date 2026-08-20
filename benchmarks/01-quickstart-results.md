# 01 - Measure: latency baseline

Model `Gemma 4 E2B` | host `Windows-AMD64` | llama.cpp `b10488`
Settings: `threads=6` `ngl=99` `ctx=2048`
`max_tokens=64` | warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 | `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 5857 | 478 / 736 | 84.1 / 84.7 | 5749 / 6057 / 6057 | 11.9 |
| UD-Q2_K_XL | 2.24 | 5346 | 434 / 797 | 60.6 / 61.6 | 4257 / 4628 / 4628 | 16.5 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.39x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Nhận xét của bạn

**Đáng dùng UD-Q2_K_XL cho chat tương tác. Không đáng cho bất cứ thứ gì phải đúng ngay lần
đầu.**

**Con số.** Decode 16.5 vs 11.9 tok/s (TPOT P50 60.6 vs 84.1 ms, nhanh 1.39x); E2E P50 giảm
5749 -> 4257 ms (-26%) với cùng 64 token; đĩa 2.24 vs 2.97 GB (-25%). Toàn bộ phần tăng tốc
nằm ở **decode**, không phải prefill: TTFT P50 gần bằng nhau (434 vs 478 ms) và TTFT P95 của
bản 2-bit còn *tệ hơn* (797 vs 736 ms). Đúng hình dạng lý thuyết dự đoán - TPOT bị chặn bởi
**memory bandwidth** nên hạ từ ~4.5 xuống ~2.4 bit/trọng số giảm gần tỉ lệ thuận số byte đọc
mỗi token; còn prefill bị chặn bởi **compute** trên matmul theo batch, ít bit gần như không
giúp, chi phí dequantization còn có thể làm chậm.

**Kiểm tra chất lượng.** Chạy song song hai bản (4-bit :8080, 2-bit :8090, `temperature=0`),
cùng 7 prompt:

| Prompt | UD-Q4_K_XL | UD-Q2_K_XL |
|:--|:--|:--|
| TTFT vs TPOT | đúng | đúng, gọi sai TPOT thành "Token Processing Time" |
| 4B params @ 4 bit -> GB | **hỏng**: bịa bước `x 4 bytes/bit`, rồi lặp ~200 chữ số 0 tới hết token cap | đúng (2 GB), dẫn giải cẩu thả |
| 8B params @ 2 bit -> GB | đúng, sạch | đúng, sạch |
| 3 tên metric `/metrics`, chỉ JSON array | bịa tên nhưng **giữ đúng format** | bịa tệ hơn và **phá format** (bọc trong khối ```json) |
| Tác giả "Attention Is All You Need" | đúng | đúng |
| Hàm p95, chỉ standard library | **vi phạm ràng buộc** (`import numpy`), lan man sang test | giữ đúng stdlib nhưng kẹt vòng lặp comment, không bao giờ viết xong hàm |
| Trả lời tiếng Việt | đúng, đủ 3 câu | đúng nhưng dài hơn ~40% token |

Không bản nào biết tên metric thật (`llamacpp:prompt_tokens_total`, `llamacpp:n_decode_total`,
`llamacpp:requests_processing`) - ở dạng kiến thức ghi nhớ, 4-bit không mua được gì. Khác biệt
nằm ở **khả năng tuân thủ chỉ dẫn**: bản 2-bit bỏ qua "nothing else", dài dòng, bám ràng buộc
yếu hơn - đúng thứ quantization bit thấp làm hỏng trước tiên. Nhưng bản 4-bit cũng không sạch:
cú lỗi thảm nhất cả đợt (vòng lặp chữ số 0) lại rơi vào bản precision *cao hơn*, nên 7 prompt
không đủ để tách hai bản về tail-risk.

**Kết luận.** 11.9 tok/s chậm hơn tốc độ đọc của người, cho cảm giác ì ạch; 16.5 tok/s xấp xỉ
sàn chấp nhận được cho trợ lý tương tác. Đổi chất lượng lấy 1.39x là đáng cho chat, tóm tắt,
soạn nháp - nơi người đọc từng câu và hỏi lại được. **Không** đáng cho RAG pipeline ở bước 03
hay bất cứ đầu vào nào của chương trình khác, vì chúng phụ thuộc vào output format ổn định -
đúng thứ bản 2-bit buông trước. Nếu chỉ chọn một: giữ UD-Q4_K_XL mặc định, dùng UD-Q2_K_XL
khi latency bị than phiền.

**Lưu ý phương pháp.** Bài kiểm tra chất lượng chạy hai server đồng thời trên cùng CPU 6
thread, nên wall-clock của lần đó bị tranh tài nguyên và không phải số benchmark. Bảng đầu
file - sinh bởi `lab.ps1 bench`, mỗi lần một server - mới là số đo chính thức.
