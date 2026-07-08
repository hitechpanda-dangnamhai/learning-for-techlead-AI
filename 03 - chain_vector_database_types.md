# Chain gối đầu — Các loại Vector Database (Types of Vector Databases)

> Nén theo chain gối đầu: đuôi node trước = đầu node sau. Thuật ngữ giữ nguyên tiếng Anh,
> giải thích tiếng Việt ngắn. Bảng so sánh / đính chính / ma trận đánh đổi giữ dạng bảng.
> Bài gốc có nhiều chỗ sai/lỗi thời → phần đính chính được giữ nguyên như mắt xích trong chain.

---

## MẠCH CHÍNH — vấn đề → embedding → brute-force → ANN
keyword match (so khớp từ khóa) không đủ => cần semantic / meaning match (so khớp ý nghĩa) => SQL/B-tree giỏi exact match + range (`age>30`) nhưng bất lực với "giống về ý nghĩa" => "giống về ý nghĩa" cần biểu diễn object thành vector => vector (embedding) là dãy số (384/768/1536 con số) biểu diễn ý nghĩa của object => một embedding model biến object (câu/ảnh/audio) thành vector => thứ giống nhau về ý nghĩa thì vector gần nhau trong không gian => độ dài của vector gọi là dimension (số chiều) => tìm k vector gần query nhất gọi là similarity search (nearest neighbor search) => similarity search là nghiệp vụ trung tâm của mọi vector database => "gần/xa" được đo bằng distance metric => cách đơn giản nhất là quét tất cả vector rồi sắp xếp => quét tất cả gọi là brute-force (flat / exhaustive search) => brute-force cho kết quả exact 100% => nhưng độ phức tạp là O(N×d) (N phần tử, d chiều) => O(N×d) chết ở scale lớn => vd N=100M, d=768 → ~76.8 tỷ phép nhân mỗi query => một query mất hàng giây, user mong <100ms, nhân nghìn QPS → bất khả thi => nên chuyển sang ANN (Approximate Nearest Neighbor) => ANN đánh đổi độ chính xác lấy tốc độ => tìm gần đúng nhưng nhanh hơn hàng trăm–nghìn lần => đo độ "đúng gần" bằng recall => recall@k = (số kết quả đúng trong top-k) / k => cả ngành vector database xoay quanh đường cong đánh đổi recall ↔ latency ↔ memory

## CHAIN — Distance metric (rẽ từ "distance metric")
distance metric có 3 loại hay gặp => loại 1 là Euclidean / L2 = khoảng cách đường chim bay => công thức d = √((x₁−x₂)² + (y₁−y₂)² + …) => loại 2 là Cosine similarity đo góc giữa 2 vector => cosine bỏ qua độ dài vector nên là loại phổ biến nhất cho text embedding => loại 3 là Dot / Inner product = tích vô hướng => mẹo: normalize vector về độ dài 1 rồi dùng inner product = tương đương cosine nhưng nhanh hơn => nếu index bằng metric sai so với lúc tạo embedding (vd model cosine mà index L2 không normalize) thì recall tụt thảm hại

## CHAIN — Curse of dimensionality (rẽ từ "vì sao không dùng cây")
tại sao không dùng KD-tree cho similarity search => vì ở số chiều cao (100+) mọi điểm gần như cách đều nhau => cấu trúc cây sụp đổ, phải duyệt gần hết cây => nên không nhanh hơn brute-force => hiện tượng này gọi là curse of dimensionality => ANN (IVF, HNSW) né nó bằng cách khai thác cấu trúc cụm của dữ liệu thật => vì embedding đời thực có cụm, không rải đều => nói cách khác ANN chỉ hiệu quả khi intrinsic dimensionality thấp => tức data thật nằm trên một manifold ít chiều hơn số chiều danh nghĩa => nên với dữ liệu ngẫu nhiên đều (uniform random) thì không index nào cứu được recall => đó là lý do benchmark trên synthetic ngẫu nhiên thường lừa dối

## CHAIN — IVF (rẽ từ "ANN index")
một họ index ANN là IVF (Inverted File Index) => IVF dùng k-means chia không gian thành nlist cụm (clusters) => khi search chỉ quét nprobe cụm gần query nhất thay vì toàn bộ => nlist là số cụm, nprobe là số cụm quét lúc query => nprobe cao thì recall cao nhưng chậm hơn => nhờ chỉ quét một phần, Big-O trung bình ≈ O(nprobe × N/nlist × d) thay vì O(N×d) => IVF bắt buộc `train()` học centroid (k-means) TRƯỚC khi `add()` => quên `train()` là lỗi kinh điển => IVF chạy thật trên data có cụm cho recall@5 = 1.0 với nprobe=8 => tăng nprobe → recall lên, latency lên → đó là cái núm xoay ở production

## CHAIN — PQ / quantization (rẽ từ "memory ở billion-scale")
ở billion-scale, memory là nút thắt: 1 tỷ × 768 chiều × 4 byte (float32) = ~3TB RAM => 3TB không khả thi trên một máy => nén bằng Product Quantization (PQ) => PQ chia mỗi vector thành m đoạn con => mỗi đoạn con thay bằng id của centroid gần nhất trong codebook => codebook 256 entry → 1 byte mỗi đoạn => vd 768 float (3072 byte) → m=96 byte → giảm ~32 lần => đánh đổi: khoảng cách trở thành xấp xỉ (lossy) => ghép IVF + PQ = IVF-PQ, xương sống của billion-scale search (FAISS, Milvus) => bản hiện đại là Binary / Scalar Quantization => Qdrant có binary quantization giảm memory tới ~40 lần

## CHAIN — LSH (rẽ từ "index algorithm")
một họ index khác là LSH (Locality-Sensitive Hashing) => LSH thiết kế hàm hash sao cho vector giống nhau rơi vào cùng bucket với xác suất cao => query → hash → chỉ so với vector cùng bucket => LSH có provable guarantees (đảm bảo lý thuyết) đẹp => nhưng thực tế cần nhiều bảng hash để đạt recall tốt => nhiều bảng hash → tốn memory và khó tune => nên LSH đã bị HNSW vượt mặt ở hầu hết use case dense (bài gốc liệt kê LSH là góc nhìn hơi cũ)

## CHAIN — HNSW cơ chế (rẽ từ "ông vua hiện tại")
index mặc định hiện nay là HNSW (Hierarchical Navigable Small World, Malkov & Yashunin 2016) => HNSW là index đồ thị nhiều tầng => layer trên cùng ít node, cạnh đường dài (như cao tốc) để nhảy khoảng cách lớn thật nhanh => layer giữa nhiều node hơn, cạnh ngắn hơn (như quốc lộ) => layer 0 chứa toàn bộ node, kết nối dày với hàng xóm gần (như đường phố) để tìm chính xác => search vào từ một entry point ở layer cao nhất => greedy đi tới node gần query nhất (dùng cao tốc tới đúng "vùng") => rồi tụt xuống layer dưới và lặp lại => tới layer 0 chạy beam search rộng ef_search để lấy k kết quả => nhờ có tầng, độ phức tạp trung bình ≈ O(log N) => O(log N) là lý do HNSW scale từ nghìn tới tỷ vector mà latency chỉ tăng theo log

## CHAIN — HNSW "aha" small-world (rẽ từ "cạnh đường dài")
vì sao HNSW cần cạnh đường dài => thử đồ thị chỉ nối hàng xóm gần (k-NN graph thuần) → greedy search cho recall@5 = 0.0 => vì greedy bị kẹt trong cụm sai (entry point ở cụm khác, không có cao tốc để nhảy sang cụm của query) => chỉ cần thêm vài cạnh đường dài (long-range edges) → greedy thoát bẫy → recall@5 = 1.0 => long-range edges chính là tính chất "small-world" (chữ S, W trong HNSW) => đây chính là động lực thiết kế các tầng trên với cạnh dài của HNSW

## CHAIN — HNSW update/delete (rẽ từ "nhược điểm HNSW")
HNSW không có thao tác "xóa node" sạch sẽ => nên đa số implementation dùng soft-delete rồi rebuild định kỳ => workload ghi/xóa nhiều (vd chat memory prune liên tục) sẽ đau => nên cân nhắc IVF-PQ (rebuild-friendly hơn) hoặc lập lịch rebuild index

## CHAIN — pgvector & database hỗ trợ vector search (rẽ từ phân loại deployment)
DB truyền thống thêm khả năng vector gọi là "database hỗ trợ vector search" => nó lưu vector như một column (blob/array/UDT) rồi cắm add-on/plugin để search => trên Postgres, extension đúng là pgvector (KHÔNG phải PostGIS — PostGIS là geospatial, xử lý toạ độ địa lý) => pgvector cung cấp kiểu dữ liệu vector(1536) => tạo HNSW index với vector_cosine_ops WITH (m, ef_construction) => toán tử `<=>` = cosine distance, `<->` = L2, `<#>` = inner product => điểm mạnh: JOIN với bảng khác + lọc `WHERE tenant_id=42` trong CÙNG một query SQL => giữ vector và dữ liệu quan hệ trong một hệ thống → một transaction, một nguồn sự thật, ít hạ tầng => nên pgvector đủ cho ~70% ứng dụng RAG (<10M vector, đã có Postgres) => nhược điểm: thường không nhanh/tối ưu bằng dedicated DB ở quy mô cực lớn

## CHAIN — dedicated vector DB đặc trưng (rẽ từ phân loại)
dedicated vector database sinh ra *chỉ để* làm vector search => nó dùng cấu trúc dữ liệu chuyên biệt: inverted index (IVF), product quantization (PQ), LSH, HNSW => tối ưu cho nearest-neighbor / similarity search / distance calc => scale được qua cluster, đặt tốc độ lên hàng đầu => cho phép tune parameter (index vs search) theo use case => ví dụ 2026: Pinecone, Qdrant, Weaviate, Milvus, Chroma

## CHAIN — khi nào NÊN / KHÔNG dùng dedicated (rẽ từ ra quyết định)
NÊN dùng dedicated vector DB khi vector search là core workload => hoặc khi >10–50M vector => hoặc cần latency thấp ổn định ở QPS cao => hoặc cần hybrid/filtered search mạnh => KHÔNG NÊN khi <1–10M vector và đã chạy Postgres → dùng pgvector (ít hạ tầng = ít lỗi = rẻ hơn) => KHÔNG NÊN khi bài toán thực ra là keyword/exact match → dùng BM25/Elasticsearch (tốt và rẻ hơn) => KHÔNG NÊN khi cần strong consistency/ACID/transaction phức tạp → hầu hết dedicated vector DB không có ACID, chỉ pgvector (trên Postgres) có => KHÔNG NÊN khi chỉ là prototype nhỏ → dùng Chroma/FAISS embedded => tóm lại staff biết nói "không"

## CHAIN — Hybrid search (rẽ từ "pure vector thua")
pure vector search thua khi query chứa tên riêng, mã sản phẩm, version number, ID => vì những thứ đó cần exact match mà embedding hay bỏ sót => nên production 2026 mặc định dùng hybrid search (không phải tùy chọn) => hybrid = vector (semantic) + BM25 (keyword) + metadata filter => vector bắt ý nghĩa, keyword bắt tên riêng/ID => Weaviate/Vespa/Qdrant hỗ trợ native, còn pgvector phải tự ghép

## CHAIN — Filtered / metadata search (rẽ từ "metadata filter")
filtered search kết hợp similarity với điều kiện metadata (vd "giống + `WHERE category='news' AND lang='vi'`") => nếu lọc SAU ANN (post-filter) có thể ra rỗng => nếu lọc TRONG lúc traverse (in-graph filtering như Qdrant/Weaviate) thì nhanh và đúng hơn 2–3 lần => đây là tính năng quyết định ở production, không phải chi tiết phụ

## CHAIN — Edge cases khác (mạch độc lập)
high-dimensional + dữ liệu không có cụm → ANN thoái hóa về gần brute-force → cân nhắc giảm chiều (dimensionality reduction) hoặc chấp nhận recall thấp => duplicate / near-duplicate vectors làm lệch phân bố cụm IVF → rebalance hoặc dedup trước => cold start của HNSW: entry point kém khi index còn nhỏ → nhiều implementation chọn entry ngẫu nhiên / nhiều entry để giảm bẫy

## CHAIN — Scale tỷ bản ghi & bottleneck memory (rẽ từ staff design)
ở tỷ bản ghi, bottleneck #1 là memory => 1 tỷ × 768 × 4 byte = ~3TB raw, chưa kể overhead đồ thị HNSW (mỗi node lưu M cạnh) => HNSW thuần ở scale này không nhét nổi vào RAM một máy => đòn bẩy 1: quantization trước tiên (PQ/binary giảm 4–40 lần), thường rẻ nhất và hiệu quả nhất => đòn bẩy 2: sharding (scale ngang) — chia index thành nhiều shard theo id/namespace, mỗi shard một node => query fan-out tới các shard rồi merge top-k (scatter-gather) => đòn bẩy 3: tách compute khỏi storage — ingest / index build / query scale độc lập (build nặng CPU/GPU, query nặng RAM/latency) => đòn bẩy 4: DiskANN — đặt đồ thị trên SSD, billion-scale trên ít máy, đổi chút latency lấy RAM rẻ hơn nhiều => đòn bẩy 5: replication — nhân bản mỗi shard cho QPS cao + fault tolerance

## CHAIN — Tail latency p99 (rẽ từ "bottleneck #2")
bottleneck #2 là tail latency (p99) => trung bình 20ms nhưng p99 200ms vẫn giết trải nghiệm => nguồn p99 cao: shard chậm nhất trong fan-out (straggler), GC pause, filter đắt => xử lý: hedged requests, cắt ef_search động, cân bằng shard

## CHAIN — Cost & TCO (rẽ từ staff)
về cost, managed (Pinecone serverless) ~$0.33/GB/tháng storage + phí read/write units => ở 10M vector ≈ vài chục USD/tháng (cạnh tranh) => nhưng ở 100M vector chi phí đội lên nhanh (hàng trăm–nghìn USD/tháng) => trong khi self-host Milvus/Qdrant có thể giữ dưới ~$100–800/tháng (đổi lại tốn công vận hành/DevOps) => quy tắc: managed thắng khi team nhỏ / muốn zero-ops / scale khó đoán => self-host thắng khi scale lớn ổn định và có năng lực DevOps => phải tính TCO (total cost of ownership) không chỉ giá sticker (lương kỹ sư vận hành cũng là chi phí)

## CHAIN — Reliability, index rebuild & failure modes (rẽ từ staff)
coi mỗi lần build index như một release artifact => tức version hóa embedding + index, validate recall@k và p99 trong CI/CD trước khi promote, rollback được về version trước => đổi embedding model = phải re-embed + rebuild toàn bộ => vì vector cũ và vector mới không so sánh được với nhau => failure modes cần giám sát: shard chết (mất một phần recall — hỏng âm thầm, không throw lỗi) => OOM khi HNSW phình => recall trôi (drift) khi phân bố dữ liệu đổi => hot shard

## CHAIN — Monitoring (rẽ từ reliability)
monitoring bắt buộc, quan trọng nhất là recall@k trên tập ground-truth => vì ANN hỏng âm thầm: hệ thống vẫn chạy nhưng trả kết quả sai mà không báo lỗi => ngoài recall còn giám sát p50/p95/p99 latency, QPS, memory/shard, index build time => và freshness (độ trễ từ lúc ghi tới lúc searchable)

## CHAIN — Ảnh hưởng tổ chức (rẽ từ staff)
giải thích cho stakeholder non-technical: vector search giúp sản phẩm hiểu *ý nghĩa* chứ không chỉ *từ khóa* => vd user gõ "áo ấm đi tuyết" vẫn ra áo phao dù mô tả không chứa đúng chữ đó => đánh đổi: nó gần đúng chứ không tuyệt đối chính xác => và chính xác / tốc độ / chi phí là ba núm phải cân theo túi tiền và kỳ vọng người dùng => về roadmap: chọn managed = ít headcount vận hành, nhanh ra thị trường, nhưng vendor lock-in + chi phí đội theo scale + không air-gap được (compliance nghiêm ngặt) => chọn self-host = làm chủ, rẻ ở scale lớn, nhưng cần đội DevOps + lịch rebuild => staff phải trình bày trade-off này cho leadership, không tự quyết trong im lặng

## CHAIN — System design 500M docs (rẽ, khung tư duy staff)
đề: semantic search 500M tài liệu, p99<100ms, 5000 QPS, filter theo tenant_id + language => bước 1 làm rõ requirement trước (điểm staff): recall mục tiêu (95%/99%), update thường hay gần tĩnh, đọc nhiều hay ghi nhiều, ngân sách, compliance (data residency) => vì requirement quyết định kiến trúc, không phải index => bước 2 ước lượng số: 500M × 768 × 4B ≈ 1.5TB raw → không nhét 1 máy → cần sharding + có thể quantization => bước 3 chọn index & metric: HNSW cho recall/latency tốt (hoặc IVF-PQ/DiskANN nếu memory là nút thắt), metric theo embedding model (thường cosine → normalize) => bước 4 kiến trúc: shard theo tenant_id (giải quyết luôn filter, mỗi tenant một namespace, tránh cross-tenant leak) => mỗi shard replicate ×2–3 cho QPS + HA => query = scatter-gather + merge top-k, in-graph filtering cho language => bước 5 đáp p99: cắt ef_search động, hedged requests chống straggler, cache query nóng, tách compute/storage => bước 6 vận hành: pipeline re-embed/rebuild coi như release có gate recall@k, monitor recall + p99 + freshness, kế hoạch khi đổi embedding model => bước 7 kết bằng trình bày trade-off, không phang "câu trả lời duy nhất đúng"

---

## BẢNG — 5 kiểu của bài gốc + đính chính (🛠️ = staff phải sửa)
| Kiểu (bài gốc) | Ý tưởng | Đính chính / lưu ý |
|---|---|---|
| **In-memory** (RedisAI, TorchServe) | lưu vector trong RAM → đọc/ghi cực nhanh | 🛠️ TorchServe KHÔNG phải vector DB (model-serving framework PyTorch); RedisAI là runtime tensor/inference, đã ngừng phát triển. Hiện đại: **Redis Stack / RediSearch** với HNSW |
| **Disk-based** (Annoy, Milvus, ScaNN) | lưu vector trên disk khi vượt RAM | 🛠️ Annoy & ScaNN là *library* không phải database; Annoy = tree-based (random projection forest), memory-map, index **tĩnh**. Chỉ **Milvus** là database đầy đủ |
| **Distributed** (FAISS, Elasticsearch, Dask-ML) | trải data qua nhiều node | 🛠️ **FAISS là library chạy 1 máy (có GPU)** — gọi "distributed database" là sai. FAISS là engine để người ta *xây* DB phân tán lên trên (Milvus dùng FAISS bên trong) |
| **Graph-based** (Neo4j, Neptune, TigerGraph) | mô hình hóa data thành đồ thị node/edge | ⚠️ đây là **graph *database*** (lưu quan hệ thực thể) + vector column — KHÁC **graph-based *index*** như HNSW (đồ thị làm cấu trúc index). Đừng lẫn 2 chữ "graph" |
| **Time-series** (InfluxDB, TimescaleDB, Prometheus) | data theo thời gian, lưu kèm vector | time-series DB đính kèm vector, hợp anomaly detection, nhưng không phải "vector database" theo nghĩa chính |

## BẢNG — 3 trục phân loại ĐÚNG (staff dùng thực chiến)
| Trục | Các lựa chọn | Ý nghĩa |
|---|---|---|
| **Index algorithm** (quan trọng nhất) | Flat/brute-force, IVF, PQ/IVF-PQ, LSH, HNSW, DiskANN | quyết định recall / latency / memory |
| **Deployment / kiến trúc** | dedicated DB (Pinecone/Qdrant/Milvus/Weaviate/Chroma) · extension (pgvector, Elasticsearch/OpenSearch/MongoDB/Redis) · embedded library (FAISS/Annoy/hnswlib/LanceDB) | = trục "dedicated vs supports-vector-search" của bài gốc |
| **Mô hình vận hành** | managed/serverless (Pinecone, Zilliz Cloud) vs self-hosted | ai lo hạ tầng |

## BẢNG — Dedicated vector DB vs Database hỗ trợ vector search
| Tiêu chí | Dedicated vector DB | DB hỗ trợ vector search |
|---|---|---|
| Mục đích | sinh ra *chỉ để* vector search | DB truyền thống *thêm* khả năng vector |
| Cấu trúc | IVF / PQ / LSH / HNSW chuyên biệt | vector là 1 column (blob/array/UDT) + add-on |
| Ưu | tối ưu NN, scale qua cluster, tune được, tốc độ hàng đầu | vector + relational cùng hệ, 1 transaction, 1 nguồn sự thật, ít hạ tầng |
| Nhược | thêm một hệ phải nuôi, thường không ACID | không nhanh/tối ưu bằng dedicated ở scale cực lớn |
| Ví dụ | Pinecone, Qdrant, Weaviate, Milvus, Chroma | PostgreSQL+pgvector, Elasticsearch/OpenSearch, MongoDB Atlas Vector Search, Redis, SingleStore, Cassandra |

## BẢNG — So sánh index (thứ interviewer yêu thích)
| Index | Recall | Query speed | Memory | Update/Delete | Build time | Khi nào dùng |
|---|---|---|---|---|---|---|
| **Flat / brute-force** | 100% (exact) | chậm O(N) | trung bình | dễ | 0 | N nhỏ (<~100k), cần exact, làm ground-truth |
| **IVF** | trung bình–cao (tune `nprobe`) | nhanh | trung bình | khá dễ | cần train | dataset lớn, cần cân bằng |
| **IVF-PQ** | trung bình (lossy) | rất nhanh | **rất thấp** ⭐ | rebuild-friendly hơn HNSW | cần train | **billion-scale**, RAM là nút thắt |
| **LSH** | thấp–trung bình | nhanh | cao (nhiều bảng) | khá dễ | nhanh | ít dùng cho dense hiện nay |
| **HNSW** | **cao–rất cao** ⭐ (tune `ef`) | **rất nhanh** ⭐ O(log N) | **cao** (lưu đồ thị) | **kém** (soft-delete + rebuild) | chậm | mặc định cho đa số production |
| **DiskANN** | cao | trung bình (đọc SSD) | thấp (graph trên SSD) | khá | chậm | billion-scale trên **một máy** ít RAM |

## BẢNG — 3 tham số HNSW
| Tham số | Điều khiển | Tăng lên thì... |
|---|---|---|
| **`M`** | số cạnh/hàng xóm mỗi node | recall ↑, **memory ↑**, build chậm hơn |
| **`ef_construction`** | độ rộng beam khi *xây* index | recall ↑, **build time ↑**, không ảnh hưởng query |
| **`ef_search`** | độ rộng beam khi *query* | recall ↑, **latency ↑** (núm chỉnh lúc runtime) |

> Default thực chiến 2026: `M=32`, `ef_construction=400`, query bắt đầu `ef_search=100` rồi đẩy lên tới trần latency; recall <~95% thì rebuild với `M=48/64`.

## BẢNG — Đính chính bài gốc (phụ lục — để không học nhầm)
| Bài gốc nói | Đúng ra là |
|---|---|
| "PostGIS cho vector" | **SAI** — PostGIS là geospatial; extension đúng là **pgvector** (và **pgvectorscale**) |
| TorchServe là in-memory vector DB | **SAI** — model-serving framework của PyTorch |
| FAISS là "distributed database" | **SAI** — *library* chạy 1 máy (có GPU); người ta xây DB phân tán *lên trên* nó |
| Annoy/ScaNN là "database" | chưa chuẩn — chúng là *library* (Annoy = tree-based, index tĩnh) |
| RedisAI là câu chuyện Redis-vector hiện tại | lỗi thời — nay là **Redis Stack / RediSearch** (HNSW) |
| Taxonomy 5 kiểu | lỏng & trộn trục — thực chiến phân theo index algorithm + deployment + managed/self-host |
| (bỏ qua) ANN, recall, HNSW, hybrid search, dedicated DB mới | đó lại là phần quan trọng nhất năm 2026 |

> Phần đúng & giá trị nhất của bài gốc: **phân biệt dedicated vs database hỗ trợ vector search** + danh sách đặc trưng của dedicated DB.

---

## TỪ KHÓA MỒI
- Mạch chính: **keyword vs meaning → O(N×d) → ANN**
- Distance metric: **cosine đo góc / normalize**
- Curse of dimensionality: **mọi điểm cách đều**
- IVF: **nlist/nprobe + train() trước add()**
- PQ / quantization: **768 float → 96 byte (~32×)**
- LSH: **nhiều bảng hash → bị HNSW vượt**
- HNSW cơ chế: **cao tốc / quốc lộ / đường phố, O(log N)**
- HNSW aha: **k-NN thuần 0.0 vs small-world 1.0**
- HNSW update/delete: **soft-delete + rebuild**
- pgvector: **`<=>` cosine, JOIN + WHERE cùng query**
- Dedicated đặc trưng: **chỉ để vector search, tune được**
- Nên/không dedicated: **staff biết nói không**
- Hybrid search: **vector + BM25 + filter**
- Filtered search: **in-graph filtering**
- Edge cases khác: **duplicate / cold start**
- Scale & memory: **~3TB → quantization + sharding**
- Tail latency: **p99 straggler**
- Cost & TCO: **$0.33/GB, 100M đội nhanh**
- Reliability: **index rebuild là release**
- Monitoring: **recall@k hỏng âm thầm**
- Tổ chức: **áo ấm đi tuyết → áo phao**
- System design: **shard theo tenant_id**

## ĐÃ PHỦ
- **Vấn đề & nền tảng:** keyword vs semantic match, SQL/B-tree exact+range bất lực với ý nghĩa ✓
- **Định nghĩa:** vector/embedding (384/768/1536), dimension, similarity/NN search (nghiệp vụ trung tâm), index (cấu trúc phụ) ✓
- **Distance metric:** L2/Euclidean (công thức chim bay), cosine (góc, phổ biến text), dot/inner product, mẹo normalize→inner product, hại khi sai metric ✓
- **Brute-force:** flat/exhaustive, exact 100%, O(N×d), 100M×768≈76.8 tỷ phép nhân, <100ms vs QPS ✓
- **ANN & recall:** đánh đổi chính xác lấy tốc độ, nhanh trăm–nghìn lần, recall@k công thức, đường cong recall↔latency↔memory ✓
- **Curse of dimensionality:** KD-tree sụp ở 100+ chiều, mọi điểm cách đều, ANN khai thác cụm, intrinsic dimensionality/manifold, uniform random không cứu được, benchmark synthetic lừa dối ✓
- **IVF:** nlist/nprobe, k-means, O(nprobe×N/nlist×d), train() trước add() (lỗi kinh điển), recall@5=1.0 nprobe=8 ✓
- **PQ:** ~3TB math, m đoạn/codebook 256→1 byte, 768 float→96 byte ~32×, lossy, IVF-PQ billion-scale, binary/scalar quantization Qdrant ~40× ✓
- **LSH:** hàm hash cùng bucket, provable guarantees, nhiều bảng hash tốn memory/khó tune, bị HNSW vượt ✓
- **HNSW cơ chế:** Malkov&Yashunin 2016, layer cao tốc/quốc lộ/đường phố, greedy+beam search, ef_search, O(log N) ✓
- **HNSW aha small-world:** k-NN thuần recall 0.0 (kẹt cụm), long-range edges → 1.0, chữ S,W, động lực thiết kế tầng ✓
- **HNSW 3 tham số** (bảng): M, ef_construction, ef_search + default 2026 ✓
- **HNSW update/delete:** không xóa sạch, soft-delete+rebuild, workload ghi/xóa đau, cân nhắc IVF-PQ/lịch rebuild ✓
- **Phân loại bài gốc 5 kiểu + đính chính** (bảng): in-memory/disk/distributed/graph/time-series, TorchServe/RedisAI/Annoy/ScaNN/FAISS/graph-index sai ✓
- **3 trục phân loại đúng** (bảng): index algorithm / deployment / managed-self-host ✓
- **Dedicated vs supports** (bảng): đặc trưng, ưu/nhược, ví dụ hai bên ✓
- **pgvector:** vector(1536), HNSW vector_cosine_ops, toán tử `<=>`/`<->`/`<#>`, JOIN+WHERE, 70% RAG, PostGIS sai/pgvectorscale ✓
- **Chroma/FAISS code facts:** FAISS IndexFlatL2/IndexIVFFlat train-add, Chroma HNSW cosine mặc định tự sinh embedding → gộp vào chain IVF + nên/không dedicated ✓
- **Khi nào nên/không dedicated:** core workload/scale, pgvector <1–10M, BM25 cho keyword, ACID chỉ pgvector, prototype Chroma/FAISS ✓
- **Hybrid search:** pure vector thua tên riêng/ID/version, vector+BM25+filter, native vs tự ghép ✓
- **Filtered search:** post-filter rỗng vs in-graph filtering nhanh/đúng 2–3× ✓
- **Edge cases khác:** high-dim không cụm→giảm chiều, duplicate→dedup, HNSW cold start→nhiều entry ✓
- **Scale tỷ:** ~3TB + overhead graph, 5 đòn bẩy (quantization/sharding/tách compute-storage/DiskANN/replication), scatter-gather ✓
- **Tail latency p99:** 20ms tb vs 200ms p99, straggler/GC/filter, hedged requests/cắt ef_search/cân bằng shard ✓
- **Cost & TCO:** $0.33/GB/tháng, 10M vài chục USD, 100M đội nhanh, self-host $100–800, managed vs self-host, TCO ✓
- **Reliability:** index rebuild là release/CI-CD/rollback, re-embed khi đổi model, shard chết/OOM/drift/hot shard ✓
- **Monitoring:** recall@k ground-truth (hỏng âm thầm), p50/p95/p99, QPS, memory/shard, build time, freshness ✓
- **Tổ chức:** giải thích stakeholder (áo ấm→áo phao), 3 núm, roadmap managed vs self-host (lock-in/air-gap/DevOps) ✓
- **System design 500M:** 7 bước từ requirement → 1.5TB → HNSW/metric → shard tenant_id+replicate → p99 → vận hành → trade-off ✓
- *Bỏ có chủ đích (không phải kiến thức):* analogy thư viện/thủ thư, self-check, one-liner tiếng Anh (chỉ diễn đạt lại ý đã phủ).
