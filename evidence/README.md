# Evidence — Day 22: LLMOps Prompt Versioning

**Học viên:** Nguyễn Như Tài — 2A202602976
**Provider:** OpenAI `gpt-4o-mini` + `text-embedding-3-small` · **LangSmith project:** `day22-lab`

| File | Nội dung |
|------|----------|
| `01_langsmith_traces.png` | ≥ 50 traces `rag-query` (Bước 1) |
| `02_prompt_hub.png` | 2 prompt `nguyen-nhu-tai-rag-prompt-v1` / `-v2` trên Prompt Hub |
| `02_ab_routing_log.txt` | Log 50 câu A/B routing có nhãn `[prompt-v1]` / `[prompt-v2]` (V1=19, V2=31) |
| `03_ragas_scores.png` | Bảng so sánh V1 vs V2 trên terminal |
| `03_ragas_report.json` | Bản sao `data/ragas_report.json` |
| `03_ragas_run_log.txt` | Log đầy đủ của lần chạy RAGAS |
| `04_pii_demo_log.txt` / `04_json_demo_log.txt` | Log demo Guardrails (6 case PII, 5 case JSON) |

## Kết quả RAGAS (50 QA pairs, k=3, chunk 500/50)

| Metric | V1 (ngắn gọn) | V2 (có cấu trúc) | Winner |
|--------|------:|------:|:------:|
| faithfulness | **0.9576** | 0.8317 | V1 |
| answer_relevancy | **0.9137** | 0.8803 | V1 |
| context_recall | 1.0000 | 1.0000 | hoà |
| context_precision | 0.9450 | 0.9450 | hoà |

Mục tiêu faithfulness ≥ 0.8 đạt ở **cả hai** phiên bản.

## Phân tích V1 vs V2

- **context_recall và context_precision bằng nhau** ở hai phiên bản. Điều này đúng như kỳ vọng: hai chỉ số này chỉ đo chất lượng *retriever*, mà V1 và V2 dùng chung FAISS index, chung `k=3`, chỉ khác system prompt. Recall = 1.0 cho thấy 3 chunk truy xuất luôn chứa đủ thông tin của đáp án chuẩn.
- **V1 thắng faithfulness (+0.126).** V1 yêu cầu trả lời 2–4 câu, chỉ dùng context. Câu trả lời ngắn chứa ít "claim", gần như mọi claim đều trích được từ context. V2 yêu cầu 3–5 câu và cấu trúc "định nghĩa → chi tiết → ý nghĩa thực tiễn". Để lấp đủ khung đó, model hay thêm câu diễn giải hoặc kiến thức chung (ví dụ ứng dụng thực tế) không có trong context. RAGAS tính các câu này là claim không được hỗ trợ, nên faithfulness giảm.
- **V1 thắng answer_relevancy (+0.033).** `answer_relevancy` sinh ngược câu hỏi từ câu trả lời rồi so với câu hỏi gốc. Câu trả lời ngắn, đi thẳng vào ý chính bám sát câu hỏi hơn. Phần mở rộng ở V2 làm loãng trọng tâm.
- **Kết luận:** với kho tài liệu ngắn, có tính định nghĩa như lab này, prompt ngắn gọn (V1) là lựa chọn tốt hơn cho production. Nếu muốn giữ phong cách chuyên gia của V2, nên bỏ yêu cầu "ý nghĩa thực tiễn" và thêm ràng buộc "không thêm thông tin ngoài context" mạnh hơn, sau đó chạy lại RAGAS để kiểm chứng. Đây đúng là vòng lặp prompt versioning: sửa prompt → push phiên bản mới lên Hub → đánh giá lại.

## Ghi chú Guardrails

- `PIIDetector` và `JSONFormatter` tự viết (không dùng Guardrails Hub), `on_fail=OnFailAction.FIX` truyền vào constructor. Validator trả về `FailResult(fix_value=...)` để FIX thay được output.
- Hạn chế đã biết: regex PHONE bắt đầu bằng `\b` nên với `(555) 867-5309` output là `([PHONE_REDACTED]`, sót dấu `(`, nhưng toàn bộ chữ số đã được che.
