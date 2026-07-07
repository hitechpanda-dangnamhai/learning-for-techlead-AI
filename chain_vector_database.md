# Chain gối đầu — Vector Databases (từ Zero đến Staff Engineer)

> Nén theo chain gối đầu: đuôi node trước = đầu node sau. Thuật ngữ giữ nguyên tiếng Anh,
> giải thích tiếng Việt ngắn. Bảng so sánh + con số chốt giữ nguyên dạng phi tuyến.

---

## MẠCH CHÍNH — vector DB là gì → vì sao cần ANN
vector database là gì => vector database lưu data dưới dạng vector => vector là mảng các con số (vd `[180, 2010, 4.6]`) => vector về mặt toán là điểm/mũi tên, định nghĩa bởi size (độ dài) + direction (hướng) => mỗi con số trong mảng là 1 dimension (chiều) => dimension còn gọi là feature/attribute (đặc trưng) => mỗi feature là 1 đặc trưng của data => embedding thật thường 384–3072 chiều => các data point được sắp trong không gian đa chiều => sắp dựa trên proximity (độ gần) => gần nghĩa là giống => nên vector DB trả lời "cái gì *giống* nhất" => khác SQL vốn trả lời "cái gì *bằng đúng*" => SQL `WHERE` chỉ so sánh bằng nhau hoặc lớn/nhỏ hơn => nên SQL bất lực với "giống về ngữ nghĩa" (tìm ảnh giống ảnh, gợi ý sản phẩm tương tự) => tìm cái giống nhất gọi là similarity search => similarity search chính là nearest neighbor (NN) search => proximity được đo bằng distance (khoảng cách) => quét mọi vector để tìm nearest gọi là exact/flat search => exact search tốn O(N·d) (N vector, d chiều) => vd 100M × 768 chiều ≈ 1.5×10¹¹ phép tính/query => O(N·d) không scale khi N cực lớn + QPS cao => không scale nên chuyển sang ANN => ANN là Approximate Nearest Neighbor => ANN là *cả một họ* thuật toán tìm gần đúng để nhanh gấp hàng nghìn lần => ANN đánh đổi recall lấy tốc độ => recall là % hàng xóm thật lọt vào top-k trả về => recall nằm trong tam giác bất khả thi Recall–Latency–Memory => vector DB là hạ tầng cốt lõi của recommendation, semantic search, và RAG => RAG là dùng vector search lấy context liên quan rồi đưa cho LLM sinh câu trả lời

## CHAIN — Embedding: vector đến từ đâu
vector không tự có => vector đến từ embedding model => embedding model là 1 mô hình ML (thường là neural network) => nó nhận input phi cấu trúc (text/ảnh/audio) => xuất ra 1 vector cố định chiều => sao cho input giống nhau về ngữ nghĩa thì vector gần nhau => vd "con chó" và "chú cún" cho 2 vector gần nhau dù không trùng ký tự nào => model thật 2026 gồm text-embedding-3-small (OpenAI, 1536 chiều), voyage-3.5 (1024 chiều), open-source bge/e5/nomic-embed, ảnh dùng CLIP => điểm 90% người mới bỏ sót: vector DB KHÔNG tự sinh ra vector => nên pipeline thật là: data thô → embedding model → vector → vector DB lưu + đánh index => query cũng qua *cùng* embedding model → query vector → vector DB tìm nearest => quy tắc vàng: query và data phải đi qua cùng một embedding model => nếu khác model thì 2 vector rơi vào 2 hệ tọa độ khác nhau => 2 hệ tọa độ khác thì kết quả vô nghĩa => đây là lý do đổi embedding model = phải re-embed toàn bộ corpus

## CHAIN — cosine & normalize (rẽ từ "distance")
distance cho text hay dùng cosine similarity => cosine đo góc giữa 2 vector => cosine bỏ qua độ dài vector => bỏ độ dài vì với text, độ dài chỉ phản ánh độ dài văn bản => nên 2 đoạn cùng chủ đề dù dài ngắn khác nhau vẫn nên coi là giống => muốn triệt tiêu ảnh hưởng độ dài thì normalize vector => normalize là đưa vector về độ dài 1 (unit vector) => sau normalize thì cosine = dot product => và khi đó cosine, dot, thứ tự của L2 cho ra cùng một ranking => nhiều hệ thống normalize sẵn rồi dùng dot product vì dot tính nhanh nhất

## CHAIN — cái bẫy scale → normalization (rẽ từ Euclidean tầng basic)
tính Euclidean thô trên feature chưa chuẩn hóa gặp cái bẫy => khoảng cách bị chi phối bởi feature có scale lớn nhất => vd cột `year` chênh 6 lấn át `rating` chênh 0.3, `pages` chênh 10 lấn át tất cả => kết quả không phản ánh cái ta thực sự quan tâm => sửa bằng normalization => normalization đưa mỗi feature về cùng thang (z-score hoặc min-max) => hoặc với embedding thật thì normalize cả vector về độ dài 1

## CHAIN — HNSW (rẽ từ "ANN")
ANN phổ biến nhất 2026 là HNSW => HNSW là graph nhiều tầng (Hierarchical Navigable Small World) => tầng trên cùng thưa để nhảy xa => tầng dưới dày để tinh chỉnh => query đi từ trên xuống, mỗi tầng greedy nhảy đến node gần query hơn => cơ chế giống skip list trong không gian vector => nhờ vậy query time ≈ O(log N) => O(log N) cho recall cao + latency thấp => nhưng HNSW ngốn RAM vì phải giữ cả graph trong bộ nhớ => và build/insert chậm (1–100M vector mất phút đến giờ) => tham số M = số cạnh mỗi node (cao hơn = recall tốt hơn, RAM nhiều hơn) => efConstruction = độ rộng tìm kiếm lúc build => efSearch = độ rộng lúc query => efSearch là knob chỉnh recall↔latency ngay tại runtime

## CHAIN — IVF (rẽ từ "ANN")
một họ ANN khác là IVF (Inverted File) => IVF dùng k-means chia không gian thành nlist cụm => mỗi cụm là 1 Voronoi cell => query chỉ quét nprobe cụm gần nhất thay vì toàn bộ => nlist là số cụm, cố định lúc build => nprobe là số cụm quét lúc query => nprobe là knob recall↔latency => IVF build nhanh (giây–phút) và insert rẻ => nên IVF hợp workload cần freshness cao => nhược điểm: recall dễ tụt với query long-tail rơi vào ranh giới cụm => efSearch của HNSW đóng vai trò y hệt nprobe của IVF

## CHAIN — nprobe trade-off (con số thực đo, rẽ từ "nprobe")
tăng nprobe thì recall tăng nhưng phải quét nhiều hơn => nprobe=1 quét 0.8% dữ liệu, recall@10 = 0.70 (nhanh nhưng bỏ sót) => nprobe=5 quét 5.9%, recall@10 = 1.00 → sweet spot (gần đúng hoàn toàn với ~1/17 công sức) => nprobe=50 quét 50.5%, recall@10 = 1.00 => nprobe=100 quét 100% = quay về brute-force exact => triết lý ANN gói gọn: chỉnh một knob để trượt trên đường cong recall↔cost

## CHAIN — PQ & quantization (rẽ từ "ngốn RAM")
HNSW ngốn RAM thì nén bằng quantization => một cách nén là PQ (Product Quantization) => PQ cắt vector d chiều thành m đoạn con => mỗi đoạn con thay bằng ID của centroid gần nhất trong codebook => nhờ đó vector 1536 chiều × 4 byte (6KB) nén xuống vài chục byte => đánh đổi: nén nhiều thì mất một phần recall => PQ thường ghép thành IVF-PQ (tiết kiệm RAM cực mạnh) hoặc HNSW-PQ => cách nén đơn giản hơn là float32 → int8 (÷4), còn PQ nén ÷10 đến ÷40 => họ hàng nén mới: RaBitQ (nhị phân ngẫu nhiên, recall ~HNSW ở memory ~IVFFlat) => DiskANN/Vamana đặt graph trên SSD cho tỷ-scale không vừa RAM => ScaNN của Google tối ưu inner-product

## CHAIN — recall@k & two-stage (rẽ từ "recall")
recall@k = số hàng xóm thật trong top-k mà ANN tìm được / k => recall@k là thước đo *chất lượng* của ANN => latency vô nghĩa nếu kết quả sai => muốn recall gần exact mà chi phí thấp thì dùng two-stage search => giai đoạn 1: ANN thô lấy dư candidate (vd top-100) => giai đoạn 2: tính lại distance full-precision (hoặc dùng reranker/cross-encoder) => rồi cắt xuống top-10

## CHAIN — curse of dimensionality (rẽ từ "chiều cao")
ở chiều cao (hàng trăm–nghìn chiều) xảy ra curse of dimensionality => curse of dimensionality: mọi điểm gần như cách đều nhau => khoảng cách hàng xóm gần nhất và xa nhất co lại gần bằng nhau => nên nhiều ANN đơn giản (LSH ngây thơ) thất bại trên dữ liệu ngẫu nhiên => kiểm chứng: LSH random-hyperplane trên 10k vector Gaussian 128 chiều → recall@5 = 0.0 => may thay embedding thật KHÔNG phân bố ngẫu nhiên => embedding thật clustered (gom cụm theo chủ đề) => chính cấu trúc cụm này là thứ ANN khai thác được => bài học: đừng benchmark ANN trên `np.random.randn`, hãy dùng data giống production

## CHAIN — memory math ở tỷ-scale (rẽ từ "Memory")
câu hỏi staff đầu tiên luôn là "nó ngốn bao nhiêu RAM" => 1 tỷ vector × 1536 chiều × 4 byte (float32) = ~6.1TB => 6.1TB mới chỉ là vector thô, chưa tính graph HNSW (thêm ~1.5–2×) => 6TB không server đơn nào giữ nổi trong RAM => đòn bẩy 1 là quantization (int8 ÷4, PQ ÷10–÷40) => 6TB có thể xuống vài trăm GB (nhưng recall giảm, phải đo không đoán) => đòn bẩy 2 là sharding (scale ngang) => sharding chia corpus ra N shard, mỗi shard 1 máy => query fan-out song song tới các shard rồi merge top-k => đòn bẩy 3 là disk-based index (DiskANN) => DiskANN giữ graph trên SSD, chỉ nạp phần nóng vào RAM

## CHAIN — bottleneck thật (rẽ từ "sharding / nhiều máy")
nhưng bottleneck thật hiếm khi nằm ở DB => đo thực tế: một query RAG ~300ms thì DB chỉ tốn 5–8ms => ~90ms là sinh embedding cho query => phần lớn thời gian còn lại là LLM inference => bài học: đừng tối ưu tầng không phải bottleneck => nên cache embedding của query hay lặp, gộp batch, hoặc dùng embedding model nhỏ hơn => và luôn đo end-to-end trước khi tối ưu bất cứ gì

## CHAIN — hybrid search (rẽ từ "similarity search dở với tên riêng")
similarity search dở với proper noun / mã đơn hàng / số phiên bản / error code => những thứ đó cần khớp chính xác => khớp chính xác thì dùng full-text / BM25 (keyword) => production nghiêm túc chạy song song BM25 + vector => ghép 2 điểm số lại thành hybrid search => hợp nhất điểm bằng Reciprocal Rank Fusion (RRF) => thường thêm một reranker (cross-encoder) ở cuối => Weaviate/Qdrant/Vespa hỗ trợ hybrid native, còn pgvector/Pinecone phải ghép tay

## CHAIN — khi nào cần / KHÔNG cần vector DB (rẽ từ "vector database")
vector DB *riêng* chỉ cần khi data đủ lớn => corpus < ~10M mà đã chạy Postgres thì dùng pgvector => pgvector 0.5.0+ có HNSW, ngang ngửa DB chuyên dụng ở scale ~1M => pgvector cho vector nằm cùng bảng ứng dụng, transaction ACID, không thêm hệ thống, không sync layer => nên pgvector là default đúng cho ~70% workload AI-agent/RAG => nếu vấn đề thật ra là exact match (mã, ID, tên riêng) thì dùng full-text/BM25, đừng ép vector => chỉ NÊN dùng DB chuyên dụng khi scale vượt 50–100M, cần filtered search cực nhanh, cần hybrid native, hoặc cần tách rời scale read/write/storage => DB chuyên dụng gồm Pinecone / Qdrant / Weaviate / Milvus

## CHAIN — cost, ops, reliability (rẽ từ "DB chuyên dụng / Pinecone")
Pinecone tiện nhưng ở enterprise-scale có thể đắt gấp 5–10× so với self-host Qdrant/Milvus => ngành bị quản lý (data sovereignty) có thể loại thẳng managed cloud => đòn bẩy giảm cost lớn nhất: giảm chiều embedding, quantization, và giảm số vector (dedup/chunk tốt) => đo latency phải theo p99, không phải trung bình => và phải tách riêng thời gian embedding vs DB vs LLM khi đo => giám sát phải theo recall@k trên một golden set cố định theo thời gian => vì recall là thứ tụt âm thầm, không có metric này thì bạn mù => failure modes cần nhớ: index rebuild làm RAM tăng đột biến (build HNSW mới song song rồi drop cũ) => node chết giữa fan-out → cần replica => embedding model đổi version âm thầm → recall drift

## CHAIN — quyết định tổ chức (rẽ từ "chọn DB")
sự thật khó chịu: phần lớn thất bại của vector DB là tự gây ra => chất lượng chunking + embedding quan trọng hơn việc chọn DB nào => nên chọn DB thường chỉ là tie-breaker sau cam kết hạ tầng sẵn có => quyết định embedding model là quyết định *khóa cứng* (đổi = re-embed toàn bộ + downtime index + chạm mọi downstream team) => nên cần versioning và A/B trên golden set trước khi rollout => build vs buy là quyết định người + tiền, không chỉ kỹ thuật => team 3–5 người thì managed (Pinecone/pgvector) ít ma sát nhất => chọn Milvus mà không có người vận hành là tự bắn chân

## CHAIN — edge cases (rẽ từ "filtered search")
filtered search ("giống + `category='shoes'`") có 2 kiểu => pre-filter (lọc trước ANN) dễ làm rỗng cụm → recall tụt thảm => post-filter (lọc sau ANN) thì tốn công => engine tốt (Qdrant) nhúng filter *trong lúc* traversal để tránh recall sụp => vector trùng lặp / near-duplicate làm loãng kết quả → cần dedup trước bước embedding => insert/delete động: HNSW không thích xóa (để lại tombstone) → phải rebuild định kỳ, IVF chịu insert tốt hơn => empty/short/out-of-distribution query trả top-k rác score thấp → đặt threshold để nói "không có kết quả đủ tốt"

## CHAIN — faiss (thư viện chuẩn, để biết công cụ)
faiss là thư viện ANN chuẩn công nghiệp => `IndexFlatIP` = brute-force theo inner product => data normalize sẵn thì inner product = cosine similarity => `IndexFlatIP` hợp corpus nhỏ / cần ground-truth => khi N lớn thì đổi sang `IndexHNSWFlat` hoặc `IndexIVFFlat`

## CHAIN — khung system design (500M docs, p99<200ms, filter tenant)
làm rõ requirement trước khi vẽ: QPS đỉnh, read/write-heavy, freshness, recall mục tiêu, budget, filter cardinality => ước lượng: 500M × 1024 × 4B ≈ 2TB float32 → phải quantize + shard => chọn int8 hoặc PQ để vừa RAM cụm, shard theo hash để phân tải đều => pipeline ingest: chunk → embed (batch, GPU) → ghi vào N shard => pipeline query: embed (cache LRU cho query nóng) → fan-out shard → merge top-k → rerank → filter tenant *trong* traversal => index: HNSW cho latency (chỉnh efSearch theo p99), hoặc IVF-PQ nếu RAM là ràng buộc cứng, kèm two-stage => chọn tool: 500M + multi-tenant + self-host → Milvus/Qdrant, chấp nhận managed → Pinecone => vận hành: replica cho HA, giám sát recall@k golden set, blue-green cho index rebuild, SLO p99, cost guardrail => chốt bằng bottleneck insight: ở scale này DB hiếm khi là nút thắt, embedding generation + rerank mới là → đo end-to-end, tối ưu tầng đắt nhất trước

---

## BẢNG — Distance metrics (chọn cách đo "gần")
| Metric | Công thức (ý tưởng) | Đo cái gì | Dùng khi |
|---|---|---|---|
| **L2 (Euclidean)** | √Σ(aᵢ−bᵢ)² | khoảng cách vị trí | features có ý nghĩa vật lý, cùng scale |
| **Cosine similarity** | (a·b)/(‖a‖‖b‖) | góc giữa 2 vector (bỏ độ dài) | text embeddings (mặc định phổ biến nhất) |
| **Dot product** | a·b | góc + độ lớn | khi độ lớn vector mang thông tin (vd recsys) |

> Sau khi normalize về độ dài 1: cosine ≡ dot ≡ ranking của L2.

## BẢNG — So sánh index
| Index | Query Big-O | Recall | RAM | Build/Insert | Chọn khi |
|---|---|---|---|---|---|
| **Flat (exact)** | O(N·d) | 100% | vừa | tức thời | corpus nhỏ (<50k), cần ground-truth |
| **HNSW** | ~O(log N) | rất cao | **cao** | chậm | mặc định cho <10M, cần latency thấp + data động |
| **IVF-Flat** | O(nprobe·N/nlist·d) | cao (tùy nprobe) | vừa | nhanh | data lớn, tĩnh, cần build nhanh / freshness |
| **IVF-PQ** | rẻ | trung bình | **rất thấp** | nhanh | tỷ-scale, RAM là bottleneck |
| **DiskANN** | log-ish | cao | thấp (SSD) | chậm | tỷ-scale không vừa RAM |

> Rule of thumb 2026: bắt đầu bằng **HNSW** cho hầu hết RAG <~10M vector; chỉ chuyển IVF/IVF-PQ khi scale hoặc RAM ép buộc.

## BẢNG — Landscape 2026 (chọn công cụ)
| Công cụ | Kiểu | Điểm mạnh | Chọn khi |
|---|---|---|---|
| **pgvector** | Postgres extension | đơn giản, ACID, không thêm hệ thống | đã có Postgres, <10–50M vector |
| **Pinecone** | Managed | zero-ops, scale tự động | không muốn lo hạ tầng, chấp nhận cost cao & ít quyền tune recall |
| **Qdrant** | OSS (Rust) | filtered search nhanh nhất, latency thấp ổn định | self-host, filter nặng, cần tốc độ |
| **Weaviate** | OSS + cloud | hybrid search & multi-modal native | cần keyword+vector trong một query |
| **Milvus** | OSS (Zilliz) | tỷ-scale, sharding trưởng thành | >100M–tỷ vector, có team data-eng |

## BẢNG — Tam giác bất khả thi Recall–Latency–Memory
| Kéo lên | Thường phải hạ | Knob điều khiển |
|---|---|---|
| **Recall** ↑ | Latency ↑ và/hoặc Memory ↑ | efSearch ↑, nprobe ↑, ít nén PQ hơn |
| **Latency** ↓ | Recall ↓ | efSearch ↓, nprobe ↓ |
| **Memory** ↓ | Recall ↓ | quantization/PQ mạnh hơn |

> Mọi tuning là một nước đi bên trong tam giác này.

---

## TỪ KHÓA MỒI
- Mạch chính: **vector database là gì**
- Embedding: **vector đến từ đâu**
- cosine & normalize: **bỏ độ dài**
- cái bẫy scale: **year lấn át rating**
- HNSW: **graph nhiều tầng / skip list**
- IVF: **k-means / nprobe**
- nprobe trade-off: **sweet spot 5.9%**
- PQ & quantization: **6KB → vài chục byte**
- recall@k & two-stage: **top-100 rồi rerank**
- curse of dimensionality: **recall@5 = 0.0**
- memory math: **6.1TB**
- bottleneck: **DB chỉ 5–8ms**
- hybrid search: **BM25 + vector + rerank**
- khi nào cần: **pgvector 70%**
- cost/ops: **đắt 5–10×**
- quyết định tổ chức: **tự gây ra**
- edge cases: **pre-filter làm rỗng cụm**
- faiss: **IndexFlatIP**
- system design: **500M, p99<200ms**

## ĐÃ PHỦ
- **Định nghĩa nền tảng:** vector, dimension, feature, size+direction, similarity/NN search, proximity, distance ✓
- **Vấn đề & vì sao tồn tại:** SQL exact/range vs "giống ngữ nghĩa", ứng dụng recommendation/semantic search/RAG, định nghĩa RAG ✓
- **Embedding:** embedding model, "vector không tự có", pipeline query=data cùng model, hệ tọa độ khác → rác, re-embed khi đổi model, model thật 2026 (text-embedding-3-small 1536 / voyage-3.5 1024 / bge/e5/nomic / CLIP) ✓
- **Distance metrics:** L2 / cosine / dot (bảng), vì sao text dùng cosine, normalize → cosine=dot=ranking L2, dot nhanh nhất ✓
- **Cái bẫy scale + normalization:** feature scale lớn lấn át, z-score/min-max/normalize vector ✓
- **Big-O & ANN:** exact O(N·d), 100M×768≈1.5×10¹¹, ANN là *họ* thuật toán, đổi recall lấy tốc độ ✓
- **Curse of dimensionality:** mọi điểm cách đều, LSH recall@5=0.0 trên Gaussian 128d, embedding clustered, đừng benchmark trên random ✓
- **HNSW:** graph nhiều tầng, skip list, O(log N), ngốn RAM + build chậm, M/efConstruction/efSearch ✓
- **IVF:** k-means/Voronoi, nlist/nprobe, build nhanh/insert rẻ/freshness, recall tụt ở ranh giới cụm, efSearch↔nprobe ✓
- **PQ & quantization:** cắt m đoạn/codebook, 6KB→vài chục byte, int8 ÷4 & PQ ÷10–40, IVF-PQ/HNSW-PQ, RaBitQ/DiskANN-Vamana/ScaNN ✓
- **recall@k & two-stage:** định nghĩa recall@k, latency vô nghĩa nếu sai, ANN thô top-100 → rerank full-precision → top-10 ✓
- **Con số nprobe:** 1→0.8%/0.70; 5→5.9%/1.00 (sweet spot); 50→50.5%; 100→100%=brute-force ✓
- **Bảng so sánh index** (Flat/HNSW/IVF-Flat/IVF-PQ/DiskANN) + rule of thumb 2026 ✓
- **Memory tỷ-scale:** 6.1TB, +1.5–2× graph, quantization/sharding-fanout/DiskANN ✓
- **Bottleneck:** RAG 300ms → DB 5–8ms, embedding ~90ms, còn lại LLM; đo end-to-end ✓
- **Khi nào cần/không:** pgvector <10M & có Postgres (ACID, không sync), ~70% workload; DB chuyên dụng khi >50–100M / filtered / hybrid / tách scale ✓
- **Landscape 2026 (bảng):** pgvector/Pinecone/Qdrant/Weaviate/Milvus ✓
- **Cost/ops/reliability:** Pinecone đắt 5–10×, data sovereignty, giảm cost (chiều/quantize/dedup), p99, golden-set recall, recall drift, index rebuild RAM spike, replica ✓
- **Hybrid search:** BM25+vector, RRF, reranker/cross-encoder, native vs ghép tay ✓
- **Tổ chức:** thất bại tự gây ra, chunking/embedding > chọn DB, embedding model khóa cứng, build-vs-buy team 3–5 người ✓
- **Edge cases:** filtered pre/post/in-traversal, dedup, HNSW tombstone vs IVF insert, threshold cho OOD query ✓
- **faiss:** IndexFlatIP=cosine khi normalize, đổi HNSWFlat/IVFFlat khi N lớn ✓
- **System design 500M:** requirement → 2TB math → quantize/shard → pipeline → index → tool → ops → bottleneck ✓
- **Tam giác Recall–Latency–Memory (bảng)** ✓
- **Ghi chú staff:** dữ liệu sai âm thầm là bug tốn kém nhất ML (bẫy genre trong bài gốc) → đã đưa vào tinh thần "soi bất nhất data" ở chain tổ chức/edge case *(bài gốc nêu như một lưu ý phương pháp, không phải khái niệm kỹ thuật riêng)*
