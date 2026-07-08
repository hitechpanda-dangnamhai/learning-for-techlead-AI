# Chain gối đầu — CAPSTONE: Tổng hợp toàn khóa Vector Database

> Nén theo chain gối đầu: đuôi node trước = đầu node sau. Thuật ngữ giữ nguyên tiếng Anh,
> giải thích tiếng Việt ngắn. Bảng myth-busting / khung quyết định / tổng kết giữ dạng bảng.
> Đây là quyển 10 (capstone): hợp nhất 10 chủ đề + đính chính lần cuối (train LLM / giảm bias /
> taxonomy 5 loại) + capstone system design + "chia ba" kiến trúc 2026.

---

## MẠCH CHÍNH — sợi chỉ đỏ của cả khóa
database truyền thống giỏi exact match ("tìm id=5") => nhưng thế giới AI cần tìm theo ý nghĩa ("tìm thứ *giống* cái này") => nên biến mọi thứ (text/ảnh/audio/hành vi) thành embedding => embedding là một điểm trong không gian nhiều chiều, thứ giống nhau nằm gần nhau => nhưng so *tất cả* thì chậm O(N) => nên dùng ANN index như HNSW để tìm gần đúng mà cực nhanh O(log N) => HNSW đánh đổi recall lấy tốc độ => đo "gần" bằng distance metric (cosine/L2) => động tác tìm k điểm gần nhất gọi là similarity search => trên nền đó xây ứng dụng (image search, recommendation) và RAG => RAG cho LLM tra cứu tài liệu lúc trả lời để bớt bịa => dữ liệu đổi nên cần CRUD (add/update/upsert/delete) => và mọi thứ phải chạy ở scale

## CHAIN — 3 lỗi bản wrap-up trực tiếp lặp lại (rẽ, đính chính)
lỗi #1: "train LLMs to understand human data" => SAI (nghiêm trọng nhất) — vector DB/RAG KHÔNG train LLM => RAG hoạt động ở inference-time: tra cứu tài liệu → nhét vào prompt → LLM đọc rồi trả lời => nên trọng số model không đổi ("đưa sách vào phòng thi" ≠ "học thuộc lại giáo trình") => lỗi #2: "reducing or *eliminating* bias and inaccuracy" => gây hiểu nhầm — RAG giảm hallucination (tỷ lệ thuận retrieval quality) nhưng không loại bỏ hết => và RAG KHÔNG giảm bias, nó kế thừa bias của corpus, thậm chí khuếch đại vì "có vẻ có căn cứ" => giảm bias là bài toán data governance, không phải đặc tính vector DB => lỗi #3: "5 loại: in-memory/disk-based/distributed/graph-based/time-based" => taxonomy lỏng và lỗi thời => phân loại staff dùng: theo index algorithm (HNSW/IVF/PQ/LSH/DiskANN), theo deployment (dedicated/extension/embedded), theo managed vs self-host => và nhớ: FAISS/Annoy là library không phải "distributed database"; "graph-based DB" (Neo4j) ≠ "graph-based *index*" (HNSW)

## CHAIN — Pipeline Indexing (offline) (rẽ)
pipeline chia hai giai đoạn: indexing (offline) và serving (online) => indexing bắt đầu từ dữ liệu thô (text/ảnh) => dữ liệu thô → CHUNK (cắt nhỏ) => chunk → EMBED bằng model => embed → nạp vào COLLECTION kèm metadata => trong collection dựng index HNSW/IVF với metric cosine/L2

## CHAIN — Pipeline Serving (online) (rẽ)
serving bắt đầu từ query của user => query → EMBED (bằng đúng EF của collection) => embed → SIMILARITY SEARCH (ANN) qua HNSW O(log N) => rồi [FILTER where] theo metadata => rồi [RE-RANK] bằng cross-encoder => kết quả rẽ vào hai nhánh: App (search/recommendation) hoặc RAG (augment + LLM) => song song, vòng đời dữ liệu là ADD / UPDATE / UPSERT / DELETE + rebuild/compaction

## CHAIN — Big-O tổng hợp cả khóa (rẽ)
brute-force là O(N·d) => HNSW search là O(log N) => add/upsert = embed (đắt) + chèn O(log N) => update document = re-embed + tombstone+insert (+ re-index nền) => delete = O(1) tombstone, RAM giảm sau compaction O(N) => get() tất cả = O(N) (tránh ở production) => còn edit distance = O(m·n), cơ chế khác hẳn

## CHAIN — Capstone system design 100M tài liệu (rẽ, khung staff)
đề: semantic knowledge assistant cho doanh nghiệp, 100M tài liệu đa ngôn ngữ, đa tenant, trả lời có trích nguồn, tôn trọng phân quyền & GDPR, p99 < 200ms => bước 1 làm rõ: đây là RAG (không fine-tune), recall mục tiêu? cập nhật thường xuyên? SLA xoá GDPR? => bước 2 indexing: tài liệu → chunk (semantic) → embed (model multilingual nhất quán, vd BGE-M3 self-host cho dữ liệu nhạy cảm) → collection theo tenant (cách ly) + metadata (acl, source, updated_at) => bước 3 index & metric: 100M → HNSW (hoặc IVF-PQ nếu RAM căng), space="cosine", normalize => bước 4 serving: query → embed (cùng EF!) → hybrid retrieve (vector + BM25) top-100 + where filter phân quyền → re-rank cross-encoder top-5 → threshold loại rác → augment prompt (có nguồn) → LLM generate → trả lời kèm citation => bước 5 scale: shard + replicate, tách compute/storage, quantization (100M không nhét 1 máy) => bước 6 CRUD & GDPR: upsert idempotent, đổi model = re-embed blue-green, xoá user = delete(where={user}) → soft-delete → hard purge trong SLA (cả WAL/backup), monitor tombstone => bước 7 monitoring: recall@k + faithfulness (RAGAS) + p99 + business metric (CTR/satisfaction), "hỏng âm thầm" → giám sát recall => bước 8 kết bằng trade-off (dedicated vs pgvector vs Atlas, hybrid, threshold chặt/lỏng, managed vs self-host)

## CHAIN — "Chia ba" kiến trúc 2026: Dedicated (rẽ, next steps)
kiến trúc vector 2026 chia ba nhánh => nhánh 1 là dedicated vector DB (Pinecone, Qdrant, Weaviate, Milvus) => dedicated cho latency similarity tốt nhất và scale tỷ vector => nhưng cái giá là phải đồng bộ với store gốc (dual-write, eventual consistency)

## CHAIN — "Chia ba": NoSQL + vector (rẽ)
nhánh 2 là NoSQL + vector (MongoDB Atlas Vector Search, Cassandra với SAI/ANN) => ở đây vector *sống cạnh* document, cùng security/backup => nó HNSW-based, hỗ trợ cả ANN + ENN => lưu ý: Atlas Vector Search là tính năng của Atlas (self-hosted MongoDB không có)

## CHAIN — "Chia ba": Relational + vector / pgvector (rẽ)
nhánh 3 là relational + vector (PostgreSQL + pgvector) => ở đây vector *sống cạnh* dữ liệu quan hệ => nên cùng transaction, cùng SQL `WHERE`, cùng backup, không dual-write => pgvector 0.8+ (2025-26) thêm HNSW + iterative index scans => iterative index scans nhanh hơn nhiều cho filtered query => nhờ đó thu hẹp khoảng cách latency với dedicated => và chạy được mọi nơi Postgres chạy (Supabase, Neon, RDS) => cái giá: bạn tự tune m/ef_construction/ef_search => và extension hụt hơi khi > 50–100M vector (lúc đó chuyển dedicated)

---

## BẢNG — MYTH-BUSTING hợp nhất toàn khóa ("đừng nói câu này trong phỏng vấn")
| # | Câu hay nói (SAI/lỗi thời) | Sự thật | Quyển |
|---|---|---|---|
| 1 | "Train LLM bằng vector DB" | RAG là **inference-time**, không train | Q3 |
| 2 | "Vector DB giảm/loại bias" | kế thừa bias corpus; giảm *hallucination* chứ không bias | Q3 |
| 3 | "5 loại: in-memory/disk/distributed/graph/time" | phân theo **index + deployment + managed** | Q1 |
| 4 | "FAISS là distributed database" | FAISS là **library** 1 máy; Annoy/ScaNN cũng là library | Q1 |
| 5 | "PostGIS cho vector" | PostGIS là **geospatial**; extension đúng là **pgvector** | Q1 |
| 6 | "Nhà hàng gần tôi = vector DB" | đó là **geospatial 2D (R-tree)**; "vector" GIS ≠ "vector" embedding | Q2 |
| 7 | "Recommendation = một cú search" | **2 tầng: retrieval (vector) → ranking (model nặng)** | Q2 |
| 8 | "Cosine similarity ∈ [0,1]" | **[−1, 1]**; chỉ [0,1] khi vector không âm | Q4 |
| 9 | "Nearest neighbor cải thiện ảnh (pixelating)" | đó là NN **interpolation** (ảnh), khác NN **search** | Q4 |
| 10 | "Nguồn embedding: TensorFlow Hub / GPT-3" | 2026: OpenAI text-embedding-3, Cohere v4, BGE-M3, Voyage... | Q4 |
| 11 | "distances = similarity scores" | Chroma trả **distance** (thấp=giống); score nghịch đảo | Q6,7 |
| 12 | "score 1.264 là cosine similarity" | là **L2 distance** (mặc định Chroma); cosine ≤ 1 | Q7 |
| 13 | "edit distance = similarity search của vector DB" | edit distance = **string/syntactic**; embedding = **semantic** | Q6 |
| 14 | "genomics similarity = vector DB" | cổ điển: **BLAST/alignment**; hiện đại: **ESM embedding** | Q6 |
| 15 | "`embeddings: ef`" (tạo collection) | property đúng: **`embeddingFunction`** | Q5,7,9 |
| 16 | "`n: 3`" (query) | đúng: **`nResults`** / `n_results` | Q6,7 |
| 17 | "`updateDocument(name,...)` / `client.collections.delete`" | **API bịa**; đúng: `collection.update/delete`, `client.delete_collection` | Q8 |
| 18 | "delete xoá vector ngay, giải phóng RAM" | **soft-delete/tombstone** + compaction; RAM giảm sau | Q8 |
| 19 | "collection = SQL table" | không schema/JOIN/ACID; là kho vector có nhãn | Q5 |
| 20 | "Chroma scale vô tận, luôn tối ưu" | mạnh ở prototype/corpus vừa; billion-scale → Milvus | Q1,9 |
| 21 | "generate embedding thủ công rồi add" | **auto-embed** từ documents (an toàn hơn) | Q5,7,9 |
| 22 | "similarity search là exact" | là **ANN (approximate)**, đánh đổi recall↔tốc độ | Q6 |

## BẢNG — Khung quyết định staff (gộp mọi trade-off của khóa)
| Câu hỏi | Nếu... | Chọn |
|---|---|---|
| Exact hay semantic? | cần khớp *chính xác* (id/tên) | SQL/keyword/edit distance |
|  | cần giống *ý nghĩa* | vector DB + embedding |
| Metric? | text embedding (đã normalize) | cosine ≈ L2 ≈ ip → chọn cái nào cũng được |
| Index? | recall/latency cao, in-memory | **HNSW** |
|  | billion-scale, RAM là nút thắt | **IVF-PQ / DiskANN / SPANN** |
| Store? | vector là *core*, scale lớn | dedicated (Pinecone/Qdrant/Milvus) |
|  | <10–50M, đã có Postgres | **pgvector** |
|  | prototype, corpus vừa | **ChromaDB** |
| Thuộc tính rõ ràng (màu/giá)? | — | **metadata filter**, không phó mặc embedding |
| Có tên riêng/ID trong query? | — | **hybrid** (vector + BM25) |
| Mutation nhiều (update/delete)? | — | lịch **compaction/rebuild**; upsert idempotent |

## BẢNG — Khung chọn store 2026 ("chia ba")
| Tình huống | Chọn | Vì sao |
|---|---|---|
| Vector là *core*, scale tỷ | **dedicated** (Pinecone/Qdrant/Milvus/Weaviate) | latency tốt nhất; giá: dual-write đồng bộ |
| Đã chạy Postgres, cần transaction/SQL/JOIN, < ~50M | **pgvector** (~70% app RAG) | cùng transaction/WHERE/backup, không dual-write; tự tune m/ef |
| Đã chạy MongoDB Atlas | **Atlas Vector Search** | khỏi thêm hệ mới; HNSW-based, ANN+ENN (chỉ Atlas, không self-host) |
| Vector đổi realtime & cần searchable ngay | **traditional DB + vector extension** | không dual-write/không lag đồng bộ |
| Prototype/dev | **ChromaDB** | dev-experience tốt, corpus vừa |

## BẢNG — Tổng kết 10 quyển
| Quyển | Chủ đề | Câu cốt lõi |
|---|---|---|
| 1 | Loại vector DB | brute-force O(N) → HNSW O(log N), recall↔tốc độ; phân loại theo index+deployment |
| 2 | Ứng dụng | embed mọi thứ → similarity search; recsys = retrieval→ranking; "vector" GIS ≠ embedding |
| 3 | ChromaDB & RAG | RAG = mở sách *inference-time*; vá cutoff/private-data, không vá bias |
| 4 | Embeddings & metrics | cosine ∈[−1,1], đo hướng; normalize → cosine≡L2 |
| 5 | Collections | collection ≠ SQL table; EF là contract |
| 6 | Similarity search | semantic (embedding) ≠ syntactic (edit distance); search = metric + HNSW; cần threshold |
| 7 | Case study florist | "score">1 là L2 distance; màu = filter, phong cách = embedding |
| 8 | Update/delete | soft-delete/tombstone/compaction; upsert idempotent; GDPR là quy trình |
| 9 | Quản lý collections | API CRUD đúng; list/dọn/naming; get() all là O(N); tránh sprawl |
| **10** | **Capstone** | **hợp nhất + myth-busting + chọn dedicated/pgvector/Atlas + capstone design** |

---

## TỪ KHÓA MỒI
- Mạch chính: **embed → index → search → RAG → CRUD → scale**
- 3 lỗi wrap-up: **train / bias / 5 loại**
- Pipeline indexing: **chunk → embed → collection**
- Pipeline serving: **embed → ANN → filter → rerank**
- Big-O tổng hợp: **O(N·d) → O(log N)**
- Capstone design: **100M, p99<200ms, GDPR**
- Chia ba — dedicated: **latency tốt nhất, dual-write**
- Chia ba — NoSQL: **Atlas Vector Search (chỉ Atlas)**
- Chia ba — pgvector: **0.8+ iterative scans, <50M**

## ĐÃ PHỦ
- **Sợi chỉ đỏ:** exact match → tìm theo ý nghĩa → embedding → O(N) chậm → ANN/HNSW O(log N) (recall↔tốc độ) → distance metric → similarity search → app + RAG → CRUD → scale ✓
- **3 lỗi wrap-up trực tiếp:** (#1) train LLM SAI = inference-time weights không đổi; (#2) "eliminating bias" gây hiểu nhầm = giảm hallucination không giảm bias (data governance); (#3) taxonomy 5 loại lỏng = phân theo index+deployment+managed, FAISS/Annoy là library, graph-DB ≠ graph-index ✓
- **Bảng myth-busting 22 dòng** (train LLM, bias, 5 loại, FAISS distributed, PostGIS, nhà hàng gần tôi/R-tree, recsys 2 tầng, cosine [−1,1], NN interpolation, nguồn embedding 2026, distances vs scores, L2 1.264, edit distance semantic, genomics BLAST/ESM, embeddingFunction, nResults, API bịa, soft-delete RAM, collection≠table, Chroma scale, auto-embed, ANN không exact) ✓
- **Pipeline indexing:** dữ liệu thô → chunk → embed → collection + metadata, index HNSW/IVF metric cosine/L2 ✓
- **Pipeline serving:** query → embed (cùng EF) → similarity search ANN O(log N) → filter where → re-rank cross-encoder → App/RAG; vòng đời add/update/upsert/delete + rebuild/compaction ✓
- **Bảng khung quyết định staff:** exact/semantic, metric, index (HNSW vs IVF-PQ/DiskANN/SPANN), store (dedicated/pgvector/Chroma), metadata filter, hybrid, mutation→compaction ✓
- **Big-O tổng hợp:** brute-force O(N·d), HNSW O(log N), add/upsert, update doc, delete O(1) tombstone + compaction O(N), get() all O(N), edit distance O(m·n) ✓
- **Capstone system design 100M:** làm rõ (RAG/recall/GDPR) → indexing (chunk/embed BGE-M3/collection theo tenant+acl) → HNSW/IVF-PQ cosine normalize → serving hybrid+filter phân quyền+rerank+threshold+augment+citation → scale shard/replicate/quantization → CRUD upsert/re-embed blue-green/GDPR soft→hard purge → monitoring recall@k/RAGAS/p99/business → trade-off ✓
- **"Chia ba" kiến trúc 2026:** dedicated (Pinecone/Qdrant/Weaviate/Milvus — latency tốt nhất, tỷ vector, dual-write); NoSQL+vector (Atlas Vector Search/Cassandra SAI — vector cạnh document, HNSW, ANN+ENN, chỉ Atlas); relational+vector (pgvector — cùng transaction/SQL/backup, không dual-write, 0.8+ HNSW+iterative index scans, Supabase/Neon/RDS, tự tune m/ef, hụt hơi >50-100M) ✓
- **Bảng khung chọn store 2026:** core/tỷ→dedicated, Postgres/<50M→pgvector (~70% RAG), Mongo→Atlas, realtime→extension, prototype→Chroma ✓
- **Keywords toàn khóa** (recap): embedding, ANN/HNSW, IVF/PQ/DiskANN/SPANN (phân cụm/nén/disk billion-scale), recall (% đúng top-k), distance metric, cosine distance, collection/EF contract, similarity search, hybrid search, re-ranking, RAG, hallucination/faithfulness, CRUD/upsert, soft-delete/tombstone/compaction, pgvector/Atlas ✓
- **10 core concepts** (recap: sợi chỉ, brute-force→ANN, RAG inference-time, cosine [−1,1] normalize, semantic≠syntactic, EF contract, CRUD≠SQL, bottleneck memory+embedding+data-quality, top-k+threshold, chọn store) ✓
- **Bảng tổng kết 10 quyển** ✓
- *Bỏ có chủ đích (không phải kiến thức):* analogy "thư viện thông minh", self-check, one-liner tiếng Anh, lời chúc mừng/định hướng nghề nghiệp chung (làm lab/public GitHub — không phải kiến thức kỹ thuật).
