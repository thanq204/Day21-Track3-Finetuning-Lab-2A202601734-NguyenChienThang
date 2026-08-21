# API keys cần cho Lab 21

## Bắt buộc

Không cần API key nào.

- Model/tokenizer được tải trực tiếp từ Hugging Face và đang là public.
- Dataset đã nằm sẵn trong repository.
- Không dùng OpenAI API, Anthropic API, Weights & Biases hay dịch vụ trả phí.
- Chỉ cần đăng nhập tài khoản Google để mở Colab và chọn T4 GPU; đây không phải API key.

## Tùy chọn

`HF_TOKEN` (Hugging Face User Access Token) chỉ cần nếu Hugging Face báo rate limit hoặc yêu cầu đăng nhập khi tải model.

Tạo token loại **Read** tại <https://huggingface.co/settings/tokens>, rồi trong Colab mở **Secrets** (biểu tượng chiếc chìa khóa) và thêm:

```text
Name: HF_TOKEN
Value: hf_...
```

Không dán token trực tiếp vào notebook và không commit token vào Git.

## Cấu hình chạy

- Runtime: **T4 GPU**.
- Lần chạy thử nhanh: đặt `EVAL_LIMIT=8` trong ô pipeline.
- Lần chạy lấy kết quả nộp: để `EVAL_LIMIT` trống; không cần thêm key nào.
