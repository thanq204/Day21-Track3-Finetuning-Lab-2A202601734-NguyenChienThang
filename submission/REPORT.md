# Lab 21 — Evaluation Report

**Name:** Nguyen Chien Thang<br>
**Student ID:** 2A202601734<br>
**Date:** 2026-08-21<br>
**Tier:** T4<br>
**Base model:** unsloth/Qwen3.5-4B<br>
**GPU:** Google Colab T4, approximately 16 GB

## 1. Setup

| Item | Value |
|---|---|
| Dataset | 250 Vietnamese customer-support tickets mapped to four-field JSON triage labels |
| Train / validation | 225 / 25, split with seed 42 |
| max_length | 1024; measured p95 was 98 tokens and the p95-based suggestion was 256 |
| MASK_MODE | assistant-only |
| Epochs / max_steps | 2 epochs / 30 optimizer steps |

The template preserved the reasoning block: `template_check.json` reports `ok=true`, with both the opening tag and reasoning body present. The T4 tier kept the configured 1024-token limit even though the shipped corpus is much shorter than that. This was recorded rather than silently pretending that the tier setting matched the measured suggestion.

## 2. Mask proof

| Measure | Value |
|---|---:|
| supervised_fraction | 0.4149 |
| Answer in loss | true |
| Question excluded from loss | true |

The supervised portion begins with the assistant response and includes the EOS marker:

```
</think>

{"intent": "doi_tra", "urgency": "trung_binh",
"product": "balo laptop", "sentiment": "trung_tinh"}<|im_end|>
```

The prompt, system message, and user ticket appear in the masked preview instead. This is the intended assistant-only behavior: the model learns to produce the answer, not to reproduce the input question.

## 3. Three baselines

| Run | target | regression | format | latency (ms) |
|---|---:|---:|---:|---:|
| (a) base + naive prompt | 0.000 | 0.7578 | 0.000 | 3185.2 |
| (b) base + optimized prompt | 0.765 | 0.7578 | 1.000 | 1050.1 |
| (c) LoRA fine-tune | 0.970 | 0.5444 | 1.000 | 1385.7 |

Baseline (b) genuinely beats (a): target rises from 0.000 to 0.765 and format rises from 0.000 to 1.000. I did not modify the shipped optimized prompt; its recorded SHA matches the expected prompt. This confirms that the fine-tune was compared against a strong prompted baseline rather than an intentionally weak baseline.

## 4. Misconfiguration autopsy

| Run | Position | r | Trainable | LR | Train loss | Target | Steps | VRAM GB |
|---|---|---:|---:|---:|---:|---:|---:|---:|
| correct | text-linear | 16 | 32,464,896 | 1e-4 | 0.6266 | 0.970 | 30 | 12.01 |
| attn_only | q,v | 283 matched | 32,456,704 | 1e-4 | 0.5387 | 0.970 | 30 | 12.02 |
| wrong_lr | text-linear | 16 | 32,464,896 | 1e-5 | 1.5704 | 0.000 | 30 | 12.01 |
| qlora | text-linear | 16 | 32,464,896 | 1e-4 | 0.7058 | 0.940 | 30 | 7.09 |

**4.1 — Adapter position versus rank.** The attention-only run used r=283 so that its trainable parameter count was almost identical to correct: 32,456,704 versus 32,464,896. The difference is about 0.025%, comfortably inside the 5% fairness rule. Both runs reached target 0.970, while attn_only had the lower training loss, 0.5387 versus 0.6266. Therefore higher rank alone did not beat the all-linear placement on this task; the target metric was tied, showing that placement and task structure mattered more than simply increasing rank.

**4.2 — Wrong learning rate.** wrong_lr changed only the learning rate, from 1e-4 to 1e-5. Its final training loss was 1.5704, much higher than correct at 0.6266, and its target and format scores both collapsed to zero. If I looked only at architecture or parameter count, I might incorrectly blame adapter placement. The controlled comparison shows that the learning-rate scale is decisive for this LoRA recipe.

**4.3 — QLoRA.** QLoRA reduced measured peak VRAM from 12.01 GB to 7.09 GB, a saving of 4.92 GB or approximately 41.0%. It did not preserve the same quality: target fell from 0.970 to 0.940, format stayed at 1.000, and latency increased from 1385.7 ms to 1760.6 ms. Its training loss was also higher than correct, 0.7058 versus 0.6266. These measurements support caution about using QLoRA for this model family when ordinary LoRA fits in the available T4 memory.

## 5. Verdict

**Regression gate: FAILED**<br>
`target_delta = +0.2050` · `regression_delta = -0.2133` · `valid_trace_rate = 0.0000`

The fine-tuned model clearly improved the target task relative to the strong optimized-prompt baseline. Target accuracy increased from 0.765 to 0.970, a gain of 0.205. JSON format also remained perfect at 1.000, so the result is not a formatting failure. However, the regression score decreased from 0.7578 to 0.5444, a loss of 0.2133. The allowed regression tolerance is only 0.020, so the gate correctly failed. This is a useful result rather than a failed experiment: the adapter memorized or specialized strongly for the 250-ticket triage task while damaging general capability measured by the 15 regression prompts. The correct conclusion is not to deploy this adapter unchanged. A next experiment should add a small replay mixture of general data, as suggested by the gate, and then repeat the same frozen evaluation. The valid trace rate was 0.0; the template preserved reasoning, but the shipped answers contained no non-empty reasoning traces, so this value does not prove that the template deleted reasoning.

## 6. Qualitative examples

| # | Ticket | Ground truth | Optimized prompt | Fine-tune | Analysis |
|---|---|---|---|---|---|
| 1 | Wireless mouse, wants return urgently | doi_tra / cao / chuột không dây / tich_cuc | Predicts hoan_tien; other fields correct | All 4 fields correct | FT fixes intent; win |
| 2 | Phone case refund, customer is upset | hoan_tien / trung_binh / ốp lưng điện thoại / tieu_cuc | Predicts urgency cao; other fields correct | All 4 fields correct | FT fixes urgency; win |
| 3 | Thermos refund not received yet | hoan_tien / thap / bình giữ nhiệt / tich_cuc | Predicts urgency trung_binh | Predicts urgency trung_binh; score 0.75 | FT still misses urgency; losing case |
| 4 | Air fryer missing an accessory | san_pham_loi / thap / nồi chiên không dầu / trung_tinh | Predicts hoan_tien and urgency cao | Fixes intent but predicts urgency trung_binh; score 0.75 | FT improves one field but remains wrong |
| 5 | Windbreaker is defective | san_pham_loi / thap / áo khoác gió / tich_cuc | Predicts urgency trung_binh; other fields correct | Predicts urgency trung_binh; score 0.75 | FT does not fix the urgency pattern; losing case |

The losing cases share a pattern: urgency is inferred as trung_binh even when the label is thap. The fine-tune learned intent and product mapping well, but it did not reliably separate low urgency from ordinary medium urgency. This is consistent with the large regression drop: specialization improved the main label mapping while reducing broader calibration.

## 7. Conclusion and lessons

I would not deploy this fine-tuned model unchanged. It achieved a strong target score of 0.970 and perfect JSON format, but it failed the safety-oriented regression gate by dropping general capability from 0.7578 to 0.5444. That trade-off is too large for a production customer-support system because a model that classifies the narrow ticket format correctly can still become less reliable on unrelated instructions. The experiment also shows that the failure is not caused by the loss mask: the answer was supervised, the question was masked, and the template check passed. The most important practical improvement is to add a small replay set of general-purpose examples during training, then compare against the same frozen baselines again. I would also test a shorter max_length based on the measured p95, while keeping the comparison controlled. Adapter placement and learning rate both matter, but the controlled autopsy shows that learning rate caused the clearest collapse, while matched attention-only rank tied the correct target score. QLoRA saved VRAM but cost quality and latency. Therefore the current adapter is valuable as an experiment, not as a deployable checkpoint.

Three specific lessons:

1. A lower training loss does not guarantee a better target score; attn_only had lower loss but only tied correct.
2. A correct loss mask must be proven from decoded supervised tokens, not assumed from a trainer flag.
3. Regression evaluation is necessary because a high narrow-task score can hide capability damage.

If I had two more hours, I would train with 1–5% replay data, rerun the frozen regression gate, and test max_length=256 with the same 30-step budget.

## Appendix

- Core run: NB1 through NB5 completed on T4.
- NB6 merge and hot-swap: not run.
- Custom dataset: not used.
- Reasoning-trace collapse bonus: not run.
- Rank sweep bonus: not run.
- Hugging Face Hub upload: not run.
