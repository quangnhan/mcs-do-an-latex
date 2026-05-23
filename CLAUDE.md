# CLAUDE.md — đồ án Thạc sĩ: Marketing Agent F&B

## Tổng quan đề tài

**Tên đề tài:** Phát triển và tối ưu hóa Marketing Agent cho SME ngành F&B dựa trên kỹ thuật Fine-tuning và Quy trình đánh giá đa lớp.

**Mục tiêu:**
1. Xây dựng AI Agent chuyên biệt sinh nội dung marketing Facebook cho SME F&B Việt Nam
2. Pipeline đánh giá chất lượng tự động đa lớp (thay thế bước kiểm duyệt thủ công)
3. Đánh giá so sánh các mô hình ngôn ngữ mã nguồn mở (Qwen, Gemma) trên tác vụ này

**Phạm vi:** Nội dung Facebook (bài đăng, không mở rộng sang TikTok hay nền tảng khác).

---

## Các repository liên quan

### 1. `D:\Github\mcs-do-an-latex` — đồ án LaTeX (repo này)
- `latex_new/` — source LaTeX chính
- `source_code/marketing_model_eval.ipynb` — notebook đánh giá (bản sao/mirror từ eval repo)
- `documents/13.4.2026_dan_y.md` — dàn ý đồ án chi tiết

### 2. `D:\Github\mcs-do-an` — Hệ thống Marketing Agent (source code thực tế)
Stack: FastAPI + LangGraph + PostgreSQL + RabbitMQ/Celery + MinIO + Langfuse + LiteLLM

**Cấu trúc chính:**
```
mcs-do-an/
├── orchestrator/          # FastAPI main API, SQLAlchemy models, Alembic migrations
│   └── alembic/versions/8b940ddcfee3_schema_v2_initial.py  # DDL thực tế
├── ai_agents/             # LangGraph agents + Celery workers
│   ├── agents/
│   │   ├── campaign_agent/     # Orchestrator (conversational gateway)
│   │   ├── content_planning_agent/
│   │   ├── content_agent/      # Sinh bài viết (template content_agent.md)
│   │   └── image_agent/        # Sinh ảnh → MinIO
│   ├── http_service/app.py     # POST /chat endpoint
│   └── config.py
├── evaluation_system/     # SEO scoring (10 criteria, 100 points) — dùng cho web content
└── docker-compose.yml     # RabbitMQ + MinIO services
```

**Database — 10 bảng thực tế:**
- Identity: `organizations`, `projects` (brand guidelines), `users`
- Knowledge: `products` (key_features[], campaign_objectives[], key_facts[])
- Chat: `chat_sessions`, `chat_messages` (role: user|assistant|system|tool, metadata JSONB)
- Pipeline: `campaign_runs` → `content_planning_jobs` → `content_generation_jobs` → `image_generation_jobs`

**Campaign lifecycle:**
1. `POST /chat` → CampaignAgent phân loại intent
2. `REQUEST_PLAN` → ContentPlanningAgent tạo ExecutionPlan → `awaiting_approval`
3. `APPROVE_PLAN` → ContentAgent + ImageAgent workers qua Celery queues → `done`
4. User review: `review_decision` ghi vào DB → làm golden samples cho lần sau

**Agents (LangGraph state machines):**
- **CampaignAgent**: States: `CHATTING → AWAITING_APPROVAL → DONE`, Intents: `CHAT|REQUEST_PLAN|APPROVE_PLAN|REJECT_PLAN`, MemorySaver cho session persistence, model: GPT-4o (T=0.7)
- **ContentPlanningAgent**: LangChain agent with TOOL_REGISTRY, model: GPT-5-mini (T=1.0)
- **ContentAgent**: Pipeline `build_prompt → generate → parse_output`, template injection 4 sections (KNOWLEDGE BASE, STYLE GUIDE, GOLDEN SAMPLES, LESSONS LEARNED), output JSON `{headline, body, hashtags}`, model: GPT-4o-mini (T=0.7)
- **ImageAgent**: gpt-image-1, lưu PNG lên MinIO, hỗ trợ `run()` và `edit()`

### 3. `D:\Github\mcs-do-an-eval-llm` — Thực nghiệm đánh giá mô hình
```
mcs-do-an-eval-llm/
├── marketing_model_eval.ipynb                      # Pipeline đánh giá chính
├── dataset/
│   ├── high_quality_mock_cases_100.json            # 100 test cases (4 danh mục)
│   └── high_quality_mock_cases_100.eval_summary.csv # Kết quả tổng hợp 9 models
└── results/runs/                                   # CSV từng lần chạy (9 files)
```

**Dataset:** 100 mock cases, 4 danh mục (25 mỗi loại): máy pha cà phê, điện thoại, bàn gập, skincare. Mỗi case: `case_id`, `input_title`, `seed_content`, `actual_output`.

**Pipeline đánh giá — 3 lớp (DeepEval framework):**

| Lớp | Công thức | Thành phần |
|-----|-----------|------------|
| Faithfulness | `0.5 × rule + 0.5 × llm` | rule: entity_presence_score (regex giá/spec/quote); llm: FaithfulnessMetric |
| Expansion | `0.9 × llm + 0.1 × length` | llm: GEval (diễn giải lợi ích); length: ratio output/seed (3x→1.0, 2x→0.7, 1.2x→0.4) |
| Vibe | `0.7 × hook + 0.3 × (0.5×md + 0.5×cta)` | hook: GEval on first paragraph; md: H1/H2/bullets; cta: keyword/emoji/URL trong 600 ký tự cuối |
| **Overall** | `(F + E + V) / 3` | Ngưỡng phê duyệt θ = 0.75 |

**Judge:** Azure OpenAI `md-gpt-5.4-mini` (giai đoạn chính) / OpenAI `gpt-5.4-mini` (giai đoạn pilot)

---

## Kết quả thực nghiệm — 9 mô hình

| Model | Faithfulness | Expansion | Vibe | **Overall** | Giai đoạn |
|-------|-------------|-----------|------|------------|-----------|
| qwen/qwen3-1.7b | 0.9981 | 0.8524 | **0.6031** | 0.8179 | Pilot (16/04, LM Studio, OpenAI judge) |
| qwen3.5:2b | 0.9952 | 0.8488 | 0.6759 | 0.8400 | Chính (20/04) |
| qwen3.5:9b | **0.9980** | 0.8389 | 0.6780 | 0.8383 | Chính (20/04) |
| **qwen3.5:27b** | 0.9950 | **0.8614** | 0.6724 | **0.8429** | Chính (20/04) |
| qwen3.6:35b | 0.9929 | 0.8470 | 0.6766 | 0.8388 | Chính (20/04) |
| gemma4:e2b | 0.9926 | 0.8524 | 0.6668 | 0.8373 | Chính (21/04) |
| gemma4:e4b | 0.9881 | 0.8254 | 0.6766 | 0.8300 | Chính (21/04) |
| gemma4:26b | 0.9942 | 0.8461 | **0.6787** | 0.8397 | Chính (21/04) |
| gemma4:31b | 0.9921 | 0.8092 | 0.6780 | 0.8264 | Chính (21/04) |

**Nhận xét chính:**
- Faithfulness đồng đều cao (~0.99) — mô hình nào cũng tốt khi có seed content đầy đủ
- Expansion phân hóa theo họ: Qwen 3.5 > Gemma4 > Qwen 3-1.7b
- Vibe thấp nhất (~0.67) trên TẤT CẢ mô hình → nhu cầu fine-tuning rõ ràng
- qwen3-1.7b có Vibe chỉ 0.60 — ngưỡng tham số tối thiểu để marketing hook đạt chất lượng
- qwen3.5:2b (2B) ≈ qwen3.6:35b (35B) về Overall → mô hình nhỏ đủ dùng nếu ≥2B

---

## Trạng thái các chương đồ án

| Chương | Tên | Trạng thái |
|--------|-----|-----------|
| 1 | Giới thiệu đề tài | ✅ Hoàn thành |
| 2 | Tìm hiểu và nghiên cứu thị trường | ✅ Hoàn thành |
| 3 | Cơ sở lý thuyết và công nghệ | ✅ Hoàn thành |
| 4 | Xây dựng hệ thống Marketing Agent và phân tích kết quả thực nghiệm | ✅ Hoàn thành (cập nhật 27/04/2026) |
| 5 | Kết luận và hướng phát triển | ⏳ Placeholder |
| — | Tài liệu tham khảo | ⏳ Cần bổ sung |

**File chương 4:** `latex_new/chuong_4_xay_dung_he_thong_va_phan_tich_ket_qua.tex`

**Nội dung chương 4 (5 sections):**
- 4.1 Thiết kế kiến trúc hệ thống (stack, 10 bảng DB, campaign lifecycle)
- 4.2 Hiện thực hóa các thành phần Agent (CampaignAgent, ContentPlanningAgent, ContentAgent, ImageAgent, FastAPI)
- 4.3 Pipeline đánh giá chất lượng đa lớp (Faithfulness, Expansion, Vibe + code snippets)
- 4.4 Kết quả thực nghiệm so sánh 9 mô hình
- 4.5 Kiểm định tương quan với đánh giá của con người (Pearson/Spearman, θ=0.75 → 86% accuracy)

---

## Quy ước LaTeX

**Compiler:** pdflatex (cần chạy 2 lần sau khi thay đổi cấu trúc để cập nhật TOC)

**Packages đã thêm vào main.tex:** `xcolor`, `listings`, `multirow` (ngoài các packages gốc)

**lstset:** basicstyle=small ttfamily, frame=single, numbers=left, gray background

**Decimal separator:** dùng dấu phẩy theo tiếng Việt: `0{,}843` (không phải `0.843`)

**Cross-references quan trọng:**
- `\ref{chap:thuc-nghiem}` — Chương 4
- `\ref{chap:thị-truong}` — Chương 2 (Approval Step)
- `\ref{chap:ly-thuyet}` — Chương 3

---

## Lưu ý quan trọng

- **Không có fine-tuning code** trong cả hai repo — thực nghiệm chỉ là *base model evaluation*. Nếu viết về fine-tuning, phải ghi rõ là đề xuất/kế hoạch, không phải kết quả thực nghiệm đã có.
- Dataset đánh giá là **mock** (cà phê, điện thoại, bàn, skincare) — không phải dữ liệu nhà hàng F&B thực tế. Hệ thống chính (`mcs-do-an`) dùng dữ liệu thực từ `products` table.
- Tên nền tảng: chỉ **Facebook**, không đề cập TikTok.
- LLM trong hệ thống production: GPT-4o / GPT-4o-mini / GPT-5-mini qua LiteLLM proxy (không phải local Qwen/Gemma).
- Kết quả tương quan (Pearson, Spearman, false positive/negative rates) là **số liệu đề xuất** cho thí nghiệm tương lai, chưa có dữ liệu thực tế.
