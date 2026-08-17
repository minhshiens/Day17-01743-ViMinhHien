# Báo Cáo Nộp Bài Lab 17 - Multi-Memory Agent với Zep

## 1. Ba câu hỏi lý thuyết chính

### Câu 1: Layer bộ nhớ quan trọng nhất
Trong bộ test này, **Long-term Memory (Zep Context Block)** là layer quan trọng nhất. Nó quyết định trực tiếp 4 case (E02, E03, E08, E09) và góp phần vào case E07. Ví dụ: **E08** yêu cầu truy xuất preference mới nhất (TypeScript/NestJS) cho dự án `BLUEBIRD-42`, loại bỏ thông tin cũ nhờ mốc thời gian `valid_at`/`invalid_at`.

### Câu 2: Trade-off Zep Managed vs Redis + Qdrant tự dựng
- **Zep Managed Context Block:** Tự động tổng hợp facts/episodes/summary, hỗ trợ graph search & temporal validity range mà không cần viết pipeline trích xuất entity (Nhược điểm: phụ thuộc network API, độ trễ ~6.9s).
- **Redis + Qdrant local:** Kiểm soát hoàn toàn dữ liệu, truy vấn cực nhanh (<1ms), offline-first nhưng đòi hỏi tự xây dựng pipeline trích xuất, tự giải quyết recency conflict và tự quản lý context budget thủ công.

### Câu 3: Guardrail chống Memory Poisoning
1. **Minimization & Consent:** Lọc bỏ PII (email, SĐT) bằng `minimize_pii` và bắt buộc opt-in trong `consent.json` trước khi ingest.
2. **Scoping & Heartbeat:** Tách biệt `user_id` tuyệt đối (tránh data leak như E09). Heartbeat script chỉ được de-duplicate/mark stale task, KHÔNG được tự thêm instruction hay nâng quyền trong durable memory.

---

## 2. Phân tích Benchmark (`reports/comparison.md`)

1. **Layer hit rate thấp nhất:** Cả 4 layer đều đạt **100% Hit Rate (11/11 PASS)** ở bản Student implementation.
2. **Query retrieve nhiều token nhất:** Case E08 (`long_term`, 139 tokens) và E07 (`mixed`, 110 tokens) do tổng hợp cả Context Block và Edges facts.
3. **Case E07 (mixed context):** Kết hợp **Long-term** (lấy preference `Python`) và **Semantic** (lấy quy tắc `Idempotency-Key`).
4. **Token reduction vs Evidence Hit Rate:** Baseline `no_memory` có reduction cao (81.8%) nhưng chỉ PASS 2/11 (Hit rate 18.2%). Memory-enabled đạt **100% Hit Rate** với reduction 14.2%, chứng minh reduction chỉ có giá trị khi giữ đủ evidence.

---

## 3. Phân tích E08 (Recency) & E10 (Compaction)

- **E08 (Recency Conflict):** Zep Context Block theo dõi mốc thời gian facts, tự động ưu tiên thông tin mới nhất (`TypeScript`/`NestJS`) đè lên preference cũ.
- **E10 (Compaction):** Khi giảm `max_recent_messages` xuống 4, Durable Notes trong Short-term Memory vẫn bảo toàn được constraint `REVIEW-DEADLINE-1600` và `Friday 16:00` dù hội thoại thô đã bị trượt khỏi sliding window.

---

## 4. Minh chứng kết quả (Screenshots)

- **Long-term Memory (E02, E03, E08, E09 PASS):** ![long_term](submission/long_term.png)
- **Episodic Memory (E04, E05 PASS):** ![episodic](submission/episodic.png)
- **Semantic Memory (E06, E11 PASS):** ![semantic](submission/semantic.png)
- **Privacy Drill (Forget & Verify):** ![privacy](submission/privacy.png)

Log xác nhận kiểm thử Privacy Drill:
```text
Zep user absent: True
Redis user keys remaining: 0
Shared semantic KB remains intact because it stores domain knowledge, not user PII.
```

