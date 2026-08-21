# Reflection — Lab 21

1. The most surprising result was that the fine-tune reached 0.970 target accuracy but still failed the overall gate because regression dropped by 0.2133. Narrow-task success was not enough.

2. The most time-consuming part was generating the four model comparisons on the T4. The controlled runs made it clear that model loading and generation, not only backpropagation, dominate the wall-clock time.

3. I no longer believe that lower training loss automatically means a better model. The attn_only run had lower loss than correct but only tied it on the target metric, while wrong_lr showed that the learning-rate scale can completely change the outcome.

4. I used an AI assistant to inspect the pipeline, interpret the artifacts, and organize the report. It was wrong to assume that a successful target score meant deployment was safe; the regression gate and qualitative examples were necessary checks.

5. For a real customer, my first step would be to freeze an untouched evaluation set and define both task success and regression-safety gates before training.
