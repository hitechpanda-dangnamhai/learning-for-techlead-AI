# Giáo trình: Các loại Vector Database (Types of Vector Databases)

> Biên soạn theo phong cách "cầm tay chỉ việc" — từ zero đến tư duy Staff Engineer.
> Giảng bằng tiếng Việt, giữ nguyên technical terms tiếng Anh để phục vụ phỏng vấn big tech.
>
> **Lưu ý quan trọng ngay từ đầu:** Bài giảng gốc (một transcript kiểu course intro) có **vài chỗ sai/lỗi thời** mà một staff engineer bắt buộc phải nhận ra. Tôi sẽ giữ lại phần đúng, **đính chính phần sai**, và mở rộng bằng bức tranh thực tế năm 2026. Những chỗ đính chính tôi đánh dấu 🛠️ **[Đính chính]** và mở rộng đánh dấu ➕ **[Mở rộng ngoài bài gốc]**.

---

## Phần 0 — 🗺️ Bản đồ bài học (Overview)

Bài này trả lời một câu hỏi cực kỳ thực tế trong kỷ nguyên AI/LLM: **"Khi tôi có hàng triệu, hàng tỷ vector (embedding) và cần tìm những vector *giống nhất* với một câu query, tôi lưu và tìm chúng ở đâu, bằng cách nào?"** Đó chính là bài toán mà **vector database** sinh ra để giải. Bài giảng gốc phân loại vector database thành 5 kiểu (in-memory, disk-based, distributed, graph-based, time-series) và phân biệt *dedicated vector database* với *database that supports vector search*. Chúng ta sẽ đi qua toàn bộ điều đó, nhưng đào sâu tới tận thuật toán bên trong (HNSW, IVF, PQ, LSH) và cách vận hành ở quy mô tỷ bản ghi.

**Sau khi học xong, bạn sẽ có thể:**
- Giải thích **tại sao** cần vector database thay vì dùng B-tree/SQL thông thường.
- Phân loại vector database theo **cách phân loại đúng đắn** (theo index algorithm + deployment model), chứ không chỉ theo taxonomy hơi lỏng lẻo của bài gốc.
- Phân biệt **dedicated vector DB** (Pinecone, Qdrant, Milvus, Weaviate...) với **database hỗ trợ vector search** (pgvector, Elasticsearch, MongoDB...).
- Tự implement được brute-force search, IVF, và greedy graph search — để *hiểu* thứ mà HNSW đang làm.
- Phân tích trade-off **recall vs latency vs memory vs cost** — thứ mà interviewer big tech luôn hỏi.
- Thiết kế một hệ thống semantic search / RAG ở quy mô lớn và biết bottleneck nằm ở đâu.

**Mạch kiến thức từ basic → staff:**
- 🟢 **Basic:** Embedding là gì → similarity search là gì → brute-force k-NN (tính bằng tay + code).
- 🟡 **Intermediate:** Tại sao brute-force chết ở quy mô lớn → ANN (Approximate Nearest Neighbor) → dùng thư viện thật (FAISS/Chroma/pgvector) → dedicated vs supports-vector-search.
- 🔴 **Advanced:** Bên trong các index — IVF, Product Quantization, LSH, và **HNSW** (ông vua hiện tại). Big-O, trade-off, tự implement.
- 🟣 **Staff:** Thiết kế ở tỷ scale, sharding, cost/latency/reliability, hybrid search, khi nào KHÔNG nên dùng, giải thích cho stakeholder.
- 🎯 **Cheatsheet:** Keywords, core concepts, code phải thuộc, câu hỏi phỏng vấn + one-liners.

---

## Phần 1 — 🟢 BASIC (Nền tảng)

### 1.1. Vấn đề thực tế trước — "tại sao"

Hãy bắt đầu bằng một vấn đề bạn gặp hàng ngày mà không để ý.

Bạn gõ vào Google/Spotify/Netflix: *"nhạc buồn để nghe lúc mưa"*. Bạn **không** gõ đúng tên bài hát. Nhưng hệ thống vẫn trả về đúng thứ bạn muốn. Làm sao? Nó không so khớp **từ khóa** (keyword match), nó so khớp **ý nghĩa** (meaning / semantic).

Database truyền thống (MySQL, PostgreSQL với index B-tree) rất giỏi trả lời câu hỏi *"tìm hàng có `email = 'a@x.com'`"* — tức so khớp **chính xác** (exact match) hoặc **theo thứ tự** (range: `age > 30`). Nhưng nó **bất lực** với câu hỏi *"tìm những thứ **giống về mặt ý nghĩa** với cái này"*. Đó là lỗ hổng mà vector database lấp vào.

### 1.2. Analogy đời thường

Tưởng tượng một **thư viện khổng lồ**, nhưng thay vì xếp sách theo bảng chữ cái (exact match), thủ thư xếp theo **chủ đề và cảm xúc**: sách về "tình yêu tuổi trẻ" đứng gần sách "hoài niệm mùa hè", và đứng xa sách "kế toán doanh nghiệp". Khi bạn cầm một cuốn và nói "cho tôi 5 cuốn *giống cuốn này*", thủ thư chỉ cần với tay ra khu vực xung quanh. **Vị trí trên kệ = ý nghĩa.** Vector database chính là "thư viện" đó, và "vị trí trên kệ" chính là **vector**.

### 1.3. Các thuật ngữ nền tảng (giải thích ngay lần đầu)

- **Vector / Embedding:** một dãy số (ví dụ 384, 768, hay 1536 con số) biểu diễn ý nghĩa của một object (câu văn, ảnh, audio). Một model AI (embedding model) biến "con mèo" thành ví dụ `[0.12, -0.83, ..., 0.05]`. Điểm mấu chốt: **những thứ giống nhau về ý nghĩa → vector gần nhau trong không gian**.
- **Dimension (số chiều):** độ dài của vector. Vector 1536 chiều = một điểm trong không gian 1536 chiều. (Đừng cố tưởng tượng hình học 1536 chiều — não người dừng ở 3 chiều; cứ coi nó là "một dãy 1536 số".)
- **Similarity search / Nearest Neighbor search:** cho một query vector, tìm k vector gần nhất trong database. Đây là **nghiệp vụ trung tâm** của mọi vector database.
- **Distance metric (thước đo khoảng cách):** cách đo "gần/xa". Ba loại hay gặp:
  - **Euclidean / L2:** khoảng cách đường chim bay giữa 2 điểm.
  - **Cosine similarity:** đo **góc** giữa 2 vector (bỏ qua độ dài) — phổ biến nhất cho text embedding.
  - **Dot product / Inner product:** tích vô hướng.
- **Index:** một cấu trúc dữ liệu phụ giúp tìm kiếm nhanh hơn là quét từng phần tử. (Giống mục lục sách.)

### 1.4. Ví dụ chạy tay (làm thủ công để "thấy" cơ chế)

Giả sử database chỉ có 3 "tài liệu", mỗi cái là vector **2 chiều** cho dễ hình dung:

| Tài liệu | Vector |
|----------|--------|
| A ("mèo con dễ thương") | `[1.0, 1.0]` |
| B ("chú chó nhỏ")        | `[0.9, 1.1]` |
| C ("báo cáo tài chính")  | `[-1.0, 0.2]` |

Query: **q = `[1.0, 0.9]`** ("thú cưng đáng yêu").

Tính **Euclidean distance** `d = √((x₁-x₂)² + (y₁-y₂)²)`:
- `d(q, A) = √((1.0-1.0)² + (0.9-1.0)²) = √(0 + 0.01) = 0.10` ✅ gần nhất
- `d(q, B) = √((1.0-0.9)² + (0.9-1.1)²) = √(0.01 + 0.04) = √0.05 ≈ 0.224`
- `d(q, C) = √((1.0-(-1.0))² + (0.9-0.2)²) = √(4 + 0.49) = √4.49 ≈ 2.119` ❌ xa nhất

→ Kết quả xếp hạng: **A → B → C**. Đúng trực giác: query về "thú cưng" gần "mèo" và "chó", xa "báo cáo tài chính". **Đó chính xác là điều vector database làm** — chỉ khác là với hàng triệu vector và hàng nghìn chiều.

### 1.5. Code "hello world" (đã chạy thật, Python)

```python
import numpy as np

# 3 "tài liệu" trong database, mỗi cái là 1 vector
db = {
    "A_meo":     np.array([1.0, 1.0]),
    "B_cho":     np.array([0.9, 1.1]),
    "C_taichinh":np.array([-1.0, 0.2]),
}
query = np.array([1.0, 0.9])   # "thú cưng đáng yêu"

# Brute-force: tính khoảng cách tới TỪNG vector rồi sắp xếp
results = []
for name, vec in db.items():
    dist = np.linalg.norm(vec - query)   # Euclidean/L2 distance
    results.append((name, dist))

results.sort(key=lambda x: x[1])          # gần nhất lên đầu
for name, dist in results:
    print(f"{name}: distance = {dist:.3f}")

# Output:
# A_meo: distance = 0.100
# B_cho: distance = 0.224
# C_taichinh: distance = 2.119
```

Giải thích 2 dòng quan trọng nhất:
- `np.linalg.norm(vec - query)` → tính độ dài của vector hiệu = khoảng cách Euclidean. Đây là **trái tim** của similarity search.
- `results.sort(...)` → sắp xếp tăng dần theo distance, phần tử đầu = nearest neighbor.

Cách làm này gọi là **brute-force** (hay **flat / exhaustive search**): so với **tất cả** phần tử. Nó cho kết quả **chính xác 100%** (exact search), nhưng — nhớ kỹ chi tiết này, nó sẽ ám ảnh cả bài — **độ phức tạp là O(N × d)**: N phần tử, mỗi phần tử d chiều. Với 3 vector thì tức thì; với 1 tỷ vector thì... chết.

### 1.6. ✅ Self-check (Basic)

1. Vì sao B-tree index trong SQL không giải quyết được bài toán "tìm thứ *giống* cái này"? *(Gợi ý: B-tree tối ưu cho exact/range match trên giá trị có thứ tự, không cho "độ gần trong không gian nhiều chiều".)*
2. Cosine similarity khác Euclidean distance ở điểm gì? *(Gợi ý: cosine đo **góc**, bỏ qua độ dài vector.)*
3. Brute-force search có độ phức tạp bao nhiêu, và vì sao nó không scale? *(O(N×d) — tuyến tính theo số lượng và số chiều.)*

---

## Phần 2 — 🟡 INTERMEDIATE (Vận dụng)

### 2.1. "Tại sao" của tầng này: brute-force chết ở quy mô lớn

Ở Basic ta có `O(N × d)`. Hãy nhét số thật vào để thấy nỗi đau. Giả sử N = 100 triệu vector, d = 768 chiều, mỗi query cần tính 100M × 768 ≈ **76.8 tỷ phép nhân**. Trên một CPU, một query có thể mất **hàng giây tới hàng chục giây**. Người dùng mong đợi **< 100ms**. Nhân với hàng nghìn query/giây (QPS) → bất khả thi. Đây là lý do sinh ra ý tưởng lớn nhất của cả lĩnh vực:

➕ **[Mở rộng ngoài bài gốc — điều bài gốc KHÔNG nói nhưng cực kỳ quan trọng]**
> **ANN — Approximate Nearest Neighbor.** Ta **đánh đổi độ chính xác lấy tốc độ**. Thay vì tìm *đúng* 5 hàng xóm gần nhất (exact), ta chấp nhận tìm *gần đúng* — có thể 4/5 hoặc 5/5 đúng — nhưng **nhanh hơn hàng trăm, hàng nghìn lần**. Chỉ số đo độ "đúng gần" này gọi là **recall**: `recall@k = (số kết quả đúng trong top-k) / k`. Toàn bộ ngành vector database xoay quanh **đường cong đánh đổi recall ↔ latency ↔ memory**. Bài giảng gốc gần như không nhắc tới ANN hay recall — nhưng đó mới là **linh hồn** của chủ đề này. Interviewer big tech sẽ hỏi thẳng: *"exact hay approximate? trade-off là gì?"*

### 2.2. Cách phân loại của bài gốc — và cách phân loại ĐÚNG hơn

Bài gốc chia 5 kiểu: **in-memory, disk-based, distributed, graph-based, time-series**. Ta điểm qua để bạn hiểu khi gặp lại, nhưng kèm đính chính:

| Kiểu (theo bài gốc) | Ý tưởng | 🛠️ Đính chính / lưu ý của staff |
|---|---|---|
| **In-memory** (RedisAI, TorchServe) | Lưu vector trong RAM → đọc/ghi cực nhanh | 🛠️ **TorchServe KHÔNG phải vector database** — nó là model-serving framework của PyTorch. RedisAI cũng là runtime chạy tensor/inference, và **đã ngừng phát triển tích cực**. Câu chuyện vector-in-Redis hiện đại là **Redis Stack / RediSearch** với HNSW. |
| **Disk-based** (Annoy, Milvus, ScaNN) | Lưu vector trên disk cho dataset vượt RAM | 🛠️ **Annoy và ScaNN là *thư viện (library)*, không phải database.** Annoy = tree-based (random projection forest), memory-map file, index **tĩnh** (không update dễ). Milvus mới đúng là một database đầy đủ. |
| **Distributed** (FAISS, Elasticsearch, Dask-ML) | Trải data qua nhiều node → scale ngang | 🛠️ **FAISS là thư viện chạy trên *một máy* (có hỗ trợ GPU), gọi nó là "distributed database" là sai.** FAISS là *engine* mà người ta *xây* database phân tán lên trên (Milvus dùng FAISS bên trong). |
| **Graph-based** (Neo4j, Neptune, TigerGraph) | Mô hình hóa dữ liệu thành đồ thị node/edge | ⚠️ **Bẫy khái niệm:** đây là **graph *database*** (lưu quan hệ giữa thực thể) có thêm vector column — **KHÁC hoàn toàn** với **graph-based *index*** như HNSW (đồ thị được dùng làm *cấu trúc index* để search nhanh). Đừng lẫn hai chữ "graph". |
| **Time-series** (InfluxDB, TimescaleDB, Prometheus) | Dữ liệu theo thời gian, lưu kèm vector | Đây là time-series DB có thể đính kèm vector — hợp lý cho anomaly detection, nhưng không phải "vector database" theo nghĩa chính. |

➕ **[Mở rộng — cách phân loại mà một staff engineer thực sự dùng]**
> Taxonomy "5 kiểu theo nơi lưu" của bài gốc **lỏng và trộn lẫn các trục phân loại**. Trong thực chiến, ta phân loại theo **3 trục độc lập**:
>
> 1. **Theo index algorithm** (quan trọng nhất): Flat/brute-force, **IVF**, **PQ / IVF-PQ**, **LSH**, **HNSW**, **DiskANN**. Đây mới là thứ quyết định recall/latency/memory.
> 2. **Theo deployment/kiến trúc:** *dedicated vector DB* (Pinecone, Qdrant, Milvus, Weaviate, Chroma) vs *extension trên DB có sẵn* (pgvector cho Postgres, vector search của Elasticsearch/OpenSearch/MongoDB/Redis) vs *embedded library* (FAISS, Annoy, hnswlib, LanceDB).
> 3. **Theo mô hình vận hành:** *managed/serverless* (Pinecone, Zilliz Cloud) vs *self-hosted*.
>
> Trục 2 chính là phần "dedicated vs supports-vector-search" mà bài gốc nói — và đó là phần **đúng và giá trị nhất** của bài gốc.

### 2.3. Dedicated vector DB vs Database hỗ trợ vector search (phần lõi của bài gốc)

Đây là phần bài gốc trình bày tốt, tôi giữ và làm rõ:

**Dedicated vector database** — sinh ra *chỉ để* làm vector search. Đặc trưng:
- Dùng cấu trúc dữ liệu chuyên biệt: **inverted index (IVF)**, **product quantization (PQ)**, **locality-sensitive hashing (LSH)**, và HNSW.
- Tối ưu cho nearest-neighbor / similarity search / distance calc.
- Scale được qua cluster, đặt **tốc độ lên hàng đầu**, cho phép **tune parameter** (index vs search) theo use case.
- Ví dụ 2026: **Pinecone, Qdrant, Weaviate, Milvus, Chroma** (bài gốc nhắc FAISS/Annoy/Milvus — nhưng FAISS/Annoy là library, và Pinecone/Qdrant/Weaviate/Chroma mới là các dedicated DB nổi bật nhất hiện nay).

**Database hỗ trợ vector search** — DB truyền thống *thêm* khả năng vector:
- Lưu vector như một column (blob/array/UDT), cắm thêm add-on/plugin để search.
- Ưu điểm: **giữ vector và dữ liệu quan hệ trong cùng một hệ thống** → một transaction, một nguồn sự thật, ít hạ tầng phải quản.
- Nhược điểm: thường **không nhanh/tối ưu bằng** dedicated DB ở quy mô cực lớn.
- Ví dụ: **PostgreSQL + pgvector**, **Elasticsearch/OpenSearch**, **MongoDB Atlas Vector Search**, **Redis**, **SingleStore**, **Cassandra**.

🛠️ **[Đính chính quan trọng — lỗi rõ ràng trong bài gốc]**
> Bài gốc viết *"PostgreSQL includes its **PostGIS** add-on for vectors in space"*. **Sai.** **PostGIS là extension *geospatial*** (xử lý toạ độ địa lý, hình học 2D/3D — kiểu "tìm nhà hàng trong bán kính 2km"), **không liên quan gì tới vector embedding của AI.** Extension đúng cho vector similarity search trên Postgres là **`pgvector`** (và bản tối ưu hiệu năng **`pgvectorscale`**). Nếu bạn nói "PostGIS" trong buổi phỏng vấn về vector search, interviewer sẽ biết ngay bạn học từ tài liệu cũ/sai.

### 2.4. Bên trong hoạt động thế nào (giới thiệu các index & tham số)

Ba "họ" index bạn phải biết trước khi vào Advanced:

- **IVF (Inverted File Index):** chia không gian thành `nlist` cụm (clusters) bằng k-means. Khi search, chỉ quét `nprobe` cụm gần query nhất thay vì toàn bộ. Tham số: **`nlist`** (số cụm), **`nprobe`** (số cụm quét lúc query). `nprobe` cao → recall cao, chậm hơn.
- **PQ (Product Quantization):** **nén** vector (ví dụ 768 float → 96 byte) để tiết kiệm RAM khủng khiếp. Đánh đổi: mất chút chính xác (lossy). Thường ghép với IVF thành **IVF-PQ** cho billion-scale.
- **HNSW (Hierarchical Navigable Small World):** index đồ thị nhiều tầng — **ông vua hiện tại**, mặc định trong hầu hết dedicated DB. Sẽ mổ xẻ ở Advanced. Tham số: **`M`**, **`ef_construction`**, **`ef_search`**.

### 2.5. Code thực tế hơn (gần với công việc thật)

Ở production bạn hiếm khi tự viết brute-force. Bạn dùng library/DB. Dưới đây là 3 ví dụ ở 3 mức độ "gần với đời thật".

**(a) FAISS — engine ANN kinh điển của Meta (chạy trên 1 máy, có GPU):**

```python
# pip install faiss-cpu   (hoặc faiss-gpu)
import faiss
import numpy as np

d = 128                      # số chiều embedding
nb = 100_000                 # số vector trong database
np.random.seed(0)
db = np.random.rand(nb, d).astype('float32')
query = np.random.rand(1, d).astype('float32')

# --- Cách 1: FLAT = brute-force, exact, recall 100% nhưng O(N) ---
index_flat = faiss.IndexFlatL2(d)     # L2 distance
index_flat.add(db)                     # nạp toàn bộ vector
D, I = index_flat.search(query, k=5)   # trả về distance D và index I của top-5
print("Exact  :", I[0])

# --- Cách 2: IVF = approximate, nhanh hơn nhiều ---
nlist = 256                            # chia thành 256 cụm
quantizer = faiss.IndexFlatL2(d)
index_ivf = faiss.IndexIVFFlat(quantizer, d, nlist)
index_ivf.train(db)                    # BẮT BUỘC: học các centroid (k-means)
index_ivf.add(db)
index_ivf.nprobe = 8                   # chỉ quét 8/256 cụm → tune recall/speed
D, I = index_ivf.search(query, k=5)
print("Approx :", I[0])
```
Điểm phải nhớ khi làm thật: IVF **cần `train()`** (học centroid) trước khi `add()`. Quên train là một lỗi kinh điển. Chỉnh **`nprobe`** để dịch chuyển trên đường cong recall↔speed.

**(b) pgvector — vector search ngay trong PostgreSQL (giữ chung với data quan hệ):**

```sql
-- Cài extension (một lần)
CREATE EXTENSION IF NOT EXISTS vector;

-- Bảng lưu tài liệu KÈM embedding 1536 chiều (ví dụ OpenAI/embedding model)
CREATE TABLE documents (
    id      bigserial PRIMARY KEY,
    content text,
    embedding vector(1536)          -- kiểu dữ liệu vector do pgvector cung cấp
);

-- Tạo HNSW index để search nhanh (vector_cosine_ops = dùng cosine distance)
CREATE INDEX ON documents
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);

-- Query: tìm 5 tài liệu GẦN NHẤT với query embedding.
-- Toán tử <=> = cosine distance trong pgvector (<-> là L2, <#> là inner product)
SELECT id, content
FROM documents
ORDER BY embedding <=> '[0.12, -0.34, ...]'   -- query vector
LIMIT 5;
```
Đây là kiểu "database hỗ trợ vector search" — điểm mạnh: bạn `JOIN` được với bảng `users`, lọc `WHERE tenant_id = 42` **trong cùng một query SQL**. Cực tiện cho ~70% ứng dụng RAG có dưới ~10M vector.

**(c) Chroma — dedicated DB "developer-friendly" cho prototyping:**

```python
# pip install chromadb
import chromadb

client = chromadb.Client()
col = client.create_collection("docs")   # HNSW bên trong, cosine mặc định

# Chroma có thể TỰ sinh embedding, hoặc bạn đưa vector sẵn.
col.add(
    ids=["1", "2", "3"],
    documents=["mèo con dễ thương", "chú chó nhỏ", "báo cáo tài chính"],
)
res = col.query(query_texts=["thú cưng đáng yêu"], n_results=2)
print(res["documents"])   # -> [['mèo con dễ thương', 'chú chó nhỏ']]
```

### 2.6. 3 lỗi thường gặp (và cách tránh)

1. **Quên `train()` cho IVF / sai thứ tự train–add.** IVF phải học centroid trước. → Luôn `train()` trên một mẫu đại diện rồi mới `add()`.
2. **Dùng sai distance metric so với lúc tạo embedding.** Nếu embedding model được huấn luyện cho **cosine** mà bạn index bằng **L2** (không normalize), recall tụt thảm hại. → Thống nhất metric; nếu dùng cosine thì **normalize vector về độ dài 1** rồi có thể dùng inner product.
3. **Kỳ vọng update/delete rẻ trên HNSW.** HNSW **không có thao tác "xóa node" sạch sẽ** — đa số implementation *soft-delete* rồi rebuild định kỳ. Workload ghi/xóa nhiều (vd: chat memory prune liên tục) sẽ đau. → Cân nhắc IVF-PQ (rebuild-friendly hơn) hoặc lịch rebuild index.

### 2.7. ✅ Self-check (Intermediate)

1. ANN đánh đổi cái gì lấy cái gì? Recall được định nghĩa thế nào?
2. Kể 2 khác biệt cốt lõi giữa *dedicated vector DB* và *DB hỗ trợ vector search*. Khi nào chọn pgvector thay vì Pinecone?
3. Vì sao FAISS bị gọi sai là "distributed database" trong bài gốc? Nó thực chất là gì?

---

## Phần 3 — 🔴 ADVANCED (Chuyên sâu)

Bây giờ ta mở nắp capo. Bài gốc chỉ liệt kê tên "inverted index, product quantization, LSH" — ta sẽ *hiểu và tự dựng* chúng.

### 3.1. Bài toán gốc rễ: "Curse of Dimensionality"

Vì sao không dùng KD-tree (cây tìm kiếm không gian cổ điển)? Vì ở **số chiều cao (100+)**, mọi điểm gần như *cách đều* nhau — cấu trúc cây sụp đổ, phải duyệt gần hết cây, không nhanh hơn brute-force. Đây gọi là **curse of dimensionality**. ANN hiện đại (IVF, HNSW) né vấn đề này bằng cách khai thác **cấu trúc cụm** của dữ liệu thật (embedding đời thực có cụm, không rải đều).

➕ **[Mở rộng — điều staff phải biết]** ANN chỉ hiệu quả khi dữ liệu có **intrinsic dimensionality thấp** (dữ liệu thật thường nằm trên một "manifold" ít chiều hơn số chiều danh nghĩa). Với dữ liệu **ngẫu nhiên đều** (uniform random), *không* index nào cứu được recall — điều này tôi sẽ minh chứng bằng code ở dưới. Đây là lý do benchmark trên dữ liệu synthetic ngẫu nhiên thường *lừa dối*.

### 3.2. IVF (Inverted File) — tự implement từ đầu

Ý tưởng: **phân vùng** không gian bằng k-means thành `nlist` cụm. Query chỉ so với các vector trong `nprobe` cụm gần nhất. Big-O trung bình ≈ **O(nprobe × N/nlist × d)** thay vì O(N×d).

```python
import numpy as np
from sklearn.cluster import KMeans

class SimpleIVF:
    """IVF (Inverted File Index) tự implement để hiểu bản chất."""
    def __init__(self, nlist=64, nprobe=8):
        self.nlist = nlist      # số cụm
        self.nprobe = nprobe    # số cụm quét khi query

    def fit(self, db):
        self.db = db
        # 1) Học nlist centroid bằng k-means => "phân vùng" không gian
        self.km = KMeans(n_clusters=self.nlist, n_init=3, random_state=0).fit(db)
        self.centroids = self.km.cluster_centers_
        # 2) Xây "inverted list": cụm -> danh sách id vector thuộc cụm đó
        self.buckets = {i: np.where(self.km.labels_ == i)[0]
                        for i in range(self.nlist)}
        return self

    def search(self, q, k=5):
        # 3) Tìm nprobe centroid gần query nhất
        cdist = np.linalg.norm(self.centroids - q, axis=1)
        probe = np.argsort(cdist)[:self.nprobe]
        # 4) Chỉ gom vector trong các cụm đó (thay vì toàn bộ N)
        cand = np.concatenate([self.buckets[c] for c in probe])
        # 5) Brute-force TRONG tập ứng viên nhỏ này
        d = np.linalg.norm(self.db[cand] - q, axis=1)
        return cand[np.argsort(d)[:k]]

# --- Kiểm chứng trên dữ liệu CÓ CỤM (giống embedding thật) ---
from sklearn.datasets import make_blobs
X, _ = make_blobs(n_samples=5000, n_features=128, centers=50, random_state=0)
db = X.astype('float32')
q = db[123] + np.random.rand(128) * 0.1

def exact_topk(q, db, k=5):
    return set(np.argsort(np.linalg.norm(db - q, axis=1))[:k].tolist())

truth = exact_topk(q, db, 5)
ivf_res = set(SimpleIVF(nlist=64, nprobe=8).fit(db).search(q, 5).tolist())
print("IVF recall@5 (clustered, nprobe=8):", len(truth & ivf_res) / 5)
# => 1.0   (recall hoàn hảo vì nprobe đủ và data có cụm)
```
**Đã chạy thật, output `recall@5 = 1.0`.** Tăng `nprobe` → recall lên, latency lên. Đó là **cái núm** bạn xoay ở production.

### 3.3. Product Quantization (PQ) — nén để sống sót ở billion-scale

Vấn đề bộ nhớ: 1 tỷ vector × 768 chiều × 4 byte (float32) = **~3 TB RAM**. Không khả thi. PQ chia mỗi vector thành `m` đoạn con, mỗi đoạn được thay bằng **id của centroid gần nhất** (codebook 256 entry → 1 byte/đoạn). Ví dụ 768 float (3072 byte) → `m=96` byte → **giảm ~32 lần**. Đánh đổi: khoảng cách trở thành *xấp xỉ* (lossy). Ghép IVF + PQ = **IVF-PQ**, xương sống của billion-scale search (FAISS, Milvus). Bản hiện đại: **Binary/Scalar Quantization** (Qdrant có binary quantization giảm memory tới ~40 lần).

### 3.4. LSH (Locality-Sensitive Hashing) — băm để cụm thứ giống nhau

Ý tưởng: thiết kế hàm hash sao cho **vector giống nhau rơi vào cùng bucket** với xác suất cao. Query → hash → chỉ so với vector cùng bucket. Có **đảm bảo lý thuyết (provable guarantees)** đẹp, nhưng thực tế cần **nhiều bảng hash** để đạt recall tốt → tốn memory và khó tune. Vì vậy **LSH đã bị HNSW vượt mặt** ở hầu hết use case dày đặc (dense), dù bài gốc vẫn liệt kê nó như đặc trưng trung tâm — đó là góc nhìn hơi cũ.

### 3.5. HNSW — ông vua hiện tại (đào sâu nhất)

**HNSW = Hierarchical Navigable Small World** (Malkov & Yashunin, 2016). Từ đó tới nay là nền tảng của **hầu hết** dedicated vector DB: Pinecone, Weaviate, Qdrant, Milvus, pgvector, Redis, Elasticsearch.

**Trực giác (analogy hệ thống đường bộ):**
- **Layer trên cùng:** ít node, cạnh **đường dài** — như **cao tốc/motorway**, giúp nhảy khoảng cách lớn thật nhanh.
- **Layer giữa:** nhiều node hơn, cạnh ngắn hơn — như **quốc lộ/tỉnh lộ**.
- **Layer 0 (đáy):** chứa **toàn bộ** node, kết nối dày với hàng xóm gần — như **đường phố** để tìm chính xác.

**Cách search:** vào từ một entry point ở layer cao nhất → **greedy** đi tới node gần query nhất (dùng cao tốc để tới đúng "vùng") → tụt xuống layer dưới → lặp lại → tới layer 0 chạy **beam search** rộng `ef_search` để lấy k kết quả. Nhờ tầng, độ phức tạp trung bình ≈ **O(log N)** — đây là lý do HNSW scale từ nghìn tới tỷ vector mà latency chỉ tăng theo log.

**3 tham số phải thuộc:**
| Tham số | Điều khiển | Tăng lên thì... |
|---|---|---|
| **`M`** | số cạnh/hàng xóm mỗi node | recall ↑, **memory ↑**, build chậm hơn |
| **`ef_construction`** | độ rộng beam khi *xây* index | recall ↑, **build time ↑**, không ảnh hưởng query |
| **`ef_search`** | độ rộng beam khi *query* | recall ↑, **latency ↑** (núm chỉnh lúc runtime) |

Default an toàn thực chiến (theo kinh nghiệm chung 2026): **`M=32`, `ef_construction=400`**, query bắt đầu **`ef_search=100`** rồi đẩy lên tới khi chạm trần latency; nếu recall vẫn < ~95% thì rebuild với `M=48/64`.

**Tự implement để *thấy vì sao HNSW được thiết kế như vậy*** — và một bài học đắt giá:

```python
import numpy as np, heapq
from sklearn.datasets import make_blobs

X, _ = make_blobs(n_samples=5000, n_features=128, centers=50, random_state=0)
db = X.astype('float32')
q = db[123] + np.random.rand(128) * 0.1
truth = set(np.argsort(np.linalg.norm(db - q, axis=1))[:5].tolist())

# --- Greedy search trên đồ thị (lõi của HNSW, ở đây rút gọn 1 tầng) ---
def greedy(graph, q, entry, k=5, ef=64):
    vis = {entry}; d0 = np.linalg.norm(db[entry] - q)
    cand = [(d0, entry)]; best = [(-d0, entry)]   # min-heap ứng viên / max-heap top-ef
    while cand:
        dist, node = heapq.heappop(cand)
        if -best[0][0] < dist and len(best) >= ef:
            break                                  # không còn hy vọng tốt hơn
        for nb in graph[node]:
            if nb in vis: continue
            vis.add(nb); dnb = np.linalg.norm(db[nb] - q)
            if len(best) < ef or dnb < -best[0][0]:
                heapq.heappush(cand, (dnb, nb)); heapq.heappush(best, (-dnb, nb))
                if len(best) > ef: heapq.heappop(best)
    return [i for _, i in sorted([(-x[0], x[1]) for x in best])[:k]]

# (1) Đồ thị k-NN THUẦN (chỉ nối hàng xóm gần) — 1 tầng
def knn_graph(db, M=16):
    return {i: np.argsort(np.linalg.norm(db - db[i], axis=1))[1:M+1].tolist()
            for i in range(len(db))}

g1 = knn_graph(db, M=16)
r1 = set(greedy(g1, q, entry=0, k=5, ef=64))
print("k-NN graph thuần   recall@5:", len(truth & r1) / 5)   # => 0.0  (!!)

# (2) Thêm CẠNH ĐƯỜNG DÀI ngẫu nhiên (tính chất "small-world" = chữ S,W trong HNSW)
def small_world_graph(db, M=16, long_range=6, seed=0):
    rng = np.random.default_rng(seed); n = len(db); g = {}
    for i in range(n):
        near = np.argsort(np.linalg.norm(db - db[i], axis=1))[1:M+1].tolist()
        far = rng.choice(n, size=long_range, replace=False).tolist()  # "cao tốc"
        g[i] = list(set(near + far))
    return g

g2 = small_world_graph(db, M=16, long_range=6)
r2 = set(greedy(g2, q, entry=0, k=5, ef=64))
print("small-world graph  recall@5:", len(truth & r2) / 5)     # => 1.0
```
**Kết quả đã chạy thật:**
```
k-NN graph thuần   recall@5: 0.0
small-world graph  recall@5: 1.0
```
👉 **Đây là khoảnh khắc "aha" của cả bài.** Đồ thị chỉ-nối-hàng-xóm-gần khiến greedy search **kẹt trong cụm sai** (entry point ở cụm khác, không có "cao tốc" để nhảy sang cụm của query) → recall = 0. Chỉ cần thêm vài **cạnh đường dài (long-range edges)**, greedy thoát bẫy → recall = 1.0. **Đó chính xác là lý do HNSW cần các *tầng trên* với cạnh dài.** Bạn vừa tự tay chứng minh động lực thiết kế của HNSW — nói được điều này trong phỏng vấn là *ăn điểm tuyệt đối*.

### 3.6. Bảng so sánh các index (thứ interviewer yêu thích)

| Index | Recall | Query speed | Memory | Update/Delete | Build time | Khi nào dùng |
|---|---|---|---|---|---|---|
| **Flat / brute-force** | 100% (exact) | Chậm O(N) | Trung bình | Dễ | 0 | N nhỏ (<~100k), cần exact, làm ground-truth |
| **IVF** | Trung bình–cao (tune `nprobe`) | Nhanh | Trung bình | Khá dễ | Cần train | Dataset lớn, cần cân bằng |
| **IVF-PQ** | Trung bình (lossy) | Rất nhanh | **Rất thấp** ⭐ | Rebuild-friendly hơn HNSW | Cần train | **Billion-scale**, RAM là nút thắt |
| **LSH** | Thấp–trung bình | Nhanh | Cao (nhiều bảng) | Khá dễ | Nhanh | Ít dùng cho dense hiện nay |
| **HNSW** | **Cao–rất cao** ⭐ (tune `ef`) | **Rất nhanh** ⭐ O(log N) | **Cao** (lưu đồ thị) | **Kém** (soft-delete + rebuild) | Chậm | Mặc định cho đa số production |
| **DiskANN** | Cao | Trung bình (đọc SSD) | Thấp (graph trên SSD) | Khá | Chậm | Billion-scale trên **một máy** không cần RAM khổng lồ |

### 3.7. Edge cases phải xử lý

- **Filtered search (metadata filtering):** "tìm vector giống + `WHERE category='news' AND lang='vi'`". Nếu lọc *sau* ANN (post-filter) có thể ra rỗng; lọc *trong lúc* traverse (in-graph filtering như Qdrant/Weaviate) nhanh và đúng hơn 2–3 lần. Đây là **tính năng quyết định** ở production, không phải chi tiết phụ.
- **High-dimensional + dữ liệu không có cụm:** ANN thoái hóa về gần brute-force; cân nhắc giảm chiều (dimensionality reduction) hoặc chấp nhận recall thấp.
- **Duplicate / near-duplicate vectors** làm lệch phân bố cụm IVF → rebalance hoặc dedup trước.
- **Cold start của HNSW:** entry point kém khi index còn nhỏ; nhiều impl chọn entry ngẫu nhiên/nhiều entry để giảm bẫy.

---

## Phần 4 — 🟣 STAFF LEVEL (Tư duy hệ thống & lãnh đạo kỹ thuật)

Đây là phần phân biệt senior với staff. Câu hỏi không còn là *"index nào nhanh?"* mà là *"hệ thống này sống thế nào với 1 tỷ vector, chi phí bao nhiêu, hỏng thì sao, và tôi giải thích cho CFO thế nào?"*

### 4.1. Thiết kế ở quy mô tỷ bản ghi — bottleneck ở đâu?

**Con số buộc phải nhẩm được:** 1 tỷ vector × 768 chiều × 4 byte = **~3 TB** chỉ riêng raw vectors, *chưa kể* overhead đồ thị HNSW (mỗi node lưu M cạnh). HNSW thuần ở scale này **không nhét nổi vào RAM một máy** → đây là **bottleneck #1: memory**.

**Các đòn bẩy scale (theo thứ tự staff thường cân nhắc):**
1. **Quantization trước tiên:** PQ / binary quantization giảm memory 4–40 lần. Thường là đòn *rẻ nhất, hiệu quả nhất*.
2. **Sharding (scale ngang):** chia index thành nhiều **shard** theo id/namespace, mỗi shard một node. Query **fan-out** tới các shard rồi **merge** top-k (scatter-gather). Đây là mô hình của Milvus/Vespa/Elasticsearch.
3. **Tách compute khỏi storage:** kiến trúc hiện đại (Milvus, Pinecone serverless) tách **ingest**, **index build**, **query** để scale độc lập — build index nặng CPU/GPU, query nặng RAM/latency, không nên nhốt chung.
4. **DiskANN** khi muốn billion-scale trên *ít máy*: để đồ thị trên SSD, đọc có chủ đích → đổi chút latency lấy chi phí RAM rẻ hơn nhiều lần.
5. **Replication** cho QPS cao & fault tolerance: nhân bản mỗi shard.

**Bottleneck #2: tail latency (p99).** Trung bình 20ms nhưng p99 200ms sẽ giết trải nghiệm. Nguồn: shard chậm nhất trong fan-out (straggler), GC pause, filter đắt. Xử lý: hedged requests, cắt `ef_search` động, cân bằng shard.

### 4.2. Trade-off ở tầm kiến trúc — khi nào NÊN và KHÔNG NÊN

**NÊN dùng dedicated vector DB khi:** vector search là *core workload*; >10–50M vector; cần latency thấp ổn định ở QPS cao; cần hybrid/filtered search mạnh.

**KHÔNG NÊN (một staff biết nói "không"):**
- **Dữ liệu < ~1–10M vector và bạn đã chạy Postgres** → dùng **pgvector**, đừng dựng thêm một hệ phân tán để nuôi. Ít hạ tầng = ít lỗi = rẻ hơn. (Đây là lựa chọn đúng cho ~70% ứng dụng RAG.)
- **Bài toán thực ra là keyword/exact match** → BM25/Elasticsearch truyền thống tốt và rẻ hơn; đừng "AI hóa" thứ không cần.
- **Cần strong consistency/ACID, transaction phức tạp** → hầu hết dedicated vector DB **không có ACID**; pgvector (trên Postgres) mới có.
- **Prototype nhỏ** → Chroma/FAISS embedded, chưa cần managed service đắt đỏ.

➕ **Hybrid search là mặc định ở production 2026, không phải tùy chọn.** Pure vector search **thua** khi query chứa tên riêng, mã sản phẩm, version number, ID — những thứ cần **exact match**. Kết hợp **vector (semantic) + BM25 (keyword) + metadata filter** cho chất lượng retrieval tốt hơn hẳn. Weaviate/Vespa/Qdrant hỗ trợ native; pgvector phải tự ghép. Nói được câu này = tư duy retrieval trưởng thành.

### 4.3. Chi phí, latency, reliability, monitoring

**Cost (rất thực tế, đây là thứ khiến bạn khác biệt):**
- Managed (Pinecone serverless): ~**\$0.33/GB/tháng** storage + phí read/write units. Ở 10M vector ≈ vài chục USD/tháng — cạnh tranh. Nhưng ở **100M vector, chi phí đội lên nhanh** (hàng trăm–nghìn USD/tháng), trong khi self-host Milvus/Qdrant có thể giữ dưới ~\$100–800/tháng (đổi lại tốn công vận hành/DevOps).
- **Quy tắc staff:** managed thắng khi team nhỏ / muốn zero-ops / scale khó đoán; self-host thắng khi scale lớn ổn định và có năng lực DevOps. **Tính TCO (total cost of ownership), không chỉ giá sticker** — lương kỹ sư vận hành cũng là chi phí.

**Reliability & failure modes:**
- **Index rebuild là "release artifact":** coi mỗi lần build index như một bản release — version hóa embedding + index, validate **recall@k và p99** trong CI/CD trước khi promote, rollback được về version trước. Đổi embedding model = phải **re-embed + rebuild toàn bộ** (vector cũ và mới **không so sánh được** với nhau).
- **Failure modes cần giám sát:** shard chết (mất một phần recall — "âm thầm" hỏng, không throw lỗi!), OOM khi HNSW phình, recall **trôi (drift)** khi phân bố dữ liệu đổi, hot shard.
- **Monitoring bắt buộc:** **recall@k trên tập ground-truth** (chỉ số dễ bị bỏ quên nhất — hệ thống *chạy* nhưng trả kết quả *sai* mà không báo lỗi!), p50/p95/p99 latency, QPS, memory/shard, index build time, freshness (độ trễ từ lúc ghi tới lúc searchable).

### 4.4. Ảnh hưởng tổ chức (staff = nhân số, không chỉ code)

**Giải thích cho stakeholder non-technical:** *"Vector search giúp sản phẩm hiểu **ý nghĩa** chứ không chỉ **từ khóa** — người dùng gõ 'áo ấm đi tuyết' vẫn ra áo phao dù mô tả sản phẩm không chứa đúng chữ đó. Đánh đổi: nó **gần đúng chứ không tuyệt đối chính xác**, và độ chính xác/tốc độ/chi phí là ba núm điều chỉnh mà ta phải cân theo túi tiền và kỳ vọng người dùng."* — dùng ngôn ngữ **đánh đổi và giá trị kinh doanh**, không dùng "HNSW/recall".

**Ảnh hưởng roadmap/team:** chọn managed (Pinecone) = ít headcount vận hành, nhanh ra thị trường, nhưng vendor lock-in + chi phí đội theo scale + không air-gap được (không hợp compliance nghiêm ngặt). Chọn self-host = làm chủ, rẻ ở scale lớn, nhưng cần đội DevOps và lịch rebuild. Đây là quyết định **staff phải trình bày trade-off cho leadership**, không tự quyết trong im lặng.

### 4.5. 🎤 Câu hỏi system design mẫu + hướng trả lời của staff

> **Đề: "Thiết kế semantic search cho 500 triệu tài liệu, p99 < 100ms, 5.000 QPS, có filter theo `tenant_id` và `language`."**

Hướng trả lời của một staff engineer (nói *khung tư duy*, không phang ngay giải pháp):

1. **Làm rõ requirement trước (đây là điểm staff):** recall mục tiêu bao nhiêu (95%? 99%)? Update thường xuyên hay gần như tĩnh? Đọc nhiều hay ghi nhiều? Ngân sách? Yêu cầu compliance (data residency)? → *quyết định kiến trúc, không phải index.*
2. **Ước lượng số:** 500M × 768 × 4B ≈ 1.5TB raw → không nhét 1 máy → **cần sharding + có thể quantization**.
3. **Chọn index & metric:** **HNSW** cho recall/latency tốt (hoặc **IVF-PQ/DiskANN** nếu memory là nút thắt); metric theo embedding model (thường cosine → normalize).
4. **Kiến trúc:** shard theo `tenant_id` (đồng thời **giải quyết luôn filter** — mỗi tenant một namespace, tránh cross-tenant leak); mỗi shard replicate ×2–3 cho QPS + HA; query = scatter-gather + merge top-k; **in-graph filtering** cho `language`.
5. **Đáp p99:** cắt `ef_search` động, hedged requests chống straggler, cache query nóng, tách compute/storage.
6. **Vận hành:** pipeline re-embed/rebuild coi như release có gate recall@k; monitor recall + p99 + freshness; kế hoạch khi đổi embedding model.
7. **Trình bày trade-off:** managed vs self-host, chi phí, khi nào hybrid search. → **Kết bằng đánh đổi, không bằng "đây là câu trả lời duy nhất đúng".**

---

## Phần 5 — 🎯 CHỐT LẠI ĐỂ ĐI PHỎNG VẤN (Interview Cheatsheet)

### 5.1. Keywords bắt buộc nhớ (định nghĩa 1 dòng)

- **Embedding / Vector:** dãy số biểu diễn ý nghĩa; thứ giống nhau → gần nhau.
- **Similarity / Nearest Neighbor (NN) search:** tìm k vector gần query nhất — nghiệp vụ trung tâm.
- **ANN (Approximate NN):** đánh đổi recall lấy tốc độ; linh hồn của mọi vector DB.
- **Recall@k:** tỉ lệ kết quả đúng trong top-k so với exact; thước đo chất lượng ANN.
- **Distance metric:** L2 (Euclidean), Cosine (góc), Inner/Dot product.
- **Flat / brute-force:** so với tất cả; exact; O(N·d); dùng làm ground-truth.
- **IVF (Inverted File):** phân cụm k-means; tune `nlist`/`nprobe`.
- **PQ / IVF-PQ (Product Quantization):** nén vector giảm memory tới hàng chục lần; lossy; cho billion-scale.
- **LSH (Locality-Sensitive Hashing):** hash để cụm thứ giống nhau; đã bị HNSW vượt cho dense.
- **HNSW:** đồ thị nhiều tầng, small-world; O(log N); mặc định hiện nay; tune `M`/`ef_construction`/`ef_search`.
- **DiskANN:** đồ thị trên SSD cho billion-scale ít RAM.
- **Curse of dimensionality:** ở chiều cao mọi điểm cách đều → KD-tree sụp.
- **Hybrid search:** vector + BM25 (keyword) + metadata filter.
- **Filtered / metadata search:** kết hợp similarity với điều kiện; in-graph filtering nhanh hơn post-filter.
- **Dedicated vector DB:** Pinecone, Qdrant, Weaviate, Milvus, Chroma.
- **DB hỗ trợ vector search:** **pgvector** (Postgres), Elasticsearch/OpenSearch, MongoDB, Redis, SingleStore, Cassandra.
- **RAG (Retrieval-Augmented Generation):** use case chính kéo vector DB bùng nổ.

### 5.2. Core concepts (nếu chỉ nhớ mấy điều này thì nhớ cái này)

1. Vector DB tồn tại để làm **semantic similarity search** — thứ SQL/B-tree không làm được.
2. **Brute-force = O(N·d) = chết ở scale** → sinh ra **ANN**, đánh đổi **recall ↔ latency ↔ memory**.
3. **HNSW là mặc định hiện tại**: đồ thị nhiều tầng, cạnh dài (small-world) giúp thoát bẫy cụm, cho **O(log N)**.
4. **IVF-PQ / quantization** là chìa khóa **billion-scale** vì memory (không phải tốc độ) mới là nút thắt lớn nhất.
5. **Dedicated vs supports-vector-search:** pgvector đủ cho ~70% app (<10M vector, đã có Postgres); dedicated khi vector search là core & scale lớn.
6. **Hybrid search** (vector + keyword + filter) thắng pure vector ở production thật.
7. Ở scale lớn: **shard + replicate + tách compute/storage**; giám sát **recall@k** (hỏng âm thầm) và **p99**.
8. **Staff biết nói "không"**: đừng dựng vector DB phân tán khi bài toán nhỏ hoặc thực ra là keyword match.

### 5.3. Ideas / mental models (nói cho trôi & gây ấn tượng)

- **"Vị trí trên kệ = ý nghĩa"** (analogy thư viện xếp theo chủ đề).
- **HNSW = hệ thống đường bộ**: cao tốc (layer trên, cạnh dài) để tới đúng vùng, đường phố (layer 0) để tìm chính xác.
- **"Curse of dimensionality"**: ở chiều cao mọi điểm cách đều nhau nên cây tìm kiếm vô dụng.
- **Ba núm điều chỉnh**: recall ↔ latency ↔ memory — chỉnh cái nào cũng ảnh hưởng hai cái kia.
- **"Index build là release artifact"**: version hóa, gate recall trong CI/CD, rollback được.
- **"ANN hỏng âm thầm"**: hệ thống *chạy* nhưng trả *kết quả sai* mà không throw lỗi → phải monitor recall.

### 5.4. Code cần thuộc lòng (interviewer hay bắt viết tại chỗ)

**(1) Brute-force top-k — phải viết được trong 60 giây:**
```python
import numpy as np
def brute_force_topk(query, db, k=5):
    # db: (N, d), query: (d,)  -> trả về index của k vector gần nhất (L2)
    dists = np.linalg.norm(db - query, axis=1)   # (N,) khoảng cách tới từng vector
    return np.argsort(dists)[:k]                  # id của k cái nhỏ nhất
```
*Khi nào dùng:* làm **ground-truth** để đo recall của ANN; khi N nhỏ; khi cần exact.

**(2) Cosine similarity (nhớ chuẩn hóa!):**
```python
def cosine_topk(query, db, k=5):
    q = query / np.linalg.norm(query)
    d = db / np.linalg.norm(db, axis=1, keepdims=True)   # normalize từng hàng
    sims = d @ q                                          # dot product = cosine
    return np.argsort(-sims)[:k]                          # sim cao nhất lên đầu
```
*Khi nào dùng:* text embedding (đa số dùng cosine). Mẹo: **normalize rồi dùng inner product** = tương đương cosine nhưng nhanh hơn.

**(3) Ý tưởng IVF trong ~10 dòng (giải thích được là đủ, không cần thuộc từng ký tự):**
```python
# 1) k-means chia db thành nlist cụm  -> centroids, buckets{cluster: [ids]}
# 2) query: tìm nprobe centroid gần nhất
# 3) chỉ brute-force trong các bucket đó  -> giảm N xuống ~ nprobe*N/nlist
```

### 5.5. Câu hỏi phỏng vấn thường gặp + gợi ý trả lời (có câu bẫy)

1. **"Vì sao không dùng SQL/B-tree index cho similarity search?"** → B-tree tối ưu exact/range trên giá trị có thứ tự; không mô hình hóa "độ gần trong không gian nhiều chiều". Cần ANN index (HNSW/IVF).

2. **"Exact hay approximate? Trade-off?"** *(câu bẫy kinh điển)* → Approximate (ANN) cho production vì exact O(N·d) không scale. Trade-off = **recall ↔ latency ↔ memory**, đo bằng recall@k; núm chỉnh: `nprobe` (IVF) / `ef_search` (HNSW).

3. **"HNSW hoạt động thế nào? Vì sao nhanh?"** → Đồ thị nhiều tầng small-world; layer trên cạnh dài để nhảy nhanh tới vùng đúng, layer 0 tìm chính xác; greedy + beam search; **O(log N)**. *Ăn điểm thêm:* nêu cạnh dài giúp thoát bẫy "kẹt trong cụm sai" (bạn đã tự chứng minh bằng code).

4. **"Billion vector, bottleneck ở đâu?"** *(câu bẫy — nhiều người trả lời "tốc độ")* → **Memory trước, không phải tốc độ.** 1B×768×4B ≈ 3TB. Giải: **quantization (PQ/binary)** + **sharding** + có thể **DiskANN**.

5. **"Khi nào KHÔNG nên dùng dedicated vector DB?"** *(câu bẫy về judgment)* → Khi <~10M vector và đã có Postgres → **pgvector**; khi bài toán thực ra là keyword match → BM25; khi cần ACID/transaction; prototype nhỏ → Chroma/FAISS. **Staff biết nói không.**

6. **"pgvector vs Pinecone, chọn gì?"** → pgvector khi vector là phụ, muốn 1 hệ thống, cần JOIN/filter SQL, <10–50M vector, cần ACID. Pinecone khi vector search là core, cần managed/zero-ops/scale lớn, chấp nhận chi phí cao & vendor lock-in.

7. **"Kết quả tệ dần theo thời gian dù code không đổi — sao?"** *(câu bẫy vận hành)* → Recall **drift** do phân bố dữ liệu đổi / index cần rebuild / đổi embedding model mà không re-embed. Cần monitor recall@k trên ground-truth + lịch rebuild + version hóa embedding.

8. **"Pure vector search có đủ cho RAG không?"** → Thường **không**: thua với tên riêng/ID/version cần exact match → dùng **hybrid search** (vector + BM25 + filter).

### 5.6. One-liner đắt giá (câu chốt sắc bén thể hiện độ sâu)

- *"Ở scale lớn, nút thắt là **memory chứ không phải tốc độ** — nên câu hỏi đầu tiên của tôi luôn là quantization và sharding, không phải chọn index nào."*
- *"HNSW nhanh vì các **cạnh đường dài** biến greedy search từ 'đi bộ trong một cụm' thành 'đi cao tốc giữa các cụm' — thiếu chúng, recall có thể rơi về 0."*
- *"ANN **hỏng âm thầm**: hệ thống vẫn trả kết quả, chỉ là sai — nên tôi monitor **recall@k trên ground-truth**, không chỉ latency."*
- *"Với ~70% ứng dụng RAG, **pgvector là câu trả lời đúng** — dựng một hệ phân tán chỉ vì nghe 'sexy' là nợ kỹ thuật."*
- *"Tôi coi mỗi lần **rebuild index như một release**: version hóa, gate bằng recall và p99 trong CI/CD, rollback được."*
- *"Production thật gần như luôn là **hybrid search** — semantic bắt ý nghĩa, keyword bắt tên riêng/ID mà embedding hay bỏ sót."*

---

### 📌 Phụ lục: những chỗ bài giảng gốc sai/lỗi thời (để bạn không học nhầm)

1. **"PostGIS cho vector"** → **SAI.** PostGIS là geospatial; extension đúng là **`pgvector`**.
2. **TorchServe là in-memory vector DB** → **SAI.** Là model-serving framework của PyTorch.
3. **FAISS là "distributed database"** → **SAI.** Là *thư viện* chạy 1 máy (có GPU); người ta xây DB phân tán *lên trên* nó.
4. **Annoy/ScaNN là "database"** → chưa chuẩn: chúng là *library* (Annoy = tree-based, index tĩnh).
5. **RedisAI là câu chuyện Redis-vector hiện tại** → lỗi thời; nay là **Redis Stack / RediSearch** (HNSW).
6. **Taxonomy 5 kiểu (in-memory/disk/distributed/graph/time-series)** → lỏng và trộn trục; thực chiến phân loại theo **index algorithm + deployment + managed/self-host**.
7. **Bài gốc gần như bỏ qua ANN, recall, HNSW, hybrid search, và toàn bộ thế hệ dedicated DB mới (Pinecone/Qdrant/Weaviate/Chroma)** — đó lại chính là phần quan trọng nhất năm 2026.

> Phần đúng và giá trị nhất của bài gốc: **phân biệt dedicated vector DB vs database hỗ trợ vector search**, và **danh sách các đặc trưng của dedicated DB** (chuyên biệt, tối ưu NN, scale, tune được). Hãy giữ phần đó, bỏ phần còn lại.
