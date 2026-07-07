# Chain gối đầu — Vector Databases vs Traditional (Relational) Databases

> Nén theo chain gối đầu: đuôi node trước = đầu node sau. Thuật ngữ giữ nguyên tiếng Anh,
> giải thích tiếng Việt ngắn. Bảng so sánh + con số chốt giữ nguyên dạng phi tuyến.
> Bài gốc là bài **so sánh** → xoáy vào trục ra quyết định kiến trúc.

---

## MẠCH CHÍNH — hai câu hỏi → hai loại DB → khác từ tầng lưu trữ
câu hỏi trên data chia làm 2 loại bản chất khác nhau => loại A là exact match / range ("bằng đúng / trong khoảng") => loại A có đáp án đúng-sai rạch ròi => relational database sinh ra để trả lời loại A cực tốt => loại B là similarity ("giống cái này") => loại B không có đúng tuyệt đối, chỉ có gần/xa về ý nghĩa => vector database sinh ra để trả lời loại B => hai loại DB là hai công cụ cho hai câu hỏi, không phải cái này thay cái kia => khác biệt của chúng bắt nguồn từ tầng lưu trữ => relational sắp data theo *giá trị có thứ tự* => "có thứ tự" cho phép đánh index bằng B-tree => vector sắp data theo *độ gần trong không gian* => "độ gần" cần index kiểu graph/cluster (ANN) => khác nhau từ tầng lưu trữ nên khác nhau ở mọi tầng phía trên

## CHAIN — Relational internals (rẽ từ "relational database")
relational database tổ chức data thành tables (bảng) => mỗi table gồm rows và columns => row là một record (bản ghi) => column là một property/attribute => key là cột dùng để định danh và nối bảng => primary key định danh duy nhất một row => foreign key trỏ sang primary key của bảng khác => foreign key tạo relationship (quan hệ) giữa các bảng => truy vấn relational bằng SQL (Structured Query Language) => SQL có 4 lệnh lõi SELECT / INSERT / UPDATE / DELETE => relational mạnh nhất khi structured data + quan hệ giữa entity được định nghĩa rõ

## CHAIN — B-tree (rẽ từ "index bằng B-tree")
relational đánh index một cột bằng B-tree => B-tree là cây cân bằng, node chứa key *đã sắp thứ tự* + con trỏ => chiều cao cây ~log_b(n) => nên exact lookup và range chạy O(log n) => insert/delete cũng O(log n), cây tự cân bằng => nhờ vậy `WHERE year>1950` nhanh: DB nhảy tới nhánh cây thay vì quét cả triệu row => B-tree hợp SQL vì data 1 chiều có thứ tự toàn phần (`<`, `=`, `>`) => nhưng thứ tự toàn phần đó biến mất ở 768 chiều => nên B-tree vô dụng cho câu hỏi "gần nhất" => phải chuyển sang vector index (ANN)

## CHAIN — Vector side + embedding (rẽ từ "vector database")
vector database lưu vector nhiều chiều biểu diễn một data item => mỗi dimension của vector ứng với một attribute/feature => vector đến từ embedding qua transformer => ảnh/text/audio đi qua transformer tương ứng (Image Transformer / NLP Transformer / Audio Transformer) => transformer biến data thô thành vector rồi mới lưu vào vector DB => truy vấn vector bằng similarity search (nearest neighbor query) => similarity search tìm các vector gần nhất với query => nhờ vậy vector gom được "chất sci-fi" của 3 cuốn bất kể năm hay rating => điều mà `WHERE genre='sci-fi'` chỉ làm được nếu đã có sẵn cột genre gán thủ công => tức vector nắm được ngữ nghĩa mà không cần ai gán nhãn tay

## CHAIN — Vector index / ANN + đính chính "metric trees" (rẽ từ "vector index")
vector index phải xử lý không gian nhiều chiều => bài gốc nói vector DB dùng "metric trees and hashing" => nhưng câu này đã lỗi thời => metric trees (KD-tree, ball tree) sụp đổ ở chiều cao vì curse of dimensionality => LSH (hashing) cũng đã nhường sân => thực tế 2026 index mặc định là HNSW (graph nhiều tầng) => rồi tới IVF / IVF-PQ, và DiskANN cho tỷ-scale => HNSW query ~O(log n) nhưng hằng số lớn + ngốn RAM => điểm khác lõi: vector index là approximate (đổi chút recall lấy tốc độ) => còn B-tree là exact => nên relational hứa "đúng", vector hứa "đủ gần, đủ nhanh"

## CHAIN — Curse of dimensionality (con số thực đo, rẽ từ "curse of dimensionality")
curse of dimensionality: chiều càng cao thì khoảng cách nearest và farthest co lại gần bằng nhau => đo bằng tỉ lệ (max−min)/min trên 2000 điểm => d=2 → 128.363 (điểm xa nhất cách query ~129 lần điểm gần nhất) => d=10 → 4.116 => d=100 → 0.543 => d=1000 → 0.152 (điểm xa nhất chỉ cách ~1.15 lần điểm gần nhất) => nghĩa là ở chiều cao mọi điểm gần như *cách đều* nhau => nên "cắt theo từng trục" kiểu KD-tree không loại được ứng viên nào => đây chính là lý do vector DB bỏ metric tree, chuyển sang graph (HNSW) khai thác cấu trúc cụm của embedding thật

## CHAIN — Vector library vs Vector database (rẽ từ "vector database")
vector library ≠ vector database => vector library (FAISS, Annoy, HNSWlib, ScaNN) chỉ là thuật toán ANN trần => library thường in-memory, tĩnh, chạy ANN trên một không gian vector cố định => nên library không quản lý được dữ liệu theo thời gian => thêm document mới vào index FAISS/Annoy rất khó vì data đã index là immutable (bất biến) => HNSWlib là ngoại lệ vì có CRUD + hỗ trợ đọc-ghi đồng thời => nhưng HNSWlib vẫn thiếu deployment ecosystem, replication, fault tolerance => vector database (Pinecone, Qdrant, Weaviate, Milvus) = thuật toán + persistence + filtering + replication + access control + API => database còn cho full CRUD + metadata filtering + distributed persistence cho dataset lớn => nên quy tắc chốt: cần CRUD thì cần database, không phải library => analogy: library = động cơ, database = cả chiếc xe

## CHAIN — ACID vs eventually consistent (rẽ từ "relational hứa đúng")
relational bảo đảm giao dịch bằng ACID => ACID = Atomicity, Consistency, Isolation, Durability => vd chuyển tiền: trừ-và-cộng cả hai hoặc không gì cả, không bao giờ mất tiền giữa chừng => đây là lãnh địa relational thống trị, vector DB không cạnh tranh => vector DB thường ưu tiên throughput/scale hơn strong consistency => nhiều vector DB là eventually consistent, index có độ trễ cập nhật => nên không đặt sổ cái tài chính lên vector DB => hệ quả: source of truth (đơn hàng, người dùng, số dư) sống ở relational => còn embedding (biểu diễn để tìm-giống) sống ở vector store => vector store được sinh ra *từ* source of truth => nên vector store là derived data => derived data mất thì re-embed lại được => nên yêu cầu durability của vector store nhẹ hơn

## CHAIN — Polyglot persistence (rẽ từ "derived data")
câu trả lời staff không phải "hoặc/hoặc" mà "cả hai" => dùng nhiều loại DB đúng việc gọi là polyglot persistence => Postgres/MySQL làm source of truth (users, orders, documents, permissions, ACID) => một embedding pipeline đọc data gốc → transformer → vector => vector store giữ embeddings (derived) cho semantic search / RAG / recsys => truy vấn người dùng thường fan-out cả hai rồi hợp nhất => hợp nhất = filter kiểu SQL + rank kiểu vector

## CHAIN — pgvector hợp nhất hai thế giới (rẽ từ "cả hai / polyglot")
nếu đã chạy Postgres và <~10–50M vector thì đừng dựng hệ mới => dùng pgvector — extension thêm vector search vào chính Postgres => pgvector cho ACID (data gốc) + vector search nằm cùng một chỗ => nên không thêm hệ thống, không sync layer => pgvector dùng toán tử `<=>` = cosine distance => một câu SQL vừa `WHERE` (filter B-tree) vừa `ORDER BY embedding <=>` (vector index HNSW) => tức filter relational + similarity vector trong MỘT câu => đây là default hợp lý cho phần lớn use case vừa và nhỏ

## CHAIN — khi nào chọn gì (rẽ từ ra quyết định)
chỉ relational là đủ khi data structured + câu hỏi exact/range/aggregation/join + cần ACID => đừng thêm vector DB chỉ vì "nghe hot" => cần vector khi câu hỏi bản chất là similarity (semantic search, RAG, recommendation, anomaly/dedup, image/audio retrieval) mà `WHERE`/`LIKE` không diễn đạt nổi => cần cả hai (đa số hệ thật) khi sản phẩm có cả nghiệp vụ giao dịch lẫn tính năng tìm theo ý nghĩa => trong nhóm vector: cần CRUD/persistence/HA thì chọn database, không phải library => FAISS hợp prototype/nghiên cứu khi latency quan trọng hơn persistence => Pinecone hợp production semantic search/RAG ít bảo trì => Milvus hợp enterprise scale phân tán => Weaviate hợp khi cần structured filtering + hybrid search

## CHAIN — Scale & cost (rẽ từ scalability)
toán bộ nhớ vector: 1 tỷ × 1536 chiều × 4 byte ≈ 6TB vector thô => 6TB không server đơn nào giữ nổi => phải quantization (int8/PQ) + sharding + fan-out => relational tỷ-row cũng đau: partitioning, read replica, đôi khi sharding => về cost, mẫu hình phổ biến là khởi đầu bằng Pinecone (dễ nhất) => rồi migrate sang self-host (Qdrant/Weaviate) khi scale để tối ưu chi phí => điểm migrate thường ~50–100M vector hoặc hóa đơn cloud >$500/tháng => managed đổi tiền lấy sự yên tâm, OSS đổi ops-overhead lấy tiền

## CHAIN — Bottleneck thật (rẽ từ latency)
bottleneck thật thường KHÔNG ở DB => trong RAG, sinh embedding cho query + LLM inference ngốn phần lớn latency => DB chỉ tốn vài ms => nên đừng tối ưu nhầm tầng => phải đo end-to-end trước khi tối ưu bất cứ gì

## CHAIN — Failure modes & monitoring (rẽ từ reliability)
relational có failure modes riêng: deadlock, replication lag, hot partition => vector có failure modes khác: index rebuild làm phình RAM, recall drift khi đổi embedding model, "no good result" cho query lạc => giám sát relational nhìn query latency / lock / replication => giám sát vector nhìn recall@k trên golden set (recall tụt âm thầm), QPS, memory

## CHAIN — Hybrid search & derived-data discipline (rẽ từ "vector mù ID")
vector search mù với ID / số / tên riêng => keyword search lại mù với ngữ nghĩa => nên production nghiêm túc chạy song song BM25 (keyword) + vector => hợp nhất điểm rồi rerank = hybrid search => về kỷ luật derived data: coi vector store là cache thông minh của relational => cần pipeline re-embed idempotent, versioned => đổi embedding model thì rebuild được từ source => nguyên tắc: đừng để vector store thành source of truth cho thứ không tái tạo được

## CHAIN — Tổ chức & roadmap (rẽ từ "thêm vector store")
thêm vector store = thêm một hệ phải vận hành, backup, monitor => cộng thêm một embedding pipeline mới => nên đây là cam kết vận hành dài hạn, không phải "cài thêm thư viện" => và quyết định embedding model là khóa cứng (đổi = re-embed cả kho) => build vs buy: team nhỏ → managed (Pinecone) hoặc pgvector (tái dùng Postgres sẵn có) ít ma sát nhất => chọn Milvus mà không có người vận hành phân tán là tự bắn chân

## CHAIN — 3 lỗi thường gặp (mạch độc lập)
lỗi 1: ép vector search làm việc của SQL (tìm theo ID, số đơn hàng, khoảng ngày) => vector search "mờ" theo bản chất nên dùng cho exact match là sai công cụ, chậm và kém chính xác hơn `WHERE` => lỗi 2: chọn library khi thật ra cần database => bắt đầu với FAISS trong notebook, đến lúc cần thêm/xóa/cập nhật vector liên tục + metadata filter + HA thì vỡ trận, phải viết lại => lỗi 3: quên rằng relational DB giờ cũng làm được vector => Elasticsearch/Postgres đều thêm được vector search => nhưng full-text DB như Elasticsearch dựa trên inverted index nên vector similarity không mạnh bằng DB chuyên dụng

## CHAIN — Edge cases khi so hai thế giới (mạch độc lập)
update-heavy trên vector tốn kém vì phải re-embed + rebuild index => trong khi relational update một cột thì rẻ => data đổi liên tục thì cân nhắc IVF (insert rẻ) hoặc chấp nhận rebuild theo lịch => join: relational join nhiều bảng là bản năng, còn vector store không có "join" đúng nghĩa => muốn kết hợp thường phải kéo id về rồi join ở relational => exact-match trong vector (ID/tên riêng/số phiên bản) làm vector "mờ" hỏng → phải kèm keyword/filter (hybrid) => cold start / OOD query: vector vẫn trả top-k dù query lạc quẻ → cần threshold để nói "không có kết quả đủ giống"

## CHAIN — System design: e-commerce (mạch độc lập)
bài toán e-commerce có (a) trang quản lý đơn hàng và (b) tính năng "sản phẩm tương tự" => tách hai loại workload trước đã => (a) là relational thuần: đơn hàng, tồn kho, thanh toán cần ACID + join + exact/range → Postgres/MySQL => (b) là similarity: "sản phẩm tương tự" theo hình ảnh/mô tả → embedding + vector search => ghép thành kiến trúc polyglot: Postgres là source of truth, một pipeline embed sản phẩm → vector store (derived) => "similar" query = fan-out: filter kiểu SQL (còn hàng, đúng danh mục, hợp giá) + rank vector, rồi merge => chọn tool: catalog <10–50M + đã có Postgres → pgvector; >100M hoặc cần scale vector độc lập → tách Qdrant/Milvus + chấp nhận sync layer => vận hành: filter *trong* traversal để không sụp recall, giám sát recall@k golden set, embedding pipeline versioned, rebuild được từ source => chốt: không thay Postgres bằng vector DB mà thêm *bên cạnh* nó, coi embedding là derived, đo end-to-end vì bottleneck thường là embedding/inference chứ không phải DB

## CHAIN — Ngân hàng (Q&A trap, mạch độc lập)
hệ ngân hàng: sổ cái phải relational + ACID => vector chỉ dùng cho tính năng phụ như phát hiện gian lận (anomaly detection) => và ngay cả anomaly cũng là derived data, không phải source of truth

---

## BẢNG — So sánh 5 trục (bản staff, có đính chính bài gốc)
| Trục | Traditional / Relational | Vector | Ghi chú staff |
|---|---|---|---|
| **Data representation** | tables, rows, columns — structured data | vector nhiều chiều — encode ảnh/text/audio/sensor (unstructured) | vector *là derived* qua embedding; data gốc vẫn nên nằm ở nơi có cấu trúc |
| **Search & retrieval** | SQL, exact & range | similarity / nearest-neighbor | không loại trừ nhau → **hybrid search** (BM25 + vector) là chuẩn production |
| **Indexing** | B-tree | *bài gốc:* "metric trees & hashing" | **thực tế 2026: HNSW / IVF / PQ**; metric trees lỗi thời ở high-dim |
| **Scalability** | scale khó, thường sharding | thiết kế cho horizontal scaling | 1B×1536×4B ≈ 6TB → buộc quantization + shard; relational scale write cũng đau |
| **Applications** | app nghiệp vụ, hệ giao dịch | RAG, recsys, semantic search, anomaly detection, multimedia | case thật thường **dùng cả hai** trong một sản phẩm |

## BẢNG — 4 "trại" của vector search (khung phân loại 2026)
| Trại | Ví dụ | Bản chất |
|---|---|---|
| **Managed** | Pinecone | gửi vector + query vector, người khác lo phần còn lại |
| **OSS purpose-built** | Qdrant, Weaviate, Milvus, Chroma | self-host hoặc dùng managed cloud của hãng |
| **Database extensions** | pgvector, Elasticsearch | thêm vector search vào DB bạn đã chạy |
| **Libraries** | FAISS, Annoy | vector search *trần*, không có tầng database |

## BẢNG — Landscape 2026 (chọn nhanh)
| Công cụ | Loại | Điểm mạnh | Chọn khi |
|---|---|---|---|
| **pgvector** | extension trên Postgres | hợp nhất relational + vector, ACID, không thêm hệ thống | đã có Postgres, <10–50M vector |
| **FAISS / Annoy / HNSWlib** | library | ANN thô, nhanh, nhẹ | prototype, static data, tự lo hạ tầng |
| **Pinecone** | managed DB | zero-ops, hybrid search, scale | không muốn lo hạ tầng, chấp nhận cost cao |
| **Qdrant** | OSS (Rust) | filtered search nhanh, latency thấp | self-host, filter nặng |
| **Weaviate** | OSS + cloud | hybrid search & multi-modal native | cần keyword+vector một query |
| **Milvus** | OSS (Zilliz) | tỷ-scale, phân tán trưởng thành | >100M–tỷ vector, có team vận hành |

## BẢNG — Ví dụ chạy tay (cùng data, hai câu hỏi)
| Câu hỏi | Cơ chế | Kết quả |
|---|---|---|
| **A — SQL** `year>1950 AND rating≥4.6` | quét/nhảy B-tree, kiểm điều kiện | Dune, Foundation (đúng-sai tuyệt đối) |
| **B — vector** "giống Dune" (cosine) | rank theo similarity | Dune 1.000 > Foundation 0.998 > Neuromancer 0.993 > Emma 0.296 > Pride&Prej 0.217 |

---

## TỪ KHÓA MỒI
- Mạch chính: **hai câu hỏi khác bản chất**
- Relational internals: **tables/rows/columns/keys**
- B-tree: **thứ tự toàn phần 1 chiều**
- Vector + embedding: **transformer → vector**
- Vector index / metric-tree: **"metric trees" lỗi thời**
- Curse of dimensionality: **d=1000 → 0.152**
- Library vs database: **động cơ vs cả chiếc xe**
- ACID vs eventually consistent: **sổ cái tài chính**
- Polyglot persistence: **cả hai đúng việc**
- pgvector: **`<=>` một câu**
- Khi nào chọn gì: **cần CRUD → database**
- Scale & cost: **>$500/tháng migrate**
- Bottleneck: **DB vài ms**
- Failure modes: **recall drift**
- Hybrid & derived-data: **cache thông minh**
- Tổ chức & roadmap: **embedding model khóa cứng**
- 3 lỗi: **ép vector làm việc SQL**
- Edge cases: **vector không có join**
- System design: **e-commerce polyglot**
- Ngân hàng: **anomaly là derived**

## ĐÃ PHỦ
- **Vấn đề nền tảng:** hai câu hỏi loại A (exact/range) vs loại B (similarity), hai loại DB không thay thế nhau ✓
- **Relational định nghĩa:** tables, rows (record), columns (property), key, primary key, foreign key, relationship, SQL + 4 lệnh SELECT/INSERT/UPDATE/DELETE, mạnh ở structured data ✓
- **B-tree:** cây cân bằng, key có thứ tự + con trỏ, ~log_b(n), exact+range O(log n), insert/delete O(log n), thứ tự toàn phần 1 chiều, chết ở 768 chiều ✓
- **Vector side:** vector nhiều chiều, dimension=attribute/feature, embedding qua Image/NLP/Audio Transformer, similarity/nearest-neighbor, gom "chất sci-fi" không cần cột genre gán tay ✓
- **Vector index + đính chính:** "metric trees & hashing" lỗi thời, KD-tree/ball tree/LSH sụp ở high-dim, 2026 HNSW→IVF/IVF-PQ→DiskANN, approximate vs exact ✓
- **Curse of dimensionality (con số):** (max−min)/min → d=2:128.363, d=10:4.116, d=100:0.543, d=1000:0.152; cắt trục vô nghĩa → chuyển graph ✓
- **Library vs database:** FAISS/Annoy/HNSWlib/ScaNN là ANN trần in-memory tĩnh, immutable index, HNSWlib ngoại lệ (CRUD) nhưng thiếu ecosystem/replication/fault-tolerance; database = +persistence/filtering/replication/access-control/API/CRUD/metadata-filter/distributed-persistence; "cần CRUD → database"; động cơ vs cả chiếc xe ✓
- **4 trại:** managed / OSS purpose-built / DB extensions / libraries (bảng) ✓
- **ACID vs eventually consistent:** ACID = A/C/I/D, chuyển tiền, vector ưu tiên throughput, eventually consistent, index trễ, source of truth ở relational vs derived embedding, durability nhẹ hơn ✓
- **Polyglot persistence:** Postgres source of truth + embedding pipeline + vector store derived + fan-out hợp nhất (SQL filter + vector rank) ✓
- **pgvector:** extension, ACID + vector cùng chỗ, không sync layer, `<=>` = cosine distance, WHERE + ORDER BY embedding một câu, default use case vừa/nhỏ ✓
- **Khi nào chọn gì:** chỉ relational / cần vector / cần cả hai; FAISS prototype, Pinecone production ít bảo trì, Milvus enterprise phân tán, Weaviate structured filter+hybrid ✓
- **Scale & cost:** 6TB math + quantization/sharding/fan-out, relational partitioning/replica/sharding, cost pattern Pinecone→self-host ở ~50–100M hoặc >$500/tháng, managed vs OSS trade-off ✓
- **Bottleneck:** không ở DB, embedding+LLM ngốn latency, DB vài ms, đo end-to-end ✓
- **Failure modes & monitoring:** relational deadlock/replication-lag/hot-partition; vector index-rebuild-RAM/recall-drift/no-good-result; golden-set recall@k, QPS, memory ✓
- **Hybrid search & derived-data discipline:** BM25+vector+rerank, cache thông minh, re-embed idempotent/versioned, đừng để vector thành source of truth không tái tạo được ✓
- **Tổ chức/roadmap:** vector store = hệ phải vận hành + embedding pipeline, cam kết dài hạn, embedding model khóa cứng, build-vs-buy team nhỏ → managed/pgvector ✓
- **3 lỗi thường gặp:** ép vector làm việc SQL; chọn library khi cần database; quên relational cũng làm vector (Elasticsearch inverted index yếu hơn) ✓
- **Edge cases:** update-heavy re-embed/rebuild vs update cột rẻ, vector không có join, exact-match cần hybrid, OOD query cần threshold ✓
- **System design e-commerce:** tách workload (a) relational (b) similarity, polyglot Postgres+vector derived, fan-out filter+rank, pgvector vs Qdrant/Milvus theo scale, vận hành, chốt bottleneck ✓
- **Q&A ngân hàng:** sổ cái relational+ACID, vector chỉ anomaly detection và vẫn là derived ✓
- **Bảng ví dụ chạy tay** (SQL vs cosine output) giữ dạng bảng ✓
- *Bỏ có chủ đích (không phải kiến thức):* analogy "thủ thư", các self-check, one-liner tiếng Anh (chỉ là diễn đạt lại các ý đã phủ).
