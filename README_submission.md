# Báo Cáo Thực Hành Lab 17 — Multi-Memory Agent

## 1. Câu hỏi thực hành cốt lõi

1. **Layer quan trọng nhất:** **Long-term Memory** quan trọng nhất vì chiếm 4/11 ca (**E02, E03, E08, E09**), giải quyết: preference qua session (E02: Python), open-loop task (E03: `benchmark report`, `16:00`), scoped recency (E08: `BLUEBIRD-42` dùng TypeScript/NestJS), và cách ly người dùng (E09: Lan không thấy `ORCHID-27` của Minh).
2. **Trade-off Zep Context Block vs Redis + Qdrant:** Zep tự động trích xuất Graph/Facts, tính quan hệ thời gian và sinh Context Block theo thread; nhược điểm là phụ thuộc Cloud SaaS, độ trễ mạng và reasoning hộp đen. Redis + Qdrant kiểm soát toàn bộ data local, độ trễ thấp, offline; nhược điểm là phải tự code pipeline trích xuất, deduplication, conflict resolution và sliding context.
3. **Guardrail chống Memory Poisoning:** Phân vùng User Namespace + Consent Registry (`consent.json`); thực thi Data Minimization/PII Redaction trước khi ghi; quy định Heartbeat chỉ dọn dẹp (de-dup, stale check), tuyệt đối không tự cấp quyền hay nạp prompt mới; áp dụng Human review cho cập nhật nhạy cảm.

## 2. Phân tích kết quả Benchmark

1. **Layer hit rate thấp nhất:** No-memory baseline đạt 0% ở Long-term, Episodic, Semantic do thiếu durable memory. Memory-enabled đạt 100% (11/11 PASS); trong đó Episodic/Semantic nhạy cảm nhất, cần `scope="episodes"` và `cap_query` để giữ literal markers (`PAYMENT-RULE-3`, `CONN-POOL-FIRST`).
2. **Query lấy nhiều token nhất:** Case **E07 (Mixed)** do truy xuất cả Long-term và Semantic.
3. **Case Mixed (E07):** Kết hợp Long-term Memory (`Python`) và Semantic Memory (`Idempotency-Key`).
4. **Token Reduction vs Hit Rate:** Memory-enabled giảm ~75-85% token so với nạp transcript thô. No-memory có reduction cao chỉ vì không nạp gì (0 token), dẫn đến fail mọi câu hỏi cross-session.

## 3. Phân tích Recency (E08) & Compaction (E10)

- **E08 (Recency):** Khi đổi backend sang `TypeScript`/`NestJS` cho `BLUEBIRD-42`, Zep gán validity và scope riêng, ưu tiên fact mới cho BLUEBIRD-42 mà bảo toàn preference `Python` cho `ORCHID-27`.
- **E10 (Compaction):** Sliding window trích xuất constraint (`REVIEW-DEADLINE-1600`, `Friday`, `16:00`) vào `DURABLE_NOTES`, không mất thông tin khi tin nhắn cũ bị evict.
