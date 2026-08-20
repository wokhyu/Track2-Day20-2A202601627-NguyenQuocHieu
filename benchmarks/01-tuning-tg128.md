# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **6 physical · 12 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 85.5 | 98% |
| 3 | 86.4 | 100% |
| 6 | 86.8 | 100% |
| 12 | 86.6 | 100% |
| 24 | 86.6 | 100% |

**Best**: `-t 6` at 86.8 tok/s
**Slowest tested**: `-t 1` at 85.5 tok/s (1.02x spread)
**Against the physical-core default** (`-t 6`, 86.8 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=6 make bench
```

## Giải thích

**Đường cong phẳng - không có knee.** Toàn dải `-t 1` đến `-t 24` chỉ chạy trong
85.5 - 86.8 tok/s (spread 1.02x). Đỉnh danh nghĩa rơi đúng vào `-t 6` (bằng số nhân vật lý),
nhưng cách `-t 3`, `-t 12`, `-t 24` chỉ 0.2 - 0.4 tok/s, tức nằm trong nhiễu đo. Kết luận
trung thực: `-t` gần như không ảnh hưởng.

**Lý do: sweep chạy `ngl=99`.** Toàn bộ layer đã offload lên GPU (RTX 3050 Ti Laptop,
4096 MiB, CUDA). Phần nặng của `tg128` - nhân ma trận - vector cho từng token - chạy trên
GPU, không phải trên CPU thread. CPU chỉ còn việc điều phối: dựng graph, enqueue kernel,
sampling, đồng bộ. Những việc đó tuần tự, một thread đủ, nên `-t 1` vẫn giữ 98% đỉnh.

**Nút cổ chai thật:** với batch = 1, `tg128` bị chặn bởi băng thông VRAM (mỗi token phải đọc
lại toàn bộ trọng số) cộng độ trễ launch kernel - không phải năng lực CPU. Tăng `-t` chỉ làm
phình một thread pool đang ngồi chờ GPU.

**Vì sao không sụt ở `-t 12`, `-t 24`:** trong sweep CPU thuần, vượt số nhân vật lý thường
tụt hiệu năng do hai thread SMT tranh cùng FPU/SIMD và L1/L2, cộng chi phí barrier mỗi layer.
Ở đây thread thừa chỉ idle chờ GPU nên không có gì để tranh. `-t 24` (4x nhân vật lý) mới lộ
chi phí oversubscription, cũng chỉ ăn mòn ~0.2 tok/s.

**Kết luận:** khi model nằm trọn trong VRAM, tinh chỉnh `-t` không đáng công.
`LAB_N_THREADS=6` an toàn nhưng để mặc định cũng tương đương. `-t` chỉ quan trọng lại khi
`ngl` thấp - lúc đó một phần layer chạy trên CPU và knee mới lộ ra ở số nhân vật lý.
