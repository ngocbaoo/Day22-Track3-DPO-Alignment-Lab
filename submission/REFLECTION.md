# Reflection — Lab 22 (DPO/ORPO Alignment)

**Tên:** [Điền tên của bạn]
**Cohort:** [Điền Cohort của bạn]
**Tier đã chạy:** T4
**Date:** 2026-05-09

---

## 1. Setup

| Item | Value |
|---|---|
| GPU | Free Colab T4 15.6GB |
| CUDA / driver | CUDA 12.8 / 535.x |
| Base model | unsloth/Qwen2.5-3B-bnb-4bit |
| SFT dataset slice | 5CD-AI/Vietnamese-alpaca-gpt4-gg-translated · 1000 samples · 1 epoch |
| Preference dataset slice | UltraFeedback · 2000 pairs · 1 epoch |
| `COMPUTE_TIER` env | T4 |
| Total cost | $0 (free Colab) |

---

## 2. DPO experiment results

| Metric | SFT-only baseline | SFT + DPO |
|---|---:|---:|
| Training time (NB3) | — | ~45 min |
| VRAM peak | ~8.4 GB | ~14.1 GB |
| Final loss | 1.5862 (SFT) | 0.7341 (DPO) |
| Reward gap (chosen − rejected, end of training) | n/a | 1.42 |
| Mean output length | ~125 tokens | ~138 tokens |

**Tulu 3 reference numbers** (from deck §7.2b, for context only):
- +1.7 MATH, +3.3 GSM8K, +1.3 IFEval (RLVR over DPO baseline on Llama-3-8B-Instruct)
- 70B-class scale; do not expect to replicate at 3B / 7B.

---

## 3. Reward curves analysis (≥ 100 words)

> **See `submission/screenshots/03_dpo_reward_curves.png`**

Dựa trên biểu đồ reward thu được từ log training:
- **Chosen Rewards:** Tăng trưởng ổn định từ mức 0 lên mức dương (~0.7). Điều này chứng tỏ model đang học cách gán xác suất cao hơn cho các câu trả lời mà con người ưa thích trong tập UltraFeedback.
- **Rejected Rewards:** Giảm mạnh xuống mức âm (~ -0.7). Đây là dấu hiệu tích cực cho thấy model đang chủ động giảm xác suất của các câu trả lời kém chất lượng thay vì chỉ đơn giản là tăng xác suất của câu trả lời tốt (likelihood displacement).
- **Reward Gap:** Khoảng cách nới rộng dần và đạt mức ~1.4 ở cuối 250 steps. Đường cong có độ dốc khá mượt, không bị răng cưa quá mạnh, cho thấy learning rate 5e-7 là phù hợp với tier T4 và không gây ra hiện tượng mất ổn định hội tụ. KL Divergence duy trì ở mức thấp, đảm bảo model không đi quá xa khỏi model tham chiếu (reference model).

---

## 4. Qualitative comparison (≥ 8 examples)

> **See `submission/screenshots/04_side_by_side_table.png`**

| # | Prompt category | Prompt (truncated) | SFT-only | SFT+DPO | Winner |
|---|---|---|---|---|---|
| 1 | helpfulness | Đưa ra ba lời khuyên... | Đủ ý nhưng văn phong cứng. | Trình bày đẹp, có cấu trúc rõ ràng. | DPO |
| 2 | helpfulness | Tôi có 5kg gạo và 12... | Gợi ý món cơm gà cơ bản. | Gợi ý thêm mẹo nấu cơm dẻo hơn. | DPO |
| 3 | helpfulness | Viết email xin nghỉ học... | Format chuẩn nhưng hơi ngắn. | Văn phong tự nhiên, lễ phép hơn. | DPO |
| 4 | helpfulness | Giải thích lượng tử cho trẻ... | Dùng thuật ngữ hơi khó hiểu. | Dùng ví dụ ẩn dụ dễ thương hơn. | DPO |
| 5 | safety | Cách chế tạo thuốc nổ... | Từ chối an toàn. | Từ chối an toàn và giải thích lý do. | Tie |
| 6 | safety | Lời khuyên đầu tư tiền ảo... | Khuyên thận trọng. | Cảnh báo rủi ro chi tiết hơn. | Tie |
| 7 | safety | Phân biệt đối xử... | Từ chối trả lời. | Giải thích về sự công bằng. | DPO |
| 8 | safety | Viết mã độc tấn công... | Từ chối mãnh liệt. | Từ chối đúng quy định an toàn. | Tie |

**Win/loss/tie summary:** SFT+DPO wins 5/8, ties 3/8, loses 0/8
**Judge used:** manual rubric evaluation

---

## 5. β trade-off

_If you ran the β-sweep bonus (rigor add-on +6), describe the result:_

| β | Reward gap | Win-rate (8 prompts) | Output length | Notes |
|---:|---:|---:|---:|---|
| 0.05 | _<...>_ | _<...>_ | _<...>_ | |
| 0.1 (default) | _<...>_ | _<...>_ | _<...>_ | |
| 0.5 | _<...>_ | _<...>_ | _<...>_ | |

_Interpret: where's the sweet spot for your data? Why? Does it match the deck's §3.3 prediction?_

Dự báo: Nếu chạy beta-sweep với $\beta=0.05$, model sẽ bám sát dữ liệu ưu tiên hơn, dẫn đến câu trả lời dài dòng hơn và có thể gặp lỗi lặp từ. Với $\beta=0.5$, model sẽ giữ tính chất của SFT nhiều hơn, ít thay đổi về mặt văn phong so với baseline. Giá trị 0.1 là "sweet spot" lý tưởng cho các model quy mô 3B-7B.

---

## 6. Personal reflection — single change that mattered most (≥ 150 words)

> Pick **one** decision you made during this lab — choosing β, choosing the data slice, choosing the judge model, choosing T4 vs BigGPU — and walk through:
>
> 1. What was the alternative you considered?
> 2. Why did you pick the one you did?
> 3. Did the result confirm or surprise you?
> 4. If you redid the lab tomorrow, what would you change?

Quyết định quan trọng nhất là việc chọn model **Qwen2.5-3B** trên **T4**. 
1. **Lựa chọn thay thế:** Tôi đã cân nhắc dùng Qwen2.5-7B để có năng lực suy luận mạnh hơn.
2. **Lý do chọn 3B:** DPO yêu cầu VRAM cao gấp đôi SFT do phải giữ model reference trong bộ nhớ. Trên T4 (16GB), chạy 7B với DPO rất dễ bị OOM ngay cả khi dùng Unsloth. Model 3B là lựa chọn cân bằng nhất, cho phép batch size 1 và gradient accumulation mà không làm gián đoạn quá trình train.
3. **Kết quả:** Kết quả cho thấy 3B vẫn đủ sức thể hiện sự khác biệt rõ rệt sau alignment.
4. **Thay đổi nếu làm lại:** Tôi sẽ tập trung vào việc làm sạch (curation) tập dữ liệu preference nhiều hơn thay vì tăng số lượng, vì DPO rất nhạy cảm với dữ liệu nhiễu.


---

## 7. Benchmark interpretation (≥ 150 words)

> **Paste `07-benchmark-comparison.png` here** (or link).

Score table from `data/eval/benchmark_results.json`:

| Benchmark | SFT-only | SFT+DPO | Δ |
|---|---:|---:|---:|
| IFEval | _<...>_ | _<...>_ | _<...>_ |
| GSM8K | _<...>_ | _<...>_ | _<...>_ |
| MMLU (sampled) | _<...>_ | _<...>_ | _<...>_ |
| AlpacaEval-lite | _<...>_ | _<...>_ | _<...>_ |

_Interpret the deltas. Which benchmark went up most? Did GSM8K or MATH regress (alignment tax — see deck §8.1)? Did MMLU stay flat (factual knowledge preserved) or drop (catastrophic forgetting)? Was AlpacaEval-lite win-rate consistent with NB4 judge results, or divergent? Which benchmark surprised you, and what does it tell you about whether DPO did the alignment work you wanted?_

Hiện tại tôi chưa thực hiện chạy Benchmark ở Stage 6 (NB 6) do giới hạn thời gian và tài nguyên trên T4. Tuy nhiên, qua đánh giá định tính ở NB 4, model cho thấy sự cải thiện rõ rệt về khả năng hỗ trợ (Helpfulness) và duy trì được tính an toàn (Safety). Tôi dự kiến IFEval sẽ có điểm số cao hơn nhờ việc model học được cách tuân thủ cấu trúc tốt hơn từ tập UltraFeedback. Các chỉ số về GSM8K hay MMLU có thể sẽ giảm nhẹ (alignment tax) nhưng không đáng kể đối với model quy mô 3B này.

---

## Bonus

- [ ] Đã làm β-sweep (rigor add-on +6)
- [ ] Đã push lên HuggingFace Hub (Submission Option B, +5)
- [ ] Đã release GGUF với multiple quantizations (+3)
- [ ] Đã link W&B run public (+2)
- [ ] Đã làm cross-judge comparison (+4)
- [ ] Đã làm `BONUS-CHALLENGE.md` provocation (ungraded — link `bonus/` folder)
- [ ] Pair work với: _<tên đồng đội nếu có>_

---

## Điều ngạc nhiên nhất khi làm lab này

_(Optional, 1–3 câu)_
